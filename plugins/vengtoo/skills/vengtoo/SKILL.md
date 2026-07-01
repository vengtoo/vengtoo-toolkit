---
name: vengtoo
description: |
  Sets up Vengtoo authorization in the current project. Detects the stack, scans
  for routes, wires the SDK on one endpoint as a working trial, and verifies with
  a live decision. The goal is confidence that it works — not a full rollout.

  Triggers: "add authorization", "set up access control", "install vengtoo",
  "protect my routes", "add vengtoo to this project", "wire up permissions",
  "how do I add auth checks", "integrate vengtoo", "add vengtoo sdk",
  "set up vengtoo", "protect this endpoint", "add access control"
metadata:
  category: [setup, integration]
  sdk: [node, python, go, all]
---

# Set Up Vengtoo Authorization

Vengtoo answers one question: **can this subject perform this action on this resource?**
Your app calls one endpoint; Vengtoo returns `decision: true/false` plus a reason.

This skill wires that call into **one endpoint** as a working trial. Once it's confirmed working, you can expand to the rest of your routes.

You are setting up **authorization** (who can do what) — not authentication (who the user is). Vengtoo never sees identity tokens. It only receives the subject ID your backend already knows.

---

## Reference routing

Load the appropriate reference file before proceeding:

| Stack detected | Reference file |
|---|---|
| Node.js + Express / Fastify / Koa | `references/node.md` |
| Node.js + Next.js | `references/node.md` (Next.js section) |
| Python + FastAPI | `references/python.md` |
| Python + Flask / Django | `references/python.md` |
| Go + Gin | `references/go.md` |
| Go + Chi / net/http | `references/go.md` |
| Unknown / multiple | Ask the user before proceeding |

---

## Progress reporting

**Emit a `✔` line every time a significant step completes.** Do not batch them at the end — output each one as it happens so the user sees live progress. Only report steps that actually ran; skip steps that were not needed and do not show them as N/A. If a step was already done (SDK installed, API key present, resource type exists), emit the tick with a note that it was already in place.

Example of what this looks like in practice (adapts to what actually happened):
```
✔ Detected Go + Chi — 8 endpoints found
✔ VENGTOO_API_KEY placeholder added to .env — fill in from app.vengtoo.com → Settings → API Keys
✔ vengtoo-go v0.3.1 installed
✔ Authorize("document", "read") wired on GET /documents/:id
✔ Resource type `document` created (actions: read, write, delete)
✔ ALLOW policy created — alice → document:read
✔ alice   → GET /documents/doc-1 → 200 (ALLOW)
✔ bob     → GET /documents/doc-1 → 403 (NO_POLICY_MATCH, denied)
```

---

## Flow

### Step 1: Detect stack and scan the project

Look only in the current working directory.

**If a project exists**, check:
- `package.json` → Node.js; read `dependencies` for framework
- `go.mod` → Go; check for gin, chi, fiber
- `requirements.txt` / `pyproject.toml` → Python; check for fastapi, flask, django

Emit ticks for what you find — only include lines that are true:
```
✔ Detected Go + Chi
✔ Found 12 API endpoints
✔ Identified likely resource types: documents, projects
```

**If no project exists** (empty directory, no recognised files), first clarify which situation this is:

> "Looks like there's no project here. Are you:
> **a)** In the wrong folder — your project is somewhere else?
> **b)** Starting fresh and want to try Vengtoo with a new demo project?"

**If (a)** — stop and tell them to navigate to their project directory, then re-run `/vengtoo`.

**If (b)** — ask which language they prefer, then scaffold a minimal demo app with one protected endpoint:
- A single route (e.g. `GET /documents/:id`) that requires authorization
- The Vengtoo SDK wired on it
- A `.env` with a placeholder for `VENGTOO_API_KEY`

Do not ask the user about routes — there are none yet. You are creating the route for them as a working example. The goal is that they can run it immediately and see a live decision.

---

### Step 2: Check for existing Vengtoo integration

- Is the SDK already installed?
- Is there an existing authorization middleware?

**If already integrated and working** → emit ticks for what's in place, tell the user it's already wired, and stop.

**If partially integrated** → diagnose and fix rather than re-install.

---

### Step 3: Add the API key placeholder

Do not try to find, read, or fill in the API key. Always add the placeholder line to `.env` (or `.env.local` for Next.js) and tell the user to fill it in themselves:

```
VENGTOO_API_KEY=
```

If `.env` already exists, append the line (only if `VENGTOO_API_KEY` is not already present — do not add a duplicate). If it does not exist, create it with that line.

Also ensure `.env` is in `.gitignore` — add it if missing.

Then tell the user:
> "I've added `VENGTOO_API_KEY` to your `.env` — fill in the value from **app.vengtoo.com → Settings → API Keys** before running."

Emit:
```
✔ VENGTOO_API_KEY placeholder added to .env
```

Do not proceed past this step until the user confirms they have filled in the key.

---

### Step 4: Pick one endpoint to try

Pick a route that requires a subject, has a clear resource, and is not a public/health endpoint. Suggest it and confirm:

> "I'll wire Vengtoo on `GET /documents/:id` first so you can see it working — does that work?"

For a fresh project, pick the route you just scaffolded.

---

### Step 5: Install SDK

Load the reference file for the detected stack, then install:

**Node.js:** `npm install @vengtoo/sdk`
**Python:** `pip install vengtoo`
**Go:** `go get github.com/vengtoo/vengtoo-go@latest`

If already installed, emit:
```
✔ @vengtoo/sdk already installed
```

Otherwise install and emit:
```
✔ @vengtoo/sdk installed
```

---

### Step 6: Wire the authorization check

Follow the framework pattern from the reference file. Key principles:

- Wire at **middleware level** on the chosen route — not inside the handler
- **Fail closed**: if Vengtoo is unreachable, deny by default

**For an existing project**: do not assume you know where the caller's identity lives. Read the actual code — trace from an existing protected route backwards through any middleware, context, session, or request struct to find where the authenticated user's ID is set and how it's accessed downstream. Projects vary widely: some use a JWT `sub` in context, some a session store, some a custom claims struct, some a user object on the request, some something unexpected. Find the real thing and use it. Never introduce the placeholder into a real project.

**For a fresh project**: use the placeholder auth from the reference file. It reads `X-User-ID` from the request header so you can test Vengtoo immediately without setting up real auth.

Show the wired code and confirm with the user before writing it. Once written, emit:
```
✔ Authorize("document", "read") wired on GET /documents/:id
```

---

### Step 7: Set up the minimum cloud resources for testing

The authorization check will return `NO_POLICY_MATCH` until there is a resource type and policy in Vengtoo. Do not stop and ask — create the minimum right now and emit a tick for each:

1. One resource type matching the chosen endpoint's resource (e.g. `document` with a `read` action) → emit `✔ Resource type 'document' created`
2. One starter ALLOW policy targeting that resource **type** (not a specific instance) → emit `✔ ALLOW policy created — alice → document:read`
3. A test subject if needed — use the ID the user will pass as their identity

**Do NOT create resource instances.** A type-level policy covers all resources of that type — the evaluation request can pass any `external_id` and the policy will match on resource type + action alone. No resource instance needs to exist in Vengtoo. Creating instances here is unnecessary and misleading.

If the user already has resource types or policies set up, use what is there and emit:
```
✔ Resource type 'document' already exists — using it
```

---

### Step 8: Verify

Run a curl or SDK test against the wired endpoint. Emit the result as a tick:

```
✔ alice   → GET /documents/doc-1 → 200 (ALLOW)
✔ bob     → GET /documents/doc-1 → 403 (NO_POLICY_MATCH, denied)
```

If `NO_POLICY_MATCH` on the allowed user → the policy or subject assignment is missing. Check the chain.
If `EXPLICIT_DENY` → a deny policy is overriding. Check priorities.
If `401` → bad or missing API key.

---

### Step 9: Confirm done

Once the verify ticks are in, close with one line:

> "Vengtoo is live. Apply `Authorize(resourceType, action)` to your other routes the same way."

Do not push the user toward other skills. If they ask what else they can do, mention roles, conditions, or a local agent — only if they ask.

---

## Error handling

| Error | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` | Bad or missing API key | Check `VENGTOO_API_KEY` |
| `NO_POLICY_MATCH` | No policy covers this request | Expected before setup — create one or run `/vengtoo-policies` |
| `Connection refused` | Wrong base URL | Check `VENGTOO_BASE_URL`; default is `https://api.vengtoo.com` |
| SDK not found | Install failed | Re-run the install command; check package manager output |
