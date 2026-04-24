# Workspace 層 設計書

> **[非推奨 / DEPRECATED]**
> 本ドキュメントは旧設計書です。HUB-017（2026-04-22）で Workspace 概念が Library と Session/Workspace に分離されました。
> **現行ドキュメントは [`../library-layer.md`](../library-layer.md) を参照してください。**
> 本ファイルは履歴保存のため archive に残しています。内容の更新は行いません。

---

> **バージョン**: v0.1 Draft
> **最終更新**: 2026-04-08
> **対応 Crate**: `crates/pool-workspace`

---

## 1. Workspace とは

**Workspace = 目的別の作業空間**。AkariPool 全体の1つ上の階層として、複数の「独立した本棚」を持てるようにする。

### 動機

ユーザーは現実に複数の文脈で AkariPool を使う:

- 🎬 結婚式動画プロジェクト（動画 + BGM + 脚本）
- 📚 バックエンド技術調査（PDF + 記事 + コード）
- 📱 SNS 発信ネタ集め（短尺動画 + 画像 + 投稿草稿）
- 🧠 個人の知識ベース（ノート + 日記 + ブックマーク）

これらを1つのプールに混ぜると:

- ❌ 動画編集中にバックエンド調査が混ざってノイズ
- ❌ 検索結果が常に汚れる
- ❌ 削除・バックアップが危険（混在してて切り分け不能）

逆に完全に別アプリにすると:

- ❌ 「結婚式プロジェクトのために調べた参考記事」を思い出せない
- ❌ ツール乱立で管理コスト爆発
- ❌ 横断検索ができない

### 解決策: Workspace 隔離 + 明示的横断

| 操作 | デフォルト | オプション |
|---|---|---|
| `pool ls` | 現在の workspace のみ | — |
| `pool search` | 現在の workspace のみ | `--cross` で全 workspace |
| `pool cat {id}` | 現在の workspace 内の id | — |
| `pool link a b` | 同 workspace 内 | `--target-workspace=...` で横断リンク |
| MCP セッション | 初期化時に workspace 固定 | 別セッションで別 workspace |

**「デフォルト隔離・明示的横断」が原則**。これで「動画編集中は動画だけ見える」「SNS ネタ探しは横断で見る」が両立する。

---

## 2. 物理構造

### 2-1. ディレクトリ

```
~/.akari-pool/
├── workspaces/
│   ├── wedding-2026/
│   │   ├── files/                    ← raw 層
│   │   │   ├── {uuid-1}/
│   │   │   │   └── ceremony.mp4
│   │   │   └── {uuid-2}/
│   │   │       └── bgm.wav
│   │   ├── notes/                    ← Wiki 層
│   │   │   ├── {uuid-1}.md
│   │   │   └── {uuid-2}.md
│   │   ├── pool.db                   ← SQLite 層（このworkspace専用）
│   │   └── workspace.toml            ← Workspace メタ情報
│   │
│   ├── backend-research/
│   │   ├── files/
│   │   ├── notes/
│   │   ├── pool.db
│   │   └── workspace.toml
│   │
│   └── ...
│
├── shared/
│   ├── tags.db                       ← 共通タグ辞書
│   ├── templates/                    ← 共有テンプレート
│   └── analyzers/                    ← 共有 Analyzer 設定
│
├── cross_relations.db                ← workspace 越え関係グラフ
└── config.toml                       ← グローバル設定
```

### 2-2. workspace.toml

各 workspace のメタ情報:

```toml
[workspace]
name = "wedding-2026"
display_name = "結婚式 2026年4月"
description = "結婚式動画プロジェクト用"
created_at = "2026-04-08T16:00:00+09:00"
icon = "🎬"
color = "#f5a3b6"

[workspace.purpose]
# 何のために作られたか（AI が文脈を理解する用）
intent = "video_project"
target_app = "akari-video"

[workspace.policy]
# プライバシーポリシー
allow_cross_search = true
allow_external_export = false
auto_lint = true
auto_filed_back = true

[workspace.analyzers]
# このworkspace で有効にする Analyzer
enabled = ["video", "audio", "image", "pdf", "note"]

[workspace.tags]
# このworkspace で頻出するタグ（hint として AI に渡す）
suggested = ["結婚式", "セレモニー", "披露宴", "BGM", "字幕"]
```

### 2-3. なぜ workspace ごとに pool.db を分けるか

| 利点 | 説明 |
|---|---|
| **隔離が物理的** | 誤って横断検索しない、コードバグでも越境しない |
| **バックアップが単純** | フォルダごとコピーすれば完全バックアップ |
| **削除が安全** | フォルダ削除＝完全削除、孤児レコードなし |
| **同時アクセス安全** | SQLite ファイルが別なので WAL 競合なし |
| **移行が単純** | 別マシンに移すときフォルダ転送だけ |
| **権限制御しやすい** | OS の filesystem permission で workspace ごとに保護可能 |

