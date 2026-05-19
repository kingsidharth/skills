# Navigation responsiveness

**When this applies:** any in-app navigation — clicking a link, submitting a form, switching tabs, going back. The single highest-leverage UX dimension after first paint.

The principle: **navigation should feel instant**. Anything > 100 ms perceptibly delayed; > 300 ms feels broken. Modern frameworks make this easy; the bug is usually that AI-generated code skips the framework's primitives.

---

## Rule 1: Always use the framework's `<Link>`

```tsx
// 🚨 plain anchor — full page reload, no prefetch, no client navigation
<a href="/dashboard">Dashboard</a>

// 🚨 button + imperative push — same problem, plus no right-click / cmd-click / middle-click
<button onClick={() => router.push('/dashboard')}>Dashboard</button>

// ✅
<Link to="/dashboard">Dashboard</Link>      // TanStack Router / React Router
<Link href="/dashboard">Dashboard</Link>    // Next.js
```

The framework's `Link`:

- Renders a real `<a>` (right-click works, cmd-click opens new tab, screen readers announce as link).
- Intercepts the click for client-side navigation.
- Prefetches on hover/focus (free perceived speed).
- Coordinates with the loader for data prefetching.

AI-generated code reflexively reaches for `<a>` or `router.push` in `onClick`. Reject this pattern in review.

---

## Rule 2: Prefetch on hover/focus

Default behaviour for most framework Links — verify it's enabled.

| Framework | Default |
|---|---|
| Next.js | Prefetches on viewport (default) |
| TanStack Router | `preload="intent"` recommended |
| React Router 7 | `prefetch="intent"` |

Intent-based prefetch (hover or focus) is the sweet spot — fires only when the user signals interest.

For data behind a route, prefetch the *query* on hover too:

```tsx
<Link
  to={`/products/${id}`}
  onMouseEnter={() => queryClient.prefetchQuery(productQueries.detail(id))}
  onFocus={() =>     queryClient.prefetchQuery(productQueries.detail(id))}
>
  ...
</Link>
```

By the time the user clicks, the data is in cache. Click → instant render.

---

## Rule 3: Retain current view while loading next

Default browser behaviour: navigate → blank → next page. Bad. With router-level transitions, you can:

- Keep the current page rendered.
- Show a subtle pending indicator.
- Swap to the new page when its data is ready.

For React Router 7 / Remix: `useNavigation()` exposes the pending state.

```tsx
function Layout({ children }: Props) {
  const navigation = useNavigation()
  const isNavigating = navigation.state !== 'idle'
  return (
    <>
      {isNavigating && <TopProgressBar />}
      <main className={isNavigating ? 'opacity-90' : ''}>{children}</main>
    </>
  )
}
```

For TanStack Router: similar API via `useRouterState`.

For Next.js App Router: `loading.tsx` files give the pending UI; React 18 `useTransition` handles client-side pending.

For SPA without a router-level pending mechanism: wrap navigation in `useTransition`:

```tsx
const [isPending, startTransition] = useTransition()
const navigate = useNavigate()

const onClick = () => startTransition(() => navigate('/next'))
```

The previous page stays visible during the transition; the new page mounts when its Suspense boundaries resolve.

---

## Rule 4: Use `<form action>` for mutations

```tsx
// 🚨 onClick + fetch + setState + navigate — recreates the whole stack manually
<button onClick={async () => {
  setLoading(true)
  await fetch('/api/save', { method: 'POST', body: JSON.stringify(data) })
  setLoading(false)
  navigate('/list')
}}>Save</button>

// ✅ Server Action (RSC / Next 14+ / React Router 7+)
<form action={saveAction}>
  <input name="title" />
  <button type="submit">Save</button>
</form>
```

The form action approach gives you, for free:

- Progressive enhancement (works without JS).
- Browser-native pending state and double-submission protection.
- Automatic revalidation of the affected data.
- Correct `enctype` handling for file uploads.

For client-side feedback, pair with React 19 `useFormStatus` and `useActionState`:

```tsx
function SubmitButton() {
  const { pending } = useFormStatus()
  return <button type="submit" disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>
}
```

---

## Rule 5: URL is the source of truth for shareable state

Filters, sort, search, tab, page number, currently-open modal — all should be in the URL. This isn't just for sharing; it makes back-button work correctly, makes refresh non-destructive, and lets the route system prefetch.

