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
from vengtoo import Vengtoo, Subject, Resource
from fastapi import Depends, HTTPException, Request

def require(resource_type: str, action: str):
    async def dependency(request: Request):
        user_id = request.state.user_id   # set by your auth middleware
        resource_id = request.path_params.get("id", resource_type)

        resp = await client.async_authorize({
            "subject": {"id": user_id, "type": "user"},
            "resource": {"type": resource_type, "id": resource_id},
            "action": {"name": action},
        })
        if not resp.decision:
            raise HTTPException(status_code=403, detail=resp.context.reason)
    return dependency

@app.get("/documents/{id}")
async def get_document(id: str, _=Depends(require("document", "read"))):
    return {"id": id}
```

### Manual check
```python
from vengtoo import Subject, Resource, Action

resp = await client.async_authorize({
    "subject": Subject(id=current_user.id, type="user"),
    "resource": Resource(type="document", id=document_id),
    "action": Action(name="write"),
    "context": {"ip": request.client.host},
})

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
from vengtoo import Vengtoo
from flask import request, jsonify, g

client = Vengtoo(api_key=os.environ["VENGTOO_API_KEY"])

def require_permission(resource_type: str, action: str):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            resource_id = kwargs.get("id", resource_type)
            resp = client.check(
                subject={"id": g.user_id, "type": "user"},
                action=action,
                resource={"type": resource_type, "id": resource_id},
            )
            if not resp:
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

---

## Batch evaluation
```python
results = client.authorize_batch({
    "subject": {"id": user_id, "type": "user"},
    "items": [
        {"resource": {"type": "document", "id": "doc-1"}, "action": {"name": "read"}},
        {"resource": {"type": "invoice", "id": "inv-7"}, "action": {"name": "approve"}},
    ]
})
for r in results:
    print(r.decision, r.context.reason)
```
