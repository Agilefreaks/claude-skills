---
name: init
description: >
  Interactive wizard that scaffolds a brand-new Android project in the current working
  directory. Walks the user through identity, target platforms, SDK levels, architecture
  choices (network, persistence, DI, auth), cross-cutting concerns (Firebase, analytics,
  logging), UI defaults, tooling, and the main feature list. Resolves latest stable
  versions, generates build-logic + core + feature modules, app(s), tooling configs, CI,
  writes project-local planner and implementer skills, then verifies the project builds
  before declaring done.
disable-model-invocation: true
---

# Android Project Starter — Wizard

You are running an interactive wizard that scaffolds a new Android project in the **current working directory**. The user invoked you explicitly via `/android-project-starter:init`. Don't assume — confirm, gather answers in phased rounds, generate, then **verify the project builds before considering the job done**.

## Step 0 — Load conventions and pre-flight

1. **Load the conventions skill** (`android-project-starter:conventions`). It is the source of truth for module layout, MVI shape, build-logic plugins, theming, tests, version-lookup sources, and tooling defaults. **Every file you generate must match the canonical shapes shown there.** The conventions skill calls out specific traps (CommonExtension, helper files, package declarations) — follow those rules literally.

2. **Confirm output directory and emptiness:**
   - Run `pwd` to confirm cwd.
   - Run `ls -la` to see what's already there.
   - If the directory is non-empty *and* contains files that suggest an existing project (`build.gradle`, `build.gradle.kts`, `settings.gradle*`, `gradle.properties`, `gradle/`, `app/`, `app-*/`, `core/`, `feature/`, `.git/` with content beyond a fresh init), **stop and ask the user** whether to (a) scaffold somewhere else, (b) cancel, or (c) proceed anyway (dangerous). Never overwrite a populated project without explicit confirmation.
   - An empty directory or one with only `.git/`, `.idea/`, `README.md`, or `.DS_Store` is fine — proceed.

3. **Greeting:** tell the user briefly what's about to happen — "I'll ask 6 short rounds of questions (~20 total), resolve the latest stable versions in parallel, generate the project, then run `./gradlew help`, `assembleDebug`, `lint`, and `test` to verify. Total time ~5–10 minutes."

## Step 1 — Identity round (plain-text)

Ask in **one chat message**, asking the user to respond inline:

```
Let's start with project identity. Please reply with:

  1. Project name (kebab-case, e.g. `acme`): ____
  2. Application ID (e.g. `com.acme.android`): ____
  3. App display name (shown on the launcher, e.g. "Acme"): ____
  4. Root Kotlin package (defaults to the applicationId if blank): ____
```

Derive:
- **Project class prefix** = PascalCase of project name (`acme` → `Acme`, `super-app` → `SuperApp`). Used for `<Project>Theme`, `<Project>Previews`, `<Project>Application`, color tokens.
- **Convention plugin id prefix** = lowercase project name with hyphens stripped (`super-app` → `superapp`). Plugin ids become `<prefix>.android.application`, etc.

Echo the derived values so the user can correct typos before proceeding.

## Step 2 — Targets + SDK (AskUserQuestion, 4 questions)

1. **Platforms** — Mobile only (recommended) / Mobile + TV / TV only
2. **minSdk** — 24 (Android 7) / 26 (Android 8) / 28 (Android 9) / 21 (Android 5)
3. **compileSdk / targetSdk** — 36 (latest, recommended) / 35 / 34
4. **JVM target** — 21 (recommended) / 17

## Step 3 — Architecture (AskUserQuestion, 4 questions)

1. **Network** — Retrofit + Moshi / Apollo (GraphQL) / Both / None
2. **Persistence** — Room + DataStore / Room only / DataStore only / None
3. **Image loading** — Coil 3 (recommended) / Glide / None
4. **Auth scaffolding** — Yes (feature/auth + session store + token interceptor) / No

## Step 4 — Cross-cutting (AskUserQuestion, 4 questions)

1. **Firebase bundle** (multiSelect) — Crashlytics / Analytics / Messaging / Remote Config / None of these
2. **Analytics** (skip if Firebase Analytics already picked) — Mixpanel / Amplitude / None
3. **Logging** — Timber (recommended) / kotlin-logging / println-only
4. **Build variants** — debug + release only / dev/staging/prod × debug/release / dev/prod × debug/release

## Step 5 — UI defaults (AskUserQuestion, 4 questions)

1. **Theme** — Material 3 + dynamic color / Material 3 (custom colors only) / Material 2
2. **Dark mode** — Follow system / Always on / Always off
3. **Splash + onboarding** — Both / Splash only / Neither
4. **Edge-to-edge** — Yes (recommended) / No

## Step 6 — Tooling (AskUserQuestion, 4 questions)

1. **Screenshot tests** — None (default) / Roborazzi / Paparazzi
2. **CI** — GitHub Actions full (build + lint + spotless + detekt + test) / Build only / None
3. **Pre-commit hook** — Spotless apply / None
4. **Dependabot** — Yes (gradle + github-actions) / No

## Step 7 — Features (plain text)

