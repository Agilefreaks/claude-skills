# Rules: Plugin Onboarding Wizard

When a user asks to configure, set up, or create a rules file for an installed plugin, follow this wizard. Trigger phrases include: "set up [plugin]", "configure [plugin]", "create rules file for [plugin]", "onboard [plugin]", "customize [plugin]".

---

## Phase 1: Discovery

Find the plugin's `onboarding.json` file. Look in `.claude-plugin/onboarding.json` relative to the plugin's source directory. If the plugin was installed from the AgileFreaks marketplace, it will be under `plugins/<name>/.claude-plugin/onboarding.json`.

If no `onboarding.json` exists for the plugin, tell the user: "This plugin doesn't include onboarding metadata. Refer to the plugin's README for companion rules file guidance."

Read the `onboarding.json` fully before proceeding.

---

## Phase 2: Project Inspection

Before asking any questions, inspect the consuming project to build project-aware suggestions.

For each extension point that has a `detect` field:

1. Glob for each pattern in `detect.hints` within the current working directory.
2. If matches are found, read the relevant files to understand the project's setup (e.g., read a CI workflow to see what test commands are used, read a linter config to understand style rules).
3. Formulate a specific, concrete suggestion based on what you found — not a vague one.

**Important:** Treat detected values as suggestions, not conclusions. You found evidence; the user decides.

Examples of good suggestions:
- "I found `.github/workflows/ci.yml` — it runs `npm test` and deploys on merge. I'd suggest configuring build verification to run `gh pr checks <PR_NUMBER>` to confirm all checks pass."
- "I found `.rubocop.yml` configured for Ruby 3.2 with a 120-char line limit. I'd suggest adding 'RuboCop compliance (120-char limit, Ruby 3.2 idioms)' to the coding conventions."
- "I found `.github/PULL_REQUEST_TEMPLATE.md` with a 'Fixes #' field. I'd suggest configuring context gathering to extract issue numbers from that pattern in the PR body."

---

## Phase 3: Introduction

Show the user:

1. The plugin's `intro` text from `onboarding.json`.
2. A summary table of all extension points, grouped by phase:

```
Phase 1: Problem Validation
  • task-location — How to locate the relevant issue or ticket
    Default: Use the PR title and description as the source of truth
    Detected: [suggestion or "—"]

Phase 2: Build & Runability
  • build-verification — How to verify the build is passing
    Default: Check for any available build indicators
    Detected: [suggestion or "—"]

...
```

Then ask: **"How would you like to proceed?"**
- **Configure all** — walk through each extension point in order
- **Select specific ones** — let the user name which ones they want to configure
- **Accept all defaults / detections** — generate the rules file using detected values where found, defaults elsewhere

If the user chooses "accept all defaults / detections" and there are no detected values, confirm that no rules file is needed (the skill works out of the box) and stop.

If there are detected values, proceed directly to Phase 5 with those values pre-filled, skipping the interactive prompts.

---

## Phase 4: Interactive Configuration

For each extension point being configured (all or selected subset), present in order:

```
--- [Phase Name] ---
[description]

Default: [default value, or "None — this capability will be inactive without configuration"]
[If detected: "Detected: [your suggestion]"]

[prompt question]
```

Handle responses:
- **User provides an answer** → record it as the configured value for this extension point.
- **User presses enter / says "use default"** and a default exists → mark as "use default"; this section will be omitted from the generated rules file.
- **User presses enter / says "use default"** and `default` is `null` → warn: "Without configuration, this capability will be inactive. The skill will still work, but this phase will be skipped." Then ask if they want to skip or provide a value.
- **User says "use detected"** → use your detected suggestion as the configured value.

After all extension points are handled, proceed to Phase 5.

---

## Phase 5: Generation

Build the companion rules file:

1. Read `assets/rules-template.md` from the plugin's directory (e.g., `plugins/code-review/assets/rules-template.md`).
2. For each extension point with a configured value (not "use default"):
   - Replace `{{id}}` with the configured value.
   - Keep `{{#id}}...{{/id}}` block (remove the markers, keep the content with the substituted value).
3. For each extension point marked "use default":
   - Remove the entire `{{#id}}...{{/id}}` block, including markers and content.
4. If no template exists, generate the rules file directly: use section headings derived from the extension point descriptions, with the configured values as content. Omit sections using defaults.

The final file should contain only sections where the user provided configuration. Sections relying on defaults are intentionally absent — the skill's built-in behavior handles them.

---

## Phase 6: Write & Confirm

1. Write the generated file to `.claude/rules/<skill-name>.md` in the current working directory.
2. Show the user the full contents of the generated file.
3. Tell the user:
   - Where the file was written
   - That it belongs to their project and can be edited freely
   - That sections not present in the file fall back to the skill's built-in defaults
   - That they can re-run the wizard at any time to reconfigure

**What to defer to a human:** Verifying that the generated rules accurately reflect their project's conventions. The wizard can detect configuration files and suggest values, but it cannot verify the generated rules are correct or complete for their team's actual practices. Encourage them to review and refine the file.
