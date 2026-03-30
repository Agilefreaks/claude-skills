# dep-update-merge

Bundles open dependency-update PRs/MRs into a single verified change with changelog analysis, breaking change triage, and full build/test verification.

## What it does

1. **Phase 1: Discovery** — Lists open PRs/MRs, filters to dependency updates by author and title patterns, and asks the user to confirm which to include.

2. **Phase 2: Changelog Analysis** — Reads changelogs and release notes for each dependency's version range. Classifies updates as patch, minor, or breaking. Produces a summary table and flags updates with breaking changes, deprecations, or migration guides.

3. **Phase 3: Breaking Change Triage** — If breaking changes are found, presents them and asks whether to include all, exclude breaking updates, or exclude specific ones. Skipped if no breaking changes were detected.

4. **Phase 4: Branch Preparation** — Creates a combined branch from the default branch by applying selected updates in alphabetical order. Reports any conflicts without silently resolving them.

5. **Phase 5: Build & Test Verification** — Runs the project's build, tests, and lint using whatever commands are already configured in the project's CLAUDE.md or rules files. Compares warnings against the default branch baseline. Reports new warnings and fails clearly on build or test failure.

6. **Phase 6: Completion & Output** — Produces a summary report of bundled updates, deferred updates with reasons, and verification results. Creates the final PR/MR if configured, otherwise leaves the branch ready for manual submission.

## Companion rules file (recommended)

The skill works out of the box with no configuration. Phase 5 uses whatever build, test, and lint commands your project already has in `CLAUDE.md` or `.claude/rules/` — do not add those here. The companion rules file only needs skill-specific settings: how to discover dependency PRs and how to submit the bundle.

Set up your project's rules with:

```
set up dep-update-merge
```

Or equivalently: "configure dep-update-merge", "onboard dep-update-merge".

Or create `.claude/rules/dep-update-merge.md` manually. Example:

```markdown
# Dependency Update Merge — Project Rules

## PR/MR Discovery

Use `gh pr list --author dependabot[bot] --state open --json number,title,headRefName` to list dependency-update PRs.

## Changelog Retrieval

Check RubyGems.org and GitHub release pages for changelogs covering each gem's version range.

## Completion Action

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
Bundle the dependency updates
Merge the open Dependabot PRs
```

To configure the skill first:

```
set up dep-update-merge
```

### Claude.ai Cowork

Install via your organization's plugin settings, then ask:

```
Bundle and verify the open dependency update PRs
```
