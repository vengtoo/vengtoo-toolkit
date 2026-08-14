# Shadow-compare: prove the translation matches

Before enforcing with Vengtoo, prove it makes the **same decisions** the incumbent
does. Replay a coverage set of real requests through Vengtoo's evaluation endpoint
(in your **non-prod** environment) and diff every decision.

## 1. Build the coverage set

A JSON array of cases, each with the request and the incumbent's known decision. Best
sources, in order:

1. The incumbent's **own test cases** (they already encode intended behavior).
2. Its **decision logs** (real traffic — the most representative).
3. A **generated grid** across the Stage-1 inventory (every subject × resource type ×
   action) — use when you have neither of the above.

```json
[
  { "subject": {"external_id": "alice", "type": "user"},
    "resource": {"type": "document", "id": "*"},
    "action": {"name": "read"},
    "expected": true },
  { "subject": {"external_id": "bob", "type": "user"},
    "resource": {"type": "document", "external_id": "doc-9"},
    "action": {"name": "delete"},
    "expected": false }
]
```

## 2. Replay through Vengtoo

Use an API key scoped to the **lower env**. The endpoint is the standard AuthZEN path;
the environment is selected by the key.

```python
#!/usr/bin/env python3
# shadow.py — replay a coverage set and report divergences.
# usage: VENGTOO_API_KEY=vgt_... python3 shadow.py coverage.json
import json, os, sys, urllib.request

ENDPOINT = os.getenv("VENGTOO_EVAL_URL", "https://api.vengtoo.com/access/v1/evaluation")
KEY = os.environ["VENGTOO_API_KEY"]  # env-scoped key -> validate in the lower env first

def decide(case):
    body = json.dumps({k: case[k] for k in ("subject", "resource", "action")
                       if k in case} | {"context": case.get("context", {})}).encode()
    req = urllib.request.Request(ENDPOINT, data=body, method="POST", headers={
        "Authorization": f"Bearer {KEY}", "Content-Type": "application/json"})
    with urllib.request.urlopen(req) as r:
        return bool(json.load(r).get("decision"))

cases = json.load(open(sys.argv[1]))
match = false_allow = false_deny = 0
for c in cases:
    got, want = decide(c), bool(c["expected"])
    if got == want:
        match += 1
    elif got and not want:
        false_allow += 1
        print(f"FALSE ALLOW  {c['subject'].get('external_id')} {c['action']['name']} "
              f"{c['resource'].get('type')} — Vengtoo allows, incumbent denied")
    else:
        false_deny += 1
        print(f"FALSE DENY   {c['subject'].get('external_id')} {c['action']['name']} "
              f"{c['resource'].get('type')} — Vengtoo denies, incumbent allowed")

total = len(cases)
print(f"\n{match}/{total} match | {false_allow} false-allow | {false_deny} false-deny")
sys.exit(1 if (false_allow or false_deny) else 0)
```

## 3. Read the results

- **False allow** (Vengtoo allows what the incumbent denied) — the **dangerous** class;
  a policy is too broad or a DENY/condition is missing. Fix before anything else.
- **False deny** (Vengtoo denies what the incumbent allowed) — will break legitimate
  access; usually a missing policy, wrong action, or a broken assignment chain.
- Iterate on Stage 2 until every case matches, or each remaining divergence is
  **explained and intended** (e.g. the incumbent had a bug you're deliberately fixing).

## 4. Gate

Only proceed to **Stage 4 (cutover)** once parity holds on a coverage set you trust to
be representative. If you can't trust the coverage set, run a **continuous production
shadow** instead — dual-run Vengtoo alongside the incumbent at the live PEP (log-only,
enforce nothing) for a few days and diff the decision logs before flipping.
