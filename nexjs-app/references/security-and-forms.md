# Security, Forms, Env Vars, CSP

Everything that keeps user data safe and input validated. Cross-cuts forms, env vars, data security, and CSP.

## Env vars

### Files

- `.env` — all environments
- `.env.local` — gitignored, overrides per-environment; not loaded when `NODE_ENV=test`
- `.env.development` / `.env.production` / `.env.test`
- `.env.development.local` etc.

Load order (first wins):
1. `process.env`
2. `.env.$(NODE_ENV).local`
3. `.env.local` (skipped in test)
4. `.env.$(NODE_ENV)`
5. `.env`

Using `/src`? Keep `.env*` at project root.

### Server-only vs client

- Server-only by default.
- Prefix with `NEXT_PUBLIC_` to expose to the client (inlined at build time, **frozen**).

```txt
# .env
DB_URL=postgres://...                # server only
NEXT_PUBLIC_ANALYTICS_ID=abc123       # client-visible, build-time inlined
```

Non-inlinable lookups stay unresolved:

```ts
// WILL NOT INLINE
const varName = 'NEXT_PUBLIC_ANALYTICS_ID'
setupAnalytics(process.env[varName])
```

### Runtime vs build-time

`NEXT_PUBLIC_*` is build-time. To vary by runtime env (e.g., single Docker image → multiple environments):

```tsx
import { connection } from 'next/server'

export default async function Component() {
  await connection()               // opts into dynamic rendering
  const value = process.env.MY_VALUE
  return /* ... */
}
```

Accessing `cookies()`/`headers()` also opts into dynamic, and they see runtime values.

### Variable expansion

```txt
USER=nextjs
URL=https://x.com/$USER   # → "https://x.com/nextjs"
LITERAL=\$50              # escape for literal $
```

### Multi-line

Quoted or `\n`-escaped:

```txt
KEY="-----BEGIN KEY-----
MIIBI...
-----END KEY-----"
```

### Loading outside Next (ORM config, test runners)

Use `@next/env`:

```ts
import { loadEnvConfig } from '@next/env'
loadEnvConfig(process.cwd())
```

## Data Access Layer (recommended for new projects)

Centralize all DB access behind a `server-only` module. Pattern:

```ts
// data/auth.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'

export const getCurrentUser = cache(async () => {
  const token = (await cookies()).get('AUTH_TOKEN')
  const decoded = await decryptAndValidate(token)
  return new User(decoded.id)  // class, not a plain object — blocks accidental serialization
})
```

```ts
// data/user-dto.ts
import 'server-only'
import { getCurrentUser } from './auth'

export async function getProfileDTO(slug: string) {
  const [row] = await sql`SELECT * FROM user WHERE slug = ${slug}`
  const viewer = await getCurrentUser()
  return {
    username: canSeeUsername(viewer) ? row.username : null,
    phone: canSeePhone(viewer, row.team) ? row.phone : null,
  }
}
```

Why:
- Only the DAL reads `process.env` secrets
- Authorization happens in one place
- DTOs return only what the UI needs — no accidental private field leaks
- Classes can't be serialized to the client → extra safety

## Tainting

`experimental.taint: true` enables React's Taint APIs. Marks objects/values as "must not reach client":

```ts
import { experimental_taintObjectReference, experimental_taintUniqueValue } from 'react'
```

Extra safety layer. Don't rely on it instead of filtering in the DAL — use both.

## Server Actions — security checklist

Every Server Action is a public POST endpoint. Each one must:

1. **Authenticate** — `const session = await auth()` and fail if missing
2. **Authorize the resource** — check ownership (prevents IDOR)
3. **Validate input** — never trust `formData`, `searchParams`, or headers
4. **Return minimal data** — not raw DB records

```ts
'use server'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')

  const post = await db.post.findUnique({ where: { id: postId } })
  if (post.authorId !== session.user.id) throw new Error('Forbidden')

  await db.post.delete({ where: { id: postId } })
}
```

Page-level `redirect('/login')` does not cover actions defined on that page. Re-check inside the action.

### Built-in protections

- Secure encrypted action IDs (rotated across builds, max 14 days cached)
- Unused action definitions dead-code-eliminated — no public endpoint
- `Origin` vs `Host` header check (same-origin enforced)
- `POST`-only — avoids many CSRF vectors + GET-side-effect bugs

### Closure encryption

Variables captured in an inline action are encrypted before being sent to the client. Still: **never capture real secrets** in a closure. Derive them in the action body.

### Multi-instance key

```bash
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=$(openssl rand -base64 32)
```

Otherwise, inline actions fail across instances with "Failed to find Server Action".

### Allowed origins (reverse proxy)

```ts
serverActions: { allowedOrigins: ['my-proxy.com', '*.my-proxy.com'] }
```

## Forms

### Basic

```tsx
import { createInvoice } from '@/app/actions'
export function InvoiceForm() {
  return (
    <form action={createInvoice}>
      <input name="customerId" />
      <input name="amount" type="number" />
      <button>Create</button>
    </form>
  )
}
```

`formData.get('name')` to extract. `Object.fromEntries(formData)` for bulk (includes `$ACTION_` prefix).

### Validation + errors + pending

See [data-mutation.md](data-mutation.md) for the `useActionState` + Zod pattern.

### Passing extra args

`updateUser.bind(null, userId)` preserves progressive enhancement.

### Optimistic updates

`useOptimistic` — see [data-mutation.md](data-mutation.md).

### Programmatic submission

```tsx
const handleKey = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
  if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') {
    e.currentTarget.form?.requestSubmit()
  }
}
```

### Multiple actions per form

Use `formAction` on individual buttons (e.g. "Save draft" vs "Publish").

## Content Security Policy

### With nonces (strict)

1. Generate a nonce per request in `proxy.ts` (see [proxy.md](proxy.md) for the full pattern)
2. Add to both the request headers (so Next can inject) and the response
3. Read the nonce in Server Components via `headers().get('x-nonce')` for third-party scripts

**Nonces force dynamic rendering on every page.** Incompatible with PPR and CDN caching.

### Without nonces (via `headers`)

Static-friendly. In `next.config.js`:

```js
const cspHeader = `
  default-src 'self';
  script-src 'self' 'unsafe-inline'${isDev ? " 'unsafe-eval'" : ''};
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data:;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
`

module.exports = {
  async headers() {
    return [{
      source: '/(.*)',
      headers: [{ key: 'Content-Security-Policy', value: cspHeader.replace(/\n/g, '') }],
    }]
  },
}
```

### SRI (experimental)

Hash-based CSP that works with static generation:

```ts
experimental: { sri: { algorithm: 'sha256' } }
```

Adds `integrity` attributes to script tags. Compatible with PPR.

### Development caveat

`'unsafe-eval'` is required in dev — React uses `eval` for better error stacks. Production doesn't need it.

## Avoiding side-effects during rendering

```tsx
// BAD — mutation during render
export default async function Page({ searchParams }: PageProps<'/'>) {
  const { logout } = await searchParams
  if (logout) (await cookies()).delete('AUTH_TOKEN')  // error
  // ...
}
```

Cookies can't be set/deleted during render. Use a Server Action triggered by a form.

## Audit checklist (for security reviews)

- **DAL**: is there one? Is `process.env` / DB client used anywhere else?
- `"use client"` files: are props overly broad? Any private fields accidentally passed?
- `"use server"` files: auth + authz + input validation + narrow returns + resource-ownership check?
- `/[param]/` folders: are params validated?
- `proxy.ts`, `route.ts`: extra scrutiny — they have the most power
