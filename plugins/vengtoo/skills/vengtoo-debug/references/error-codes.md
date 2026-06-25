# Vengtoo Error Codes Reference

All `reason_code` values returned in the evaluation response `context` block, with causes and fixes.

---

## Decision is `false`

### `NO_POLICY_MATCH`
**Meaning**: No policy was found that covers this subject + resource + action.

**Common causes:**
1. Policy exists but isn't assigned to the subject (directly or via role)
2. Policy is assigned to a role, but the role isn't assigned to the subject
3. Action name doesn't match what's in the policy
4. Policy targets a specific resource, but the request uses the resource type name (or vice versa)

**Fix:**
- Check `list_policy_assignments` for the subject — does the policy appear?
- Check `list_role_assignments` for the subject — does the role appear?
- Compare the action name in the policy vs the action name in the request (case-sensitive)
- Check whether the policy uses `resource_types` (type-wide) or `resources` (instance-specific)

---

### `CONDITION_FAILED`
**Meaning**: A policy matched on subject/resource/action, but an ABAC condition blocked it.

**Common causes:**
1. Time condition: request is outside the allowed time window
2. IP condition: request IP not in the allowed ranges
3. Resource attribute: resource metadata doesn't have the expected value
4. Subject attribute: subject doesn't have the expected attribute
5. Context attribute: client didn't pass the required `context` field

**Fix:**
- Read the `failed_condition` block in the response for specifics
- Check that the attribute exists on the resource/subject (not just the condition definition)
- Verify the client is passing `context` fields in the evaluation request (IP, current_time, etc.)
- Check `value_json` type: strings need inner quotes `"\"value\""`, numbers don't `"42"`

---

### `DENY_OVERRIDE`
**Meaning**: An ALLOW policy matched, but a DENY policy at higher priority also matched and won.

**Common causes:**
1. A broad DENY policy is catching requests it shouldn't
2. A DENY policy was added for a specific role but the subject has that role unexpectedly

**Fix:**
- Use `list_policies` with `effect=DENY` to find all DENY policies
- Check which DENY policy matched (the response context includes `matched_policy`)
- Either: remove the DENY policy, add a condition to narrow it, or lower its priority

---

### `ACTION_NOT_PERMITTED`
**Meaning**: The action name exists in the evaluation request but is not in the resource type's `actions` list.

**Fix:**
- Check the resource type's allowed actions via `get_resource_type`
- Add the missing action to the resource type's actions list
- Common typo: `"delete"` vs `"destroy"`, `"edit"` vs `"write"`, `"view"` vs `"read"`

---

### `ASSIGNMENT_MISSING`
**Meaning**: A matching policy was found, but it has no assignment linking it to this subject.

**Fix:**
- Create a `policy_assignment` linking the policy to the subject or role
- If assigning via role: also create a `role_assignment` linking the role to the subject

---

### `SUBJECT_NOT_FOUND`
**Meaning**: No subject in Vengtoo matches the provided `subject.id`.

**Common causes:**
1. Using the Vengtoo internal UUID, but the subject was created with `external_id`
2. Using the `external_id`, but the subject was created without one
3. Typo in the subject ID
4. Subject exists in a different namespace/application

**Fix:**
- Use `list_subjects` to find the subject; check both `id` and `external_id`
- Set `external_id` on the subject if using your own user IDs
- Pass `external_id` value as `subject.id` in the evaluation request — Vengtoo resolves it

---

### `RESOURCE_NOT_FOUND`
**Meaning**: No resource in Vengtoo matches the provided `resource.id`.

**Fix:**
- Use `list_resources` to check what exists
- Verify you're using the correct field: `external_id` vs internal UUID
- For type-wide policies (no specific resource instance), use `resource.type` only without `resource.id`

---

### `RESOURCE_TYPE_NOT_FOUND`
**Meaning**: No resource type matches the `resource.type` string in the request.

**Fix:**
- Check `list_resource_types` for the exact name (case-sensitive)
- Create the resource type if it doesn't exist

---

### `DELEGATION_NOT_PERMITTED`
**Meaning**: `on_behalf_of` was passed, but the agent's role doesn't allow delegation.

**Fix:**
- Add a delegation policy to the agent's subject or role:
  - Effect: ALLOW
  - Action: `delegate`
  - Resource type: the type being accessed

---

### `DELEGATION_DEPTH_EXCEEDED`
**Meaning**: The delegation chain is too deep (orchestrator → agent → sub-agent → sub-sub-agent...).

**Fix:**
- Flatten the agent hierarchy
- Maximum delegation depth is 3 hops by default

---

### `INSUFFICIENT_PRIVILEGE`
**Meaning**: The subject exists, policies exist, but the subject's roles don't include sufficient privilege for this action.

**Use:** Trigger a human-in-the-loop flow when this code appears — see `/vengtoo-ai-agents`.

---

## Decision is `true` (but shouldn't be)

If a decision returns `true` unexpectedly:

1. Check `access_path` in the response — it shows exactly which policy + role chain granted access
2. Look for a broad ALLOW policy that's catching too many subjects or resources
3. Check if `failOpen: true` is set in the agent or SDK config (this forces `true` on errors)

---

## HTTP-level errors

| HTTP status | Meaning | Fix |
|---|---|---|
| `401 Unauthorized` | Missing or invalid API key | Check `VENGTOO_API_KEY`; regenerate in console |
| `403 Forbidden` | API key doesn't have access to this namespace | Check key scope in console |
| `404 Not Found` | Endpoint path wrong | Verify URL; evaluation is `POST /access/v1/evaluation` |
| `422 Unprocessable` | Request body missing required fields | Check `subject`, `resource`, `action` are all present |
| `429 Too Many Requests` | Rate limit exceeded | Add backoff; consider local agent to bypass cloud rate limits |
| `503 Service Unavailable` | Cloud temporarily unavailable | Retry; if using agent, check agent `/healthz` |
