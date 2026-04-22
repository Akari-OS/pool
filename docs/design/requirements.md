# AkariPool — 要件定義

> 最終更新: 2026-04-22

本文書は AkariPool の機能要件（FR）・非機能要件（NFR）・受入条件（AC）を定義する。
各 Phase のタスクは `../planning/tasks.md` で、FR 番号を通じて本文書と双方向トレース可能。

---

## 1. ユーザー像（Personas）

| ID | 名称 | 説明 |
|----|------|------|
| P1 | AI エージェント | Claude Code / Cursor / 自作 Agent 等。MCP 経由で AkariPool を叩き、知識を検索・追加・分析する |
| P2 | 個人の知識蓄積者 | 開発者個人ユースケースを想定。動画・記事・ノート等を日常的に投入し、後から横断検索して再利用する |
| P3 | 消費者アプリ開発者 | AKARI Video / 将来の AkariNotes / AkariCMS / AkariSearch 等。pool-core crate を依存に取り込み、モダリティ特化の編集ビューを提供する |

---

## 2. 主要ユースケース（User Stories）

| ID | As a | I want to | So that |
|----|------|-----------|---------|
| UC-01 | P2 | 動画ファイルを workspace に追加し、AI に分析させる | 後から内容で検索できる |
| UC-02 | P2 | 記事 `.md` を analyze → Wiki Compile → Obsidian で閲覧する | 機械可読と人間可読の両形式で知識を持てる |
| UC-03 | P1/P2 | 特定の workspace 内で全文検索する | 目的別の作業空間内に閉じた調査ができる |
| UC-04 | P1/P2 | `--cross` で全 workspace 横断検索する | 文脈を越えて過去の知見を引ける |
| UC-05 | P2 | Lint を実行して知識ベースの健全性を監視する | 表記揺れ・関係抜け・未分析アイテムを早期発見できる |
| UC-06 | P1 | MCP 経由で pool_search / pool_add 等のツールを呼ぶ | エージェントがユーザー機の知識に直接アクセスできる |
| UC-07 | P2 | Obsidian で Wiki ノートを編集し、AkariPool に反映する（Phase 5） | 人間・AI 共同編集が成立する |
| UC-08 | P3 | pool-core crate を依存に追加して Workspace::search を叩く | 消費者アプリ間で同じ Pool を共有できる |
| UC-09 | P1 | アイテム間の関係（related / derived / cites 等）を追加する | 知識グラフが育っていく |
| UC-10 | P2 | モデルプリセット（節約/バランス/高品質）を切り替える | コストと品質のバランスを状況に応じて制御できる |

---

## 3. 機能要件（Functional Requirements）

### FR-01: Pool & Workspace 基盤

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-01-01 | `Pool::open(root)` で AkariPool ルートを初期化し、SQLite schema v1 を自動適用 | Must | ✅ Phase 1 |
| FR-01-02 | WAL モードで SQLite を開く | Must | ✅ Phase 1 |
| FR-01-03 | Workspace の作成 / 一覧 / 情報取得 / 削除 | Must | ✅ Phase 2 |
| FR-01-04 | 各 Library は独立した `pool.db` / `files/` / `notes/` / `library.toml` を持つ | Must | ✅ Phase 2 |
| FR-01-05 | 現在 Library の解決順序: `--library` フラグ > `AKARI_POOL_LIBRARY` env > `config.toml` > `"default"` | Must | ✅ Phase 2 |
| FR-01-06 | パストラバーサル対策（library 名の lexical 検査） | Must | ✅ Phase 2 |
| FR-01-07 | `GlobalConfig` を `~/.akari-pool/config.toml` に永続化（current_library 等） | Must | ✅ Phase 2 |
| FR-01-08 | Workspace に display 名・icon を設定可能 | Should | ✅ Phase 2 |
| FR-01-09 | `Pool` 内で Workspace を `HashMap<String, Arc<Workspace>>` キャッシュ | Should | ✅ Phase 2 |

