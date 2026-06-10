# feature-development

End-to-end feature development methodology for Claude Code. Six phases: frame the
requirement, explore and establish a green baseline, plan, implement with a configurable
test-first loop, verify by driving the running app, and hand off for review.

> **Status:** v0.1.0 — initial release. The six-phase methodology is distilled from
> established agentic engineering practice. The three test-collaboration modes are new
> and pending validation across real projects. Feedback welcome.

## What it does

The skill encodes a complete feature development workflow:

| Phase | What happens |
|-------|-------------|
| **Frame** | Read the ticket/spec; write acceptance criteria before touching code |
| **Explore & Baseline** | Parallel Explore subagents map the codebase; run the full suite to establish a green baseline |
| **Plan** | Implementation steps as red/green pairs (Prove → Implement); branch created; hard human checkpoint |
| **Implement** | Test-first entry gate; test-first loop (one test at a time); follows project testing strategy; checkpoint commits |
| **Verify** | Full suite + agent drives the running app against acceptance criteria |
| **Hand Off** | Curate checkpoints into logical commits; linear walkthrough; self-review before presenting |

## Test collaboration modes

Configured during Setup. Three modes are available:

### Solo AI (default)

The agent runs the full red/green loop: write one failing test → confirm genuine red →
implement minimum to pass → green → commit. Repeats until all acceptance criteria are
covered. Includes an escape hatch for genuinely untestable work (reroutes to agentic
manual testing instead of faking a passing test).

### Assert-in-the-loop

The agent scaffolds one test (arrange + act + an empty assertion placeholder). **You
write the assertions** — that's where correctness is encoded. The agent then confirms a
genuine red (fixing only test scaffolding if the failure is a compile error, never
touching production code), implements to green, and scaffolds the next test.

Use this when you want to stay in control of what "correct" means for each test without
writing the full test yourself.

### Ping-pong pairing

Alternating authorship. You write a failing test → agent makes it pass and writes the
next failing test → you make it pass and write the next failing test → repeat.

Use this when you want to stay deeply involved in the implementation and enjoy the
discipline of writing tests.

### Ask each feature

No fixed mode — at the start of each feature's implementation the skill asks which of the
three modes to use. Good when you want a different strategy on different features.

**Invariant across all modes:** exactly one test at a time. No batch-generating 2–10 tests
up front.

## Setup

Run `set up feature-development` in Claude Code or Claude.ai Cowork. The Setup wizard:

1. Reads your existing project rules (`.claude/rules/`, `CLAUDE.md`) — does not re-ask
   build, test, or lint commands already configured there.
2. Detects whether a `code-review` or `run`/`verify` skill is present.
3. Asks only skill-specific questions: test collaboration mode, branch convention,
   spec artifact location, and commit granularity.
4. Writes `.claude/rules/feature-development.md` with only your non-default choices.

If you accept all defaults, no rules file is needed.

## Usage

Just describe what you want to build:

```
develop a feature: add email validation to the sign-up form
```

or reference a ticket:

```
work ticket AF-412
```

The skill walks through all six phases, pausing for your explicit approval before
implementing (Phase 3 checkpoint) and before merging (Phase 6 hand-off).

## Extension points

Project-specific configuration lives in `.claude/rules/feature-development.md` in your
consuming project (written by Setup). Configurable:

- Test collaboration mode
- Commit granularity (default: checkpoint while building, curate into logical commits at hand-off)
- Branch naming convention
- Spec artifact location
- Review delegation (auto-detected from existing rules)

Build, test, lint, and run commands are **not** configured here — the skill uses whatever
your project already has in `CLAUDE.md` or `.claude/rules/`.

## What it does not do

- Merge to the default branch (always a human decision)
- Bypass the Phase 3 human checkpoint
- Skip verification when test-first doesn't fit (switches to manual testing instead)
- Write tests that always pass (escape hatch routes to agentic manual testing)
