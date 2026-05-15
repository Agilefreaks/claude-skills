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
│       └── src/main/kotlin/                # the 8 convention plugins
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

## The 8 convention plugins

All build configuration lives in `build-logic/convention/`. Modules apply a **single** convention plugin instead of three. Every plugin reads versions and library coordinates from `libs.versions.toml` via the `libs` extension (`ProjectExtensions.kt`).

### Hard rules — read before generating any plugin file

These are non-negotiable. Diverging from them produces silent build failures on AGP 9+:

1. **The ONLY files allowed in `build-logic/convention/src/main/kotlin/`** are exactly the 8 plugin classes plus `ProjectExtensions.kt`:
   ```
   AndroidApplicationConventionPlugin.kt
   AndroidApplicationComposeConventionPlugin.kt
   AndroidLibraryConventionPlugin.kt
   AndroidLibraryComposeConventionPlugin.kt
   AndroidFeatureConventionPlugin.kt
   AndroidLintConventionPlugin.kt
   AndroidRoomConventionPlugin.kt          # only if Room was picked
   JvmLibraryConventionPlugin.kt
   ProjectExtensions.kt
   ```
   **DO NOT** invent helper files (`AndroidCommon.kt`, `AndroidConfig.kt`, `KotlinExtensions.kt`, etc.) to share code between plugins. Even when Application and Library plugins look like they share configuration, **inline the duplication**. The extraction has burned every project that tried it because of the AGP 9 type-system issue described below.

2. **No `package` declaration** on plugin classes. Files in `build-logic/convention/src/main/kotlin/` go in the default (root) package. The plugin id resolution in Gradle relies on top-level class names.

3. **Use `ApplicationExtension` and `LibraryExtension` directly — never `CommonExtension`.** In AGP 9 `CommonExtension` is no longer parameterized (the `<*>` style fails: "No type arguments expected for 'interface CommonExtension : ExtensionAware'") and even when typed correctly, `apply { compileSdk = ... }` can't resolve properties like `compileSdk`, `defaultConfig`, `compileOptions` through `CommonExtension`. Configure each extension separately in its own plugin. Yes, this duplicates ~6 lines between `AndroidApplicationConventionPlugin` and `AndroidLibraryConventionPlugin` — that duplication is the conventional Now-In-Android pattern and is correct.

4. **The build-logic Gradle plugin IDs use the project name prefix verbatim.** For project name `test`, plugin IDs are `test.android.application`, `test.android.library`, etc. The catalog plugin aliases and `gradlePlugin.plugins { ... }` block must use the same id, kebab/dot form.

5. **`build-logic/convention/build.gradle.kts` must register all 8 plugins** (or 7 if Room was skipped) under `gradlePlugin { plugins { register("...") { id = "..."; implementationClass = "..." } } }`. Forgetting one means the catalog alias resolves to "unspecified" at apply time.



| Plugin id | Class | What it does |
|---|---|---|
| `<project>.android.application` | `AndroidApplicationConventionPlugin` | Applies `com.android.application` + `kotlin.android`, configures `compileSdk`/`minSdk`/`targetSdk` from the catalog, sets Java 21 source/target. |
| `<project>.android.application.compose` | `AndroidApplicationComposeConventionPlugin` | Applies the Compose compiler plugin, enables `buildFeatures.compose`, adds the Compose BOM + ui/material3/tooling/test deps to an application module. |
| `<project>.android.library` | `AndroidLibraryConventionPlugin` | Applies `com.android.library` + `kotlin.android`, configures `compileSdk`/`minSdk`, Java 21, and adds the standard library deps (coroutines, serialization, koin-core) plus the test stack (junit, mockito-kotlin, truth, coroutines-test, turbine) and a `testImplementation(project(":core:testing"))` for every library except `:core:testing` itself. |
| `<project>.android.library.compose` | `AndroidLibraryComposeConventionPlugin` | Applies the Compose compiler plugin, enables `buildFeatures.compose`, adds the Compose BOM + ui/tooling/test deps to a library module. |
| `<project>.android.feature` | `AndroidFeatureConventionPlugin` | The aggregator for feature ui modules. Applies `<project>.android.library` + `<project>.android.library.compose` + `<project>.android.lint`, plus the feature-extra deps: `lifecycle-runtime-ktx`, `lifecycle-runtime-compose`, `lifecycle-viewmodel-compose`, `navigation3-runtime`, `kotlinx-collections-immutable`, `koin-androidx-compose`. Every `feature/<x>/ui-mobile` module applies just this one plugin. |
| `<project>.android.lint` | `AndroidLintConventionPlugin` | Sets `lint { abortOnError = true; checkDependencies = true; lintConfig = rootProject.file(".lint/config.xml") }` on whichever Android extension is present (Application or Library). |
| `<project>.android.room` | `AndroidRoomConventionPlugin` | Applies `androidx.room` + `com.google.devtools.ksp`, sets `room { schemaDirectory("$projectDir/schemas") }`, adds `room-runtime`, `room-ktx`, and `ksp(room-compiler)`. Only generated if Room was picked. |
| `<project>.jvm.library` | `JvmLibraryConventionPlugin` | Pure-Kotlin (JVM) library — used by modules with no Android dependency (e.g. `:core:model`). Java 21, jvmToolchain(21), standard test stack. |

