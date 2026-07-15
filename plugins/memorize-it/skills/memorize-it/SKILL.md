---
name: memorize-it
description: "Capture the conclusion of a prompting/working session as structured context in git commits (why, what, tried, next, concerns) and recall it later so a future session or reviewer understands why decisions were made. Use when committing, when the user says 'memorize this' / 'save the session', when investigating why code exists or what was decided, and as context for a code review. Squash-and-merge aware: rich per-commit context survives onto the trunk and is recalled with a tiered fallback (commit body → PR/MR description → individual PR commits). Also use when asked to set up, configure, or onboard this skill, or to audit commits that don't follow the standard."
---

# Memorize It

> Turn the conclusion of prompting work into durable, structured context stored in git — so the next session (human or Claude) understands *why* things were built and decided, not just *what* changed.
>
> The commit convention, issue-key derivation, merge/squash behavior, and platform mechanics (how to read/update PRs or MRs) are project-specific and defined separately in your project's rules. This skill ships working defaults so it is useful with zero configuration.

This skill has four working modes, plus setup:

- **Capture** — write a session conclusion into the commit (and optionally the PR/MR).
- **Preserve on merge** — make sure that context survives a squash-and-merge onto the trunk.
- **Recall** — read stored context back, tiered, so a future session or a review is grounded in prior decisions.
- **Audit** — find commits that don't follow the standard and prompt the developer.

