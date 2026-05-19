# Proxy (formerly Middleware)

**In Next.js 16, Middleware was renamed to Proxy.** File is now `proxy.ts` (or `.js`), not `middleware.ts`. The runtime and API are unchanged.

Runs before a request is completed. Can rewrite, redirect, modify headers, or respond directly.

## When to use it

- Modify headers for all or some pages (CSP nonces, correlation IDs)
- Rewrite based on A/B test or feature flag
- Programmatic redirects based on request properties (locale, cookies, headers)
- Optimistic auth checks (permission-based redirects — *not* a substitute for in-action auth)

**Don't use it for:**
- Slow data fetching (it runs on the hot path)
- Full session management or authorization (verify inside Server Actions and Route Handlers; Proxy is optimistic only)

Simple static redirects? Use `redirects` in `next.config.ts`, not Proxy.

## Placement

One `proxy.ts` per project, at project root (or inside `src/` if you use it) — same level as `app/`.

You can split logic into multiple files and import them into `proxy.ts`, but only one file is recognized.

## Shape

```ts
// proxy.ts
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  return NextResponse.redirect(new URL('/home', request.url))
}

export const config = {
  matcher: '/about/:path*',
}
```

Default export works too:

```ts
export default function proxy(request: NextRequest) { /* ... */ }
```

## Matchers

Filters which paths run Proxy. A single string, an array of strings, or objects for more control:

```ts
export const config = {
  matcher: [
    {
      source: '/((?!api|_next/static|_next/image|favicon.ico).*)',
      missing: [
        { type: 'header', key: 'next-router-prefetch' },
        { type: 'header', key: 'purpose', value: 'prefetch' },
      ],
    },
  ],
}
```

This pattern: match everything except API/static, and skip prefetch requests. Common for CSP nonces (nonces must be fresh per request, but prefetches aren't real navigations).

## Runtime

Proxy uses the **Edge runtime** by default — a subset of Node.js APIs. Keep it lean. Node.js runtime is available experimentally.

If you need full Node APIs, move the logic into a layout (Server Component) or use headers/cookies matching in `rewrites`/`redirects` config. Custom server as last resort.

## What Proxy can return

- `NextResponse.next()` — continue with the request (optionally modify upstream headers)
- `NextResponse.rewrite(url)` — serve a different internal path transparently
- `NextResponse.redirect(url)` — 307 redirect
- `NextResponse.json(body, init)` / `new Response(...)` — respond directly

## Modifying upstream vs response headers

`NextResponse.next({ request: { headers } })` modifies headers your server receives. These are not exposed to the client.

```ts
const requestHeaders = new Headers(request.headers)
requestHeaders.set('x-nonce', nonce)
const response = NextResponse.next({ request: { headers: requestHeaders } })
response.headers.set('Content-Security-Policy', cspHeaderValue) // client-visible
return response
```

## CSP nonce pattern (full)

See [security-and-forms.md](security-and-forms.md) for the full CSP setup. Outline:

```ts
// proxy.ts
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
    style-src 'self' 'nonce-${nonce}';
    ...
  `.replace(/\s{2,}/g, ' ').trim()

  const requestHeaders = new Headers(request.headers)
  requestHeaders.set('x-nonce', nonce)
  requestHeaders.set('Content-Security-Policy', csp)

  const response = NextResponse.next({ request: { headers: requestHeaders } })
  response.headers.set('Content-Security-Policy', csp)
  return response
}

export const config = {
  matcher: [/* skip prefetches, static, api */]
}
```

Read the nonce in a Server Component via `headers().get('x-nonce')`.

Using nonces forces dynamic rendering — incompatible with PPR.

## Non-obvious

- `fetch` inside Proxy: `options.cache`, `options.next.revalidate`, `options.next.tags` have no effect.
- Third-party libraries may still ship a `middleware` factory — wire it into `proxy.ts`, the rename doesn't affect semantics.
- Negative matching (exclude paths) uses regex lookaheads: `/((?!api|_next).*)`.