### FR-02: アイテム CRUD

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-02-01 | `add_item(path, options)` で raw ファイルを `files/{uuid}/` に保存し DB に登録 | Must | ✅ Phase 1 |
| FR-02-02 | MIME 自動判定で `ItemType`（Video / Audio / Image / Pdf / Article / Code / Dataset / Note / Custom）を決定 | Must | ✅ Phase 1 |
| FR-02-03 | `list_items(filter)` で ItemType / タグ / limit 等でフィルタ一覧 | Must | ✅ Phase 1 |
| FR-02-04 | `get_item(id)` で 1 件取得 | Must | ✅ Phase 1 |
| FR-02-05 | `delete_item(id)` で DB エントリ + raw ファイルを削除 | Must | ✅ Phase 1 |
| FR-02-06 | `count_items()` で件数取得 | Should | ✅ Phase 1 |
| FR-02-07 | アイテムに Role（source / derived / output 等）・Layer（L1/L2/L3）を持たせる | Should | ✅ Phase 1 |
| FR-02-08 | `ItemType` / `Role` / `Layer` に `Display + FromStr` を実装 | Should | ✅ Phase 2 |

### FR-03: 検索（FTS5 + trigram）

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-03-01 | カレント workspace 内の全文検索 | Must | ✅ Phase 2 |
| FR-03-02 | `--cross` で全 workspace 横断検索 | Must | ✅ Phase 2 |
| FR-03-03 | 日本語部分一致（FTS5 trigram トークナイザ） | Must | ✅ Phase 3.5 |
| FR-03-04 | SQL インジェクション対策（FTS5 エスケープ） | Must | ✅ Phase 2 |
| FR-03-05 | 1〜2 文字クエリは空結果を返す（trigram の性質上） | Accepted Limitation | ✅ Phase 3.5 |
| FR-03-06 | 埋め込みベース類似検索（sqlite-vec） | Could | ⬜ Phase 5+ |

### FR-04: Analyzer 基盤

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-04-01 | `Analyzer` trait（`analyze(ctx, item) -> AnalysisResult`）を `pool-core` に定義 | Must | ✅ Phase 3 |
| FR-04-02 | `AnalyzerRegistry` で ItemType ごとに Analyzer を登録・解決 | Must | ✅ Phase 3 |
| FR-04-03 | `AnalyzerContext` に LLM クライアント・workspace ハンドルを注入 | Must | ✅ Phase 3 |
| FR-04-04 | `ArticleAnalyzer`: `.md` / `.html` 対応、pulldown-cmark + html2text、LLM で要約 + タグ抽出 | Must | ✅ Phase 3 / 3.5 |
| FR-04-05 | `ArticleFormat` enum + `detect_format` + 空ファイル検出 | Must | ✅ Phase 3.5 |
| FR-04-06 | `Workspace::analyze_item` + `add_item_and_analyze`（パーシャル成功対応） | Must | ✅ Phase 3 |
| FR-04-07 | `update_analysis_result` で ai_summary / ai_tags / context_json を DB 保存 | Must | ✅ Phase 3 |
| FR-04-08 | `ImageAnalyzer`（Vision API、Gemini Flash 想定） | Should | ⬜ Phase 3 積み残し |
| FR-04-09 | `AudioAnalyzer`（Whisper + BGM 検出） | Should | ⬜ Phase 3 積み残し |
| FR-04-10 | `PDFAnalyzer`（PyMuPDF / LlamaParse） | Should | ⬜ Phase 3 積み残し |
| FR-04-11 | `CodeAnalyzer`（tree-sitter AST） | Could | ⬜ Phase 3 積み残し |
| FR-04-12 | `VideoAnalyzer`（FFmpeg + Vision、M2C video） | Could | ⬜ Phase 3 積み残し |
| FR-04-13 | `DatasetAnalyzer`（pandas/polars schema） | Could | ⬜ Phase 3 積み残し |

### FR-05: LLM クライアント

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-05-01 | `LlmClient` trait + `OpenAiCompatibleClient`（OpenRouter / Ollama / vLLM / LM Studio） | Must | ✅ Phase 3 |
| FR-05-02 | `MockLlmClient`（テスト用） | Must | ✅ Phase 3 |
| FR-05-03 | 標準セットアップは `OPENROUTER_API_KEY` 1 つだけで動く | Must | ✅ Phase 3 |
| FR-05-04 | `.env` 自動読み込み（`~/.akari-pool/.env` → カレント順） | Must | ✅ Phase 3 |
| FR-05-05 | `RetryPolicy` + exponential backoff + `Retry-After` 尊重（最大 5 分キャップ） | Must | ✅ Phase 3.5 |
| FR-05-06 | 主力モデル: DeepSeek V3.2 / Qwen3 / GLM-4.6 / Gemini 2.5 Flash | Must | ✅ Phase 3 |
| FR-05-07 | Claude Sonnet/Opus/Haiku / GPT-4o は禁止リスト | Must | ✅ Phase 3 |
| FR-05-08 | モデルプリセット 6 種（💰節約/⚖️バランス/⚡パフォーマンス/🔒オフライン/🧪ハイブリッド/🎓高品質） | Should | ⬜ Phase 5+ |
| FR-05-09 | 3 層設定（グローバル config → workspace.toml → CLI フラグ） | Should | ⬜ Phase 5+ |

