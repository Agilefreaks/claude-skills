# Claude Code — AgileFreaks Skills Repo

## What this repo is

A plugin marketplace for Claude Code and Claude.ai Cowork. It contains skills distilled from real AgileFreaks project experience, packaged as installable plugins.

**No build system. No test runner. No package dependencies.** All content is Markdown and JSON.

## Plugin structure

```
plugins/<name>/
  .claude-plugin/plugin.json   # plugin manifest
  skills/<name>/SKILL.md       # skill content
  README.md                    # plugin documentation
```

The marketplace manifest lives at `.claude-plugin/marketplace.json`.

## Adding a plugin

1. Create `plugins/<name>/` with the three files above
2. Add an entry to `.claude-plugin/marketplace.json`
3. Add a row to the Plugin Catalogue table in `README.md`
4. Add an entry under `[Unreleased]` in `CHANGELOG.md`

See `.claude/rules/marketplace.md` for field requirements and conventions.

## Writing SKILL.md files

Skills must be generic and reusable across any project. No project-specific names, paths, tools, or commands belong in a skill file. Project-specific configuration lives in companion rules files in the consuming project.

See `.claude/rules/skill-authoring.md` for the full authoring rules.

## Versioning

- Plugin versions are semver, tracked in both `plugin.json` and the plugin's entry in `marketplace.json`
- `CHANGELOG.md` follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) — new entries under `[Unreleased]`, promoted to a dated version on release

## Guiding principles

Before contributing, read `.claude/soul.md`. It captures why this repo exists and the principles that should inform every change.

## Rules files

Detailed rules are in `.claude/rules/`:
- `skill-authoring.md` — how to write SKILL.md files
- `marketplace.md` — how to maintain marketplace structure and versioning
