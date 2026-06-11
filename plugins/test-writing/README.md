# test-writing

> **Status: skeleton.** The plugin structure and extension points are in place; the methodology content is still being written.

Test-writing skill based on Robert C. Martin's (Uncle Bob) testing discipline:

- **The Three Laws of TDD** and the red-green-refactor cycle — *Clean Craftsmanship: Disciplines, Standards, and Ethics* (2021) and the [Clean Coder blog](https://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html)
- **The Test Automation Pyramid** — *The Clean Coder* (2011), ch. 8 "Testing Strategies"
- **F.I.R.S.T. and clean tests** — *Clean Code* (2008), ch. 9 "Unit Tests"

## What it does

Decides what to test, how many tests to write, and at which level of the test pyramid — then writes them. Covers three modes:

1. **TDD** — tests drive new production code (Three Laws, red-green-refactor)
2. **New/changed code** — tests accompanying a feature, bug fix, or diff
3. **Backfill** — characterization/regression tests for existing untested code

Six phases:

1. **Mode Selection** — TDD, new/changed code, or backfill
2. **Decide What to Test** — behavior over implementation
3. **Decide How Much and at What Level** — pyramid level and quantity proportionate to risk
4. **Write the Tests** — mode-specific discipline
5. **Test Quality Gate** — F.I.R.S.T. checklist
6. **Verify and Report** — what was tested, what wasn't, human checklist

## Companion rules file (recommended)

The SKILL.md contains the generic methodology. Test framework, runner commands, coverage expectations, and style conventions are deferred to your project. Run `set up test-writing` in your project to generate `.claude/rules/test-writing.md` interactively.

Project-level concerns (how to run the test suite, build commands) stay in your project's own `CLAUDE.md` / rules — the skill uses whatever is already configured.

## Usage

### Claude Code (terminal)

After installing via `/plugin install test-writing@agilefreaks-skills`:

```
Use the test-writing skill to add tests for the changes on this branch
```

### Claude.ai Cowork

Once the plugin is distributed to your org, use it from any Cowork project. See your org's Cowork plugin settings for availability.
