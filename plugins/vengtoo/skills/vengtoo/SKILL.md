---
name: vengtoo
description: |
  Sets up Vengtoo authorization in the current project. Detects the stack, installs
  the right SDK, wires authorization at the best interception point, and verifies
  with a live test call.

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
Your app calls one endpoint; Vengtoo returns `allowed: true/false` plus a reason.
This skill wires that call into your project.

You are setting up **authorization** (who can do what) — not authentication (who the user is).
Vengtoo never sees identity tokens. It only receives an opaque subject ID your backend already knows.

---

## Reference routing

Load the appropriate reference file based on the detected stack before proceeding:

| Stack detected | Reference file to load |
|---|---|
| Node.js + Express / Fastify / Koa | `references/node.md` |
| Node.js + Next.js | `references/node.md` (Next.js section) |
| Python + FastAPI | `references/python.md` |
| Python + Flask / Django | `references/python.md` |
| Go + Gin | `references/go.md` |
| Go + Chi / Fiber | `references/go.md` |
| Unknown / multiple | Ask the user before proceeding |

---

## Flow

### Step 1: Identify the stack (current directory only, then ask if unclear)

Look **only in the current working directory** — do not read files from parent directories, sibling folders, or any path outside the project root.

Check only these files if they exist in the project root:
- `package.json` → Node.js; read `dependencies` for framework
- `go.mod` → Go; read for gin, chi, fiber, echo
- `requirements.txt` / `pyproject.toml` / `setup.py` → Python; read for fastapi, flask, django
- `Cargo.toml` → Rust (note: no SDK yet, offer curl-based approach)

Also check:
- Is `VENGTOO_API_KEY` set in the shell environment or in `.env` / `.env.local` in the project root?

**If `VENGTOO_API_KEY` is found, tell the user explicitly:**
> "I found a `VENGTOO_API_KEY` already set in your environment — I'll use that. Let me know if you'd like to use a different one."

Never use a discovered key silently without informing the user.

**If no stack can be detected (empty project, no recognised files), stop and ask:**
> "What language and framework is this project using? (e.g. Node.js + Express, Python + FastAPI, Go + Gin)"

Do NOT guess, do NOT look in other directories. Wait for the user's answer before continuing.

### Step 2: Check existing integration

If the project already has Vengtoo integrated, read the existing code and follow its patterns — the user's own middleware is ground truth. Use the reference file only when starting from scratch or when the existing code is clearly broken. Never copy patterns from external demo or example projects outside this codebase.

If Vengtoo SDK is already installed, check if it's configured and working:
- Is `VENGTOO_API_KEY` present?
- Is there a client instantiation in source files?
- Run a quick curl test against `POST /access/v1/evaluation` to verify

If already working → report status and suggest next steps (`/vengtoo-policies`, `/vengtoo-agent`, `/vengtoo-abac`).

If misconfigured → diagnose and fix rather than re-install.

### Step 3: Get API key

If `VENGTOO_API_KEY` is not set:
- Tell the user: "Get your API key from the Vengtoo console at https://app.vengtoo.com → Settings → API Keys"
- Once provided, add it to `.env` (or `.env.local` for Next.js projects)
- Add `.env` to `.gitignore` if not already there — **never commit the key**

### Step 4: Install SDK

Load the language-specific reference file, then install:

**Node.js:**
```bash
npm install @vengtoo/sdk
```

**Python:**
```bash
pip install vengtoo
```

**Go:**
```bash
go get github.com/vengtoo/vengtoo-go@latest
```
The module path is `github.com/vengtoo/vengtoo-go`. Never use `github.com/authzx/authzx-go` — that is the old deprecated path.

### Step 5: Wire the first authorization check

Load the reference file for the detected framework and follow the framework-specific integration pattern. Key principles:

- **Where to intercept**: middleware level is always better than per-handler — one change protects all routes
- **Subject ID**: use whatever ID your auth layer already has (JWT sub, session user_id, etc.)
- **Resource ID**: the entity being accessed (document ID, resource UUID, etc.)
- **Action**: what the subject is trying to do (`read`, `write`, `delete`, etc.)
- **Fail closed**: if Vengtoo is unreachable, deny by default — never fail open

### Step 6: Verify with a live test

After wiring, generate and run a curl test against the protected endpoint. Confirm:
1. The request reaches the authorization check
2. A `decision` is returned (even if `false` — that means it's working)
3. The reason code makes sense (`NO_POLICY_MATCH` is expected before policies exist)

If `NO_POLICY_MATCH`: tell the user this is expected — they need policies. Suggest `/vengtoo-policies` as next step.

---

## After setup — suggest next steps

Once the SDK is wired and the first check returns a decision:

1. **`/vengtoo-policies`** — Create resource types, subjects, roles, and policies so requests are actually allowed
2. **`/vengtoo-agent`** — Deploy the local agent for sub-millisecond decisions without cloud round-trips
3. **`/vengtoo-abac`** — Add fine-grained conditions (time windows, IP allowlists, attribute matching)
4. **`/vengtoo-mcp`** — If this project involves AI agents and MCP tool calls

---

## Error handling

| Error | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` | Bad or missing API key | Check `VENGTOO_API_KEY` is set and matches console |
| `decision: false, NO_POLICY_MATCH` | No policy covers this request | Expected — run `/vengtoo-policies` to create one |
| `Connection refused` | Wrong base URL or agent not running | Check `VENGTOO_BASE_URL`; if using agent, run `/vengtoo-agent` |
| SDK not found | Wrong package manager / install failed | Check package manager output; try `npm install` / `pip install` again |
| `VENGTOO_API_KEY not set` | Env var not loaded | Check `.env` file exists and is being loaded at startup |

---

## When NOT to use this skill

- **Already set up, just debugging a denial** → use `/vengtoo-debug`
- **Need to create policies** → use `/vengtoo-policies`
- **Authorizing MCP tool calls** → use `/vengtoo-mcp`
- **Deploying the local agent** → use `/vengtoo-agent`
