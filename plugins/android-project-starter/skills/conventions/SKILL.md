---
name: conventions
description: >
  Architectural conventions for Android projects scaffolded by the android-project-starter plugin.
  The single source of truth for module layout, MVI shape, build-logic plugin responsibilities,
  Navigation 3 wiring, Koin conventions, theming, test patterns, version-lookup sources, and
  tooling defaults. Load this skill whenever scaffolding or modifying an Android project
  generated (or to be generated) by this plugin. The init wizard defers to this skill for
  every file shape — it is the single source of truth for "what good looks like."
---

# Android Project Conventions

This skill is the conventions bible for projects scaffolded by `android-project-starter`. It is **not** a wizard — it describes the target architecture so other skills can generate code that matches.

Whenever you generate or modify a file in a project scaffolded by this plugin, follow what's written here. If you see a conflict between this file and the user's instructions, the user wins — but flag the conflict.

## Top-level layout

A scaffolded project always has this shape at the root:

```
<project-root>/
├── .claude/
│   └── skills/
│       ├── <project>-android-planner/SKILL.md
│       └── <project>-android-implementer/SKILL.md
├── .editorconfig
├── .gitignore
├── .lint/
│   └── config.xml                          # Android Lint baseline rules
├── .github/
│   └── workflows/                          # CI (build, lint, spotless, detekt, test)
├── app-mobile/                             # phone app module (always)
├── app-tv/                                 # tv app module (only if TV picked)
├── build-logic/
│   ├── settings.gradle.kts
│   └── convention/
│       ├── build.gradle.kts
│       └── src/main/kotlin/                # the 9 convention plugins
├── core/
│   ├── common/                             # BaseViewModel + ViewAction/State/Effect
│   ├── data/                               # session, network, persistence wiring
│   ├── designsystem-base/                  # theme tokens (colors, typography, spacing)
│   ├── designsystem-mobile/                # phone-shaped components + theme
│   ├── designsystem-tv/                    # (TV-only) TV-shaped components + theme
│   ├── model/                              # pure-Kotlin domain models
│   ├── testing/                            # MainCoroutineRule + test helpers
│   ├── ui-mobile/                          # phone-shaped shared composables
│   └── ui-tv/                              # (TV-only) TV-shaped shared composables
├── feature/<feature>/
│   ├── data/                               # repository + network/db wiring for the feature
│   ├── ui-mobile/                          # phone UI (MVI + Compose)
│   └── ui-tv/                              # (TV-only) TV UI — independent MVI + ViewModel + Compose under `.tv` sub-package
├── gradle/
│   └── libs.versions.toml
├── build.gradle.kts                        # only `apply false` plugin declarations
├── settings.gradle.kts
├── gradle.properties
└── local.properties                        # gitignored
```

**Rules:**
- `feature/<feature>/data/` exists for every feature, even if it just exposes a `Module.kt` with a stub repository. Don't merge data into the ui module.
- TV modules (`app-tv`, `core/designsystem-tv`, `core/ui-tv`, `feature/<x>/ui-tv`) only exist if the user picked TV at wizard time.
- `core/designsystem-base` holds the parts shared between mobile and TV (color tokens, typography tokens, spacing). The `-mobile` and `-tv` variants hold form-factor-specific components.
- If only mobile is picked, drop the `-base` suffix and put everything in `core/designsystem` + `core/ui`. The split adds noise when there's only one form factor.

## The 9 convention plugins

All build configuration lives in `build-logic/convention/`. Modules apply a **single** convention plugin instead of three. Every plugin reads versions and library coordinates from `libs.versions.toml` via the `libs` extension (`ProjectExtensions.kt`).

### Hard rules — read before generating any plugin file

These are non-negotiable. Diverging from them produces silent build failures on AGP 9+:

1. **The ONLY files allowed under `build-logic/convention/src/main/kotlin/`** are exactly the 9 plugin classes (in the default package) plus `ProjectExtensions.kt` (in its own package). Layout:
   ```
   build-logic/convention/src/main/kotlin/
   ├── AndroidApplicationConventionPlugin.kt         # default package
   ├── AndroidApplicationComposeConventionPlugin.kt  # default package
   ├── AndroidLibraryConventionPlugin.kt             # default package
   ├── AndroidLibraryComposeConventionPlugin.kt      # default package
   ├── AndroidFeatureConventionPlugin.kt             # default package
   ├── AndroidFlavorsConventionPlugin.kt             # default package
   ├── AndroidLintConventionPlugin.kt                # default package
   ├── AndroidRoomConventionPlugin.kt                # default package — only if Room was picked
   ├── JvmLibraryConventionPlugin.kt                 # default package
   └── <project>/android/buildlogic/
       └── ProjectExtensions.kt                      # package <project>.android.buildlogic
   ```
   **DO NOT** invent helper files (`AndroidCommon.kt`, `AndroidConfig.kt`, `KotlinExtensions.kt`, etc.) to share code between plugins. Even when Application and Library plugins look like they share configuration, **inline the duplication**. The extraction has burned every project that tried it because of the AGP 9 type-system issue described below.

2. **No `package` declaration on the 9 plugin classes.** They go in the default (root) package — the plugin id resolution in Gradle relies on top-level class names. `ProjectExtensions.kt` is the **only exception**: it declares `package <project>.android.buildlogic` and lives in a matching subdirectory. Every plugin that uses the catalog adds `import <project>.android.buildlogic.libs` near the top so calls like `libs.findLibrary("…")` and `libs.findVersion("…")` resolve to our `VersionCatalog` accessor, not Gradle's auto-generated `Project.libs: LibrariesForLibs` (which would otherwise collide and cause "Overload resolution ambiguity" or silent wrong-symbol pick).

