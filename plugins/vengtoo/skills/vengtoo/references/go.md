# Vengtoo — Go Integration Reference

## Install
```bash
go get github.com/vengtoo/vengtoo-go@latest
```

The module path is `github.com/vengtoo/vengtoo-go`. Do not use `github.com/authzx/authzx-go` — that is the old deprecated path and will fail.

---

## How subject identity works

Vengtoo needs the caller's identity — the `sub` claim from the JWT the client sent.

**User calls**: the client sends `Authorization: Bearer <jwt>`. Your auth middleware verifies the token and sets `sub` in context. Vengtoo middleware reads it.

**Service-to-service**: the service sends a client credentials JWT. The `sub` is the `client_id`. Same flow — auth middleware sets it in context, Vengtoo reads it.

Never trust a raw header like `X-User-ID` from the client. Always extract identity from a verified token.

---

## Auth middleware

### Demo / fresh project — placeholder

Use this when you want to try Vengtoo without setting up real authentication first.
It reads `X-User-ID` from the request header and sets it as `sub` in context — the same
slot the real middleware would fill. The Vengtoo middleware below does not change either way.

```go
func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // DEMO ONLY: reads a plain header supplied by the caller.
        // This is intentionally insecure — any client can claim any identity.
        // In production, replace this with the JWT middleware below:
        // parse the Bearer token, verify the signature, and extract sub from claims.
        // That way identity is cryptographically proven, not self-asserted.
        sub := r.Header.Get("X-User-ID")
        if sub == "" {
            http.Error(w, `{"error":"missing X-User-ID header"}`, http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), SubjectKey, sub)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

Test with:
```bash
curl -H "X-User-ID: alice" http://localhost:8080/documents/doc-1
```

### Production — extract sub from Bearer JWT

Wire this before the Vengtoo middleware. It verifies the token and sets `sub` in context.

```go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "github.com/golang-jwt/jwt/v5"
)

type contextKey string
const SubjectKey contextKey = "sub"

func Auth(jwtSecret []byte) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            header := r.Header.Get("Authorization")
            if !strings.HasPrefix(header, "Bearer ") {
                http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
                return
            }
            tokenStr := strings.TrimPrefix(header, "Bearer ")

            token, err := jwt.Parse(tokenStr, func(t *jwt.Token) (interface{}, error) {
                return jwtSecret, nil
            }, jwt.WithExpirationRequired())
            if err != nil {
                http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
                return
            }

            claims, ok := token.Claims.(jwt.MapClaims)
            sub, hasSub := claims["sub"].(string)
            if !ok || !hasSub || sub == "" {
                http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
                return
            }

            ctx := context.WithValue(r.Context(), SubjectKey, sub)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

> If you use an external IdP (Auth0, Cognito, Keycloak), replace the HMAC key with JWKS verification. The `sub` extraction is the same.

---

## Gin

```go
package middleware

import (
    "net/http"
    "os"

    "github.com/gin-gonic/gin"
    vengtoo "github.com/vengtoo/vengtoo-go"
)

var client = vengtoo.NewClient(
    os.Getenv("VENGTOO_API_KEY"),
    // vengtoo.WithBaseURL("http://localhost:8181"), // local agent
)

func Authorize(resourceType, action string) gin.HandlerFunc {
    return func(c *gin.Context) {
        sub := c.GetString("sub") // set by auth middleware

        // Type-level policy (covers all resources of this type — most common):
        // Omit ExternalID. Matches on type + action alone; no resource instance
        // needs to exist in Vengtoo. Note: "*" is a REST-layer sentinel only —
        // never pass it via the SDK, just leave ExternalID empty.
        resource := vengtoo.Resource{Type: resourceType}

        // Instance-level policy (one specific resource registered in Vengtoo):
        // resource = vengtoo.Resource{Type: resourceType, ExternalID: c.Param("id")}

        allowed, err := client.Check(
            c.Request.Context(),
            vengtoo.Subject{ExternalID: sub, Type: "user"},
            action,
            resource,
        )
        if err != nil || !allowed {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "forbidden"})
            return
        }
        c.Next()
    }
}

// Usage — auth middleware runs first, then Vengtoo
// router.GET("/documents/:id", Auth(jwtSecret), Authorize("document", "read"), getDocument)
```

### Full authorize response (with reason)
```go
resp, err := client.Authorize(ctx, &vengtoo.AuthorizeRequest{
    Subject:  vengtoo.Subject{ExternalID: sub, Type: "user"},
    Resource: vengtoo.Resource{Type: "document", ExternalID: c.Param("id")},
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

```go
package middleware

import (
    "net/http"
    "os"

    vengtoo "github.com/vengtoo/vengtoo-go"
)

var client = vengtoo.NewClient(
    os.Getenv("VENGTOO_API_KEY"),
    // vengtoo.WithBaseURL("http://localhost:8181"), // local agent
)

func Authorize(resourceType, action string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            sub, ok := r.Context().Value(SubjectKey).(string)
            if !ok || sub == "" {
                http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
                return
            }

            // Type-level: omit ExternalID — matches on type + action, no instance needed.
            // Instance-level: vengtoo.Resource{Type: resourceType, ExternalID: chi.URLParam(r, "id")}
            allowed, err := client.Check(
                r.Context(),
                vengtoo.Subject{ExternalID: sub, Type: "user"},
                action,
                vengtoo.Resource{Type: resourceType},
            )
            if err != nil || !allowed {
                http.Error(w, `{"error":"forbidden"}`, http.StatusForbidden)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Usage — auth runs first in the chain
// r.Use(Auth(jwtSecret))
// r.With(Authorize("document", "read")).Get("/documents/{id}", getDocument)
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
    Subject: vengtoo.Subject{ExternalID: sub, Type: "user"},
    Items: []vengtoo.EvaluationItem{
        {Resource: vengtoo.Resource{Type: "document", ExternalID: "doc-1"}, Action: vengtoo.Action{Name: "read"}},
        {Resource: vengtoo.Resource{Type: "invoice", ExternalID: "inv-7"}, Action: vengtoo.Action{Name: "approve"}},
    },
})
```
