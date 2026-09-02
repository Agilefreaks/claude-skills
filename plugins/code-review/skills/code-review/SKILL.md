---
name: code-review
description: "Outside-in, risk-driven code review methodology. Use when reviewing pull requests, evaluating code changes, or conducting code reviews for any change type: bug fixes, new features, add-ons, extensions, and refinements. Also use when asked to assess code quality, review a diff, or provide feedback on proposed changes. Also use when asked to re-review a pull request or follow up on a previous review. Also use when asked to set up, configure, or onboard this skill."
---

# Code Review Skill

> Outside-in, risk-driven code review methodology based on Gregory Brown's "Effective Code Reviews" (Programming Beyond Practices, 2015).
>
> Platform integration and project-specific checks are defined separately in your project's rules.

---

## Recording findings — applies from Phase 0 onward

**Record every finding with a severity and a confidence, from the first phase.** Do not narrow the search while reviewing: a concern you decline to write down in an early phase cannot be recovered in Phase 6. Filtering happens once, in the summary, where a human can see what was filtered and disagree with it.

- **Severity** — `blocker` (must not ship as-is: breaks the build, solves the wrong problem, introduces a security hole, or removes a contract other code depends on), `important` (should be addressed before merge), `nice-to-have` (worth noting, safe to defer).
- **Confidence** — `high` (verified in the diff or the surrounding code), `medium` (consistent with what the diff shows, not confirmed), `low` (a question worth asking, not yet a claim). Say plainly what you did not check rather than raising the confidence to match the concern.

This applies to inline comments as much as to the summary: an inline finding carries the same two tags.

**Write each finding as claim, evidence, consequence — and stop there.**

- **Claim** — one bold sentence naming what is wrong. A reader who reads only this line knows the problem.
- **Evidence** — the file, the line, the quoted text or output that shows it.
- **Consequence** — what breaks, who hits it, or what it costs to leave.

**If you cannot state a consequence, you have found a style preference, not a defect.** Drop it, or let the project's linters and conventions carry it. Brevity makes proofreading findings cheap to write, and a review made of cheap findings is the mediocre review this methodology exists to avoid.

Leave out how you found it, what you ruled out on the way, and where the project solves this elsewhere unless that *is* the fix. If the author wants the reasoning they will ask, and the thread is the place for it — a review is a conversation, not a report.

This is about prose, not coverage. It never changes how many findings you record or how many reach the author; where the two seem to pull against each other, the rule above wins.

This shape is the default. If a project has set a review detail level (see Setup), follow it: at *Standard*, keep the same three parts and add the reasoning behind the finding. The level changes how much prose a finding gets, never how many findings there are.

---

## Phase 0: Prior Review State (re-reviews)

**Before reviewing, find out what has already been said.**

Gather the existing review conversation on this change:
- All inline discussion threads, including replies and their resolved/unresolved state — yours and human reviewers' alike.
- Your own previous review summary, if one exists (identified by its hidden marker — see Phase 6).

If your project defines platform posting mechanics, use them to fetch this state. If no platform mechanics are configured (for example, the review goes to stdout), treat this as a first review and note in the summary that prior review state could not be checked.

If no prior review state exists, this is a first review — proceed to Phase 1 and skip the re-review steps in Phase 6.

**Read every thread; act only on your own.** Human reviewers' threads inform the review — never raise a concern a human reviewer has already raised, whether or not it was answered. Identify your own threads by the hidden marker embedded in each finding you post (see Phase 6), never by the posting account — the account differs between CI and local runs.

For each of your own prior findings, classify it and carry the disposition into the rest of the review:

1. **Fix pushed** — the author changed code in response. Verify the fix in the current diff during the review. Actually fixed: reply in the thread confirming it (for example, "Verified fixed in <revision>") and resolve the thread. Claimed but not fixed: reply explaining what is still missing and leave the thread open.
2. **Answered** — the author explained or disputed the finding. Their response is authoritative; do not re-raise. Reply acknowledging it and resolve the thread.
3. **Still open** — no response, no fix. Do not post a duplicate comment. Leave the thread untouched and list it under "Previous findings" in the new summary.

Genuinely new findings from this review are posted as usual in Phase 6.

If your project configures a re-review thread handling mode (see Setup), follow it — some teams reserve thread resolution for humans. The default is reply-and-resolve.

**What to defer to a human:** Adjudicating disputes — when an author pushes back, this skill stands down; if the disputed point still matters, a human reviewer must make the call. Whether an automated reviewer may resolve threads at all is a team convention — humans own it (configure via Setup).

---

## Phase 1: Problem Validation

**Start here — before reading a single line of code.**

Ask two questions:
1. Does this code actually solve the problem it's meant to solve?
2. Will shipping this improvement increase the value of the project as a whole?

