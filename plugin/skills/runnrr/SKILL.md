---
name: runnrr
description: >
  This skill should be used when the user says "add to runnrr", "push to runnrr",
  "track this", "add these decisions", "add these tasks", "log this to the board",
  "pick up [task] from runnrr", "what's on my runnrr board", "work on the X card",
  "mark [task] done", or any similar phrase about their runnrr Kanban board.
version: 0.4.0
---

# Runnrr — Personal Kanban Board

Runnrr is a personal Kanban board that connects meetings (Granola), tasks, and Claude.
Use the runnrr MCP tools when available. Fall back to curl if MCP is not connected.

**Base URL**: `$RUNNRR_URL` (default: `https://runnrr.io`)
**Auth**: `Authorization: Bearer $RUNNRR_API_TOKEN` on all requests

---

## PUSH MODE — send items to the board

Triggered by: "add to runnrr", "track this", "push these decisions", "log to board"

### Step 1 — Extract all items from the conversation

Scan the full conversation and identify **all** of the following:

**Tasks** — things worth doing or tracking:
- Each item in a numbered or bulleted list is a **separate candidate** — never collapse a list into one summary item
- Follow-up work, bugs to fix, features to build, things to verify

**Decisions** — prefix title with `Decision:`
- Technical or product choices made, with rationale worth preserving

**Advice / practices** — prefix title with `Practice:`
- Each recommended practice in a list = separate candidate

### Step 2 — Show confirmation list before pushing anything

```
Here's what I found to push to runnrr:

1. **[Title]**
   [1-2 sentence description]

2. **Decision: [Title]**
   [Description]

Send all, or tell me which to keep / skip / edit / add?
```

### Step 3 — Offer Markdown attachment

After the user confirms the items, **proactively ask:**

```
Want to attach a Markdown file with deeper context to any of these?
I can generate it from our conversation — you control what goes in.
```

If the user says yes:
1. Ask which item(s) to attach Markdown to
2. Generate markdown content from the conversation (design decisions, requirements, rationale, technical details)
3. Show the generated markdown as a preview
4. Let the user approve, edit instructions, or skip
5. Include approved markdown in the `context` field when pushing

If the user says no or skips, push without context.

### Step 4 — Push confirmed items

**Via MCP tool** (preferred if runnrr MCP is connected):
```
create_task({ title: "...", description: "...", context: "# markdown content..." })
```

**Via curl** (fallback):
```bash
curl -s -X POST "${RUNNRR_URL:-https://runnrr.io}/api/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"title":"TITLE","description":"DESCRIPTION","context":"# Markdown content here..."}'
```

Report results with ✓/✗ per item after sending.

---

## PULL MODE — get task context from the board

Triggered by: "pick up [task name] from runnrr", "what's on my board", "work on the X card"

**Via MCP tools** (preferred):
1. `list_tasks()` — get all tasks
2. Find the closest title match to what the user named
3. `get_task_context({ id: "..." })` — fetch full context as markdown
4. Present the context and ask: "Here's the context for [title]. What would you like me to do first?"

**Via curl** (fallback):
```bash
# List tasks
curl -s "${RUNNRR_URL:-https://runnrr.io}/api/tasks" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN"

# Get context for a specific task
curl -s "${RUNNRR_URL:-https://runnrr.io}/api/tasks/TASK_ID/context" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN"
```

The context endpoint returns `{ claudeCode, claudeChat, cowork }` — use the `cowork` field.

---

## UPDATE MODE — mark tasks done, add notes, or attach Markdown

Triggered by: "mark [task] done", "update runnrr", "log the result", "attach context to [task]"

**Via MCP**: `update_task({ id: "...", status: "done" })` or `update_task({ id: "...", context: "# markdown..." })`

**Via curl**:
```bash
curl -s -X PATCH "${RUNNRR_URL:-https://runnrr.io}/api/tasks/TASK_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"status":"done"}'

# Attach or replace Markdown content
curl -s -X PATCH "${RUNNRR_URL:-https://runnrr.io}/api/tasks/TASK_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNNRR_API_TOKEN" \
  -d '{"context":"# Updated markdown content..."}'
```

---

## Setup reminder

If requests fail with 401, the user needs to set their API key:
- Get it from: https://runnrr.io/onboarding
- Set in environment: `export RUNNRR_API_TOKEN=your-key-here`
