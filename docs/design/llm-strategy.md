# LLM 戦略設計書

> **バージョン**: v0.1 Draft
> **最終更新**: 2026-04-08
> **対応 Crate**: `crates/pool-core`（`LlmClient` trait + 設定）

---

## 1. 目的

AkariPool のすべての AI 処理（要約・タグ・関係推論・Linting・Vision・コード解析）について、**ユーザーが好きなモデルを UI から選べる**ようにする。**主力は OpenRouter 経由の安価な中華モデル**（DeepSeek / Qwen / GLM）。ローカル LLM は将来の選択肢の1つとして残す。

### 設計原則

| # | 原則 | 説明 |
|---|---|---|
| 1 | **モデル中立** | AkariPool 本体は特定のモデルベンダーに依存しない |
| 2 | **OpenAI 互換 API** | OpenRouter / Ollama / vLLM / LM Studio すべて同じインターフェース |
| 3 | **クラウド安価モデル主力** | デフォルトは OpenRouter 経由の DeepSeek / Qwen / GLM。Sonnet / GPT-4o は使わない |
| 4 | **ローカルは選択肢の1つ** | Ollama 等を入れたユーザーが任意で切替可能。デフォルトでは要求しない |
| 5 | **モデルカスケード** | タスクの軽さに応じて軽量/中規模/大規模を使い分け |
| 6 | **ユーザー選択可能** | UI から1クリックでモデル切替 |
| 7 | **3層設定** | グローバル → library → セッション の優先順位 |
| 8 | **コスト透明性** | リアルタイムでトークン消費・推定料金を表示 |
| 9 | **プライバシーファースト** | センシティブな library は「外部 API 禁止」フラグで保護（このときローカル必須） |

---

## 2. 2026年4月のモデルマップ

### クラウド（OpenRouter 経由を推奨）

| モデル | 入力 ($/1M tok) | 出力 ($/1M tok) | 性能 | 用途 |
|---|---|---|---|---|
| **DeepSeek V3.2** | $0.14 | $0.28 | 標準 | 🌟 主力（汎用） |
| **Qwen3 235B (A22B)** | $0.20 | $0.60 | 推論強い | 関係性推論・複雑な分析 |
| **Qwen3-Coder 480B** | $0.20 | $0.80 | コード SOTA | CodeAnalyzer |
| **Qwen2.5-VL 72B** | $0.40 | $1.20 | Vision 強い | ImageAnalyzer / VideoAnalyzer フレーム解析 |
| **GLM-4.6** (Zhipu) | $0.30 | $1.20 | バランス型 | 中規模タスク |
| **Gemini 2.5 Flash** | $0.075 | $0.30 | 速い・安い | バッチ Vision |
| **Gemini 2.5 Pro** | $1.25 | $5.00 | 高品質 Vision | 重い Vision |
| Claude Sonnet 4.6 | $3 | $15 | (非推奨) | コスト高すぎ |
| GPT-4o | $2.50 | $10 | (非推奨) | コスト高すぎ |

**主力候補**: DeepSeek V3.2 + Qwen3 シリーズ + GLM-4.6 + Gemini Flash。

### ローカル（オプション、将来の選択肢の1つ）

ユーザーが Ollama / llama.cpp / LM Studio 等を自分でセットアップしている場合、AkariPool はそれを認識して使える。**デフォルトでは要求しない**。

| モデル | サイズ | 用途 | 推奨環境 |
|---|---|---|---|
| **Qwen3-4B-Instruct** | 4B | タグ抽出・分類 | M1 以上、4GB RAM |
| **Qwen3-7B-Instruct** | 7B | 標準（要約・要点抽出） | M1 Pro 以上、8GB RAM |
| **Qwen3-14B-Instruct** | 14B | 中規模推論 | M2 Pro 以上、16GB RAM |
| **GLM-4-9B-Chat** | 9B | 汎用 | M1 Pro 以上 |
| **Qwen2.5-VL-7B** | 7B | Vision | M1 Pro 以上 |
| **whisper.cpp large-v3** | — | 文字起こし | どの Mac でも |
| **fastembed (BGE-M3)** | — | 埋め込み 1024次元 | CPU で十分 |

