# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **dep-update-merge** plugin — 6-phase dependency update bundling skill. Discovers open dependency PRs/MRs, analyzes changelogs for breaking changes, offers to exclude breaking updates, creates a combined branch, runs build/test/lint verification with warning baseline comparison, and produces a verified bundle ready for human review.

### Changed

- **dep-update-merge** — Removed `onboarding.json` and `rules-template.md` (dead files — nothing in the consuming project reads them). Added interactive Setup section directly in SKILL.md. Removed build/test/lint extension points; those are project-level concerns sourced from existing CLAUDE.md or .claude/rules/. Simplified trigger to "set up dep-update-merge".
- **code-review** — Same cleanup: removed `onboarding.json` and `rules-template.md`, added interactive Setup section to SKILL.md.
- **skill-authoring.md** — Replaced `onboarding.json` / `rules-template.md` authoring guidance with Setup section guidance. Skills now embed setup instructions directly in SKILL.md.
- **CLAUDE.md** — Updated plugin onboarding section to reflect the new Setup-section-in-SKILL.md approach.

### Removed

- `.claude/rules/onboarding.md` — wizard behavior that only worked inside the skills repo, never reached consuming projects.
- `onboarding.json` and `assets/rules-template.md` from both code-review and dep-update-merge plugins — dead files nothing in consuming projects read.

## [1.0.1] - 2026-04-25

### Fixed

- **code-review** — `.github/workflows/code-review.yml` now grants `id-token: write` (required by `anthropics/claude-code-action@v1` to mint an OIDC token) and uses `claude_args: --allowedTools "..."` in place of the now-invalid `allowed_tools` input. Without these, the workflow failed at the OIDC step and silently ran with default tool permissions.

## [1.0.0] - 2026-03-11

### Added

- **code-review** plugin — 6-phase outside-in risk-driven code review methodology based on Gregory Brown's *Effective Code Reviews*. Covers bug fixes, new features, add-ons, extensions, and refinements. Project-specific checks and posting mechanics configured via companion rules file.
- `.github/workflows/code-review.yml` — GitHub Actions template for automated PR reviews using `anthropics/claude-code-action@v1`.
