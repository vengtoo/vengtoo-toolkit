---
name: vengtoo-ai-agents
description: |
  Advanced authorization patterns for AI agents: delegation chains (agent acting on
  behalf of a user), human-in-the-loop escalation, JIT time-boxed access grants,
  and multi-agent trust boundaries. Use after the base SDK integration is in place.

  Triggers: "agent acting on behalf of user", "delegation", "agent trust",
  "human in the loop", "human approval", "jit access", "time-boxed grant",
  "temporary access", "multi-agent authorization", "agent impersonation",
  "agent permissions", "escalate to human", "ai agent trust boundary",
  "sub-agent authorization", "orchestrator permissions", "agent scoping"
metadata:
  category: [ai-agents, advanced]
---

# AI Agent Authorization — Advanced Patterns

Three patterns for production AI agent deployments. Apply them in combination.

---

## Reference routing

| User need | Reference file |
|---|---|
| Delegation chains (agent-on-behalf-of) | `references/delegation.md` |
| Human-in-the-loop escalation flows | `references/hitl.md` |
| JIT time-boxed access grants | `references/jit.md` |

---

## Overview of the three patterns

```
1. DELEGATION    — Agent acts as a user, scoped to the user's permissions
2. HITL          — Risky tool calls require human approval before proceeding
3. JIT           — Temporary elevated access with automatic expiry
```

These are independent — use one, two, or all three depending on risk level.

---

## 1. Delegation

An AI agent acts on behalf of a user. The agent gets no more rights than the user has.

**The principle**: the agent + user combination must both have permission. If Alice can't
delete a file, `agent:claude` acting on behalf of Alice also can't delete it.

```ts
// Node.js — pass both agent and user identities
const resp = await vengtoo.authorize({
  subject: {
    id: 'agent:claude-code',
    type: 'agent',
    onBehalfOf: {
      id: userId,
      type: 'user',
    },
  },
  resource: { type: 'document', id: documentId },
  action: { name: 'write' },
})
```

```python
# Python
resp = client.authorize({
    "subject": {
        "id": "agent:research-bot",
        "type": "agent",
        "on_behalf_of": {
            "id": user_id,
            "type": "user",
        },
    },
    "resource": {"type": "report", "id": report_id},
    "action": {"name": "export"},
})
```

**Policy setup**: Create two policies:
1. Policy for the agent role (what agents are allowed to do in general)
2. Policy for the user role (user's existing permissions)

Vengtoo intersects both: the agent can only do what BOTH the agent policy AND the user policy allow.

Load `references/delegation.md` for multi-hop delegation (orchestrator → sub-agent → tool).

---

## 2. Human-in-the-loop

Before executing a high-risk tool, the agent pauses and asks a human.

**When to use**: when `reason_code == "INSUFFICIENT_PRIVILEGE"` or for specific tools
you've pre-classified as requiring human review regardless of policy.

### Basic pattern (any language)
```python
from vengtoo import Vengtoo

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

async def call_tool_with_hitl(tool_name: str, args: dict, agent_id: str, user_id: str):
    resp = client.authorize({
        "subject": {"id": agent_id, "type": "agent", "on_behalf_of": {"id": user_id, "type": "user"}},
        "resource": {"type": "tool", "id": tool_name, "properties": args},
        "action": {"name": "invoke"},
    })

    if resp.decision:
        return await execute_tool(tool_name, args)

    if resp.context.reason_code == "INSUFFICIENT_PRIVILEGE":
        approved = await request_human_approval(
            user_id=user_id,
            tool_name=tool_name,
            args=args,
            reason=resp.context.reason,
        )
        if approved:
            return await execute_tool(tool_name, args)

    raise PermissionError(f"Denied: {resp.context.reason}")
```

### Pre-classify risky tools (proactive HITL)
```ts
const HIGH_RISK_TOOLS = new Set(['send_email', 'delete_record', 'transfer_funds', 'deploy'])

async function callTool(name: string, args: unknown, agentId: string, userId: string) {
  // Always require human approval for high-risk tools
  if (HIGH_RISK_TOOLS.has(name)) {
    const approved = await requestHumanApproval({ name, args, userId })
    if (!approved) throw new Error(`Tool '${name}' requires human approval`)
  }

  // Still check Vengtoo regardless
  const resp = await vengtoo.authorize({
    subject: { id: agentId, type: 'agent', onBehalfOf: { id: userId, type: 'user' } },
    resource: { type: 'tool', id: name, properties: args },
    action: { name: 'invoke' },
  })
  if (!resp.decision) throw new Error(`Denied: ${resp.context.reason}`)

  return executeTool(name, args)
}
```

Load `references/hitl.md` for approval notification integrations (Slack, email, webhook).

---

## 3. JIT (Just-in-Time) time-boxed access

Grant temporary elevated access that automatically expires.

**Built in Vengtoo via**: Direct Grant + TTL on the policy assignment.

```bash
# Via MCP tool: create_policy_assignment
# policy_id: <elevated-access-policy-id>
# entity_type: "subject"
# entity_id: <agent-subject-id>
# expires_at: "2026-06-25T14:00:00Z"   ← auto-expires
```

Or via SDK:
```python
# Grant time-boxed access (expires in 2 hours)
from datetime import datetime, timedelta, timezone

expires = datetime.now(timezone.utc) + timedelta(hours=2)

client.create_policy_assignment(
    policy_id="elevated-access-policy-id",
    entity_type="subject",
    entity_id=agent_id,
    expires_at=expires.isoformat(),
)
```

After `expires_at`, the assignment is automatically ignored. No cleanup needed.

**JIT request flow:**
1. Agent requests elevated access for a task
2. System (or human approver) creates a time-boxed assignment
3. Agent performs the task
4. Assignment expires automatically at the specified time

Load `references/jit.md` for the full request → approve → grant → expire flow.

---

## Multi-agent trust boundaries

When an orchestrator spawns sub-agents, each sub-agent should have its own identity
and be evaluated independently. Sub-agents should NOT inherit the orchestrator's full permissions.

```
orchestrator (agent:orchestrator)    →  full plan execution rights
  sub-agent-1 (agent:data-fetcher)  →  read-only, no write
  sub-agent-2 (agent:emailer)       →  send email only, no data read
  sub-agent-3 (agent:writer)        →  write to draft store only
```

**Setup:**
1. Create a subject for each agent identity with its own role
2. Assign the minimal policy needed for that agent's job
3. Each agent passes its own `agent_id` in evaluation requests

```ts
// Sub-agent evaluates with its own identity, not the orchestrator's
const resp = await vengtoo.authorize({
  subject: { id: 'agent:data-fetcher', type: 'agent' },
  resource: { type: 'database', id: 'prod-db' },
  action: { name: 'query' },
})
```

---

## Error handling

| Scenario | Handle by |
|---|---|
| `INSUFFICIENT_PRIVILEGE` | Trigger HITL approval flow |
| `DELEGATION_DEPTH_EXCEEDED` | Sub-agent chain too deep; flatten agent hierarchy |
| `DELEGATION_NOT_PERMITTED` | Agent role doesn't allow on_behalf_of; add delegation policy |
| Agent timeout calling Vengtoo | Fail closed — deny the action; log for review |
| JIT access not active yet | Check `expires_at` timezone handling (always use UTC) |

---

## When NOT to use this skill

- **Basic SDK setup** → use `/vengtoo`
- **Authorizing tool calls (MCP)** → use `/vengtoo-mcp`
- **Creating the policies** → use `/vengtoo-policies`
- **Debugging a specific failure** → use `/vengtoo-debug`