**欠点**:

- 横断検索時に複数の DB を open する必要 → Phase 2 で対処
- 共通タグの一貫性 → `shared/tags.db` で集約

---

## 3. API 設計

### 3-1. Rust crate API

```rust
use akari_pool::{Pool, Workspace, WorkspaceConfig};

let pool = Pool::open("~/.akari-pool")?;

// Workspace 一覧
let workspaces: Vec<WorkspaceInfo> = pool.list_workspaces()?;

// Workspace 作成
let ws = pool.create_workspace(WorkspaceConfig {
    name: "wedding-2026".into(),
    display_name: Some("結婚式 2026年4月".into()),
    intent: Some("video_project".into()),
    icon: Some("🎬".into()),
    ..Default::default()
})?;

// Workspace 取得
let ws = pool.workspace("wedding-2026")?;

// Workspace 内操作（このAPIは workspace に閉じている）
let items = ws.list_items()?;
let result = ws.search("結婚式 笑顔").await?;
let item_id = ws.add_item(req).await?;

// 横断検索
let cross_results = pool.cross_search(CrossSearchRequest {
    query: "RAG".into(),
    workspaces: None,  // None = 全 workspace
    item_types: None,
    limit: 20,
}).await?;

// Workspace 越えリンク
pool.add_cross_relation(CrossRelation {
    source_workspace: "wedding-2026".into(),
    source_item_id: item_a,
    target_workspace: "personal-kb".into(),
    target_item_id: item_b,
    relation_type: RelationType::Related,
    strength: 0.8,
    reason: "両方とも音響設計の参考".into(),
})?;

// Workspace 削除
pool.delete_workspace("wedding-2026", DeleteOptions {
    confirm: true,
    keep_files: false,
})?;
```

### 3-2. CLI

```bash
# Workspace 管理
akari-pool workspace list
akari-pool workspace create wedding-2026 --display "結婚式 2026" --icon 🎬
akari-pool workspace info wedding-2026
akari-pool workspace use wedding-2026          # current workspace を切替
akari-pool workspace delete wedding-2026 --confirm

# Workspace 切替（環境変数 or 設定ファイル）
export AKARI_POOL_WORKSPACE=wedding-2026

# あるいは毎回 --workspace で指定
akari-pool ls --workspace wedding-2026

# 横断検索
akari-pool search "BGM" --cross
akari-pool search "BGM" --workspaces wedding-2026,sns-content

# 横断リンク
akari-pool link {id-in-ws-a} {id-in-ws-b} \
  --target-workspace personal-kb \
  --type related --strength 0.8 \
  --reason "両方とも音響設計の参考"
```

### 3-3. MCP プロトコル

MCP セッション初期化時に workspace を固定する:

```json
{
  "method": "initialize",
  "params": {
    "capabilities": { "roots": { "listChanged": true } },
    "clientInfo": { "name": "claude-code", "version": "1.0" },
    "protocolVersion": "2025-11-25",
    "_meta": {
      "akari-pool/workspace": "wedding-2026"
    }
  }
}
```

これ以降のすべての tool/resource 呼び出しはこの workspace 内に制限される。

横断検索が必要な場合は **別セッション** を立てる、または特別なツール `pool_cross_search` を呼ぶ。

---

## 4. 共有レイヤー（shared/）

複数 workspace から共通で参照される情報。

### 4-1. shared/tags.db

```sql
CREATE TABLE shared_tags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    canonical_name TEXT NOT NULL UNIQUE,    -- 正規化された名前
    aliases TEXT,                            -- JSON 配列：別表記
    description TEXT,
    category TEXT,                           -- mood/object/style/role/...
    usage_count INTEGER DEFAULT 0,
    created_at TEXT NOT NULL
);
```

**役割**: タグ表記揺れの正規化辞書。

- ある workspace で "結婚式" タグが追加される
- 別 workspace で "ウェディング" タグが追加される
- Linter が「同じ意味の可能性」を検出して `shared_tags` に統合提案

### 4-2. shared/templates/

レシピテンプレート、Analyzer 設定、Wiki 出力フォーマット等の共有資源。

- AKARI Video の3モードエディタの「設計図」テンプレートはここに置く
- 全 workspace から参照可能

### 4-3. shared/analyzers/

Analyzer プラグインの共有設定:

- 使用する LLM モデル（OpenRouter 経由）
- API キー（暗号化保存）
- 分析頻度・優先度

---

## 5. 横断検索の実装方針

### 5-1. 単純実装（Phase 2 v1）

```rust
pub async fn cross_search(req: CrossSearchRequest) -> Result<Vec<SearchResult>> {
    let workspaces = req.workspaces
        .unwrap_or_else(|| self.list_workspaces());

    let mut all_results = Vec::new();
    for ws_name in workspaces {
        let ws = self.workspace(&ws_name)?;
        let results = ws.search(&req.into_workspace_search()).await?;
        all_results.extend(results.into_iter().map(|r| {
            SearchResult { workspace: ws_name.clone(), ..r }
        }));
    }

    // リランキング
    all_results.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());
    all_results.truncate(req.limit);
    Ok(all_results)
}
```

