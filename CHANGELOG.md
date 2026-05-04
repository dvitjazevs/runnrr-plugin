# Changelog

## v0.6.1 — 2026-05-04

**Skill addition — long task context as attached `.md` file:**

PUSH MODE now supports attaching a task's supporting body as a linked markdown file in the user's vault, instead of stuffing it into the description field. Triggered explicitly ("attach as md", "attach as a file", "as an attached file") or automatically when the description body would exceed ~1500 characters. The flow: `create_task` (with a 1-3 sentence summary) → `create_file` (under a `tasks/` subfolder) → `attach_file` (link by `file_id`). The Step 2 confirmation list now flags these entries so the user sees the attach intent before pushing.

No breaking changes; no MCP server changes required. Existing v0.6.0 install (bearer-token registration) continues to work — just `/plugin marketplace update runnrr` then `/reload-plugins` to pick up the new skill.

## v0.6.0 — 2026-04-28

**Breaking:** Plugin no longer ships an auto-registering `.mcp.json`. The local MCP server now requires a per-machine bearer token (security: prevents any local process from invoking `delete_task` etc. on your board). Static auto-registration can't carry a per-machine secret, so registration moves to the runnrr Mac app's onboarding.

**What changed:**
- `.mcp.json` removed. Connecting Claude Code to the runnrr Mac app is now a one-time `claude mcp add …` command shown in the app's onboarding (Connect Claude Code screen) or About → Setup. The command embeds the bearer token from `~/Library/Application Support/runnrr/mcp.token`.
- SKILL.md gains a new "Server-alive precheck" table that distinguishes the four failure modes Claude should diagnose before falling back to a generic error: not registered (run install command), not running (launch the app), 401 (re-run install command — token mismatched), no vault (open runnrr → Files → Add Folder).
- Plugin and marketplace descriptions now lead with "Requires the runnrr Mac app (runnrr.io/download)" so the dependency is obvious before install.

**For existing users (upgrading from v0.5.0):**
- After updating, your previous `runnrr-local` registration in `~/.claude/settings.json` still points at `localhost:19840/mcp` — but without an Authorization header. Tool calls will start returning 401 once you update the runnrr Mac app to a build that requires the bearer token.
- Fix: open `runnrr.app` → About → Setup → copy the install command shown there → paste into a terminal and run it. This replaces the existing registration with one that includes the bearer header. One-time, takes 5 seconds.

**For new users:**
- Install the runnrr Mac app from runnrr.io/download. Launch it once.
- Install this plugin from the marketplace.
- The Mac app's onboarding shows the `claude mcp add …` install command. Copy + run.
- Done — Claude Code now talks to your local runnrr board.

## v0.5.0 — 2026-04-27

**Breaking:** Claude Code no longer registers the cloud MCP server (`runnrr.io/api/mcp`). Plugin's `.mcp.json` now lists only `runnrr-local`.

**What changed:**
- Single-server architecture: one MCP for Claude Code (local), no routing logic.
- Notes are now markdown files in your local runnrr vault. The skill maps "save a note" to `create_file` with a `notes/` subfolder. Filename collision is handled by the local MCP.
- Sharing a note publicly now requires cloud sync to be enabled in the runnrr macOS app. The skill surfaces a clear message instead of silently failing.
- Skill rewritten for clarity now that there's no two-server cohabitation.

**For existing users:**
- After updating, restart Claude Code so the new `.mcp.json` is picked up.
- If you previously ran `claude mcp add` for the cloud server outside the plugin, that entry persists in `~/.claude/settings.json`. Remove it manually or it will produce 401 errors on session start.
- Your `RUNNRR_API_TOKEN` env var is still useful for cloud sync (when enabled in the macOS app) and for Claude Chat. It is no longer needed for Claude Code tool availability.

**For Claude Chat users:** install the cloud MCP separately from runnrr.io → Settings → Connect to Claude Chat (shipped in coordinated runnrr-web release).

## v0.4.2 — 2026-04-27

Skill alignment + dual-MCP awareness + harness-agnostic copy.
