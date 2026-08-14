# Migrating from AWS Cedar / Verified Permissions

Cedar policies are `permit`/`forbid` statements over `(principal, action, resource)`
with optional `when`/`unless` conditions, plus a **schema** describing entity types and
their attributes. AWS Verified Permissions (AVP) stores these in a **policy store**.

## Discover

- **Policies:** `.cedar` files, or fetch from AVP via `ListPolicies` / `GetPolicy` on the
  `verifiedpermissions` API for the policy store.
- **Schema:** the Cedar schema (entity types, actions, attribute shapes) — this maps
  almost directly to Vengtoo resource types + attributes.
- **Entities:** the entity data (principals, resources, and their `in` group memberships
  and attributes) the PEP passes at `IsAuthorized` time.

## Map

| Cedar construct | Vengtoo equivalent |
|---|---|
| `permit(principal, action, resource)` | **`ALLOW`** policy |
| `forbid(principal, action, resource)` | **`DENY`** policy at higher priority |
| `principal == User::"alice"` | **subject** (user) |
| `principal in Role::"admin"` / group | **role** (or **group**) + assignment |
| `action == Action::"read"` | **action** |
| `resource == Document::"123"` | **resource** instance (instance-level) |
| a `resource` entity **type** | **resource type** (+ actions) |
| entity attributes (schema) | subject / resource **attributes** |
| `when { resource.owner == principal }` | **condition** `resource.owner == subject.id` (→ `/vengtoo-abac`) |
| `when { context.mfa == true }` | **condition** on context (mfa_required) |
| `unless { ... }` | invert into a `DENY` policy or a negated condition |
| the **policy store** | a Vengtoo **environment** (do the migration in a non-prod env first) |

## Doesn't translate automatically — flag for human review

- **Deep `in` hierarchies** (nested groups/OUs) → map to Vengtoo groups / role hierarchy;
  a multi-level Cedar entity tree may need restructuring.
- Cedar **record / set operations** (`has`, `.contains`, complex `&&`/`||` nests) →
  express as multiple conditions, or split into multiple policies.
- **Template-linked policies** (AVP policy templates) → expand to concrete policies, or
  model the parameter as an attribute condition.

## Note

Cedar's default is deny; a request is allowed only if a `permit` matches and no `forbid`
does. Vengtoo's DENY-overrides-ALLOW with priority reproduces this: translate `forbid`
to higher-priority `DENY`, and keep the base default closed (no matching policy = deny).

Once mapped, hand model creation to `/vengtoo-policies`, then validate with the
`references/shadow.md` method before promoting.
