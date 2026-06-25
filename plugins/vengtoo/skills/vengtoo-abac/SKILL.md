---
name: vengtoo-abac
description: |
  Adds attribute-based conditions to existing Vengtoo policies — time windows,
  IP ranges, resource properties, environment checks, and custom attributes.
  Use this when role-based allow/deny is not granular enough.

  Triggers: "add condition to policy", "time-based access", "restrict by ip",
  "only allow during business hours", "department attribute", "abac condition",
  "attribute based", "condition failed", "policy condition", "resource property check",
  "environment condition", "tenant isolation", "row level security", "contextual access"
metadata:
  category: [policy, abac]
---

# Add Attribute-Based Conditions

ABAC conditions refine a policy: instead of "Alice can read documents", you get
"Alice can read documents WHEN she is in the Finance department AND it is a weekday."

Conditions are attached to an existing policy. The base allow/deny still applies — the
condition is an additional gate that must pass.

---

## Reference routing

| User need | Reference file |
|---|---|
| Full conditions catalog (all types and operators) | `references/conditions-catalog.md` |

---

## Flow

### Step 1: Identify the policy to refine

Check existing policies via MCP tools. Report which policy (name + ID) will be modified.
Confirm with the user before adding conditions.

### Step 2: Determine what attribute controls access

Ask: "What should determine whether this action is allowed?"

Common answers:
- **Time**: only during business hours, only on weekdays
- **IP / location**: only from corporate network, only from specific country
- **Resource property**: only documents in a certain state, amount below a threshold
- **Subject attribute**: only users in a specific department or with a specific clearance
- **Environment**: only in production namespace, only when request comes via internal network

### Step 3: Check whether the attribute is already on the resource/subject

For `resource_attribute` conditions: the attribute must be set on the resource as metadata.
For `subject_attribute` conditions: the attribute must be set on the subject.

If not set, set it first:
```bash
# Via MCP tool: update_resource
# id: <resource_id>
# metadata: { "status": "published", "department": "finance" }
```

### Step 4: Add the condition

Use the MCP tool `add_policy_condition` or update the policy:

```bash
# Via MCP tool: update_policy
# id: <policy_id>
# conditions: [
#   {
#     "type": "time_of_day",
#     "from": "09:00",
#     "to": "17:00",
#     "timezone": "America/New_York",
#     "days": ["Mon", "Tue", "Wed", "Thu", "Fri"]
#   }
# ]
```

Multiple conditions are ANDed by default (all must pass).

### Step 5: Verify

Test at the boundary — both cases must behave correctly:

```bash
# Test should-pass case
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "alice", "type": "user"},
    "resource": {"type": "document", "id": "doc-42"},
    "action": {"name": "read"},
    "context": {"current_time": "2026-06-25T14:00:00-04:00"}
  }'

# Test should-deny case (outside hours)
# ... same but with "current_time": "2026-06-25T20:00:00-04:00"
```

---

## Common condition recipes

### Business hours only
```json
{
  "type": "time_of_day",
  "from": "09:00",
  "to": "17:00",
  "timezone": "America/New_York",
  "days": ["Mon", "Tue", "Wed", "Thu", "Fri"]
}
```

### IP allowlist (corporate network)
```json
{
  "type": "ip_range",
  "ranges": ["10.0.0.0/8", "172.16.0.0/12"],
  "source": "request_context"
}
```

Pass client IP in the evaluation request context:
```json
{ "context": { "ip": "10.0.1.55" } }
```

### Resource amount ceiling (finance approval)
```json
{
  "type": "resource_attribute",
  "field": "amount",
  "operator": "lte",
  "value_json": "10000"
}
```

Resource must have `metadata.amount` set.

### Department match (HR can see HR records only)
```json
{
  "type": "subject_attribute",
  "field": "department",
  "operator": "eq",
  "value_json": "\"hr\""
}
```

Subject must have `metadata.department` set.

### Status gate (only published documents are readable)
```json
{
  "type": "resource_attribute",
  "field": "status",
  "operator": "eq",
  "value_json": "\"published\""
}
```

### TTL / expiring access (JIT)
```json
{
  "type": "valid_until",
  "timestamp": "2026-06-30T00:00:00Z"
}
```

---

## Condition logic

| Behaviour | How to achieve |
|---|---|
| All conditions must pass (AND) | Add multiple conditions to the same policy |
| Any condition can pass (OR) | Create two separate policies at the same priority |
| Condition negation | Use `operator: "neq"` or `operator: "not_in"` |
| Deny override | Create a DENY policy at priority 80 with the inverse condition |

---

## Error handling

| Error | Cause | Fix |
|---|---|---|
| `CONDITION_FAILED` in response | Condition evaluated but did not pass | Check attribute values; test boundary values |
| `CONDITION_ERROR` | Attribute referenced doesn't exist | Set attribute on subject/resource first |
| Condition ignored (always passes) | `value_json` type mismatch | Ensure numeric values use numbers, strings use quotes: `"\"published\""` |
| Context attribute not evaluated | Client not passing `context` in request | Add `"context": {"ip": ..., "current_time": ...}` to eval request |

---

## Passing context from the SDK

**Node.js:**
```ts
const resp = await vengtoo.authorize({
  subject: { id: userId, type: 'user' },
  resource: { type: 'document', id: documentId },
  action: { name: 'read' },
  context: {
    ip: req.ip,
    current_time: new Date().toISOString(),
  },
})
```

**Python:**
```python
resp = client.authorize({
    "subject": {"id": user_id, "type": "user"},
    "resource": {"type": "document", "id": document_id},
    "action": {"name": "read"},
    "context": {
        "ip": request.client.host,
        "current_time": datetime.utcnow().isoformat() + "Z",
    },
})
```

**Go:**
```go
resp, err := client.Authorize(ctx, &vengtoo.AuthorizeRequest{
    Subject:  vengtoo.Subject{ID: userID, Type: "user"},
    Resource: vengtoo.Resource{Type: "document", ID: documentID},
    Action:   vengtoo.Action{Name: "read"},
    Context:  map[string]interface{}{
        "ip":           r.RemoteAddr,
        "current_time": time.Now().UTC().Format(time.RFC3339),
    },
})
```

---

## When NOT to use this skill

- **Creating the base policy** → use `/vengtoo-policies` first, then return here to add conditions
- **Debugging why a condition failed** → use `/vengtoo-debug`
- **MCP tool argument conditions** → covered in `/vengtoo-mcp` (same mechanism, different setup)
