---
name: wrap-up
description: Summarize the current Claude Code session into a durable wrap-up .md file with sections for humans and a token-optimized section for Claude to resume seamlessly. Use when the user says "wrap up", "wrap up the session", "summarize and save", "end of session", "close out this session", or invokes /wrap-up. Captures purpose, completed work, open items, next steps, active todos, git state, and a copy-paste resume prompt. Supports optional context compression and interactive Q&A modifiers.
user_invocable: true
---

# wrap-up

End a Claude Code session by writing a structured wrap-up file. The file serves two readers from one document: a human section (what happened, what's next) and a Claude section (dense, token-optimized context for seamless resume).

Pairs with the `pick-up` skill, which ingests the written wrap-up in a future session.

## Step 1 — Decide the mode

Parse the invocation text for modifiers:

- `--compact`, "compact", "compress", "and compress" → **compact mode**
- `--interactive`, "interactive", "ask me", "questions first" → **interactive mode**
- Modes compose: both can be active at once.

If **no modifier** was specified, ask the user via `AskUserQuestion`:

- Question: "How should I wrap up?"
- Options:
  1. "Just create the wrap-up file" (default)
  2. "Wrap up and compress context" (also runs `/compact` after saving)
  3. "Wrap up with questions first" (interactive Q&A before writing)
  4. "Wrap up with questions, then compress" (both)

Respect the user's answer.

## Step 2 — Interactive Q&A (only if interactive mode is active)

Ask 2–4 focused questions via `AskUserQuestion` to surface non-obvious context that likely isn't in the chat transcript. Good targets:

- Blockers discussed verbally or implied but never named
- Decisions that were made but not spelled out
- Stakeholders / deadlines / external dependencies
- Scope changes the user hasn't stated explicitly

Keep questions multiple-choice where possible. Fold the answers into the wrap-up content in Step 4.

Skip this step entirely if interactive mode is not active.

## Step 3 — Gather environment state

Always run (regardless of mode):

```bash
git rev-parse --is-inside-work-tree 2>/dev/null
```

If the above prints `true`, gather git state:

```bash
git branch --show-current
git status --short | wc -l
git log -1 --oneline
```

Also read the **active `TodoWrite` list** from the current session context — Claude already has this; no tool call needed. If there is no active todo list, note "none".

Determine the **project memory dir**: encode the current working directory by replacing `/` with `-` and prefixing with `-`. Example: `/Users/alice/Projects/foo` → `~/.claude/projects/-Users-alice-Projects-foo/memory/`. The directory may or may not exist; do not create it yet.

## Step 4 — Write the wrap-up file

**Path:** `./wrap-ups/<YYYY-MM-DD>-<HHMM>-<slug>.md`

- `<slug>` is a 2–4 word kebab-case summary of the session topic (e.g., `auth-bug-fix`, `wrap-up-skill-design`). Derive from the session's actual work, not generic phrases.
- Create `./wrap-ups/` if missing.
- If in a git repo, ensure `wrap-ups/` is on a line by itself in `./.gitignore`. Append it if absent. Do not commit. If `.gitignore` doesn't exist, create it with a single `wrap-ups/` line.

**Use this exact template:**

````markdown
# Session Wrap-up — <topic>
_<YYYY-MM-DD HH:MM> · <working-dir> · branch: <branch-or-"n/a">_

## For People

### Purpose
<1–3 sentences on why this session happened and what the user set out to accomplish>

### Completed
- <deliverable>
- <deliverable>

### Open / In Progress
- <item> — <state / blocker>

### Next Steps
1. <specific, actionable next step>
2. <…>

### Environment Snapshot
- Branch: <name> · Uncommitted: <N> files · Last commit: <hash> <subject>
- Active todos: <N> (verbatim list in the Claude section below)

---

## For Claude (resume context — token-optimized)

<!-- Dense, abbreviated. No prose flourishes. Prioritize file paths, function names, invariants, current state. -->

- **Goal:** <one line>
- **Stack/tools:** <list>
- **Key files:**
  - `path/to/file.ext:line` — <role>
- **State:** <what's done, what's next, shortest form>
- **Invariants/constraints:** <things that must hold>
- **Active todos:**
  - [ ] <todo 1>
  - [ ] <todo 2>
- **Open threads:** <unresolved questions / pending decisions>
- **Do NOT:** <known pitfalls, approaches ruled out>

### Resume instructions
Read this file. Verify state (git branch, file existence). First action: `<specific command or edit>`.

---

## Copy-paste prompt for new session

```
Resume this session. Read ./wrap-ups/<filename>.md — especially the "For Claude" section. Verify state, then execute the Resume instructions. First action: <specific thing>.
```
````

**Writing guidance for the Claude section:**

- Prefer lists over prose.
- Include `file:line` references wherever a specific location matters.
- Abbreviate: "fn" over "function", "impl" over "implementation", "w/" over "with".
- Do not restate content already in the "For People" section.
- Target: a fresh Claude reads this once and has operating context for the next action.

## Step 5 — Memory sync (default behavior)

Review the wrap-up content for durable facts worth persisting to the auto-memory system. Good candidates:

- User role, responsibilities, domain expertise
- Project goals or constraints that outlive this session
- Explicit feedback ("don't do X because…", "prefer Y approach")
- Reference pointers (external dashboards, tracking systems, docs)

For each candidate:

1. Check `<project-memory-dir>/MEMORY.md` — does a matching memory already exist?
2. If yes and accurate, skip.
3. If yes but stale, update the existing memory file.
4. If no, create `<project-memory-dir>/<descriptive-name>.md` with the standard frontmatter (`name`, `description`, `type`) and add a one-line entry to `MEMORY.md`.

Follow the auto-memory protocol already documented in the session system prompt — do not duplicate, keep `MEMORY.md` as a concise index, and include **Why:** and **How to apply:** lines for feedback/project memories.

## Step 6 — Compression (only if compact mode is active)

After the file is written and memory is synced, tell the user:

> Wrap-up saved to `<path>`. Memory synced. Running `/compact` now.

Then invoke Claude Code's native `/compact` command.

## Step 7 — Report

At the end, in one short message, tell the user:

- Where the file was written (clickable markdown link).
- How many memory entries were added or updated (if any).
- Whether compaction ran.
- The exact copy-paste prompt they can use in a new session (or remind them `/pick-up` also works).

---

## Optional: Auto-invoke on session end (Stop hook)

Not installed by default. If the user wants a wrap-up every time a session ends, they can configure a `Stop` hook in `~/.claude/settings.json`. Direct them to the `update-config` skill, or show them this reminder-style hook as a starting point:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "echo 'Session ending — run /wrap-up to save context' >&2" }
        ]
      }
    ]
  }
}
```

Caveats to flag if asked:

- `Stop` hooks fire on every session end, including short ones where a wrap-up adds no value.
- The example above only prints a reminder. Fully auto-running the skill from a hook requires a deeper harness integration (e.g., a hook that invokes `claude -p "/wrap-up"` in a follow-up session) which depends on the user's setup. Recommend starting with the reminder and upgrading only if needed.
