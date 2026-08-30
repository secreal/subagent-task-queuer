---
name: subagent-start
description: Dispatch a task to the main agent or a new subagent, defaulting to compound-engineering:lfg when no companion skill is named.
---

# Subagent Start

Use this skill as a dispatcher. Invoke it with a task and optionally a companion
skill, for example `$subagent-start $ce-debug "investigate the failing test"`.

If no companion skill is named, use the exact available skill
`compound-engineering:lfg` by default. Do not silently substitute `$ce-work`.

## Invocation

A valid invocation must include a concrete task; the companion skill is optional:

```text
$subagent-start [<companion-skill>] <task description>
```

Examples: `$subagent-start "ship CSV export in reports/"` (defaults to
`compound-engineering:lfg`) and `$subagent-start $ce-debug "find the regression"`.
If the task is missing, ask for it and do not start local implementation or pretend
that a subagent was created.

Use `--force-subagent` after the optional companion skill only when the user
explicitly wants a new subagent despite the main path being idle.

## Start contract

- Read the repository root `AGENTS.md`, continuity notes, and the companion skill
  before planning or dispatching.
- Inventory repositories, worktrees, branches, and existing dirty files. Preserve all
  unrelated WIP; never reset, checkout, or delete it destructively.
- Identify one main critical-path task and only independent, bounded side tasks. Do
  not delegate the immediate blocker that the main agent needs next.
- Each delegated task must have a disjoint write scope, an exact repository/path
  boundary, required checks, exclusions, and a handoff format.
- Reconcile the durable queue with live agent state before dispatching. If the main
  path is idle, make the requested task the main task and execute it locally under
  the named companion skill. Do not spawn a subagent merely to make the main agent
  wait. If the main path is active, spawn a new subagent only for an independent
  task with a disjoint write scope and an immediately executable main action.
- Unless the user explicitly authorizes shipping, a delegated subagent must not
  commit, push, merge, deploy, mutate production, send messages, or restart services.
- Record task name, agent id, nickname, branch/worktree when known, status, write
  scope, owner (`main` or `subagent`), and dependency in the visible queue.
- Persist the queue before spawning and update it after every transition. Use a
  per-repository file under `${CODEX_HOME}/subagent-queues/` (or `~/.codex/` when
  `CODEX_HOME` is unset), creating the directory if needed. Store task metadata only;
  never store credentials, tokens, or sensitive file contents.
- Set `auto_takeover: true` by default for delegated tasks and record the main-path
  gate that unlocks takeover. The queue is durable coordination state, not a
  replacement for actual agent/worktree state.

At minimum, persist one record with `task_id`, `repository_root`, `companion_skills`,
`agent_id`, `branch`, `worktree`, `write_scope`, `owner`, `dependency`, `main_gate`,
`main_gate_status`, `main_next_action`, `auto_takeover`, and `status`. For a
main-owned task, use `owner: main`, `agent_id: null`, and the current
repository/worktree. Set `main_gate_status` to `satisfied` when the main path is
complete; set it back to `pending` only when a new main gate is explicitly
established.

## Dispatch action

Once the invocation is valid, write the queue record and follow exactly one route:

1. `main` route: when the main path is idle, mark the task `main_running`, invoke
   the companion skill directly, and report that no subagent was needed. With no
   explicit companion, invoke `compound-engineering:lfg` and follow its full ordered
   pipeline.
2. `subagent` route: when the main path is active and the task is independent, persist
   `pending_spawn`, then call `multi_agent_v1__spawn_agent` immediately with a
   self-contained message naming the task, exact companion skill, repository/path
   boundary, required checks, exclusions, and handoff format. Do not merely describe
   delegation in chat.
3. `force-subagent` route: honor `--force-subagent` only when explicitly present;
   still validate scope and record why the main path is idle.

For a successful subagent start, record the returned agent id/nickname and update the
queue to `running`. If the spawn tool errors or is unavailable, report `not spawned`
with the error and keep the task out of `running`; never claim success without an
agent id. If the task overlaps the active main scope, keep it queued and explain the
dependency instead of spawning a conflicting writer.

A subagent lifecycle is: `pending_spawn` → `running` → `handoff`/`takeover` → `done`
or `blocked`. A main-owned lifecycle is: `main_running` → `done` or `blocked`.

## Main-path continuity

Before a subagent spawn, record the exact next main action, such as the next file
edit, command, or test. After `multi_agent_v1__spawn_agent` returns successfully,
the next operation must execute that `main_next_action`. Do not call `wait_agent`, do
not present a completion response, and do not ask the user for another instruction
unless that action is genuinely blocked on the new subagent's result. Refresh the
queue only after the main action has made progress. This prevents the main agent
from becoming idle immediately after delegation.

## Companion skill

The companion skill is part of the task contract, not a suggestion. Pass its exact
skill name to a delegated subagent and require it to read and follow that skill. If
the user names more than one companion skill, preserve their order and explain which
governs implementation versus review or verification.

## Waiting and automatic takeover

After a successful spawn, leave the subagent running in the background and execute
the recorded `main_next_action` immediately. Do not reflexively call `wait_agent`;
only wait when the main path is blocked on a result. When the main path is idle or
has no executable next action, work its own task directly instead of spawning another
agent.

When the main path reaches its gate, reconcile the queue and actual agent/worktree
state, then automatically invoke the takeover protocol for the highest-priority
ready delegated task (creation order breaks ties). Do not wait for the subagent's
final message; request a handoff and stop it if necessary, then take over immediately.

On a resumed turn, reconcile the durable queue before starting unrelated engineering
work. If the recorded main gate is already satisfied and `auto_takeover` is true,
perform the takeover automatically. A fully terminated session cannot execute code;
the first new session that loads this skill or the compatibility router must perform
this reconciliation without requiring the user to repeat `$subagent-takeover`.

## Queue status

Report transitions using human-readable task names:

```text
Queue:
- [main] <task> — current main task, <phase>
- [running] <task> — <agent/nickname>, <branch/worktree>, <phase>
- [queued] <task> — waiting for <dependency>
- [handoff] <task> — ready for automatic takeover or $subagent-takeover
- [done] <task> — validation and shipment status

Main path: <current task and phase>
Next gate: <what must finish before takeover or shipment>
```

Do not claim a task is done from a generic waiting state. On every later invocation,
rebuild the queue from actual agent status, worktree status, commits, and test
evidence rather than carrying stale state forward.

## Handoff required from the subagent

Require a concise report containing:

- task name and status;
- agent id, branch, and worktree;
- files changed and files intentionally untouched;
- completed units versus remaining units;
- tests/checks run and exact results;
- blockers, assumptions, and next safe action.

The start phase does not ship work unless the selected companion skill requires it.
Automatic takeover applies only to delegated tasks after the main gate is free. Use
`$subagent-takeover <task>` to override queue selection or recover a specific task
early.
