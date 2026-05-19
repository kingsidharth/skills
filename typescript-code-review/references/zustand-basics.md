# Zustand basics

**When this applies:** designing or reviewing client-side state shared across components that aren't parent-child. Read after first checking `02-feature-review/data-flow-and-state-shape.md` to confirm that Zustand is the right tool (most of the time it isn't).

Zustand is the right tool when:

- State is genuinely shared across the tree (not just parent → child).
- It's not server state (that's TanStack Query).
- It's not URL state (that's the router).
- It's not local UI state (that's `useState`).

If none of those alternatives apply, read on.

---

## Rule 1: Only export custom hooks — never the raw store

```ts
// 🚨 exposing the store invites whole-store subscriptions
export const useBearStore = create<BearState>((set) => ({ ... }))

// at call site:
const { bears } = useBearStore()  // subscribes to entire store, re-renders on any change
```

```ts
// ✅ keep the store private; export atomic hooks
const useBearStore = create<BearState>((set) => ({
  bears: 0,
  fish: 0,
  actions: {
    increasePopulation: (by) => set((s) => ({ bears: s.bears + by })),
    eatFish: () => set((s) => ({ fish: s.fish - 1 })),
  },
}))

export const useBears = () => useBearStore((s) => s.bears)
export const useFish = () => useBearStore((s) => s.fish)
export const useBearActions = () => useBearStore((s) => s.actions)
```

Consumers can't accidentally subscribe to the whole store. Adding new state later doesn't break existing consumers. The custom hook is the API.

---

## Rule 2: Atomic selectors

Each exported hook should select **one** atomic value. Don't combine.

```ts
// 🚨 returns a new object every render — re-renders on any store change
export const useBearAndFish = () =>
  useBearStore((s) => ({ bears: s.bears, fish: s.fish }))

// ✅ separate atomic hooks
export const useBears = () => useBearStore((s) => s.bears)
export const useFish = () => useBearStore((s) => s.fish)

// component picks both
function Display() {
  const bears = useBears()
  const fish = useFish()
  return <span>{bears} bears, {fish} fish</span>
}
```

The pattern: one hook, one slice, one re-render trigger. Components that want multiple slices call multiple hooks.

If you genuinely need an object selector, use shallow comparison:

```ts
import { useShallow } from 'zustand/react/shallow'

export const useBearAndFish = () =>
  useBearStore(useShallow((s) => ({ bears: s.bears, fish: s.fish })))
```

But default to atomic. It's almost always cleaner.

---

## Rule 3: Separate actions namespace

Group action functions under a single `actions` key. Actions never change identity, so subscribing to the whole `actions` object is free.

```ts
const useBearStore = create<BearState>((set) => ({
  bears: 0,
  fish: 0,
  actions: {
    increasePopulation: (by) => set((s) => ({ bears: s.bears + by })),
    eatFish: () => set((s) => ({ fish: s.fish - 1 })),
    removeAllBears: () => set({ bears: 0 }),
  },
}))

// ✅ one selector for all actions, no re-render concern
export const useBearActions = () => useBearStore((s) => s.actions)

// usage
function Buttons() {
  const { increasePopulation, eatFish } = useBearActions()
  return <>...</>
}
```

This separation:

- Decouples reads from writes — components that only call actions never re-render on state change.
- Lets you pass `actions` as a stable object to memoized children.
- Works well with TypeScript's `Readonly` type.

---

## Rule 4: Model actions as events, not setters

```ts
// 🚨 setter-shaped actions leak business logic to the caller
actions: {
  setBears: (n: number) => set({ bears: n }),
}
// caller has to know the rule: "after eating fish, bears go up by 1"
useBearActions().setBears(bears + 1)

// ✅ event-shaped actions encapsulate the rule
actions: {
  eatFish: () =>
    set((s) => ({ fish: s.fish - 1, bears: s.bears + 1 })),
}
useBearActions().eatFish()
```

Same rule as Redux's style guide: name actions for **what happened** ("user clicked save", "fish was eaten"), not **what to mutate** ("setBears", "decrementFish"). Encapsulating the rule once means it can't be wrong at any of N call sites.

---

## Rule 5: Keep stores small and focused

Zustand encourages many small stores, not one big one. Group state that genuinely changes together; split state that doesn't.

```ts
// ✅ separate stores per concern
const useFilterStore = create<FilterState>((set) => ({ ... }))
const useNotificationStore = create<NotificationState>((set) => ({ ... }))
const useThemeStore = create<ThemeState>((set) => ({ ... }))
```

Combine in custom hooks where needed:

```ts
function useFilteredTodos() {
  const filters = useFilterStore((s) => s.applied)
  return useQuery({
    queryKey: ['todos', filters],
    queryFn: () => fetchTodos(filters),
  })
}
```

The filter store doesn't need to know about the query. The query consumes whatever's in the filter store. Composition.

---

## Rule 6: Don't store server state in Zustand

Server data — anything you fetched — does not go in Zustand. It goes in TanStack Query. The Zustand store holds the *intent* / *user-facing client state*; the query holds the *data*.

```ts
// 🚨 caching server data in zustand
const useUsersStore = create((set) => ({
  users: [],
  fetchUsers: async () => set({ users: await api.getUsers() }),
}))

// ✅ TanStack Query owns the data; Zustand owns "which user is selected"
const useUsers = () => useQuery({ queryKey: ['users'], queryFn: fetchUsers })
const useSelectedUserId = () => useUserSelectionStore((s) => s.selectedId)
```

See `02-feature-review/data-flow-and-state-shape.md`.

---

## Rule 7: Persistence (when needed)

Use the `persist` middleware. Be explicit about what to persist (most state shouldn't be):

```ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: 'system',
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'theme-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ theme: state.theme }),  // exclude transient state
    },
  ),
)
```

Versioning: when the shape changes, bump `version` and provide a migration. Don't break users' existing localStorage.

---

## Rule 8: TypeScript shape

Type the state once, derive everything else:

```ts
type BearState = {
  bears: number
  fish: number
  actions: {
    increasePopulation: (by: number) => void
    eatFish: () => void
    removeAllBears: () => void
  }
}

const useBearStore = create<BearState>()((set) => ({ ... }))
```

The double-call pattern `create<BearState>()(...)` is intentional — it gives you better inference on middlewares than `create<BearState>(...)`.

For middlewares, compose them in this order: `devtools(persist(immer(...)))` — devtools outermost so it sees everything.

---

## Rule 9: When to skip Zustand

You don't need it if:

- The state can be **lifted** to a common parent. (Most cases.)
- It's URL state. (`nuqs`, route params.)
- It's server state. (Query.)
- A specific `Provider` already exists for the concern (theme, auth).

Reach for Zustand only when those alternatives are genuinely worse than a global store.

---

## References

- TkDodo — *Working with Zustand*
- Zustand docs — patterns, middleware, TypeScript
- Mark Erikson — *Modern Redux with Redux Toolkit* (the same patterns apply)
- See also: `state-management/zustand-with-context.md`
