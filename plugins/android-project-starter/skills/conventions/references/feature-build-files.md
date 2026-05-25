# Feature module `build.gradle.kts` — required deps

Each feature has one shared `data/` module (no flavors) and one ui module per form factor (`ui-mobile`, and `ui-tv` when TV was picked). All ui modules apply the same `<project>.android.feature` convention plugin — they share Compose, lifecycle, navigation3, and Koin Compose dependencies. Only the design-system / ui module references differ.

## `feature/<feature>/ui-mobile/build.gradle.kts`

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

## `feature/<feature>/ui-tv/build.gradle.kts` (only generated when TV was picked)

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
