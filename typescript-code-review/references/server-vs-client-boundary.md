# Server vs Client component boundary

**When this applies:** any project using React Server Components — Next.js App Router, TanStack Start (RSC support landing), Waku, etc. Determines what runs where, and where to draw the `'use client'` line.

The single most common mistake: putting `'use client'` too high in the tree, dragging the entire subtree onto the client and defeating the point of RSC.

---

## The mental model

There are **two execution environments**:

- **Server Components (default in App Router)** — run on the server, return JSX. Have access to the file system, databases, secrets, and any Node API. **Never run in the browser.** Can't use hooks, state, refs, or browser-only APIs.
- **Client Components (`'use client'`)** — run on the server during initial render (for HTML), then hydrate and run on the client. Can use hooks, state, refs, event handlers. Cannot use Node APIs, secrets, file system.

The boundary between them is the `'use client'` directive. A file with `'use client'` and everything it imports become client code. Everything above it in the tree (its parents) stays server.

---

## Rule 1: Default to Server Components

Every component starts as a Server Component. Add `'use client'` *only* when one of these is true:

- The component uses `useState`, `useReducer`, `useEffect`, `useRef`, `useContext`, or any other React hook.
- The component handles a DOM event (`onClick`, `onChange`, `onSubmit` — though `<form action>` works in server).
- The component uses a browser-only API (`window`, `document`, `localStorage`, `IntersectionObserver`).
- The component uses a third-party library that depends on hooks or browser APIs (most styling libraries, animation libraries, almost all UI kits).

If none of these apply, leave it as a Server Component.

---

## Rule 2: `'use client'` goes at the **leaf**, not the layout

```tsx
// 🚨 every child of this layout is now a client component
'use client'
export default function DashboardLayout({ children }) {
  return (
    <div>
      <Sidebar />
      <main>{children}</main>
    </div>
  )
}
```

If `<Sidebar>` only needs interactivity for one button, push the directive to that button:

```tsx
// ✅ layout stays server; only the interactive island is client
export default function DashboardLayout({ children }) {
  return (
    <div>
      <Sidebar />     {/* server, with one client island inside */}
      <main>{children}</main>
    </div>
  )
}

// sidebar.tsx — server component with one client child
export function Sidebar() {
  return (
    <nav>
      <Logo />          {/* server */}
      <NavLinks />      {/* server */}
      <UserMenu />      {/* client (uses dropdown state) */}
    </nav>
  )
}

// user-menu.tsx
'use client'
export function UserMenu() { /* uses useState for open/closed */ }
```

The layout, logo, and nav links never ship to the client. Only the dropdown JS does.

---

## Rule 3: Server components can render client components, not vice versa

A server component can `import` and render a client component:

```tsx
// page.tsx (server)
import { LikeButton } from './like-button'  // client

export default async function Post({ id }: Props) {
  const post = await db.post.findUnique({ where: { id } })
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <LikeButton postId={post.id} />
    </article>
  )
}
```

A client component **cannot import a server component**, but it can receive one as `children`:

```tsx
// client-modal.tsx
'use client'
export function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)
  return open ? <div className="modal">{children}</div> : null
}

// page.tsx (server)
export default function Page() {
  return (
    <Modal>
      <ServerContent />  {/* server component as children — works */}
    </Modal>
  )
}
```

This pattern — **server component slotted into a client component via `children`** — is the most powerful composition tool RSC gives you. Use it to keep client islands minimal while still letting them wrap server content.

---

## Rule 4: Props serialized across the boundary must be minimal

When a server component passes props to a client component, the props are **serialized** into the response. They become part of the page weight.

```tsx
// 🚨 passes the entire user record (including secrets, internal IDs, audit logs)
const user = await db.user.findUnique({ where: { id }, include: { everything: true } })
return <UserCard user={user} />

// ✅ project to what the client actually needs
const user = await db.user.findUnique({ where: { id } })
return <UserCard user={{ name: user.name, avatar: user.avatar }} />
```

Failure modes if you don't:

- **Bundle bloat** — every byte of the prop ships to every visitor.
- **Security leak** — internal fields, secrets, or other users' data shipped to the client.
- **Serialization errors** — Date objects, Maps, class instances don't serialize cleanly.

What can cross the boundary: primitives, arrays, plain objects, `Date` (auto-converted), `Promise` (for streaming), `ReactNode`. What can't: functions (other than server actions), class instances, Maps/Sets (until React supports them natively).

---

## Rule 5: Suspense for streaming

