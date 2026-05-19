# Route Handlers

Custom request handlers in `route.ts` files. Equivalent to Pages Router's API Routes. Use for public HTTP endpoints — things clients/third parties call.

`route.ts` and `page.tsx` **cannot coexist** at the same segment.

## Shape

```ts
// app/api/route.ts
export async function GET(request: Request) { /* ... */ }
export async function POST(request: Request) { /* ... */ }
// PUT, PATCH, DELETE, HEAD, OPTIONS also supported
```

Unsupported methods → 405. If you don't define `OPTIONS`, Next adds it automatically with an `Allow` header.

## Typed route context

Params come via a `RouteContext` helper (global, no import):

```ts
// app/users/[id]/route.ts
import type { NextRequest } from 'next/server'

export async function GET(_req: NextRequest, ctx: RouteContext<'/users/[id]'>) {
  const { id } = await ctx.params
  return Response.json({ id })
}
```

## `Request`/`Response` extensions

Native `Request`/`Response` work. Next.js provides `NextRequest` and `NextResponse` with helpers:

- `request.nextUrl` — parsed URL with easy `pathname`/`searchParams` access
- `NextResponse.redirect(url)` / `.rewrite(url)` / `.json(body)` / `.next(options)`
- Cookie helpers on both

```ts
import { type NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const q = request.nextUrl.searchParams.get('q')
  return NextResponse.json({ q })
}
```

## Reading the body

`request.json()`, `request.formData()`, `request.text()`. `GET`/`HEAD` have no body.

Body can only be read once. `request.clone()` for a second read.

```ts
export async function POST(request: Request) {
  const form = await request.formData()
  const email = form.get('email')
  return Response.json({ email })
}
```

## Caching (without Cache Components)

Route Handlers are **not cached by default**. Only `GET` can be cached. Opt in with segment config:

```ts
export const dynamic = 'force-static'

export async function GET() {
  const data = await fetch('https://api.example.com/').then(r => r.json())
  return Response.json(data)
}
```

Other methods are never cached, even next to a cached `GET`.

## With Cache Components enabled

`GET` handlers follow the same prerendering model as UI pages. They run at request time, but prerender if no runtime/uncached data is accessed.

Static (prerendered):

```ts
export async function GET() {
  return Response.json({ name: 'Next.js' })
}
```

Runtime (dynamic):

```ts
import { headers } from 'next/headers'
export async function GET() {
  const ua = (await headers()).get('user-agent')
  return Response.json({ ua })
}
```

Cached (prerendered despite DB access):

```ts
import { cacheLife } from 'next/cache'

async function getProducts() {
  'use cache'
  cacheLife('hours')
  return db.query('SELECT * FROM products')
}

export async function GET() {
  return Response.json(await getProducts())
}
```

Note: `'use cache'` **cannot** be placed directly in the `GET` body — extract to a helper.

## Non-UI responses

Route Handlers can serve any content type. Common patterns:

### RSS

```ts
// app/rss.xml/route.ts
export async function GET() {
  const feed = await buildRssFeed() // XML string
  return new Response(feed, {
    headers: { 'content-type': 'application/xml' },
  })
}
```

### Content negotiation (HTML for browsers, Markdown for agents)

Rewrite in `next.config.ts`:

```ts
async rewrites() {
  return [{
    source: '/docs/:slug*',
    destination: '/docs/md/:slug*',
    has: [{ type: 'header', key: 'accept', value: '(.*)text/markdown(.*)' }],
  }]
}
```

Handler at the destination:

```ts
export async function GET(_: Request, ctx: RouteContext<'/docs/md/[...slug]'>) {
  const { slug } = await ctx.params
  const md = await getDocsMd({ slug })
  if (!md) return new Response(null, { status: 404 })
  return new Response(md, {
    headers: { 'Content-Type': 'text/markdown; charset=utf-8', Vary: 'Accept' },
  })
}
```

`Vary: Accept` is essential — without it, shared caches may serve the wrong variant.

### Webhooks

```ts
import { revalidateTag } from 'next/cache'
import { type NextRequest, NextResponse } from 'next/server'

export async function POST(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('token')
  if (token !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 })
  }
  const tag = request.nextUrl.searchParams.get('tag')
  if (!tag) return NextResponse.json({ error: 'missing tag' }, { status: 400 })
  revalidateTag(tag)
  return NextResponse.json({ success: true })
}
```

### Auth callback

```ts
export async function GET(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('session_token')
  const redirect = request.nextUrl.searchParams.get('redirect_url')
  const response = NextResponse.redirect(new URL(redirect!, request.url))
  response.cookies.set({
    name: '_token', value: token!, path: '/', secure: true, httpOnly: true,
  })
  return response
}
```

### Proxy to a backend

```ts
export async function POST(request: Request, ctx: RouteContext<'/api/[...slug]'>) {
  const cloned = request.clone()
  if (!(await isValidRequest(cloned))) return new Response(null, { status: 400 })
  const { slug } = await ctx.params
  const proxyURL = new URL(slug.join('/'), 'https://upstream.example.com')
  return fetch(new Request(proxyURL, request))
}
```

## Security

- **Always authenticate/authorize inside the handler.** Route Handlers are public POST targets.
- **Validate input** — size, content type, schema. Don't trust anything.
- **Avoid using incoming request headers as outgoing response headers.** Modify upstream headers via `NextResponse.next({ request: { headers } })` (not visible to clients); set response headers explicitly.
- **Rate limit** — either in code or at your platform.
- **Timeouts** on upstream calls to prevent abuse.
- Don't expose sensitive data in error messages.

## Library patterns

Many libraries expose a factory:

```ts
// app/api/[...path]/route.ts
import { createHandler } from 'third-party-lib'
const handler = createHandler({ /* opts */ })
export { handler as GET, handler as POST }
```

Library may also provide a `proxy.ts` factory — older libraries still call it "middleware".

## Caveats

- **Don't fetch from Route Handlers inside Server Components.** It adds an HTTP round trip; prerendered Server Components will fail at build time (no server is listening).
- **Static export (`output: 'export'`)**: only `GET` handlers with `export const dynamic = 'force-static'`. No dynamic reads.
- **Serverless deployment**: Route Handlers are typically lambdas — ephemeral, no shared state, no WebSockets.
