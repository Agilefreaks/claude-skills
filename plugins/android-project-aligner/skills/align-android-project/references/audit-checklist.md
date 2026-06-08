# Audit Checklist

This is the operational checklist the aligner walks during **Step 2 — Inventory audit**. Each section maps a structural check to (a) a grep / read recipe you can run, (b) the conventions skill section it should match, and (c) the smell → action mapping the gap report uses.

Read this top to bottom on the first pass. On re-runs you can jump to the section the user wants to focus on.

## How to record findings

For every check that diverges from the conventions, record one finding:

```
{ category, severity, effort, risk, conventions_ref, description }
```

- `category`: one of `Foundation`, `Module layout`, `MVI base`, `Screen/ScreenContent`, `Repo encapsulation`, `DI`, `Navigation`, `Persistence/Network`, `Theming`, `Flavors/dev-tools`, `Tooling/CI`, `Tests`, `Project-local skills`.
- `severity`: `blocker` / `important` / `nice-to-have`.
- `effort`: `S` (one file) / `M` (a few files in one module) / `L` (cross-cutting).
- `risk`: `low` / `medium` / `high` (mechanical / behaviorally subtle / semantically invasive).
- `conventions_ref`: pointer like `conventions §The 9 convention plugins`.
- `description`: one-line human summary used in the gap report.

## 1. Foundation

### 1.1 Version catalog

```bash
test -f gradle/libs.versions.toml && echo catalog:present || echo catalog:missing
```

- **Missing entirely** → blocker, L, low. Conventions ref: `conventions §libs.versions.toml structure`. Action: introduce the catalog in Phase 1.
- **Present but no `[plugins]` section** → blocker, S, low. Action: add `[plugins]` with the 9 convention plugin aliases (`version = "unspecified"`) + third-party plugin aliases via `version.ref`.
- **Present but no convention plugin aliases** → blocker, S, low. Action: register all 9 (or 8) in `[plugins]`.

### 1.2 AGP / Kotlin / Compose Compiler / KSP alignment

Read `gradle/libs.versions.toml` (or the relevant build script if no catalog) and extract:
- AGP version (look for `agp = "..."` or `androidGradlePlugin = "..."`)
- Kotlin version (`kotlin = "..."`)
- Compose Compiler plugin version
- KSP version

Apply these compatibility rules (same as `init-android-project` Step 9.5):

- **AGP ≥ 9.2.0** ↔ `android.builtInKotlin=true` (or omitted) + NO manual `apply("org.jetbrains.kotlin.android")` in convention plugins.
- **AGP 9.0.x–9.1.x** ↔ `android.builtInKotlin=false` + manual `apply("org.jetbrains.kotlin.android")`.
- **AGP 8.x** ↔ no explicit `builtInKotlin` setting + manual Kotlin apply.
- **Kotlin == Compose Compiler plugin version**. Mismatch → important, S, low.
- **KSP version pattern `<kotlin major.minor.patch>-<X.Y.Z>`**. Mismatch → blocker, S, low.

### 1.3 build-logic/

```bash
test -d build-logic/convention && echo buildlogic:present || echo buildlogic:missing
ls build-logic/convention/src/main/kotlin/ 2>/dev/null
```

- **Missing** → blocker, L, low. Conventions ref: `conventions §The 9 convention plugins`. Action: Phase 1 scaffolds it entirely.
- **Present but wrong file layout** (helper files like `AndroidCommon.kt`, `KotlinExtensions.kt`, or `package` declarations on the 9 plugin classes) → important, M, medium. Action: restore the canonical layout — inline helpers, remove package declarations.
- **Present but uses `CommonExtension`** (grep `build-logic/convention/src/main/kotlin/` for `CommonExtension`) → blocker on AGP 9+, S, low. Action: rewrite affected plugin to use `ApplicationExtension` or `LibraryExtension` directly.
- **`ProjectExtensions.kt` missing or in the wrong package** → blocker, S, low. Conventions ref: `conventions/references/build-logic.md`. Must be at `build-logic/convention/src/main/kotlin/<project>/android/buildlogic/ProjectExtensions.kt` with `package <project>.android.buildlogic`.
- **Plugin not registered in `build-logic/convention/build.gradle.kts`** (grep `gradlePlugin.plugins` block, count `register(...)` calls — should be 9 or 8) → blocker, S, low.

