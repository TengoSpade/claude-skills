---
name: pick-up
description: Resume a Claude Code session from a previously written wrap-up file. Use when the user says "pick up where I left off", "resume session", "resume from wrap-up", "continue last session", "pick up", or invokes /pick-up. Finds the most recent wrap-up in ./wrap-ups/ (or accepts a specific path as argument), reads the "For Claude" section, verifies environment state still holds, re-populates TodoWrite, and stages the first action without executing it.
user_invocable: true
---

# pick-up

Ingest a wrap-up file written by the `wrap-up` skill and resume the session. Counterpart to `wrap-up`.

## Step 1 — Locate the wrap-up file

If the user passed a path or filename as an argument, use it. Accept both:

- Absolute paths and paths relative to the current dir
- Bare filenames (assume `./wrap-ups/<filename>`)

If no argument was passed, auto-discover:

1. List `./wrap-ups/*.md`.
2. If none exist, tell the user: "No wrap-ups found in `./wrap-ups/`. Pass a path if the wrap-up lives elsewhere." Stop.
3. If one or more exist, pick the most recent by the `YYYY-MM-DD-HHMM` prefix in the filename. Fall back to `mtime` if the naming is inconsistent.

Announce which file was selected before proceeding.

## Step 2 — Read and parse

Read the full file with the `Read` tool. Focus extraction on the `## For Claude` section — that's the operating context. From it, capture:

- Goal
- Stack/tools
- Key files (with `:line` references)
- State summary
- Invariants / constraints
- Active todos
- Open threads
- "Do NOT" list
- Resume instructions (especially the first action)

Also skim the "For People" section for context on why the work exists, but do not rely on it for operating details.

## Step 3 — Verify environment

Spot-check that the state described in the file still holds:

1. **Git branch.** If the file's header lists a branch, run `git branch --show-current` and compare. On mismatch, tell the user — do not auto-switch branches.
2. **Key files exist.** For each `path/to/file.ext:line` reference in the Claude section, confirm the file exists. Use `Glob` or direct `Read` checks sparingly; don't waste turns on every path. Warn on any missing file.
3. **Uncommitted files count.** If the file lists a count, run `git status --short | wc -l` and note drift in the announcement (don't block on it — the user may have committed or made new changes in the meantime).

## Step 4 — Re-populate TodoWrite

If the Claude section has an "Active todos" list, call `TodoWrite` to set the session's todo tracker to match. Preserve the item text verbatim. Mark all as `pending` unless the wrap-up indicates a different status.

## Step 5 — Consult memory

Read `<project-memory-dir>/MEMORY.md` (if it exists) — same encoding as in `wrap-up`: replace `/` with `-` in the working dir and prefix with `-`. Surface any entries that bear on the work described in the wrap-up (for example, user role, feedback rules, project constraints). Fold those into the announcement so the user sees the full operating context.

## Step 6 — Announce and stage (do not execute)

Report to the user in ≤4 sentences:

- Which wrap-up file was loaded.
- Headline state verification (git branch match, missing files, todo count).
- Any relevant memory facts found.
- The first action from Resume instructions, verbatim.

End the report by explicitly waiting for user confirmation. Example phrasing:

> Loaded `wrap-ups/2026-04-20-1800-auth-bug.md`. Branch matches, 3 todos restored, all key files present. First action: edit `src/auth.ts:42` to wire the new token check. Ready when you are — say "go" to proceed.

**Do not execute the first action automatically.** The user may want to adjust, amend the todos, or bail. Wait for an explicit go-ahead.
