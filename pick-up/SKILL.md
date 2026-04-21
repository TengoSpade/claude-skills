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

Also check the header area (first 5 lines of the file) for a `↳ Continues from:` line. If present, capture the bare predecessor filename from the href of the markdown link — the text inside the parentheses, after stripping the `./wrap-ups/` prefix (e.g. `2026-04-15-auth-setup.md`).

Also skim the "For People" section for context on why the work exists, but do not rely on it for operating details.

## Step 3 — Verify environment

Spot-check that the state described in the file still holds:

1. **Git branch.** If the file's header lists a branch, run `git branch --show-current` and compare. On mismatch, tell the user — do not auto-switch branches.
2. **Staleness check (git repos only).** First verify you're in a git repo:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null
```

If the above returns nothing or exits non-zero, skip the rest of this step entirely.

Otherwise, extract the full timestamp from the wrap-up's header line (format: `_2026-04-20 22:30 · ...`). Capture both the date (`YYYY-MM-DD`) and time (`HH:MM`). Then run:

```bash
WRAP_DATETIME="<YYYY-MM-DD HH:MM:00 from header>"
COMMIT_COUNT=$(git log --oneline --after="$WRAP_DATETIME" 2>/dev/null | wc -l | tr -d ' ')
echo "commits since wrap-up: $COMMIT_COUNT"
```

Incorporate the result into the announcement (Step 6):
- 0 commits → omit (don't clutter the announcement)
- 1–10 commits → "N commits since this wrap-up was written."
- 11+ commits → "⚠ N commits since this wrap-up — context may be stale."

Never block resume based on this. Informational only.

3. **Key files exist.** For each `path/to/file.ext:line` reference in the Claude section, confirm the file exists. Use `Glob` or direct `Read` checks sparingly; don't waste turns on every path. Warn on any missing file.
4. **Uncommitted files count.** If the file lists a count, run `git status --short | wc -l` and note drift in the announcement (don't block on it — the user may have committed or made new changes in the meantime).

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

> Loaded `wrap-ups/2026-04-20-1800-auth-bug.md` _(chained from `2026-04-15-auth-setup.md`)_. Branch matches, 3 todos restored, all key files present. 4 commits since wrap-up. First action: edit `src/auth.ts:42` to wire the new token check. Ready when you are — say "go" to proceed, or "load chain" to read the predecessor for full project lineage.

If a chain predecessor was detected: include it in the announcement as shown. If the user says "load chain" instead of "go": read the predecessor wrap-up file's `## For Claude` section and surface it as additional context before beginning work. If the predecessor file is not found, tell the user ('Predecessor wrap-up not found — proceeding without chain context') and continue with the current wrap-up only. If no chain predecessor was found: omit the chain portion of the announcement entirely.

**Do not execute the first action automatically.** The user may want to adjust, amend the todos, or bail. Wait for an explicit go-ahead.