### FR-06: Wiki Compile

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-06-01 | `compile_item_to_markdown` で notes/{uuid}.md を生成 | Must | ✅ Phase 4 |
| FR-06-02 | YAML frontmatter + 本文（要約・タグ・構造・関連・原文抜粋） | Must | ✅ Phase 4 |
| FR-06-03 | Obsidian 互換のハッシュタグ（CJK 保持、`sanitize_tag`） | Must | ✅ Phase 4 |
| FR-06-04 | YAML エスケープ（`\n` / `\r` / `\t` 等） | Must | ✅ Phase 4 |
| FR-06-05 | `RelatedItemView` で backlink 表示 | Must | ✅ Phase 4 |
| FR-06-06 | 原文抜粋は 2000 文字カット + コードフェンスで囲む | Must | ✅ Phase 4 |
| FR-06-07 | `add --analyze` / `analyze` 成功時に auto-compile 連鎖 | Must | ✅ Phase 4 |
| FR-06-08 | Workspace 跨ぎの relation は compile 対象外（v1 制限） | Accepted Limitation | ✅ Phase 4 |
| FR-06-09 | Obsidian 双方向同期（.md 編集 → Pool 反映） | Should | ⬜ Phase 5 |

### FR-07: Relations CRUD

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-07-01 | `pool_relations` テーブル + `PoolRelation` 型 | Must | ✅ Phase 4 |
| FR-07-02 | `add_relation / list_relations / list_item_relations` API | Must | ✅ Phase 4 |
| FR-07-03 | `RelationType`（related / derived / cites / ...） + `CreatedBy`（user / ai / system） | Must | ✅ Phase 4 |
| FR-07-04 | `RelationType` / `CreatedBy` に `Display + FromStr` | Must | ✅ Phase 4 |
| FR-07-05 | Workspace 跨ぎ relation（`target_workspace` 非 NULL） | Should | ✅ Phase 4 |
| FR-07-06 | Relation CLI（`akari-pool relation add / list / rm`） | Must | ⬜ Phase 4.5 |
| FR-07-07 | Filed-back ループ（会話から自動追記） | Should | ⬜ Phase 5 |

### FR-08: Linter（知識ベース健全性）

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-08-01 | `Linter` struct + `LintFinding` 型 + `Severity` enum | Must | ✅ Phase 4 |
| FR-08-02 | Missing チェッカー: analyzed_at が NULL のアイテムを warn | Must | ✅ Phase 4 |
| FR-08-03 | Orphan チェッカー: relations に登場しないアイテムを info | Must | ✅ Phase 4 |
| FR-08-04 | Inconsistency チェッカー: 編集距離 + LLM でタグ表記揺れ検出 | Must | ⬜ Phase 4.5 |
| FR-08-05 | ConnectionGap チェッカー: A→B / B→C はあるが A→C が無いケースをグラフ探索 | Must | ⬜ Phase 4.5 |
| FR-08-06 | `run_and_save` で lint findings を DB 永続化（既存 unresolved を上書き） | Must | ✅ Phase 4 |
| FR-08-07 | `akari-pool lint show` で保存済み findings を閲覧 | Must | ⬜ Phase 4.5 |
| FR-08-08 | バックグラウンド定期実行 | Could | ⬜ Phase 5+ |

