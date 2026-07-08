# agent-skills

Skills for AI coding agent CLIs (Claude Code, Codex, etc.), installable via [skills.sh](https://skills.sh/).

## Skills

### `tmux`

Coordinate multiple independent agent sessions (potentially different harnesses -- Claude Code, Codex, etc.) running in separate tmux panes on the same machine: read/write each other's panes directly, with conventions for tagging peer messages, budgeting how long an exchange runs, signaling completion, detecting no-progress loops, and escalating to the human operator instead of resolving permission prompts on their behalf.

Install:

```bash
npx skills add lexuzieel/agent-skills --skill tmux
```

See [`skills/tmux/SKILL.md`](skills/tmux/SKILL.md) for the full protocol.
