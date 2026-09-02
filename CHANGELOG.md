# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Plugin validation in CI** (`.github/workflows/validate.yml`) — runs `claude plugin validate`
  against the marketplace manifest and every plugin directory on each pull request. The repo has
  no build system or test runner, so this is its first automated gate. Requires Claude Code
  v2.1.233 or later.
- Tables of contents in the six `references/` files over 100 lines that lacked one
  (`audit-checklist.md`, `migration-playbook.md`, `build-logic.md`, `dev-tools.md`,
  `compose-authoring.md`, `file-templates.md`). Long reference files are often previewed with a
  partial read; without a contents list there is no signal of what was missed.

### Changed

- **Re-baselined every skill for the current Claude model generation** (Opus 5, released
  2026-07-24; Fable 5.1, released 2026-09-01). Both models ship documented behavioural shifts
  that prompt authors are told to tune for, and they move in opposite directions on several of
  them — so all guidance added here is written in model-neutral terms rather than pinned to a
  model. The audit found no dated prompting patterns to remove: no reasoning incantations, no
  assistant prefill, no anti-formatting rules, no anti-laziness boilerplate, no pinned model
  ids. The work was additive and structural.

- **Standing rules moved above the compaction cut** in the three oversized skills. Claude Code
  does not re-read a `SKILL.md` on later turns, and auto-compaction re-attaches only the first
  5,000 tokens of each invoked skill. `init-android-project` (~11.9k tokens),
  `conventions` (~11.4k) and `align-android-project` (~7.3k) all kept their guardrails in a
  closing section, which is exactly where they get dropped partway through a long run. Each now
  carries its non-negotiables in a standing block before the first step.

- **code-review** (1.2.0 → 1.3.0) — every finding now carries a **severity**
  (`blocker` / `important` / `nice-to-have`, the vocabulary `align-android-project` already
  used) and a **confidence** (`high` / `medium` / `low`), and filtering moved from discovery
  into the Phase 6 summary. A severity filter stated in the prompt is followed literally and
  suppresses real findings at the point where they cannot be recovered. New Core Principle 7,
  "Report what you find, filter afterwards". Phase 6 gained outcome-first reporting guidance.
  Setup's CI Integration now offers *Fable* alongside *Opus* (still the default) and *Sonnet*,
  with the trade stated: strongest reviewer available, roughly double the per-token cost, and
  safety classifiers that can decline a security-heavy diff — which surfaces as a failed CI run
  rather than a review. The generated workflow passes a model **alias**, never a dated id.

- **feature-development** (0.2.0 → 0.3.0) — Phase 2's exploration is now sized to the unknown
  rather than unconditionally launching parallel Explore subagents, and commits to a
  delegation once made instead of re-deriving its findings. Phase 4 gained a "stay inside the
  change" bound: no fixing pre-existing bugs found in passing, targeted edits over whole-file
  rewrites, and no promoting scratch checks into permanent test files — with the explicit
  carve-out that this limits extras only, never the acceptance criteria. Core Principles and a
  new reporting convention moved into a standing block at the top. Phase 4's restatement of the
  three test-collaboration modes now points at `references/test-collaboration-modes.md`, which
  already held the full state machines. Phase 5's Verify phase is deliberately unchanged: it is
  a methodology phase, not the self-check scaffolding current guidance says to delete.

- **dep-update-merge** (1.1.0 → 1.2.0) — a standing "while you work" block requiring every
  progress claim to be tied to a command actually run, and outcome-first reporting in Phase 6.

- **android-project-starter** (0.3.0 → 0.4.0) — `conventions` is now `user-invocable: false`;
  it is background knowledge for other skills, and `/conventions` was never a meaningful action
  for a user to take. Its closing guidance moved into an opening "How to use this skill" block
  carrying the repo's first "What to defer to a human" note for this skill. Dropped a reference
  to a `bump-versions` skill that does not exist in this marketplace.
  `init-android-project` hoisted its Guardrails and Question rules into a single standing block
  before Step 0, dropping the one duplicated statement of the one-question-at-a-time rule, and
  gained a deferral note at the declaring-done gate: a green build gate proves the scaffold
  compiles and tests clean, not that the architecture suits the product.

