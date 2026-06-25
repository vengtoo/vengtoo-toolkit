# Vengtoo — Agent Delegation Reference

Delegation lets an agent act on behalf of a user. The agent's permissions are intersected
with the user's permissions — the agent can never exceed what the user is allowed to do.

---

## The `on_behalf_of` field

```json
{
  "subject": {
    "id": "agent:claude-code",
    "type": "agent",
    "on_behalf_of": {
      "id": "user-alice",
      "type": "user"
    }
  },
  "resource": {"type": "document", "id": "doc-42"},
  "action": {"name": "write"}
}
```

Vengtoo evaluates:
1. Does `agent:claude-code` have permission to write documents? (agent policy)
2. Does `user-alice` have permission to write `doc-42`? (user policy)
3. Both must be true for `decision: true`

---

## Setting up delegation

### 1. Create the agent subject
```bash
# Via MCP tool: create_subject
# name: "Claude Code"
# type: "agent"
# external_id: "agent:claude-code"
```

### 2. Create the agent role
```bash
# Via MCP tool: create_role
# name: "ai-assistant"
# description: "AI agent that can act on behalf of users"
```

### 3. Create the agent policy

The agent policy defines what actions agents are *capable* of — this is the upper bound.
If the agent policy doesn't include `write`, agents can never write, even if the user can.

```bash
# Via MCP tool: create_policy
# name: "agent-capabilities"
# effect: "ALLOW"
# priority: 50
# resource_types: [{ resource_type_id: <document_type_id>, actions: ["read", "write"] }]
# Allow delegation: true
```

### 4. Assign agent policy to agent role
```bash
# create_policy_assignment: policy=agent-capabilities, entity=role:ai-assistant
# create_role_assignment: subject=agent:claude-code, role=ai-assistant
```

---

## Multi-hop delegation

```
User (Alice) → Orchestrator agent → Sub-agent → Tool
```

The chain is evaluated at each hop. Maximum depth: 3.

```json
{
  "subject": {
    "id": "agent:sub-agent",
    "type": "agent",
    "on_behalf_of": {
      "id": "agent:orchestrator",
      "type": "agent",
      "on_behalf_of": {
        "id": "user-alice",
        "type": "user"
      }
    }
  },
  "resource": {"type": "document", "id": "doc-42"},
  "action": {"name": "read"}
}
```

All three must have `read` permission for `decision: true`.

---

## Agent-specific scoping

Sometimes you want an agent to have a strict subset of the user's permissions.
Use an agent policy with tighter resource targeting:

```hcl
# Vengtoo Terraform
resource "vengtoo_policy" "research_agent_capabilities" {
  name   = "research-agent-read-only"
  effect = "ALLOW"

  resource_types = [
    {
      resource_type_id = vengtoo_resource_type.document.id
      actions          = ["read"]   # read only — even if user can write
    }
  ]
}
```

Even if Alice has write permission, `agent:research-bot` acting on her behalf can only read.

---

## Node.js — full delegation helper

```ts
import { Vengtoo } from '@vengtoo/sdk'

const client = new Vengtoo({ apiKey: process.env.VENGTOO_API_KEY! })

interface DelegationContext {
  agentId: string
  userId: string
  userType?: string
}

async function authorizeAsAgent(
  delegation: DelegationContext,
  resourceType: string,
  resourceId: string,
  action: string,
) {
  return client.authorize({
    subject: {
      id: delegation.agentId,
      type: 'agent',
      onBehalfOf: {
        id: delegation.userId,
        type: delegation.userType ?? 'user',
      },
    },
    resource: { type: resourceType, id: resourceId },
    action: { name: action },
  })
}

// Usage in a tool handler
const resp = await authorizeAsAgent(
  { agentId: 'agent:claude-code', userId: session.userId },
  'document',
  documentId,
  'write',
)
if (!resp.decision) throw new Error(`Denied: ${resp.context.reason}`)
```

---

## Python — full delegation helper

```python
from vengtoo import Vengtoo
import os

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def authorize_as_agent(agent_id: str, user_id: str, resource_type: str, resource_id: str, action: str):
    resp = client.authorize({
        "subject": {
            "id": agent_id,
            "type": "agent",
            "on_behalf_of": {"id": user_id, "type": "user"},
        },
        "resource": {"type": resource_type, "id": resource_id},
        "action": {"name": action},
    })
    return resp

# Usage
resp = authorize_as_agent("agent:research-bot", user_id, "document", doc_id, "read")
if not resp.decision:
    raise PermissionError(f"Denied: {resp.context.reason}")
```

---

## Go — delegation

```go
resp, err := client.Authorize(ctx, &vengtoo.AuthorizeRequest{
    Subject: vengtoo.Subject{
        ID:   "agent:claude-code",
        Type: "agent",
        OnBehalfOf: &vengtoo.Principal{
            ID:   userID,
            Type: "user",
        },
    },
    Resource: vengtoo.Resource{Type: "document", ID: documentID},
    Action:   vengtoo.Action{Name: "write"},
})
```

---

## Error codes specific to delegation

| Code | Cause | Fix |
|---|---|---|
| `DELEGATION_NOT_PERMITTED` | Agent role doesn't have delegation enabled | Add `allow_delegation: true` to agent policy or role |
| `DELEGATION_DEPTH_EXCEEDED` | Chain is longer than 3 hops | Flatten the agent hierarchy |
| `DELEGATED_SUBJECT_NOT_FOUND` | `on_behalf_of.id` doesn't match any subject | Check subject `external_id` |
