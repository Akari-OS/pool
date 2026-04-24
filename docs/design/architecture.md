# AkariPool アーキテクチャ設計書

> **バージョン**: v0.1 Draft
> **最終更新**: 2026-04-08
> **ステータス**: Phase 0–6 完了（2026-04 時点）→ 次は Phase 7（AkariShell 統合、AKARI Video のアプリ化）。詳細進捗は impl 側 `akari-pool-impl/README.md` を正典とする

---

## 1. 目的とスコープ

AkariPool は **AI エージェントが本棚を散歩するように知識を探索できる、ローカルファーストの汎用 Knowledge Store** である。

### スコープ内
- 任意モダリティ（動画/音声/画像/PDF/記事/コード/データセット/ノート等）の取り込み・保存
- AI による自動構造化（M2C 分析）と Wiki 化
- 全文検索 + セマンティック検索 + 関係グラフ探索
- Library による目的別隔離
- Linting（健全性チェック）と Filed back（学習ループ）
- CLI / MCP / Rust crate の3面提供

### スコープ外
- GUI（消費者アプリの責務）
- 動画編集機能（AKARI Video の責務）
- 記事編集機能（AKARI CMS の責務）
- クラウド同期（AKARI Cloud の責務、将来の opt-in レイヤー）
- LLM の選定（OpenRouter/Cloud 連携は呼び出し元の責務）

---

## 2. 設計原則

| # | 原則 | 説明 |
|---|---|---|
| 1 | **モダリティ中立** | Video Analyzer は数ある Plugin の1つ。コアは modality を知らない |
| 2 | **二段構え** | raw（生）+ SQLite（構造）+ Wiki（人間可読 .md）の三段で同じ情報を持つ |
| 3 | **Library 隔離** | デフォルトは現在の library 内のみ。明示で横断 |
| 4 | **GUI を持たない** | 消費者アプリ（AKARI Video 等）に GUI を任せる |
| 5 | **3入口統一** | crate / CLI / MCP すべて同じ pool-core を呼ぶ |
| 6 | **ローカルファースト** | データはユーザーマシンに留まる、Cloud sync は別レイヤー |
| 7 | **拡張性** | Analyzer/Linter/Output は trait で追加可能 |
| 8 | **自己治癒** | Linting で表記揺れ・欠損・関係抜けを自動検出 |
| 9 | **学習ループ** | Filed back で「使うほど賢くなる本棚」 |
| 10 | **AGPL** | OSS 戦略の根幹 |

---

## 3. 全体図

