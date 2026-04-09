# AkariPool — タスク分解

> 最終更新: 2026-04-09
> 要件トレース: `../design/requirements.md`
> 上位ロードマップ: `roadmap.md`

---

## 現在地 — Phase 4 完了（2026-04-08 @ `7a089d4`）

Phase 0〜4 を完走。次は **Phase 4.5（Linter 完成 + Relation CLI）**。

### 実装済みコンポーネント一覧

| コンポーネント | 状態 | Phase | コミット |
|---|:-:|:-:|---|
| Cargo workspace + 設計書 | ✅ | 0 | — |
| pool-core schema v1 + WAL | ✅ | 1 | `d5755fb` |
| Pool 基本 CRUD | ✅ | 1 | `d5755fb` |
| MIME 自動判定（8 モダリティ） | ✅ | 1 | `d5755fb` |
| CLI `info / add / ls / cat / rm` | ✅ | 1 | `d5755fb` |
| Workspace 隔離 + `workspace.toml` | ✅ | 2 | `906f972` |
| `GlobalConfig` + カレント解決 | ✅ | 2 | `906f972` |
| FTS5 全文検索 + `--cross` | ✅ | 2 | `906f972` |
| パストラバーサル対策 | ✅ | 2 | `906f972` |
| Analyzer trait + Registry | ✅ | 3 | `d17b987` |
| LlmClient + OpenAiCompatible | ✅ | 3 | `d17b987` |
| MockLlmClient | ✅ | 3 | `d17b987` |
| ArticleAnalyzer（.md MVP） | ✅ | 3 | `d17b987` |
| `.env` 自動読み込み | ✅ | 3 | `d17b987` |
| `add --analyze` / `analyze` CLI | ✅ | 3 | `d17b987` |
| FTS5 trigram（日本語部分一致） | ✅ | 3.5 | `54e3af5` |
| HTML 対応（html2text） | ✅ | 3.5 | `54e3af5` |
| RetryPolicy + Retry-After | ✅ | 3.5 | `54e3af5` |
| `ArticleFormat` enum + 空ファイル検出 | ✅ | 3.5 | `54e3af5` |
| Wiki Compile（Obsidian 互換） | ✅ | 4 | `7a089d4` |
| Relations CRUD + 型 | ✅ | 4 | `7a089d4` |
| Linter（Missing + Orphan） | ✅ | 4 | `7a089d4` |
| `compile-notes` / `lint` CLI | ✅ | 4 | `7a089d4` |
| `add --analyze` → auto-compile 連鎖 | ✅ | 4 | `7a089d4` |

### テスト状況（Phase 4 完了時点）

- `cargo test --workspace` → **47/47 passed**
- pool-core: 28 tests
- pool-lint: 3 tests
- analyzer-article: 16 tests

### 先行実装 / 積み残し

現時点では該当なし。Phase 0〜4 のスコープは全て完走。
Phase 3 の追加 Analyzer（Image / Audio / PDF / Code / Video / Dataset）は Phase 3 積み残しとして FR-04-08 〜 FR-04-13 に定義済み、Phase 4.5 完了後の選択肢に残る。

---

## Phase 1 — pool-core 基本 CRUD ✅

- [x] **T1.1** SQLite schema v1 + WAL 適用 — `crates/pool-core/src/schema.rs` (FR-01-01, FR-01-02)
- [x] **T1.2** `Pool::open` 実装 — `crates/pool-core/src/pool.rs` (FR-01-01)
- [x] **T1.3** `add_item / list_items / get_item / delete_item / count_items` (FR-02-01 〜 FR-02-06)
- [x] **T1.4** MIME 自動判定（8 モダリティ） (FR-02-02)
- [x] **T1.5** CLI `info / add / ls / cat / rm` — `crates/pool-cli/src/main.rs` (FR-09-01 〜 FR-09-05)
- [x] **T1.6** `~/.cargo/bin/akari-pool` へのインストール
- [x] **AC**: `akari-pool add ./file` → `ls` で可視、`cat` で取得、`rm` で削除まで一通り動く

---

## Phase 2 — Workspace 隔離 + 横断検索 ✅

