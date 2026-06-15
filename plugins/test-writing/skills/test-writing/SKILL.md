---
name: test-writing
description: "Write the right tests when required during development. Decides what is worth testing, at which level of the test pyramid, where to draw the test boundary (real owned persistence vs. mocked vendor services), in what order, then writes and refactors them clean. Based on Robert C. Martin's testing discipline (Three Laws of TDD, red-green-refactor, the Test Automation Pyramid, F.I.R.S.T.) and Simon Willison's agentic testing patterns (first run the tests, confirm red, never trust unexecuted code). Use when writing tests for new or changed code, doing TDD, backfilling tests for untested or legacy code, or improving test quality. Also use when asked to set up, configure, or onboard this skill."
---

# Test Writing Skill

> A methodology for writing the right tests as part of a development workflow.
> It does not run the workflow — it is invoked when tests are required.
>
> Built on Robert C. Martin's testing discipline — the Three Laws of TDD and
> red-green-refactor (*Clean Craftsmanship*, 2021), the Test Automation Pyramid
> (*The Clean Coder*, 2011, ch. 8), and F.I.R.S.T. / clean tests (*Clean Code*,
> 2008, ch. 9) — and Simon Willison's agentic testing patterns: first run the
> tests, confirm red before green, and never trust code that hasn't been executed.
>
> Test framework, runner commands, and project-specific conventions are deferred
> to your project's rules — this skill uses whatever the project has configured.

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. Read all existing project rules (`.claude/rules/`, `CLAUDE.md`) first. Do not
   duplicate test commands, framework choices, or conventions that already exist.
2. Inspect the project for testing indicators: test directories, test-framework
   dependencies in package manifests, CI test steps, coverage tooling, and the
   existing tests themselves (to read the prevailing style).
3. Present the user with interactive choices for each skill-specific decision:
   - **Test levels in use** — which pyramid levels the project maintains (unit,
     component, integration, system/E2E) and where each lives.
   - **Persistence vs. external services** — which datastores are owned (test for
     real) and which dependencies are vendor/third-party (mock at the HTTP layer).
   - **Coverage expectations** — what the project considers adequate per change type.
   - **Test style conventions** — naming, structure (AAA / Given-When-Then),
     one-concept-per-test policy.
   - **Test data strategy** — fixtures, factories, builders, and any sanctioned
     provisional setup pattern (see Phase 3).
4. Write `.claude/rules/test-writing.md` containing only the user's choices. Omit
   any decision where the user accepts the default — the skill's built-in behavior
   handles those.

If the user accepts all defaults, confirm that no rules file is needed and stop.

---

## Phase 1: Learn the project, then triage what's worth testing

Before writing anything, ground yourself in the project, then decide what
actually deserves a test.

### Learn the project first

- **Run the existing tests.** This proves tests exist, reveals how to run them,
  signals the size and complexity of the project (test count as a proxy), and
  surfaces the prevailing conventions. Use whatever command the project has
  configured — this skill does not define test commands.
- **Read the existing tests** to learn the local idiom: structure, naming, how
  setup/arrangement is done, what helpers/factories exist, which pyramid levels
  are actually maintained.
- **Classify the dependencies** into *owned persistence* (databases, caches,
  queues the application controls) and *external/vendor services* (third-party
  APIs you don't own). This drives the boundary decision in Phase 2.

Establish which situation you're in:

- **Established suite** → conform to the existing style, even where it doesn't
  fully match the principles below. Consistency with the surrounding tests beats
  imposing the ideal. Do not restyle what's there unasked.
- **Greenfield / no real suite** → apply the principles in this skill directly.

### Triage what's worth testing

For the change in front of you, decide whether tests are warranted and *which
behaviors* deserve them:

- Test **behavior and contracts**, not implementation structure — so tests
  survive refactoring. A test that knows about private methods or internal
  wiring breaks the moment that wiring changes, for no real defect.
- Prioritize behavior with the highest risk or blast radius, edge cases, and
  anything a regression would silently break.
- Skip tests that only restate the framework, assert trivial getters, or pin
  implementation detail with no behavioral meaning.

> **Extension point:** if the project defines coverage expectations per change
> type, follow them. Default: cover the meaningful behaviors of the change,
> proportionate to risk (Phase 2).

**What to defer to a human:** whether a behavior is important enough to test is a
judgment call at the margins — surface borderline cases rather than silently
deciding to skip them.

---

## Phase 2: Decide how much, at what level, and where to draw the boundary

### Level and quantity (the Test Automation Pyramid)

Choose the lowest level that can meaningfully exercise the behavior, and let the
quantity follow the risk:

- **Unit** — fast, isolated, the bulk of the suite. Prefer these.
- **Component / integration** — a slice of collaborating units; fewer.
- **System / E2E** — end-to-end through the running application; fewest, reserved
  for critical paths. Slow and brittle in volume.

Quantity is proportionate to blast radius, not uniform. A high-risk change
warrants more cases (edge conditions, failure modes); a low-risk one needs less.

### Where to draw the test boundary

- **Owned persistence is part of the system under test — do not mock it.** The
  database's behavior (queries, constraints, transactions, what was actually
  persisted) *is* application behavior. Exercise it for real against a test
  datastore. This is also what makes front-door arrangement (Phase 3, Phase 4)
  honest: a real `Create` writes a real row that a Read test reads back.
- **External/vendor services are mocked — at the lowest level possible.**
  Intercept the **HTTP request/response** (the wire boundary), not the client or
  SDK wrapper defined in your codebase. This keeps your own adapter code under
  test (serialization, headers, error/retry handling, response parsing) and
  decouples tests from the wrapper's interface, so refactoring the client does
  not shatter tests. Mocking the high-level client tests *implementation*;
  mocking HTTP tests *behavior*.