```tsx
// ✅ TanStack Router with typed search params
const Route = createFileRoute('/invoices')({
  validateSearch: z.object({
    filter: z.enum(['all', 'paid', 'unpaid']).default('all'),
    sort: z.enum(['date', 'amount']).default('date'),
    page: z.number().int().positive().default(1),
  }),
  loader: ({ deps }) => fetchInvoices(deps),
  loaderDeps: ({ search }) => search,
})
```

Changing search params → loader runs again with new args → data updates → page re-renders. No client-side `useState` for the filter; no effect to sync URL.

For Next.js: `useSearchParams` + `useRouter().push` (or `Link`).
For SPA without typed search: `nuqs` provides a typed `useQueryState`.

---

## Rule 6: Don't navigate inside a `useEffect`

```tsx
// 🚨 effect-driven navigation
useEffect(() => {
  if (user.role === 'admin') navigate('/admin')
}, [user])
```

Problems: fires on remount, flickers the wrong page first, races with other navigations.

Either:

- Decide where to navigate **on the server** (RSC redirect / loader redirect).
- Decide **at the moment of the trigger** (event handler).
- Use a route guard that runs before the page renders.

```tsx
// ✅ in a route loader (TanStack Router)
loader: ({ context }) => {
  if (context.user.role !== 'admin') throw redirect({ to: '/' })
  return fetchAdminData()
}
```

```tsx
// ✅ in the event that caused the state
const onLogin = async (creds) => {
  const user = await login(creds)
  navigate(user.role === 'admin' ? '/admin' : '/')
}
```

---

## Rule 7: Back/forward must work

If the user clicks Back, they should land on what they were looking at. Test this on every page:

- Filters and search remained set.
- Scroll position remembered (most routers do this; verify).
- Form drafts didn't get wiped (use `key`, not effects, for resets).
- The page didn't refetch from scratch.

If something breaks the back button, your state is in the wrong place — usually local state that should be in the URL.

---

## Rule 8: Avoid navigation jank

Common culprits:

- **Layout shift on route load** — set `min-height` on layout containers; reserve space for late-loading content (images, ads).
- **Scroll restored too early or too late** — rely on framework default; don't override unless needed.
- **Heavy synchronous work on mount** — see `performance/rendering-performance.md`. Defer with `useTransition` or split components.
- **Bundle for the new route is huge** — code-split routes; check per-route bundle size in production build.

---

## Rule 9: Avoid full reloads in SPAs

Things that cause unintended full reloads:

- `<a href>` instead of `<Link>` (above).
- `window.location.assign(...)` / `window.location.href = ...`.
- Returning a hard redirect from a client-side navigation when a soft redirect would work.
- Third-party widgets that overwrite `<a>` defaults (test).

Audit your codebase: `grep -rEn "window\.location" src/` — any non-test match deserves a comment justifying it.

---

## Rule 10: Native primitives when possible

For navigation-adjacent UI: use the platform.

- **Anchor link to in-page section** — `<a href="#section">`. Browser handles scroll, focus, and hash.
- **Forward/backward buttons** — `<button onClick={() => history.back()}>` (with care; usually the framework gives you a back link in the layout).
- **Open in new tab** — let users do it themselves with cmd-click; don't `target="_blank"` unless there's a strong reason. If you must, add `rel="noopener noreferrer"`.

---

## Quick checklist

```
- [ ] Every in-app nav uses framework <Link>, never <a> or window.location
- [ ] Prefetch on hover/focus enabled at the router and data layer
- [ ] Navigation transitions retain previous content + show pending indicator
- [ ] Mutations are <form action> where possible, not onClick + fetch
- [ ] Filters/sort/page/search are in the URL
- [ ] No <useEffect> for navigation decisions
- [ ] Back button works correctly
- [ ] No window.location.* (except in narrow auth-callback scenarios)
```

---

## References

- Ryan Florence — *When To Fetch* (Reactathon talk; the foundation for loader-based navigation)
- TanStack Router — type-safe `<Link>`, search-param schemas
- React Router 7 — `useNavigation`, `prefetch`, `viewTransition`
- Next.js — App Router `<Link>`, `loading.tsx`, View Transitions
- Vercel `agent-skills/react-best-practices` — `bundle-preload`, navigation rules
