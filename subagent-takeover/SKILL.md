---
name: subagent-takeover
description: Take over one selected delegated task from its isolated worktree, audit the handoff, finish the work, and validate it safely.
---

# Subagent Takeover

Use this skill when the main workflow automatically selects a ready delegated task,
or when the user selects one explicitly with `$subagent-takeover <task-name-or-agent-id>`.
It is the takeover phase of `$subagent-start`, not a new delegation.

## Takeover contract

- Re-read the repository `AGENTS.md`, continuity notes, the selected task contract,
  and every companion skill named by that contract before editing.
- Rebuild the queue from actual agent and worktree state. Resolve the exact task,
  agent id, branch, worktree, intended base, repository, and write scope.
- Read the durable queue record from `${CODEX_HOME}/subagent-queues/` (or
  `~/.codex/subagent-queues/` when `CODEX_HOME` is unset), then reconcile it with
  the live agent and worktree state. Mark the selected task as `takeover` before
  editing and persist transitions after validation.
- If the agent is still running, request a concise handoff and stop it before
  becoming the sole writer. Do not take over an ambiguous task or a task whose
  worktree cannot be identified.
- Inspect without discarding work: `git status`, branch/upstream, recent commits,
  diff against the intended base, untracked files, plans, and test artifacts.
- Separate completed work, partial work, unrelated WIP, and missing work. Continue
  from the existing implementation; do not restart from scratch.
- Keep edits limited to the inherited task scope. Leave unrelated WIP, other
  worktrees, and other delegated tasks untouched.
- If this takeover was automatic, do not ask the user to confirm the selection when
  the queue has one ready task or a deterministic priority/order. Manual selection
  remains available for ambiguous or early takeover requests.

## Mandatory pre-edit report

Show this compact report before making any edit:

```text
Takeover:
- Task: <human-readable task name>
- Source: <agent id>, <branch>, <worktree>, <status>
- Completed by subagent: <completed units and verification evidence>
- Continuing now: <remaining units the main agent will implement>
- Untouched: <unrelated WIP or files intentionally left alone>
- Blockers/assumptions: <if any>
```

The `Completed by subagent` and `Continuing now` lines are mandatory even when
either is empty.

## Validation and handoff

- Add or preserve focused regression tests before fixing confirmed behavioral gaps.
- Run focused tests first, then the affected package/repository suite and applicable
  build, lint, vet, browser, or runtime checks. Use the companion skill's required
  checks and report exact commands and results.
- Do not commit, push, merge, deploy, send external messages, or restart services
  unless the current user instruction or an explicitly invoked shipping skill
  authorizes it.
- Report task status, agent id, branch/worktree, files changed and intentionally
  untouched, completed versus remaining units, checks and results, blockers or
  assumptions, and the next safe action.

If the same external blocker persists through three meaningful checks and no safe
progress is possible, report the blocker and leave the durable task record in
`blocked`/`handoff` state. Do not invent completion or silently widen the task scope.
