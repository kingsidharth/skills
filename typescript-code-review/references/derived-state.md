# Derived state from server state

**When this applies:** any client state (`useState`, Zustand, URL state) whose validity depends on server data — selections, filters, "currently editing" indicators, focused-row IDs.

The naive solution: a `useEffect` that watches server data and calls `setSelectedId(null)` when the selection becomes invalid. **Don't do that.** Derive instead.

---

## The bug

```tsx
// 🚨 sync client state to server state via effect
const useSelectedUser = () => {
  const { data: users } = useQuery(usersQuery)
  const { selectedUserId, setSelectedUserId } = useUserStore()

  useEffect(() => {
    if (users && !users.some(u => u.id === selectedUserId)) {
      setSelectedUserId(null)
    }
  }, [users, selectedUserId, setSelectedUserId])

  return [selectedUserId, setSelectedUserId]
}
```

Problems:

- Wipes selection if a refetch transiently doesn't include it (server inconsistency, racing pagination).
- If the user is re-added later, the selection is permanently gone.
- Adds a render pass (effect → setState → re-render).
- Hides the relationship: nothing in the function shape says "selection depends on the user list".

---

## The fix: derive

```tsx
// ✅ store the user's *intent*; derive what's *currently valid*
const useSelectedUser = () => {
  const { data: users } = useQuery(usersQuery)
  const { selectedUserId, setSelectedUserId } = useUserStore()

  const validSelection = users?.find(u => u.id === selectedUserId) ?? null

  return { selectedUserId, validSelection, setSelectedUserId }
}
```

Behaviour:

- `selectedUserId` records the last user the *user* selected.
- `validSelection` is the user object iff the selection is still in the list.
- Both are exposed; consumer picks based on intent ("show selection state in UI" vs "use the selected user's data").
- If the user is deleted then re-added, selection is restored automatically.
- No effect, no re-renders, no sync.

---

## Pattern 1: Selection that may become invalid

```tsx
function useSelectedItem() {
  const { data: items } = useQuery(itemsQuery)
  const [selectedId, setSelectedId] = useState<string | null>(null)

  const selected = items?.find(i => i.id === selectedId) ?? null
  const isSelectionStale = selectedId !== null && selected === null

  return {
    selectedId,
    setSelectedId,
    selected,
    isSelectionStale,  // useful for "this item was deleted" UI
  }
}
```

The component can show:

- The selection in the UI when valid.
- A subtle "this item is no longer available" state when stale, without auto-clearing.
- Restoration if the item comes back.

---

## Pattern 2: Default value from server data

A common shape: form initialized from server data. The naive version uses an effect:

```tsx
// 🚨 pre-fill via effect — overwrites user edits on refetch
function Editor() {
  const { data: user } = useQuery(userQuery)
  const [draft, setDraft] = useState('')
  useEffect(() => { if (user) setDraft(user.bio) }, [user])
  return <textarea value={draft} onChange={e => setDraft(e.target.value)} />
}
```

Two correct alternatives.

### Derive the effective value

```tsx
function Editor() {
  const { data: user } = useQuery(userQuery)
  const [draft, setDraft] = useState<string | undefined>(undefined)
  const effectiveValue = draft ?? user?.bio ?? ''
  return <textarea value={effectiveValue} onChange={e => setDraft(e.target.value)} />
}
```

`draft === undefined` means "user hasn't typed anything yet, show server value". On first keystroke, `draft` becomes a real string and takes over.

### Use `key` to remount per entity (preferred when applicable)

```tsx
function EditorPage({ userId }: Props) {
  const { data: user } = useQuery(userQueries.detail(userId))
  if (!user) return <Skeleton />
  return <Editor key={userId} initialBio={user.bio} />
}

function Editor({ initialBio }: { initialBio: string }) {
  const [draft, setDraft] = useState(initialBio)  // initial value used because key forces remount
  return <textarea value={draft} onChange={e => setDraft(e.target.value)} />
}
```

See `react/props-to-state.md` for the full breakdown.

---

## Pattern 3: Combining filters with server data

Filters are client/URL state; the filtered list is server state, parameterized by filters. Derive nothing manually — make filters part of the query key.

```tsx
const [filter, setFilter] = useQueryState('filter')  // URL state
const { data: items } = useQuery({
  queryKey: ['items', { filter }],
  queryFn: () => fetchItems(filter),
})
```

Changing `filter` automatically triggers a refetch (different cache key). No effect, no manual filtering of a global cache.

If you need to filter *client-side* (e.g., across data already in cache), `select`:

```tsx
const { data: filteredItems } = useQuery({
  queryKey: ['items'],
  queryFn: fetchAllItems,
  select: (items) => items.filter(i => i.matches(filter)),
})
```

The query is still keyed on `['items']` so the underlying request is one. The select runs per render and is cheap thanks to structural sharing.

---

## Pattern 4: Combined zustand + query

```tsx
// zustand store of user-applied filters
const useAppliedFilters = () => useFilterStore(s => s.applied)
const useFilterActions  = () => useFilterStore(s => s.actions)

// query that reads filters from the store
function useFilteredTodos() {
  const filters = useAppliedFilters()
  return useQuery({
    queryKey: ['todos', filters],
    queryFn: () => fetchTodos(filters),
  })
}
```

The store *drives* the query via the key. Changing filters → new key → new query. The store doesn't store the todos. The query doesn't store the filters. Clean separation.

---

## Anti-patterns that look like this but aren't

### Caching server data in client state

```tsx
// 🚨 storing a copy
const { data: users } = useQuery(usersQuery)
const [usersCopy, setUsersCopy] = useState<User[]>([])
useEffect(() => { if (users) setUsersCopy(users) }, [users])
```

There is no reason for `usersCopy` to exist. Use `users` directly. If you want to mutate, use a mutation.

### "Pre-loading" via effect

```tsx
// 🚨
useEffect(() => {
  queryClient.prefetchQuery(productsQueries.list())
}, [])
```

This works but is unnecessary. Prefetch where the user *signals* intent — on hover, on focus, on route prefetch — not on mount. For initial-page prefetch, do it in a route loader, not an effect.

```tsx
// ✅ prefetch on hover
<Link
  to={`/products/${id}`}
  onMouseEnter={() => queryClient.prefetchQuery(productQueries.detail(id))}
>
  ...
</Link>
```

---

## The mental shift

| Imperative thinking | Declarative thinking |
|---|---|
| "When server data changes, sync client state" | "Client state is intent. The current valid value is derived." |
| "When selection becomes invalid, clear it" | "Selection is the user's last action. Validity is a property, not a side effect." |
| "Pre-fill the form when data loads" | "The form's effective value falls back to server data when no draft exists." |

The declarative version has fewer effects, fewer race conditions, and matches React's model of "UI is a function of state".

---

## References

- TkDodo — *Deriving Client State from Server State*
- Kent C. Dodds — *Don't Sync State, Derive It*
- See also: `react/useeffect-discipline.md`, `react/props-to-state.md`, `02-feature-review/data-flow-and-state-shape.md`
