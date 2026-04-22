# AkariPool — ロードマップ

> 最終更新: 2026-04-09
> 上位ビジョン: `../../CLAUDE.md`（北極星）
> 要件定義: `../design/requirements.md`
> タスク詳細: `tasks.md`

---

## ビジョン概要

AkariPool は「AkariOS — 個人のための AI OS」構想における **中核データ層**。
AI エージェントが本棚を散歩するように知識を探索できる、ローカルファーストの汎用 Knowledge Store を作る。

- 動画・音声・画像・PDF・記事・コード・データセット・ノート — **どんなモダリティでも入る**
- **GUI を持たない**（DB + Analyzer + MCP + CLI のみ）
- **複数の消費者アプリ**（AKARI Video / 将来の AkariNotes / AkariCMS / AkariSearch）から共有される
- **raw 層 + Wiki 層（.md）** の二段構え。機械可読 / 人間可読 / 生データを同時に持つ
- **Library モデル** で目的別に隔離、明示的に横断可能
- **AGPL-3.0** でソース公開を担保

```
Phase 0-1:  pool-core（SQLite + CRUD + CLI）
            → Pool の基本を動かす

Phase 2:    Library 隔離 + 横断検索
            → 目的別に分離された作業空間

Phase 3-3.5: Analyzer trait + LlmClient + ArticleAnalyzer
             → AI が「読んで理解する」基盤

Phase 4:    Wiki Compile + Relations + Linter
            → raw 層と人間可読層の二段構え

Phase 4.5:  Linter 完成 + Relation CLI
            → 知識ベース健全性監視 4 種を揃える

Phase 5:    Filed-back ループ
            → Obsidian 双方向同期、使うほど賢くなる

Phase 6:    MCP サーバー
            → 外部 AI エージェントから接続

Phase 7:    AKARI Video 統合
            → 消費者アプリ #1 を pool-core 依存に

Phase 8:    別アプリ実装
            → AkariNotes / AkariCMS / AkariSearch
```

---

## 方針

**Phase は順番に着実に全部やり切る。並列化・飛ばし・中抜きはしない。**

各 Phase は：

1. `requirements.md` の該当 FR を特定・追記
2. 設計（既存 `design/*.md` か新規）
3. `tasks.md` にタスク展開
4. 実装 → `cargo build` → `cargo test --workspace` → 実機スモーク
5. AC 検証
6. commit → push
7. handoff 更新（大きな節目のみ）

feature-dev フロー: Discovery → 質問整理 → アーキ設計 → 承認 → 実装 → レビュー → 修正 → commit。

---

## 全体像

