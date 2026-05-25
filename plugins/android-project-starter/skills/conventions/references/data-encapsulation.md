# Feature data layer — encapsulation patterns

Each feature's data module is **internal to the feature**. App modules (`app-mobile`, `app-tv`) never depend on `:feature:<x>:data` — only on the feature's ui module(s). The ui module aggregates the data module's Koin module into `<feature>Modules`, which is the only thing the app imports per feature.

Two reinforcing mechanisms keep this encapsulated:

1. **Gradle scope** — ui modules declare `implementation(project(":feature:<feature>:data"))`, never `api(...)`. The data module's classes are not on the app's compile classpath even transitively.
2. **Kotlin visibility** — repository implementations (and any other data-module classes the app shouldn't reach) are marked `internal`. Only the `Repository` interface and the `<feature>DataModule` Koin value are `public`; everything else stays inside the data module.

Substitute `<root-pkg>`, `<Feature>`, `<feature>` with the project's values.

## `<Feature>Repository.kt` — public interface only

```kotlin
package <root-pkg>.feature.<feature>.data

interface <Feature>Repository {
    suspend fun get<Feature>(): <Result>
}
```

## `<Feature>RepositoryImpl.kt` — `internal class`

```kotlin
package <root-pkg>.feature.<feature>.data

internal class <Feature>RepositoryImpl(
    // injected api client / dao / etc.
) : <Feature>Repository {
    override suspend fun get<Feature>(): <Result> = TODO("stub")
}
```

## `<Feature>DataModule.kt` — Koin module (the only other public symbol)

```kotlin
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

## `app-mobile/build.gradle.kts` — dependencies (and `app-tv/build.gradle.kts` analogously)

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

If you find yourself adding `implementation(project(":feature:<x>:data"))` to an app module to fix a compile error, stop: you're reaching into a feature's internals from the wrong layer. The right fix is almost always to expose what you need through the ui module's `<feature>Modules` (Koin) or a public interface in `core/model`.

## What the wizard's Step 11.2 self-check verifies

Items 6 and 7 in the structural self-check (in `init-android-project/SKILL.md`) enforce this at scaffold time:

- **Item 6** — grep each `app-*/build.gradle.kts` for `:feature:<x>:data`. Match must be empty for every feature.
- **Item 7** — grep each `feature/<x>/data/src/main/kotlin/` for `class <Feature>RepositoryImpl`. The match must be preceded by `internal`.

Both checks block the build-success gate if they fail.
