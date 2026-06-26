# Vengtoo — Python Integration Reference

## Install
```bash
pip install vengtoo
```

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

### Dependency injection (recommended — reusable across routes)
```python
from vengtoo import Vengtoo, Subject, Resource, Action, AuthorizeRequest
from fastapi import Depends, HTTPException, Request

def require(resource_type: str, action: str):
    async def dependency(request: Request):
        user_id = request.state.user_id   # set by your auth middleware
        resource_id = request.path_params.get("id")

        resp = await client.async_authorize(AuthorizeRequest(
            subject=Subject(external_id=user_id, type="user"),
            resource=Resource(type=resource_type, external_id=resource_id) if resource_id else Resource(type=resource_type),
            action=Action(name=action),
        ))
        if not resp.decision:
            raise HTTPException(status_code=403, detail=resp.context.reason if resp.context else "forbidden")
    return dependency

@app.get("/documents/{id}")
async def get_document(id: str, _=Depends(require("document", "read"))):
    return {"id": id}
```

> `external_id` is your own system's identifier (user ID, slug, DB UUID). Use `id` only when you have a Vengtoo-internal UUID.

> For **type-level checks** (policy applies to all resources of this type), omit the resource ID entirely: `Resource(type=resource_type)`. The SDK sends `id: "*"` automatically.

### FastAPI shorthand (built-in dependency factory)
```python
# The SDK has a built-in `require()` for simple cases:
@app.get("/documents/{id}")
async def get_document(id: str, _=Depends(client.require("document", "read"))):
    return {"id": id}
```

### Manual check (when you need the full response)
```python
from vengtoo import Subject, Resource, Action, AuthorizeRequest

resp = await client.async_authorize(AuthorizeRequest(
    subject=Subject(external_id=current_user.id, type="user"),
    resource=Resource(type="document", external_id=document_id),
    action=Action(name="write"),
    context={"ip": request.client.host},
))

if not resp.decision:
    raise HTTPException(status_code=403, detail=resp.context.reason)
```

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

### Decorator pattern
```python
from functools import wraps
from vengtoo import Vengtoo, Subject, Resource, Action, AuthorizeRequest
from flask import request, jsonify, g

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def require_permission(resource_type: str, action: str):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            resource_id = kwargs.get("id")
            resp = client.authorize(AuthorizeRequest(
                subject=Subject(external_id=g.user_id, type="user"),
                resource=Resource(type=resource_type, external_id=resource_id) if resource_id else Resource(type=resource_type),
                action=Action(name=action),
            ))
            if not resp.decision:
                return jsonify({"error": "Forbidden"}), 403
            return f(*args, **kwargs)
        return wrapper
    return decorator

@app.route("/documents/<id>")
@require_permission("document", "read")
def get_document(id):
    return {"id": id}
```

---

## Environment setup

`.env`:
```
VENGTOO_API_KEY=azx_...
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
| FastAPI async route | `await client.async_check()` / `await client.async_authorize()` |
| Flask / sync code | `client.check()` / `client.authorize()` |
| Background task | sync is fine; async if already in async context |

`authorize()` and `async_authorize()` both take a typed `AuthorizeRequest` object — not a plain dict.

---

## Batch evaluation
```python
from vengtoo import BatchEvaluationRequest, BatchEvalItem, Subject, Resource, Action

results = await client.async_authorize_batch(BatchEvaluationRequest(
    subject=Subject(external_id=user_id, type="user"),
    evaluations=[
        BatchEvalItem(resource=Resource(type="document", external_id="doc-1"), action=Action(name="read")),
        BatchEvalItem(resource=Resource(type="invoice", external_id="inv-7"), action=Action(name="approve")),
    ]
))
for r in results.evaluations:
    print(r.decision, r.context.reason if r.context else "")
```
