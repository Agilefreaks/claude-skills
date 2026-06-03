# Migration Playbook

Per-phase recipes the aligner follows during **Step 7 — Apply**. Each recipe describes the **delta** from the existing-project state to the canonical conventions state. The canonical *bodies* are in `android-project-starter:conventions` and its references — don't restate them here; pull them at apply time.

Every recipe assumes:
- The conventions skill is loaded.
- Project identifiers are resolved: `<project>` (plugin id prefix), `<Project>` (class prefix), `<root-pkg>` (root Kotlin package), and the module inventory.
- You're on the migration branch and the working tree was clean at branch creation.

## Phase 1 — Foundation: catalog + build-logic

**Goal:** the project has a single version catalog and the 9 (or 8) convention plugins, registered and resolvable.

**Steps:**

1. **Catalog**. If `gradle/libs.versions.toml` is missing, create it. If it exists, consolidate any scattered version literals into the `[versions]` section. Move every dependency reference in module build files to use `libs.findLibrary(...)` (imperative form per conventions skill `§libs.versions.toml structure`).

2. **Convention plugin aliases**. In `[plugins]`, add:

   ```toml
   <project>-android-application = { id = "<project>.android.application", version = "unspecified" }
   <project>-android-application-compose = { id = "<project>.android.application.compose", version = "unspecified" }
   <project>-android-library = { id = "<project>.android.library", version = "unspecified" }
   <project>-android-library-compose = { id = "<project>.android.library.compose", version = "unspecified" }
   <project>-android-feature = { id = "<project>.android.feature", version = "unspecified" }
   <project>-android-flavors = { id = "<project>.android.flavors", version = "unspecified" }
   <project>-android-lint = { id = "<project>.android.lint", version = "unspecified" }
   <project>-android-room = { id = "<project>.android.room", version = "unspecified" }   # only if Room is used
   <project>-jvm-library = { id = "<project>.jvm.library", version = "unspecified" }
   ```