### Canonical plugin file contents

Generate **exactly** these shapes. Substitute `<project>` for the project's plugin-id prefix (lowercase, hyphens stripped). Substitute version aliases only if the catalog uses different names. Do not refactor.

**`ProjectExtensions.kt`** — the shared `libs` accessor (no package declaration):

```kotlin
import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalog
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.kotlin.dsl.getByType

val Project.libs: VersionCatalog
    get() = extensions.getByType<VersionCatalogsExtension>().named("libs")
```

**`AndroidApplicationConventionPlugin.kt`** — configures `ApplicationExtension` directly. **Must enable `buildConfig`** because the scaffolded `<Project>Application.kt` references `BuildConfig.DEBUG` (e.g. to gate `Timber.plant(DebugTree())`). In AGP 9 `buildConfig` defaults to `false` and the legacy escape hatch `android.defaults.buildfeatures.buildconfig=true` in `gradle.properties` was **removed** — adding it now hard-errors at configuration time. The only correct fix is to enable it in this plugin:

```kotlin
import com.android.build.api.dsl.ApplicationExtension
import org.gradle.api.JavaVersion
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            // AGP 9.2+: do NOT apply org.jetbrains.kotlin.android manually — AGP's new DSL
            // hard-errors on the combination. Kotlin is auto-applied by com.android.application
            // when android.builtInKotlin=true (the default).
            pluginManager.apply("com.android.application")

            extensions.configure<ApplicationExtension> {
                compileSdk = libs.findVersion("compileSdk").get().toString().toInt()
                defaultConfig {
                    minSdk = libs.findVersion("minSdk").get().toString().toInt()
                    targetSdk = libs.findVersion("targetSdk").get().toString().toInt()
                }
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_21
                    targetCompatibility = JavaVersion.VERSION_21
                }
                buildFeatures {
                    buildConfig = true   // BuildConfig.DEBUG gates Timber init in <Project>Application
                }
            }
        }
    }
}
```

**Do NOT add `android.defaults.buildfeatures.buildconfig=true` to `gradle.properties`** — it's removed in AGP 9 and will fail with:
```
EvalIssueException: The option 'android.defaults.buildfeatures.buildconfig' is deprecated.
The current default is 'false'.
It was removed in version 9.0 of the Android Gradle plugin.
```
The convention plugin above is the only place to enable it.

**`AndroidLibraryConventionPlugin.kt`** — configures `LibraryExtension` directly (yes, the duplication with the Application plugin is intentional):

```kotlin
import com.android.build.api.dsl.LibraryExtension
import org.gradle.api.JavaVersion
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.project

class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            // AGP 9.2+: do NOT apply org.jetbrains.kotlin.android manually (see Application plugin).
            pluginManager.apply("com.android.library")

            extensions.configure<LibraryExtension> {
                compileSdk = libs.findVersion("compileSdk").get().toString().toInt()
                defaultConfig { minSdk = libs.findVersion("minSdk").get().toString().toInt() }
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_21
                    targetCompatibility = JavaVersion.VERSION_21
                }
            }
            dependencies {
                add("implementation", libs.findLibrary("kotlinx-coroutines-core").get())
                add("implementation", libs.findLibrary("kotlinx-coroutines-android").get())
                add("implementation", libs.findLibrary("kotlinx-serialization-json").get())
                add("implementation", libs.findLibrary("koin-core").get())
                add("testImplementation", libs.findLibrary("junit").get())
                add("testImplementation", libs.findLibrary("mockito-kotlin").get())
                add("testImplementation", libs.findLibrary("truth").get())
                add("testImplementation", libs.findLibrary("kotlinx-coroutines-test").get())
                add("testImplementation", libs.findLibrary("turbine").get())
                if (path != ":core:testing") {
                    add("testImplementation", project(":core:testing"))
                }
            }
        }
    }
}
```

**`AndroidApplicationComposeConventionPlugin.kt`** and **`AndroidLibraryComposeConventionPlugin.kt`** — each configures its own extension (`ApplicationExtension` / `LibraryExtension`), applies `org.jetbrains.kotlin.plugin.compose`, enables `buildFeatures { compose = true }`, sets `testOptions.unitTests.isIncludeAndroidResources = true`, and adds the Compose BOM-managed deps.