3. **Use `ApplicationExtension` and `LibraryExtension` directly — never `CommonExtension`.** In AGP 9 `CommonExtension` is no longer parameterized (the `<*>` style fails: "No type arguments expected for 'interface CommonExtension : ExtensionAware'") and even when typed correctly, `apply { compileSdk = ... }` can't resolve properties like `compileSdk`, `defaultConfig`, `compileOptions` through `CommonExtension`. Configure each extension separately in its own plugin. Yes, this duplicates ~6 lines between `AndroidApplicationConventionPlugin` and `AndroidLibraryConventionPlugin` — that duplication is the conventional Now-In-Android pattern and is correct.

4. **The build-logic Gradle plugin IDs use the project name prefix verbatim.** For project name `test`, plugin IDs are `test.android.application`, `test.android.library`, etc. The catalog plugin aliases and `gradlePlugin.plugins { ... }` block must use the same id, kebab/dot form.

5. **`build-logic/convention/build.gradle.kts` must register all 9 plugins** (or 8 if Room was skipped) under `gradlePlugin { plugins { register("...") { id = "..."; implementationClass = "..." } } }`. Forgetting one means the catalog alias resolves to "unspecified" at apply time.

| Plugin id | Class | What it does |
|---|---|---|
| `<project>.android.application` | `AndroidApplicationConventionPlugin` | Applies `com.android.application` + `kotlin.android`, configures `compileSdk`/`minSdk`/`targetSdk` from the catalog, sets Java 21 source/target. |
| `<project>.android.application.compose` | `AndroidApplicationComposeConventionPlugin` | Applies the Compose compiler plugin, enables `buildFeatures.compose`, adds the Compose BOM + ui/material3/tooling/test deps to an application module. |
| `<project>.android.library` | `AndroidLibraryConventionPlugin` | Applies `com.android.library` + `kotlin.android`, configures `compileSdk`/`minSdk`, Java 21, and adds the standard library deps (coroutines, serialization, koin-core) plus the test stack (junit, mockito-kotlin, truth, coroutines-test, turbine) and a `testImplementation(project(":core:testing"))` for every library except `:core:testing` itself. |
| `<project>.android.library.compose` | `AndroidLibraryComposeConventionPlugin` | Applies the Compose compiler plugin, enables `buildFeatures.compose`, adds the Compose BOM + ui/tooling/test deps to a library module. |
| `<project>.android.feature` | `AndroidFeatureConventionPlugin` | The aggregator for feature ui modules. Applies `<project>.android.library` + `<project>.android.library.compose` + `<project>.android.lint`, plus the feature-extra deps: `lifecycle-runtime-ktx`, `lifecycle-runtime-compose`, `lifecycle-viewmodel-compose`, `navigation3-runtime`, `kotlinx-collections-immutable`, `koin-androidx-compose`. Every `feature/<x>/ui-mobile` module applies just this one plugin. |
| `<project>.android.flavors` | `AndroidFlavorsConventionPlugin` | Adds the `env` product-flavor dimension with `qa` (default) and `prod`. Applied **only** to `app-mobile`, `app-tv`, and `core/data` — feature modules and other core modules do not flavor. Detects whether the target is an Application or Library and configures the correct extension. Emits `BuildConfig.IS_QA` (boolean) and `BuildConfig.API_BASE_URL` (String) on every variant. |
| `<project>.android.lint` | `AndroidLintConventionPlugin` | Sets `lint { abortOnError = true; checkDependencies = true; lintConfig = rootProject.file(".lint/config.xml") }` on whichever Android extension is present (Application or Library). |
| `<project>.android.room` | `AndroidRoomConventionPlugin` | Applies `androidx.room` + `com.google.devtools.ksp`, sets `room { schemaDirectory("$projectDir/schemas") }`, adds `room-runtime`, `room-ktx`, and `ksp(room-compiler)`. Only generated if Room was picked. |
| `<project>.jvm.library` | `JvmLibraryConventionPlugin` | Pure-Kotlin (JVM) library — used by modules with no Android dependency (e.g. `:core:model`). Java 21, jvmToolchain(21), standard test stack. |

### Canonical plugin file contents

**Read `references/build-logic.md`** for the verbatim source of every convention plugin, plus `ProjectExtensions.kt`, `build-logic/convention/build.gradle.kts`, `build-logic/settings.gradle.kts`, the root `settings.gradle.kts`, and the AGP-version-sensitive `gradle.properties` toggle. Generate exactly those shapes — substitute `<project>` with the plugin-id prefix, do not refactor, do not extract helpers, and keep the package layout exactly as the reference shows (the 9 plugin classes in the default package; `ProjectExtensions.kt` alone in `package <project>.android.buildlogic` so its `libs` accessor doesn't collide with Gradle's auto-generated `Project.libs: LibrariesForLibs`).

Key non-obvious rules from that reference (so the conventions skill can be linked safely without loading it):

