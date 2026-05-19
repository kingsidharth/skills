# Rendering performance

**When this applies:** when an interaction feels janky, the React DevTools Profiler shows long commits, or you're auditing a component that renders large amounts of data.

The first rule: **fix the slow render before fixing the re-render.** A single 300 ms render is worse than ten 5 ms re-renders. Profile first, optimize second.

---

## Triage order

When a page feels slow, check in this order. Most apps clear up after the first two.

1. **Are you shipping too much JavaScript?** — bundle size and waterfalls dominate any in-render optimization. See `01-project-and-review/component-inventory-control.md`.
2. **Is one render slow?** — open Profiler, find the component with the longest commit. Fix that.
3. **Are too many components re-rendering on a small change?** — only after 1 and 2 are clean.

---

## Slow render diagnosis

In the React DevTools Profiler:

1. Record an interaction (typing, clicking, scrolling).
2. Sort commits by duration.
3. Find the longest commit, then the longest component within it.
4. Look at why: doing too much computation? Rendering too many children? Using a slow third-party component?

### Common causes

| Symptom | Fix |
|---|---|
| `arr.filter().map().sort()` recomputed every render with thousands of items | Move computation behind `useMemo` keyed on the source array |
| A list of 1000+ rows | Virtualize with `@tanstack/react-virtual` |
| A heavy computation in render path that shouldn't run on every input event | `useDeferredValue` for the input result, or `useTransition` to mark it non-urgent |
| Large markdown / code block re-rendered on each typing keystroke | Memoize the rendered output, or split state |
| Whole-page re-render on a small change | State lifted too high — push it down |

---

## Re-render diagnosis

Re-renders are usually caused by one of:

- **State lifted too high** — typing in one input re-renders an unrelated chart at the page root.
- **Provider value not memoized** — every render recreates the value, every consumer re-renders.
- **Unstable props** — passing a new object/function/array per render to a memoized child.
- **Whole-store subscription** — see `tanstack-query/render-optimizations.md` for tracked queries; see `state-management/zustand-basics.md` for atomic selectors.

```tsx
// 🚨 every consumer of UserContext re-renders on every render
function UserProvider({ user, children }) {
  return (
    <UserContext.Provider value={{ user, isAdmin: user.role === 'admin' }}>
      {children}
    </UserContext.Provider>
  )
}

// ✅
function UserProvider({ user, children }) {
  const value = useMemo(
    () => ({ user, isAdmin: user.role === 'admin' }),
    [user],
  )
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>
}
```

If a Context value is heterogeneous (some parts change often, some don't), split into multiple Contexts. This is one of the few cases where more files = better perf.

---

## Memoization that actually helps

The React Compiler (stable in React 19+) handles most of this automatically. For projects on older React or where the compiler isn't enabled:

```tsx
// Useful: the wrapped value's identity matters for downstream
const handleSubmit = useCallback((data: FormData) => save(data), [save])

// Useful: the computation is expensive and re-runs without need
const filtered = useMemo(
  () => items.filter(predicate),
  [items, predicate],
)

// Useful: the wrapped child is React.memo and renders are expensive
const MemoizedChart = useMemo(
  () => <Chart data={data} />,
  [data],
)
```

Useless or harmful:

```tsx
// 🚨 memoizing a primitive — useless
const count = useMemo(() => items.length, [items])

// 🚨 useCallback on something never passed as a stable dep — useless
const handleClick = useCallback(() => console.log('hi'), [])

// 🚨 memoizing a child whose props are new every render — no effect
const memoChild = useMemo(() => <Child config={{}} />, [])
```

Default rule: **don't memoize unless you measured the bottleneck**.

---

## List performance

Three concerns:

1. **Keys.** Stable per-item identity. Don't use array index unless the list is genuinely append-only and immutable.
2. **Length.** If > ~100 items rendered at once, virtualize. `@tanstack/react-virtual`.
3. **Per-row work.** A row component should be cheap. If it's doing heavy formatting, memoize it (`React.memo` + stable props).

```tsx
// ✅
const Row = React.memo(function Row({ item }: { item: Item }) {
  return <div>{item.name} — {formatCurrency(item.amount)}</div>
})
```

For sortable / filterable tables, prefer libraries that handle virtualization + sorting + selection (TanStack Table) over rolling your own.

---

## Transitions and deferred values (React 18+)

`useTransition` and `useDeferredValue` mark non-urgent updates so the UI stays responsive during heavy work.

```tsx
function Search() {
  const [query, setQuery] = useState('')
  const deferredQuery = useDeferredValue(query)
  // expensive list rendering uses deferredQuery
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ResultList query={deferredQuery} />
    </>
  )
}
```

Typing keeps the input responsive; the list updates when React has time. The list shows the *previous* result during the transition (no skeleton flash).

```tsx
function TabSwitcher() {
  const [tab, setTab] = useState('home')
  const [isPending, startTransition] = useTransition()
  return (
    <>
      <button onClick={() => startTransition(() => setTab('reports'))}>
        Reports {isPending && '…'}
      </button>
      <TabContent tab={tab} />
    </>
  )
}
```

Don't wrap urgent state in `startTransition` (the input value, hover state, button feedback). That makes the UI feel laggy. Wrap *the heavy follow-on work*.

---

## What the React Compiler does for you

In a project with the React Compiler enabled:

- Most `useMemo`/`useCallback` become unnecessary — the compiler memoizes automatically.
- `React.memo` is largely redundant because props are referentially stable.
- The mental model shifts from "memoize everything I might forget" to "let the compiler do it; only intervene if profiling shows a real cost."

Adopt the compiler if your project's React version supports it. It's the largest single win available for re-render performance.

---

## Hot-spot anti-patterns

```tsx
// 🚨 reading expensive computed value via getter on every render
const total = items.reduce((sum, i) => sum + i.amount, 0)

// ✅ memoize when items is large and stable across most renders
const total = useMemo(() => items.reduce((s, i) => s + i.amount, 0), [items])
```

```tsx
// 🚨 reading localStorage on every render
function Theme() {
  const stored = localStorage.getItem('theme') ?? 'light'
  return <div className={stored}>...</div>
}

// ✅ read once at init
function Theme() {
  const [theme] = useState(() => localStorage.getItem('theme') ?? 'light')
  return <div className={theme}>...</div>
}
```

```tsx
// 🚨 creating a Date or Intl.NumberFormat on every render
function Price({ amount }) {
  const fmt = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
  return <span>{fmt.format(amount)}</span>
}

// ✅ hoist to module scope
const usd = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
function Price({ amount }) {
  return <span>{usd.format(amount)}</span>
}
```

---

## What to measure, not guess

| Tool | What it shows |
|---|---|
| React DevTools Profiler | Commit duration, render reasons, what changed |
| Chrome Performance Panel | Long tasks, layout thrashing, paint, frame budget |
| Lighthouse / Web Vitals | LCP, INP (interaction-to-next-paint), CLS — the metrics users feel |
| Bundle analyzer | Total JS, per-route, large modules |
| `why-did-you-render` (sparingly) | Components that re-rendered with identical props |

Don't optimize what you didn't measure. The optimizations above are well-tested patterns; applying them blindly still wastes time.

---

## References

- Kent C. Dodds — *Fix the slow render before you fix the re-render*
- Mark Erikson — *A (Mostly) Complete Guide to React Rendering Behavior*
- Nadia Makarevich — *Advanced React*, performance flame graphs series
- Vercel `agent-skills/react-best-practices` — `rendering-*`, `js-*` rules
- See also: `tanstack-query/render-optimizations.md`, `state-management/zustand-basics.md`
