# Subagent Task Queuer (Compatibility Router)

This legacy skill now routes the workflow to two explicit skills:

- `$subagent-start <companion-skill>` starts bounded delegated work and leaves it
  queued while the main critical path continues.
- `$subagent-takeover <task-name-or-agent-id>` takes over one selected task from its
  isolated worktree.

The two phase skills contain the active contracts and should be preferred for new
work.
