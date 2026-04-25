---
name: dep-update-merge
description: "Bundle and merge dependency update PRs/MRs into a single verified change. Use when asked to merge dependency updates, combine Dependabot or Renovate PRs, batch dependency bumps, update dependencies, or consolidate open dependency PRs. Also use when asked to check changelogs for dependency updates, verify that dependency bumps are safe, or create a bundled dependency update branch. Also use when asked to set up, configure, or onboard this skill."
---

# Dependency Update Merge Skill

> Bundles open dependency-update PRs/MRs into one verified change with changelog analysis and breaking change triage.
>
> Platform integration and project-specific checks are defined separately in your project's rules. Build, test, and lint commands are not configured here — the skill uses whatever your project already has in CLAUDE.md or .claude/rules/.

---

## Setup

When asked to set up, configure, onboard, or create a rules file for this skill:

1. Read all existing project rules (`.claude/rules/`, `CLAUDE.md`) to understand what is already configured. Do not duplicate build, test, or lint commands — those are project-level concerns that belong in the project's own configuration, not here.
2. Inspect the project for dependency bot configuration (`.github/dependabot.yml`, `renovate.json`, `.renovaterc`) and package manifests (`Gemfile`, `package.json`, `go.mod`, etc.).
3. Present the user with interactive choices for each skill-specific decision, using a choice dialog with options for each:
   - **PR/MR Discovery** — how to list and filter dependency-update PRs/MRs; suggest a concrete command based on the bot detected
   - **Changelog Retrieval** — how to find changelogs; suggest based on detected package ecosystem
   - **Completion Action** — how to submit the bundle (e.g., create PR via CLI, leave branch for manual submission)
4. Write `.claude/rules/dep-update-merge.md` containing only the user's choices. Omit any decision where the user accepts the default — the skill's built-in behavior handles those.

If the user accepts all defaults and no choices were made, confirm that no rules file is needed and stop.

---

## Phase 1: Discovery

List all open PRs/MRs and identify those that are dependency updates.

If your project defines how to list and filter dependency-update PRs/MRs (platform command, label filter, author filter), follow that. Otherwise, list all open PRs/MRs and identify dependency updates by examining titles and authors for common patterns: titles containing "bump", "update", "upgrade", or "chore(deps)", and authors matching common bot names such as "dependabot" or "renovate".

Present the discovered list to the user with: PR/MR number, title, source branch, dependency name, and version range (old → new). Ask the user to confirm which PRs/MRs to include before proceeding.

If no dependency-update PRs/MRs are found, report that and stop.

**What to defer to a human:** Deciding whether a PR/MR that looks like a dependency update but has non-standard formatting or an unexpected author should be included. When in doubt, ask rather than assume.

---

## Phase 2: Changelog Analysis

For each confirmed dependency, find and read the changelog or release notes covering the version range being applied.

If your project defines how to retrieve changelogs (registry URL convention, local vendored changelogs, changelog location pattern), follow that. Otherwise, attempt to find changelogs by:
- Looking for CHANGELOG.md, CHANGES.md, HISTORY.md, or RELEASES.md in the dependency's source repository
- Checking the forge's release page for the dependency
- Reading the registry page (npm, RubyGems, PyPI, crates.io, pkg.go.dev, etc.) for release notes
- As a last resort, performing a web search starting from the author's website or project homepage declared in the package manifest (e.g., `homepage`, `homepage_uri`, `Home-page`) to find release notes, blog posts, or doc-site changelogs published outside the registry

Classify each update as:
- **patch** — bug fixes only, no API changes
- **minor** — new features, backward-compatible
- **major / breaking** — breaking changes, API removals, required migration steps, behavior changes

Produce a summary table:

| Dependency | Old Version | New Version | Classification | Notable Changes |
|------------|-------------|-------------|----------------|-----------------|

Flag any update that contains deprecation notices, migration guides, required code changes, or sections explicitly labeled "breaking changes."

If a changelog cannot be found for a dependency, flag it as "changelog unavailable" and treat it as uncertain.

**What to defer to a human:** Verifying that a change labeled "non-breaking" is truly non-breaking for how this specific project uses that dependency. Changelogs are written by the dependency author and may understate impact. A human familiar with the codebase should review any flagged update.

---

## Phase 3: Breaking Change Triage