The `isIncludeAndroidResources = true` line is required for Robolectric Compose tests. `createComposeRule()` launches `ComponentActivity` under the hood via `ActivityScenarioRule`. For **unit tests** (`src/test/`, not `src/androidTest/`), AGP does not merge AAR manifests by default — so `ComponentActivity`'s registration (provided by `compose-ui-test-manifest.aar`) is invisible to the test runtime and tests crash with `Unable to resolve activity for Intent { ... cmp=org.robolectric.default/androidx.activity.ComponentActivity }`. Enabling `isIncludeAndroidResources` merges AAR resources/manifests for unit tests and fixes this.

Canonical shape of the Compose convention plugins (substitute `ApplicationExtension` or `LibraryExtension` per plugin):

```kotlin
import com.android.build.api.dsl.LibraryExtension   // or ApplicationExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies

class AndroidLibraryComposeConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

            extensions.configure<LibraryExtension> {
                buildFeatures {
                    compose = true
                }
                testOptions {
                    unitTests {
                        // Required for Robolectric Compose tests — without this, createComposeRule()
                        // can't find ComponentActivity in the merged manifest and tests crash with
                        // "Unable to resolve activity for Intent".
                        isIncludeAndroidResources = true
                    }
                }
            }

            dependencies {
                val bom = libs.findLibrary("androidx-compose-bom").get()
                add("implementation", platform(bom))
                add("implementation", libs.findLibrary("androidx-compose-ui").get())
                add("implementation", libs.findLibrary("androidx-compose-ui-graphics").get())
                add("implementation", libs.findLibrary("androidx-compose-ui-tooling-preview").get())
                add("debugImplementation", libs.findLibrary("androidx-compose-ui-tooling").get())
                add("debugImplementation", libs.findLibrary("androidx-compose-ui-test-manifest").get())
                add("testImplementation", platform(bom))
                add("testImplementation", libs.findLibrary("androidx-compose-ui-test-junit4").get())
                add("testImplementation", libs.findLibrary("robolectric").get())
                add("androidTestImplementation", platform(bom))
                add("androidTestImplementation", libs.findLibrary("androidx-compose-ui-test-junit4").get())
            }
        }
    }
}
```

The Application variant is identical except for `LibraryExtension` → `ApplicationExtension` and `add("implementation", libs.findLibrary("androidx-compose-material3").get())` for the Material 3 dep.

**`AndroidFeatureConventionPlugin.kt`** — applies `<project>.android.library`, `<project>.android.library.compose`, `<project>.android.lint`, then adds feature-specific deps (lifecycle, navigation3, koin-androidx-compose, kotlinx-collections-immutable). Does NOT configure any Android extension itself — that's the library plugin's job.

**`AndroidLintConventionPlugin.kt`** — uses `pluginManager.hasPlugin(...)` to detect whether the target is an Application or Library module, then configures the correct extension's `lint { ... }` block.

**`AndroidRoomConventionPlugin.kt`** — only generated if Room was picked; applies `androidx.room` + `com.google.devtools.ksp`, sets `schemaDirectory`, adds `room-runtime`, `room-ktx`, and `ksp(room-compiler)`.

**`JvmLibraryConventionPlugin.kt`** — applies `org.jetbrains.kotlin.jvm`, configures Java 21 + jvmToolchain(21), adds the standard test stack.

### `build-logic/convention/build.gradle.kts`

```kotlin
plugins {
    `kotlin-dsl`
}

group = "<project>.buildlogic"

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.compose.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
    // include room.gradlePlugin only if Room was picked
}

gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "<project>.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidApplicationCompose") {
            id = "<project>.android.application.compose"
            implementationClass = "AndroidApplicationComposeConventionPlugin"
        }
        register("androidLibrary") {
            id = "<project>.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidLibraryCompose") {
            id = "<project>.android.library.compose"
            implementationClass = "AndroidLibraryComposeConventionPlugin"
        }
        register("androidFeature") {
            id = "<project>.android.feature"
            implementationClass = "AndroidFeatureConventionPlugin"
        }
        register("androidLint") {
            id = "<project>.android.lint"
            implementationClass = "AndroidLintConventionPlugin"
        }
        // include androidRoom only if Room was picked:
        register("androidRoom") {
            id = "<project>.android.room"
            implementationClass = "AndroidRoomConventionPlugin"
        }
        register("jvmLibrary") {
            id = "<project>.jvm.library"
            implementationClass = "JvmLibraryConventionPlugin"
        }
    }
}
```

The plugin id prefix is the project name in lowercase, dot-separated. For project name `acme`, the plugin id is `acme.android.application`. Convention plugins are registered in `build-logic/convention/build.gradle.kts` under `gradlePlugin.plugins`.

**`build-logic/settings.gradle.kts`** must read the same `libs.versions.toml`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
rootProject.name = "build-logic"
include(":convention")
```

**Root `settings.gradle.kts`** must `includeBuild("build-logic")`:

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        mavenCentral()
        gradlePluginPortal()
    }
}
plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "<latest>"
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
```

**`gradle.properties` Kotlin handling — AGP version matters.**

