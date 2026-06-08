# android-project-aligner

A Claude Code plugin that **aligns an existing Android project** with the conventions used by [`android-project-starter`](../android-project-starter) — multi-module MVI + Jetpack Compose + Koin + Navigation 3 + a `build-logic` convention plugins setup. Where `android-project-starter` scaffolds a green-field project, this plugin **audits, plans, and migrates** a project that already exists.

The skill runs in three explicit phases — **audit → plan → apply** — with checkpoints between them. The user can stop after the audit (and act on the gap report manually), accept the plan as-is, or hand-edit it before applying. Every applied phase commits on a feature branch; nothing is pushed.

## What you get

After a successful run on your existing project:

```
<your-project-root>/
├── .claude/skills/
│   ├── <project>-android-planner/SKILL.md
│   └── <project>-android-implementer/SKILL.md   (same shape as init-android-project produces)
├── build-logic/convention/                       (added if missing, or normalized if partial)
├── core/{common,data,designsystem-*,model,testing,ui-*}/   (filled in where missing)
├── feature/<x>/{data,ui-mobile,ui-tv?}/          (existing features split + normalized)
├── gradle/libs.versions.toml                     (consolidated, with convention plugin aliases)
├── .lint/, .detekt/, .github/workflows/          (added if missing)
└── git branch: align/architecture-conventions    (all migration commits land here, not main)
```

The migration **never overwrites your existing code blindly**. For each gap it finds, the planner:

1. Reads the existing file/module/structure
2. Cross-references the conventions skill for the canonical shape
3. Produces a per-phase diff plan you can review
4. Applies the phase and verifies the project still compiles before moving on

## Prerequisites

- The `android-project-starter` plugin must be installed in the same Claude Code instance. The aligner reuses its `conventions` skill as the single source of truth for "what good looks like." If you don't have it:

  ```
  /plugin install android-project-starter@agilefreaks-skills --scope user
  ```

- Your project must be a git repository with a clean working tree (or one the aligner can stash). The skill will refuse to start with uncommitted unrelated changes.
- A working Gradle wrapper (`./gradlew`) — the aligner verifies after every phase by running gradle.

## Install

```
/plugin marketplace list             # agilefreaks-skills should already appear
/plugin install android-project-aligner@agilefreaks-skills --scope user
/reload-plugins
```

## Usage

```
cd /path/to/existing-android-project
claude
> /android-project-aligner:align-android-project
```

The skill runs end-to-end but pauses at every phase boundary so you can review or skip.

### Phase 1 — Audit

Inventories the project against the conventions:

- Top-level layout (`app*`, `core/*`, `feature/*/*`, `build-logic/`)
- AGP / Kotlin / Compose Compiler / KSP version alignment
- Convention plugin presence (9 expected, or 8 without Room)
- Version catalog shape (`gradle/libs.versions.toml`)
- MVI base types (`BaseViewModel`, `ViewAction`/`ViewState`/`ViewSideEffect`)
- Screen vs. ScreenContent split per feature
- Repository encapsulation (public interface, `internal` impl, never depended-on from app)
- DI library — Koin shape expected; Hilt or other DI is flagged for migration decision
- Navigation library — Navigation 3 expected; Compose Navigation / nav-compose v2 / custom is flagged
- Persistence (Room/DataStore)
- Network (Retrofit/Apollo/Ktor/none)
- Theming — `<Project>Theme`, `@<Project>Previews`, hardcoded colors/text styles
- Quality gates (Spotless, detekt, Android Lint, CI)
- Product flavors (`qa` default + `prod`)
- Dev-tools (shake/broadcast env switcher)

Outputs a written **gap report**: one line per finding, with severity (blocker / important / nice-to-have), effort (S/M/L), and risk (low / medium / high).

### Phase 2 — Plan

The planner converts the gap report into an **ordered migration plan**. Phases run in dependency order so each one stays buildable:

1. **Foundation** — version catalog consolidation, AGP/Kotlin alignment, `build-logic/convention/` scaffold + 9 convention plugins
2. **Core modules** — `core/common` (MVI base), `core/model`, `core/testing`, `core/designsystem-*`, `core/ui-*`, `core/data` (env config + dev-tools if `qa`/`prod` flavors are being added)
3. **Feature module split** — for each feature, split `feature/<x>/` into `feature/<x>/{data, ui-mobile, ui-tv?}`; normalize Screen/ScreenContent
4. **App wiring** — `app-mobile` (and `app-tv` if applicable) — apply convention plugins, restructure DI, ensure no `:feature:<x>:data` dependency leaks
5. **Quality + CI** — Spotless, detekt, lint config, GitHub Actions, Dependabot
6. **Operational** — `qa`/`prod` flavors, env config + dev-tools, theming normalization
7. **Project-local skills** — generate / update `.claude/skills/<project>-android-planner` and `<project>-android-implementer`

You'll see the plan as a numbered list with file-level changes per phase. You can:

- Accept all phases
- Deselect specific phases (e.g. skip Nav3 migration if your team isn't ready)
- Edit phase ordering (rarely needed)
- Abort

### Phase 3 — Apply

The aligner:

1. Creates a branch (default `align/architecture-conventions`, or a name you supply).
2. Captures a baseline `./gradlew help` to confirm the project compiles before any changes.
3. Applies each approved phase in order. After each phase:
   - Re-runs the phase's verification command (typically `./gradlew :app-mobile:compileQaDebugKotlin` after a module change; `./gradlew help` after a build-logic change).
   - If green, commits with message `align: <phase name>`.
   - If red, retries up to 5 times, reading the gradle error and fixing the file. After 5 failed retries it halts and asks you what to do.
4. Final verification: full quality gate (`spotlessApply`, `spotlessCheck`, `detekt`, `lint`, `test`, `compileQaDebugKotlin`, `compileProdDebugKotlin`) + warning sweep.
5. Reports the branch name, phases applied, gradle results, and the two project-local skills it wrote.

**The aligner never pushes.** You decide when and where the migration branch lands.

## What the plugin DOES NOT do

- **It does not pick your DI / nav / network library for you.** If your project uses Hilt and the conventions say Koin, the audit flags it and asks you. The aligner will either (a) leave the choice intact and only apply stack-agnostic conventions, or (b) migrate the stack — whichever you pick.
- **It does not rename your applicationId, package, or project name.** The audit captures what your project already declares; the plan and code generated reuse those identifiers.
- **It does not push, set remotes, or merge.** The branch is local. You ship when you're ready.
- **It does not bump dependency versions** — that's `dep-update-merge`'s job. The aligner only sets the convention-plugin aliases and ensures AGP/Kotlin/Compose/KSP are *internally consistent* with each other.
- **It does not delete code without confirmation.** If a planned migration would remove a file (e.g. an old custom MVI base class that the conventions replace), the plan calls it out explicitly and asks before doing it.

## When to use this vs. `init-android-project`

| Situation | Use |
|---|---|
| Brand-new project, empty directory | `init-android-project` |
| Existing project with no architecture (or a different one) | `align-android-project` |
| Existing project that already follows the conventions but needs the project-local planner/implementer skills | `align-android-project` (audit finds zero gaps; only the final project-local-skills phase applies) |
| Existing project where you only want to bump versions | `dep-update-merge` |

## Iterating on this plugin

The plugin source lives at `~/.claude/plugins/marketplaces/agilefreaks-skills/plugins/android-project-aligner/`. Edit any `SKILL.md` under `skills/` and run `/reload-plugins` to pick up the changes — no reinstall needed.

The aligner's logic is intentionally thin — most of the "what shape should this file have" knowledge lives in `android-project-starter:conventions` and its reference files. If you change a convention there, the aligner picks it up automatically on the next run.

## Skills shipped by this plugin

| Skill | Invocation | Purpose |
|---|---|---|
| `align-android-project` | `/android-project-aligner:align-android-project` | The audit → plan → apply workflow described above. |
