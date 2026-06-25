# Vengtoo — Human-in-the-Loop Reference

Patterns for pausing agent execution and requesting human approval before proceeding.

---

## When to trigger HITL

| Trigger | When |
|---|---|
| `reason_code == "INSUFFICIENT_PRIVILEGE"` | Vengtoo says the agent lacks privilege — escalate |
| Tool is in a pre-classified high-risk list | Always escalate regardless of Vengtoo decision |
| Amount or impact exceeds threshold | Business rule: e.g., transactions > $10,000 |
| Irreversible action detected | Deletes, deployments, sends — require human sign-off |

---

## Core pattern

```python
from vengtoo import Vengtoo
from enum import Enum

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

HIGH_RISK_TOOLS = {"send_email", "delete_record", "transfer_funds", "deploy", "publish"}

async def call_tool_safe(tool_name: str, args: dict, agent_id: str, user_id: str):
    # 1. Check Vengtoo
    resp = client.authorize({
        "subject": {"id": agent_id, "type": "agent", "on_behalf_of": {"id": user_id, "type": "user"}},
        "resource": {"type": "tool", "id": tool_name, "properties": args},
        "action": {"name": "invoke"},
    })

    # 2. Hard-stop: Vengtoo denied (not an escalation code)
    if not resp.decision and resp.context.reason_code != "INSUFFICIENT_PRIVILEGE":
        raise PermissionError(f"Denied: {resp.context.reason}")

    # 3. HITL: Vengtoo says insufficient privilege, or tool is high-risk
    needs_approval = (
        not resp.decision and resp.context.reason_code == "INSUFFICIENT_PRIVILEGE"
    ) or tool_name in HIGH_RISK_TOOLS

    if needs_approval:
        approved = await request_human_approval(user_id, tool_name, args, resp.context)
        if not approved:
            raise PermissionError(f"Human rejected tool call: {tool_name}")

    # 4. Execute
    return await execute_tool(tool_name, args)
```

---

## Notification integrations

### Slack
```python
import httpx

async def request_human_approval_slack(user_id: str, tool_name: str, args: dict, context) -> bool:
    approval_id = generate_approval_id()

    await httpx.AsyncClient().post(
        os.environ["SLACK_WEBHOOK_URL"],
        json={
            "blocks": [
                {"type": "section", "text": {"type": "mrkdwn",
                    "text": f"*Agent approval requested*\nUser: `{user_id}`\nTool: `{tool_name}`\nReason: _{context.reason}_"}},
                {"type": "actions", "elements": [
                    {"type": "button", "text": {"type": "plain_text", "text": "Approve"},
                     "style": "primary", "value": approval_id, "action_id": "approve"},
                    {"type": "button", "text": {"type": "plain_text", "text": "Deny"},
                     "style": "danger", "value": approval_id, "action_id": "deny"},
                ]},
            ]
        }
    )

    # Poll for approval (Slack sends a webhook to your server on button click)
    return await wait_for_approval(approval_id, timeout=300)
```

### Email (simple)
```python
import smtplib
from email.mime.text import MIMEText

def send_approval_email(approver_email: str, tool_name: str, args: dict, approve_url: str, deny_url: str):
    body = f"""
    An AI agent is requesting approval to call: {tool_name}

    Arguments: {json.dumps(args, indent=2)}

    [Approve]({approve_url}) | [Deny]({deny_url})
    """
    msg = MIMEText(body, "plain")
    msg["Subject"] = f"[Vengtoo] Approval required: {tool_name}"
    msg["From"] = "no-reply@yourdomain.com"
    msg["To"] = approver_email

    with smtplib.SMTP(os.environ["SMTP_HOST"]) as s:
        s.sendmail(msg["From"], [approver_email], msg.as_string())
```

### Webhook (generic)
```python
async def request_human_approval_webhook(user_id: str, tool_name: str, args: dict) -> bool:
    approval_id = generate_approval_id()

    await httpx.AsyncClient().post(
        os.environ["APPROVAL_WEBHOOK_URL"],
        json={
            "approval_id": approval_id,
            "agent_id": "agent:my-bot",
            "user_id": user_id,
            "tool": tool_name,
            "args": args,
            "expires_at": (datetime.utcnow() + timedelta(minutes=30)).isoformat() + "Z",
        }
    )

    return await wait_for_approval(approval_id, timeout=1800)
```

---

## Approval state store

A simple in-memory store for development. Use Redis or a DB for production.

```python
import asyncio
from typing import Optional

_pending: dict[str, asyncio.Future] = {}

def generate_approval_id() -> str:
    import uuid
    return str(uuid.uuid4())

async def wait_for_approval(approval_id: str, timeout: int = 300) -> bool:
    future: asyncio.Future[bool] = asyncio.get_event_loop().create_future()
    _pending[approval_id] = future
    try:
        return await asyncio.wait_for(future, timeout=timeout)
    except asyncio.TimeoutError:
        return False
    finally:
        _pending.pop(approval_id, None)

def resolve_approval(approval_id: str, approved: bool):
    """Call this from your webhook handler when the human responds."""
    if approval_id in _pending and not _pending[approval_id].done():
        _pending[approval_id].set_result(approved)
```

**Webhook handler (FastAPI):**
```python
@app.post("/approvals/{approval_id}/approve")
async def approve(approval_id: str):
    resolve_approval(approval_id, True)
    return {"status": "approved"}

@app.post("/approvals/{approval_id}/deny")
async def deny(approval_id: str):
    resolve_approval(approval_id, False)
    return {"status": "denied"}
```

---

## Node.js HITL

```ts
const HIGH_RISK = new Set(['sendEmail', 'deleteRecord', 'transferFunds'])

async function callToolSafe(name: string, args: unknown, agentId: string, userId: string) {
  const resp = await vengtoo.authorize({
    subject: { id: agentId, type: 'agent', onBehalfOf: { id: userId, type: 'user' } },
    resource: { type: 'tool', id: name, properties: args },
    action: { name: 'invoke' },
  })

  const needsApproval =
    (!resp.decision && resp.context.reasonCode === 'INSUFFICIENT_PRIVILEGE') ||
    HIGH_RISK.has(name)

  if (needsApproval) {
    const approved = await requestApproval({ userId, toolName: name, args, context: resp.context })
    if (!approved) throw new Error(`Human denied: ${name}`)
  } else if (!resp.decision) {
    throw new Error(`Denied: ${resp.context.reason}`)
  }

  return executeTool(name, args)
}
```

---

## Audit logging approved actions

Always log HITL approvals:
```python
async def call_tool_safe(tool_name, args, agent_id, user_id):
    # ... authorization + approval ...
    if needs_approval:
        approved = await request_human_approval(...)
        logger.info({
            "event": "hitl_decision",
            "tool": tool_name,
            "agent_id": agent_id,
            "user_id": user_id,
            "approved": approved,
            "args_hash": hashlib.sha256(json.dumps(args, sort_keys=True).encode()).hexdigest(),
            "timestamp": datetime.utcnow().isoformat(),
        })
```
