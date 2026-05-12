# Compose authoring rules

These rules apply to every Compose file the wizard and the project-local implementer write. They're distilled from the patterns in [compose-expert](https://github.com/aldefy/compose-skill/tree/master/skills/compose-expert), adapted to this project's stack (Android-only, Material 3, MVI in ViewModel via `StateFlow`, Navigation 3, Koin).

## State management — where state lives and how to read it

**ViewModel owns business state. Composables never own business state.**

- Business state lives in the ViewModel as a `MutableStateFlow<<Feature>State>` exposed as `StateFlow`. The ScreenContent collects it via `viewModel.state.collectAsStateWithLifecycle()` in the Screen — never `collectAsState()`. Background lifecycle awareness is non-negotiable; `collectAsState()` keeps collecting when the app is backgrounded, burning battery and producing stale-state flashes on return.
- For one-shot events, use the project's `BaseViewModel` effects `Channel` (already provided). Don't introduce a separate `SharedFlow` for events unless you have multiple consumers.
- **Never** put `mutableStateOf(...)` inside a ViewModel — that's a Compose-runtime construct, not a domain construct.

**Composables own UI-local state only.**

- `remember { mutableStateOf(...) }` for transient UI state that doesn't need to survive process death (input focus, expanded panels, scroll position derived flags).
- `rememberSaveable { mutableStateOf(...) }` only when state must survive process death (e.g., user-entered form text on a long form). Don't use it inside list items — `rememberSaveable` writes to the savedInstanceState Bundle, which has a ~1 MB cap.
- Primitives: `mutableIntStateOf(0)`, `mutableFloatStateOf(0f)`, `mutableLongStateOf(0L)` — not `mutableStateOf(0)`. The latter boxes on every read/write.
- Observable collections: `mutableStateListOf(...)`, `mutableStateMapOf(...)`. Don't wrap them in `mutableStateOf` — they're already observable.

**Derive, don't duplicate.**

- `derivedStateOf { expensiveComputation(state) }` when a Compose-state-derived value is expensive AND read by composables that re-read often. Wrap in `remember(keys)` so the `derivedStateOf` itself survives recomposition.
- Don't use `derivedStateOf` for cheap operations (string concatenation, single boolean checks) — the overhead exceeds the benefit.
- **Anti-pattern**: deriving a count (`derivedStateOf { items.count { … } }`) while another composable iterates the full collection. The two reads can desync between snapshots. Derive the filtered collection, not the count.

**Bridge state to coroutines via `snapshotFlow`.**

- Inside a `LaunchedEffect`, `snapshotFlow { state.query }.debounce(500).collect { ... }` reacts to Compose-state changes. Reading state directly inside `LaunchedEffect(Unit)` does not retrigger when the state changes — that's a bug, not a feature. Pass the state as the key (`LaunchedEffect(query) { ... }`) or use `snapshotFlow`.

## Stability & recomposition skipping

Compose skips a composable's body if all parameters are equal to the previous call AND all parameter types are stable. Unstable parameters cause every parent recomposition to re-run the child. This is the single biggest source of jank in real Compose code.

- **`@Immutable`** on data classes whose public properties are all read-only primitives, `String`, or other `@Immutable` types. Tells the compiler equality is well-defined and structural.
- **`@Stable`** on classes that have mutable fields but expose them through `State<T>` (or `MutableState<T>`) so the compiler can observe changes through Compose's snapshot system. Use sparingly; prefer `@Immutable` for new types.
- **`ImmutableList<T>` from `kotlinx.collections.immutable`** in public composable parameters. `List<T>` is unstable — the compiler can't prove the implementation is stable. The catalog already includes `kotlinx-collections-immutable` for this reason. Use `persistentListOf(...)` or `.toImmutableList()` at the boundary where the list is built.
- **Lambdas as parameters defeat skipping** unless they're remembered. Either:
  - Pass a method reference (`onAction = viewModel::setAction`) — stable.
  - Wrap inline lambdas in `remember`: `val onClick = remember { { dispatch(Action.X) } }` — only when the lambda is hot.
  - Accept the lambda as a `(Action) -> Unit` and dispatch typed actions — our `onAction: (FeatureAction) -> Unit` pattern is already a stable shape.
