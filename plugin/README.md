# runnrr Plugin

Push tasks, decisions, and practices from any Claude session directly to your runnrr Kanban board.

## Overview

After any conversation — a coding session, a planning discussion, advice on a topic — just say "add to runnrr" and the skill will scan the conversation, extract all actionable items (including every item in lists), and present them for confirmation before pushing anything to the board.

## Components

| Component | Name | Purpose |
|-----------|------|---------|
| Skill | `runnrr` | Extract items from conversation and push to board |

## Setup

1. In Claude Code (or any Claude harness that supports plugins), paste:
   ```
   Please install the runnrr plugin from https://runnrr.io/runnrr.plugin
   ```
2. Claude will present an install card — click Accept.
3. Set your API token (from runnrr.io/onboarding):
   ```
   export RUNNRR_API_TOKEN=your-token-here
   ```

## Usage

Say any of the following in any Claude conversation:
- "add to runnrr"
- "push to runnrr"
- "add these decisions"
- "track this"
- "log this to the board"

The skill will show a numbered list of everything it found. Respond with:
- `all` — push everything
- `1, 3` — push only those items
- `skip 2` — push all except item 2
- Edit requests inline before confirming

## What gets extracted

- **Tasks** — things to do or follow up on (via `create_task`)
- **Decisions** — architectural or product choices (via the dedicated `save_decision` tool — saved to the Decisions log, not the board)
- **Notes** — things you want to remember (via `save_note`); shareable as public links via `share_note`
- **Practices / advice** — recommended actions from a list, prefixed with `Practice:` (a convention; uses `create_task`) — each bullet is treated as a separate item, not collapsed into one

## Scope

This plugin targets the runnrr **web** service. The native macOS runnrr app has its own local MCP at `localhost:19840` exposing files, tags, columns, due dates, and priorities — those are out of scope here. Notes are web-only.