3. **build-logic/**. If missing entirely, create:
   - `build-logic/settings.gradle.kts` — reads `../gradle/libs.versions.toml` per conventions.
   - `build-logic/convention/build.gradle.kts` — registers all 9 (or 8) plugins under `gradlePlugin.plugins`.
   - `build-logic/convention/src/main/kotlin/<project>/android/buildlogic/ProjectExtensions.kt` — verbatim from `conventions/references/build-logic.md`, package `<project>.android.buildlogic`.
   - The 9 (or 8) plugin classes in the default (root) package, each in its own file. Bodies verbatim from `conventions/references/build-logic.md`. No helper files. No package declarations on the 9 classes.

4. **If build-logic/ exists but is wrong**: do not blow it away. Fix it surgically:
   - If a plugin class has a `package` declaration → remove the `package` line.
   - If a helper file exists (`AndroidCommon.kt`, `KotlinExtensions.kt`, etc.) → inline its body into each plugin that referenced it, then delete the helper.
   - If `CommonExtension` is used → rewrite the plugin to configure `ApplicationExtension` or `LibraryExtension` directly.
   - If `ProjectExtensions.kt` is missing or in the wrong package → create it / move it to `<project>.android.buildlogic`. Add `import <project>.android.buildlogic.libs` near the top of every plugin that calls `libs.findLibrary(...)` / `libs.findVersion(...)`.

5. **`settings.gradle.kts`** — ensure `pluginManagement { includeBuild("build-logic") }` is present.

**Verification:**

```bash
./gradlew help --no-daemon
```

This catches "convention plugin doesn't compile" / "implementationClass doesn't match class name" / "catalog alias missing" early.

**Commit:** `align: foundation: catalog + build-logic`

---

## Phase 2 — Foundation: gradle.properties + wrapper

**Goal:** `gradle.properties` matches the AGP version's expectations; wrapper version supports the resolved AGP.

**Steps:**

1. **gradle.properties** — set / remove keys per the AGP table in `conventions §AGP version matters`:
   - AGP ≥ 9.2.0: `android.builtInKotlin=true` (or omit) + remove any legacy `android.defaults.buildfeatures.*` keys.
   - AGP 9.0.x–9.1.x: `android.builtInKotlin=false`.
   - AGP 8.x: no `builtInKotlin` setting.
   - Ensure: `android.useAndroidX=true`, `kotlin.code.style=official`, `org.gradle.parallel=true`, `org.gradle.caching=true`, `android.nonTransitiveRClass=true`.
   - Add (if flavors will be added in Phase 5): `<project>.qaApiBaseUrl=...` and `<project>.prodApiBaseUrl=...`. Ask the user for the values if unknown.

2. **Wrapper** — if `distributionUrl` in `gradle/wrapper/gradle-wrapper.properties` points to a Gradle version older than the AGP's minimum, bump it to the latest stable that supports the AGP. Don't change the wrapper jar unless `./gradlew --version` is broken — the jar is forward-compatible.

**Verification:**

```bash
./gradlew --version
./gradlew help --no-daemon
```

**Commit:** `align: foundation: gradle.properties + wrapper`

---

## Phase 3 — Core: common + testing

**Goal:** `core/common` (MVI base types) and `core/testing` (MainCoroutineRule) exist with the canonical shapes.

**Steps:**

1. **`core/common/`** — apply `<project>.jvm.library` or `<project>.android.library` (depending on whether `BaseViewModel` needs the Android lifecycle dep — it does, so `<project>.android.library`).
   - `core/common/src/main/kotlin/<root-pkg>/core/ViewAction.kt` — verbatim from `conventions/references/mvi-base.md`.
   - `core/common/src/main/kotlin/<root-pkg>/core/ViewState.kt` — verbatim.
   - `core/common/src/main/kotlin/<root-pkg>/core/ViewSideEffect.kt` — verbatim.
   - `core/common/src/main/kotlin/<root-pkg>/core/BaseViewModel.kt` — verbatim. **No `import <root-pkg>.core.View*` lines** — same package.

2. **`core/testing/`** — apply `<project>.android.library`.
   - `core/testing/src/main/kotlin/<root-pkg>/core/testing/MainCoroutineRule.kt` — verbatim from `conventions/references/test-patterns.md`. **Default to `UnconfinedTestDispatcher`** — accept `StandardTestDispatcher()` as an explicit argument for time-based tests.

3. **If a custom MVI base already exists** (e.g. `MyBaseViewModel`, Orbit container, etc.) — **do not delete it**. Introduce the canonical `BaseViewModel` alongside in `core/common`. Later phases (or the project-local implementer skill) migrate ViewModels feature-by-feature. Document the coexistence in the deliverable summary so the user knows there are two MVI styles in the codebase temporarily.

**Verification:**

```bash
./gradlew :core:common:compileDebugKotlin :core:testing:compileDebugKotlin --no-daemon
```

**Commit:** `align: core: common + testing`

---

## Phase 4 — Core: designsystem + ui

**Goal:** design tokens, theme, and the shared composables live in `core/designsystem-base` + `core/designsystem-mobile` (+ `-tv` if TV) + `core/ui-mobile` (+ `core/ui-tv` if TV).

**Steps:**

1. **For mobile-only projects**: use `core/designsystem` + `core/ui` (no `-base`/`-mobile` split). The split adds noise without payoff for a single form factor.

2. **For mobile + TV**:
   - `core/designsystem-base/` — color tokens (`object <Project>Colors`), typography (`val <Project>Typography`), spacing (`object Spacing`), shapes if needed. Pure-Kotlin (`<project>.jvm.library`) if tokens are non-Compose; `<project>.android.library` if any token references Compose types.
   - `core/designsystem-mobile/` — `<Project>Theme` composable wrapping `MaterialTheme`, `@<Project>Previews` multi-preview annotation, mobile-shaped components (`<Project>Button`, `<Project>Loading`, `<Project>ErrorState`, `HandleEffects`).
   - `core/designsystem-tv/` — TV-shaped components + theme (uses `androidx.tv:tv-foundation`/`tv-material`).

3. **`core/ui-mobile/`** — shared mobile composables (anything reused across features, not just within one). If dev-tools is in scope this phase touches `core/ui-mobile/dev/` (else leave for Phase 5).

4. **`core/ui-tv/` (TV only)** — shared TV composables.

5. **Don't auto-rewrite hardcoded colors / text styles in feature modules.** This phase introduces the design system; the Boy Scout pass in the implementer skill picks them up over time. Flag them in the final summary so the user knows what's outstanding.

**Verification:**

```bash
./gradlew :core:designsystem-base:compileDebugKotlin :core:designsystem-mobile:compileDebugKotlin :core:ui-mobile:compileDebugKotlin --no-daemon
# if TV:
./gradlew :core:designsystem-tv:compileDebugKotlin :core:ui-tv:compileDebugKotlin --no-daemon
```

**Commit:** `align: core: designsystem + ui`

---

## Phase 5 — Core: data + env + dev-tools (when flavors phase opts in)

**Goal:** `core/data` skeleton with the env/dev-tools data layer; `<project>.android.flavors` applied so `BuildConfig.IS_QA` + `BuildConfig.API_BASE_URL` are emitted.

**Steps:**

1. **`core/data/`** — apply `<project>.android.library` + `<project>.android.flavors` (last). Scaffold:
   - Network interceptor scaffold (if Network was already in the project, leave the existing interceptors; just ensure they live here).
   - Session store interface (if Auth is in scope).
   - `env/EnvironmentConfig.kt`, `env/EnvironmentConfigImpl.kt`, `env/EnvironmentConfigDataStore.kt` — verbatim from `conventions/references/dev-tools.md`.
   - `env/di/EnvironmentModule.kt` — exposes `val environmentModule`.
   - `network/EnvironmentBaseUrlInterceptor.kt` — registered FIRST in the OkHttp client's interceptor chain.
   - Add `androidx-datastore-preferences` to its dependencies.

2. **Dev-tools UI** — in `core/ui-mobile/dev/`: `ShakeDetector.kt`, `DevToolsBroadcastListener.kt`, `EnvSelectorDialog.kt`, `DevToolsHost.kt`. Verbatim from `conventions/references/dev-tools.md`.

3. **TV variant (TV-only)** — in `core/ui-tv/dev/DevToolsHost.kt` (broadcast-only, no shake). When mobile + TV is picked, lift `DevToolsBroadcastListener.kt` and `EnvSelectorDialog.kt` into a shared `core/ui-base/` module so `core/ui-tv` doesn't depend on `core/ui-mobile`.

4. **The literal Staging/Prod URLs in `environmentModule`** must match the values in `gradle.properties` (`<project>.qaApiBaseUrl`, `<project>.prodApiBaseUrl`). Single source of truth = the wizard / aligner — re-derive both at apply time.

**Verification:**

```bash
./gradlew :core:data:compileQaDebugKotlin :core:data:compileProdDebugKotlin --no-daemon
./gradlew :core:ui-mobile:compileQaDebugKotlin --no-daemon
# if TV:
./gradlew :core:ui-tv:compileQaDebugKotlin --no-daemon
```

**Commit:** `align: core: data + env + dev-tools`

---

## Phase 6 — Feature split

**Goal:** each feature `<x>/` is split into `feature/<x>/{data, ui-mobile, ui-tv?}` with the canonical module structure.

**Per feature** (re-run the phase per feature; commit per feature so the history is easy to revert):

1. **If the feature is currently a single module** (`feature/<x>/` with `build.gradle.kts` at its root, all code mixed):
   - Create `feature/<x>/data/` (apply `<project>.android.library`) and `feature/<x>/ui-mobile/` (apply `<project>.android.feature`). If TV, also `feature/<x>/ui-tv/`.
   - **Use `git mv`** to relocate existing files into the right module. Don't create+delete — preserve history.
     - Repository + repository impl + data sources → `feature/<x>/data/src/main/kotlin/<root-pkg>/feature/<x>/data/`.
     - ViewModel, MVI types, Route/Entry, Screen/ScreenContent → `feature/<x>/ui-mobile/src/main/kotlin/<root-pkg>/feature/<x>/`.
   - Update `settings.gradle.kts`: remove the old `:feature:<x>` include; add `:feature:<x>:data` and `:feature:<x>:ui-mobile` (+`:ui-tv` if TV).
   - Update `app-mobile/build.gradle.kts`: replace `implementation(project(":feature:<x>"))` with `implementation(project(":feature:<x>:ui-mobile"))`. **Never** add `:feature:<x>:data` as a dependency on the app.

2. **If the feature is already split** but the structure is slightly off (e.g. the ui module is named `feature/<x>/ui` instead of `feature/<x>/ui-mobile`):
   - For mobile-only projects, `feature/<x>/ui` is acceptable — don't force the `-mobile` rename. The convention is form-factor-suffixed when there's more than one form factor. Document either way.
   - For mobile + TV projects, the rename to `-mobile` is required so the TV module can be added without ambiguity. Use `git mv`.

3. **For each feature ui module, regenerate the build.gradle.kts** per `conventions/references/feature-build-files.md`:
   - `plugins { alias(libs.plugins.<project>-android-feature) }`
   - `android { namespace = "<root-pkg>.feature.<x>" }`
   - `dependencies { implementation(project(":feature:<x>:data")) ... }` with `implementation` scope (never `api`).

4. **For each feature data module, regenerate the build.gradle.kts** per `conventions/references/data-encapsulation.md`:
   - `plugins { alias(libs.plugins.<project>-android-library) alias(libs.plugins.<project>-android-lint) }` (+ KSP / Room if used).
   - `android { namespace = "<root-pkg>.feature.<x>.data" }`.

**Verification per feature:**

```bash
./gradlew :feature:<x>:ui-mobile:compileQaDebugKotlin --no-daemon
# if TV:
./gradlew :feature:<x>:ui-tv:compileQaDebugKotlin --no-daemon
```

**Commit per feature:** `align: feature/<x>: split into data + ui-mobile`

---

## Phase 7 — Feature normalize

**Goal:** for each feature, the Screen/ScreenContent split, `(modifier, state, onAction)` signature, repository encapsulation, and at least one `@<Project>Previews` are in place.

**Per feature** (commit per feature):

1. **Screen / ScreenContent split**:
   - If the feature's UI is currently one file: extract the UI body into `<Feature>ScreenContent.kt` with signature `fun <Feature>ScreenContent(modifier: Modifier = Modifier, state: <FeatureState>, onAction: (<FeatureAction>) -> Unit)`. The remaining `<Feature>Screen.kt` keeps the `koinViewModel()`, `HandleEffects`, and navigation callbacks.
   - Replace bare callbacks (`onClick`, `onSubmit`, `onItemClick`) in the new ScreenContent with `onAction(<TypedAction>)` calls. Add the new typed actions to the feature's `<Feature>Action` sealed interface.

2. **Repository encapsulation**:
   - Add `internal` modifier to `<Feature>RepositoryImpl`.
   - Ensure `<Feature>Repository` interface stays public.
   - Ensure `<feature>DataModule` Koin value is public.

3. **Previews**:
   - If `<Feature>ScreenContent.kt` has no preview, add at least one `@<Project>Previews`-annotated composable at the bottom that constructs a sample `State` and passes `onAction = {}`.
   - Add separate previews for loading / empty / error states if the screen renders them differently.
   - If you find `@<Project>Previews` and `@Preview` stacked on the same function, delete the `@Preview`.

4. **App→data leak**:
   - In `app-mobile/build.gradle.kts` (and `app-tv/`), grep for `:feature:<x>:data`. If found, remove the line.

5. **Don't touch business logic.** This phase is structural — re-shape file boundaries, add scaffolding. If existing logic looks buggy, leave it; that's not the aligner's job.

**Verification per feature:**

```bash
./gradlew :feature:<x>:ui-mobile:compileDebugUnitTestKotlin --no-daemon
```

**Commit per feature:** `align: feature/<x>: normalize Screen/ScreenContent + repo encapsulation`

---

## Phase 8 — App wiring

**Goal:** `app-mobile` (and `app-tv` if TV) apply the right convention plugins, no `:feature:<x>:data` leak, DI/nav wiring matches conventions.

**Steps:**

1. **`app-mobile/build.gradle.kts`**:
   - `plugins { alias(libs.plugins.<project>-android-application) alias(libs.plugins.<project>-android-application-compose) alias(libs.plugins.<project>-android-lint) alias(libs.plugins.<project>-android-flavors) }` — flavors **last**.
   - `android { namespace = "<root-pkg>" ... }`
   - `dependencies { implementation(project(":app-mobile:feature:<x>:ui-mobile")) ... }` — **never** `:data`.

2. **`<Project>Application.kt`** — `startKoin { androidContext(this@<Project>Application); modules(listOf(appModule, environmentModule) + <feature>Modules + ...) }`. Include `environmentModule` if Phase 5 ran.

3. **`MainActivity.kt`** — `setContent { <Project>Theme { DevToolsHost(enabled = BuildConfig.IS_QA) { <Project>NavHost() } } }` if Phase 5 ran. Otherwise omit the `DevToolsHost` wrap.

4. **`navigation/<Project>NavHost.kt`** — `NavDisplay` with `entryProvider { homeEntry(...); searchEntry(...); ... }`. Each `<feature>Entry` is the extension function defined in the feature's ui module.

5. **For mobile + TV**: replicate for `app-tv/build.gradle.kts` + `MainActivity.kt`. TV uses the TV variant of `DevToolsHost`.

**Verification:**

```bash
./gradlew :app-mobile:compileQaDebugKotlin :app-mobile:compileProdDebugKotlin --no-daemon
# if TV:
./gradlew :app-tv:compileQaDebugKotlin :app-tv:compileProdDebugKotlin --no-daemon
```

**Commit:** `align: app wiring`

---

## Phase 9 — Quality + CI

**Goal:** Spotless, detekt, Lint, EditorConfig, GitHub Actions, Dependabot are present and configured.

**Steps:**

1. **Spotless** — in root `build.gradle.kts`:

   ```kotlin
   subprojects {
       apply(plugin = "com.diffplug.spotless")
       configure<com.diffplug.gradle.spotless.SpotlessExtension> {
           kotlin {
               target("**/*.kt")
               targetExclude("**/build/**")
               ktlint(libs.versions.ktlint.get())
           }
           kotlinGradle {
               target("**/*.kts")
               ktlint(libs.versions.ktlint.get())
           }
       }
   }
   ```

2. **detekt** — `.detekt/config.yml` per `init-android-project:references/file-templates.md`. Includes Compose-aware overrides.

3. **Android Lint** — `.lint/config.xml` (minimal baseline). The `<project>.android.lint` plugin reads it.

4. **.editorconfig** — per `init-android-project:references/file-templates.md`. Disables ktlint's `function-naming` (Composables) and `backing-property-naming` (MVI buffers).

5. **`.github/workflows/build.yml`** — runs on push/PR to main: `./gradlew spotlessCheck detekt lint test`.

6. **`.github/dependabot.yml`** — bumps gradle + github-actions ecosystems.

7. **`.gitignore`** — comprehensive Android/Kotlin/Gradle ignore list per `init-android-project:references/file-templates.md`. If the project's existing `.gitignore` is missing entries (`.idea/`, `**/build/`, `local.properties`, `.gradle/`), merge them in.

**Verification:**

```bash
./gradlew spotlessCheck detekt lint --no-daemon
```

If `spotlessCheck` fails, run `./gradlew spotlessApply` and re-stage.

**Commit:** `align: quality + CI`

---

## Phase 10 — Tests

**Goal:** test conventions match — `MainCoroutineRule` default dispatcher, `robolectric.properties`, Compose tests in `src/test/`, per-feature ViewModel + ScreenContent test files present.

**Steps:**

1. **`MainCoroutineRule`** — if its current default is `StandardTestDispatcher`, change to `UnconfinedTestDispatcher`. (Reason: `BaseViewModel.init { ... }` action collector needs eager dispatch for Turbine assertions to not time out.) Existing tests using `StandardTestDispatcher` semantics need explicit `MainCoroutineRule(StandardTestDispatcher())` — leave them.

2. **`src/test/resources/robolectric.properties`** — `sdk=34` (or whatever Robolectric supports). Add to every module that has Compose tests.

3. **`src/androidTest/` → `src/test/`** — for Compose UI tests only. Use `git mv`. Add the Robolectric runner annotation if missing.

4. **Per-feature ViewModel test** — if `<Feature>ViewModelTest.kt` is missing, add a minimal one with one action-handling and one effect-emission test (Turbine + Mockito-Kotlin). Don't add tests where a test file already exists — respect the existing structure.

5. **Per-feature ScreenContent test** — if missing, add a minimal Robolectric Compose test targeting `<Feature>ScreenContent` with one click → captured-action assertion and one state-driven rendering assertion.

**Verification:**

```bash
./gradlew test --no-daemon
```

**Commit:** `align: tests`

---

## Phase 11 — Project-local skills

**Goal:** `.claude/skills/<project>-android-planner/` and `<project>-android-implementer/` exist (or are refreshed) with project-specific identifiers baked in.

**Steps:**

1. Use the same generation logic as `init-android-project` Step 10.9. Generate:

   ```
   .claude/skills/<project>-android-planner/
   ├── SKILL.md
   └── references/
       └── project-architecture.md
   .claude/skills/<project>-android-implementer/
   ├── SKILL.md
   └── references/
       ├── project-architecture.md       # copy or symlink of planner's
       └── compose-authoring.md          # copy + project-adapt from conventions/references/compose-authoring.md
   ```

2. **`SKILL.md` bodies** — short and operational. The planner is ~200–400 lines; the implementer is ~400–600 lines. Both load `references/project-architecture.md` at the start of every run. The implementer's "Compose authoring rules" section is a **pointer block** — never inline the Compose rules; point at `references/compose-authoring.md`.

3. **`references/project-architecture.md`** — captures the module inventory, MVI base type signatures, navigation map, network/persistence stack, Koin module conventions, theme tokens. Generate from the audit's findings — this is the file that "knows" what this specific project looks like.

4. **`references/compose-authoring.md`** — copy `android-project-starter:conventions §Compose authoring rules` (the body, not just the pointer) — adapt by substituting `<Project>` / `<root-pkg>` and example file paths to match this project. Remove the TV section if mobile-only.

5. **If the skills already exist** (e.g. from a previous aligner run or hand-written) — **diff first, ask before overwriting**. Hand-edited skills are a feature, not a bug — respect them. Show the diff; the user picks "apply", "merge", or "skip".

**Verification:** file existence + `<Project>Previews` substitution check (grep for any unresolved `<Project>` / `<root-pkg>` placeholders — there should be none).

**Commit:** `align: project-local planner + implementer skills`

---

## Deferred-decision migrations

These migrations are not in the default phase order. They require explicit user opt-in (Step 4 of the SKILL).

### D1 — DI swap (Hilt → Koin)

High risk, L effort. Touches every `@Inject` constructor and every Dagger module. The aligner only does this if the user explicitly chooses. When migrating:

- One feature at a time. Pick the simplest feature as the worked example.
- For each ViewModel: remove `@HiltViewModel`, remove `@Inject` from the constructor, register in the feature's Koin `Module.kt` via `viewModel { <Feature>ViewModel(get(), get(), ...) }`.
- For each repository: remove Dagger module, add Koin module under `feature/<x>/data/di/<Feature>DataModule.kt`.
- Drop Hilt dependencies from the catalog and module build files.
- Drop `@HiltAndroidApp` from `<Project>Application`, replace with `startKoin { ... }`.

Verification per migrated feature: `:feature:<x>:ui-mobile:compileQaDebugKotlin` + the feature's existing tests.

### D2 — Nav swap (Nav2 → Nav3)

High risk, L effort. Route shape changes (string-based → `@Serializable data class`). When migrating:

- Replace `NavController` + `composable(...)` with `NavDisplay` + `EntryProviderScope<NavKey>.<feature>Entry(...)`.
- Replace string routes with `@Serializable data object/class XxxRoute : NavKey`.
- Replace `navController.navigate("home")` with `backStack.add(HomeRoute)`.
- Cross-feature navigation: through callbacks passed into `<feature>Entry`, never through `NavController` (Nav3 doesn't expose one).

Verification: `./gradlew :app-mobile:compileQaDebugKotlin` + manual smoke test of the nav graph.

### D3 — MVI base swap (custom / Orbit / etc. → canonical `BaseViewModel`)

High risk, M–L effort per feature. When migrating:

- Phase 3 already introduces the canonical `BaseViewModel` alongside.
- One feature at a time: rewrite the ViewModel to extend `BaseViewModel<Action, State, Effect>`, replace state stream with `StateFlow`, replace effects with `Channel`, replace actions with `MutableSharedFlow`.
- Update the feature's tests to use Turbine + `MainCoroutineRule(UnconfinedTestDispatcher())`.
- Leave un-migrated features on the old base. Document the coexistence in the implementer skill.

Verification per migrated feature: `:feature:<x>:ui-mobile:compileDebugUnitTestKotlin :feature:<x>:ui-mobile:test`.

### D4 — `app/` → `app-mobile/` rename

High risk, S effort but invasive. When applying:

- `git mv app app-mobile` (one git operation, full history preserved).
- Update `settings.gradle.kts`: `include(":app-mobile")` (was `:app`).
- Grep for `:app` references in any other build file or doc; rewrite.
- Update `.github/workflows/*.yml` paths if any reference `app/`.

Verification: `./gradlew help :app-mobile:compileQaDebugKotlin`.

This rename is required if Phase 5 introduces flavors and the user wants `app-tv/` later — keeping `app/` while having `app-tv/` is asymmetric and confusing.

---

## Cross-phase rules

- **Never `--amend`.** Each phase is its own commit. If a phase's pre-commit hook rejects, fix the cause, re-stage, and create a *new* commit (the original didn't land).
- **Never `--no-verify`.** Pre-commit hooks exist for a reason; bypassing them lands broken state in source control.
- **Use `git mv` for moves.** Preserve history.
- **Re-read the conventions skill** if you can't remember a canonical shape. Don't reconstruct from memory.
- **The aligner re-shapes; it doesn't rewrite logic.** If a phase tempts you to change *what the code does*, stop — that's an implementer skill task, not an aligner task. Note it for the Boy Scout pass in the implementer skill.
- **Per-feature commits beat one big feature-split commit.** Easier to revert one feature if it breaks.
- **Verification cap is 5 fix-loop iterations.** Halt and ask if you can't make a phase green in 5 tries.
