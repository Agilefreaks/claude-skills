# Reference: The memorize-it PR/MR description

How to write the conclusion for a change as a whole: the shape, a worked example, consolidating several commits into one description, revising on a re-review, and adapting to a project template.

## Contents

- [Shape](#shape)
- [Worked example](#worked-example)
- [Consolidating several commits](#consolidating-several-commits)
- [Revising on a re-review](#revising-on-a-re-review)
- [Adapting to a project template](#adapting-to-a-project-template)
- [Common failure modes](#common-failure-modes)

---

## Shape

```markdown
## Why

<One paragraph: what was broken or missing, and how it showed up. In terms
someone who wasn't in the session recognizes.>

## What

- **<Decision>** — <the reasoning, compressed to a clause.>
- **<Decision>** — <…>

## Limits of this change

<What it doesn't cover, what stayed unverified, what a reviewer has to decide.
Facts about the change, not offers to do more.>
```

Three sections, that order. `Why` before `What` because a reviewer decides whether to care before deciding whether the approach is right. `Limits` last because it's what they carry into the review.

The title follows the project's commit convention — on hosts that squash from the PR/MR title, it *becomes* the trunk subject.

## Worked example

A small infrastructure fix, whole-change altitude:

```markdown
## Why

The realm's browser flow silently reordered itself in prod: the lookup-or-create
step ended up running last, so logins for unknown emails failed. Every execution
sat at the default priority `0`, and the server sorts by priority then falls back
to database row order — so any later write could reshuffle the flow.

## What

- **Explicit `priority` on every execution and subflow**, gapped by 10 so a step
  can be inserted without renumbering. This, not declaration order, is what fixes
  the sequence.
- **Dropped the sibling `depends_on` chain** — it only sequenced creation, which
  the server never reads back. The one on the flow binding stays, so the flow
  isn't bound as the browser flow before its steps exist.
- **Stopped sending empty CAPTCHA config** on the abuse-detection execution; the
  server drops empty values, so every plan re-added them as a permanent no-op diff.

## Limits of this change

- Applied and verified in prod — the admin API reports the intended order. Staging
  still needs an apply; it carries the same all-zeros priorities and can reorder
  the same way.
- Removing the chain lets sibling executions be created in parallel. No ordering
  hazard remains, but contention on the shared executions endpoint is unproven —
  it would only surface on a from-scratch flow creation, which an existing realm
  never repeats.
```

What makes it work: `Why` names a user-visible symptom and the mechanism behind it. Each `What` bullet is a decision with its reason attached, and one of them is a *removal* — the things deliberately taken out are exactly what a reviewer would otherwise flag. `Limits` separates what was verified (prod, by a named method) from what wasn't (staging) and from what can't be settled here (the parallel-create question), without a single offer or first-person aside.

## Consolidating several commits

A description is not a concatenation of the branch's commit blocks. Read them (`git log --reverse --format='--- %h %s%n%b' <trunk>..HEAD`) and lift:

- **`Why`** — the *branch's* motivation, once. Individual commits often share it; state it a single time at the level of the whole change.
- **`What`** — one bullet per decision that survived into the final state. A decision made in commit 2 and reversed in commit 5 doesn't appear; the reversal's reasoning may belong in `Limits` or in `tried` on the commit itself.
- **`Limits`** — the union of the still-open `concerns` and `next` across commits, minus anything the branch went on to resolve.

Where the host squashes from **commit details**, per-commit blocks reach the trunk on their own, so the description is for the reviewer and needn't be parseable.

Where it squashes from the **PR/MR title + description**, the description *is* the trunk message, so it carries the prose sections *and* a labeled block — the only copy recall will find. The block goes last, in a marker-delimited managed section, with **every label at column 0**:

```markdown
## Why

…

## What

- …

## Limits of this change

- …

<!-- memorize-it:begin -->
why: every execution sat at the default priority, so the flow reordered itself
  on a later write and logins for unknown emails failed.
what: explicit priority on each execution and subflow, gapped by 10; dropped the
  sibling depends_on chain, which never controlled read-back order.
next: apply in the staging env — it carries the same defaults.
<!-- memorize-it:end -->
```

Indenting those labels under a heading or wrapping them in a fenced code block breaks both the parse and the bundled CI check in `pr` mode. The markers let mirroring rewrite its own section without touching the prose above it.

## Revising on a re-review

Rewrite in place; never append a changelog of your own edits. The description records the change's final state, not the negotiation that produced it — "Updated after review feedback" tells a future reader nothing about the code.

- New commits pushed → fold their decisions into `What` and update `Limits`.
- A `Limits` item resolved → remove it, or restate it as a verified fact under `What`.
- A reviewer's concern accepted → the change absorbs it; the description reflects the new state as if it had always been that way.

The review conversation lives in the review threads. The description stays a conclusion.

## Adapting to a project template

If the project has a PR/MR template, answer the same three questions under *its* headings — a template with `### Context / ### Changes / ### Risks` gets `Why` / `What` / `Limits` content in that order, no extra headings bolted on.

- Keep every section the project's process depends on (checklists, ticket links, deploy notes).
- A section that genuinely doesn't apply gets a one-line "n/a — <reason>", not deletion.
- Where the template asks for something this standard doesn't cover (a screenshot, a migration plan), fill it; the standard is a floor, not a ceiling.

## Common failure modes

| Failure | What it looks like | Fix |
|---|---|---|
| Diff narration | `What` reads as a file-by-file walkthrough | One bullet per *decision*; delete anything the diff states plainly |
| Missing limits | Ends confidently at `What` | Add what wasn't verified and what's left to judgment |
| Offers and questions | "Say the word if you want…", "Should I also…?" | State the fact instead: what exists, what doesn't |
| Session narration | "First I tried X, then discovered Y" | Keep the outcome; a rejected option belongs in `tried` on the commit |
| Local instructions | "Delete your local copy of X before pulling" | Cut it, or phrase a real migration step for anyone on the team |
| Edit log | "Update 3: fixed after review" | Rewrite in place; the description is the final state |
| Unearned confidence | "Tested and working" | Name what was verified and how, and what wasn't |
