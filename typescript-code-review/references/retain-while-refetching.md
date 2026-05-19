# Retain current view while refetching

**When this applies:** any time fetching new data would cause a flash to skeleton — pagination, filtering, sorting, switching between detail views, polling, or refetching after a mutation.

The principle: **never throw away rendered data while loading the next batch**. The user has already seen something. Replacing it with a skeleton is a visual regression.

---

## The bug

```tsx
// 🚨 every page change flashes the skeleton
function InvoiceList() {
  const [page, setPage] = useState(1)
  const { data, isPending } = useQuery({
    queryKey: ['invoices', page],
    queryFn: () => fetchInvoices(page),
  })

  if (isPending) return <Skeleton />   // 👈 fires on every page
  return (
    <>
      <List items={data} />
      <button onClick={() => setPage(p => p + 1)}>Next</button>
    </>
  )
}
```

`isPending` is `true` whenever the cache for that key is empty. Page 1 renders fine. Click "Next" → page key changes → no cache → `isPending: true` → flash to skeleton → data arrives → list reappears. Jittery.

---

## Fix 1: `placeholderData: keepPreviousData`

```tsx
import { keepPreviousData } from '@tanstack/react-query'

function InvoiceList() {
  const [page, setPage] = useState(1)
  const { data, isPending, isPlaceholderData, isFetching } = useQuery({
    queryKey: ['invoices', page],
    queryFn: () => fetchInvoices(page),
    placeholderData: keepPreviousData,
  })

  if (isPending) return <Skeleton />  // only on the very first load
  return (
    <>
      <List items={data} dim={isPlaceholderData} />
      <button
        onClick={() => setPage(p => p + 1)}
        disabled={isPlaceholderData}
      >
        Next
      </button>
    </>
  )
}
```

Behaviour:

- First load → no cache, no placeholder → `isPending: true` → skeleton.
- Subsequent navigations → previous page's data is shown as placeholder while the new page loads.
- `isPlaceholderData: true` while serving the placeholder → use it to dim the list and disable "Next" so the user can't double-click.
- New data arrives → `isPlaceholderData` flips to `false`, list updates in place.

This is the right pattern for **paginated lists**, **filter changes**, and **switching between detail views of the same entity type**.

---

## Fix 2: Specific `placeholderData` from another query

When clicking from a list to a detail view, the list query already has the row. Don't make the user wait — seed the detail with what you have:

```tsx
function InvoiceDetail({ id }: { id: string }) {
  const queryClient = useQueryClient()

  const { data } = useQuery({
    queryKey: ['invoices', 'detail', id],
    queryFn: () => fetchInvoice(id),
    placeholderData: () => {
      // pull from any list cache that contains this id
      return queryClient
        .getQueriesData<Invoice[]>({ queryKey: ['invoices', 'list'] })
        .flatMap(([, list]) => list ?? [])
        .find(inv => inv.id === id)
    },
  })

  // data renders immediately from the list cache; query refetches the full detail in the background
}
```

Patterns:

- The detail page renders *instantly* with whatever the list had.
- A background fetch replaces it with the full detail (more fields, fresh data).
- No skeleton flash, no perceived navigation latency.

This is the TanStack Query equivalent of optimistic navigation.

---

## Fix 3: Distinguish `isPending` from `isFetching` in the UI

| Flag | Meaning | UI treatment |
|---|---|---|
| `isPending` | No data yet (initial load) | Skeleton / empty layout |
| `isFetching` | A request is in flight, but data may already be present | Subtle indicator (top progress bar, faint pulse) |
| `isPlaceholderData` | Currently rendering placeholder, real data fetching | Slightly dim the placeholder, disable mutation triggers |

```tsx
return (
  <Layout>
    {isFetching && !isPending && <TopProgressBar />}
    <List items={data} dim={isPlaceholderData} />
  </Layout>
)
```

