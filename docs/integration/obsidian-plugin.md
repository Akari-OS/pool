# Obsidian プラグイン設計書

> **バージョン**: v0.1 Draft
> **最終更新**: 2026-04-08
> **プラグイン名（仮）**: `obsidian-akari-pool`
> **将来のリポジトリ**: `Akari-OS/obsidian-pool`（将来リポ名・未作成。独立プロジェクト化を推奨）

---

## 1. 目的

AkariPool の Wiki 層（`~/.akari-pool/libraries/{name}/notes/`）を Obsidian Vault としてマウントし、**Obsidian を AkariPool のリッチクライアントにする**。動画・音声・PDF 全部 Obsidian 内で再生・閲覧でき、関係グラフを Obsidian の Graph View で可視化できる。

### なぜ Obsidian か

- 🧠 **既に世界中で使われている**: ノート管理の事実上の標準
- 📝 **Markdown ネイティブ**: AkariPool の Wiki 層と完全互換
- 🎬 **メディア再生標準対応**: 動画/音声/PDF/画像をネイティブで開ける
- 🔌 **プラグイン API が成熟**: TypeScript で何でもできる
- 🔗 **バックリンク・Graph View**: 関係グラフが既に実装されている
- 🏠 **ローカルファースト**: AkariPool の哲学と一致
- 💰 **個人利用無料**: 商用版もリーズナブル

→ **Obsidian = AkariPool のリーディングUI** という位置付けが最適。

---

## 2. Obsidian の標準メディア対応

プラグインなしで何が動くか確認しておく:

| 形式 | 対応 | 記法 |
|---|---|---|
| 🎬 MP4 / WebM / OGV | ✅ | `![[video.mp4]]` |
| 🎵 MP3 / WAV / OGG / M4A / FLAC | ✅ | `![[audio.mp3]]` |
| 📄 PDF (ページ指定可) | ✅ | `![[paper.pdf#page=3]]` |
| 🖼️ JPG / PNG / WebP / GIF / SVG | ✅ | `![[image.png]]` |
| 📊 CSV | ⚠️ プレーンテキスト | DataView プラグインで強化 |
| 💻 コード | ✅ シンタックスハイライト | コードブロック |
| 🌐 HTML | ⚠️ 一部 | iframe プラグイン |

### 標準で動かないもの → プロキシ生成で対応

| 問題 | 解決策 |
|---|---|
| 📷 HEIC / HEIF (iPhone 写真) | AkariPool 取り込み時に JPEG プロキシ生成 |
| 🎥 HEVC / H.265 動画 | 取り込み時に H.264/MP4 プロキシ生成 |
| 🎬 ProRes / DNxHD | 同上 |
| 🎞️ MKV (一部コーデック) | 同上 |

→ AKARI Video の既存 FFmpeg ラッパで対応可能。`workspaces/{ws}/files/{uuid}/proxy.mp4` のような命名で並置する。

---

## 3. プラグインができること（機能仕様）

### 3-1. コア機能（v0.1 — MVP）

| # | 機能 | 説明 | 工数 |
|---|---|---|---|
| 1 | **Vault マウント自動検出** | `~/.akari-pool/libraries/*/notes/` を Obsidian Vault として開ける | 半日 |
| 2 | **Workspace セレクタ** | 左サイドバーで workspace 切替 | 1日 |
| 3 | **`pool://` リンク解決** | `[[pool:item-id]]` で他アイテムへリンク、自動補完 | 1日 |
| 4 | **メタデータパネル** | 開いているノートの Pool DB 情報を右ペインに表示 | 1日 |
| 5 | **Pool 検索コマンド** | `Cmd+Shift+P` → "AkariPool: 検索" で FTS5 検索 | 1日 |
| 6 | **動画/音声プロキシ自動切替** | HEVC 等を検出してプロキシに置換 | 半日 |

→ **MVP は 1〜2 週間で動く**。

### 3-2. 拡張機能（v0.2〜）

| 機能 | 説明 |
|---|---|
| **関係グラフ統合** | Obsidian の Graph View に `pool_relations` をマージ表示 |
| **Linting バッジ** | 右ペインに Lint 結果（表記揺れ・欠損・関係抜け）を表示 |
| **タグ管理 UI** | `shared/tags.db` の正規化辞書を編集 |
| **Filed back 通知** | AI が新しい関係を追加したら notice |
| **LLM モデルセレクタ** | 設定 paneで AkariPool の LLM をワンクリック切替 |
| **コマンドパレット拡張** | "AkariPool: 再分析" "AkariPool: Lint 実行" 等 |
| **タイムライン埋め込み** | AKARI Video のタイムラインを iframe で埋め込み |
| **MCP クライアント機能** | プラグイン自体が MCP クライアントになって `pool-mcp` を叩く |