The right setting flipped between AGP 9.0 and AGP 9.2:

| Resolved AGP | `gradle.properties` | Convention plugins |
|---|---|---|
| `9.0.x` – `9.1.x` | `android.builtInKotlin=false` | **Must** `apply("org.jetbrains.kotlin.android")` manually in `AndroidApplicationConventionPlugin` and `AndroidLibraryConventionPlugin`. Built-in Kotlin broke KSP in early 9.0. |
| `9.2.0+` | `android.builtInKotlin=true` (or omit — `true` is the default with `android.newDsl=true`) | **Do NOT** `apply("org.jetbrains.kotlin.android")` manually — AGP 9.2's new DSL hard-errors with `The 'org.jetbrains.kotlin.android' plugin is not compatible with AGP's 9.0 new DSL`. AGP auto-applies Kotlin. |

The wizard **must** detect the resolved AGP version after Step 9 (version resolution) and pick the matching template. If AGP ≥ 9.2.0, generate the Application/Library convention plugin canonicals **without** the `apply("org.jetbrains.kotlin.android")` line and set `android.builtInKotlin=true` (or omit it). If AGP is 9.0.x or 9.1.x, keep the manual-apply form. New scaffolds should default to the AGP 9.2+ shape since "latest stable" resolves there now.

**Do NOT add these legacy escape hatches to `gradle.properties` — they were removed in AGP 9 and fail at configuration time:**
- `android.defaults.buildfeatures.buildconfig` — enable `buildConfig` in the `AndroidApplicationConventionPlugin` instead (already done above).
- `android.defaults.buildfeatures.aidl` — enable per-module if you ever need AIDL.
- `android.defaults.buildfeatures.renderscript` — RenderScript is end-of-life; don't use it.

## AndroidManifest.xml

Every Android module has `src/main/AndroidManifest.xml`, even library modules with no components. The Android Gradle Plugin merges them into the app manifest at build time. Two rules:

1. **Always declare the `xmlns:android` namespace on `<manifest>`** — without it, any `android:*` attribute (e.g. `android:name` on `<uses-permission>` or `<application>`) fails to resolve and the AAPT merger errors out:
   ```
   AndroidManifest.xml:N: AAPT: error: attribute 'android:name' not found.
   ```

2. **Library manifests are minimal.** No `<application>` tag, no `package` attribute (AGP derives it from the module's `namespace` in `build.gradle.kts`).

**Canonical library `AndroidManifest.xml` shapes:**

A library module that contributes nothing to the merged manifest:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" />
```

