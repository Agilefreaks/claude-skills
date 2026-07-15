# Reference: The memorize-it commit format

The standard, the field glossary, and two alternative encodings a project can select at setup.

## Contents

- [The default format](#the-default-format)
- [Field glossary](#field-glossary)
- [Writing good conclusions](#writing-good-conclusions)
- [Variant: native git trailers](#variant-native-git-trailers)
- [Variant: conventional commits](#variant-conventional-commits)
- [How the block survives a squash](#how-the-block-survives-a-squash)

---

## The default format

```
[<key>] <summary>

why: <required>
what: <required>
tried: <optional>
next: <optional>
concerns: <optional>
```

- **`<key>`** — the issue/ticket key (e.g. `feat-123`, `PROJ-456`). Derivation is a project decision.
- **`<summary>`** — a concise, imperative subject line, ideally ≤ 72 characters.
- **The block** — a run of `key: value` lines after a blank line. There are **no delimiters**: the field labels *are* the structure. A value may wrap onto indented continuation lines until the next `key:` label. The labels stay recognizable even after a squash concatenates several messages, so the block reads back out of a collapsed body (a regex over the known labels, or the host's own `* <subject>` separators, finds each block).

A worked example:

```
[feat-123] Add token-bucket rate limiter

why: API had no per-key throttling; a handful of keys were saturating the
  shared pool and degrading latency for everyone.
what: token-bucket middleware in api/limiter.ts, 100 req/min default,
  applied to all /v1/* routes.
tried: fixed-window — rejected, too bursty at window edges. leaky-bucket —
  rejected, more moving parts than we needed today.
next: make the limit configurable per plan tier (blocked on the billing work).
concerns: the bucket state is in-process, so limits are per-instance; this is
  fine single-node but needs a shared store (Redis) before we scale out.
```

An optional one-line summary paragraph may precede the block if it helps a human skim; the labeled lines remain the machine-recognizable part.

## Field glossary

| Field | Required | Captures |
|---|---|---|
| `why` | yes | The problem or motivation. Why this change exists at all. |
| `what` | yes | What actually changed, at a level a reader can act on. |
| `tried` | no | Alternatives considered and **why they were rejected**. The most valuable field for stopping a future session from re-exploring dead ends. |
| `next` | no | Known follow-ups, deferred work, TODOs this change implies. |
| `concerns` | no | Risks, assumptions, blast radius, things to watch, things left unverified. |

**Extending the set.** A project may add fields in its rules file. Useful additions seen in the wild:

- `rejected` — a stricter form of `tried` (dismissed options only).
- `constraint` — a rule that shaped the decision and may still be active.
- `related` — cross-references to other commits/issues by hash or key.
- `tested` / `not-tested` — verified behaviors vs. unverified assumptions.

Keep required fields minimal; optional fields should be filled only when they carry real information. `next: none` is noise.

## Writing good conclusions

- **Write for a reader with no memory of the session.** Not for `git blame`, not for a changelog.
- **`tried` is where the value is.** A diff shows the chosen path; only the commit can record the paths not taken and why.
- **Be concrete in `concerns`.** "Might have edge cases" helps no one. "Assumes UTC; breaks for non-UTC tenants" is a gift to the next person.
- **Don't pad.** Leave an optional field out rather than filling it with filler.
- **No AI attribution.** The commit is the developer's — never add a `Co-Authored-By:` trailer for Claude or a "Generated with" line. The block captures the reasoning, not the tool.

## Variant: native git trailers

Selectable at setup for teams that want `git`-native parsing. Trailers are `Key: value` lines in the last paragraph of the message, parseable by `git interpret-trailers --parse` and readable via `git log --format='%(trailers:key=Why)'`. Unknown keys are ignored, so this degrades gracefully. (Note: `git interpret-trailers` only parses the *final* trailer block, so after a squash concatenation only the last commit's trailers stay in trailer position — recall on the trunk still relies on a label regex, same as the default format.)

```
[feat-123] Add token-bucket rate limiter

A one-line summary paragraph if useful.

Why: API had no per-key throttling; keys were saturating the shared pool.
What: token-bucket middleware, 100 req/min, on /v1/*.
Tried: fixed-window (bursty); leaky-bucket (heavier) — token-bucket simplest.
Next: per-tier limits (blocked on billing).
Concerns: in-process state, per-instance limits; needs Redis before scale-out.
```

Trade-off: trailers are best for short, single-line values. Multi-paragraph prose in `why`/`what` is more awkward than in the delimited block. Choose trailers when values are terse and `git`-native tooling matters; choose the delimited block when conclusions are prose.

## Variant: conventional commits

If the project uses conventional commits, keep that subject and carry the key in the scope or a `Refs:` trailer; the context block is unchanged.

```
feat(limiter): add token-bucket rate limiter

why: ...
what: ...

Refs: feat-123
```

## How the block survives a squash

- **Concatenation (e.g. GitHub "commit details" squash default):** each compliant commit's message — block included — is concatenated into the squash body, separated by the host's `* <subject>` bullets. A trunk commit therefore ends up with several labeled blocks, one per original commit. Recall reads them all.
- **PR/MR description (e.g. GitHub "PR title and description" squash default):** only the PR/MR body reaches the trunk. Mirror a single consolidated block into the PR/MR description so it survives.

Either way, the field labels (or a regex over them) let a reader or parser pull the blocks back out of a collapsed message. See `git-recipes.md`.