```
┌──────────────────────────────────────────────────────────────────┐
│                       消費者アプリ層                                │
│  AKARI Video  /  AKARI CMS  /  AkariNotes  /  外部AI(Claude等)    │
└────┬─────────────────┬─────────────────┬─────────────────┬───────┘
     │ Rust crate     │ MCP             │ MCP             │ CLI
     │                │                 │                 │
┌────▼────────────────▼─────────────────▼─────────────────▼───────┐
│                       AkariPool 提供層                             │
│  pool-core (crate)  /  pool-mcp (server)  /  pool-cli (binary)   │
└────┬─────────────────────────────────────────────────────────────┘
     │
┌────▼─────────────────────────────────────────────────────────────┐
│                       コアロジック層                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ pool-core    │  │ pool-library  │  │ pool-lint    │            │
│  │ CRUD/Search  │  │ 隔離/横断検索  │  │ 健全性チェック │            │
│  └──────┬───────┘  └──────────────┘  └──────────────┘            │
│         │                                                          │
│  ┌──────▼─────────────────────────────────────────────┐           │
│  │ Analyzer Trait (Plugin)                            │           │
│  │  ├─ Video    🎬 FFmpeg + Vision                    │           │
│  │  ├─ Audio    🎵 Whisper + BGM 検出                 │           │
│  │  ├─ Image    🖼️ Vision API                         │           │
│  │  ├─ PDF      📄 LlamaParse / PyMuPDF                │           │
│  │  ├─ Article  📝 HTML/Markdown parser                │           │
│  │  ├─ Code     💻 tree-sitter AST                     │           │
│  │  ├─ Dataset  📊 pandas/polars schema                │           │
│  │  └─ Note     🧠 markdown + frontmatter              │           │
│  └────────────────────────────────────────────────────┘           │
└──────┬───────────────────────────────────────────────────────────┘
       │
┌──────▼───────────────────────────────────────────────────────────┐
│                       ストレージ層                                  │
│  ~/.akari-pool/                                                   │
│  ├── libraries/{name}/                                            │
│  │   ├── files/{uuid}/      ← raw 層（生ファイル）                   │
│  │   ├── notes/{uuid}.md    ← Wiki 層（人間可読）                   │
│  │   └── pool.db            ← SQLite 層（構造データ + FTS5 + vec）  │
│  └── shared/                                                      │
│      ├── tags/              ← 共通タグ辞書                          │
│      └── templates/         ← レシピテンプレート                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. ストレージ層

### 4-1. ディレクトリ構造

```
~/.akari-pool/
├── libraries/
│   ├── wedding-2026/
│   │   ├── files/
│   │   │   ├── {uuid-1}/
│   │   │   │   └── ceremony.mp4
│   │   │   ├── {uuid-2}/
│   │   │   │   └── bgm.wav
│   │   │   └── ...
│   │   ├── notes/
│   │   │   ├── {uuid-1}.md     ← AI が整形した人間可読版
│   │   │   ├── {uuid-2}.md
│   │   │   └── ...
│   │   └── pool.db              ← SQLite（このlibrary専用）
│   │
│   ├── backend-research/
│   │   ├── files/
│   │   ├── notes/
│   │   └── pool.db
│   │
│   └── ...
│
├── shared/
│   ├── tags.db                 ← 共通タグ辞書
│   ├── templates/
│   └── analyzers/              ← Analyzer プラグイン共通設定
│
└── config.toml                 ← グローバル設定（埋め込みモデル選択等）
```

### 4-2. なぜ library ごとに pool.db を分けるか
- **隔離が物理的に強制される**（誤って横断検索しない）
- **バックアップ/移動が単純**（フォルダごと持っていける）
- **削除が安全**（library 削除 = フォルダ削除のみ）
- **横断検索は呼び出し側で複数 DB を open する**（Phase 2 で実装）

詳細: [`library-layer.md`](./library-layer.md)

### 4-3. SQLite スキーマ（v1 ドラフト）

```sql
-- ワークスペース内の全アイテム
CREATE TABLE pool_items (
    id TEXT PRIMARY KEY,                  -- UUID v4
    name TEXT NOT NULL,
    file_path TEXT,                       -- raw ファイルへの相対パス
    source_path TEXT,                     -- 元の取り込み元
    mime_type TEXT,
    item_type TEXT NOT NULL,              -- video/audio/image/pdf/article/code/dataset/note/...
    size_bytes INTEGER,
    role TEXT,                            -- source/output/reference
    layer TEXT,                           -- content/style/taste
    ai_summary TEXT,                      -- 短い要約
    ai_tags TEXT,                         -- JSON 配列
    context_json TEXT,                    -- 構造化コンテキスト（モダリティ別）
    context_compressed TEXT,              -- 圧縮版（トークン節約用）
    context_embedding BLOB,               -- 埋め込みベクトル
    embedding_model TEXT,
    embedding_dimension INTEGER,
    analyzer_version TEXT,                -- どの Analyzer プラグインで分析したか
    analyzer_raw TEXT,                    -- Analyzer 固有の生データ（ffprobe 等）
    analyzed_at TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- アイテム間の関係グラフ
CREATE TABLE pool_relations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_item_id TEXT NOT NULL,
    target_item_id TEXT NOT NULL,
    relation_type TEXT NOT NULL,          -- derived/related/variant/series/style_used/remix/shared_tag/cross_library
    strength REAL NOT NULL,               -- 0.0-1.0
    created_by TEXT,                      -- ai/user/lint
    reason TEXT,                          -- 推論根拠
    target_library TEXT,                  -- NULL=同library, 値あり=横断リンク
    created_at TEXT NOT NULL,
    FOREIGN KEY (source_item_id) REFERENCES pool_items(id) ON DELETE CASCADE
);

