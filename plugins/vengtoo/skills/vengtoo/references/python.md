# Vengtoo — Python Integration Reference

## Install
```bash
pip install vengtoo
```

---

## How subject identity works

Vengtoo needs the caller's `sub` from the JWT sent in `Authorization: Bearer <token>`.

**User calls**: parse the Bearer token, extract `sub`, pass it as `external_id`.
**Service-to-service**: the `sub` in a client credentials token is the `client_id`. Same flow.

Never trust a raw header like `X-User-ID` from the client. Always extract from a verified token.

---

## FastAPI

### Client setup (`app/vengtoo.py`)
```python
from vengtoo import Vengtoo
import os

client = Vengtoo(
    api_key=os.environ["VENGTOO_API_KEY"],
    # base_url="http://localhost:8181"  # uncomment when using local agent
)
```

### Demo / fresh project — placeholder auth

Use this when you want to try Vengtoo without setting up real authentication first.
It reads `X-User-ID` from the request header and returns it as `sub` — the same value
the real dependency would produce. The Vengtoo middleware below does not change either way.

```python
from fastapi import Header, HTTPException

def get_subject(x_user_id: str = Header(default=None)) -> str:
    # DEMO ONLY: reads a plain header supplied by the caller.
    # This is intentionally insecure — any client can claim any identity.
    # In production, replace this with JWT verification below:
    # parse the Bearer token, verify the signature, and extract sub from claims.
    # That way identity is cryptographically proven, not self-asserted.
    if not x_user_id:
        raise HTTPException(status_code=401, detail="missing X-User-ID header")
    return x_user_id
```

Test with:
```bash
curl -H "X-User-ID: alice" http://localhost:8000/documents/doc-1
```

### Production — extract sub from Bearer JWT
```python
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
import jwt  # pip install pyjwt

bearer = HTTPBearer()
JWT_SECRET = os.environ["JWT_SECRET"]

def get_subject(credentials: HTTPAuthorizationCredentials = Security(bearer)) -> str:
    try:
        payload = jwt.decode(
            credentials.credentials,
            JWT_SECRET,
            algorithms=["HS256"],
        )
        sub = payload.get("sub")
        if not sub:
            raise HTTPException(status_code=401, detail="unauthorized")
        return sub
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="unauthorized")
```

> If using Auth0, Cognito, or another IdP, use their JWKS endpoint for verification. The `sub` extraction is the same.

### Vengtoo dependency
```python
from vengtoo import Subject, Resource, Action, AuthorizeRequest

def authorize(resource_type: str, action: str):
    async def dependency(
        request: Request,
        sub: str = Depends(get_subject),
    ):
        resource_id = request.path_params.get("id")
        resp = await client.async_authorize(AuthorizeRequest(
            subject=Subject(external_id=sub, type="user"),
            resource=Resource(type=resource_type, external_id=resource_id) if resource_id else Resource(type=resource_type),
            action=Action(name=action),
        ))
        if not resp.decision:
            raise HTTPException(
                status_code=403,
                detail=resp.context.reason if resp.context else "forbidden"
            )
    return dependency

# Usage — auth runs as part of the dependency chain
@app.get("/documents/{id}")
async def get_document(id: str, _=Depends(authorize("document", "read"))):
    return {"id": id}
```

### Manual check (when you need the full response)
```python
resp = await client.async_authorize(AuthorizeRequest(
    subject=Subject(external_id=sub, type="user"),
    resource=Resource(type="document", external_id=document_id),
    action=Action(name="write"),
    context={"ip": request.client.host},
))

if not resp.decision:
    raise HTTPException(status_code=403, detail=resp.context.reason)
```

> Use `external_id` for your own system's identifiers. Use `id` only when you have a Vengtoo-internal UUID.

> For **type-level checks**, omit the resource ID: `Resource(type=resource_type)`.

### Client lifecycle (FastAPI lifespan)
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await client.async_close()

app = FastAPI(lifespan=lifespan)
```

---

## Flask

### Auth decorator — extract sub from Bearer JWT
```python
from functools import wraps
from flask import request, jsonify, g
import jwt

JWT_SECRET = os.environ["JWT_SECRET"]

def auth_required(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        header = request.headers.get("Authorization", "")
        if not header.startswith("Bearer "):
            return jsonify({"error": "unauthorized"}), 401
        try:
            payload = jwt.decode(header[7:], JWT_SECRET, algorithms=["HS256"])
            g.sub = payload.get("sub")
            if not g.sub:
                return jsonify({"error": "unauthorized"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "unauthorized"}), 401
        return f(*args, **kwargs)
    return wrapper
```

### Vengtoo decorator
```python
from vengtoo import Vengtoo, Subject, Resource, Action, AuthorizeRequest

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def authorize(resource_type: str, action: str):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            resource_id = kwargs.get("id")
            resp = client.authorize(AuthorizeRequest(
                subject=Subject(external_id=g.sub, type="user"),
                resource=Resource(type=resource_type, external_id=resource_id) if resource_id else Resource(type=resource_type),
                action=Action(name=action),
            ))
            if not resp.decision:
                return jsonify({"error": "forbidden"}), 403
            return f(*args, **kwargs)
        return wrapper
    return decorator

# Usage — auth_required runs first
@app.route("/documents/<id>")
@auth_required
@authorize("document", "read")
def get_document(id):
    return {"id": id}
```

---

## Environment setup

`.env`:
```
VENGTOO_API_KEY=azx_...
JWT_SECRET=your-secret
# VENGTOO_BASE_URL=http://localhost:8181
```

Load with `python-dotenv`:
```python
from dotenv import load_dotenv
load_dotenv()
```

---

## Sync vs async

| Use case | Method |
|---|---|
| FastAPI async route | `await client.async_authorize()` |
| Flask / sync code | `client.authorize()` |

---

## Batch evaluation
```python
from vengtoo import BatchEvaluationRequest, BatchEvalItem

results = await client.async_authorize_batch(BatchEvaluationRequest(
    subject=Subject(external_id=sub, type="user"),
    evaluations=[
        BatchEvalItem(resource=Resource(type="document", external_id="doc-1"), action=Action(name="read")),
        BatchEvalItem(resource=Resource(type="invoice", external_id="inv-7"), action=Action(name="approve")),
    ]
))
```
