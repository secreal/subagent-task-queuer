# Subagent Task Queuer (Compatibility Router)

This legacy skill now routes the workflow to two explicit skills:

- `$subagent-start [<companion-skill>] <task>` lets the main agent work when idle, or
  starts bounded delegated work when the main path is active. The default companion
  skill is `compound-engineering:lfg`.
- `$subagent-takeover <task-name-or-agent-id>` manually takes over one selected task
  from its isolated worktree, or recovers a task after interruption.

The two phase skills contain the active contracts and should be preferred for new
work.
