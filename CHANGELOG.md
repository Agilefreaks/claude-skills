# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **android-project-starter** plugin (0.3.0) — wizard-driven scaffolder for multi-module Android projects (MVI + Compose + Koin + Navigation 3). Interactive `/android-project-starter:init-android-project` runs 6 phased rounds (~20 questions), resolves the latest stable versions in parallel, then generates a complete project: `build-logic/` with 9 convention plugins (or 8 without Room), `core/{common,data,model,designsystem-*,testing,ui-*}`, `feature/<name>/{data,ui-mobile,ui-tv?}` per requested feature (shared data + form-factor-specific MVI stacks), `app-mobile` (+ `app-tv` if TV picked), `gradle/libs.versions.toml`, Spotless/detekt/lint/CI/Dependabot, an Android-shaped `.gitignore`, and project-local `<project>-android-planner` / `<project>-android-implementer` skills. Highlights:
  - **qa/prod product flavors** baked in. Every scaffold gets `qa` (default, `isDefault = true`) and `prod` (`applicationIdSuffix = ".prod"`) so both can coexist on one device. A dedicated `<project>.android.flavors` convention plugin applies to `app-mobile`, `app-tv`, and `core/data` only — feature modules stay single-variant.
  - **Shake/broadcast dev-tools dialog.** On `IS_QA` builds, `DevToolsHost` in `core/ui-mobile` mounts a `SensorManager` shake listener (2.7g threshold, 1-second debounce) and a `RECEIVER_NOT_EXPORTED` broadcast receiver listening for `<applicationId>.OPEN_DEV_TOOLS`. Either trigger opens an `EnvSelectorDialog` (Staging / Prod / Custom URL) backed by a DataStore-persisted `EnvironmentConfig`. An `EnvironmentBaseUrlInterceptor` rewrites every outgoing OkHttp request's origin so overrides take effect without an app restart. TV builds use a broadcast-only variant — fire the dialog via `adb shell am broadcast -a <applicationId>.OPEN_DEV_TOOLS -p <applicationId>`.
  - **AGP 9.2+ ready.** Convention plugins (Application, Library) do NOT manually apply `org.jetbrains.kotlin.android`; `gradle.properties` uses `android.builtInKotlin=true`. The wizard's Step 9.5 detects the resolved AGP version and picks the matching template (9.0–9.1.x manual-apply, 9.2+ built-in Kotlin).
  - **Imperative catalog accessors** in module build scripts (`libs.findLibrary("X").get()` via a `lib(...)` helper) to dodge the Gradle 9.5.1 + AGP 9.2 type-safe accessor regression with `includeBuild("build-logic")`.
  - **Compose-aware lint defaults.** `.editorconfig` disables ktlint `function-naming` (Composables are PascalCase) and `backing-property-naming` (`_actions`/`_effects` MVI buffers). `.detekt/config.yml` exempts `Preview`/`<Project>Previews`/`Composable` from `UnusedPrivateMember` and pre-allows scaffold-injected field names (`repository|service|store|client|dispatcher|context`).
  - **Nav3 1.1.x quick-reference.** Documents that `entry<K>` is a member of `EntryProviderScope<T>` (not a top-level function — there is no `import androidx.navigation3.runtime.entry`) and ships a verified-package-paths table.
  - **`MainCoroutineRule` defaults to `UnconfinedTestDispatcher`** so `BaseViewModel`'s action-collector launches synchronously in tests and Turbine assertions don't time out at 3s.
  - **`testOptions.unitTests.isIncludeAndroidResources = true`** on both Compose convention plugins — required for Robolectric `createComposeRule()` to find `ComponentActivity` in the merged manifest.
  - **6-step build-success gate.** `./gradlew help`, `:app-mobile:dependencies --configuration qaDebugCompileClasspath`, `compileQaDebugKotlin`, `compileProdDebugKotlin`, `:feature:<first>:ui-mobile:compileDebugUnitTestKotlin`, then `spotlessCheck detekt lint test`. A pre-generation Step 9.5 validates AGP↔Kotlin↔Compose↔KSP↔Nav3 compatibility before any file is written.
  - **Gradle wrapper bootstrap automation** (Step 11.1) — tries system `gradle` on PATH, then scans for a reusable local `gradle-wrapper.jar`, then falls back to downloading from `raw.githubusercontent.com/gradle/gradle/v<version>`. No hard stop when system Gradle is missing.
  - **Expanded fix-loop pattern table** with diagnostic recipes for the failure modes seen on real scaffolds (`CommonExtension` traps, helper-file pitfalls, missing `package` decls, unresolved Nav3 `entry`, type-safe accessor regression, Turbine timeouts, ComponentActivity resolution, AGP 9 manual Kotlin apply).
  - **Generated implementer skill** ships with `references/compose-authoring.md` distilled from [compose-expert](https://github.com/aldefy/compose-skill): state management, recomposition stability, modifier ordering, side-effect API selection, lazy-list discipline, animation, theming, Canvas safety, accessibility, TV focus rules, a production crash-pattern table, and deprecated patterns to avoid.

- **dep-update-merge** plugin — 6-phase dependency update bundling skill. Discovers open dependency PRs/MRs, analyzes changelogs for breaking changes, offers to exclude breaking updates, creates a combined branch, runs build/test/lint verification with warning baseline comparison, and produces a verified bundle ready for human review.

### Changed

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
