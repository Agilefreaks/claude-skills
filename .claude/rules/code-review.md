# Code Review Rules

## Context Gathering

Look for a linked GitHub issue in the PR body or PR title. If a GitHub issue is found, read it with `gh issue view` before reviewing the diff. If no issue exists, use the PR title and description as the source of truth.

## Build Verification

This repo has no build system (pure Markdown and JSON). Skip build verification and note in the human checklist that functional validation must be done manually.

## Posting Mechanics

Post the review in two parts:

**1. Inline comments** — for any finding that is specific to a line in the diff, post it as an inline PR review comment using the GitHub API:

```
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews \
  --method POST \
  --field commit_id="$(gh pr view {pr_number} --json headRefOid --jq .headRefOid)" \
  --field event="COMMENT" \
  --field body="" \
  --field "comments[][path]"="path/to/file" \
  --field "comments[][line]"={line_number} \
  --field "comments[][body]"="Finding description"
```

Use a single review API call with all inline comments grouped together. Only use line-level comments when the finding is genuinely tied to a specific line in the diff.

**2. Summary comment** — post the overall review (context, what looks good, risk level, human checklist) as a PR review body:

```
gh pr review {pr_number} --comment --body "{review summary}"
```

If there are no line-specific findings, skip step 1 and post only the summary comment.
