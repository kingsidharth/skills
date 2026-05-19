# Performance

Lazy loading, bundle analysis, memory, UI state preservation, view transitions, scripts. One stop for everything perf-related.

## Lazy loading (Client Components)

Server Components are auto-code-split. Lazy loading applies to **Client Components** and imported libraries.

### `next/dynamic`

Composite of `React.lazy` + `Suspense`:

```tsx
'use client'
import { useState } from 'react'
import dynamic from 'next/dynamic'

const ComponentA = dynamic(() => import('../components/A'))
const ComponentB = dynamic(() => import('../components/B'), {
  loading: () => <p>Loading...</p>,
})
const ComponentC = dynamic(() => import('../components/C'), { ssr: false })
```

- `{ ssr: false }` — Client Component only, no prerender. Must live inside a Client Component (error if used in a Server Component).
- Named exports: `dynamic(() => import('./mod').then((m) => m.Named))`.

### Importing Server Components dynamically

Works, but only Client Components inside the Server Component become lazy. The Server Component itself stays in the RSC Payload.

### Loading external libraries on demand

```tsx
'use client'
export default function Search() {
  return (
    <input onChange={async (e) => {
      const Fuse = (await import('fuse.js')).default
      // use it
    }} />
  )
}
```

### Magic comments

Only work on **dynamic** `import()`, `require()`, `require.resolve()`, `new Worker()`. Not on static `import x from 'y'`.

| Comment | Effect |
|---|---|
| `/* webpackIgnore: true */` | Skip bundling (Webpack). Import happens at runtime. |
| `/* turbopackIgnore: true */` | Skip bundling (Turbopack). |
| `/* turbopackOptional: true */` | Don't error at build if the module is missing. Throws at runtime if missing. |

```js
const runtime = await import(/* webpackIgnore: true */ 'runtime-mod')
const feature = await import(/* turbopackOptional: true */ './optional')
```

`webpackOptional` is **not supported**. Use `turbopackOptional`.

## Bundle analysis

### Turbopack Bundle Analyzer (v16.1+, experimental)

```bash
bun next experimental-analyze
```

Interactive treemap in the browser with filtering by route/environment/type and full import-chain tracing. Save for diffing:

```bash
bun next experimental-analyze --output
# writes to .next/diagnostics/analyze
```

### `@next/bundle-analyzer` (Webpack)

```bash
bun add @next/bundle-analyzer
```

```ts
// next.config.ts
import bundleAnalyzer from '@next/bundle-analyzer'

const withBundleAnalyzer = bundleAnalyzer({ enabled: process.env.ANALYZE === 'true' })

export default withBundleAnalyzer({ /* your config */ })
```

```bash
ANALYZE=true bun run build
```

## Optimizing large bundles

### Packages with many exports

Icon libraries, utility libraries with hundreds of modules:

```ts
experimental: { optimizePackageImports: ['icon-library'] }
```

Only used modules are loaded. Some packages are auto-optimized without being listed.

### Heavy client workloads

Expensive rendering in Client Components ships the library to the browser. Move to Server Components when browser APIs aren't needed:

```tsx
// BAD: prism ships to client
'use client'
import Highlight from 'prism-react-renderer'
```

```tsx
// GOOD: shiki runs on server, client gets plain HTML
import { codeToHtml } from 'shiki'

export default async function Page() {
  const html = await codeToHtml(code, { lang: 'tsx', theme: 'github-dark' })
  return <pre><code dangerouslySetInnerHTML={{ __html: html }} /></pre>
}
```

### Opting packages out of server bundling

```ts
serverExternalPackages: ['package-name']
```

Resolved via native `require` at runtime instead of being bundled.

## Memory usage

If builds or dev runs out of memory:

| Option | When to reach for it |
|---|---|
| `experimental.webpackMemoryOptimizations: true` | First thing to try; low risk |
| `next build --experimental-debug-memory-usage` | Live memory + GC stats during build |
| `node --heap-prof node_modules/next/dist/bin/next build` | Record a heap profile for Chrome DevTools |
| `NODE_OPTIONS=--inspect next build` | Attach inspector; `SIGUSR2` to snapshot |
| `experimental.webpackBuildWorker: true` | Run Webpack in a worker (auto-enabled w/o custom config) |
| Disable Webpack disk cache (via `webpack` callback) | Reduce steady-state memory |
| `typescript.ignoreBuildErrors: true` | If typecheck is the OOM source — run it separately in CI |
| `productionBrowserSourceMaps: false`, `experimental.serverSourceMaps: false`, `experimental.enablePrerenderSourceMaps: false` | Skip source map generation |
| `experimental.preloadEntriesOnStart: false` | Smaller initial server memory; modules load on demand |

## Preserving UI state across navigations (Cache Components)

With Cache Components, Next.js hides pages using React `<Activity>` instead of unmounting them. Up to 3 routes preserved. State (React state + DOM: form drafts, scroll, `<details>`, video position) survives navigation.

### Choosing what to preserve

Default is preservation. Decide per-component whether to reset.

**Reset dropdown-style open/close** — close in `useLayoutEffect` cleanup:

```tsx
useLayoutEffect(() => () => setIsOpen(false), [])
```

Or close immediately on link click via `Link`'s `onNavigate`.

**Dialog with init logic (focus, fetch)** — don't derive state from component state; derive from `searchParams`:

```tsx
const searchParams = useSearchParams()
const isOpen = searchParams.get('edit') === 'true'
useEffect(() => { if (isOpen) inputRef.current?.focus() }, [isOpen])
```

Navigation changes the URL, so `isOpen` truly changes and the effect re-runs.

**Reset form inputs on hide** — callback ref cleanup:

```tsx
<form ref={(form) => () => form?.reset()}>
```

**`useActionState` — reset stale success messages** — add a `RESET` action dispatched in `useLayoutEffect` cleanup:

```tsx
useLayoutEffect(() => () => {
  if (shouldReset.current) {
    shouldReset.current = false
    startTransition(() => dispatch({ type: 'RESET' }))
  }
}, [dispatch])
```

**Global styles that shouldn't leak when hidden** — toggle `media`:

```tsx
<style ref={(s) => { if (s) s.media = ''; return () => { if (s) s.media = 'not all' } }}>
  {`:root { --page-accent: blue; }`}
</style>
```

**Logout** — use `window.location.href` (full reload) to clear all client state.

**User-scoped form reset** — `<Form key={userId} />` lets React handle the reset.

### Testing gotcha

Hidden `<Activity>` content has `display: none` but is still in the DOM. Use visibility-aware selectors in E2E:

```ts
// Playwright — good; filters by accessibility tree visibility
await page.getByRole('button', { name: 'Submit' }).click()

// Explicit
await page.locator('.card').filter({ visible: true }).first().click()
```

### `<Activity>` directly — prerendering hidden content

Start fetching in the Server Component, pass the promise to a Client Component, wrap in `<Activity>` + `<Suspense>`:

```tsx
// server
const commentsPromise = getCommentsData()
return <ExpandableComments commentsPromise={commentsPromise} />
```

```tsx
// client
'use client'
import { Activity, Suspense, useState, use } from 'react'
export function ExpandableComments({ commentsPromise }: { commentsPromise: Promise<Comment[]> }) {
  const [expanded, setExpanded] = useState(false)
  return (
    <>
      <button onClick={() => setExpanded((e) => !e)}>{expanded ? 'Hide' : 'Show'}</button>
      <Activity mode={expanded ? 'visible' : 'hidden'}>
        <Suspense fallback={<Skeleton />}>
          <Comments commentsPromise={commentsPromise} />
        </Suspense>
      </Activity>
    </>
  )
}
```

### Media cleanup

`display: none` doesn't stop `<video>`/`<audio>`. Cleanup in `useLayoutEffect`:

```tsx
useLayoutEffect(() => () => videoRef.current?.pause(), [])
```

Playback position is preserved since the DOM node isn't removed.

### First mount vs re-show

Effects run on every hide→visible transition. Distinguish with a ref:

```tsx
const hasMounted = useRef(false)
useEffect(() => {
  if (!hasMounted.current) { hasMounted.current = true; /* first-mount logic */ }
  else { /* re-show logic */ }
}, [])
```

## Building public pages (PPR in practice)

Public pages (landing pages, product pages) share data across users — prerender them. Flow:

1. **Start static** — pure components, no data fetching.
2. **Add data that's shared across users** — wrap with `'use cache'`. Still prerenders.
3. **Add user-specific parts** — can't cache. Wrap in `<Suspense>`. The fallback prerenders; the content streams at request time.

Build output markers:
- `○ (Static)` — fully prerendered
- `◐ (Partial Prerender)` — shell prerendered, dynamic holes streamed
- `ƒ (Dynamic)` — rendered on demand

See [caching.md](caching.md) for the `use cache` / `cacheLife` / `cacheTag` details.

## View transitions (React `<ViewTransition>`)

Enable:

```ts
experimental: { viewTransition: true }
```

Import from React:

```tsx
import { ViewTransition } from 'react'
```

Route navigations are React transitions, so view transitions activate automatically.

### Four patterns

| Pattern | What it says | Mechanism |
|---|---|---|
| Shared element morph | "Same thing, going deeper" | `<ViewTransition name="photo-123">` with matching name on both routes |
| Suspense reveal | "Data loaded" | `<ViewTransition enter="slide-up">` around content, `exit="slide-down"` around fallback |
| Directional slide | "Going forward/back" | `<Link transitionTypes={['nav-forward']}>` + `<ViewTransition enter={{'nav-forward': 'nav-forward', default: 'none'}}>` |
| Same-route crossfade | "Same place, different content" | `<ViewTransition key={slug} share="auto" enter="auto">` — `key` change triggers |

### CSS pseudo-elements

```css
::view-transition-old(.slide-up) { /* leaving */ }
::view-transition-new(.slide-up) { /* arriving */ }
::view-transition-group(.slide-up) { animation-duration: 400ms; }
```

Use asymmetric timing: fast exit, slightly slower enter delayed to match.

### Anchor stable UI (e.g. header)

```tsx
<header style={{ viewTransitionName: 'site-header' }}>
```

```css
::view-transition-group(site-header) { animation: none; z-index: 100; }
::view-transition-old(site-header) { display: none; }
::view-transition-new(site-header) { animation: none; }
```

### Reduced motion

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*), ::view-transition-new(*), ::view-transition-group(*) {
    animation-duration: 0s !important;
    animation-delay: 0s !important;
  }
}
```

### Browser-initiated back

Swipe/back button don't carry `transitionTypes`, so directional slides won't play. Shared-element morph still works if names match.

## Scripts — `next/script`

Load third-party scripts with strategy control:

```tsx
import Script from 'next/script'

export default function Layout({ children }: LayoutProps<'/'>) {
  return (
    <html><body>
      {children}
      <Script src="https://example.com/script.js" />
    </body></html>
  )
}
```

Scripts in a layout load once across that layout's routes — no re-execution on nav within the layout.

### Strategies

| Strategy | Behavior |
|---|---|
| `beforeInteractive` | Load before any Next.js code, before hydration. Use for critical scripts (polyfills). |
| `afterInteractive` (default) | Load early but after some hydration. Most analytics/marketing scripts. |
| `lazyOnload` | Browser idle time. Low-priority widgets. |
| `worker` (experimental) | Partytown web worker. Doesn't yet work with App Router. |

### Inline scripts

Must have `id`:

```tsx
<Script id="show-banner">
  {`document.getElementById('banner').classList.remove('hidden')`}
</Script>
```

### Event handlers (Client Components only)

```tsx
'use client'
<Script src="..." onLoad={() => { /* ... */ }} onError={() => { /* ... */ }} />
```

`onLoad`, `onReady`, `onError` require `'use client'`.

### Additional attributes

Forwarded automatically:

```tsx
<Script src="..." id="x" nonce="XUENAJFW" data-test="script" />
```

Critical when combining with CSP — pass the nonce from `headers().get('x-nonce')`.