### 3-3. 上級機能（v1.0〜）

- **シーン別ノート**: 動画の特定シーンに紐づくノートを作成（タイムスタンプリンク）
- **コラボ機能**: 複数ユーザーで Pool を共有（Cloud sync 経由）
- **Recipe Builder**: AKARI Video のレシピを Obsidian で組み立て

---

## 4. アーキテクチャ

### 4-1. データアクセスの3方式

プラグインから AkariPool データへのアクセスは3パターンある:

#### 方式 A: 直接 SQLite アクセス（最速）

```typescript
import Database from 'better-sqlite3';

const db = new Database('~/.akari-pool/libraries/wedding-2026/pool.db', {
  readonly: true
});
const items = db.prepare('SELECT * FROM pool_items LIMIT 10').all();
```

**メリット**: 起動が早い、依存少ない
**デメリット**: AkariPool 側のスキーマ変更に弱い、書き込みは慎重に

#### 方式 B: MCP クライアント

```typescript
import { McpClient } from '@modelcontextprotocol/sdk';

const client = new McpClient({
  endpoint: 'http://localhost:7890',  // akari-pool mcp-serve --http
});
const result = await client.callTool('pool_search', { query: '結婚式' });
```

**メリット**: スキーマ抽象化、リアルタイム通知 (subscribe)
**デメリット**: `akari-pool mcp-serve` 起動が必要、HTTP オーバーヘッド

#### 方式 C: Sidecar プロセス（最も洗練）

プラグインが起動時に `akari-pool mcp-serve --stdio` をサブプロセスとして起動。stdin/stdout で MCP 通信。Obsidian 終了時に kill。

**メリット**: 自動起動、ユーザーは何も意識せず
**デメリット**: クロスプラットフォーム配布が複雑

### 4-2. 推奨方針

**v0.1 (MVP)**: 方式 A + 限定的な書き込み（`pool_relations` だけ）
**v0.2**: 方式 B のサポート追加（ユーザーが選べる）
**v1.0**: 方式 C を標準に

### 4-3. 全体構造

```
~/.akari-pool/libraries/wedding-2026/
├── files/                              ← raw 層（動画・音声・画像）
│   ├── {uuid-1}/ceremony.mp4
│   └── {uuid-1}/ceremony_proxy.mp4    ← Obsidian 用プロキシ
├── notes/                              ← Obsidian Vault のルート
│   ├── .obsidian/
│   │   ├── plugins/
│   │   │   └── obsidian-akari-pool/   ← プラグイン本体
│   │   │       ├── main.js
│   │   │       ├── manifest.json
│   │   │       ├── styles.css
│   │   │       └── data.json
│   │   ├── workspace.json
│   │   └── ...
│   ├── {uuid-1}.md                    ← Wiki エントリ
│   └── ...
└── pool.db                             ← SQLite (プラグインから read-only)
```

### 4-4. プラグイン内部構成

```
obsidian-akari-pool/src/
├── main.ts                    — エントリ、Plugin クラス
├── views/
│   ├── pool-sidebar.ts        — 左サイドバー（workspace セレクタ）
│   ├── metadata-panel.ts      — 右ペイン（メタデータ表示）
│   ├── search-modal.ts        — 検索モーダル
│   └── lint-panel.ts          — Linting 結果表示
├── data/
│   ├── pool-db.ts             — SQLite クライアント
│   ├── mcp-client.ts          — MCP クライアント（v0.2）
│   └── workspace-detector.ts  — Vault → workspace 検出
├── commands/
│   ├── search.ts              — Pool 検索コマンド
│   ├── reanalyze.ts           — 再分析コマンド
│   └── lint.ts                — Lint 実行コマンド
├── ui/
│   ├── components/            — 再利用可能 UI 部品
│   └── icons.ts
├── utils/
│   ├── proxy-resolver.ts      — HEVC → MP4 プロキシ自動切替
│   ├── link-resolver.ts       — pool:// リンク解決
│   └── markdown-extender.ts
└── settings.ts                — プラグイン設定
```

---

## 5. Wiki エントリのフォーマット

AkariPool が生成する `.md` は Obsidian で開くことを前提に最適化する:

```markdown
---
pool_id: a1b2c3d4-...
name: ceremony.mp4
type: video
workspace: wedding-2026
mime_type: video/mp4
analyzed_at: 2026-04-08T16:00:00+09:00
analyzer: video-m2c v0.2
tags:
  - 結婚式
  - 屋外
  - セレモニー
duration_sec: 432
size_mb: 245
---

# ceremony.mp4

屋外での結婚式セレモニー。晴天、参列者多数。明るい雰囲気。

## 動画

![[../files/a1b2c3d4-.../ceremony_proxy.mp4]]

## シーン

| 開始 | 終了 | 内容 | キーフレーム |
|---|---|---|---|
| 00:00 | 00:45 | 入場シーン、明るい屋外 | ![[../files/a1b2c3d4-.../scene_0.jpg]] |
| 00:45 | 02:30 | 誓いの言葉 | ![[../files/a1b2c3d4-.../scene_1.jpg]] |
| 02:30 | 04:12 | リング交換、BGM フェードイン | ![[../files/a1b2c3d4-.../scene_2.jpg]] |

## 関連素材

- [[bgm-001|🎵 wedding-bgm.wav]] — style_used (strength=0.8)
- [[rehearsal-001|🎬 rehearsal.mp4]] — variant (strength=0.6)
- [[script-001|📝 ceremony-script.md]] — reference (strength=0.9)

## メタ

- **役割**: source
- **層**: content
- **追加日**: 2026-04-08
- **元のパス**: `~/Movies/wedding/ceremony.mp4`
- **AkariPool**: [pool://item/a1b2c3d4](pool://item/a1b2c3d4)
```

→ **Obsidian で開くだけで動画再生 + シーン別キーフレーム表示 + 関連素材へのバックリンク**が全部効く。プラグインなしでも 80% 動く。プラグインで残り 20%（リアルタイム同期、検索、LLM 切替）を補う。

---

## 6. UI 設計

### 6-1. メイン画面レイアウト

