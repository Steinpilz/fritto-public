# Fritto Public

Public marketplace of Fritto (TimeTracker) AI plugins and docs for AI coding agents.

## Plugins

| Plugin | Description |
|--------|-------------|
| [`fritto-timetracker`](plugins/fritto-timetracker) | TimeTracker MCP tools: time tracking, approvals, user management, reporting |

## Install

### Claude Code

```
/plugin marketplace add Steinpilz/fritto-public
/plugin install fritto-timetracker@fritto
```

### GitHub Copilot CLI

```
copilot plugin marketplace add Steinpilz/fritto-public
```

Then install `fritto-timetracker` from the `fritto` marketplace.

### Cursor

See [plugins/fritto-timetracker/README.md](plugins/fritto-timetracker/README.md) for manual setup.

## Repository layout

```
.claude-plugin/marketplace.json   Catalog (Claude Code; Copilot fallback)
.github/plugin/marketplace.json   Catalog (Copilot CLI)
plugins/                          One directory per plugin
docs/                             Fritto AI docs (coming soon)
```

## License

[MIT](LICENSE)
