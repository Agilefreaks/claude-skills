---
name: code-review
description: "Outside-in, risk-driven code review methodology. Use when reviewing pull requests, evaluating code changes, or conducting code reviews for any change type: bug fixes, new features, add-ons, extensions, and refinements. Also use when asked to assess code quality, review a diff, or provide feedback on proposed changes."
---

# Code Review Skill

> Outside-in, risk-driven code review methodology based on Gregory Brown's "Effective Code Reviews" (Programming Beyond Practices, 2015).
>
> Platform integration and project-specific checks are defined separately in your project's rules.

---

## Phase 1: Problem Validation

**Start here — before reading a single line of code.**

Ask two questions:
1. Does this code actually solve the problem it's meant to solve?
2. Will shipping this improvement increase the value of the project as a whole?

If the answer to either is no — **stop**. Don't review the code. Get in touch with whoever decides what gets built and talk it through. Reviewing code that won't survive in production is a waste of time.

To answer these questions: read the linked issue or ticket, supporting documentation, and any relevant discussion. If no issue exists, use the PR title and description as the source of truth. Check whether the diff plausibly addresses what was described.

**What to defer to a human:** Actually running the feature and verifying it works from the user's perspective. You can validate logical alignment between the stated problem and the diff; a human must verify it actually works.

---

## Phase 2: Build & Runability

- Is the build passing? If not, pause the review. A broken build can't be trusted. Even if the new change works correctly, other things could be broken. Don't assume a build failure is innocuous.
- Is the code running somewhere other than the author's machine? A production environment is ideal; a staging environment or another developer's machine is an acceptable smoke test.
- Did getting the code running require asking someone for help? If so, documentation is missing or setup needs to be automated — or both.

**What to defer to a human:** Running the code locally or in staging, trying out the feature from the perspective of the people who will use it, and raising any concerns about usability, error handling, or surprising behavior.

---

## Phase 3: Test Audit

**Modified or deleted tests are a red flag — investigate before anything else.**

When you see changes to existing tests, slow down. Ask: why were they modified? Understand the reason before forming any opinion.

Many modifications are relatively innocent — removing a test for behavior that was incorrect, changing test granularity, or replacing something this PR supersedes. None of that is inherently dangerous, but it requires thought. The key question is: **does this modification change a contract that other parts of the codebase depend on?**

- Changes to tests for isolated, self-contained code = relatively safe
- Changes to tests for shared or reused code = higher risk; needs explicit justification

If you can't answer the question from the diff alone, pause and ask the author. These conversations build shared understanding and surface mistaken assumptions before they cause real damage.

---

## Phase 4: Coverage Assessment

Categorize the change, then apply risk-appropriate expectations. A passing test suite is only as good as the behaviors it describes — you cannot automate the process of reviewing tests.

### Bug Fix

Every bug fix should be accompanied by regression tests. At minimum, one high-level acceptance test that follows the same path described in the bug report — mapping to actions a real user could take, not just a direct trigger of the internal failure.

Unit tests should augment acceptance tests for anything complex or severe, to provide finer-grained feedback when something breaks in the future.

Also look for the **canary-in-the-coal-mine pattern**: is this bug truly unique, or is it a sign of a deeper, more widespread issue? Check whether similar patterns exist elsewhere in the codebase. If they do, file tickets — you don't need to fix them all now, but surfacing them is a major contribution.

### New Feature

Apply the **zero-test thought experiment**: if *all* of the following conditions hold, minimal test coverage is acceptable:

1. The change consists almost entirely of new code, depending only on infrastructure, framework, libraries, and generic data access objects
2. The new functionality is isolated to its own area (its own page, namespace, or endpoint)
3. It does not introduce new system-wide side effects (no changed dependencies, configs, or load on shared resources)
4. It is not planned to be used as a building block for other functionality on the roadmap
5. It is not an intermediate step in a workflow chain that ties other features together
6. Failures cannot escape outward and cause instability elsewhere
7. Any conceivable failure would be acceptable to stakeholders and would not cause significant harm
8. Tests could be written later without unacceptable cost or delay

