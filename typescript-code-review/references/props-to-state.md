# Props to state

**When this applies:** any time a component is initialized from props but also has its own internal draft / editing state. Detail panels, forms in modals, edit-in-place inputs.

The reflexive solution — `useEffect` to sync prop into state — is wrong. Three correct alternatives below.

---

## The bug

```tsx
// 🚨 sync prop → state with effect
function Detail({ initialEmail }: { initialEmail: string }) {
  const [email, setEmail] = useState(initialEmail)
  useEffect(() => { setEmail(initialEmail) }, [initialEmail])
  return <input value={email} onChange={e => setEmail(e.target.value)} />
}
```

Problems:

- If `initialEmail` changes for any reason mid-edit (a refetch, a parent re-render with new props), the user's draft is wiped.
- If two people have the same `initialEmail`, switching between them doesn't reset state.
- The intent (re-mount when *the entity* changes) is hidden behind data-shape checks.

---

## Why the initial value of `useState` doesn't help

```tsx
const [email, setEmail] = useState(initialEmail)
```

The initial value is **only used on the first render**. On every subsequent render of the same component instance, it's ignored. So if the parent passes a new `initialEmail`, the child still shows the original. That's why people reach for the effect — but the effect introduces the bugs above.

The correct fix is one of the three options below. Pick by use case.

---

## Option 1: Reset by `key` (most common)

The `key` prop is React's signal: "if this changes, throw away the component instance and mount a fresh one". Putting a stable identifier on `key` does exactly what we want.

```tsx
function Parent() {
  const [selectedId, setSelectedId] = useState<string>(...)
  const selected = useSelected(selectedId)

  return (
    <Detail
      key={selectedId}                // 👈 forces remount on change
      initialEmail={selected.email}
    />
  )
}
```

Behaviour:

- Selecting a new entity → key changes → React unmounts old `Detail`, mounts a new one with the new initial value.
- Within an entity, `Detail` keeps its draft state.
- No effect, no syncing, no race conditions.

This is the right fix in 80% of cases.

---

## Option 2: Lift state up

Move the state to the parent. The child becomes fully controlled.

```tsx
function Parent() {
  const [email, setEmail] = useState(selected.email)
  return <Detail email={email} onEmailChange={setEmail} />
}

function Detail({ email, onEmailChange }: Props) {
  return <input value={email} onChange={e => onEmailChange(e.target.value)} />
}
```

The parent controls when to reset (e.g., when the user selects a different entity, the parent does `setEmail(newEntity.email)` from the click handler — explicitly, not via effect).

Use when:

- The parent already knows when to reset.
- The state is small and not performance-sensitive (no need to memoize half the tree on every keystroke).

Don't use when:

- Lifting forces re-renders of unrelated UI.
- The child is genuinely encapsulated (modal with its own life cycle).

---

## Option 3: Conditionally mount

If the component only renders sometimes (modals, drawers, popovers), conditional mount/unmount gives you the same effect for free:

```tsx
function Parent() {
  const [editing, setEditing] = useState<Item | null>(null)
  return (
    <>
      <List onEdit={setEditing} />
      {editing && (
        <EditModal
          item={editing}
          onClose={() => setEditing(null)}
        />
      )}
    </>
  )
}
```

`EditModal` mounts when an item is selected, unmounts when closed. Internal state is fresh every time. No `key`, no syncing.

This is the best option when applicable.

---

## When you genuinely need to react to a prop change *during* the same instance

Rare. Almost always indicates a design issue. If you're sure, use the `setState`-during-render trick — *not* an effect:

```tsx
// React supports calling setState during render to adjust state when a prop changes
function Detail({ initialEmail }: { initialEmail: string }) {
  const [prevInitial, setPrevInitial] = useState(initialEmail)
  const [email, setEmail] = useState(initialEmail)

  if (initialEmail !== prevInitial) {
    setPrevInitial(initialEmail)
    setEmail(initialEmail)
  }

  return <input value={email} onChange={e => setEmail(e.target.value)} />
}
```

This is documented in the React docs ("Adjusting some state when a prop changes"). It works because React detects the same-render setState and short-circuits the render with the new value, no extra commit.

This pattern is rarely the right choice. If you're reaching for it, reconsider option 1 (`key`).

---

## A real example combining server data and a draft

The list of users (server state) + currently selected user (URL state) + email being edited (client state):

```tsx
function UserDetailPage() {
  // URL state
  const { userId } = useParams<{ userId: string }>()

  // Server state
  const { data: user, isPending } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  if (isPending) return <Skeleton />
  if (!user) return <NotFound />

  return (
    <UserEditor
      key={userId}                      // remount on user change
      initialEmail={user.email}
      onSave={(email) => updateUser({ id: userId, email })}
    />
  )
}

function UserEditor({ initialEmail, onSave }: Props) {
  const [email, setEmail] = useState(initialEmail)  // initial value only used on mount; key handles reset
  return (
    <form onSubmit={(e) => { e.preventDefault(); onSave(email) }}>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <button type="submit">Save</button>
    </form>
  )
}
```

No effects. No syncing. Selecting a different user → URL changes → query refetches → new user data → `key` changes → editor remounts with fresh state.

---

## What to look for in code review

Search for `useEffect.*setState\(.*props` patterns. Each one is a candidate for `key`, lift, or remount.

```sh
# heuristic
grep -rEn "useEffect\(.*\bset[A-Z]" src/ | grep -E "props|initial"
```

Each match deserves a comment asking "could this be a `key`?"

---

## References

- TkDodo — *Putting props to useState*
- React docs — *You Might Not Need an Effect* → "Resetting all state when a prop changes" / "Adjusting some state when a prop changes"
- Brian Vaughn — *You Probably Don't Need Derived State* (2018, still the canonical reference)
