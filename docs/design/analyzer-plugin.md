# Analyzer Plugin 設計書

> **バージョン**: v0.1 Draft
> **最終更新**: 2026-04-08
> **対応 Crate**: `crates/pool-core` (trait) + `analyzers/*` (実装)

---

## 1. 目的とスコープ

AkariPool がどんなモダリティのデータでも扱えるように、**モダリティ別の分析器を Plugin として差し替え可能**にする。AkariPool コア本体は具体的なモダリティを知らない。

### なぜ Plugin か
1. **モダリティ追加が trait 実装だけで完結** — コア本体への変更不要
2. **依存関係を分離** — FFmpeg, LlamaParse 等の重い依存をオプション化できる
3. **テストしやすい** — モック Analyzer で本体ロジックをテスト可能
4. **将来の動的ロード** — wasm/dylib で配布可能になる余地を残す
5. **コミュニティ拡張** — 第三者が独自モダリティを追加できる

### スコープ内
- Analyzer trait の定義
- 標準アナライザ一覧と各仕様
- 登録メカニズム
- ライフサイクル管理
- 再分析・キャッシュ・エラー処理

### スコープ外
- 各アナライザ内部の LLM 呼び出し詳細（OpenRouter 連携は呼び出し元の責務）
- 動的ロード機構の実装（v0.2 で検討）

---

## 2. Trait 定義

```rust
use async_trait::async_trait;

/// 1モダリティを処理する分析器
#[async_trait]
pub trait Analyzer: Send + Sync {
    /// アナライザ識別名（ログ・DB 記録用）
    /// 例: "video-m2c", "pdf-llamaparse", "code-treesitter"
    fn name(&self) -> &'static str;

    /// バージョン（再分析判定に使う）
    /// 上がった場合、既存アイテムは再分析候補になる
    fn version(&self) -> &'static str;

    /// このアナライザが処理できる MIME type と拡張子
    /// 複数返してよい
    fn supported_types(&self) -> AnalyzerSupport;

    /// 1アイテムを分析
    async fn analyze(
        &self,
        item: &PoolItem,
        ctx: &AnalyzerContext,
    ) -> Result<AnalyzeResult, AnalyzeError>;

    /// このアナライザが必要とする外部依存（FFmpeg 等）が揃っているか
    /// 起動時の health check で使う
    async fn check_dependencies(&self) -> Result<(), DependencyError> {
        Ok(())
    }

    /// 部分的な再分析が可能か（例: タグだけ更新）
    fn supports_partial(&self) -> bool {
        false
    }
}

/// アナライザのサポート定義
pub struct AnalyzerSupport {
    pub mime_types: Vec<&'static str>,    // ["video/mp4", "video/quicktime"]
    pub extensions: Vec<&'static str>,    // [".mp4", ".mov"]
    pub priority: u32,                     // 同じファイルに複数該当する場合の優先度
}

/// 分析実行時のコンテキスト
pub struct AnalyzerContext {
    pub workspace: WorkspaceInfo,
    pub config: AnalyzerConfig,
    pub progress_tx: ProgressSender,        // 進捗通知用
    pub cancel_token: CancellationToken,    // 中断要求
    pub llm_client: Arc<dyn LlmClient>,     // OpenRouter 等の抽象
    pub embed_client: Arc<dyn EmbedClient>, // 埋め込みモデル
}

/// 分析結果
pub struct AnalyzeResult {
    /// 短い要約（人間可読、~200文字）
    pub summary: String,

    /// タグのリスト
    pub tags: Vec<String>,

    /// 構造化コンテキスト（M2C 準拠）
    /// モダリティごとに異なる schema、JSON Schema は別途定義
    pub context_json: serde_json::Value,

    /// 圧縮版（Phase 4 で実装、token 節約用）
    pub context_compressed: Option<String>,

    /// 埋め込みベクトル（Phase 2 で実装）
    pub embedding: Option<EmbeddingResult>,

    /// アナライザ固有の生データ（ffprobe 出力等）
    /// 再分析時の参考にする
    pub raw: Option<serde_json::Value>,

    /// このアイテムから派生する子アイテム（例: 動画→キーフレーム画像）
    /// 子アイテムは別 PoolItem として登録される
    pub derived_items: Vec<DerivedItem>,
}

pub struct EmbeddingResult {
    pub model: String,           // "fastembed/bge-small-ja"
    pub dimension: u32,
    pub vector: Vec<f32>,
}

pub struct DerivedItem {
    pub item_type: ItemType,
    pub file_path: PathBuf,
    pub relation_type: RelationType,
    pub strength: f32,
}

/// 分析エラー
#[derive(thiserror::Error, Debug)]
pub enum AnalyzeError {
    #[error("依存ツールが見つからない: {0}")]
    DependencyMissing(String),

    #[error("ファイルが読めない: {0}")]
    FileError(#[from] std::io::Error),

    #[error("LLM 呼び出し失敗: {0}")]
    LlmError(String),

    #[error("分析中断")]
    Cancelled,

    #[error("サポート外のフォーマット: {0}")]
    UnsupportedFormat(String),

    #[error("分析タイムアウト ({0}秒)")]
    Timeout(u64),

    #[error("その他: {0}")]
    Other(#[from] anyhow::Error),
}
```

