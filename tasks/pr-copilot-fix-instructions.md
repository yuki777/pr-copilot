# pr-copilot スキル修正指示書

## 問題

PR #419 (Shueisha.Oto) が pr-copilot で検出されなかった。

### 原因

Step 1 で使用している GitHub Notifications API (`gh api 'notifications?all=true'`) は、リポジトリを **Watch していない** 場合に `review_requested` 通知を生成しない。Shueisha.Oto は Watch 設定がなかったため、レビュー依頼通知が存在せず候補リストに含まれなかった。

### 影響範囲

Watch していない全リポジトリのレビュー依頼が検出漏れする。実測では Notifications API で 14件だったのに対し、Search API では 42件が検出された。

---

## 修正内容

### 変更対象ファイル

`skills/review/SKILL.md` の **Step 1** セクションのみ。Step 2 以降は変更不要。

### Before（現在の Step 1）

```bash
gh api 'notifications?all=true' --jq '.[] | select(.reason == "review_requested") | {title: .subject.title, repo: .repository.full_name, url: .subject.url}'
```

### After（修正後の Step 1）

```bash
gh api -H "Accept: application/vnd.github+json" \
  '/search/issues?q=is:pr+state:open+org:shueisha-arts-and-digital+review-requested:yuki777&sort=updated&order=desc&per_page=100' \
  --jq '.items[] | {
    org: (.repository_url | split("/")[-2]),
    repository: (.repository_url | split("/")[-1]),
    pr_number: .number,
    title,
    pr_url: (.pull_request.html_url // .html_url)
  }'
```

### Step 1 セクションの説明文も以下に差し替え

```markdown
### Step 1: レビュー対象PR候補の取得

GitHub Search API でレビュー依頼されている Open PR を取得する。
Notifications API と異なり、リポジトリの Watch 設定に依存しない。

\`\`\`bash
gh api -H "Accept: application/vnd.github+json" \
  '/search/issues?q=is:pr+state:open+org:shueisha-arts-and-digital+review-requested:yuki777&sort=updated&order=desc&per_page=100' \
  --jq '.items[] | {
    org: (.repository_url | split("/")[-2]),
    repository: (.repository_url | split("/")[-1]),
    pr_number: .number,
    title,
    pr_url: (.pull_request.html_url // .html_url)
  }'
\`\`\`

**クエリパラメータの説明:**
- `is:pr` — PR のみ（Issue を除外）
- `state:open` — Open な PR のみ
- `org:shueisha-arts-and-digital` — 組織内リポジトリに限定（ノイズ防止）
- `review-requested:yuki777` — 自分がレビュー依頼されている PR（チームレビュー依頼も含む）
- `sort=updated&order=desc` — 更新日時の降順（最新を優先。best match のデフォルトだと古い PR が漏れる）
- `per_page=100` — 最大 100 件取得

**注意事項:**
- Search API はインデックスベースのため、レビュー依頼から数分の遅延が発生し得る（10分ポーリングなら許容範囲）
- レスポンスに `head_sha` と `branch` は含まれない（Step 2 の `gh pr view` で取得する既存ロジックで対応済み）
- `review-requested:USERNAME` は GitHub docs 上、ユーザー直接指定とチーム経由の両方を含むと明記されている
```

---

## Step 2 の出力形式変更に伴う調整

Step 1 の出力フォーマットが変わるため、Step 2 の候補走査で参照するフィールド名を合わせる。

| Notifications API（旧） | Search API（新） |
|---|---|
| `repo` (例: `org/repo`) | `org` + `repository` (分離済み) |
| `url` (API URL から PR 番号を抽出) | `pr_number` (直接取得可能) |
| `title` | `title` (変更なし) |

Step 2 の「上から順に走査」のロジックは、Search API が `sort=updated&order=desc` で返すため、**更新日時の新しい順** になる（Notifications API と同じく上から処理すればよい）。

---

## 検証方法

修正後に以下を確認:

1. `Shueisha.Oto` PR #419 が候補リストに含まれること
2. 既存のレビュー済み PR（#3484, #2046 等）が引き続き検出され、status.json / head_sha による重複排除が正常に機能すること
3. Search API のレート制限（30 req/min）に対して、10分間隔のポーリングが問題ないこと（1回あたり 0.1 req/min なので十分余裕あり）

---

## 検証済み事項（Codex レビュー結果）

この修正方針は Codex CLI（GPT-5.4）と GitHub 公式 docs を元に検証済み:

- **レート制限**: 問題なし（30 req/min に対し 0.1 req/min）
- **チームレビュー依頼**: `review-requested:USERNAME` でチーム経由も検出される
- **情報の網羅性**: `head_sha` / `branch` は Search API に含まれないが、Step 2 の `gh pr view` で取得する既存ロジックでカバー済み
- **インデックス遅延**: 数分の遅延はあるが、10分ポーリングなら許容範囲
- **incomplete_results**: Search API が `incomplete_results: true` を返す場合は不完全な結果。現在の実装では無視しても大きな問題にはならないが、将来的にチェックを追加してもよい