ローカル LLM が活きるシーン:
- 🔒 **プライバシー必須の library**（センシティブ素材）
- ✈️ **オフライン環境**（出張先・機内）
- 💸 **ヘビー利用でコスト削減**したい場合
- 🧪 **特殊な fine-tune モデル**を使いたい場合

ローカル推論バックエンド（ユーザー任意）: Ollama / llama.cpp / LM Studio / MLX-LM。OpenAI 互換 API が出せれば何でも使える。

---

## 3. モデルカスケード戦略

タスクの軽さに応じて段階的にモデルを選ぶ。**主力は OpenRouter 経由の安価モデル**。ローカルは選択肢として残す。

### デフォルト戦略（クラウド主力）

```
軽 ──────────────────────────────────────► 重

タグ抽出           → DeepSeek V3.2 (OpenRouter)       $0.14/1M
要約               → DeepSeek V3.2 (OpenRouter)       $0.14/1M
分類               → DeepSeek V3.2 (OpenRouter)       $0.14/1M
表記揺れ判定        → DeepSeek V3.2 (OpenRouter)       $0.14/1M
セマンティック関係推論 → Qwen3 235B (OpenRouter)       $0.20/1M
コード解析          → Qwen3-Coder 480B (OpenRouter)   $0.20/1M
Vision (軽)        → Gemini 2.5 Flash (OpenRouter)    $0.075/1M
Vision (重)        → Qwen2.5-VL 72B (OpenRouter)      $0.40/1M
文字起こし          → Whisper API (OpenAI)             $0.006/分
埋め込み            → OpenAI text-embedding-3-small   $0.02/1M
```

→ **API キー1個（OpenRouter）と OpenAI キー1個**だけで全部回る。Ollama 等のセットアップ不要。

### 代替戦略 ① — ローカル併用（任意）

ユーザーが Ollama を入れて軽い処理だけローカルに回すことも可能:

```
軽量タスク → Qwen3-7B (ローカル / Ollama)              無料
重い推論   → DeepSeek V3.2 / Qwen3 235B (OpenRouter)   $0.14〜0.20/1M
Vision    → Qwen2.5-VL-7B (ローカル) または Gemini Flash (クラウド)
```

### 代替戦略 ② — フルローカル（オフライン専用）

外部 API 完全ブロックの library 用。ユーザーが Ollama + Qwen3 + Qwen2.5-VL + whisper.cpp + fastembed を全部入れた場合のみ動く。

```
全タスク → ローカル LLM のみ                          無料（電気代と GPU 負荷のみ）
```

### カスケードの判定ロジック

1. ユーザーが選択したプリセット/設定に従う
2. 該当モデルが使えない場合（API キーなし、レート制限、ローカルモデル未 DL）はフォールバック先に切替
3. ユーザーが手動で「このタスクだけ上位モデル」と指定可能

---

## 4. LlmClient trait 設計

```rust
// crates/pool-core/src/llm.rs

#[async_trait]
pub trait LlmClient: Send + Sync {
    /// チャット補完
    async fn complete(&self, req: ChatRequest) -> Result<ChatResponse, LlmError>;

    /// 埋め込み生成
    async fn embed(&self, req: EmbedRequest) -> Result<EmbedResponse, LlmError>;

    /// このクライアントが使うモデル名
    fn model_id(&self) -> &str;

    /// この呼び出しのおおよそのコスト見積もり (USD)
    fn estimate_cost(&self, req: &ChatRequest) -> f64;
}

pub struct ChatRequest {
    pub messages: Vec<Message>,
    pub temperature: Option<f32>,
    pub max_tokens: Option<u32>,
    pub response_format: Option<ResponseFormat>,
}

pub struct ChatResponse {
    pub content: String,
    pub usage: Usage,
    pub model: String,
}

pub struct Usage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub estimated_cost_usd: f64,
}

pub enum LlmError {
    /// ネットワークエラー
    NetworkError(String),
    /// レート制限
    RateLimited { retry_after_sec: u64 },
    /// モデル読み込み失敗（ローカル）
    ModelLoadFailed(String),
    /// JSON validation 失敗
    InvalidResponse(String),
    /// その他
    Other(String),
}
```

### 実装は1つだけで全部対応

