---
name: do-parallel
description: Trigger when the user says "do parallel" or requests maximum safe parallel execution. Dynamically decomposes implementation plans into dependency-aware hierarchical agent trees, schedules ready tasks concurrently, preserves repository understanding, isolates concurrent Git changes when supported, and integrates, resolves conflicts, and validates all work.
---

# Do Parallel — Adaptive Agent Orchestration

## Mission

Execute the user's complete task in the shortest practical time without sacrificing correctness, repository understanding, maintainability, or Git safety.

Optimize for **minimum total completion time**, not maximum agent count.

Use hierarchical delegation, dependency-aware scheduling, explicit ownership, incremental integration, and evidence-based verification.

The workflow must adapt to the actual plan, repository, agent capabilities, concurrency limits, and emerging dependencies. Never impose a fixed agent count, batch size, tree depth, or numbered-step grouping.

## 1. Inspect Before Planning

Before implementation:

1. Read applicable `CLAUDE.md`, `AGENTS.md`, and nested project instructions.
2. Inspect the Git branch, working tree, uncommitted changes, untracked files, and relevant diffs.
3. Understand the user's complete objective, constraints, original plan steps, and acceptance criteria.
4. Inspect relevant architecture, implementations, tests, package scripts, and established conventions.
5. Identify available agent-spawning, nested delegation, Agent Teams, worktree, and integration capabilities. Use only capabilities actually supported by the current environment.
6. Identify architectural decisions, shared contracts, high-conflict files, and likely bottlenecks.

Do not overwrite, discard, reset, or misattribute existing user changes.

Do not read every unrelated repository file. Inspect enough relevant code to build an accurate understanding of the requested work.

Do not begin implementation delegation until the task boundaries and important dependencies are understood.

## 2. Build and Maintain an Architecture Brief

Create a concise, evidence-based architecture brief for the requested work.

Include:

- Relevant modules and responsibilities.
- Entry points and important data flows.
- Existing services, utilities, abstractions, and patterns to reuse.
- DTOs, interfaces, database schemas, APIs, events, and other contracts.
- Dependency injection and module-registration conventions where relevant.
- Validation, error handling, logging, and testing conventions.
- Relevant cross-module dependencies.
- Shared files and likely integration risks.
- Existing user changes that must be preserved.

Base the brief on inspected source code, not filenames or assumptions alone.

Share the overall brief with manager agents and the relevant subset with each worker. Workers must inspect the surrounding implementation before making changes.

Update the brief when significant architectural decisions or contracts change.

Maintain one coherent understanding of the integrated architecture. Never assume that all agents independently possess complete or identical repository context.

## 3. Convert the Complete Plan into a Dependency Graph

Analyze the entire plan before deciding how to parallelize it.

Break the plan into meaningful, independently verifiable tasks. Preserve the original task IDs, requirements, ordering constraints, and acceptance criteria.

For every task, track:

- Task ID and original plan reference.
- Deliverable and acceptance criteria.
- Prerequisites and downstream dependents.
- Expected complexity and uncertainty.
- Relevant modules and files.
- Required interfaces or shared contracts.
- Assigned owner and workspace.
- Conflict and integration risk.
- Testing requirements.
- Current status.

Classify tasks as:

- **Foundation:** Establishes prerequisites, architecture, or shared contracts.
- **Independent:** Can proceed without unfinished implementation from other tasks.
- **Contract-dependent:** Can proceed after an interface or contract is agreed.
- **Integration:** Combines or reconciles completed work.
- **Verification:** Checks behavior, compatibility, and acceptance criteria.

Split large tasks when the resulting subtasks have clear boundaries and can be independently verified. Combine tiny tasks when delegation overhead would exceed the benefit.

Do not group tasks merely because their numbers are consecutive.

## 4. Construct an Adaptive Hierarchical Agent Tree

Use the smallest effective hierarchy that provides useful concurrency.

### Root orchestrator

The root owns:

- The complete objective and architecture brief.
- The global dependency graph and task ledger.
- Shared contracts and ownership decisions.
- Scheduling and concurrency allocation.
- Cross-workstream coordination.
- Final integration and verification.

### Manager agents

Managers own coherent subsystems or workstreams.

A manager may:

- Decompose its assigned tasks further.
- Spawn or assign leaf workers when supported.
- Maintain local dependencies and file ownership.
- Review implementation results.
- Integrate its workstream.
- Resolve local conflicts.
- Report results and remaining risks to the root.

### Leaf workers

Leaf workers own bounded implementation tasks with explicit deliverables, acceptance criteria, file ownership, dependencies, and tests.

