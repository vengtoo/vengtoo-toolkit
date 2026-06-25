# Vengtoo Agent — Config Reference

All `VENGTOO_*` environment variables. Every config can also be set via `authzx-agent.yml`.

---

## Core (required)

| Variable | Default | Notes |
|---|---|---|
| `VENGTOO_API_KEY` | — | **Required.** API key used to pull the policy bundle from Vengtoo Cloud. |
| `VENGTOO_CLOUD_URL` | `https://api.vengtoo.com` | Cloud endpoint. Override for private deployments. |

---

## HTTP server

| Variable | Default | Notes |
|---|---|---|
| `VENGTOO_PORT` | `8181` | Port the agent listens on. |
| `VENGTOO_HOST` | `0.0.0.0` | Bind address. Use `127.0.0.1` to restrict to localhost. |
| `VENGTOO_TLS_CERT` | — | Path to TLS certificate. If set, serves HTTPS. |
| `VENGTOO_TLS_KEY` | — | Path to TLS private key. Required if `VENGTOO_TLS_CERT` is set. |

---

## Bundle sync

| Variable | Default | Notes |
|---|---|---|
| `VENGTOO_BUNDLE_SYNC_INTERVAL` | `30s` | How often to poll cloud for an updated bundle. Minimum: `10s`. |
| `VENGTOO_CACHE_DIR` | `/var/lib/vengtoo/bundles` | Where the bundle is cached to disk. Agent serves from cache if cloud is unreachable. |
| `VENGTOO_BUNDLE_SIGNATURE_REQUIRED` | `false` | If `true`, rejects bundles that don't have a valid Ed25519 signature. Recommended for production. |
| `VENGTOO_BUNDLE_PUBLIC_KEY` | — | Path to Ed25519 public key for bundle signature verification. Required if `VENGTOO_BUNDLE_SIGNATURE_REQUIRED=true`. |

---

## Decision logging

| Variable | Default | Notes |
|---|---|---|
| `VENGTOO_DECISION_LOG` | `false` | If `true`, logs every evaluation decision locally. |
| `VENGTOO_DECISION_LOG_PATH` | stdout | File path to write decision logs. Default: stdout (JSON). |
| `VENGTOO_AUDIT_FORWARD` | `false` | If `true`, forwards decision logs to Vengtoo Cloud audit service. |
| `VENGTOO_AUDIT_BATCH_SIZE` | `100` | Number of decisions to batch before forwarding. |
| `VENGTOO_AUDIT_FLUSH_INTERVAL` | `5s` | Max time to hold decisions before flushing, even if batch not full. |

---

## Performance

| Variable | Default | Notes |
|---|---|---|
| `VENGTOO_EVAL_TIMEOUT` | `100ms` | Max time per evaluation. Returns error if exceeded. |
| `VENGTOO_MAX_CONCURRENT_EVALS` | `1000` | Max parallel evaluations. |
| `VENGTOO_ENABLE_METRICS` | `true` | Exposes Prometheus metrics at `/metrics`. |

---

## Agent hosting mode

| Variable | Default | Notes |
|---|---|---|
| `VENGTOO_AGENT_HOSTING` | `self` | `self` = you run the agent. `vengtoo` = Vengtoo-hosted agent (no binary needed). |

Set `VENGTOO_AGENT_HOSTING=vengtoo` if using Vengtoo's managed agent — all other config is ignored.

---

## Example `authzx-agent.yml`

```yaml
# VENGTOO_* env vars override these values at runtime

port: 8181
host: "0.0.0.0"

cloud_url: "https://api.vengtoo.com"

bundle:
  sync_interval: "30s"
  cache_dir: "/var/lib/vengtoo/bundles"
  signature_required: false

decision_log:
  enabled: true
  path: "/var/log/vengtoo/decisions.jsonl"

audit:
  forward: true
  batch_size: 100
  flush_interval: "5s"

metrics:
  enabled: true

eval:
  timeout: "100ms"
  max_concurrent: 1000
```

---

## Docker Compose example (full config)

```yaml
services:
  vengtoo-agent:
    image: vengtoo/agent:latest
    ports:
      - "8181:8181"
    environment:
      - VENGTOO_API_KEY=${VENGTOO_API_KEY}
      - VENGTOO_BUNDLE_SYNC_INTERVAL=30s
      - VENGTOO_BUNDLE_SIGNATURE_REQUIRED=true
      - VENGTOO_BUNDLE_PUBLIC_KEY=/etc/vengtoo/bundle-signing.pub
      - VENGTOO_DECISION_LOG=true
      - VENGTOO_AUDIT_FORWARD=true
      - VENGTOO_CACHE_DIR=/var/lib/vengtoo/bundles
    volumes:
      - vengtoo-cache:/var/lib/vengtoo/bundles
      - ./bundle-signing.pub:/etc/vengtoo/bundle-signing.pub:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8181/readyz"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  vengtoo-cache:
```
