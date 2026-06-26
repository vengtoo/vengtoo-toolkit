---
name: vengtoo-policies
description: |
  Builds or audits the Vengtoo authorization model for the current project.
  Reads live cloud state via MCP tools, then guides through creating resource types,
  resources, subjects, roles, policies, and assignments.

  Triggers: "create a policy", "set up roles", "add subjects", "define resource types",
  "build authorization model", "allow user to read", "grant access", "create role",
  "why is everyone denied", "no policies exist", "set up permissions",
  "manage policies", "who can access what", "authorization model"
metadata:
  category: [policy, model-design]
---

# Build the Authorization Model

Vengtoo's model has five layers. Build them in order:

```
Resource Type → Resource → Subject → Role → Policy → Assignment
```

This skill reads your existing cloud state first, then fills the gaps.

---

## Reference routing

| User need | Reference file |
|---|---|
| Understanding the model (concepts) | `references/model-design.md` |
| Direct API calls (curl reference) | `references/api-reference.md` |
| Terraform approach | run `/vengtoo-terraform` instead |

---

## Flow

### Step 1: Read current state (using MCP tools)

Call the Vengtoo MCP tools to see what already exists:
- List namespaces
- List resource types
- List subjects
- List roles
- List policies

Report what's found. If nothing exists, this is a fresh setup — proceed to Step 2.
If things exist, identify the gap (e.g., policy exists but not assigned, resource type missing).

### Step 2: Understand what needs protecting

Ask the user:
1. **What is the resource?** (e.g., "documents", "invoices", "API endpoints", "MCP tools")
2. **What actions are possible?** (e.g., read, write, delete, approve, invoke)
3. **Who needs access?** (e.g., specific users, a role like "editor", all authenticated users)
4. **Type-level or instance-level?** — always confirm this with the user before continuing.
   - **Type-level**: the policy covers *all* resources of that type — e.g., "editors can read any document". No resource instances needed. Evaluation uses `id: "*"` as a sentinel.
   - **Instance-level**: the policy covers *one specific resource* — e.g., "Alice can read doc-123 only". Requires creating the resource instance.

   Example prompt: *"Should this policy apply to all documents (type-level), or to a specific document (instance-level)?"*

   Do not infer this from context — the choice has structural consequences and the user must confirm.

Use this to design the model — don't create unnecessary layers.

### Step 3: Create resource type

The resource type defines the blueprint — what kinds of resources exist and what actions they support.

```bash
# Via MCP tool: create_resource_type
# name: document
# actions: ["read", "write", "delete", "share"]
```

One resource type per logical category. If you have documents AND invoices, create two resource types.

### Step 4: Create resource (instance-level only — skip for type-level)

**Skip this step entirely if you answered "type-level" in Step 2.** Type-level policies work without any resource instances — evaluation uses `id: "*"` as a sentinel.

Only create a resource instance if the policy targets ONE specific resource:

```bash
# Via MCP tool: create_resource
# name: "Engineering Wiki"
# type: <resource_type_id>
# external_id: "wiki-001"  (your own system ID, optional)
```

### Step 5: Create subject(s)

A subject is any actor: user, service, AI agent, device.

```bash
# Via MCP tool: create_subject
# name: "Alice"
# type: "user"
# external_id: "alice-uuid-from-your-db"  (lets Vengtoo resolve by your ID)
```

`external_id` is strongly recommended — it lets you pass your own user ID in evaluation requests without needing to know Vengtoo's internal UUID.

**For testing:** create subjects manually via the MCP tool.

**In production:** don't pre-create users in Vengtoo. Hook into your signup or onboarding flow and call the API programmatically when a user is created:

```python
# on user registration
vengtoo.create_subject(external_id=user.id, type="user")
vengtoo.assign_role(subject_id=user.id, role="customer")  # if using RBAC
```

The same applies to resources — create them in Vengtoo when they are created in your system, not upfront. For type-level policies (most common case), you don't need to register resource instances at all.

