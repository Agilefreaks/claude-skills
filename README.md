# AgileFreaks Skills

Internal Claude skills library. Works across Claude.ai Cowork, Claude Code (terminal), and GitHub Actions.

## Plugin Catalogue

| Plugin | Description | Version | Category |
|--------|-------------|---------|----------|
| [android-project-starter](plugins/android-project-starter/) | Wizard-driven Android project scaffolder. Multi-module MVI + Compose + Koin + Navigation 3, with build-logic convention plugins (incl. qa/prod product flavors), shake/broadcast dev-tools dialog, tests, CI, and project-local planner/implementer skills. | 0.3.0 | productivity |
| [android-project-aligner](plugins/android-project-aligner/) | Brownfield companion to android-project-starter. Audits an existing Android project against the starter's conventions, produces a phased migration plan, and applies it on a fresh git branch with build verification between phases. Also generates the project-local planner/implementer skills. | 0.1.0 | productivity |
| [code-review](plugins/code-review/) | Outside-in, risk-driven code review. Covers bug fixes, new features, add-ons, extensions, and refinements. | 1.1.0 | productivity |
| [dep-update-merge](plugins/dep-update-merge/) | Bundles dependency-update PRs/MRs into one verified change with changelog analysis and breaking change triage. | 1.1.0 | productivity |
| [feature-development](plugins/feature-development/) | End-to-end feature development: frame → explore → plan → implement (TDD) → verify → hand off. Works with plan mode. | 0.2.0 | productivity |
| [web-frontend-tooling](plugins/web-frontend-tooling/) | Blueprint for wiring up linting, formatting, type-checking, Git hooks, and dead-code detection in a Node/TypeScript project. Prefers the Oxc toolchain (oxlint + oxfmt) with husky + lint-staged; adapts to existing ESLint/Prettier. | 0.1.0 | productivity |

## Usage

### Claude.ai Cowork

In your organization's Cowork settings, add this repository under **Org Settings > Plugins > Add plugin source**. Once synced, plugins can be made available or auto-installed for org members.

### Claude Code (terminal)

```bash
# Add the marketplace
/plugin marketplace add agilefreaks/claude-skills

# Install a plugin
/plugin install code-review@agilefreaks-skills
```

Or add to your project's `.claude/settings.json` to install automatically:

```json
{
  "enabledPlugins": {
    "code-review@agilefreaks-skills": true
  }
}
```

### GitHub Actions

Run `set up code-review` in Claude Code or Claude.ai Cowork. The Setup wizard detects GitHub Actions usage, asks which model to use (Opus recommended), and generates `.github/workflows/code-review.yml` in your project.

After the file is generated, add `CLAUDE_CODE_OAUTH_TOKEN` as a repository secret (generate it with `claude setup-token`, then add it under **GitHub repo → Settings → Secrets and variables → Actions**).

## Adding Plugins

1. Create `plugins/<name>/` with:
   - `.claude-plugin/plugin.json` — plugin manifest
   - `skills/<name>/SKILL.md` — skill content
   - `README.md` — documentation
2. Add an entry to `.claude-plugin/marketplace.json`
3. Update `CHANGELOG.md`
