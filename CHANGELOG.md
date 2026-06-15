# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **test-writing** plugin (0.1.0) — methodology for writing the right tests when required during a development workflow. Combines Robert C. Martin's testing discipline (the Three Laws of TDD and red-green-refactor — *Clean Craftsmanship*, 2021; the Test Automation Pyramid — *The Clean Coder*, 2011, ch. 8; F.I.R.S.T. / clean tests — *Clean Code*, 2008, ch. 9) with Simon Willison's agentic testing patterns (first run the tests, confirm red before green, never trust unexecuted code). Five phases: learn the project & triage what's worth testing; decide level and test boundary (owned persistence tested for real, vendor services mocked at the HTTP layer); write the tests (TDD/new/backfill, sequenced within a feature with swappable setup seams); refactor & quality gate (dedup, front-door arrangement, F.I.R.S.T.); verify by executing & report. Includes extension points and a Setup section for project-specific configuration.

## [1.2.0] - 2026-06-08

### Fixed

- **code-review** — Setup wizard not running for this repo: added `.claude/rules/code-review.md` with context gathering (linked GitHub issue → PR title/description fallback), build verification skipped (pure Markdown/JSON repo), and posting mechanics (inline comments via GitHub API for line-specific findings + summary via `gh pr review --comment`).

### Added

