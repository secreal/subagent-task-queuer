---
name: subagent-start
description: Start a bounded delegated task under a companion skill, persist its queue state, and automatically take it over when the main path reaches its gate.
---

# Subagent Start

Use this skill to start the delegation phase. Invoke it with the companion skill that
must govern the work, for example `$subagent-start $ce-debug` or
`$subagent-start $ce-work`.

## Invocation

A valid invocation must include both a companion skill and a concrete task:

```text
$subagent-start <companion-skill> <task description>
```

For example: `$subagent-start $ce-work "implement CSV export in reports/"`.
If either the companion skill or task is missing, ask for the missing value and do
not start local implementation or pretend that a subagent was created.

## Start contract

- Read the repository root `AGENTS.md`, continuity notes, and the companion skill
  before planning or delegating.
- Inventory repositories, worktrees, branches, and existing dirty files. Preserve all
  unrelated WIP; never reset, checkout, or delete it destructively.
- Identify one main critical-path task and only independent, bounded side tasks. Do
  not delegate the immediate blocker that the main agent needs next.
- Each delegated task must have a disjoint write scope, an exact repository/path
  boundary, required checks, exclusions, and a handoff format.
- Spawn the subagent in an isolated worktree. Tell it to apply the named companion
  skill and to edit only its assigned scope.
- Unless the user explicitly authorizes shipping, the subagent must not commit,
  push, merge, deploy, mutate production, send messages, or restart services.
- Record task name, agent id, nickname, branch/worktree when known, status, write
  scope, and dependency in the visible queue.
- Persist the queue before spawning and update it after every transition. Use a
  per-repository file under `${CODEX_HOME}/subagent-queues/` (or `~/.codex/` when
  `CODEX_HOME` is unset), creating the directory if needed. Store task metadata only;
  never store credentials, tokens, or sensitive file contents.
- Set `auto_takeover: true` by default and record the main-path gate that unlocks
  takeover. The queue is durable coordination state, not a replacement for the
  actual agent/worktree state.

At minimum, persist one record with `task_id`, `repository_root`, `companion_skills`,
`agent_id`, `branch`, `worktree`, `write_scope`, `dependency`, `main_gate`,
`main_gate_status`, `auto_takeover`, and `status`. Set `main_gate_status` to
`satisfied` when the main path is complete; set it back to `pending` only when a new
main gate is explicitly established.

## Mandatory spawn

Once the invocation is valid and the bounded task contract is written, call
`multi_agent_v1__spawn_agent` immediately with a self-contained message that names
the task, companion skill, repository/path boundary, required checks, exclusions, and
handoff format. Do not merely describe the delegation in chat. A successful start
must record the returned agent id/nickname and report the queue entry. If the spawn
tool errors or is unavailable, report `not spawned` with the error and keep the task
out of `running`; never claim success without an agent id.

Persist the record as `pending_spawn` before the spawn call, then update it to
`running` only after the tool returns successfully. A minimal lifecycle is:
`pending_spawn` → `running` → `handoff`/`takeover` → `done` or `blocked`.

## Companion skill

The companion skill is part of the task contract, not a suggestion. Pass its exact
skill name to the subagent and require it to read and follow that skill. If the user
names more than one companion skill, preserve their order and explain which governs
implementation versus review or verification.

## Waiting model

After a successful spawn, leave the subagent running in the background and return to the main
critical path. Do not reflexively call `wait_agent`; only wait when the main path is
blocked on a result. The delegated task remains in the queue until the main path
reaches its requested gate.

When the main path reaches its gate, reconcile the queue and actual agent/worktree
state, then automatically invoke the takeover protocol for the highest-priority
ready task (creation order breaks ties). Do not wait for the subagent's final message;
request a handoff and stop it if necessary, then take over immediately. If no task is
ready, continue the main work and report why.

On a resumed turn, reconcile the durable queue before starting unrelated engineering
work. If the recorded main gate is already satisfied and `auto_takeover` is true,
perform the takeover automatically. A fully terminated session cannot execute code;
the first new session that loads this skill or the compatibility router must perform
this reconciliation without requiring the user to repeat `$subagent-takeover`.

## Queue status

Report transitions using human-readable task names:

```text
Queue:
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

The start phase does not ship the work. Automatic takeover is the default when the
main critical path is free. Use `$subagent-takeover <task>` to override the queue
selection or recover a specific task early.
