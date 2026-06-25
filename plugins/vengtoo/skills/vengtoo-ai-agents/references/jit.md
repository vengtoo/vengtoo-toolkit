# Vengtoo — JIT (Just-in-Time) Access Reference

JIT grants temporary, time-boxed access that automatically expires. No cleanup required.

---

## Two JIT mechanisms in Vengtoo

| Mechanism | How | Best for |
|---|---|---|
| `expires_at` on assignment | Assignment is ignored after timestamp | Temporary role/policy grants |
| `valid_until` condition | Policy condition fails after timestamp | Narrowing an existing broad policy |

Most JIT use cases use `expires_at` on the assignment — simpler and more explicit.

---

## Full JIT flow

```
1. Agent or user requests elevated access for a specific task
2. System (or human approver) creates a time-boxed policy assignment
3. Agent calls Vengtoo — gets ALLOW during the window
4. Assignment expires automatically — no revocation needed
5. Next call after expiry returns NO_POLICY_MATCH or ASSIGNMENT_EXPIRED
```

---

## Creating a JIT grant (Python SDK)

```python
from vengtoo import Vengtoo
from datetime import datetime, timedelta, timezone

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def grant_jit_access(
    policy_id: str,
    subject_id: str,
    duration_minutes: int = 60,
):
    """Grant time-boxed access to a subject via policy assignment."""
    expires = datetime.now(timezone.utc) + timedelta(minutes=duration_minutes)

    assignment = client.create_policy_assignment(
        policy_id=policy_id,
        entity_type="subject",
        entity_id=subject_id,
        expires_at=expires.isoformat(),
    )

    return {
        "assignment_id": assignment.id,
        "expires_at": expires.isoformat(),
        "expires_in_minutes": duration_minutes,
    }
```

---

## JIT for AI agents (common pattern)

An agent requests elevated access before performing a sensitive operation:

```python
ELEVATED_POLICY_ID = "policy-prod-deploy-access"
DEPLOY_AGENT_ID = "agent:deploy-bot"

async def deploy_with_jit(environment: str, user_id: str):
    # 1. Pre-check without JIT
    resp = client.authorize({
        "subject": {"id": DEPLOY_AGENT_ID, "type": "agent", "on_behalf_of": {"id": user_id, "type": "user"}},
        "resource": {"type": "environment", "id": environment},
        "action": {"name": "deploy"},
    })

    if resp.decision:
        # Already has access
        return await execute_deploy(environment)

    # 2. Request JIT grant (may require human approval — see hitl.md)
    approved = await request_human_approval(
        approver=get_approver_for(environment),
        reason=f"Agent needs deploy access to {environment}",
        duration_minutes=30,
    )
    if not approved:
        raise PermissionError("Deploy access denied by approver")

    # 3. Create time-boxed assignment
    grant = grant_jit_access(
        policy_id=ELEVATED_POLICY_ID,
        subject_id=DEPLOY_AGENT_ID,
        duration_minutes=30,
    )

    # 4. Execute the task
    result = await execute_deploy(environment)

    # 5. (Optional) Revoke early if done before expiry
    client.delete_policy_assignment(grant["assignment_id"])

    return result
```

---

## Node.js JIT grant

```ts
import { Vengtoo } from '@vengtoo/sdk'

const client = new Vengtoo({ apiKey: process.env.VENGTOO_API_KEY! })

async function grantJit(policyId: string, subjectId: string, durationMs: number) {
  const expiresAt = new Date(Date.now() + durationMs).toISOString()

  return client.createPolicyAssignment({
    policyId,
    entityType: 'subject',
    entityId: subjectId,
    expiresAt,
  })
}

// Usage: grant 30-minute access
const grant = await grantJit('policy-admin-access', 'agent:onboarding-bot', 30 * 60 * 1000)
console.log(`Access expires at: ${grant.expiresAt}`)
```

---

## Terraform JIT (time-boxed assignment)

```hcl
resource "vengtoo_policy_assignment" "incident_response_access" {
  policy_id   = vengtoo_policy.incident_response.id
  entity_type = "subject"
  entity_id   = vengtoo_subject.on_call_engineer.id
  expires_at  = "2026-06-26T06:00:00Z"   # expires at end of on-call shift
}
```

Apply during the on-call shift begins. After `expires_at`, the assignment is automatically ignored — no need to `terraform apply` again to remove it.

---

## Early revocation

If the task completes before the JIT window expires, revoke the assignment to reduce exposure:

```python
# Python
client.delete_policy_assignment(assignment_id)
```

```ts
// Node.js
await client.deletePolicyAssignment(assignmentId)
```

```bash
# Via MCP tool: delete_policy_assignment
# id: <assignment_id>
```

---

## `valid_until` condition (alternative mechanism)

Instead of expiring the assignment, add a condition to the policy:

```json
{
  "type": "valid_until",
  "timestamp": "2026-06-26T00:00:00Z"
}
```

When the condition expires, the policy evaluation returns `CONDITION_FAILED` rather than `NO_POLICY_MATCH`. The assignment remains but the condition blocks it.

**Use `valid_until` when:**
- You want to reuse the policy after extending the time window
- The assignment is shared between multiple subjects with different expiries

**Use `expires_at` on the assignment when:**
- Each subject has their own time window
- You want clean removal (no stale assignments)

---

## Checking remaining JIT time

```python
assignment = client.get_policy_assignment(assignment_id)
expires_at = datetime.fromisoformat(assignment.expires_at)
remaining = expires_at - datetime.now(timezone.utc)
print(f"JIT access expires in {int(remaining.total_seconds() / 60)} minutes")
```

---

## Audit trail

Every evaluation during a JIT window is logged with the assignment ID in the decision response context. Use this to reconstruct what the agent did during elevated access:

```bash
# Filter audit log for a specific JIT assignment
vengtoo audit logs --assignment-id <id> --since 2026-06-25T00:00:00Z
```

Or via the Vengtoo Decision Log in the dashboard.