### Test agent (single, dedicated)

Spawn exactly ONE long-lived Test agent (`tester`) at the start. It is the only agent allowed to run heavy checks (full test suites, builds, type-checks of the whole project, e2e, lint of the whole repo).

Flow:

1. A worker (or manager) finishes its task and sends a `SendMessage` to `tester` with: task ID, files changed, workspace/branch, and the checks that apply.
2. The worker then ends or is reused for the next ready task. It never waits for the tester.
3. The tester processes its inbox strictly ONE task at a time (FIFO, critical-path tasks first), so only one heavy test process runs at any moment.
4. If checks fail, the tester fixes the problem itself, limited to the files of the finished task (their writer is done, so no ownership conflict). If the fix needs a shared contract or another task's files, it reports to the owning manager/root instead of editing.
5. The tester reports pass/fail with real command output to the root. The root marks the task Integrated only after a pass, and then releases dependents.
6. When the queue is empty the tester idles and waits for the next finished task. It uses no CPU while idle.

Workers run only cheap checks on their own files (single-file test, lint/typecheck of touched files). They must not run the full suite or build.

### Determine tree depth dynamically

Use a flat group for small tasks and nested delegation for large, separable workstreams.

For example, if step 1 establishes a shared interface, steps 2–5 might be assigned to a manager who delegates them to four workers. Steps 6–19 might then be regrouped into multiple independent workstreams, each with its own manager and additional workers.

However, some of steps 6–19 may be ready immediately after step 1, while others may depend on specific outputs from steps 2–5.

This example illustrates the strategy, not a mandatory schedule.

Do not add hierarchy merely for its own sake. If recursive delegation is unsupported, simulate the hierarchy using available orchestration capabilities. Never claim to have spawned agents that were not actually started.

## 5. Schedule by Readiness, Not Fixed Waves

Maintain a live dependency graph and task ledger.

Possible task states:

- Pending
- Ready
- In progress
- Blocked
- Review
- Integrated
- Complete
- Failed

At every scheduling decision:

1. Identify every task whose prerequisites are satisfied.
2. Check file ownership, shared contracts, resource availability, and integration risk.
3. Launch all useful independent tasks that fit the available capacity.
4. Keep dependent tasks blocked until their prerequisites or approved contracts are ready.
5. Prioritize critical-path tasks and tasks that unblock many downstream tasks.
6. Continue unrelated work while blocked tasks wait.
7. Review completed tasks and immediately release newly ready tasks.
8. Recalculate concurrency when workers finish, fail, or discover new dependencies.
9. Update the task ledger and dependency graph.

**Never wait for an entire numbered range or workstream if a downstream task needs only one completed prerequisite.**

Example:

- Step 1 establishes a shared contract.
- Steps 2, 3, 8, and 12 depend only on step 1.
- Steps 4 and 5 depend on step 2.
- Step 9 depends on steps 3 and 8.

After step 1 passes review, steps 2, 3, 8, and 12 can start concurrently if their implementation boundaries are safe. Steps 4 and 5 can start as soon as step 2 provides their required outputs. Step 9 must wait for both steps 3 and 8.

Do not force this graph into sequential batches.

## 6. Maximize Useful Concurrency

Choose concurrency according to:

- Number of ready, independent tasks.
- Expected task duration and complexity.
- Critical-path dependencies.
- Available agent and runtime capacity.
- Context and token consumption.
- Cost of spawning and coordinating agents.
- File overlap and merge-conflict risk.
- Manager review and integration capacity.

Prefer more parallelism when tasks are independent, substantial, and isolated.

Prefer fewer workers when tasks are tiny, tightly coupled, or dependent on rapidly changing shared decisions.

Avoid both extremes:

- One giant agent handling everything sequentially.
- A large number of agents making overlapping edits with no effective coordination.

A manager must not delegate every subtask automatically. It should identify which tasks benefit from independent execution and retain tightly coupled work when coordination would be faster.

If the environment has concurrency limits, keep ready tasks queued and reuse capacity as workers finish.

**CPU budget (default):** run at most 3-4 writing workers at once, plus the single Test agent. Never launch more just because more tasks are ready; queue them. Raise the cap only if the user asks. Never run installs, builds, or full test suites in parallel; only the Test agent runs them, one at a time.

**Manager type and depth:** spawn managers as `general-purpose` agents (some agent types, such as `Explore` and `Plan`, have no Agent tool and cannot spawn children). Go at most 2 levels below the root (root → manager → worker). Workers never spawn.

