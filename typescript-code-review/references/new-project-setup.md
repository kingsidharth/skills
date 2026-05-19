# New project setup

**When this applies:** scaffolding a new TypeScript + React project, or a new app inside a monorepo.

## Required from day one

These are not optional. Add before writing any feature code.

### TypeScript

`tsconfig.json` must include:

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true
  }
}
```

Rationale: `strict` is the floor, not the ceiling. `noUncheckedIndexedAccess` catches `arr[i]` returning `T | undefined` — the single highest-leverage option for catching real bugs.

### Lint — `error` not `warn`

Two ESLint rules must be `error`:

- `react-hooks/exhaustive-deps`
- `@typescript-eslint/no-explicit-any` (allow only with inline `// eslint-disable-next-line` and a comment justifying)

Recommended additions:

- `eslint-plugin-react-you-might-not-need-an-effect` — strict config — automated detection of the patterns in `react/useeffect-discipline.md`.
- `@typescript-eslint/no-unsafe-*` family.
- `eslint-plugin-import` with `no-restricted-imports` configured to ban barrel imports of large icon / UI libraries (see `component-inventory-control.md`).

### Validation library

Pick one and use it for every external boundary (HTTP responses, env vars, form input, localStorage, URL params):

- **Zod** — default choice (Zod 4 — 7× faster, 100× fewer tsc instantiations vs v3).
- **Valibot** if bundle size is critical.
- Both implement Standard Schema, so TanStack Router/Form consume either.

### Server state

- **TanStack Query** for any client-side server-state (or framework loaders if RSC/Remix/Next).
- Configure `QueryClient` defaults explicitly. Do not accept defaults silently:

```ts
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,         // not 0; choose per-app
      gcTime: 5 * 60_000,
      retry: (failureCount, error) => /* don't retry 4xx */,
      refetchOnWindowFocus: true,
    },
  },
})
```

### Routing

- **Next.js App Router** if RSC/SSR is required and Vercel-shaped deployment is fine.
- **TanStack Router** if SPA / type-safe URL state matters (search params with schemas, fully-typed `<Link>`).
- **React Router v7** if migrating from Remix or want browser-platform-aligned forms/loaders.

Never roll a custom router. Never `<a href>` for in-app navigation.

### State management — only what you need

In order of preference:

1. **URL state** (search params via the router) — for anything shareable / bookmarkable.
2. **Server state** in TanStack Query / loaders.
3. **Local `useState`** for UI-only state (open/closed, hover, draft input).
4. **Zustand** only when state must be shared across siblings far apart in the tree.
5. **Redux Toolkit** for very large apps with complex action choreography. Not the default in 2025+.

Do not reach for Zustand by default. Most apps don't need it.

### Folder shape

Feature-folder, not type-folder.

```
src/
  features/
    invoices/
      api.ts            // queries + mutations
      schemas.ts        // zod
      use-invoice.ts    // custom hooks combining query + derived state
      InvoiceList.tsx
      InvoiceDetail.tsx
      InvoiceList.test.tsx
  components/           // primitives only — Button, Card, Input
  lib/                  // utilities, query client, fetcher
  routes/               // pages
```

Rules:

- A primitive component (`Button`, `Card`) has no business logic and no domain knowledge. It does not import from `features/`.
- A feature component imports primitives from `components/`, never the reverse.
- Cross-feature imports go through a feature's barrel only if absolutely needed.

### Bundle hygiene from day one

- Direct imports only — never `import * as Icons from 'lucide-react'`. See `component-inventory-control.md`.
- `next dynamic` / `React.lazy` for routes and any component over ~50 kB.
- Tree-shakeable libraries (`date-fns/format`, not `import _ from 'lodash'`).

### Testing

- **Vitest** + **Testing Library** for unit / component.
- **Playwright** for end-to-end / cross-browser.
- **MSW** for HTTP mocking — same handlers in tests and dev.

For React Query specifically: turn off `retry` in tests, use a fresh `QueryClient` per test. See TkDodo *Testing React Query*.

---

## What NOT to install on day one

- A CSS-in-JS runtime library (Emotion, styled-components) unless the team has a hard requirement. Tailwind + CSS Modules cover 95% of cases with zero runtime cost.
- A "form library" before you have a form. Native `<form>` + `useActionState` (React 19) handles many cases.
- Lodash / Ramda — modern JS / TS covers most needs. If specifically needed, import per-function.
- A custom design-system package as a separate repo on day one. Inline `components/` until repeated extraction is justified.
- Storybook — only after primitive components stabilize.

---

## References

- Vercel `agent-skills/react-best-practices` — bundle and waterfall rules
- TkDodo *Practical React Query*, *Working with Zustand*
- React docs — Thinking in React
