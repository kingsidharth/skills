# Caching — Cache Components

The current caching model. Enable with:

```ts
// next.config.ts
const config: NextConfig = { cacheComponents: true }
```

Under Cache Components, rendering is **partial prerendering (PPR) by default**: a static shell is served instantly, dynamic sections stream in.

## Three building blocks

1. **`'use cache'`** — marks a function or component's return value as cacheable. Arguments and closed-over values form the cache key.
2. **`cacheLife`** — sets `stale`/`revalidate`/`expire` for the cache entry.
3. **`cacheTag`** — attaches tags for on-demand invalidation (see [revalidation.md](revalidation.md)).

## `use cache`

**Data-level** (cache a function that returns data):

```ts
import { cacheLife } from 'next/cache'

export async function getUsers() {
  'use cache'
  cacheLife('hours')
  return db.query('SELECT * FROM users')
}
```

**UI-level** (cache a whole component):

```tsx
import { cacheLife } from 'next/cache'

export default async function Page() {
  'use cache'
  cacheLife('hours')
  const users = await db.query('SELECT * FROM users')
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

**File-level** — add `'use cache'` at the top of the file and all exported async functions are cached.

Cache key = function arguments + any closed-over values. Different inputs → separate entries. You must only pass serializable args.

## `cacheLife` profiles

| Profile | `stale` | `revalidate` | `expire` |
|---|---|---|---|
| `seconds` | 0 | 1s | 60s |
| `minutes` | 5m | 1m | 1h |
| `hours` | 5m | 1h | 1d |
| `days` | 5m | 1d | 1w |
| `weeks` | 5m | 1w | 30d |
| `max` | 5m | 30d | ~indefinite |

Custom: `cacheLife({ stale: 3600, revalidate: 7200, expire: 86400 })`.

Short-lived caches (`seconds`, `revalidate: 0`, or `expire < 5m`) are **excluded from prerenders** — they become dynamic holes. Prefer longer `expire` + tag invalidation for content you want prerendered.

## Streaming uncached data

Components that must fetch fresh data every request should be wrapped in `<Suspense>`:

```tsx
export default function Page() {
  return (
    <>
      <h1>Blog</h1>
      <Suspense fallback={<p>Loading posts...</p>}>
        <LatestPosts />
      </Suspense>
    </>
  )
}

async function LatestPosts() {
  const data = await fetch('https://api.example.com/posts')
  return <PostList posts={await data.json()} />
}
```

The fallback goes in the static shell; the content streams in at request time.

## Runtime APIs force dynamic

These opt a component into request-time rendering:
- `cookies()`, `headers()`, `draftMode()`
- `searchParams` prop on a page
- `params` on a dynamic segment without `generateStaticParams`
- `connection()` (explicit opt-in)

Wrap them in `<Suspense>` so the rest of the route can still prerender:

```tsx
import { cookies } from 'next/headers'

async function UserGreeting() {
  const theme = (await cookies()).get('theme')?.value ?? 'light'
  return <p>Theme: {theme}</p>
}

export default function Page() {
  return (
    <>
      <h1>Dashboard</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <UserGreeting />
      </Suspense>
    </>
  )
}
```

## Passing runtime values into cached functions

Extract the runtime value in an uncached component, pass it as an arg to a cached component — the arg becomes part of the cache key:

```tsx
async function ProfileContent() {
  const sessionId = (await cookies()).get('session')?.value
  return <CachedContent sessionId={sessionId!} />
}

async function CachedContent({ sessionId }: { sessionId: string }) {
  'use cache'
  const data = await fetchUserData(sessionId)
  return <div>{data.name}</div>
}
```

## Non-deterministic operations

`Math.random()`, `Date.now()`, `crypto.randomUUID()` must be handled explicitly. Two options:

**Per-request**: call `connection()` first and wrap in `<Suspense>`:

```tsx
import { connection } from 'next/server'
async function UniqueContent() {
  await connection()
  return <p>{crypto.randomUUID()}</p>
}
```

**Cached** (same value for everyone until revalidation):

```tsx
export default async function Page() {
  'use cache'
  return <p>Build ID: {crypto.randomUUID()}</p>
}
```

Without either, the build fails with an "Uncached data accessed" error.

## Deterministic synchronous work

`fs.readFileSync`, static module imports, pure computations are fine in a page — they complete during prerendering and become part of the static shell.

## Rendering model at a glance

At build time, Next renders the route's component tree:
- `use cache` → cached, in the static shell
- `<Suspense>` → fallback in the shell, content streams at request
- Deterministic sync → in the shell

Output: static shell (HTML + serialized RSC) served from CDN, dynamic holes streamed at request.

This is **Partial Prerendering (PPR)**. The build output marks these routes with a special indicator.

## Opting out of the static shell

Putting an empty-fallback `<Suspense>` above `<body>` in the root layout makes the whole app dynamic:

```tsx
<html>
  <Suspense fallback={null}>
    <body>{children}</body>
  </Suspense>
</html>
```

Rarely useful — prefer per-segment control via multiple root layouts.

## Route Handlers with Cache Components

`GET` route handlers follow the same model: they run at request time by default, but prerender if they don't access runtime data. Use `'use cache'` inside a helper (not directly in the `GET` body) to cache results:

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

## Metadata and viewport

`generateMetadata` and `generateViewport` track runtime data access separately from the page. If they access uncached data, you need to handle it explicitly. See "Metadata with Cache Components" in the official docs if you hit this.

## Putting it together

```tsx
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { cacheLife, cacheTag, updateTag } from 'next/cache'

export default function BlogPage() {
  return (
    <>
      <header><h1>Blog</h1></header>
      <BlogPosts />                              {/* cached, prerendered */}
      <Suspense fallback={<p>Loading prefs...</p>}>
        <UserPreferences />                       {/* streams per-user */}
      </Suspense>
    </>
  )
}

async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  cacheTag('posts')
  const posts = await fetch('https://api.example.com/blog').then(r => r.json())
  return <PostList posts={posts} />
}

async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value ?? 'light'
  return <aside>Theme: {theme}</aside>
}
```

## If Cache Components isn't on

You're using the "Previous Model": `fetch` with `{ cache: 'force-cache', next: { revalidate, tags } }`, `export const revalidate`, `unstable_cache`. Those APIs still work but are a different mental model — the Cache Components docs page flags this at the top.
