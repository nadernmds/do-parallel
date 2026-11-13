# do-parallel

A [Claude Code](https://claude.com/claude-code) skill that runs a plan with the
most useful safe parallelism and low CPU use.

- **Hierarchical agent tree**: root → managers → workers (max 2 levels below the root), scheduled by dependency readiness, not fixed waves.
- **One serialized test agent**: finished tasks are queued to a single tester that runs checks one at a time, so only one heavy test process runs at once. Busy ports/resources are skipped and retried, not waited on.
- **CPU budget**: 3-4 writing workers by default, throttled test processes, tiered checks (related tests per task, full suite only at integration gates).
- **Resumable**: state is saved to `.do-parallel/` in your repo, so you can continue after a pause or usage limit ("do parallel resume").
- **Git-safe**: explicit file ownership, optional worktrees, never discards your changes.
- **No model switching**: every agent inherits the model of your session.

## Install

```bash
git clone https://github.com/nadernmds/do-parallel.git ~/.claude/skills/do-parallel
```

Then, in Claude Code, give it a plan and say **"do parallel"**.

## Requirements

- Claude Code with the Agent tool. Managers that spawn workers need a
  `general-purpose` agent that can itself spawn agents; the skill checks this at
  run time and falls back to a flat tree if nesting is unavailable.
- Git is optional but recommended (worktree isolation).

## License

Apache-2.0
