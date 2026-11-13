# Using do-parallel with GitHub Copilot

GitHub Copilot supports the same `SKILL.md` format, so the skill file works unchanged.
Only the install location and agent tooling differ.

## Install

Copilot discovers skills from these folders:

- Project: `.github/skills/`, `.claude/skills/`, or `.agents/skills/` in the repo
- Personal (Copilot cloud agent and Copilot CLI): `~/.copilot/skills/` or `~/.claude/skills/`

```bash
# project-level (shared with your team via the repo)
git clone https://github.com/nadernmds/do-parallel.git .github/skills/do-parallel

# personal, for Copilot CLI / cloud agent
git clone https://github.com/nadernmds/do-parallel.git ~/.copilot/skills/do-parallel
```

Remove the cloned `.git` folder if you commit the skill into your own repo
(`rm -rf .github/skills/do-parallel/.git`).

Skills work in Copilot agent mode in VS Code, the Copilot CLI, and the Copilot
cloud agent. Support in the stable VS Code release may depend on your version, so
check your release notes.

## Use

In agent mode, ask for the work and say **"do parallel"**. Copilot activates the
skill when it matches your request.

## What works the same

- Dependency graph, task ledger, file ownership, and integration rules.
- Saved state in `.do-parallel/` and resume ("do parallel resume").
- Throttled tests, tiered checks, and busy-port handling.

## What may differ (check on your setup)

The skill text refers to the Claude Code `Agent` tool, `SendMessage`, and nested
`general-purpose` agents. Copilot's agent tooling is different:

- **Nesting is not guaranteed.** The skill's "real nesting check" falls back to a
  flat tree if agents cannot spawn their own. Expect a flat tree unless your
  Copilot setup supports sub-agents.
- **Tester handoff:** if agents cannot message each other, the main agent runs the
  tester step itself, one task at a time, from the queue in `.do-parallel/state.json`.
  The CPU benefit (one heavy test process at a time) stays.
- **Cloud agent:** it runs in its own environment and opens a pull request, so
  local ports are not shared. Saved state in `.do-parallel/` should not be
  committed (the skill adds it to `.git/info/exclude`; review the PR diff).
- **Model:** the skill never switches models. Copilot uses the model you selected.

## Tips

- Use agent mode, not ask mode.
- Commit or stash your own changes before starting.
- Keep one writer per file, as the skill's ownership rules require.
