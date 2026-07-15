# Reference: git recipes

Concrete commands for capture, squash survival, tiered recall, and audit. These are defaults — where a project configures platform mechanics or a different convention, follow the project rules.

Conventions used below: `<trunk>` is the default branch (e.g. `main`); `<key>` is the issue key; `<path>` is a file or directory.

## Contents

- [Capture](#capture)
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

Trailer variant — read a single field across history:

```sh
git log --format='%H %(trailers:key=Why,valueonly)'
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
- a required field label (`why`, `what` by default) is absent from the block.

Report the list with what each is missing, then offer fixes:

```sh
# Most recent commit:
git commit --amend -F <message-file>

# Earlier UNPUSHED commits — reword one at a time via an interactive rebase.
# (Interactive rebase is a human-driven step; prepare the messages and let the
#  developer run it. Never rewrite already-pushed/shared history automatically.)
```
