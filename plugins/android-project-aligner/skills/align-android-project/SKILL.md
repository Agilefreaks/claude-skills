---
name: align-android-project
description: >
  Interactive aligner that audits an EXISTING Android project against the
  android-project-starter conventions (multi-module MVI + Compose + Koin +
  Nav3 + build-logic), produces a phased migration plan, and applies it on a
  fresh git branch with build verification between every phase. Generates the
  project-local <project>-android-planner and <project>-android-implementer
  skills as the final phase. Never pushes; commit boundary is the phase.
disable-model-invocation: true
---

# Android Project Aligner

You are running an interactive workflow that **aligns an existing Android project** to the conventions defined by `android-project-starter`. You were invoked explicitly via `/android-project-aligner:align-android-project`. Don't assume — confirm, gather answers per round, and **verify the project still builds after every applied phase**.

The workflow has three stages with checkpoints between them:

1. **Audit** (Steps 0–3) — read-only inventory + gap report. Stops here on request.
2. **Plan** (Steps 4–5) — convert gaps into an ordered phased plan. Stops here on request.
3. **Apply** (Steps 6–10) — branch + phased implementation + verification + final commit. Never pushes.

Each phase commits on the migration branch. The user controls when (or whether) to merge.

## Step 0 — Load conventions + pre-flight

### 0.1 — Load the conventions skill

Load `android-project-starter:conventions`. It is the **single source of truth** for module layout, MVI shape, build-logic plugins, theming, tests, version-lookup sources, and tooling defaults. Every file you generate or rewrite must match the canonical shapes shown there.