If the answer to either is no — **stop**. Don't review the code. Get in touch with whoever decides what gets built and talk it through. Reviewing code that won't survive in production is a waste of time.

To answer these questions: read the linked issue or ticket, supporting documentation, and any relevant discussion. If your project defines how to locate the relevant issue or ticket (task management integration, branch naming conventions), follow that. If no issue exists or no integration is configured, use the PR title and description as the source of truth. Check whether the diff plausibly addresses what was described.

**What to defer to a human:** Actually running the feature and verifying it works from the user's perspective. You can validate logical alignment between the stated problem and the diff; a human must verify it actually works.

---

## Phase 2: Build & Runability

- Is the build passing? If not, pause the review. A broken build can't be trusted. Even if the new change works correctly, other things could be broken. Don't assume a build failure is innocuous.
- Is the code running somewhere other than the author's machine? A production environment is ideal; a staging environment or another developer's machine is an acceptable smoke test.
- Did getting the code running require asking someone for help? If so, documentation is missing or setup needs to be automated — or both.

If your project defines how to verify build status (CI system, status checks), use that. Otherwise, check for any available build indicators in the change context.

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

If your project defines coding conventions, architecture patterns, or style rules, apply them to the diff and check for violations.

If no project conventions are defined, check for: inconsistent naming within the diff, dead code introduced by the change, obvious duplication, missing error handling at system boundaries, and security concerns (unsanitized input, hardcoded secrets, overly broad permissions).

If your project has defined platform-specific posting mechanics (how to post a review, format for inline comments, etc.), follow them in the next phase.

---

## Phase 6: Risk Assessment & Output

**Assign a risk level:**
- **Low** — isolated change, no shared code touched, tests are clean and appropriate for the change type
- **Medium** — touches existing behavior or shared code, but well-tested and the dependencies are understood
- **High** — modifies shared components with dependents, tests changed or missing for the change type, security or auth involved

**Filter here, not earlier.** Every finding recorded in Phases 0–5 reaches this point; decide now which ones reach the author. A low-confidence blocker is still worth raising; a low-confidence nice-to-have usually is not. Lead the concerns with the blockers.

**Produce a review summary.** Four sections are always present, in this order:

1. **Verdict** — one or two sentences: can this merge, and what is the one thing standing in the way.
2. **Concerns** — the findings that reach the author, blockers first, each in the claim/evidence/consequence shape. Tag both general and line-level findings.
3. **Risk** — the level, always. Add reasoning only where the concerns don't already show it.
4. **For a human** — the checklist below. It is never omitted; it is the only place the review states what it did not verify.

Write the concerns first and the verdict from them. A verdict written first becomes a target the concerns get trimmed to fit.

Add either of the following only where it tells the reader something those four do not:

- **Context** — when the change's purpose is not evident from its title and diff.
- **What was checked** — only the checks whose result would change the merge decision, one line each. Not a log of everything you looked at.

On a re-review, add **Previous findings** immediately after the verdict: one line per prior finding — *resolved* (verified fixed), *answered* (author response accepted), or *still open* — plus a few words. Not a restatement of the finding. First-time reviews omit it.

**Produce a human reviewer checklist.** One standing line — this review did not run the code — then one line for each thing you could not verify: a deferral you made in Phases 1–4, a claim you could not check, a behaviour only a person can judge. Nothing else. If you deferred nothing beyond the standing line, say so and move on.

Do not restate a concern that already appears above. A finding and a deferral are different things: a finding is something you found, a deferral is something you could not look at.

**Maintain exactly one summary per change.** Post the summary as a standalone comment that begins with a hidden marker (default: `<!-- code-review-summary -->`) and that your platform allows you to later delete. On a re-review: find your previous summary by its marker, delete it, and post the fresh summary — never accumulate summaries. Mark each inline finding the same way (default: `<!-- code-review-finding -->` as the first line of the comment body) so a future re-review can recognize its own threads. On a re-review, also execute the thread dispositions from Phase 0 (in-thread replies and resolutions, per the configured thread handling mode) when posting.

If a project defines an output format or a review detail level, follow it. Otherwise use the shape above. If your project defines posting mechanics, follow them for how to fetch, reply to, resolve, delete, and post comments on your platform.

Write it for a reviewer who did not watch you work: name the file and the behaviour rather than referring to a finding by number, and spell out an acronym the first time it appears.

An example of the whole shape, for a first review of a small bug fix:

```
**Verdict** — Safe to merge once the regression test covers the reported path. The fix is
right; the test proves something narrower than the bug report describes.

**Concerns**

`[important]` `[confidence: high]` **The regression test bypasses the reported failure path.**
`spec/signup_spec.rb:42` asserts the validation error on the model. The bug report describes a
blank email on the signup form rendering a 500.
A future change to the controller re-opens this bug with the suite green.

`[nice-to-have]` `[confidence: medium]` **The blank-email guard has no sibling for invalid emails.**
`User#validate_email` handles `nil` but not malformed input, and the same form accepts both.
If this bug arrived by that route, the invalid case is still open — worth a ticket either way.