-- 全文検索（FTS5）
CREATE VIRTUAL TABLE pool_items_fts USING fts5(
    name, ai_summary, ai_tags, context_json,
    content='pool_items', content_rowid='rowid'
);

-- Linting 結果のキャッシュ
CREATE TABLE lint_findings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    finding_type TEXT NOT NULL,           -- inconsistency/missing/orphan/connection_gap
    severity TEXT NOT NULL,               -- info/warn/error
    target_item_id TEXT,
    description TEXT NOT NULL,
    suggested_action TEXT,
    detected_at TEXT NOT NULL,
    resolved_at TEXT
);

-- Filed back ログ（学習履歴）
CREATE TABLE filed_back_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source TEXT NOT NULL,                 -- chat_session/user_action/lint
    action_type TEXT NOT NULL,            -- add_relation/add_tag/promote_layer/...
    payload_json TEXT NOT NULL,
    created_at TEXT NOT NULL
);
```

詳細スキーマと migration は Phase 1 で `crates/pool-core/src/schema.rs` に実装。

---

## 5. コアロジック層

### 5-1. クレート構成

| Crate | 責務 | 依存 |
|---|---|---|
| **pool-core** | SQLite 操作、CRUD、Wiki compile、Filed back | rusqlite, serde, tokio |
| **pool-library** | Library 隔離、横断検索、共有レイヤー | pool-core |
| **pool-lint** | 4種の健全性チェック、Linter trait | pool-core |
| **pool-mcp** | MCP server (rmcp) - tools/resources/subscribe | pool-core, pool-library, pool-lint, rmcp |
| **pool-cli** | `akari-pool` バイナリ、clap | 全部 |

### 5-2. Analyzer Plugin モデル

詳細は [`analyzer-plugin.md`](./analyzer-plugin.md)（次ドキュメント）に分離するが、概要:

```rust
#[async_trait]
pub trait Analyzer: Send + Sync {
    /// このアナライザが処理できる MIME type / 拡張子のリスト
    fn supported_types(&self) -> Vec<&'static str>;

    /// アナライザ名（"video-m2c", "pdf-llamaparse" 等）
    fn name(&self) -> &'static str;

    /// バージョン（再分析判定に使う）
    fn version(&self) -> &'static str;

    /// 分析を実行
    async fn analyze(&self, item: &PoolItem, ctx: &AnalyzerContext)
        -> Result<AnalyzeResult, AnalyzeError>;
}

pub struct AnalyzeResult {
    pub summary: String,
    pub tags: Vec<String>,
    pub context_json: serde_json::Value,   // M2C compliant
    pub embedding: Option<Vec<f32>>,
    pub raw: Option<serde_json::Value>,
}
```

### 5-3. Wiki Compile

各 PoolItem を `notes/{uuid}.md` に書き出す。フォーマット例:

```markdown
---
pool_id: {uuid}
name: {name}
type: video
library: wedding-2026
mime_type: video/mp4
analyzed_at: 2026-04-08T16:00:00+09:00
analyzer: video-m2c v0.2
tags: [結婚式, セレモニー, 屋外]
layer: content
duration_sec: 432
---

# {name}

{ai_summary}

## シーン

- **00:00 - 00:45** — 入場シーン。明るい屋外、笑顔多め
- **00:45 - 02:30** — 誓いの言葉。緊張感
- **02:30 - 04:12** — リング交換、BGM フェードイン
- ...

## 関連素材

- [[{related_uuid_1}]] — BGM (style_used, strength=0.8)
- [[{related_uuid_2}]] — リハーサル動画 (variant, strength=0.6)

## メタ
- **役割**: source
- **追加日**: 2026-04-08
- **元のパス**: `~/Movies/wedding/ceremony.mp4`
```

このフォーマットは Obsidian 互換で、Vault としてそのまま開ける。

### 5-4. Linting エンジン

```rust
#[async_trait]
pub trait Linter: Send + Sync {
    fn name(&self) -> &'static str;
    async fn check(&self, lib: &Library) -> Vec<Finding>;
}