If the conventions skill is not available (the `android-project-starter` plugin isn't installed), stop and tell the user:

> The aligner depends on `android-project-starter:conventions` as the source of truth. Install it with `/plugin install android-project-starter@agilefreaks-skills --scope user` and re-run.

Do not invent conventions to fill the gap — the whole point of this plugin is "apply the same rules" defined elsewhere.

### 0.2 — Confirm cwd is an Android project

Run these checks in parallel:

```bash
pwd
ls -la
test -f settings.gradle.kts -o -f settings.gradle && echo settings:ok || echo settings:missing
test -f build.gradle.kts -o -f build.gradle && echo root_build:ok || echo root_build:missing
find . -maxdepth 4 -name AndroidManifest.xml -not -path "*/build/*" | head -3
test -x gradlew && echo wrapper:ok || echo wrapper:missing
```

If `settings.gradle*` or `build.gradle*` is missing, this is not an existing Android project — recommend `init-android-project` instead and stop.

If the gradle wrapper is missing, note it (you'll handle it before the first verification step using the same procedure described in `android-project-starter:init-android-project` Step 11.1).

### 0.3 — Confirm git state

```bash
git rev-parse --is-inside-work-tree
git status --porcelain
git rev-parse --abbrev-ref HEAD
```

- If cwd is not a git repo: stop and tell the user — the aligner needs git to checkpoint each phase. Suggest `git init` + initial commit before re-running.
- If the working tree has uncommitted changes: print them and ask whether to (a) stash them and proceed, (b) commit them first, (c) abort. Don't silently sweep them into the migration commits.
- Capture the current branch name; you'll restore the user to it if they abort mid-apply.

### 0.4 — Greeting

Tell the user briefly what's about to happen:

> I'll audit this project against the android-project-starter conventions, produce a phased migration plan, and apply it on a new branch. Three stages with checkpoints between them — you can stop after the audit (gap report only) or after the plan (review before applying). Every applied phase verifies with gradle and commits on the branch. Nothing gets pushed. Total time depends on the project's size and how many phases you approve.

## Step 1 — Project identity discovery

Discover what the project already calls itself. Don't ask if you can infer.

### 1.1 — Read `settings.gradle.kts` (or `.gradle`)

Extract:
- `rootProject.name = "..."` → **project name** (the kebab-case identifier used in convention plugin ids: project name with hyphens stripped → `acme`, `superapp`).
- The list of `include(...)` calls → **module inventory** (raw).

### 1.2 — Read the app module's `build.gradle.kts`

Search for an app module under (in priority order): `app-mobile/`, `app/`, `app-tv/`. Read each one found and extract:
- `namespace = "..."` → **root package** (e.g. `com.acme.android`)
- `applicationId = "..."` → **applicationId**
- The `plugins { }` block → presence of `com.android.application`, `kotlin.android`, `kotlin.compose`, Compose Compiler plugin, any existing convention plugin aliases.
- The `dependencies { }` block → identifies DI library (Hilt vs Koin), nav library (`androidx.navigation3` vs `androidx.navigation` vs custom), and other major stack picks.

If both `app-mobile/` and `app-tv/` exist, this is a mobile + TV project. If only `app/` exists, it's likely a single-form-factor project that needs renaming to `app-mobile/` (flag this as a Phase 1 candidate; don't do it silently).

### 1.3 — Read `gradle/libs.versions.toml` (if present)

Extract:
- AGP version (`agp` or `androidGradlePlugin`)
- Kotlin version (`kotlin`)
- KSP version (`ksp`)
- Compose BOM version (`composeBom`, `compose-bom`, etc.)
- Compose Compiler plugin version (should match Kotlin)
- Other major libs (`koin`, `retrofit`, `apollo`, `room`, `datastore`, etc.)

If `libs.versions.toml` is missing entirely, every dependency is presumably scattered across module build files. Flag this as a Phase 1 blocker — a single-catalog setup is a prerequisite for the convention plugins.

### 1.4 — Derive identifiers

From what you read, derive (and echo back to the user for confirmation):

- **Project name** (kebab-case, used in project-local skill names): e.g. `acme`, `super-app`
- **Project class prefix** (PascalCase of project name with hyphens stripped): `Acme`, `SuperApp`
- **Convention plugin id prefix** (lowercase, hyphens stripped): `acme`, `superapp`
- **Root Kotlin package**: from the app `namespace`
- **applicationId**: from the app `applicationId`
- **Mobile + TV?**: yes if `app-tv/` exists; no otherwise

Print these and ask the user to confirm or correct them in one batch (this is the *one* exception to the question-per-message rule from `init-android-project` — the data comes from the project itself, so the user is confirming what's already on disk).

## Step 2 — Inventory audit (read-only)

Walk through the project and classify every module. **This step writes nothing.** It only reads and grep's.

Read `references/audit-checklist.md` for the canonical checklist — what to look for in each module, the grep recipes, and the "smell → convention" mapping.

For each finding, record a tuple `(finding, severity, effort, risk, conventions-section)`:

- **severity**: `blocker` (project won't compile or convention-plugin scaffold can't proceed without this fix), `important` (architecture diverges materially from conventions), `nice-to-have` (cosmetic or low-impact).
- **effort**: `S` (one file or one dependency), `M` (a few files within one module), `L` (cross-cutting, multiple modules).
- **risk**: `low` (mechanical change, well-tested by gradle), `medium` (could break runtime behavior in subtle ways), `high` (semantic change — DI provider swap, navigation library swap, etc.).
- **conventions-section**: pointer like "conventions §The 9 convention plugins" or "conventions §Screen / ScreenContent contract" so the user can read the canonical shape themselves.

Categories to walk, in this order (matches the phasing in Step 4):

### 2.1 — Foundation
- Version catalog presence + shape (single `gradle/libs.versions.toml`, `[versions]` / `[libraries]` / `[plugins]` sections, convention plugin aliases with `version = "unspecified"`).
- AGP / Kotlin / KSP / Compose Compiler version alignment (Kotlin == Compose Compiler version; KSP `<kotlin>-<X.Y.Z>`).
- `build-logic/` presence. If missing: blocker. If present: enumerate plugin classes in `build-logic/convention/src/main/kotlin/` and compare against the 9 expected (or 8 without Room).
- `gradle.properties` — `android.builtInKotlin` toggle matching AGP version; no legacy `android.defaults.buildfeatures.*` keys.
- `gradlew` + wrapper version (latest stable supporting the resolved AGP).

### 2.2 — Module layout
- `core/common`, `core/model`, `core/data`, `core/testing` — present?
- `core/designsystem-base` + `core/designsystem-mobile` (and `-tv` if TV) — present? Or single `core/designsystem`?
- `core/ui-mobile` (and `core/ui-tv` if TV) — present?
- Each feature under `feature/<x>/` — split into `data/`, `ui-mobile/`, (`ui-tv/` if TV)? Or single-module?
- App modules — `app-mobile/`, `app-tv/`? Or legacy `app/`?

### 2.3 — MVI base types
- `core/common/src/main/kotlin/<root-pkg>/core/`: `ViewAction.kt`, `ViewState.kt`, `ViewSideEffect.kt`, `BaseViewModel.kt`?
- If present, do their shapes match the conventions skill verbatim (`StateFlow<State>` for state, single-consumer `Channel<Effect>`, buffered `MutableSharedFlow<Action>(extraBufferCapacity = 64)`, `init { viewModelScope.launch { _actions.onEach(::onAction).collect() } }`)?
- If custom MVI base (different signatures, different framework like Orbit, etc.): flag as a `high`-risk migration — propose to introduce the canonical `BaseViewModel` alongside and migrate ViewModels feature-by-feature.

### 2.4 — Screen / ScreenContent split
- For each feature ui module, grep `components/`:
  - `<Feature>Screen.kt` present?
  - `<Feature>ScreenContent.kt` present as a separate file?
  - ScreenContent signature: `(modifier: Modifier = Modifier, state: <FeatureState>, onAction: (<FeatureAction>) -> Unit)`?
  - ScreenContent has no `koinViewModel`, `LocalContext.current`, `rememberLauncherForActivityResult`?
  - At least one `@<Project>Previews` composable at the bottom that constructs a sample State and passes `onAction = {}`?

### 2.5 — Repository encapsulation
- For each feature with a `data/` module:
  - `<Feature>Repository.kt` is a public interface?
  - `<Feature>RepositoryImpl.kt` is declared `internal class`?
  - `di/<Feature>DataModule.kt` exposes `val <feature>DataModule` (public)?
  - Grep `app-mobile/build.gradle.kts` (and `app-tv/`) for `:feature:<x>:data` — must be empty.
- If repositories are split per form-factor (a mobile repo and a TV repo): flag — conventions say the data layer is shared.

### 2.6 — DI library
- Koin? Then check `<Feature>Modules.kt` aggregator presence per feature, `<Project>Application` `startKoin { ... }` shape.
- Hilt? Flag as `high`-risk migration. Ask the user: (a) leave Hilt intact and apply only stack-agnostic conventions, (b) migrate to Koin (involves rewriting every `@Inject` constructor + module). Default to (a) unless the user explicitly wants (b).
- Manual DI / no DI: flag as a Koin-introduction migration.

### 2.7 — Navigation
- `androidx.navigation3.runtime` import or dependency? If yes: check `EntryProviderScope<NavKey>.<feature>Entry(...)` shape per feature, `NavDisplay` wiring in the app, no string-based routes.
- `androidx.navigation:navigation-compose` (Nav2)? Flag as a Nav3 migration. `high`-risk because route shape changes.
- Custom navigation? Flag and ask.

### 2.8 — Persistence + Network
- Room dependency + `AndroidRoomConventionPlugin` registered + module applies it for any module that uses Room?
- Retrofit / Apollo / Ktor / nothing? Conventions don't mandate one; check that whatever's used is wired through `core/data` and exposed via Koin.
- DataStore?

### 2.9 — Theming
- `<Project>Theme.kt` in `core/designsystem-mobile`?
- `@<Project>Previews` multi-preview annotation?
- Hardcoded colors / text styles in feature modules? Grep for `Color(0x`, `TextStyle(`, `fontSize =` outside `core/designsystem-*`.

### 2.10 — Flavors + dev-tools
- `qa` and `prod` flavors declared (via `<project>.android.flavors` or directly)?
- `BuildConfig.IS_QA` / `BuildConfig.API_BASE_URL` accessible?
- `core/data/env/` (EnvironmentConfig + DataStore + Koin module)?
- `core/ui-mobile/dev/` (ShakeDetector + DevToolsBroadcastListener + EnvSelectorDialog + DevToolsHost)?
- App-level `DevToolsHost(enabled = BuildConfig.IS_QA)` wrap?

### 2.11 — Tooling + CI
- Spotless plugin applied? ktlint version pinned in catalog?
- detekt plugin applied? `.detekt/config.yml` present?
- `.lint/config.xml` present?
- `.editorconfig` present?
- `.github/workflows/build.yml` (or equivalent) running spotlessCheck + lint + detekt + test?
- `.github/dependabot.yml`?

### 2.12 — Test conventions
- `MainCoroutineRule` in `core/testing` defaulting to `UnconfinedTestDispatcher`?
- ViewModel tests use Turbine + Mockito-Kotlin + Truth?
- Compose tests live in `src/test/kotlin/` (not `src/androidTest/`)?
- `src/test/resources/robolectric.properties` with `sdk=34`?

### 2.13 — Project-local skills
- `.claude/skills/<project>-android-planner/SKILL.md` present?
- `.claude/skills/<project>-android-implementer/SKILL.md` present?
- If present, do they reference the correct project class prefix + module inventory? (Re-generating them in Phase 7 is cheap; flag for refresh.)

## Step 3 — Gap report

Print the audit findings as a tabulated gap report — one row per finding, grouped by category. Format:

```
Foundation
  [blocker]   No build-logic/ module — convention plugins missing entirely
    effort=L, risk=low, ref=conventions §The 9 convention plugins
  [important] AGP 9.0.1 vs Kotlin 2.0.21 — AGP 9.2+ requires Kotlin 2.1+ for built-in apply
    effort=S, risk=medium, ref=conventions §AGP version table
  ...

Module layout
  [important] feature/home is a single module — should be feature/home/{data,ui-mobile}
    effort=M, risk=medium, ref=conventions §Feature module shape
  ...

(etc.)
```

After the table, print a **summary line**:

```
Found 23 gaps: 4 blockers, 11 important, 8 nice-to-have. Estimated effort: 6 phases.
```

Then ask:

> Three options:
>   1. Stop here — you'll act on the gap report manually.
>   2. Continue to the plan — I'll group these into a phased migration plan you can review.
>   3. Abort — no changes made.

If they pick (1), end the workflow cleanly and remind them they can re-run later. If (3), exit. Otherwise continue to Step 4.

## Step 4 — Build the migration plan

Convert the gap report into an **ordered phase list**. Each phase has:

- **Name** — short, imperative, used as the commit subject.
- **Goal** — one sentence: what the phase achieves.
- **Includes** — the specific findings from the gap report that this phase resolves.
- **File changes** — concrete list of files to create / modify / delete (paths, not bodies).
- **Verification** — the gradle command(s) that must pass before committing.
- **Risk** — propagated from the highest-risk finding included.
- **Skippable** — `yes` if the phase is independent of later phases; `no` if later phases depend on it.

### Canonical phase order

Read `references/migration-playbook.md` for the canonical phase recipes. The default ordering is:

1. **Foundation: catalog + build-logic** — consolidate `gradle/libs.versions.toml`, scaffold `build-logic/convention/` with the 9 (or 8) plugin classes + `ProjectExtensions.kt`, register them in `build-logic/convention/build.gradle.kts`, declare convention plugin aliases in the catalog. Verification: `./gradlew help --no-daemon`. Skippable: no — every later phase depends on the convention plugin ids being resolvable.

2. **Foundation: gradle.properties + wrapper** — set `android.builtInKotlin` toggle per AGP version, remove legacy keys, ensure wrapper version supports the resolved AGP. Verification: `./gradlew --version`. Skippable: no.

3. **Core: common + testing** — introduce `core/common` (BaseViewModel + ViewAction/State/Effect) and `core/testing` (MainCoroutineRule). Apply `<prefix>.jvm.library` or `<prefix>.android.library` per conventions. If a custom MVI base already exists, **do not delete it yet** — introduce the canonical base alongside; later phases migrate ViewModels feature-by-feature. Verification: `./gradlew :core:common:compileDebugKotlin :core:testing:compileDebugKotlin --no-daemon`. Skippable: yes (but later feature phases will be much harder without it).

4. **Core: designsystem + ui** — introduce `core/designsystem-base` (tokens) + `core/designsystem-mobile` (`<Project>Theme`, `@<Project>Previews`, shared components) + `core/ui-mobile` (shared composables). If TV: also `core/designsystem-tv` + `core/ui-tv`. If hardcoded colors / text styles exist in features, the migration plan calls them out but does NOT auto-rewrite — that's flagged for the Boy Scout phase in the project-local implementer skill. Verification: each module's `compileQaDebugKotlin`. Skippable: yes.

5. **Core: data + env + dev-tools (if flavors are added)** — `core/data` skeleton (Koin Qualifiers, session store if Auth picked, network interceptor if Network picked) + `env/EnvironmentConfig*.kt` + `network/EnvironmentBaseUrlInterceptor.kt`. Apply `<prefix>.android.flavors` to `core/data`. Verification: `:core:data:compileQaDebugKotlin` and `:core:data:compileProdDebugKotlin`. Skippable: yes (only if the user opts out of flavors).

6. **Feature split** — for each existing feature, split into `feature/<x>/{data, ui-mobile, ui-tv?}`. Apply convention plugins per conventions skill. Move existing files into the right module while preserving git history (`git mv`). If a feature is already split correctly, this phase is a no-op for it. Verification per feature: `:feature:<x>:ui-mobile:compileQaDebugKotlin`. Skippable: per-feature, yes.

7. **Feature normalize** — for each feature: ensure Screen + ScreenContent are separate files, `(modifier, state, onAction)` signature, at least one `@<Project>Previews`, no Koin in ScreenContent, repository impl marked `internal`, no app→data dependency leak. The phase modifies existing files conservatively — it never rewrites logic, only re-shapes the file boundary. Verification per feature: `:feature:<x>:ui-mobile:compileDebugUnitTestKotlin`. Skippable: per-feature, yes.

8. **App wiring** — `app-mobile/build.gradle.kts` (and `app-tv/` if TV) — apply convention plugins, `<prefix>.android.flavors` last, ensure no `:feature:<x>:data` dependency leak. `<Project>Application` collects all feature `<feature>Modules`. `MainActivity` wraps content in `DevToolsHost(enabled = BuildConfig.IS_QA)` if flavors + dev-tools are being added. Verification: `:app-mobile:compileQaDebugKotlin :app-mobile:compileProdDebugKotlin`. Skippable: no (this phase is what ties everything else together at the app level).

9. **Quality + CI** — Spotless block in root `build.gradle.kts`, `.detekt/config.yml`, `.lint/config.xml`, `.editorconfig`, `.github/workflows/build.yml`, `.github/dependabot.yml`. Verification: `./gradlew spotlessCheck detekt lint --no-daemon`. Skippable: yes.

10. **Tests** — `MainCoroutineRule` defaults to `UnconfinedTestDispatcher`, `robolectric.properties` `sdk=34`, Compose tests live in `src/test/kotlin/`. For each feature, add at least one ViewModel test (action-handling + effect-emission via Turbine) and one ScreenContent Compose test (action dispatch + state-driven rendering) **only if missing** — don't replace existing tests. Verification: `./gradlew test --no-daemon`. Skippable: yes (but the project-local implementer skill expects this shape).

11. **Project-local skills** — generate (or update) `.claude/skills/<project>-android-planner/SKILL.md` + references and `.claude/skills/<project>-android-implementer/SKILL.md` + references. Uses the same generation logic as `init-android-project` Step 10.9 — see that skill for the exact layout. Verification: file existence + `<Project>Previews` placeholder substitution. Skippable: no (this is the deliverable that makes future work cheap).

### Phase decisions to defer to the user

Some migrations are not safe to do without explicit user buy-in. For each of these, the plan lists the phase as `pending user decision`:

- **DI library swap** (Hilt → Koin). Only proceed if the user explicitly chooses to migrate. Default: leave Hilt intact and document the deviation in the project-local implementer skill.
- **Navigation library swap** (Nav2 → Nav3, or custom → Nav3). Only proceed if the user explicitly chooses. Default: leave existing nav intact and document.
- **MVI base swap** (custom MVI / Orbit / etc. → canonical `BaseViewModel`). Default: introduce canonical base, migrate one feature as a worked example, leave the rest unchanged and flag in the implementer skill.
- **Module rename** (`app/` → `app-mobile/`). High-risk because it touches `settings.gradle.kts` and every cross-module reference. Default: skip unless user opts in.

For each deferred phase, print the trade-off and ask. One question at a time, plain-text chat message (the question-per-message rule applies here — these are real decisions, not echo-confirms).

## Step 5 — Confirm plan + checkpoints

Print the final plan as a numbered list (one line per phase: name, risk, skippable, included findings count) and a footer:

```
Plan: 11 phases. Estimated 8 commits on branch align/architecture-conventions.

Options:
  1. Apply all
  2. Apply phases 1–8 (skip Quality+CI, Tests, Skills for now)
  3. Apply specific phases (you tell me which numbers)
  4. Edit the plan (rare — usually means a deferred decision needs re-answering)
  5. Stop — I'll save the plan to .claude/migration-plan.md and you can act on it manually
  6. Abort
```

If the user picks (5), write the plan to `.claude/migration-plan.md` (create the directory if needed), tell them where it is, and exit. The plan in that file is the same content you just printed.

If the user picks any of (1)–(3), continue to Step 6.

## Step 6 — Create the branch

Default branch name: `align/architecture-conventions`. If a branch with that name already exists, append a UTC date suffix: `align/architecture-conventions-2026-06-03`.

Ask the user (one plain-text question) whether to use the default or supply a custom name. If they answer with a name, validate it against git's branch-name rules (no spaces, no `..`, etc.) and use that.

```bash
git checkout -b <branch-name>
git rev-parse --short HEAD       # capture baseline SHA
./gradlew help --no-daemon       # capture baseline compile state
```

If `./gradlew help` fails on the baseline, **the project already doesn't compile**. Stop and tell the user — the aligner needs a green baseline to verify after each phase. Suggest they fix the baseline first (or run the aligner anyway with the understanding that some phases may not be able to verify; in practice the user should fix first).

Print the baseline SHA so the user can `git reset --hard <sha>` if they want to undo the whole migration later.

## Step 7 — Apply each phase

For each approved phase in order:

### 7.1 — Announce the phase

```
Phase 3/11: Core: common + testing
  Includes: 4 findings
  Risk: low
  Verification: ./gradlew :core:common:compileDebugKotlin :core:testing:compileDebugKotlin --no-daemon
```

### 7.2 — Apply the changes

Follow `references/migration-playbook.md` for the specific recipe. Substitute project-specific identifiers (project name, root package, plugin id prefix, class prefix). When creating files, **reproduce the canonical shape from the conventions skill verbatim** — the migration playbook only describes the *delta from current state*, not the canonical body.

Strict rules (same as `init-android-project` Step 10):
- **Reproduce convention plugin shapes verbatim.** No helper files, no `package` declarations on the 9 plugin classes. `ProjectExtensions.kt` is the one exception (`package <project>.android.buildlogic`).
- **Use `ApplicationExtension` and `LibraryExtension` directly** — never `CommonExtension`.
- **Skip `AndroidRoomConventionPlugin.kt`** if the project doesn't use Room.
- **Preserve git history when moving files** — `git mv <old> <new>` rather than create + delete.
- **Never rewrite business logic** — the aligner re-shapes files (split, move, rename) and adds missing scaffolding; it does not change what the code *does*. Behavior changes are out of scope.

When in doubt about a canonical shape, re-read the relevant section of `android-project-starter:conventions` or its `references/` files.

### 7.3 — Run verification

Run the phase's verification command. Use `--no-daemon` to avoid stale daemon false-negatives.

**Fix-loop policy** (same as `init-android-project` Step 11.3):
- Read the full stdout AND stderr.
- Identify the offending file + line.
- Cross-reference against conventions — if the broken file diverges from canonical, restore canonical.
- Common failures: same list as init-android-project Step 11.3 (CommonExtension misuse, missing `import <project>.android.buildlogic.libs`, AGP/Kotlin/Compose Compiler version drift, etc.).
- **Cap at 5 iterations per phase.** If still failing, stop and ask the user how to proceed (rollback this phase, skip remaining phases, abort).

### 7.4 — Commit the phase

```bash
git add -A
git commit -m "align: <phase name>"
```

Use the phase's name as-is. No Claude co-author trailer on these — this is conventions enforcement, not authored content. Let any user-configured commit signing run; **never `--no-verify`**.

If a pre-commit hook (e.g. Spotless) fails: run the hook's fix (`./gradlew spotlessApply`), re-stage with `git add -A`, and recommit. **Create a new commit if the hook rejected — never `--amend`** (the original commit didn't land).

After commit, print:

```
✓ Phase 3/11 committed: <sha> align: Core: common + testing
```

### 7.5 — Move to the next phase

Don't pause between phases unless the user asked to checkpoint. After all approved phases are applied, continue to Step 8.

## Step 8 — Project-local skills (always run, even if Phase 11 was skipped above)

The `<project>-android-planner` and `<project>-android-implementer` skills are the deliverable that makes future work cheap. Always generate (or update) them at the end of the apply stage, even if the user deselected the "Project-local skills" phase explicitly — instead, **ask once more** before skipping:

> The project-local planner/implementer skills weren't in your selected phases. They're the main deliverable of this workflow — without them you'll re-explain the project's conventions to Claude every time. Add them now? [y/N]

If the user says no, respect that — but log it in the final summary so they know why the `.claude/skills/` directory is empty.

If yes (or if Phase 11 was already selected), generate the skills exactly as `init-android-project` Step 10.9 describes:

```
<project-root>/.claude/skills/
├── <project>-android-planner/
│   ├── SKILL.md
│   └── references/
│       └── project-architecture.md       # module inventory, MVI base types, navigation map
└── <project>-android-implementer/
    ├── SKILL.md
    └── references/
        ├── project-architecture.md       # symlink or copy of the planner's reference
        └── compose-authoring.md          # the full Compose rules (copy + project-adapt)
```

**If the skills already exist** (e.g. from a previous aligner run or from `init-android-project`), this is a *refresh* — diff the current files against what would be generated now, and apply the diff. Don't blindly overwrite — the user may have hand-edited their planner/implementer skills. Show the diff and ask before applying.

Commit:

```bash
git add .claude/skills/
git commit -m "align: project-local planner + implementer skills"
```

## Step 9 — Final verification + warning sweep

Run the full quality gate across all phases applied:

```bash
./gradlew spotlessApply --no-daemon
./gradlew spotlessCheck detekt lint test --no-daemon
./gradlew :app-mobile:compileQaDebugKotlin --no-daemon
./gradlew :app-mobile:compileProdDebugKotlin --no-daemon   # only if flavors phase was applied
# if TV:
./gradlew :app-tv:compileQaDebugKotlin --no-daemon
./gradlew :app-tv:compileProdDebugKotlin --no-daemon
```

If anything fails: same fix-loop policy (5 iterations). If still broken, halt and ask.

Then run the warning sweep (same as `init-android-project` Step 11.4):

1. `./gradlew :app-mobile:compileQaDebugKotlin --warning-mode=all --no-daemon` — fix deprecations, unused params, unchecked casts.
2. `./gradlew lint --no-daemon` — triage Lint warnings: correctness + security fix immediately, accessibility fix unless decorative, performance/style fix the easy ones.
3. `./gradlew detekt --no-daemon` — fix style violations; suppress narrowly with comments if a category is genuinely disputed.

Any warnings introduced by the migration itself must be fixed. Warnings already in the project before the migration started can be left for the Boy Scout phase in the implementer skill — but log them in the final summary so the user knows what's outstanding.

If `spotlessApply` produced new changes, commit them:

```bash
git add -A
git commit -m "align: spotlessApply"
```

## Step 10 — Final summary

Print to the user:

```
Migration complete on branch: align/architecture-conventions
Baseline SHA: <baseline>
Final SHA:    <final>
Commits on branch: <N>

Phases applied:
  ✓ 1. Foundation: catalog + build-logic
  ✓ 2. Foundation: gradle.properties + wrapper
  ✓ 3. Core: common + testing
  ⊗ 4. Core: designsystem + ui (skipped by user)
  ✓ 5. Core: data + env + dev-tools
  ✓ 6. Feature split (3 features)
  ✓ 7. Feature normalize
  ✓ 8. App wiring
  ✓ 9. Quality + CI
  ⊗ 10. Tests (skipped by user)
  ✓ 11. Project-local skills

Verification:
  ✓ ./gradlew help
  ✓ ./gradlew :app-mobile:compileQaDebugKotlin
  ✓ ./gradlew :app-mobile:compileProdDebugKotlin
  ✓ ./gradlew spotlessCheck detekt lint test

Warnings:
  Pre-existing: 14 (unchanged from baseline — flagged in implementer skill for Boy Scout)
  Introduced by migration: 0
  Suppressed with justification: 3

Project-local skills written:
  .claude/skills/<project>-android-planner/SKILL.md
  .claude/skills/<project>-android-implementer/SKILL.md

Deferred decisions (not migrated — left intact):
  • DI library: Hilt (you chose to keep it; the implementer skill documents the deviation)

Next step:
  Review the branch:  git log --oneline align/architecture-conventions
  Open in Android Studio and try /<project>-android-planner on a new feature.
  When you're ready, merge into main yourself (the aligner never pushes).
```

If any phase was skipped by the user, list the gaps that remain unaddressed (so they can pick them up later).

### 10.1 — Do not push

Never run `git push`, never set a remote, never `git push --set-upstream`. The branch is local. The user merges when ready. This skill stops at the local commits.

## Guardrails

- **Never modify files outside cwd.**
- **Never push, never set a remote.** Commit boundary is the phase; merge is the user's job.
- **Never `--no-verify`.** Pre-commit hooks exist for a reason and run in CI anyway. Fix the root cause.
- **Never overwrite the user's hand-edited files without asking.** Particularly project-local skills (`.claude/skills/`) and config files (`.detekt/config.yml`, `.lint/config.xml`, `.editorconfig`, CI workflows) — diff first, ask before applying.
- **Never rewrite business logic.** Re-shape files (split/move/rename) and add missing scaffolding. Behavior changes are out of scope for the aligner.
- **Never delete files without explicit confirmation.** If a phase would remove a file (e.g. a custom MVI base the conventions replace), call it out in the plan and ask before doing it during apply.
- **Use `git mv` when moving files** so history is preserved.
- **Defer the high-risk migrations** (DI swap, nav swap, custom-MVI swap, app/ rename) until the user explicitly opts in. Default is "leave intact and document the deviation."
- **Stop after 5 fix-loop iterations** on a phase. Don't churn. Tell the user exactly what's failing and let them decide.
- **The conventions skill is the source of truth.** When in doubt, re-read the relevant section. Don't invent new conventions in this skill — point at the canonical reference.
- **Refuse to start** if cwd isn't an Android project, isn't a git repo, or has uncommitted unrelated changes the user hasn't decided what to do with.
