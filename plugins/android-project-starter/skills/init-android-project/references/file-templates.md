# File templates — verbatim bodies the wizard generates

This reference holds the verbatim file bodies the wizard writes during Step 10. Each entry pairs with the bullet in `SKILL.md` that references it.

Substitute `<Project>` / `<Project>Previews` / `<applicationId>` / `<root-pkg>` / `<project>` with the values gathered in Step 1. Do not trim — these bodies are sized for real projects.

## `.editorconfig` (Step 10.1)

Compose-aware ktlint defaults so the first lint pass after generation doesn't trip on Composable function naming or the MVI `_actions` backing-property pattern. Substitute `<Project>Previews` with the project's actual multi-preview annotation name (e.g. `AcmePreviews`):

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

## `.gitignore` (Step 10.1)

Comprehensive Android+Kotlin+Gradle ignore. Use this exact body — don't trim:

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

## `.detekt/config.yml` (Step 10.7)

Compose-aware overrides so the first detekt pass doesn't trip on Preview composables and scaffold-injected fields. Substitute `<Project>Previews` with the actual project preview annotation:

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

## `google-services.json` placeholder (Step 10.8)

A bare `{ "project_id": "X" }` is **not enough** — the google-services plugin's schema check rejects it during `processDebugGoogleServices` with errors like `No matching client found for package name`.

Every scaffold has two product flavors with different application IDs (`<applicationId>` for `qa`, `<applicationId>.prod` for `prod`), so the placeholder needs **two `client` entries** — one per variant. The google-services plugin matches `package_name` against the resolved applicationId per variant; missing one fails that variant.

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
        "android_client_info": { "package_name": "<applicationId>" }
      },
      "oauth_client": [],
      "api_key": [ { "current_key": "AIzaSyPLACEHOLDER_REPLACE_BEFORE_SHIPPING_PLACEHO" } ],
      "services": { "appinvite_service": { "other_platform_oauth_client": [] } }
    },
    {
      "client_info": {
        "mobilesdk_app_id": "1:000000000000:android:0000000000000001",
        "android_client_info": { "package_name": "<applicationId>.prod" }
      },
      "oauth_client": [],
      "api_key": [ { "current_key": "AIzaSyPLACEHOLDER_REPLACE_BEFORE_SHIPPING_PLACEHO" } ],
      "services": { "appinvite_service": { "other_platform_oauth_client": [] } }
    }
  ],
  "configuration_version": "1"
}
```

**After writing**, print this warning to the user:

```
⚠️  The google-services.json files are PLACEHOLDERS. Firebase/Crashlytics will compile
    but not report at runtime. Replace app-mobile/google-services.json (and app-tv if TV)
    with the real file from the Firebase Console before shipping.
```

## Implementer SKILL.md — Compose authoring pointer block (Step 10.9)

Use this exact shape inside the generated implementer's `SKILL.md`, substituting project-specific names:

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

## Generated project's README — "Build variants" section (Step 10.10)

```markdown
## Build variants

Two product flavors × two build types = four variants. `qa` is the default flavor, so
`./gradlew assembleDebug` resolves to `assembleQaDebug`.

| Variant         | Application ID            | Notes                                  |
| --------------- | ------------------------- | -------------------------------------- |
| `qaDebug`       | `<applicationId>`         | Default. Dev dialog enabled.           |
| `qaRelease`     | `<applicationId>`         | Signed qa for distribution.            |
| `prodDebug`     | `<applicationId>.prod`    | Debuggable prod build for triage.      |
| `prodRelease`   | `<applicationId>.prod`    | Shipping build. Dev dialog disabled.   |

Install commands:

    ./gradlew installQaDebug          # default — shake/broadcast dev dialog enabled
    ./gradlew installProdDebug        # prod-flavored debug build, dev dialog off
    ./gradlew assembleProdRelease     # signed prod release
```

## Generated project's README — "Development tools" section (Step 10.10)

```markdown
## Development tools (qa builds only)

QA builds (`IS_QA = true`) include a runtime dialog for switching the API base URL between
staging, prod, and an arbitrary custom URL. Prod builds compile the dialog out — the
`DevToolsHost` becomes a zero-cost passthrough.

### Open the dialog

**On a phone (qa build only):** shake the device.

**From CLI (phone or TV, qa build only):**

    adb shell am broadcast -a <applicationId>.OPEN_DEV_TOOLS -p <applicationId>

The `-p <applicationId>` scopes the broadcast to this app so it doesn't leak to other apps
on the device.

**On the Android emulator (mobile):** the broadcast command above is the reliable path —
the emulator's virtual-sensor sliders for accelerometer are finicky and the gesture rarely
clears the shake threshold. If you do want to use the sensors UI, click the `...` button on
the emulator side panel → **Virtual sensors** → **Accelerometer** tab → snap the X axis to
a high value, then back. Repeat 2–3× to clear the 1-second debounce.

There is no keyboard shortcut for shake. `Ctrl/Cmd+M` opens the menu, not the shake.

### TV

TV remotes can't shake, so the broadcast command is the only trigger. The TV `DevToolsHost`
mounts only the broadcast receiver — running the `adb shell am broadcast` command above on
a connected (or networked) TV device opens the same dialog.
```

Substitute `<applicationId>` with the user's actual application id.
