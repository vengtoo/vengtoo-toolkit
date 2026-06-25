---
name: vengtoo-terraform
description: |
  Manages the Vengtoo authorization model as Terraform — resource types, resources,
  subjects, roles, policies, and assignments as code. Handles import of existing
  resources and generates HCL from a live Vengtoo namespace.

  Triggers: "vengtoo terraform", "infra as code authorization", "manage policies as code",
  "terraform vengtoo", "vengtoo provider", "import vengtoo to terraform",
  "authorization as code", "policy as code terraform", "vengtoo hcl",
  "terraform resource type", "terraform policy", "terraform role",
  "generate terraform from vengtoo", "sync vengtoo to terraform"
metadata:
  category: [infrastructure, terraform]
---

# Manage Vengtoo with Terraform

The `vengtoo` Terraform provider (`registry.terraform.io/providers/vengtoo/vengtoo`)
lets you version-control your entire authorization model as HCL.

---

## Reference routing

| User need | Reference file |
|---|---|
| All resource types and arguments | `references/resources.md` |
| Importing existing resources | `references/importing.md` |

---

## Flow

### Step 1: Pre-flight checks

Silently check:
- Is `terraform` available? `terraform version`
- Is there an existing `main.tf` or `vengtoo.tf` in the project?
- Is `VENGTOO_API_KEY` set?

### Step 2: Provider configuration

```hcl
# vengtoo.tf
terraform {
  required_providers {
    vengtoo = {
      source  = "vengtoo/vengtoo"
      version = "~> 1.2"
    }
  }
}

provider "vengtoo" {
  api_key = var.vengtoo_api_key
  # base_url = "http://localhost:8181"  # local agent
}

variable "vengtoo_api_key" {
  description = "Vengtoo API key"
  sensitive   = true
}
```

```bash
terraform init
```

### Step 3: Define the authorization model

Build the five layers in dependency order:

```hcl
# 1. Resource type
resource "vengtoo_resource_type" "document" {
  name        = "document"
  description = "Company documents"
  actions     = ["read", "write", "delete", "share"]
}

# 2. Resource (instance-specific; skip for type-wide policies)
resource "vengtoo_resource" "engineering_wiki" {
  name              = "Engineering Wiki"
  resource_type_id  = vengtoo_resource_type.document.id
  external_id       = "wiki-001"
}

# 3. Subject
resource "vengtoo_subject" "alice" {
  name        = "Alice"
  type        = "user"
  external_id = "user-alice-uuid"
}

# 4. Role
resource "vengtoo_role" "editor" {
  name        = "editor"
  description = "Can read and write documents"
}

# 5. Policy
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

# 6. Policy assignment (to role)
resource "vengtoo_policy_assignment" "editors_can_edit_to_role" {
  policy_id   = vengtoo_policy.editors_can_edit.id
  entity_type = "role"
  entity_id   = vengtoo_role.editor.id
}

# 7. Role assignment (to subject)
resource "vengtoo_role_assignment" "alice_is_editor" {
  subject_id = vengtoo_subject.alice.id
  role_id    = vengtoo_role.editor.id
}
```

### Step 4: Plan and apply

```bash
terraform plan -var="vengtoo_api_key=$VENGTOO_API_KEY"
terraform apply -var="vengtoo_api_key=$VENGTOO_API_KEY"
```

### Step 5: Verify

After apply, test the authorization chain:
```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "user-alice-uuid", "type": "user"},
    "resource": {"type": "document", "id": "wiki-001"},
    "action": {"name": "read"}
  }' | jq .decision
```

Expected: `true`

---

## Common patterns

### DENY override (block specific users)
```hcl
resource "vengtoo_policy" "block_contractor_delete" {
  name     = "block-contractor-delete"
  effect   = "DENY"
  priority = 80   # higher than the ALLOW policy

  resource_types = [
    {
      resource_type_id = vengtoo_resource_type.document.id
      actions          = ["delete"]
    }
  ]
}

resource "vengtoo_policy_assignment" "block_contractor_delete" {
  policy_id   = vengtoo_policy.block_contractor_delete.id
  entity_type = "role"
  entity_id   = vengtoo_role.contractor.id
}
```

### Time-boxed access (JIT)
```hcl
resource "vengtoo_policy_assignment" "temp_admin_access" {
  policy_id   = vengtoo_policy.admin_access.id
  entity_type = "subject"
  entity_id   = vengtoo_subject.on_call_engineer.id
  expires_at  = "2026-07-01T00:00:00Z"
}
```

### Multiple namespaces (dev / staging / prod)
```hcl
# Use a Terraform workspace or separate state per environment
# Pass the API key as an env var per workspace:
# TF_VAR_vengtoo_api_key=$VENGTOO_API_KEY_PROD terraform apply
```

---

## Importing existing resources

If you've already built your model in the Vengtoo console, generate Terraform:

```bash
# Generate HCL from live namespace (requires vengtoo CLI)
vengtoo terraform export > vengtoo-imported.tf
terraform import vengtoo_resource_type.document <id-from-console>
```

Load `references/importing.md` for the full import workflow.

---

## Error handling

| Error | Cause | Fix |
|---|---|---|
| `Error: 409 Conflict` on apply | Resource already exists outside Terraform | Import it: `terraform import vengtoo_<type>.<name> <id>` |
| `Error: 404 Not Found` on destroy | Resource was deleted manually in console | Run `terraform refresh` to sync state |
| `priority` conflict warning | Two policies at the same priority | Adjust priority values; DENY should be 70-90, ALLOW 30-60 |
| API key not authorized | Wrong key or insufficient scope | Check key in Vengtoo console; regenerate if needed |

---

## When NOT to use this skill

- **Quick one-off policy** → use `/vengtoo-policies` via MCP (faster for manual edits)
- **Debugging a denied decision** → use `/vengtoo-debug`
- **ABAC conditions** → add them to the HCL policy block; see `references/resources.md`
