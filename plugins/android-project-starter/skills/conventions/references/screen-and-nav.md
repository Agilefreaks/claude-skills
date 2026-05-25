# Screen / ScreenContent contract and Navigation 3 wiring — code examples

This reference holds the Kotlin code samples for the Screen/ScreenContent split and the Navigation 3 wiring. The rules and rationale live in `SKILL.md`'s "Screen / ScreenContent contract" and "Navigation 3 wiring" sections.

## Screen + ScreenContent split

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

## Right vs. wrong: `@<Project>Previews` vs. stacked `@Preview`

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

## Navigation 3 — feature entry

Each feature exposes an `<Feature>Entry` extension on `EntryProviderScope<NavKey>` that registers its routes:

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

## Navigation 3 — app-level NavHost

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
