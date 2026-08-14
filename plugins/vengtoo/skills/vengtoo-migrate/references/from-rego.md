# Migrating from OPA / Rego

Rego splits authorization into **logic** (the `.rego` rules) and **data** (the JSON
documents OPA loads — roles, group memberships, permission tables). In most OPA
deployments the *grants* live in the data, and the Rego is a thin matcher. Discover
both.

## Discover

- **Rules:** all `.rego` files (often a `policy/` or `bundle/` dir). Look for the entry
  decision — usually `default allow = false` plus one or more `allow { ... }` rules,
  sometimes `deny { ... }`.
- **Data:** the JSON/YAML documents loaded as `data.*` (role bindings, user→role maps,
  resource ownership). This is where the actual assignments are.
- **Input shape:** the `input` object the PEP sends — typically `input.subject`/`input.user`,
  `input.action`, `input.resource`. This tells you the request contract to reproduce.

## Map

| Rego construct | Vengtoo equivalent |
|---|---|
| `input.user` / `input.subject` | **subject** (`external_id` = the same user id) |
| `input.action` (e.g. `"read"`) | **action** on a resource type |
| `input.resource.type` / a resource kind | **resource type** (+ its action list) |
| `input.resource.id` | **resource** instance (only for instance-level rules) |
| `allow { input.role == "admin" }` | **role** `admin` + an `ALLOW` policy on the role |
| `data.role_bindings[input.user][_] == "editor"` | **role assignments** (subject→role) |
| `allow { input.action == "read"; input.resource.owner == input.user.id }` | `ALLOW` policy + **condition** `resource.owner == subject.id` (→ `/vengtoo-abac`) |
| `deny { ... }` / explicit deny rule | **`DENY` policy** at higher priority (DENY beats ALLOW) |
| `data.permissions[role]` (role→permission table) | policies per role, assigned to that role |
| time / IP / tenant guards in the rule body | Vengtoo **conditions** (time_window, ip_allowlist, tenant match) |

## Doesn't translate automatically — flag for human review

- Custom Rego **functions**, comprehensions, `some`/`every` loops → re-express the intent.
- `http.send` or any **external fetch inside policy** → in Vengtoo, fetch the value in
  the PEP and pass it as a subject/resource attribute in the request; policy conditions
  read attributes, they don't call out.
- Partial-set / multi-value rules used for **data filtering** (list endpoints) → this is
  Vengtoo's Search / partial-evaluation surface, not a boolean policy; note it separately.

## Note

Vengtoo *exports* its own policies **to** OPA bundles internally (`/policies/bundle/opa`)
— that's the reverse direction and is unrelated to migrating. You are translating an
external Rego ruleset *into* the Vengtoo model, not importing a Vengtoo bundle.

Once mapped, hand model creation to `/vengtoo-policies`, then validate with the
`references/shadow.md` method before promoting.
