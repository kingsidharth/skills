# Data Mutation — Server Actions

Mutations in Next.js go through **Server Functions** marked with `'use server'`. When used in a form's `action` prop or a button's `formAction`, they're called **Server Actions**. Invoked via POST under the hood.

## Security first

Server Functions are **public endpoints**. The encrypted action ID is reachable via direct POST — not just through your UI. Every action must:
- Verify authentication (session/token)
- Verify authorization (this user may perform this action on *this* resource)
- Validate all input (don't trust `formData`, `searchParams`, headers)
- Return only what the UI needs (not raw DB records)

Page-level checks don't cover actions defined on that page — the action is a separate entry point.

## Defining actions

**File-level** — all exports are server functions:

```ts
// app/lib/actions.ts
'use server'

import { auth } from '@/lib/auth'

export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')
  const title = formData.get('title')
  // mutate, revalidate, maybe redirect
}
```

**Function-level** — inside a Server Component:

```tsx
export default function Page() {
  async function createPost(formData: FormData) {
    'use server'
    // ...
  }
  return <form action={createPost}>{/* ... */}</form>
}
```

**From Client Components** — import a file-level `'use server'` module:

```tsx
'use client'
import { createPost } from '@/app/actions'
export function Button() {
  return <button formAction={createPost}>Create</button>
}
```

**Passing to Client Components as props** — pass it like any prop:

```tsx
<ClientComponent updateAction={updateItem} />
```

## Forms

```tsx
import { createInvoice } from '@/app/actions'
export function InvoiceForm() {
  return (
    <form action={createInvoice}>
      <input name="customerId" />
      <input name="amount" />
      <button type="submit">Create</button>
    </form>
  )
}
```

Extract fields with `formData.get('name')` or `Object.fromEntries(formData)` (includes `$ACTION_` keys, so destructure known fields). For progressive enhancement, this works without JS — React queues submissions during hydration.

## Validation

Client-side: HTML attributes (`required`, `type="email"`).

Server-side: Zod (or similar) inside the action:

```ts
'use server'
import { z } from 'zod'

const schema = z.object({ email: z.email() })

export async function createUser(prev: State, formData: FormData) {
  const parsed = schema.safeParse({ email: formData.get('email') })
  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors }
  }
  // mutate
}
```

## `useActionState` — pending + returned state

When you need to show errors or a pending state, wrap with `useActionState`. The action signature gains a `prevState` first argument:

```tsx
'use client'
import { useActionState } from 'react'
import { createUser } from '@/app/actions'

export function Signup() {
  const [state, formAction, pending] = useActionState(createUser, { message: '' })
  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      {state?.message && <p aria-live="polite">{state.message}</p>}
      <button disabled={pending}>Sign up</button>
    </form>
  )
}
```

`useFormStatus` (from `react-dom`) is the alternative for pure submit-button pending state — only works **inside a form**, not on a sibling.

## Passing extra arguments (beyond form fields)

Use `Function.bind`:

```tsx
'use client'
import { updateUser } from './actions'
export function Profile({ userId }: { userId: string }) {
  const action = updateUser.bind(null, userId)
  return <form action={action}><input name="name" /><button>Save</button></form>
}
```

```ts
// actions.ts
'use server'
export async function updateUser(userId: string, formData: FormData) { /* ... */ }
```

`bind` preserves progressive enhancement; hidden inputs expose the value in the HTML.

## After the mutation — pick one

| Goal | Call |
|---|---|
| Refresh the current page, show latest data | `refresh()` from `next/cache` |
| Invalidate tagged caches, stale-while-revalidate (slight delay OK) | `revalidateTag('tag', 'max')` |
| Invalidate tagged caches, show fresh immediately (read-your-own-writes) | `updateTag('tag')` — Server Actions only |
| Invalidate all caches on a path | `revalidatePath('/posts')` |
| Navigate to a different page | `redirect('/posts')` from `next/navigation` |

`redirect` throws a control-flow exception — any code after it won't run. Call `revalidate*`/`updateTag` first, then `redirect`.

See [revalidation.md](revalidation.md) for the details on `revalidateTag` vs `updateTag`.

## Optimistic updates

`useOptimistic` applies a change locally before the server responds:

```tsx
'use client'
import { useOptimistic } from 'react'
import { send } from './actions'

export function Thread({ messages }: { messages: Message[] }) {
  const [optimistic, addOptimistic] = useOptimistic<Message[], string>(
    messages,
    (state, newMsg) => [...state, { message: newMsg }]
  )
  const action = async (formData: FormData) => {
    const msg = formData.get('message') as string
    addOptimistic(msg)
    await send(msg)
  }
  return (
    <div>
      {optimistic.map((m, i) => <div key={i}>{m.message}</div>)}
      <form action={action}>
        <input name="message" />
        <button>Send</button>
      </form>
    </div>
  )
}
```

## Cookies in actions

```ts
'use server'
import { cookies } from 'next/headers'
export async function setTheme(theme: string) {
  const store = await cookies()
  store.set('theme', theme)
}
```

Setting or deleting a cookie triggers a re-render of the current tree on the server. Client state is preserved; effects re-run only if dependencies changed.

## Event-handler and `useEffect` invocations

Server Actions aren't limited to forms. Call them directly from `onClick` or inside `useEffect`:

```tsx
'use client'
import { incrementViews } from './actions'
import { useEffect, useTransition } from 'react'

export function ViewTracker() {
  const [, startTransition] = useTransition()
  useEffect(() => {
    startTransition(async () => { await incrementViews() })
  }, [])
  return null
}
```

Actions are dispatched one-at-a-time per client — fine for mutations, wrong for parallel reads. For parallel *reads*, use a Route Handler or fetch in a Server Component.

## Allowed origins / CSRF

Next.js compares the `Origin` and `Host` headers on every action request. Behind a reverse proxy or on multi-origin setups, set `experimental.serverActions.allowedOrigins` in `next.config.ts`:

```ts
serverActions: { allowedOrigins: ['my-proxy.com', '*.my-proxy.com'] }
```

## Encryption keys in multi-instance deploys

Closed-over variables in inline actions are encrypted with a per-build key. Multi-instance deploys need a consistent key:

```bash
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=$(openssl rand -base64 32)
```

Set in the environment before `next build`. Otherwise you'll get "Failed to find Server Action" errors across instances.
