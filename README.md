# claude-skills

A collection of Claude Code skills for session management and workflow.

## Skills

### [wrap-up](./wrap-up/SKILL.md)

End a Claude Code session by writing a structured `.md` file that serves two readers: a human summary (what happened, what's next) and a dense token-optimized section for Claude to resume seamlessly.

**Invoke:** `/wrap-up`

Supports optional modifiers: `--compact` (runs `/compact` after saving), `--interactive` (Q&A before writing).

### [pick-up](./pick-up/SKILL.md)

Resume a Claude Code session from a previously written wrap-up file. Auto-discovers the most recent wrap-up in `./wrap-ups/`, verifies environment state, re-populates todos, and stages the first action without executing it.

**Invoke:** `/pick-up` or `/pick-up path/to/wrap-up.md`

---

## Installation

Copy the skill folder(s) into `~/.claude/skills/`:

```bash
# wrap-up
cp -r wrap-up ~/.claude/skills/wrap-up

# pick-up
cp -r pick-up ~/.claude/skills/pick-up
```

Claude Code auto-discovers skills in `~/.claude/skills/` — no further config needed.

---

## Usage

```
# At the end of a session
/wrap-up

# At the start of a new session in the same project
/pick-up
```