```
═══ Phase 0: 基盤（完了）═══════════════════════════════════════════

  Cargo workspace 構造              ██████████ 100%  ✅
  設計書（architecture / layers）   ██████████ 100%  ✅
  GitHub リポジトリ + CI 準備       ██████████ 100%  ✅

═══ Phase 1: pool-core 基本 CRUD（完了）════════════════════════════

  SQLite schema v1 + WAL            ██████████ 100%  ✅
  Pool::open / add / list / ...     ██████████ 100%  ✅
  MIME 自動判定（8 モダリティ）      ██████████ 100%  ✅
  CLI info / add / ls / cat / rm    ██████████ 100%  ✅

═══ Phase 2: Library 隔離 + 横断検索（完了）══════════════════════

  Pool を library マネージャに     ██████████ 100%  ✅
  Library 構造体 + CRUD            ██████████ 100%  ✅
  library.toml + config.toml       ██████████ 100%  ✅
  FTS5 検索 + --cross              ██████████ 100%  ✅
  パストラバーサル対策             ██████████ 100%  ✅

═══ Phase 3: Analyzer trait + LlmClient（完了）═════════════════════

  Analyzer trait + Registry         ██████████ 100%  ✅
  LlmClient + OpenAiCompatible      ██████████ 100%  ✅
  ArticleAnalyzer (.md MVP)         ██████████ 100%  ✅
  .env 自動読み込み                  ██████████ 100%  ✅
  add --analyze / analyze           ██████████ 100%  ✅

═══ Phase 3.5: 既知の制限を潰す（完了）═════════════════════════════

  FTS5 trigram（日本語部分一致）    ██████████ 100%  ✅
  HTML 対応（html2text）            ██████████ 100%  ✅
  RetryPolicy + Retry-After         ██████████ 100%  ✅

═══ Phase 4: Wiki Compile + Relations + Linter（完了）══════════════

  wiki.rs::compile_item_to_markdown ██████████ 100%  ✅
  Relations CRUD + 型               ██████████ 100%  ✅
  Linter（Missing + Orphan）        ██████████ 100%  ✅
  CLI compile-notes / lint          ██████████ 100%  ✅

═══ Phase 4.5: Linter 完成 + Relation CLI（次）═════════════════════

  Inconsistency チェッカー          ░░░░░░░░░░   0%  🟡 次
  ConnectionGap チェッカー          ░░░░░░░░░░   0%  🟡 次
  Relation CLI                      ░░░░░░░░░░   0%  🟡 次
  lint show                         ░░░░░░░░░░   0%  🟡 次

═══ Phase 5: Filed-back ループ ════════════════════════════════════

  notes/ 編集 → Pool 反映           ░░░░░░░░░░   0%
  [[backlink]] → relation 自動作成  ░░░░░░░░░░   0%
  モデルプリセット UI               ░░░░░░░░░░   0%

═══ Phase 6: MCP サーバー ═════════════════════════════════════════

  pool-mcp crate（rmcp）            ░░░░░░░░░░   0%
  stdio + Streamable HTTP           ░░░░░░░░░░   0%
  pool_* ツール + pool://library/ リソース  ░░░░░░░░░░   0%

═══ Phase 7: AKARI Video 統合 ═════════════════════════════════════

  AKARI Video を pool-core 依存に   ░░░░░░░░░░   0%
  VideoAnalyzer 実装                ░░░░░░░░░░   0%

═══ Phase 8: 別アプリ実装 ═════════════════════════════════════════

  AkariNotes（Obsidian 風ビュー）   ░░░░░░░░░░   0%
  AkariCMS（記事編集 CMS）          ░░░░░░░░░░   0%
  AkariSearch（横断検索 Web UI）    ░░░░░░░░░░   0%
  AkariBot（Slack/Discord）         ░░░░░░░░░░   0%
```

---

## Phase 依存関係

```
Phase 0（基盤）
  └→ Phase 1（pool-core）
       └→ Phase 2（library）
            └→ Phase 3（analyzer）─ Phase 3.5（制限解消）
                 └→ Phase 4（wiki + lint）
                      ├→ Phase 4.5（lint 完成）─┐
                      ├→ Phase 5（filed-back） ─┤
                      └→ Phase 6（MCP）─────── ┤
                                                 └→ Phase 7（AKARI Video 統合）
                                                      └→ Phase 8（別アプリ）
```

Phase 4.5 / 5 / 6 は Phase 4 を親とする兄弟。3 つとも Phase 7 の前提条件。
「順番に全部やる」方針なので、Phase 4.5 → 5 → 6 → 7 → 8 の直列で進める。

---

## Phase 別ビジョン

### Phase 0: 基盤 ✅ 完了 (2026-04-08)

**目的**: プロジェクト構造・設計書・Cargo workspace を立ち上げる。

**成果物**:
- `Cargo.toml` workspace ルート
- `docs/design/*.md`（architecture / library-layer / analyzer-plugin / llm-strategy / akari-video-integration）
- GitHub リポジトリ（Private）

### Phase 1: pool-core 基本 CRUD ✅ 完了 (2026-04-08)

**目的**: SQLite に raw アイテムを投入・取得・削除できる最小構成を成立させる。

**完了基準**:
- `akari-pool add <file>` → `ls` で可視、`cat` で取得可能、`rm` で削除可能
- `pool.db` が WAL モードで開く
- MIME 自動判定で ItemType が確定

### Phase 2: Library 隔離 + 横断検索 ✅ 完了 (2026-04-08)

**目的**: 目的別に作業空間を隔離し、明示的に横断検索できるようにする。

**完了基準**:
- `library create / list / use / info / delete` が動く
- カレント library が `--library` > env > config > "default" で解決される
- パストラバーサル攻撃を拒否する
- `search --cross` で全 library 横断できる

### Phase 3: Analyzer trait + LlmClient + ArticleAnalyzer ✅ 完了 (2026-04-08)

**目的**: AI が raw データを読んで要約・タグ付けする基盤を作る。まずは `.md` 記事を対象。

**完了基準**:
- `Analyzer` trait + Registry が `pool-core` にある
- `OpenRouter` 経由で ArticleAnalyzer が `.md` を分析し、DB に ai_summary / ai_tags を書く
- 失敗時も item は保持（パーシャル成功）