```
┌──────────────────────────────────────────────────────────────┐
│  Obsidian + obsidian-akari-pool                               │
├──────────────────────────────────────────────────────────────┤
│ ┌──────┐ ┌─────────────────────────┐ ┌────────────────────┐ │
│ │ 📁 WS │ │  ceremony.md             │ │ 📋 Metadata        │ │
│ │       │ │                          │ │                    │ │
│ │wedding│ │  # ceremony.mp4          │ │ ID: a1b2c3d4      │ │
│ │  ●    │ │                          │ │ Type: video        │ │
│ │backend│ │  屋外での結婚式...        │ │ Duration: 7:12    │ │
│ │sns    │ │                          │ │ Size: 245 MB      │ │
│ │       │ │  [▶ video player]        │ │                   │ │
│ │ ─────│ │                          │ │ 🏷️ Tags           │ │
│ │ 🔍 Q │ │  ## シーン                │ │  結婚式            │ │
│ │       │ │  - 00:00 入場            │ │  屋外              │ │
│ │ ⭐3   │ │  - 00:45 誓い            │ │  セレモニー        │ │
│ │ Lint  │ │                          │ │                    │ │
│ │       │ │  ## 関連                  │ │ 🔗 Relations      │ │
│ │ 🤖    │ │  - [[bgm-001]]            │ │  → bgm-001        │ │
│ │qwen3:7b│ │  - [[rehearsal-001]]    │ │  → rehearsal-001  │ │
│ │       │ │                          │ │                   │ │
│ │ 📊    │ │                          │ │ ⚠️ Lint           │ │
│ │$1.23  │ │                          │ │ なし              │ │
│ └──────┘ └─────────────────────────┘ └────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### 6-2. 左サイドバー要素

| 要素 | 機能 |
|---|---|
| 📁 Workspace 一覧 | クリックで切替 |
| 🔍 検索バー | FTS5 全文検索 |
| ⭐ Lint バッジ | 未解決の Lint 件数 |
| 🤖 LLM セレクタ | 現在のモデル + ワンクリック切替 |
| 📊 コスト表示 | 今月の消費 |

### 6-3. コマンドパレット拡張

`Cmd+Shift+P` で:

- AkariPool: 検索
- AkariPool: 新規 Pool アイテム追加
- AkariPool: このアイテムを再分析
- AkariPool: Lint を実行
- AkariPool: Workspace を切替
- AkariPool: LLM モデルを変更
- AkariPool: コスト ダッシュボード

---

## 7. LLM モデル切替 UI

[`llm-strategy.md`](../design/llm-strategy.md) と整合する形で、Obsidian 内でモデル切替 UI を提供:

```
┌─────────────────────────┐
│ 🤖 LLM Settings          │
├─────────────────────────┤
│                          │
│ Active Profile:          │
│ ◉ ⚖️ Balanced            │
│ ○ 💰 Saver               │
│ ○ ⚡ Performance         │
│ ○ 🔒 Offline             │
│                          │
│ Per Task:                │
│  Light: qwen3:4b ▼       │
│  Std:   qwen3:7b ▼       │
│  Heavy: deepseek/v3.2 ▼  │
│  Vision: qwen2.5-vl:7b ▼ │
│                          │
│ Status:                  │
│  ✅ Ollama running       │
│  ✅ OpenRouter (key set) │
│                          │
│ This Month: $1.23/$10    │
│ ▓▓░░░░░░░░ 12.3%        │
│                          │
└─────────────────────────┘
```

設定変更は即座に AkariPool 側の `workspace.toml` に書き込まれる。

---

## 8. 開発スタック

| 項目 | 技術 |
|---|---|
| 言語 | TypeScript |
| ランタイム | Obsidian Plugin API (Electron) |
| ビルド | esbuild |
| SQLite | better-sqlite3 |
| MCP クライアント | @modelcontextprotocol/sdk |
| UI フレームワーク | Obsidian 標準 + Svelte (オプション) |
| テスト | Vitest |

### サンプル `manifest.json`

```json
{
  "id": "obsidian-akari-pool",
  "name": "AkariPool",
  "version": "0.1.0",
  "minAppVersion": "1.5.0",
  "description": "Universal Knowledge Store integration for AkariPool",
  "author": "AKARI Project",
  "authorUrl": "https://akari-oss.app",
  "isDesktopOnly": true
}
```

---

## 9. 配布方法

### Phase A: GitHub リリース（手動インストール）

ユーザーは `obsidian-akari-pool/main.js` を `.obsidian/plugins/obsidian-akari-pool/` に配置。

### Phase B: BRAT 経由

[obsidian42-brat](https://github.com/TfTHacker/obsidian42-brat) で β テスター向けに配布。

### Phase C: Community Plugin Store 申請

正式リリース後、Obsidian Community Plugin Store に申請。承認されればワンクリックインストール可能。

---

## 10. ロードマップ

| Version | 内容 | 期間 |
|---|---|---|
| **v0.1 MVP** | 6つのコア機能、SQLite 直アクセス | 1〜2週間 |
| **v0.2** | MCP クライアント対応、Lint UI | 1〜2週間 |
| **v0.3** | LLM モデルセレクタ、コスト表示 | 1週間 |
| **v0.5** | グラフ統合、タグ管理 UI | 1週間 |
| **v1.0** | Filed back 通知、シーン別ノート、Community Plugin 申請 | 2週間 |

→ **トータル 6〜8 週間で v1.0**。

---

## 11. リスクと制約

| リスク | 対策 |
|---|---|
| Obsidian の API breaking change | minAppVersion を厳密に指定、CI でバージョンマトリクステスト |
| SQLite のロック競合 | プラグインは read-only、書き込みは MCP 経由 |
| 大きい動画ファイルの再生負荷 | プロキシ生成（小さく軽い MP4） |
| HEVC 等の非対応コーデック | プロキシ生成 |
| 複数 Vault 間の workspace 衝突 | Vault パスから workspace を一意特定する規約 |
| Cross-platform 動作（Win/Linux）| `better-sqlite3` の prebuild 配布、初期は macOS のみサポート |

---

## 12. 関連ドキュメント

- [`../design/architecture.md`](../design/architecture.md) — AkariPool 全体設計
- [`../design/workspace-layer.md`](../design/workspace-layer.md) — Workspace モデル
- [`../design/analyzer-plugin.md`](../design/analyzer-plugin.md) — Analyzer Plugin
- [`../design/llm-strategy.md`](../design/llm-strategy.md) — LLM モデル戦略
- [`../design/akari-video-integration.md`](../design/akari-video-integration.md) — AKARI Video 統合

---

## 13. 未解決の論点

- [ ] **複数 workspace の同時表示**: 1つの Obsidian ウィンドウで複数 workspace を見たい場合の UI
- [ ] **プラグインを別プロジェクトとして独立化するか**: `Akari-OS/obsidian-pool`（将来リポ名・未作成）を作るタイミング
- [ ] **Cross-platform**: Windows / Linux ユーザーへの対応優先度
- [ ] **モバイル対応**: Obsidian Mobile からの利用は可能か（iOS だと SQLite ネイティブが厳しい）
- [ ] **AkariPool MCP との認証**: OAuth フロー or トークン
- [ ] **Vault 内の `.obsidian/` の取り扱い**: AkariPool 側で自動コピーするテンプレートを用意するか
- [ ] **既存 Obsidian Vault との統合**: ユーザーの既存 Vault に AkariPool を「サブ Vault」として組み込めるか
