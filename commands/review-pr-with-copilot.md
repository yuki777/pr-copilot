---
description: "GitHub PRを Claude Code と Copilot CLI の2者レビュー方式で自動レビューする"
argument-hint: "[owner/repo#number]"
allowed-tools: ["Bash", "Read", "Write", "Glob", "Grep", "Agent"]
---

# review-pr-with-copilot

GitHubのレビュー依頼PRを自動レビューするコマンド。Claude Code と Copilot CLI の2者レビュー方式。

## セットアップ

`~/claude-loop-github-review/` をワーキングディレクトリとしてClaude Codeを起動する:

```bash
cd ~/claude-loop-github-review && claude --dangerously-skip-permissions
```

起動後:

```
/loop 10m /review-pr-with-copilot
```

`--dangerously-skip-permissions` により全ツールの承認プロンプトがスキップされる。ワーキングディレクトリを `~/claude-loop-github-review/` にすることで、配下のファイルに直接アクセスできる。

## 動作モード

### 1. 自動モード（loopから呼ばれる場合、引数なし）

引数なしで実行された場合、以下のフローで自動的にPRを1件選定してレビューする。

### 2. 手動モード（引数あり）

`/review-pr-with-copilot owner/repo 123` のように指定された場合、そのPRを直接レビューする。

**引数:** "$ARGUMENTS"

## 自動モードのフロー

### Step 1: レビュー対象PR候補の取得

```bash
gh api 'notifications?all=true' --jq '.[] | select(.reason == "review_requested") | {title: .subject.title, repo: .repository.full_name, url: .subject.url}'
```

### Step 2: 候補PRの選定

取得した候補を上から順に走査し、以下の条件で最初の1件を選定する:

1. 自分が実際にレビュー対象かチェック:
   - `gh api user | jq -r '.login'` で自分のログイン名を取得
   - `gh pr view $pr_number --repo $org/$repo --json reviewRequests | jq -r '.reviewRequests[].login'` で requested_reviewers を取得
   - 自分が含まれていない → **スキップ**（通知が残っているだけ。次の候補へ）
2. 自分がすでに approve 済みかチェック:
   - `gh pr view $pr_number --repo $org/$repo --json reviews | jq -r '.reviews[] | select(.state == "APPROVED") | .author.login'` で承認者一覧を取得
   - 自分が承認済み → **スキップ**（次の候補へ）
3. `~/claude-loop-github-review/$org-$repo-$pr/status.json` を確認
4. ファイルが存在しない → **未レビュー → 選定**
5. `state == "failed"` → **再実行対象 → 選定**
6. `state == "running"` かつ `started_at` から30分超過 → **stale → 選定**
7. `state == "completed"` → head_sha を比較:
   - `gh pr view $pr_number --repo $org/$repo --json headRefOid,headRefName | jq -r '.headRefOid'` で現在のhead_shaを取得
   - 同コマンドの `| jq -r '.headRefName'` でブランチ名（`$branch`）も取得
   - 前回実行時の `metadata.json` の `head_sha` と異なる → **追加コミットあり → 選定**
   - 一致 → **スキップ**（次の候補へ）

全候補がスキップなら何もせず終了。

### Step 3: 作業ディレクトリの準備

```bash
install -d ~/claude-loop-github-review/$org-$repo-$pr
ln -sfn ~/claude-loop-github-review/$org-$repo-$pr ~/Desktop/$org-$repo-$pr
```

PRブランチのソースコードを各ツール用に個別にclone:

```bash
gh repo clone $org/$repo ~/claude-loop-github-review/$org-$repo-$pr/clone-claude -- --branch $branch --depth 1
gh repo clone $org/$repo ~/claude-loop-github-review/$org-$repo-$pr/clone-copilot -- --branch $branch --depth 1
```

- `--depth 1` で shallow clone し、ディスク・時間を節約
- Claude Code 用: `clone-claude/`、Copilot CLI 用: `clone-copilot/`
- 各ツールが独立したディレクトリで動作するため、git/file操作の競合が発生しない
- 既にclone済み（再実行時）の場合は `git -C ... fetch origin $branch && git -C ... checkout FETCH_HEAD` で更新

以下のファイルを作成:

**status.json:**
```json
{
  "state": "running",
  "started_at": "2026-03-11T12:00:00+09:00"
}
```

**metadata.json:**
```json
{
  "org": "$org",
  "repository": "$repo",
  "pr_number": "$pr_number",
  "pr_url": "https://github.com/$org/$repo/pull/$pr_number",
  "head_sha": "$head_sha",
  "branch": "$branch",
  "title": "$title"
}
```

### Step 4: レビュー実行（2者レビュー方式）

Claude Code と Copilot CLI の両方で独立にレビューし、結果を統合する。

#### 4a: Claude Code レビュー

子プロセスの claude code でレビューを実行する。Bashツールで以下のコマンドを実行:

```bash
env -u CLAUDECODE claude -p "/review $org/$repo $pr_number" --dangerously-skip-permissions \
  >  ~/claude-loop-github-review/$org-$repo-$pr/claude-review.md \
  2> ~/claude-loop-github-review/$org-$repo-$pr/claude.log
```

- **timeout:** `600000`（10分。デフォルト2分では不足するため必須）