- **android-project-aligner** plugin — brownfield companion to android-project-starter. Audits an existing multi-module Android project against the starter's conventions (MVI + Compose + Koin + Navigation 3 + build-logic), produces a phased audit → plan → apply migration, and applies it on a fresh git branch with build verification between every phase. Reuses `android-project-starter:conventions` as the single source of truth and generates the project-local `<project>-android-planner` / `<project>-android-implementer` skills as the final phase. Never pushes.
- **android-project-starter** plugin — wizard-driven scaffolder for Android projects. Interactive `/android-project-starter:init-android-project` runs 6 phased rounds (~20 questions), resolves the latest stable versions for every dependency in parallel, then generates a complete multi-module project: `build-logic/` with 9 convention plugins (or 8 without Room), `core/{common,data,model,designsystem-*,testing,ui-*}`, `feature/<name>/{data,ui-mobile,ui-tv?}` per requested feature with shared data + form-factor-specific MVI stacks, `app-mobile` (+ `app-tv` when TV is picked), `gradle/libs.versions.toml`, Spotless/detekt/lint/CI/Dependabot, an Android-shaped `.gitignore`, and project-local `<project>-android-planner` / `<project>-android-implementer` skills. Verifies the project builds (`./gradlew help`, `:app-mobile:dependencies`, `compileDebugKotlin`, `spotless+detekt+lint+test`) and sweeps for warnings before declaring done, then commits the scaffold as `setup architecture`. The generated implementer skill ships with `references/compose-authoring.md` distilled from [compose-expert](https://github.com/aldefy/compose-skill) covering state management, recomposition stability, modifier ordering, side-effect API selection, lazy-list discipline, animation, theming, Canvas safety, accessibility, TV focus rules, a production crash-pattern table, and deprecated patterns to avoid.
- **Release Notify** GitHub Actions workflow (`.github/workflows/release-notify.yml`) — posts published GitHub Releases to the Slack channel bound to `SLACK_RELEASE_WEBHOOK_URL`. See `.claude/rules/marketplace.md` ("Publishing a release").
- **dep-update-merge** plugin — 6-phase dependency update bundling skill. Discovers open dependency PRs/MRs, analyzes changelogs for breaking changes, offers to exclude breaking updates, creates a combined branch, runs build/test/lint verification with warning baseline comparison, and produces a verified bundle ready for human review.

### Changed

- **android-project-starter** — qa/prod product flavors + shake/broadcast dev-tools dialog, and renamed wizard skill from `init` to `init-android-project` (0.2.0 → 0.3.0). The new slash command is `/android-project-starter:init-android-project` (the old `:init` no longer resolves). Drops the old "build variants" wizard question in favor of a fixed convention: every scaffold gets `qa` (default) and `prod` product flavors on top of `debug` + `release` build types. A new ninth convention plugin, `<project>.android.flavors`, applies to exactly `app-mobile`, `app-tv`, and `core/data` — feature modules and other core modules stay single-variant. The `qa` flavor uses the clean `applicationId`; `prod` gets `applicationIdSuffix = ".prod"` so qa + prod can coexist on one device. Each flavor emits `BuildConfig.IS_QA` and `BuildConfig.API_BASE_URL` from `gradle.properties` keys (`<project>.qaApiBaseUrl`, `<project>.prodApiBaseUrl`) collected in a new Step 4.5 wizard prompt. On `IS_QA` builds, a `DevToolsHost` composable in `core/ui-mobile` mounts a shake listener (`SensorManager` accelerometer, 2.7g threshold, 1-second debounce) and a broadcast receiver listening for `<applicationId>.OPEN_DEV_TOOLS`; either trigger opens an `EnvSelectorDialog` with Staging / Prod / Custom URL choices, persisted via DataStore in a new `EnvironmentConfig` in `core/data`. An `EnvironmentBaseUrlInterceptor` rewrites every outgoing OkHttp request's origin so the runtime override takes effect immediately. TV builds get a broadcast-only variant of `DevToolsHost` (shake is meaningless on a remote). The generated project's README now documents the variant matrix, install commands, and the `adb shell am broadcast -a <applicationId>.OPEN_DEV_TOOLS -p <applicationId>` fire command for triggering the dialog on emulators and TV devices. The build-success gate gains a sixth step that compiles `compileProdDebugKotlin` so flavor-conditional code paths can't sneak past verification.

- **android-project-starter** — incorporates a real-scaffold post-mortem worth of fixes (0.1.0 → 0.2.0). Each fix shaves a known fix-loop off future runs:
  - **AGP 9.2+ Kotlin handling.** Convention plugins (Application, Library) no longer apply `org.jetbrains.kotlin.android` manually. `gradle.properties` now uses `android.builtInKotlin=true` (the AGP 9.2+ default) instead of `=false`. The conventions skill documents both forms with a version-detection rule and adds a Step 9.5 compatibility check so the wizard picks the matching template.
  - **Imperative catalog accessors in module build scripts.** Type-safe `libs.androidx.X` accessors silently break in subproject classpaths on Gradle 9.5.1 + AGP 9.2 with `includeBuild("build-logic")`. The conventions skill now prescribes the `lib("alias")` helper form (`libs.findLibrary("X").get()`) for all module `build.gradle.kts` files, matching the form already used in build-logic.
  - **Compose-aware ktlint defaults in `.editorconfig`.** Disables `function-naming` (Composable functions are PascalCase) and `backing-property-naming` (MVI BaseViewModel uses `_actions`/`_effects` with no public mirror).
  - **Compose-aware detekt defaults in `.detekt/config.yml`.** `UnusedPrivateMember.ignoreAnnotated: [Preview, <Project>Previews, Composable]` so preview composables aren't flagged. `UnusedPrivateProperty.allowedNames` includes `repository|service|store|client|dispatcher|context` so scaffold-injected fields don't trip detekt.
  - **Nav3 `entry<K>` clarification.** The conventions skill now documents that `entry<K>` is a member of `EntryProviderScope<T>` — auto-available inside any function with that receiver — and explicitly warns "do NOT add `import androidx.navigation3.runtime.entry`" (that import doesn't exist; the symbol is a member, not a top-level function). Adds a Nav3 1.1.x quick-reference table with verified package paths.
  - **`MainCoroutineRule` defaults to `UnconfinedTestDispatcher`.** Ensures `BaseViewModel`'s `init { _actions.onEach(::onAction).collect() }` runs synchronously in tests so Turbine effect assertions don't time out at 3s. Pass `StandardTestDispatcher()` explicitly only when you need manual scheduling.
  - **`testOptions.unitTests.isIncludeAndroidResources = true`** added to both Compose convention plugins. Required for Robolectric Compose tests via `createComposeRule()` to find `ComponentActivity` in the merged manifest.
  - **Tighter Step 11.2 self-check + Step 9.5 compatibility check.** Adds a `:feature:<first>:ui-mobile:compileDebugUnitTestKotlin` smoke step before the full test suite so generated test imports are validated early. The pre-generation compatibility check validates AGP↔Kotlin↔Compose↔KSP version pairings before any file is written.
  - **Gradle wrapper bootstrap automation** (Step 11.1). Tries `gradle` on PATH first, then scans the user's projects for a reusable `gradle-wrapper.jar`, then falls back to downloading from `raw.githubusercontent.com/gradle/gradle/v<version>`. No more hard stops when system Gradle is missing.
  - **Expanded fix-loop pattern table** in Step 11.3 with concrete diagnostic recipes for each of the seven failure modes seen on real scaffolds.
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
