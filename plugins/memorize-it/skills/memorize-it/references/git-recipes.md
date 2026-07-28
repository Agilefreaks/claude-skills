# Reference: git recipes

Concrete commands for capture, squash survival, tiered recall, and audit. These are defaults — where a project configures platform mechanics or a different convention, follow the project rules.

Conventions used below: `<trunk>` is the default branch (e.g. `main`); `<key>` is the issue key; `<path>` is a file or directory.

## Contents

- [Capture](#capture)
- [PR/MR description](#prmr-description)
- [Preserve on merge](#preserve-on-merge)
- [Recall (tiered)](#recall-tiered)
- [Audit](#audit)

---

## Capture

Commit with a block. Compose the message (multi-line) and pass it via `-F`:

```sh
# Write the message to a temp file, then:
git commit -F <message-file>

# Or amend the message of the most recent (unpushed) commit:
git commit --amend -F <message-file>
```

Check that a message carries the labeled block (the required labels are present):

```sh
git log -1 --format=%B | grep -iE '^(why|what|tried|next|concerns):'
```

Sweep a composed message (or a PR/MR body) for what must never land — credential-shaped values, one developer's local setup, conversational residue, and attribution footers:

```sh
# If the enforcement hook is installed, reuse ITS patterns — one source of truth,
# so the manual sweep can never be weaker than what will reject the commit:
eval "$(grep -E '^(SECRET|LOCAL|ATTRIBUTION)_REGEX=' .githooks/commit-msg 2>/dev/null)"

# Fallback when no hook is installed (keep in sync with assets/commit-msg and
# assets/commit-lint.yml — all three carry the same three patterns):
: "${SECRET_REGEX:=(AKIA|ASIA)[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{16,}|xox[baprs]-[0-9A-Za-z-]{10,}|sk-[A-Za-z0-9]{20,}|AIza[0-9A-Za-z_-]{20,}|eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}|BEGIN [A-Z ]*PRIVATE KEY|(pass(word|wd)?|secret|token|api[_-]?key|access[_-]?key)[[:space:]]*[:=][[:space:]]*.?[A-Za-z0-9+/=_-]{12,}}"
: "${ATTRIBUTION_REGEX:=Co-Authored-By:.*(Claude|Copilot|Cursor|Codex)|Generated with .*(Claude|Copilot|Cursor|Codex)}"
: "${LOCAL_REGEX:=/(Users|home)/[A-Za-z0-9._-]+/|\.git/info/exclude|(^|[^A-Za-z])\.env($|[^A-Za-z])}"

# Conversational residue has no enforced counterpart — judgment only:
RESIDUE='say the word|let me know|want me to|(^|[^[:alpha:]])I |\byou\b'

git log -1 --format=%B | grep -niE -- "$SECRET_REGEX|$ATTRIBUTION_REGEX|$LOCAL_REGEX|$RESIDUE"

# Same sweep over a PR/MR body:
gh pr view <number> --json body --jq .body | grep -niE -- "$SECRET_REGEX|$ATTRIBUTION_REGEX|$LOCAL_REGEX|$RESIDUE"
```


Hits are candidates, not verdicts — `you` inside a quoted error message is fine, a stray `I see no hazard` is not. Read each hit and decide. Nothing matching is the expected state.

A `$SECRETS` hit is the one exception: treat it as real until proven otherwise. If the value ever reached a pushed commit or a PR/MR body, rewriting the message does **not** contain it — say so and get it rotated.

Trailer variant — read a single field across history:

```sh
git log --format='%H %(trailers:key=Why,valueonly)'
```

## PR/MR description

Read the branch's blocks to consolidate them into one description (see `pr-description.md` for what to lift from where):

```sh
git log --reverse --format='--- %h %s%n%b' <trunk>..HEAD
```

Read and replace a description (GitHub shapes; GitLab equivalents use `glab mr`, recorded in project rules when the host is GitLab):

```sh
gh pr view <number> --json title,body --jq .body > <file>

# Edit <file>, then replace the whole body — never append an edit log:
gh pr edit <number> --body-file <file>
```

Detect a project template before writing a description for the first time:

```sh
ls .github/PULL_REQUEST_TEMPLATE.md .github/pull_request_template.md \
   .github/PULL_REQUEST_TEMPLATE/ .gitlab/merge_request_templates/ 2>/dev/null
```

## Preserve on merge

List the blocks on the current branch (everything not yet on the trunk), to verify compliance and to compose a consolidated conclusion before a squash:

```sh
# Per-commit subjects on the branch:
git log --format='%h %s' <trunk>..HEAD

# All block bodies on the branch, in order (subject + body per commit):
git log --reverse --format='--- %h %s%n%b' <trunk>..HEAD
```

- **Concatenation squash source:** if every branch commit is compliant, no assembly is needed — remind the developer not to clear the host UI's pre-filled squash body.
- **PR/MR-description squash source:** consolidate the branch blocks into one block and place it in the PR/MR description's managed section (use the project's configured PR/MR update mechanics).

## Recall (tiered)

**Tier 1 — local commit body.** Read blocks for a path or a commit, offline:

```sh
# Decision history touching a path (most recent first):
git log --format='%H%n%s%n%b%n---' -- <path>

# Just the labeled lines for a path (first line of each field; read full bodies above):
git log --format=%B -- <path> | grep -iE '^(why|what|tried|next|concerns):'

# For one commit:
git show -s --format=%B <sha>
```

**Tier 2 — PR/MR description (mandatory fallback when Tier 1 is empty).** First map the trunk commit to its PR/MR, then read the description. The mapping and read commands are host-specific and come from the project rules; the common GitHub shapes:

```sh
# The squash subject usually ends with (#123); otherwise find the PR by SHA:
gh pr list --search "<sha>" --state merged --json number,title,body
gh pr view <number> --json title,body
```

(GitLab equivalents use `glab mr` — recorded in project rules when the host is GitLab.)

**Tier 3 — individual PR/MR commits (deepest).** After a squash the original commits remain attached to the PR/MR on the host even though they are absent from the trunk's history:

```sh
gh pr view <number> --json commits \
  --jq '.commits[] | .messageHeadline + "\n" + .messageBody'
```

**Degradation.** If no tier yields a block, fall back to plain subjects (`git log --oneline -- <path>`) and PR/MR titles, and state that no structured context was found.

## Audit

Find non-compliant commits in a range (default: the current branch's commits not on the trunk):

```sh
git log --format='%H%x09%s%x09%b' <trunk>..HEAD
```

For each commit, flag it when:

- the subject does not match the configured convention (default: starts with `[<key>]`), or
- the body carries no labeled block (no `why:`/`what:`/… lines; or no recognized trailers, in the trailer variant), or
- a required field label (`why`, `what` by default) is absent from the block, or
- the body carries an assistant-attribution footer, machine-local detail, or conversational residue — use the sweep under [Capture](#capture).

Report the list with what each is missing, then offer fixes:

```sh
# Most recent commit:
git commit --amend -F <message-file>

# Earlier UNPUSHED commits — reword one at a time via an interactive rebase.
# (Interactive rebase is a human-driven step; prepare the messages and let the
#  developer run it. Never rewrite already-pushed/shared history automatically.)
```
