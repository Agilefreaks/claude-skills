# android-project-starter

A Claude Code plugin that scaffolds a complete, opinionated, multi-module Android project — MVI + Jetpack Compose + Koin + Navigation 3 + a `build-logic` convention plugins setup — and writes two project-local Claude skills (`<project>-android-planner` and `<project>-android-implementer`) that travel with the repo so the same conventions are enforced every time you plan or build a feature.

The wizard asks ~20 questions across 6 short rounds, resolves the latest stable versions for every dependency in parallel, generates the project, then **runs Gradle to verify the project builds before declaring done**.

## What you get

```
<project-root>/
├── .claude/skills/
│   ├── <project>-android-planner/SKILL.md
│   └── <project>-android-implementer/SKILL.md
├── app-mobile/                                (always)
├── app-tv/                                    (if TV picked)
├── build-logic/convention/                    (8 or 9 convention plugins)
├── core/{common,data,designsystem-*,model,testing,ui-*}/
├── feature/<each-feature>/{data,ui-mobile,ui-tv?}/
├── gradle/libs.versions.toml                  (latest stable, freshly resolved)
├── .lint/, .detekt/, .github/workflows/
└── settings.gradle.kts, build.gradle.kts, gradle.properties, README.md
```

Each feature module ships with a list+detail MVI screen, ViewModel, Routes, Entry, Koin module, ViewModel tests (TDD style with Turbine + Mockito-Kotlin + Truth), and Compose UI tests (Robolectric).

**Mobile + TV projects:** every feature gets a shared `data/` module plus *separate* `ui-mobile/` and `ui-tv/` modules. Each ui module has its own ViewModel, its own MVI trio (Action/State/Effect), its own Routes/Entry, and its own Compose layer. TV classes live in the `.tv` sub-package so they don't collide with mobile (same class names, different imports). `app-mobile` and `app-tv` each wire their own form-factor's modules into Koin.

## Install

The plugin is exposed via a local marketplace at `~/.claude/plugins/marketplaces/local-android/`. Once installed and enabled in `~/.claude/settings.json`, the command is available globally from any directory.

```
/plugin marketplace list             # local-android should appear
/plugin install android-project-starter@local-android --scope user
/reload-plugins
```

## Usage — three workflows

### 1. Scaffold a new project

```
mkdir my-new-app && cd my-new-app
claude
> /android-project-starter:init-android-project
```

The wizard runs in your current working directory. Pre-flight checks ensure the directory is empty or contains only `.git/` / `.idea/` / `README.md` — never overwrites a populated project.

After all files are generated, the wizard:

1. Materializes the Gradle wrapper (uses your system `gradle` if available, otherwise asks you to run `gradle wrapper`).
2. Runs `./gradlew help` (verifies `settings.gradle.kts` and `build-logic/` compile).
3. Runs `./gradlew :app-mobile:dependencies` (verifies the version catalog resolves).
4. Runs `./gradlew :app-mobile:compileDebugKotlin` (verifies all Kotlin compiles).
5. Runs `./gradlew lint test` (verifies lint passes and tests pass).

If any step fails, the wizard reads the error, fixes the file, and retries (up to 5 times per step). It will only declare the project done after all four gradle commands have passed.

### 2. Use the generated planner skill — `/<project>-android-planner`

Every scaffolded project gets its own planner skill at `.claude/skills/<project>-android-planner/SKILL.md` (where `<project>` is the kebab-case project name you chose). It knows your project's:

- Module inventory (which features exist, which `core/*` modules)
- Network library (Retrofit/Apollo/etc.)
- Persistence stack (Room/DataStore/none)
- Whether you have TV support
- Convention plugin ids
- Theming conventions

Invoke it when you're about to add a new feature or screen:

```
/<project>-android-planner   # e.g. /acme-android-planner
```

It asks 3–6 clarifying questions (product/UX angle + technical angle), then produces a detailed plan: file structure, MVI contract (Actions/State/Effects), Compose component tree, navigation, DI, data layer, edge cases, testing strategy, and implementation order. The plan is detailed enough to hand off to the implementer skill (or to another developer) without ambiguity.

If the new screen has a design, ask the user for a Figma link or PNG before planning — the component breakdown is dramatically better with a design reference.

### 3. Use the generated implementer skill — `/<project>-android-implementer`

```
/<project>-android-implementer
```

Executes a plan (just produced by the planner, or supplied directly by you). It writes code in TDD order and embeds the project's full Compose authoring rules — state management (where state lives, `mutable*StateOf` for primitives, `derivedStateOf` / `snapshotFlow`), stability & recomposition skipping (`@Immutable` / `@Stable` / `ImmutableList<T>`), modifier ordering, side-effect API picks (`LaunchedEffect` vs `rememberUpdatedState` vs `produceState` etc.), lazy-list keys + `contentType`, Material 3 motion, composition-local discipline, Canvas zero-size guards, accessibility, TV focus management (when TV is picked), and a production crash-pattern table. The rules are distilled from [compose-expert](https://github.com/aldefy/compose-skill) and adapted to this stack (Android-only, MVI in ViewModel via `StateFlow`, Navigation 3, Koin).

Implementation order:

1. MVI Contract (Actions, State, Effects)
2. ViewModel tests (RED) → ViewModel implementation (GREEN)
3. Repository + data layer (tests-first for anything with logic)
4. ScreenContent + previews
5. Compose UI tests (Robolectric — action dispatch + state-driven rendering)
6. Screen wiring (`koinViewModel()`, `HandleEffects`)
7. Navigation registration in the app's NavHost
8. Koin module registration

After every logical chunk it runs the project's quality gates: `./gradlew spotlessApply lint test`. It commits at natural boundaries on a feature branch (never on `main`).

The implementer also enforces the **Boy Scout Rule** on every file it touches — missing previews get added, missing tests get added, Screen/ScreenContent files in the same file get split.

## Connecting Figma — for design-driven workflows

Install the official Figma plugin (separate Claude Code plugin) and add your Figma API key:

```
/plugin install figma@claude-plugins-official --scope user
```

Add the key to `~/.claude/settings.json`:

```json
{
  "env": {
    "FIGMA_API_KEY": "figd_..."
  }
}
```

With Figma installed, both the planner and implementer skills will:

- Read designs via `get_design_context` when you paste a Figma URL
- Cross-reference the screenshot while building components
- Use design tokens from your Figma library where they map cleanly to your project's design system

In practice: when you start a new screen, paste the Figma URL into the planning conversation. The planner asks the Figma MCP for the design context, screenshot, and any Code Connect mappings, then bases the component tree on what's actually in the design.

## Bumping dependency versions

The plugin doesn't ship a dedicated bump command — version freshness is handled in two ways:

**At project scaffold time:** The `init` wizard resolves the latest stable versions for every entry in `libs.versions.toml` via parallel `WebSearch` / `WebFetch` calls. So a freshly scaffolded project is always on the latest stable everything.

**Over the project's lifetime:** Either let Dependabot raise PRs (the wizard wires it up if you opted in), then use the existing `dep-update-merge` plugin to bundle and verify them:

```
/plugin install dep-update-merge@agilefreaks-skills --scope user
```

Then in your scaffolded project:

```
/dep-update-merge
```

It lists open Dependabot PRs, fetches the changelog for each upstream artifact, classifies safe vs. risky bumps, and produces a single bundled branch with `spotlessCheck`, `lint`, and `test` verified before opening the merge PR.

For a manual one-off bump, edit `gradle/libs.versions.toml` directly, then run `./gradlew --refresh-dependencies build` to verify nothing broke.

## What the plugin DOES NOT do

- **It does not commit or push.** The user controls the commit boundary.
- **It does not modify files outside the current working directory.**
- **It does not install Gradle, Android SDK, or any Java runtime.** The host machine needs Android Studio (or at least JDK 21 + Android SDK) installed.
- **It does not add features after the initial scaffold.** Use the project-local `<project>-android-planner` and `<project>-android-implementer` skills for that — they understand your project's specifics in a way that a generic feature-add skill couldn't.
- **It does not bump versions in an existing project.** Use `dep-update-merge` or edit `libs.versions.toml` directly.

## Architecture cheat sheet

- **MVI**: `BaseViewModel<State, Action, Effect>` in `core/common`. State as hot `StateFlow`, effects as a single-consumer `Channel`, actions as a `MutableSharedFlow`.
- **Screen vs. ScreenContent**: every feature screen splits into `<Feature>Screen.kt` (wiring: `koinViewModel`, `HandleEffects`, navigation callbacks) and `<Feature>ScreenContent.kt` (pure UI with `(modifier, state, onAction)` signature + `@<Project>Previews`).
- **Navigation**: Navigation 3 with `@Serializable data object` / `data class` routes implementing `NavKey`. Each feature exposes `fun EntryProviderScope<NavKey>.<feature>Entry(...)`. The app-level `NavDisplay` composes them.
- **DI**: Koin. Each feature has `Module.kt` (ui ViewModels) + `di/<Feature>DataModule.kt` (data layer) aggregated into `<Feature>Modules.kt`. The Application class collects them all.
- **Build-logic**: 9 convention plugins (8 if Room not picked), one per concern. Each module applies a single convention plugin instead of three. `app-mobile`, `app-tv`, and `core/data` also apply the `android.flavors` plugin to get the `qa` (default) and `prod` product flavors.
- **Testing**: TDD for ViewModels / repos / utilities (Turbine, Mockito-Kotlin, Truth). Test-after for Compose UI (Robolectric). Tests live in `src/test/kotlin/` — never `androidTest`.
- **Resource ownership**: feature-only assets in the feature module's `res/`; cross-feature assets in `core/designsystem-mobile/res/`.

## Iterating on the plugin itself

The plugin source lives at `~/.claude/plugins/marketplaces/local-android/plugins/android-project-starter/`. Edit any `SKILL.md` file under `skills/` and run `/reload-plugins` to pick up the changes — no reinstall needed.

The single source of truth for *what good looks like* is `skills/conventions/SKILL.md`. The `init` skill defers to it for every file shape. If you change a convention there, the wizard's next run picks it up automatically.

## Skills shipped by this plugin

| Skill | Invocation | Purpose |
|---|---|---|
| `init-android-project` | `/android-project-starter:init-android-project` | The wizard. Scaffolds a new project end-to-end with the build-success gate. |
| `conventions` | (auto-loaded by `init-android-project`) | Architectural conventions reference — canonical file shapes, MVI patterns, version-lookup sources. Not user-facing. |