```
What are the main features of your app? Comma-separated, kebab-case
(e.g. `home, search, profile, settings`). For each feature I'll scaffold:

  feature/<name>/data/        — repository + Koin module
  feature/<name>/ui-mobile/   — MVI screen (list + detail), ViewModel + tests,
                                Compose previews + UI tests
```

If the user picked TV in Step 2, each feature also gets `feature/<name>/ui-tv/` with **its own independent ViewModel + MVI trio + Compose layer** (in the `.tv` sub-package). The `feature/<name>/data/` module is shared between mobile and TV; the ui layers are not.

## Step 8 — Echo plan, confirm

Print a concise summary of all answers (~15 lines). Ask the user to confirm before generating. If they want to change anything, jump back to that step.

## Step 9 — Resolve latest versions

Use `WebSearch` and `WebFetch` per the **Version-lookup sources** section in the conventions skill. **Latest stable only — skip alpha/beta/rc/SNAPSHOT/M*/eap/dev.** Resolve in parallel batches; total time should be under a minute.

Print the resolved version table to the user so they can flag anything they want to pin to an older version.

## Step 9.5 — Pre-flight compatibility check

Before generating files, run a compatibility-matrix check on the resolved versions and pick the right template variant. This catches "latest stable everything" combinations that don't compile together.

**Required checks:**

1. **AGP ↔ `android.builtInKotlin`** — read the resolved AGP version:
   - **AGP ≥ 9.2.0** → set `android.builtInKotlin=true` (or omit) and DO NOT apply `org.jetbrains.kotlin.android` manually in the convention plugins. AGP's new DSL hard-errors otherwise.
   - **AGP 9.0.x – 9.1.x** → set `android.builtInKotlin=false` and apply Kotlin manually. Early AGP 9 broke KSP without this.
   - **AGP 8.x** → no explicit `builtInKotlin` setting needed; apply Kotlin manually (this is the pre-AGP-9 default).
   
   Pick the matching `AndroidApplicationConventionPlugin.kt` / `AndroidLibraryConventionPlugin.kt` template variant from the conventions skill.

2. **Kotlin ↔ Compose Compiler plugin** — they must share the same version. Both reference `kotlin = "<X.Y.Z>"` in the catalog. Confirm `compose-gradlePlugin` resolves to `org.jetbrains.kotlin:compose-compiler-gradle-plugin:<kotlin version>`.

3. **Kotlin ↔ KSP** — KSP versioning is `<kotlin major.minor.patch>-<ksp revision>`. If the resolved Kotlin is `2.3.21`, KSP must be `2.3.21-X.Y.Z` (or, if no patch-matching KSP exists yet, `2.3.20-X.Y.Z` is usually safe). Mismatched KSP fails at configuration time with `KSP not compatible with Kotlin X.Y.Z`.

4. **Compose BOM ↔ Compose Compiler** — the BOM month should not lead the compiler version by more than ~3 months. If BOM is `2026.04.01` and compiler is from a Kotlin from 2025, downgrade the BOM to the matching window or upgrade Kotlin.

5. **Robolectric ↔ Android SDK** — `robolectric.properties` `sdk=` value must be ≤ Robolectric's supported SDK ceiling. Robolectric 4.16 supports up to Android 14 (SDK 34); newer SDKs may require Robolectric `5.x`. Default `sdk=34`; bump only if Robolectric supports it.

6. **navigation3 ↔ Compose BOM** — Nav3 1.1.x targets Compose 1.7+. If Compose BOM resolves to a Compose < 1.7, downgrade Nav3 or upgrade Compose.

**Output:** print the matrix to the user so they can confirm or override:

```
Compatibility check:
  AGP                          9.2.0  → using built-in Kotlin variant of convention plugins ✓
  Kotlin                       2.3.21
  Compose Compiler             2.3.21 ✓ matches Kotlin
  KSP                          2.3.21-2.0.0 ✓ matches Kotlin patch
  Compose BOM                  2026.04.01
  navigation3                  1.1.1 ✓ targets Compose 1.7+
  Robolectric                  4.16.1 → robolectric.properties sdk=34 ✓

All compatible.
```

If a check fails, propose a fix (downgrade or upgrade one specific library) and ask the user to confirm before proceeding. Don't push through with a known-incompatible set — that's the bug class that ate ~3 min on real scaffolds.

## Step 10 — Generate files

Follow the conventions skill for every file. Strict rules:

- **Reproduce the canonical plugin file contents in the conventions skill verbatim** for the 8 build-logic plugins and `ProjectExtensions.kt`. Do not refactor, do not extract helpers, do not add `package` declarations.
- **Use `ApplicationExtension` and `LibraryExtension` directly** — never `CommonExtension`. This is the AGP 9 trap that breaks projects.
- **Skip `AndroidRoomConventionPlugin.kt`** entirely if Room was not picked (and remove its `register("androidRoom")` block from `build-logic/convention/build.gradle.kts`).

