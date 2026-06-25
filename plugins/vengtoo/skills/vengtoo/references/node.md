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

// Protect a resource type across all matching routes
app.use('/documents', vengtoo.middleware('document', 'read', {
  subject: (req) => req.user?.id,          // extract subject ID from your auth layer
  resource: (req) => req.params.id,        // extract resource ID from route param
  onDenied: (req, res) => res.status(403).json({ error: 'Forbidden' }),
}))
```

### Manual check (when you need the full response)
```ts
const resp = await vengtoo.authorize({
  subject: { id: req.user.id, type: 'user' },
  resource: { type: 'document', id: req.params.id },
  action: { name: 'write' },
  context: { ip: req.ip },
})

if (!resp.decision) {
  return res.status(403).json({ error: resp.context.reason })
}
```

### Error handling
```ts
import { VengtooError } from '@vengtoo/sdk'

try {
  const resp = await vengtoo.authorize(req)
} catch (err) {
  if (err instanceof VengtooError && err.isAuthError()) {
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
    subject: { id: session.user.id, type: 'user' },
    resource: { type: 'document', id: params.id },
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

export async function middleware(request: NextRequest) {
  const userId = request.headers.get('x-user-id')
  const resourceId = request.nextUrl.pathname.split('/').pop()

  const resp = await vengtoo.authorize({
    subject: { id: userId!, type: 'user' },
    resource: { type: 'document', id: resourceId! },
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
  subject: { id: userId, type: 'user' },   // default for all items
  items: [
    { resource: { type: 'document', id: 'doc-1' }, action: { name: 'read' } },
    { resource: { type: 'document', id: 'doc-2' }, action: { name: 'write' } },
    { resource: { type: 'invoice', id: 'inv-7' }, action: { name: 'approve' } },
  ]
})
// results[0].decision, results[1].decision, results[2].decision
```
