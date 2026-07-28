---
name: memorize-it
description: "Capture the conclusion of a prompting/working session as structured context in git commits (why, what, tried, next, concerns) and recall it later so a future session or reviewer understands why decisions were made. Use when committing, when writing or revising a PR/MR description, when the user says 'memorize this' / 'save the session', when investigating why code exists or what was decided, and as context for a code review. Squash-and-merge aware: rich per-commit context survives onto the trunk and is recalled with a tiered fallback (commit body → PR/MR description → individual PR commits). Also use when asked to set up, configure, or onboard this skill, or to audit commits that don't follow the standard."
---

# Memorize It

> Turn the conclusion of prompting work into durable, structured context stored in git — so the next session (human or Claude) understands *why* things were built and decided, not just *what* changed.
>
> The commit convention, issue-key derivation, merge/squash behavior, any PR/MR description template, and platform mechanics (how to read/update PRs or MRs) are project-specific and defined separately in your project's rules. This skill ships working defaults so it is useful with zero configuration.

This skill has four working modes, plus setup:

- **Capture** — write a session conclusion into the commit and into the PR/MR description.
- **Preserve on merge** — make sure that context survives a squash-and-merge onto the trunk.
- **Recall** — read stored context back, tiered, so a future session or a review is grounded in prior decisions.
- **Audit** — find commits and descriptions that don't follow the standard and prompt the developer.