- [x] **T2.1** `Pool` を workspace マネージャに refactor（`HashMap<String, Arc<Workspace>>`） (FR-01-09)
- [x] **T2.2** `Workspace` 構造体 + CRUD + `workspace.toml` (FR-01-03, FR-01-04, FR-01-08)
- [x] **T2.3** `GlobalConfig` (`~/.akari-pool/config.toml`) (FR-01-07)
- [x] **T2.4** カレント workspace 解決（`--workspace` > env > config > `"default"`） (FR-01-05)
- [x] **T2.5** パストラバーサル対策（lexical 検査） (FR-01-06, NFR-04-03)
- [x] **T2.6** FTS5 検索（カレント + `--cross`） (FR-03-01, FR-03-02, FR-03-04)
- [x] **T2.7** `ItemType / Role / Layer` に `Display + FromStr` (FR-02-08)
- [x] **T2.8** `db.rs` 削除 → `pool.rs` + `workspace.rs` に責務分離
- [x] **T2.9** Review 修正: SQL injection / cross_search 取りこぼし / パストラバーサルバイパス / DRY
- [x] **AC**: `workspace create / list / use / info / delete --confirm` が動作、`search --cross` で全 workspace 横断可能

---

## Phase 3 — Analyzer trait + LlmClient + ArticleAnalyzer ✅

- [x] **T3.1** `LlmClient` trait + `OpenAiCompatibleClient` (FR-05-01, FR-05-03)
- [x] **T3.2** `MockLlmClient`（テスト用） (FR-05-02)
- [x] **T3.3** `Analyzer` trait + `AnalyzerContext`（`llm_client: Arc<dyn LlmClient>`） (FR-04-01, FR-04-02, FR-04-03)
- [x] **T3.4** `analyzers/article` crate: pulldown-cmark アウトライン抽出 + LLM 要約 + タグ生成 (FR-04-04)
- [x] **T3.5** `Workspace::analyze_item` + `add_item_and_analyze`（パーシャル成功対応） (FR-04-06)
- [x] **T3.6** `update_analysis_result` で DB 保存（ai_summary / ai_tags / context_json） (FR-04-07)
- [x] **T3.7** CLI: `add --analyze` + `analyze <id>` (FR-09-02, FR-09-08)
- [x] **T3.8** `.env` 自動読み込み（dotenvy、`~/.akari-pool/.env` → カレント順） (FR-05-04)
- [x] **T3.9** Review 修正: strip_code_fence / context_json 読み返し / L3_full_text カット / FTS5 エスケープ / CLI 重複
- [x] **AC**: `add --analyze` で `.md` が要約・タグ付けされ DB に保存、失敗しても item は保持

---

## Phase 3.5 — 既知の制限を潰す ✅

- [x] **T3.5.1** FTS5 トークナイザを unicode61 → trigram に変更（日本語部分一致） (FR-03-03)
- [x] **T3.5.2** `html2text` で `.html` / `.htm` 対応 (FR-04-04)
- [x] **T3.5.3** `RetryPolicy` + exponential backoff + `Retry-After` 尊重（最大 5 分キャップ） (FR-05-05)
- [x] **T3.5.4** `complete_once` 分離（責務分割）
- [x] **T3.5.5** `ArticleFormat` enum + `detect_format` + 空ファイル検出 (FR-04-05)
- [x] **T3.5.6** Review 修正: Retry-After 上限 / html_to_text 空チェック / last_error デッドコード / コメント番号
- [x] **AC**: 日本語 3 文字以上で部分一致ヒット、`.html` が analyze できる、429 時にリトライする

---

## Phase 4 — Wiki Compile + Relations + Linter ✅

