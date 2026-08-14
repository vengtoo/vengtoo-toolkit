# Vengtoo MCP Gateway — Reference

The gateway is a local proxy that sits between your MCP client and one or more MCP servers.
Every tool call passes through it and is checked against Vengtoo before the server sees it.

---

## Architecture

```
MCP Client (Claude Code / Cursor)
        ↓
vengtoo-mcp-gateway  ←→  Vengtoo Cloud / Agent
        ↓
Downstream MCP servers (any number)
```

The client connects to one server: the gateway. The gateway manages all downstream connections.

---

## Full `gateway.config.json`

```json
{
  "vengtoo": {
    "cloudUrl": "https://api.vengtoo.com/access/v1/evaluation",
    "agentUrl": "http://localhost:8181/access/v1/evaluation",
    "apiKey": "vgt_...",
    "subject": "agent:claude-code",
    "subjectType": "agent",
    "resourceType": "mcp_tool",
    "failOpen": false,
    "timeout": 500,
    "preferAgent": true
  },
  "audit": {
    "enabled": true,
    "logDenied": true,
    "logAllowed": false
  },
  "servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
      "overrides": {
        "write_file": { "allow": false },
        "delete_file": { "allow": false }
      }
    },
    "database": {
      "command": "node",
      "args": ["./db-server.js"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

### Key fields

| Field | Default | Notes |
|---|---|---|
| `cloudUrl` | api.vengtoo.com | Used when agent unreachable or `preferAgent: false` |
| `agentUrl` | — | Local agent URL; used when `preferAgent: true` |
| `preferAgent` | false | Try agent first, fall back to cloud |
| `failOpen` | false | If true, allows on timeout/error. **Not recommended for production.** |
| `timeout` | 500ms | Per-evaluation timeout; gateway denies if exceeded and `failOpen: false` |
| `overrides` | — | Static allow/deny per tool, bypassing Vengtoo (escape hatch) |

---

## Tool naming convention

The gateway names resources as `{server}__{tool_name}`:
- Server `filesystem`, tool `read_file` → resource ID: `filesystem__read_file`
- Server `database`, tool `query` → resource ID: `database__query`

Create Vengtoo resources using this exact naming so policies match.

---

## Reviewing discovered tools (cloud-governed)

When the gateway is configured with an `apiKey`/`cloudUrl`, it syncs discovered
tools to Vengtoo automatically on startup. They arrive as **pending** (denied by
default) and Vengtoo classifies each tool's risk server-side. You do not generate
a policy file — you review the pending tools and approve/allow/block them:

```
list_pending_tools for gateway <gateway_id>
```

Then per tool:
- **Allow** → `create_policy` (ALLOW, resource `{server}__{tool}`, action `invoke`)
  **plus** `approve_tool` to set the drift baseline.
- **Block** → `block_tool` (hard deny at the gateway).
- **Leave pending** → denied by default until reviewed.

`approve_tool` only baselines the schema for drift detection — it does not make a
tool callable. The ALLOW policy is what authorizes invocation.

---

## Offline alternative — generate a local policy file

For the standalone local agent (no cloud governance), generate a starter YAML
policy to hand-edit instead:

```bash
npx vengtoo-mcp-gateway --config ./gateway.config.json --generate-policy ./policy.yaml
```

It lists all tools, classifies them (LOW allowed, MEDIUM gated, HIGH disabled by
default), and writes a YAML policy for `vengtoo-agent --policy ./policy.yaml`.
Review before use.

---

## Client config reference

**Claude Code** (add once per project):
```bash
claude mcp add --transport stdio vengtoo-gateway -- \
  npx @vengtoo/mcp-gateway --config ./gateway.config.json
```

**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "vengtoo-gateway": {
      "command": "npx",
      "args": ["@vengtoo/mcp-gateway", "--config", "/absolute/path/to/gateway.config.json"]
    }
  }
}
```

**VS Code** (`.vscode/mcp.json`):
```json
{
  "servers": {
    "vengtoo-gateway": {
      "type": "stdio",
      "command": "npx",
      "args": ["@vengtoo/mcp-gateway", "--config", "${workspaceFolder}/gateway.config.json"]
    }
  }
}
```

---

## Logs and debugging

```bash
# Run in foreground with debug output
VENGTOO_GATEWAY_LOG=debug npx @vengtoo/mcp-gateway --config ./gateway.config.json

# Check decision log (requires audit.logDenied: true)
cat ~/.vengtoo/gateway-audit.jsonl | jq 'select(.decision == false)'
```

Each log line includes: `tool`, `subject`, `decision`, `reason`, `latency_ms`.
