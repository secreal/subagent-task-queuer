# Subagent Task Queuer (Compatibility Router)

This legacy skill now routes the workflow to two explicit skills:

- `$subagent-start <companion-skill> <task>` starts bounded delegated work, persists its
  queue, and automatically takes it over when the main critical path reaches its
  gate.
- `$subagent-takeover <task-name-or-agent-id>` manually takes over one selected task
  from its isolated worktree, or recovers a task after interruption.

The two phase skills contain the active contracts and should be preferred for new
work.