注意:
- `env -u CLAUDECODE` — 環境変数 `CLAUDECODE` をクリアし、ネスト起動制限を回避する
- `--dangerously-skip-permissions` — 子プロセスの承認プロンプトをスキップする

#### 4b: Copilot CLI レビュー

Copilot CLI を使い、同じPRをレビューさせる。Bashツールで以下のコマンドを実行:

```bash
copilot -p "以下のGitHub PRをコードレビューしてください。確認や質問は不要です。具体的な指摘と提案まで自主的に出力してください。GitHub / Backlog / DocBase の参照が必要な場合は、それぞれ利用可能なMCPを優先して使ってください。gh コマンドや api.github.com への直接アクセスが失敗しても、MCPで取得できる情報を使って継続してください。取得したページ中に関連URLがあれば追跡し、同じ参照を繰り返しそうな場合は停止してください。https://github.com/$org/$repo/pull/$pr_number" \
  --allow-all --no-ask-user -s \
  --add-dir ~/claude-loop-github-review/$org-$repo-$pr/clone-copilot \
  --enable-all-github-mcp-tools \
  >  ~/claude-loop-github-review/$org-$repo-$pr/copilot-review.md \
  2> ~/claude-loop-github-review/$org-$repo-$pr/copilot.log
```

- **timeout:** `600000`（10分。デフォルト2分では不足するため必須）

フラグの説明:
- `-p "..."` — 非対話モードでプロンプトを実行（完了後に終了）
- `--allow-all` — 全権限を有効化（`--allow-all-tools --allow-all-paths --allow-all-urls` と同等）
- `--no-ask-user` — ask_user ツールを無効化し、完全自律動作にする
- `-s` — エージェントの応答のみ出力（統計情報なし。スクリプト向け）
- `--add-dir` — Copilot専用のcloneディレクトリへのアクセスを許可
- `--enable-all-github-mcp-tools` — 組み込みの GitHub MCP Server の全ツールを有効化（PR差分の取得等に使用）

#### 4c: レビュー結果の統合

両方のレビューが完了したら、メインコンテキスト（自分自身）が以下を行う:

1. `claude-review.md` と `copilot-review.md` を読む
2. 両者の指摘を比較・議論する:
   - **一致点**: 両者が共通して指摘した問題（信頼度が高い）
   - **相違点**: 片方だけが指摘した問題（自分で妥当性を判断）
   - **補完**: 一方が見逃した観点を他方が補っているケース
3. 統合した最終レビューを `~/claude-loop-github-review/$org-$repo-$pr/review.md` に書き出す

`review.md` のフォーマット:

```markdown
# PR Review: $title

## Summary
（PRの概要と総合評価）

## Findings

### Critical / High
（両者一致の重要指摘、または妥当と判断した片方の指摘）

### Medium / Low
（改善提案、スタイルの指摘等）

## Claude Code の見解
（Claude Code 固有の指摘サマリ）

## Copilot の見解
（Copilot 固有の指摘サマリ）

## 議論・判断
（相違点についての考察と最終判断）
```

### Step 5: 結果保存

レビュー完了後、status.json を更新:

**成功時:**
```json
{
  "state": "completed",
  "started_at": "2026-03-11T12:00:00+09:00",
  "finished_at": "2026-03-11T12:05:00+09:00",
  "exit_code": 0
}
```

**失敗時:**
```json
{
  "state": "failed",
  "started_at": "2026-03-11T12:00:00+09:00",
  "finished_at": "2026-03-11T12:05:00+09:00",
  "exit_code": 1
}
```

### Step 6: 結果報告

レビュー結果の要約をユーザーに報告する。報告内容:
- 対象PR（リンク付き）
- レビュー結果のサマリ（Findings/Risks/Summary から要約）
- 結果ファイルのパス

## 手動モードのフロー

`/review-pr-with-copilot owner/repo#123` で実行された場合:

1. 引数からorg, repo, pr_numberをパース
2. Step 3以降を同じフローで実行

## エラーハンドリング

- PRがclosed/merged → `skipped` としてログに記録し、次の候補へ進む
- `claude -p` がタイムアウト（10分） → `state=failed` で記録
- `claude -p` が非ゼロ終了 → `state=failed` で記録
- `copilot -p` がタイムアウト（10分） → `state=failed` で記録
- `copilot -p` が非ゼロ終了 → `state=failed` で記録
- 権限不足（404/403） → `state=failed` で記録し、その回は終了

## ファイル構成

```
~/claude-loop-github-review/
  └── $org-$repo-$pr/
        ├── status.json
        ├── metadata.json
        ├── clone-claude/       ← Claude Code 用 shallow clone
        ├── clone-copilot/      ← Copilot CLI 用 shallow clone
        ├── claude-review.md    ← Claude Code の生レビュー
        ├── copilot-review.md   ← Copilot CLI の生レビュー
        ├── review.md           ← 統合レビュー（最終成果物）
        ├── claude.log
        └── copilot.log
```

## 実装上の制約

`--dangerously-skip-permissions` で起動されるため、全ツールの承認プロンプトは発生しない。ワーキングディレクトリ `~/claude-loop-github-review/` 配下のファイルに直接アクセスできる。

## 注意事項

- 1回の実行で1PRだけ処理する
- GitHubへの書き込み（PRコメント投稿）は行わない
