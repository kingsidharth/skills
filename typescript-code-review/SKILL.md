---
name: typescript-code-quality
description: Code-quality patterns and anti-patterns for TypeScript apps using React, TanStack Query, Zustand, Next.js / RSC. Use when writing new components or pages, reviewing existing code, setting up a new project, debugging re-renders, structuring data fetching, deciding between server and client components, designing UI states, choosing between WebSocket and HTTP, or anytime an `any` or `useEffect` is about to be added.
---

# typescript-code-quality

## When to invoke this skill

Invoke whenever any of the following are true:

- A new component, hook, page, or route is being written.
- An existing component is being refactored or has a bug related to state, re-renders, or fetching.
- A code review is requested ("review this", "audit this PR", "is this pattern correct?").
- A new project is being scaffolded.
- About to add `useEffect`, `any`, `as`, a `setState` chain, or a wrapper component.
- About to install an icon library, UI library, or any package adding many components.
- A websocket, polling, or live-data feature is involved.
- Choosing between server components and client components.
- A user-visible state (loading / empty / error / success) is being designed.
- Animation or navigation feels janky.

## How this skill is organised

This SKILL.md is a **table of contents**. Each linked file is a focused, agent-readable rule set with `Incorrect → Correct` examples. Read the relevant file before writing or reviewing code in its area. Multiple files frequently apply at once.

### Part 1 — Project setup and deep review

For new projects, big refactors, or whole-codebase audits.

- [`01-project-and-review/new-project-setup.md`](01-project-and-review/new-project-setup.md) — strict TypeScript, lint config, baseline dependencies, project shape.
- [`01-project-and-review/deep-code-review.md`](01-project-and-review/deep-code-review.md) — top-down audit checklist: data layer, state shape, render performance, dead code, dependency bloat.
- [`01-project-and-review/component-inventory-control.md`](01-project-and-review/component-inventory-control.md) — preventing icon-library / wrapper-component explosions (the *phosphor-react-icons added 20k components* problem).
- [`01-project-and-review/refactor-discipline.md`](01-project-and-review/refactor-discipline.md) — refactor for impact only; when not to refactor.

### Part 2 — Feature / component / page review

For per-PR review of one component, hook, or route.

- [`02-feature-review/new-component-checklist.md`](02-feature-review/new-component-checklist.md) — the "do not create `HomepageButtonCTAAnimated`" rule, plus naming, prop API, and composition checks.
- [`02-feature-review/ui-states.md`](02-feature-review/ui-states.md) — loading / empty / error / partial / success modeled as discrete branches with a shared layout.
- [`02-feature-review/data-flow-and-state-shape.md`](02-feature-review/data-flow-and-state-shape.md) — where state lives, what is server vs client vs derived vs URL state.

### Performance

- [`performance/rendering-performance.md`](performance/rendering-performance.md) — re-renders, memoisation, list keys, structural sharing, the "fix slow render before fix re-render" rule.
- [`performance/animation-performance.md`](performance/animation-performance.md) — compositor-only properties, `will-change`, View Transitions, `prefers-reduced-motion`, GPU vs CPU paint.
- [`performance/navigation-responsiveness.md`](performance/navigation-responsiveness.md) — `<Link>` vs `<a>`, prefetch, transitions, retaining current view while loading next, optimistic navigation.

### React core rules

- [`react/useeffect-discipline.md`](react/useeffect-discipline.md) — useEffect is an escape hatch; chains are almost always wrong. Maps directly to React's *You Might Not Need an Effect*.
- [`react/stale-closures-and-deps.md`](react/stale-closures-and-deps.md) — exhaustive-deps must be `error`, never lie about dependencies.
- [`react/callback-refs-over-effects.md`](react/callback-refs-over-effects.md) — DOM-side-effect on mount → callback ref, not `useRef` + `useEffect`.
- [`react/composition-over-conditionals.md`](react/composition-over-conditionals.md) — early returns + shared layout component beats a forest of `?:` inside JSX.
- [`react/props-to-state.md`](react/props-to-state.md) — never sync props to state with an effect; use `key`, lift state, or derive.

### TypeScript

- [`typescript/any-and-inference.md`](typescript/any-and-inference.md) — `any` is banned; `unknown` + narrowing or generics; let inference flow; `satisfies` over annotation.