It is designed to work alongside — not replace — the `feature-development` skill (capture at hand-off) and the `code-review` skill (recall feeds problem validation). See [Integration](#integration).

---

## The standard commit

The default format is a subject with an issue key, followed by a delimited, parseable context block:

```
[feat-123] Add token-bucket rate limiter

why: API had no per-key throttling; a few keys were saturating the pool.
what: token-bucket middleware, 100 req/min default, applied to /v1/*.
tried: fixed-window (too bursty) and leaky-bucket (heavier) — token-bucket was simplest.
next: make limits configurable per plan tier.
concerns: in-memory store is single-instance; needs a shared store before horizontal scale.
```

Two things matter:

- **Subject** — `[<key>] <summary>`, where `<key>` is the issue/ticket key. How the key is derived (branch name, PR title, task tool) is a project decision; see [Extension points](#extension-points).
- **Context block** — a run of `key: value` lines after a blank line. The field labels are the structure — no delimiters. Values may wrap onto indented continuation lines. The labels stay recognizable even after a squash concatenates several commit messages, so the block can be read back out of a collapsed message.

**Fields.** `why` and `what` are required; `tried`, `next`, and `concerns` are optional and filled when relevant. The field set is extensible — a project may add fields (e.g. `rejected`, `constraint`, `related`). See `references/commit-format.md` for the full field glossary, a git-trailer variant (for `git`-native parsing), and a conventional-commits variant.

**Write the block for the next reader, not for `git blame`.** It is a briefing: enough that someone with no memory of this session knows why the change exists and what to watch out for.

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. **Read existing project rules first** (`.claude/rules/`, `CLAUDE.md`). Do not duplicate a commit convention, trunk branch name, or issue-linking rule that already exists. If the project already defines a commit format, adapt to it rather than imposing this skill's default.
2. **Inspect the project** for signals:
   - Recent commit history (`git log --oneline -30`) — is there a prevailing subject style? Do subjects already carry an issue key? Do any bodies already carry structured context?
   - Branch naming (`git branch -a`) — does it encode an issue key (e.g. `feat/PROJ-123-...`)?
   - The remote host (`git remote -v`) — GitHub, GitLab, or other — which determines available PR/MR mechanics.
   - A conventional-commits config (`commitlint`, `.commitlintrc*`) or a PR/MR template.
3. **Present interactive choices** (use a choice dialog per decision, pre-filled from what you detected). **Phrase every question in plain, user-facing language — never expose this skill's internal mode names (e.g. "Capture", "Preserve on merge", "Recall", "Audit") in a question or option label.** Those names orient the agent, not the user.
   - **Commit convention** — if the history already shows a prevailing style, **summarize what you found in plain language and ask to confirm or adjust**, rather than presenting a bare either/or. For example: *"Your commits already look like `PROJ-123: summary` and carry the ticket key from the branch name — I'll follow that and add the why/what block below the subject. Use it, or adjust?"* If nothing is detected, offer: bracket-key `[key] summary` (default), conventional-commits `type(scope): summary`, or a custom subject rule. The context block is kept either way.
   - **Issue-key derivation** — from branch name, PR/MR title, a task-management integration, or none.
   - **Required fields** — default `why` + `what` required; `tried`/`next`/`concerns` optional. Offer to add project-specific fields.
   - **Context block style** — labeled `key: value` lines (default, handles multi-line prose) or native git trailers (`git interpret-trailers`-parseable; unknown keys ignored). See `references/commit-format.md`.
   - **Squash-message source** — how your host composes a squash commit message: *commit details* (concatenates commit messages — blocks survive automatically) or *PR/MR title + description* (blocks must live in the PR/MR body). This determines the [Preserve on merge](#preserve-on-merge) path. Default: commit details.
   - **PR/MR mirroring** — off by default. If on (recommended when the squash source is the PR/MR description), record the platform mechanics for reading and updating the PR/MR description (a marker-delimited managed section).
   - **Audit scope** — the default range to scan (e.g. `<trunk>..HEAD`) and whether the developer wants to be prompted about non-compliant commits proactively.
4. **Write `.claude/rules/memorize-it.md`** containing only the choices that differ from the defaults. Omit anything left at default — the skill's built-in behavior covers it. Record platform mechanics (PR/MR read/update commands) here when mirroring is enabled, so the skill stays generic.

**What to defer to a human (Setup):** verifying that the chosen squash-message source matches the host's actual repository setting (e.g. GitHub's "Default commit message" for squash merges). The skill records the intended behavior; a human must confirm the host is configured to match.

If the user accepts every default and nothing needs recording, say so and skip writing a rules file.

---

## Capture

**Trigger:** whenever a commit is created (including the `feature-development` hand-off), or when the user says "memorize it" / "save the session".

1. **Compose the conclusion from the session you actually had.** Draw `why`/`what`/`tried`/`next`/`concerns` from the work just done — the problem being solved, the approach chosen, alternatives you rejected and why, follow-ups, and risks or deferrals. Do not invent fields you have no basis for; leave optional fields out rather than padding them.
2. **Derive the issue key** per the project's rule (branch name, PR/MR title, task tool). If no key can be derived, use the configured convention's neutral form and note in the block that no key was found — this is something [Audit](#audit) will later flag.
3. **Build the subject and block** per [the standard](#the-standard-commit) (or the project's configured convention).
4. **Create the commit** with the subject and block as its message.
5. **Optionally mirror to the PR/MR** — if PR/MR mirroring is enabled, update the marker-delimited managed section of the PR/MR description with a consolidated conclusion for the change as a whole. Follow the platform mechanics from the project rules.

**Where the block belongs when commits get reshaped.** Under a checkpoint-then-curate flow (e.g. `feature-development`'s hand-off, which reshapes many working checkpoints into a few logical commits), do not spend effort memorizing on throwaway checkpoints — write the conclusion on the **final, reshaped commits**. And because most branches squash-merge, the message that ultimately matters is the one that lands on the trunk: ensure the branch's blocks (or the PR/MR description) are compliant *before* the merge, per [Preserve on merge](#preserve-on-merge).

**What to defer to a human:** the factual accuracy of the conclusion. You can record what was decided this session; a human confirms it reflects reality — especially `concerns` and `next`, which commit the team to follow-ups.

---

## Preserve on merge

A squash-and-merge collapses every branch commit into a single trunk commit. The per-commit context must survive that collapse, or recall on the trunk finds nothing.

**How the squash commit message is composed is host-specific — it is an extension point.** The two common cases and their defaults:

- **Host concatenates commit messages** (e.g. GitHub squash with the "commit details" default): every compliant per-commit block is carried into the squash commit body automatically, and the host's own `* <subject>` separators keep them apart. **No assembly step is needed** — just compliant commits. The one caution: whoever performs the merge must **not clear the pre-filled squash body** in the host UI. Surface that reminder at merge time.
- **Host composes from the PR/MR description** (e.g. GitHub squash with the "PR title and description" default): the block must be in the PR/MR body to survive. Keep the consolidated conclusion mirrored into the PR/MR description's managed section (enable PR/MR mirroring in [Capture](#capture)).

When you are asked to prepare a branch for merge, verify the survival path holds: either the branch commits are compliant (concatenation case) or the PR/MR description carries the consolidated block (description case). If neither, say so and offer to fix it before merge (amend/reword unpushed commits, or update the PR/MR description).

**What to defer to a human:** the squash-and-merge itself is performed in the host UI. The skill prepares and verifies the message that will land on the trunk and reminds the developer what to confirm; the human clicks merge and is responsible for not discarding the pre-filled body.

See `references/git-recipes.md` for the commands to list a branch's blocks and compose a consolidated conclusion.

---

## Recall

**Trigger:** when the user asks why code exists or what was decided, when investigating a file's history, as an input to a code review, and whenever prior decisions are relevant to the work at hand.

Recall is **tiered** — cheapest first, and it always produces *something* useful even when no structured context exists:

1. **Local commit body** (`git log`) — read the labeled `key: value` blocks from the relevant commits. On a branch these are per-commit; on the trunk a squash commit body holds the concatenated/consolidated blocks. This is offline and free; it is often enough.
2. **PR/MR description** — **if step 1 finds no context, always fall back to this at minimum.** Map the trunk commit to its PR/MR (hosts like GitHub append `(#123)` to the squash subject by default; otherwise search the host for the PR/MR containing the commit SHA) and read the managed section / description. Requires host access; the mechanics are project-configured.
3. **Individual PR/MR commits** — the deepest layer, for the step-by-step decision trail. After a squash the original commits remain attached to the PR/MR on the host even when they are gone from the trunk's history; fetch them and read each block. Use this when the consolidated context is not enough.

Tiers 2 and 3 require the project to have configured platform mechanics (how to map a commit to its PR/MR and read it). If those are not configured, tier 1 still works fully offline — say plainly that the PR/MR layers are unavailable because no platform mechanics are configured, rather than treating it as a failure.

**Graceful degradation:** if none of the available layers yields a structured block, fall back to plain commit subjects and PR/MR titles. Say explicitly that no structured context was found — do not imply decisions were recorded when they weren't.

Present recalled context as a short briefing: what the change did, why, alternatives rejected, and any `concerns`/`next` still open. When recall feeds a code review, pass this into the review's problem-validation step.

**What to defer to a human:** whether recalled decisions are still valid. Context reflects what was true when it was written; a `concern` may be resolved or a `constraint` lifted. Flag anything that looks stale rather than treating it as current fact.

See `references/git-recipes.md` for the exact commands per tier.

---

## Audit

**Trigger:** on request, per the configured audit scope, or when preparing a branch for merge.

1. **Scan the range** (default `<trunk>..HEAD` on the current branch; configurable).
2. **Flag non-compliant commits** — any commit missing the `[<key>]` subject (per the project's convention), missing the context block, or missing a required field label (`why`/`what` by default).
3. **Report and prompt the developer** — list each non-compliant commit with what it's missing, then offer to fix what is safely fixable:
   - The **most recent** unpushed commit → `git commit --amend` with the corrected message.
   - **Earlier** unpushed commits → rewording these means an interactive rebase, which needs a terminal editor the agent can't drive. Prepare the corrected messages and hand the reword to the developer to run, rather than attempting it.
4. **Prefer fixing the squash message over rewriting branch history.** Because most branches squash-merge, the message that lands on the trunk is the one that matters. If per-commit compliance can't be fixed cleanly, the higher-value move is to ensure the consolidated block is correct at merge time (see [Preserve on merge](#preserve-on-merge)) and to apply the convention going forward.
5. **Never silently rewrite.** Present every proposed message and let the developer approve it.

**What to defer to a human:** rewriting **already-published** history (amending pushed commits forces a shared-history rewrite that can disrupt collaborators) and running any **interactive rebase**. Report those commits as non-compliant, recommend fixing the convention going forward, and leave the decision to the developer.

---

## Integration

These skills stay independent; the touchpoints are soft:

- **`feature-development`** — its hand-off phase is a natural [Capture](#capture) trigger. When a feature or fix is handed off, memorize the conclusion into the commit so the decision trail is recorded at the moment it's freshest.
- **`code-review`** — [Recall](#recall) is an input to the review's problem-validation phase. Before reviewing, pull the stored context for the change so the reviewer knows the intent and the rejected alternatives, and doesn't re-raise settled decisions.

If neither skill is installed, `memorize-it` works on its own.

---

## Extension points

Every one has a working default, so the skill is useful out of the box and richer when configured. Configure them in `.claude/rules/memorize-it.md` via [Setup](#setup).

| Decision | Default |
|---|---|
| Commit convention | `[key] summary` bracket-key subject |
| Issue-key derivation | from the branch name |
| Required fields | `why`, `what` |
| Context block style | labeled `key: value` lines |
| Squash-message source | commit details (blocks concatenated) |
| PR/MR mirroring | off |
| Platform mechanics (read/update PR/MR) | none — required only if mirroring is on |
| Trunk branch | the repository's default branch |
| Audit scope | `<trunk>..HEAD`, on request |

---

## Core principles

1. **Briefing, not blame.** The context block is written for a future reader with no memory of this session — the intent and the rejected paths, not a restatement of the diff.
2. **Survive the squash.** Context that doesn't reach the trunk is lost. Always confirm the survival path before merge.
3. **Tiered recall, guaranteed fallback.** Cheap-and-local first; when `git log` is empty, always check the PR/MR description at least. Never claim context that isn't there.
4. **Prompt, don't rewrite.** Audit surfaces non-compliant commits and offers fixes; it never rewrites published history on its own.
5. **Generic core, project edges.** Conventions, key derivation, and platform mechanics live in the project's rules. The methodology here is the same everywhere.
6. **Defer honestly.** The accuracy of a conclusion, the validity of an old decision, and the merge click itself are human calls — say so.
