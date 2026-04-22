# Workspace / Session 層 設計書

> **バージョン**: v0.1 (HUB-017 実装に対応)
> **最終更新**: 2026-04-22
> **対応 Crate**: `crates/pool-workspace`
> **関連 spec**: AKARI-HUB-017 `spec-memory-layer-structure`（非公開 Hub リポ管理）

---

## 1. 概要

Workspace（session-scoped）は **短命な作業領域** である。CAA（Costume Agent Architecture）の着ぐるみが 1 つのタスクに集中する間だけ存在し、完了後は Library に archive する。

### 前提: CAA との関係

- 着ぐるみ（LLM インスタンス）はステートレス。思考の継続性はファイル経由でのみ担保される
- Workspace がその「一時的な作業台」となる。`focus.md` が今の文脈、`conversation.jsonl` が対話ログ
- 着ぐるみを換える・セッションを再起動するときは Workspace を渡す（=引き継ぎ）

### Session-id 形式

```
YYYY-MM-DD-{slug}
YYYY-MM-DD-{slug}-{nnn}   # 同日同 slug の衝突時（例: -001, -002）
```

例: `2026-04-22-blog-draft`, `2026-04-22-video-edit-001`

- slug は kebab-case 英数字のみ
- 衝突処理: 同日同 slug が存在する場合は `-001` から連番サフィックスを付与
- session-id はディレクトリ名としてそのまま使われるため、ファイルシステム安全な文字のみ使用可

---

## 2. Library との関係

### 所属関係

Workspace は必ず 1 つの Library に所属する。`workspaces/{session-id}/` ディレクトリは Library ルート配下ではなく、Pool のトップレベルに置かれる。

```
~/.akari-pool/
├── libraries/
│   ├── wedding-2026/     — Library（長命）
│   └── ...
└── workspaces/           — Workspace（短命）
    ├── 2026-04-22-blog-draft/
    └── ...
```

> Library への所属は `focus.md` frontmatter の `library` フィールドで管理する。

### Archive フロー

完了した Workspace の成果物は `pool_archive` MCP ツールまたは CLI で Library に移動する:

```
workspaces/{session-id}/ (Work) → pool_archive → libraries/{name}/raw/ (Pool)
```

- Archive 後も `workspaces/{session-id}/` のメタ情報（focus.md）は残る（履歴）
- SQLite の `workspaces` テーブルで `status` を `archived` に更新し、`archived_to` にパスを記録

---

## 3. ディレクトリ構造

```
workspaces/
└── 2026-04-22-blog-draft/       — session-id がディレクトリ名
    ├── focus.md                  — Session: 現在のフォーカス・決定事項・引き継ぎ
    ├── conversation.jsonl        — Session: 対話履歴（append-only）
    ├── article-draft.md          — Work: 作成中の成果物
    ├── references.md             — Work: Pool から参照した素材へのリンク集
    └── handoff-*.md              — Work: 着ぐるみ間引き継ぎメモ（任意）
```

### ファイル種別の役割

| ファイル | 種別 | 説明 |
|---|---|---|
| `focus.md` | Session | 必須。YAML frontmatter + 4 セクション構成。着ぐるみが最初に読む |
| `conversation.jsonl` | Session | append-only。1 行 1 JSON（role / content / timestamp）|
| `*.md` / 成果物 | Work | 作業の実体。形式はタスクによって異なる |
| `handoff-*.md` | Work | 任意。長期タスクでの中間引き継ぎ |

---

## 4. focus.md スキーマ

### YAML frontmatter

```yaml
---
session-id: 2026-04-22-blog-draft
library: wedding-2026            # 所属 Library（省略可、デフォルト: default）
started: 2026-04-22T10:00:00+09:00
updated: 2026-04-22T11:30:00+09:00
status: in-progress              # in-progress | done | paused
owner: partner                   # このセッションを開始した着ぐるみ
costumes:                        # このセッションで起動した着ぐるみの一覧
  - partner
  - writer
---
```

### 4 セクション構成（Markdown 本文）

```markdown
## Current Focus
{今何をやっているか 1〜3 文。着ぐるみが最初に読む場所}

## Decisions
- {このセッションで決まったこと 1}
- {このセッションで決まったこと 2}

## Next Actions
- [ ] {次にやること 1}
- [ ] {次にやること 2}

## Notes for Next Costume
{次に着ぐるみを着る者への申し送り。引き継ぎ文脈・注意点・迷い中の事柄}
```

### frontmatter バリデーション

起動時（`workspace_create` / `workspace_read_focus`）に以下を検証する:

- `session-id` が必須
- `status` が列挙値 `in-progress | done | paused` のいずれか
- `started` が ISO 8601 フォーマット
- frontmatter が壊れている場合はエラーを返し、Git から復旧を促す

---

## 5. FocusStatus 状態遷移

```
          workspace_create
               ↓
         [in-progress]
          /          \
    session_update_focus  session_update_focus
     (status: done)    (status: paused)
          ↓                   ↓
        [done]            [paused]
          ↓                   ↓
      pool_archive     (再開: in-progress に戻す)
          ↓
      archived
```