- [x] **T4.1** `wiki.rs::compile_item_to_markdown(item, workspace_name, related)` 本実装 (FR-06-01)
- [x] **T4.2** YAML frontmatter + 本文（要約・タグ・構造・関連・原文抜粋） (FR-06-02)
- [x] **T4.3** `yaml_escape`（`\n` / `\r` / `\t` エスケープ） (FR-06-04)
- [x] **T4.4** `sanitize_tag`（Obsidian 互換、CJK 保持） (FR-06-03)
- [x] **T4.5** `RelatedItemView` 型で backlink 表示 (FR-06-05)
- [x] **T4.6** 原文抜粋 2000 文字カット + コードフェンス (FR-06-06)
- [x] **T4.7** `NewRelation` 型 + `add_relation / list_relations / list_item_relations` (FR-07-01, FR-07-02)
- [x] **T4.8** `RelationType` / `CreatedBy` + `Display + FromStr` (FR-07-03, FR-07-04)
- [x] **T4.9** `Linter` struct + `LintFinding` 型 + `Severity` enum (FR-08-01)
- [x] **T4.10** `check_missing`: analyzed_at NULL → warn (FR-08-02)
- [x] **T4.11** `check_orphan`: relations に登場しないアイテム → info (FR-08-03)
- [x] **T4.12** `run_and_save`: 既存 unresolved 上書き (FR-08-06)
- [x] **T4.13** CLI `compile-notes [--item <id>]` (FR-09-09)
- [x] **T4.14** CLI `lint` (FR-09-10)
- [x] **T4.15** `add --analyze` / `analyze` 成功時の auto-compile 連鎖 (FR-06-07)
- [x] **T4.16** Review 修正: YAML 改行エスケープ / `?1` 二重バインド / 変換関数統合 / compile_all_notes fail-fast 解消
- [x] **AC**: `compile-notes` で Obsidian 互換 .md 生成、`lint` で Missing / Orphan 検出 & DB 保存、`analyze` → auto-compile 連鎖

---

## Phase 4.5 — Linter 完成 + Relation CLI 🟡 次

Phase 4 で積み残した Lint チェッカー 2 種を完成させ、Relation を CLI から操作可能にする。

### 規模: 中 / 依存: Phase 4（Relations CRUD + Linter フレームワーク）

- [ ] **T4.5.1** `Linter::check_inconsistency` 実装 (FR-08-04)
  - [ ] 編集距離ベースのタグ揺れ検出（レーベンシュタイン、n-gram）
  - [ ] LLM による正規化提案（OpenRouter 経由）
  - [ ] `Severity::Warn` で finding 生成
  - [ ] 単体テスト（タグ集合を入力、期待 finding を検証）
- [ ] **T4.5.2** `Linter::check_connection_gap` 実装 (FR-08-05)
  - [ ] `pool_relations` からグラフを構築
  - [ ] A→B / B→C はあるが A→C が無いケースを探索
  - [ ] `Severity::Info` で finding 生成
  - [ ] 推移探索の深さ上限を設定（爆発防止）
- [ ] **T4.5.3** Relation CLI 実装 — `crates/pool-cli/src/main.rs` (FR-07-06, FR-09-11)
  - [ ] `akari-pool relation add <from> <to> --type <type> [--strength <f>] [--note <text>]`
  - [ ] `akari-pool relation list [--item <id>]`
  - [ ] `akari-pool relation rm <relation_id> --confirm`
- [ ] **T4.5.4** `akari-pool lint show [--severity <s>]` 実装 (FR-08-07, FR-09-12)
  - [ ] DB 保存済み（resolved_at IS NULL）の findings を列挙
  - [ ] severity フィルタ
  - [ ] 件数サマリ表示
- [ ] **T4.5.5** `docs/design/` に `linter-extended.md` を追加（4 種の検出ロジックと閾値）
- [ ] **T4.5.6** Review ラウンド（2〜3 並列）: アルゴリズム計算量 / LLM プロンプト / エッジケース / CLI UX
- [ ] **T4.5.7** 実機スモーク: 10〜20 アイテム + 数件 relation で 4 種の finding を目視確認
- [ ] **T4.5.8** `cargo test --workspace` + `cargo install --path crates/pool-cli --force`
- [ ] **T4.5.9** commit `[機能追加] Phase 4.5 — Linter 拡張 + Relation CLI`
- [ ] **AC**: 4 種全ての Lint チェッカーが動作、Relation を CLI から追加・一覧・削除できる、`lint show` で保存済み findings を severity フィルタ付きで閲覧できる

---

## Phase 5 — Filed-back ループ ⬜

### 規模: 大 / 依存: Phase 4（Wiki Compile）