```rust
// OpenAI 互換 API（OpenRouter / Ollama / llama.cpp / vLLM / LM Studio すべてこれ）
pub struct OpenAiCompatibleClient {
    base_url: String,
    api_key: Option<String>,
    model: String,
    pricing: ModelPricing,
}
```

→ **endpoint URL とモデル名を変えるだけで、ローカル/クラウドが切り替わる**。

### モデルレジストリ

```rust
pub struct ModelRegistry {
    models: HashMap<String, ModelDescriptor>,
}

pub struct ModelDescriptor {
    pub id: String,                    // "qwen3-7b-local"
    pub display_name: String,          // "Qwen3 7B (ローカル)"
    pub provider: ModelProvider,       // Local / OpenRouter / Custom
    pub endpoint: String,              // "http://localhost:11434/v1"
    pub model_name: String,            // "qwen3:7b-instruct"
    pub api_key_env: Option<String>,   // "OPENROUTER_API_KEY"
    pub context_window: u32,
    pub capabilities: Capabilities,    // text / vision / function_calling
    pub pricing: ModelPricing,         // input/output ($/1M tok), 0 if local
    pub priority: u32,                 // カスケード順
}

pub struct Capabilities {
    pub text: bool,
    pub vision: bool,
    pub function_calling: bool,
    pub long_context: bool,
}

pub struct ModelPricing {
    pub input_per_1m: f64,
    pub output_per_1m: f64,
    pub currency: &'static str,
}
```

---

## 5. 設定の3層階層

優先順位は **セッション > library > グローバル**。

### Layer 1: グローバル設定

`~/.akari-pool/config.toml`:

```toml
[llm]
# デフォルト: OpenRouter 経由のクラウド主力
default_endpoint = "https://openrouter.ai/api/v1"
default_model = "deepseek/deepseek-v3.2"

[llm.cascade]
light = "deepseek/deepseek-v3.2"          # タグ・分類
medium = "deepseek/deepseek-v3.2"         # 標準
heavy = "qwen/qwen3-235b-a22b"            # 重い推論
code = "qwen/qwen3-coder-480b"            # コード
vision = "google/gemini-2.5-flash"        # Vision
embed = "openai/text-embedding-3-small"   # 埋め込み

[llm.providers.openrouter]
api_key_env = "OPENROUTER_API_KEY"
base_url = "https://openrouter.ai/api/v1"

[llm.providers.openai]
api_key_env = "OPENAI_API_KEY"
base_url = "https://api.openai.com/v1"

# ローカルは任意。ユーザーが入れていれば認識される
[llm.providers.ollama]
base_url = "http://localhost:11434/v1"
enabled = false  # デフォルト無効、ユーザーが有効化

[llm.budget]
# 月額予算（USD）。超えそうになったら警告
monthly_limit_usd = 10.0
warn_at_percent = 80
```

### Layer 2: Library 設定

`~/.akari-pool/libraries/wedding-2026/library.toml`:

```toml
[library.llm]
# この library 専用のオーバーライド
default_model = "qwen3:14b"  # 結婚式は質を優先

[library.llm.privacy]
# プライバシー: 外部 API 禁止
external_api_allowed = false   # ← これで OpenRouter 系全部ブロック
local_only = true

[library.llm.cascade]
heavy = "qwen3:32b"  # ヘビータスクもローカルで
```

### Layer 3: セッション/コマンド

```bash
# 1回だけ別モデルを使う
akari-pool analyze {item_id} --model qwen3:32b

# CLI 全体に環境変数で
AKARI_POOL_LLM_MODEL=deepseek/deepseek-v3.2 akari-pool ls
```

---

## 6. UI 設計（モデル選択を簡単にする）

ユーザーが「面倒な YAML を触らずに」モデルを切り替えられるよう、3つの UI を提供する。

### UI ① — Tauri GUI（AKARI Video / 将来の AkariCMS 等）

設定画面に「LLM」セクションを追加:

```
┌─────────────────────────────────────────────────┐
│  🤖 LLM 設定                                      │
├─────────────────────────────────────────────────┤
│                                                  │
│  ◉ ローカル優先 (推奨)                            │
│  ○ ローカル + クラウドフォールバック              │
│  ○ クラウドのみ                                  │
│  ○ オフライン (ローカルのみ)                      │
│                                                  │
│  ─── タスク別モデル ──────────────────           │
│                                                  │
│  軽量タスク (タグ・分類)                          │
│   [Qwen3 4B (ローカル, 無料)        ▼]           │
│                                                  │
│  標準タスク (要約)                                │
│   [Qwen3 7B (ローカル, 無料)        ▼]           │
│                                                  │
│  重いタスク (推論)                                │
│   [DeepSeek V3.2 (OpenRouter, $0.14/1M) ▼]      │
│                                                  │
│  Vision (画像・動画フレーム)                       │
│   [Qwen2.5-VL 7B (ローカル, 無料)   ▼]           │
│                                                  │
│  ─── プロバイダー ────────────────────           │
│                                                  │
│  OpenRouter API キー: [●●●●●●●●●●●● 設定済み]    │
│  Ollama: ✅ 接続済み (localhost:11434)            │
│                                                  │
│  ─── 予算管理 ──────────────────────             │
│                                                  │
│  月額予算: [$10.00] (現在 $1.23 / 12.3%)         │
│  ⚠️ 80% 超えたら警告                              │
│                                                  │
│  ─── プリセット ────────────────────             │
│                                                  │
│  [💰 節約モード]  [⚖️ バランス]  [⚡ パフォーマンス] │
│                                                  │
└─────────────────────────────────────────────────┘
```

**プリセット機能**: ワンクリックで設定セット切替

| プリセット | 内容 | 月額目安 (プロ利用) |
|---|---|---|
| 💰 **節約** | 全タスク DeepSeek V3.2 (OpenRouter) | $1〜3 |
| ⚖️ **バランス** (デフォルト) | DeepSeek + Qwen3 235B + Gemini Flash の組み合わせ | $3〜8 |
| ⚡ **パフォーマンス** | Qwen3 235B + Qwen2.5-VL 72B + Gemini Pro | $10〜25 |
| 🔒 **オフライン** (任意) | ローカル LLM のみ。要 Ollama セットアップ | $0 |
| 🧪 **ハイブリッド** (任意) | 軽量タスクはローカル、重いのはクラウド | $0.5〜2 |
| 🎓 **高品質** | Qwen3-235B / Gemini Pro / Qwen2.5-VL 72B | $15〜40 |

### UI ② — CLI

```bash
# 現在の設定確認
akari-pool config llm show

# モデル切替
akari-pool config llm set --task heavy --model deepseek/deepseek-v3.2

# プリセット適用
akari-pool config llm preset save
akari-pool config llm preset balanced
akari-pool config llm preset offline

# 利用可能モデル一覧
akari-pool config llm list-available

# 接続テスト
akari-pool config llm test --model qwen3:7b
```

### UI ③ — Obsidian プラグイン

サイドペインに小さな LLM セレクタ:

```
┌──────────────┐
│ 🤖 Active LLM │
│              │
│ [Qwen3 7B  ▼]│
│ ローカル     │
│              │
│ 今月: $1.23  │
└──────────────┘
```

ワンクリックで切替可能。

---

## 7. コスト試算（モデル選定別）

シナリオ別 × 3つのモデル戦略で再計算。

### シナリオA: 個人ユーザー（月10動画）

| 戦略 | LLM 構成 | 月額 |
|---|---|---|
| 💰 節約 | 全部ローカル (Qwen3-7B + whisper.cpp + fastembed) | **$0** |
| ⚖️ バランス | ローカル + DeepSeek フォールバック | **$0.05〜0.20** |
| ⚡ パフォーマンス | DeepSeek + Qwen3-235B 中心 | **$0.50〜1** |

### シナリオB: プロクリエイター（月100動画 + 50PDF + 20音源）

| 戦略 | 月額 |
|---|---|
| 💰 節約 | **$0.10〜0.50** |
| ⚖️ バランス | **$1〜3** |
| ⚡ パフォーマンス | **$5〜10** |

### シナリオC: ヘビーユーザー（複数 library）

| 戦略 | 月額 |
|---|---|
| 💰 節約 | **$0.50〜2** |
| ⚖️ バランス | **$5〜15** |
| ⚡ パフォーマンス | **$20〜40** |

→ **デフォルトのバランス設定で、業務ヘビー利用でも月 $15 以下**。Sonnet/GPT-4o 中心だと月 $80〜120 だったので **1/10 の経済性**。

---

