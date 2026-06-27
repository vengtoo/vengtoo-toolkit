# Vengtoo ABAC Conditions Catalog

All supported condition types, their fields, and valid operators. Conditions are a flat JSON object on the policy `conditions` field — each key is an active check; all present keys must pass (AND).

---

## Attribute conditions

### `subject_attrs`

Checks attributes on the subject at evaluation time.

```json
{
  "conditions": {
    "subject_attrs": [
      { "attr": "department", "op": "==", "value": "engineering" },
      { "attr": "clearance", "op": ">=", "value": 3 }
    ]
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `attr` | yes | Attribute key. Supports dot-path: `"manager.region"` traverses `subject.attributes.manager.region` (up to 4 levels) |
| `op` | yes | Comparison operator (see table below) |
| `value` | yes | Right-hand side. Numbers and booleans as native JSON — not strings |

All rows are ANDed. Pass subject attribute values in `subject.attributes` in the eval request, or store them on the subject via `update_subject`.

---

### `resource_attrs`

Checks attributes on the resource being accessed.

```json
{
  "conditions": {
    "resource_attrs": [
      { "attr": "status", "op": "==", "value": "published" },
      { "attr": "region", "op": "in", "value": ["us-east-1", "us-west-2"] }
    ]
  }
}
```

Same shape as `subject_attrs`. Attribute values sourced from `resource.attributes` in the eval request or stored on the resource via `update_resource`.

---

### `context_attrs`

Checks keys passed in the evaluation request `context` block.

```json
{
  "conditions": {
    "context_attrs": [
      { "key": "env", "op": "==", "value": "prod" }
    ]
  }
}
```

Note: field name is `key` (not `attr`) for context rows.

Pass values as `"context": { "env": "prod" }` in the eval request.

---

## Attribute operators

| Operator | Meaning | Notes |
|---|---|---|
| `==` | Equal | String, number, boolean |
| `!=` | Not equal | |
| `>=` | Greater than or equal | Numeric |
| `<=` | Less than or equal | Numeric |
| `>` | Greater than | Numeric |
| `<` | Less than | Numeric |
| `in` | Value in array | `value` must be a JSON array |
| `not_in` | Value not in array | `value` must be a JSON array |

**Type coercion:** stored attributes that are numbers or booleans but the policy stores the value as a string are coerced automatically — `5 >= "3"` and `true == "true"` both work. Prefer native JSON types in the policy for clarity.

**Missing attributes fail closed:** if the attribute is absent from both stored values and the eval request, the condition fails and access is denied.

---

## Guardrails

### `time_window`

Policy only applies within a configured time range.

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

| Field | Required | Notes |
|---|---|---|
| `start` | yes | HH:MM in 24h format |
| `end` | yes | HH:MM; if `end` < `start`, wraps midnight |
| `days` | no | `mon`/`tue`/`wed`/`thu`/`fri`/`sat`/`sun`; omit to allow all days |
| `tz` | no | IANA timezone string; defaults to UTC |

Pass `"context": { "time": "2026-06-25T14:30:00-04:00" }` in the eval request. Fails closed if `context.time` is absent.

---

### `ip_allowlist`

Policy only applies when `context.ip` is within an allowed CIDR.

```json
{
  "conditions": {
    "ip_allowlist": {
      "cidrs": ["10.0.0.0/8", "172.16.0.0/12"]
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `cidrs` | yes | Array of CIDR blocks |

Pass `"context": { "ip": "10.0.1.55" }` in the eval request. Fails closed if `context.ip` is absent.

---

### `geo_restriction`

Policy only applies when `context.geo` matches the configured country list.

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

| Field | Required | Notes |
|---|---|---|
| `mode` | yes | `allow` — only these countries pass; `deny` — these countries are blocked |
| `countries` | yes | ISO 3166-1 alpha-2 codes |

Derive `context.geo` in your backend from the request IP; pass as `"context": { "geo": "US" }`. Fails closed if absent.

---

### `mfa_required`

Policy only applies when a specified claim is truthy on the subject or context.

```json
{
  "conditions": {
    "mfa_required": {
      "claim_path": "mfa_verified"
    }
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `claim_path` | yes | Key to check; checked first on `subject.attributes`, then on `context` |

Pass `"subject": { "attributes": { "mfa_verified": true } }` or `"context": { "mfa_verified": true }`. Fails closed if key is absent or falsy.

---

### `trust_level`

Policy only applies when the subject meets a minimum trust tier.

```json
{
  "conditions": {
    "trust_level": {
      "enabled": true,
      "minimum": "verified"
    }
  }
}
```

| Level | Rank |
|---|---|
| `new` | 0 — default for subjects with no explicit trust level |
| `verified` | 1 |
| `trusted` | 2 |
| `blocked` | never passes any minimum |

Set a subject's trust level via `PATCH /v1/entities/:id/trust-level`.

---

### `rate_limit`

Policy only applies when the subject has not exceeded a configured request rate.

```json
{
  "conditions": {
    "rate_limit": {
      "requests": 100,
      "window_seconds": 60
    }
  }
}
```

Tracked per subject per policy. Exceeding the limit causes the condition to fail — policy does not match — which effectively denies access for the remainder of the window.

---

### `requires_human_approval`

Policy triggers an approval workflow before granting access. Access is denied until an approver acts.

```json
{
  "conditions": {
    "requires_human_approval": {
      "timeout_seconds": 3600
    }
  }
}
```

---

## Combining conditions

| Behaviour | How |
|---|---|
| AND (all must pass) | Add multiple keys to the same `conditions` object |
| OR (any can pass) | Create two separate policies at the same priority |
| Negation | Use `op: "!="` or `op: "not_in"` in attribute rows |
| Deny override | Create a DENY policy at priority 70–90 with the inverse condition |

**Example — AND (department + business hours):**
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

## `_disabled` soft-off

When you toggle a guardrail off in the dashboard without deleting it, the value is stashed under `conditions._disabled.<key>`. The evaluation engine never reads `_disabled` — it is invisible to all condition checks. Toggling back on moves the key out of `_disabled` and restores the original configuration.

Do not set `_disabled` manually via the API — use the dashboard toggle or omit the key entirely.

---

## Setting attribute values

**Subject attributes** — store on the subject, sourced automatically at eval time:
```bash
# MCP: update_subject
{ "id": "<subject_id>", "attributes": { "department": "engineering", "clearance": 3 } }

# REST
PUT /v1/entities/<id>
{ "attributes": { "department": "engineering", "clearance": 3 } }
```

**Resource attributes** — store on the resource, sourced automatically at eval time:
```bash
# MCP: update_resource
{ "id": "<resource_id>", "attributes": { "status": "published", "classification": "internal" } }

# REST
PUT /v1/resources/<id>
{ "attributes": { "status": "published", "classification": "internal" } }
```

**Inline at eval time** — pass in the eval request body alongside stored values. Stored values win on conflict:
```json
{
  "subject": { "external_id": "alice", "type": "user", "attributes": { "clearance": 3 } },
  "resource": { "external_id": "doc-1", "type": "document", "attributes": { "status": "published" } },
  "action": { "name": "read" },
  "context": { "ip": "10.0.1.55", "time": "2026-06-25T14:00:00Z" }
}
```

---

## Subject attribute definitions (schema, loose mode)

Declare the vocabulary of subject attributes your policies use. Loose mode — the backend does not reject subjects with undeclared keys; definitions exist to power dashboard autocomplete and type validation.

```bash
# MCP: define_subject_attribute
{ "name": "department", "type": "string", "description": "Employee department", "required": false }

# REST
POST /v1/subject-attributes
{ "name": "department", "type": "string", "description": "Employee department", "required": false }
```

Valid types: `string`, `int`, `bool`, `enum`, `object`, `json`.

For resource attributes, declare the schema on the resource type via `attribute_schema` in `create_resource_type`.

---

## Debugging failed conditions

When a policy condition fails, the evaluation response includes a trace. Common signals:

| Symptom | Likely cause |
|---|---|
| `conditions_and_modifiers: false` | A condition key is present but its check failed or its context value is missing |
| Access denied despite correct role | Policy has a condition that fails closed (missing context key) |
| Condition always passes unexpectedly | Attribute key not present in `conditions` (omitted key = not evaluated) |
| `_disabled` key shows in conditions | Guardrail toggled off in dashboard — expected, not a bug |
