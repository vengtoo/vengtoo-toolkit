# Vengtoo — Go Integration Reference

## Install
```bash
go get github.com/vengtoo/vengtoo-go@latest
```

The module path is `github.com/vengtoo/vengtoo-go`. Do not use `github.com/authzx/authzx-go` — that is the old deprecated path and will fail.

---

## Gin

### Client setup (`internal/authz/client.go`)
```go
package authz

import (
    vengtoo "github.com/vengtoo/vengtoo-go"
    "os"
)

var Client = vengtoo.NewClient(
    os.Getenv("VENGTOO_API_KEY"),
    // vengtoo.WithBaseURL("http://localhost:8181"), // local agent
)
```

### Gin middleware
```go
func Require(resourceType, action string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetString("user_id") // set by your auth middleware
        resourceID := c.Param("id")      // your system's ID (e.g. "doc-123"), not a Vengtoo UUID

        allowed, err := authz.Client.Check(
            c.Request.Context(),
            vengtoo.Subject{ExternalID: userID, Type: "user"},
            action,
            vengtoo.Resource{Type: resourceType, ExternalID: resourceID},
        )
        if err != nil || !allowed {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "forbidden"})
            return
        }
        c.Next()
    }
}

// Usage
router.GET("/documents/:id", authMiddleware, authz.Require("document", "read"), getDocument)
```

> Use `ExternalID` when passing your own system's identifiers (URL params, DB IDs, slugs).
> Use `ID` only when you have the Vengtoo internal UUID.

> For **type-level checks** (policy applies to all resources of this type), omit both `ID` and `ExternalID`:
> `vengtoo.Resource{Type: resourceType}` — the SDK sends `id: "*"` automatically.

> `AuthorizeWithPolling` is for Human-in-the-Loop flows only — it pauses and waits for a human to approve in the Vengtoo dashboard. Use `Authorize()` or `Check()` for all standard authorization checks.

### Full authorize response (with reason)
```go
resp, err := authz.Client.Authorize(ctx, &vengtoo.AuthorizeRequest{
    Subject:  vengtoo.Subject{ExternalID: userID, Type: "user"},
    Resource: vengtoo.Resource{Type: "document", ExternalID: documentID},
    Action:   vengtoo.Action{Name: "write"},
    Context:  map[string]interface{}{"ip": c.ClientIP()},
})
if err != nil {
    c.AbortWithStatusJSON(500, gin.H{"error": "authorization service unavailable"})
    return
}
if !resp.Decision {
    c.AbortWithStatusJSON(403, gin.H{
        "error":       "forbidden",
        "reason":      resp.Context.Reason,
        "access_path": resp.Context.AccessPath,
    })
    return
}
```

---

## Chi

### Middleware
```go
func VengtooMiddleware(resourceType, action string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            userID := r.Context().Value("user_id").(string)
            resourceID := chi.URLParam(r, "id")

            allowed, err := authz.Client.Check(
                r.Context(),
                vengtoo.Subject{ExternalID: userID, Type: "user"},
                action,
                vengtoo.Resource{Type: resourceType, ExternalID: resourceID},
            )
            if err != nil || !allowed {
                http.Error(w, `{"error":"forbidden"}`, http.StatusForbidden)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Usage
r.With(VengtooMiddleware("document", "read")).Get("/documents/{id}", getDocument)
```

---

## Environment setup

`.env`:
```
VENGTOO_API_KEY=azx_...
# VENGTOO_BASE_URL=http://localhost:8181
```

Load with `godotenv` at startup:
```go
import "github.com/joho/godotenv"

func main() {
    _ = godotenv.Load()
    // ...
}
```

---

## Error handling
```go
resp, err := client.Authorize(ctx, req)
if err != nil {
    switch {
    case vengtoo.IsAuthError(err):
        // 401 — bad API key
    case vengtoo.IsServerError(err):
        // 5xx — retries exhausted, fail closed
    default:
        // network error — fail closed
    }
}
```

---

## Batch evaluation
```go
results, err := client.CheckBatch(ctx, &vengtoo.BatchEvaluationRequest{
    Subject: vengtoo.Subject{ExternalID: userID, Type: "user"},
    Items: []vengtoo.EvaluationItem{
        {Resource: vengtoo.Resource{Type: "document", ExternalID: "doc-1"}, Action: vengtoo.Action{Name: "read"}},
        {Resource: vengtoo.Resource{Type: "invoice", ExternalID: "inv-7"}, Action: vengtoo.Action{Name: "approve"}},
    },
})
```
