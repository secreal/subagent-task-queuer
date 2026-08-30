---
name: subagent-task-queuer
description: Compatibility router for the split subagent-start and subagent-takeover workflows.
---

# Subagent Task Queuer (Compatibility Router)

This legacy entrypoint is retained so existing references keep working. The workflow
is now split into two explicit phases:

- Use `$subagent-start <companion-skill> <task>` to delegate bounded work under a named
  skill, persist the queue, and automatically take it over when the main path's
  gate is reached.
- Use `$subagent-takeover <task-name-or-agent-id>` to select one queued task, audit
  its isolated worktree, and finish it manually or as recovery.

Do not infer a phase from this compatibility name when the request is ambiguous;
route to the explicit phase or ask the user to choose one. Read the selected skill's
current instructions before acting. On every invocation, reconcile durable queue
records and automatically resume any ready `auto_takeover` task before unrelated
engineering work. The two phase skills own the detailed contracts, queue format,
handoff requirements, and safety boundaries.