---

## 3. 標準アナライザ一覧

最初に揃える9つの標準アナライザ。各 crate は独立しており、必要なものだけ feature flag で有効化する。

### 3-1. VideoAnalyzer 🎬

**Crate**: `analyzers/video`
**バージョン**: `v0.2.0`
**対応**: `video/mp4`, `video/quicktime`, `video/webm`, `.mp4`, `.mov`, `.webm`, `.mkv`, `.avi`

#### 依存
- `ffmpeg` (CLI)
- Vision API (OpenRouter経由 — Gemini Vision / GPT-4o / Claude Sonnet)
- Whisper (音声トランスクリプト用、ローカル or OpenRouter)

#### 出力 (M2C video v0.2)
```json
{
  "modality": "video",
  "duration_sec": 432.5,
  "resolution": [1920, 1080],
  "fps": 30,
  "L0_tags": ["結婚式", "屋外", "明るい"],
  "L0_moods": ["祝祭", "感動"],
  "L1_summary": "屋外での結婚式セレモニー、晴天、参列者多数",
  "L1_color_palette": ["#f5e8d4", "#e8f5d4", ...],
  "L1_speakers": [{"id": "speaker_0", "duration_sec": 120}],
  "L2_scenes": [
    {
      "start": 0.0, "end": 45.2,
      "summary": "入場シーン、明るい屋外",
      "keyframe_uri": "thumbs/scene_0.jpg",
      "objects": ["人物", "花", "椅子"],
      "transcript": "..."
    }
  ],
  "L3_transcription": [...],
  "L3_acoustic": {"bgm": [...], "silence": [...]},
  "L3_visual": {"faces": [...], "colors": [...]}
}
```

#### 派生アイテム
- 各シーンのキーフレーム画像 → ImageAnalyzer に渡す
- 音声トラック → AudioAnalyzer に渡す（オプション）

### 3-2. AudioAnalyzer 🎵

**Crate**: `analyzers/audio`
**バージョン**: `v0.1.0`
**対応**: `audio/wav`, `audio/mp3`, `audio/flac`, `.wav`, `.mp3`, `.flac`, `.ogg`, `.m4a`

#### 依存
- `ffmpeg` (波形・スペクトログラム生成)
- Whisper (トランスクリプト)
- LLM (BGM/効果音/会話の分類)

#### 出力 (M2C audio v0.1)
```json
{
  "modality": "audio",
  "duration_sec": 180.3,
  "sample_rate": 48000,
  "channels": 2,
  "L0_tags": ["BGM", "ピアノ", "穏やか"],
  "L1_summary": "ピアノのソロ曲、穏やかなテンポ",
  "L1_bpm": 72,
  "L1_key": "C major",
  "L2_segments": [
    {"start": 0.0, "end": 30.0, "type": "intro", "energy": 0.3},
    {"start": 30.0, "end": 90.0, "type": "verse", "energy": 0.6}
  ],
  "L3_transcription": null,
  "L3_acoustic_features": {...}
}
```

### 3-3. ImageAnalyzer 🖼️

**Crate**: `analyzers/image`
**バージョン**: `v0.1.0`
**対応**: `image/jpeg`, `image/png`, `image/webp`, `image/heic`, `.jpg`, `.png`, `.webp`, `.heic`

