---
name: feature-development
description: "End-to-end development methodology for features and bug fixes: frame the
  requirement, explore the codebase and establish a green test baseline (plus root-cause
  analysis for bugs), plan, implement with a configurable test-first loop, verify by
  driving the running app, and hand off for review. Use when developing a new feature,
  fixing a bug or regression, debugging broken behaviour, working a ticket or issue, or
  implementing a spec from scratch. Also use when asked to set up, configure, or onboard
  this skill."
---

# Feature Development Skill

> End-to-end development for features and bug fixes: frame → explore → plan → implement → verify → hand off.
>
> Build/test/lint/run commands are not configured here — the skill uses whatever your
> project already has in CLAUDE.md or .claude/rules/. Review hand-off, branch conventions,
> and test collaboration mode are configured via Setup.

Throughout, "feature" means the unit of work in flight — a new feature or a bug fix. The methodology is identical for both; bug-specific differences are called out as **For a bug fix** notes where the two diverge.

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. Read all existing project rules (`.claude/rules/`, `CLAUDE.md`) to understand what is
   already configured. Do **not** re-ask build, test, lint, or run commands — those are
   project-level concerns already configured elsewhere.
2. Inspect the project:
   - Existing skill rules (`code-review.md`, `run.md`, `verify.md` in `.claude/rules/`)
   - Any documented **spec or issue workflow** — a rules file or CLAUDE.md section that
     describes where the spec lives (inline note, tracking issue, external doc) or how to
     handle a feature ticket or bug report before writing code
   - Any documented **branch naming convention** — in CI configs, CLAUDE.md, or rules files
   - Any documented **testing strategy** — a rules file or CLAUDE.md section describing
     which component types systematically require tests, required test patterns or idioms to
     follow or avoid, test file locations, and where in the test pyramid different changes
     should be covered. This is **not a setup question** — carry this information directly
     into the implementation loop so every test written follows the project's strategy
     automatically. Mention in the closing setup summary that a strategy was found and will
     be followed.
3. Present **skill-specific choices only** via interactive dialogs. **All questions must be
   phrased in plain, user-facing language — never expose the skill's internal phase numbers
   or names (e.g. "Phase 1", "Phase 3", "Frame", "Hand Off") in a question or option label.
   Those names are for the agent's own orientation, not for the user.**

   **Spec, issue & branch workflow:**
   If the project documents how to capture a spec and name branches, **summarize what you
   found in plain language and ask to confirm or adjust** — do not present a jargon
   either/or. For example:

   > "Your project documents a workflow: the spec lives in a GitHub issue (I'll use the
   > existing one, or create it if there isn't one yet) and branches are named
   > `{issue_number}-short-desc`. I'll follow that instead of the skill's defaults — use
   > it, or adjust?"

   Key rule: **never force creating a tracking issue** when one already exists for the
   work at hand. Detect and use the existing issue; create one only when the documented
   workflow calls for it and none exists. If nothing is documented, fall back to the
   skill defaults (inline spec note; `feature/<ticket-slug>` from `main`).

   **Other choices:**
   - **Test collaboration mode** — the loop used while implementing:
     - *Solo AI* (default): agent runs red/green itself, one test at a time; escape hatch
       to agentic manual testing when test-first genuinely doesn't fit
     - *Assert-in-the-loop*: agent scaffolds one test (arrange + act + empty assertion
       placeholder); human writes the assertions; agent confirms genuine red and implements
       to green
     - *Ping-pong*: human writes failing test → agent greens it and writes next failing
       test → human greens it and writes next → …
     - *Ask each feature*: no fixed mode; at the start of each feature's implementation
       the skill asks which of the three modes to use for that feature only
   - **Review delegation** — detected automatically (check for `code-review.md` in
     `.claude/rules/`); ask only if ambiguous
   - **Commit granularity** — how commits are made while implementing and reshaped at
     hand-off:
     - *Checkpoint + curate* (default): checkpoint-commit after each green; reshape into
       a few logical, behaviour-level commits at hand-off
     - *One commit per criterion*: commit once per acceptance criterion (may span 2–3 tests);
       no reshape step needed
     - *Keep every checkpoint*: commit per green, never reshape (maximum bisect granularity,
       noisiest published history)
4. Write `.claude/rules/feature-development.md` containing only the non-default choices.
   If the user accepts all defaults (or the detected workflow covers everything), confirm no
   rules file is needed and stop.

**What to defer to a human (Setup):** the correct branch base and naming convention if the
project doesn't document them — Setup cannot infer this reliably from `.git/config` alone.

---

## Phase 1: Frame

Read the ticket, issue, or spec. Write a short spec (a scratch note is fine) containing:

- **Problem statement** — what is broken or missing, and for whom
- **Acceptance criteria** — numbered, testable, explicit; these drive Phase 4 tests and
  Phase 5 verification
- **Out of scope** — what this change is NOT; prevents scope creep and clarifies intent

*Delivering new code is near-free; delivering correct code is still expensive. Invest the
savings in precise acceptance criteria here — they prevent rework later.*

