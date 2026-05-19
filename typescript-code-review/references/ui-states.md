# UI states

**When this applies:** any component that fetches data, performs an async action, or has multiple visual modes.

A query is never *just* loading or done. Every async UI has at minimum **five distinct states**, and conflating them produces bugs and bad UX.

| State | Trigger | What user sees |
|---|---|---|
| **Initial loading** | First fetch, no cached data | Skeleton or spinner |
| **Empty** | Fetch succeeded, but result has no items | "No invoices yet" + CTA |
| **Error** | Fetch failed | Error message + retry button |
| **Success with data** | Fetch succeeded, items present | The list / detail / chart |
| **Refetching with previous data** | Background refresh on existing data | Previous data still rendered + subtle indicator |

Critical mistake: treating "refetching" the same as "initial loading". When the user filters a list, they should *see the previous list dim slightly*, not see a flash of skeleton.

---

## Rule 1: Branch on the right flags

For TanStack Query (v5):

| Flag | True when |
|---|---|
| `isPending` | No data in cache yet (initial load) |
| `isLoading` | Same as `isPending` && `isFetching` (alias kept for back-compat) |
| `isFetching` | Any request in flight, including background refetches |
| `isError` | Last request failed |
| `data === undefined` | Same as `isPending` |
| `data?.length === 0` | Empty result |

The bug pattern: using `isFetching` to gate the loading skeleton.

```tsx
// 🚨 flashes skeleton on every background refetch
if (isFetching) return <Skeleton />
return <List items={data} />
```

```tsx
// ✅ skeleton only on first load
if (isPending) return <Skeleton />
if (isError)   return <ErrorState onRetry={refetch} />
if (data.length === 0) return <EmptyState />
return <List items={data} isFetching={isFetching} />
```

`isFetching` becomes a *prop on the rendered list* — used for a subtle "refreshing" indicator, not a blocking skeleton.

---

## Rule 2: Branches with a shared layout — not nested ternaries

Conditional ternaries inside JSX are the wrong shape. See `react/composition-over-conditionals.md`. The version that works:

```tsx
function InvoiceList() {
  const { data, isPending, isError, refetch, isFetching } = useQuery(invoicesQuery)

  if (isPending) {
    return <Layout title="Invoices"><InvoiceListSkeleton /></Layout>
  }
  if (isError) {
    return <Layout title="Invoices"><ErrorState onRetry={refetch} /></Layout>
  }
  if (data.length === 0) {
    return <Layout title="Invoices"><EmptyInvoicesState /></Layout>
  }
  return (
    <Layout title="Invoices" isRefreshing={isFetching}>
      {data.map(invoice => <InvoiceRow key={invoice.id} {...invoice} />)}
    </Layout>
  )
}
```

Each state is a complete, self-contained branch. TypeScript narrows `data` to "defined and non-empty" in the success branch automatically.

---

## Rule 3: Empty state ≠ error state ≠ loading state

Each gets a distinct visual treatment.

### Loading (initial)

- Skeleton matching the eventual layout (preserves layout, prevents CLS).
- Avoid spinners for predictable layouts; use them for unpredictable ones.
- Never show a spinner *and* the previous data — pick one.

### Error

- Concrete message ("Couldn't load invoices") — not "Something went wrong".
- A **retry button** that calls `refetch`.
- If recoverable, suggest cause ("Check your connection").
- Log the error with context (route, query key) but show a friendly message.

```tsx
function ErrorState({ onRetry }: { onRetry: () => void }) {
  return (
    <div role="alert">
      <h3>Couldn't load invoices.</h3>
      <p>Check your connection and try again.</p>
      <button type="button" onClick={onRetry}>Retry</button>
    </div>
  )
}
```

### Empty

- A specific *first-time* empty state with a CTA ("Create your first invoice").
- Different from a "no results" empty state for a search/filter ("No invoices match 'foo'. Clear filters.").
- These two are different states even though both have `data.length === 0` — distinguish by checking if filters are applied.

```tsx
if (data.length === 0) {
  return filters.applied
    ? <Layout><NoResultsForFilter onClear={clearFilters} /></Layout>
    : <Layout><FirstRunEmptyState onCreate={createInvoice} /></Layout>
}
```

### Success (with data)

- The actual content.
- A subtle refreshing indicator if `isFetching` — usually a thin progress bar at the top of the page or a faint pulse on the data area.
- Never block interaction during a background refetch.

### Partial / streaming

If using RSC streaming, `<Suspense>` per logical chunk. Don't wrap the whole page in one `<Suspense>` — that defeats streaming. See `server-components/server-vs-client-boundary.md`.

---

## Rule 4: Mutations have their own state machine

For a `useMutation`:

```tsx
const { mutate, isPending, isError, isSuccess, reset } = useMutation({ ... })
```

The button's UI state pattern:

```tsx
<button
  type="submit"
  disabled={isPending}
  aria-busy={isPending}
>
  {isPending ? 'Saving…' : 'Save'}
</button>
{isError && <ErrorMessage onDismiss={reset} />}
```

For optimistic updates, use `useOptimistic` (React 19) or `onMutate` + rollback (TanStack Query). The pattern: render the optimistic value immediately, mark it visually as *pending* (slightly faded), revert on error.

---

## Rule 5: Skeleton must match layout

The skeleton's job is to prevent layout shift, not to entertain. Match real layout dimensions.

```tsx
// 🚨 generic spinner — layout will jump when data loads
if (isPending) return <Spinner />

// ✅ skeleton matches the real list shape
if (isPending) return (
  <ul>
    {[...Array(5)].map((_, i) => (
      <li key={i} className="h-12 rounded bg-muted/50 animate-pulse" />
    ))}
  </ul>
)
```

For dynamic-length data, use a sensible default (5–10 placeholder rows).

---

## Rule 6: A retry must work

The error state's retry button must not cause a fresh full-page skeleton flash. With TanStack Query, calling `refetch()` from an error state does the right thing — it re-uses the existing query observer, and during retry `isFetching: true` while `data` remains whatever it was (which may still be `undefined` after an initial failure — handle that case in the branch).

If the *initial* fetch failed, the retry naturally goes through the `isPending` branch again. That's correct.

---

## Quick visual map

```
       ┌──────────────┐
       │   isPending  │  → <Skeleton/>
       └──────┬───────┘
              ↓
       ┌──────────────┐
       │   isError    │  → <ErrorState onRetry={refetch}/>
       └──────┬───────┘
              ↓
       ┌──────────────┐
       │ data.length  │  → <EmptyState/>  (no filters)
       │     === 0    │      <NoResults onClear/> (with filters)
       └──────┬───────┘
              ↓
       ┌──────────────┐
       │   success    │  → <List items={data} isRefreshing={isFetching} />
       └──────────────┘
```

---

## References

- TkDodo — *Status Checks in React Query*, *Practical React Query*
- React docs — *Suspense*
- See also: `tanstack-query/retain-while-refetching.md`
