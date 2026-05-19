# Data flow and state shape

**When this applies:** designing a new feature, page, or component that has more than trivial state. Run before writing the first `useState`.

The single highest-leverage decision in a React feature is *where each piece of state lives*. Get this wrong and you'll fight bugs forever; get it right and the code mostly writes itself.

---

## Step 1: Categorise every piece of state

For each piece of data the feature touches, put it in exactly one bucket:

| Bucket | Definition | Where it lives |
|---|---|---|
| **Server state** | Owned by the server. We fetch, cache, sync. | TanStack Query / RSC loader |
| **URL state** | Affects what's shown on the page; should be shareable, bookmarkable, back-button-able. | Search params (router) |
| **Client UI state** | Local, ephemeral. Open/closed, hover, draft input, focused tab. | `useState` / `useReducer` |
| **Shared client state** | Used by siblings far apart in the tree. | Zustand (or context for static values only) |
| **Derived state** | Can be computed from any of the above. | Computed during render — not stored |

A piece of data lives in **one** bucket. If you find the same value in two buckets ("we keep the user list in Query *and* in Zustand"), one of them is wrong.

---

## Step 2: The default order of operations

When in doubt, prefer earlier buckets:

1. **Derive** — can this be computed from existing state?
2. **URL state** — should this survive a page refresh / be shareable?
3. **Server state** — does the server own this?
4. **Client UI state** — is it local to one component subtree?
5. **Shared client state** — is it genuinely shared across the app?

Most React projects have too much in buckets 4 and 5, and not enough in 1 and 2.

---

## Rule: Derive, don't sync

If a value can be computed, do not store it.

```ts
// 🚨 storing what could be derived
const [count, setCount] = useState(0)
const [isOdd, setIsOdd] = useState(false)
useEffect(() => { setIsOdd(count % 2 === 1) }, [count])

// ✅ derive
const [count, setCount] = useState(0)
const isOdd = count % 2 === 1
```

Same applies to derived server state:

```ts
// 🚨 storing whether selection is still valid
const { data: users } = useQuery(usersQuery)
const [selectedId, setSelectedId] = useState<string | null>(null)
useEffect(() => {
  if (selectedId && !users?.some(u => u.id === selectedId)) {
    setSelectedId(null)
  }
}, [users, selectedId])

// ✅ derive what's effectively selected
const { data: users } = useQuery(usersQuery)
const [selectedId, setSelectedId] = useState<string | null>(null)
const effectiveSelection = users?.some(u => u.id === selectedId) ? selectedId : null
```

The derived version automatically restores selection if a deleted user comes back, and lets you separately track "was this ever selected" from "is the selection currently valid". See TkDodo *Deriving Client State from Server State*.

---

## Rule: URL state for anything shareable

Filters, sort, pagination, search query, selected tab, currently-open modal — all should be in the URL.

Symptoms that something should be in the URL:

- Refreshing the page loses the user's place.
- The user can't share a link to "what they're looking at".
- The browser back button does the wrong thing.

For Next.js: `useSearchParams` + `router.push`. For TanStack Router: validated search-params with Zod. For React Router: `useSearchParams`. For SPA without a router with this baked in: `nuqs`.

```ts
// ✅ URL-driven filter
import { useQueryState } from 'nuqs'
const [filter, setFilter] = useQueryState('filter')

// in the same hook tree
const { data } = useQuery({
  queryKey: ['items', filter],
  queryFn: () => fetchItems(filter),
})
```

This is correct: changing `filter` updates the URL, which changes the query key, which triggers a refetch. One source of truth.

---

## Rule: Don't put server state in Zustand or `useState`

A `useEffect` that fetches and `setState`s is a re-implementation of TanStack Query, badly:

```ts
// 🚨 hand-rolled fetch with all the bugs
const [data, setData] = useState<Invoice[] | null>(null)
const [loading, setLoading] = useState(true)
const [error, setError] = useState<Error | null>(null)
useEffect(() => {
  setLoading(true)
  fetchInvoices()
    .then(setData)
    .catch(setError)
    .finally(() => setLoading(false))
}, [])
```

This has at least the following bugs/missing features:

- No request cancellation on unmount.
- No deduplication if two components want the same data.
- No background refetching.
- No retry on failure.
- Race condition if dependencies change mid-fetch.
- No cache.

Replace with `useQuery`. See `tanstack-query/essentials.md`.

---

## Rule: Lift state to the lowest common ancestor — and no higher

When two siblings need the same state, lift to the closest common parent. Not to the root. Not to a global store.

```tsx
// 🚨 lifting too high — every keystroke re-renders the whole app
function App() {
  const [draft, setDraft] = useState('')
  return <Layout><Sidebar /><Editor draft={draft} setDraft={setDraft} /></Layout>
}

// ✅ keep state where it's actually shared
function EditorPanel() {
  const [draft, setDraft] = useState('')
  return <><EditorToolbar draft={draft} /><Editor draft={draft} setDraft={setDraft} /></>
}
```

If the lowest common ancestor is the root and the state is genuinely needed everywhere, that's when Zustand earns its place.

---

## Rule: Server state and client state interact via *derivation*, not synchronization

Common pattern: a list (server state) and a selection (client state). The selection *depends on* the list — but you don't sync.

```ts
// ✅ selection is stored client-side; "valid selection" is derived
const { data: users } = useQuery(usersQuery)
const [selectedId, setSelectedId] = useUserSelectionStore()
const validSelection = users?.find(u => u.id === selectedId) ?? null
const isSelectionValid = validSelection !== null
```

Patterns:

- The selection survives if the user comes back.
- "Is the selection valid?" is a derived flag for UI.
- No effects, no syncing, no race conditions.

---

## Rule: Forms

Forms blur the line between client and server state.

- **Submitted data** lives server-side (you're going to POST it). Don't pre-store it in a global client store.
- **Draft input** is local UI state. Use `useState` (React 19 native form actions can simplify further).
- **Default values** come from server state. Pass them as props from the loader/query, don't sync via effect.

For complex forms: React Hook Form or Conform. For simple forms: native `<form action={serverAction}>` (React 19 / RSC).

---

## Step 3: A worked example

Designing an "Invoices" page with filters, sorting, pagination, and selection.

```ts
// 1. URL state — filters, sort, page (shareable)
const [searchParams, setSearchParams] = useSearchParams()
const filter = searchParams.get('filter') ?? 'all'
const sort = searchParams.get('sort') ?? 'date'
const page = Number(searchParams.get('page') ?? '1')

// 2. Server state — the invoice list, parameterized by URL state
const { data, isPending, isError, isFetching, refetch } = useQuery({
  queryKey: ['invoices', { filter, sort, page }],
  queryFn: () => fetchInvoices({ filter, sort, page }),
  placeholderData: keepPreviousData, // retain previous list while loading next page
})

// 3. Client UI state — which invoice is selected (ephemeral, not shareable)
const [selectedId, setSelectedId] = useState<string | null>(null)

// 4. Derived state — current selected invoice object
const selectedInvoice = data?.items.find(i => i.id === selectedId) ?? null
```

Notes:

- Filter / sort / page → URL → query key → automatic refetch on change.
- Selection → local state. If we want it to persist across page refresh, move to URL too.
- `selectedInvoice` is derived. Never stored.
- `placeholderData: keepPreviousData` keeps the previous list visible while the new one loads (see `tanstack-query/retain-while-refetching.md`).

---

## References

- TkDodo — *Don't over useState*, *Deriving Client State from Server State*, *React Query as a State Manager*
- React docs — *You Might Not Need an Effect*
- See also: `tanstack-query/essentials.md`, `state-management/zustand-basics.md`