If any updates from Phase 2 were classified as breaking or flagged as uncertain, present them with their changelog summaries and ask the user how to proceed:

- **Include all** — bundle all updates, accepting the risk of breaking changes
- **Exclude breaking updates** — bundle only patch and minor updates; defer the rest
- **Exclude specific updates** — let the user name which ones to remove from the bundle

If the user chooses to exclude any updates, remove them from the working set and note them as "deferred — breaking changes detected" in the Phase 6 report.

If no breaking changes or uncertain updates were found in Phase 2, skip this phase entirely and proceed to Phase 4.

---

## Phase 4: Branch Preparation

Create a single working branch that combines all selected dependency updates.

1. Start from the project's default branch (main, master, or equivalent)
2. Create a new branch named `deps/bundled-updates` (or follow your project's branch naming convention)
3. Apply each selected update by merging or cherry-picking from its source branch
4. Apply updates in alphabetical order by dependency name for reproducibility

If conflicts arise between updates, stop and report them. Do not silently resolve conflicts — present the conflicting updates and ask whether to resolve manually, exclude one of the conflicting updates, or abort.

**What to defer to a human:** Resolving merge conflicts that require understanding of how two dependency updates interact. Some conflicts are mechanical; others require knowing which version's behavior the project needs.

---

## Phase 5: Build & Test Verification

Run the project's full verification suite on the combined branch and compare results against the default branch.

### Build and tests

Run the project's build, test, and lint commands as already configured in its rules, CLAUDE.md, or project configuration files. This skill does not define its own build or test commands — those are project-level concerns. Look for them in `.claude/rules/`, `CLAUDE.md`, and common project files (Makefile, package.json scripts, etc.).

If no build or test commands are configured anywhere in the project, report that Phase 5 requires project-level configuration (in CLAUDE.md or `.claude/rules/`) and stop. Do not ask the user to configure commands through this skill.

Capture the full output. Report a clear pass/fail.

If the build or tests fail:
- Report which step failed and the relevant error output
- Suggest options: exclude the update most likely responsible, investigate the failure, or abort
- Do not proceed to Phase 6 on failure

### Warning baseline comparison

After the build and tests pass, compare warnings against the default branch baseline. Scan build and test output for lines matching common warning patterns: "warning", "WARN", "deprecated", "deprecation notice".

Run the same commands on the default branch (or compare against a stored baseline if available). Report:
- New warnings introduced by the bundle
- Warnings resolved by the bundle
- Net change

If new warnings were introduced, report them and ask whether to proceed or investigate. The goal is zero new warnings, but the user decides the threshold.

**What to defer to a human:** Diagnosing build failures that require domain knowledge about the project's architecture or the dependency's internals. Evaluating whether a new warning is acceptable or indicates upcoming breakage.

---

## Phase 6: Completion & Output

Produce a summary report:

```
## Dependency Bundle: <branch-name>

### Bundled Updates
| Dependency | Old Version | New Version | Classification | Notes |
|------------|-------------|-------------|----------------|-------|

### Deferred Updates
| Dependency | Old Version | New Version | Reason Deferred |
|------------|-------------|-------------|-----------------|

### Verification
- Build: PASS / FAIL
- Tests: PASS / FAIL
- Lint: PASS / FAIL / SKIPPED
- New warnings: N

### Branch
<branch-name> — ready for review
```

If your project defines how to create the final PR/MR or how to post the output (platform command, PR template), follow that. Otherwise, output the report to stdout and leave the branch ready for the user to create a PR/MR manually.

**What to defer to a human:** The final decision to merge. This skill prepares and verifies the combined change but does not merge to the default branch. A human must review the bundle and approve the merge.

---

## Core Principles

1. **Safety first.** Never merge automatically. The skill prepares a verified bundle; a human decides to ship it.

2. **Transparency.** Every exclusion, conflict, warning, and failure is reported. Nothing is silently skipped or resolved.

3. **Platform-agnostic.** The methodology works regardless of forge or CI system. Platform specifics live in the companion rules file.

4. **Reproducible.** Alphabetical merge order and explicit conflict reporting mean the same inputs produce the same bundle.

5. **Conservative on breaking changes.** Default to offering exclusion of breaking updates rather than silently including them. The user can always opt in.
