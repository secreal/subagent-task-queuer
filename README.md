# Subagent Task Queuer

Codex skill for a delegated-work queue that keeps the main critical path moving and
lets the main agent safely take over unfinished subagent work from its isolated
worktree.

Invoke it as `$subagent-task-queuer` before starting a new engineering task. Pair it
with the relevant Compound Engineering skill, such as `ce-debug`, `ce-work`,
`ce-code-review`, or `lfg`.

The skill keeps task status human-readable, preserves unrelated work in progress,
limits delegated work to explicit scopes, and validates a worktree before takeover.
When the main path is free, takeover starts immediately even if the subagent has not
finished. It treats “unlimited” as a logical queue constrained by available agent
capacity; it does not spawn beyond that capacity.
