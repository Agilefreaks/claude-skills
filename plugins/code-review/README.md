# code-review

Outside-in, risk-driven code review skill based on Gregory Brown's *Effective Code Reviews* (Programming Beyond Practices, 2015).

## What it does

Reviews pull requests in six phases:

1. **Problem Validation** — does the code solve the right problem?
2. **Build & Runability** — is the build passing, is it running somewhere real?
3. **Test Audit** — are modified/deleted tests a red flag?
4. **Coverage Assessment** — categorizes the change (bug fix, new feature, add-on, extension, refinement) and applies risk-appropriate expectations
5. **Project-Specific Checks** — deferred to your project's rules file
6. **Risk Assessment & Output** — Low / Medium / High risk level, review summary, human reviewer checklist

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

### > Automated Review

**Context**
- **PR:** #<number> - <title>
- **Issue:** #<number> - <title> (or "No linked issue")
- **Change Type:** [Bug Fix / New Feature / Enhancement / Refactoring / Config/Docs]
- **Scope:** [Brief description]

**Looks Good**
- [Specific passing checks]

**Concerns Found**

_See inline comments in the Files Changed tab for specific line-level concerns._

_General concerns:_
- [Concern and why it matters]

(If none: "No issues found")

**Risk Assessment**
- **Risk Level:** [Low / Medium / High]
- **Reasoning:** [Why]

---

[Human Reviewer Checklist — generated per SKILL.md Phase 6]

## Posting Mechanics

### Part 1: Summary

\`\`\`bash
gh pr review <PR_NUMBER> --comment --body "$(cat <<'EOF'
...output format above...
EOF
)"
\`\`\`

### Part 2: Inline Comments

\`\`\`bash
COMMIT_SHA=$(gh pr view <PR_NUMBER> --json headRefOid --jq '.headRefOid')

# Verify the line appears in the diff before posting
gh pr diff <PR_NUMBER>

jq -n \
  --arg commit_id "$COMMIT_SHA" \
  --arg path "relative/path/to/file.rb" \
  --arg body "**Concern:** Description and suggestion" \
  --argjson line 42 \
  '{commit_id: $commit_id, event: "COMMENT", body: "", comments: [{path: $path, line: $line, side: "RIGHT", body: $body}]}' \
| gh api repos/${REPO}/pulls/${PR_NUMBER}/reviews --method POST --input -
\`\`\`

**Rules:**
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

### Claude.ai Cowork

Once the plugin is distributed to your org, use it from any Cowork project. See your org's Cowork plugin settings for availability.

### GitHub Actions

See `.github/workflows/code-review.yml` in this repo for a template workflow that checks out this plugin and runs the skill on every PR.
