# Compounding Corrections

This reference supports Phase 7 (Compound) in SKILL.md: how to keep the running note the
standing rules describe, and worked calls for routing a correction to project rules, project
memory, personal memory, or nothing. Read it when Phase 7 has an entry to route and the call
isn't obvious.

---

## The running note

One line per correction, appended the moment it happens — not reconstructed later. Each line
names the correction and the assumption behind it, for example:

> Assumed migrations run against a scratch DB; reviewer corrected — this project's test suite
> shares one DB across runs, migrations must be additive-only.

Append-only. It survives between features, so a correction seen once and never revisited simply
sits there — it costs nothing to keep and nothing to ignore. Do not summarize or prune it
yourself; Phase 7 reads the whole thing and decides what has earned a write.

## Worked routing calls

**→ Project rules.** A reviewer rejects a plan for touching a shared config file directly and
says every change to it goes through a generator script instead. That is a convention someone
could follow or violate; it binds the team, not just this session. Recurred or not, it cost
rework this round — it earns a write to the project's rules.

**→ Project memory.** Twenty minutes go into discovering that the test suite truncates its
database between every example, not every file — a fact, not a policy. It would have shortened
this round's exploration and will shorten the next one. It is true regardless of who is working;
nobody could disagree with it the way they could disagree with a convention. → CLAUDE.md.

**→ Personal memory.** The human asks, unprompted by any project convention, to always run the
full suite before proposing a plan, even when a partial run would answer the immediate question.
Nothing in the project documents this; it is how this person works, and it will hold in their
next repository too. It must not land in a file their teammates inherit.

**Dropped — already derivable.** A correction reads "remember that `UserValidator` lives in
`app/validators/`." `git log -- app/validators/` or a single directory listing answers this in
one command. Writing it down taxes every future session to save one lookup; drop it.

**Held, not raised — first sighting.** A reviewer prefers squash commits over the project's
usual merge commits, once, on one PR. It didn't cost rework — the human simply requested a
different history shape at the end. Nothing says it will recur. It stays on the note. If a later
round hits the same preference, that second sighting is what makes it a candidate.

## "New rule" vs. "existing rule, unclear or unread"

Before writing a new project rule, check whether one already exists that should have covered
this. Read the relevant rules file, not just its section headings.

- **If no rule exists**, or the closest one addresses a different concern, this is a genuine new
  rule.
- **If a rule exists and says this**, the correction is not a new finding — the session simply
  didn't read it, or read it and found it ambiguous. Nothing to propose, unless the miss reveals
  the rule is hard to find or worded so it can be misread; then the write is an *edit* to that
  rule (sharper wording, a clearer example, a better section), not a second rule beside it.
- **If a rule exists but says something subtly different** — narrower, or for a different
  situation — propose tightening that rule's scope rather than adding one that overlaps it.
  Two rules that almost say the same thing are worse than one, because a future reader has to
  reconcile them.
