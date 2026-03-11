# Rules: Marketplace Structure

Rules for maintaining the plugin marketplace in this repository.

## marketplace.json

Every plugin entry in `.claude-plugin/marketplace.json` must include:

- `name` — matches the plugin's directory name under `plugins/`
- `source` — relative path from repo root: `./plugins/<name>`
- `description` — one sentence describing what the skill does
- `version` — semver, matches `plugin.json`
- `category` — single category string (e.g. `productivity`)
- `keywords` — array of lowercase strings for discoverability

The top-level `metadata.version` tracks the marketplace itself, separate from individual plugin versions.

## plugin.json

`plugins/<name>/.claude-plugin/plugin.json` is the plugin manifest. Keep it minimal:

- `name` — plugin name
- `description` — one or two sentences; mention what is handled externally
- `version` — semver
- `author` — object with `name`
- `keywords` — array

Do not add fields that duplicate marketplace.json or that aren't used by the plugin system.

## Source paths

All `source` values in marketplace.json must be relative paths starting with `./plugins/`. Never use absolute paths or repository URLs.

## Version bumps

When updating a plugin:
1. Bump the version in `plugins/<name>/.claude-plugin/plugin.json`
2. Bump the matching version in the plugin's entry in `.claude-plugin/marketplace.json`

Both must always be in sync. The marketplace `metadata.version` is bumped separately when the marketplace itself changes (new plugins, structural changes).

## CHANGELOG.md

Follow [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) conventions:

- New or changed entries go under `[Unreleased]` at the top
- On release, rename `[Unreleased]` to the version with a date: `[1.1.0] - 2026-03-11`
- Use `### Added`, `### Changed`, `### Fixed`, `### Removed` subsections as appropriate
- One line per meaningful change; link to the relevant plugin or file where helpful

## Adding a new plugin

1. Create `plugins/<name>/` with:
   - `.claude-plugin/plugin.json`
   - `skills/<name>/SKILL.md`
   - `README.md`
2. Add an entry to `.claude-plugin/marketplace.json`
3. Add a row to the Plugin Catalogue table in `README.md`
4. Add an entry under `[Unreleased]` in `CHANGELOG.md`