pub enum FindingType {
    Inconsistency { items: Vec<ItemId>, reason: String },
    Missing { item: ItemId, missing_field: String },
    Orphan { item: ItemId },
    ConnectionGap { from: ItemId, via: ItemId, to: ItemId },
}
```

Phase 1.5 で実装する4つの Linter:
- `TagInconsistencyLinter` — 表記揺れ
- `MissingAnalysisLinter` — 分析欠損
- `OrphanItemLinter` — 未分類アラート
- `TransitiveRelationLinter` — A→B, B→C あるが A→C なし

### 5-5. Filed back ループ

```rust
pub trait FiledBackHook: Send + Sync {
    /// AI チャット中・ユーザー操作時に呼ばれる
    async fn on_event(&self, event: PoolEvent) -> Vec<FiledBackAction>;
}

pub enum FiledBackAction {
    AddRelation { source: ItemId, target: ItemId, relation: RelationType, strength: f32 },
    AddTag { item: ItemId, tag: String },
    PromoteLayer { item: ItemId, from: Layer, to: Layer },
}
```

実装は Phase 5。

---

## 6. インターフェース層

### 6-1. Rust crate API（pool-core）

```rust
use akari_pool::{Pool, Library, PoolItem, ItemType};

// プールを開く
let pool = Pool::open("~/.akari-pool")?;

// Library を取得 or 作成
let lib = pool.library_or_create("wedding-2026")?;

// アイテム追加（Analyzer 自動選択 + 非同期分析）
let item_id = lib.add_item(AddRequest {
    file_path: "./ceremony.mp4".into(),
    item_type: None,  // None = MIME type から自動判定
    role: Some(Role::Source),
    layer: Some(Layer::Content),
    initial_tags: vec![],
}).await?;

// 検索
let results = lib.search(SearchRequest {
    query: "結婚式 笑顔".into(),
    item_types: Some(vec![ItemType::Video]),
    limit: 20,
    mode: SearchMode::Hybrid,  // FTS5 + embedding + graph
}).await?;

// 横断検索
let cross = pool.cross_search("RAG").await?;
```

### 6-2. CLI（akari-pool）

```bash
# Library
akari-pool library list
akari-pool library create wedding-2026
akari-pool library use wedding-2026   # 現在の library を切り替え
akari-pool library info

# Item CRUD
akari-pool add ./ceremony.mp4
akari-pool ls
akari-pool ls --type video
akari-pool cat {item_id}                 # context_json + 関連を返す
akari-pool rm {item_id}

# 検索
akari-pool search "結婚式"
akari-pool search "結婚式" --cross
akari-pool grep "BGM" --type audio

# 関係
akari-pool link {a} {b} --type derived --strength 0.8
akari-pool relations {item_id}

# Wiki 操作
akari-pool compile-notes                 # 全アイテムを .md に書き出し
akari-pool compile-notes {item_id}       # 1つだけ

# Lint
akari-pool lint                          # 全 Linter を実行
akari-pool lint --type inconsistency     # 特定の Linter のみ
akari-pool lint --fix                    # 自動修正可能なものを修正

# Analyzer
akari-pool analyze {item_id}             # 再分析
akari-pool analyze --pending             # 未分析を全て分析

# MCP
akari-pool mcp-serve                     # stdio
akari-pool mcp-serve --http --port 7890
```

### 6-3. MCP Server

詳細は Phase 6 設計時に別ドキュメントに分離。

**Resources**:
- `pool://library/{name}` — Library メタ情報
- `pool://item/{id}` — アイテム本体（mime_type に応じて）
- `pool://item/{id}/thumbnail` — サムネイル（動画/画像）
- `m2c://item/{id}/context` — 構造化コンテキスト JSON
- `wiki://item/{id}` — Wiki .md 形式

**Tools**:
- `pool_ls`, `pool_cat`, `pool_grep`, `pool_search`
- `pool_add`, `pool_link`, `pool_relations`
- `pool_lint`, `pool_compile_notes`
- `pool_analyze`, `pool_library_*`

**Subscribe**:
- `notifications/resources/updated` — Pool 変更時に通知（Filed back の即座反映）
- `notifications/resources/list_changed` — 新規アイテム追加時

---

## 7. データフロー

### 7-1. 取り込みフロー

