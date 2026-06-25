# Vengtoo Terraform — Importing Existing Resources

If you've already built your authorization model in the Vengtoo console or via the API,
use these steps to bring it under Terraform management without recreating anything.

---

## Overview

Terraform import maps an existing Vengtoo resource (by ID) to a Terraform resource block.
After import, Terraform tracks the resource; future `apply` calls manage it.

---

## Step 1: Find resource IDs

Use the Vengtoo MCP tools or the dashboard to get IDs:

```bash
# Via MCP — list all resource types and their IDs
List all resource types in Vengtoo

# Via dashboard
Settings → Resource Types → copy the ID from the URL or ID column
```

Or via curl:
```bash
# Resource types
curl -s https://api.vengtoo.com/resource-srv/v1/resource-types \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq '.[] | {name, id}'

# Resources
curl -s https://api.vengtoo.com/resource-srv/v1/resources \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq '.[] | {name, id}'

# Subjects
curl -s https://api.vengtoo.com/entity-srv/v1/subjects \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq '.[] | {name, id}'

# Roles
curl -s https://api.vengtoo.com/entity-srv/v1/roles \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq '.[] | {name, id}'

# Policies
curl -s https://api.vengtoo.com/policy-srv/v1/policies \
  -H "Authorization: Bearer $VENGTOO_API_KEY" | jq '.[] | {name, id}'
```

---

## Step 2: Write the resource block

Write a Terraform resource block for each resource you want to import.
The arguments must match the existing resource exactly — `terraform plan` will show a diff if they don't.

```hcl
# vengtoo.tf

resource "vengtoo_resource_type" "document" {
  name    = "document"
  actions = ["read", "write", "delete"]
}

resource "vengtoo_subject" "alice" {
  name        = "Alice"
  type        = "user"
  external_id = "user-alice-uuid"
}

resource "vengtoo_role" "editor" {
  name = "editor"
}
```

---

## Step 3: Import

```bash
# Format: terraform import <resource_type>.<name> <vengtoo_id>

terraform import vengtoo_resource_type.document <id-from-step-1>
terraform import vengtoo_subject.alice <subject-id>
terraform import vengtoo_role.editor <role-id>
terraform import vengtoo_policy.editors_can_edit <policy-id>
terraform import vengtoo_policy_assignment.editors_can_edit_to_role <assignment-id>
terraform import vengtoo_role_assignment.alice_is_editor <role-assignment-id>
```

---

## Step 4: Validate

```bash
terraform plan
```

If the plan shows no changes, the import is clean. If it shows changes, adjust your HCL to match the existing resource.

Common differences after import:
- `description` field: add it to HCL if it's set in Vengtoo
- `metadata` map: add any metadata attributes set on the resource
- `conditions` list: add conditions that exist on the policy

---

## Bulk import with `generate-config-out` (Terraform 1.5+)

Terraform can generate HCL from imported resources automatically:

```bash
# Add import blocks to a temporary file
cat > import.tf << 'EOF'
import {
  to = vengtoo_resource_type.document
  id = "<resource-type-id>"
}
import {
  to = vengtoo_subject.alice
  id = "<subject-id>"
}
EOF

# Generate HCL
terraform plan -generate-config-out=generated.tf

# Review generated.tf, merge into your main config, then apply
terraform apply
rm import.tf
```

---

## Export all (vengtoo CLI)

If the vengtoo CLI is available:
```bash
# Export all resources from your namespace as Terraform HCL
vengtoo terraform export > vengtoo-imported.tf
```

This generates a complete `.tf` file with all resources and import blocks.
Review and clean up before checking into version control.

---

## Common import errors

| Error | Cause | Fix |
|---|---|---|
| `Resource already managed by Terraform` | Already imported; duplicate import | Remove the `import` command; run `terraform state list` to confirm |
| `Error: Cannot import non-existent remote object` | Wrong ID or resource deleted | Re-fetch the ID; check if resource still exists in Vengtoo |
| `Plan shows unexpected changes after import` | HCL doesn't match live state | Adjust HCL arguments to match; run `terraform plan` until clean |
| `id format must be <uuid>` | Passing name instead of ID | Use the UUID from the API, not the display name |

---

## State management

```bash
# List all resources in state
terraform state list

# Show a specific resource's state
terraform state show vengtoo_resource_type.document

# Remove from state (stops Terraform managing it, doesn't delete from Vengtoo)
terraform state rm vengtoo_role_assignment.alice_is_editor

# Move resource in state (e.g., after renaming the block)
terraform state mv vengtoo_role.editor vengtoo_role.document_editor
```
