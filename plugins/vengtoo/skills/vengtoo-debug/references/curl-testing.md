# Vengtoo Curl Testing Reference

Ready-to-use curl commands for manually testing Vengtoo authorization.

---

## Basic evaluation

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "user-123", "type": "user"},
    "resource": {"type": "document", "id": "doc-456"},
    "action": {"name": "read"}
  }' | jq .
```

---

## With external_id (recommended)

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "your-own-user-uuid", "type": "user"},
    "resource": {"type": "document", "id": "your-own-resource-id"},
    "action": {"name": "read"}
  }' | jq '{decision: .decision, reason: .context.reason, path: .context.access_path}'
```

---

## With context (for ABAC conditions)

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"subject\": {\"id\": \"user-123\", \"type\": \"user\"},
    \"resource\": {\"type\": \"document\", \"id\": \"doc-456\"},
    \"action\": {\"name\": \"read\"},
    \"context\": {
      \"ip\": \"10.0.1.55\",
      \"current_time\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"environment\": \"production\"
    }
  }" | jq .
```

---

## Batch evaluation

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluations \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "user-123", "type": "user"},
    "items": [
      {"resource": {"type": "document", "id": "doc-1"}, "action": {"name": "read"}},
      {"resource": {"type": "document", "id": "doc-1"}, "action": {"name": "write"}},
      {"resource": {"type": "document", "id": "doc-1"}, "action": {"name": "delete"}},
      {"resource": {"type": "invoice", "id": "inv-7"}, "action": {"name": "approve"}}
    ]
  }' | jq '.results[] | {resource: .resource.id, action: .action.name, decision: .decision}'
```

---

## Against local agent

```bash
curl -s -X POST http://localhost:8181/access/v1/evaluation \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "user-123", "type": "user"},
    "resource": {"type": "document", "id": "doc-456"},
    "action": {"name": "read"}
  }' | jq .
```

Note: The local agent doesn't require the `Authorization` header — auth is established at the agent level via `VENGTOO_API_KEY` at startup.

---

## Delegation (agent on behalf of user)

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {
      "id": "agent:claude-code",
      "type": "agent",
      "on_behalf_of": {"id": "user-123", "type": "user"}
    },
    "resource": {"type": "document", "id": "doc-456"},
    "action": {"name": "write"}
  }' | jq .
```

---

## Check all actions for a subject (manual batch)

```bash
SUBJECT_ID="user-123"
RESOURCE_TYPE="document"
RESOURCE_ID="doc-456"
ACTIONS=("read" "write" "delete" "share")

for action in "${ACTIONS[@]}"; do
  result=$(curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
    -H "Authorization: Bearer $VENGTOO_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{
      \"subject\": {\"id\": \"$SUBJECT_ID\", \"type\": \"user\"},
      \"resource\": {\"type\": \"$RESOURCE_TYPE\", \"id\": \"$RESOURCE_ID\"},
      \"action\": {\"name\": \"$action\"}
    }" | jq -r '.decision')
  echo "$action: $result"
done
```

---

## Health endpoints (local agent)

```bash
# Liveness
curl -s http://localhost:8181/healthz | jq .

# Readiness (bundle loaded)
curl -s http://localhost:8181/readyz | jq .

# AuthZEN discovery
curl -s http://localhost:8181/.well-known/authzen-configuration | jq .

# Prometheus metrics
curl -s http://localhost:8181/metrics | grep vengtoo_agent
```

---

## Useful jq filters

```bash
# Just the decision
| jq '.decision'

# Decision + reason
| jq '{decision: .decision, reason: .context.reason}'

# Full context
| jq '{decision: .decision, reason: .context.reason, code: .context.reason_code, path: .context.access_path}'

# Only show denied items in batch
| jq '.results[] | select(.decision == false) | {resource: .resource.id, action: .action.name, reason: .context.reason}'
```

---

## Testing specific edge cases

### Should be denied (confirm DENY works)
```bash
# Use a subject that you know has no policies
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "nonexistent-user", "type": "user"},
    "resource": {"type": "document", "id": "doc-456"},
    "action": {"name": "delete"}
  }' | jq '{decision: .decision, code: .context.reason_code}'
# Expected: {"decision": false, "code": "SUBJECT_NOT_FOUND"}
```

### Boundary test for time condition
```bash
# Inside window
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "user-123", "type": "user"},
    "resource": {"type": "document", "id": "doc-456"},
    "action": {"name": "read"},
    "context": {"current_time": "2026-06-25T14:00:00Z"}
  }' | jq '.decision'
# Expected: true

# Outside window
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "user-123", "type": "user"},
    "resource": {"type": "document", "id": "doc-456"},
    "action": {"name": "read"},
    "context": {"current_time": "2026-06-25T22:00:00Z"}
  }' | jq '.decision'
# Expected: false
```