- [ ] **T5.1** 設計書新規作成 `docs/design/filed-back-loop.md`
- [ ] **T5.2** ファイル変更監視（`notify` crate）or `akari-pool sync` コマンド (FR-09-14)
- [ ] **T5.3** frontmatter 差分抽出（ai_summary / ai_tags の編集を検知）
- [ ] **T5.4** `[[item-id]]` 記法 → relation 自動作成 (FR-07-07)
- [ ] **T5.5** モデルプリセット UI（CLI フラグ） (FR-05-08)
  - [ ] 💰節約 / ⚖️バランス / ⚡パフォーマンス / 🔒オフライン / 🧪ハイブリッド / 🎓高品質
- [ ] **T5.6** 3 層設定（global → workspace.toml → セッション/CLI） (FR-05-09)
- [ ] **AC**: Obsidian で編集 → `sync` で Pool 反映、`[[item-id]]` で relation 作成、6 プリセットが CLI から選べる

---

## Phase 6 — MCP サーバー ⬜

### 規模: 中〜大 / 依存: Phase 4（全機能）

- [ ] **T6.1** `rmcp` crate 最新バージョン確認 + `pool-mcp` 実装 (FR-10-01)
- [ ] **T6.2** stdio + Streamable HTTP 両対応 (FR-10-02)
- [ ] **T6.3** MCP ツール: `pool_ls / pool_cat / pool_grep / pool_search / pool_add / pool_lint / pool_compile_notes` (FR-10-03)
- [ ] **T6.4** MCP リソース: `pool://item/{id}` / `pool://workspace/{name}` / `m2c://item/{id}/context` (FR-10-04)
- [ ] **T6.5** workspace 引数はセッション初期化時に固定 (FR-10-05)
- [ ] **T6.6** `akari-pool mcp-serve [--http] [--port]` (FR-09-13)
- [ ] **T6.7** AKARI Video の既存 MCP と整合性確認 (FR-10-06)
- [ ] **T6.8** 実機テスト: Claude Code から接続し `pool_search` を呼ぶ
- [ ] **AC**: Claude Code / Cursor から MCP で AkariPool に接続し、ツールを呼べる

---

## Phase 7 — AKARI Video 統合 ⬜

### 規模: 大 / 依存: Phase 6（外部 AI 接続基盤）

- [ ] **T7.1** AKARI Video 側の Pool DB 層を `pool-core` に置換 (FR-11-01)
- [ ] **T7.2** `VideoAnalyzer` 実装（FFmpeg + Vision、M2C video） (FR-04-12)
- [ ] **T7.3** 動画系 workspace（例: `wedding-2026`）での実運用テスト
- [ ] **T7.4** AKARI Video 側の回帰テスト
- [ ] **AC**: AKARI Video が `pool-core` だけで動作、`add video.mp4 --analyze` で字幕・シーン情報が DB に入る

---

## Phase 8 — 別アプリ実装 ⬜

### 規模: 非常に大 / 依存: Phase 7

- [ ] **T8.1** AkariNotes（Obsidian 風ビュー、ノート系アイテム） (FR-11-03)
- [ ] **T8.2** AkariCMS（記事系アイテム編集 CMS）
- [ ] **T8.3** AkariSearch（全 workspace 横断検索 Web UI）
- [ ] **T8.4** AkariBot（Slack/Discord で Pool 検索）
- [ ] **AC**: 少なくとも 2 つの別消費者アプリが同じ pool.db を参照して動く

---

## Phase 3 積み残し — 追加 Analyzer ⬜（Phase 4.5 完了後の選択肢）

Phase 4.5 が完了したあと、Phase 5 に進む前に Image / Audio / PDF のいずれかを先に入れる選択肢がある。

- [ ] **TX.1** `ImageAnalyzer`（Vision API、Gemini Flash） (FR-04-08)
- [ ] **TX.2** `AudioAnalyzer`（Whisper + ffmpeg + BGM 検出） (FR-04-09)
- [ ] **TX.3** `PDFAnalyzer`（PyMuPDF / LlamaParse） (FR-04-10)
- [ ] **TX.4** `CodeAnalyzer`（tree-sitter AST） (FR-04-11)
- [ ] **TX.5** `DatasetAnalyzer`（polars schema） (FR-04-13)

> これらは Phase 3 のフレームワーク上で動く独立タスク。いつ着手するかは Phase 4.5 完了時点で判断する。