### FR-09: CLI 一式

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-09-01 | `akari-pool info` — Pool 情報表示 | Must | ✅ Phase 1 |
| FR-09-02 | `akari-pool add <file> [--name] [--role] [--analyze]` | Must | ✅ Phase 1 / 3 |
| FR-09-03 | `akari-pool ls [--item-type] [--limit]` | Must | ✅ Phase 1 |
| FR-09-04 | `akari-pool cat <id>` | Must | ✅ Phase 1 |
| FR-09-05 | `akari-pool rm <id>` | Must | ✅ Phase 1 |
| FR-09-06 | `akari-pool library create / list / info / use / delete` | Must | ✅ Phase 2 |
| FR-09-07 | `akari-pool search "query" [--cross]` | Must | ✅ Phase 2 / 3.5 |
| FR-09-08 | `akari-pool analyze <id>` | Must | ✅ Phase 3 |
| FR-09-09 | `akari-pool compile-notes [--item <id>]` | Must | ✅ Phase 4 |
| FR-09-10 | `akari-pool lint` | Must | ✅ Phase 4 |
| FR-09-11 | `akari-pool relation add / list / rm` | Must | ⬜ Phase 4.5 |
| FR-09-12 | `akari-pool lint show` | Must | ⬜ Phase 4.5 |
| FR-09-13 | `akari-pool mcp-serve [--http] [--port]` | Must | ⬜ Phase 6 |
| FR-09-14 | `akari-pool sync`（Obsidian → Pool 書き戻し） | Should | ⬜ Phase 5 |
| FR-09-15 | `--json` 出力フラグ全コマンド対応 | Should | ⬜ Phase 5+ |

### FR-10: MCP サーバー（Phase 6）

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-10-01 | `pool-mcp` crate を `rmcp` で実装 | Must | ⬜ Phase 6 |
| FR-10-02 | stdio + Streamable HTTP 両対応 | Must | ⬜ Phase 6 |
| FR-10-03 | MCP ツール: `pool_ls / pool_cat / pool_grep / pool_search / pool_add / pool_lint / pool_compile_notes` | Must | ⬜ Phase 6 |
| FR-10-04 | MCP リソース: `pool://item/{id}` / `pool://library/{name}` / `m2c://item/{id}/context` | Must | ⬜ Phase 6 |
| FR-10-05 | library はセッション初期化時に固定 | Must | ⬜ Phase 6 |
| FR-10-06 | AKARI Video の既存 MCP サーバーと整合性を取る | Should | ⬜ Phase 6 |

### FR-11: 統合（Phase 7+）

| ID | 要件 | 優先度 | 状態 |
|----|------|--------|------|
| FR-11-01 | AKARI Video を `pool-core` 依存に切り替え | Must | ⬜ Phase 7 |
| FR-11-02 | Obsidian プラグインで notes/ ディレクトリを閲覧 | Should | ⬜ Phase 7 |
| FR-11-03 | AkariNotes / AkariCMS / AkariSearch の別消費者アプリ | Could | ⬜ Phase 8 |

---

## 4. 非機能要件（Non-Functional Requirements）

### NFR-01: ローカルファースト

| ID | 要件 |
|----|------|
| NFR-01-01 | 全データはユーザーマシンに留まる。ネットワークに出るのは LLM API のみ |
| NFR-01-02 | オフライン時でも CRUD / 検索 / Wiki Compile は動作する（Analyzer は skip） |
| NFR-01-03 | Cloud sync は将来の opt-in、別レイヤー。コアには組み込まない |

### NFR-02: モダリティ中立

| ID | 要件 |
|----|------|
| NFR-02-01 | 動画・音声・画像・PDF・記事・コード・データセット・ノートを同じ抽象で扱う |
| NFR-02-02 | 新モダリティは `Analyzer` trait 実装で追加できる（本体クレート改修不要） |
| NFR-02-03 | `pool-core` にモダリティ固有のコードを置かない |

### NFR-03: パフォーマンス

| ID | 要件 |
|----|------|
| NFR-03-01 | カレント workspace の search は 10 万件規模で 100ms 以内 |
| NFR-03-02 | `analyze` は 1 アイテムあたり 30 秒以内（LLM レイテンシ依存） |
| NFR-03-03 | `compile-notes` は 1000 アイテムで 10 秒以内 |
| NFR-03-04 | SQLite は WAL モードで並行読み取り許容 |

### NFR-04: セキュリティ

| ID | 要件 |
|----|------|
| NFR-04-01 | API キーは `~/.akari-pool/.env`（600 permission、.gitignore 済み） |
| NFR-04-02 | SQL インジェクション対策（バインドパラメータ、FTS5 エスケープ） |
| NFR-04-03 | パストラバーサル対策（workspace 名の lexical 検査） |
| NFR-04-04 | 破壊的コマンド（`rm` / `workspace delete`）は `--confirm` 必須 |
| NFR-04-05 | 外部 API キーをログ・stderr に出力しない |