**Real nesting check:** before promising managers that spawn workers, confirm that a spawned agent can itself call the Agent tool. If it cannot, the root spawns all workers directly and managers act as reviewers/integrators only. State this plainly in the execution outline instead of claiming a deeper tree.

## 7. Establish Contracts Before Dependent Work

Identify shared contracts before parallel implementation, including:

- DTOs, types, and interfaces.
- Database schemas, models, indexes, and migrations.
- Public API request and response formats.
- Event payloads and message formats.
- Shared services and utilities.
- Configuration, dependency injection, and module exports.
- Generated artifacts and build interfaces.

Assign a designated owner for high-conflict shared resources.

When multiple workers depend on a common interface:

1. Agree on the contract.
2. Record it in the task brief or architecture notes.
3. Communicate it to every affected worker.
4. Allow independent implementations to proceed against it.
5. Revalidate affected work if the contract changes.

Do not let workers independently invent incompatible versions of a shared interface.

If a contract can be established quickly, establish it first rather than forcing all dependent work to wait for a complete implementation.

## 8. Enforce Explicit File Ownership

Every implementation task must have a clear write boundary.

Give each worker:

- Overall objective and relevant architecture.
- Exact task and acceptance criteria.
- Relevant existing implementations.
- Approved contracts and dependencies.
- Files or modules it owns.
- Files or areas it must not modify.
- Required tests and expected report.

Rules:

- Prefer one writer per shared file.
- Assign one owner for global configuration, package manifests, lockfiles, central exports, shared schemas, and similar high-conflict resources.
- If multiple tasks require edits to the same file, coordinate a single owner, serialize those edits, or integrate proposals through the manager.
- Avoid concurrent dependency installation and shared generated-file updates unless explicitly coordinated.
- Search for existing functionality before introducing new helpers, services, abstractions, or dependencies.
- Avoid unrelated refactoring and scope expansion.

If a worker discovers a necessary change outside its ownership boundary, it must report the need and coordinate with the owner rather than silently modifying the shared resource.

## 9. Isolate Concurrent Git Changes

Use isolated worktrees or dedicated branches for concurrent writers when the environment supports them and isolation materially reduces conflict risk.

Before creating workspaces:

1. Record the initial Git state.
2. Identify existing branches, worktrees, uncommitted changes, and untracked files.
3. Decide which tasks need writable isolation.
4. Assign unique workspaces and clear ownership.

When using worktrees or branches:

- Use descriptive names tied to task IDs.
- Do not assume two worktrees can check out the same branch simultaneously.
- Require meaningful commits when branch-based integration needs them.
- Track which task owns each branch or workspace.
- Integrate changes through the supported workflow.
- Never force-push, reset, clean, overwrite, or discard user work to simplify merging.
- Do not merge into protected or production branches without authorization.

If agents must share a working directory, coordinate their writable files explicitly and avoid simultaneous edits to shared files.

Do not create worktrees or branches for read-only analysis tasks unless there is a clear reason.

Isolation reduces contention but does not eliminate semantic conflicts.

## 10. Require Evidence-Based Completion Reports

Every worker must report:

1. Task IDs completed.
2. Summary of implemented behavior.
3. Files changed.
4. Existing patterns or functionality reused.
5. Interfaces, schemas, or contracts affected.
6. Workspace, branch, and commit references where applicable.
7. Cheap checks actually run, then confirmation that the task was queued to `tester` via SendMessage.
8. Actual results of those checks (full-suite results come from the Test agent).
9. Assumptions and known risks.
10. Remaining work or blockers.

Managers must review worker results before marking their workstream integrated.

The root must inspect the actual changes rather than relying solely on completion summaries.

A worker saying "done" does not prove that its acceptance criteria have been met.

## 11. Integrate Continuously and Bottom-Up

Do not postpone all integration until every worker has finished.

### Leaf-level integration

Review each worker's changes against its acceptance criteria. Check relevant tests and ownership boundaries.

### Manager-level integration

Managers integrate their workers' completed changes, reconcile local interfaces, detect duplication, and run subsystem checks.

A manager must not report its workstream as integrated while known required conflicts remain unresolved.

### Root-level integration

The root integrates manager workstreams, validates cross-module compatibility, and runs broader verification.

At every integration gate:

1. Inspect the actual diff.
2. Verify the intended behavior.
3. Check textual merge conflicts.
4. Check semantic conflicts even if Git merges cleanly.
5. Reconcile interface and schema assumptions.
6. Remove duplicate or incompatible implementations.
7. Run relevant tests and checks.
8. Update the task ledger.
9. Release downstream tasks only when their prerequisites are validated.

