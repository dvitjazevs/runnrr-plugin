---
name: runnrr
description: >
  This skill should be used when the user says "add to runnrr", "push to runnrr",
  "track this", "add these decisions", "add these tasks", "log this to the board",
  "pick up [task] from runnrr", "what's on my runnrr board", "work on the X card",
  "mark [task] done", "save a note", "save this note", "take a note", "log this as a note",
  "share this note", or any similar phrase about their runnrr Kanban board, decisions
  log, or notes.
version: 0.5.0
---

# Runnrr — Personal Kanban Board + Decisions + Notes

Runnrr is a personal Kanban board that connects meetings (Granola), tasks, decisions, and notes to Claude. Everything lives locally in the runnrr macOS app and is reached through one MCP server.

**One server, no routing.** This plugin registers a single MCP server, `runnrr-local`, at `http://localhost:19840/mcp`. It runs inside the runnrr-native macOS app and reads from the local SQLite database + on-disk markdown vault. No bearer token, no internet round-trip, no sign-in required for tools to work.

If the runnrr macOS app isn't running, none of these tools will work — the user needs to launch the app.

---

## Tool surface

**Tasks:** `create_task`, `update_task`, `complete_task`, `move_task`, `delete_task`, `set_due_date`, `find_task_by_title`, `get_task_context`, `list_tasks`, `get_board_summary`

**Board structure:** `list_columns`, `list_tags`

**Decisions:** `save_decision` (writes to the Decisions log, not the Kanban board)

**Files (= notes):** `create_file`, `read_file`, `list_files`, `attach_file`, `delete_file`, `tag_file`

Notes ARE markdown files in the user's local vault. There is no separate "notes" tool. When the user says *"save a note"*, use `create_file` with `subfolder: "notes"` and a slugified filename — see the NOTES MODE below for the exact pattern.

---

## PUSH MODE — send items to the board / decisions / notes

**Triggered by:** "add to runnrr", "track this", "push these decisions", "log to board", "remember this", "save this note"

### Step 1 — Extract all items

Scan the conversation and identify each candidate. Pick the right tool for each kind:

- **Tasks** — things worth doing or tracking → `create_task`. Each item in a list is a separate candidate; never collapse a list into one.
- **Decisions** — technical or product choices made, with rationale worth preserving → `save_decision({ title, description, area })`. `title` is verb-first, ≤10 words. `area` is Product / Engineering / GTM / Design / etc. Don't use `create_task` with a "Decision:" prefix — there's a dedicated tool.
- **Practices / advice** — recommended ongoing practice → `create_task` with title prefixed `Practice:` (convention only; no dedicated tool).
- **Notes** — content the user wants to keep ("remember this", "save this note", "take a note") → `create_file` with `subfolder: "notes"` (see NOTES MODE).

### Step 2 — Show a confirmation list before pushing

```
Here's what I found to push to runnrr:

1. **[Task title]**
   [1-2 sentence description]

2. **Decision: [Title]** (area: [Product/Engineering/etc])
   [Why — saved to the Decisions log, not the board]

3. **Note: [first line / title]**
   [→ <vault>/notes/<slug>.md]

Send all, or tell me which to keep / skip / edit / add?
```

### Step 3 — Push confirmed items

Use the corresponding MCP tools. Report results with ✓/✗ per item.

---

## PULL MODE — get task context from the board

**Triggered by:** "pick up [task name] from runnrr", "what's on my board", "work on the X card"

1. `find_task_by_title({ query: "<task name>" })` — returns up to 5 ranked matches. Use this rather than scanning `list_tasks` manually.
2. If nothing matches or the user wants the whole board, fall back to `list_tasks` or `get_board_summary`.
3. `get_task_context({ id: "..." })` — fetch full markdown context (description, saved context, recent activity).
4. Present the context and ask: *"Here's the context for [title]. What would you like me to do first?"*

---

## UPDATE MODE — move tasks, mark them done, edit content

**Triggered by:** "mark [task] done", "move [task] to [column]", "log the result"

- **Mark done:** `complete_task({ id })`.
- **Move to a column:** `move_task({ id, column: "Done" })`. The column name is case-insensitive against the user's actual columns. Cannot target the Inbox.
- **Edit content:** `update_task({ id, title?, description?, context?, priority?, tags? })`. `context` replaces existing context; pass `null` for `priority` to clear it; pass `[]` for `tags` to clear all tags.

---

## NOTES MODE — save and read notes (= markdown files in your vault)

**Triggered by:** "save a note", "take a note", "log this as a note", "remember this"

A note is a markdown file in the user's local vault under `<vault>/notes/`. Use `create_file`:

```
create_file({
  filename: "<slugified>.md",
  content: "<the note body>",
  subfolder: "notes"
})
```

### Filename derivation

1. Take the explicit title if given, otherwise the first 60 characters of the content.
2. Slugify: lowercase, ASCII alphanumerics + hyphens, collapse runs.
3. Append `-YYYY-MM-DD` (today's date in the user's locale).
4. Add the `.md` extension.

Examples:
- *"save a note: standup highlights — restart picked up v0.4.2"* → `standup-highlights-restart-picked-up-v0-4-2-2026-04-27.md`
- *"take a note titled Q2 plan"* → `q2-plan-2026-04-27.md`

The local MCP handles **collision detection automatically** — if a file with the same name already exists at the target path, it auto-suffixes `-2`, `-3`, etc. and returns the final filename in the response. Confirm the actual filename to the user.

### No vault configured

If `create_file` returns the error message *"No file vault configured. Open runnrr and add a folder in Files → Add Folder."*, surface it to the user **verbatim** — do not paraphrase or reword.

### Listing and reading

- `list_files()` — list every indexed markdown file (filter by tags or folder if needed).
- `read_file({ id })` or `read_file({ path })` — return the full content of a file.

### Sharing a note (cloud-only)

If the user asks to *"share this note"* or *"make this public"*, respond with:

> *"Sharing a note publicly requires cloud sync. Open runnrr → Profile → Data & Sync to enable it, then ask again. Once sync is on, your notes become accessible at `https://runnrr.io/n/<token>`."*

Do not attempt to share via this skill — the cloud surface isn't reachable from Claude Code in this plugin version.

---

## What this skill does NOT cover

- **Granola scan** is UI-only — kicked off from the macOS app's Board view, not from the MCP.
- **Public note sharing** is cloud-only and requires the runnrr.io UI to enable cloud sync first.
- **Claude Chat integration** is a separate install path (`claude mcp add` against `https://runnrr.io/api/mcp` with a Bearer token from runnrr.io → Settings → Connect to Claude Chat). This skill is Claude-Code-specific.

---

## Setup

If a tool call fails with `connection refused` or returns no response: the runnrr macOS app isn't running. Tell the user to launch it — the local MCP server starts automatically on app launch and listens on port 19840.

The plugin auto-registers the local MCP server via `.mcp.json` — no manual `claude mcp add` is required.
