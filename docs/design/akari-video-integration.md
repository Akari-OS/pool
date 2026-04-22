# AKARI Video 統合設計書

> **バージョン**: v0.1 Draft
> **最終更新**: 2026-04-08
> **対象**: AKARI Video (`~/_project/akari-video/`) の Pool モジュールを AkariPool に置き換える

---

## 1. 目的

AKARI Video の `src-tauri/src/pool/` モジュールを **AkariPool への薄いラッパに置き換え**、AKARI Video を「AkariPool の動画系ビュー」として再定義する。

### ゴール
- AKARI Video 側のコードベースを軽量化（pool ロジックを delete）
- 同じ Pool を AKARI Video / AKARI CMS / akari-pool CLI / 外部 AI から共有
- ユーザー体験は変えない（既存の Pool 画面はそのまま動く）
- 段階的移行で **Distributed Monolith の罠を避ける**

### 非ゴール
- AKARI Video の動画編集機能の変更
- ユーザーの既存プールデータの破壊（マイグレーションは慎重に）

---

## 2. 現状（Before）

### 2-1. AKARI Video 内の Pool 関連コード

```
~/_project/akari-video/src-tauri/src/
├── pool/                                    ← これを全部 AkariPool に移譲
│   ├── mod.rs
│   ├── db.rs                                — SQLite スキーマ + マイグレーション v1-v5
│   ├── crud.rs                              — CRUD + FTS5 検索 + リレーション操作
│   ├── analysis.rs                          — 旧来型分析パイプライン
│   ├── m2c/
│   │   ├── types.rs                         — M2CContext L0-L3
│   │   ├── modules.rs                       — モダリティ別ディスパッチャ
│   │   └── graph.rs                         — セマンティックリレーション生成
│   └── ...
│
├── commands/
│   └── pool_commands.rs                     ← Tauri State<PoolDb> に依存
│
└── lib.rs                                   ← PoolDb Singleton を .manage()
```

### 2-2. データ配置

- `~/.akari/pool/pool.db` (SQLite WAL)
- `~/.akari/pool/files/{uuid}/` (raw ファイル)

### 2-3. 課題
- AKARI Video のプロセス内に Pool DB が組み込まれており、他アプリから使えない
- `Tauri::State<PoolDb>` への結合度が高い
- MCP ツールが毎回 SQLite を open し直している（Connection 共有なし）
- スキーマが AKARI Video 専用設計（汎用化されていない）

---

## 3. 目標（After）

### 3-1. 構成

```
AKARI Video (~/_project/akari-video/)
├── src-tauri/src/
│   ├── pool/                                ← 削除（AkariPool に移譲）
│   │   └── (削除)
│   │
│   ├── pool_client.rs                       ← NEW: AkariPool への薄いラッパ
│   │
│   ├── commands/
│   │   └── pool_commands.rs                 ← pool_client を呼ぶように変更
│   │
│   └── Cargo.toml                           ← akari-pool-core を依存追加
│
└── src/                                     ← フロントエンドは触らない
    └── components/pool/                     ← 既存 UI そのまま動く
```

### 3-2. データ配置の移行

```
旧: ~/.akari/pool/pool.db
新: ~/.akari-pool/workspaces/akari-video-default/pool.db
```

旧パスから新パスへの **自動マイグレーション**を `pool_client.rs` の起動時に1回だけ実行。

### 3-3. 依存関係

`akari-video/src-tauri/Cargo.toml`:
```toml
[dependencies]
akari-pool-core = { path = "../../akari-pool-impl/crates/pool-core" }
akari-pool-workspace = { path = "../../akari-pool-impl/crates/pool-workspace" }
```

開発時は path 依存、リリース時は git tag 参照に切り替え。

---

## 4. 移行ステップ

### Step 1: AkariPool Phase 1 完了を待つ ⏳
- `pool-core` の SQLite + CRUD 実装
- `pool-workspace` の Workspace 隔離
- 単体テスト通過

### Step 2: 新スキーマへのマッピング設計
AKARI Video の既存スキーマ v5 と AkariPool スキーマ v1 の差分を埋めるマッピング表を作る:

| AKARI Video v5 | AkariPool v1 | マッピング |
|---|---|---|
| `pool_items.id` | `pool_items.id` | 直接 |
| `pool_items.name` | `pool_items.name` | 直接 |
| `pool_items.file_path` | `pool_items.file_path` | パス書き換え（workspace 相対に） |
| `pool_items.item_type` | `pool_items.item_type` | enum 値統一 |
| `pool_items.role` | `pool_items.role` | 直接 |
| `pool_items.layer` | `pool_items.layer` | 直接 |
| `pool_items.context_json` | `pool_items.context_json` | 直接 |
| `pool_items.context_compressed` | `pool_items.context_compressed` | 直接 |
| `pool_items.context_embedding` | `pool_items.context_embedding` | 直接 |
| `pool_items.ffprobe_json` | `pool_items.analyzer_raw` | リネーム |
| (なし) | `pool_items.analyzer_name` | "video-m2c" 固定値で埋める |
| (なし) | `pool_items.analyzer_version` | M2C 分析時のバージョン |
| `pool_relations.*` | `pool_relations.*` | ほぼ直接（`target_workspace` は NULL） |

### Step 3: マイグレーションスクリプト
AkariPool 側に `crates/pool-core/src/migrate/from_akari_video_v5.rs` を用意:

```rust
pub fn migrate_from_akari_video_v5(
    old_db_path: &Path,
    new_pool: &Pool,
    workspace_name: &str,
) -> Result<MigrationReport> {
    // 1. 旧 DB を read-only で open
    // 2. 新 workspace を作成
    // 3. pool_items を1件ずつ変換して INSERT
    // 4. raw ファイルを workspace 配下にコピー（ハードリンク or シンボリックリンク）
    // 5. pool_relations を移行
    // 6. 検証: 件数一致、ハッシュ一致
    // 7. 旧 DB はバックアップとして保持（削除しない）
}
```

### Step 4: pool_client.rs を実装
AKARI Video 側に薄いラッパを作る:

```rust
// src-tauri/src/pool_client.rs
use akari_pool_core::Pool;
use akari_pool_workspace::WorkspaceConfig;

pub struct PoolClient {
    pool: Pool,
    default_workspace: String,
}

impl PoolClient {
    pub fn new() -> Result<Self> {
        let pool = Pool::open(akari_pool_root())?;

        // 初回起動時: マイグレーション
        if needs_migration() {
            run_migration(&pool)?;
        }

        // デフォルト workspace を取得 or 作成
        let ws_name = "akari-video-default".to_string();
        if !pool.workspace_exists(&ws_name)? {
            pool.create_workspace(WorkspaceConfig {
                name: ws_name.clone(),
                display_name: Some("AKARI Video デフォルト".into()),
                intent: Some("video_project".into()),
                icon: Some("🎬".into()),
                ..Default::default()
            })?;
        }

        Ok(Self { pool, default_workspace: ws_name })
    }

    // 既存 Tauri コマンドが期待する API をそのまま提供
    pub async fn list_items(&self) -> Result<Vec<PoolItem>> {
        let ws = self.pool.workspace(&self.default_workspace)?;
        ws.list_items().await
    }

    // ... 他の薄いラッパ
}
```

### Step 5: pool_commands.rs を書き換え
既存の `Tauri::State<PoolDb>` を `Tauri::State<PoolClient>` に置換:

```rust
#[tauri::command]
pub async fn pool_list(client: State<'_, PoolClient>) -> Result<Vec<PoolItem>, String> {
    client.list_items().await.map_err(|e| e.to_string())
}
```

### Step 6: 旧 pool/ モジュールを削除
- `src-tauri/src/pool/` 配下を delete
- `lib.rs` から関連の `mod pool` を削除
- `Cargo.toml` から旧依存を整理

### Step 7: 動作確認
- 全フロントエンドテストを通す
- 既存のプール画面で表示・追加・削除が動くことを目視確認
- マイグレーション前後でアイテム数・関係数が一致することを確認

### Step 8: MCP サーバーの統合
AKARI Video の `src-tauri/src/mcp/` を以下のように扱う:

**選択肢 A**: AKARI Video の MCP サーバーを残し、内部を `pool_client` に置き換える
- メリット: 既存 109 ツールを維持、ユーザー影響なし
- デメリット: 二重実装

**選択肢 B**: AKARI Video の MCP サーバーを削除し、akari-pool MCP サーバーに完全移行
- メリット: 1つのソースで済む
- デメリット: AKARI Video 固有の動画編集ツールが行き場を失う

**推奨**: **選択肢 C**（中道）
- pool 系 MCP ツールは akari-pool MCP に移譲
- AKARI Video の MCP サーバーは「動画編集ツール（カット/結合/書き出し等）」のみに絞る
- 両方を並列起動して「pool 系は AkariPool に、編集系は AKARI Video に」とロールを分ける

### Step 9: AkariPool プロジェクトを正式リリース
- v0.1.0 タグ
- README / docs を整備
- 他の人に貸し出し可能に

---

## 5. リスクと対策

### リスク 1: マイグレーション中のデータ損失
**対策**:
- 旧 DB は絶対に削除しない（rename して保持）
- マイグレーション前にユーザーに明示的な確認ダイアログ
- ハッシュベースの整合性チェック
- ロールバック手順をドキュメント化

### リスク 2: パフォーマンス劣化
**対策**:
- 旧実装は同一プロセス内 SQLite アクセス
- 新実装も最初は同一プロセス内（path 依存）なので速度は同じ
- 将来 Phase 3 で別プロセス化したときに計測 → 100µs/call の許容範囲内

### リスク 3: 既存のフロントエンドが壊れる
**対策**:
- Tauri コマンドのシグネチャは変えない（PoolClient が同じ型を返す）
- フロントエンドコードは一切触らない
- E2E テストで担保

### リスク 4: AkariPool の Phase 1 が遅れて AKARI Video の開発が止まる
**対策**:
- AKARI Video は **AkariPool に依存しない** 旧 pool 実装を残したまま開発継続可能
- マイグレーションは「opt-in feature flag」として段階導入
- 完全切り替えは AkariPool が安定してから

---

## 6. タイムライン（仮）

| Phase | 内容 | 期間目安 |
|---|---|---|
| **A** | AkariPool Phase 1 (pool-core 実装) 完了を待つ | TBD |
| **B** | スキーマ・マッピング設計 | 数日 |
| **C** | マイグレーションスクリプト実装 | 数日 |
| **D** | pool_client.rs 実装 + Tauri コマンド書き換え | 1週間 |
| **E** | 旧 pool/ 削除 + テスト | 数日 |
| **F** | MCP サーバー再設計 | 1週間 |
| **G** | AkariPool v0.1.0 リリース | TBD |

合計: AkariPool Phase 1 完了後、約 3〜4 週間で AKARI Video 統合完了。

---

## 7. 関連ドキュメント

- [`architecture.md`](./architecture.md) — AkariPool 全体設計
- [`workspace-layer.md`](./workspace-layer.md) — Workspace モデル
- [`analyzer-plugin.md`](./analyzer-plugin.md) — Analyzer Plugin
- AKARI Video 側: `docs/research/hosted-filesystems-for-agents-survey-2026.md`
- AKARI Video 側: `docs/design/akari-pkb-architecture-visual.html`

---

## 8. 未解決の論点

- [ ] **マイグレーション時の UX**: 自動 vs 手動確認 vs 段階的
- [ ] **複数 AKARI Video プロジェクト**を別 workspace にすべきか、1つの workspace に集約すべきか
  - 案 A: 1 AKARI プロジェクト = 1 workspace（直接対応）
  - 案 B: workspace = ユーザー意図、プロジェクトはその中の概念
- [ ] **Cargo.lock の管理**: 開発時の path 依存と git tag 依存をどう切り替えるか
- [ ] **macOS App Sandbox**: AKARI Video の sandbox 内から `~/.akari-pool` にアクセスできるか検証
- [ ] **既存の M2C 分析結果**を再利用するか、Analyzer 移行時に再分析するか
- [ ] **MCP サーバーの複数起動**: AKARI Video MCP と AkariPool MCP を並列起動する場合のポート/socket 衝突
