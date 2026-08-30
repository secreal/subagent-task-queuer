---
name: subagent-task-queuer
description: Orchestrate Compound Engineering work by delegating bounded side tasks to subagents, keeping a visible queue, and taking over unfinished work in its isolated worktree when the main path is free.
---

# Subagent Task Queuer

Use this skill at the start of a coding session or before a new engineering task when the preferred workflow is: delegate first, keep the critical path moving, then audit and finish unfinished subagent work. Pair the actual implementation with the relevant `compound-engineering:*` skill such as `ce-debug`, `ce-work`, `ce-code-review`, or `lfg`.

## Operating contract

- Work from the repository root and read the repository's `AGENTS.md` plus the user's continuity notes before making changes.
- Inventory every repository and worktree involved. Record existing dirty files before delegating or editing; preserve unrelated WIP and never reset, checkout, or delete it destructively.
- Split the request into one primary critical-path task and bounded side tasks. Delegate only independent side tasks with a disjoint write scope. Do not delegate the immediate blocker that the main agent needs next.
- Spawn a subagent in an isolated worktree for each bounded side task when delegation is explicitly requested or this skill is invoked. Give it the exact objective, allowed repositories/files, required tests, exclusions, and handoff format.
- Treat the queue as logical backlog, not as permission to exceed the host's active-agent limit. Keep queued work recorded with a task name, agent id, worktree, status, and dependency.
- A delegated task must not silently broaden its scope. Unless the user explicitly authorizes shipping, default side-task instructions are: no commit, push, merge, deploy, production mutation, message sending, or service restart.
- Continue useful non-overlapping primary work while side agents run. Do not redo a side task merely because its agent has not reported yet.

## Queue status

Keep status understandable to the user. Use task names rather than opaque agent nicknames and report at meaningful transitions:

```text
Queue:
- [running] <task> — <agent/worktree>, current phase
- [queued] <task> — waiting for <dependency>
- [handoff] <task> — agent stopped or incomplete; ready for takeover
- [done] <task> — validation and shipment status

Main path: <current task and phase>
Next gate: <what must happen before the next takeover or shipment>
```

Do not claim an agent is done from a generic waiting state. When progress is requested, inspect the actual agent status, worktree status, commits, and test evidence.

## Takeover protocol

Take over a subagent task only when the main critical-path work is complete at its requested gate (for example, merged/deployed) or the user explicitly asks for takeover. Then:

1. Identify the exact agent, branch, worktree, repositories, and task contract. If the agent is still running, request a concise handoff or stop it through the available agent control tool before becoming the sole writer.
2. Inspect the worktree without discarding anything: `git status`, branch/upstream, recent commits, diff against the intended base, untracked files, and any plan or test artifacts.
3. Compare the implementation against the delegated objective and the relevant Compound Engineering plan. Separate completed work, partial work, unrelated WIP, and missing work. Do not restart from scratch.
4. Read the relevant implementation/test instructions for the active Compound Engineering skill. Continue from the partial implementation, adding tests before fixing confirmed behavioral gaps.
5. Validate proportionally: focused tests first, then the affected package/repository suite, build/lint/vet as applicable, and browser or runtime checks only when the environment is safe and available. State exact commands and results.
6. Keep the takeover write scope limited to the inherited task. Leave unrelated subagent work, other worktrees, and pre-existing WIP untouched.
7. Before shipping, run the appropriate review/quality step. Commit, push, merge, deploy, or send external messages only when the current user instruction or an explicitly invoked shipping skill authorizes it. Otherwise leave the validated changes uncommitted and report the handoff.

## Handoff requirements

Every subagent handoff and takeover report must include:

- task name and current status;
- agent id, branch, and worktree;
- files changed and files intentionally untouched;
- completed units versus remaining units;
- tests/checks run and their results;
- blockers, assumptions, and the next safe action.

If a subagent is interrupted, its worktree is the source of truth. If a subagent has no usable worktree or the same external blocker persists through three meaningful checks, report the blocker instead of inventing progress. Keep the task in the queue until it is genuinely complete or the user changes its scope.

## Safety boundaries

- Tenant data, production WhatsApp sessions, payment gateways, and infrastructure are external state: use read-only inspection unless the user explicitly authorizes a mutation.
- Do not expose secrets in status reports, prompts, logs, or commits. Do not ask an agent to paste credentials.
- Never merge or deploy merely because a PR is merge-ready; use the user's explicit shipping instruction.
- If multiple tasks touch the same files, serialize them and make the dependency visible instead of creating conflicting worktrees.