## 8. プライバシー考慮

### Library 単位でのプライバシーレベル

```toml
[library.llm.privacy]
external_api_allowed = false  # 外部 API 禁止
local_only = true             # ローカルのみ
allow_models = ["qwen3:7b", "qwen3:14b"]  # 許可リスト
deny_models = []              # 拒否リスト
log_level = "minimal"         # 何を記録するか
pii_redaction = true          # PII を自動マスク
```

### 「local_only」モードの保証

- すべての LLM 呼び出しを `localhost` に強制
- OpenRouter / Anthropic / OpenAI の API への通信を **ネットワーク層でブロック**
- 違反検出時はエラー + ログ
- センシティブな library（医療・法務・個人情報）に必須

### PII 自動マスキング

- メール / 電話 / 住所 / クレカ番号 / マイナンバー を正規表現で検出
- LLM に渡す前にプレースホルダ化（`[EMAIL_1]`, `[PHONE_1]`）
- 応答後に復元

---

## 9. 推奨セットアップ手順（最短ルート）

### 標準セットアップ（クラウドのみ、3分で動く）

```bash
# 1. OpenRouter API キーを取得（https://openrouter.ai/）
export OPENROUTER_API_KEY="sk-or-..."

# 2. (Whisper 用) OpenAI API キー
export OPENAI_API_KEY="sk-..."

# 3. AkariPool 初期化
akari-pool config llm preset balanced
akari-pool config llm test
# ✅ deepseek/deepseek-v3.2: ok (1200ms)
# ✅ qwen/qwen3-235b-a22b: ok (1500ms)
# ✅ google/gemini-2.5-flash: ok (800ms)
# ✅ openai/text-embedding-3-small: ok (200ms)

# 4. プール作成して動かす
akari-pool library create my-first-pool
akari-pool add ./video.mp4 --library my-first-pool
```

→ **API キー2つだけで動く**。インストール不要。

### オプション: ローカル併用（任意、節約・プライバシー目的）

ヘビー利用や機密データを扱うユーザーは追加で:

```bash
# Ollama を入れる（任意）
brew install ollama
ollama pull qwen3:7b-instruct
ollama pull qwen2.5-vl:7b

# AkariPool に認識させる
akari-pool config llm provider enable ollama
akari-pool config llm preset hybrid  # ローカル + クラウド併用
```

→ **やりたい人だけやればいい**。デフォルトでは要求しない。

---

## 10. 将来の拡張

### Phase 5+ で検討

- **モデル評価ハーネス**: ユーザーのプール内データで複数モデルの精度を比較
- **自動カスケード調整**: 「このタスクはローカルで失敗率高い → 自動で上位にエスカレート」
- **ファインチューニング**: ユーザーのプールデータで Qwen3-7B を LoRA 学習
- **エージェント記憶からのモデル選定**: AMP 統合後、過去の成功/失敗ログから「このタイプのタスクには Qwen3-32B が向く」を学習
- **モデルマーケットプレイス**: ユーザー間で「俺の library 設定」を共有

---

## 11. 関連ドキュメント

- [`architecture.md`](./architecture.md) — 全体設計
- [`analyzer-plugin.md`](./analyzer-plugin.md) — Analyzer がどう LlmClient を使うか
- [`library-layer.md`](./library-layer.md) — Library の privacy 設定
- [`../integration/obsidian-plugin.md`](../integration/obsidian-plugin.md) — Obsidian でのモデル切替

---

## 12. 未解決の論点

- [ ] **ローカルモデル自動 DL**: AkariPool が初回起動時に Ollama 経由で必要モデルを引っ張るか?
- [ ] **MLX バックエンド統合**: M系 Mac で Ollama より高速な MLX-LM をサポートするか?
- [ ] **モデルバージョン固定**: 「このモデルが GA になったから安定版を使う」みたいな pin 機構
- [ ] **マルチクライアント協調**: 複数アプリが同時に同じ Ollama を叩くときのキュー
- [ ] **AKARI Cloud 統合**: クラウド経由のモデル選定もユーザーに透明にするか
- [ ] **Cost guardrail**: 予算超過時の自動切り替え（ローカルにフォールバック）
- [ ] **モデル評価**: どのモデルが実際にユーザーの手元で良いかを測る仕組み
