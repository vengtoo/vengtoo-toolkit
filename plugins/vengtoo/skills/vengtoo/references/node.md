# Vengtoo — Node.js Integration Reference

## Express / Fastify / Koa

### Install
```bash
npm install @vengtoo/sdk
```

### Client setup (`src/vengtoo.ts`)
```ts
import { Vengtoo } from '@vengtoo/sdk'

export const vengtoo = new Vengtoo({
  apiKey: process.env.VENGTOO_API_KEY!,
  // baseUrl: 'http://localhost:8181'  // uncomment when using local agent
})
```

### Express middleware (recommended — protects all routes in one place)
```ts
import { vengtoo } from './vengtoo'

// 3rd arg: function that extracts the subject ID from the request
app.use('/documents', vengtoo.middleware('document', 'read', (req) => req.user?.id))
```

### Manual check (when you need the full response)
```ts
const resp = await vengtoo.authorize({
  subject: { external_id: req.user.id, type: 'user' },
  resource: { type: 'document', external_id: req.params.id },
  action: { name: 'write' },
  context: { ip: req.ip },
})

if (!resp.decision) {
  return res.status(403).json({ error: resp.context?.reason })
}
```

> Use `external_id` for your own system's identifiers (user IDs, slugs, DB UUIDs). Use `id` only when you have a Vengtoo-internal UUID.

> For **type-level checks** (policy covers all resources of this type), omit the resource ID entirely: `{ type: 'document' }`. The SDK sends `id: "*"` automatically.

### Error handling
```ts
import { VengtooError } from '@vengtoo/sdk'

try {
  const resp = await vengtoo.authorize(req)
} catch (err) {
  if (err instanceof VengtooError && err.isAuthError) {  // isAuthError is a getter, not a method
    // Bad API key — check VENGTOO_API_KEY
  }
  // Fail closed on all other errors
  return res.status(403).json({ error: 'Authorization service unavailable' })
}
```

---

## Next.js (App Router)

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

### API Route middleware
```ts
// middleware.ts
import { vengtoo } from '@/lib/vengtoo'
import { getToken } from 'next-auth/jwt'

export async function middleware(request: NextRequest) {
  // Extract the user's identity from the JWT — never trust a raw header from the client
  const token = await getToken({ req: request })
  const userId = token?.sub   // JWT sub claim — your auth provider's user ID
  const resourceId = request.nextUrl.pathname.split('/').pop()

  const resp = await vengtoo.authorize({
    subject: { external_id: userId!, type: 'user' },
    resource: { type: 'document', external_id: resourceId! },
    action: { name: 'read' },
  })

  if (!resp.decision) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }
  return NextResponse.next()
}

export const config = { matcher: ['/api/documents/:path*'] }
```

---

## Environment setup

`.env.local` (Next.js) or `.env`:
```
VENGTOO_API_KEY=azx_...
# VENGTOO_BASE_URL=http://localhost:8181  # local agent
```

Add to `.gitignore`:
```
.env
.env.local
.env*.local
```

---

## Batch evaluation (multiple checks in one call)
```ts
const results = await vengtoo.checkBatch({
  subject: { external_id: userId, type: 'user' },
  evaluations: [
    { resource: { type: 'document', external_id: 'doc-1' }, action: { name: 'read' } },
    { resource: { type: 'document', external_id: 'doc-2' }, action: { name: 'write' } },
    { resource: { type: 'invoice', external_id: 'inv-7' }, action: { name: 'approve' } },
  ]
})
// results[0], results[1], results[2] — booleans
```
