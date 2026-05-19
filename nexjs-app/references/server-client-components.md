# Server and Client Components

All App Router components are **Server Components by default**. Opt in to client with `'use client'` at the top of the file.

## Picking the right one

Use a **Client Component** when the component needs any of:
- `useState`, `useReducer`, `useEffect`, other hooks
- Event handlers (`onClick`, `onChange`, …)
- Browser APIs (`window`, `localStorage`, `navigator`, media APIs)
- React context consumers

Use a **Server Component** for everything else:
- Data fetching close to the source (DB, internal API, filesystem)
- Reading env vars and secrets
- Heavy dependencies you don't want in the client bundle
- Improving FCP — stream HTML without waiting for JS

Default to Server. Push `'use client'` down as far in the tree as possible.

## The `'use client'` boundary

`'use client'` marks a module as client. **Every module it imports is also client.** This means you don't repeat the directive on every file; only on the boundary between server and client graphs.

```tsx
// app/ui/like-button.tsx
'use client'
import { useState } from 'react'
export default function LikeButton({ initial }: { initial: number }) {
  const [likes, setLikes] = useState(initial)
  // ...
}
```

Then use it from a Server Component:

```tsx
// app/[id]/page.tsx
import LikeButton from '@/app/ui/like-button'
export default async function Page({ params }: PageProps<'/[id]'>) {
  const { id } = await params
  const post = await getPost(id)
  return <LikeButton initial={post.likes} />
}
```

## Rendering model (1-minute version)

- Server: Server Components render to an **RSC Payload** (binary format with placeholders for Client Components). Client Components + RSC Payload → prerendered HTML.
- Client (first load): HTML shows immediately; RSC Payload reconciles the tree; JS hydrates Client Components.
- Subsequent nav: only RSC Payload is fetched and cached; no server-rendered HTML needed.

## Passing data

Props from Server to Client must be **serializable** (no functions, classes, Symbols). Functions/classes are silently blocked.

To share the same data between Server and Client components in one request, wrap the fetcher in `React.cache` and use a context provider:

```tsx
// app/lib/user.ts
import { cache } from 'react'
export const getUser = cache(async () => {
  const res = await fetch('https://api.example.com/user')
  return res.json()
})
```

```tsx
// app/user-provider.tsx
'use client'
import { createContext } from 'react'
export const UserContext = createContext<Promise<User> | null>(null)
export default function UserProvider({ children, userPromise }: {
  children: React.ReactNode
  userPromise: Promise<User>
}) {
  return <UserContext value={userPromise}>{children}</UserContext>
}
```

```tsx
// app/layout.tsx
export default function RootLayout({ children }: LayoutProps<'/'>) {
  const userPromise = getUser()  // don't await
  return (
    <html><body>
      <UserProvider userPromise={userPromise}>{children}</UserProvider>
    </body></html>
  )
}
```

Client component then reads with `use(userPromise)` inside a `<Suspense>`. Server Components can also `await getUser()` — the `React.cache` memo deduplicates within a single request.

## Interleaving — Server inside Client

Server Components can be passed **as `children`** to Client Components. They still render on the server; the Client Component just holds the rendered output in a slot:

```tsx
// ui/modal.tsx (client)
'use client'
export default function Modal({ children }: { children: React.ReactNode }) {
  return <div className="modal">{children}</div>
}
```

```tsx
// page.tsx (server)
import Modal from './ui/modal'
import Cart from './ui/cart' // server component
export default function Page() {
  return <Modal><Cart /></Modal>
}
```

You cannot `import` a Server Component directly inside a Client Component — use the `children` slot pattern.

## Context providers

Context is client-only. Put providers in a Client Component, place them **as deep as possible** (don't wrap the entire `<html>` — it defeats static optimization). Wrap `{children}` inside the provider.

## Third-party components without `'use client'`

If a library uses hooks but doesn't ship `'use client'`, wrap it once:

```tsx
// app/carousel.tsx
'use client'
export { Carousel as default } from 'acme-carousel'
```

Then import the wrapper inside Server Components.

## Preventing environment poisoning

Secrets leak when a module that reads `process.env.SECRET` is accidentally imported into the client graph. Next.js replaces non-`NEXT_PUBLIC_` env vars with empty strings client-side, but the *code* can still end up in the client bundle.

Add `import 'server-only'` at the top of server-only modules:

```ts
// lib/data.ts
import 'server-only'
export async function getData() {
  return fetch('https://api.example.com', {
    headers: { authorization: process.env.API_KEY! },
  }).then((r) => r.json())
}
```

If this module is imported from a Client Component, the build fails. The `client-only` package does the inverse.

Installing `server-only`/`client-only` is optional — Next.js provides its own types; the build-time check works either way.
