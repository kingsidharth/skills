# Zustand with React Context

**When this applies:** when a Zustand store should *not* be a global singleton — when state should be scoped to a route, a feature subtree, or a reusable component instance.

This is not Context-as-state-management (a known anti-pattern that re-renders everything subscribed to the value). It's Context-as-store-injection — the Context value is the **store instance**, which itself doesn't change. Subscriptions still flow through Zustand's optimized selector path.

---

## When global isn't right

The default `create(...)` pattern produces a singleton — one store per app load. That's fine for truly app-wide state (theme, auth, toast queue). It's wrong for:

- **Route-scoped state.** A "Dashboard filters" store has no business existing on `/login`.
- **Reusable components.** Two `<MultiSelect>` instances on the same page must not share state.
- **Initialization from props.** A global store can't be initialized with values that come from outside React (a server-rendered prop, a URL slug).
- **Testing.** Global state requires explicit reset between tests; scoped state is gone when the component unmounts.

---

## The pattern

```ts
import { createStore, useStore } from 'zustand'
import { createContext, useContext, useState } from 'react'

// 1. Define the store creator (not the singleton itself)
type FilterState = {
  applied: string[]
  actions: {
    addFilter: (f: string) => void
    clearFilters: () => void
  }
}

function createFilterStore(initial: string[]) {
  return createStore<FilterState>((set) => ({
    applied: initial,
    actions: {
      addFilter: (f) => set((s) => ({ applied: [...s.applied, f] })),
      clearFilters: () => set({ applied: [] }),
    },
  }))
}

type FilterStore = ReturnType<typeof createFilterStore>

// 2. Context holds the store instance
const FilterStoreContext = createContext<FilterStore | null>(null)

// 3. Provider creates a store per render tree, initialized from props
export function FilterStoreProvider({
  initial,
  children,
}: {
  initial: string[]
  children: React.ReactNode
}) {
  // useState initializer runs exactly once — even in Strict Mode
  const [store] = useState(() => createFilterStore(initial))
  return <FilterStoreContext.Provider value={store}>{children}</FilterStoreContext.Provider>
}

// 4. Consumers go through a custom hook that uses the right store instance
function useFilterStore<T>(selector: (state: FilterState) => T): T {
  const store = useContext(FilterStoreContext)
  if (!store) throw new Error('FilterStoreProvider missing in tree')
  return useStore(store, selector)
}

// 5. Atomic-selector hooks — same as global pattern
export const useAppliedFilters = () => useFilterStore((s) => s.applied)
export const useFilterActions  = () => useFilterStore((s) => s.actions)
```

Usage:

```tsx
function DashboardPage() {
  const initial = useDefaultFilters()  // from URL, server, etc.
  return (
    <FilterStoreProvider initial={initial}>
      <FilterBar />
      <DashboardGrid />
    </FilterStoreProvider>
  )
}
```

---

## Why `useState(() => createStore(...))`?

The initializer form is critical:

- The store is created **once**, on first render of the provider.
- React 18 Strict Mode double-invokes function bodies but not `useState` initializers — so the store is created exactly once.
- The store is captured in `useState` and never replaced — `setStore` is never called.

A `useRef` would also work (`const ref = useRef<FilterStore | null>(null); if (!ref.current) ref.current = createFilterStore(initial)`), but `useState` is more idiomatic.

Don't do this:

```tsx
// 🚨 creates a new store on every render
const store = createFilterStore(initial)
```

---

## Why this isn't the "Context as state manager" antipattern

The thing that makes Context unsuitable for state management is that **every consumer re-renders when the Context value changes**. Here, the Context value is the *store instance* — a stable object that never changes after mount. Subscribers go through `useStore(store, selector)`, which uses Zustand's `useSyncExternalStore` integration to subscribe granularly.

| | "Context as state" | "Context as store injection" |
|---|---|---|
| Context value | Changes on every state update | Stable for the lifetime of the provider |
| Re-renders | All consumers, on every change | Only consumers whose selectors return new values |

The pattern keeps Zustand's render-optimization story intact while gaining scoping.

---

## Initialization from props

The whole point: the store can be initialized with values that vary per provider instance.

```tsx
function MultiSelect({ options, defaultSelected }: Props) {
  return (
    <MultiSelectStoreProvider options={options} initial={defaultSelected}>
      <MultiSelectImpl />
    </MultiSelectStoreProvider>
  )
}
```

Two `<MultiSelect>` on the same page each get their own store. No global collision.

---

## Testing

Tests render the component naturally; the store is implicit:

```tsx
test('selecting an option updates the badge', async () => {
  render(
    <MultiSelectStoreProvider options={['a', 'b']} initial={[]}>
      <MultiSelectImpl />
    </MultiSelectStoreProvider>,
  )
  await user.click(screen.getByRole('option', { name: 'a' }))
  expect(screen.getByText('1 selected')).toBeInTheDocument()
})
```

No "reset the global store between tests" boilerplate. Each test gets a fresh store via the provider.

---

## When to skip Context-scoping

If you're scoping a store to "the current route" and your router can pass data down, it's often simpler to make the route component own a normal React state and pass values via props. Don't reach for Zustand-with-context just because Zustand-without-context is your default — check if you need the global-ish access pattern at all.

---

## When to keep the store global

| Use Context-scoped | Use global |
|---|---|
| Per-route or per-feature state | Theme |
| Multi-instance reusable component | Auth user |
| Initialization from props | Toast / notification queue |
| Test isolation matters | Connection / WebSocket status |

If a store would be created exactly once per app load no matter what, just use `create()`. Don't overengineer.

---

## References

- TkDodo — *Zustand and React Context*
- Zustand docs — *Initialize state with props*
- React docs — `useSyncExternalStore`
- See also: `state-management/zustand-basics.md`