Don't use `isFetching` to gate the skeleton. That's the bug from `02-feature-review/ui-states.md`.

---

## Fix 4: Mutations — show optimistic immediately, never blank

After a mutation, the standard pattern is `onSuccess: () => queryClient.invalidateQueries(...)`. That marks the cache stale and triggers a background refetch — but `data` remains rendered the whole time, and `isFetching` flips true. The list never blanks.

```tsx
const updateInvoice = useMutation({
  mutationFn: api.updateInvoice,
  onSuccess: (_, variables) => {
    queryClient.invalidateQueries({ queryKey: ['invoices', 'detail', variables.id] })
    queryClient.invalidateQueries({ queryKey: ['invoices', 'list'] })
  },
})
```

If the mutation has a clear visual outcome (toggling a checkbox, renaming), wrap with `onMutate` + rollback for true optimistic UI:

```tsx
const toggle = useMutation({
  mutationFn: api.toggleInvoice,
  onMutate: async ({ id, paid }) => {
    await queryClient.cancelQueries({ queryKey: ['invoices', 'detail', id] })
    const prev = queryClient.getQueryData<Invoice>(['invoices', 'detail', id])
    queryClient.setQueryData<Invoice>(['invoices', 'detail', id], (old) =>
      old ? { ...old, paid } : old,
    )
    return { prev }
  },
  onError: (_err, { id }, ctx) => {
    if (ctx?.prev) queryClient.setQueryData(['invoices', 'detail', id], ctx.prev)
  },
  onSettled: (_, __, { id }) => {
    queryClient.invalidateQueries({ queryKey: ['invoices', 'detail', id] })
  },
})
```

React 19 `useOptimistic` is a more idiomatic alternative when you don't need to write to the cache.

---

## Fix 5: When invalidating from a WebSocket, don't blank either

WebSocket-driven invalidation should also keep the existing view rendered:

```tsx
ws.onmessage = (e) => {
  const event = JSON.parse(e.data)
  queryClient.invalidateQueries({ queryKey: event.entity })
}
```

`invalidateQueries` marks queries stale and refetches if they have active observers. `data` stays rendered the whole time. Same pattern, no UI changes needed.

For **partial** updates (single field changes), use `setQueryData` to write directly to the cache — no refetch needed:

```tsx
ws.onmessage = (e) => {
  const event = JSON.parse(e.data)
  if (event.type === 'invoice-paid') {
    queryClient.setQueryData<Invoice>(
      ['invoices', 'detail', event.id],
      (prev) => prev ? { ...prev, paid: true } : prev,
    )
  }
}
```

See `realtime/websocket-with-api-fallback.md`.

---

## Quick decision table

| Situation | Mechanism |
|---|---|
| Pagination / filtering / sorting | `placeholderData: keepPreviousData` |
| List → detail navigation | `placeholderData: () => /* extract from list cache */` |
| Form save → list refresh | `invalidateQueries` (data stays rendered) |
| Optimistic toggle / rename | `onMutate` + rollback, or `useOptimistic` |
| WebSocket entity event (full refresh) | `invalidateQueries` (background refetch, no blank) |
| WebSocket partial field update | `setQueryData` (no refetch) |

---

## What never works

- Manually copying `data` into `useState` to "preserve it across loads" — recreates Query badly. Not necessary.
- A loading flag *inside* `useState` to prevent the skeleton — Query already has this; use the right flag.
- Suspense boundaries that wrap the whole page — defeats partial loading. Wrap the dynamic part only, or use `useQuery` (not `useSuspenseQuery`) for places where you want background refetching with rendered data.

---

## References

- TkDodo — *Practical React Query*, *Mastering Mutations in React Query*, *Seeding the Query Cache*
- TanStack Query docs — `placeholderData`, `keepPreviousData` migration
- React 19 — `useOptimistic`
- See also: `02-feature-review/ui-states.md`, `realtime/websocket-with-api-fallback.md`