- **Each plugin configures `ApplicationExtension` or `LibraryExtension` directly — never `CommonExtension`** (AGP 9 broke the parametric form). The 6-line duplication between Application and Library plugins is intentional.
- **`AndroidApplicationConventionPlugin` must enable `buildFeatures { buildConfig = true }`** — `BuildConfig.DEBUG` gates the scaffolded Timber init. AGP 9 defaults `buildConfig` to false and the legacy `gradle.properties` escape hatch (`android.defaults.buildfeatures.buildconfig=true`) was removed in AGP 9 — enabling it via gradle.properties hard-errors at configuration time. The convention plugin is the only place to enable it.
- **Both Compose plugins must set `testOptions.unitTests.isIncludeAndroidResources = true`** — Robolectric `createComposeRule()` fails to find `ComponentActivity` otherwise.
- **`AndroidFlavorsConventionPlugin` apply order matters** — it reads the extension configured by `<project>.android.application` (or `.library`), so it must be **last** in the `plugins { }` block of `app-mobile`, `app-tv`, or `core/data`. It is not applied anywhere else.
- **AGP version table:** `9.0.x – 9.1.x` requires `android.builtInKotlin=false` + manual `apply("org.jetbrains.kotlin.android")` in both Application/Library plugins. `9.2.0+` requires `android.builtInKotlin=true` (or omit) and NO manual Kotlin apply (AGP auto-applies it; manual apply hard-errors).
- **Never add legacy `android.defaults.buildfeatures.*` keys** to `gradle.properties` — `buildconfig`, `aidl`, `renderscript` were all removed in AGP 9 and fail at configuration time.

## AndroidManifest.xml

**Read `references/manifests.md`** for the canonical shapes of library manifests (empty + permission-declaring), application manifest (`app-mobile`), and the TV variant (`app-tv` with `LEANBACK_LAUNCHER` + banner).

Non-negotiable rules:

1. **Always declare `xmlns:android="http://schemas.android.com/apk/res/android"` on `<manifest>`.** Without it, every `android:*` attribute fails to resolve and AAPT errors out (`attribute 'android:name' not found`).
2. **Library manifests are minimal** — no `<application>` tag, no `package` attribute. AGP derives the package from the module's `namespace` in `build.gradle.kts`.
3. **Library modules still need a manifest file even when they contribute nothing** — generate the single-line self-closing form rather than omitting the file.

## MVI base types

**Read `references/mvi-base.md`** for the verbatim source of `ViewAction`, `ViewState`, `ViewSideEffect`, and `BaseViewModel`. All four live in `core/common/src/main/kotlin/<root-pkg>/core/`, package `<root-pkg>.core`. Generate them exactly — no deviation.

Key contract (so this skill stays self-contained):

- State is a hot `StateFlow<State>` (`_state`/`asStateFlow()`); effects flow through a single-consumer `Channel<Effect>` exposed as `Flow<Effect>` via `receiveAsFlow()`.
- Actions are buffered (`MutableSharedFlow<Action>(extraBufferCapacity = 64)`) so `setAction` is non-suspending.
- `BaseViewModel.init { viewModelScope.launch { _actions.onEach(::onAction).collect() } }` is what drains actions — this is why `MainCoroutineRule` must default to `UnconfinedTestDispatcher` (see Test conventions section).
- Subclasses override `protected abstract suspend fun onAction(action: Action)`. Mutate state via `setState { copy(...) }`; emit effects via `suspend setEffect(...)`.
- **Do NOT add `import <root-pkg>.core.View*` lines inside `BaseViewModel.kt`** — `ViewAction`/`ViewState`/`ViewSideEffect` live in the same package and resolve automatically.

## Feature module shape

Each feature has **one shared data module** and **one ui module per form factor** (`ui-mobile`, and `ui-tv` when TV was picked). **Every ui module owns its own ViewModel, its own MVI trio, its own Routes/Entry, its own Compose layer, and its own Koin module.** Mobile and TV diverge on focus model, layout, input (touch vs D-pad), and often state shape — sharing a ViewModel forces lowest-common-denominator design and ends in baggage. Sharing happens at the data layer and below, not the ui layer.

### Mobile-only project layout

```
feature/<feature>/
├── data/src/main/kotlin/<root-pkg>/feature/<feature>/data/
│   ├── <Feature>Repository.kt               # public interface (only)
│   ├── <Feature>RepositoryImpl.kt           # internal class
│   └── di/<Feature>DataModule.kt            # val <feature>DataModule (public)
└── ui-mobile/src/main/kotlin/<root-pkg>/feature/<feature>/
    ├── <Feature>Route.kt                    # @Serializable data object/class : NavKey
    ├── <Feature>Entry.kt                    # fun EntryProviderScope<NavKey>.<feature>Entry(...)
    ├── <Feature>Modules.kt                  # val <feature>Modules = listOf(<feature>Module, <feature>DataModule)
    ├── Module.kt                            # val <feature>Module = module { viewModel { ... } }
    ├── <Feature>ViewModel.kt                # BaseViewModel subclass
    ├── mvi/{<Feature>Action,State,Effect}.kt
    └── components/{<Feature>Screen,<Feature>ScreenContent}.kt
```

### Mobile + TV project layout

