# New component checklist

**When this applies:** every time a new `function MyComponent` or `const MyComponent = (...)` is about to be written. Run through this before writing the body.

The single most common AI-generated mistake is a wrapper component that does not justify its existence. This file's primary job is to prevent that.

---

## Step 0: Does this component need to exist at all?

A new component file is justified only when one of these is true:

| Justification | Example |
|---|---|
| The same JSX appears in **3+ places** with the same shape | `Card`, `EmptyState`, `LoadingSpinner` |
| It encapsulates **state, refs, effects, or events** the caller would otherwise have to wire up | `<Combobox>`, `<DataTable>`, `<DragHandle>` |
| It defines a **clear composition boundary** (children / slots) | `<Modal>`, `<Tabs>`, `<Form>` |
| It names a **domain concept** the team uses in conversation | `<InvoiceStatusPill>`, `<UserAvatar>` |

If none apply, do not extract. Inline at the call site.

### Anti-patterns to refuse

```tsx
// 🚨 placement-named wrapper
function HomepageButtonCTAAnimated({ children, onClick }) {
  return <Button variant="primary" size="lg" className="animate-pulse" onClick={onClick}>{children}</Button>
}

// 🚨 single-prop pass-through
function PrimaryButton(props) {
  return <Button variant="primary" {...props} />
}

// 🚨 hook with no logic
function useUser(id: string) {
  return useQuery({ queryKey: ['user', id], queryFn: () => fetchUser(id) })
}
// ...if nothing else lives here, just put useQuery at the call site.

// 🚨 div soup wrappers
function CardWrapper({ children }) {
  return <div className="card-wrapper"><div className="card-inner">{children}</div></div>
}
```

If AI-generated code introduces any of these, reject and inline.

---

## Step 1: Naming

Names describe **what the thing IS**, never **where it appears**.

| Bad | Good |
|---|---|
| `HomepageHeroSection` | `HeroSection` (placement comes from caller) |
| `LoginPagePrimaryButton` | inline `<Button variant="primary">` |
| `DashboardEmptyState` | `EmptyState` with a `title` prop |
| `BlueRoundedSmallButton` | inline; don't bake style words into a name |

Layered placement names (`Homepage*`, `Dashboard*`, `Pricing*`) are an alarm — they multiply with every new page × variant combination.

---

## Step 2: Prop API

A component's prop API is the contract. Keep it minimal and discoverable.

### Use `ComponentProps<typeof X>` to inherit native types

```tsx
// 🚨 narrow prop type — forces re-declaring everything
type ButtonProps = {
  onClick?: () => void
  children: ReactNode
  className?: string
}

// ✅ inherit from the underlying primitive
type ButtonProps = ComponentProps<'button'> & {
  variant?: 'primary' | 'secondary'
  size?: 'sm' | 'md' | 'lg'
}
```

This is foundational. A component wrapping a `<button>` should accept all `<button>` attributes by default — `disabled`, `type`, `onClick`, `aria-*`, `data-*`, `form`, `name`, etc. — without you re-declaring them.

For wrapping another component:

```tsx
type CtaButtonProps = ComponentProps<typeof Button> & { /* extras */ }
```

### Discriminated unions for mutually exclusive props

```tsx
// 🚨 both optional, both could be passed, neither could be
type Props = { href?: string; onClick?: () => void }

// ✅ either-or, never neither, never both
type Props =
  | { href: string; onClick?: never }
  | { href?: never; onClick: () => void }
```

### Don't accept `style` and `className` "for flexibility" if you don't need to

If consumers will need to override styles, accept `className`. If they won't, don't. Open APIs invite drift.

### `children` typing

Use `ReactNode` for content; do not narrow to `ReactElement` unless you genuinely need it.

```tsx
function Card({ children }: { children: ReactNode }) { ... }
```

---

## Step 3: Composition over conditional rendering

If a component's JSX has more than one or two `condA ? <X/> : null` lines, restructure with **early returns + a shared layout**. See `react/composition-over-conditionals.md` for the full rule.

```tsx
// 🚨 conditional soup
return (
  <Card>
    <Heading>{title}</Heading>
    {isPending ? <Skeleton /> : null}
    {!data && !isPending ? <EmptyState /> : null}
    {data ? data.items.map(...) : null}
  </Card>
)

// ✅ one branch per state, shared Layout
if (isPending) return <Layout title={title}><Skeleton /></Layout>
if (!data)     return <Layout title={title}><EmptyState /></Layout>
return <Layout title={title}>{data.items.map(...)}</Layout>
```

---

## Step 4: State

Before adding state, ask:

1. Can this be **derived** from props or server data? → derive (`react/composition-over-conditionals.md`).
2. Is it **server state**? → TanStack Query, not `useState`.
3. Is it **URL state** (filter, sort, pagination)? → search params, not `useState`.
4. Is it **shared with siblings far away**? → lift up, or Zustand.
5. Otherwise → `useState`.

Never `useEffect` to sync prop → state. See `react/props-to-state.md`.

---

## Step 5: Effects

Default answer: **don't**. See `react/useeffect-discipline.md`.

If you genuinely need one:

- Subscribing to a non-React system (DOM event, WebSocket, IntersectionObserver, geolocation): OK.
- Anything else: not OK.

---

## Step 6: Refs

If interacting with the DOM after render (focus, measure, scroll, animate), prefer **callback refs** over `useRef` + `useEffect`.

```tsx
// ✅ callback ref binds to node lifecycle, not component
const inputRef = useCallback((node: HTMLInputElement | null) => {
  node?.focus()
}, [])

return <input ref={inputRef} />
```

See `react/callback-refs-over-effects.md`.

---

## Step 7: Accessibility floor

Non-negotiable:

- Use the right element. `<button>` not `<div onClick>`. `<a href>` not `<div onClick={navigate}>`. `<form>` for submission.
- Labels on inputs (`<label htmlFor>` or wrapping).
- Visible focus state (do not `outline: none` without replacing).
- `aria-*` only when there's no native equivalent.
- `prefers-reduced-motion` respected for non-decorative animation.

---

## Step 8: Tests (when applicable)

Tests verify **behaviour from the user's perspective**, not implementation:

- ✅ "When the user clicks Submit with an invalid email, an error appears."
- ❌ "useState is called with `''`" / "the `validate` function returns false".

Use Testing Library queries that mirror real assistive tech (`getByRole`, `getByLabelText`).

---

## Quick checklist (paste this into PR template)

```
- [ ] Component name describes what it IS, not where it APPEARS
- [ ] Used ≥3 times OR encapsulates state/effects/composition/domain concept
- [ ] Prop type extends ComponentProps<...> when wrapping a primitive
- [ ] No useEffect that just runs setState
- [ ] No prop-to-state sync via useEffect
- [ ] UI states are discrete branches with a shared Layout
- [ ] Real <button>, <a>, <form> — not divs with onClick
- [ ] No `any`; no unjustified `as`
```

---

## References

- TkDodo — *Component Composition is great btw*
- Matt Pocock — *ComponentProps in React TypeScript*
- React docs — *Thinking in React*