#### 依存
- Vision API (OpenRouter)
- `image` crate (Rust、メタデータ抽出)
- EXIF パーサ

#### 出力
```json
{
  "modality": "image",
  "dimensions": [3840, 2160],
  "L0_tags": ["人物", "屋外", "ポートレート"],
  "L0_moods": ["明るい", "穏やか"],
  "L1_summary": "屋外でのポートレート写真、自然光、背景は緑",
  "L1_color_palette": ["#7a9e5c", "#f5d4a8", ...],
  "L1_objects": ["人物", "木", "花"],
  "L1_faces": [{"bbox": [...], "age_range": "20-30"}],
  "L2_composition": {"rule_of_thirds": true, "depth_of_field": "shallow"},
  "L3_exif": {...}
}
```

### 3-4. PDFAnalyzer 📄

**Crate**: `analyzers/pdf`
**バージョン**: `v0.1.0`
**対応**: `application/pdf`, `.pdf`

#### 依存
- LlamaParse (推奨) または PyMuPDF / pdf-extract
- LLM (要約 + 構造化)

#### 出力 (Nia の tree reasoning パターン採用)
```json
{
  "modality": "pdf",
  "page_count": 24,
  "L0_tags": ["論文", "RAG", "ベクトル検索"],
  "L1_summary": "RAG システムにおけるチャンク戦略の比較研究",
  "L1_authors": ["Smith, J.", "Tanaka, H."],
  "L1_year": 2026,
  "L2_tree": {
    "type": "section",
    "title": "Abstract",
    "page": 1,
    "node_id": "n_001",
    "children": [
      {"type": "section", "title": "Introduction", "page": 1, "node_id": "n_002", "children": [...]}
    ]
  },
  "L3_full_text": "...",
  "L3_figures": [{"page": 5, "caption": "...", "bbox": [...]}]
}
```

#### 派生アイテム
- 図表をそれぞれ ImageAnalyzer に渡す（オプション）

### 3-5. ArticleAnalyzer 📝

**Crate**: `analyzers/article`
**バージョン**: `v0.1.0`
**対応**: `text/html`, `text/markdown`, `.html`, `.md`, `.rst`

#### 依存
- `pulldown-cmark` (Markdown)
- `scraper` (HTML)
- LLM (要約 + 概念抽出)

#### 出力
```json
{
  "modality": "article",
  "word_count": 3200,
  "language": "ja",
  "L0_tags": ["技術解説", "RAG", "AI"],
  "L1_summary": "次世代RAGアーキテクチャの解説記事",
  "L1_author": "...",
  "L1_published_at": "2026-03-15",
  "L1_source_url": "...",
  "L2_outline": [
    {"level": 1, "text": "はじめに", "anchor": "intro"},
    {"level": 1, "text": "従来のRAG", "anchor": "old-rag"},
    {"level": 2, "text": "問題点", "anchor": "issues"}
  ],
  "L3_full_text": "...",
  "L3_links": [{"text": "...", "href": "..."}],
  "L3_code_blocks": [...]
}
```

### 3-6. CodeAnalyzer 💻

**Crate**: `analyzers/code`
**バージョン**: `v0.1.0`
**対応**: `text/x-rust`, `text/x-python`, `text/x-typescript`, `.rs`, `.py`, `.ts`, `.js`, `.go`, `.java`, ...

#### 依存
- `tree-sitter` + 言語別 grammar
- LLM (関数説明・docstring 補完)

#### 出力 (Greptile/Nia パターン)
```json
{
  "modality": "code",
  "language": "rust",
  "loc": 245,
  "L0_tags": ["pool", "sqlite", "crud"],
  "L1_summary": "SQLite をバックエンドとする Pool の CRUD 実装",
  "L2_symbols": [
    {
      "kind": "function",
      "name": "Pool::open",
      "signature": "pub fn open(path: impl AsRef<Path>) -> Result<Self>",
      "docstring": "プールを指定パスから開く...",
      "line_range": [42, 78],
      "node_id": "f_001"
    }
  ],
  "L2_imports": ["rusqlite", "serde", "..."],
  "L2_dependencies": [{"name": "rusqlite", "version": "0.32"}],
  "L3_full_text": "..."
}
```

### 3-7. DatasetAnalyzer 📊

