# Callback refs over `useEffect`

**When this applies:** any time you need to do something with a DOM node *after it mounts* — focus, measure, scroll, attach a non-React widget, observe, etc.

The reflexive answer is `useRef` + `useEffect`. The better answer is almost always a callback ref.

---

## The rule

If the side-effect is bound to the **DOM node's lifecycle**, use a callback ref. If it's bound to the **component's lifecycle**, use `useEffect`.

These are not the same. A node can mount, unmount, or be replaced *without* the component remounting (conditional rendering, suspended children, lazy children). A `useEffect` that runs once on component mount won't fire when the node finally appears.

---

## Anti-pattern: `useRef` + `useEffect` to focus an input

```tsx
// 🚨 fragile: assumes ref is filled when effect runs
function Form() {
  const inputRef = useRef<HTMLInputElement>(null)
  useEffect(() => {
    inputRef.current?.focus()
  }, [])
  return <input ref={inputRef} />
}
```

This works in the simple case where the input is rendered immediately. It fails the moment the input is conditionally rendered, lazy-loaded, suspended, or behind a parent that delays mounting.

---

## Correct: callback ref

```tsx
function Form() {
  const focusOnMount = useCallback((node: HTMLInputElement | null) => {
    node?.focus()
  }, [])
  return <input ref={focusOnMount} />
}
```

A ref prop accepts a **function**. React calls it with the node when the node mounts, and with `null` when the node unmounts. This matches the node's lifecycle exactly.

The `useCallback` is there to keep the ref stable across renders — without it, React would call the ref with `null` then the node again on every render, focusing repeatedly.

React 19 + the React Compiler will often stabilize the function for you, but writing `useCallback` explicitly is still correct and portable.

---

## Pattern: measuring a node

```tsx
function Measured() {
  const [height, setHeight] = useState(0)
  const measure = useCallback((node: HTMLElement | null) => {
    if (node) setHeight(node.getBoundingClientRect().height)
  }, [])
  return <div ref={measure}>{/* ... */}</div>
}
```

Note: this measures *once* on mount. For continuous measurement, use a `ResizeObserver` set up inside the same callback ref:

```tsx
const measure = useCallback((node: HTMLElement | null) => {
  if (!node) return
  const observer = new ResizeObserver((entries) => {
    setHeight(entries[0].contentRect.height)
  })
  observer.observe(node)
  return () => observer.disconnect()  // React 19 cleanup support for callback refs
}, [])
```

In React 19+, callback refs can return a cleanup function — same shape as `useEffect`. Older React versions need the cleanup managed manually (e.g., observer in a ref).

---

## Pattern: integrating an imperative library

```tsx
function ChartCanvas({ data }: { data: number[] }) {
  const setupChart = useCallback((node: HTMLCanvasElement | null) => {
    if (!node) return
    const chart = new Chart(node, { type: 'bar', data })
    return () => chart.destroy()
  }, [data])  // re-runs when data changes, with proper cleanup
  return <canvas ref={setupChart} />
}
```

Two-arg pattern: the ref runs whenever its identity changes, *and* whenever the node changes. Listing `data` in the `useCallback` deps means a new ref function on every data change → React tears down the old chart, runs the new one with the new node, fresh chart.

This is cleaner than the `useRef` + two `useEffect`s version most code defaults to.

---

## Pattern: forwarding callback refs

If your component wraps a primitive and consumers may want a ref:

```tsx
type Props = ComponentProps<'input'>

const Input = forwardRef<HTMLInputElement, Props>((props, ref) => (
  <input {...props} ref={ref} />
))
```

In React 19, forwardRef is unnecessary — refs pass through:

```tsx
function Input({ ref, ...props }: Props & { ref?: Ref<HTMLInputElement> }) {
  return <input {...props} ref={ref} />
}
```

Either way, the consumer gets to decide between an object ref or a callback ref. Don't restrict their choice.

---

## Pattern: combining refs

A wrapping component sometimes wants its own ref *and* a forwarded ref to share the same node:

```tsx
function useMergedRefs<T>(...refs: Array<Ref<T>>): RefCallback<T> {
  return useCallback((node: T | null) => {
    refs.forEach(ref => {
      if (typeof ref === 'function') ref(node)
      else if (ref) (ref as MutableRefObject<T | null>).current = node
    })
  }, refs)
}
```

Or use the well-tested utility from `@radix-ui/react-compose-refs`.

---

## When `useEffect` is actually correct

`useEffect` (not callback ref) is correct when:

- The work depends on **multiple** refs being filled simultaneously.
- The effect responds to **state**, not to node mount.
- You need the effect's lifecycle (component mount/unmount), not the node's.

Example (correct effect use):

```tsx
useEffect(() => {
  const handler = (e: KeyboardEvent) => { /* ... */ }
  window.addEventListener('keydown', handler)
  return () => window.removeEventListener('keydown', handler)
}, [])
```

There's no specific node here. The subscription lives for the component's lifetime.

---

## Quick test

Before using `useRef` + `useEffect`, ask: "Is what I'm doing about *this specific DOM node*?"

- Yes → callback ref.
- No (it's about the document, the window, a non-DOM thing) → `useEffect` is fine.

---

## References

- TkDodo — *Avoiding useEffect with callback refs*, *Ref Callbacks, React 19 and the Compiler*
- React docs — *Manipulating the DOM with Refs* (callback ref section)
- React 19 — cleanup return from ref callbacks
