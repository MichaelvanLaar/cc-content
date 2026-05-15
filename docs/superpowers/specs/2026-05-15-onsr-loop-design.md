# Design: ONSR Auto-Learning Loop for cc-content

**Date:** 2026-05-15
**Status:** Approved

## Problem

cc-content skills currently collect explicit user corrections at session end and store
them to `.claude/learnings.md`, but no skill reads that file at the start of a run.
A correction from session N has zero effect on session N+1 until `cc-content-session-wrap`
manually detects and promotes the pattern — which requires the user to remember to run it.

The `cc-config` plugin solved this with an Observe-Notice-Store-Recall (ONSR) loop that
makes learning automatic. cc-content needs the same, without conflicting with cc-config
when both plugins are active in the same project.

## Solution: Full ONSR Loop (Approach B)

Add the ONSR loop to all five cc-content skills. Each skill gets:

- **Step 0 — Recall** at the start: silently read and apply all entries in
  `.claude/learnings.md` (all plugin tags, not just cc-content).
- **Store phase** at the end: auto-append qualifying observations before asking the
  user for explicit feedback.

`cc-content-session-wrap` is a special case: it already IS the store mechanism
(Steps 3–5). It gets recall at the start, and its pattern-detection step is scoped
to `[cc-content:*]` tags only.

## Coordination with Other Plugins (e.g., cc-config)

All plugins share `.claude/learnings.md`. Coordination is achieved by convention, not
code — no plugin needs to detect or import another:

- **Recall reads everything.** A cc-config entry like "client uses formal German" is
  useful context when writing a LinkedIn post. All entries are applied silently.
- **Store writes only own-tagged entries.** cc-content skills tag entries
  `[cc-content:<skill-name>]`; cc-config tags `[cc-config-*]`. No namespace collision.
- **Pattern detection is scoped.** `cc-content-session-wrap` only surfaces and promotes
  `[cc-content:*]` patterns. cc-config entries are left for cc-config tooling to handle.
- **Semantic deduplication prevents cross-plugin bloat.** Before appending any entry,
  the store phase checks whether a semantically equivalent fact already exists anywhere
  in `learnings.md` (any tag). Whichever plugin writes a fact first "wins"; later
  plugins see it in their Step 0 recall and skip writing it again.

This approach scales to any number of future plugins without changes to existing ones.

## The ONSR Loop Definition

### Step 0 — Recall

> If `.claude/learnings.md` exists, read it silently. Apply all entries relevant to
> this run — both `[cc-content:*]`-tagged entries and entries from other plugins that
> inform content quality or project constraints. Do not announce this step. If the file
> is absent, continue normally.

### Store Phase

Before asking the user for explicit feedback, review the run and append qualifying
observations to `.claude/learnings.md`, tagged `[cc-content:<skill-name>]`, one line
per entry. Create the file with a standard header if it does not exist.

**Qualifies:**

- A content preference or constraint revealed during the run that is not already
  captured in any `context/` file loaded by this skill or in `CLAUDE.md`
- A correction or adjustment the user made to the output
- A discovered project fact that would change future skill behavior
- Accepted or rejected suggestions that deviate from generic best practices

**Does not qualify:**

- Standard best-practice behavior applied without deviation
- Facts already in any `context/` file or `CLAUDE.md` — even if the user mentioned
  them again
- Anything derivable by re-reading context files without running the skill
- Facts semantically equivalent to an entry already present in `.claude/learnings.md`
  under any plugin tag (whichever plugin wrote it first owns it; when in doubt, skip —
  redundancy is worse than a missed entry)

**Entry format:**

```
[cc-content:<skill-name>] <concise fact or correction, one line> — <YYYY-MM-DD>
```

Example entries:

```
[cc-content:cc-content-linkedin-post] client prefers hooks starting with a question rather than a statement — 2026-05-15
[cc-content:cc-content-onboarding] client uses "Mittelstand" as key identity term; avoid "SME" — 2026-05-15
```

**Standard file header** (write only when creating the file):

```markdown
# Learnings

Corrections and feedback collected during content sessions.
Entries are tagged by skill and dated.

---
```

