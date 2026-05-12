# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed

- **code-review** — Setup wizard not running for this repo: added `.claude/rules/code-review.md` with context gathering (linked GitHub issue → PR title/description fallback), build verification skipped (pure Markdown/JSON repo), and posting mechanics (inline comments via GitHub API for line-specific findings + summary via `gh pr review --comment`).

### Added

- **android-project-starter** plugin — wizard-driven scaffolder for Android projects. Interactive `/android-project-starter:init` runs 6 phased rounds (~20 questions), resolves the latest stable versions for every dependency in parallel, then generates a complete multi-module project: `build-logic/` with 8 convention plugins (or 7 without Room), `core/{common,data,model,designsystem-*,testing,ui-*}`, `feature/<name>/{data,ui-mobile,ui-tv?}` per requested feature with shared data + form-factor-specific MVI stacks, `app-mobile` (+ `app-tv` when TV is picked), `gradle/libs.versions.toml`, Spotless/detekt/lint/CI/Dependabot, an Android-shaped `.gitignore`, and project-local `<project>-android-planner` / `<project>-android-implementer` skills. Verifies the project builds (`./gradlew help`, `:app-mobile:dependencies`, `compileDebugKotlin`, `spotless+detekt+lint+test`) and sweeps for warnings before declaring done, then commits the scaffold as `setup architecture`. The generated implementer skill ships with `references/compose-authoring.md` distilled from [compose-expert](https://github.com/aldefy/compose-skill) covering state management, recomposition stability, modifier ordering, side-effect API selection, lazy-list discipline, animation, theming, Canvas safety, accessibility, TV focus rules, a production crash-pattern table, and deprecated patterns to avoid.
- **Release Notify** GitHub Actions workflow (`.github/workflows/release-notify.yml`) — posts published GitHub Releases to the Slack channel bound to `SLACK_RELEASE_WEBHOOK_URL`. See `.claude/rules/marketplace.md` ("Publishing a release").
- **dep-update-merge** plugin — 6-phase dependency update bundling skill. Discovers open dependency PRs/MRs, analyzes changelogs for breaking changes, offers to exclude breaking updates, creates a combined branch, runs build/test/lint verification with warning baseline comparison, and produces a verified bundle ready for human review.

### Changed

- **code-review** (1.1.0) — Setup wizard now detects GitHub Actions and offers to generate `.github/workflows/code-review.yml` (Opus by default). The workflow template is bundled as a plugin asset (`skills/code-review/assets/code-review.yml`), fixing two bugs in the previous standalone template: PR-number expression corrected from `github.event.workflow_run.pull_requests[0].number` (always empty under a `pull_request` trigger) to `github.event.pull_request.number`; dead `.skills` checkout step removed. Setup instructions now include `claude setup-token` for generating the required OAuth token.
- **dep-update-merge** — Removed `onboarding.json` and `rules-template.md` (dead files — nothing in the consuming project reads them). Added interactive Setup section directly in SKILL.md. Removed build/test/lint extension points; those are project-level concerns sourced from existing CLAUDE.md or .claude/rules/. Simplified trigger to "set up dep-update-merge".
- **dep-update-merge** — Phase 2 changelog discovery now falls back to a web search starting from the package's declared homepage before flagging "changelog unavailable". Catches packages whose release notes live on the author's website rather than in the source repo, forge, or registry. Bumped to 1.1.0.
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
