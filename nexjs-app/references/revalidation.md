# Revalidation

Four APIs, each for a different situation.

## Decision table

| Situation | API | Where |
|---|---|---|
| User just made a change, should see it immediately | `updateTag('tag')` | Server Actions only |
| Background refresh is fine (minutes of staleness OK) | `revalidateTag('tag', 'max')` | Server Actions + Route Handlers |
| Don't know which tags, just invalidate a path | `revalidatePath('/posts')` | Server Actions + Route Handlers |
| Just refresh the router (no cache change) | `refresh()` from `next/cache` | Server Actions |
| Set an expiry on the cached data itself | `cacheLife(profile)` | Inside `use cache` scope |

## Tag-first

Tag invalidation is more precise than path invalidation. Tag your cached functions once, invalidate them from anywhere:

```ts
import { cacheTag } from 'next/cache'
export async function getProducts() {
  'use cache'
  cacheTag('products')
  return db.query('SELECT * FROM products')
}
```

Multiple functions can share a tag and be invalidated together.

## `updateTag` — read-your-own-writes

Immediate expiry. The user who just submitted sees the change right away. Server Actions only:

```ts
// app/lib/actions.ts
'use server'
import { updateTag } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  const post = await db.post.create({
    data: { title: formData.get('title') as string },
  })
  updateTag('posts')
  redirect(`/posts/${post.id}`)
}
```

## `revalidateTag` — stale-while-revalidate

Serves stale immediately; generates fresh in the background. Better for public content where a small delay is acceptable. Works in actions **and** route handlers (webhooks):

```ts
import { revalidateTag } from 'next/cache'
export async function updateUser(id: string) {
  await db.user.update({ where: { id }, data: { /* ... */ } })
  revalidateTag('user', 'max') // 'max' = longest stale window
}
```

Second arg controls how long stale content can be served while fresh regenerates. `'max'` gives the biggest window; after it, requests block until fresh is ready.

## `revalidatePath`

Coarser — invalidates everything tied to a route path. Prefer tags when you can:

```ts
import { revalidatePath } from 'next/cache'
revalidatePath('/profile')
```

## `refresh` — router refresh without cache invalidation

Refreshes the current page in the client router. Useful when the mutation already updated underlying data but you need the UI to re-fetch RSC:

```ts
'use server'
import { refresh } from 'next/cache'
export async function updatePost(formData: FormData) {
  // mutate
  refresh()
}
```

Does not revalidate tags. Use in combination with `updateTag`/`revalidateTag` when cached content changed.

## Typical mutation flow

```ts
'use server'
import { auth } from '@/lib/auth'
import { updateTag } from 'next/cache'
import { redirect } from 'next/navigation'

export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')

  const post = await db.post.findUnique({ where: { id: postId } })
  if (post.authorId !== session.user.id) throw new Error('Forbidden')

  await db.post.delete({ where: { id: postId } })
  updateTag('posts')
  redirect('/posts')
}
```

Order: mutate → invalidate → redirect. `redirect` throws, so nothing after it runs.

## Webhooks

Route handler that revalidates on CMS content change:

```ts
// app/webhook/route.ts
import { revalidateTag } from 'next/cache'
import { type NextRequest, NextResponse } from 'next/server'

export async function POST(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('token')
  if (token !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ success: false }, { status: 401 })
  }
  const tag = request.nextUrl.searchParams.get('tag')
  if (!tag) return NextResponse.json({ success: false }, { status: 400 })
  revalidateTag(tag)
  return NextResponse.json({ success: true })
}
```

## Multi-instance invalidation

Calling `revalidateTag()` invalidates only the instance it ran on. Others keep serving stale until they learn about the invalidation. Solution: implement `refreshTags()` in a custom cache handler (Redis/DB-backed) — it's called before each request and syncs tag state from shared storage. See self-hosting docs for the full contract.
