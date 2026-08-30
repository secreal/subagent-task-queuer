---
name: subagent-task-queuer
description: Compatibility router for the split subagent-start and subagent-takeover workflows.
---

# Subagent Task Queuer (Compatibility Router)

This legacy entrypoint is retained so existing references keep working. The workflow
is now split into two explicit phases:

- Use `$subagent-start <companion-skill>` to delegate bounded work under a named
  skill and leave it running while the main critical path continues.
- Use `$subagent-takeover <task-name-or-agent-id>` to select one queued task, audit
  its isolated worktree, and finish it.

Do not infer a phase from this compatibility name when the request is ambiguous;
route to the explicit phase or ask the user to choose one. Read the selected skill's
current instructions before acting. The two phase skills own the detailed contracts,
queue format, handoff requirements, and safety boundaries.