- **`composeCompiler { reportsDestination = layout.buildDirectory.dir("compose_reports"); metricsDestination = layout.buildDirectory.dir("compose_metrics") }`** — add this to `build-logic/convention/AndroidLibraryComposeConventionPlugin.kt` and `AndroidApplicationComposeConventionPlugin.kt` only when actively investigating recomposition. Inspect generated `<module>_release-composables.txt` for `restartable skippable` markers — missing `skippable` means an unstable parameter is present.
- **Strong skipping mode** is on by default in Compose 1.6+. Don't fight it; structure types to be stable instead of disabling it.

## Modifier ordering — first principle: modifiers paint left-to-right

Modifier chain order is execution order, and modifiers are applied from the outside in. `Modifier.padding(16.dp).background(Red)` puts the red **inside** the padding (smaller red area, padding outside). `Modifier.background(Red).padding(16.dp)` paints red first, then pads — the red fills the unpadded area.

Memorize these:

- **Click handling lives near the inside** — wrap content tightly. `Modifier.clip(shape).clickable { … }` confines the touch target to the visible shape; `.clickable { }.clip(shape)` accepts clicks outside the clip.
- **Size constraints early, drawing late** — `.fillMaxWidth().background(Red)` colors the full width; `.background(Red).fillMaxWidth()` colors only the natural-width area, then expands.
- **Composition-local-dependent modifiers go last** in a custom modifier function so callers can override.
- **`Modifier` is the first optional parameter** (we already require this) — `modifier: Modifier = Modifier` after required params, before other optionals.
- **Prefer lambda-form layout modifiers for performance**: `Modifier.offset { IntOffset(x.toInt(), 0) }` defers the read to the layout phase (no recomposition); `Modifier.offset(x.dp, 0.dp)` is read during composition (triggers recomposition every change).

## Side effects — pick the right API

| Goal | API | Why |
|---|---|---|
| Run a suspend block tied to a key | `LaunchedEffect(key) { ... }` | Cancels and restarts when key changes. Don't pass `Unit` if the block depends on a value. |
| Acquire a resource, release on exit | `DisposableEffect(key) { ...; onDispose { ... } }` | The cleanup runs on disposal or key change. |
| Launch a coroutine from a click handler | `val scope = rememberCoroutineScope(); scope.launch { ... }` | Scope tied to composable lifetime, manually controlled. |
| Convert a callback API to state | `produceState(initial, key) { value = ... }` | Returns `State<T>`. Use for one-shot async producers. |
| Read latest value of a frequently-changing param inside a long-running effect | `val current by rememberUpdatedState(callback); LaunchedEffect(Unit) { ...; current(result) }` | Stops the effect from restarting on every param change. |
| Sync Compose state with non-Compose code after every composition | `SideEffect { ... }` | Runs after **every** successful composition — use sparingly; analytics screen-view logging is the canonical case. |
| Bridge Compose state to a Flow | `snapshotFlow { state.x }` inside an effect | Cold Flow that re-emits on state change. |

Common pitfalls:

- **Stale captured state in long-running effects.** A `LaunchedEffect(Unit)` that captures a parameter sees the **initial** value forever. Either pass the param as a key (restart on change) or wrap with `rememberUpdatedState` (read latest without restart). The decision: should the effect cancel/restart when the value changes? Yes → key. No → `rememberUpdatedState`.
- **`SideEffect` running too often.** It fires on every recomposition. If your composable recomposes 60 times a second, so does `SideEffect`.

## Lists & lazy scrolling

- **Always pass `key` to `items(...)`** when the backing list can mutate (add/remove/reorder). Without it, lazy state (animation, scroll restoration) attaches by index and produces wrong results on mutation.
  ```kotlin
  items(items = list, key = { it.id }) { item -> Row(item) }
  ```
- **`contentType` for heterogeneous lists.** Lazy lists pool composables by content type. Without `contentType`, all items share one pool and a header recycles into a row body, blowing up. Use when items render meaningfully different layouts.
  ```kotlin
  items(items = list, key = { it.id }, contentType = { it::class }) { ... }
  ```
