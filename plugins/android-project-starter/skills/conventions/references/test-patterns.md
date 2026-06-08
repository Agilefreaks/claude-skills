# Test patterns — verbatim code

This reference holds the canonical test code shapes referenced by `SKILL.md`'s "Test conventions" section.

## `MainCoroutineRule` — defaults to `UnconfinedTestDispatcher`

`core/testing/src/main/kotlin/<root-pkg>/testing/MainCoroutineRule.kt`. The canonical rule **must** default to `UnconfinedTestDispatcher`, not `StandardTestDispatcher` — see "Why `UnconfinedTestDispatcher` is the default" below.

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

## ViewModel test pattern

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

## Compose UI test pattern

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
