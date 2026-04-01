# runnrr — Claude Code Plugin

Push tasks, decisions, and notes from any Claude Code session to your [runnrr](https://runnrr.io) Kanban board.

## Install

```
/plugin marketplace add https://github.com/dvitjazevs/runnrr-plugin
/plugin install runnrr
```

Restart Claude Code. Then get your API token at [runnrr.io/onboarding](https://runnrr.io/onboarding) and add it to your environment:

```bash
export RUNNRR_API_TOKEN=your-token-here
```

## Usage

Say any of the following in a Claude Code session:

- "add to runnrr"
- "push to runnrr"
- "track this"
- "add these decisions"
- "what's on my runnrr board"
- "pick up [task name] from runnrr"
- "mark [task] done"

## What gets captured

- **Tasks** — follow-up work, bugs, features
- **Decisions** — technical or product choices with rationale
- **Practices** — recommended approaches worth preserving

Claude shows a confirmation list before pushing anything. You choose what goes through.
