# Stale closures and hook dependencies

**When this applies:** writing or reviewing any `useEffect`, `useMemo`, `useCallback`, or custom hook with a dependency array. Also when wrapping components in `React.memo` or designing custom equality functions.

The single most common runtime bug in React code is a function that captures stale values because its dependency array lied. The fix is preventive: never lie.

---

## The mental model

A function defined inside a component is *re-created on every render*. It "closes over" whatever variables were in scope at the moment it was created. That snapshot is fixed. The next render makes a new function with a new snapshot.

When you wrap that function in `useCallback(fn, [deps])`, you're telling React: "Give me back the **old** snapshot if `deps` haven't changed." That works *if and only if* you list every value the function reads.

Omit one and you get a function still seeing a value from an earlier render — a **stale closure**. The bug appears intermittently, in production, weeks later.

---

## Rule 1: `react-hooks/exhaustive-deps` is `error`, never `warn`

```jsonc
{
  "rules": {
    "react-hooks/exhaustive-deps": "error"
  }
}
```

Treat the lint rule as the spec. The rule's job is to flag every reference your function makes that isn't in the deps. If the rule is `warn`, it gets ignored. Set it to `error`.

---

## Rule 2: Never silence the rule without an explanation

If you genuinely must omit a dep, write *why* in a comment, in the same line as the disable:

```ts
// eslint-disable-next-line react-hooks/exhaustive-deps -- ref.current is intentionally unstable; we only want the *current* node, never the historical one
```

If you can't write a coherent reason, you can't omit the dep.

---

## Rule 3: Don't fight stable refs

Stable values don't need to be in deps:

- `useRef(...).current` — the ref object itself is stable; you don't need it in deps unless you read `.current` and want to react to it (which is itself a code smell).
- `setState` from `useState` — stable.
- `dispatch` from `useReducer` — stable.
- `queryClient` from `useQueryClient` — stable.
- `navigate` from React Router / Next — stable.

These are exempted by the lint rule. If the rule still flags one, it's lying — investigate.

---

## Rule 4: If a dependency keeps changing, fix the dependency, not the deps array

```tsx
// 🚨 onChange recreated every render → effect runs every render
function Parent() {
  return <Child onChange={(v) => console.log(v)} />
}

function Child({ onChange }: { onChange: (v: string) => void }) {
  useEffect(() => { /* ... uses onChange ... */ }, [onChange])
}
```

The fix is in the *parent* — stabilize the function:

```tsx
function Parent() {
  const handleChange = useCallback((v: string) => console.log(v), [])
  return <Child onChange={handleChange} />
}
```

Or better, lift the work elsewhere so the effect doesn't need to depend on a function at all.

---

## Rule 5: Functions that only need to read latest values → escape hatch via ref

There's a real case where you want a function that *always* reads the latest value of something but is itself stable. Do this with a ref, not by lying about deps:

```tsx
// ✅ stable callback that reads latest count
function useStableCallback<T extends unknown[]>(fn: (...args: T) => void) {
  const ref = useRef(fn)
  useEffect(() => { ref.current = fn })
  return useCallback((...args: T) => ref.current(...args), [])
}
```

React 19 introduces `useEffectEvent` for the same purpose — prefer it when available:

```tsx
const onSomething = useEffectEvent((arg: string) => {
  // can read latest props/state without listing them as deps
})
```

`useEffectEvent` is for code called *from inside an effect*. Don't use it as a general "skip deps" lever.

---

## Rule 6: Memoization should match the use site

Wrapping `<Child />` in `React.memo` only helps if `Child`'s props are referentially stable across renders. If the parent passes a fresh object/function/array each render, `memo` does nothing.

```tsx
// 🚨 memo + unstable props = no memoization
const Child = React.memo(InnerChild)

function Parent() {
  return <Child config={{ size: 10 }} onClick={() => {}} />  // both unstable
}
```

Fix at the parent or accept that memo isn't appropriate here.

```tsx
function Parent() {
  const config = useMemo(() => ({ size: 10 }), [])
  const handleClick = useCallback(() => {}, [])
  return <Child config={config} onClick={handleClick} />
}
```

But: don't reflexively memo. Most of the time it's not the bottleneck. The React Compiler is increasingly handling this automatically. Measure first.

---

## Rule 7: Custom equality functions for `React.memo` are a footgun

```tsx
// 🚨 ignoring onChange in equality means it stays stale
const Memoized = React.memo(SlowComponent, (prev, next) => prev.value === next.value)
```

If you exclude a prop from the equality check, the component will use the **first** version of that prop forever (or until something else triggers a re-render). Functions defined in render are recreated each time, so this is effectively saying "always use the original `onChange`" — and that closes over original state. Stale closure, hard to find.

If `onChange` keeps changing, fix it at the parent. Don't paper over it.

---

## Rule 8: Computed deps must themselves be stable

```ts
// 🚨 keys array is recreated every render
useEffect(() => { /* uses keys */ }, [Object.keys(obj)])
```

`Object.keys(obj)` is a fresh array every time. The effect runs on every render. If you genuinely want to depend on the keys, derive a stable representation:

```ts
const keysJoined = Object.keys(obj).sort().join(',')
useEffect(() => { /* ... */ }, [keysJoined])
```

Or restructure so you don't need to derive a stable form.

---

## Rule 9: Don't use deps as a re-run trigger for code that should be elsewhere

Common shape:

```ts
// 🚨 using deps to "react to" prop change
useEffect(() => { onSelect(value) }, [value])
```

This is a side-effect chain disguised as a hook. If you really want to call `onSelect` whenever `value` changes due to a user action, call it in the event handler that changed `value`. If `value` changes for a non-user reason and `onSelect` represents a user intent, the design is wrong — there should be no effect.

---

## Rule 10: When deps must include functions, name them

For readability and correctness, give inline functions to hooks names so they're easier to spot in the dep array:

```ts
// 🚨 inline lambda — easy to omit accidentally from review
useMemo(() => items.filter((i) => i.active), [items, /* ??? */])

// ✅ stabilize then use
const isActive = useCallback((i: Item) => i.active, [])
const active = useMemo(() => items.filter(isActive), [items, isActive])
```

For the trivial filter case, just inline the predicate (no function dep at all). For more complex cases, naming reveals the dep surface.

---

## References

- TkDodo — *Hooks, Dependencies and Stale Closures*, *Refs, Events and Escape Hatches*, *Ref Callbacks, React 19 and the Compiler*
- React docs — *Reusing Logic with Custom Hooks*, *Removing Effect Dependencies*
- React 19 — `useEffectEvent`