- **android-project-aligner** (0.1.0 → 0.2.0) — Guardrails hoisted above Step 0 and extended
  with a stay-inside-the-change bound (re-shape and scaffold, don't improve) and a
  progress-grounding rule. Step 2's thirteen inline audit sub-sections collapsed to the area
  list plus a pointer to `references/audit-checklist.md`, which already carried the same
  thirteen areas with grep recipes and severity mappings the SKILL.md restated more thinly.

- **`.claude/rules/skill-authoring.md`** — the frontmatter reference was roughly a third of the
  current surface. Now split into Agent Skills spec fields (portable to Cowork) and Claude Code
  only fields, with guidance on the two invocation switches and a rule against setting `model`
  or `effort` in a published skill. New "Size and placement" section covering the 5,000-token
  compaction budget and the fact that skills are not re-read on later turns. New "Writing for
  the current model generation" section: no model names in a skill, say how to report back,
  don't add self-check scaffolding, say what to leave alone, and don't narrow the search to
  widen the signal. Reference-file table-of-contents threshold lowered from 300 lines to 100 to
  match the platform guidance.

- **`.claude/rules/marketplace.md`** — `claude plugin validate` added to the new-plugin
  checklist, with a note that it does not check version agreement between the two manifests.

## [1.6.0] - 2026-07-08

### Changed

- **code-review** (1.1.0 → 1.2.0) — re-review support. New Phase 0 gathers prior review
  state before reviewing: all inline threads (human reviewers' included — never duplicates
  a concern a human already raised) plus the previous automated summary. Own prior findings
  are classified: fix pushed → verified in the diff, confirmed in-thread and resolved (or
  left open with what's missing); answered/disputed → acknowledged and resolved (author
  response is authoritative); still open → never re-posted, reported as "still open". Single
  summary per PR: the summary is now an issue comment starting with a hidden
  `<!-- code-review-summary -->` marker; on re-review the previous one is found by marker,
  deleted, and a fresh summary with a "Previous findings" status section is posted. Inline
  findings carry a `<!-- code-review-finding -->` marker so own threads are recognized
  regardless of posting identity (CI bot vs local user). New Setup choice: Re-review Thread
  Handling (reply + resolve default / reply only / summary only). README posting-mechanics
  templates and this repo's `.claude/rules/code-review.md` updated with the GitHub mechanics
  (GraphQL `reviewThreads` + `resolveReviewThread`, in-thread replies, issue-comment summary
  lifecycle). CI workflow template gains a `concurrency` group to prevent overlapping runs
  from racing the summary delete-then-post sequence.

## [1.5.0] - 2026-06-30

### Changed

- **feature-development** (0.1.0 → 0.2.0) — improved plan-mode triggering and compatibility.
  Added a "Works with plan mode" section to the SKILL.md body mapping the skill's phases to
  plan mode: Phases 1–3 (frame, explore, plan) run during plan mode; Phase 3's hard checkpoint
  is ExitPlanMode; Phases 4–6 run after approval. Reworded the `description` frontmatter to
  lead with planning-phase framing so the skill is invoked at the first turn even when plan
  mode is active, not only when implementation is being requested. Added an opt-in
  trigger-enforcement step to Setup: teams that have seen the skill under-trigger can install a
  `UserPromptSubmit` hook in their project via `/update-config`; no hook is bundled with the
  plugin.

## [1.4.0] - 2026-06-26

### Added

- **web-frontend-tooling** plugin — blueprint for wiring up linting, formatting,
  type-checking, Git hooks, and dead-code detection in a Node/TypeScript project. Prefers
  the Oxc toolchain (`oxlint` + `oxfmt`) with `husky` + `lint-staged`; adapts to an existing
  ESLint/Prettier setup. Ships minimal starting configs but imposes no specific rules —
  reconciles with the existing codebase rather than overwriting its style.

## [1.3.0] - 2026-06-19

### Added

- **feature-development** plugin — end-to-end development methodology for features and bug
  fixes, distilled from agentic engineering best practices. Six phases: frame the requirement
  (or capture a defect report with reproduction steps + expected-vs-actual for bugs), explore
  and baseline (parallel Explore subagents + green baseline + project testing strategy surfaced
  automatically; root-cause analysis agent for bugs), plan with red/green-shaped steps
  (Prove → Implement; for bugs, first pair is the regression test), implement with a test-first
  entry gate and configurable test loop (Solo AI / Assert-in-the-loop / Ping-pong / Ask each
  feature), verify by driving the running app, and hand off with a linear walkthrough for
  review. Triggers on feature, bug, and regression phrasing. Every test follows the project's
  documented testing strategy out of the box. Configurable commit granularity: checkpoint while
  building (the safety net), then curate into a few logical commits at hand-off by default.
  Never merges unreviewed code. Initial v0.1.0 — the test-collaboration modes are pending
  real-project validation.

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