- **Duplicate keys crash with `IllegalArgumentException`.** Backend data that ships duplicate ids (rare but happens — multi-source feed, optimistic insert, reconnect duplicate) needs a deduplication step before reaching the lazy list. Either prefix with a section (`"pinned_${it.id}"`) or carry a deduplication index in the UI model.
- **`itemsIndexed(items, key)`** when you need the index in the row — never call `list.indexOf(item)` inside `items {}` (O(n²)) and never index by `indexOf` (returns `-1` on equality miss, throws `IndexOutOfBoundsException`).
- **`derivedStateOf` for `firstVisibleItemIndex` thresholds.** `if (lazyListState.firstVisibleItemIndex > 0) ShowFab()` recomposes on every scroll position change. Wrap: `val showFab by remember { derivedStateOf { lazyListState.firstVisibleItemIndex > 0 } }` — only recomposes when the boolean flips.

## Animation & motion

- **`animate*AsState`** (`animateFloatAsState`, `animateDpAsState`, etc.) for single-value transitions driven by Compose state. The simplest tool; reach for it first.
- **`AnimatedVisibility(visible) { ... }`** for show/hide with enter/exit transitions. Pair with `EnterTransition` + `ExitTransition` factories (`fadeIn() + slideInVertically()` etc.).
- **`AnimatedContent(targetState) { value -> ... }`** for cross-fades between states (e.g., loading → content). Configure with `transitionSpec` lambda.
- **`updateTransition`** when one source state drives multiple animated values that must stay coordinated.
- **Material 3 motion tokens.** Use `MaterialTheme.motionScheme` / `MotionTokens` (or the equivalent durations/easing in the project's `core/designsystem-base`) — never hardcode `300.milliseconds` durations in feature code.
- **Defer animated reads to the layout phase** whenever the animated value drives layout. `Modifier.offset { ... }` (lambda form) instead of `Modifier.offset(value.dp, ...)` — see the modifier-ordering section.

## Composition locals — use sparingly

- **Only introduce a `CompositionLocal` when:** (a) the value needs to be available at many points in a subtree, (b) prop-drilling would touch dozens of files, and (c) the value rarely changes (a CL update triggers everything reading it to recompose).
- **Define under the design system.** Reusable composition locals belong in `core/designsystem-base` or `core/designsystem-mobile`/`core/designsystem-tv`, never inside a feature.
- **Provide at the narrowest scope** that needs the value — `CompositionLocalProvider(LocalFoo provides x) { ... }` around the subtree, not at the app root.
- **Don't use `LocalContext` to reach into platform APIs from ScreenContent.** That's a host concern; route through Effects → Screen.

## Theming (Material 3)

- **Always reference `MaterialTheme.colorScheme.*`, `MaterialTheme.typography.*`, `MaterialTheme.shapes.*`** — never hardcoded `Color(0xFF...)` or `12.sp`.
- **Custom tokens** (project-specific gradients, brand-specific spacings beyond `Spacing.xs..xl`) live in `core/designsystem-base` and are surfaced via `MaterialTheme` extensions or `CompositionLocal`. Don't sprinkle them per-feature.
- **Dark mode is on by default.** `@<Project>Previews` already generates both light and dark variants — make sure your components look correct in both. Test in the preview before reaching for `isSystemInDarkTheme()` in code.

## Canvas & DrawScope safety

- **Guard against zero size.** Canvas blocks during initial layout pass can receive `Size.Zero`. Drawing math that divides by `size.width` then explodes.
  ```kotlin
  Canvas(modifier = Modifier.fillMaxWidth().height(120.dp)) {
      if (size.minDimension <= 0f) return@Canvas
      drawCircle(/* ... */)
  }
  ```
- **Always constrain Canvas size explicitly** (`.height(...)` or `.size(...)`). Never just `.fillMaxSize()` without a height constraint — vertical-scroll parents pass infinite max height and Canvas crashes.
- **Wrap shimmer / complex draw blocks in `runCatching`** with a flat fallback color. Shimmer math under `SubcomposeLayout` is fragile.

## Accessibility

- **Every interactive element has a meaningful semantic.** Buttons with text are handled by the framework. Icon-only buttons (`IconButton { Icon(...) }`) need `contentDescription` on the Icon **or** `Modifier.semantics { contentDescription = "..." }` on the button.
- **Decorative-only Icons explicitly set `contentDescription = null`.** A null description tells TalkBack to skip the element. Don't pass `""` — that's read aloud as "empty".
- **Use `Modifier.semantics(mergeDescendants = true)`** on compound clickable rows so TalkBack reads a single combined description instead of each child individually.
- **Touch targets ≥ 48.dp.** Wrap small icons in a `Box(modifier = Modifier.size(48.dp), contentAlignment = Alignment.Center) { ... }` or use a `IconButton` (which is 48 dp by default).

## TV-specific Compose (only if TV picked)

- **D-pad focus needs explicit `FocusRequester`.** On screen entry, request focus on the primary action with `LaunchedEffect(Unit) { focusRequester.requestFocus() }`. Without this, the user lands on a screen with nothing focused and the remote does nothing.
- **`androidx.tv:tv-foundation` and `androidx.tv:tv-material`** are the TV-shaped components. Use `Carousel` for living-room horizontal pagers (handles focus and looping), `LazyRow` only as a fallback.
- **10-foot UI: minimum 16.sp text, 48.dp touch targets** (same as mobile a11y), but plan for distance — increase typography one scale level higher than mobile.
- **Test on actual TV emulators**, not just resizable phones. Focus traversal behaves differently.

## Crash patterns to avoid (production playbook)

| Symptom | Cause | Fix |
|---|---|---|
| `IllegalArgumentException: Key … was already used` | Duplicate keys in `LazyColumn { items(list, key) }` | Deduplicate at the data layer; or prefix keys with section ID |
| `IndexOutOfBoundsException` from a list | `list.indexOf(item)` inside `items {}` returns `-1` | Use `itemsIndexed(list, key)` instead |
| `NaN`/`Infinity` math errors in `Canvas` | `size.width == 0` during initial layout | Guard with `if (size.minDimension <= 0f) return@Canvas` |
| Background battery drain after navigating away | `collectAsState()` keeps collecting in background | Use `collectAsStateWithLifecycle()` |
| Stale value in callback inside long-running `LaunchedEffect` | Lambda captured the initial param | `rememberUpdatedState(callback)` |
| Composable recomposes constantly with the same data | An unstable parameter (raw `List<T>`, plain class) | Switch to `ImmutableList<T>` or annotate `@Immutable` |
| `OutOfMemoryError` on rotation with a long form | `rememberSaveable` used inside list items | Hoist saveable state to screen level |

## Performance verification

- **Compose compiler metrics** — enable per-module in the conventions Compose plugins only when investigating. Inspect `build/compose_metrics/*.txt` and `build/compose_reports/*-composables.txt`. Every composable should be `restartable skippable`. Any that aren't have an unstable parameter — find it and fix the type.
- **Layout Inspector → "Show Recomposition Counts"** during development. A counter that hits triple digits during a normal interaction is a smell.
- **Macrobenchmark + baseline profiles** for app startup and primary user flows (only when shipping to production). Run after every major feature ships. Set up the baseline profile generator once; re-run when a P95 startup regression appears.

## Deprecated patterns to avoid

- **`androidx.compose.material:material` (M2)** — we use Material 3 (`androidx.compose.material3:material3`). The M2 artifact is for legacy projects only.
- **`mutableStateOf<Int>(0)`** when `mutableIntStateOf(0)` exists. The latter is faster and clearer.
- **`Modifier.absolutePadding(...)`** — use `padding(...)` with `LayoutDirection`-aware semantics.
- **`onCommit`, `onActive`, `onDispose` (Compose 1.0-era)** — replaced by `LaunchedEffect` and `DisposableEffect`. If you see these in old samples, treat as deprecated.
- **`androidx.compose.foundation.Lazy*Grid` snap behaviors** built by hand — use the official snapping APIs.
- **String-based navigation routes** (`"home/$id"`) — Navigation 3 with `@Serializable` `NavKey` data classes / objects is type-safe; we already use this.

