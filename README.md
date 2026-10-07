# spawn-subagents

A Claude Code skill that teaches how to split work across subagents and routes each piece to the right model:

- **Haiku 5.5**: speed and efficiency (lookups, sweeps, summaries)
- **Sonnet 5.5**: well-scoped work (clear spec, clear done-criteria)
- **Opus 5.5**: everyday intelligence (ambiguity, debugging, design, review)

When invoked on a task, it decomposes the work, groups it into waves, and spawns the subagents in parallel with the Agent tool.

## Install

```bash
mkdir -p ~/.claude/skills && cp -r spawn-subagents ~/.claude/skills/
```

Then run `/spawn-subagents` in Claude Code with a task.
