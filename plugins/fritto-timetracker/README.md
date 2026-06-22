# Fritto TimeTracker — AI Plugin

Connects your AI coding agent to the Fritto (TimeTracker) MCP server and provides
tool-usage guidance for time tracking, approvals, user management, and reporting.

## What's Included

- **MCP server connection** — pre-configured HTTP transport with OAuth 2.0
- **Three role-based skills** — each documents only the tools its role needs, to keep
  context focused:
  - `time-submitter` — track your own time (log, edit, submit, own reports)
  - `time-reviewer` — review/approve team time, act on behalf, team reports & exports
  - `admin` — everything, plus user management, cost rates, project assignments, groups

## Prerequisites

- A Fritto instance with the MCP server enabled (default endpoint:
  `https://mcp.steinpilz-fritto.de/mcp`). Self-hosters set `TIMETRACKER_MCP_URL`.

## Install

### Claude Code

```
/plugin marketplace add Steinpilz/fritto-public
/plugin install fritto-timetracker@fritto
```

Invoke a role skill manually any time:

```
/fritto-timetracker:time-submitter
/fritto-timetracker:time-reviewer
/fritto-timetracker:admin
```

### GitHub Copilot CLI

```
copilot plugin marketplace add Steinpilz/fritto-public
```

Then install `fritto-timetracker` from the `fritto` marketplace.

### Cursor (manual)

Cursor reads MCP servers from `~/.cursor/mcp.json`. Add:

```json
{
  "mcpServers": {
    "fritto-timetracker": {
      "type": "http",
      "url": "https://mcp.steinpilz-fritto.de/mcp"
    }
  }
}
```

For tool guidance in Cursor, copy the relevant role skill (e.g.
`skills/time-submitter/SKILL.md`) into your project's `.cursor/skills/<name>/SKILL.md`
(or your personal skills directory).

## Configuration

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TIMETRACKER_MCP_URL` | No | `https://mcp.steinpilz-fritto.de/mcp` | MCP server endpoint URL |
| `TIMETRACKER_API_KEY` | No | — | API key for non-interactive auth (see below) |

## Authentication

Uses **OAuth 2.0** by default. On first connection your agent opens a browser to
sign in with your Fritto credentials. The MCP server implements:

- OAuth 2.0 Authorization Code flow with PKCE
- Dynamic Client Registration (RFC 7591)
- Auto-discovery via `/.well-known/oauth-authorization-server`

### Alternative: API Key

For non-interactive use, switch `.mcp.json` to header auth:

```json
{
  "mcpServers": {
    "fritto-timetracker": {
      "type": "http",
      "url": "${TIMETRACKER_MCP_URL:-https://mcp.steinpilz-fritto.de/mcp}",
      "headers": { "X-Api-Key": "${TIMETRACKER_API_KEY}" }
    }
  }
}
```

Set `TIMETRACKER_API_KEY`. Format: `{UUID}.{Secret}` (from the Fritto admin panel).

## Tools

29 MCP tools total, scoped across the three skills:

- **`time-submitter`** — log time, view/edit own entries, submit for approval, own reports
- **`time-reviewer`** — submitter tools (+ on-behalf), search pending, approve/decline, team reports & exports
- **`admin`** — all of the above, plus user CRUD, cost rates, project assignments, employee groups