```
ユーザー or AI が pool add
       ↓
[1] ファイルを libraries/{lib}/files/{uuid}/ にコピー
       ↓
[2] pool_items に基本メタを INSERT（item_type は MIME から自動判定）
       ↓
[3] Analyzer 自動選択（item_type → AnalyzerRegistry）
       ↓
[4] 非同期で Analyzer.analyze() 実行
       ↓
[5] 結果を pool_items に UPDATE（context_json, ai_summary, embedding 等）
       ↓
[6] notes/{uuid}.md を Wiki Compile
       ↓
[7] auto_link_by_shared_tags + generate_semantic_links
       ↓
[8] notifications/resources/list_changed 発火（MCP subscribe 中の全エージェントに通知）
```

### 7-2. 検索フロー

```
クエリ受信（pool_search "明るい結婚式"）
       ↓
[1] FTS5 で name/summary/tags/context を全文検索 → 候補 A
       ↓
[2] 埋め込みでクエリベクトル化 → context_embedding と cos 類似 → 候補 B
       ↓
[3] pool_relations を walks → 強関連の隣接ノード → 候補 C
       ↓
[4] A∪B∪C をマージしてリランキング
       ↓
[5] tiny model（Phase 2）で結果セットを要約圧縮
       ↓
[6] エージェントに返却（item_id と引用付き）
```

### 7-3. Filed back フロー

```
パートナーAI とのチャットで「この動画とあの動画は雰囲気似てる」
       ↓
FiledBackHook.on_event(PoolEvent::Inference { ... })
       ↓
[1] LLM が判定: 信頼できる関係性か?
       ↓
[2] AddRelation アクション生成
       ↓
[3] pool_relations に INSERT (created_by='ai', reason=会話ログ抜粋)
       ↓
[4] filed_back_log に履歴記録
       ↓
[5] notifications/resources/updated 発火
       ↓
[6] 次回検索でこの関係性が活きる
```

---

## 8. セキュリティ

### 8-1. 境界モデル

| レイヤー | 境界 | 強制方法 |
|---|---|---|
| Library 間 | 物理的に DB ファイル別 | `Library::open` でパス検証 |
| File system 越境 | `~/.akari-pool/` 配下のみ書き込み | path canonicalize、startsWith 禁止 |
| MCP クライアント | Roots ベース、初期化時に library 固定 | rmcp の roots 機能 |
| LLM 経由のインジェクション | 書き込み API に sanitization gateway | Phase 5 |
| PII 漏洩 | 出力時に redaction | Phase 5 |

### 8-2. 既知の脆弱性パターンの対策
- **CVE-2025-53109/53110 (EscapeRoute)** — `path::canonicalize` 必須、`startsWith` だけでは不可
- **MINJA メモリ汚染攻撃** — 書き込みパスに信頼度スコア必須、Filed back は人間レビュー可能に
- **Prompt injection from analyzed content** — Analyzer 出力を Tool description に直接埋め込まない

### 8-3. 将来: macOS App Sandbox 対応
- Phase 7 で AKARI Video 統合時に App Sandbox 対応
- pool-mcp を XPC service 化して権限を隔離
- 書き込み権限と読み取り権限を別 capability に

---

## 9. 拡張性

### 9-1. 新しい Analyzer を追加する

```rust
// crates/analyzer-myformat/src/lib.rs
use akari_pool_core::{Analyzer, AnalyzeResult, PoolItem, AnalyzerContext};

pub struct MyFormatAnalyzer;

#[async_trait]
impl Analyzer for MyFormatAnalyzer {
    fn supported_types(&self) -> Vec<&'static str> {
        vec!["application/x-myformat", ".myf"]
    }
    fn name(&self) -> &'static str { "myformat" }
    fn version(&self) -> &'static str { "0.1.0" }

    async fn analyze(&self, item: &PoolItem, ctx: &AnalyzerContext)
        -> Result<AnalyzeResult, AnalyzeError>
    {
        // 解析ロジック
        todo!()
    }
}

// 登録
pool.register_analyzer(Box::new(MyFormatAnalyzer));
```

### 9-2. 新しい Linter を追加する

`Linter` trait を実装して `pool.register_linter()` で登録。

### 9-3. 新しい Output フォーマット