| 状態 | 説明 |
|---|---|
| `in-progress` | 作業中。着ぐるみが読み書き可能 |
| `done` | 完了。成果物を pool_archive で Library に移せる状態 |
| `paused` | 一時停止。再開可能。別の Workspace を優先する場合に使用 |
| `archived` | SQLite のみの論理状態。ディレクトリは残る |

---

## 6. Atomic I/O 契約

記憶層の書き込みはすべて atomic でなければならない。

### 上書き書き込み（Work ファイル）

```rust
// tmp ファイルに書いて rename（POSIX rename は atomic）
fn atomic_write(path: &Path, content: &[u8]) -> Result<()> {
    let tmp = path.with_extension("tmp");
    fs::write(&tmp, content)?;
    fs::rename(&tmp, path)?;
    Ok(())
}
```

### 追記（conversation.jsonl）

`O_APPEND` フラグで open することで、複数プロセスからの追記が race-free になる:

```rust
fn atomic_append(path: &Path, line: &[u8]) -> Result<()> {
    use std::fs::OpenOptions;
    use std::io::Write;
    let mut f = OpenOptions::new().create(true).append(true).open(path)?;
    f.write_all(line)?;
    f.write_all(b"\n")?;
    Ok(())
}
```

### 並行書き込みの扱い

複数の着ぐるみが同一 Workspace に同時書き込みする場合:

- `focus.md` の更新は `session_update_focus` を通じて行い、内部で atomic write
- `conversation.jsonl` は append-only なので O_APPEND で安全
- Work ファイルの競合は現バージョンでは未対処（オープンクエスチョン参照）

---

## 7. MCP ツール一覧

HUB-017 で追加された MCP ツール（既存の `pool_ls` / `pool_search` 等と合わせて合計 21 ツール）:

| ツール | 説明 |
|---|---|
| `workspace_create` | 新しい Workspace を作成し、focus.md を初期化する |
| `workspace_list` | Workspace 一覧を返す（status フィルタ対応） |
| `workspace_read_focus` | `focus.md` を読んで FocusMd 構造体で返す |
| `session_append` | `conversation.jsonl` に 1 行追記する（atomic append） |
| `session_write_file` | Work ファイルを atomic 書き込みする |
| `session_read_file` | Work ファイルを読む |
| `session_update_focus` | `focus.md` の特定セクション / frontmatter フィールドを更新する |
| `pool_archive` | Workspace の成果物を Library の raw 層に archive する |
| `pool_read` | Pool サブレイヤー（raw / items / categories）からファイルを読む |
| `pool_list` | raw / items / categories の一覧を返す |

### ツール命名の方針

- `workspace_*`: Workspace ライフサイクル管理（作成・一覧・フォーカス読み取り）
- `session_*`: セッション内の I/O（ファイル読み書き・対話履歴・フォーカス更新）
- `pool_*`: Pool サブレイヤー操作（既存の `pool_ls` 等と同じプレフィックス）

---

## 8. CLI コマンド

```bash
# Workspace を作成
akari-pool workspace create 2026-04-22-blog-draft --library wedding-2026

# Workspace 一覧
akari-pool workspace list
akari-pool workspace list --status in-progress

# Workspace の詳細表示（focus.md の内容）
akari-pool workspace show 2026-04-22-blog-draft
```

### workspace create

```
akari-pool workspace create <session-id> [--library <name>]
```

- `<session-id>`: `YYYY-MM-DD-{slug}` 形式。形式が不正な場合はエラー
- `--library`: 所属 Library。省略時は current library（または `default`）
- 衝突時: `-001` 等の連番サフィックスを自動付与して成功させる（または `--no-auto-suffix` でエラーにする）
- 実行後: `workspaces/{session-id}/focus.md` が初期化された状態で作成される

### workspace list

```
akari-pool workspace list [--status <status>] [--library <name>]
```

- `--status`: `in-progress` / `done` / `paused` でフィルタ
- `--library`: 特定 Library に所属する Workspace のみ表示

### workspace show

```
akari-pool workspace show <session-id>
```

- `focus.md` の frontmatter と本文を整形表示する

---

## 9. オープンクエスチョン

以下は現時点で未解決の設計上の問いである（実装が進んだ段階で判断する）:

- **-nnn 衝突処理の UI**: 自動連番 vs. エラー → どちらを使うべきか。現状は自動連番を採用
- **並行セッション**: 同一 Library 内で複数の `in-progress` Workspace を持てるか。現状は制限しない
- **完了判定**: `status: done` への遷移は着ぐるみが自己申告するか、人間が明示するか。現状は `session_update_focus` を通じて着ぐるみが更新
- **Git LFS**: `raw/` 内の大容量ファイル（動画等）は LFS に移す閾値をどこに設定するか
- **archive 再開ワークフロー**: archive 済みの Workspace を掘り返して再開するフローが未定義

---

## 10. 参考

- 関連 spec: AKARI-HUB-017 `spec-memory-layer-structure`（非公開 Hub リポ管理）
- Library 層の設計書（長命な知識隔離ユニット）: [`./library-layer.md`](./library-layer.md)
- 全体アーキテクチャ: [`./architecture.md`](./architecture.md)