**問題**: workspace が増えると線形にコストが上がる。

### 5-2. 最適化（Phase 2 v2）

- workspace ごとに pool.db を `Arc<Mutex<Connection>>` でキャッシュ
- 並列実行（tokio::spawn）
- 各 workspace の上位 N 件だけ取って合算（早期打ち切り）
- 検索頻度の高い workspace は優先キャッシュ

### 5-3. 究極形（Phase 4 以降）

- 全 workspace のメタデータを `shared/index.db` に集約（読み取り専用ミラー）
- workspace 内の変更を WAL hook で集約 DB に反映
- 横断検索は集約 DB 一発で完結

---

## 6. UI/UX ガイド（消費者アプリ向け）

AkariPool 自体は GUI を持たないが、消費者アプリ（AKARI Video 等）が Workspace を扱う際の標準的な見せ方:

### 6-1. Workspace セレクタ

VSCode の `File > Open Workspace` 風:

- アプリ起動時に workspace 選択ダイアログ
- ウィンドウタイトルバーに現在の workspace 表示
- セレクタクリックで切替（編集中の状態を保存して切替）

### 6-2. Workspace 横断ビュー（オプション）

「全 workspace を見る」モードを別画面として用意:

- 検索ボックス + workspace フィルタ
- アイテム表示時に「どの workspace か」を必ず表示
- ドラッグで workspace 間移動 → 内部的には `pool.move_item` を呼ぶ

### 6-3. Workspace ごとの推奨色・アイコン

- `workspace.toml` の `color` と `icon` を尊重
- アプリ全体のアクセントカラーが workspace に応じて変わると識別しやすい

---

## 7. 他システムとの比較

| システム | Workspace 概念 | 隔離強度 | 横断 |
|---|---|---|---|
| **VSCode** | `.code-workspace` ファイル | プロセス分離 | 別ウィンドウ |
| **Obsidian** | Vault | フォルダ別、別アプリ起動 | プラグインで横断可 |
| **Notion** | Workspace | テナント分離 | 検索のみ横断 |
| **Slack** | Workspace | 完全分離 | 不可（別ログイン） |
| **AkariPool** | Workspace | ファイル+SQLite別 | crate API + `--cross` で明示横断 |

AkariPool は **「Obsidian Vault の物理隔離 + Notion の workspace UX + 明示的横断 API」** の組み合わせ。

---

## 8. セキュリティ考慮事項

### 8-1. パストラバーサル対策

```rust
// ❌ 悪い例
let path = format!("{}/{}", base, workspace_name);

// ✅ 良い例
let workspace_name = sanitize_workspace_name(workspace_name)?; // 英数+ハイフンのみ
let path = base.join(&workspace_name);
let canonical = path.canonicalize()?;
if !canonical.starts_with(&base.canonicalize()?) {
    return Err(Error::PathTraversal);
}
```

### 8-2. Workspace 名のバリデーション

正規表現: `^[a-z0-9][a-z0-9_-]{0,63}$`

- 小文字 + 数字 + ハイフン + アンダースコアのみ
- 先頭は英数字
- 1〜64文字

### 8-3. 横断検索の権限

将来マルチユーザー対応した場合:

- workspace ごとに owner/reader を設定
- 横断検索は reader 権限のある workspace のみ対象

---

## 9. 未解決の論点

- [ ] **Workspace のネスト**を許すか？（例: `clients/acme/wedding-2026`）→ v0.1 ではフラット
- [ ] **Workspace のアーカイブ**機能（read-only にして検索対象から外す）
- [ ] **Workspace のタイプ別テンプレート**（"video_project" を選ぶと初期 Analyzer 設定が来る）
- [ ] **Workspace 越え関係**の DB スキーマ詳細（`cross_relations.db` の構造）
- [ ] **複数マシン同期**したときの workspace ID 衝突（UUID にすべき？）
- [ ] **AMP との統合**: workspace = エージェント memory scope と一致させるか？
- [ ] **AKARI Video 側**の「プロジェクト」概念と workspace の対応関係
  - 案A: 1プロジェクト = 1 workspace（直接対応）
  - 案B: workspace = 上位の文脈、プロジェクトはその中の概念
  - 案C: 既存 AKARI プロジェクトを workspace に自動マイグレーション

---

## 10. 参考実装

実装は Phase 2 で `crates/pool-workspace/` に着手。本ドキュメントは API 契約として更新していく。

関連:

- [`architecture.md`](./architecture.md) — 全体アーキテクチャ
- [`analyzer-plugin.md`](./analyzer-plugin.md) — Analyzer プラグイン（次に書く）
