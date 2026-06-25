# Vengtoo Terraform Provider — Resource Reference

Provider: `registry.terraform.io/providers/vengtoo/vengtoo` (version `~> 1.2`)

---

## Provider configuration

```hcl
terraform {
  required_providers {
    vengtoo = {
      source  = "vengtoo/vengtoo"
      version = "~> 1.2"
    }
  }
}

provider "vengtoo" {
  api_key  = var.vengtoo_api_key
  # base_url = "https://api.vengtoo.com"  # default
}

variable "vengtoo_api_key" {
  description = "Vengtoo API key"
  sensitive   = true
}
```

---

## `vengtoo_resource_type`

Defines what kinds of resources exist and what actions they support.

```hcl
resource "vengtoo_resource_type" "document" {
  name        = "document"
  description = "Company documents and files"
  actions     = ["read", "write", "delete", "share", "export"]
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Unique within namespace. Case-sensitive. |
| `description` | string | no | Human-readable. |
| `actions` | list(string) | yes | All actions that policies can reference. |

**Outputs**: `id`

---

## `vengtoo_resource`

A specific resource instance.

```hcl
resource "vengtoo_resource" "engineering_wiki" {
  name             = "Engineering Wiki"
  resource_type_id = vengtoo_resource_type.document.id
  external_id      = "wiki-001"
  metadata = {
    department = "engineering"
    status     = "published"
    owner      = "alice"
  }
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Display name. |
| `resource_type_id` | string | yes | ID from `vengtoo_resource_type`. |
| `external_id` | string | no | Your own system ID. Used for external_id resolution in evaluations. |
| `metadata` | map(string) | no | Key-value attributes. Used in ABAC conditions. |

**Outputs**: `id`, `external_id`

---

## `vengtoo_subject`

An actor: user, service, AI agent, device.

```hcl
resource "vengtoo_subject" "alice" {
  name        = "Alice"
  type        = "user"
  external_id = "user-alice-uuid-from-db"
  metadata = {
    department     = "finance"
    clearance_level = "3"
    region         = "us-east"
  }
}

resource "vengtoo_subject" "deploy_agent" {
  name        = "Deploy Agent"
  type        = "agent"
  external_id = "agent:deploy-bot"
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Display name. |
| `type` | string | yes | `user`, `agent`, `service`, `device` |
| `external_id` | string | no | Strongly recommended — lets evaluations use your own IDs. |
| `metadata` | map(string) | no | For ABAC subject attribute conditions. |

**Outputs**: `id`, `external_id`

---

## `vengtoo_role`

Groups policies. Subjects are assigned roles; policies are assigned to roles.

```hcl
resource "vengtoo_role" "editor" {
  name        = "editor"
  description = "Can read and write documents"
}

resource "vengtoo_role" "admin" {
  name        = "admin"
  description = "Full access to all resources"
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Unique within namespace. |
| `description` | string | no | |

**Outputs**: `id`

---

## `vengtoo_policy`

Defines what actions are allowed or denied on which resources.

```hcl
# Type-wide policy (covers all current + future resources of this type)
resource "vengtoo_policy" "editors_can_edit" {
  name     = "editors-can-edit"
  effect   = "ALLOW"
  priority = 50

  resource_types = [
    {
      resource_type_id = vengtoo_resource_type.document.id
      actions          = ["read", "write"]
    }
  ]
}

# Instance-specific policy
resource "vengtoo_policy" "wiki_readers" {
  name     = "wiki-readers"
  effect   = "ALLOW"
  priority = 50

  resources = [
    {
      resource_id = vengtoo_resource.engineering_wiki.id
      actions     = ["read"]
    }
  ]
}

# DENY policy (override)
resource "vengtoo_policy" "block_contractor_delete" {
  name     = "block-contractor-delete"
  effect   = "DENY"
  priority = 80   # higher than ALLOW policies

  resource_types = [
    {
      resource_type_id = vengtoo_resource_type.document.id
      actions          = ["delete"]
    }
  ]
}

# Policy with ABAC conditions
resource "vengtoo_policy" "business_hours_only" {
  name     = "business-hours-read"
  effect   = "ALLOW"
  priority = 50

  resource_types = [
    {
      resource_type_id = vengtoo_resource_type.document.id
      actions          = ["read"]
    }
  ]

  conditions = [
    {
      type     = "time_of_day"
      from     = "09:00"
      to       = "17:00"
      timezone = "America/New_York"
      days     = ["Mon", "Tue", "Wed", "Thu", "Fri"]
    }
  ]
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Unique within namespace. |
| `effect` | string | yes | `ALLOW` or `DENY` |
| `priority` | number | yes | 0–100. Higher = evaluated first. DENY should be 70–90, ALLOW 30–60. |
| `resource_types` | list(object) | no | Type-wide targeting. |
| `resources` | list(object) | no | Instance-specific targeting. |
| `conditions` | list(object) | no | ABAC conditions. All must pass (AND). |

**Note:** `resource_types` and `resources` can both be set. A policy matches if it matches either.

**Outputs**: `id`

---

## `vengtoo_policy_assignment`

Links a policy to a role or subject.

```hcl
# Policy → role
resource "vengtoo_policy_assignment" "editors_can_edit_assignment" {
  policy_id   = vengtoo_policy.editors_can_edit.id
  entity_type = "role"
  entity_id   = vengtoo_role.editor.id
}

# Policy → subject (direct grant)
resource "vengtoo_policy_assignment" "alice_direct_access" {
  policy_id   = vengtoo_policy.editors_can_edit.id
  entity_type = "subject"
  entity_id   = vengtoo_subject.alice.id
}

# Time-boxed (JIT)
resource "vengtoo_policy_assignment" "temp_admin" {
  policy_id   = vengtoo_policy.admin_access.id
  entity_type = "subject"
  entity_id   = vengtoo_subject.on_call.id
  expires_at  = "2026-07-01T00:00:00Z"
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `policy_id` | string | yes | |
| `entity_type` | string | yes | `role` or `subject` |
| `entity_id` | string | yes | |
| `expires_at` | string | no | RFC3339 UTC. Assignment ignored after this timestamp. |

**Outputs**: `id`

---

## `vengtoo_role_assignment`

Links a role to a subject.

```hcl
resource "vengtoo_role_assignment" "alice_is_editor" {
  subject_id = vengtoo_subject.alice.id
  role_id    = vengtoo_role.editor.id
}
```

| Argument | Type | Required | Notes |
|---|---|---|---|
| `subject_id` | string | yes | |
| `role_id` | string | yes | |

**Outputs**: `id`

---

## Priority guidelines

| Priority range | Use for |
|---|---|
| 0–30 | Baseline / default policies |
| 30–60 | Standard ALLOW policies |
| 60–70 | Elevated ALLOW (admin-level) |
| 70–90 | DENY overrides |
| 90–100 | Hard blocks (never override) |

Two policies at the same priority with the same effect both apply. Two policies at the same priority with conflicting effects: DENY wins.