If your project defines where specs are tracked or what format they follow, use that.
Otherwise, produce the spec inline as a numbered list before continuing.

**For a bug fix:** the spec is a defect report — capture exact reproduction steps,
expected behaviour, and the actual (wrong) behaviour observed. The first acceptance
criterion becomes "the bug no longer reproduces", proven by a test that fails *because of
the defect* and passes once fixed.

**What to defer to a human:** any acceptance criterion you cannot derive from the ticket
without making assumptions. Stop and ask rather than guess.

---

## Phase 2: Explore & Baseline

Launch parallel Explore subagents — each gets a fresh context window and does not consume
top-level context:

- One agent: locate existing patterns, utilities, and components relevant to the feature
- One agent: find the test suite, understand test organization and the testing harness, and
  surface any documented testing strategy from project rules or CLAUDE.md (which layers
  require tests, required/forbidden patterns, test locations). Carry this summary into the
  plan and implementation phases — every test written in this session follows it.
- Add agents for specific unknowns (data layer, UI components, integration points) as needed
- **For a bug fix:** dedicate one Explore agent to root-cause analysis — starting from
  the reproduction steps, trace the failure to the smallest code location responsible.
  Distinguish symptom from cause; fixing only the symptom leaves the underlying defect alive.

Apply the *Hoard* principle: explicitly note "build this by combining X and Y" where existing
working examples in the codebase can be recombined. Build new things by combining known things
rather than from scratch.

Run the full test suite before touching any production code. This step:
- Records the green baseline — pre-existing failures are documented here, not fixed later
- Forces the agent to locate the test harness and understand the project's test scale
- Establishes a testing-first bias for the rest of the session

If your project defines how to run the tests, use that. Otherwise, look for the suite in
CLAUDE.md, `.claude/rules/`, Makefile, or package.json scripts.

**What to defer to a human:** pre-existing test failures that pre-date this work. Document
them; do not fix them as part of this feature. For a bug fix: if the root cause remains
unlocated after Explore, surface what is known and what is still unknown rather than
guessing a fix.

---

## Phase 3: Plan

Produce an implementation plan:

- **Ordered steps, each written as a red/green pair:**
  > Prove: *\<the behaviour the failing test demonstrates\>*
  > Implement: *\<the minimal production change to make it pass\>*

  A step with no "Prove" half is not an implementation step — it is setup or refactor
  preparation, and must be labelled as such. This format is load-bearing: it ensures the
  plan itself encodes test-first, so the intent survives into execution.

  **For a bug fix:** the first pair is the regression test —
  *Prove: a test that reproduces the reported defect (fails for the right reason) /
  Implement: the root-cause fix that makes it pass.* Subsequent pairs cover related cases
  the fix must also satisfy.

- Alternatives considered and why they were rejected
- Risk areas: dependencies, edge cases, integration points, known unknowns

Create the git branch. Branching is the first physical act — before any code is written.
Name the branch according to the configured or default convention.

**Hard human checkpoint.** Present the plan and wait for explicit approval. Do not write a
single line of production code until the human confirms. This is the authorization gate.

If your project defines a branch naming convention (via CI config, CLAUDE.md, or rules),
follow it. Default: `feature/<ticket-slug>` branched from `main`.

**What to defer to a human:** the checkpoint itself. The human's approval authorizes
implementation. Do not skip or abbreviate this step.

---

## Phase 4: Implement

**If the configured test collaboration mode is *Ask each feature* (or no mode was set),
ask the human now** — before writing any code — which mode to use for this feature:
Solo AI / Assert-in-the-loop / Ping-pong. Record the answer for the rest of this feature.
Phrase the question in plain language; do not reference phase numbers or internal names.

**Entry gate: the first file you create or edit in this phase must be a test file.**
Before touching any production code, name the failing test that edit serves. If you cannot
name it, stop — write the test first. Exception: the explicitly named escape hatch below.

Drive the test-first loop using the chosen test collaboration mode. The full state machine
for each mode is in `references/test-collaboration-modes.md`.

**Invariant across all modes: exactly one test at a time.** Never generate 2–10 tests
up front. One failing test → implement → green → commit → next test.

**Solo AI (default):** write one failing test, confirm it genuinely fails, implement the
minimum to make it pass, re-run to confirm green, checkpoint-commit, repeat.

**Assert-in-the-loop:** agent scaffolds one test (arrange + act + assertion placeholder);
human writes the assertions; agent confirms genuine red; agent implements to green; commit;
next test.

**Ping-pong:** human writes a failing test; agent makes it pass and writes the next failing
test; human makes it pass and writes the next failing test; repeat.

**Testing strategy:** every test written in this phase follows the project's documented
testing strategy surfaced in Explore — the appropriate level and kind of test for the layer
being changed, required patterns, forbidden idioms, and correct file location. When no
strategy is documented, default to the lowest-level (fastest, most isolated) test that can
prove the behaviour.

After each green: make a checkpoint commit. These are working checkpoints — cheap, reversible,
and (under the default *Checkpoint + curate* granularity) reshaped into logical commits at
Phase 6 hand-off. Don't agonize over the message mid-loop; a one-line description naming the
behaviour just proved is enough. Every mistake is reversible; use that guarantee freely.

