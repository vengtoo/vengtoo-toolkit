# Migrating from Casbin

Casbin splits authorization into a **model** (`model.conf` — request shape, policy shape,
role-inheritance rules, and the matcher expression) and **policy data** (`policy.csv`, or a
DB-backed adapter — the actual `p` grant rows and `g` role-binding rows). Discover both;
the model tells you the request contract and matching logic, the policy data is where the
actual grants live.

## Discover

- **Model:** `model.conf` (or wherever `casbin.NewEnforcer(modelPath, ...)` points). Read
  `[request_definition]` for the request shape (commonly `r = sub, obj, act`, or
  `r = sub, dom, obj, act` for RBAC-with-domains), `[role_definition]` for how `g` rows are
  interpreted (per-domain or flat), and `[matchers]` — the matcher is an arbitrary boolean
  expression, not just a lookup, so read it carefully for anything beyond a plain role check.
- **Policy data:** the CSV (or adapter-backed rows). `p` rows are grants (optionally with an
  explicit `eft` column for `allow`/`deny`); `g` rows are role bindings — `g, subject, role`
  or `g, subject, role, domain` — **and** `g, roleA, roleB[, domain]` rows are role
  **inheritance** (roleA includes everything roleB grants), not subject bindings. Don't
  confuse the two shapes.
- **Objects:** Casbin objects are frequently path-like strings (`/projects/:id`) matched via
  `keyMatch`/`keyMatch2`/`regexMatch` in the matcher, not typed resource identifiers. Note
  what each distinct object *shape* actually represents (a resource kind, a resource kind +
  state, a wildcard scope) before mapping it — see below.

## Map

| Casbin construct | Vengtoo equivalent |
|---|---|
| `p, role, obj, act, allow` (or no `eft` column, default allow) | **`ALLOW`** policy on the role |
| `p, role, obj, act, deny` (explicit `eft=deny` row) | **`DENY`** policy at higher priority |
| `g, subject, role[, domain]` | **role assignment** (subject → role) |
| `g, roleA, roleB[, domain]` (role inheritance) | **no direct equivalent** — Vengtoo roles are flat. Build `roleA`'s Vengtoo role as the full flattened union of its own grants **plus everything `roleB` (and `roleB`'s ancestors) grant** — see "Doesn't translate automatically" below |
| `dom` (domain/tenant) | one Vengtoo **role per domain** (e.g. `admin_acme`, `admin_globex`) + a literal `resource_attrs` condition on a tenant attribute (e.g. `org == "acme"`) — not a dynamic comparison, this works cleanly |
| a path-like object with a state segment (e.g. `/projects/archived/:id`) | fold the state into a **resource attribute** on the base resource type (e.g. `archived: true` on `project`), not a separate resource type or a preserved path string — see the positive note below |
| a plain path object (e.g. `/issues/:id`) | a Vengtoo **resource type** (`issue`) + its actions |
| `r.sub == r.obj.Owner` (or any matcher expression comparing a request field to another request field) | **this is Casbin's native relationship/ownership support** — Vengtoo has no equivalent condition type. Falls under "doesn't translate automatically" below: needs an **instance-level direct grant** per resource, not a reusable condition |
| a wildcard action in the matcher (e.g. `act == ".*"` or an unconstrained `act`) | Vengtoo policies target an explicit action list — **enumerate** the resource type's full action set and grant all of them explicitly; there's no wildcard-action policy |

## Doesn't translate automatically — flag for human review

- **Role inheritance (`g, roleA, roleB`)** — must be flattened by hand at translation time.
  Re-derive each tier's full cumulative action list; there's nothing enforcing the superset
  relationship afterward, so a later change to one tier can silently drift out of sync with
  the tiers built on top of it. Flag this to whoever owns the model going forward.
- **Matcher expressions comparing two request fields to each other** (`r.sub == r.obj.Owner`,
  `r.sub.dept == r.obj.dept`) — Vengtoo's ABAC conditions compare an attribute to a fixed
  value, not to another entity's attribute. Needs a per-resource instance-level grant
  (resolved at the point the relationship is established — e.g. when a resource is created
  or reassigned — not a static policy).
- **Custom matcher functions** beyond the built-ins (`keyMatch2`, `regexMatch`, `ipMatch`,
  etc.) — re-express the intent by hand; there's no generic way to port a custom Go/Python
  function embedded in a matcher.

## A genuine positive: path-pattern objects get *better*, not just ported

Casbin's path-pattern matching (`keyMatch2`, `:param` wildcards) is a real source of subtle
bugs — a deny rule scoped to `/projects/archived/:id` (a single path segment) simply does not
match a deeper path like `/projects/archived/:id/subtasks`, since `:id` never matches across
a `/`. Moving the "archived" concept from a URL path segment into a **resource attribute** on
a typed resource doesn't just port this rule — it eliminates the entire bug class, since
Vengtoo resources are identified by type + attributes, not matched against a URL pattern.
There's no equivalent "deeper path" for a bypass to hide in.

## Note

Casbin models range from pure RBAC (no relationship/ownership matcher terms at all — these
migrate cleanly with no workaround needed) to heavily ABAC-flavored (matcher expressions
comparing subject and resource fields directly — these hit the instance-grant limitation
above). Read the matcher first; it tells you which kind of migration this actually is before
you start mapping rows.

Once mapped, hand model creation to `/vengtoo-policies`, then validate with the
`references/shadow.md` method before promoting.
