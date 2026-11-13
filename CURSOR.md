# Using do-parallel in Cursor

Cursor supports the same `SKILL.md` format, so the skill file itself works unchanged.
What differs is how agents are spawned, so read the limits below.

## Install

Cursor discovers skills from these folders (project or user level):

- `.cursor/skills/` or `~/.cursor/skills/`
- `.claude/skills/` or `~/.claude/skills/` (Cursor also reads these)

```bash
# user-level, available in every project
git clone https://github.com/nadernmds/do-parallel.git ~/.cursor/skills/do-parallel

# or project-level
git clone https://github.com/nadernmds/do-parallel.git .cursor/skills/do-parallel
```

If you already installed it for Claude Code in `~/.claude/skills/do-parallel`, Cursor
may pick it up from there; no second copy is needed.

## Use

In Agent chat, type `/do-parallel` (or say **"do parallel"**) with your plan.
If skills are not listed, check Cursor's settings for Agent Skills (older builds
had them behind a beta toggle).

## What works the same

- Dependency graph, task ledger, file ownership, and integration rules.
- Saved state in `.do-parallel/` and resume ("do parallel resume").
- Throttled tests, tiered checks, and busy-port handling.

## What may differ (check on your Cursor version)

The skill text mentions the Claude Code `Agent` tool, `SendMessage`, and nested
`general-purpose` agents. Cursor's agent tooling is different:

- **Nesting is not guaranteed.** The skill's "real nesting check" tells the agent
  to fall back to a flat tree (root + workers, no managers) if sub-agents cannot
  spawn their own. Expect a flat tree unless your Cursor version supports it.
- **The tester handoff** uses messages between agents. If Cursor cannot message
  between agents, the root runs the tester step itself, one task at a time, using
  the same queue in `.do-parallel/state.json`. The CPU benefit (one heavy test
  process at a time) stays.
- **Worktrees:** Cursor can run parallel agents in separate Git worktrees. This
  fits the skill's isolation rules; keep one writer per file.
- **Model:** Cursor uses the model you picked in the chat. The skill never
  switches models.

## Tips

- Run it in Agent mode, not Ask mode.
- Keep the agent's terminal limited to one test process at a time to avoid port
  clashes.
- Commit or stash your own changes before starting.
