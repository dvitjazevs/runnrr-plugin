# Changelog

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
