# React Query render optimizations

**When this applies:** when a component is re-rendering more than expected, or when wiring up a query for a deeply-rendered subtree (lists, dashboards, graphs).

Re-renders are not inherently bad. Most of the time, "fix the slow render before fixing the re-render" is the right priority. But TanStack Query has a small number of free-or-near-free wins that are worth knowing.

---

## Rule 1: Tracked queries (the default in v4+) — don't break them

By default, `useQuery` tracks which fields you read during render and only re-renders the component when *those fields* change.

```ts
// only re-renders when `data` changes
const { data } = useQuery(...)

// re-renders on data OR isFetching change
const { data, isFetching } = useQuery(...)
```

This is automatic. Don't fight it.

### Things that break tracking

```ts
// 🚨 rest spread invokes every getter — subscribes to all fields
const { isLoading, ...rest } = useQuery(...)

// 🚨 same
const queryInfo = useQuery(...)
return <Foo {...queryInfo} />

// ✅ destructure what you need
const { data, error } = useQuery(...)
```

```ts
// 🚨 tracked queries don't track effect-only access
const queryInfo = useQuery(...)
useEffect(() => { console.log(queryInfo.data) })  // not tracked → may be stale

// ✅ access during render or via dep array
useEffect(() => { console.log(queryInfo.data) }, [queryInfo.data])
```

If you must opt out of tracking for some reason: `notifyOnChangeProps: 'all'`. Don't do this without a measured reason.

---

## Rule 2: `select` for fine-grained subscription

A component that only needs `user.email` should re-render only when `email` changes — not on any other change to the user.

```ts
// 🚨 re-renders on every user field change
function UserEmail({ id }: Props) {
  const { data: user } = useQuery({ ...userQueries.detail(id) })
  return <span>{user?.email}</span>
}

// ✅ subscribed only to email
function UserEmail({ id }: Props) {
  const { data: email } = useQuery({
    ...userQueries.detail(id),
    select: (u) => u.email,
  })
  return <span>{email}</span>
}
```

`select` works with structural sharing — TanStack Query checks the *result of the selector* for referential equality and only notifies the observer if it changed. This is the right hammer for partial subscriptions.

You can return derived shapes too:

```ts
const { data: stats } = useQuery({
  ...invoiceQueries.list(),
  select: (invoices) => ({
    total: invoices.length,
    paid: invoices.filter(i => i.status === 'paid').length,
    overdue: invoices.filter(i => i.status === 'overdue').length,
  }),
})
```

The component re-renders only when the *computed stats object* changes (which, with structural sharing, means the underlying counts changed).

---

## Rule 3: Custom hooks per slice

Wrap each `select` shape in a custom hook so consumers don't reimplement:

```ts
function useUserEmail(id: string) {
  return useQuery({ ...userQueries.detail(id), select: u => u.email })
}

function useUserName(id: string) {
  return useQuery({ ...userQueries.detail(id), select: u => u.name })
}
```

Each component imports the slice it needs. The underlying query is shared — one network request, one cache entry.

---

## Rule 4: Structural sharing — leave it on

By default, TanStack Query performs structural sharing on every cache update: identical sub-objects keep their reference identity. This is what makes `select` cheap and what enables granular re-rendering.

Only turn off when you know you have a giant non-JSON-serializable cache (and at that point, maybe the cache shape is wrong):

```ts
useQuery({ ...query, structuralSharing: false })
```

Default = on. Don't change it without a reason.

---

## Rule 5: Don't memoize what TanStack Query already memoizes

Returned `data`, `error`, etc. are referentially stable across re-renders if the underlying values haven't changed. You don't need to wrap them in `useMemo` before passing to children.

```ts
// 🚨 redundant
const { data } = useQuery(...)
const memoizedData = useMemo(() => data, [data])

// ✅ pass directly
const { data } = useQuery(...)
return <List items={data} />
```

---

## Rule 6: List virtualization for long lists

If you have hundreds or thousands of rows, React's reconciliation cost dominates. Use `@tanstack/react-virtual` to render only the visible rows:

```tsx
const rowVirtualizer = useVirtualizer({
  count: data.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 48,
})
```

This is independent of TanStack Query but pairs naturally — the cache holds all items, the virtualizer renders ~30 of them.

---

## Rule 7: When `useMemo` / `useCallback` actually help

For most components, the React Compiler (or future React Compiler in stable) handles this automatically. Where manual memoization still helps:

- The wrapped value is the prop of a `React.memo`-wrapped child whose render is genuinely expensive.
- The wrapped value is in a hook dependency array, and recreating it on every render thrashes the hook.

Don't reflexively wrap. The cost of `useMemo` itself is non-zero. Measure with the React DevTools Profiler.

---

## Rule 8: Don't memoize selectors

Selectors passed to `useQuery({ select })` get re-created on every render. That's *fine* — TanStack Query doesn't compare select function identity, it compares the select *result*. So:

```ts
// ✅ inline selector — fine
useQuery({ ...query, select: (u) => u.email })

// 😐 useMemo on the selector — adds work without benefit
const select = useMemo(() => (u: User) => u.email, [])
useQuery({ ...query, select })
```

If your selector is expensive (heavy compute), memoize the *computation* via `useMemo` *outside* `select`. But for typical field-picking, inline is correct.

---

## Rule 9: Identifying the actual bottleneck

Before any of this, profile. The typical priority order, from highest impact to lowest:

1. **Eliminate request waterfalls** (parallelize fetches, prefetch on hover).
2. **Reduce bundle size** (the JS that re-renders is the JS that ships — see `01-project-and-review/component-inventory-control.md`).
3. **Slow renders** — a single 200ms render is worse than ten 5ms re-renders.
4. **Re-render frequency** — only after the above are clean.

The Vercel `react-best-practices` skill's category ordering is exactly this: critical → high → medium → low. Re-renders are medium.

---

## References

- TkDodo — *React Query Render Optimizations*, *React Query Selectors, Supercharged*, *React Query Data Transformations*
- Kent C. Dodds — *Fix the slow render before you fix the re-render*
- Vercel `agent-skills/react-best-practices` — `rendering-*` rules
- See also: `performance/rendering-performance.md`
