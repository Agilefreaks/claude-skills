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

## Onboarding metadata

When your skill has extension points, declare them in `.claude-plugin/onboarding.json` alongside `plugin.json`. This enables the onboarding wizard (see `.claude/rules/onboarding.md`) to walk consuming projects through interactive setup and generate their companion rules file.

### onboarding.json structure

```json
{
  "skill": "skill-name",
  "intro": "One or two sentences describing the skill and what setup will do.",
  "extensionPoints": [
    {
      "id": "kebab-case-id",
      "phase": "Phase N: Phase Name",
      "description": "What this extension point controls.",
      "default": "Fallback behavior if not configured, or null if none.",
      "required": false,
      "prompt": "The question the wizard asks the user.",
      "detect": {
        "description": "What to look for in the consuming project.",
        "hints": ["glob/pattern/**", "another-file.yml"]
      }
    }
  ]
}
```

**Field reference:**
- `id` — kebab-case, unique within the plugin. Must match placeholder IDs in `assets/rules-template.md`.
- `phase` — the SKILL.md phase this belongs to, for display grouping.
- `description` — what this extension point controls (one sentence).
- `default` — the skill's fallback behavior as a string, or `null` if the phase is skipped without configuration.
- `required` — set `true` only if the skill cannot function at all without this configured.
- `prompt` — the question asked during interactive setup. Be concrete; give examples.
- `detect` (optional) — project inspection hints. The wizard globs for `hints` patterns and reads matching files to suggest project-aware defaults. Omit `detect` for extension points that are pure preferences (not detectable from project structure).

Keep `onboarding.json` in sync with SKILL.md. When you add, rename, or remove an extension point in the skill, update the corresponding entry here.

### Rules template

If you ship `assets/rules-template.md`, the wizard uses it as the skeleton for the generated companion rules file. Use these placeholder conventions:

- `{{id}}` — replaced with the user's answer for that extension point.
- `{{#id}}...{{/id}}` — the entire block is included only if the user provided a value (not "use default"). Remove the markers; keep the content with the substitution applied.

Sections for extension points where the user accepted the default are omitted from the generated file — the skill's built-in behavior handles them.

If no template is present, the wizard generates a minimal rules file from the extension point metadata directly.

## Versioning

Skills are versioned via `plugin.json` and the marketplace entry. When updating a skill's behavior materially, bump the version. Document the change in `CHANGELOG.md`.