### Step 6: Create role (if using RBAC)

A role groups policies. Skip if doing direct grants only.

```bash
# Via MCP tool: create_role
# name: "editor"
# description: "Can read and write documents"
```

### Step 7: Create policy

```bash
# Via MCP tool: create_policy
# name: "editors-can-edit"
# effect: "ALLOW"
# priority: 50
# resources: [{ resource_id: <id>, actions: ["read", "write"] }]
# OR
# resource_types: [{ resource_type_id: <id>, actions: ["read", "write"] }]
```

**Resource instance vs type targeting:**
- `resources` → policy applies to specific resources only
- `resource_types` → policy applies to ALL resources of that type (current + future)
- Both can be combined

**Effect and priority:**
- `ALLOW` with priority 50 = standard grant
- `DENY` with priority 80 = override (DENY at higher priority beats ALLOW at lower)
- Priority range: 0–100; higher = evaluated first

### Step 8: Assign policy

The full RBAC chain is: **policy → role → subject**

```
assign_policy(policy_id, entity_type: "role", entity_id: <role_id>)
    then
assign_role_to_subject(subject_id: <subject_id>, role_id: <role_id>)
```

For a direct one-off grant (no role), assign the policy straight to the subject:
```
assign_policy(policy_id, entity_type: "entity", entity_id: <subject_id>)
```

> **Important**: `entity_type` must be `"entity"` (not `"subject"`) for direct subject assignment, `"role"` for roles, `"group"` for groups. Using the wrong value causes silent `NO_POLICY_MATCH`.

### Step 9: Verify

**For type-level policies** — no resource instance needed. Use `id: "*"` as a sentinel meaning "any resource of this type":

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": { "external_id": "alice-from-your-db", "type": "user" },
    "resource": { "type": "document", "id": "*" },
    "action": { "name": "read" }
  }'
```

**For instance-level policies** — pass the specific resource:

```bash
# By external_id (recommended — your own system's identifier):
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": { "external_id": "alice-from-your-db", "type": "user" },
    "resource": { "type": "document", "external_id": "doc-123" },
    "action": { "name": "read" }
  }'

# By Vengtoo UUID (only if you have the Vengtoo-assigned UUID):
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": { "external_id": "alice-from-your-db", "type": "user" },
    "resource": { "type": "document", "id": "<vengtoo_resource_uuid>" },
    "action": { "name": "read" }
  }'
```

Expected: `{"decision": true, "context": {"reason": "...", "access_path": "role"}}`

---

## Common model patterns

| Pattern | Layers needed |
|---|---|
| All users can read all documents | Resource Type + Policy (type-targeted) + assign to all subjects |
| Role-based (editor/viewer/admin) | Resource Type + Roles + Policies per role + Role assignments |
| Per-resource permissions | Resource Type + Resource + Policy (instance-targeted) + Subject assignment |
| AI agent tool authorization | Resource Type (`mcp_tool`) + Resources (tool names) + Subject (agent) + Policy |
| Service-to-service | Subject (type: service) + OAuth CC + Policy scoped to service |

---

## Error handling

| Error | Cause | Fix |
|---|---|---|
| Policy created but decision still `false` | Policy not assigned | Add policy assignment |
| `NO_POLICY_MATCH` | No policy covers this subject+resource+action | Check resource type, actions, assignment chain |
| `ACTION_NOT_PERMITTED` | Action not in resource type's action list | Add action to resource type |
| `DENY_OVERRIDE` | A DENY policy at higher priority matched | Check for conflicting DENY policies |
| `CONDITION_FAILED` | ABAC conditions not satisfied | Run `/vengtoo-abac` to debug conditions |

---

## When NOT to use this skill

- **Managing policies as Terraform** → use `/vengtoo-terraform`
- **Adding ABAC conditions to an existing policy** → use `/vengtoo-abac`
- **Debugging a specific denied request** → use `/vengtoo-debug`
