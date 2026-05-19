# useEffect discipline

**When this applies:** before writing any `useEffect`. This is the single most-misused hook in React; the floor for getting it right is to assume you don't need one.

This file maps directly to React's official docs page [*You Might Not Need an Effect*](https://react.dev/learn/you-might-not-need-an-effect). Read that page if you have not. The patterns below are the actionable summary.

---

## The rule, stated plainly

`useEffect` is an **escape hatch** for synchronizing React with **systems outside React**. Outside-React = the DOM, the network, browser APIs, third-party widgets, timers.

If your effect's body only touches React state and props, you almost certainly don't need it.

---

## Decision tree

When tempted to write `useEffect`, ask in this order:

1. **Is the body just transforming data for rendering?** → compute during render.
2. **Is it caching an expensive calculation?** → `useMemo`.
3. **Is it resetting state when a prop changes?** → `key` prop on the component.
4. **Is it adjusting state based on a prop change?** → derive instead, or set state during render with previous-state comparison.
5. **Is it a chain (effect → setState → effect → setState)?** → collapse into a single render-time computation or event handler.
6. **Is it for an event (click, submit, etc.)?** → event handler, not effect.
7. **Is it notifying parent of state change?** → lift state to parent.
8. **Is it fetching data?** → TanStack Query / loader.
9. **Is it initializing application state once?** → top-level module code or one-time `useState` initializer.
10. **Is it subscribing to a non-React system (DOM event, store, WebSocket, IntersectionObserver)?** → effect, **but** consider callback ref for DOM-bound work (see `react/callback-refs-over-effects.md`).

Only step 10 justifies an effect.

---

## Anti-pattern: chains

```tsx
// 🚨 chained effects each trigger the next
useEffect(() => { setIsActive(items.length > 0) }, [items])
useEffect(() => { if (isActive) setStatus('open') }, [isActive])
useEffect(() => { if (status === 'open') setBadgeCount(items.length) }, [status, items])
```

Three render passes for nothing. Compute everything during render:

```tsx
const isActive = items.length > 0
const status = isActive ? 'open' : 'closed'
const badgeCount = isActive ? items.length : 0
```

Effect chains exist because someone is thinking imperatively ("when X changes, then Y, then Z"). React renders are values of state — express the relationships directly.

---

## Anti-pattern: transforming data

```tsx
// 🚨 effect that filters, then sets state
const [filtered, setFiltered] = useState<Item[]>([])
useEffect(() => {
  setFiltered(items.filter(i => i.active))
}, [items])
```

```tsx
// ✅ derive
const filtered = items.filter(i => i.active)
```

If filtering is expensive *and you've measured it*, wrap in `useMemo`. Not by default.

---

## Anti-pattern: handling events

```tsx
// 🚨 effect-based event handler
useEffect(() => {
  if (status === 'success') {
    showToast('Saved!')
    navigate('/dashboard')
  }
}, [status])
```

This fires on remount, on Strict Mode double-mount, and at any time `status` is `'success'` for any reason. Move into the event:

```tsx
const handleSubmit = async () => {
  await save()
  showToast('Saved!')
  navigate('/dashboard')
}
```

---

## Anti-pattern: prop-to-state sync

```tsx
// 🚨 syncing prop into state with effect — race-condition-prone
function Detail({ initialEmail }: { initialEmail: string }) {
  const [email, setEmail] = useState(initialEmail)
  useEffect(() => { setEmail(initialEmail) }, [initialEmail])
  // ...
}
```

Three correct alternatives:

```tsx
// ✅ Option 1: reset by changing key
<Detail key={selectedId} initialEmail={selectedEmail} />

// ✅ Option 2: lift state up — make Detail fully controlled
<Detail email={email} onEmailChange={setEmail} />

// ✅ Option 3: conditional rendering (mount/unmount)
{selectedId && <Detail key={selectedId} initialEmail={selectedEmail} />}
```

See `react/props-to-state.md` for full treatment.

---

## Anti-pattern: notifying the parent

