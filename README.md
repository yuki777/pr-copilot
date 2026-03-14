# pr-copilot

GitHub PRを **Claude Code** と **Copilot CLI** の2者レビュー方式で自動レビューする [Claude Code](https://docs.anthropic.com/en/docs/claude-code) プラグイン。

## 特徴

- **2者レビュー**: Claude Code と Copilot CLI が独立してPRをレビューし、結果を統合
- **自動巡回**: `/loop` と組み合わせて定期的にレビュー依頼PRを検出・処理
- **冪等実行**: status.json による状態管理で、完了済みPRのスキップや失敗時の再実行に対応
- **最小権限**: 各レビューツールは読み取り専用で動作し、PRへのコメント投稿は行わない

## 必要なもの

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Copilot CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli)
- [GitHub CLI (`gh`)](https://cli.github.com/)

## インストール

Claude Code セッション内でマーケットプレイスを追加し、プラグインをインストール:

```
/plugin marketplace add yuki777/pr-copilot
/plugin install pr-copilot@yuki777/pr-copilot
```

## 使い方

ワーキングディレクトリを作成し、Claude Code を起動:

```bash
cd ~/claude-loop-pr-copilot && claude
```

10分間隔で自動レビューを開始:

```
/loop 10m /pr-copilot
```

## レビューフロー

1. **PR候補の取得** — GitHub通知から `review_requested` のPRを一覧取得
2. **候補の選定** — 未レビュー・失敗・追加コミットありの最初の1件を選定
3. **作業ディレクトリの準備** — PRブランチを各ツール用に個別に shallow clone
4. **2者レビュー実行** — Claude Code と Copilot CLI が並行してレビュー
5. **結果の統合** — 両者の指摘を比較・議論し、統合レビューを作成
6. **結果報告** — レビュー結果の要約をユーザーに報告

## ファイル構成

```
~/claude-loop-pr-copilot/
  └── $org-$repo-$pr/
        ├── status.json           # 実行状態（running / completed / failed）
        ├── metadata.json         # PR情報（org, repo, pr_number, head_sha 等）
        ├── clone-claude/         # Claude Code 用 shallow clone
        ├── clone-copilot/        # Copilot CLI 用 shallow clone
        ├── claude-review.md      # Claude Code の生レビュー
        ├── copilot-review.md     # Copilot CLI の生レビュー
        ├── review.md             # 統合レビュー（最終成果物）
        ├── claude.log
        └── copilot.log
```

## ライセンス

MIT