## Per-Skill Changes

### `cc-content-linkedin-post` — Full ONSR

**Add Step 0** (before current Step 1):

> Read `.claude/learnings.md` silently if it exists. Apply all relevant entries to
> inform this run. Do not announce this step.

**Modify Step 5** (Feedback): insert the store phase before the feedback question.
Claude auto-stores qualifying observations from this run, then asks:

> "Did this post meet expectations? If you have any corrections or notes for future
> posts, share them here — or press Enter to finish."

If the user provides a correction, append it as a tagged entry (existing behavior).
After the user responds, confirm any entries written: "✓ N learning(s) saved."

---

### `cc-content-onboarding` — Full ONSR with post-write constraint

**Add Step 0** (before current Step 1):

> Read `.claude/learnings.md` silently if it exists. Apply relevant entries — in
> particular, prior path preferences, rejected interview answers, or project constraints
> that would change how questions are asked this run. Do not announce this step.

**Add store step after Step 7** (summary): auto-store nuances discovered during the
interview that were not formalized into any context file just created in this run.

The qualification check must explicitly verify the candidate fact is not present in
any file written during Steps 2–5 of this run before appending.

---

### `cc-content-samples-curation` — Recall + lightweight store

**Add Step 0** (before current Step 1):

> Read `.claude/learnings.md` silently if it exists. Apply relevant entries. Do not
> announce this step.

**Add store step after Step 5** (loop end, when user declines to curate more):
evaluate the full curation session — not individual loops — for patterns worth storing
(e.g., "client consistently curates posts under 1,500 characters", "client rejected
long-form samples as too time-consuming to read").

---

### `cc-content-session-wrap` — Recall + scoped pattern detection

**Add Step 0** (before current Step 1):

> Read `.claude/learnings.md` silently if it exists. This ensures pattern detection
> in Step 5 operates on the most current state of the file. Do not announce this step.

**Modify Step 5** (detect recurring patterns): scope the scan to entries tagged
`[cc-content:*]` only. Add this instruction to the step:

> "Only scan entries prefixed `[cc-content:`. Entries from other plugins are managed
> by their own tooling and must not be surfaced or promoted here."

No additional auto-store: Steps 3–5 are the store mechanism for this skill.

---

### `cc-content-new-skill` — Recall + template update

**Add Step 0** (before current Step 1):

> Read `.claude/learnings.md` silently if it exists. Apply relevant entries (e.g.,
> prior skill-creation observations, project-level defaults). Do not announce this step.

**Modify Step 5** (generate SKILL.md skeleton): update the generated Step 0 and Step 8
in the skeleton so every skill produced by new-skill inherits the full ONSR pattern
automatically:

- Generated **Step 0**: recall all learnings silently.
- Generated **Step 8** (Feedback): store phase (auto-store qualifying observations)
  runs before asking the user for explicit corrections.

## Files to Change

| File                                                             | Change                                |
| ---------------------------------------------------------------- | ------------------------------------- |
| `plugins/cc-content/skills/cc-content-linkedin-post/SKILL.md`    | Add Step 0; modify Step 5             |
| `plugins/cc-content/skills/cc-content-onboarding/SKILL.md`       | Add Step 0; add store after Step 7    |
| `plugins/cc-content/skills/cc-content-samples-curation/SKILL.md` | Add Step 0; add store after Step 5    |
| `plugins/cc-content/skills/cc-content-session-wrap/SKILL.md`     | Add Step 0; scope Step 5 pattern scan |
| `plugins/cc-content/skills/cc-content-new-skill/SKILL.md`        | Add Step 0; update SKILL.md skeleton  |

## Out of Scope

- No changes to `cc-config` or any other plugin.
- No shared `_shared/onsr-loop.md` file: the qualification logic varies enough per skill
  (especially onboarding's post-write constraint and session-wrap's scoped scan) that a
  shared file would require per-skill overrides and add per-invocation token cost without
  meaningful deduplication benefit.
- No `learnings.md` file size cap or garbage-collection mechanism — out of scope for this
  iteration. The smart-filter qualification criteria are the primary defense against bloat.