It is designed to work alongside — not replace — the `feature-development` skill (capture at hand-off) and the `code-review` skill (recall feeds problem validation). See [Integration](#integration).

---

## The standard commit

The default format is a subject with an issue key, followed by a labeled, parseable context block:

```
[feat-123] Add token-bucket rate limiter

why: API had no per-key throttling; a few keys were saturating the pool.
what: token-bucket middleware, 100 req/min default, applied to /v1/*.
tried: fixed-window (too bursty) and leaky-bucket (heavier) — token-bucket was simplest.
next: make limits configurable per plan tier.
concerns: in-memory store is single-instance; needs a shared store before horizontal scale.
```

Two things matter:

- **Subject** — `[<key>] <summary>`, where `<key>` is the issue/ticket key. How the key is derived (branch name, PR title, task tool) is a project decision; see [Extension points](#extension-points).
- **Context block** — a run of `key: value` lines after a blank line. The field labels are the structure — no delimiters. Values may wrap onto indented continuation lines. The labels stay recognizable even after a squash concatenates several commit messages, so the block can be read back out of a collapsed message.

**Fields.** `why` and `what` are required; `tried`, `next`, and `concerns` are optional and filled when relevant. The field set is extensible — a project may add fields (e.g. `rejected`, `constraint`, `related`). See `references/commit-format.md` for the full field glossary, a git-trailer variant (for `git`-native parsing), and a conventional-commits variant.

**Write the block for the next reader, not for `git blame`.** It is a briefing: enough that someone with no memory of this session knows why the change exists and what to watch out for.

### Getting the craft right

The format is easy; the writing is where these go wrong. Six things that separate a block worth reading from a block that gets skipped:

- **The subject names the change, not the area.** Imperative mood, ≤ 72 characters, no trailing period. *"Pin auth flow step order with explicit priorities"*, not *"auth flow fixes"* or *"update module"*. If the subject could describe a dozen different commits, it says nothing.
- **`what` names decisions, not files.** The diff already lists what changed; only the message can say *which choices were made and why they look like that* — the value picked, the scope it applies to, the thing deliberately left alone. Skip a file inventory; git has one.
- **Length in proportion to the change.** A one-line fix gets a two-line block. A subtle change with a real failure mode behind it earns a long `why`. Inflating a small commit trains readers to skim past the block entirely.
- **One logical change per commit.** Under a concatenating squash each block becomes a section of the trunk message, so a commit mixing two concerns produces a block that briefs neither. Split the commit; the blocks get shorter and sharper.
- **Keep the field order** (`why`, `what`, `tried`, `next`, `concerns`). Predictable to skim, predictable to parse — no reader has to hunt for the risks.
- **Say what was verified, and how.** "Applied in prod; admin API confirms the live order" is worth more than "tested". Name what was *not* verified with the same precision — that is what `concerns` is for.

---

## The standard PR/MR description

A commit block is the conclusion for one commit. A PR/MR description is the conclusion for the **whole change**, written for the person deciding whether to merge it. The content rules are identical (see [What belongs in the block](#what-belongs-in-the-block) — project altitude, no secrets, no one developer's setup); only the level of detail differs.

**This standard applies to any description this skill writes or updates** — opening a PR/MR, revising one, or maintaining a managed section. That is separate from *mirroring*, which is only about keeping a consolidated block in the description so it survives a squash (see [Extension points](#extension-points)).

**The title is a commit subject.** On hosts that compose the squash message from the PR/MR title, the title *becomes* the trunk subject and outlives the PR — so it follows the project's commit convention, not conversational labelling. `[KEY] Pin auth flow step order`, never `fixes as discussed`.

**The body — three sections, in this order:**

1. **Why** — the problem in terms someone outside the session recognizes: what was broken or missing, and how it showed up. One paragraph. If a reviewer can't tell from this section whether the change is worth making, nothing below will save it.
2. **What** — one bullet per decision a reviewer would want named, with the reasoning compressed to a clause. Not one bullet per file, and not a re-narration of the diff.
3. **Limits of this change** — what it does not cover, what stayed unverified, and what is left to a reviewer's judgment. Stated as facts about the change: *"the convention is advisory — no hook, no CI lint, so nothing rejects a non-compliant message"*, not *"say the word if you want a hard gate"*.

The third section is the one that earns trust, and the one most often dropped. A description that admits its own gaps is reviewable; one that reads as a finished sales pitch forces the reviewer to go find the gaps themselves.

**Leave out** anything that isn't load-bearing: a test plan that restates what CI already runs, a "how to review this" preamble, screenshots of unchanged UI, and setup instructions aimed at one person's machine. If the diff says it plainly, the description doesn't repeat it.

**When the squash source is the PR/MR description, the description is also a commit message.** Trunk recall finds labeled `key: value` lines, not markdown headings — so in that case the description carries *both*: the three prose sections for the reviewer, and a labeled block in a marker-delimited managed section for the trunk. That is what [PR/MR mirroring](#extension-points) means, and it has a hard shape requirement: **the labels sit at column 0** — not indented under a heading, not inside a fenced code block — or nothing can parse them back out, and the bundled CI check in `pr` mode will reject the description. Mirroring is therefore a prerequisite for that mode, not a recommendation.

**Adapt to an existing template.** If the project has a PR/MR template, fill *its* sections with this content instead of imposing these headings — the same three questions answered under whatever names the project uses. Never delete template sections that the project's process depends on (a checklist, a ticket link); mark a genuinely inapplicable one as such rather than silently dropping it.

See `references/pr-description.md` for a worked example, how to consolidate several commits into one description, and how to revise a description on a re-review.

---

## Setup

**Setup runs on first use.** If `.claude/rules/memorize-it.md` does not exist yet, the skill is not configured — run Setup **before** doing anything else, including when the user invokes the skill bare (`/memorize-it`) or asks to capture/recall/audit. Do not silently fall back to defaults on an unconfigured project; walk the user through Setup first, then continue with what they asked. Setup also runs on request ("set up", "configure", "onboard").

**A rules file that exists but isn't in the repository doesn't count as configured.** Check that too (`git ls-files --error-unmatch`, `git check-ignore -v`): if the file is untracked or ignored, an earlier Setup left the config in one clone only, so the convention never reached the trunk — and therefore never reached a new clone, a fresh worktree, a teammate, or the next session. Don't re-ask every question in that case: read the existing file, confirm its recorded choices still hold, and resume at step 5 below to get it tracked.

1. **Read existing project rules first** (`.claude/rules/`, `CLAUDE.md`). Do not duplicate a commit convention, trunk branch name, or issue-linking rule that already exists. If the project already defines a commit format, adapt to it rather than imposing this skill's default.
2. **Inspect the project** for signals to *pre-fill* the questions — never to answer them silently:
   - Recent commit history (`git log --oneline -30`) — is there a prevailing subject style? Do subjects already carry an issue key? Do any bodies already carry structured context?
   - Branch naming (`git branch -a`) — does it encode an issue key (e.g. `feat/PROJ-123-...`)?
   - The remote host (`git remote -v`) — GitHub, GitLab, or other — which determines available PR/MR mechanics.
   - A conventional-commits config (`commitlint`, `.commitlintrc*`) or a PR/MR template.
3. **Confirm every decision with the user — one at a time, no assumptions.** Present each aspect below as its own plain-language choice, pre-filled with what you detected (or the default), and wait for the user to confirm or change it. **Detection is a suggestion, not an answer:** even when a detected value matches the default, still surface it and get an explicit confirmation — never adopt a value silently because it "matches the default." **Phrase every question in plain, user-facing language — never expose this skill's internal mode names (e.g. "Capture", "Preserve on merge", "Recall", "Audit") in a question or option label.**
   - **Commit convention** — if the history shows a prevailing style, summarize it in plain language and ask to confirm or adjust (e.g. *"Your commits look like `[WEB-123] Summary (#PR)` — keep that subject and add the why/what block below it?"*). If nothing is detected, offer: bracket-key `[key] summary` (default), conventional-commits `type(scope): summary`, or a custom rule. The context block is kept either way.
   - **Issue-key derivation** — from branch name, PR/MR title, a task-management integration, or none. Also settle **what happens when no key can be derived**: ask the developer (default), or a keyless subject form the project accepts (e.g. plain `<summary>`, or conventional-commits without a key). If a keyless form is chosen, the generated hook and CI `SUBJECT_REGEX` must admit it — otherwise the documented fallback is one the enforcement blocks.
   - **Required fields** — default `why` + `what` required; `tried`/`next`/`concerns` optional. Offer to add project-specific fields.
   - **Context block style** — labeled `key: value` lines (default) or native git trailers (`git interpret-trailers`-parseable). See `references/commit-format.md`.
   - **Squash-message source** — how the host composes a squash commit message: *commit details* (concatenates commit messages — blocks survive automatically) or *PR/MR title + description* (blocks must live in the PR/MR body). Determines the [Preserve on merge](#preserve-on-merge) path. Default: commit details.
   - **PR/MR description structure** — the default is `Why` / `What` / `Limits of this change` (see [the standard](#the-standard-prmr-description)). **If the project has a PR/MR template, say so and offer to fill its sections instead** — the same content under the project's headings. Record which was chosen.
   - **PR/MR mirroring** — off by default. This is narrower than the description standard above: it's whether to keep a labeled, parseable block in the description so context survives a squash. **Required — not merely recommended — when the squash source is the PR/MR description**, since the description then becomes the trunk message and only labeled lines survive as recallable context. If the user picks that squash source and declines mirroring, say plainly what breaks (trunk recall finds nothing, and CI enforcement in `pr` mode rejects every PR) and ask again before recording it. If on, record the platform mechanics for reading and updating the description, and that the labels sit at column 0.
   - **Audit scope** — the default range to scan (e.g. `<trunk>..HEAD`) and whether to prompt about non-compliant commits proactively.
   - **Auto-use nudge** — this skill installs no hooks by default, so on its own it only fires when the model recognizes a trigger. Offer (recommended, default yes) to add a one-line nudge to the project's `CLAUDE.md` so the convention is in front of Claude every session and it reaches for the skill when committing. Ask before writing to `CLAUDE.md`; if the user declines, the skill still works when invoked explicitly ("memorize it").
   - **Enforcement hook** *(optional, off by default)* — offer to generate a `commit-msg` git hook under `.githooks/` that **enforces** the standard on local commits (rejects a commit whose subject doesn't match the convention, whose body is missing a required field, or whose message carries an assistant-attribution footer, a credential-shaped value, or a local path). Enforcement only — it never rewrites messages. Off by default because it can interrupt flow; recommend it only to teams that want a hard local gate. Be clear about its limitation before the user opts in (see the deferral note).
   - **CI enforcement** *(optional, off by default)* — a local hook only guards the clone it's in; the trunk's squash message is composed server-side. Offer to generate a CI workflow that runs the same checks (convention, required fields, no attribution, no credential-shaped values, no local paths) on every PR so that, made a **required status check**, a non-compliant PR cannot merge — the way to guard everything reaching the trunk on any host, cloud included. Off by default. Only meaningful if the project uses CI.
4. **Write the config, then the nudge and hook if accepted:**
   a. **Always write `.claude/rules/memorize-it.md`** — even when every decision was left at its default. The file records the confirmed choices (and platform mechanics, when mirroring is on) and, crucially, marks the skill as configured so it is not treated as first-run again. When all defaults were accepted, write a minimal file that says so (e.g. a note that Setup ran and all defaults were confirmed) rather than leaving no file behind.
   b. **If the auto-use nudge was accepted,** add a concise line to the project's `CLAUDE.md` (create the file if absent), for example:
      > When making a commit, use the `memorize-it` skill to record the session conclusion (why / what, plus tried / next / concerns where relevant) in the commit message, following `.claude/rules/memorize-it.md`.

      Be idempotent — check for an existing memorize-it nudge first and don't duplicate it.
   c. **If the enforcement hook was accepted:**
      - Copy `assets/commit-msg` (bundled with this skill) to `.githooks/commit-msg`, adapting its CONFIG block to the confirmed choices — set `SUBJECT_REGEX` to match the chosen commit convention — **including the keyless form, if one was configured** — and `REQUIRED_FIELDS` to the chosen required fields. The content checks (`ATTRIBUTION_REGEX`, `SECRET_REGEX`, `LOCAL_REGEX`) reject assistant footers, credential-shaped values, and local paths respectively — leave them unless the project wants one off. Make it executable (`chmod +x .githooks/commit-msg`).
      - Point git at the directory: `git config core.hooksPath .githooks`. **First check whether the project already manages hooks** (an existing `core.hooksPath`, a `.husky/` directory, or existing `.githooks/` scripts). If so, do **not** overwrite or repoint — integrate the `commit-msg` check into the existing setup, or hand the integration to the user. Never clobber another tool's hook wiring.
   d. **If CI enforcement was accepted:**
      - For a GitHub Actions project, copy `assets/commit-lint.yml` (bundled with this skill) to `.github/workflows/memorize-it-commit-lint.yml`, adapting its CONFIG block: `SUBJECT_REGEX` and `REQUIRED_FIELDS` to the confirmed choices (the `ATTRIBUTION_REGEX` / `SECRET_REGEX` / `LOCAL_REGEX` content checks need no adaptation), and `CHECK_MODE` to the squash-message source — `commits` when the host concatenates commit messages, `pr` when it composes from the PR title + description. **`pr` requires mirroring to be on:** the check looks for the labeled block in the description, which only mirroring puts there. If mirroring was declined, do not generate the workflow in `pr` mode — a required status check would then fail every PR; offer to turn mirroring on, or generate nothing and say why. If a workflow of that name already exists, show a diff and confirm before overwriting.
      - For other CI systems (e.g. GitLab CI), don't force the GitHub template — describe the equivalent job (run the same subject/field checks over the MR's commits or title/description) and let the user place it in their pipeline.
      - Tell the user the two human steps CI enforcement needs: make the check a **required status check** in branch protection, and (where the platform supports it, e.g. GitHub Enterprise pre-receive hooks or GitLab push rules) add a server-side commit-message rule for a hard gate on all pushes.
5. **Get the config into the repository.** Everything Setup writes — the rules file, the `CLAUDE.md` nudge, the hook, the CI workflow — is **shared project configuration**, not a local preference. Config that lives in a single clone is the standard way this skill gets "set up" and then has no effect: the trunk never carries the convention, so a fresh clone or worktree has nothing telling it the convention exists, nothing puts it in front of Claude, and commits drift back to the project's old habits (attribution footers included).
   - **Never** put these paths behind an ignore mechanism — not `.gitignore`, not `.git/info/exclude`, not `core.excludesFile` — and never write the rules content to a local-only file such as `CLAUDE.local.md`.
   - **Check for an ignore rule that already covers them** (`git check-ignore -v <path>`); a broad `.claude/` ignore is common. If one matches, say so and ask whether to narrow the ignore or force-add the paths. Don't quietly leave the config ignored.
   - **Read the generated files before staging them.** They are about to become public repo history. The platform-mechanics section is the risk: record *how* to reach the host (commands, endpoints, which credential is needed and where it lives), never a token, cookie, or password. If a mechanic can't be described without a secret, leave the secret out and note what the operator must supply.
   - **Force-add only this skill's own config paths.** If the matching ignore rule exists to keep secrets out of the repo (`.env*`, `*.tfvars`, key material), that rule is doing its job — never override it, and never relocate config into a file the project ignores for that reason.
   - **Stage and commit them** — using this skill's own convention, now that it's configured — and tell the user the config reaches everyone else only once that commit lands on the trunk.

**What to defer to a human (Setup):**
- Landing the config. Setup can commit it; pushing, opening a PR/MR, and merging are the user's calls. Until it is on the trunk, the skill is configured in this clone only.
- Verifying that the chosen squash-message source matches the host's actual repository setting (e.g. GitHub's "Default commit message" for squash merges). The skill records the intended behavior; a human must confirm the host is configured to match.
- The nudge is advisory, not enforcement: it raises the odds Claude uses the skill but does not guarantee it on every commit. Editing `CLAUDE.md` requires the user's go-ahead.
- The enforcement hook guards **local** commits only. A squash-merge performed in a host's web UI composes the message server-side, where the hook does not run — so it cannot guarantee the message that lands on the trunk. Also, `core.hooksPath` is repo-wide: if the project already uses a hook manager, integrating (not overwriting) is a human decision.
- CI enforcement only bites once it is a **required status check** — Setup generates the workflow but cannot configure branch protection; a human must mark the check required. Even then, on cloud hosts the person merging can hand-edit the squash message textbox at merge time, which no CI check sees; closing that fully needs a server-side rule (GitHub Enterprise pre-receive hook / GitLab push rule) that only some platforms offer.

---

## Capture

**Trigger:** whenever a commit is created (including the `feature-development` hand-off), or when the user says "memorize it" / "save the session".

1. **Compose the conclusion from the session you actually had.** Draw `why`/`what`/`tried`/`next`/`concerns` from the work just done — the problem being solved, the approach chosen, alternatives you rejected and why, follow-ups, and risks or deferrals. Do not invent fields you have no basis for; leave optional fields out rather than padding them.
2. **Derive the issue key** per the project's rule (branch name, PR/MR title, task tool). **If none can be derived, ask the developer for one — don't invent a placeholder.** A made-up key is worse than no key (it points at nothing forever), and the shipped subject regexes require digits inside the brackets, so a wordy stand-in like `[no-key]` is rejected outright by the hook. If the project configured a keyless subject form at Setup, use that. If it didn't and the developer has no key, say plainly that the convention requires one and that enforcement will reject the commit without it, then let them decide: supply a key, add a keyless form to the project rules, or commit outside the convention knowingly — which [Audit](#audit) will flag.
3. **Build the subject and block** per [the standard](#the-standard-commit) (or the project's configured convention).
4. **Re-read what you wrote before it lands** — check it against [What belongs in the block](#what-belongs-in-the-block) and cut anything that fails: a secret or sensitive value first, then machine-local detail, session residue, and AI attribution. `references/git-recipes.md` has a one-line sweep that catches the mechanical cases.
5. **Create the commit** with the subject and block as its message.
6. **Write the PR/MR description** whenever the change has one — opening it, revising it, or (with mirroring on) maintaining its managed section with a consolidated conclusion for the change as a whole. Follow [the description standard](#the-standard-prmr-description) for shape and the project rules for platform mechanics, then re-read it with the same sweep as step 4.

**Keep the commit and the description in agreement.** They are two altitudes of one conclusion, not two drafts: the commit carries its own slice, the description carries the whole change. When a line is corrected in one, correct it in the other in the same pass — the commit is what lands on the trunk and gets recalled, so it is the copy that matters most and the easiest to forget. Neither should be a copy-paste of the other.

### What belongs in the block

**Write at project altitude.** The reader is someone in a fresh clone on another machine, months from now. What survives that trip is the shared, reproducible view of the project: the change, the repository as everyone gets it, and the running system. Every developer's setup below that line legitimately differs — different paths, different tooling, different local ignores, a different `.env` — so a fact about *one* environment is not just noise, it is wrong for nearly every reader, and it reads as an instruction they can't act on.

That line also protects two kinds of privacy, and it is the record's permanence that makes it matter: **the developer's** (a home-directory path carries a username; a worktree layout and local config are nobody else's business) and **the project's** (see the secret rule below).

**Never write a secret into the record.** A commit body and a PR/MR description are permanent, world-readable to anyone with repo access, and mirrored into forks, CI logs, and host APIs — a secret placed there is not deleted by amending, and rotating it is the only remedy. So: no passwords, tokens, keys, connection strings, or session cookies; no customer or personal data; and no internal hostnames, IPs, or account identifiers that a project treats as sensitive. When a secret is genuinely part of the story, name its *location* rather than its value (*"the credential lives in the secret manager under `<name>`"*) — and check even that against the project's rules, since some treat the path itself as sensitive. If a value already reached a message, say so plainly and immediately: it must be rotated, not just rewritten out.

Keep out of the commit body and the PR/MR description:

- **Anyone's local setup** — local ignore entries (`.git/info/exclude`), local hook wiring or `git config`, an untracked `.env` or local override file, editor and shell config, stashes, branch cleanups. All of it is per-developer by design. A change to something *tracked* (a committed `.gitignore`, a checked-in hook) is part of the project and belongs in the record; the local counterpart never does.
- **Absolute paths, other clones, and worktrees** — including "delete your local copy of X". A path under a home directory names a person and points at a layout nobody else has; repo-relative paths are the portable form. If a real migration step is needed, phrase it for anyone on the team, not for the machine you're on.
- **Housekeeping outside the diff** — if it isn't in the change, it isn't part of the record.
- **Your own access limits.** State the state of the *change or the system*, never the state of your credentials, tooling, or permissions.
- **The session itself.** Offers ("say the word if you want…"), questions, first person, and anything addressing the reader directly turn the record into a transcript of a prompting session. It is a **conclusion**: what the change is and what is true about it, in the third person, decided.

The last two are the easiest to miss, because both read as helpful. Rewrite them as facts about the change:

| Don't write | Write |
|---|---|
| *"Staging was never verified — its admin password isn't in the env's vars file, so I couldn't read it."* | *"Staging still needs an apply — it carries the same defaults and can reorder the same way. Prod is applied and verified; staging is not."* |
| *"Removed the `.git/info/exclude` entry (local change, not in the diff)."* | *(cut entirely — nobody else has that entry)* |
| *"Say the word if you want a hard gate (a `commit-msg` hook, or a required CI check)."* | *"The convention is advisory: no `commit-msg` hook and no CI lint, so nothing rejects a non-compliant message."* |
| *"I see no ordering hazard left."* | *"No ordering hazard remains."* |

A missing verification is worth recording; *why you personally couldn't do it* isn't. An unbuilt gate is worth recording; *offering to build it* isn't. The test before every line: **would this still be true, and still actionable, for someone reading it in a fresh clone?** If not, cut it.

**Authorship stays with the developer.** The commit is the developer's work. Do **not** add AI co-authorship — no `Co-Authored-By:` trailer for Claude, no "Generated with" line, no assistant attribution anywhere in the message or in the PR/MR description. The block records the reasoning behind the change, not the tool that helped write it.

A general instruction elsewhere in the environment may tell you to append those footers by default. **For the commit message and PR/MR description this skill writes, this rule takes precedence** — compose them without the footers, and afterwards verify none was appended (`git log -1 --format=%B`, and re-read the PR/MR body). It is scoped to what this skill writes and says nothing about other artifacts. A team that needs assistant attribution for provenance should record that in its project rules and turn the corresponding check off (`ATTRIBUTION_REGEX=''` in the generated hook and CI workflow).

**The block is where the long rationale lives.** Once the reasoning is in the commit, it doesn't also need to be pasted into the code as a wall of comments. Keep in-code comments to the short "what you must know if you edit this line" form and let the block carry the why, the rejected options, and the failure it prevents. Duplicating the rationale in the source means two copies that drift.

**A standing rule belongs in the project's living docs, phrased forward.** When the session's conclusion implies a rule that applies to *future* work — not just an account of this change — the incident narrative stays in the commit, and the docs (the project's conventions file, a rules file, a code comment) get the instruction that follows from it: what to do, in what order, and what breaks if you don't. A gotcha that only recounts what went wrong once gives the next person nothing to act on. Convert it: *"leaving X at the default scrambled the order in prod"* → *"set X explicitly on anything you add; the default lets it reorder itself."*

**Where the block belongs when commits get reshaped.** Under a checkpoint-then-curate flow (e.g. `feature-development`'s hand-off, which reshapes many working checkpoints into a few logical commits), do not spend effort memorizing on throwaway checkpoints — write the conclusion on the **final, reshaped commits**. And because most branches squash-merge, the message that ultimately matters is the one that lands on the trunk: ensure the branch's blocks (or the PR/MR description) are compliant *before* the merge, per [Preserve on merge](#preserve-on-merge).

**What to defer to a human:** the factual accuracy of the conclusion. You can record what was decided this session; a human confirms it reflects reality — especially `concerns` and `next`, which commit the team to follow-ups. Whether a line is machine-local is also judgment, not a check: an enforcement hook can reject an attribution footer but cannot tell that "the other worktree has a copy" is about your laptop. If the user says a line doesn't belong, treat it as a class of mistake and re-scan the rest of the message and the PR/MR body for the same thing, rather than fixing only the line they quoted.

---

## Preserve on merge

A squash-and-merge collapses every branch commit into a single trunk commit. The per-commit context must survive that collapse, or recall on the trunk finds nothing.

**How the squash commit message is composed is host-specific — it is an extension point.** The two common cases and their defaults:

- **Host concatenates commit messages** (e.g. GitHub squash with the "commit details" default): every compliant per-commit block is carried into the squash commit body automatically, and the host's own `* <subject>` separators keep them apart. **No assembly step is needed** — just compliant commits. The one caution: whoever performs the merge must **not clear the pre-filled squash body** in the host UI. Surface that reminder at merge time.
- **Host composes from the PR/MR description** (e.g. GitHub squash with the "PR title and description" default): the block must be in the PR/MR body to survive — the description *is* the trunk message, so it is the only copy recall will find. Keep the consolidated conclusion mirrored into the description's managed section (enable PR/MR mirroring in [Capture](#capture)), written per [the description standard](#the-standard-prmr-description). The PR/MR title becomes the trunk subject, so it must satisfy the commit convention too.

When you are asked to prepare a branch for merge, verify the survival path holds: either the branch commits are compliant (concatenation case) or the PR/MR description carries the consolidated block (description case). If neither, say so and offer to fix it before merge (amend/reword unpushed commits, or update the PR/MR description).

**What to defer to a human:** the squash-and-merge itself is performed in the host UI. The skill prepares and verifies the message that will land on the trunk and reminds the developer what to confirm; the human clicks merge and is responsible for not discarding the pre-filled body.

See `references/git-recipes.md` for the commands to list a branch's blocks and compose a consolidated conclusion.

---

## Recall

**Trigger:** when the user asks why code exists or what was decided, when investigating a file's history, as an input to a code review, and whenever prior decisions are relevant to the work at hand.

Recall is **tiered** — cheapest first, and it always produces *something* useful even when no structured context exists:

1. **Local commit body** (`git log`) — read the labeled `key: value` blocks from the relevant commits. On a branch these are per-commit; on the trunk a squash commit body holds the concatenated/consolidated blocks. This is offline and free; it is often enough.
2. **PR/MR description** — **if step 1 finds no context, always fall back to this at minimum.** Map the trunk commit to its PR/MR (hosts like GitHub append `(#123)` to the squash subject by default; otherwise search the host for the PR/MR containing the commit SHA) and read the managed section / description. Requires host access; the mechanics are project-configured.
3. **Individual PR/MR commits** — the deepest layer, for the step-by-step decision trail. After a squash the original commits remain attached to the PR/MR on the host even when they are gone from the trunk's history; fetch them and read each block. Use this when the consolidated context is not enough.

Tiers 2 and 3 require the project to have configured platform mechanics (how to map a commit to its PR/MR and read it). If those are not configured, tier 1 still works fully offline — say plainly that the PR/MR layers are unavailable because no platform mechanics are configured, rather than treating it as a failure.

**Graceful degradation:** if none of the available layers yields a structured block, fall back to plain commit subjects and PR/MR titles. Say explicitly that no structured context was found — do not imply decisions were recorded when they weren't.

Present recalled context as a short briefing: what the change did, why, alternatives rejected, and any `concerns`/`next` still open. When recall feeds a code review, pass this into the review's problem-validation step.

**What to defer to a human:** whether recalled decisions are still valid. Context reflects what was true when it was written; a `concern` may be resolved or a `constraint` lifted. Flag anything that looks stale rather than treating it as current fact.

See `references/git-recipes.md` for the exact commands per tier.

---

## Audit

**Trigger:** on request, per the configured audit scope, or when preparing a branch for merge.

1. **Scan the range** (default `<trunk>..HEAD` on the current branch; configurable).
2. **Flag non-compliant commits** — any commit missing the `[<key>]` subject (per the project's convention), missing the context block, or missing a required field label (`why`/`what` by default). Also flag four content failures, which a missing-field check won't catch:
   - **Secrets or sensitive values** — credentials, tokens, keys, personal or customer data. Report these first and separately: a published one needs **rotation**, and rewriting the message alone does not fix it.
   - **AI attribution** — a `Co-Authored-By:` trailer for an assistant, a "Generated with" line, or similar.
   - **Machine-local detail** — absolute home-directory paths, other worktrees or clones, local ignore/hook state, or first-person access limits ("I couldn't read…"). See [What belongs in the block](#what-belongs-in-the-block).
   - **Conversational residue** — offers, questions, first person, or second-person address, which make the record read as a prompting session instead of a conclusion.
3. **Report and prompt the developer** — list each non-compliant commit with what it's missing, then offer to fix what is safely fixable:
   - The **most recent** unpushed commit → `git commit --amend` with the corrected message.
   - **Earlier** unpushed commits → rewording these means an interactive rebase, which needs a terminal editor the agent can't drive. Prepare the corrected messages and hand the reword to the developer to run, rather than attempting it.
4. **Check the PR/MR description too, when the change has one.** Same content sweep, plus the shape: does it answer why, name the decisions, and state its limits? A description that stops at `What` is the common gap. On a description-sourced squash it is also the message that lands on the trunk, so it is the higher-priority fix.
5. **Prefer fixing the squash message over rewriting branch history.** Because most branches squash-merge, the message that lands on the trunk is the one that matters. If per-commit compliance can't be fixed cleanly, the higher-value move is to ensure the consolidated block is correct at merge time (see [Preserve on merge](#preserve-on-merge)) and to apply the convention going forward.
6. **Never silently rewrite.** Present every proposed message and let the developer approve it.

**What to defer to a human:** rewriting **already-published** history (amending pushed commits forces a shared-history rewrite that can disrupt collaborators) and running any **interactive rebase**. Report those commits as non-compliant, recommend fixing the convention going forward, and leave the decision to the developer.

---

## Integration

These skills stay independent; the touchpoints are soft:

- **`feature-development`** — its hand-off phase is a natural [Capture](#capture) trigger. When a feature or fix is handed off, memorize the conclusion into the commit so the decision trail is recorded at the moment it's freshest.
- **`code-review`** — [Recall](#recall) is an input to the review's problem-validation phase. Before reviewing, pull the stored context for the change so the reviewer knows the intent and the rejected alternatives, and doesn't re-raise settled decisions.

If neither skill is installed, `memorize-it` works on its own.

---

## Extension points

Every one has a working default, so the skill is useful out of the box and richer when configured. Configure them in `.claude/rules/memorize-it.md` via [Setup](#setup).

| Decision | Default |
|---|---|
| Commit convention | `[key] summary` bracket-key subject |
| Issue-key derivation | from the branch name |
| Required fields | `why`, `what` |
| Context block style | labeled `key: value` lines |
| Squash-message source | commit details (blocks concatenated) |
| PR/MR description structure | `Why` / `What` / `Limits of this change`, or the project's template |
| PR/MR mirroring | off |
| Platform mechanics (read/update PR/MR) | none — required only if mirroring is on |
| Trunk branch | the repository's default branch |
| Audit scope | `<trunk>..HEAD`, on request |
| Auto-use nudge in `CLAUDE.md` | offered during Setup (recommended on) |
| Enforcement hook (`.githooks/commit-msg`) | offered during Setup (off by default) |
| CI enforcement (PR commit lint) | offered during Setup (off by default) |

---

## Core principles

1. **Briefing, not blame.** The context block is written for a future reader with no memory of this session — the intent and the rejected paths, not a restatement of the diff.
2. **Only what travels — and nothing that shouldn't.** Record facts about the change and the system, never facts about the machine the session ran on or the tooling access it had; if it isn't true in a fresh clone, it doesn't belong. That cuts both ways: the record is permanent and world-readable, so no secret or sensitive value ever goes into it. (It also applies to the skill's own configuration, which is worthless until committed on the trunk — and must be readable by everyone before it gets there.)
3. **Survive the squash.** Context that doesn't reach the trunk is lost. Always confirm the survival path before merge.
4. **Tiered recall, guaranteed fallback.** Cheap-and-local first; when `git log` is empty, always check the PR/MR description at least. Never claim context that isn't there.
5. **Prompt, don't rewrite.** Audit surfaces non-compliant commits and offers fixes; it never rewrites published history on its own.
6. **Generic core, project edges.** Conventions, key derivation, and platform mechanics live in the project's rules. The methodology here is the same everywhere.
7. **Defer honestly.** The accuracy of a conclusion, the validity of an old decision, and the merge click itself are human calls — say so.