**Crate**: `analyzers/dataset`
**バージョン**: `v0.1.0`
**対応**: `text/csv`, `application/json`, `application/x-parquet`, `.csv`, `.json`, `.jsonl`, `.parquet`

#### 依存
- `polars` (Rust)
- LLM (スキーマ説明)

#### 出力
```json
{
  "modality": "dataset",
  "format": "csv",
  "row_count": 12500,
  "column_count": 8,
  "size_bytes": 4520000,
  "L0_tags": ["売上", "時系列", "店舗別"],
  "L1_summary": "店舗別月次売上データ、2020-2025、8カラム",
  "L1_columns": [
    {"name": "date", "type": "date", "nullable": false},
    {"name": "store_id", "type": "int", "cardinality": 42},
    {"name": "revenue", "type": "float", "min": 0.0, "max": 9876543.0}
  ],
  "L2_sample_rows": [...],
  "L2_statistics": {...}
}
```

### 3-8. NoteAnalyzer 🧠

**Crate**: `analyzers/note`
**バージョン**: `v0.1.0`
**対応**: `.md` (frontmatter 付き、Obsidian 形式)

#### 依存
- `pulldown-cmark`
- `serde_yaml` (frontmatter)

#### 特徴
- Obsidian の `[[backlink]]` 記法を解析して `pool_relations` に直接変換
- frontmatter のタグ・カテゴリを `ai_tags` にマージ
- Article より「個人ノート」前提の処理

#### 出力
```json
{
  "modality": "note",
  "L0_tags": ["メモ", "アイデア"],
  "L1_summary": "...",
  "L1_frontmatter": {...},
  "L2_outline": [...],
  "L2_backlinks": [
    {"target_pool_id": "...", "alias": "..."}
  ],
  "L3_full_text": "..."
}
```

### 3-9. URLAnalyzer 🔗

**Crate**: `analyzers/url`
**バージョン**: `v0.1.0`
**対応**: 仮想モダリティ（実体は HTTP fetch + 動的ディスパッチ）

#### 動作
1. URL を fetch
2. Content-Type を見て適切な Analyzer に委譲
3. 例: `https://example.com/article` → ArticleAnalyzer
4. 例: `https://example.com/paper.pdf` → PDFAnalyzer
5. 例: `https://youtube.com/watch?v=...` → YouTube downloader 経由で VideoAnalyzer

#### 特殊扱い
- ダウンロードした実体は raw 層に保存
- メタとして `source_url` を保持

---

## 4. 登録メカニズム

### 4-1. AnalyzerRegistry

```rust
pub struct AnalyzerRegistry {
    analyzers: Vec<Box<dyn Analyzer>>,
}

impl AnalyzerRegistry {
    pub fn new() -> Self { Self { analyzers: vec![] } }

    pub fn register(&mut self, analyzer: Box<dyn Analyzer>) {
        self.analyzers.push(analyzer);
    }

    /// MIME type / 拡張子から最適な Analyzer を選ぶ
    /// 複数該当する場合は priority が高いものを返す
    pub fn find_for(&self, item: &PoolItem) -> Option<&dyn Analyzer> {
        self.analyzers
            .iter()
            .filter(|a| a.supported_types().matches(item))
            .max_by_key(|a| a.supported_types().priority)
            .map(|b| b.as_ref())
    }
}
```

### 4-2. 起動時登録

```rust
// crates/pool-cli/src/main.rs
fn build_registry() -> AnalyzerRegistry {
    let mut reg = AnalyzerRegistry::new();

    #[cfg(feature = "analyzer-video")]
    reg.register(Box::new(analyzers_video::VideoAnalyzer::new()));

    #[cfg(feature = "analyzer-audio")]
    reg.register(Box::new(analyzers_audio::AudioAnalyzer::new()));

    #[cfg(feature = "analyzer-image")]
    reg.register(Box::new(analyzers_image::ImageAnalyzer::new()));

    #[cfg(feature = "analyzer-pdf")]
    reg.register(Box::new(analyzers_pdf::PDFAnalyzer::new()));

    // ... 他

    reg
}
```

各アナライザは feature flag で切り替え可能。FFmpeg を入れたくない環境では `analyzer-video` を無効化できる。

### 4-3. 動的ロード（v0.2 検討）

将来は wasm/dylib を `~/.akari-pool/shared/analyzers/` に置けば自動ロードする仕組みを検討。

