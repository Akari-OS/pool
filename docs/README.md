# AkariPool ドキュメント

> 最終更新: 2026-04-09

**AkariPool は [AkariOS](../../PJ26c21_AkariOS) の中核データ層。**
AI エージェントのための汎用 Knowledge Store。動画・音声・画像・PDF・記事・コード・データセット — どんなモダリティのデータでも入る、ローカルファーストのデータ基盤。

プロジェクト全体の北極星・設計思想は上位 `CLAUDE.md` を参照。このリポジトリのドキュメントは **Knowledge Store としての要件・設計・進捗** を扱う。

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
└── handoff-YYYY-MM-DD.md  ← セッション末スナップショット
```

各ディレクトリの役割：

- **design/** — 「何を作るか」「どう作るか」。要件定義（`requirements.md`）と各モジュールの詳細設計を置く
- **integration/** — AkariPool と外部システム（Obsidian / AKARI Video 等）との接続設計
- **planning/** — 「いつ・どの順で作るか」。Phase ロードマップと実行可能タスクリスト
- **handoff-\*.md** — セッション跨ぎの状態スナップショット。Phase 完了や大きな区切りで作成し、次セッション開始時のウォームアップ文書として使う

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

1. **`../CLAUDE.md`** — プロジェクト北極星・設計思想・技術スタック
2. **`README.md`（本ファイル）** — ドキュメント全体の地図
3. **`planning/roadmap.md`** — Phase ビジョンと依存関係
4. **`design/requirements.md`** — FR / NFR / AC（何を満たせば完了か）
5. **`design/architecture.md`** — 全体アーキテクチャ
6. **`planning/tasks.md`** — 今の実装状況と次のタスク
7. **最新の `handoff-*.md`** — セッション再開のための state スナップショット

個別トピックを深掘りするときは `design/` 配下の該当ファイルへ。