Integrate in dependency order, but allow unrelated work to continue.

Never silently discard another worker's valid changes to resolve a conflict.

## 12. Resolve Conflicts Deliberately

Treat textual conflicts and semantic conflicts as separate problems.

A textual conflict means the changes overlap incompatibly at the file or line level.

A semantic conflict means the changes may merge cleanly but disagree on behavior, contracts, architecture, naming, error handling, or runtime assumptions.

For every meaningful conflict:

1. Identify the conflicting changes and affected tasks.
2. Determine the intended behavior from the original plan and established repository conventions.
3. Compare implementations and tests.
4. Select or combine the best compatible solution.
5. Update affected tests and contracts.
6. Run relevant verification.
7. Assign a focused correction to a worker if that is the most efficient option.
8. Notify downstream owners when their assumptions have changed.

Do not blindly accept one side, combine both versions without analysis, or leave contradictory implementations in the repository.

Ask the user only when a material product or architectural decision cannot be inferred safely.

## 13. Verify at Multiple Levels

Use the appropriate levels of verification:

- **Worker:** Cheap checks only (touched files); then hand off to the Test agent.
- **Test agent:** Full tests, build, and type-check per finished task, serialized; fixes failures within that task's files.
- **Manager:** Integrated subsystem behavior and shared interfaces.
- **Root:** Cross-module compatibility and complete acceptance criteria.
- **Final:** Relevant broader tests, builds, linting, type checks, and end-to-end checks when available.

Review the integrated diff for:

- Duplicate services, helpers, or business logic.
- Inconsistent DTOs, APIs, schemas, or interfaces.
- Missing validation and error handling.
- Incompatible assumptions across modules.
- Unnecessary abstractions and dependencies.
- Accidental unrelated changes.
- Generated-file or formatting noise.
- Missing tests and incomplete integration.
- Untracked files or unsafe cleanup.

A successful merge is not proof of correctness.

Never claim a test passed unless it actually ran successfully. Clearly report checks that were unavailable, skipped, or failed.

## 14. Recover and Adapt

When a worker fails, stalls, or discovers an unexpected dependency:

1. Determine the cause and identify any valid work already completed.
2. Decide whether to retry, split, reassign, or implement the task directly.
3. Preserve valid changes.
4. Update task ownership and dependencies.
5. Recalculate available concurrency.
6. Verify recovered work before integration.

If an interface changes, notify affected workers and update their task briefs.

If two tasks turn out to be more tightly coupled than expected, coordinate or serialize the conflicting part while allowing unrelated tasks to continue.

If a task turns out to be independent, release it without waiting for unrelated work.

Use established project conventions for routine implementation decisions. Ask the user only when a genuine decision affects scope, architecture, data safety, or irreversible actions.

Do not perform destructive migrations, production deployments, or irreversible external actions without appropriate authorization.

## 14a. Persist State and Resume After Pause or Limit

Chat context and running agents are lost on pause, crash, or usage limit. The file is the source of truth.

