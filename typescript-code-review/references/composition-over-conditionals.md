# Composition over conditional rendering

**When this applies:** any component that renders different content for different states (loading, error, empty, success, role, mode, etc.).

The wrong shape:

```tsx
return (
  <Card>
    <Heading>{title}</Heading>
    {isPending ? <Skeleton /> : null}
    {isError ? <ErrorState /> : null}
    {!isPending && !isError && !data ? <EmptyState /> : null}
    {data ? data.items.map(...) : null}
  </Card>
)
```

The right shape: **early returns + a shared layout component**.

---

## The rule

If a component has more than one or two `condA ? <X /> : null` lines, restructure with:

1. A `Layout` (or similar) component that captures the shared shell.
2. Early returns from the main component, one per state, each rendering through the layout.

This pattern:

- Makes each state visually obvious in the code.
- Allows TypeScript to narrow types per branch (e.g., `data` is non-null in the success branch).
- Lets new states be added without breaking existing ones.

---

## Worked example

### Before

```tsx
function ShoppingList() {
  const { data, isPending, isError } = useQuery(/* ... */)

  return (
    <Card>
      <CardHeading>Welcome 👋</CardHeading>
      <CardContent>
        {isPending ? <Skeleton /> : null}
        {isError ? <ErrorState /> : null}
        {!isPending && !isError && !data ? <EmptyState /> : null}
        {data?.assignee ? <UserInfo {...data.assignee} /> : null}
        {data ? data.items.map(item => <Item key={item.id} {...item} />) : null}
      </CardContent>
    </Card>
  )
}
```

Five conditions in seven lines. To understand "what does the user see in pending state?" you have to mentally evaluate every branch.

### After

```tsx
function Layout({ children, title }: { children: ReactNode; title?: string }) {
  return (
    <Card>
      <CardHeading>Welcome 👋 {title}</CardHeading>
      <CardContent>{children}</CardContent>
    </Card>
  )
}

function ShoppingList() {
  const { data, isPending, isError } = useQuery(/* ... */)

  if (isPending) return <Layout><Skeleton /></Layout>
  if (isError)   return <Layout><ErrorState /></Layout>
  if (!data)     return <Layout><EmptyState /></Layout>

  return (
    <Layout title={data.title}>
      {data.assignee ? <UserInfo {...data.assignee} /> : null}
      {data.items.map(item => <Item key={item.id} {...item} />)}
    </Layout>
  )
}
```

Each branch is one state. Adding a new state is one new `if`. TypeScript automatically narrows `data` to non-null in the final branch. Layout duplication is shallow — three lines repeated, vs three nested conditions removed.

---

## Why "this duplicates the layout" is the wrong concern

Common pushback: "I'm rendering `<Layout>` four times." That's not a bug; it's an asset. It means each branch is fully independent and can evolve independently. If two branches diverge ("error layout has a different header"), the divergence shows up at the call site, not buried in another conditional.

If `<Layout>` accumulates many props for "this branch needs X, that branch needs Y" — you've found the wrong abstraction. Pull the differing parts out of `Layout` rather than parameterizing further.

---

## Patterns where conditional rendering is fine

A single conditional inside JSX, for an *optional* piece of content, is fine:

```tsx
return (
  <Card>
    <Heading>{title}</Heading>
    {description ? <p>{description}</p> : null}
    <Body>{children}</Body>
  </Card>
)
```

This isn't a state branch. It's just optional content. One line, low cognitive load.

The smell starts when conditionals are **mutually exclusive states** — only one of them ever shows at a time, but you can't tell that from the code.

---

## TypeScript narrowing for free

```tsx
function Foo() {
  const { data, isPending, isError } = useQuery(...)

  if (isPending) return <Skeleton />
  if (isError) return <ErrorState />
  // here, TypeScript knows data is not undefined
  return <Detail item={data} />
}
```

Compare to the conditional-soup version where TS can't narrow:

```tsx
return data ? <Detail item={data} /> : <Fallback />
// data?.foo? data!.foo? both annoying
```

The early-return form earns you cleaner types as a side effect.

---

## Multiple components vs one big component

If your component has 5+ branches, that's a sign it's doing too much. Split:

```tsx
// 🚨 one component, every state
function InvoiceArea({ id }) {
  const query = useQuery(...)
  if (query.isPending) return <Pending />
  if (query.isError)   return <Err onRetry={query.refetch} />
  if (!query.data)     return <Empty />
  return <InvoiceDetail invoice={query.data} />
}

// ✅ split when the success path is large
function InvoiceArea({ id }) {
  const query = useQuery(...)
  if (query.isPending) return <Pending />
  if (query.isError)   return <Err onRetry={query.refetch} />
  if (!query.data)     return <Empty />
  return <InvoiceDetail invoice={query.data} />
}

function InvoiceDetail({ invoice }: { invoice: Invoice }) {
  // success-only logic, with a non-nullable invoice prop
}
```

`InvoiceDetail` only ever runs in the success state and its prop type reflects that. This is the same pattern as discriminated unions for state machines: each state has its own type, its own component.

---

## Don't render `null` from a component just to "skip" it

```tsx
// 🚨 component shouldn't be in the tree if it has nothing to show
function Banner({ message }: { message?: string }) {
  if (!message) return null
  return <div className="banner">{message}</div>
}
```

Sometimes this is fine. But often the parent already knows whether `message` exists, so:

```tsx
{message ? <Banner message={message} /> : null}
```

The `Banner` component then has a non-optional `message` prop. Cleaner types, cleaner intent.

---

## References

- TkDodo — *Component Composition is great btw*
- Josh W. Comeau — *Statements vs Expressions*
- React docs — *Conditional Rendering*, *Thinking in React*
