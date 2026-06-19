# Test Collaboration Modes

This reference defines the complete state machine for each of the three test collaboration
modes available in the `feature-development` skill. SKILL.md Phase 4 summarizes each mode;
this file encodes the precise rules the skill enforces.

## Contents

- [Universal invariants](#universal-invariants) (5 rules)
- [Mode 1: Solo AI (default)](#mode-1-solo-ai-default)
  - [State machine](#state-machine)
  - [Escape hatch](#escape-hatch)
  - [Examples](#solo-ai-examples)
- [Mode 2: Assert-in-the-loop](#mode-2-assert-in-the-loop)
  - [State machine](#assert-state-machine)
  - [Scaffold template](#scaffold-template)
  - [What counts as a genuine red](#what-counts-as-a-genuine-red)
  - [What the agent may and may not fix](#what-the-agent-may-and-may-not-fix)
  - [Examples](#assert-examples)
- [Mode 3: Ping-pong pairing](#mode-3-ping-pong-pairing)
  - [State machine](#ping-pong-state-machine)
  - [Turn structure](#turn-structure)
  - [Examples](#ping-pong-examples)
- [Commit cadence & history curation](#commit-cadence--history-curation)
- [Choosing a mode](#choosing-a-mode)
- [Switching modes mid-feature](#switching-modes-mid-feature)

---

## Universal invariants

These rules apply in every mode without exception:

1. **Exactly one test at a time.** Never generate, scaffold, or write 2–10 tests up front
   in a batch. One test is in flight at a time: write it → reach genuine red → implement →
   green → commit → next test.

2. **Confirm genuine red before implementing.** A test that already passes when first run
   is not a failing test — it is a test that verifies nothing. Discovering this means
   either the test is wrong (fix the test) or the feature already exists (re-examine the
   acceptance criteria). Either way, do not proceed to implementation until the test
   fails for the right reason. For a bug fix: genuine red additionally means the test
   reproduces the reported defect — the same wrong behaviour, not merely "feature not
   built yet." If the first test passes, the bug was not reproduced; revisit the
   reproduction steps.

3. **Checkpoint after every green.** After each test passes, make a git checkpoint commit.
   This keeps every mistake reversible. These are *working* checkpoints — the published shape
   of the history is set by the configured commit granularity (see
   [Commit cadence & history curation](#commit-cadence--history-curation)). Under the default
   *Checkpoint + curate*, they are reshaped into logical commits at Phase 6 hand-off.

4. **Never fake a passing test.** If test-first genuinely cannot apply (see escape hatch
   below), switch to the Phase 5 agentic manual testing path. Do not write an empty test,
   a trivially-always-passing assertion, or a `assertTrue(true)` stand-in.

5. **The test shape follows the project's testing strategy.** The one-test-at-a-time
   invariant governs *cadence*; the project's documented testing strategy (surfaced in
   Explore) governs *what kind* of test to write and *how* to write it — the appropriate
   test level for the layer being changed, required patterns, and forbidden idioms. When no
   strategy is documented, default to the lowest-level (fastest, most isolated) test that
   can prove the behaviour.

---

## Mode 1: Solo AI (default)

The agent runs the full red/green loop independently. No human input is required between
the start and the checkpoint commit for each test.

### State machine

```
START
  │
  ▼
[1] Write one failing test
    │  The test must correspond to exactly one acceptance criterion (or one
    │  well-bounded sub-behaviour). Write only the test — no production code yet.
    ▼
[2] Run the test
    │
    ├── PASSES on first run ──► The test is wrong or the feature already exists.
    │                            Fix the test so it fails, or revisit the acceptance
    │                            criteria. Return to [1].
    │
    └── FAILS ──► Continue to [3]
                  │
                  ├── Compile error / broken setup ──► Fix the test scaffolding
                  │                                    (imports, setup, test harness).
                  │                                    Do not touch production code.
                  │                                    Return to [2].
                  │
                  └── Assertion failure ──► GENUINE RED confirmed. Continue to [4].
    ▼
[4] Implement the minimum production code to make the test pass
    │  Minimum means: the smallest change that turns this specific test green.
    │  Do not implement more than the test requires.
    ▼
[5] Run the test
    │
    ├── FAILS ──► The implementation is wrong. Fix production code. Return to [5].
    │
    └── PASSES ──► Continue to [6]
    ▼
[6] Run the full suite (regression check)
    │
    ├── ANY EXISTING TEST NOW FAILS ──► Regression introduced. Fix production code
    │                                   without breaking the new test. Return to [6].
    │
    └── ALL PASS ──► Continue to [7]
    ▼
[7] Checkpoint commit
    │  Message: describes the behaviour proved by this test (not the file changed).
    │  Example: "add: email validation rejects addresses without @"
    ▼
[8] Is there another acceptance criterion not yet covered by tests?
    │
    ├── YES ──► Return to [1]
    │
    └── NO ──► Phase 4 complete. Proceed to Phase 5 (Verify).
```

### Escape hatch

The escape hatch is an explicit, named switch — not a silent skip. Use it only when
test-first genuinely cannot apply. Valid trigger conditions:

- **Pure UI spike**: exploring what a layout looks like before committing to an approach.
  The acceptance criterion is visual; the output is a screenshot reviewed by a human.
- **Throwaway exploration**: the code will be deleted. No permanent code is being written.
- **Test infrastructure does not yet exist**: the project has no test harness for this
  layer and building one exceeds the scope of this feature. (This is rare; usually the
  right answer is to add a thin test harness as part of the feature.)

When activating the escape hatch:

1. Name it explicitly in a comment or note: "Escape hatch: switching to manual testing
   for this criterion because [reason]."
2. Implement the code.
3. Switch to the Phase 5 agentic manual testing path for this criterion only.
4. Return to the normal loop for subsequent criteria if they are testable.

Do **not** use the escape hatch to avoid writing a test that is merely inconvenient to
write. Inconvenient tests are the tests most worth writing.

### Solo AI examples

**Good:** "Write a test that verifies `UserValidator.validate()` returns an error when the
email field is empty. Run it — it fails with `AssertionError: expected Error got null`.
Implement the empty-email check. Re-run — passes. Run the suite — all green. Commit:
`add: email validation rejects empty address`."

**Bad:** "Here are the five tests I'll implement for this feature: [...]" — violates
the one-test-at-a-time invariant.

**Bad:** "The test passes already — I'll proceed to the next one." — violates genuine-red
confirmation. Investigate why it passes before proceeding.

---

## Mode 2: Assert-in-the-loop

The agent scaffolds the test structure; the human writes the assertions. This keeps the
human in the loop on the part that encodes what correctness means, while delegating the
scaffolding and implementation to the agent.

### Assert state machine

```
START
  │
  ▼
[1] Agent writes one test scaffold:
    │  - arrange (setup: create objects, configure dependencies, set preconditions)
    │  - act (call the thing under test: one function call, one user action)
    │  - assertion placeholder: # YOUR ASSERTION HERE
    │
    │  The scaffold must be runnable (compiles, imports resolve, test harness wired up)
    │  before being handed to the human.
    ▼
[2] Human writes the assertion(s)
    │  The human fills in the # YOUR ASSERTION HERE placeholder with real assertions
    │  that encode what the correct result looks like.
    │  The human may add more than one assertion if a single behaviour has multiple
    │  observable properties, but they should all relate to the same behaviour.
    ▼
[3] Agent runs the test
    │
    ├── COMPILE ERROR or BROKEN SCAFFOLD ──► Agent fixes ONLY the test scaffolding.
    │    │  Permitted: fixing imports, fixing setup, fixing the harness wiring.
    │    │  Forbidden: touching production code. Forbidden: changing the assertions
    │    │  the human wrote. Return to [3].
    │    │
    │    │  If the scaffold cannot be made to compile without changing the assertions,
    │    │  surface the conflict to the human and ask them to revise the assertions.
    │
    ├── PASSES on first run ──► The test is wrong or the feature already exists.
    │                            Surface this to the human. Do not proceed to
    │                            implementation. Human decides: revise assertions or
    │                            investigate whether the feature already exists.
    │
    └── FAILS with ASSERTION FAILURE ──► GENUINE RED confirmed. Continue to [4].
    ▼
[4] Agent implements the minimum production code to make the test pass
    │  Same constraint as Solo AI: minimum means smallest change that turns this
    │  specific test green. Do not implement more than the test requires.
    ▼
[5] Agent runs the test
    │
    ├── FAILS ──► Fix production code. Return to [5].
    │
    └── PASSES ──► Continue to [6]
    ▼
[6] Agent runs the full suite (regression check)
    │
    ├── ANY EXISTING TEST NOW FAILS ──► Regression introduced. Fix production code.
    │                                   Return to [6].
    │
    └── ALL PASS ──► Continue to [7]
    ▼
[7] Checkpoint commit
    ▼
[8] More acceptance criteria?
    ├── YES ──► Return to [1]
    └── NO ──► Phase 4 complete. Proceed to Phase 5 (Verify).
```

### Scaffold template

The scaffold should follow this structure, adapted to the project's test language and
framework:

```
// [ARRANGE]
// Set up the objects, data, and dependencies needed for the test.
val subject = SomeClass(dependency = FakeDependency())
val input = validInput()

// [ACT]
// Call the single thing under test.
val result = subject.doTheThing(input)

// [ASSERT]
// YOUR ASSERTION HERE
```

The comment `// YOUR ASSERTION HERE` (or the language-appropriate equivalent) is the
signal to the human that this is their responsibility. Do not write `assert(true)`,
`// TODO`, or any other placeholder that could pass accidentally.

### What counts as a genuine red

A genuine red is a test failure caused by an **assertion failure** — the code ran to
completion, but the actual result did not match the expected result encoded by the human.

These are **not** genuine reds (fix the scaffold, not the production code):
- `CompilationError` — something doesn't compile
- `ClassNotFoundException`, `NoSuchMethodException` — test harness wiring issue
- `NullPointerException` in the arrange block — setup is broken
- The test framework reports the test as "skipped" or "ignored"
- The test runner errors before executing any test code

These **are** genuine reds (proceed to implementation):
- `AssertionError: expected X but was Y`
- `AssertionFailedError`
- Any assertion library failure (`shouldBe`, `assertEquals`, `expect`, etc.)
- An exception thrown by the production code (not the test infrastructure) when one
  was not expected, or no exception thrown when one was expected

### What the agent may and may not fix

| Situation | Agent may fix | Agent may NOT fix |
|---|---|---|
| Compile error in scaffold | ✅ Test imports, test class setup | ❌ Production code |
| Broken test harness | ✅ Test harness configuration | ❌ Production code |
| Assertion that passes immediately | ❌ Surface to human | ❌ Silently proceed |
| Assertion failure (genuine red) | ❌ Not yet | ✅ Production code in step [4] |
| Human's assertion is logically wrong | ❌ Surface to human | ❌ Silently rewrite it |

The critical rule: **the agent never modifies the assertions the human wrote.** The
human's assertion is the specification. If the agent disagrees with it, it surfaces the
disagreement rather than silently changing it.

### Assert examples

**Good scaffold:**
```python
# [ARRANGE]
validator = EmailValidator()
invalid_email = "notanemail"

# [ACT]
result = validator.validate(invalid_email)

# [ASSERT]
# YOUR ASSERTION HERE
```

**Human fills in:**
```python
assert result.is_valid == False
assert "invalid format" in result.error_message
```

**Agent runs it — AssertionError: `result` is None because `validate()` returns None.**
That's a genuine red (the production code is broken, not the test). Agent implements
`validate()` to return a `ValidationResult`. Green. Commit.

**Bad:** agent fills in `assert result is not None` before handing to human — that's
the agent writing the assertion. The human must write it.

---

## Mode 3: Ping-pong pairing

Alternating authorship: human and agent take turns holding the implementation pen. The
human always writes the test that defines what is correct; the agent always implements
to make the current test pass and writes the next test to keep the momentum going.

### Ping-pong state machine

```
START (human goes first)
  │
  ▼
[HUMAN TURN]
[1] Human writes a failing test
    │  The test must fail when run. The human is encoding what correctness means for
    │  one behaviour. Hand off to agent.
    ▼
[AGENT TURN]
[2] Agent runs the test — confirms it genuinely fails (assertion failure, not compile error)
    │  If compile error: agent fixes only the test scaffold and re-runs.
    │  If passes already: surface to human before proceeding.
    │
    └── GENUINE RED confirmed ──► Continue to [3]
    ▼
[3] Agent implements the minimum production code to make the test pass
    ▼
[4] Agent runs the test — confirms green
    ▼
[5] Agent runs the full suite — confirms no regressions
    ▼
[6] Agent makes a checkpoint commit
    ▼
[7] Agent writes the NEXT failing test (one test, genuinely fails)
    │  The agent encodes the next logical behaviour to implement, based on the
    │  acceptance criteria not yet covered. Hand off to human.
    ▼
[HUMAN TURN]
[8] Human runs the test — confirms it genuinely fails
    │  If the human disagrees with the agent's test: revise it. The human is
    │  authoritative on what is correct; if the agent's test encodes the wrong
    │  behaviour, the human changes it before proceeding.
    ▼
[9] Human implements the minimum production code to make the test pass
    ▼
[10] Human runs the test — confirms green
    ▼
[11] Human makes a checkpoint commit
    ▼
[12] Human writes the NEXT failing test
    │  Return to [AGENT TURN] → [2]
    │
    └── If no more acceptance criteria: Phase 4 complete. Proceed to Phase 5.
```

### Turn structure

Each turn has exactly the same shape:

1. **Receive** one failing test
2. **Confirm** it is a genuine red (not a compile error, not already green)
3. **Implement** the minimum to make it green
4. **Confirm** green + no regressions
5. **Commit** a checkpoint
6. **Write** the next failing test (one test) and hand off

Neither party skips steps. The agent does not implement more than the current test
requires. The human does not implement more than the current test requires.

### Turn ownership

| Who writes the assertion | Who implements the production code |
|---|---|
| Human (turn 1, 5, 9, …) | Agent (turn 2, 6, 10, …) |
| Agent (turn 3, 7, 11, …) | Human (turn 4, 8, 12, …) |

The agent's tests (odd turns after the first) should encode the next logical behaviour
toward the acceptance criteria. They should not be trivial (verifying something already
covered) or over-reaching (testing more than one new behaviour).

### Ping-pong examples

**Session start:**
> Human: writes `test_empty_cart_has_zero_total()` — fails because `Cart.total` is not
> implemented. Hands off.

> Agent: confirms genuine red. Implements `total` property returning 0. Green. Commits.
> Writes `test_adding_one_item_updates_total()` — fails because `add_item()` is not
> implemented. Hands off.

> Human: confirms genuine red. Implements `add_item()`. Green. Commits. Writes
> `test_adding_item_twice_doubles_quantity()` — fails. Hands off.

> … and so on until all acceptance criteria are covered.

**Disagreement:**
> Agent writes `test_discount_applies_before_tax()`.
> Human thinks tax applies before discount in this domain.
> Human revises the test to `test_tax_applies_before_discount()` and proceeds.
> The human is authoritative on domain correctness.

---

## Commit cadence & history curation

Checkpoint commits are the working safety net. The *published* shape of the commit history is
governed by the **Commit granularity** choice made in Setup:

| Granularity | What happens during Phase 4 | What happens at Phase 6 |
|---|---|---|
| **Checkpoint + curate** (default) | Commit after each green — cheap, one-liner message | Reshape into a few logical, behaviour-level commits before the walkthrough |
| **One commit per criterion** | Commit once per acceptance criterion (may cover 2–3 tests) | No reshape — commits are already the intended shape |
| **Keep every checkpoint** | Commit after each green, permanently | No reshape — full micro-history is preserved |

### Reshaping checkpoints (Checkpoint + curate only)

Use non-interactive git only — `git rebase -i` requires a TTY and editor; an agent cannot
drive it reliably.

**Recipe:**
```bash
# 1. Find the merge-base (where the feature branch diverged from the base branch)
git merge-base <base-branch> HEAD

# 2. Un-commit all checkpoints and unstage them (changes preserved in the working tree)
git reset <merge-base-hash>

# 3. Re-commit as logical groups (one commit per behaviour cluster)
git add <paths-for-group-1>
git commit -m "…"

git add <paths-for-group-2>
git commit -m "…"
# … and so on
```

**Granularity limits:**
- **Path-level grouping** (`git add <paths>`) — reliable non-interactively; use this.
- **Hunk-level splitting** (one file contributing to two logical commits) — requires
  `git add -p`; defer to the human if needed.

**Hard rules:**
- **Curate before the first push.** Reshaping rewrites local history — run this recipe before
  the branch is pushed. If checkpoints were already pushed, the rule below applies.
- Never rewrite history that is already pushed or shared. Reshape only local, un-pushed
  checkpoints. If the branch was already pushed, do not force-push without explicit human
  approval.
- Commit message format follows the project's convention — this reference does not prescribe
  one.

---

## Choosing a mode

| Mode | Best when |
|---|---|
| **Solo AI** | You want full velocity; tests are straightforward; you'll review in Phase 6 |
| **Assert-in-the-loop** | You want to stay in control of what "correct" means for each test, without writing the full test each time |
| **Ping-pong** | You want to stay deeply involved; you enjoy writing tests; the domain requires careful correctness decisions at every step |

All three modes produce the same intermediate output: a set of checkpoint commits, each
representing one green test, covering the acceptance criteria from Phase 1. Under the default
*Checkpoint + curate* granularity, those checkpoints are reshaped into a small set of logical,
behaviour-level commits at Phase 6 hand-off before presenting the work for review.

The mode can be fixed at setup (written to `.claude/rules/feature-development.md`) or
configured as *Ask each feature*, in which case the skill asks which mode to use at the
start of each feature's implementation. Once a mode is active for a feature, the
"Switching modes mid-feature" rules apply if a change is needed mid-way.

---

## Switching modes mid-feature

Modes can be switched between acceptance criteria (never mid-test-cycle). To switch:

1. Complete the current test cycle (reach green + commit).
2. Note the switch explicitly: "Switching from Solo AI to Assert-in-the-loop for the
   remaining criteria."
3. Proceed with the new mode from the next test.

Switching is reasonable when the nature of the remaining acceptance criteria changes
(e.g., the first few criteria were algorithmic and easily verified by the agent; the
remaining ones are domain-specific and the human wants to own the assertions).
