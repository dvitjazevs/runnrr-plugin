---
name: runnrr
description: >
  This skill should be used when the user says "add to runnrr", "push to runnrr",
  "track this", "add these decisions", "add these tasks", "log this to the board",
  "pick up [task] from runnrr", "what's on my runnrr board", "work on the X card",
  "mark [task] done", "save this note", "share this note", or any similar phrase
  about their runnrr Kanban board, decisions log, or notes.
version: 0.4.2
---

# Runnrr — Personal Kanban Board + Decisions + Notes

Runnrr is a personal Kanban board that connects meetings (Granola), tasks, decisions, and notes to Claude.

**Scope of this skill: both runnrr surfaces.** The plugin's `.mcp.json` registers two MCP servers:
- **`runnrr-local`** at `localhost:19840` — the native macOS app's MCP. Full surface: tasks, files, tags, columns, due dates, priorities, decisions, `complete_task` / `move_task` / `delete_task` / `set_due_date`, `attach_file` (creates a real file on disk and links it), `tag_file`, `read_file` / `list_files` / `delete_file`, etc.
- **`runnrr`** at `${RUNNRR_URL:-https://runnrr.io}/api/mcp` — the web service. Narrower surface: tasks, decisions, notes (with public sharing).

**Local-first routing.** Use `runnrr-local` tools whenever they're available — that's where the user's macOS app reads. Only fall back to `runnrr` (web) when the native MCP isn't reachable (`localhost:19840` not listening), or when the intent requires a web-only capability (notes, public link sharing). **Never push to web when the user is looking at the native board** — they won't see the card. Fall back to curl only as a last resort if both MCPs are disconnected.

## Dual-MCP routing rule

