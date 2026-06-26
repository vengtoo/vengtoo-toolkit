---
name: vengtoo-agent
description: |
  Deploys and configures the Vengtoo Agent locally — a sidecar that syncs your
  policy bundle from the cloud and evaluates authorization decisions in-memory,
  eliminating cloud round-trips on the hot path.

  Triggers: "deploy local agent", "run vengtoo locally", "set up local evaluation",
  "install vengtoo agent", "run agent", "sub-millisecond authorization",
  "offline authorization", "local policy evaluation", "vengtoo sidecar",
  "docker vengtoo", "vengtoo localhost", "self-hosted vengtoo"
metadata:
  category: [deployment, infrastructure]
---

# Deploy the Vengtoo Agent

The agent syncs your policy bundle from Vengtoo Cloud every 30s and evaluates decisions
locally. No round-trip to the cloud on each request — decisions happen in-memory in microseconds.
It caches the bundle to disk, so it keeps serving even if the cloud is temporarily unreachable.

---

## Reference routing

| User need | Reference file |
|---|---|
| Full config reference (all env vars) | `references/config-reference.md` |
| Kubernetes / sidecar deployment | `references/kubernetes.md` |

---

## Pre-flight checks (run silently)

1. Is Docker available? `docker info 2>/dev/null`
2. Is there a `docker-compose.yml` or `docker-compose.yaml` **in the current project root only** (`./docker-compose.yml`)? Do not search parent directories or sibling folders.
3. Is `VENGTOO_API_KEY` set or in `.env`?
4. Is the agent already running? `curl -s http://localhost:8181/healthz`

---

## Flow

### Step 1: Verify API key

If `VENGTOO_API_KEY` is not set, ask for it. The agent needs it to pull the policy bundle.
Store in `.env` if not already there.

### Step 2: Deploy

**If `./docker-compose.yml` exists in the current project root**, add the agent as a service:

```yaml
services:
  vengtoo-agent:
    image: vengtoo/agent:latest
    ports:
      - "8181:8181"
    environment:
      - VENGTOO_API_KEY=${VENGTOO_API_KEY}
      - VENGTOO_DECISION_LOG=true
      - VENGTOO_CACHE_DIR=/var/lib/vengtoo/bundles
    volumes:
      - vengtoo-cache:/var/lib/vengtoo/bundles
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8181/readyz"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  vengtoo-cache:
```

Then: `docker compose up -d vengtoo-agent`

**If no Compose file**, run directly:
```bash
docker run -d \
  --name vengtoo-agent \
  -e VENGTOO_API_KEY=${VENGTOO_API_KEY} \
  -e VENGTOO_DECISION_LOG=true \
  -p 8181:8181 \
  -v vengtoo-cache:/var/lib/vengtoo/bundles \
  --restart unless-stopped \
  vengtoo/agent:latest
```

**If Docker not available**, offer binary install:
```bash
# macOS / Linux
curl -sSL https://github.com/vengtoo/agent/releases/latest/download/install.sh | sh

# Or Go install
go install github.com/vengtoo/agent/cmd/agent@latest

# Run
VENGTOO_API_KEY=azx_... vengtoo-agent
```

### Step 3: Wait for readiness

Poll `/readyz` until 200 — the agent is loading the bundle from cache or cloud:
```bash
until curl -sf http://localhost:8181/readyz; do sleep 1; done
echo "Agent ready"
```

### Step 4: Verify with a test decision
```bash
curl -s -X POST http://localhost:8181/access/v1/evaluation \
  -H "Content-Type: application/json" \
  -d '{"subject":{"id":"test-user","type":"user"},"resource":{"type":"test","id":"r1"},"action":{"name":"read"}}'
```

Expected: `{"decision":false,"context":{"reason":"No matching policy","reason_code":"NO_POLICY_MATCH"}}`
`NO_POLICY_MATCH` means the agent is working — no policy covers this test request yet.

### Step 5: Update SDK to use local agent

Update the project's Vengtoo client to point to the agent instead of the cloud:

**Node.js:**
```ts
const vengtoo = new Vengtoo({
  apiKey: process.env.VENGTOO_API_KEY,
  baseUrl: process.env.VENGTOO_BASE_URL ?? 'http://localhost:8181',
})
```

**Python:**
```python
client = Vengtoo(
    api_key=os.environ["VENGTOO_API_KEY"],
    base_url=os.environ.get("VENGTOO_BASE_URL", "http://localhost:8181"),
)
```

**Go:**
```go
client := vengtoo.NewClient(
    os.Getenv("VENGTOO_API_KEY"),
    vengtoo.WithBaseURL(getEnvOrDefault("VENGTOO_BASE_URL", "http://localhost:8181")),
)
```

Add to `.env`:
```
VENGTOO_BASE_URL=http://localhost:8181
```

---

## Health monitoring

```bash
# Liveness — is the process running?
curl http://localhost:8181/healthz

# Readiness — is a bundle loaded?
curl http://localhost:8181/readyz

# Prometheus metrics
curl http://localhost:8181/metrics | grep vengtoo_agent
```

Key metrics to watch:
- `vengtoo_agent_degraded` — `1` means serving stale cache (cloud unreachable)
- `vengtoo_agent_bundle_last_sync_timestamp_seconds` — when bundle last synced
- `vengtoo_agent_decisions_total` — decision volume

---

## Error handling

| Symptom | Cause | Fix |
|---|---|---|
| `/readyz` returns 503 `"reason":"no bundle loaded"` | No bundle loaded yet | Wait; check `VENGTOO_API_KEY` is valid |
| `/readyz` returns 503 `"reason":"bundle loaded but OPA query compilation failed"` | Bundle synced but OPA can't compile the policy (e.g., unregistered built-in, syntax error) | Check `docker logs vengtoo-agent` for `rego_type_error` or parse errors; report to Vengtoo support |
| `/healthz` shows `degraded: true` | Cloud unreachable; serving stale cache | Check network; agent continues serving |
| `connection refused` on port 8181 | Agent not running | `docker ps` to check; restart container |
| `401` from agent | Agent couldn't authenticate with cloud | Verify `VENGTOO_API_KEY` |
| Bundle never syncs | Firewall blocking outbound to api.vengtoo.com | Open egress on port 443 |
| Policies with rate-limiting modifiers always pass | `vengtoo_rate_limit` is cloud-only (needs Redis shared state). Agent registers a no-op returning `true` (not rate-limited). Rate limiting is silently skipped on the local agent — use cloud mode if enforcement is required. | By design |

---

## When NOT to use this skill

- **Already deployed, just debugging decisions** → use `/vengtoo-debug`
- **Production Kubernetes deployment** → load `references/kubernetes.md` first
- **Just want cloud mode** → skip this skill; cloud mode works without it
