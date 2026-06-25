# Vengtoo ABAC Conditions Catalog

All supported condition types, their fields, and valid operators.

---

## Condition types

### `time_of_day`

Restrict access to a time window within a day.

```json
{
  "type": "time_of_day",
  "from": "09:00",
  "to": "17:00",
  "timezone": "America/New_York",
  "days": ["Mon", "Tue", "Wed", "Thu", "Fri"]
}
```

| Field | Required | Notes |
|---|---|---|
| `from` | yes | HH:MM in 24h format |
| `to` | yes | HH:MM; if `to` < `from`, wraps midnight |
| `timezone` | no | IANA timezone string; defaults to UTC |
| `days` | no | Mon/Tue/Wed/Thu/Fri/Sat/Sun; omit to allow all days |

Pass `current_time` in evaluation `context`:
```json
{ "context": { "current_time": "2026-06-25T14:30:00Z" } }
```

---

### `valid_until` (JIT expiry)

Policy condition that passes only before a specific timestamp.

```json
{
  "type": "valid_until",
  "timestamp": "2026-07-01T00:00:00Z"
}
```

| Field | Required | Notes |
|---|---|---|
| `timestamp` | yes | RFC3339 UTC timestamp |

Unlike `expires_at` on an assignment (which removes the assignment), `valid_until` on a condition keeps the assignment but conditions the effect. Use `expires_at` on the assignment for most JIT use cases.

---

### `ip_range`

Restrict access based on source IP address.

```json
{
  "type": "ip_range",
  "ranges": ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"],
  "operator": "in"
}
```

| Field | Required | Notes |
|---|---|---|
| `ranges` | yes | Array of CIDR blocks |
| `operator` | no | `in` (allowlist) or `not_in` (blocklist); default `in` |

Pass `ip` in evaluation `context`:
```json
{ "context": { "ip": "10.0.1.55" } }
```

---

### `resource_attribute`

Compare a resource's metadata field to a value.

```json
{
  "type": "resource_attribute",
  "field": "status",
  "operator": "eq",
  "value_json": "\"published\""
}
```

| Field | Required | Notes |
|---|---|---|
| `field` | yes | Attribute key in `resource.metadata`. Supports dot-path: `"owner.department"` |
| `operator` | yes | See operators table below |
| `value_json` | yes | JSON-encoded value: strings need inner quotes: `"\"published\""`, numbers: `"100"`, booleans: `"true"` |

**Operators:**
| Operator | Meaning |
|---|---|
| `eq` | Equal |
| `neq` | Not equal |
| `lt` | Less than (numeric) |
| `lte` | Less than or equal |
| `gt` | Greater than (numeric) |
| `gte` | Greater than or equal |
| `in` | Value is in array |
| `not_in` | Value is not in array |
| `contains` | String contains substring |
| `starts_with` | String starts with prefix |

**Array `in` example:**
```json
{
  "type": "resource_attribute",
  "field": "department",
  "operator": "in",
  "value_json": "[\"finance\", \"accounting\"]"
}
```

---

### `subject_attribute`

Compare a subject's metadata field to a value.

```json
{
  "type": "subject_attribute",
  "field": "clearance_level",
  "operator": "gte",
  "value_json": "3"
}
```

Same operators as `resource_attribute`. The `field` references `subject.metadata.<field>`.

Set attributes on the subject:
```bash
# Via MCP tool: update_subject
# id: <subject_id>
# metadata: { "clearance_level": 3, "department": "finance" }
```

---

### `context_attribute`

Compare a value from the evaluation request `context` block.

```json
{
  "type": "context_attribute",
  "field": "environment",
  "operator": "eq",
  "value_json": "\"production\""
}
```

Pass `environment` in the evaluation request:
```json
{
  "subject": {...},
  "resource": {...},
  "action": {...},
  "context": { "environment": "production" }
}
```

Useful for:
- Restricting to specific environments (`environment: production`)
- Trust level checks (`trust_level: high`)
- Custom business context (`tenant_tier: enterprise`)

---

### `mfa_required`

Allow only if the subject's session included MFA.

```json
{
  "type": "mfa_required"
}
```

Pass `mfa_verified: true` in evaluation `context`:
```json
{ "context": { "mfa_verified": true } }
```

If not passed, defaults to `false`.

---

## Combining conditions (AND / OR)

### AND — all conditions must pass (default)

Add multiple conditions to the same policy:
```json
{
  "conditions": [
    { "type": "time_of_day", "from": "09:00", "to": "17:00" },
    { "type": "ip_range", "ranges": ["10.0.0.0/8"] }
  ]
}
```

Both must pass.

### OR — any condition can pass

Create two separate policies with the same effect and priority. Vengtoo evaluates both; if either returns ALLOW, access is granted.

### NOT (negation)

Use `operator: "neq"`, `operator: "not_in"`, or `operator: "not"` (boolean flip):

```json
{
  "type": "resource_attribute",
  "field": "archived",
  "operator": "eq",
  "value_json": "false"
}
```

### DENY override

Create a separate DENY policy at a higher priority (70–90). The DENY policy can have its own conditions.

```
priority 80 DENY: archived == true
priority 50 ALLOW: role editor
```

Result: editors can access everything EXCEPT archived resources.

---

## Setting attributes on resources and subjects

```bash
# Resource attributes (via MCP or REST)
PUT /resource-srv/v1/resources/{id}
{
  "metadata": {
    "status": "published",
    "department": "finance",
    "amount": 5000,
    "archived": false
  }
}

# Subject attributes
PUT /entity-srv/v1/subjects/{id}
{
  "metadata": {
    "clearance_level": 2,
    "department": "hr",
    "region": "eu"
  }
}
```

Attributes are arbitrary key-value pairs. Vengtoo stores them and evaluates conditions against them at decision time.

---

## Debugging conditions

When a decision returns `CONDITION_FAILED`, the response context includes which condition failed:

```json
{
  "decision": false,
  "context": {
    "reason": "Condition failed: time_of_day check",
    "reason_code": "CONDITION_FAILED",
    "failed_condition": {
      "type": "time_of_day",
      "evaluated_value": "2026-06-25T21:00:00Z",
      "expected": { "from": "09:00", "to": "17:00" }
    }
  }
}
```

Use this to identify which condition is blocking and verify the attribute values.