A library module that declares a permission (e.g. `core/data` declaring `INTERNET`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
</manifest>
```

**Application module manifest** (`app-mobile/src/main/AndroidManifest.xml`, similarly `app-tv/`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".<Project>Application"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.<Project>">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.<Project>">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>

</manifest>
```

**For `app-tv`**, add `android.intent.category.LEANBACK_LAUNCHER` to the main activity's intent-filter and use `android:banner="@drawable/banner"` on the application.

## MVI base types

`core/common/src/main/kotlin/<root-pkg>/core/` — four files, each in package `<root-pkg>.core`. Generate them **verbatim** as below. `ViewAction`, `ViewState`, and `ViewSideEffect` live in the same package as `BaseViewModel`, so **do not import them** inside `BaseViewModel.kt` — they're resolved automatically.

**`ViewAction.kt`**
```kotlin
package <root-pkg>.core

/** Marker interface for user-driven actions sent to a [BaseViewModel]. */
interface ViewAction
```

**`ViewState.kt`**
```kotlin
package <root-pkg>.core

/** Marker interface for MVI view state — should be a stable, immutable data class. */
interface ViewState
```

**`ViewSideEffect.kt`**
```kotlin
package <root-pkg>.core

/** Marker interface for one-shot side effects (navigation, snackbars, …). */
interface ViewSideEffect
```

**`BaseViewModel.kt`** — exact shape. No `import <root-pkg>.core.View*` lines. No deviation.

```kotlin
package <root-pkg>.core

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.onEach
import kotlinx.coroutines.flow.receiveAsFlow
import kotlinx.coroutines.launch

/**
 * Base MVI view model. Subclasses provide an initial [State] and an [onAction] handler.
 * State is exposed as a hot [StateFlow]; side effects flow through a single-consumer [Channel].
 */
abstract class BaseViewModel<State : ViewState, Action : ViewAction, Effect : ViewSideEffect>(
    initialState: State,
) : ViewModel() {
    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<State> = _state.asStateFlow()

    private val _actions = MutableSharedFlow<Action>(extraBufferCapacity = 64)

    private val _effects = Channel<Effect>(Channel.BUFFERED)
    val effects: Flow<Effect> = _effects.receiveAsFlow()

    init {
        viewModelScope.launch {
            _actions.onEach(::onAction).collect()
        }
    }

    /** Subclasses override to react to user actions (mutate state, fire effects, etc.). */
    protected abstract suspend fun onAction(action: Action)

    /** Send an action from the UI. Non-suspending — actions are buffered. */
    fun setAction(action: Action) {
        _actions.tryEmit(action)
    }

    /** Replace state. */
    protected fun setState(reduce: State.() -> State) {
        _state.value = _state.value.reduce()
    }

    /** Emit a side effect. */
    protected suspend fun setEffect(effect: Effect) {
        _effects.send(effect)
    }
}

private suspend fun <T> Flow<T>.collect() = collect {}
```

State is a hot `StateFlow`; effects flow through a single-consumer `Channel`. Actions are buffered (`extraBufferCapacity = 64`) so `setAction` is non-suspending.

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
- The data module is **internal to the feature**. App modules (`app-mobile`, `app-tv`) never list `:feature:<feature>:data` as a dependency — they only depend on the feature's ui module(s). See *App module dependencies* below.
- Inside the data module, only the `Repository` interface and the `<feature>DataModule` Koin value are `public`. Repository implementations and any other concrete data-layer classes are marked `internal` so they cannot be referenced from outside the data module — not even from ui modules, which work against the interface.
- Each ui module has its own Koin `Module.kt` with its own `viewModel { <Feature>ViewModel(...) }` binding. The data module is included in both aggregators (Koin de-duplicates module registration if both apps were to load both — but they don't, since mobile and TV are separate APKs).
- Tests live in each ui module's own `src/test/kotlin/` — the mobile ViewModel/Compose tests test the mobile shape, the TV ones test the TV shape.

### Naming

- Route data object for parameterless routes: `@Serializable data object HomeRoute : NavKey` (mobile) and `@Serializable data object HomeRoute : NavKey` in the `.tv` package for TV.
- Route data class for routes with args: `@Serializable data class SeriesDetailRoute(val id: String) : NavKey`.
- Entry function: `fun EntryProviderScope<NavKey>.<feature>Entry(...)` — kebab-camel of the feature. Same name in both packages; the import path tells them apart.
- Modules aggregator (`<Feature>Modules.kt`) in each ui module: `val <feature>Modules: List<Module>` combining that module's `<feature>Module` + the shared `<feature>DataModule`.
- Data module's Koin module: `<feature>DataModule`, lives in `feature/<feature>/data/.../<feature>/data/di/<Feature>DataModule.kt`. Singular — never split per form factor.

### Data module template (visibility = encapsulation)

```kotlin
// feature/<feature>/data/.../<Feature>Repository.kt
package <root-pkg>.feature.<feature>.data

interface <Feature>Repository {
    suspend fun get<Feature>(): <Result>
}
```

```kotlin
// feature/<feature>/data/.../<Feature>RepositoryImpl.kt
package <root-pkg>.feature.<feature>.data

internal class <Feature>RepositoryImpl(
    // injected api client / dao / etc.
) : <Feature>Repository {
    override suspend fun get<Feature>(): <Result> = TODO("stub")
}
```

```kotlin
// feature/<feature>/data/.../di/<Feature>DataModule.kt
package <root-pkg>.feature.<feature>.data.di

import org.koin.core.module.dsl.singleOf
import org.koin.dsl.bind
import org.koin.dsl.module
import <root-pkg>.feature.<feature>.data.<Feature>Repository
import <root-pkg>.feature.<feature>.data.<Feature>RepositoryImpl

val <feature>DataModule = module {
    singleOf(::<Feature>RepositoryImpl) { bind<<Feature>Repository>() }
}
```

The Koin DSL lambda can reference the `internal` impl because it lives inside the same Gradle (= Kotlin) module. Outside the data module, only the interface and the `<feature>DataModule` value are visible. The ui module injects `<Feature>Repository`, not the impl; the app never sees either.

### Required deps in `feature/<feature>/ui-mobile/build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.<project>.android.feature)
}

android {
    namespace = "<root-pkg>.feature.<feature>.ui.mobile"
}

dependencies {
    implementation(project(":feature:<feature>:data"))
    implementation(project(":core:common"))
    implementation(project(":core:model"))
    implementation(project(":core:data"))
    implementation(project(":core:designsystem-base"))   // or ":core:designsystem" if mobile-only
    implementation(project(":core:designsystem-mobile"))
    implementation(project(":core:ui-mobile"))
}
```

### Required deps in `feature/<feature>/ui-tv/build.gradle.kts` (only generated when TV was picked)

```kotlin
plugins {
    alias(libs.plugins.<project>.android.feature)
}

android {
    namespace = "<root-pkg>.feature.<feature>.ui.tv"
}

dependencies {
    implementation(project(":feature:<feature>:data"))
    implementation(project(":core:common"))
    implementation(project(":core:model"))
    implementation(project(":core:data"))
    implementation(project(":core:designsystem-base"))
    implementation(project(":core:designsystem-tv"))
    implementation(project(":core:ui-tv"))
}
```

Note: both ui modules apply the same `<project>.android.feature` convention plugin — they share Compose, lifecycle, navigation3, and Koin Compose dependencies. Only the design-system / ui module references differ.

### Required deps in `app-mobile/build.gradle.kts` (and `app-tv/build.gradle.kts` analogously)

```kotlin
dependencies {
    // core
    implementation(project(":core:common"))
    implementation(project(":core:model"))
    implementation(project(":core:data"))
    implementation(project(":core:designsystem-base"))   // omit `-base` if mobile-only
    implementation(project(":core:designsystem-mobile"))
    implementation(project(":core:ui-mobile"))

    // features — ONLY the ui module per feature, never `:feature:<x>:data`
    implementation(project(":feature:home:ui-mobile"))
    implementation(project(":feature:search:ui-mobile"))
    // ... one ui module per feature

    // library deps (compose BOM, koin-android, koin-compose, splash if picked, etc.)
}
```

**Hard rule: the app module never depends on `:feature:<feature>:data`.** The feature's data layer is internal to the feature. The app reaches the data layer indirectly through the ui module's aggregated `<feature>Modules` list (which already bundles `<feature>DataModule` for Koin). This holds for both `app-mobile` and `app-tv`.

Two reinforcing mechanisms keep the data layer encapsulated:

1. **Gradle scope** — ui modules declare `implementation(project(":feature:<feature>:data"))`, never `api(...)`. The data module's classes are not on the app's compile classpath even transitively.
2. **Kotlin visibility** — repository implementations (and any other data-module classes the app shouldn't reach) are marked `internal`. Only the `Repository` interface and the `<feature>DataModule` Koin value are `public`; everything else stays inside the data module. See *Data module template* above.

If you find yourself adding `implementation(project(":feature:<x>:data"))` to an app module to fix a compile error, stop: you're reaching into a feature's internals from the wrong layer. The right fix is almost always to expose what you need through the ui module's `<feature>Modules` (Koin) or a public interface in `core/model`.

## Screen / ScreenContent contract

**Always split into two files.** Screen handles ViewModel + effect routing; ScreenContent is pure UI.

```kotlin
// FeatureScreen.kt
@Composable
fun FeatureScreen(
    modifier: Modifier = Modifier,
    onNavigateToX: (X) -> Unit,
) {
    val viewModel: FeatureViewModel = koinViewModel()
    val state by viewModel.state.collectAsStateWithLifecycle()
    HandleEffects(viewModel.effects) { effect ->
        when (effect) {
            is FeatureEffect.NavigateToX -> onNavigateToX(effect.x)
        }
    }
    FeatureScreenContent(
        modifier = modifier,
        state = state,
        onAction = { action: FeatureAction -> viewModel.setAction(action) },
    )
}
```

```kotlin
// FeatureScreenContent.kt
@Composable
fun FeatureScreenContent(
    modifier: Modifier = Modifier,
    state: FeatureState,
    onAction: (FeatureAction) -> Unit,
) { /* pure UI */ }

@<Project>Previews   // e.g. @AcmePreviews — generated in core/designsystem-mobile
@Composable
private fun FeatureScreenContentPreview() {
    FeatureScreenContent(state = FeatureState(/* sample */), onAction = {})
}
```

**Never stack `@Preview` next to `@<Project>Previews`.** The custom multi-preview annotation already aggregates `@Preview` (typically one for light, one for dark, sometimes one per font scale). Adding `@Preview` on top produces a duplicate, unconfigured render and clutters the preview pane. Use exactly one of the two:
- `@<Project>Previews` for ScreenContent and any reusable component that should render in light + dark — **this is the default**.
- A bare `@Preview` (possibly with arguments like `widthDp = 320`) only when you genuinely need a one-off configuration not in the multi-preview set (e.g. a wide-screen smoke check, RTL preview).

```kotlin
// ✅ Right
@<Project>Previews
@Composable
private fun FeatureScreenContentPreview() { … }

// ❌ Wrong — duplicate render, no extra signal
@<Project>Previews
@Preview
@Composable
private fun FeatureScreenContentPreview() { … }
```

**Modifier convention (project-wide):** `modifier: Modifier = Modifier` is the **first optional parameter** in every Composable signature. Required params first, then `modifier`, then other optionals.

**Rules (every feature, no exceptions):**

1. **Two files always.** `<Feature>Screen.kt` and `<Feature>ScreenContent.kt` are separate files in `components/`. Never inline ScreenContent into Screen.
2. **ScreenContent signature is `(modifier, state, onAction)`** — modifier first optional, state required, onAction required. Never add bare callbacks (`onClick`, `onSubmit`, `onItemClick`). All clicks dispatch typed Actions; cross-feature navigation flows through Effects that the Screen's `HandleEffects` block translates into host callbacks.
3. **ScreenContent has no Koin / Activity / host dependencies.** No `koinViewModel`, no `LocalContext.current`, no `rememberLauncherForActivityResult`. Those live in the Screen. This is what makes ScreenContent testable and previewable.
4. **At least one `@<Project>Previews`-annotated composable** lives at the bottom of `<Feature>ScreenContent.kt`, constructs a sample `State`, and passes `onAction = {}`. Add **separate previews for each meaningfully different state** — at minimum, the content state; add loading, empty, and error previews when the screen renders them differently. **Use `@<Project>Previews` alone — never stack `@Preview` on top of it.** The custom annotation already aggregates `@Preview` for light + dark (and any other configured variants). Adding `@Preview` duplicates the render.
5. **Compose UI tests target `<Feature>ScreenContent`, not `<Feature>Screen`.** Robolectric tests in `src/test/kotlin/` call `composeTestRule.setContent { <Feature>ScreenContent(state = …, onAction = { capturedActions.add(it) }) }`. Test action dispatch (clicks → captured actions) and state-driven conditional rendering. Don't try to test `<Feature>Screen` — it depends on Koin and won't instantiate in a unit test.
6. **The Screen tests are: there are no Screen tests.** Tests run against ScreenContent for UI, against ViewModel for logic. The Screen is just wiring — covered by the integration on-device run.

These rules are checked structurally by the wizard's Step 11.2 self-check before gradle runs. A feature that violates any of them blocks the build-success gate.

## Compose authoring rules

**Read `references/compose-authoring.md` before touching any Compose file.** That reference is the operational source of truth for Compose in this project. It covers: state management (where state lives, primitive `mutable*StateOf`, `derivedStateOf`, `snapshotFlow`), stability & recomposition skipping (`@Immutable` / `@Stable` / `ImmutableList<T>` / stable lambdas), modifier ordering (chain order = visual order; click handling near content; lambda-form layout reads), side-effect API selection (`LaunchedEffect` vs `DisposableEffect` vs `rememberUpdatedState` vs `produceState` vs `snapshotFlow` vs `SideEffect`), lazy lists (`key`, `contentType`, `itemsIndexed`, `derivedStateOf` for scroll thresholds), animation (M3 motion tokens, `AnimatedVisibility` / `AnimatedContent` / `updateTransition`), composition locals (use sparingly, narrowest scope), theming (always `MaterialTheme.*`), Canvas safety, accessibility (semantics, decorative `contentDescription = null`, 48.dp targets), TV-specific rules (focus, `androidx.tv:tv-foundation`/`tv-material`, 10-foot UI sizing), a production crash-pattern table, performance verification (compose compiler metrics, recomposition counts, Macrobenchmark), and deprecated patterns to avoid (M2, plain `mutableStateOf<Int>(0)`, string-based nav routes, old `onCommit`/`onActive`).

Re-read the relevant section whenever a Compose decision is in front of you — don't write Compose from memory, write it from the reference.

## Navigation 3 wiring

The wizard scaffolds Navigation 3 (the new type-safe Compose nav library: `androidx.navigation3`). Each feature exposes an `<Feature>Entry` extension on `EntryProviderScope<NavKey>` that registers its routes:

```kotlin
package <root-pkg>.feature.home

import androidx.navigation3.runtime.EntryProviderScope
import androidx.navigation3.runtime.NavKey
// NOTE: do NOT add `import androidx.navigation3.runtime.entry` — that import does not exist.
// `entry<K>` is a member function on EntryProviderScope<T> and is auto-available inside
// any function with EntryProviderScope<NavKey> as its receiver.

fun EntryProviderScope<NavKey>.homeEntry(
    onNavigateToSeries: (String) -> Unit,
) {
    entry<WatchRoute> {
        HomeScreen(onNavigateToSeries = onNavigateToSeries)
    }
}
```

**Nav3 1.1.x quick-reference (verified package paths):**

| Symbol | Package | Notes |
|---|---|---|
| `NavKey` | `androidx.navigation3.runtime.NavKey` | Marker interface every `@Serializable` route implements. |
| `EntryProviderScope<T>` | `androidx.navigation3.runtime.EntryProviderScope` | DSL receiver for `entryProvider { ... }`. Has the `entry<K>` member. |
| `entry<K>` | **member function** on `EntryProviderScope<T>` | NOT a top-level function — no separate import. Available inside any function with `EntryProviderScope<T>` as receiver. Signature: `inline fun <reified K : T> EntryProviderScope<T>.entry(contentKey: Any = ..., noinline metadata: (K) -> Map<String, Any> = ..., noinline content: @Composable (K) -> Unit)`. |
| `entryProvider` | `androidx.navigation3.runtime.entryProvider` | Top-level builder function that creates the `EntryProviderScope`. |
| `NavDisplay` | `androidx.navigation3.ui.NavDisplay` | The Compose host. |
| `rememberNavBackStack` | `androidx.navigation3.runtime.rememberNavBackStack` | Optional helper; plain `mutableStateListOf<NavKey>()` works too. |

The app-level NavHost composes feature entries inside a `NavDisplay` with `entryProvider { ... }`:

```kotlin
import androidx.navigation3.runtime.entryProvider
import androidx.navigation3.ui.NavDisplay

NavDisplay(
    backStack = backStack,
    onBack = { if (backStack.isNotEmpty()) backStack.removeAt(backStack.lastIndex) },
    entryProvider = entryProvider {
        homeEntry(onNavigateToSeries = onNavigateToSeries)
        searchEntry(onNavigateToSeries = onNavigateToSeries)
        // ... one entry call per feature
    },
)
```

The back stack is `mutableStateListOf<NavKey>()`. Forward nav appends; back pops the last item. Cross-feature navigation flows through callbacks passed into each `<feature>Entry`, never through `NavController` (Nav3 doesn't expose one — that's the point).

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

## libs.versions.toml structure

Single catalog file at `gradle/libs.versions.toml`. Sections:

1. `[versions]` — every version is named. SDK numbers (`compileSdk = "36"`, `minSdk = "24"`, `targetSdk = "36"`) and `jvmTarget = "21"` also live here so convention plugins can read them.
2. `[libraries]` — alias keys are kebab-case scoped by ecosystem (`androidx-core-ktx`, `koin-android`, `retrofit-moshi`). Use `version.ref` for shared versions; omit `version` for BOM-managed deps.
3. `[plugins]` — Gradle plugin aliases. The 8 convention plugins are listed here with `version = "unspecified"`:

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

**TDD for everything except Compose.** ViewModels, repositories, mappers, validators, extension functions — write the test first.

**Test-after for Compose** (you need testTags and button text to write the test). Compose UI tests run on Robolectric (JVM), not androidTest.

### Stack
- JUnit 4 (`junit`)
- Mockito-Kotlin (`mockito-kotlin`)
- Google Truth (`truth`)
- Turbine for effect/flow assertions (`turbine`)
- Robolectric for Compose tests (`robolectric`)
- `MainCoroutineRule` (provided by `core/testing`)

### MainCoroutineRule — defaults to `UnconfinedTestDispatcher`

The canonical `MainCoroutineRule` in `core/testing` must default to `UnconfinedTestDispatcher`, not `StandardTestDispatcher`:

```kotlin
package <root-pkg>.testing

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.TestDispatcher
import kotlinx.coroutines.test.UnconfinedTestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.setMain
import org.junit.rules.TestWatcher
import org.junit.runner.Description

@OptIn(ExperimentalCoroutinesApi::class)
class MainCoroutineRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

**Why `UnconfinedTestDispatcher` is the default:** `BaseViewModel`'s `init { viewModelScope.launch { _actions.onEach(::onAction).collect() } }` block is what drains the action buffer. With `StandardTestDispatcher`, that `launch` doesn't run until the dispatcher is advanced (`advanceUntilIdle()` etc.) — so a test that calls `setAction(...)` and then awaits an effect via Turbine times out at 3s because `onAction` was never invoked. `UnconfinedTestDispatcher` runs launched coroutines eagerly, so `setAction` → `onAction` → `setEffect` all happen synchronously, and Turbine sees the effect immediately.

Pass `StandardTestDispatcher()` explicitly only when you need manual scheduling (e.g., testing time-based behavior with `advanceTimeBy`).

### ViewModel test pattern

```kotlin
@get:Rule
val mainCoroutineRule = MainCoroutineRule()

@Test
fun `LoadData transitions to loaded state`() = runTest {
    val items = listOf(Item("1"), Item("2"))
    whenever(repository.getItems()).thenReturn(items)

    val viewModel = FeatureViewModel(repository)
    viewModel.setAction(FeatureAction.LoadData)

    viewModel.state.test {
        assertThat(awaitItem().items).isEqualTo(items)
    }
}
```

### Compose UI test pattern

Tests in `src/test/kotlin` (NOT `androidTest`). Add `src/test/resources/robolectric.properties`:
```
sdk=34
```

```kotlin
@RunWith(AndroidJUnit4::class)
class FeatureScreenContentTest {
    @get:Rule val composeTestRule = createComposeRule()
    private val capturedActions = mutableListOf<FeatureAction>()

    private fun setContent(state: FeatureState = FeatureState()) {
        composeTestRule.setContent {
            FeatureScreenContent(state = state, onAction = { capturedActions.add(it) })
        }
    }

    @Test
    fun `clicking item dispatches OpenDetail`() {
        setContent(state = FeatureState(items = listOf(Item("42"))))
        composeTestRule.onNodeWithText("Item 42").performClick()
        assertThat(capturedActions).contains(FeatureAction.OpenDetail("42"))
    }
}
```

**Test the meaningful things:** action dispatch from clicks, state-driven conditional rendering, error state visibility. Don't test that static text renders — previews cover that.

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
