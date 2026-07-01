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

## Flow

### Step 1: Detect stack and scan the project

Look only in the current working directory.

**If a project exists**, check:
- `package.json` → Node.js; read `dependencies` for framework
- `go.mod` → Go; check for gin, chi, fiber
- `requirements.txt` / `pyproject.toml` → Python; check for fastapi, flask, django

Then scan for route definitions and report what you find:
```
✔ Detected Go + Chi
✔ Found 12 API endpoints
✔ Identified likely resource types: documents, projects
✔ Identified likely actions: read, write, delete
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

- Is `VENGTOO_API_KEY` set in the environment or `.env`?
- Is the SDK already installed?
- Is there an existing authorization middleware?

**If already integrated and working** → report status, suggest `/vengtoo-policies` or `/vengtoo-debug` as next steps. Stop here.

**If partially integrated** → diagnose and fix rather than re-install.

**If `VENGTOO_API_KEY` is found**, say so explicitly:
> "Found a `VENGTOO_API_KEY` already set — I'll use that. Let me know if you'd like a different one."

---

### Step 3: Get API key

If no key is found:
> "Get your API key from app.vengtoo.com → Settings → API Keys"

Once provided, add it to `.env` (or `.env.local` for Next.js). Add `.env` to `.gitignore` — never commit the key.

---

### Step 4: Pick one endpoint to try

Show the user what was found (or ask for a route if fresh project) and confirm:

> "I'll wire Vengtoo on one endpoint first so you can see it working. Which route should we try? I'd suggest `GET /documents/:id` — does that work?"

Pick a route that:
- Requires a subject (user is known by this point)
- Has a clear resource (something with an ID)
- Is not a public/health-check endpoint

---

### Step 5: Install SDK

Load the reference file for the detected stack, then install:

**Node.js:** `npm install @vengtoo/sdk`
**Python:** `pip install vengtoo`
**Go:** `go get github.com/vengtoo/vengtoo-go@latest`

---

### Step 6: Wire the authorization check

Follow the framework pattern from the reference file. Key principles:

- Wire at **middleware level** on the chosen route — not inside the handler
- **Resource ID**: pass as `ExternalID` — your own system's identifier, not a Vengtoo UUID
- **Fail closed**: if Vengtoo is unreachable, deny by default

**For an existing project**: do not assume you know where the caller's identity lives. Read the actual code — trace from an existing protected route backwards through any middleware, context, session, or request struct to find where the authenticated user's ID is set and how it's accessed downstream. Projects vary widely: some use a JWT `sub` in context, some a session store, some a custom claims struct, some a user object on the request, some something unexpected. Find the real thing and use it. Never introduce the placeholder into a real project.

**For a fresh project**: use the placeholder auth from the reference file. It reads `X-User-ID` from the request header so you can test Vengtoo immediately without setting up real auth. The placeholder comment explains exactly what to replace it with when moving to production.

Show the wired code and confirm with the user before writing it.

---

### Step 7: Offer a quick cloud setup for testing

The authorization check will return `NO_POLICY_MATCH` until there is a resource type and policy in Vengtoo. Ask:

> "To test this end-to-end, I can create a resource type (`document`) and a simple ALLOW policy in your Vengtoo account. Want me to do that now, or will you set it up manually in the console?"

**If yes** — use the `vengtoo-policies` API reference to create:
1. One resource type (matching the chosen endpoint's resource)
2. One starter ALLOW policy covering the tested action
3. A test subject (or use an existing one if they have it)

**If no** — tell them what to create manually and link to the console.

---

### Step 8: Verify

Run a test against the wired endpoint. The expected result after the policy is in place:

```
✔ GET /documents/123 → decision: true (ALLOW) ✓
```

If `NO_POLICY_MATCH` → the policy or assignment is missing. Check the chain.
If `decision: false, EXPLICIT_DENY` → a deny policy is overriding. Check priorities.
If `401` → bad or missing API key.

---

### Step 9: Confirm and suggest next steps

Once working:

> "Vengtoo is wired and returning decisions. This is on one endpoint — you can now expand the middleware to the rest of your routes."

Suggest:
- **`/vengtoo-policies`** — build out the full authorization model (roles, more resource types, RBAC)
- **`/vengtoo-agent`** — run a local agent for sub-millisecond decisions without cloud round-trips
- **`/vengtoo-abac`** — add conditions (time windows, IP allowlists, attribute matching)

---

## Error handling

| Error | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` | Bad or missing API key | Check `VENGTOO_API_KEY` |
| `NO_POLICY_MATCH` | No policy covers this request | Expected before setup — create one or run `/vengtoo-policies` |
| `Connection refused` | Wrong base URL | Check `VENGTOO_BASE_URL`; default is `https://api.vengtoo.com` |
| SDK not found | Install failed | Re-run the install command; check package manager output |
