# 修正指示: GitHub書き込み権限の制限

## 背景

SKILL.md の注意事項に「GitHubへの書き込み（PRコメント投稿）は行わない」と記載しているが、実装上はプロンプトの指示に頼っているだけで、権限レベルでの制限がない。

## 対象ファイル

`skills/pr-copilot/SKILL.md`

## 変更内容

### 4a: Claude Code レビュー（113行目付近）

**現状:**

```
--allowedTools "Read Glob Grep Bash(gh:*,git:*)"
```

**問題:** `Bash(gh:*)` は `gh pr review`, `gh pr comment`, `gh api --method POST` 等の書き込み操作も許可してしまう。

**変更後:**

```
--allowedTools "Read Glob Grep Bash(git diff:*) Bash(git show:*) Bash(git log:*) Bash(git rev-parse:*) Bash(gh pr view:*) Bash(gh pr diff:*)"
```

**説明文（124行目付近）も変更:**

現状:

```
- `--allowedTools` — Read/Glob/Grep + gh/git のみに制限（最小権限）
```

変更後:

```
- `--allowedTools` — レビューに必要な read-only コマンドのみ許可（gh pr view/diff, git diff/show/log）
```

### 4b: Copilot CLI レビュー（137行目付近）

**現状:**

```
--enable-all-github-mcp-tools
```

**問題:** GitHub MCP の書き込み系ツール（PR comment, review submit 等）も有効になる。

**変更後:** この行を削除する。

**説明文（152行目付近）も変更:**

現状:

```
- `--enable-all-github-mcp-tools` — 組み込みの GitHub MCP Server の全ツールを有効化（PR差分の取得等に使用）
```

変更後: この行を削除する。
