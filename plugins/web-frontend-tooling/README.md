# web-frontend-tooling

A blueprint for wiring up code quality tooling — linting, formatting, type-checking, Git hooks, and dead-code detection — in a Node/TypeScript project.

## What it does

Walks through installing and wiring together a complete code quality control stack:

| Concern            | Preferred tool | Config file          |
| ------------------ | -------------- | -------------------- |
| Linter             | oxlint         | `.oxlintrc.json`     |
| Formatter          | oxfmt          | `.oxfmtrc.json`      |
| Git hook manager   | husky          | `.husky/`            |
| Staged-file runner | lint-staged    | `.lintstagedrc.json` |
| Type checker       | tsc            | `tsconfig.json`      |
| Unused-code finder | knip           | `knip.json` (opt.)   |

The wiring:

```
git commit  ──► husky pre-commit  ──► lint-staged ──► oxfmt (staged files only)
git push    ──► husky pre-push    ──► tsc --noEmit && oxlint
```

It **prefers the Oxc toolchain** (oxlint + oxfmt — Rust-based, fast) but adapts to an existing ESLint/Prettier setup: the husky + lint-staged wiring is identical, only the command names change.

It is a **blueprint, not a ruleset**. It supplies minimal, opinion-light starting configs and does not impose specific lint/format rules. When the target codebase deviates from the starting config, it presents two reconciliation options — change the config to match the code, or reformat the code to match the config — and lets you pick per project. It never silently imposes a style on existing code.

## Companion rules file (optional)

The skill works out of the box with no configuration — the Oxc toolchain and the format-on-commit / typecheck-lint-on-push hook split are the defaults. A companion rules file only records skill-specific deviations: a different linter/formatter choice, a custom hook split, framework plugins, or ignore paths.

Set up your project's rules with:

```
set up web-frontend-tooling
```

Or equivalently: "configure web-frontend-tooling", "onboard web-frontend-tooling".

## Usage

### Claude Code (terminal)

```
Set up linting and formatting for this project
Add husky pre-commit and pre-push hooks
Configure code quality tooling with oxlint and oxfmt
```

### Claude.ai Cowork

Install via your organization's plugin settings, then ask:

```
Wire up code quality tooling for this frontend project
```