### 1.4 gradle.properties

```bash
test -f gradle.properties && cat gradle.properties
```

Required:
- `android.useAndroidX=true`
- `kotlin.code.style=official`
- `<project>.qaApiBaseUrl=...` and `<project>.prodApiBaseUrl=...` (if flavors phase will run)
- `android.builtInKotlin=...` matching AGP version (see 1.2)

Forbidden (legacy keys that hard-error on AGP 9+):
- `android.defaults.buildfeatures.buildconfig=...`
- `android.defaults.buildfeatures.aidl=...`
- `android.defaults.buildfeatures.renderscript=...`

→ blocker if any forbidden key is present; important if any required key is missing.

### 1.5 Gradle wrapper

```bash
test -x gradlew && ./gradlew --version 2>&1 | head -5
test -f gradle/wrapper/gradle-wrapper.properties && grep distributionUrl gradle/wrapper/gradle-wrapper.properties
```

- **Missing entirely** → blocker, S, low. Action: materialize per `init-android-project` Step 11.1.
- **Version older than the AGP requires** → important, S, low. Action: bump `distributionUrl` to the latest stable supporting the resolved AGP.

## 2. Module layout

### 2.1 Top-level inventory

```bash
ls -d app* core/* feature/*/* 2>/dev/null | sort
cat settings.gradle.kts 2>/dev/null | grep -E "^include\("
```

Expected modules:
- `app-mobile/` (always)
- `app-tv/` (only if TV)
- `core/common`, `core/data`, `core/designsystem-base` (or `core/designsystem` for mobile-only), `core/designsystem-mobile`, `core/designsystem-tv` (TV only), `core/model`, `core/testing`, `core/ui-mobile`, `core/ui-tv` (TV only)
- For every feature `<x>`: `feature/<x>/data`, `feature/<x>/ui-mobile`, `feature/<x>/ui-tv` (TV only)

