# Workflow Layer — `pool-workflow` crate

> AKARI-HUB-018 / AKARI-HUB-019 の akari-pool 側実装。
> Skill / Workflow / Template の宣言的フレームワーク + Workflow Runner + workflow-state.yaml + Selection（HUB-019）連携を提供する。

---

## 位置づけ

AKARI 5 層で言うと **道具層（揮発）** + **記憶層（永続）** の橋渡し：

- **YAML 定義** (Skill / Workflow / Template) は記憶層に保存（人間可読）
- **Runner** は道具層（揮発、プロセス再起動で消える）
- **workflow-state.yaml** は記憶層に永続化（pause/resume + クラッシュ復旧の媒体）

| 入力 | 出力 |
|---|---|
| Workflow YAML / Template YAML | `~/.akari-pool/workspaces/{session-id}/workflow-state.yaml` |
| `!from-memory` 引数 | 解決済み文字列（runner が `inputs` に注入） |
| 承認シグナル（MCP `workflow_approve`） | 再開された run、最終的に `completed` または次の pause |

## 依存関係

`pool-workflow` → `pool-core`（Workspace / SessionId / Pool アクセス用）

`pool-mcp` → `pool-workflow` + `pool-core`（5 つの workflow_* MCP ツールを exposing）

## モジュール

| モジュール | 役割 | テスト数 |
|---|---|:-:|
| `error.rs` | `WorkflowError` 共通エラー型 | — |
| `skill.rs` | Skill `.md` frontmatter パーサ + 5 カテゴリ | 7 |
| `workflow.rs` | Workflow YAML モデル + toposort + 循環検出 | 6 |
| `template.rs` | Template YAML パーサ + Workspace 起動 + initial-works コピー | 4 |
| `memory.rs` | `!from-memory` resolver（pool: / work: / session: / my:） | 11 |
| `state.rs` | `workflow-state.yaml` の serde + atomic save + list_paused | 3 |
| `runner.rs` | `WorkflowRunner` start/approve/reject + 変数解決 + step 実行 | 7 |

加えて `crates/pool-mcp/tests/workflow_integration.rs` に 3 件の E2E（pause→approve→completed のライフサイクル等）。

## Step 種別と実行戦略

| `type` | モデル化 | 実 Invoker |
|---|---|---|
| `costume` | `StepType::Costume { costume }` | StubInvoker（MVP）。Agent Runtime 実装時に差し替え |
| `workflow` | `StepType::Workflow { workflow }` | StubInvoker。再帰起動は次フェーズ |
| `mcp-app` | `StepType::McpApp { app }` | StubInvoker。HUB-005 Declarative App ランタイム実装時に差し替え |
| `skill` | `StepType::Skill { skill }` | StubInvoker。HUB-024 Skill API 実装時に差し替え |

`StepInvoker` trait を介して差し込み式で拡張可能。MVP の `StubInvoker` は「step を成功扱いし、宣言された outputs を返すだけ」。

## `!from-memory` プレフィックス

| プレフィックス | 解決先（Pool ルート相対） |
|---|---|
| `pool:` | `libraries/{current_library}/<path>` |
| `work:` | `workspaces/{current_session}/<path>` |
| `session:` | 同上（focus.md 等を指す慣例） |
| `my:` | `libraries/{current_library}/items/personal/my-<basename>`（`my-` プレフィックス自動付与） |

セキュリティ：`..` セグメント / 絶対パス / canonicalize 後のルート外参照はすべて即時エラー。サイレントフォールバックなし。

## pause/resume の永続化

`workflow-state.yaml` は HUB-018 §6.10 に準拠：

- `run-id` / `workflow-id` / `session-id` / `status` (running|paused|completed|failed)
- `awaiting-approval` (paused 時のみ): `step-id` / `step-type` / `description`
- `variable-bindings`: workflow inputs と `!from-memory` 解決済み値、`__approved.<step-id>` フラグ
- `step-results`: 完了/失敗 step の `outputs` / `started_at` / `finished_at` / `error`
- `spec-version`: workflow の version。approve 時に現在の workflow YAML と一致しないと拒否

書き込みは tmp + rename の atomic_write。

## MCP ツール（5 本）

`pool-mcp/src/server.rs` の `#[tool_router]` に追加：

| ツール | 引数 | 用途 |
|---|---|---|
| `workflow_run` | `workflow_path` / `session_id` / `library` / `inputs_json?` / `run_id?` | 新規 run を起動。最初の approval ゲートで paused、または最後まで進めば completed |
| `workflow_status` | `session_id` | `workflow-state.yaml` を JSON で返す |
| `workflow_approve` | `session_id` / `workflow_path` / `library` | 承認待ち step を承認して再開 |
| `workflow_reject` | `session_id` / `reason?` / `library` | 承認待ち step を却下、run を failed に |
| `workflow_list` | `status?` | 全 workspaces の workflow-state.yaml を列挙 |

Selection（HUB-019）3 ツールと合わせて、pool-mcp は計 **29 ツール**を提供する。

## 既知の限界（次フェーズ）

- `parallel: true` は記録のみで、現在は同グループ内シリアル実行
- `type: workflow` の再帰起動は未実装
- `${steps.X.outputs.Y}` 形式の step 間参照は未実装（`${inputs.X}` のみ対応）
- Daemon 化（Workflow Runner プロセス常駐）は未着手 — 現在は MCP ツール 1 callで完結する短命実行
- 着ぐるみ（Agent Runtime）連携は StubInvoker のみ

## 関連 spec

- AKARI-HUB-018 `spec-skill-workflow-template-framework` — フレームワーク仕様（v0.4 / accepted）（非公開 Hub リポ管理）
- AKARI-HUB-019 `spec-context-handoff-tools` — Selection（v0.2 / accepted）（非公開 Hub リポ管理）
- AKARI-HUB-017 `spec-memory-layer-structure` — Memory Layer（依存）（非公開 Hub リポ管理）
