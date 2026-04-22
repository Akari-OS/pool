# AkariPool

> **Universal Knowledge Store for AI Agents.**
> AI エージェントのための、モダリティ中立な汎用ナレッジベース。

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Status](https://img.shields.io/badge/status-early--development-orange.svg)](#status)
[![Part of Akari-OS](https://img.shields.io/badge/part_of-Akari--OS-f97316.svg)](https://github.com/Akari-OS)

---

## 🌟 これは何？

**AkariPool は「AI エージェントが本棚を散歩するように知識を探索できる」ことを目指した、ローカルファーストの汎用 Knowledge Store です。**

動画・音声・画像・PDF・記事・コード・データセット — **どんなモダリティのデータでも**同じ抽象で扱える。既存の RAG システムが「テキストチャンク」に縛られるのに対し、AkariPool は **Personal Knowledge Base** の思想でエージェントに知識の体系を提供します。

```
📹 Video  ─┐
🎵 Audio  ─┤
🖼 Image  ─┤   ┌─────────────┐   ┌──────────┐
📄 PDF    ─┼──▶│  AkariPool  │──▶│ AI Agent │
📝 Notes  ─┤   │  (一元管理)   │   │  (探索)  │
💾 Data   ─┤   └─────────────┘   └──────────┘
💻 Code   ─┘
```

---

## 🎯 なぜ AkariPool か

### RAG の限界を超える

既存の RAG ツール（LlamaIndex / LangChain 等）は **テキスト中心**で、動画・画像・音声を扱うには追加パイプラインが必要。AkariPool は最初から**全モダリティ対応**。

### ローカルファースト

クラウド依存ではなく、**全データはあなたのマシンに留まる**。AGPL-3.0 でソース公開を担保。将来の Cloud sync は opt-in で別レイヤー。

### エージェントが育てる知識庫

AI との会話で発見された関係性を自動で蓄積する **Filed back ループ** で、使えば使うほど賢くなる本棚を実現。

---

## 🏗 アーキテクチャ

### 3 つの層

```
┌─────────────────────────────────────────┐
│ Layer 3: MCP Server / SDK                │
│ Claude Code, Cursor, カスタム AI と接続     │
├─────────────────────────────────────────┤
│ Layer 2: Wiki (Markdown)                 │
│ AI が整形した人間可読なノート                │
├─────────────────────────────────────────┤
│ Layer 1: Raw Files                       │
│ オリジナルの動画・PDF・画像をそのまま保存      │
├─────────────────────────────────────────┤
│ Layer 0: SQLite + FTS5 + Analyzers       │
│ メタデータ・全文検索・モダリティ別解析         │
└─────────────────────────────────────────┘
```

同じ情報を「**機械可読（SQLite）**」「**人間可読（Wiki）**」「**生データ（raw）**」の 3 形式で同時に保持。

### 6 つの設計思想

1. **モダリティ中立** — Video/Audio/Image/PDF/Code を同じ抽象で扱う。新モダリティは trait 実装で追加可能
2. **Raw + Wiki の二段構え** — 生データは永久保存、Wiki 層で AI 整形・圧縮
3. **Workspace 隔離** — プロジェクトごとに分離、明示的に横断可能
4. **Linting（自己治癒）** — 表記揺れ検出・関係抜け補完・未分類アラート
5. **Filed back ループ** — AI との対話で発見された関係性を自動追記
6. **ローカルファースト + AGPL** — 全データはあなたのマシンに留まる

詳細は [docs/design/architecture.md](./docs/design/architecture.md) を参照。

---

## 🔌 3 つの接続面

AkariPool は 3 つのインターフェースを提供します：

### 1. Rust Crate（他の Rust アプリから直接依存）

```rust
use akari_pool::{Pool, Workspace, Item};

let pool = Pool::open("~/.akari-pool")?;
let ws = pool.workspace("wedding-2026")?;
let items = ws.search("結婚式")?;
```

### 2. CLI (`akari-pool`)

```bash
akari-pool workspace create wedding-2026
akari-pool add ./video.mp4 --workspace wedding-2026
akari-pool ls --workspace wedding-2026
akari-pool grep "BGM" --workspace wedding-2026
akari-pool search "明るい" --cross   # 全 workspace 横断
```

### 3. MCP Server（Claude Code / Cursor / 他 AI と接続）

```bash
akari-pool mcp-serve --workspace wedding-2026
akari-pool mcp-serve --http --port 7890
```

**提供ツール**: `pool_ls`, `pool_cat`, `pool_grep`, `pool_search`, `pool_add`, `pool_lint`, `pool_compile_notes`, ...
**提供リソース**: `pool://item/{id}`, `pool://workspace/{name}`, `m2c://item/{id}/context`

---

## 📚 ドキュメント

- 📖 [**アーキテクチャ**](./docs/design/architecture.md) — Pool の内部構造
- 📝 [**要件定義**](./docs/design/requirements.md) — FR / NFR / AC
- 🗺 [**Workspace 設計**](./docs/design/workspace-layer.md) — 隔離モデル
- 🧩 [**Analyzer Plugin**](./docs/design/analyzer-plugin.md) — モダリティ拡張方法
- 🧠 [**LLM 戦略**](./docs/design/llm-strategy.md) — LLM の使い方
- 🗓 [**ロードマップ**](./docs/planning/roadmap.md) — Phase 別計画
- 🔗 [**Obsidian 連携**](./docs/integration/obsidian-plugin.md) — Obsidian との統合
- 🎬 [**AKARI Video 統合**](./docs/design/akari-video-integration.md) — Video アプリとの連携

---

## 🌐 Akari-OS エコシステムでの位置

AkariPool は [**Akari-OS**](https://github.com/Akari-OS) エコシステムの **Core 層** — 中核データ基盤です。

```
📱 Apps (Video / CMS / Notes / ...)
        ↓ 依存
🧬 Core: Pool  ⟷  M2C  ⟷  AMP
```

- **[M2C](https://github.com/Akari-OS/m2c)** — Media-to-Context プロトコル（Analyzer の出力フォーマット）
- **[AMP](https://github.com/Akari-OS/amp)** — Agent Memory Protocol（将来統合）
- **消費者アプリ**: AKARI Video, AKARI CMS（開発中）, AkariNotes（構想）

詳細は [Akari-OS の VISION](https://github.com/Akari-OS/.github/blob/main/VISION.md) を参照。

---

## 🚧 Status

**Early development — Phase 4.5**

現在実装中：
- ✅ SQLite + FTS5 基盤
- ✅ Workspace 隔離 + 横断検索
- ✅ ArticleAnalyzer (MVP)
- ✅ Wiki compile + Relations CRUD + Linter (2/4 種)
- 🟡 **次**: Inconsistency / ConnectionGap Linter + Relation CLI
- ⬜ Filed back ループ
- ⬜ MCP server 化
- ⬜ AKARI Video 統合

詳細は [docs/planning/roadmap.md](./docs/planning/roadmap.md) を参照。

> **注**: 本リポジトリは現時点ではドキュメントのみ公開しています。
> ソースコード本体は別途公開予定です。

---

## 🤝 Contributing

Akari-OS エコシステムへの貢献は大歓迎です。

- 🐛 **バグ報告**: Issue を立ててください
- 💡 **機能提案**: Discussions で議論歓迎
- 📖 **ドキュメント改善**: typo 修正でも PR 歓迎
- 🧩 **Analyzer プラグイン**: 新モダリティのサポート貢献

共通の貢献ガイド・行動規範は [Akari-OS organization](https://github.com/Akari-OS) を参照。

---

## 📜 License

**[AGPL-3.0](./LICENSE)** — GNU Affero General Public License v3.0

AkariPool はローカルファーストかつ自由なソフトウェアとして設計されています。
派生物を SaaS 化する場合も、ソースコードの公開が求められます。

---

## 🔗 リンク

- 🌐 **Akari-OS エコシステム**: https://github.com/Akari-OS
- 📖 **公開版ビジョン**: https://github.com/Akari-OS/.github
- ☕ **プロジェクト応援**: [Buy Me a Coffee](https://buymeacoffee.com/kyacharsx)

---

*Built with ❤ by the [Akari-OS](https://github.com/Akari-OS) community.
個人のための AI OS を、日本から。*
