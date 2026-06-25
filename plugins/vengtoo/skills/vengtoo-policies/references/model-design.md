# Vengtoo Authorization Model — Design Reference

Concepts and design patterns for building a well-structured authorization model.

---

## The five layers

```
Resource Type  →  Resource  →  Subject  →  Role  →  Policy  →  Assignment
```

Each layer builds on the previous. You don't need all layers for every model.

---

## Resource Type

**What it is**: A blueprint for a category of resources. Defines what actions are possible.

**Example**: `document` with actions `["read", "write", "delete", "share"]`

**Rules:**
- One resource type per logical category (documents, invoices, environments, tools)
- Actions in the resource type are the full universe — you can only grant what's listed here
- Adding an action to the resource type doesn't grant anyone access; that's the policy's job

**Design questions:**
- What categories of things are you protecting?
- What operations make sense on each category?
- Should `view` and `read` be the same action? (Keep it consistent — pick one convention)

---

## Resource

**What it is**: A specific instance of a resource type.

**Example**: `engineering-wiki` of type `document`

**When to create one:**
- You need per-instance policies (Alice can read *this* document, not all documents)
- You reference the resource by your own ID system → set `external_id`

**When NOT to create one:**
- Your policy applies to all resources of a type (just use resource type targeting in the policy)
- You don't need instance-level granularity yet

---

## Subject

**What it is**: Any actor — user, service, AI agent, device.

**Design questions:**
- Will you use your own user IDs? → Set `external_id` to your UUID/email/user handle
- Do you have AI agents? → Create subjects with `type: "agent"`
- Service-to-service? → Create subjects with `type: "service"`

**External ID usage**: Pass your own ID in evaluation requests; Vengtoo resolves to its internal ID.
This decouples your user management from Vengtoo's internals.

---

## Role

**What it is**: A named collection of policy assignments.

**When to use:**
- You have groups of users with the same permissions (editor, viewer, admin, analyst)
- You want to reassign permissions to many users at once by changing the role

**When NOT to use:**
- One-off direct grants (individual user gets access to one resource → direct assignment)
- Very dynamic permissions that change per-request (use ABAC conditions instead)

**Role hierarchy**: Vengtoo supports flat roles only. If you need inheritance (admin inherits editor), create two policies and assign both to the admin role.

---

## Policy

**What it is**: An effect (ALLOW or DENY) on specific resources/types and actions.

### Effect

- **ALLOW**: grants access when the policy matches
- **DENY**: blocks access when the policy matches. Higher-priority DENY beats ALLOW.

Use DENY sparingly — it's for override scenarios, not default behavior. Default is deny-all.

### Priority

Range: 0–100. Higher = evaluated first.

| Priority range | Use for |
|---|---|
| 30–60 | Standard ALLOW policies |
| 70–90 | DENY overrides |

Two policies at the same priority: if both ALLOW → allowed. If one ALLOW, one DENY → DENY wins.

### Resource targeting

| Approach | When |
|---|---|
| `resource_types` targeting | Policy applies to all resources of this type (current + future) |
| `resources` targeting | Policy applies to specific resource instances only |
| Both combined | Policy applies to specific instances AND all of a type |

**Recommendation**: Start with type targeting. Move to instance targeting only when you need per-resource granularity.

---

## Assignment

**What it is**: The link between a policy and an entity (role or subject).

Without an assignment, a policy has no effect — it exists but nothing can trigger it.

| Assignment type | Use when |
|---|---|
| Policy → Role | Most users — role-based access |
| Policy → Subject (direct) | One-off grants, JIT access, service accounts |

For role-based access, you also need a **role assignment** (subject → role).

---

## Common model patterns

### Pattern 1: Simple RBAC
```
Resource Type: document (read, write, delete)
Roles: viewer, editor, admin
Policies:
  - viewer-read: ALLOW read on document type → assigned to viewer role
  - editor-write: ALLOW read+write on document type → assigned to editor role
  - admin-all: ALLOW read+write+delete on document type → assigned to admin role
```

### Pattern 2: Per-resource (fine-grained)
```
Resource Type: report
Resource: Q1-earnings (external_id: report-q1-2026)
Policy: finance-team-read: ALLOW read on Q1-earnings resource → assigned to finance role
```

### Pattern 3: Default allow, targeted deny
```
Policy: all-users-read: ALLOW read on document type, priority 50 → assigned to authenticated-users role
Policy: archived-deny: DENY read on document type, priority 80, condition: status == archived → assigned to authenticated-users role
```

Result: Everyone can read documents except archived ones.

### Pattern 4: AI agent with delegation
```
Subject: agent:my-bot (type: agent)
Role: ai-assistant
Policy: agent-read-only: ALLOW read on document type → assigned to ai-assistant role
Role assignment: agent:my-bot → ai-assistant

Evaluation: agent:my-bot on-behalf-of user-alice
Result: ALLOW only if both agent policy AND user-alice's policy allow read
```

### Pattern 5: MCP tool authorization
```
Resource Type: mcp_tool (actions: ["invoke"])
Resources: one per tool (database__query, filesystem__read_file, filesystem__write_file)
Subject: agent:claude-code
Policy: safe-tools: ALLOW invoke on database__query and filesystem__read_file → assigned to agent
Policy: deny-write: DENY invoke on filesystem__write_file, priority 80 → assigned to agent
```

---

## What to avoid

| Anti-pattern | Problem | Fix |
|---|---|---|
| One policy per user | Doesn't scale; hard to audit | Use roles |
| DENY as the primary access control | Confusing; easy to accidentally grant | DENY is for overrides only; default is deny-all |
| All permissions in one policy | Hard to update; can't revoke partially | One policy per logical permission group |
| Skipping external_id | Forces use of Vengtoo UUIDs in your app code | Always set external_id on subjects and resources |
| Actions not in resource type | Policy creation succeeds but evaluation always fails | Keep resource type actions complete |
