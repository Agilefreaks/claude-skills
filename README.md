# AgileFreaks Skills

Internal Claude skills library. Works across Claude.ai Cowork, Claude Code (terminal), and GitHub Actions.

## Plugin Catalogue

| Plugin | Description | Version | Category |
|--------|-------------|---------|----------|
| [code-review](plugins/code-review/) | Outside-in, risk-driven code review. Covers bug fixes, new features, add-ons, extensions, and refinements. | 1.0.0 | productivity |
| [dep-update-merge](plugins/dep-update-merge/) | Bundles dependency-update PRs/MRs into one verified change with changelog analysis and breaking change triage. | 1.0.0 | productivity |

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

Copy `.github/workflows/code-review.yml` from this repo into your project. It checks out this skills repo at `.skills/` and passes the relevant skill to `anthropics/claude-code-action@v1`.

See the workflow file for required secrets and permissions.

## Adding Plugins

1. Create `plugins/<name>/` with:
   - `.claude-plugin/plugin.json` — plugin manifest
   - `skills/<name>/SKILL.md` — skill content
   - `README.md` — documentation
2. Add an entry to `.claude-plugin/marketplace.json`
3. Update `CHANGELOG.md`
