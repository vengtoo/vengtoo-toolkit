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

Conditions are attached to an existing policy via the `conditions` field — a flat JSON
object where each key is an active guardrail or attribute check. All present keys must
pass (AND logic).

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
- **Environment**: only in production, only when request comes via internal network

### Step 3: Make sure the attribute is defined and its value is set

There are two distinct steps — **defining** the attribute schema and **setting** the value on the individual subject or resource.

**For subject attributes:**

First, declare the attribute definition so the dashboard knows about it (powers the subject form dropdown and type hints):

```json
// MCP: define_subject_attribute
{
  "name": "department",
  "type": "string",
  "description": "Employee department for policy scoping",
  "required": false
}
```

This is **loose mode** — the backend never rejects a subject that has an undeclared key. Defining it here is purely for dashboard autocomplete and type validation; it has no effect on evaluation.

Then set the actual value on the subject:

```json
// MCP: update_subject
{
  "id": "<subject_id>",
  "attributes": { "department": "engineering", "clearance": 3 }
}
```

Note: `update_subject.attributes` replaces the whole object — include existing attributes you want to keep.

**For resource attributes:**

Attribute definitions live on the resource type schema. Pass `attribute_schema` when creating the resource type, or check the dashboard (**Schema → Resource Types** detail page) to see what's already declared:

```json
// MCP: create_resource_type
{
  "name": "document",
  "actions": ["read", "write", "delete"],
  "attribute_schema": [
    { "name": "status", "type": "string", "description": "Draft / published / archived" },
    { "name": "classification", "type": "string", "description": "internal / confidential / public" }
  ]
}
```

Again loose mode — resources can carry keys not in the schema. The schema drives the dashboard's resource form and the `resource_attrs` condition picker.

Then set values on the resource with `update_resource`:

```json
// MCP: update_resource
{
  "id": "<resource_id>",
  "attributes": { "status": "published", "classification": "internal" }
}
```

### Step 4: Add the condition

Use the MCP tool `update_policy` with a `conditions` object:

```json
{
  "id": "<policy_id>",
  "conditions": {
    "time_window": {
      "start": "09:00",
      "end": "17:00",
      "days": ["mon", "tue", "wed", "thu", "fri"],
      "tz": "America/New_York"
    }
  }
}
```

Multiple keys in `conditions` are ANDed — all must pass.

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
    "context": {"time": "2026-06-25T14:00:00-04:00"}
  }'

# Test should-deny case (outside hours)
# ... same but with "time": "2026-06-25T20:00:00-04:00"
```

---

## Common condition recipes

All conditions go inside the `conditions` object. Mix and match any keys.

### Business hours only
```json
{
  "conditions": {
    "time_window": {
      "start": "09:00",
      "end": "17:00",
      "days": ["mon", "tue", "wed", "thu", "fri"],
      "tz": "America/New_York"
    }
  }
}
```

### IP allowlist (corporate network)
```json
{
  "conditions": {
    "ip_allowlist": {
      "cidrs": ["10.0.0.0/8", "172.16.0.0/12"]
    }
  }
}
```

Pass client IP in the evaluation request context: `"context": { "ip": "10.0.1.55" }`

### Geo restriction (allow-list)
```json
{
  "conditions": {
    "geo_restriction": {
      "mode": "allow",
      "countries": ["US", "CA", "GB"]
    }
  }
}
```

### Resource amount ceiling (finance approval)
```json
{
  "conditions": {
    "resource_attrs": [
      { "attr": "amount", "op": "<=", "value": 10000 }
    ]
  }
}
```

Resource must have `amount` set as an attribute.

### Department match (HR can see HR records only)
```json
{
  "conditions": {
    "subject_attrs": [
      { "attr": "department", "op": "==", "value": "hr" }
    ]
  }
}
```

Subject must have `department` set as an attribute.

### Status gate (only published documents are readable)
```json
{
  "conditions": {
    "resource_attrs": [
      { "attr": "status", "op": "==", "value": "published" }
    ]
  }
}
```

### Clearance level gate
```json
{
  "conditions": {
    "subject_attrs": [
      { "attr": "clearance", "op": ">=", "value": 3 }
    ]
  }
}
```

### MFA required
```json
{
  "conditions": {
    "mfa_required": {
      "claim_path": "mfa_verified"
    }
  }
}
```

Pass `"subject": { "attributes": { "mfa_verified": true } }` or `"context": { "mfa_verified": true }` in the evaluation request.

### Context attribute check
```json
{
  "conditions": {
    "context_attrs": [
      { "key": "env", "op": "==", "value": "prod" }
    ]
  }
}
```

Pass `"context": { "env": "prod" }` in the evaluation request.

### Combined: department + business hours
```json
{
  "conditions": {
    "subject_attrs": [
      { "attr": "department", "op": "==", "value": "finance" }
    ],
    "time_window": {
      "start": "09:00",
      "end": "18:00",
      "days": ["mon", "tue", "wed", "thu", "fri"],
      "tz": "UTC"
    }
  }
}
```

---

## Attribute operators

| Operator | Meaning |
|---|---|
| `==` | Equal |
| `!=` | Not equal |
| `>=` | Greater than or equal |
| `<=` | Less than or equal |
| `>` | Greater than |
| `<` | Less than |
| `in` | Value in array — `"value"` must be an array |
| `not_in` | Value not in array — `"value"` must be an array |

---

## Condition logic

| Behaviour | How to achieve |
|---|---|
| All conditions must pass (AND) | Add multiple keys to the same `conditions` object |
| Any condition can pass (OR) | Create two separate policies at the same priority |
| Condition negation | Use `op: "!="` or `op: "not_in"` |
| Deny override | Create a DENY policy at priority 80 with the inverse condition |

---

## Error handling

| Error | Cause | Fix |
|---|---|---|
| `CONDITION_FAILED` in response | Condition evaluated but did not pass | Check attribute values; test boundary values |
| Condition ignored (always passes) | Attribute value type mismatch | Ensure numeric values are numbers not strings in the policy |
| Context attribute not evaluated | Client not passing `context` in request | Add `"context": {"ip": ..., "time": ...}` to eval request |
| Missing subject attribute | Subject attribute not stored and not passed inline | Set attribute via dashboard or pass in `subject.attributes` |

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
    time: new Date().toISOString(),
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
        "time": datetime.utcnow().isoformat() + "Z",
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
        "ip":   r.RemoteAddr,
        "time": time.Now().UTC().Format(time.RFC3339),
    },
})
```

---

## When NOT to use this skill

- **Creating the base policy** → use `/vengtoo-policies` first, then return here to add conditions
- **Debugging why a condition failed** → use `/vengtoo-debug`
- **MCP tool argument conditions** → covered in `/vengtoo-mcp` (same mechanism, different setup)
