# code-review

Outside-in, risk-driven code review skill based on Gregory Brown's *Effective Code Reviews* (Programming Beyond Practices, 2015).

## What it does

Reviews pull requests in six phases, with a re-review pass when the PR was reviewed before:

0. **Prior Review State** *(re-reviews)* — reads all existing threads, verifies/acknowledges its own prior findings in-thread, never duplicates a concern already raised, and keeps a single summary comment per PR
1. **Problem Validation** — does the code solve the right problem?
2. **Build & Runability** — is the build passing, is it running somewhere real?
3. **Test Audit** — are modified/deleted tests a red flag?
4. **Coverage Assessment** — categorizes the change (bug fix, new feature, add-on, extension, refinement) and applies risk-appropriate expectations
5. **Project-Specific Checks** — deferred to your project's rules file
6. **Risk Assessment & Output** — Low / Medium / High risk level, review summary, human reviewer checklist. Every finding carries a severity and a confidence; filtering happens in the summary, never during the review itself

## Companion rules file (recommended)

The SKILL.md contains the generic methodology. Phase 5 ("Project-Specific Checks") and Phase 6 output mechanics intentionally defer to your project. You should create `.claude/rules/code-review.md` in your project repo with:

### How code changes are fetched

```markdown
## Context Gathering

1. Get PR details: `gh pr view <PR_NUMBER>`
2. Find the linked issue:
   - From branch name: pattern `^\d+` (e.g. `733-fix-login` → issue #733)
   - From PR body: look for `Fixes #N`, `Closes #N`, `Resolves #N`
3. If found: `gh issue view <ISSUE_NUMBER>`
4. If not found: use PR title/description as source of truth
5. Get the diff: `gh pr diff <PR_NUMBER>`
```

### Project-specific checks (Phase 5)

```markdown
## Project-Specific Checks

Run checks relevant to the changes. Skip sections for unaffected areas.

### Architecture
- [Your architecture rules, e.g. bounded context boundaries, layer separation]

### Performance
- [Your performance checks, e.g. N+1 queries, missing indexes]

### Security
- [Your security checks, e.g. auth changes, raw SQL, sensitive data in logs]

### Testing
- [Your testing conventions, e.g. framework, required coverage for change types]
```

### How reviews are posted (Phase 6)

```markdown
## Output Format

Four sections are always present — Verdict, Concerns, Risk and For a human. The rest appear
only where they say something those four do not, so a small change gets a short review.

**Verdict** — one or two sentences: can this merge, and what stands in the way.

**Previous Findings** _(re-reviews only)_ — one line each: Resolved / Answered / Still open, plus a few words.

**Concerns** — blockers first. Each finding is claim, evidence, consequence:

`[blocker | important | nice-to-have]` `[confidence: high | medium | low]` **What is wrong.**
The file and line, or the quoted output, that shows it.
What breaks, who hits it, or what it costs to leave.

(If none: "No issues found")

Inline findings use the same three parts, after the marker:

