# Rules: Skill Authoring

Rules for writing `SKILL.md` files in this repository.

## Skills must be generic

A skill file must contain no project-specific names, paths, tools, commands, or conventions. If you find yourself writing something like "open a PR on GitHub" or "run `bundle exec rspec`" or "post the review to #engineering", stop. That content belongs in a companion rules file in the consuming project, not here.

A useful test: could this skill be installed in any AgileFreaks project — or any software project — without modification? If not, extract the project-specific parts.

## Structure

Use clear phase or section headings. Skills that describe a multi-step process should use numbered phases. Skills that describe a reference or checklist can use descriptive headings.

Each phase or section should have a focused purpose. Don't combine concerns in a single section.

## "What to defer to a human"

Any phase where the skill cannot fully verify an outcome must include a "What to defer to a human" note. Be explicit about what you're not checking. Do not imply coverage you can't provide.

## Project-specific configuration

Describe in the skill — typically in the relevant phase — that project-specific behavior is configured via a companion rules file in the consuming project. Don't specify what the rules file is called or where it lives; that's up to the consuming project.

## Trigger description

Open the skill with a blockquote (or frontmatter if supported) that describes:
- What the skill does in one or two sentences
- What is handled externally (platform mechanics, project conventions)

Example:
```
> Outside-in, risk-driven code review methodology.
>
> Platform integration and project-specific checks are defined separately in your project's rules.
```

## Versioning

Skills are versioned via `plugin.json` and the marketplace entry. When updating a skill's behavior materially, bump the version. Document the change in `CHANGELOG.md`.