A user may have both MCPs connected at the same time:
- **Native** (`localhost:19840`) — runnrr-native macOS app, exposes the full task lifecycle, files, tags, columns, due dates, priorities, scan_granola.
- **Web** (`runnrr.io/api/mcp`, this skill's primary target) — tasks, decisions, notes (with sharing), plus the parity tools landing in Phase 4a.

When both are connected, **prefer native for any tool that exists on both** — file system, due dates, columns, priority, tags. Web is the fallback only when native isn't reachable. On the first runnrr action in a session, name the surface you routed to so the user has a clear mental model:

> "Routing through the native runnrr MCP at `localhost:19840` (preferred when both are connected)."

If the user's request needs a tool that exists only on the web side (e.g. `save_note` / `share_note`), use the web MCP and say so explicitly.

**Base URL**: `$RUNNRR_URL` (default: `https://runnrr.io`)
**Auth**: `Authorization: Bearer $RUNNRR_API_TOKEN` on all requests

---

## Available MCP tools (web)

Read: `list_tasks`, `get_board_summary`, `find_task_by_title({ query })`, `get_task_context({ id })`, `list_notes`.

Write: `create_task({ title, description?, context? })`, `update_task({ id, title?, status?, description?, context? })`, `save_decision({ title, description?, area? })`, `save_note({ content })`, `share_note({ note_id })`.

Semantics worth knowing:

- **`update_task` `status`** is a column *title* (case-insensitive match against the user's actual columns, e.g. `"To Do"`, `"Doing"`, `"Done"`). It cannot move a task into the Inbox. If no column matches, the call returns the available column titles as an error.
- **`create_task`** always lands in the first non-Inbox standard column. It never places into Inbox.
- **`context`** is markdown attached to a task as deeper background — useful when handing off to Claude Code or capturing a meeting transcript snippet. On `update_task` it replaces existing context.
- **`save_decision`** writes to the Decisions log (separate table), not the Kanban board.
- **`save_note`** writes to Notes (separate table). `share_note` makes a note public and returns `https://runnrr.io/n/{token}`.
- **Notes are web-only.** The native macOS app does not expose `save_note` / `list_notes` / `share_note` via its local MCP. If a user is on native-only and asks to save or share a note, tell them this requires the runnrr web service.

---

## PUSH MODE — send items to the board / decisions / notes

Triggered by: "add to runnrr", "track this", "push these decisions", "log to board", "remember this", "save this note"

### Step 1 — Extract all items from the conversation

Scan the full conversation and identify **all** of the following. Use the right tool for each kind:

**Tasks** — things worth doing or tracking → `create_task`
- Each item in a numbered or bulleted list is a **separate candidate** — never collapse a list into one summary item
- Follow-up work, bugs to fix, features to build, things to verify

**Decisions** — technical or product choices made, with rationale worth preserving → `save_decision`
- Pass `title` (verb-first, ≤10 words, e.g. `"Cut visual roadmap from v1 scope"`), `description` (the why, 1–2 sentences), `area` (Product / Engineering / GTM / Design / etc.)
- Don't use `create_task` with a "Decision:" prefix — there's a dedicated tool

**Advice / practices** — recommended practices to keep → `create_task` with title prefixed `Practice:` (convention only; no dedicated tool)
- Each bullet in a list = separate candidate

**Notes** — things to remember (`"remember this"`, `"save this"`, `"note this"`) → `save_note`
- Keep `content` concise — one clear sentence or short paragraph

### Step 2 — Show confirmation list before pushing anything

```
Here's what I found to push to runnrr:

1. **[Task title]**
   [1-2 sentence description]

2. **Decision: [Title]**
   [Why — saved to the Decisions log, not the board]

3. **Note: [first words]**
   [Content — saved to Notes]

Send all, or tell me which to keep / skip / edit / add?
```

### Step 3 — Push confirmed items

**Via MCP** (preferred):
```
create_task({ title, description?, context? })
save_decision({ title, description?, area? })
save_note({ content })
```

**Via curl** (fallback):
```bash
# Tasks
curl -s -X POST "${RUNNRR_URL:-https://runnrr.io}/api/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"title":"TITLE","description":"DESCRIPTION","context":"OPTIONAL_MARKDOWN"}'

# Decisions
curl -s -X POST "${RUNNRR_URL:-https://runnrr.io}/api/decisions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"title":"TITLE","description":"WHY","area":"AREA"}'

# Notes
curl -s -X POST "${RUNNRR_URL:-https://runnrr.io}/api/notes" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"content":"CONTENT"}'
```

Report results with ✓/✗ per item after sending.

---

## PULL MODE — get task context from the board

Triggered by: "pick up [task name] from runnrr", "what's on my board", "work on the X card"

**Via MCP** (preferred):
1. `find_task_by_title({ query: "<task name>" })` — returns up to 5 ranked matches with id and status. **Use this rather than scanning `list_tasks` manually.**
2. If nothing matches or the user wants the whole board, fall back to `list_tasks` or `get_board_summary`.
3. `get_task_context({ id: "..." })` — fetch full markdown context (description, saved context, recent activity).
4. Present the context and ask: "Here's the context for [title]. What would you like me to do first?"

**Via curl** (fallback):
```bash
# List tasks
curl -s "${RUNNRR_URL:-https://runnrr.io}/api/tasks" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN"

# Get context for a specific task — returns { claudeCode, claudeChat, cowork }
curl -s "${RUNNRR_URL:-https://runnrr.io}/api/tasks/TASK_ID/context" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN"
```

The HTTP context endpoint returns `{ claudeCode, claudeChat, cowork }` — use the `cowork` field. (The MCP `get_task_context` tool returns the same content as plain markdown text without the envelope.)

---

## UPDATE MODE — move tasks, mark them done, edit content

Triggered by: "mark [task] done", "move [task] to [column]", "update runnrr", "log the result"

**Move to a column** — `status` is a column *title* (case-insensitive). Cannot target Inbox.
```
update_task({ id: "...", status: "Done" })
```
If the column doesn't exist, the response lists available columns.

**Edit content**:
```
update_task({ id: "...", title?: "...", description?: "...", context?: "..." })
```
`context` replaces existing context entirely.

**Via curl**:
```bash
curl -s -X PATCH "${RUNNRR_URL:-https://runnrr.io}/api/tasks/TASK_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"status":"Done"}'
```

---

## NOTES MODE — save and share notes (web-only)

Triggered by: "remember this", "save this", "note this for later", "share this note", "make this public"

- **Save**: `save_note({ content })` — concise, source auto-set to `claude-code`.
- **List**: `list_notes` — newest first, format `N. [source] (id: ID) content`.
- **Share**: `share_note({ note_id })` — returns `https://runnrr.io/n/{token}` and flips the note public. If you don't have the id, call `list_notes` first.

Notes are not exposed by the native macOS app's local MCP. If the user is on native-only and asks to save or share a note, tell them this requires the runnrr web service and offer to push it via this skill instead.

---

## What this skill does NOT cover

The native macOS `runnrr-native` app exposes its own MCP at `localhost:19840` with a different toolset. If the user mentions any of the following, route to the native MCP, not this skill:

- File CRUD on registered folder vaults: `create_file`, `read_file`, `list_files`, `delete_file`, `attach_file`, `tag_file`
- Column listing: `list_columns`
- Tag listing: `list_tags`
- Task lifecycle beyond status: `complete_task`, `move_task`, `delete_task`, `set_due_date`
- Priority and tag fields on tasks
- Granola scan (UI-only — not exposed via either MCP today)

---

## Setup reminder

If requests fail with a JSON-RPC error containing `"Missing or invalid API token"`:
1. Show the user the **exact error message** from the response — it contains the setup URL and command
2. Do NOT show raw API errors or stack traces
3. Guide the user: "Visit https://runnrr.io/onboarding to get your token, then run the setup command it gives you."

If requests fail with 401 or `Unauthorized`:
- Same guidance as above — the token is missing or expired
- Get it from: https://runnrr.io/onboarding
- Set in environment: `export RUNNRR_API_TOKEN=your-token-here`