For each condition that *isn't* met, expect tests that specifically address that risk. A feature that's allowed to fail, fully isolated, and easy to test later is not a major risk. A feature that ties into existing workflows, can cause failures elsewhere, or will be built upon — is.

### Enhancement: Add-On

New code that lives alongside an existing feature. Apply the new feature criteria above, then go further: because it lives next to existing functionality, **failure isolation is the primary concern**.

Can a failure in the add-on take down the existing feature? Think through worst-case failure scenarios. Are edge cases and boundary conditions covered?

### Enhancement: Extension

Builds on existing behavior by reusing existing components. Before evaluating the new code, conduct a mini-audit of what's being reused:

- Do the reused components have tests?
- Are they documented at the business and/or code level?
- Were they designed with reuse in mind, or are they being stretched beyond their original purpose?
- Have they appeared in recent or unresolved bug reports?
- How likely are they to change? If they do change, will it break this extension?

You don't need to spend more than a few minutes on this — it's a thought experiment, not a deep investigation. But known and calculated risks are always better than surprises. File tickets for anything worth tracking.

### Enhancement: Refinement

Modifies existing system behavior. **Highest risk category.** Start by applying everything from the extension review above. Then go further.

Find every piece of functionality that depends on the code being modified. Each dependent is potentially affected. Ideally you'd run each through the full review process, but in practice that's rarely feasible. At minimum: identify the dependents, check whether their tests still hold, and flag the full list for the human reviewer to test manually.

---

## Phase 5: Project-Specific Checks

Apply your project's coding conventions, architecture patterns, and style rules to the diff. Check for violations of whatever standards the project has established.

If your project has defined platform-specific posting mechanics (how to post a review, format for inline comments, etc.), follow them in the next phase.

---

## Phase 6: Risk Assessment & Output

**Assign a risk level:**
- **Low** — isolated change, no shared code touched, tests are clean and appropriate for the change type
- **Medium** — touches existing behavior or shared code, but well-tested and the dependencies are understood
- **High** — modifies shared components with dependents, tests changed or missing for the change type, security or auth involved

**Produce a review summary** that includes:
- Context: what the PR does, what problem it solves
- What looks good
- Concerns found (both general and line-level where applicable)
- Risk level and reasoning

**Produce a human reviewer checklist** that includes only relevant items:
- Problem validation (always) — have a human verify it actually works, not just that the code looks aligned with the issue
- Build and functional verification (always) — run the feature, try it from a user perspective
- Anything explicitly deferred in Phases 1–4: testing refinement dependents, verifying add-on failure isolation, trying out the feature in a real environment
- Project-specific items where relevant: resetting projectors, testing integration points, checking authorization flows

Omit checklist sections that don't apply to this specific change. A config-only change doesn't need event sourcing verification. A documentation update doesn't need performance testing.

If a project output format is defined, follow it. Otherwise, structure the output clearly with the sections above.

---

## Core Principles

1. **Outside-in.** Validate the problem before the code. Validate behavior before style. How much code have you read by the end of Phase 2? In a well-organized project, possibly none — yet you've done quite a bit to uphold software quality standards.

2. **Risk-proportionate.** The depth of review should match the blast radius of the change. A trivial isolated addition doesn't warrant the same scrutiny as a refinement to a shared domain model.

3. **Defer honestly.** If you can't verify something — runtime behavior, user experience, whether a dependent feature still works — say so explicitly and put it in the human checklist. Don't imply you've checked what you haven't.

4. **Stop on blockers.** A broken build, a change that solves the wrong problem, or a fundamental design concern is a reason to stop and flag, not continue reviewing code that may not survive.

5. **Ask when uncertain.** Pausing a review to ask the author a question is always the right move when something is unclear. These conversations build shared understanding and catch mistaken assumptions before they cause damage.