---

## 5. ライフサイクル

### 5-1. 取り込み → 分析 → 保存

```
[1] pool add ./video.mp4
       ↓
[2] PoolItem 作成（item_type は MIME 判定）
       ↓
[3] AnalyzerRegistry.find_for(item) → VideoAnalyzer 選択
       ↓
[4] dependencies check (ffmpeg 存在確認)
       ↓
[5] 分析キューに投入
       ↓
[6] (非同期) Analyzer.analyze(item, ctx).await
       ↓        ↓
       │        └→ progress_tx で進捗通知
       ↓
[7] AnalyzeResult を受け取る
       ↓
[8] pool_items を UPDATE
       ↓
[9] derived_items があれば子アイテムを作成（再帰的に分析）
       ↓
[10] notes/{uuid}.md を Wiki Compile
       ↓
[11] notifications/resources/list_changed 発火
```

### 5-2. 進捗通知

```rust
pub struct AnalyzerProgress {
    pub item_id: ItemId,
    pub stage: ProgressStage,
    pub percent: f32,
    pub message: String,
}

pub enum ProgressStage {
    Queued,
    Initializing,
    Extracting,    // メタデータ抽出
    Analyzing,     // LLM 呼び出し
    Embedding,     // 埋め込み生成
    Storing,
    Done,
    Failed(String),
}
```

CLI では progress bar、MCP では `notifications/progress` で通知。

### 5-3. キャンセル

長時間ジョブは `CancellationToken` で中断可能。Tauri GUI / CLI どちらも Ctrl+C や stop ボタンから停止できる。

---

## 6. 再分析判定

### 6-1. バージョン管理

`pool_items.analyzer_version` に分析時の Analyzer バージョンを記録。

```rust
fn needs_reanalysis(item: &PoolItem, current: &dyn Analyzer) -> bool {
    item.analyzer_version.as_deref() != Some(current.version())
}
```

### 6-2. マイグレーション

Analyzer のバージョンが上がった場合:
1. `akari-pool analyze --pending` で対象を一覧表示
2. ユーザー確認後、再分析実行
3. 既存の `context_json` をバックアップしてから上書き
4. `lint_findings` に「再分析した素材」を記録

### 6-3. 再分析トリガー

- ✅ Analyzer バージョン更新時
- ✅ ファイル本体が変更されたとき（ハッシュ比較）
- ✅ ユーザーが明示的に `pool analyze {id} --force`
- ❌ メタデータだけの変更では再分析しない（無駄なコスト）

---

## 7. エラー処理

### 7-1. リトライポリシー

```rust
pub struct RetryPolicy {
    pub max_attempts: u32,           // デフォルト 3
    pub initial_backoff_sec: u64,    // デフォルト 5
    pub backoff_multiplier: f32,     // デフォルト 2.0
    pub retry_on: Vec<RetryableErrorKind>,
}

pub enum RetryableErrorKind {
    LlmRateLimit,
    LlmTimeout,
    NetworkError,
}
```

`DependencyMissing` や `UnsupportedFormat` はリトライ対象外（即失敗）。

### 7-2. パーシャル成功

LLM 呼び出しが失敗しても部分的な結果を保存:
- ファイルメタデータ（`ffprobe` 結果等）は保存
- LLM が失敗した部分は `null` のまま
- `lint_findings` に「LLM 分析未完了」を記録 → 後で再試行可能

### 7-3. 失敗時の DB 状態

- `pool_items.analyzed_at` は `null` のまま
- `lint_findings` にエラー詳細を記録
- リトライキューに再投入

---

## 8. キャッシュ

### 8-1. ファイルハッシュベース

同じファイル（ハッシュ一致）の再分析は `pool_items.analyzer_raw` から復元するだけ。

### 8-2. LLM 結果キャッシュ

オプションで LLM 呼び出しを `~/.akari-pool/shared/llm-cache/` にキャッシュ。
- key: `hash(prompt + model)`
- value: 応答 JSON
- 同一プロンプトの再分析で API コスト削減

### 8-3. 埋め込みキャッシュ

同じテキストの埋め込みは再計算しない。`embedding_cache` テーブルに保存。

---

## 9. 新しい Analyzer を追加する手順

### 9-1. ステップバイステップ