```
feature/<feature>/
├── data/src/main/kotlin/<root-pkg>/feature/<feature>/data/
│   ├── <Feature>Repository.kt               # public interface — shared between mobile and tv
│   ├── <Feature>RepositoryImpl.kt           # internal class
│   └── di/<Feature>DataModule.kt            # val <feature>DataModule — public, shared
├── ui-mobile/src/main/kotlin/<root-pkg>/feature/<feature>/
│   ├── <Feature>Route.kt                    # mobile route
│   ├── <Feature>Entry.kt                    # mobile entry
│   ├── <Feature>Modules.kt                  # val <feature>Modules = listOf(<feature>Module, <feature>DataModule)
│   ├── Module.kt                            # val <feature>Module — mobile VM binding
│   ├── <Feature>ViewModel.kt                # MOBILE ViewModel
│   ├── mvi/{<Feature>Action,State,Effect}.kt  # mobile MVI trio
│   └── components/{<Feature>Screen,<Feature>ScreenContent}.kt
└── ui-tv/src/main/kotlin/<root-pkg>/feature/<feature>/tv/
    ├── <Feature>Route.kt                    # TV route (different NavKey)
    ├── <Feature>Entry.kt                    # TV entry
    ├── <Feature>Modules.kt                  # val <feature>Modules = listOf(<feature>Module, <feature>DataModule)
    ├── Module.kt                            # val <feature>Module — TV VM binding
    ├── <Feature>ViewModel.kt                # TV ViewModel (independent from mobile)
    ├── mvi/{<Feature>Action,State,Effect}.kt  # TV MVI trio (independent)
    └── components/{<Feature>Screen,<Feature>ScreenContent}.kt
```

