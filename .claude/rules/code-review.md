# Code Review Rules

## Context Gathering

Look for a linked GitHub issue in the PR body or PR title. If a GitHub issue is found, read it with `gh issue view` before reviewing the diff. If no issue exists, use the PR title and description as the source of truth.

## Build Verification

This repo has no build system or test runner (pure Markdown and JSON), but it does have one automated gate: `.github/workflows/validate.yml` runs `claude plugin validate` on the marketplace manifest and every plugin directory, then checks that each plugin's version agrees across `plugin.json`, its `marketplace.json` entry, and its README Plugin Catalogue row.

Check that workflow's status on the PR and report it. A red run is a blocker; treat it as Phase 2's broken build and pause the review. A green run says the manifests parse and the versions agree — it says nothing about whether the skill content is correct, so still note in the human checklist that functional validation of skill behaviour must be done manually.

## Posting Mechanics

**0. Prior review state (re-reviews)** — before reviewing, fetch existing threads and your own previous summary:

```
gh api graphql -f query='
  query($owner:String!, $repo:String!, $pr:Int!) {
    repository(owner:$owner, name:$repo) {
      pullRequest(number:$pr) {
        reviewThreads(first:100) {
          nodes { id isResolved
            comments(first:50) { nodes { databaseId author { login } body path line } } } } } } }' \
  -f owner={owner} -f repo={repo} -F pr={pr_number}

gh api repos/{owner}/{repo}/issues/{pr_number}/comments --paginate \
  --jq '.[] | select(.body | startswith("<!-- code-review-summary -->")) | .id'
```

Own inline threads = root comment body starts with `<!-- code-review-finding -->`. For threads predating the marker, fall back to matching `author.login` against the account previous automated reviews were posted from — note GraphQL reports the bot's login without the `[bot]` suffix REST uses (`claude` vs `claude[bot]`). Never raise a concern already raised in a human reviewer's thread, answered or not.

**1. Thread replies and resolution (re-reviews)** — for each of your own prior findings: fix verified in the diff or author response accepted → reply in the thread, then resolve it; still open with no response → leave the thread untouched, don't re-post:

```
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies \
  --method POST --field body="Verified fixed in {sha}."

gh api graphql -f query='
  mutation($thread:ID!) {
    resolveReviewThread(input:{threadId:$thread}) { thread { isResolved } } }' \
  -f thread={thread_id}
```

**2. Inline comments** — for genuinely new findings specific to a line in the diff, post an inline PR review comment using the GitHub API, with the body prefixed by the finding marker:

```
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --method POST \
  --field commit_id="$(gh pr view {pr_number} --json headRefOid --jq .headRefOid)" \
  --field event="COMMENT" \
  --field body="" \
  --field "comments[][path]"="path/to/file" \
  --field "comments[][line]"={line_number} \
  --field "comments[][body]"="<!-- code-review-finding -->
Finding description"
```

Use a single review API call with all inline comments grouped together. Only use line-level comments when the finding is genuinely tied to a specific line in the diff. Never post a new inline comment for a finding that already has a thread — reply in it instead (step 1).

**3. Summary comment** — maintain exactly one summary per PR, as an issue comment (not a PR review — reviews can't be deleted via the API):

```
# Delete every previous summary found in step 0
gh api repos/{owner}/{repo}/issues/comments/{comment_id} --method DELETE

gh api repos/{owner}/{repo}/issues/{pr_number}/comments --method POST \
  --field body="<!-- code-review-summary -->
{review summary}"
```

On a re-review, the summary includes a "Previous findings" section (resolved / answered / still open) ahead of new concerns. If there are no line-specific findings, skip step 2 and post only the summary.

**Migration note:** summaries posted before this rule existed were PR reviews (via `gh pr review --comment`) and cannot be deleted through the API — leave them; a human may minimize them manually.
