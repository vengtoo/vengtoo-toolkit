# Vengtoo API Reference — Policy Model

Direct curl commands for every operation in the authorization model.
Set `VENGTOO_API_KEY` before running.

---

## Resource Types

```bash
# List
curl -s https://api.vengtoo.com/resource-srv/v1/resource-types \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Get one
curl -s https://api.vengtoo.com/resource-srv/v1/resource-types/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create
curl -s -X POST https://api.vengtoo.com/resource-srv/v1/resource-types \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "document",
    "description": "Company documents",
    "actions": ["read", "write", "delete", "share"]
  }' | jq .

# Update
curl -s -X PUT https://api.vengtoo.com/resource-srv/v1/resource-types/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"actions": ["read", "write", "delete", "share", "export"]}' | jq .

# Delete
curl -s -X DELETE https://api.vengtoo.com/resource-srv/v1/resource-types/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY"
```

---

## Resources

```bash
# List (with optional filter)
curl -s "https://api.vengtoo.com/resource-srv/v1/resources?resource_type_id=<type-id>" \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create
curl -s -X POST https://api.vengtoo.com/resource-srv/v1/resources \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Engineering Wiki",
    "resource_type_id": "<type-id>",
    "external_id": "wiki-001",
    "metadata": {
      "department": "engineering",
      "status": "published"
    }
  }' | jq .

# Update metadata
curl -s -X PUT https://api.vengtoo.com/resource-srv/v1/resources/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"status": "archived"}}' | jq .
```

---

## Subjects

```bash
# List
curl -s https://api.vengtoo.com/entity-srv/v1/subjects \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Search by external_id
curl -s "https://api.vengtoo.com/entity-srv/v1/subjects?external_id=user-alice-uuid" \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create
curl -s -X POST https://api.vengtoo.com/entity-srv/v1/subjects \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice",
    "type": "user",
    "external_id": "user-alice-uuid",
    "metadata": {
      "department": "finance",
      "clearance_level": "3"
    }
  }' | jq .

# Update metadata
curl -s -X PUT https://api.vengtoo.com/entity-srv/v1/subjects/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"department": "engineering"}}' | jq .
```

---

## Roles

```bash
# List
curl -s https://api.vengtoo.com/entity-srv/v1/roles \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create
curl -s -X POST https://api.vengtoo.com/entity-srv/v1/roles \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "editor", "description": "Can read and write documents"}' | jq .
```

---

## Policies

```bash
# List
curl -s https://api.vengtoo.com/policy-srv/v1/policies \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create (type-wide ALLOW)
curl -s -X POST https://api.vengtoo.com/policy-srv/v1/policies \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "editors-can-edit",
    "effect": "ALLOW",
    "priority": 50,
    "resource_types": [
      {
        "resource_type_id": "<type-id>",
        "actions": ["read", "write"]
      }
    ]
  }' | jq .

# Create (instance-specific ALLOW)
curl -s -X POST https://api.vengtoo.com/policy-srv/v1/policies \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "wiki-readers",
    "effect": "ALLOW",
    "priority": 50,
    "resources": [
      {
        "resource_id": "<resource-id>",
        "actions": ["read"]
      }
    ]
  }' | jq .

# Create with ABAC condition
curl -s -X POST https://api.vengtoo.com/policy-srv/v1/policies \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "business-hours-read",
    "effect": "ALLOW",
    "priority": 50,
    "resource_types": [{"resource_type_id": "<type-id>", "actions": ["read"]}],
    "conditions": [
      {
        "type": "time_of_day",
        "from": "09:00",
        "to": "17:00",
        "timezone": "America/New_York",
        "days": ["Mon", "Tue", "Wed", "Thu", "Fri"]
      }
    ]
  }' | jq .
```

---

## Policy Assignments

```bash
# List assignments for a policy
curl -s "https://api.vengtoo.com/policy-srv/v1/policy-assignments?policy_id=<policy-id>" \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create (assign policy to role)
curl -s -X POST https://api.vengtoo.com/policy-srv/v1/policy-assignments \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "policy_id": "<policy-id>",
    "entity_type": "role",
    "entity_id": "<role-id>"
  }' | jq .

# Create (assign policy to subject directly)
curl -s -X POST https://api.vengtoo.com/policy-srv/v1/policy-assignments \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "policy_id": "<policy-id>",
    "entity_type": "entity",
    "entity_id": "<subject-id>",
    "expires_at": "2026-07-01T00:00:00Z"
  }' | jq .

# Delete assignment
curl -s -X DELETE https://api.vengtoo.com/policy-srv/v1/policy-assignments/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY"
```

---

## Role Assignments

```bash
# List roles for a subject
curl -s "https://api.vengtoo.com/entity-srv/v1/role-assignments?subject_id=<subject-id>" \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq .

# Create
curl -s -X POST https://api.vengtoo.com/entity-srv/v1/role-assignments \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject_id": "<subject-id>",
    "role_id": "<role-id>"
  }' | jq .

# Delete
curl -s -X DELETE https://api.vengtoo.com/entity-srv/v1/role-assignments/<id> \
  -H "Authorization: Bearer $VENGTOO_API_KEY"
```

---

## Evaluation

> `id` must be a Vengtoo internal UUID. For your own system identifiers (DB IDs, slugs), use `external_id` instead.

```bash
# Single evaluation — using external_id (recommended)
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"external_id": "user-alice-uuid", "type": "user"},
    "resource": {"type": "document", "external_id": "wiki-001"},
    "action": {"name": "read"}
  }' | jq .

# Single evaluation — using Vengtoo UUIDs
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "<vengtoo-subject-uuid>", "type": "user"},
    "resource": {"type": "document", "id": "<vengtoo-resource-uuid>"},
    "action": {"name": "read"}
  }' | jq .

# Batch evaluation
curl -s -X POST https://api.vengtoo.com/access/v1/evaluations \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"external_id": "user-alice-uuid", "type": "user"},
    "items": [
      {"resource": {"type": "document", "external_id": "wiki-001"}, "action": {"name": "read"}},
      {"resource": {"type": "document", "external_id": "wiki-001"}, "action": {"name": "write"}}
    ]
  }' | jq '.results[] | {action: .action.name, decision: .decision}'
```