1. **新しい crate を作る**
   ```bash
   cd ~/_project/PJ26c23_AkariPool
   cargo new --lib analyzers/my-format
   ```

2. **`Cargo.toml` を設定**
   ```toml
   [package]
   name = "akari-pool-analyzer-myformat"
   version = "0.1.0"
   edition = "2024"

   [dependencies]
   akari-pool-core = { path = "../../crates/pool-core" }
   async-trait = "0.1"
   serde_json = "1"
   thiserror = "2"
   ```

3. **`Analyzer` trait を実装**
   ```rust
   use akari_pool_core::{Analyzer, AnalyzerContext, AnalyzeResult, AnalyzeError, PoolItem, AnalyzerSupport};
   use async_trait::async_trait;

   pub struct MyFormatAnalyzer;

   impl MyFormatAnalyzer {
       pub fn new() -> Self { Self }
   }

   #[async_trait]
   impl Analyzer for MyFormatAnalyzer {
       fn name(&self) -> &'static str { "myformat" }
       fn version(&self) -> &'static str { "0.1.0" }

       fn supported_types(&self) -> AnalyzerSupport {
           AnalyzerSupport {
               mime_types: vec!["application/x-myformat"],
               extensions: vec![".myf"],
               priority: 100,
           }
       }

       async fn analyze(&self, item: &PoolItem, ctx: &AnalyzerContext)
           -> Result<AnalyzeResult, AnalyzeError>
       {
           // 1. ファイル読み込み
           // 2. 解析
           // 3. LLM で要約・タグ生成
           // 4. AnalyzeResult を返す
           todo!()
       }
   }
   ```

4. **テストを書く** (`tests/integration_test.rs`)

5. **CLI に登録** — `crates/pool-cli/Cargo.toml` に optional dependency 追加、`build_registry()` に登録

6. **Wiki Compile テンプレートを追加** — `pool-core/src/wiki/templates/myformat.md`

---

## 10. テスト戦略

### 10-1. 単体テスト
- 各 Analyzer crate 内で実ファイルを使ったテスト
- LLM クライアントは `MockLlmClient` で差し替え

### 10-2. 統合テスト
- `pool-cli` の crate 内で `tempfile` を使った E2E
- `pool add → ls → cat` のフロー

### 10-3. ゴールデンファイルテスト
- 既知の入力ファイルに対する `AnalyzeResult` を JSON snapshot として保存
- `insta` crate で diff 検出

### 10-4. パフォーマンステスト
- `criterion` で各 Analyzer のスループット計測
- 1動画あたりの分析時間目標: 動画長の 10% 以下（10分動画なら 60秒以内）

---

## 11. 既存設計との関係

### 11-1. M2C プロトコル
- AkariPool の Analyzer は **M2C v0.3 (汎用化版)** の reference implementation
- 各モダリティの `context_json` schema は M2C プロトコルに従う
- M2C v0.2 はメディア中心、v0.3 で全モダリティ対応に拡張する

### 11-2. AMP プロトコル
- 将来 Filed back ループの実装で AMP の4型記憶モデル（episodic/semantic/procedural/working）を採用
- Analyzer の出力を AMP に流し込む経路を v0.2 で設計

### 11-3. AKARI Video
- AKARI Video の既存 `src-tauri/src/pool/m2c/modules.rs` のディスパッチャを VideoAnalyzer / AudioAnalyzer / ImageAnalyzer に分離して移植
- AKARI Video は VideoAnalyzer の最初の重要ユーザー

---

## 12. 未解決の論点

- [ ] **マルチ Analyzer 同時実行**の上限制御（CPU/GPU/API quota）
- [ ] **Analyzer 間の依存**（PDFAnalyzer の中で ImageAnalyzer を呼ぶ等の連鎖）の表現
- [ ] **動的ロード**: wasm vs dylib vs サブプロセス
- [ ] **GPU 利用**: ローカル Whisper や ローカル Vision の GPU acceleration の抽象化
- [ ] **Analyzer 出力のバリデーション**: M2C schema で JSON Schema 検証
- [ ] **ストリーミング分析**: 大きい動画を全部読まずにシーン単位で逐次出力
- [ ] **増分分析**: 動画の一部だけ変更されたときに該当シーンだけ再分析
- [ ] **協調分析**: 複数 Analyzer の結果を統合する meta-analyzer