### 10.1 — Root scaffolding
- `gradle.properties` — **for AGP ≥ 9.2.0** (the default for new scaffolds): include `android.builtInKotlin=true` (or omit; it's the default with `android.newDsl=true`). **For AGP 9.0.x–9.1.x**: include `android.builtInKotlin=false`. Always include `kotlin.code.style=official`, `org.gradle.parallel=true`, `org.gradle.caching=true`, `android.useAndroidX=true`, `android.nonTransitiveRClass=true`. See the conventions skill's "AGP version matters" note.
- `settings.gradle.kts` — per conventions, `includeBuild("build-logic")`, `rootProject.name = "<project-name>"`, `include(":app-mobile")` and every core + feature module
- Root `build.gradle.kts` — only `apply false` plugin declarations; Spotless + detekt configured in a `subprojects { }` block
- `local.properties` (placeholder), `.lint/config.xml` (minimal lint baseline)
- `.editorconfig` — must include the Compose-aware ktlint defaults below so the first lint pass after generation doesn't trip on Composable function naming or the MVI `_actions` backing-property pattern. Substitute `<Project>Previews` with the project's actual multi-preview annotation name (e.g. `AcmePreviews`):

  ```ini
  root = true

  [*]
  charset = utf-8
  end_of_line = lf
  insert_final_newline = true
  trim_trailing_whitespace = true

  [*.{kt,kts}]
  indent_style = space
  indent_size = 4
  max_line_length = 140
  ij_kotlin_allow_trailing_comma = true
  ij_kotlin_allow_trailing_comma_on_call_site = true

  # Compose @Composable functions are PascalCase; ktlint's standard rule expects camelCase.
  ktlint_standard_function-naming = disabled
  ktlint_function_naming_ignore_when_annotated_with = Composable, Preview, <Project>Previews

  # BaseViewModel uses `_actions` / `_effects` as private buffers with no public mirror
  # (actions are dispatched via setAction(), effects exposed as a Flow only).
  ktlint_standard_backing-property-naming = disabled

  [*.{xml,yml,yaml,json,md}]
  indent_style = space
  indent_size = 2
  ```
- `.gitignore` — comprehensive Android+Kotlin+Gradle ignore. Use this exact body (don't trim):

  ```gitignore
  # Gradle
  .gradle/
  build/
  **/build/
  out/
  
  # Gradle wrapper — DO commit gradle-wrapper.jar and .properties; ignore only backups
  gradle/wrapper/gradle-wrapper.jar.bak
  
  # Android Studio / IntelliJ
  .idea/
  *.iml
  *.ipr
  *.iws
  captures/
  .navigation/
  
  # Kotlin
  .kotlin/
  
  # Android build outputs
  *.apk
  *.aab
  *.ap_
  *.dex
  *.class
  bin/
  gen/
  release/
  
  # Local config (per-developer, never commit)
  local.properties
  
  # Logs / OS junk
  *.log
  .DS_Store
  Thumbs.db
  
  # Lint cache
  .lint/cache/
  
  # Test coverage
  *.exec
  
  # Crashlytics
  crashlytics.properties
  crashlytics-build.properties
  fabric.properties
  
  # Firebase — google-services.json is normally committed; leave the line below commented.
  # Uncomment only if your project policy keeps it out of source control.
  # app-mobile/google-services.json
  # app-tv/google-services.json
  
  # Generated keystores (debug keystores are auto-generated; release keystores must never be in repo)
  *.jks
  *.keystore
  ```
- `gradle/wrapper/gradle-wrapper.properties` (pin Gradle to the latest stable that supports the chosen AGP)

### 10.2 — Version catalog
- `gradle/libs.versions.toml` per conventions (with resolved versions from Step 9). Include the 8 convention plugin aliases with `version = "unspecified"`.

### 10.3 — build-logic
- `build-logic/settings.gradle.kts` per conventions (reads `../gradle/libs.versions.toml`)
- `build-logic/convention/build.gradle.kts` — exactly the shape shown in the conventions skill, registering all 8 plugins (or 7 if Room was skipped)
- The 8 plugin Kotlin files (or 7) — verbatim shapes from the conventions skill, **no helper files**, **no package declarations**
- `ProjectExtensions.kt` — verbatim from conventions

### 10.4 — core modules
- `core/common/` — `BaseViewModel`, `ViewAction`, `ViewState`, `ViewSideEffect`
- `core/model/` — placeholder; uses `<project>.jvm.library`
- `core/data/` — Koin Qualifiers, session store interface (if Auth), network interceptor scaffold (if Network)
- `core/designsystem-base/` (or `core/designsystem/` if mobile-only) — color tokens, typography, Spacing
- `core/designsystem-mobile/` — `<Project>Theme`, `@<Project>Previews`, `<Project>Loading`, `<Project>ErrorState`, `HandleEffects`
- `core/ui-mobile/` — shared composables placeholder
- `core/testing/` — `MainCoroutineRule` + shared test helpers
- If TV: `core/designsystem-tv/`, `core/ui-tv/`

### 10.5 — feature modules

For each user-listed feature `<feature>`:

- `feature/<feature>/data/` — **shared between mobile and TV.** Build file applies `<prefix>.android.library` + `<prefix>.android.lint` (+ `ksp` if Moshi codegen / Room). Generates `<Feature>Repository.kt` (interface + stub impl) and `di/<Feature>DataModule.kt` exposing `val <feature>DataModule`.
- `feature/<feature>/ui-mobile/` — **owns its own MVI stack**: `build.gradle.kts` applying `<prefix>.android.feature`, Route + Entry in `<root-pkg>.feature.<feature>`, MVI trio, `<Feature>ViewModel`, Screen + ScreenContent + `@<Project>Previews` + Robolectric Compose tests, `Module.kt` with `val <feature>Module` registering the mobile VM, `<Feature>Modules.kt` aggregating `<feature>Module + <feature>DataModule`. Depends on `:core:designsystem-mobile` and `:core:ui-mobile`.
- **If TV: `feature/<feature>/ui-tv/`** — **independent MVI stack, in the `.tv` sub-package**. Same shape as ui-mobile but under `<root-pkg>.feature.<feature>.tv` so class names (`<Feature>ViewModel`, `<Feature>Route`, `<Feature>Action/State/Effect`, `<Feature>Screen`, `<Feature>ScreenContent`) don't collide. Has its own `<Feature>ViewModel` whose state, actions, and effects can diverge from mobile (focus-driven nav, D-pad input, different layouts). Has its own Koin `Module.kt`, its own `<Feature>Modules.kt` aggregator (same identifier `<feature>Modules`, but in the `.tv` package — `app-tv` imports it from there). Depends on `:core:designsystem-tv` and `:core:ui-tv`, and reuses the **same** `:feature:<feature>:data` module. Has its own ViewModel tests and Compose UI tests under `src/test/kotlin/`.

**Rule:** every ui module has its own ViewModel. Never share a ViewModel between mobile and TV. The data module is shared, the ui logic is not.

### 10.6 — app modules
- `app-mobile/build.gradle.kts` applying `<prefix>.android.application` + `<prefix>.android.application.compose` + `<prefix>.android.lint` (+ serialization, google-services, crashlytics as needed)
- `AndroidManifest.xml`
- `<Project>Application.kt` — `startKoin { ... modules(...) }`
- `MainActivity.kt` — `enableEdgeToEdge()`, `setContent { <Project>Theme { <Project>NavHost() } }`
- `navigation/<Project>NavHost.kt`, `BottomNavTab.kt`, `<Project>BottomNav.kt`
- splash/ if picked
- If TV: `app-tv/` analogous

### 10.7 — Tooling

- **`.detekt/config.yml`** — must include Compose-aware overrides so the first detekt pass doesn't trip on Preview composables and scaffold-injected fields. Substitute `<Project>Previews` with the actual project preview annotation:

  ```yaml
  build:
    maxIssues: 0

  style:
    UnusedPrivateMember:
      active: true
      allowedNames: '(_|ignored|expected|serialVersionUID)'
      # Preview functions and Composable helpers are tooling-only — detekt can't see the
      # @Preview entry points so it'd flag every one of them as unused.
      ignoreAnnotated: ['Preview', '<Project>Previews', 'Composable']
    UnusedPrivateProperty:
      active: true
      # Repository / service / store fields are injected at scaffold time and used later
      # by the implementer. Pre-allow the common names so detekt doesn't fail the build
      # on a fresh scaffold.
      allowedNames: '(_|ignored|expected|serialVersionUID|repository|service|store|client|dispatcher|context)'
    MagicNumber:
      active: false   # Compose/ Material uses sp/dp magic numbers; whole-file suppress isn't useful
    LongMethod:
      active: false   # ScreenContent composables can be long without being complex

  naming:
    MatchingDeclarationName:
      active: true
      # Allow Kind enums / Variant sealed classes to coexist with their parent composable in one file
      # if you really want; we don't pre-disable, since the right move is usually to split files.

  complexity:
    LongParameterList:
      active: true
      functionThreshold: 8     # Composable parameter lists run long; raise from the default 6
      constructorThreshold: 8
  ```

- **Spotless block in root build.gradle.kts** — target `**/*.kt` and `**/*.kts`, ktlint at the catalog-pinned version.
- If pre-commit: `.githooks/pre-commit` + setup note.
- If CI: `.github/workflows/ci.yml`.
- If Dependabot: `.github/dependabot.yml`.

### 10.8 — Firebase (if any)

- Apply the google-services + crashlytics plugins in `app-mobile/build.gradle.kts` (and `app-tv/` if TV).
- Add the chosen Firebase libs (`firebase-crashlytics`, `firebase-analytics`, etc.) using the Firebase BOM.
- Write `app-mobile/google-services.json` (and `app-tv/google-services.json` if TV) using the **schema-valid placeholder** below — a minimal-but-valid JSON the google-services plugin will accept so the project compiles. The values are dummies; Firebase calls won't work at runtime until the user replaces it.

A bare `{ "project_id": "X" }` placeholder is **not enough** — the google-services plugin's schema check rejects it during `processDebugGoogleServices` with errors like `No matching client found for package name`. Use this exact shape, substituting `<applicationId>` with the user's actual `applicationId` (Step 1):

```json
{
  "project_info": {
    "project_number": "000000000000",
    "project_id": "placeholder",
    "storage_bucket": "placeholder.appspot.com"
  },
  "client": [
    {
      "client_info": {
        "mobilesdk_app_id": "1:000000000000:android:0000000000000000",
        "android_client_info": {
          "package_name": "<applicationId>"
        }
      },
      "oauth_client": [],
      "api_key": [
        { "current_key": "AIzaSyPLACEHOLDER_REPLACE_BEFORE_SHIPPING_PLACEHO" }
      ],
      "services": {
        "appinvite_service": {
          "other_platform_oauth_client": []
        }
      }
    }
  ],
  "configuration_version": "1"
}
```

If the user picked build variants with different application IDs (e.g. `<appId>.dev`, `<appId>.staging`, `<appId>.prod`), add one `client` entry per variant — the google-services plugin matches `package_name` against the resolved applicationId per variant. The placeholder must match every variant or it'll fail one of them.

**After writing**, print a prominent warning to the user:

```
⚠️  The google-services.json files are PLACEHOLDERS. Firebase/Crashlytics will compile
    but not report at runtime. Replace app-mobile/google-services.json (and app-tv if TV)
    with the real file from the Firebase Console before shipping.
```

### 10.9 — Project-local skills

Generate the project-local planner and implementer skills with this layout:

```
<project-root>/.claude/skills/
├── <project>-android-planner/
│   ├── SKILL.md
│   └── references/
│       └── project-architecture.md       # module inventory, MVI base types, navigation map
└── <project>-android-implementer/
    ├── SKILL.md
    └── references/
        ├── project-architecture.md       # symlink or copy of the planner's reference
        └── compose-authoring.md          # the full Compose rules (copy + project-adapt)
```

**Rules for the generated files:**

- **`SKILL.md` is short and operational.** ~200–400 lines for the planner, ~400–600 lines for the implementer. It covers workflow (TDD-for-VM / test-after-for-Compose, branch setup, quality gates, Boy Scout Rule, commit cadence, before-you're-done checklist), the MVI base type signatures, the `(modifier, state, onAction)` ScreenContent contract, and the project-specific module inventory. The SKILL.md does NOT inline the Compose authoring rules; it points at `references/compose-authoring.md` and tells the reader to re-read the relevant section before any Compose decision.
- **`references/compose-authoring.md`** is generated by copying the content of `android-project-starter:conventions` → `references/compose-authoring.md` verbatim, then adapting:
  - Replace generic `<Project>` / `<root-pkg>` placeholders with the actual project's class prefix and root package.
  - Replace generic file references (`HomeScreenContent.kt`) with the actual file path under this project's feature inventory (e.g. `feature/home/ui-mobile/src/main/kotlin/.../HomeScreenContent.kt`) — pick whichever feature the user generated as the closest canonical example.
  - Remove the TV-specific subsection if the project didn't pick TV. Keep it if TV was picked, and point at the actual TV-flavored feature module.
  - Replace the generic `LocalScrollFade` / design-system token examples with whatever this project actually generated (or strike the example if it doesn't apply).
- **`references/project-architecture.md`** captures the module inventory, MVI base type signatures, navigation map, GraphQL/Retrofit operation list (if any), Koin module conventions, and theme tokens. The planner and implementer both load it at the start of every run. Generate one canonical copy and either symlink it from both skill directories or copy it; either is fine.
- **The implementer's SKILL.md "Compose authoring rules" section is a pointer block**, not a content section. Use this exact shape, substituting project-specific names:

  ```markdown
  ## Compose authoring rules

  **Read `references/compose-authoring.md` before touching any Compose file.** It is
  the operational source of truth for Compose in this project. Re-read the relevant
  section whenever you're about to:

  - Decide where state lives (VM `StateFlow` vs `remember` vs `rememberSaveable`)
  - Make a composable skippable (`@Immutable`, `@Stable`, `ImmutableList<T>`,
    stable lambdas)
  - Order modifiers (chain order = visual order; click handling near content;
    size before background; lambda-form layout modifiers)
  - Pick a side-effect API (`LaunchedEffect` vs `DisposableEffect` vs
    `rememberUpdatedState` vs `produceState` vs `snapshotFlow` vs `SideEffect`)
  - Build a list (`key`, `contentType`, `itemsIndexed`, `derivedStateOf` for
    scroll thresholds)
  - Add animation (`animate*AsState`, `AnimatedVisibility`, `AnimatedContent`,
    `updateTransition`, M3 motion tokens)
  - Introduce a `CompositionLocal`
  - Use `Canvas`
  - Address accessibility
  - Diagnose a Compose crash (the table in §Crash patterns to avoid)
  - Profile recomposition
  ```

This split keeps the SKILL.md scannable while keeping the operational depth available on demand. The planner skill is shorter and doesn't ship `compose-authoring.md` — it ships `project-architecture.md` and trusts the implementer to load Compose rules when it actually writes code.

### 10.10 — README

Write `<project-root>/README.md` summarizing the project: stack (Kotlin + Compose + MVI + Koin), module layout, how to run, how to test, links to the two `.claude/skills/` for planning and implementation. Short — ~50 lines.

## Step 11 — Build-success gate (REQUIRED — do not skip)

After all files are written, **verify the project builds before declaring done**.

### 11.1 — Materialize the Gradle wrapper

If `./gradlew` doesn't exist yet, materialize it in this order — try each path and stop at the first success:

1. **System Gradle on PATH** — `command -v gradle && gradle --version`. If present, run `gradle wrapper --gradle-version <resolved-gradle-version> --distribution-type bin` in cwd. Done.

2. **Reuse an existing wrapper jar from another local project.** Search common locations:
   ```bash
   find ~/Documents ~/Projects ~/code ~/workspace ~/IdeaProjects 2>/dev/null \
     -maxdepth 5 -name "gradle-wrapper.jar" -not -path "*/build/*" \
     -not -path "*/.gradle/*" 2>/dev/null | head -5
   ```
   If any are found, pick the most recently modified one. Copy it into `<project>/gradle/wrapper/gradle-wrapper.jar`. The wrapper jar is forward-compatible — any reasonably recent jar (Gradle 8.x+) can bootstrap any target version. Also copy the matching `gradlew` script and `gradlew.bat` from the same project. Then write `gradle/wrapper/gradle-wrapper.properties`:
   ```properties
   distributionBase=GRADLE_USER_HOME
   distributionPath=wrapper/dists
   distributionUrl=https\://services.gradle.org/distributions/gradle-<resolved-gradle-version>-bin.zip
   networkTimeout=10000
   validateDistributionUrl=true
   zipStoreBase=GRADLE_USER_HOME
   zipStorePath=wrapper/dists
   ```
   First `./gradlew` invocation will download the target distribution and the wrapper will then self-update on next run. Once the daemon is up, run `./gradlew wrapper --gradle-version <resolved-gradle-version>` to canonicalize the bootstrap.

3. **Download from gradle.org as last resort.** Curl the wrapper jar directly:
   ```bash
   mkdir -p gradle/wrapper
   curl -sSL -o gradle/wrapper/gradle-wrapper.jar \
     "https://raw.githubusercontent.com/gradle/gradle/v<resolved-gradle-version>/gradle/wrapper/gradle-wrapper.jar"
   curl -sSL -o gradlew \
     "https://raw.githubusercontent.com/gradle/gradle/v<resolved-gradle-version>/gradlew"
   curl -sSL -o gradlew.bat \
     "https://raw.githubusercontent.com/gradle/gradle/v<resolved-gradle-version>/gradlew.bat"
   chmod +x gradlew
   ```
   Then write `gradle-wrapper.properties` per (2).

4. **If steps 1–3 all fail**, tell the user the wrapper couldn't be auto-materialized and they need to install Gradle (`brew install gradle` on macOS) and run `gradle wrapper --gradle-version <ver>` themselves. Don't hard-block — continue to Step 11.2 (structural self-check) so they have a partially-validated scaffold to work with.

Verify the wrapper works: `./gradlew --version`. If that fails, retry (2) or (3) with a different source.

### 11.2 — Structural self-check (before running gradle)

Before invoking gradle, verify the generated structure is complete. For **every** feature module, confirm:

1. `<Feature>Screen.kt` exists (wiring: `koinViewModel`, `HandleEffects`, no preview).
2. `<Feature>ScreenContent.kt` exists as a separate file (pure UI with `(modifier, state, onAction)` signature, **no Koin/Activity deps**).
3. `<Feature>ScreenContent.kt` contains **at least one** `@<Project>Previews`-annotated composable that constructs a sample `State` and `onAction = {}`. If the feature has meaningful loading/empty/error states, add a preview for each.
4. `<Feature>ScreenContentTest.kt` exists in `src/test/kotlin/` and contains Robolectric Compose tests that target **`<Feature>ScreenContent`** (not `<Feature>Screen`). Tests must cover at least: one action-dispatch assertion (click → captured action) and one state-driven rendering assertion. **Compose tests against `<Feature>Screen` are wrong — Screen depends on Koin and can't be instantiated in a unit test.**
5. `<Feature>ViewModelTest.kt` exists with at least one action-handling test and one effect-emission test (use Turbine `effects.test { awaitItem() }`).

If any of these are missing for any feature, add them before running gradle. If TV is enabled, both `ui-mobile` and `ui-tv` must satisfy 1–5 independently (each ui module owns its own ScreenContent, previews, and tests).

### 11.3 — Run the verification cycle

Run each step. If a step fails, **read the error, fix the offending file(s), and re-run**. Iterate until all five pass. Use `--no-daemon` to avoid stale-daemon false negatives.

1. `./gradlew help --no-daemon` — verifies `settings.gradle.kts` parses and `build-logic/` compiles. Most "convention plugin doesn't compile" errors surface here.
2. `./gradlew :app-mobile:dependencies --no-daemon --configuration debugCompileClasspath` — verifies the version catalog resolves and module dependencies are reachable.
3. `./gradlew :app-mobile:compileDebugKotlin --no-daemon` (and `:app-tv:compileDebugKotlin` if TV) — verifies Kotlin source compiles across all modules.
4. `./gradlew :feature:<first>:ui-mobile:compileDebugUnitTestKotlin --no-daemon` — **test-compile smoke check**. Pick any one feature module and compile its unit tests. Catches generated-test issues (wrong imports, stale type references in mocks, Compose test-class wiring) before the full test suite runs. A failure here usually means a generated test file references a symbol that doesn't exist (e.g. a Nav3 import the conventions skill once recommended but that isn't a real API) — fix the template, then re-run.
5. `./gradlew spotlessCheck detekt lint test --no-daemon` — runs Spotless, detekt, Android Lint, and unit tests across the project. (If `spotlessCheck` fails, run `./gradlew spotlessApply` and re-run.)

**Fix-loop policy:**
- Read the full stdout AND stderr from a failure. Identify the file + line.
- Cross-reference against the conventions skill — if the broken file diverges from the canonical shape, restore the canonical shape.
- Common failures to expect:
  - `CommonExtension` used anywhere → replace with `ApplicationExtension` or `LibraryExtension`
  - Helper file in `build-logic/convention/src/main/kotlin/` → delete it, inline its body where it was used
  - Plugin class has a `package` declaration → remove it
  - `implementationClass` in `gradlePlugin.plugins` doesn't match a top-level class name → align them
  - Catalog alias unresolved → check spelling matches `libs.versions.toml`
  - **`Unresolved reference 'entry'`** in an `<Feature>Entry.kt` file → remove the `import androidx.navigation3.runtime.entry` line; `entry<K>` is a member of `EntryProviderScope<T>`, not a top-level function (see conventions skill Nav3 section).
  - **`Unresolved reference 'androidx'` / `'kotlinx'` / `'koin'` in module build scripts** → the type-safe catalog accessor isn't visible to the subproject classpath on Gradle 9.5.1 + AGP 9.2. Switch the affected module's `build.gradle.kts` to the `lib("alias")` helper form (see conventions skill).
  - **Turbine `awaitItem()` times out at 3s** → `MainCoroutineRule` is using `StandardTestDispatcher`. The canonical rule defaults to `UnconfinedTestDispatcher` so `BaseViewModel.init { … }` action-collector launches synchronously. Fix the `MainCoroutineRule` template in `core/testing`.
  - **`Unable to resolve activity for Intent { ... cmp=org.robolectric.default/androidx.activity.ComponentActivity }`** → the Compose convention plugins are missing `testOptions.unitTests.isIncludeAndroidResources = true`. Add it to `AndroidLibraryComposeConventionPlugin` and `AndroidApplicationComposeConventionPlugin`.
  - **`The 'org.jetbrains.kotlin.android' plugin is not compatible with AGP's 9.0 new DSL`** → AGP 9.2+ auto-applies Kotlin. Remove the manual `apply("org.jetbrains.kotlin.android")` from `AndroidApplicationConventionPlugin` and `AndroidLibraryConventionPlugin`, and set `android.builtInKotlin=true` in `gradle.properties` (or omit — it's the default).
  - Version catalog version key referenced via `findVersion(...)` doesn't exist → add it to the catalog
- After fixing, re-run the same gradle command. Cap at **5 iterations per step**. If you still can't make a step pass after 5 tries, stop and tell the user exactly what's failing — don't keep guessing.

### 11.4 — Warning sweep (REQUIRED — do not skip)

A clean compile/test run is not enough — **warnings must be addressed too**. After the four gradle commands pass, scan their output for warnings and fix them:

1. **Kotlin compiler warnings** — re-run `./gradlew :app-mobile:compileDebugKotlin --warning-mode=all --no-daemon` (and `:app-tv:compileDebugKotlin` if TV). Look for lines starting with `w:` (warning) or containing `warning:`. Categories to expect and how to handle:
   - **Deprecation warnings** (`'X' is deprecated`) — switch to the recommended replacement. Don't suppress unless the replacement isn't available yet (then `@Suppress("DEPRECATION")` with a one-line comment explaining why).
   - **Unused parameters / variables** — remove them, or rename to `_` if required by an override.
   - **Unchecked casts** — narrow the type at the source, don't blanket-suppress.
   - **`@OptIn` required** — add the targeted `@OptIn(...)` to the call site, not file-level.
2. **Android Lint warnings** — re-run `./gradlew lint --no-daemon`, then open the per-module HTML reports under each module's `build/reports/lint-results-debug.html`. Triage:
   - **Correctness** (`MissingPermission`, `WrongConstant`, `InvalidPackage`) — fix immediately.
   - **Security** (`HardcodedDebugMode`, `SetJavaScriptEnabled`) — fix immediately.
   - **Accessibility** (`ContentDescription`, `LabelFor`) — fix unless the element is decorative; if decorative, set `contentDescription = null` explicitly.
   - **Performance** (`UnusedResources`, `UseSparseArrays`) — remove the unused, accept the rest unless the user reports a real impact.
   - **Style / I18n / Usability** — fix the easy ones; for the rest, add a lint baseline only if there are too many to fix in scope (and tell the user a baseline was created).
3. **Detekt warnings** — `./gradlew detekt --no-daemon` and inspect `build/reports/detekt/detekt.html` per module. Fix style violations. For categories the user might disagree with (`MagicNumber`, `LongMethod`, `ComplexCondition`), suppress at the function level with a justification comment — don't disable globally.
4. **Spotless** — `./gradlew spotlessApply --no-daemon` to auto-fix formatting; commit the changes.

After fixing, re-run the full verification cycle (Step 11.3). It must pass with **zero new warnings introduced by your fixes**.

If you genuinely cannot fix a warning (e.g. it's in generated code, or it's a library bug not yours), suppress narrowly with a comment:

```kotlin
@Suppress("DEPRECATION") // GraphqlClient.startWatching is deprecated upstream — replacement not in 4.x yet
```

A blanket suppression without a comment is a regression — don't do it.

### 11.5 — Declaring done

You may only tell the user "done" after:

1. The structural self-check (Step 11.2) passes for every feature.
2. All four gradle commands in Step 11.3 have passed at least once.
3. The warning sweep (Step 11.4) has been run and either the project is clean OR every remaining warning has a narrow `@Suppress` with a justification comment.

If you skip any of these, the user lands on a project that compiles but is full of latent issues (deprecated APIs about to break, missing tests, ScreenContents that aren't actually tested). The verification cycle is the contract.

## Step 12 — Initial commit

Once the verification cycle (Step 11) is green and the warning sweep is clean, **initialize git and create the first commit yourself**.

### 12.1 — Check git state

1. Run `git rev-parse --is-inside-work-tree 2>/dev/null` to detect whether the cwd is already a git repository.
   - If **not** a repo: run `git init` and continue.
   - If it **is** a repo: skip `git init`. Check `git status --porcelain` — if there are pre-existing tracked files unrelated to this scaffold, stop and ask the user whether to (a) commit only the scaffold (use `git add` with explicit paths), (b) commit everything together, or (c) abort and let them commit manually. Don't silently sweep their work into one commit.

2. Run `git status --porcelain` and sanity-check what's about to be staged. **Refuse to commit if any of these appear unintentionally**:
   - `local.properties` (should be gitignored — if it shows up, the `.gitignore` is wrong, fix it)
   - `**/build/`, `.gradle/`, `.idea/` (same — the ignore is broken)
   - Anything that looks like a real secret: `*.keystore`, `*.jks`, files with `password`/`secret`/`token` in the name, raw `google-services.json` if the user explicitly said they don't want it committed (ask if unsure)

   If the `.gitignore` is letting something through, fix the `.gitignore` first, then re-run `git status`.

### 12.2 — Stage and commit

```bash
git add .
git commit -m "setup architecture"
```

Use the exact message `setup architecture` (no trailing period, no Claude co-author trailer for this one — this is the architectural baseline, not a Claude-authored feature commit). If the user has commit signing configured (`commit.gpgsign=true`), let it sign normally; don't pass `--no-gpg-sign`.

If the commit fails because of a pre-commit hook (e.g. Spotless), run the hook's fix (`./gradlew spotlessApply`), `git add .` again, and re-commit. **Never use `--no-verify`** — the hook is there for a reason and will run in CI anyway.

If the commit fails for any other reason (e.g. nothing to commit, missing user.email), report the exact error to the user and ask how to proceed.

### 12.3 — Report

Print to the user:

1. **What was committed** — file count, commit SHA (`git rev-parse --short HEAD`), commit message.
2. **The four gradle commands and their results** (✓ help, ✓ deps, ✓ compile, ✓ spotless/detekt/lint/test).
3. **The warning-sweep result** (e.g. "0 warnings remaining" or "3 suppressed with justification, 0 actionable").
4. **Per-feature confirmation** — Screen + ScreenContent + ≥1 preview + ViewModel test + ScreenContent Compose test all present for every feature × form factor.
5. **The two project-local skills** they now have: `<project>-android-planner` and `<project>-android-implementer`.
6. **Next step**: "Open the project in Android Studio and invoke `/<project>-android-planner` to plan your first real feature."

### 12.4 — Do not push

Never run `git push`, never set a remote, never `git push --set-upstream`. The user wires up their remote when they're ready. This skill stops at the local commit.

## Guardrails

- **Never overwrite an existing file without asking.** Bail in Step 0 if cwd looks like an existing project.
- **Commit only the scaffold, only after verification passes.** `git init` + `git add .` + `git commit -m "setup architecture"` runs in Step 12, after the build-success gate is green. Don't commit before verification. Don't push — never run `git push`, never set a remote.
- **Never `--no-verify`.** If a pre-commit hook fails, fix the root cause (usually `spotlessApply`) and re-stage. Bypassing the hook lands a broken state in source control.
- **Never modify files outside cwd.**
- **Never invent helpers in build-logic.** Use exactly the 8 (or 7) plugin classes the conventions skill names. Inline duplication is the right call.
- **Never use `CommonExtension`.** Each plugin configures its own `ApplicationExtension` or `LibraryExtension`.
- **Never declare a `package`** on convention plugin classes — they live in the default package.
- **Don't skip the build-success gate.** Step 11 is not optional. The user has been burned by half-broken scaffolds; the verification cycle is the contract.
- **If a step fails 5 times**, stop, report what's failing, and let the user decide whether to retry or abort. Don't keep churning.
- **Refuse to commit if the working tree contains sensitive or generated files** that the `.gitignore` should have caught (e.g. `local.properties`, `**/build/`, `*.keystore`). Fix the ignore, re-stage, then commit.