\`\`\`
<!-- code-review-finding -->
`[important]` `[confidence: high]` **What is wrong.**
The evidence.
The consequence.
\`\`\`

**Risk** — the level, always. Reasoning only where the concerns don't already show it.

**For a human** — one standing line saying the review did not run the code, then one line per
thing it could not verify. Nothing that already appears above as a concern.

Add only where they carry something the required sections do not:

**Context** — when the change's purpose is not evident from its title and diff.
**What was checked** — only checks whose result would change the merge decision, one line each.


## Posting Mechanics

### Part 0: Prior review state (re-reviews)

\`\`\`bash
# Inline review threads with resolution state; databaseId joins GraphQL to REST ids
gh api graphql -f query='
  query($owner:String!, $repo:String!, $pr:Int!) {
    repository(owner:$owner, name:$repo) {
      pullRequest(number:$pr) {
        reviewThreads(first:100) {
          nodes { id isResolved
            comments(first:50) { nodes { databaseId author { login } body path line } } } } } } }' \
  -f owner=${OWNER} -f repo=${REPO} -F pr=${PR_NUMBER}

# Own previous summary: the issue comment starting with the marker
gh api repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments --paginate \
  --jq '.[] | select(.body | startswith("<!-- code-review-summary -->")) | .id'
\`\`\`

Own inline threads = threads whose root comment body starts with `<!-- code-review-finding -->`. For threads that predate the marker, fall back to matching `author.login` against the account your previous automated reviews were posted from. Note GitHub's GraphQL API reports a bot's login without the `[bot]` suffix that REST uses (e.g. a REST login of `claude[bot]` appears as `claude` via GraphQL) — compare against whichever form the query you're using actually returns, don't hardcode one.

### Part 1: Thread replies and resolution (re-reviews)

\`\`\`bash
# Reply in an existing thread (COMMENT_ID = databaseId of the thread's root comment, from Part 0)
gh api repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/comments/${COMMENT_ID}/replies \
  --method POST --field body="Verified fixed in ${SHA}."

# Resolve a thread (THREAD_ID = GraphQL thread id from Part 0)
gh api graphql -f query='
  mutation($thread:ID!) {
    resolveReviewThread(input:{threadId:$thread}) { thread { isResolved } } }' \
  -f thread=${THREAD_ID}
\`\`\`

### Part 2: Inline comments (new findings only)

\`\`\`bash
COMMIT_SHA=$(gh pr view <PR_NUMBER> --json headRefOid --jq '.headRefOid')

# Verify the line appears in the diff before posting
gh pr diff <PR_NUMBER>

jq -n \
  --arg commit_id "$COMMIT_SHA" \
  --arg path "relative/path/to/file.rb" \
  --arg body "<!-- code-review-finding -->
\`[important]\` \`[confidence: high]\` **What is wrong.**
The evidence. The consequence." \
  --argjson line 42 \
  '{commit_id: $commit_id, event: "COMMENT", body: "", comments: [{path: $path, line: $line, side: "RIGHT", body: $body}]}' \
| gh api repos/${REPO}/pulls/${PR_NUMBER}/reviews --method POST --input -
\`\`\`

### Part 3: Summary — single issue comment per PR

\`\`\`bash
# Delete every previous summary found in Part 0 (normally just one; a race could leave two)
gh api repos/${OWNER}/${REPO}/issues/comments/${COMMENT_ID} --method DELETE

# Post the fresh summary as an ISSUE comment — submitted PR reviews cannot be deleted via the API,
# which is why the summary lives here instead of in a `gh pr review` comment.
gh api repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments --method POST \
  --field body="$(cat <<'EOF'
<!-- code-review-summary -->
...output format above...
EOF
)"
\`\`\`

**Rules:**
- Never post a new inline comment for a finding that already has a thread — reply in the thread instead (Part 1)
- Only reply to / resolve threads carrying the `code-review-finding` marker (or matched by the author-login fallback)
- `event`: Always `"COMMENT"` — never approve or request changes
- `line`: Must be present in the diff or the API rejects it
- `side`: Always `"RIGHT"`
- `path`: Relative to repo root, no leading `/`
- **Fallback:** If inline comment fails, include the concern in the summary instead
```

## Usage

### Claude Code (terminal)

After installing via `/plugin install code-review@agilefreaks-skills`:

```
Use the code-review skill to review PR #42
```

```
Use the code-review skill to re-review PR #42
```

### Claude.ai Cowork

Once the plugin is distributed to your org, use it from any Cowork project. See your org's Cowork plugin settings for availability.

### GitHub Actions

Run `set up code-review` in Claude Code or Claude.ai Cowork to generate `.github/workflows/code-review.yml` in your project automatically. The Setup wizard asks which model to use and writes the workflow for you.

Opus is the default and the right choice for almost every repo. Fable is available for high-stakes reviews at roughly double the per-token cost; note that its safety classifiers can decline a security-heavy diff, which shows up as a failed CI run rather than a posted review. The workflow passes a model *alias* rather than a dated model id, so it follows the current release instead of pinning one that will be retired.

The most capable models write longer prose by default, and effort does not reliably shorten it — so length is set by the skill, not the model. Reviews are **Brief** out of the box: two required sections, and each finding written as claim, evidence, consequence. If your team would rather have the reasoning inline than ask for it in the thread, set **Review Detail** to *Standard* during Setup. Neither setting changes which findings are reported; only how much prose each one gets.

After the file is generated, add `CLAUDE_CODE_OAUTH_TOKEN` as a repository secret:

1. Generate the token: `claude setup-token`
2. Add it at: **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**

Alternatively, copy the template from `skills/code-review/assets/code-review.yml` in this repo and add the secret manually.