### TanStack Query (React Query)

- [`tanstack-query/essentials.md`](tanstack-query/essentials.md) — query keys, query-fn shape, defaults, when to set `staleTime`, why `useQuery` is your state manager.
- [`tanstack-query/render-optimizations.md`](tanstack-query/render-optimizations.md) — `select`, structural sharing, tracked queries, no rest-spread.
- [`tanstack-query/derived-state.md`](tanstack-query/derived-state.md) — don't sync client state to server state, derive it.
- [`tanstack-query/retain-while-refetching.md`](tanstack-query/retain-while-refetching.md) — keep current view rendered while next data loads (`isFetching` ≠ `isPending`, `placeholderData`, `keepPreviousData`).

### Server components / Next.js

- [`server-components/server-vs-client-boundary.md`](server-components/server-vs-client-boundary.md) — what should be a Server Component, what must be Client, where to draw `'use client'`, hydration discipline, props serialization.

### State management

- [`state-management/zustand-basics.md`](state-management/zustand-basics.md) — only export custom hooks, atomic selectors, separate actions namespace, model actions as events.
- [`state-management/zustand-with-context.md`](state-management/zustand-with-context.md) — when to scope a store via React Context for initialization, testing, reusability.

### Realtime

- [`realtime/websocket-with-api-fallback.md`](realtime/websocket-with-api-fallback.md) — WebSocket primary, query/polling fallback; entity-event invalidation; `staleTime: Infinity` with WS.

---

## Global non-negotiables (apply to every file Claude writes or edits)

These override anything inferred from existing code in the repo. If existing code violates them, prefer fixing it (small, in-scope) over copying the bad pattern.

1. **No `any`.** Use `unknown` + narrowing, or generics. See `typescript/any-and-inference.md`.
2. **No `useEffect` for transformations, derivations, prop-syncing, parent-notification, or chains.** Read React's *You Might Not Need an Effect*. See `react/useeffect-discipline.md`.
3. **No `useEffect` chains** (effect that sets state that triggers another effect that sets state). Compute during render.
4. **`react-hooks/exhaustive-deps` is `error`, not `warn`.** Never silence it without a comment justifying why.
5. **Server state lives in TanStack Query (or RSC), not `useState` + `useEffect`.**
6. **One source of truth for any piece of state.** Don't duplicate server state into Zustand or `useState`. Derive instead.
7. **UI states are discrete branches with a shared `<Layout>`**, not `condA ? … : condB ? … : …` inside JSX.
8. **Don't wrap a primitive component just to set props.** A `Button` with style variants does not need `HomepageButtonCTAAnimated`. See `02-feature-review/new-component-checklist.md`.
9. **Never `import * as Icons from 'lucide-react'` or equivalent.** Import specific symbols. See `01-project-and-review/component-inventory-control.md`.
10. **Always use the framework's `<Link>` (Next, TanStack Router, React Router) for in-app nav.** Never `<a>` or `window.location`.
11. **Refactor only with measurable user or maintainer impact.** Cosmetic refactors are rejected.
12. **Prefer browser-native behaviour.** A real `<form>` over `onClick` on a div. A real `<button type="submit">` over a custom div with role="button".

---

## Reading order recommendation when first invoked

If unsure which file to read, default to:

1. This file (you are here).
2. `react/useeffect-discipline.md` — most pattern violations land here.
3. The Part-2 file matching the task (`new-component-checklist`, `ui-states`, or `data-flow-and-state-shape`).
4. The library file matching what's imported (`tanstack-query/*`, `state-management/*`, `server-components/*`).

---

## Authoritative sources this skill is built on

Cited per file. Primary:

- React docs — *You Might Not Need an Effect*, *Thinking in React*, *Rules of Hooks*
- TkDodo's blog (Dominik Dorfmeister, TanStack Query maintainer) — entire React Query series, Zustand series, useEffect series
- TanStack docs — Query, Router type-safety guide
- Vercel `agent-skills/react-best-practices` — same audience, complementary rules
- Mark Erikson — *A (Mostly) Complete Guide to React Rendering Behavior*
- Rauno Freiberg — Web Interface Guidelines
- Nadia Makarevich (DeveloperWay) — Advanced React, performance investigations