将来 `Compiler` trait を導入。markdown 以外（HTML, LaTeX, JSON-LD）にも書き出せるように。

---

## 10. 既存OSSとの位置付け

| プロジェクト | 比較 |
|---|---|
| **AgentFS (Turso)** | inode/dentry 風の汎用仮想 FS。AkariPool は **モダリティ別の構造化** に踏み込む |
| **Nia / Nozomio** | クラウド型コード/PDF特化。AkariPool は **ローカルファースト** + **メディア対応** |
| **OpenViking** | viking:// URI の階層モデル。AkariPool の Library モデルが類似 |
| **Obsidian + LLM Wiki** | テキスト個人 KB の参照実装。AkariPool は **マルチモダリティ版** + **共有可能** |
| **AWS S3 Files** | クラウドスケールの object/file ハイブリッド。AkariPool は **ローカル版** の同思想 |
| **Letta** | 永続プロセス + LLM が memory を編集。AkariPool は **記憶じゃなくて素材** に特化 |
| **mem0 / Zep** | 対話履歴のメモリ層。AkariPool は補完関係（こちらは外部知識） |

詳細は AKARI Video の `docs/research/hosted-filesystems-for-agents-survey-2026.md` を参照。

---

## 11. 開発フェーズ

**方針**: Phase は順番に着実に全部やり切る。並列化や飛ばしはしない。

| Phase | 対象 crate | 内容 | 状態 |
|---|---|---|:-:|
| **0** | 全体 | プロジェクト構造、設計書、Cargo workspace | ✅ 完了 |
| **1** | pool-core | SQLite スキーマ、CRUD、基本 CLI、PoolItem の add/list/get/rm | ✅ 完了 |
| **2** | pool-library | Library 隔離、横断検索、共有レイヤー | ✅ 完了 |
| **3** | analyzer crates | Analyzer trait、article/image/audio/video を順次実装 | ✅ 完了 |
| **3.5** | pool-core + analyzer | FTS5 trigram、HTML 対応、LLM リトライ | ✅ 完了 |
| **4** | pool-core + pool-lint | Wiki Compile、`pool_compile_notes`、Linter 2/4 種 | ✅ 完了 |
| **4.5** | pool-lint + CLI | Linter 4 種揃え + Relation CLI | ✅ 完了 |
| **5** | pool-core | LLM プリセット 6 種、3 層設定マージ、Obsidian sync | ✅ 完了 |
| **6** | pool-mcp | MCP サーバー実装（rmcp、34 ツール）+ 会話履歴永続化 + Pool Browser MVP | ✅ 完了 |
| **7** | (AkariShell 側) | AkariShell 統合、AKARI Video をアプリとして再利用 | 🟡 **次** |
| **8** | 別プロジェクト | AkariNotes / AkariCMS / AkariSearch 着手 | ⬜ |

> **注**: 本表は `akari-pool-impl/README.md` の Phase 表を SSOT とし、同期のうれしくない遅延を防ぐため impl 側 commit と合わせて本ファイルも更新する運用（運用ルール: `docs/README.md` 末尾を参照）。

### Phase 1 完了サマリ (2026-04-08)

- Pool::open + schema v1 自動適用
- add_item / list_items / get_item / delete_item / count_items
- MIME 自動判定 (8モダリティ)
- CLI: info / add / ls / cat / rm
- バイナリ `~/.cargo/bin/akari-pool` にインストール済み

詳細は `docs/handoff-2026-04-08.md` を参照。

---

## 12. 未解決の論点（v0.2 で検討）

- [ ] 横断検索のパフォーマンス（複数 SQLite を open するコスト）
- [ ] Cloud sync の opt-in 設計（AKARI Cloud との連携プロトコル）
- [ ] Library 間の関係グラフ表現（cross_library relation の DB スキーマ）
- [ ] 大規模ファイル（10GB 級動画）のストリーミング読み出し
- [ ] Analyzer プラグインの動的ロード（dylib? wasm?）
- [ ] バックアップ・移行・エクスポート形式
- [ ] マルチユーザー対応（家族で1台共有等）
- [ ] AMP との統合（4型記憶モデル: episodic/semantic/procedural/working）
- [ ] M2C v0.3（汎用化版）の策定
