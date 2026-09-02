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
- `name` — kebab-case, max 64 characters. Lowercase letters, digits and hyphens only; no XML tags; must not contain the reserved words "anthropic" or "claude".
- `description` — max 1024 characters. This is the **primary triggering mechanism**. Describe what the skill does AND the contexts or phrases that should activate it. Write it in the third person ("Bundles open dependency PRs…", never "I can help you…"), and put the key use case first: the listing entry is truncated at 1,536 characters, and Claude Code shortens descriptions further when many skills are installed. Lean slightly "pushy" to ensure the skill triggers when useful.

### Optional fields

Plugins in this marketplace are installed in Claude Code *and* Claude.ai Cowork, so it matters which half of the surface a field belongs to.

**Agent Skills spec — portable everywhere:**
`license`, `metadata`, `compatibility`, `allowed-tools`.

**Claude Code only — accepted and ignored elsewhere:**
`disable-model-invocation`, `user-invocable`, `disallowed-tools`, `argument-hint`, `arguments`, `model`, `effort`, `context`, `agent`, `background`, `hooks`, `paths`, `shell`.

The two invocation switches are the ones worth reaching for:

- `disable-model-invocation: true` — only a human may invoke it. Use it for a skill with side effects the model should not decide to trigger: scaffolding a project, deploying, sending a message.
- `user-invocable: false` — only Claude may invoke it. Use it for background knowledge that is not a meaningful action for a user to take, such as a conventions or reference skill.

**Do not set `model` or `effort` in a skill published here.** Both override the consuming session, their valid values vary by model, and a shared library should not take that decision away from the project installing it. A consumer can always set them in their own copy.

Verify the frontmatter parses with `claude plugin validate <plugin-dir>` before committing.

## Skill anatomy

```
skill-name/
├── SKILL.md          # Required — under 500 lines; see Size and placement
├── scripts/          # Executable code for deterministic tasks
├── references/       # Docs loaded into context as needed
└── assets/           # Templates, icons, fonts used in output
```

Move detailed reference content to `references/`. When a skill supports multiple domains, organize by variant under `references/`.

Two rules keep that content actually readable, both for the same reason — Claude frequently previews a long or indirectly-reached file with a partial read, and cannot tell what it missed:

- **Give any reference over 100 lines a table of contents**, so a partial read still shows the full scope of what is in the file.
- **Every reference must be reachable in one hop from a SKILL.md that names it.** Before adding one, check: does its own SKILL.md name this file? A reference *citing* another file is fine, and sometimes necessary — the Android plugins point at `android-project-starter:conventions` for canonical file shapes instead of copying them, because a second copy would drift. What the rule forbids is content whose only route is a chain.

### Size and placement

Keep SKILL.md under 500 lines. Aim for under roughly 5,000 tokens (about 20,000 characters) as well, and treat that number as a placement constraint rather than a size limit: what matters is that everything past it is safe to lose.

Two platform behaviours are the reason:

- **Claude Code does not re-read SKILL.md on later turns.** The rendered content enters the conversation once, when the skill is invoked, and stays as it was. A rule that must hold for the whole task has to be written as a standing instruction, not as a step buried in phase four.
- **Auto-compaction keeps only the first 5,000 tokens of each re-attached skill**, with a 25,000-token budget shared across every skill invoked in the session. Anything past that point silently stops applying partway through a long run.

So put the non-negotiables — guardrails, prohibitions, core principles, how to communicate while working — in a standing block near the top, before the first numbered step. Ordering by narrative flow and leaving the guardrails as a closing section puts them exactly where they get dropped.

Past the first 5,000 tokens, put only content the skill can afford to lose halfway through a long run: setup wizards, per-phase detail that will already have been read by the time compaction hits, and appendices. A skill that needs more than that in force at all times is a skill whose reference material belongs in `references/`.

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

When your skill has extension points, include a **Setup** section in SKILL.md that describes how to interactively configure the skill. It travels with the plugin and is available wherever the skill is installed — no external files needed.

**Put it after the phases, not before them**, once a skill approaches the size boundary above. Setup runs on its own invocation ("set up <skill>"), where the file has just been loaded and position costs nothing; during an actual run it is dead weight in the one part of the file that survives compaction. This is the case the Size and placement section has in mind when it names setup wizards as content to keep past the first 5,000 tokens. In a small skill, either position is fine.

The Setup section should:

1. **Read existing project rules first.** Instruct Claude to read `.claude/rules/` and `CLAUDE.md` before asking anything. Do not duplicate configuration that already exists in the project (build commands, test commands, coding conventions, etc.).
2. **Inspect the project.** List what to look for: CI configs, linter configs, dependency bot configs, package manifests — whatever is relevant to the skill's extension points.
3. **Present interactive choices.** List each skill-specific decision with a brief description. Use choice dialogs (not walls of text) so the user can select from detected options.
4. **Write the companion rules file.** Generate `.claude/rules/<skill-name>.md` containing only the user's choices. Omit decisions where the user accepts the default.

### Only configure skill-specific decisions

Extension points should be decisions unique to the skill's workflow. Project-level concerns like build commands, test commands, and lint commands belong in the project's own `CLAUDE.md` or `.claude/rules/` — not in a skill's companion rules file. If a skill needs to run the project's build, it should say "use whatever the project has configured" rather than asking the user to re-enter those commands.

### Add setup triggers to the description

Include "set up", "configure", and "onboard" in the frontmatter `description` so the skill activates when users ask to configure it.

## Writing for the current model generation

A skill is a per-model artifact. A line that is load-bearing on one generation becomes dead weight — sometimes actively harmful — on the next. Re-read this section whenever a new Claude model ships, and re-audit the skills against the model's prompting guide.

### No model names in a skill

Never name a model, and never encode a workaround for one model's quirk. Pinned names degrade silently at the next release, and the current generation splits both ways on the behaviours worth tuning: one model narrates too much where another goes quiet, one over-delegates where another under-delegates. Write the behaviour you want in model-neutral terms and it stays correct across both.

The exception is a real configuration value a consumer will act on, such as a `--model` flag in a generated CI workflow. Prefer an alias (`opus`) over a dated identifier so it tracks the current release.

### Say how to report back

Skills in this repo run long, multi-phase sessions and end in something a human reads. Current models diverge sharply on how much they say while working and how long they write, so the skill has to state it. Include this in the phase that produces the skill's output:

> Lead with the outcome: your first sentence answers what happened or what you found. Supporting detail comes after. Write it for someone who did not watch you work — spell out identifiers and drop the shorthand you built up along the way. Match the length of anything you write to a file to what the task needs; do not pad it with filler sections or redundant summaries.

For a skill that runs a long tool-calling loop, also put this in the standing block at the top:

> Say in one line what you are about to do before your first tool call, and give a brief update when you find something load-bearing or change direction. Before reporting progress, tie each claim to a command you actually ran: if a check fails, say so with its output; if a step was skipped, say that.

A skill that produces files rather than a report, or that is pure reference with no run loop, needs neither. Adding them there is noise.

A skill may also drop any sentence it has encoded structurally instead. `code-review` keeps only "write it for a reviewer who did not watch you work"; its output shape puts the verdict first and admits sections conditionally, so the outcome-first and match-the-length sentences would restate what the format already enforces. Prefer the structure — an instruction competing with a template loses. Note the substitution where you make it, so a later audit reads it as a decision rather than a gap.

### Don't add self-check scaffolding

Current models verify their own work without being told, and instructions like "double-check your answer" or "use a subagent to verify" now cause redundant work rather than better output. This does **not** apply to a genuine verification *phase* — running the test suite, driving the app against acceptance criteria, checking a build is green. That is methodology, and it stays.

### Say what to leave alone

Current models expand scope: fixing nearby code, adding tests the task did not ask for, rewriting a whole file where a targeted edit would do. Any skill with an implement-or-apply phase should bound it:

> If you find a pre-existing bug, a performance concern, or behaviour the task does not mention, do not fix or extend it here unless the requested change cannot work without it — report it as a follow-up. Prefer a targeted edit to rewriting a whole file when the result is the same. Commit tests only where the task asks for them or the repository already keeps tests for this kind of change; scratch checks do not become permanent test files.

### Don't narrow the search to widen the signal

If a skill produces findings, have it report everything it finds with a severity and a confidence, and filter in the summary. Telling a model to "only report high-severity issues" or to "be conservative" is followed literally and suppresses real findings at discovery time, where they cannot be recovered.

## Versioning

Skills are versioned via `plugin.json` and the marketplace entry. When updating a skill's behavior materially, bump the version. Document the change in `CHANGELOG.md`.