### Phase 3.5: 既知の制限を潰す ✅ 完了 (2026-04-08)

**目的**: Phase 3 の MVP で先送りした制限（日本語検索・HTML 未対応・リトライなし）を塞ぐ。

**完了基準**:
- 日本語 3 文字以上で部分一致検索できる
- `.html` / `.htm` も ArticleAnalyzer で分析できる
- LLM 429 / Retry-After を尊重したリトライが動く

### Phase 4: Wiki Compile + Relations CRUD + Linter ✅ 完了 (2026-04-08)

**目的**: raw 層の隣に人間可読な Wiki 層を生やし、関係性と健全性監視の基盤を置く。

**完了基準**:
- `compile-notes` で Obsidian 互換の `.md` が生成される
- Relations CRUD API が動く
- `lint` で Missing / Orphan が検出され DB 保存される
- `analyze` 成功時に auto-compile が連鎖する

### Phase 4.5: Linter 完成 + Relation CLI 🟡 次

**目的**: Phase 4 で積み残した Lint チェッカー 2 種を実装し、Relation を CLI から操作可能にする。

**内容**:
- Inconsistency チェッカー（編集距離 + LLM でタグ表記揺れ検出）
- ConnectionGap チェッカー（A→B / B→C はあるが A→C が無いケース）
- `akari-pool relation add / list / rm`
- `akari-pool lint show` で保存済み findings 閲覧

**完了基準**:
- 4 種の Lint チェッカーが全て動作
- Relation を CLI から追加・一覧・削除できる
- `requirements.md` の FR-07-06 / FR-08-04 / FR-08-05 / FR-08-07 / FR-09-11 / FR-09-12 を満たす

**依存**: Phase 4 の Relations CRUD と Linter フレームワーク

### Phase 5: Filed-back ループ

**目的**: ユーザーが Obsidian で編集した `notes/{uuid}.md` を Pool に書き戻し、知識ベースが人間・AI 共同編集可能な状態にする。

**内容**:
- `notes/{uuid}.md` の変更監視（`notify` crate or `akari-pool sync`）
- frontmatter の ai_summary / ai_tags 編集を DB に反映
- `[[backlink]]` 追加で relation 自動作成
- モデルプリセット UI（💰節約 / ⚖️バランス / ⚡パフォーマンス / 🔒オフライン / 🧪ハイブリッド / 🎓高品質）
- 設計書新規作成: `docs/design/filed-back-loop.md`

**完了基準**:
- Obsidian で編集 → `akari-pool sync` で Pool 反映
- `[[item-id]]` 記法で relation が作られる
- CLI フラグで 6 プリセットを選択できる

### Phase 6: MCP サーバー

**目的**: 外部 AI エージェント（Claude Code / Cursor 等）から AkariPool にアクセス可能にする。

**内容**:
- `pool-mcp` crate 本実装（`rmcp` crate 使用）
- stdio + Streamable HTTP 両対応
- MCP ツール: `pool_ls / pool_cat / pool_grep / pool_search / pool_add / pool_lint / pool_compile_notes`
- MCP リソース: `pool://item/{id}` / `pool://library/{name}` / `m2c://item/{id}/context`
- `akari-pool mcp-serve [--http] [--port]`

**完了基準**:
- Claude Code から MCP で接続し `pool_search` を呼べる
- AKARI Video の既存 MCP サーバーと整合性が取れている

### Phase 7: AKARI Video 統合

**目的**: 消費者アプリ #1 である AKARI Video を `pool-core` 依存に切り替え、動画系アイテムの編集ビューを成立させる。

**内容**:
- AKARI Video の Pool DB 層を `pool-core` に置換
- `VideoAnalyzer` 実装（FFmpeg + Vision、M2C video）
- 動画系 library（例: `wedding-2026`）での実運用テスト

**完了基準**:
- AKARI Video が `pool-core` だけで動作
- `akari-pool add video.mp4 --analyze` で字幕・シーン情報が DB に入る

### Phase 8: 別アプリ実装

**目的**: 「同じ Pool を違う視点で見る」思想を他モダリティで実証する。

**内容**:
- **AkariNotes**: ノート系アイテムを編集する Obsidian 風ビュー
- **AkariCMS**: 記事系アイテムを編集する CMS
- **AkariSearch**: 全 library 横断検索の Web UI
- **AkariBot**: Slack/Discord で Pool 検索

**完了基準**:
- 少なくとも 2 つの別消費者アプリが同じ pool.db を参照して動く
