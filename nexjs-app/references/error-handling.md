# Error Handling

Two categories: **expected errors** (validation, failed requests) are returned as values. **Uncaught exceptions** bubble to error boundaries.

## Expected errors — return, don't throw

### In Server Actions

Use `useActionState`; return the error as part of state:

```ts
// app/actions.ts
'use server'
export async function createPost(prev: any, formData: FormData) {
  const res = await fetch('https://api.example.com/posts', {
    method: 'POST',
    body: JSON.stringify({ title: formData.get('title') }),
  })
  if (!res.ok) return { message: 'Failed to create post' }
}
```

```tsx
'use client'
import { useActionState } from 'react'
import { createPost } from '@/app/actions'

export function Form() {
  const [state, action, pending] = useActionState(createPost, { message: '' })
  return (
    <form action={action}>
      <input name="title" required />
      {state?.message && <p aria-live="polite">{state.message}</p>}
      <button disabled={pending}>Create</button>
    </form>
  )
}
```

### In Server Components

Check the response, render an error state or redirect:

```tsx
export default async function Page() {
  const res = await fetch('https://...')
  if (!res.ok) return <p>There was an error.</p>
  const data = await res.json()
  return /* ... */
}
```

### Not found

`notFound()` triggers the nearest `not-found.tsx`:

```tsx
import { notFound } from 'next/navigation'
export default async function Page({ params }: PageProps<'/blog/[slug]'>) {
  const { slug } = await params
  const post = await getPostBySlug(slug)
  if (!post) notFound()
  return <article>{post.title}</article>
}
```

```tsx
// app/blog/[slug]/not-found.tsx
export default function NotFound() {
  return <div>Post not found</div>
}
```

## Uncaught exceptions — error boundaries

### `error.tsx` (segment-level)

Catches errors in its segment and subtree. Must be a Client Component. Receives `error` and **`unstable_retry`** (replaces `reset`):

```tsx
// app/dashboard/error.tsx
'use client'
import { useEffect } from 'react'

export default function ErrorPage({
  error,
  unstable_retry,
}: {
  error: Error & { digest?: string }
  unstable_retry: () => void
}) {
  useEffect(() => { console.error(error) }, [error])
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => unstable_retry()}>Try again</button>
    </div>
  )
}
```

Errors bubble up to the nearest parent `error.tsx`. Add them at different depths for granular fallbacks.

### `global-error.tsx`

Root-level boundary. Replaces the root layout when active, so it must include `<html>`/`<body>`:

```tsx
// app/global-error.tsx
'use client'
export default function GlobalError({ error, unstable_retry }: {
  error: Error & { digest?: string }
  unstable_retry: () => void
}) {
  return (
    <html>
      <body>
        <h2>Something went wrong!</h2>
        <button onClick={() => unstable_retry()}>Try again</button>
      </body>
    </html>
  )
}
```

### `unstable_catchError` — wrap anywhere

For component-level boundaries outside the route file structure. Takes a fallback renderer, returns a wrapper component:

```tsx
// app/custom-error-boundary.tsx
'use client'
import { unstable_catchError as catchError, type ErrorInfo } from 'next/error'

function Fallback(props: { title: string }, { error, unstable_retry }: ErrorInfo) {
  return (
    <div>
      <h2>{props.title}</h2>
      <p>{error.message}</p>
      <button onClick={() => unstable_retry()}>Try again</button>
    </div>
  )
}

export default catchError(Fallback)
```

Usage:

```tsx
import ErrorBoundary from './custom-error-boundary'
export function Widget({ children }: { children: React.ReactNode }) {
  return <ErrorBoundary title="Widget error">{children}</ErrorBoundary>
}
```

## What error boundaries do and don't catch

Error boundaries catch **rendering errors**. They do NOT catch:
- Errors inside event handlers (`onClick`) — catch manually with `try/catch` + `useState`
- Errors inside `useEffect`
- Errors inside async callbacks that aren't awaited during render

Exception: unhandled errors inside `startTransition` from `useTransition` **do** bubble to error boundaries.

Handle event-handler errors manually:

```tsx
'use client'
import { useState } from 'react'
export function Button() {
  const [error, setError] = useState<Error | null>(null)
  const handleClick = () => {
    try { doStuff() } catch (e) { setError(e as Error) }
  }
  if (error) return <p>{error.message}</p>
  return <button onClick={handleClick}>Click</button>
}
```
