# dep-update-merge

Bundles open dependency-update PRs/MRs into a single verified change with changelog analysis, breaking change triage, and full build/test verification.

## What it does

1. **Phase 1: Discovery** — Lists open PRs/MRs, filters to dependency updates by author and title patterns, and asks the user to confirm which to include.

2. **Phase 2: Changelog Analysis** — Reads changelogs and release notes for each dependency's version range. Classifies updates as patch, minor, or breaking. Produces a summary table and flags updates with breaking changes, deprecations, or migration guides.

3. **Phase 3: Breaking Change Triage** — If breaking changes are found, presents them and asks whether to include all, exclude breaking updates, or exclude specific ones. Skipped if no breaking changes were detected.

4. **Phase 4: Branch Preparation** — Creates a combined branch from the default branch by applying selected updates in alphabetical order. Reports any conflicts without silently resolving them.

5. **Phase 5: Build & Test Verification** — Runs build, tests, and lint on the combined branch. Compares warnings against the default branch baseline. Reports new warnings and fails clearly on build or test failure.

6. **Phase 6: Completion & Output** — Produces a summary report of bundled updates, deferred updates with reasons, and verification results. Creates the final PR/MR if configured, otherwise leaves the branch ready for manual submission.

## Companion rules file (recommended)

The skill is fully generic and works out of the box for Phases 1–4. Phases 5 and 6 require project configuration to be effective — without build and test commands, verification cannot run.

Set up your project's rules with:

```
Use the dep-update-merge skill to configure this project
```

Or create `.claude/rules/dep-update-merge.md` manually. Example:

```markdown
# Dependency Update Merge — Project Rules

## PR/MR Discovery

Use `gh pr list --label dependencies --state open --json number,title,headRefName,author` to list dependency-update PRs. Filter to PRs authored by `dependabot[bot]` or `renovate[bot]`.

## Build Command

npm run build

## Test Command

npm test

## Lint Command

npm run lint

## Warning Patterns

Scan for lines matching: `warning TS`, `DeprecationWarning`, `ExperimentalWarning`.

## Completion Action

Create the PR with:
gh pr create \
  --title "chore: bundle dependency updates" \
  --label dependencies \
  --body "Automated bundle of dependency updates. See the skill report for changelog summaries and verification results."
```

### GitLab example

```markdown
## PR/MR Discovery

Use `glab mr list --state opened --label dependencies` to list open dependency MRs.

## Completion Action

glab mr create \
  --title "chore: bundle dependency updates" \
  --label dependencies \
  --description "Automated bundle of dependency updates."
```

## Usage

### Claude Code (terminal)

```
Use the dep-update-merge skill to bundle open dependency PRs
```

Or after configuring your rules file:

```
Bundle the dependency updates
Merge the open Dependabot PRs
```

### Claude.ai Cowork

Install via your organization's plugin settings, then ask:

```
Bundle and verify the open dependency update PRs
```
