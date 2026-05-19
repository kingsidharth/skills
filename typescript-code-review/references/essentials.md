# TanStack Query essentials

**When this applies:** any client-side data-fetching using `@tanstack/react-query`. Read this before writing the first `useQuery`.

---

## Rule 1: One source of truth — Query owns server state

Server state belongs in the query cache. Period. Do not duplicate into:

- `useState` ("I need a copy to mutate locally") — derive instead, or use `useMutation`.
- Zustand / Redux ("I need it in another part of the app") — call `useQuery` again with the same key; Query dedupes.
- Effects that mirror data ("syncing to my store") — never. See `react/useeffect-discipline.md`.

If two components need the same data, both call `useQuery(['users'])`. There's one network request, one cache entry, both render.

---

## Rule 2: Query keys describe the data, hierarchically

```ts
// ✅ structured, partial-match invalidation works
['users']                   // root
['users', 'list']           // list view
['users', 'list', { filter, sort }]  // filtered list
['users', 'detail', userId]
```

`queryClient.invalidateQueries({ queryKey: ['users'] })` invalidates everything user-related. `{ queryKey: ['users', 'list'] }` invalidates only lists.

Use a query factory to keep keys consistent:

```ts
export const userQueries = {
  all: () => ({ queryKey: ['users'] as const }),
  lists: () => ({ queryKey: [...userQueries.all().queryKey, 'list'] as const }),
  list: (filters: Filters) => ({
    queryKey: [...userQueries.lists().queryKey, filters] as const,
    queryFn: () => fetchUsers(filters),
  }),
  details: () => ({ queryKey: [...userQueries.all().queryKey, 'detail'] as const }),
  detail: (id: string) => ({
    queryKey: [...userQueries.details().queryKey, id] as const,
    queryFn: () => fetchUser(id),
  }),
}

// usage
const { data } = useQuery(userQueries.detail(id))
```

This:

- Centralizes the queryFn next to the key.
- Eliminates drift between consumers.
- Lets you invalidate any layer.
- TypeScript infers everything.

In TanStack Query v5+, use `queryOptions()` helper for the same pattern with built-in helpers.

---

## Rule 3: `staleTime` is not optional

Default `staleTime` is `0` — every query is immediately stale, refetches on every focus and remount. That's almost never what you want.

| Data type | `staleTime` |
|---|---|
| Configuration / rarely changes | `Infinity` |
| User profile, slowly changing | `5 * 60_000` (5 min) |
| Live-ish data with WS push | `Infinity` (WS handles updates) |
| Lists that other users mutate | `30_000`–`2 * 60_000` |
| Real-time critical (prices) | `0` (or use WS) |

Set defaults on the `QueryClient`:

```ts
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,
      gcTime: 5 * 60_000,
      retry: (count, err) => {
        if (err instanceof HttpError && err.status >= 400 && err.status < 500) return false
        return count < 2
      },
    },
  },
})
```

Override per-query when it differs:

```ts
useQuery({ ...userQueries.detail(id), staleTime: Infinity })
```

---

## Rule 4: Don't supply explicit generics to `useQuery` — let inference flow

```ts
// 🚨 forces the entire chain to use User, breaking select-narrowing
useQuery<User>({ queryKey: [...], queryFn: fetchUser })

// ✅ infer from queryFn return type
useQuery({
  queryKey: [...],
  queryFn: fetchUser,  // returns Promise<User>
})

// then select narrows correctly:
useQuery({
  queryKey: [...],
  queryFn: fetchUser,
  select: (u) => u.email,  // data: string ✅
})
```

Type the queryFn's return; everything else flows. Manual generics break `select`, partial subscriptions, and combined hooks.

---

## Rule 5: `select` for partial subscription, not derivation in the component

```tsx
// 🚨 subscribes to the whole user, re-renders on any user change
function UserEmail({ id }: Props) {
  const { data: user } = useQuery(userQueries.detail(id))
  return <span>{user?.email}</span>
}

// ✅ subscribes only to user.email; re-renders only when email changes
function UserEmail({ id }: Props) {
  const { data: email } = useQuery({
    ...userQueries.detail(id),
    select: (u) => u.email,
  })
  return <span>{email}</span>
}
```

