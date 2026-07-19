# claude-agents

An agent skill for managing Claude Code background agents: the list you see when you run `claude agents`.

Claude Code can spawn background sessions with `claude --bg`, but there is no official CLI for renaming them, recoloring them, or cleaning up the list. This skill documents how the daemon actually stores that state (verified against Claude Code 2.1.215 by reading the binary), so an agent can do all of it for you:

- spawn a new background agent with the right flags
- list every job with its state and last result
- read and summarize any agent's full conversation transcript
- rename a job (`state.json` edit)
- set a session's color from outside (the same `agent-color` transcript event that `/color` writes)
- stop a running job and delete finished ones safely

## Install

```bash
npx skills add gaojude/claude-agents-skill
```

Or copy `SKILL.md` into `~/.claude/skills/claude-agents/` yourself.

## Portability

The skill resolves everything from `${CLAUDE_CONFIG_DIR:-$HOME/.claude}`, calls `command claude` to bypass shell aliases, and contains no machine-specific paths. The file formats it touches are internal to Claude Code, so it records the version it was verified against and tells the agent how to re-verify if something breaks.

## License

MIT
