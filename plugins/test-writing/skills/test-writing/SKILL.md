---
name: test-writing
description: "Test-writing methodology based on Robert C. Martin's (Uncle Bob) testing discipline. Decides what to test, how many tests to write, and at which level of the test pyramid. Use when writing tests for new or changed code, doing TDD (red-green-refactor), backfilling tests for untested or legacy code, or when asked to improve test coverage or test quality. Also use when asked to set up, configure, or onboard this skill."
---

# Test Writing Skill

> Test-writing methodology based on Robert C. Martin's testing discipline:
> the Three Laws of TDD and red-green-refactor (*Clean Craftsmanship*, 2021),
> the Test Automation Pyramid (*The Clean Coder*, 2011, ch. 8 "Testing Strategies"),
> and F.I.R.S.T. / clean tests (*Clean Code*, 2008, ch. 9 "Unit Tests").
>
> Test framework, runner commands, and project-specific testing conventions are
> defined separately in your project's rules.

<!-- TODO: This skill is a skeleton. Each phase below names its purpose and
     extension points; the methodology content is still to be written. -->

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. Read all existing project rules (`.claude/rules/`, `CLAUDE.md`) to understand what is already configured. Do not duplicate test commands, framework choices, or conventions that already exist in the project's configuration.
2. Inspect the project for testing indicators: test directories, test framework dependencies in package manifests, CI test steps, coverage tooling configuration.
3. Present the user with interactive choices for each skill-specific decision:
   <!-- TODO: finalize the decision list once the phases are written. Candidates: -->
   - **Test levels in use** — which pyramid levels the project actually maintains (unit, component, integration, system/E2E) and where each lives
   - **Coverage expectations** — what the project considers adequate per change type
   - **Test style conventions** — naming, structure (e.g. Arrange-Act-Assert / Given-When-Then), one-concept-per-test policy
   - **Test data strategy** — fixtures, factories, builders — what the project uses
4. Write `.claude/rules/test-writing.md` containing only the user's choices. Omit any decision where the user accepts the default — the skill's built-in behavior handles those.

If the user accepts all defaults and no choices were made, confirm that no rules file is needed and stop.

---

## Phase 1: Mode Selection

<!-- TODO: write the decision logic for picking a mode. -->

Determine which mode applies before writing anything:

- **TDD** — the production code doesn't exist yet; tests drive it (Three Laws, red-green-refactor)
- **New/changed code** — code was just written or modified; tests accompany the diff
- **Backfill** — existing untested code needs characterization/regression coverage before it can be safely changed

> Extension point: if your project defines when each mode applies (e.g. TDD mandated for certain layers), follow that. Default: infer the mode from the state of the code and the user's request.

---

## Phase 2: Decide What to Test

<!-- TODO: distill Uncle Bob's guidance on testing behavior (contracts) rather
     than implementation structure, so tests don't break on refactoring. -->

**What to defer to a human:** <!-- TODO: define once the phase is written. -->

---

## Phase 3: Decide How Much and at What Level

<!-- TODO: distill the Test Automation Pyramid (The Clean Coder ch. 8) into
     guidance for choosing the level (unit / component / integration / system)
     and the quantity of tests proportionate to the risk of the change. -->

> Extension point: project coverage expectations and which pyramid levels the project maintains. Default: <!-- TODO -->.

---

## Phase 4: Write the Tests

<!-- TODO: per-mode instructions.
     - TDD: the Three Laws, red-green-refactor cycle granularity
     - New/changed code: map each behavior identified in Phase 2 to a test
     - Backfill: characterization tests — pin current behavior before judging it -->

Run the tests using whatever test commands the project has configured (`CLAUDE.md`, project rules, or CI config). This skill does not define test commands.

> Extension point: test style conventions (naming, structure, test data strategy). Default: follow the conventions visible in the project's existing tests.

---

## Phase 5: Test Quality Gate

<!-- TODO: distill F.I.R.S.T. (Fast, Independent, Repeatable, Self-validating,
     Timely) and clean-test rules (Clean Code ch. 9) into a checklist applied
     to every test written in Phase 4. -->

---

## Phase 6: Verify and Report

<!-- TODO: define the output — what was tested, at which level, what was
     deliberately not tested and why, and the human checklist. -->

**What to defer to a human:** <!-- TODO: at minimum — judging whether the pinned behavior in backfill mode is the *desired* behavior, and exploratory testing of the running feature. -->

> Extension point: if a project output format is defined, follow it. Otherwise, structure the output clearly with the sections above.

---

## Core Principles

<!-- TODO: distill from the sources. Candidates:
     1. Tests are first-class code — kept as clean as production code.
     2. Test behavior, not implementation — refactoring must not break tests.
     3. Coverage proportionate to risk — quantity follows blast radius.
     4. The suite must be fast and trustworthy, or it will be abandoned.
     5. Defer honestly — say what the tests do not prove. -->
