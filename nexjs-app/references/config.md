# Configuration — `next.config.ts`, TypeScript, `src/`

## `next.config.ts`

Prefer `.ts` for type-checked config:

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
  typedRoutes: true,
}

export default nextConfig
```

Variants:
- `next.config.js` with `module.exports` — classic, works everywhere
- `next.config.mjs` with `export default` — ESM in CommonJS projects
- `next.config.ts` — TypeScript (v15+). CommonJS resolution by default; ESM with Node 22.10+ native TS resolver

`.cjs` and `.cts` are **not** supported.

Function form for phase-specific configs:

```ts
import { PHASE_DEVELOPMENT_SERVER } from 'next/constants'

export default (phase, { defaultConfig }) => {
  if (phase === PHASE_DEVELOPMENT_SERVER) return { /* dev-only */ }
  return { /* non-dev */ }
}
```

## Common config options

Grouped by frequency of use. Full list at `/docs/app/api-reference/config/next-config-js`.

### Routing/rendering

| Option | Purpose |
|---|---|
| `cacheComponents: true` | Enable Cache Components + PPR |
| `typedRoutes: true` | Typed `<Link href>` and router methods |
| `basePath: '/docs'` | Serve the app under a path prefix |
| `trailingSlash: true` | Normalize URLs with trailing slash |
| `output: 'standalone'` \| `'export'` | Standalone Docker output; static site export |
| `redirects()`, `rewrites()`, `headers()` | Path-based routing/header rules |
| `experimental.viewTransition: true` | Enable React `<ViewTransition>` |

### Images

```ts
images: {
  remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com', pathname: '/**' }],
  minimumCacheTTL: 60,
}
```

### Bundling

| Option | Purpose |
|---|---|
| `optimizePackageImports: ['icon-lib']` | Only load used exports from large packages |
| `serverExternalPackages: ['sharp']` | Don't bundle a package into Server Component output (use native require) |
| `transpilePackages: ['@my/internal']` | Transpile otherwise-untranspiled deps (monorepo) |
| `turbopack: {...}` | Turbopack-specific config |

### Logging

```ts
logging: { fetches: { fullUrl: true } }
```

Shows cached/uncached status for every `fetch` in dev.

### Security

| Option | Purpose |
|---|---|
| `experimental.taint: true` | Enable React Taint APIs |
| `experimental.serverActions.allowedOrigins: [...]` | CSRF allowlist for Server Actions behind proxy |
| `poweredByHeader: false` | Remove `x-powered-by` |

### Output / deployment

| Option | Purpose |
|---|---|
| `output: 'standalone'` | Minimal Node runtime in `.next/standalone` (for Docker) |
| `output: 'export'` | Generate `out/` static HTML (SPA-like) |
| `assetPrefix: 'https://cdn...'` | Serve static assets from a CDN |
| `deploymentId: process.env.DEPLOYMENT_VERSION` | Version skew protection |
| `generateBuildId: async () => process.env.GIT_HASH` | Consistent build IDs across containers |
| `cacheHandler: require.resolve('./cache-handler.js')` | Shared cache across instances |

## TypeScript

Next.js auto-installs TS on first run with a `.ts`/`.tsx` file. `next dev` generates `tsconfig.json` and `next-env.d.ts`.

### Global route-aware types

Generated during `next dev` / `next build` / `next typegen`:

- `PageProps<'/blog/[slug]'>` — `{ params: Promise<{ slug: string }>, searchParams: Promise<...> }`
- `LayoutProps<'/dashboard'>` — `{ children: ReactNode, /* named slots typed here */ }`
- `RouteContext<'/users/[id]'>` — `{ params: Promise<{ id: string }> }`

No imports needed. Static routes resolve to `{}` for params.

### IDE plugin

Enable in VS Code: command palette → "TypeScript: Select TypeScript Version" → "Use Workspace Version". Provides:
- Warnings on invalid segment config values
- `'use client'` directive validation
- Checks that client hooks aren't used in Server Components

### Typed routes

`typedRoutes: true` in config. Requires TS.

```tsx
import type { Route } from 'next'
<Link href="/blog" />                    // valid
<Link href="/aboot" />                   // TS error
<Link href={('/blog/' + slug) as Route} /> // cast for non-literals
```

Also types `router.push`, `router.replace`, `router.prefetch`.

Ensure `.next/types/**/*.ts` is in `tsconfig.json`'s `include`.

### Typed env vars (experimental)

```ts
experimental: { typedEnv: true }
```

Generates types from `.env*` files for `process.env` IntelliSense.

### Custom `tsconfig` path for builds

```ts
typescript: { tsconfigPath: 'tsconfig.build.json' }
```

Useful in monorepos to relax checks during production builds while keeping IDE strict.

### Skipping typecheck in build

```ts
typescript: { ignoreBuildErrors: true }
```

Dangerous. Only useful if you run typecheck separately in CI.

### Custom `.d.ts`

`next-env.d.ts` is overwritten on every build. Make a `new-types.d.ts` instead and add to `tsconfig.json` `include`.

## `src/` folder

Optional pattern: move `app/` (and `pages/`, if used) into `src/app/`.

Rules:
- `public/` stays in project root
- `package.json`, `next.config.ts`, `tsconfig.json` stay in project root
- `.env*` stays in project root
- If both `/app` and `/src/app` exist, root wins
- Put `proxy.ts` inside `src/`
- Tailwind: add `src/` to `content` in `tailwind.config.js`
- TS path aliases (`@/*`): update `paths` in `tsconfig.json` to `src/*`

## ESLint

`create-next-app` installs ESLint with `eslint-config-next`. Includes `eslint-plugin-jsx-a11y` for accessibility checks. Config lives in `eslint.config.mjs`.