Confirm each test *genuinely* fails before implementing. A test that already passes means
you are about to build code that exercises nothing and verify nothing. **For a bug fix:**
genuine red means the test reproduces the reported defect — the same wrong behaviour the
report describes. If the first test already passes, the bug was not reproduced; revisit
the reproduction steps before writing any fix.

**What to defer to a human:**
- Calling the escape hatch: if test-first genuinely doesn't fit (pure UI spike, throwaway
  exploration), say so explicitly and switch to the agentic manual testing path from Phase 5.
  Never write fake always-passing tests. Never skip verification.
- In Assert-in-the-loop and Ping-pong modes: the human writes the parts that encode what
  correctness means. The agent implements; the human judges.

---

## Phase 5: Verify

Run the full test suite and confirm all tests pass (no regressions against the Phase 2
baseline).

Then perform agentic manual testing: drive the running app against each acceptance criterion
from Phase 1.

For each criterion:
- Execute the interaction
- Record the command **and its actual output** — not what you hoped would happen
- For UI: take screenshots and use vision capabilities to evaluate visual appearance

Any issues found: fix them and, where appropriate, add a permanent automated test for the
regression so it can't return silently.

If your project has a `run`, `verify`, or equivalent skill configured, delegate to it.
Otherwise use the inline agentic manual testing approach above.

**What to defer to a human:** UX quality, visual design correctness, and "does this feel
right" judgments. The agent verifies functional correctness against explicit criteria; aesthetic
and ergonomic quality is always a human call.

---

## Phase 6: Hand Off

**Curate the commit history** (skip this step under *One commit per criterion* or *Keep every
checkpoint* granularity — those are already the intended shape):

Under the default *Checkpoint + curate* granularity, your branch carries many small working
checkpoints. Before the walkthrough, reshape them into a small set of logical, behaviour-level
commits that tell a clear story to a reviewer:

1. Propose a grouping — one commit per behaviour / acceptance-criterion cluster. Show the
   proposed groups to the human and wait for approval.
2. Reshape using non-interactive git only (never rely on `git rebase -i` — it needs a TTY and
   editor):
   ```
   git reset --soft <merge-base>    # un-commit all checkpoints, keep changes staged
   git commit -m "…"                # recommit as first logical group
   git commit -m "…"                # …and so on per group
   ```
   Path-level grouping (`git add <paths>` per commit) is reliable non-interactively. Hunk-level
   splitting (one file contributing to two logical commits) requires `git add -p` — defer that to
   the human if needed.
3. **Never rewrite history that is already pushed or shared.** Reshape only local, un-pushed
   checkpoints. If the branch was already pushed, do not force-push without explicit approval.
4. Commit message format follows the project's convention — do not prescribe one here.

**What to defer to a human (history curation):** approval of the final commit grouping and
messages; any force-push of a shared remote branch.

Produce a *linear walkthrough*: a structured, sequential narrative of what changed and why,
in the order a reviewer would walk through the diff. Break the change into digestible steps.
Turn a large or unfamiliar diff into something reviewable.

Review your own code before presenting it. The anti-pattern to avoid is opening a PR with
agent-produced code you haven't read — that delegates the actual work to your collaborators.
Reading your own diff is the job, not an optional extra.

If a `code-review` skill is configured in the project, delegate to it for the structured
review. Otherwise, apply an inline checklist:
- Functional correctness against the Phase 1 acceptance criteria
- Test coverage appropriate for the change type
- Security boundaries (no unsanitized input, no hardcoded secrets, no overly broad permissions)
- Naming consistency and no dead code introduced

Prepare the PR/MR description from the linear walkthrough and the Phase 1 spec. If your
project defines a PR template or posting mechanics, follow them.

**What to defer to a human:** the final review, approval, and merge decision. The agent
prepares and narrates; a human decides whether to ship.

---

## Core Principles

1. **Code is cheap; good code is not.** Delivering new code is near-free; delivering
   secure, maintainable, correct code remains expensive. Those savings belong in spec
   and verification — not skipped steps.

2. **Aim higher, not just faster.** Shipping lower-quality code faster with agents is a
   deliberate choice — and the wrong one. If adopting agentic practices is reducing quality,
   address that directly rather than accepting it.

3. **First run the tests.** Before touching any production code, run the full test suite
   to locate the harness, gauge the project's scale, and establish a testing-first bias
   for the session.

4. **Red before green.** Write one test, confirm it genuinely fails, then implement.
   Skipping the red phase risks tests that always pass and code that exercises nothing.

5. **Git is a safety net — use it actively.** Branch before the first line of code.
   Commit checkpoints after each green — then reshape those checkpoints into a few logical
   commits before hand-off, so a big feature reads as several meaningful commits, not one
   and not thirty. Every mistake is reversible; use that guarantee freely.

6. **Never inflict unreviewed code.** Opening a PR with agent-produced lines you haven't
   read delegates the actual work to your collaborators. Review your own diff before asking
   anyone else to.
