# Contributing to AKARI Pool Docs

AKARI Pool ドキュメントへの貢献を歓迎します。

## プロジェクト概要

AKARI Pool Docs は、AI エージェントのための汎用 Knowledge Store **AKARI Pool** の公開ドキュメント・仕様リポジトリです。

- **スコープ**: Pool の設計・仕様・ガイド・FAQ の公開ドキュメント
- **実装は別リポ**: コード本体は [`Akari-OS/pool-impl`](https://github.com/Akari-OS/pool-impl) で管理（private）
- **このリポで完結**: 単独で clone して読んでも分かる構成

## 貢献の種類

- 📖 ドキュメント改善（誤字・わかりやすさ・構成等）
- 📝 仕様の明確化提案（Issue または PR）
- 🐛 仕様の矛盾・誤情報の報告
- 🌐 翻訳（日本語 ↔ 英語）
- ❓ FAQ・ガイドの拡充提案

## 実装バグ報告

CONTRIBUTING 改善提案や実装の bug は **[`Akari-OS/pool-impl`](https://github.com/Akari-OS/pool-impl) に issue を立ててください**。本リポジトリは docs-only です。

## PR フロー

1. **小さな変更** — 直接 PR
2. **大きな提案** — 先に Issue で議論
3. commit メッセージはプレフィックス付き（例: `[ドキュメント] アーキテクチャセクションを明確化`）
4. review は 1 maintainer 以上
5. merge

## コミット規約

AKARI エコシステム共通のプレフィックス：

- `[ドキュメント]` — README / docs セクション / CHANGELOG
- `[バグ修正]` — 誤字・リンク切れ・事実誤認
- `[リファクタ]` — 構成や見出しの整理

変数・ファイル名は英語、ドキュメント本文は日本語。

## ライセンス

本リポジトリのドキュメントは **CC-BY-SA 4.0** で提供されます。

> あなたの貢献を CC-BY-SA 4.0 で提供することに同意した上での PR をお願いします。

## 関連

- [Code of Conduct](./CODE_OF_CONDUCT.md)
- [Security Policy](./SECURITY.md)
- [CHANGELOG](./CHANGELOG.md)
- [Akari-OS Vision](https://github.com/Akari-OS/.github)
- [池実装リポ](https://github.com/Akari-OS/pool-impl)
