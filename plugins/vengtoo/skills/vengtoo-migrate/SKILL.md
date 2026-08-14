---
name: vengtoo-migrate
description: |
  Migrates an existing authorization system onto Vengtoo without a risky big-bang:
  discover the incumbent's policies, translate them into the Vengtoo model in a
  non-prod environment, shadow-test for decision parity, then promote and cut over.
  Supports OPA/Rego, AWS Cedar / Verified Permissions, Casbin, and homegrown RBAC.

  Triggers: "migrate to vengtoo", "move off OPA", "migrate from rego", "replace OPA",
  "migrate from cedar", "off AWS Verified Permissions", "convert my rego policies",
  "translate opa to vengtoo", "import existing policies", "we already use OPA",
  "switch from casbin", "port our authorization", "migrate authorization",
  "get off open policy agent", "replace our authz"
metadata:
  category: [migration, onboarding]
---

# Migrate to Vengtoo

Move an existing authorization system onto Vengtoo without a big-bang. Four stages —
and you never enforce with Vengtoo until the translation is *proven* to match the
incumbent decision-for-decision:

```
Discover → Translate (into a lower env) → Shadow (prove parity) → Cutover (promote → flip)
```

The safety comes from Vengtoo **environments**: do the translation and validation in
a non-prod environment, then **promote** the proven model to production with drift
detection and provenance. Prod is only touched once the decisions already agree.

---

## Reference routing

| Source system / need | Reference |
|---|---|
| OPA / Rego | `references/from-rego.md` |
| AWS Cedar / Verified Permissions | `references/from-cedar.md` |
| Casbin | `references/from-casbin.md` |
| Homegrown RBAC | `references/from-homegrown.md` |
| The shadow-compare method + a runnable script | `references/shadow.md` |
| Actually creating the target model (any source) | run `/vengtoo-policies` |
| Managing the migrated model as code | run `/vengtoo-terraform` |
| Porting attribute/condition logic | run `/vengtoo-abac` |

---

## The four stages

| Stage | Goal | Touches prod? |
|---|---|---|
| 1. Discover | Inventory the incumbent + capture a coverage set of real decisions | No |
| 2. Translate | Build the equivalent Vengtoo model **in a non-prod env** | No |
| 3. Shadow | Replay the coverage set through Vengtoo; prove parity | No |
| 4. Cutover | Promote to prod, then flip enforcement one slice at a time | Yes (last) |

---

## Stage 1 — Discover

Find and inventory the incumbent authorization. **Ask the user where their policies
live** if it isn't obvious, then locate them (see the per-source references for exact
file/API locations):

- **OPA/Rego** → `.rego` files **and** the `data` JSON documents (the data is where
  role/permission assignments usually live — the Rego is just the logic).
- **Cedar / AVP** → `.cedar` policy files + schema, or the Verified Permissions
  policy store via its API.
- **Casbin** → the `model.conf` + policy CSV/DB.
- **Homegrown RBAC** → the roles/permissions tables or config.

Produce two outputs:
1. **An inventory** — the resource types, actions, subjects/roles, and conditions the
   incumbent expresses (the raw material Stage 2 maps).
2. **A coverage set** — a list of real `(subject, resource, action, context) → decision`
   examples to replay in Stage 3. Best sources, in order: the incumbent's own test
   cases, its decision logs, then a generated grid across the inventory.

---

## Stage 2 — Translate (into a non-prod environment)

**First, pick or create a non-production environment** — whatever you've named it
(dev, staging, sandbox, a throwaway; the name is yours) — and use an API key scoped to
it. The only fixed name is **production**, which Vengtoo protects. Never translate
straight into production; the whole point is to validate first.

Then map incumbent constructs onto the Vengtoo model using the per-source mapping
tables in the references. Hand the actual model creation to **`/vengtoo-policies`** —
this skill supplies the *mapping*; that skill builds resource types, roles, policies,
assignments, and conditions via the MCP tools.

Rules of thumb:
- One incumbent **resource kind** → one Vengtoo **resource type** (+ its actions).
- **Roles / role-assignment data** → Vengtoo roles + assignments.
- **allow/permit rules** → `ALLOW` policies; **deny/forbid rules** → `DENY` policies at
  higher priority.
- **Attribute conditions** (owner-equals, time, tenant match) → Vengtoo conditions →
  hand to `/vengtoo-abac`.
- **Flag anything that doesn't map cleanly** (see "What doesn't translate" below) for
  explicit human review — do not silently guess.

---

## Stage 3 — Shadow (prove parity)

Replay the Stage 1 coverage set through Vengtoo's standard evaluation endpoint **in the
lower env**, and diff Vengtoo's decision against the incumbent's. See
`references/shadow.md` for the method and a runnable script. Categorize every result:

- **Match** — good.
- **False allow** (Vengtoo allows, incumbent denied) — **the dangerous class**; fix the
  translation before proceeding.
- **False deny** (Vengtoo denies, incumbent allowed) — will break legitimate access; fix.

Iterate on Stage 2 until you reach parity (or every divergence is explained and
intended). This is a *lightweight* shadow — it validates the translation. **Continuous
production shadow** (dual-running Vengtoo alongside the incumbent at the live PEP for
days) is a heavier, separate step; do it only if the coverage set can't be trusted to
be representative.

---

## Stage 4 — Cutover (promote, then hand off the flip)

Cutover builds nothing new — the "flip" is just pointing your enforcement point at
Vengtoo instead of the incumbent, which the integration skills already do.

1. **Promote** the validated model from the lower env to production. Vengtoo's promotion
   carries drift detection, provenance, and a verification pass — so you ship the *exact*
   model you proved, not a hand-rebuild. (Alternative: reproduce it in prod via
   `/vengtoo-terraform` if you manage the model as code.)
2. **Wire the PEP to Vengtoo — hand off, don't rebuild:**
   - In-app SDK enforcement → run **`/vengtoo`**.
   - Gateway / MCP enforcement (Kong, ContextForge, Envoy, MCP gateway) → run **`/vengtoo-mcp`**.

   The flip itself is either a **config change** (gateway repointed at Vengtoo) or swapping
   the incumbent's check call for `vengtoo.authorize(...)` (in-app) — no migration-specific
   code.
3. **Roll out safely:** keep the incumbent running as a fallback, flip one route/service/
   tenant at a time, and watch the decision log for divergence as you expand.
4. **Rollback = point back at the incumbent.** It keeps running until you deliberately
   decommission it, so cutover is always reversible.

---

## What doesn't translate cleanly

Set expectations early — these need human modeling, not automatic translation:

- **Arbitrary imperative logic** in Rego (custom functions, comprehensions, loops).
- **External data lookups at decision time** (`http.send`, DB calls inside policy) —
  in Vengtoo these become subject/resource attributes fed into the request, not
  in-policy fetches.
- **Deeply nested entity hierarchies** (Cedar `in` chains) → map to Vengtoo groups /
  role hierarchy, which may need restructuring.
- Anything whose *intent* is unclear from the source — flag it and ask.

Translate the **intent**, not the syntax. When in doubt, surface it rather than guess.

---

## When NOT to use this skill

- **Fresh project, no incumbent system** → use `/vengtoo` (greenfield setup).
- **Building a model from scratch** (not porting one) → use `/vengtoo-policies`.
- **Managing the model as Terraform** → use `/vengtoo-terraform`.
- **Debugging a specific denied request** after migration → use `/vengtoo-debug`.
