# Fritto TimeTracker — AI Plugin

Connects your AI coding agent to the Fritto (TimeTracker) MCP server and provides
tool-usage guidance for time tracking, approvals, user management, and reporting.

## What's Included

- **MCP server connection** — pre-configured HTTP transport with OAuth 2.0
- **TimeTracker tools skill** — persona-based guide (time submitter, reviewer/approver,
  company admin) covering 29 MCP tools, with workflows and error handling

## Prerequisites

- A Fritto instance with the MCP server enabled (default endpoint:
  `https://mcp.steinpilz-fritto.de/mcp`). Self-hosters set `TIMETRACKER_MCP_URL`.

## Install

### Claude Code

```
/plugin marketplace add Steinpilz/fritto-public
/plugin install fritto-timetracker@fritto
```

Invoke the guidance skill manually any time:

```
/fritto-timetracker:timetracker-tools
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

For tool guidance in Cursor, copy `skills/timetracker-tools/SKILL.md` into your
project's `.cursor/skills/timetracker-tools/SKILL.md` (or your personal skills directory).

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

The skill covers 29 MCP tools organized by role:

- **Time Submitter** — log time, view/edit entries, submit for approval
- **Reviewer/Approver** — search pending, approve/decline, reports, exports
- **Company Admin** — user CRUD, cost rates, project assignments, employee groups