**Key rules:**
- TV classes live under the `.tv` sub-package so class names can stay the same as mobile (`<Feature>ViewModel`, `<Feature>Route`, etc.) without collision. `app-mobile` imports from `<root-pkg>.feature.<feature>.*`, `app-tv` imports from `<root-pkg>.feature.<feature>.tv.*`.
- Both ui modules expose `val <feature>Modules: List<Module>` (same identifier, different packages). Each app's `Application` class imports the right one and aggregates them.
- The data module is shared by both ui modules — never duplicated. Both ui modules depend on `:feature:<feature>:data` with **`implementation` scope only** (never `api`) so the data module's classes don't leak onto the app's compile classpath transitively.
- The data module is **internal to the feature**. App modules (`app-mobile`, `app-tv`) never list `:feature:<feature>:data` as a dependency — they only depend on the feature's ui module(s). The app reaches the data layer indirectly through the ui module's aggregated `<feature>Modules` (which already bundles `<feature>DataModule` for Koin).
- Inside the data module, only the `Repository` interface and the `<feature>DataModule` Koin value are `public`. Repository implementations and any other concrete data-layer classes are marked `internal` so they cannot be referenced from outside the data module — not even from ui modules, which work against the interface. See `references/data-encapsulation.md`.
- Each ui module has its own Koin `Module.kt` with its own `viewModel { <Feature>ViewModel(...) }` binding. The data module is included in both aggregators (Koin de-duplicates module registration if both apps were to load both — but they don't, since mobile and TV are separate APKs).
- Tests live in each ui module's own `src/test/kotlin/` — the mobile ViewModel/Compose tests test the mobile shape, the TV ones test the TV shape.

### Naming

- Route data object for parameterless routes: `@Serializable data object HomeRoute : NavKey` (mobile) and `@Serializable data object HomeRoute : NavKey` in the `.tv` package for TV.
- Route data class for routes with args: `@Serializable data class SeriesDetailRoute(val id: String) : NavKey`.
- Entry function: `fun EntryProviderScope<NavKey>.<feature>Entry(...)` — kebab-camel of the feature. Same name in both packages; the import path tells them apart.
- Modules aggregator (`<Feature>Modules.kt`) in each ui module: `val <feature>Modules: List<Module>` combining that module's `<feature>Module` + the shared `<feature>DataModule`.
- Data module's Koin module: `<feature>DataModule`, lives in `feature/<feature>/data/.../<feature>/data/di/<Feature>DataModule.kt`. Singular — never split per form factor.

### Required deps for `feature/<feature>/ui-mobile/build.gradle.kts` and `ui-tv/build.gradle.kts`

**Read `references/feature-build-files.md`** for the verbatim `plugins { }`, `android { namespace = ... }`, and `dependencies { }` blocks for both ui modules. Both apply the same `<project>.android.feature` convention plugin; only the design-system / ui module references differ. Neither applies `<project>.android.flavors`. The ui→data dependency is `implementation` scope (never `api`), so the data layer stays off the app's transitive compile classpath.

### Data module encapsulation

**Read `references/data-encapsulation.md`** for the verbatim `<Feature>Repository.kt` (public interface), `<Feature>RepositoryImpl.kt` (`internal class`), `<Feature>DataModule.kt` (Koin), and the canonical `app-mobile/build.gradle.kts` dependency block showing the "never depend on `:feature:<x>:data`" rule. The reference also covers why both Gradle scope and Kotlin visibility are needed — they're reinforcing mechanisms, not redundant ones.

## Screen / ScreenContent contract

**Always split into two files.** `<Feature>Screen.kt` handles ViewModel + effect routing; `<Feature>ScreenContent.kt` is pure UI.

**Read `references/screen-and-nav.md`** for the verbatim Screen and ScreenContent samples (including `@<Project>Previews` placement and the right-vs-wrong `@Preview` stacking example).

**Rules (every feature, no exceptions):**

1. **Two files always.** `<Feature>Screen.kt` and `<Feature>ScreenContent.kt` are separate files in `components/`. Never inline ScreenContent into Screen.
2. **ScreenContent signature is `(modifier, state, onAction)`** — `modifier: Modifier = Modifier` is the **first optional parameter**, state required, onAction required. Required params first, then `modifier`, then other optionals. Never add bare callbacks (`onClick`, `onSubmit`, `onItemClick`). All clicks dispatch typed Actions; cross-feature navigation flows through Effects that the Screen's `HandleEffects` block translates into host callbacks.
3. **ScreenContent has no Koin / Activity / host dependencies.** No `koinViewModel`, no `LocalContext.current`, no `rememberLauncherForActivityResult`. Those live in the Screen. This is what makes ScreenContent testable and previewable.
4. **At least one `@<Project>Previews`-annotated composable** lives at the bottom of `<Feature>ScreenContent.kt`, constructs a sample `State`, and passes `onAction = {}`. Add **separate previews for each meaningfully different state** — at minimum, the content state; add loading, empty, and error previews when the screen renders them differently. **Use `@<Project>Previews` alone — never stack `@Preview` on top of it.** The custom annotation already aggregates `@Preview` for light + dark (and any other configured variants). Adding `@Preview` duplicates the render.
5. **Compose UI tests target `<Feature>ScreenContent`, not `<Feature>Screen`.** Robolectric tests in `src/test/kotlin/` set content via `composeTestRule.setContent { <Feature>ScreenContent(state = …, onAction = { capturedActions.add(it) }) }`. Test action dispatch (clicks → captured actions) and state-driven conditional rendering. Don't try to test `<Feature>Screen` — it depends on Koin and won't instantiate in a unit test.
6. **The Screen tests are: there are no Screen tests.** Tests run against ScreenContent for UI, against ViewModel for logic. The Screen is just wiring — covered by the integration on-device run.

These rules are checked structurally by the wizard's Step 11.2 self-check before gradle runs. A feature that violates any of them blocks the build-success gate.

## Compose authoring rules

**Read `references/compose-authoring.md` before touching any Compose file.** That reference is the operational source of truth for Compose in this project. It covers: state management (where state lives, primitive `mutable*StateOf`, `derivedStateOf`, `snapshotFlow`), stability & recomposition skipping (`@Immutable` / `@Stable` / `ImmutableList<T>` / stable lambdas), modifier ordering (chain order = visual order; click handling near content; lambda-form layout reads), side-effect API selection (`LaunchedEffect` vs `DisposableEffect` vs `rememberUpdatedState` vs `produceState` vs `snapshotFlow` vs `SideEffect`), lazy lists (`key`, `contentType`, `itemsIndexed`, `derivedStateOf` for scroll thresholds), animation (M3 motion tokens, `AnimatedVisibility` / `AnimatedContent` / `updateTransition`), composition locals (use sparingly, narrowest scope), theming (always `MaterialTheme.*`), Canvas safety, accessibility (semantics, decorative `contentDescription = null`, 48.dp targets), TV-specific rules (focus, `androidx.tv:tv-foundation`/`tv-material`, 10-foot UI sizing), a production crash-pattern table, performance verification (compose compiler metrics, recomposition counts, Macrobenchmark), and deprecated patterns to avoid (M2, plain `mutableStateOf<Int>(0)`, string-based nav routes, old `onCommit`/`onActive`).

Re-read the relevant section whenever a Compose decision is in front of you — don't write Compose from memory, write it from the reference.

## Navigation 3 wiring

The wizard scaffolds Navigation 3 (`androidx.navigation3`). Each feature exposes a `<Feature>Entry` extension on `EntryProviderScope<NavKey>` that registers its routes; the app-level NavHost composes them inside a `NavDisplay` with `entryProvider { ... }`. The back stack is `mutableStateListOf<NavKey>()`. Forward nav appends; back pops the last item. Cross-feature navigation flows through callbacks passed into each `<feature>Entry`, never through `NavController` (Nav3 doesn't expose one — that's the point).

**Read `references/screen-and-nav.md`** for the verbatim Kotlin for `homeEntry`, `NavDisplay`, and the Screen + ScreenContent samples.

**Nav3 1.1.x quick-reference (verified package paths — kept inline because this is the highest-traffic lookup):**

| Symbol | Package | Notes |
|---|---|---|
| `NavKey` | `androidx.navigation3.runtime.NavKey` | Marker interface every `@Serializable` route implements. |
| `EntryProviderScope<T>` | `androidx.navigation3.runtime.EntryProviderScope` | DSL receiver for `entryProvider { ... }`. Has the `entry<K>` member. |
| `entry<K>` | **member function** on `EntryProviderScope<T>` | NOT a top-level function — no separate import. Available inside any function with `EntryProviderScope<T>` as receiver. **Do NOT add `import androidx.navigation3.runtime.entry`** — that import does not exist. |
| `entryProvider` | `androidx.navigation3.runtime.entryProvider` | Top-level builder function that creates the `EntryProviderScope`. |
| `NavDisplay` | `androidx.navigation3.ui.NavDisplay` | The Compose host. |
| `rememberNavBackStack` | `androidx.navigation3.runtime.rememberNavBackStack` | Optional helper; plain `mutableStateListOf<NavKey>()` works too. |

## Koin conventions

- One Koin `Module.kt` per feature ui module, named `<feature>Module`, contains `viewModel { ... }` for the feature's ViewModel(s) and any `single { }`/`factory { }` it needs.
- One Koin `Module.kt` per feature data module (under `feature/<feature>/data/.../di/<Feature>DataModule.kt`), named `<feature>DataModule`. This is the **only** public symbol the data module exposes besides the `Repository` interface — everything else stays `internal`.
- `<Feature>Modules.kt` in the ui module aggregates: `val <feature>Modules = listOf(<feature>Module, <feature>DataModule)`. The ui module imports `<feature>DataModule` from the data module (allowed: ui depends on data with `implementation`). The app does **not** import `<feature>DataModule` — it only imports `<feature>Modules` from the ui module.
- `<Project>Application` wires all feature modules:

  ```kotlin
  startKoin {
      androidContext(this@<Project>Application)
      modules(listOf(appModule) + dataModules + homeModules + searchModules + ...)
  }
  ```

- ViewModels are constructed via `koin-androidx-compose`'s `koinViewModel()` in the Screen file. **Never** in ScreenContent.

## Theming

`core/designsystem-base` exposes:
- Color tokens (`object <Project>Colors`)
- Typography tokens (`val <Project>Typography`)
- Spacing tokens (`object Spacing { val xs/sm/md/lg/xl }`)
- Shape tokens if needed

`core/designsystem-mobile` exposes:
- `<Project>Theme` composable wrapping `MaterialTheme` with the project's tokens
- `@<Project>Previews` multi-preview annotation generating light + dark
- Shared mobile components (`<Project>Button`, `<Project>Loading`, `<Project>ErrorState`, etc.)

**Hardcoded colors and text styles are forbidden.** Use `MaterialTheme.colorScheme.*`, `MaterialTheme.typography.*`, or resource references (`colorResource(R.color.*)`). If the design needs a color that doesn't exist, add it to the token set first.

## Environment + dev tools (qa-only)

Every scaffolded project has `qa` and `prod` product flavors (see `<project>.android.flavors`). On top of the compile-time `BuildConfig.API_BASE_URL` baked into each flavor, the runtime lets the qa user override the base URL via a shake-or-broadcast dialog — useful for pointing a single build at staging, prod, or an arbitrary URL (local mock server, teammate's tunnel, etc.) without rebuilding.

### Where each piece lives

| Concern | Module | Files |
|---|---|---|
| Runtime base URL override | `core/data` | `env/EnvironmentConfig.kt`, `env/EnvironmentConfigImpl.kt`, `env/EnvironmentConfigDataStore.kt` |
| Koin module for env config | `core/data` | `env/di/EnvironmentModule.kt` (`val environmentModule`) |
| OkHttp interceptor that rewrites the host | `core/data` | `network/EnvironmentBaseUrlInterceptor.kt` |
| Shake detector composable | `core/ui-mobile` | `dev/ShakeDetector.kt` |
| Env selector dialog | `core/ui-mobile` | `dev/EnvSelectorDialog.kt` |
| Host that mounts the detector + receiver + dialog | `core/ui-mobile` | `dev/DevToolsHost.kt` |
| TV-side dev tools wrapper (broadcast-only) | `core/ui-tv` (TV projects only) | `dev/DevToolsHost.kt` (same name, TV variant — no shake) |

`core/ui-mobile` and `core/ui-tv` **do not** apply `<project>.android.flavors`. The dev-tools code compiles on every variant and the **app** decides whether to mount it via `BuildConfig.IS_QA`.

**Read `references/dev-tools.md`** for the verbatim Kotlin source of every file in this list — `EnvironmentConfig` interface and DataStore impl, `environmentModule` Koin module, `EnvironmentBaseUrlInterceptor`, `ShakeListener`, `DevToolsBroadcastListener`, `DevToolsHost` (mobile and TV variants), and `EnvSelectorDialog`. That reference also covers app-layer wiring (`DevToolsHost(enabled = BuildConfig.IS_QA)`) and how to trigger the dialog on phone, emulator, and TV. Read it before generating any file under `core/data/env/`, `core/data/network/`, `core/ui-mobile/dev/`, or `core/ui-tv/dev/`.

Key non-obvious rules so this skill stays self-contained:

- `EnvironmentBaseUrlInterceptor` rewrites only **scheme/host/port** of outgoing OkHttp requests, never the path/query. Register it **first** in the client's interceptor chain so auth/logging interceptors see the rewritten URL.
- `EnvironmentConfigImpl` reads the persisted override from DataStore on init; before the first emission completes, callers see the compile-time `BuildConfig.API_BASE_URL` from `core/data`'s flavor.
- The literal Staging/Prod URLs in `environmentModule` must match the values the wizard wrote into `gradle.properties` (`<project>.qaApiBaseUrl`, `<project>.prodApiBaseUrl`). Source-of-truth for both is the wizard.
- The broadcast action is `<applicationId>.OPEN_DEV_TOOLS` and the receiver is registered `RECEIVER_NOT_EXPORTED` — other apps cannot deliver it. The CLI trigger is `adb shell am broadcast -a <applicationId>.OPEN_DEV_TOOLS -p <applicationId>`.
- `DevToolsHost(enabled = false)` is a zero-cost passthrough — prod builds do not register the receiver, do not subscribe to the accelerometer, and do not mount the dialog. Always gate via `BuildConfig.IS_QA`, never via a runtime flag.
- For **mobile + TV projects**, lift the reusable composables (`DevToolsBroadcastListener`, `EnvSelectorDialog`, `ShakeListener`) into a `core/ui-base` module so `core/ui-tv` doesn't depend on `core/ui-mobile`. Mobile-only projects keep everything under `core/ui-mobile/dev/`.
## libs.versions.toml structure

Single catalog file at `gradle/libs.versions.toml`. Sections:

1. `[versions]` — every version is named. SDK numbers (`compileSdk = "36"`, `minSdk = "24"`, `targetSdk = "36"`) and `jvmTarget = "21"` also live here so convention plugins can read them.
2. `[libraries]` — alias keys are kebab-case scoped by ecosystem (`androidx-core-ktx`, `koin-android`, `retrofit-moshi`). Use `version.ref` for shared versions; omit `version` for BOM-managed deps.
3. `[plugins]` — Gradle plugin aliases. The 9 convention plugins are listed here with `version = "unspecified"`:

   ```toml
   <project>-android-application = { id = "<project>.android.application", version = "unspecified" }
   <project>-android-library     = { id = "<project>.android.library", version = "unspecified" }
   # ... etc for all 8
   ```

   And the third-party plugins use `version.ref`:

   ```toml
   android-application = { id = "com.android.application", version.ref = "agp" }
   kotlin-android      = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
   spotless            = { id = "com.diffplug.spotless", version.ref = "spotless" }
   ```

### Catalog accessor form — use the imperative API in module build scripts

In `build-logic/convention/` Kotlin code, `libs.findLibrary("X").get()` is the only API available (the type-safe `libs.androidx.core.ktx` accessors aren't generated for build-logic).

In **module** `build.gradle.kts` files (`app-mobile`, `core/*`, `feature/*/data`, `feature/*/ui-mobile`), there are two accessor forms:

1. **Type-safe** — `implementation(libs.androidx.core.ktx)`. Idiomatic; what most public docs show.
2. **Imperative** — `implementation(libs.findLibrary("androidx-core-ktx").get())`. Works everywhere.

**Use the imperative form throughout this project.** Reasons:

- On Gradle 9.5.1 + AGP 9.2 with `includeBuild("build-logic")` and `repositoriesMode = FAIL_ON_PROJECT_REPOS`, the generated `LibrariesForLibs` type-safe accessor class is sometimes not visible to subproject build script classpaths — even when `libs` itself resolves. The result is `Unresolved reference 'androidx'` (or `'kotlinx'`, `'koin'`, ...) errors in module scripts that compile cleanly with the imperative form. Root cause is still under investigation upstream.
- The convention plugins already use the imperative API; sticking with it across module scripts removes a mixed style.
- It tolerates catalog reshuffles — renaming an alias is a single-spot grep.

**Canonical helper at the top of every module's `build.gradle.kts` that uses catalog deps directly:**

```kotlin
import org.gradle.api.artifacts.VersionCatalogsExtension

plugins {
    alias(libs.plugins.<project>.android.application)
    // ... other convention + standalone plugin aliases
}

val libsCatalog = extensions.getByType<VersionCatalogsExtension>().named("libs")
fun lib(alias: String) = libsCatalog.findLibrary(alias).get()

android {
    namespace = "<root-pkg>"
    // ... rest of android block
}

dependencies {
    implementation(project(":core:common"))
    // ... other project deps

    implementation(lib("androidx-core-ktx"))
    implementation(lib("androidx-activity-compose"))
    implementation(lib("koin-android"))
    // ... library deps via the helper
}
```

The `lib(...)` helper is a 3-line workaround that keeps call-sites readable. If the type-safe accessor regression is fixed in a future Gradle/AGP version, switching back is a global find-replace.

**Plugin aliases (`libs.plugins.<x>`) and version refs (`libs.versions.<x>`) work fine** — only the `libs.<library>` access path is affected. Keep `alias(libs.plugins.<project>.android.feature)` in `plugins { }` blocks unchanged.

## Test conventions

**Read `references/test-patterns.md`** for the verbatim `MainCoroutineRule` source, ViewModel test pattern, and Compose UI test pattern.

**Rules of thumb:**

- **TDD for everything except Compose.** ViewModels, repositories, mappers, validators, extension functions — write the test first.
- **Test-after for Compose** (you need testTags and button text to write the test). Compose UI tests run on Robolectric (JVM), not androidTest.
- **Stack:** JUnit 4 (`junit`), Mockito-Kotlin (`mockito-kotlin`), Google Truth (`truth`), Turbine (`turbine`), Robolectric (`robolectric`), and `MainCoroutineRule` from `core/testing`.
- **`MainCoroutineRule` must default to `UnconfinedTestDispatcher`** — `StandardTestDispatcher` makes `BaseViewModel.init { ... }` action-collection lazy and Turbine effect assertions time out at 3s. Pass `StandardTestDispatcher()` explicitly only when testing time-based behavior with `advanceTimeBy`.
- **Compose tests live in `src/test/kotlin/` (NOT `androidTest`)** and need `src/test/resources/robolectric.properties` with `sdk=34`. Target `<Feature>ScreenContent` (pure UI, no Koin), never `<Feature>Screen` (which depends on Koin and can't be instantiated in a unit test).
- **Test the meaningful things** — action dispatch from clicks, state-driven conditional rendering, error-state visibility. Don't test that static text renders; previews cover that.

## Resource ownership

A resource (drawable, string, dimen, color, raw asset) belongs to the **owning module** — the module whose code references it. Feature-only assets live in `feature/<x>/ui-mobile/src/main/res/`. Cross-feature assets live in `core/designsystem-mobile/src/main/res/`. The moment a second feature needs an asset, promote it to designsystem.

Mirror density buckets (`drawable-nodpi`, `drawable-sw600dp-*-nodpi`) under the destination module's `res/` so all device classes resolve.

## Tooling defaults

### Spotless
- ktlint at the version pinned in the catalog (`ktlint` version key)
- Target `**/*.kt` and `**/*.kts`
- Apply via `./gradlew spotlessApply`, check via `./gradlew spotlessCheck`
- Configured in root `build.gradle.kts` via `subprojects { apply(plugin = ...spotless...) }` block or as a standalone module

### detekt
- Latest stable
- Config: `.detekt/config.yml` at root
- Apply to all subprojects
- Run via `./gradlew detekt`

### Android Lint
- `.lint/config.xml` at root
- `abortOnError = true`, `checkDependencies = true` (set by the `<project>.android.lint` convention plugin)

### .editorconfig
- 4-space indent for Kotlin, 2-space for YAML/JSON/markdown
- LF line endings
- `insert_final_newline = true`

### CI (.github/workflows/)
- `build.yml` — runs on push/PR to main: `spotlessCheck`, `lint`, `detekt`, `test`
- `dependabot.yml` (under `.github/`) — bumps gradle + github-actions ecosystems, labels PRs `version update`

## Version-lookup sources

When the wizard runs (or `bump-versions` is invoked), resolve the **latest stable** version for each library. **Skip alpha / beta / rc / SNAPSHOT releases** unless the user explicitly opted into pre-release for that specific library. Sources, in priority order:

| Library family | Primary source | Notes |
|---|---|---|
| AGP (Android Gradle Plugin) | `https://developer.android.com/build/releases/gradle-plugin` | Or `mvnrepository.com/artifact/com.android.tools.build/gradle` |
| Kotlin | `https://kotlinlang.org/docs/releases.html` | Or `github.com/JetBrains/kotlin/releases` |
| KSP | `https://github.com/google/ksp/releases` | KSP version must match the Kotlin major.minor; format is `<kotlin>-<ksp>` |
| Compose BOM | `https://developer.android.com/jetpack/compose/bom/bom-mapping` | Use the BOM, not individual compose libs |
| AndroidX libraries (core-ktx, lifecycle, activity, navigation3, datastore, room, security-crypto, browser) | `https://developer.android.com/jetpack/androidx/versions` | Each lib has its own page; check release notes for breaking changes |
| Kotlinx (coroutines, serialization, collections-immutable) | `https://github.com/Kotlin/kotlinx.coroutines/releases` etc. | Stable releases only |
| Koin | `https://github.com/InsertKoinIO/koin/releases` | Match `koin-core`, `koin-android`, `koin-androidx-compose`, `koin-test-junit4` to the same version |
| Retrofit / OkHttp | `https://github.com/square/retrofit/releases` / `https://github.com/square/okhttp/releases` | |
| Apollo | `https://github.com/apollographql/apollo-kotlin/releases` | |
| Coil 3 | `https://github.com/coil-kt/coil/releases` | Use the `coil3` artifacts (`io.coil-kt.coil3`) |
| Moshi | `https://github.com/square/moshi/releases` | |
| Firebase BOM | `https://firebase.google.com/support/release-notes/android` | BOM pins all firebase libs |
| google-services / crashlytics plugins | `https://developers.google.com/android/guides/google-services-plugin` / Gradle Plugin Portal | |
| Spotless | `https://github.com/diffplug/spotless/releases` | |
| ktlint | `https://github.com/pinterest/ktlint/releases` | |
| detekt | `https://github.com/detekt/detekt/releases` | |
| Robolectric | `https://github.com/robolectric/robolectric/releases` | Match `robolectric.properties` sdk to the host android version |
| Turbine | `https://github.com/cashapp/turbine/releases` | |
| Mockito-Kotlin | `https://github.com/mockito/mockito-kotlin/releases` | |
| foojay-resolver | Gradle Plugin Portal: `https://plugins.gradle.org/plugin/org.gradle.toolchains.foojay-resolver-convention` | |

**How to resolve:** prefer `WebSearch` for "latest stable <lib>" queries, then `WebFetch` the GitHub releases page to confirm. For Maven artifacts, `https://search.maven.org/solrsearch/select?q=g:<group>+AND+a:<artifact>&core=gav&rows=20&wt=json` returns versions sorted by recency. Always pin the resolved versions in the catalog — never use `+` or dynamic versions.

## Naming conventions

- **Project name** (user-supplied, kebab-case): `acme`, `super-app`, `my-store`
- **Project class prefix** (PascalCase): `Acme`, `SuperApp`, `MyStore` — used for `<Project>Theme`, `<Project>Previews`, `<Project>Application`, color tokens, etc.
- **Root package** (user-supplied, dot-separated): `com.acme.android`, `com.example.superapp`
- **Convention plugin id prefix**: lowercase project name with hyphens stripped: `acme.android.application`, `superapp.android.application`
- **Module namespaces**: `<root-pkg>.feature.<feature>.ui`, `<root-pkg>.feature.<feature>.data`, `<root-pkg>.core.common`, etc.

## Quality gates (every commit boundary)

```bash
./gradlew spotlessApply        # auto-fix formatting
./gradlew spotlessCheck        # verify clean
./gradlew detekt               # static analysis
./gradlew lint                 # Android Lint
./gradlew test                 # unit tests (including Robolectric Compose tests)
```

Run before every commit. If `spotlessCheck` fails, the formatting was rolled back somehow — re-apply.

## What the generated project-local skills should know

After scaffolding, the wizard writes two skills into the new project's `.claude/skills/`:

- **`<project>-android-planner/SKILL.md`** — knows the architecture (loads `references/project-architecture.md` co-bundled with the skill) and plans features.
- **`<project>-android-implementer/SKILL.md`** — knows the same architecture and writes the code.

Both skills should be derived from this conventions file with the user's choices baked in (e.g. project name, root package, the specific convention plugin ids, whether TV exists, which network library was picked). Keep them short and pointed — the conventions don't need to be re-explained, they need to be **applied**.

## When in doubt

- This skill is the reference. Re-read the relevant section.
- Don't invent new patterns — the canonical shapes in this skill are the source of truth.
- If something is genuinely undefined here, ask the user. Don't invent new conventions.