Wrap the pattern in a custom hook:

```ts
function useUserEmail(id: string) {
  return useQuery({
    ...userQueries.detail(id),
    select: (u) => u.email,
  })
}
```

`select` runs on data change; structural sharing means components subscribed to other slices don't re-render. See `tanstack-query/render-optimizations.md`.

---

## Rule 6: Don't access cache via `useEffect`

```ts
// 🚨 subscribing to data via effect
useEffect(() => { console.log(data) }, [data])

// ✅ derive what you need during render, or use queryClient
const data = useQuery(...).data
```

If you genuinely need to react to data updates (e.g., to call a side-effecting third-party library), use `useQuery`'s `meta` or a dedicated subscription via `queryClient.getQueryCache().subscribe(...)` — never an effect.

---

## Rule 7: Mutations describe intent, not setters

```ts
// ✅ named for the thing the user is doing
const updateEmail = useMutation({
  mutationFn: (email: string) => api.updateEmail({ id: userId, email }),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
})

// usage
<button onClick={() => updateEmail.mutate(newEmail)}>Save</button>
```

Pair with optimistic updates via `onMutate` / `onError` / `onSettled` (see TanStack docs) or React 19 `useOptimistic`.

---

## Rule 8: Suspense mode for clean components

If your project supports Suspense:

```tsx
// query
const { data } = useSuspenseQuery(userQueries.detail(id))
// data is non-nullable from here — no `data?.` needed

// boundary
<Suspense fallback={<Skeleton />}>
  <ErrorBoundary fallback={<ErrorState />}>
    <UserDetail id={id} />
  </ErrorBoundary>
</Suspense>
```

The Suspense boundary handles loading; ErrorBoundary handles errors. The component itself is just the success branch — no conditional soup, no narrowing. See `react/composition-over-conditionals.md` and `02-feature-review/ui-states.md`.

---

## Rule 9: Don't disable retry without thinking

`retry: false` is occasionally right (4xx errors, rate-limit). Default retry-with-exponential-backoff handles most transient network issues invisibly. Disable selectively, not globally:

```ts
queries: {
  retry: (count, err) => {
    if (err instanceof HttpError && err.status >= 400 && err.status < 500) return false
    return count < 2
  },
}
```

---

## Rule 10: `enabled` for dependent queries — not effects

```ts
// 🚨 effect-driven dependent fetch
useEffect(() => { if (userId) fetchProjects(userId) }, [userId])

// ✅ enabled flag
const { data: projects } = useQuery({
  queryKey: ['projects', userId],
  queryFn: () => fetchProjects(userId!),
  enabled: !!userId,
})
```

`enabled: false` keeps the query in `pending` state without firing. Use also for "don't fetch until user clicks" scenarios — pair with `refetch()`.

---

## Rule 11: Tracked queries (default in v4+) — don't break them

By default, only the fields a component reads are subscribed to. This means:

```ts
const { data } = useQuery(...)  // re-renders only when data changes
const { data, isFetching } = useQuery(...)  // re-renders on either
```

Don't break tracking by spreading:

```ts
// 🚨 spread invokes every getter — subscribes to everything
const queryInfo = useQuery(...)
return <Foo {...queryInfo} />

// ✅ destructure only what you need
const { data, error } = useQuery(...)
```

See `tanstack-query/render-optimizations.md`.

---

## Rule 12: Test with a fresh `QueryClient` per test

```ts
// in test setup
function renderWithClient(ui: ReactElement) {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false }, mutations: { retry: false } },
  })
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>)
}
```

Disable retry in tests so failures surface immediately. Mock HTTP with MSW so the same handlers serve dev and tests.

---

## References

- TkDodo — *Practical React Query*, *React Query as a State Manager*, *Effective React Query Keys*, *React Query and TypeScript*, *Type-safe React Query*, *The Query Options API*
- TanStack docs — Query Options, Suspense Mode
- See also: `tanstack-query/render-optimizations.md`, `tanstack-query/derived-state.md`, `tanstack-query/retain-while-refetching.md`
