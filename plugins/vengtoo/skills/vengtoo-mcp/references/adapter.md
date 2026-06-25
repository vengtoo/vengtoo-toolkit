# Vengtoo MCP Adapter — Reference

The adapter is an npm package that wraps tool handlers inside your MCP server.
It intercepts calls before the handler executes and evaluates them with Vengtoo.

---

## Install

```bash
npm install @vengtoo/mcp-adapter
```

---

## Setup

```ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { createVengtooAdapter } from '@vengtoo/mcp-adapter'

const server = new McpServer({ name: 'my-server', version: '1.0.0' })

const vengtoo = createVengtooAdapter({
  apiKey: process.env.VENGTOO_API_KEY!,
  subject: 'agent:claude-code',
  subjectType: 'agent',
  resourceType: 'mcp_tool',
  // baseUrl: 'http://localhost:8181',  // use local agent
  failOpen: false,
})
```

---

## Wrapping tools

### Wrap all at once (recommended)
```ts
vengtoo.wrapAllTools(server)

// Register tools AFTER wrapping
server.tool('read_file', { path: z.string() }, async ({ path }) => {
  return { content: [{ type: 'text', text: fs.readFileSync(path, 'utf-8') }] }
})
```

### Wrap selectively
```ts
// Wrap only specific tools
vengtoo.wrap(server, ['delete_record', 'update_user', 'execute_query'])
```

### Exclude from wrapping
```ts
vengtoo.wrapAllTools(server, {
  exclude: ['health_check', 'list_schemas'],  // low-risk tools; skip auth
})
```

---

## ABAC on tool arguments

Tool arguments are automatically passed as `resource.properties` in the evaluation request.
Use them in Vengtoo policy conditions:

**Example: Allow `transfer_funds` only when `amount < 10000`**

1. Create a policy for `mcp_tool` / `transfer_funds` / action `invoke`
2. Add condition:
   - Type: `resource_attribute`
   - Field: `amount`
   - Operator: `lt`
   - Value: `10000`

The adapter sends:
```json
{
  "subject": { "id": "agent:claude-code", "type": "agent" },
  "resource": {
    "type": "mcp_tool",
    "id": "transfer_funds",
    "properties": { "amount": 500, "to_account": "acc-9f3" }
  },
  "action": { "name": "invoke" }
}
```

---

## Dynamic subject per request

If different agents/users call the server and you have their identity at tool-call time:

```ts
const vengtoo = createVengtooAdapter({
  apiKey: process.env.VENGTOO_API_KEY!,
  resourceType: 'mcp_tool',
  subjectResolver: (toolName, args, requestContext) => {
    // requestContext is headers / session metadata passed by your transport
    return {
      id: requestContext.agentId ?? 'agent:unknown',
      type: 'agent',
    }
  },
})
```

---

## Error shape

When a tool call is denied, the adapter returns a standard MCP error:

```json
{
  "error": {
    "code": -32603,
    "message": "Access denied: NO_POLICY_MATCH",
    "data": {
      "tool": "delete_record",
      "reason": "No matching policy",
      "reason_code": "NO_POLICY_MATCH"
    }
  }
}
```

The MCP client (Claude Code, Cursor, etc.) surfaces this as a tool error.

---

## Local agent integration

```ts
const vengtoo = createVengtooAdapter({
  apiKey: process.env.VENGTOO_API_KEY!,
  baseUrl: process.env.VENGTOO_BASE_URL ?? 'http://localhost:8181',
  resourceType: 'mcp_tool',
  subject: 'agent:claude-code',
})
```

Set `VENGTOO_BASE_URL=http://localhost:8181` in `.env` when using the local agent.

---

## Full server example

```ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { createVengtooAdapter } from '@vengtoo/mcp-adapter'
import { z } from 'zod'
import fs from 'fs'

const server = new McpServer({ name: 'secure-fs', version: '1.0.0' })
const vengtoo = createVengtooAdapter({
  apiKey: process.env.VENGTOO_API_KEY!,
  subject: 'agent:claude-code',
  resourceType: 'mcp_tool',
})

vengtoo.wrapAllTools(server)

server.tool('read_file', { path: z.string() }, async ({ path }) => ({
  content: [{ type: 'text', text: fs.readFileSync(path, 'utf-8') }],
}))

server.tool('write_file', { path: z.string(), content: z.string() }, async ({ path, content }) => {
  fs.writeFileSync(path, content)
  return { content: [{ type: 'text', text: 'written' }] }
})

server.tool('delete_file', { path: z.string() }, async ({ path }) => {
  fs.unlinkSync(path)
  return { content: [{ type: 'text', text: 'deleted' }] }
})

const transport = new StdioServerTransport()
await server.connect(transport)
```

With Vengtoo wrapping: `read_file` might be allowed while `write_file` and `delete_file` require explicit grants.
