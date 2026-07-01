# Vengtoo — Node.js Integration Reference

## Install
```bash
npm install @vengtoo/sdk
```

---

## How subject identity works

Vengtoo needs the caller's `sub` from the JWT sent in `Authorization: Bearer <token>`.

**User calls**: parse the Bearer token, extract `sub`, pass it as `external_id`.
**Service-to-service**: the `sub` in a client credentials token is the `client_id`. Same flow.

Never trust a raw header like `X-User-ID` from the client. Always extract from a verified token.

---

## Client setup (`src/vengtoo.ts`)
```ts
import { Vengtoo } from '@vengtoo/sdk'

export const vengtoo = new Vengtoo({
  apiKey: process.env.VENGTOO_API_KEY!,
  // baseUrl: 'http://localhost:8181'  // uncomment when using local agent
})
```

---

## Express / Fastify

### Demo / fresh project — placeholder auth

Use this when you want to try Vengtoo without setting up real authentication first.
It reads `X-User-ID` from the request header and attaches it as `sub` — the same field
the real middleware would set. The Vengtoo middleware below does not change either way.

```ts
export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  // DEMO ONLY: reads a plain header supplied by the caller.
  // This is intentionally insecure — any client can claim any identity.
  // In production, replace this with JWT verification below:
  // parse the Bearer token, verify the signature, and extract sub from claims.
  // That way identity is cryptographically proven, not self-asserted.
  const sub = req.headers['x-user-id'] as string
  if (!sub) return res.status(401).json({ error: 'missing X-User-ID header' })
  req.sub = sub
  next()
}
```

Test with:
```bash
curl -H "X-User-ID: alice" http://localhost:3000/documents/doc-1
```

### Production — extract sub from Bearer JWT

Wire this before the Vengtoo middleware. It verifies the token and attaches `sub` to the request.

```ts
import jwt from 'jsonwebtoken'
import { Request, Response, NextFunction } from 'express'

const JWT_SECRET = process.env.JWT_SECRET!

export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'unauthorized' })
  }

  try {
    const token = jwt.verify(header.slice(7), JWT_SECRET) as jwt.JwtPayload
    if (!token.sub) return res.status(401).json({ error: 'unauthorized' })
    req.sub = token.sub  // attach to request for downstream middleware
    next()
  } catch {
    return res.status(401).json({ error: 'unauthorized' })
  }
}

// Extend Express Request type (src/types.d.ts)
declare global {
  namespace Express {
    interface Request { sub?: string }
  }
}
```

> If using Auth0, Cognito, or another IdP, use their JWKS-based verification instead of a shared secret. The `sub` extraction is the same.

### Vengtoo middleware
```ts
import { vengtoo } from './vengtoo'

function requirePermission(resourceType: string, action: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      const resp = await vengtoo.authorize({
        subject: { external_id: req.sub!, type: 'user' },
        resource: { type: resourceType, external_id: req.params.id },
        action: { name: action },
      })
      if (!resp.decision) {
        return res.status(403).json({ error: resp.context?.reason ?? 'forbidden' })
      }
      next()
    } catch {
      // Fail closed — deny if Vengtoo is unreachable
      return res.status(403).json({ error: 'authorization service unavailable' })
    }
  }
}

// Usage — auth middleware runs first
app.get('/documents/:id', authMiddleware, requirePermission('document', 'read'), getDocument)
```

### Manual check (when you need the full response)
```ts
const resp = await vengtoo.authorize({
  subject: { external_id: req.sub!, type: 'user' },
  resource: { type: 'document', external_id: req.params.id },
  action: { name: 'write' },
  context: { ip: req.ip },
})

if (!resp.decision) {
  return res.status(403).json({ error: resp.context?.reason })
}
```

> Use `external_id` for your own system's identifiers. Use `id` only when you have a Vengtoo-internal UUID.

> For **type-level checks** (policy covers all resources of this type), omit the resource ID: `{ type: 'document' }`.

---

## Next.js (App Router)

### Middleware (`middleware.ts`)
```ts
import { NextRequest, NextResponse } from 'next/server'
import { getToken } from 'next-auth/jwt'
import { vengtoo } from '@/lib/vengtoo'

export async function middleware(request: NextRequest) {
  // Extract sub from the JWT — never trust a raw header from the client
  const token = await getToken({ req: request })
  if (!token?.sub) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 })
  }

  const resourceId = request.nextUrl.pathname.split('/').pop()

  const resp = await vengtoo.authorize({
    subject: { external_id: token.sub, type: 'user' },
    resource: { type: 'document', external_id: resourceId! },
    action: { name: 'read' },
  })

  if (!resp.decision) {
    return NextResponse.json({ error: 'forbidden' }, { status: 403 })
  }
  return NextResponse.next()
}

export const config = { matcher: ['/api/documents/:path*'] }
```

### Server Component check
```ts
// app/documents/[id]/page.tsx
import { vengtoo } from '@/lib/vengtoo'
import { auth } from '@/lib/auth'
import { notFound } from 'next/navigation'

export default async function DocumentPage({ params }) {
  const session = await auth()
  const resp = await vengtoo.authorize({
    subject: { external_id: session.user.id, type: 'user' },
    resource: { type: 'document', external_id: params.id },
    action: { name: 'read' },
  })

  if (!resp.decision) notFound()
  // render...
}
```

---

## Environment setup

`.env.local` (Next.js) or `.env`:
```
VENGTOO_API_KEY=azx_...
JWT_SECRET=your-secret
# VENGTOO_BASE_URL=http://localhost:8181  # local agent
```

Add to `.gitignore`:
```
.env
.env.local
.env*.local
```

---

## Batch evaluation
```ts
const results = await vengtoo.checkBatch({
  subject: { external_id: req.sub!, type: 'user' },
  evaluations: [
    { resource: { type: 'document', external_id: 'doc-1' }, action: { name: 'read' } },
    { resource: { type: 'document', external_id: 'doc-2' }, action: { name: 'write' } },
    { resource: { type: 'invoice', external_id: 'inv-7' }, action: { name: 'approve' } },
  ]
})
```
