---
name: runnrr
description: >
  This skill should be used when the user says "add to runnrr", "push to runnrr",
  "track this", "add these decisions", "add these tasks", "log this to the board",
  "pick up [task] from runnrr", "what's on my runnrr board", "work on the X card",
  "mark [task] done", "save a note", "save this note", "take a note", "log this as a note",
  "share this note", or any similar phrase about their runnrr Kanban board, decisions
  log, or notes.
version: 0.6.1
---

# Runnrr — Personal Kanban Board + Decisions + Notes

Runnrr is a personal Kanban board that connects meetings (Granola), tasks, decisions, and notes to Claude. Everything lives locally in the runnrr macOS app and is reached through one MCP server.

**One server.** This plugin's tools all route through `runnrr-local`, an MCP server at `http://localhost:19840/mcp` running inside the runnrr macOS app. The app reads from a local SQLite database + on-disk markdown vault. No internet round-trip; no cloud sign-in needed for tools to work.

**Requires the runnrr Mac app.** Without the app installed and running, no MCP tools are reachable. Download at [runnrr.io/download](https://runnrr.io/download).

**Bearer token required.** As of plugin v0.6.0, the local MCP server requires a per-machine bearer token (security: prevents any local process from calling `delete_task` etc.). The token is generated automatically by the Mac app on first launch and stored at `~/Library/Application Support/runnrr/mcp.token`. The app's onboarding shows the full `claude mcp add …` install command with the token inlined — copy and run it once.

---

## First-time setup

If `runnrr-local` isn't registered with Claude Code yet (or if tool calls fail with `connection refused` / `401`), the user needs to run the install command. Walk them through:

1. **Make sure the Mac app is running.** `runnrr.app` should be visible in `/Applications`. Launch it. The menu-bar icon (lime square) confirms it's up.
2. **Copy the install command.** Open the runnrr menu-bar icon → **About runnrr** → **Setup** → copy the displayed `claude mcp add …` command. (Pre-1.0: the command is also visible in the onboarding flow's "Connect Claude Code" step.) The command embeds the per-machine bearer token, which is why we don't ship it as a static `.mcp.json`.
3. **Paste into a terminal and run it once.** Claude Code now knows where the local MCP is and how to authenticate. No further setup needed across sessions.

---

## Tool surface

**Tasks:** `create_task`, `update_task`, `complete_task`, `move_task`, `delete_task`, `set_due_date`, `find_task_by_title`, `get_task_context`, `list_tasks`, `get_board_summary`

**Board structure:** `list_columns`, `list_tags`

**Decisions:** `save_decision` (writes to the Decisions log, not the Kanban board)

**Files (= notes):** `create_file`, `read_file`, `list_files`, `attach_file`, `delete_file`, `tag_file`

Notes ARE markdown files in the user's local vault. There is no separate "notes" tool. When the user says *"save a note"*, use `create_file` with `subfolder: "notes"` and a slugified filename — see the NOTES MODE below for the exact pattern.

---

## Server-alive precheck (do this before invoking any tool)

Tools fail in two distinct ways. Distinguish them upfront so the user gets the right fix:

| Symptom | Cause | What to tell the user |
|---|---|---|
| MCP tools not registered / not visible in Claude Code | First-time setup not done | Walk them through "First-time setup" above. |
| Tool call returns connection error (`refused`, `unreachable`) | Mac app isn't running | "Open `runnrr.app` from your Applications folder. The local MCP server starts automatically when the app launches." |
| Tool call returns HTTP 401 / `Unauthorized — missing or invalid bearer token` | Bearer token mismatch (token rotated, or install command was run against an older token) | "Re-run the `claude mcp add …` install command from the runnrr Mac app's About → Setup. The token regenerates if you ever delete the `mcp.token` file." |
| Tool call returns `No file vault configured` | User hasn't picked a vault folder yet | "Open runnrr → Files → Add Folder, pick a folder, then try again." Surface this **verbatim** — do not paraphrase. |

If you don't know whether the server is reachable, optionally probe `curl -fsSL http://localhost:19840/mcp` once. The endpoint is unauthenticated for liveness checks and returns `{"status":"ok","server":"runnrr-local"}` when the app is running.

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

### Long task context — attach as .md file instead of stuffing the description

A task description is for a 1-3 sentence summary. Substantial supporting material (a brief, a code dump, meeting notes, design rationale) belongs in a linked markdown file, not the description field.

**Trigger this branch when either:**
- The user explicitly asks for it: *"attach as md"*, *"attach as a file"*, *"put the context in a linked note"*, *"as an attached file"*
- OR the body you'd otherwise put into `description` exceeds ~1500 characters

Explicit user intent always wins — if they say "attach as md" with a short body, still attach it.

**Flow:**

1. `create_task({ title, description })` — `description` is a 1-3 sentence summary that points to the attached file (e.g., *"Full brief in attached file."*).
2. `create_file({ filename: "<slug>-YYYY-MM-DD.md", content: <full body>, subfolder: "tasks" })` — capture the returned file `id`.
3. `attach_file({ card_id: <new task id>, file_id: <file id from step 2> })` — link the file to the card.
4. Confirm: `✓ Added task [title] with attached file [filename]`.

**Filename derivation:** slugify the task title, append today's date in YYYY-MM-DD, add `.md`. Same rules as NOTES MODE. Collision handling is automatic — confirm the actual filename returned.

**In the Step 2 confirmation list**, mark these entries so the user sees the intent before pushing:

```
1. **[Task title]** (+ attached file: <slug>-YYYY-MM-DD.md)
   [1-2 sentence description that will go in the description field]
   [→ separately: full body goes into the linked .md file]
```

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
