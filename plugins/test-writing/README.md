# test-writing

Write the right tests when required during a development workflow. Based on two sources:

- **Robert C. Martin's (Uncle Bob) testing discipline** — the Three Laws of TDD and the red-green-refactor cycle (*Clean Craftsmanship*, 2021 and the [Clean Coder blog](https://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html)), the Test Automation Pyramid (*The Clean Coder*, 2011, ch. 8), and F.I.R.S.T. / clean tests (*Clean Code*, 2008, ch. 9)
- **Simon Willison's agentic testing patterns** — [first run the tests, red/green TDD (confirm red), and agentic manual testing](https://simonwillison.net/guides/agentic-engineering-patterns/): never trust code that hasn't been executed

## What it does

Decides what is worth testing, at which level of the pyramid, where to draw the test boundary, and in what order — then writes the tests and refactors them clean. It is invoked as part of a development workflow; it does not run the workflow itself.

Highlights:

- **Learns the project first** — conforms to an established suite's idiom; applies the principles directly on greenfield. Improves legacy suites incrementally, by suggestion.
- **Draws the test boundary deliberately** — owned persistence (your database) is treated as application behavior and tested for real; external/vendor services are mocked at the lowest level (HTTP requests, not codebase clients).
- **Sequences within a feature** — prefers logical (dependency) order so setup uses real front-door APIs; when forced out of order, isolates provisional setup behind a single swappable seam.
- **Verifies for real** — passing tests aren't enough; the code is actually executed before it's trusted.

Five phases:

1. **Learn the project & triage what's worth testing** — run/read existing tests, classify dependencies, decide which behaviors deserve coverage
2. **Decide how much, at what level, and where to draw the boundary** — pyramid level + real-persistence vs. HTTP-mocked vendors
3. **Write the tests** — TDD (confirm red), new/changed, or backfill; sequenced within the feature
4. **Refactor & quality gate** — remove duplication, arrange through the front door, F.I.R.S.T.
5. **Verify & report** — never trust unexecuted code; report what was and wasn't tested

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