Server Components support streaming via `<Suspense>`. The page returns a shell instantly and individual sections fill in as their data resolves.

```tsx
export default function Page() {
  return (
    <>
      <Header />                          {/* fast, renders immediately */}
      <Suspense fallback={<Skeleton />}>
        <SlowDataSection />               {/* awaits a slow query, streams in */}
      </Suspense>
      <Footer />                          {/* fast */}
    </>
  )
}

async function SlowDataSection() {
  const data = await fetchSlowData()       // server-side await
  return <List items={data} />
}
```

Place `<Suspense>` boundaries at points where independent slow work happens. **Don't wrap the whole page in one boundary** — that defeats streaming. **Don't omit them** — that blocks the page on the slowest dependency.

---

## Rule 6: Don't double-fetch

```tsx
// 🚨 server fetches, then client fetches the same data again
async function Page() {
  const data = await fetchInvoices()  // server fetch
  return <InvoiceList />              // client component that also useQuery's invoices
}
```

Either:

- Fetch on the server and pass `data` as a prop to a client component.
- Fetch on the client (TanStack Query / SWR) and don't fetch on the server.
- Use TanStack Query SSR / hydration to seed the client cache from the server fetch.

Pick one path per data dependency.

---

## Rule 7: No shared module-level mutable state

Server Components run **once per request** in a long-lived process. Module-level state is shared across requests:

```tsx
// 🚨 catastrophic — every request reads/writes the same `cart`
let cart: Item[] = []

export async function addToCart(item: Item) {
  'use server'
  cart.push(item)  // 💥 affects every user
}
```

State that varies per request belongs in the request scope: cookies, session store, DB, headers. Never module scope.

For per-request memoization across server-rendered components in the same request, use `React.cache`:

```ts
import { cache } from 'react'
export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } })
})
```

`React.cache` deduplicates within one request; multiple components calling `getUser('x')` share one DB call.

---

## Rule 8: Server Actions for mutations

Forms and mutations cross the boundary via Server Actions:

```tsx
// invoice-form.tsx (can be server or client)
import { createInvoice } from './actions'

export function InvoiceForm() {
  return (
    <form action={createInvoice}>
      <input name="amount" />
      <button type="submit">Create</button>
    </form>
  )
}

// actions.ts
'use server'
import { revalidatePath } from 'next/cache'

export async function createInvoice(formData: FormData) {
  const amount = Number(formData.get('amount'))
  await db.invoice.create({ data: { amount } })
  revalidatePath('/invoices')
}
```

For client-side optimistic UI, combine with `useFormStatus`, `useActionState`, and `useOptimistic` (React 19).

**Authenticate server actions like API routes.** A server action is a public POST endpoint — auth checks must be inside the action, not the page that renders the form.

---

## Rule 9: Hydration discipline

Hydration is the moment React attaches event handlers to the server-rendered HTML. Common errors:

- **Hydration mismatch** — server and client render different HTML. Causes: random IDs, dates without consistent timezone, conditional rendering based on `typeof window`.
- **Fix** — use `useId` for stable IDs; use a stable timezone or `<time>` with `suppressHydrationWarning`; gate browser-only logic behind `useEffect` or `'use client'` + `mounted` flag.

```tsx
// 🚨 generates different HTML on server vs client
function Foo() {
  const id = `id-${Math.random()}`
  return <div id={id}>...</div>
}

// ✅
function Foo() {
  const id = useId()
  return <div id={id}>...</div>
}
```

For genuinely client-only content, `useEffect` to set a "mounted" flag, or use `next/dynamic` with `ssr: false`.

---

## Rule 10: A reasonable default split

For most apps:

- **Layouts**, **pages**, **data-displaying components** → Server Components.
- **Forms** (using `<form action>`) → can be Server.
- **Form fields with live validation, dropdowns, modals, popovers, animations** → Client.
- **Buttons that just submit a form** → Server (the form is the action).
- **Buttons with onClick or stateful UI** → Client.
- **Search bars, filters, sortable tables** → Client (they need state and event handlers).

When in doubt: try Server first. Move to Client only when something concrete fails.

---

## References

- React docs — *Server Components*, *use client*, *use server*
- Next.js docs — App Router, Server Actions, caching
- Dan Abramov 2025 RSC series (overreacted.io) — *Two worlds, two doors*, *What does `use client` do?*, *JSX Over The Wire*
- Vercel `agent-skills/react-best-practices` — `server-*` rules: serialization, parallel fetching, no shared module state
