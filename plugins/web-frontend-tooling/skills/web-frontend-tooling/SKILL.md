---
name: web-frontend-tooling
description: "Set up code quality tooling — linting, formatting, type-checking, Git hooks, and dead-code detection — in a Node/TypeScript project. Use when asked to add or configure linting, formatting, pre-commit/pre-push hooks, husky, lint-staged, oxlint, oxfmt, ESLint, Prettier, knip, or to wire up code quality control for a frontend/web project. Prefers the Oxc toolchain but adapts to an existing ESLint/Prettier setup. Also use when asked to set up, configure, or onboard this skill."
---

# Web Frontend Tooling

> A blueprint for configuring linting, formatting, type-checking, and Git hooks in a Node/TypeScript project.
>
> This skill tells you which tools to install, how they wire together, and what bare-minimum config looks like. It does **not** impose specific lint/format rules — pick your own, or start from the minimal sets below and adjust. The version numbers shown are reference points, not requirements; resolve the latest stable versions at install time.

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. Read all existing project rules (`.claude/rules/`, `CLAUDE.md`) to understand what is already configured. Do not duplicate package-manager, build, or test commands — those are project-level concerns.
2. Inspect the project for existing tooling: `package.json` (scripts + devDependencies + `packageManager` field), lockfiles (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`), `.eslintrc*`, `.prettierrc*`, `.oxlintrc.json`, `.oxfmtrc.json`, `.husky/`, `.lintstagedrc*`, `tsconfig.json`, `knip.json`. Detect the package manager from the lockfile / `packageManager` field (see Step 0) and use it for every command — do not assume npm.
3. Present interactive choices for each skill-specific decision:
   - **Linter** — oxlint (preferred) or keep/adopt ESLint. Detect which is present.
   - **Formatter** — oxfmt (preferred) or keep/adopt Prettier. Detect which is present.
   - **Hook split** — pre-commit (format staged) + pre-push (type-check + lint) is the default. Offer to also lint staged files at commit time (heavier pre-commit).
   - **Dead-code detection** — include knip as an on-demand/CI script (recommended) or skip.
   - **Config reconciliation** — when the existing codebase deviates from the starting configs, choose per concern: change the config to match the code, or reformat/fix the code to match the config.
4. Write `.claude/rules/web-frontend-tooling.md` containing only the user's choices that deviate from the defaults below. If all defaults are accepted, confirm no rules file is needed and stop.

**What to defer to a human:** The reconcile-vs-reformat decision when an existing codebase deviates from the starting config — never silently impose a style on existing code. Always present both options and let the human pick per project.

---

## Tool Stack Overview

| Concern              | Tool          | Config file          |
| -------------------- | ------------- | -------------------- |
| Linter               | oxlint        | `.oxlintrc.json`     |
| Formatter            | oxfmt         | `.oxfmtrc.json`      |
| Git hook manager     | husky         | `.husky/`            |
| Staged-file runner   | lint-staged   | `.lintstagedrc.json` |
| Type checker         | tsc           | `tsconfig.json`      |
| Unused-code finder   | knip          | `knip.json` (opt.)   |

Flow:

```
git commit  ──► husky pre-commit  ──► lint-staged ──► oxfmt (staged files only)
git push    ──► husky pre-push    ──► tsc --noEmit && oxlint
```

---

## Step 0 — Detect the package manager

Do **not** assume npm. Detect the project's package manager and use it for every
install/run/dlx command below. Detect from, in order of precedence:

1. Lockfile: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb`/`bun.lock` → bun,
   `package-lock.json` → npm.
2. `package.json` `"packageManager"` field (Corepack), e.g. `"pnpm@9.x"`.
3. If none found, ask the user; default to npm only if they have no preference.

Command equivalents — substitute `<install>`, `<run>`, and `<dlx>` everywhere in this
guide with the row matching the detected manager:

| Manager | `<install>` (add dev dep) | `<run>` (run script) | `<dlx>` (run binary)  |
| ------- | ------------------------- | -------------------- | --------------------- |
| npm     | `npm install --save-dev`  | `npm run`            | `npx`                 |
| pnpm    | `pnpm add --save-dev`     | `pnpm run`           | `pnpm dlx`            |
| yarn    | `yarn add --dev`          | `yarn run`           | `yarn dlx`            |
| bun     | `bun add --dev`           | `bun run`            | `bunx`                |

**Extension point:** The package manager. The skill follows whatever the codebase
already uses. The examples below are written with the npm form for readability —
translate them via the table above.

---

## Step 1 — Choosing Linter & Formatter

This skill **prefers the Oxc toolchain** (`oxlint` + `oxfmt`) — Rust-based, fast,
zero-config-friendly. But it does not force it. Decide before installing:

### Linter

- **No linter found in target repo** → ask: install `oxlint` or `eslint`?
  Prefer `oxlint`.
- **`eslint` already present** → ask: switch to `oxlint`?
  - Switch → remove `eslint`, `.eslintrc*`, `eslint-*` plugins; install `oxlint`.
  - Keep → skip linter section, keep existing ESLint config.

### Formatter

- **No formatter found** → ask: install `oxfmt` or `prettier`?
  Prefer `oxfmt`.
- **`prettier` already present** → ask: switch to `oxfmt`?
  - Switch → remove `prettier`, `.prettierrc*`, `prettier-*` plugins; install `oxfmt`.
  - Keep → skip formatter section, keep existing Prettier config.

> The rest of this guide assumes the Oxc choice. If you keep ESLint/Prettier,
> the husky + lint-staged wiring is identical — only the command names change.

**Extension point:** The project's chosen linter and formatter. If the project rules
record a choice, follow it. Otherwise default to the Oxc toolchain and ask before
removing any existing tool.

---

## Step 2 — Install

```bash
<install> oxlint oxfmt husky lint-staged knip typescript
```

Drop `typescript` if the project already depends on it (most do).

Pin versions for reproducibility (pin exact versions for tooling deps — no `^`).
Resolve the latest stable version of each at install time; the numbers below are a
reference snapshot:

```jsonc
"devDependencies": {
  "husky": "9.1.7",
  "lint-staged": "17.0.6",
  "oxfmt": "0.52.0",
  "oxlint": "1.67.0",
  "typescript": "6.0.3"
}
```

---

## Step 3 — package.json scripts

```jsonc
"scripts": {
  "lint": "oxlint --max-warnings 0",
  "format": "oxfmt",
  "format:check": "oxfmt --check",
  "type-check": "tsc --noEmit",
  "unused-code": "knip",
  "prepare": "husky"
}
```

- `prepare` runs automatically on install (`<install>` / `npm install`) → installs husky hooks.
- `--max-warnings 0` makes any warning fail CI / the pre-push hook.
- `unused-code` runs knip → reports unused files, exports, deps (see Step 8).

---

## Step 4 — Formatter config (`.oxfmtrc.json`)

Bare-minimum, opinion-light starting point:

```json
{
  "$schema": "./node_modules/oxfmt/configuration_schema.json",
  "tabWidth": 2,
  "singleQuote": false,
  "semi": true,
  "trailingComma": "all",
  "printWidth": 110
}
```

These are oxfmt's Prettier-compatible keys (a subset — oxfmt does not yet support
every Prettier option). The bundled `$schema` validates the file against the
installed oxfmt version. If a key is rejected, check the
[oxfmt config reference](https://oxc.rs/docs/guide/usage/formatter/config) for the
options the installed version supports.

**If the target codebase deviates** from these defaults (e.g. it uses single
quotes, no semicolons, 80-col width), present both choices — do not pick silently:

1. **Change the config** to match the existing codebase (lowest churn).
2. **Change the codebase** by running the formatter once and committing the
   reformat (one big diff, then consistent forever).

Pick per project. Do not silently impose these defaults on an existing codebase.

**What to defer to a human:** Which of the two reconciliation options to take. The
churn-vs-consistency trade-off is a project call.

---

## Step 5 — Linter config (`.oxlintrc.json`)

Bare-minimum starting point. Enable the `correctness` category as errors, wire up
plugins for your framework, ignore generated/vendored paths:

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["typescript", "import"],
  "categories": {
    "correctness": "error"
  },
  "ignorePatterns": [
    "node_modules/**",
    "dist/**",
    "build/**",
    "coverage/**",
    "*.min.js"
  ],
  "rules": {
    "no-unused-vars": "off",
    "typescript/no-unused-vars": [
      "error",
      {
        "argsIgnorePattern": "^_",
        "varsIgnorePattern": "^_",
        "caughtErrorsIgnorePattern": "^_"
      }
    ],
    "typescript/no-explicit-any": "warn",
    "no-empty": ["error", { "allowEmptyCatch": false }],
    "no-debugger": "error",
    "eqeqeq": ["error", "always", { "null": "ignore" }],
    "no-var": "error",
    "prefer-const": "error"
  },
  "overrides": [
    {
      "files": ["tests/**/*.ts", "tests/**/*.tsx"],
      "rules": {
        "typescript/no-explicit-any": "off",
        "typescript/no-unused-vars": "off",
        "no-empty": "off"
      }
    }
  ]
}
```

The config above is framework-agnostic — `typescript` + `import` is the floor, and
the ignore paths cover only the common build/vendor dirs. **Detect the project's
framework / build system and adapt** before writing the file:

- **Next.js** (`next.config.*`) → add the `nextjs` + `react` plugins; ignore `.next/**`
  and `next-env.d.ts`.
- **Vite** (`vite.config.*`) → ignore `dist/**` (already listed); add `react` if a
  React plugin is present, or the relevant framework plugin (Vue, Solid, etc.).
- **React without a meta-framework** → add the `react` plugin.
- **Plain Node / library** → keep just `typescript` + `import`.

Then adjust:

- **`ignorePatterns`** — match the detected build output and any generated/vendored dirs.
- **`overrides`** — loosen rules for tests, generated code, etc.
- The specific `rules` above are a suggestion. This guide does not impose them.
  **If the codebase deviates**, present the same two options as the formatter:
  relax the rule, or fix the codebase.

**Extension point:** Framework plugins, ignore paths, and rule set. Detect the build
system from its config file and adapt; record project-specific deviations in the
companion rules file.

---

## Step 6 — Husky hooks

Initialize once:

```bash
<dlx> husky init
```

This creates `.husky/` and adds the `prepare` script. Then write the two hooks,
substituting the detected manager's `<dlx>` / `<run>` form (Step 0) — do not
hardcode npm into a project that uses another manager:

**`.husky/pre-commit`**

```sh
<dlx> lint-staged
```

**`.husky/pre-push`**

```sh
<run> type-check && <run> lint
```

> Hook bodies run literal commands — there is no placeholder resolution at runtime,
> so write the resolved form. For pnpm: `pnpm dlx lint-staged`, then
> `pnpm run type-check && pnpm run lint`. For npm: `npx lint-staged`, then
> `npm run type-check && npm run lint`.

Make them executable (husky usually handles this):

```bash
chmod +x .husky/pre-commit .husky/pre-push
```

Rationale for the split:

- **pre-commit** → fast, scoped to staged files (format only). Keeps commits clean
  without blocking on a full lint/typecheck.
- **pre-push** → slower, whole-repo (type-check + lint). Catches errors before they
  reach the remote, but doesn't slow down every commit.

**Extension point:** The hook split. The default is format-on-commit + typecheck/lint-on-push.
A project may move linting into pre-commit (see Step 7) — follow the project rules if set.

---

## Step 7 — lint-staged config (`.lintstagedrc.json`)

```json
{
  "*": "oxfmt --no-error-on-unmatched-pattern"
}
```

- `"*"` → run on every staged file; oxfmt ignores files it can't format.
- `--no-error-on-unmatched-pattern` → don't fail when a staged file isn't a
  formattable type (e.g. an image, a `.lock` file).

If you keep Prettier instead, the equivalent is:

```json
{ "*": "prettier --write --ignore-unknown" }
```

To also lint staged files at commit time (optional — heavier pre-commit):

```json
{
  "*.{ts,tsx,js,jsx}": ["oxlint --fix", "oxfmt"],
  "*": "oxfmt --no-error-on-unmatched-pattern"
}
```

---

## Step 8 — Unused-code finder (knip)

[knip](https://knip.dev) finds dead code: unused files, exports, types, and
dependencies. Catches what the linter can't — cross-file dead code and stale
`package.json` deps. Recommended for keeping a codebase lean.

Script (added in Step 3):

```jsonc
"scripts": {
  "unused-code": "knip"
}
```

Run on demand:

```bash
<run> unused-code
```

Config is optional — knip auto-detects most setups. Add `knip.json` only to tune
entry points or ignore false positives:

```json
{
  "$schema": "https://unpkg.com/knip@latest/schema.json",
  "entry": ["src/index.ts", "scripts/*.ts"],
  "project": ["src/**/*.ts", "src/**/*.tsx"],
  "ignore": ["**/*.d.ts"],
  "ignoreDependencies": []
}
```

Usage notes:

- Run **manually** or in **CI**, not in a Git hook — full-project scan is too slow
  for pre-commit/pre-push.
- `knip --fix` auto-removes some unused exports/files. Review the diff before
  committing — it can be aggressive.
- Treat findings as suggestions, not hard errors; some "unused" exports are public
  API or dynamically referenced.

**What to defer to a human:** Whether a knip "unused" finding is truly dead code or
intentional public API / dynamically-referenced code. Never auto-delete on knip's
word alone — review the diff.

---

## Replication Checklist

- [ ] Detect package manager (lockfile / `packageManager` field; ask if none)
- [ ] Decide linter (oxlint preferred; ask if eslint present)
- [ ] Decide formatter (oxfmt preferred; ask if prettier present)
- [ ] `<install> oxlint oxfmt husky lint-staged knip`
- [ ] Add `lint` / `format` / `format:check` / `type-check` / `unused-code` / `prepare` scripts
- [ ] Add `.oxfmtrc.json` (reconcile with existing code style)
- [ ] Add `.oxlintrc.json` (reconcile rules + ignore paths with project)
- [ ] `<dlx> husky init`
- [ ] Write `.husky/pre-commit` → `<dlx> lint-staged`
- [ ] Write `.husky/pre-push` → `<run> type-check && <run> lint`
- [ ] Add `.lintstagedrc.json`
- [ ] (optional) Add `knip.json`; run `<run> unused-code` to find dead code
- [ ] Verify: stage a file → commit (formats) → push (type-checks + lints)
