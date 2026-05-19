# Deep code review

**When this applies:** auditing an existing codebase, onboarding to a new repo, or doing a quarterly tech-debt review. Not for per-PR review (that's `02-feature-review/`).

The goal is to surface *high-impact* problems — things that affect users (perf, bugs) or maintainers at scale (bloat, unsafe types). Cosmetic findings are noise.

---

## The audit, in priority order

Walk these sections top-to-bottom. Stop spending time on a section once it's clean.

### 1. Type safety floor

Run, and report counts:

- `grep -rn ": any\b" src/ | wc -l` — every `any` must be justified. Replace with `unknown` + narrowing or generics.
- `grep -rn " as " src/ | wc -l` — every `as` is a type lie. Distinguish unavoidable (`as const`, `as HTMLInputElement` in a known DOM context) from genuine lies.
- `tsc --noEmit` with strict on — every error counts as a finding.
- `npx type-coverage --detail` — aim for ≥99%.

Also check `tsconfig.json` for the strictness floor in `new-project-setup.md`. Missing flags = findings.

### 2. `useEffect` audit

`grep -rn "useEffect" src/` and classify each occurrence:

| Bucket | Disposition |
|---|---|
| Subscribing to external store / DOM API / browser event | OK — keep |
| Setting up WebSocket / EventSource / IntersectionObserver | OK — keep |
| Effect → setState (no external system involved) | **Bug** — convert to derive-during-render or event handler |
| Chains (effect → setState → another effect runs) | **Bug** — collapse |
| `useEffect(() => fetch(...), [])` | **Bug** — convert to TanStack Query / loader |
| Syncing prop → state | **Bug** — use `key` prop, lift state, or derive |
| Notifying parent of state change | **Bug** — lift state up |

See `react/useeffect-discipline.md` for replacement patterns. Most React projects have a 60–80% reduction available here.

### 3. Server-state duplication

For every `useEffect` paired with `setState` and a `fetch`, that data should be in TanStack Query or a route loader. Also check for:

- Server data copied into Zustand stores.
- Server data copied into `useState` and "kept in sync" with effects.
- Multiple `useQuery` hooks for the same key with different `queryFn` shapes (consolidate into a query factory; see `tanstack-query/essentials.md`).

### 4. Component inventory

Run a count: `find src -name "*.tsx" | wc -l`.

Sanity check against complexity. Surface symptoms:

- `HomepageButtonCTA`, `DashboardHeaderTitle`, `LoginPagePrimaryButton` — single-use wrappers around a primitive that exist only to set props or styles. Inline them. See `02-feature-review/new-component-checklist.md`.
- Two components with > 70% identical JSX — consolidate.
- A `components/` folder that imports from `features/` — inversion; primitives must be feature-agnostic.
- Icon imports in module-graph form (`import * as Icons from 'lucide-react'`) — see `component-inventory-control.md`.

### 5. Bundle audit

Run a bundle analyzer (`@next/bundle-analyzer`, `vite-bundle-visualizer`, `source-map-explorer`).

Findings checklist:

- Top 10 largest modules — any obvious overkill (moment.js, full lodash, an icon library shipping every glyph).
- Barrel imports loading entire libraries — `import { Cog } from 'lucide-react'` is fine; `import { Cog } from '@radix-ui/react-icons'` may not be.
- Duplicate copies of a library at different versions (`npm ls react`).
- Per-route bundle: anything ≫ 200 kB gzip on a non-marketing route is a finding.

### 6. Re-render hot spots

Open React DevTools Profiler. Record an interaction (typing, clicking, navigating). Findings:

- Components rendering with identical props commit-after-commit → memoization or context split.
- A `Provider` whose value is recreated every render → wrap in `useMemo`.
- A list where every row re-renders on a single-row change → check `key`, lift filter state, atomic selectors.
- A page where typing in one input re-renders an unrelated chart → state should be local, not lifted.

See `performance/rendering-performance.md`.

### 7. State shape

For each top-level page / feature, list the state:

- What is server state (in Query / loader)?
- What is URL state (search params)?
- What is client state (Zustand / useState)?
- What is **derived** (and is it actually being derived, not stored)?

Anything that *can* be derived but is being stored is a finding. Anything that should be in the URL but is in component state breaks share/back/refresh — also a finding.

See `02-feature-review/data-flow-and-state-shape.md`.

### 8. Loading / empty / error states

For every component fetching data:

- Is there a distinct loading state?
- A distinct empty state (data fetched, but nothing to show)?
- A distinct error state with a retry path?
- Does a refetch *retain the current view* (not flash to skeleton)?

Mixing these into one conditional inside JSX is a finding. See `02-feature-review/ui-states.md`.

### 9. Navigation and links

- All in-app navigation uses the framework's `<Link>` — never `<a href>` or `window.location` or `router.push` from a `<button onClick>` that should have been a link.
- Forms use real `<form action={...}>` (or `<Form>` from React Router / Remix) — not `onClick` on a div.
- Button-like things use `<button type="button">`, not `<div onClick>`.

These are accessibility issues *and* perf issues (no prefetch, full reloads).

### 10. Server / client boundary (RSC / Next App Router)

If the project uses Server Components:

- Every `'use client'` is at the deepest possible point (a leaf, not a layout).
- No data is double-fetched (server + client useQuery).
- Props serialized across the boundary contain only what the client uses (no full DB rows).
- No shared module-level mutable state.

See `server-components/server-vs-client-boundary.md`.

### 11. WebSocket / live data

If real-time features exist:

- WebSocket is *primary*; HTTP queries / polling are *fallback* — not the other way around.
- WebSocket events drive `queryClient.invalidateQueries` (or `setQueryData` for partial updates) — not a parallel state store.
- `staleTime` is high (often `Infinity`) for WS-backed queries.

See `realtime/websocket-with-api-fallback.md`.

### 12. Animation and navigation perf

- Animations only drive `transform` / `opacity` / `filter` (compositor-only). Never `width`, `height`, `top`, `left`, `margin` for animated transitions.
- `prefers-reduced-motion` respected.
- Route transitions use View Transitions API (Next 14+ / React Router 7+) where available.

See `performance/animation-performance.md` and `performance/navigation-responsiveness.md`.

---

## Output format

When this audit is run by an agent, the output should be a markdown report grouped by section, with:

- Section name
- Finding count
- Top 5 findings (file:line, summary, severity)
- Recommended fix (link to the relevant file in this skill)

Skip sections with zero findings.

---

## What is NOT a finding

To keep signal high, do not report:

- Style preferences (semicolons, single vs double quote, file naming).
- "Could be more functional" / "could use a different state lib".
- Any refactor without measurable user or maintainer impact (see `refactor-discipline.md`).
- Missing tests on stable code that hasn't changed in 6+ months.

---

## References

- Vercel `agent-skills/react-best-practices/AGENTS.md` — overlapping audit categories
- Mark Erikson — *State of React 2025*
- TkDodo — *Refactor impactfully*
