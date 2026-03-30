# Rules: Skill Authoring

Rules for writing `SKILL.md` files in this repository.

## Skills must be generic

A skill file must contain no project-specific names, paths, tools, commands, or conventions. If you find yourself writing something like "open a PR on GitHub" or "run `bundle exec rspec`" or "post the review to #engineering", stop. That content belongs in a companion rules file in the consuming project, not here.

A useful test: could this skill be installed in any AgileFreaks project — or any software project — without modification? If not, extract the project-specific parts.

The mechanism for this separation is the Template Method pattern — described in [Project-specific configuration](#project-specific-configuration) below.

## Frontmatter

Every SKILL.md **must** begin with a YAML frontmatter block:

```yaml
---
name: skill-name
description: "What the skill does and when to trigger it."
---
```

Required fields:
- `name` — kebab-case, max 64 characters
- `description` — max 1024 characters. This is the **primary triggering mechanism**. Describe what the skill does AND the contexts or phrases that should activate it. Lean slightly "pushy" to ensure the skill triggers when useful.

Optional fields: `license`, `allowed-tools`, `metadata`, `compatibility`. Do not add unrecognized keys — validation rejects them.

## Skill anatomy

```
skill-name/
├── SKILL.md          # Required — keep under 500 lines
├── scripts/          # Executable code for deterministic tasks
├── references/       # Docs loaded into context as needed
└── assets/           # Templates, icons, fonts used in output
```

Keep SKILL.md under 500 lines. Move detailed reference content to `references/`. For large reference files (>300 lines), include a table of contents. When a skill supports multiple domains, organize by variant under `references/`.

## Structure

Use clear phase or section headings. Skills that describe a multi-step process should use numbered phases. Skills that describe a reference or checklist can use descriptive headings.

Each phase or section should have a focused purpose. Don't combine concerns in a single section.

## "What to defer to a human"

Any phase where the skill cannot fully verify an outcome must include a "What to defer to a human" note. Be explicit about what you're not checking. Do not imply coverage you can't provide.

## Project-specific configuration

Skills follow the Template Method pattern: the skill defines the methodology skeleton with explicit extension points that projects can override through companion rules files.

### Extension points must be explicit

Each phase or section where behavior varies by project should include a clear callout identifying what the project can configure. Name the decision the project needs to make, but don't prescribe the shape of the project's rules file — that's up to the consuming project.

Good example:

> If a project output format is defined, follow it. Otherwise, structure the output clearly with the sections above.

### Provide defaults where possible

Every extension point should supply a reasonable fallback so the skill works with zero project configuration. A skill that requires setup before it produces any value has a high adoption barrier. Aim for: useful out of the box, richer when configured.

### When no default is possible, say so

Some extension points genuinely need project configuration to be effective. Be explicit:

> This phase requires project configuration to be effective. Define your project's conventions in a companion rules file.

Don't silently skip the phase or produce empty output — tell the user what's missing so they can act on it.

### What to name, what not to prescribe

Extension points should name the decision ("output format", "platform posting mechanics", "coding conventions") without dictating file names, directory structures, or rule formats. The consuming project decides how to organize its configuration.

## Trigger description

The `description` frontmatter field is the primary trigger. After the frontmatter block, you may optionally open the skill body with a blockquote summarizing what is handled externally (platform mechanics, project conventions). This is for human readers; triggering relies solely on the frontmatter `description`.

Example blockquote:
```
> Outside-in, risk-driven code review methodology.
>
> Platform integration and project-specific checks are defined separately in your project's rules.
```

## Setup section

When your skill has extension points, include a **Setup** section in SKILL.md (before Phase 1) that describes how to interactively configure the skill. This section travels with the plugin and is available wherever the skill is installed — no external files needed.

The Setup section should:

1. **Read existing project rules first.** Instruct Claude to read `.claude/rules/` and `CLAUDE.md` before asking anything. Do not duplicate configuration that already exists in the project (build commands, test commands, coding conventions, etc.).
2. **Inspect the project.** List what to look for: CI configs, linter configs, dependency bot configs, package manifests — whatever is relevant to the skill's extension points.
3. **Present interactive choices.** List each skill-specific decision with a brief description. Use choice dialogs (not walls of text) so the user can select from detected options.
4. **Write the companion rules file.** Generate `.claude/rules/<skill-name>.md` containing only the user's choices. Omit decisions where the user accepts the default.

### Only configure skill-specific decisions

Extension points should be decisions unique to the skill's workflow. Project-level concerns like build commands, test commands, and lint commands belong in the project's own `CLAUDE.md` or `.claude/rules/` — not in a skill's companion rules file. If a skill needs to run the project's build, it should say "use whatever the project has configured" rather than asking the user to re-enter those commands.

### Add setup triggers to the description

Include "set up", "configure", and "onboard" in the frontmatter `description` so the skill activates when users ask to configure it.

## Versioning

Skills are versioned via `plugin.json` and the marketplace entry. When updating a skill's behavior materially, bump the version. Document the change in `CHANGELOG.md`.
