# Rules: Skill Authoring

Rules for writing `SKILL.md` files in this repository.

## Skills must be generic

A skill file must contain no project-specific names, paths, tools, commands, or conventions. If you find yourself writing something like "open a PR on GitHub" or "run `bundle exec rspec`" or "post the review to #engineering", stop. That content belongs in a companion rules file in the consuming project, not here.

A useful test: could this skill be installed in any AgileFreaks project — or any software project — without modification? If not, extract the project-specific parts.

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

Describe in the skill — typically in the relevant phase — that project-specific behavior is configured via a companion rules file in the consuming project. Don't specify what the rules file is called or where it lives; that's up to the consuming project.

## Trigger description

The `description` frontmatter field is the primary trigger. After the frontmatter block, you may optionally open the skill body with a blockquote summarizing what is handled externally (platform mechanics, project conventions). This is for human readers; triggering relies solely on the frontmatter `description`.

Example blockquote:
```
> Outside-in, risk-driven code review methodology.
>
> Platform integration and project-specific checks are defined separately in your project's rules.
```

## Versioning

Skills are versioned via `plugin.json` and the marketplace entry. When updating a skill's behavior materially, bump the version. Document the change in `CHANGELOG.md`.
