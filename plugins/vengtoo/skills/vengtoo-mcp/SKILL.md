---
name: vengtoo-mcp
description: |
  Wires Vengtoo authorization into MCP tool calls — every tool invocation by an AI
  agent is checked before executing. Supports Gateway mode (zero server changes) and
  Adapter mode (embedded in your MCP server). Also covers non-MCP agent frameworks.

  Triggers: "authorize tool calls", "mcp gateway", "mcp authorization", "gate tool use",
  "authorize my ai agent", "mcp adapter", "protect mcp tools", "vengtoo mcp",
  "ai agent authorization", "tool call policy", "langchain authorization",
  "langgraph authorization", "prevent agent from calling", "restrict tool access"
metadata:
  category: [ai-agents, mcp, integration]
---

# Authorize MCP Tool Calls

Every tool call an AI agent makes can be checked against your Vengtoo policies before it executes.
Two approaches — choose based on your setup:

| Approach | When to use |
|---|---|
| **Gateway** | You don't control the MCP server code. Drop-in proxy, zero server changes. |
| **Adapter** | You own the MCP server. Embed authorization directly into the server. |

---

## Reference routing

| User need | Reference file |
|---|---|
| Gateway deep-dive (config, policy generation) | `references/gateway.md` |
| Adapter deep-dive (wrapping tools, ABAC on args) | `references/adapter.md` |
| Non-MCP tools (LangChain, LangGraph, custom agents) | `references/non-mcp.md` |

---

## API key — tell the user this upfront

Before doing anything else, tell the user:

> "The Vengtoo MCP server reads `VENGTOO_API_KEY` from your shell environment.
> If you haven't set it yet, add this to `~/.zshrc` (or `~/.zprofile`):
> ```bash
> export VENGTOO_API_KEY=azx_...
> ```
> Then restart your terminal and Claude Code so the MCP server picks it up.
> A `.env` file in your project is **not** enough — MCP servers don't load `.env` files automatically."

Do not attempt to read `settings.json`, modify any Claude config files, or hardcode the key anywhere. The shell export is the correct and only approach.

---

## Flow

### Step 1: Detect setup (current directory only)

Look **only in the current working directory** — do not traverse parent directories, sibling folders, or any path outside the project root.

Check for:
- `.mcp.json` in the project root → MCP client config exists
- `.cursor/mcp.json` in the project root → Cursor MCP config
- `package.json` in the project root → check `dependencies` for `@modelcontextprotocol/sdk`

If none of these are found, **stop and ask the user** — do not search further:
> "Are you setting this up in an existing MCP server you own, or wiring authorization into an MCP client config (like Claude Code or Cursor)?"

### Step 2: Choose approach

**Gateway** (recommended default unless user owns the server):
- Zero code changes to existing MCP servers
- Intercepts all tool calls at the transport layer
- Single config file manages all downstream servers
- Works with any MCP client (Claude Code, Cursor, VS Code, Claude Desktop)

**Adapter** (when user owns the MCP server):
- Authorization logic embedded in the server
- Tool arguments available for ABAC conditions (e.g., allow `amount < 100`)
- More granular; works without a separate proxy process

---

## Gateway setup

### Install
```bash
npm install -g @vengtoo/mcp-gateway
```

### Create `gateway.config.json`
```json
{
  "vengtoo": {
    "cloudUrl": "https://api.vengtoo.com/access/v1/evaluation",
    "apiKey": "azx_...",
    "subject": "agent:claude-code",
    "subjectType": "agent",
    "resourceType": "mcp_tool"
  },
  "servers": {
    "database": {
      "command": "node",
      "args": ["./my-database-server.js"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    }
  }
}
```

### Generate starter policy
```bash
npx vengtoo-mcp-gateway --config ./gateway.config.json --generate-policy ./authz.rego
```

This discovers all tools and classifies them by trust level:
- **LOW** (read-only tools like `list_files`, `get_user`) → allowed by default
- **MEDIUM** (write operations like `create_record`, `send_email`) → requires explicit grant
- **HIGH** (destructive ops like `delete_table`, `drop_database`) → commented out, denied by default

### Wire into MCP client

**Claude Code:**
```bash
claude mcp add --transport stdio vengtoo-gateway -- \
  npx vengtoo-mcp-gateway --config ./gateway.config.json
```

**Cursor** (`.cursor/mcp.json`):
```json
{
  "mcpServers": {
    "vengtoo-gateway": {
      "command": "npx",
      "args": ["vengtoo-mcp-gateway", "--config", "./gateway.config.json"]
    }
  }
}
```

### Create policies in Vengtoo

Use the MCP tools to create:
1. Resource type: `mcp_tool` with action `invoke`
2. Resources: one per tool (e.g., `database__query`, `filesystem__read_file`)
3. Subject: the agent identity (e.g., `agent:claude-code`)
4. Policies: ALLOW specific tools, DENY destructive ones

Or run `/vengtoo-policies` to be guided through this.

---

## Adapter setup

### Install
```bash
npm install @vengtoo/mcp-adapter
```

### Wrap all tools (one line)
```ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { createVengtooAdapter } from '@vengtoo/mcp-adapter'

const server = new McpServer({ name: 'my-server', version: '1.0.0' })

const vengtoo = createVengtooAdapter({
  apiKey: process.env.VENGTOO_API_KEY!,
  subject: 'agent:claude-code',
  resourceType: 'mcp_tool',
})

vengtoo.wrapAllTools(server)  // every tool call now goes through Vengtoo
```

### ABAC on tool arguments
```ts
// In Vengtoo policy, add a condition:
// type: resource_attribute
// field: amount
// operator: lt
// value_json: "1000"

// The adapter automatically passes tool args as resource.properties:
// resource.properties.amount = <whatever the agent sent>
```

---

## Non-MCP tools (LangChain / LangGraph / custom)

Load `references/non-mcp.md` for full patterns. Quick example:

```python
from vengtoo import Vengtoo

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def vengtoo_tool(name: str, fn, args: dict, agent_id: str):
    resp = client.authorize({
        "subject": {"id": agent_id, "type": "agent"},
        "resource": {"type": "tool", "id": name, "properties": args},
        "action": {"name": "invoke"},
    })
    if not resp.decision:
        raise PermissionError(f"Tool '{name}' denied: {resp.context.reason}")
    return fn(**args)
```

---

## Error handling

| Error | Cause | Fix |
|---|---|---|
| All tool calls denied | No policies exist for `mcp_tool` type | Run `/vengtoo-policies` to create them |
| Gateway not in MCP panel | Not added to client config | Re-run `claude mcp add` or check config file |
| Tool name mismatch | Gateway uses `server__tool` format | Policy resource name must match exactly |
| ABAC condition fails | Tool arg value doesn't match condition | Check condition operator and value; test with `/vengtoo-debug` |

---

## When NOT to use this skill

- **Setting up the SDK in a regular API** → use `/vengtoo`
- **Creating the underlying policies** → use `/vengtoo-policies` (this skill handles the MCP-specific wiring)
- **Debugging a denied tool call** → use `/vengtoo-debug`