**Risk** — Low. Isolated to signup validation; no shared code touched.

**For a human**
- This review did not run the code.
- Nothing else deferred.
```

---

## Core Principles

1. **Outside-in.** Validate the problem before the code. Validate behavior before style. How much code have you read by the end of Phase 2? In a well-organized project, possibly none — yet you've done quite a bit to uphold software quality standards.

2. **Risk-proportionate.** The depth of review should match the blast radius of the change. A trivial isolated addition doesn't warrant the same scrutiny as a refinement to a shared domain model.

3. **Defer honestly.** If you can't verify something — runtime behavior, user experience, whether a dependent feature still works — say so explicitly and put it in the human checklist. Don't imply you've checked what you haven't.

4. **Stop on blockers.** A broken build, a change that solves the wrong problem, or a fundamental design concern is a reason to stop and flag, not continue reviewing code that may not survive.

5. **Ask when uncertain.** Pausing a review to ask the author a question is always the right move when something is unclear. These conversations build shared understanding and catch mistaken assumptions before they cause damage.

6. **One conversation per finding.** A finding that already has a thread is continued in that thread — verified, acknowledged, or left open — never re-posted. Accumulating duplicate comments erodes trust in the review.

7. **Report what you find, filter afterwards.** Narrowing the search while reviewing loses real findings; narrowing the summary does not. Record everything with a severity and a confidence, then decide in Phase 6 what reaches the author.

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. Read all existing project rules (`.claude/rules/`, `CLAUDE.md`) to understand what is already configured. Do not duplicate conventions, commands, or checks that already exist in the project's configuration.
2. Inspect the project for issue/PR linking conventions, CI configuration, linter/formatter configs, and code review platform indicators.
3. Present the user with interactive choices for each skill-specific decision, using a choice dialog with options for each:
   - **Context Gathering** — how to locate the relevant issue or ticket for a PR (branch naming, PR body keywords, task management integration)
   - **Build Verification** — how to verify the build is passing (CI status checks, specific commands)
   - **Coding Conventions** — project-specific checks beyond the skill's defaults (architecture rules, style rules already enforced by linters)
   - **Output Format** — custom structure for the review output, or use the built-in format
   - **Review Detail** — how much prose each finding gets: *Brief* (default — claim, evidence, consequence), or *Standard* (adds the reasoning behind each finding, for teams who review asynchronously and want fewer round trips). This dials prose only; neither setting changes which findings are reported.
   - **Posting Mechanics** — how to post the review (GitHub PR comment, inline comments, stdout)
   - **Re-review Thread Handling** — on a re-review, what to do with your own prior finding threads: *Reply and resolve* (default — reply in-thread and resolve threads that are fixed or answered), *Reply only* (reply in-thread but leave resolution to humans), or *Summary only* (never touch prior threads; report their status only in the summary)
   - **CI Integration** — offer to generate a GitHub Actions workflow that runs this review automatically on every PR. Always ask this, even if no `.github/workflows/` directory exists yet — this may be the project's first workflow. Present model choices: *Opus (recommended)* / *Sonnet* / *Fable* / *Skip*. Default: Opus. When presenting Fable, say what the trade is: the strongest reviewer available, at roughly double the per-token cost, and its safety classifiers can decline a security-heavy diff — which surfaces as a failed CI run rather than a review. Pass model **aliases**, not dated identifiers, so the workflow tracks the current release instead of pinning a retired one.
4. Write `.claude/rules/code-review.md` containing only the user's choices. Omit any decision where the user accepts the default — the skill's built-in behavior handles those.
5. If the user opted in to CI Integration:
   a. Read `assets/code-review.yml` (bundled with this skill) as the workflow template.
   b. Substitute `--model opus` with `--model <chosen-model>` if the user chose a different model.
   c. If `.github/workflows/code-review.yml` already exists in the project, show a diff and ask for confirmation before overwriting.
   d. Write the workflow to `.github/workflows/code-review.yml` (create `.github/workflows/` if it doesn't exist).
   e. Remind the user to add `CLAUDE_CODE_OAUTH_TOKEN` as a repository secret:
      - Generate the token locally with: `claude setup-token`
      - Add it at: **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**

**What to defer to a human (CI Integration):** The user must add the `CLAUDE_CODE_OAUTH_TOKEN` secret and verify that branch protection rules allow the action to post review comments. Setup cannot check or configure those.

If the user accepts all defaults and no choices were made, confirm that no rules file is needed and stop.