Mismatches:
- **`app/` exists but `app-mobile/` does not** → important, M, high (it's a rename touching settings + every cross-module ref). Defer to user.
- **`core/data` missing** → important, M, low. Action: scaffold a skeleton in Phase 5.
- **`core/common` missing** → blocker, M, low (every other module's MVI depends on it). Phase 3.
- **`core/testing` missing** → important, S, low. Phase 3.
- **`core/designsystem*` missing** → important, M, low. Phase 4.
- **`core/ui-*` missing** → important, M, low. Phase 4.
- **`feature/<x>/` is a single module (not split into data/ui-mobile)** → important, M, medium. Phase 6.

### 2.2 settings.gradle.kts shape

```bash
cat settings.gradle.kts
```

Should include:
- `pluginManagement { includeBuild("build-logic") }`
- `rootProject.name = "<project>"`
- `include(":<every module>")`

→ blocker if `includeBuild("build-logic")` is missing once Phase 1 lands.

## 3. MVI base types

```bash
find . -path '*/core/common/src/main/kotlin*' -type f \( -name 'BaseViewModel.kt' -o -name 'ViewAction.kt' -o -name 'ViewState.kt' -o -name 'ViewSideEffect.kt' \) 2>/dev/null
```

Check each file against the conventions skill `references/mvi-base.md`:
- `ViewAction` is a marker interface (`interface ViewAction`).
- `ViewState` is a marker interface.
- `ViewSideEffect` is a marker interface.
- `BaseViewModel<Action: ViewAction, State: ViewState, Effect: ViewSideEffect>` has:
  - `_state: MutableStateFlow<State>` exposed via `asStateFlow()`.
  - `_effects: Channel<Effect>` exposed via `receiveAsFlow()`.
  - `_actions: MutableSharedFlow<Action>` with `extraBufferCapacity = 64`.
  - `init { viewModelScope.launch { _actions.onEach(::onAction).collect() } }`.
  - `protected abstract suspend fun onAction(action: Action)`.
  - `protected fun setState(reducer: State.() -> State)`.
  - `protected suspend fun setEffect(effect: Effect)`.
  - `fun setAction(action: Action)` (non-suspending).
- No `import <root-pkg>.core.View*` lines inside `BaseViewModel.kt` (same package — they resolve automatically).

Mismatches:
- **Missing entirely** → blocker, M, low. Phase 3.
- **Shape diverges** (e.g. state is a `Flow<State>` instead of `StateFlow`, or actions are a `Channel` instead of `MutableSharedFlow`) → important, M, medium. Action: replace with canonical shape. If existing ViewModels rely on the divergent shape, scope it to "introduce canonical alongside, migrate feature-by-feature."
- **Different MVI framework** (Orbit, MVIKotlin, custom) → important, L, high. Defer to user — default keep existing, introduce canonical alongside, migrate one example feature, document deviation in implementer skill.

## 4. Screen / ScreenContent split

For each feature ui module under `feature/<x>/ui-mobile/` (and `ui-tv/` if TV):

```bash
ls feature/<x>/ui-mobile/src/main/kotlin/**/components/*.kt 2>/dev/null
```

Per feature:
- `<Feature>Screen.kt` exists?
- `<Feature>ScreenContent.kt` exists as a separate file (NOT a nested function inside Screen)?
- ScreenContent signature: grep for `fun <Feature>ScreenContent(` — must have `modifier: Modifier = Modifier` as the first optional parameter, `state: <FeatureState>` (or whatever the project's state type is) required, `onAction: (<FeatureAction>) -> Unit` required. No bare callbacks (`onClick: () -> Unit`, `onSubmit: () -> Unit`, etc.).
- ScreenContent body has NO `koinViewModel(`, `LocalContext.current`, `rememberLauncherForActivityResult`, `LocalLifecycleOwner.current` references.
- At least one `@<Project>Previews`-annotated composable at the bottom of `<Feature>ScreenContent.kt` that constructs a sample `State` and passes `onAction = {}`. The annotation may also be written as `@Preview` if the project doesn't yet have a custom multi-preview annotation — in that case flag it.
- **Never stack** `@<Project>Previews` and `@Preview` on the same function — if both are present on the same preview composable, that's a duplicate render.

Mismatches:
- **ScreenContent missing (everything inlined in Screen)** → important, M, low. Phase 7. Action: extract the UI body into a separate file with the canonical signature; Screen retains only wiring.
- **Bare callbacks in ScreenContent signature** → important, S, medium. Phase 7. Action: replace with typed actions dispatched via `onAction`. Add the new action types to the feature's `<Feature>Action` sealed interface.
- **Koin / Activity deps in ScreenContent** → blocker for unit testability, S, medium. Phase 7. Action: move dependencies up into Screen.
- **No preview** → important, S, low. Phase 7.
- **Stacked `@<Project>Previews` + `@Preview`** → nice-to-have, S, low. Phase 7 cleanup.

## 5. Repository encapsulation

For each feature `<x>` with a `data/` module:

```bash
find feature/<x>/data/src/main/kotlin -type f -name '*.kt' | xargs grep -l 'class.*Repository' 2>/dev/null
grep -E '^(public |internal |private )?(open )?(abstract )?(class|interface)' feature/<x>/data/src/main/kotlin/**/*Repository*.kt
```

Per feature:
- `<Feature>Repository.kt` is a public `interface` (or `sealed interface` etc., as long as it's not `internal`).
- `<Feature>RepositoryImpl.kt` is declared `internal class` (or `private` — but `internal` is the canonical form).
- `di/<Feature>DataModule.kt` exposes `val <feature>DataModule = module { ... }` and `<feature>DataModule` is `public` (no modifier or explicit `public`).
- Grep the app modules for `:feature:<x>:data` references:
  ```bash
  grep -E ':feature:<x>:data' app-mobile/build.gradle.kts app-tv/build.gradle.kts 2>/dev/null
  ```
  → must be empty. If a match is found, that's a leak.

Mismatches:
- **Impl is public** → important, S, low. Phase 7. Action: prepend `internal` modifier.
- **`<feature>DataModule` is not exposed (or named differently)** → important, S, low. Phase 7. Action: rename to canonical, mark public.
- **App module depends on `:feature:<x>:data`** → important, S, medium. Phase 8. Action: remove the dependency; the ui module's `<feature>Modules` aggregator already includes `<feature>DataModule`.

## 6. DI library

```bash
grep -r 'org.koin' gradle/libs.versions.toml app-mobile/build.gradle.kts 2>/dev/null
grep -r 'dagger.hilt' gradle/libs.versions.toml app-mobile/build.gradle.kts 2>/dev/null
grep -r '@Inject' --include='*.kt' . 2>/dev/null | head -3
```

Decision tree:
- **Koin present, no Hilt** → conventions-compliant. Continue.
- **Hilt present (with or without Koin)** → important, L, high. Defer to user. Default: leave Hilt intact, mark deviation in implementer skill. If user opts to migrate, the migration is its own multi-phase project; the aligner can scope a single feature as a worked example.
- **No DI (manual constructor wiring)** → important, L, medium. Phase 8. Action: introduce Koin, scaffold `<Project>Application.startKoin { ... }`, per-feature `Module.kt` + `<Feature>Modules.kt` aggregator.
- **Other DI framework (Anvil, Kodein, manual)** → important, L, high. Defer to user.

## 7. Navigation

```bash
grep -r 'androidx.navigation3' gradle/libs.versions.toml app-mobile/build.gradle.kts 2>/dev/null
grep -r 'androidx.navigation:navigation-compose' gradle/libs.versions.toml app-mobile/build.gradle.kts 2>/dev/null
grep -rE 'NavHost\(|composable\(' --include='*.kt' . 2>/dev/null | head -3
```

Decision tree:
- **Nav3 present** → check shape: `EntryProviderScope<NavKey>.<feature>Entry(...)` per feature, `NavDisplay` in the app, `@Serializable` route objects implementing `NavKey`, no string-based routes. Mismatches are S–M, low–medium.
- **Nav2 (`androidx.navigation:navigation-compose`)** → important, L, high. Defer to user. Migration to Nav3 is its own project.
- **Custom navigation** → important, L, high. Defer to user.

## 8. Persistence + Network

### Room

```bash
grep -r 'androidx.room' gradle/libs.versions.toml 2>/dev/null
find . -path '*/build/*' -prune -o -name '*.kt' -print 2>/dev/null | xargs grep -l '@Database\|@Dao\|@Entity' 2>/dev/null | head -5
```

If Room is used:
- `AndroidRoomConventionPlugin.kt` registered in build-logic? → important, S, low if missing.
- Module that uses Room applies `<prefix>.android.room`? → important, S, low.
- `room { schemaDirectory("$projectDir/schemas") }` set? → nice-to-have, S, low.

### Network

```bash
grep -r 'retrofit\|apollo\|ktor' gradle/libs.versions.toml 2>/dev/null
```

Conventions don't mandate a network library. Check that whatever's present is wired through `core/data` and not duplicated across features. Smells:
- Each feature directly imports Retrofit/OkHttp → important, M, low. Phase 5+8 — central `core/data` networking.
- Network interceptor registered ad-hoc per feature → same.

## 9. Theming

```bash
find core/designsystem* -name '*.kt' 2>/dev/null | head -10
grep -r '<Project>Theme\|<Project>Previews' --include='*.kt' . 2>/dev/null | head -5
grep -rE 'Color\(0x[0-9A-Fa-f]{8}\)' --include='*.kt' feature/ 2>/dev/null | head -5
grep -rE 'TextStyle\(|fontSize = ' --include='*.kt' feature/ 2>/dev/null | head -5
```

(Substitute `<Project>` with the actual class prefix from Step 1.4.)

Checks:
- `<Project>Theme` composable in `core/designsystem-mobile`? → important, S, low if missing.
- `@<Project>Previews` multi-preview annotation? → important, S, low if missing.
- Hardcoded colors in feature modules → important, S–M, low. Boy Scout phase or Phase 4. Action: promote to design-system tokens.
- Hardcoded text styles → same.

## 10. Flavors + dev-tools

```bash
grep -rE 'productFlavors|flavorDimensions' app-mobile/build.gradle.kts core/data/build.gradle.kts 2>/dev/null
grep -rE 'BuildConfig\.(IS_QA|API_BASE_URL)' --include='*.kt' . 2>/dev/null | head -5
find . -path '*/env/*' -name '*.kt' 2>/dev/null | head -10
find . -path '*/dev/*' -name '*.kt' 2>/dev/null | head -10
```

Checks:
- `qa` (default) + `prod` flavors declared in `app-mobile`, `app-tv` (if TV), and `core/data`? → important, M, medium if missing. Phase 5.
- `BuildConfig.IS_QA` + `BuildConfig.API_BASE_URL` emitted? → blocker for dev-tools, S, low.
- `EnvironmentConfig` + DataStore impl in `core/data/env/`? → important, M, low if missing.
- `EnvironmentBaseUrlInterceptor` registered first in the OkHttp client chain? → important, S, medium if registered anywhere else (it must run first).
- `ShakeDetector` + `DevToolsBroadcastListener` + `EnvSelectorDialog` + `DevToolsHost` in `core/ui-mobile/dev/`? → important, M, low if missing.
- `MainActivity` wraps content in `DevToolsHost(enabled = BuildConfig.IS_QA)`? → important, S, low.

## 11. Tooling + CI

```bash
grep -r 'spotless' gradle/libs.versions.toml build.gradle.kts 2>/dev/null
test -f .detekt/config.yml && echo detekt:present || echo detekt:missing
test -f .lint/config.xml && echo lint:present || echo lint:missing
test -f .editorconfig && echo editorconfig:present || echo editorconfig:missing
ls .github/workflows/ 2>/dev/null
test -f .github/dependabot.yml && echo dependabot:present || echo dependabot:missing
```

Each missing piece → important–nice-to-have, S, low. Phase 9. Bodies are in `init-android-project:references/file-templates.md`.

## 12. Test conventions

```bash
find . -path '*/core/testing*' -name 'MainCoroutineRule.kt' 2>/dev/null
grep -r 'UnconfinedTestDispatcher\|StandardTestDispatcher' --include='*.kt' core/testing/ 2>/dev/null
find . -path '*/src/androidTest*' -name '*Test.kt' 2>/dev/null | head -3
find . -path '*/src/test/resources*' -name 'robolectric.properties' 2>/dev/null | head -3
```

Checks:
- `MainCoroutineRule` defaults to `UnconfinedTestDispatcher`? (otherwise Turbine times out at 3s) → important, S, medium.
- Compose UI tests in `src/androidTest/` instead of `src/test/`? → important, M, low. Phase 10. Action: move with `git mv`.
- `src/test/resources/robolectric.properties` with `sdk=34`? → important, S, low.
- Tests target `<Feature>ScreenContent`, not `<Feature>Screen`? Grep test files for `<Feature>Screen(` (open paren) without `Content` suffix — if a Compose test instantiates Screen, that's wrong. → important, S, medium.

## 13. Project-local skills

```bash
test -d .claude/skills && ls .claude/skills/ 2>/dev/null
```

- `<project>-android-planner/SKILL.md` present? → important, M, low if missing. Phase 11.
- `<project>-android-implementer/SKILL.md` present? → important, M, low if missing. Phase 11.
- If present: read their first 20 lines and check the front-matter `name:` matches `<project>-android-planner` / `<project>-android-implementer`. If the names are stale (older project name in the file vs. current `rootProject.name`), flag for refresh.

## Quick-grep cheat sheet

```bash
# Find every "hardcoded color" violation
grep -rE 'Color\(0x[0-9A-Fa-f]{8}\)' --include='*.kt' feature/ app-mobile/ app-tv/ 2>/dev/null

# Find every "Screen has koinViewModel" smell — that's correct, but ScreenContent must not.
# So check the inverse: ScreenContent files with koinViewModel.
grep -rE 'koinViewModel\(' --include='*ScreenContent.kt' . 2>/dev/null

# Find every public RepositoryImpl
grep -rE '^(public )?class .+RepositoryImpl' --include='*.kt' feature/*/data 2>/dev/null

# Find every app→data dependency leak
grep -rE ':feature:[^:]+:data' app-mobile/build.gradle.kts app-tv/build.gradle.kts 2>/dev/null

# Find every CommonExtension misuse (AGP 9 broke this)
grep -r 'CommonExtension' build-logic/ 2>/dev/null

# Find every legacy gradle.properties key (AGP 9 hard-errors)
grep -E 'android\.defaults\.buildfeatures\.' gradle.properties 2>/dev/null
```
