# memorize-it

Capture the conclusion of your prompting/working sessions as structured context in git commits, and recall it later so a future session — or a reviewer — understands *why* things were built and decided, not just *what* changed.

Inspired by the emerging practice of treating commit messages as a briefing for the next session ([git-as-memory](https://dev.to/henrywangxf/using-git-commits-as-claude-codes-memory-48e3), the [Lore](https://arxiv.org/html/2603.15566v1) protocol), with a first-class answer to the problem those approaches leave open: **surviving a squash-and-merge.**

## What it does

Four modes, plus interactive setup:

- **Capture** — writes a session conclusion into the commit and the PR/MR description. The commit gets an issue-keyed subject plus a block of labeled `key: value` lines — `why` / `what` (required) and `tried` / `next` / `concerns` (optional); the description gets the same conclusion at whole-change altitude. Written for a reader in a fresh clone: no machine-local detail, no assistant attribution.
- **Preserve on merge** — makes sure that context survives a squash-and-merge onto the trunk, whichever way your host composes the squash message.
- **Recall** — reads context back, tiered: local commit body → PR/MR description → individual PR/MR commits, with a guaranteed fallback so it always produces something useful.
- **Audit** — finds commits that don't follow the standard — missing subject key, missing field, assistant-attribution footer, machine-local detail, or leftover conversational residue — and prompts the developer to fix the safe ones.

It integrates with the [`code-review`](../code-review) skill (recall feeds problem validation) and the [`feature-development`](../feature-development) skill (capture at hand-off), and works standalone if neither is installed.

## The standard commit

```
[feat-123] Add token-bucket rate limiter

why: API had no per-key throttling; a few keys were saturating the pool.
what: token-bucket middleware, 100 req/min default, applied to /v1/*.
tried: fixed-window (bursty) and leaky-bucket (heavier) — token-bucket simplest.
next: make limits configurable per plan tier.
concerns: in-process state is per-instance; needs a shared store before scale-out.
```

No delimiters — the field labels are the structure, and they stay recognizable even after a squash concatenates several commit messages into one. See `skills/memorize-it/references/commit-format.md` for the field glossary, a native git-trailer variant, and a conventional-commits variant.

The format is the easy part; the skill also carries the craft rules that decide whether a block gets read — a subject that names the change rather than the area, a `what` that names decisions rather than files, length in proportion to the change, one logical change per commit (so a concatenating squash yields readable sections), a fixed field order, and verification stated precisely: *"applied in prod, admin API confirms the order"* beats *"tested"*.

## The standard PR/MR description

A commit block is the conclusion for one commit; the PR/MR description is the conclusion for the **whole change**, for whoever decides to merge it. Three sections, that order:

- **Why** — the problem in terms someone outside the session recognizes, and how it showed up.
- **What** — one bullet per decision a reviewer would want named, reasoning attached. Not one bullet per file.
- **Limits of this change** — what it doesn't cover, what stayed unverified, what's left to the reviewer's judgment.

That third section is the one that earns trust and the one most often dropped. And the title follows the commit convention, because on hosts that squash from the PR/MR title it *becomes* the trunk subject.

If the project has a PR/MR template, the skill fills *its* sections with the same content instead of imposing these headings. `skills/memorize-it/references/pr-description.md` has a worked example, how to consolidate a branch's commits into one description, how to revise on a re-review (rewrite in place — never an appended edit log), and a table of the common failure modes.

## Project altitude — never one developer's machine, never a secret

Both records are read by someone in a fresh clone, on another machine, months later, and both are
permanent and world-readable within the repo. So they carry the shared, reproducible view of the
project — the change, the repo as everyone gets it, the running system — and nothing below that
line, because every developer's setup legitimately differs:

- **No secrets or sensitive values.** No passwords, tokens, keys, connection strings, customer or
  personal data. A leaked value has to be *rotated*, not edited out — name a credential's
  location, never its value.
- **No one developer's setup.** No local ignore entries or hook wiring, no untracked `.env` or
  local overrides, no absolute home-directory paths (they name a person), no sibling worktrees, no
  housekeeping that isn't in the diff. A change to a *tracked* `.gitignore` or checked-in hook is
  project context and belongs; the local counterpart never does.
- **No access limits of your own.** *"Staging was never verified, I couldn't read its password"* →
  *"staging still needs an apply; it carries the same defaults and can drift the same way."*

It is also a **conclusion, not a transcript** of the session that produced it: no offers ("say
the word if you want a CI gate"), no questions, no first person, nothing addressing the reader —
those turn a record into a chat log. And where the conclusion implies a rule for *future* work,
the incident stays in the commit while the project's living docs get the instruction that
follows from it, phrased forward: "set X explicitly on anything you add", not "X was left at the
default and prod scrambled".

Commits also stay authored by the developer: no `Co-Authored-By:` trailer for an assistant and
no "Generated with" line, in the commit or the PR/MR description. The optional hook and CI
check enforce the mechanical part of all of this deterministically — attribution footers,
credential-shaped values, and local paths — and `references/git-recipes.md` has a one-line sweep
for the rest.

## Surviving squash-and-merge

A squash collapses every branch commit into one trunk commit. Two host behaviors, both handled:

- **Squash body = concatenated commit messages** (e.g. GitHub's "commit details" default) — every compliant per-commit block lands on the trunk automatically. Just don't clear the pre-filled body in the merge UI.
- **Squash body = PR/MR title + description** — the block must live in the PR/MR description; enable PR/MR mirroring so a consolidated block is kept there.

Recall reads the trunk commit body first, then **always falls back to the PR/MR description** if `git log` yields nothing, and can drill into the individual PR/MR commits (which the host keeps attached to the PR even after a squash) for the full step-by-step trail.

## Companion rules file (recommended)

The SKILL.md is the generic methodology. Project-specific decisions defer to `.claude/rules/memorize-it.md`, generated by Setup. Configurable:

| Decision | Default |
|---|---|
| Commit convention | `[key] summary` bracket-key subject |
| Issue-key derivation | from the branch name |
| Required fields | `why`, `what` |
| Context block style | labeled `key: value` lines (or native git trailers) |
| Squash-message source | commit details (blocks concatenated) |
| PR/MR description structure | `Why` / `What` / `Limits`, or the project's template |
| PR/MR mirroring + platform mechanics | off |
| Trunk branch | the repository's default branch |
| Audit scope | `<trunk>..HEAD`, on request |

Every decision has a working default, so the skill is useful out of the box and richer when configured.

## Usage

### Setup

Setup runs automatically the first time you use the skill in a project (when no
`.claude/rules/memorize-it.md` exists yet) — or on request:

```
set up memorize-it
```

It reads your existing rules, inspects commit history / branch naming / remote host to
*pre-fill* the questions, then **confirms every decision with you one at a time — no silent
assumptions, even when a detected value matches the default**. It always writes
`.claude/rules/memorize-it.md` at the end (recording your confirmed choices, and marking the
skill as configured so Setup doesn't re-trigger on the next run).

Everything Setup writes is **shared project configuration and gets committed** — never
git-ignored, never left local-only. A convention that lives in one clone never reaches the
trunk, so a fresh clone, a new worktree, or a teammate has nothing telling them it exists, and
commits quietly drift back to the old habits. Setup checks for an ignore rule already covering
those paths, stages them, and tells you the config takes effect elsewhere only once it lands on
the trunk. If it finds an existing rules file that is untracked or ignored, it treats the
project as half-configured: it confirms the recorded choices instead of re-asking everything,
then gets the file tracked.

By default the skill installs **no git hooks**, so Setup offers (recommended) to add a one-line
nudge to your project's `CLAUDE.md` so the convention is in front of Claude every session and
it reaches for the skill automatically when you commit — without a hard hook that could block
commits. Decline it and the skill still works whenever you invoke it explicitly.

If you *do* want a hard local gate, Setup can also generate an **enforcement hook** (off by
default): a `.githooks/commit-msg` script that rejects a local commit whose subject doesn't
match your convention, whose body is missing a required field (`why`/`what`), or whose message
carries an assistant-attribution footer, a credential-shaped value, or a local path. It's
enforcement-only — it never rewrites messages — and it guards local commits only (a web-UI
squash-merge composes the message server-side, where the hook can't run). Setup adapts the
bundled `assets/commit-msg` template to your confirmed convention and sets `core.hooksPath`,
integrating with any existing hook manager rather than overwriting it.

To guard **everything that reaches the trunk** — including on cloud hosts where local hooks
can't help — Setup can generate a **CI enforcement** workflow (off by default): a deterministic
PR check (no Claude token needed) that validates either every commit the PR adds or the PR
title + description (matching your squash-message source). Made a **required status check** in
branch protection, it blocks non-compliant PRs from merging. Setup adapts the bundled
`assets/commit-lint.yml` for GitHub Actions and describes the GitLab CI equivalent. For an
absolute server-side gate on every push, pair it with a GitHub Enterprise pre-receive hook or
a GitLab push rule where available.

### Claude Code (terminal)

After installing via `/plugin install memorize-it@agilefreaks-skills`:

```
memorize this session
```

```
why does the rate limiter exist? — recall the context
```

```
audit this branch for commits that don't follow the standard
```

### Claude.ai Cowork

Once the plugin is distributed to your org, use it from any Cowork project. See your org's Cowork plugin settings for availability.

## What is deferred to a human

- The **factual accuracy** of a captured conclusion (especially `concerns` and `next`).
- Judging whether a line is machine-local. The hook and CI check catch attribution footers
  deterministically; "the other worktree already has a copy" is a judgment call no regex makes.
- Landing the config commit — pushing and merging it is yours; until then the skill is
  configured in that clone only.
- Confirming the host's squash setting matches the configured squash-message source.
- The **squash-and-merge click** itself — the skill prepares and verifies what will land on the trunk; the human merges.
- Rewriting **already-published** history flagged by the audit.
