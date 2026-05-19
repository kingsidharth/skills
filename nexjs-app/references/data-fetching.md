# Data Fetching

Default to fetching in Server Components. Add `<Suspense>` around anything that blocks the page; add `'use cache'` when the result can be reused.

## Server Component fetching

Direct `fetch` or direct ORM/DB calls — both work. Server Components are async functions:

```tsx
// app/blog/page.tsx
export default async function Page() {
  const res = await fetch('https://api.vercel.app/blog')
  const posts = await res.json()
  return <PostList posts={posts} />
}
```

```tsx
// ORM/DB direct
import { db, posts } from '@/lib/db'
export default async function Page() {
  const allPosts = await db.select().from(posts)
  return <PostList posts={allPosts} />
}
```

Key facts:
- Identical `fetch` calls in a single request tree are **deduplicated** automatically (React memoization).
- `fetch` is **not cached by default** in v16. Uncached data blocks rendering until resolved.
- To cache a fetch result, wrap the fetching function in `use cache` (see [caching.md](caching.md)).
- Don't call your own Route Handlers from Server Components. Extra HTTP hop, slower, and fails at build time for prerendered routes.

## Streaming

When uncached data blocks the route, either:
1. Wrap the page with `loading.tsx` to stream the whole route, or
2. Wrap specific components with `<Suspense>` for granular streaming.

**`loading.tsx`**: boundary wraps `page.tsx` automatically. Shows while the page renders.

```tsx
// app/blog/loading.tsx
export default function Loading() { return <PostListSkeleton /> }
```

**`<Suspense>`**: granular. Shows the outer page immediately; only the boundaried component streams.

```tsx
import { Suspense } from 'react'
export default function BlogPage() {
  return (
    <>
      <header><h1>Blog</h1></header>
      <Suspense fallback={<PostListSkeleton />}>
        <PostList />
      </Suspense>
    </>
  )
}

async function PostList() {
  const posts = await fetch('...').then((r) => r.json())
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

Important nuance: a `loading.tsx` covers the `page`, not uncached access in a `layout`. If a layout at the same segment awaits `cookies()` or similar, navigation will block until the layout finishes. Put such access behind its own `<Suspense>`.

Cache Components turns this into a build-time error (`Uncached data accessed outside of Suspense`), which keeps you honest.

## Client Component fetching

Two options:

**1. `use()` API — stream server-fetched data into a Client Component.** Don't await in the Server Component; pass the promise:

```tsx
// server component
import Posts from './posts'
import { Suspense } from 'react'
export default function Page() {
  const posts = getPosts() // no await
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Posts posts={posts} />
    </Suspense>
  )
}
```

```tsx
// app/ui/posts.tsx (client)
'use client'
import { use } from 'react'
export default function Posts({ posts }: { posts: Promise<Post[]> }) {
  const allPosts = use(posts)
  return <ul>{allPosts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

**2. SWR / React Query** — when you need client-side features: polling, retries, cache invalidation based on focus, optimistic updates outside a form. Most apps don't need this if they stick to Server Components + Server Actions.

## Sequential vs parallel

Sequential (bad — waterfall):

```tsx
const artist = await getArtist(username)
const albums = await getAlbums(username) // waits for artist even though independent
```

Parallel with `Promise.all`:

```tsx
const [artist, albums] = await Promise.all([
  getArtist(username),
  getAlbums(username),
])
```

Parallel + progressive reveal: start the fetch, wrap each consumer in `<Suspense>`, let each resolve independently.

`Promise.all` rejects on first failure. Use `Promise.allSettled` when partial success is OK.

## When requests *are* genuinely sequential

If `B` needs a value from `A`, you can still stream `A` immediately and suspend `B`:

```tsx
export default async function Page({ params }: PageProps<'/artist/[username]'>) {
  const { username } = await params
  const artist = await getArtist(username)
  return (
    <>
      <h1>{artist.name}</h1>
      <Suspense fallback={<div>Loading playlists...</div>}>
        <Playlists artistId={artist.id} />
      </Suspense>
    </>
  )
}
```

To avoid blocking the outer layout on the artist fetch, put a `loading.tsx` at the segment or wrap the whole block in a parent `<Suspense>`.

## Logging fetches

In dev, enable `logging.fetches` in `next.config.ts` to see every fetch's cache status.