State directory: `<repo>/.do-parallel/` (if Git, add it to `.git/info/exclude`, never to the user's `.gitignore`). Files:

- `state.json`: plan summary, task ledger (id, status, deps, owner, files, workspace/branch/commit, attempts, last result), approved contracts, port map, tester queue, timestamp.
- `brief.md`: the architecture brief. Workers receive its path, not a pasted copy.
- `log.md`: one line per event (task started, handed to tester, passed, failed, integrated).

Rules:

1. Write-ahead: update `state.json` BEFORE launching a task (status In progress) and immediately after every transition. Write to a temp file, then `mv` it over `state.json`, so a crash never leaves it half-written. Only the root writes `state.json`; workers report via message and the root records it.
2. Update `brief.md` whenever a contract or architecture decision changes.
3. Resume: if `.do-parallel/state.json` exists with unfinished tasks (or the user says "do parallel resume"/"continue"), do NOT re-plan from scratch:
   - Read the state files, then compare them with reality (`git status`, branches/worktrees, commits). Trust Git over the ledger when they disagree.
   - Tasks marked In progress or Review whose agents no longer exist: inspect the partial work, keep what is valid, mark Ready (or Needs-recheck) and re-dispatch.
   - Tasks passed by the tester stay Integrated. Never redo them.
   - Respawn the tester and rebuild its queue from the ledger.
4. On a limit warning or a user pause request: stop launching new tasks, let in-flight workers finish if possible, flush `state.json`, and tell the user the exact resume command.
5. When everything is Complete, mark `state.json` as `"status":"complete"`. Delete `.do-parallel/` only if the user agrees.

## 14b. Test Lane: Speed, Ports, Low CPU

Applies to the single Test agent.

- **Tiered checks:**
  - Per finished task: only tests related to the changed files (e.g. `--findRelatedTests`/changed-files mode) plus an incremental type-check/lint of touched files.
  - Full suite, full build, and e2e only at integration gates (a manager's workstream is merged, or the root's final check). Docs and config-only tasks skip tests.
- **Throttle:** run test and build processes with `nice -n 10` and a worker cap (e.g. `--maxWorkers=50%`). No watch mode. Reuse incremental/build caches. Never start dev servers or daemons that outlive the check.
- **Ports and shared resources (DBs, emulators):**
  - Never assume a default port. Prefer port 0/auto-assign. Otherwise take a free port from a per-task range recorded in `state.json`'s port map (e.g. base + task index).
  - If a port or resource is busy, do not sit and wait. Pick another free port, or defer that task and run the next queued task whose resources are free. Retry the deferred one later and set a timeout. After the timeout, report Blocked to the root.
  - Hold `.do-parallel/test.lock` while running. Never kill a process the tester did not start. Start only what it needs and stop it when the check ends.
- **Queue order:** critical-path tasks first, then FIFO. Never block workers on the tester. Workers keep going while results arrive later.
- **Failed check:** the tester fixes within that task's files and re-runs only the failed check. It escalates to the root after 2 failed attempts.

## 15. Keep the Repository Clean and Understandable

The final implementation should feel like one coherent engineering effort, not a collection of disconnected agent contributions.

Prefer existing project patterns over new abstractions.

Avoid:

- Duplicate utilities and services.
- Multiple competing implementations of the same behavior.
- Unnecessary framework changes.
- Unrelated refactors.
- Unnecessary dependencies.
- Temporary coordination files in production directories.
- Redundant comments and documentation.
- Unreviewed generated files.
- Agent-specific scaffolding that does not belong in the project.

After integration, remove only temporary resources created by this workflow and only after confirming they are no longer needed.

Preserve pre-existing branches, worktrees, uncommitted changes, and untracked user files.

Keep the final diff focused on the requested objective.

## 16. Present the Execution Outline

Before substantial implementation, present a concise outline containing:

- **Foundation:** What must happen first and why.
- **Ready tasks:** Which tasks can start concurrently immediately.
- **Nested delegation:** Which managers can delegate further and why.
- **Dependency releases:** Which outputs unlock subsequent tasks.
- **Integration gates:** Where review, merging, and tests occur.
- **Critical path:** The longest dependency chain and its bottlenecks.
- **Conflict controls:** Shared-file ownership and agreed contracts.

Show logical dependencies rather than forcing tasks into fixed numbered batches.

If the user has already authorized execution, proceed without redundant approval requests.

Keep the outline concise and actionable. Update it when important dependencies change.

## 17. Final Reconciliation and Report

Before declaring completion:

1. Reconcile every original task against the task ledger.
2. Confirm that required changes are integrated.
3. Verify shared contracts and cross-module behavior.
4. Run available final checks.
5. Inspect the final combined diff and Git status.
6. Confirm that cleanup preserved existing user work.
7. Identify incomplete acceptance criteria and unresolved risks.

Report:

- What was implemented.
- How the work was divided and parallelized.
- Which tasks are complete, incomplete, or blocked.
- Important integration decisions.
- Relevant branches or commits, where applicable.
- Tests actually run and their results.
- Remaining risks and follow-up work.

Do not claim the overall task is complete while required acceptance criteria remain unmet.

## Trigger Behavior

When the user says **"do parallel"**, apply this workflow to the current task or plan. First check for an unfinished `.do-parallel/state.json` and resume per section 14a instead of starting over.

Model rule: never pass a `model` override when spawning agents. Every agent, nested or not, inherits the root's model, which the user chooses.

Speed rules: launch all read-only research in one parallel batch at the start. Give workers the path to `.do-parallel/brief.md` plus only their task, not the whole plan.

Inspect the repository and available orchestration capabilities, understand the architecture, build the dependency graph, choose an adaptive agent hierarchy, schedule maximum useful concurrency, integrate incrementally, resolve conflicts, and verify the final result.

Do not require the user to specify the number of agents, which steps should run concurrently, or how to organize the hierarchy. Those are orchestration responsibilities.

If the user is only discussing parallelization rather than requesting implementation, explain the strategy without launching agents.

## Non-Negotiable Principle

**Parallelize independent work aggressively. Establish shared decisions explicitly. Integrate continuously. Preserve repository coherence. Verify the result.**
