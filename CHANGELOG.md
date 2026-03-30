# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **dep-update-merge** plugin — 6-phase dependency update bundling skill. Discovers open dependency PRs/MRs, analyzes changelogs for breaking changes, offers to exclude breaking updates, creates a combined branch, runs build/test/lint verification with warning baseline comparison, and produces a verified bundle ready for human review.

### Changed

- **dep-update-merge** — Removed `onboarding.json` and `rules-template.md` (dead files — nothing in the consuming project reads them). Added interactive Setup section directly in SKILL.md. Removed build/test/lint extension points; those are project-level concerns sourced from existing CLAUDE.md or .claude/rules/. Simplified trigger to "set up dep-update-merge".
- **Onboarding wizard** — `.claude/rules/onboarding.md` defines an interactive setup experience for any plugin that ships `onboarding.json`. Inspects the consuming project to suggest project-aware defaults, walks through each extension point, and generates a companion rules file.
- **`onboarding.json`** for code-review plugin — declares all 5 extension points (task-location, build-verification, coding-conventions, posting-mechanics, output-format) with detect hints for project inspection.
- **`assets/rules-template.md`** for code-review plugin — template with `{{id}}` placeholders used by the wizard to generate the companion rules file.

### Changed

- `skill-authoring.md` — added `onboarding.json` authoring guidance: schema reference, `detect` hints, and rules template placeholder conventions.

## [1.0.0] - 2026-03-11

### Added

- **code-review** plugin — 6-phase outside-in risk-driven code review methodology based on Gregory Brown's *Effective Code Reviews*. Covers bug fixes, new features, add-ons, extensions, and refinements. Project-specific checks and posting mechanics configured via companion rules file.
- `.github/workflows/code-review.yml` — GitHub Actions template for automated PR reviews using `anthropics/claude-code-action@v1`.
