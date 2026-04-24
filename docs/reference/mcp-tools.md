# MCP Tools Reference

> **対象**: AkariPool の MCP サーバー (`pool-mcp` crate) が提供する全ツールの一覧
> **生成元**: `akari-pool-impl/crates/pool-mcp/src/server.rs` の `#[tool(...)]` 注釈（2026-04-24 時点で **34 ツール**）
> **呼び出し**: Claude Code / Cursor 等の MCP クライアントから `akari-pool mcp-serve` 経由で利用

このリポジトリは公開ドキュメント側のため、各ツールの実装本体・入力 struct 定義・エラー型は実装リポ側を参照のこと。詳細パラメータは [`akari-pool-impl/crates/pool-mcp/src/server.rs`](https://github.com/Akari-OS/pool-impl/blob/main/crates/pool-mcp/src/server.rs) を見るのが最も正確。

---

## 1. Library 操作（メタ・一覧系）

Library の存在確認・Item 一覧・関連取得を提供する。Library 自体の作成・削除は CLI (`akari-pool library create|use|info|delete`) に分離されている。

| Tool | 概要 |
|---|---|
| `pool_libraries` | AkariPool の Library 一覧を取得する |
| `pool_ls` | 指定 Library のアイテム一覧を取得する。`item_type` でフィルタ可 |
| `pool_filter` | 指定 Library のアイテムを `role` / `layer` / `sort` 等の条件でフィルタ取得 |
| `pool_relations` | Library 内のリレーション一覧を取得。`item_id` 指定時はそのアイテムの関連のみ |

## 2. Item 追加・取り込み

ファイル / URL / テキストの 3 経路から Library に Item を投入する。MIME 自動判定・メタデータ抽出・Analyzer 起動は内部で分岐される。

| Tool | 概要 |
|---|---|
| `pool_add` | ファイルを指定 Library に追加する。MIME タイプは自動判定 |
| `pool_add_url` | URL を fetch して指定 Library に追加。Web ページ / PDF / 画像などに対応 |
| `pool_add_text` | コピーしたテキストを markdown item として指定 Library に追加 |
| `pool_cat` | 指定アイテムの全フィールドを取得する |
| `pool_add_relation` | 2 つのアイテム間にリレーションを追加する |
| `pool_archive` | ワークスペースの内容を Library の `raw/{target_relpath}/` にコピーして archive する |

## 3. 検索（Search）

Library 内部の FTS / grep と、`shared/index.db` 経由の横断インデックス検索の 2 系統を提供する。

| Tool | 概要 |
|---|---|
| `pool_search` | AkariPool の全文検索。Library 指定で絞り込み、省略で全 Library 横断検索 |
| `pool_grep` | Library 内のアイテムの生テキスト内容を文字列パターンで検索 |
| `pool_index_refresh` | Pool / Work / Session ファイルを走査し `shared/index.db` を再構築する |
| `pool_index_search` | `shared/index.db` で Pool / Work / Session のテキストファイルを横断検索する |

## 4. 整合性チェック / コンパイル

| Tool | 概要 |
|---|---|
| `pool_lint` | Library の健全性チェックを実行する（`missing` / `orphan` / `inconsistency` / `connection_gap`） |
| `pool_compile_notes` | Library 内のアイテムを Markdown にコンパイルする（Wiki Compile） |

## 5. Layer 別 Read / List（低レベル）

Library 内の `raw/` `items/` `categories/` `notes/` レイヤーに対する読み取り用ツール。LLM から直接ファイルを覗く用途。

| Tool | 概要 |
|---|---|
| `pool_read` | Library の指定レイヤー（`raw` / `items` / `categories` / `notes`）内のファイルを読む |
| `pool_list` | Library の指定レイヤー（`raw` / `items` / `categories`）内のファイル一覧を JSON 配列で返す |

## 6. Workspace / Session ファイル操作

ワークスペース（セッションスコープの作業領域）の CRUD と atomic write を提供する。`session_id` は `YYYY-MM-DD-slug` 形式。

| Tool | 概要 |
|---|---|
| `workspace_create` | セッションスコープのワークスペースを作成する |
| `workspace_list` | ワークスペース一覧を返す。`library` で絞り込み可 |
| `workspace_read_focus` | ワークスペースの `focus.md` をパースして JSON で返す |
| `session_append` | ワークスペースのセッションファイルに追記（OS-level atomic append） |
| `session_write_file` | ワークスペース内のファイルを atomic write（tmp+rename） |
| `session_read_file` | ワークスペース内のファイルを読む |

## 7. Workflow / Agent

YAML ベースの Workflow を走らせて、承認待ちを挟みつつ Agent を呼び出す系統。`workflow_run` → `workflow_status` → `workflow_approve|reject` → `workflow_run` 再開、というループを想定。

| Tool | 概要 |
|---|---|
| `workflow_run` | Workflow YAML を読み込んで実行を開始。承認待ち step で pause、最後まで進めば completed |
| `workflow_status` | `session_id` 配下の `workflow-state.yaml` を読んで JSON で返す |
| `workflow_approve` | 承認待ちの step を承認して run を再開 |
| `workflow_reject` | 承認待ち step を却下して run を failed にする |
| `workflow_list` | 全 workspaces 配下の `workflow-state.yaml` を列挙。`status` でフィルタ可 |
| `agent_invoke` | 指定 Agent (partner / researcher / writer / ユーザー定義) を 1 回呼び出し、会話を `workspaces/{session_id}/conversation.jsonl` に追記 |

## 8. Focus / Selection 操作

ワークスペース内の `focus.md` ＝ 「いま何に集中しているか」を記述したファイルを機械読み書きするための系統。

| Tool | 概要 |
|---|---|
| `update_current_selection` | `focus.md` の `## Current Selection` を更新 |
| `clear_current_selection` | `focus.md` の `## Current Selection` をクリア |
| `get_current_selection` | `focus.md` の `## Current Selection` を JSON で返す。未設定なら null |
| `session_update_focus` | `focus.md` を部分更新（`current_focus` / `status` / `add_decision` / `add_next_action`） |

---

## MCP Resources

ツールに加え、MCP の Resource として以下のテンプレートが登録されている（実装は `server.rs` の `list_resource_templates` / `list_resources` / `read_resource`）:

- `pool://item/{id}` — Item の全フィールド
- `pool://library/{name}` — Library のメタ
- `m2c://item/{id}/context` — Item の M2C context_json（Semantic Layer）

---

## 変更履歴メモ

- **2026-04-24**: 初版。`workspace-session-layer.md:205` が「合計 21 ツール」と記載していた状態から、HUB-017 追加分 + workflow + focus 系を合わせて 34 ツールに揃えた
- 今後のツール追加時は `server.rs` の `#[tool(...)]` 注釈と本ファイルを同時更新する（実装単独の変更は docs 側に反映漏れが出やすい — §整合性ルールとして明記予定）
