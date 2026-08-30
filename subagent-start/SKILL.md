---
name: subagent-start
description: Start a bounded delegated task under a companion skill, register it in the queue, and leave it running while the main critical path continues.
---

# Subagent Start

Use this skill to start the delegation phase. Invoke it with the companion skill that
must govern the work, for example `$subagent-start $ce-debug` or
`$subagent-start $ce-work`.

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

## Companion skill

The companion skill is part of the task contract, not a suggestion. Pass its exact
skill name to the subagent and require it to read and follow that skill. If the user
names more than one companion skill, preserve their order and explain which governs
implementation versus review or verification.

## Waiting model

After spawning, leave the subagent running in the background and return to the main
critical path. Do not reflexively call `wait_agent`; only wait when the main path is
blocked on a result. The delegated task remains in the queue until the main path
reaches its requested gate or the user explicitly invokes `$subagent-takeover`.

Do not automatically take over merely because the subagent reports completion. The
main agent or user chooses when integration or takeover happens.

## Queue status

Report transitions using human-readable task names:

```text
Queue:
- [running] <task> — <agent/nickname>, <branch/worktree>, <phase>
- [queued] <task> — waiting for <dependency>
- [handoff] <task> — ready for $subagent-takeover
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

The start phase does not integrate or ship the work. Use `$subagent-takeover <task>`
when the main critical path is free and the selected task should become the main
agent's current work.