> **Extension point:** the exact boundary is project-specific — a team may use a
> vendor sandbox, contract tests, recorded HTTP cassettes, or an in-memory
> datastore. If the project's rules specify, follow them; the defaults above
> otherwise. Tool names are examples only — the project supplies the mechanism.

---

## Phase 3: Write the tests

### Adapt to the situation

The same discipline applies whether the production code exists yet or not:

- **Test-first (TDD / red-green-refactor):** write the test before the code,
  **confirm it fails for the right reason (red)**, then implement the minimum to
  make it pass (green). Confirming red is the step most easily skipped — without
  it you cannot tell a passing test from a test that never actually exercised the
  change. (Three Laws of TDD: write no production code until a failing test
  demands it; write no more of a test than is sufficient to fail; write no more
  production code than is sufficient to pass.)
- **New / changed code:** map each behavior identified in Phase 1 to a test that
  accompanies the diff.
- **Backfill / legacy:** write **characterization tests** that pin the *current*
  behavior before changing it. These describe what the code does, not what it
  should do — judging desirability comes later (and is a human call).

### Sequencing within a larger feature

When the component under test is one slice of a bigger feature:

1. **Prefer logical (dependency) order, if you control it.** Build slices so each
   one's setup can arrange state through APIs already implemented. In CRUD, that
   usually means **Create before Read/Update/Delete** — then Read's setup is just
   a real front-door `Create`, with no shortcuts.
2. **When you can't reorder** (the setup API doesn't exist yet — e.g. Read must
   come first), **isolate the temporary setup behind a single seam**: one
   clearly-named setup helper that uses whatever's available now (direct insert,
   fixture, factory), marked as provisional. Two requirements:
   - It is the **only place** the shortcut lives, so swapping to front-door
     arrangement when the real API lands is a one-spot refactor, not a sweep.
   - The test's **behavior assertions stay unchanged** — only the arrange step
     migrates to the front door later.

> **Extension point:** if the project has a convention for provisional setup
> seams (a factories/builders layer, a `setup_*` helper pattern), use it. Default:
> a single clearly-named helper per component.

Run the tests using whatever command the project has configured.

---

## Phase 4: Refactor and quality gate

Tests are first-class code. Once green, clean them before moving on.

- **Remove duplication and redundancy.** Extract repeated arrange/assert into
  builders, factories, or helpers. *Balance:* tests favor readability (DAMP) over
  aggressive DRY — pull out *incidental* duplication, but stop before a test no
  longer reads as a self-contained story.
- **Arrange through the front door.** Set up state via the system's public
  interface or domain factories, not by calling internal/private functions or
  inserting rows directly into the database. Arrangement, like assertion, should
  couple to behavior, not internals — so tests don't break on refactoring.
  - *Extension point:* if the project deliberately seeds state via direct
    DB/fixtures (for speed or hard-to-reach states) and its rules sanction it,
    follow them. Default is front-door arrangement.
- **Apply the F.I.R.S.T. gate** to every test:
  - **Fast** — milliseconds, or the suite gets abandoned.
  - **Independent** — no ordering or shared-state dependence between tests.
  - **Repeatable** — same result every run, any environment; no clocks, network,
    or randomness leaking in.
  - **Self-validating** — a clear pass/fail, no manual interpretation.
  - **Timely** — written with (ideally just before) the code they cover.
- **Clean-test rules:** one concept per test, descriptive names, no logic
  (conditionals/loops) in the test body.

### Improving an established suite incrementally

In a project with an existing idiom, raise quality by suggestion, not by force:

- Write *new* tests to the local convention so they fit in.
- Where a principle is being violated (arrangement reaching into DB/internals,
  heavy duplication, slow/flaky tests, testing implementation), **surface it as a
  small, scoped suggestion** on the tests you're already touching — not a silent
  rewrite or a suite-wide refactor. Let improvements compound so the suite drifts
  toward the principles without a disruptive big-bang change.

**What to defer to a human:** whether to adopt a non-ideal existing convention vs.
push an improvement — flag any larger style shift for sign-off rather than
deciding unilaterally.

---

## Phase 5: Verify and report

- **Never assume LLM-generated code works until it has been executed.** Passing
  automated tests is necessary but not sufficient — a test can be green and the
  feature still wrong. Actually run the thing: invoke the function, exercise the
  endpoint, run the CLI, drive the UI and look at the result. (Mechanism is
  project-specific — use whatever the project provides.)
- **Report** clearly:
  - what was tested, and at which pyramid level;
  - what was deliberately left untested, and why;
  - any provisional setup seams left behind (Phase 3) and the API that will let
    them migrate to the front door;
  - the human checklist below.

**What to defer to a human:**
- Judging whether the behavior pinned by characterization tests is the *desired*
  behavior (backfill mode pins what *is*, not what *should be*).
- Exploratory and acceptance testing of the running feature.
- Sign-off on any larger test-style change in an established suite.

> **Extension point:** if the project defines an output format for test work,
> follow it. Otherwise, structure the report with the sections above.

---

## Core Principles

1. **Tests are first-class code** — kept as clean as production code.
2. **Test behavior, not implementation** — at the assertion *and* the arrangement
   boundary; refactoring must not break tests.
3. **Owned persistence is behavior; vendor services are mocked at the wire** —
   test your database for real, intercept third-party HTTP at the lowest level.
4. **Coverage proportionate to risk** — quantity follows blast radius; prefer the
   lowest meaningful level of the pyramid.
5. **Confirm red, then green, then refactor** — a test that never failed proves
   nothing; execute before you trust.
6. **Match the project's idiom; improve it incrementally by suggestion** — never
   by silent wholesale rewrite.
7. **Defer honestly** — say what the tests do not prove.
