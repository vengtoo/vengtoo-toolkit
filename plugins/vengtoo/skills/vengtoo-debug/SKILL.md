---
name: vengtoo-debug
description: |
  Diagnoses why a Vengtoo authorization check is returning an unexpected result —
  denied when it should be allowed, or allowed when it should be denied.
  Runs a structured investigation and fixes the root cause.

  Triggers: "why is access denied", "authorization not working", "policy not matching",
  "always getting 403", "decision is false", "no policy match", "condition failed",
  "wrong access path", "permission denied", "vengtoo returning false",
  "check is failing", "debug authorization", "why can't user access",
  "deny override", "investigate access", "troubleshoot vengtoo"
metadata:
  category: [debug, policy]
---

# Debug Authorization Failures

Structured five-step investigation for unexpected Vengtoo decisions.
Follow in order — most issues are caught in steps 1-3.

---

## Reference routing

| User need | Reference file |
|---|---|
| Error codes (all reason_codes and fixes) | `references/error-codes.md` |
| Curl test patterns | `references/curl-testing.md` |

---

## Step 1: Reproduce with a raw curl call

Bypass the SDK. This isolates whether the issue is in your code or in the policy.

```bash
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "<subject_id>", "type": "<subject_type>"},
    "resource": {"type": "<resource_type>", "id": "<resource_id>"},
    "action": {"name": "<action>"}
  }' | jq .
```

**If this returns the expected result**: the bug is in your SDK integration, not the policy. Skip to Step 5.
**If this also fails**: continue to Step 2.

---

## Step 2: Read the `reason_code`

The response includes a `context` block:
```json
{
  "decision": false,
  "context": {
    "reason": "No matching policy found",
    "reason_code": "NO_POLICY_MATCH",
    "access_path": null
  }
}
```

Load `references/error-codes.md` for the fix for each `reason_code`.

**Common codes and their implications:**

| `reason_code` | Implication |
|---|---|
| `NO_POLICY_MATCH` | No policy covers this subject+resource+action combination |
| `CONDITION_FAILED` | Policy matched but an ABAC condition blocked it |
| `DENY_OVERRIDE` | A DENY policy at higher priority overruled an ALLOW |
| `ASSIGNMENT_MISSING` | Policy exists but isn't linked to the subject (direct or via role) |
| `ACTION_NOT_PERMITTED` | Action name not in the resource type's allowed actions list |
| `SUBJECT_NOT_FOUND` | Subject ID doesn't match any registered subject |
| `RESOURCE_NOT_FOUND` | Resource ID doesn't match any registered resource |
| `RESOURCE_TYPE_NOT_FOUND` | Resource type name doesn't exist |

---

## Step 3: Trace the full model chain

Use the MCP tools to walk the chain manually:

```
1. Does the subject exist?          → list_subjects, search by ID
2. Does the resource type exist?    → list_resource_types
3. Does the resource exist?         → list_resources (if instance-specific)
4. Is there a policy for this?      → list_policies, check actions + resource targeting
5. Is the policy assigned?          → list_policy_assignments for this subject/role
6. If via role: is the role assigned to the subject?  → list_role_assignments
7. Is there a DENY policy?          → list_policies with effect=DENY
```

For `NO_POLICY_MATCH`: usually steps 4, 5, or 6 reveal the gap.
For `DENY_OVERRIDE`: step 7 reveals the conflicting DENY.

---

## Step 4: Check ABAC conditions (if `CONDITION_FAILED`)

```bash
# Test with explicit context values
curl -s -X POST https://api.vengtoo.com/access/v1/evaluation \
  -H "Authorization: Bearer $VENGTOO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": {"id": "<subject_id>", "type": "user"},
    "resource": {"type": "<resource_type>", "id": "<resource_id>"},
    "action": {"name": "<action>"},
    "context": {
      "ip": "10.0.0.1",
      "current_time": "2026-06-25T10:00:00Z"
    }
  }' | jq .
```

Use the MCP tool `get_policy` to read the conditions on the matching policy.
Check that:
- The attribute exists on the resource/subject (`metadata` field)
- The value type matches (`value_json` numeric vs string)
- Context fields are being passed in the evaluation request

---

## Step 5: Check SDK wiring

If the raw curl works but the SDK call fails, the bug is in how you're calling the SDK.

Checklist:
- Is the subject ID the same as what's in Vengtoo? (external_id vs internal UUID)
- Is the resource type name spelled exactly right (case-sensitive)?
- Is the action name spelled exactly right?
- Is `VENGTOO_API_KEY` pointing to the right namespace/environment?
- Is `VENGTOO_BASE_URL` set correctly? (If using local agent, is it running?)

Enable SDK debug logging:

**Node.js:**
```ts
const vengtoo = new Vengtoo({
  apiKey: process.env.VENGTOO_API_KEY!,
  debug: true,  // logs the full request and response
})
```

**Python:**
```python
import logging
logging.getLogger("vengtoo").setLevel(logging.DEBUG)
```

**Go:**
```go
client := vengtoo.NewClient(apiKey, vengtoo.WithDebug(true))
```

---

## Fix patterns by symptom

| Symptom | Most likely fix |
|---|---|
| `NO_POLICY_MATCH` | Add policy assignment: `create_policy_assignment` linking policy to subject/role |
| `NO_POLICY_MATCH` but policy exists | Check action name matches exactly; check resource type targeting |
| `DENY_OVERRIDE` | Find the DENY policy; either remove it or lower its priority |
| `CONDITION_FAILED` at all times | Check attribute is set on resource/subject; check `value_json` type |
| `CONDITION_FAILED` at runtime only | Client not passing `context` block in SDK call |
| SDK gets 401 | Wrong API key; namespace mismatch |
| SDK gets 200 but wrong decision | Subject ID mismatch (check `external_id` vs internal UUID) |

---

## When NOT to use this skill

- **Building the policy model from scratch** → use `/vengtoo-policies`
- **Adding conditions to a policy** → use `/vengtoo-abac`
- **MCP tool call denied** → start here, then check `/vengtoo-mcp` gateway config
