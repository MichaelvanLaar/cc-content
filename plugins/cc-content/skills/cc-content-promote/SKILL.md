---
name: cc-content-promote
description: >
  Use this skill to register an existing file as context in CLAUDE.md so content skills
  can discover and load it for future work.
  Invoke when the user says "register this as context", "promote to context",
  "add this to context", "add this file to the context table", "reference this in CLAUDE.md",
  or "make this a context file".
allowed-tools: Read, Write, Edit, Bash
argument-hint: "[optional: path to the file to register as context]"
---

# Promote to Context

You are registering an existing file as context in `CLAUDE.md` so content skills can
discover and load it for future work.

## Step 1: Identify the file

If `$ARGUMENTS` contains a file path, use it. Otherwise ask:

> "Which file would you like to register as context? (Provide the file path.)"

Read the file.

## Step 2: Suggest a label

Generate a short, descriptive label (2–5 words) based on the file content. Good labels
are specific enough to distinguish this file from similar ones in the same project:

- Too generic: "Writing style"
- Good: "Brand voice – recruiting"
- Good: "Whitepaper – AI in HR (Q3 2025)"
- Good: "Audience – enterprise buyers"

Present the suggestion and ask the owner to confirm or change it:

> "Suggested label: **[label]**
> (or enter your own — no restrictions)"

## Step 3: Generate a summary

Write a one-sentence summary (10–25 words) from the file content. The summary is the key
signal skills use to decide which file to load for a given task — be specific:

- Too vague: "Writing style guidelines"
- Good: "Formal German, em-dash preferred, no exclamation marks — all corporate copy"
- Good: "Whitepaper on AI in HR published Q3 2025 — use as reference for follow-up content"
- Good: "Three enterprise buyer personas: CHROs, IT directors, and works councils"

Present the suggestion and ask the owner to confirm or change it:

> "Suggested summary: [summary]
> (or enter your own)"

## Step 4: Determine the target CLAUDE.md

Check which CLAUDE.md files exist in or above the file's location:

```bash
find . -name "CLAUDE.md" | sort
```

If there is one obvious candidate (project root or closest parent folder), use it without asking.

If multiple exist at meaningfully different levels (e.g., project root vs. campaign subfolder),
ask:

> "Which CLAUDE.md should this be registered in?
> <list paths with brief location notes>"

## Step 5: Add the row

Show the proposed row before writing:

> "I'll add this row to `[CLAUDE.md path]`:
>
> | [label] | [file path] | [summary] |
>
> Proceed? (yes / edit / skip)"

- **Yes**: add the row to the `## Context files` table in the target CLAUDE.md.
  If the `## Context files` section does not exist yet, create it first:

  ```markdown
  ## Context files

  Skills read all registered files and load what's relevant for each task.

  | Label | File | Summary |
  | ----- | ---- | ------- |
  ```

  Then append the row. Confirm: "✓ Registered `[file path]` as context in `[CLAUDE.md path]`."

- **Edit**: ask what to change, update label/summary/path, show again.

- **Skip**: say "Registration cancelled — nothing was written." and exit.
