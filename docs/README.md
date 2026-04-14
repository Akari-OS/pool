# AkariPool Documentation（公開版）

> **このリポの立ち位置**: AkariPool の**公開ドキュメント版**。仕様・API リファレンス・Getting Started を公開する場所。
> **扱う範囲**: 公開 API、プロトコル仕様、Getting Started、外部統合ガイド、公開 Phase ロードマップ
> **扱わない範囲**: Rust 実装コード（→ `PJ26c23_AkariPool`）、非公開戦略・競合分析（→ Hub `PJ26c21_AkariOS/docs/strategy/`）
>
> - 🌐 正典: [Akari-OS](https://github.com/Akari-OS)
> - 🏛 Hub（非公開）: `PJ26c21_AkariOS` — 横断研究・戦略・Master Index
> - 🗺 全リポマップ: `PJ26c21_AkariOS/MAP.md`

---

> 最終更新: 2026-04-13

**AkariPool は AKARI エコシステムの中核データ層。**
AI エージェントのための汎用 Knowledge Store。動画・音声・画像・PDF・記事・コード・データセット — どんなモダリティのデータでも入る、ローカルファーストのデータ基盤。

実装リポ（非公開・Rust 本体）は `PJ26c23_AkariPool`。本リポはその公開ドキュメント版。

## ドキュメント駆動開発

**最重要ルール: ドキュメント → コードの順序を必ず守る。**

新機能は以下のフローで進める：

1. `design/requirements.md` に FR / NFR / AC を追記
2. `planning/roadmap.md` の該当 Phase を更新（必要なら新 Phase を追加）
3. `planning/tasks.md` に `T<phase>.<n>` のタスクを展開（対応 FR 番号を明記）
4. 詳細設計は `design/<topic>.md` または `design/architecture.md` の該当節に
5. 実装 → テスト → AC 検証 → commit
6. tasks.md のチェックを埋めて handoff を更新

---

## 構造

```
docs/
├── README.md              ← 本ファイル（入口）
├── design/                ← 要件・アーキテクチャ・個別設計
├── integration/           ← 外部システム統合
├── planning/              ← ロードマップ・タスク管理
├── api/                   ← 公開 API リファレンス
└── handoff-YYYY-MM-DD.md  ← セッション末スナップショット
```

---

## 主要文書

### design/ — 要件 & 設計

| ファイル | 内容 |
|---|---|
| [requirements.md](design/requirements.md) | 機能要件（FR）・非機能要件（NFR）・受入条件（AC）・ユースケース |
| [architecture.md](design/architecture.md) | 全体アーキテクチャ — crate 構成、データモデル、MCP/CLI 層 |
| [workspace-layer.md](design/workspace-layer.md) | Workspace 隔離モデル、現在 workspace の解決、横断検索 |
| [analyzer-plugin.md](design/analyzer-plugin.md) | Analyzer trait、モダリティ別プラグイン設計、Registry |
| [llm-strategy.md](design/llm-strategy.md) | LLM モデル戦略 — OpenRouter 主力、プリセット 6 種、リトライ |
| [akari-video-integration.md](design/akari-video-integration.md) | AKARI Video 側を pool-core 依存に切り替える段取り |

### integration/ — 外部統合

| ファイル | 内容 |
|---|---|
| [obsidian-plugin.md](integration/obsidian-plugin.md) | Obsidian プラグインによる Wiki 層の閲覧・編集 |

### planning/ — 計画 & 進捗

| ファイル | 内容 |
|---|---|
| [roadmap.md](planning/roadmap.md) | Phase 別ロードマップ — ビジョン・依存関係・完了基準 |
| [tasks.md](planning/tasks.md) | タスク分解・実装状況・チェックボックス形式の実行単位 |

### handoff/ — セッションスナップショット

| ファイル | 内容 |
|---|---|
| [handoff-2026-04-08.md](handoff-2026-04-08.md) | Phase 4 完了時点（最新） |

---

## 新規参加者の読む順序

1. **本 README**（全体の地図）
2. **`planning/roadmap.md`** — Phase ビジョンと依存関係
3. **`design/requirements.md`** — FR / NFR / AC（何を満たせば完了か）
4. **`design/architecture.md`** — 全体アーキテクチャ
5. **`planning/tasks.md`** — 今の実装状況と次のタスク
6. **最新の `handoff-*.md`** — セッション再開のための state スナップショット

個別トピックを深掘りするときは `design/` 配下の該当ファイルへ。