### NFR-05: ライセンス & 配布

| ID | 要件 |
|----|------|
| NFR-05-01 | AGPL-3.0 でソース公開を担保 |
| NFR-05-02 | Rust crate / CLI バイナリ / MCP サーバー の 3 面で提供する |
| NFR-05-03 | `cargo install --path crates/pool-cli` でバイナリが入る |

### NFR-06: スケール

| ID | 要件 |
|----|------|
| NFR-06-01 | パーソナル規模（〜10 万アイテム / workspace）を想定 |
| NFR-06-02 | Workspace 数は数十程度まで |
| NFR-06-03 | マルチユーザー協調はスコープ外 |

### NFR-07: 開発規約

| ID | 要件 |
|----|------|
| NFR-07-01 | Rust edition 2021 固定（1.85 の edition 2024 は避ける） |
| NFR-07-02 | 変数・関数・ファイル名は英語、コメント・コミットメッセージは日本語 |
| NFR-07-03 | エラーは `thiserror` で型定義、`anyhow` は CLI / 統合層のみ |
| NFR-07-04 | 各 crate に `tests/` を配置。E2E テストは `pool-cli` crate 内に |
| NFR-07-05 | コミットメッセージは `[機能追加] / [バグ修正] / [ドキュメント] / [リファクタ]` プレフィックス |

---

## 5. 受入条件（Acceptance Criteria）

各 FR グループの「これを満たせば完了」を列挙。タスク側（`planning/tasks.md`）の AC 行と対応する。

| FR グループ | 受入条件 |
|---|---|
| FR-01 基盤 | `akari-pool library create` → `library info` で情報取得、`library use` で永続化、`library delete --confirm` で削除まで一通り動く |
| FR-02 CRUD | `add` → `ls` で可視、`cat` で本文取得、`rm` で消せる。MIME 自動判定が 8 モダリティで効く |
| FR-03 検索 | 日本語 3 文字以上のクエリで部分一致ヒット、`--cross` で全 library 横断 |
| FR-04 Analyzer | `add --analyze` で `.md` / `.html` 記事が要約・タグ付けされ DB 保存、失敗しても item は保持 |
| FR-05 LLM | `OPENROUTER_API_KEY` だけで動作、429 / Retry-After 時に指数バックオフでリトライ |
| FR-06 Wiki | `compile-notes` で Obsidian 互換 frontmatter + 本文の .md が生成、`analyze` 成功で自動連鎖 |
| FR-07 Relations | `add_relation` で関係追加、`list_item_relations` で backlink 取得 |
| FR-08 Lint | Missing / Orphan が検出され DB 保存。Phase 4.5 完了時は Inconsistency / ConnectionGap も |
| FR-09 CLI | 上記すべてが CLI から操作可能。`--help` で使い方が出る |
| FR-10 MCP | `akari-pool mcp-serve` で Claude Code / Cursor から接続し、pool_search 等のツールを呼べる（library はセッション初期化時に固定） |
| FR-11 統合 | AKARI Video が pool-core を依存に取り込み、動画系アイテムを編集できる |

---

## 6. スコープ外（Out of Scope）

明示的に「作らない」もの。要件の肥大化を防ぐため定期的に再確認する。

- **GUI** — AkariPool 本体は GUI を持たない。GUI は消費者アプリ（AKARI Video / 将来の AkariNotes 等）の責務
- **Cloud sync** — 将来 opt-in の別レイヤー（AKARI Cloud）として分離。コアには組み込まない
- **マルチユーザー協調** — パーソナル規模想定。複数人同時編集・権限管理は扱わない
- **Federation / P2P** — 横断は同一マシン内の workspace 間のみ
- **Full LLM ローカル化の強制** — ローカル LLM は「選択肢の 1 つ」。デフォルトでは要求しない
- **Undo / Version Stacking** — raw ファイルは immutable。編集履歴は消費者アプリ側の責務
- **認証・アクセス制御** — ローカルファイルシステムの権限に依拠

---

## 7. 実装済みだが本文書に未記載の機能

> 将来の書き起こし時にここに追記する。現時点では該当なし。