```tsx
// 🚨 child useEffect to inform parent
function Toggle({ onChange }: { onChange: (v: boolean) => void }) {
  const [on, setOn] = useState(false)
  useEffect(() => { onChange(on) }, [on, onChange])
  // ...
}
```

The state should not live in the child if the parent needs it. Lift up:

```tsx
function Toggle({ on, onChange }: { on: boolean; onChange: (v: boolean) => void }) {
  return <button onClick={() => onChange(!on)}>{on ? 'On' : 'Off'}</button>
}
```

---

## Anti-pattern: data fetching

```tsx
// 🚨 useEffect + fetch
useEffect(() => {
  let cancelled = false
  setLoading(true)
  fetch(url)
    .then(r => r.json())
    .then(d => { if (!cancelled) setData(d) })
    .finally(() => { if (!cancelled) setLoading(false) })
  return () => { cancelled = true }
}, [url])
```

Replace with TanStack Query (or a route loader):

```ts
const { data, isPending } = useQuery({
  queryKey: ['resource', url],
  queryFn: () => fetch(url).then(r => r.json()),
})
```

You get cancellation, dedup, retry, cache, refetch on focus, devtools — for free.

---

## Anti-pattern: one-time initialization in App

```tsx
// 🚨 fires twice in dev (Strict Mode), and on every remount
function App() {
  useEffect(() => { initAnalytics() }, [])
  // ...
}
```

If it must run exactly once per app load, run it at module scope or guard it:

```ts
// ✅ module-level — runs once on import
initAnalytics()

export function App() { ... }
```

If it depends on browser APIs, guard for SSR:

```ts
if (typeof window !== 'undefined') {
  initAnalytics()
}
```

For per-component "run once" *inside* render — use the `useState` initializer:

```tsx
const [client] = useState(() => createSomeClient())
```

The initializer runs exactly once even in Strict Mode.

---

## What's left for `useEffect`?

Legitimate uses:

```tsx
// ✅ subscribing to a non-React store
useEffect(() => {
  const unsub = externalStore.subscribe(() => forceUpdate())
  return unsub
}, [])

// ✅ DOM event the platform requires
useEffect(() => {
  const onKey = (e: KeyboardEvent) => { /* ... */ }
  window.addEventListener('keydown', onKey)
  return () => window.removeEventListener('keydown', onKey)
}, [])

// ✅ third-party imperative library
useEffect(() => {
  const map = new MapboxMap(containerRef.current!)
  return () => map.remove()
}, [])

// ✅ WebSocket connection (but see realtime/websocket-with-api-fallback.md)
useEffect(() => {
  const ws = new WebSocket(url)
  return () => ws.close()
}, [url])
```

Even some of these have better alternatives:

- DOM-bound side-effect on mount: callback ref. See `react/callback-refs-over-effects.md`.
- External-store subscription: `useSyncExternalStore`.
- IntersectionObserver / ResizeObserver: callback ref.

---

## Strict Mode is your friend

React 18+ Strict Mode runs effects **twice** in dev. If your effect breaks under double-invocation, it's brittle code, not a Strict Mode bug. Either:

- Make the effect idempotent (cleanup properly, guard repeated work).
- Replace the effect with a non-effect pattern (most cases).

---

## Tooling

- `react-hooks/exhaustive-deps` must be `error`. Never silence without an inline justification comment.
- `eslint-plugin-react-you-might-not-need-an-effect` (`strict` config) — automated detection of:
  - `no-derived-state` — effect that derives state
  - `no-chain-state-updates` — effect chain
  - `no-event-handler` — event-shaped effect
  - `no-adjust-state-on-prop-change` — prop-sync
  - `no-reset-all-state-on-prop-change` — should be `key`
  - `no-pass-live-state-to-parent` — should lift state
  - `no-pass-data-to-parent` — should lift fetch
  - `no-initialize-state` — should be `useState` initializer
  - `no-empty-effect`

Add this plugin to every project.

---

## References

- React docs — *You Might Not Need an Effect*
- React docs — *Synchronizing with Effects*
- TkDodo — *Avoiding useEffect with callback refs*, *Hooks, Dependencies and Stale Closures*
- `eslint-plugin-react-you-might-not-need-an-effect`
