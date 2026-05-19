# Deployment

Deploying, self-hosting, static exports, production checklist.

## Deployment targets

| Target | All features? | Notes |
|---|---|---|
| Node.js server (`next start`) | Yes | The reference runtime. |
| Docker container | Yes | Use `output: 'standalone'` for a minimal image. |
| Static export (`output: 'export'`) | Limited | Pure HTML/CSS/JS. No server features. |
| Verified adapters (Vercel, Bun) | Yes | Pass the adapter test suite. |
| Other adapters (Appwrite, AWS Amplify, Cloudflare, Deno Deploy, Firebase, Netlify) | Partial | Feature set varies by adapter. |

"Functional fidelity" = adapter passes the test suite, every feature works. "Performance fidelity" (PPR shell at CDN latency, sub-second ISR propagation) is a spectrum and depends on platform architecture.

## Self-hosting on Node

### Reverse proxy

Put nginx (or similar) in front. Handles malformed requests, slow-loris, body size limits, rate limiting. Lets the Next.js server focus on rendering.

### Image optimization

Works zero-config with `next start`. On glibc Linux, `sharp` may need memory-allocator tuning. Configure TTL with `images.minimumCacheTTL`. Use `{ unoptimized: true }` to opt out entirely.

### Proxy (formerly middleware)

Works zero-config with `next start`. Runs on the Edge runtime by default.

### Runtime env vars

```tsx
import { connection } from 'next/server'
export default async function Component() {
  await connection()                       // opts into dynamic rendering
  const value = process.env.MY_VALUE       // evaluated at runtime
}
```

Enables a single Docker image promoted across environments.

### Caching

In-memory + on-disk by default (50MB memory cap). Works for single persistent instance. For ephemeral compute or multi-instance:

```ts
// next.config.ts
cacheHandler: require.resolve('./cache-handler.js'),
cacheMaxMemorySize: 0,  // disable in-memory; rely on shared store
```

Skeleton cache handler:

```js
// cache-handler.js
const cache = new Map()

module.exports = class CacheHandler {
  async get(key) { return cache.get(key) }
  async set(key, data, ctx) {
    cache.set(key, { value: data, lastModified: Date.now(), tags: ctx.tags })
  }
  async revalidateTag(tags) {
    tags = [tags].flat()
    for (const [key, value] of cache) {
      if (value.tags?.some((t) => tags.includes(t))) cache.delete(key)
    }
  }
  async refreshTags() {
    // Multi-instance: sync tag state from shared store here
  }
  resetRequestCache() {}
}
```

Swap `new Map()` for Redis/KV/S3 in production.

### Build ID across instances

```ts
// next.config.ts
generateBuildId: async () => process.env.GIT_HASH,
```

All containers running the same commit must emit the same ID.

## Multi-instance deployments

Three things to set:

1. **`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`** — same base64-encoded AES key across instances. Otherwise actions encrypted by one instance can't be decrypted by another.

   ```bash
   NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=$(openssl rand -base64 32) next build
   ```

2. **`deploymentId`** — version skew protection. Static assets get `?dpl=<id>`; nav requests send `x-deployment-id`. Mismatch → hard navigation instead of client-side.

   ```ts
   deploymentId: process.env.DEPLOYMENT_VERSION,
   ```

3. **Shared cache + `refreshTags`** — `revalidateTag()` only affects the instance it ran on. For coordination, implement `refreshTags()` in the cache handler (called before every request). Syncs from Redis/similar.

Client-side state is lost on hard navigation. Design for it (URL state, localStorage).

## Streaming behind a proxy

Streaming requires end-to-end chunked transfer. nginx buffers by default:

```ts
// next.config.ts
async headers() {
  return [{
    source: '/:path*{/}?',
    headers: [{ key: 'X-Accel-Buffering', value: 'no' }],
  }]
},
```

Also check:
- Load balancers must pass through chunked or HTTP/2 streaming (AWS ALB + Lambda sometimes buffers).
- Reverse proxies between LB and app must not buffer.
- PPR requires streaming. Without it, static shell + dynamic content arrive together and you lose the TTFB advantage.

## `after()` — graceful shutdown

`after()` (post-response background work) is fully supported on `next start`. On shutdown:

- Send `SIGINT`/`SIGTERM` to the server
- Wait 10–30s drain period before `SIGKILL`
- Server finishes in-flight requests and runs pending `after` callbacks

## Docker

`output: 'standalone'` in `next.config.ts` generates `.next/standalone` with a minimal `server.js`:

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

FROM deps AS builder
COPY . .
RUN bun run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

## Static export (`output: 'export'`)

For pure-static / SPA deployments. Generates `out/` with HTML + assets.

### Supported

- Server Components that run at build time (no runtime data)
- Client Components + SWR for client-side data
- `next/image` with a **custom loader** only
- Route Handlers with `GET` only, no request reads

### Unsupported

- Dynamic routes with `dynamicParams: true`
- Dynamic routes without `generateStaticParams()`
- Route Handlers that read request
- `cookies()`, `headers()`
- `rewrites`, `redirects`, `headers` config
- `proxy.ts`
- Incremental Static Regeneration (ISR)
- Default image optimization loader
- Draft Mode
- Server Actions
- Intercepting Routes

### nginx for static export

```nginx
server {
  listen 80;
  root /var/www/out;

  location / { try_files $uri $uri.html $uri/ =404; }
  location /blog/ { rewrite ^/blog/(.*)$ /blog/$1.html break; }  # if trailingSlash: false

  error_page 404 /404.html;
}
```

## Production checklist

### Already automatic

Server Components by default, route-segment code splitting, viewport prefetching, prerendering, caching. Don't disable without reason.

### During dev

- Use `<Link>` for all internal nav
- Place `'use client'` as deep as possible
- Wrap request-time APIs (`cookies`/`headers`/`searchParams`) in `<Suspense>`
- Fetch in Server Components; Route Handlers only for client-reachable data
- Parallel fetches with `Promise.all`; stream with `loading.tsx` + `<Suspense>`
- Cache with `'use cache'` + `cacheTag`
- Static images in `public/` (or static imports for layout shift prevention)
- Forms → Server Actions; validate server-side
- Add `error.tsx`, `not-found.tsx`, `global-error.tsx`, `global-not-found.tsx`
- `<Image>`, `next/font`, `<Script>`
- Metadata API + OG images + sitemap + robots
- ESLint with `eslint-plugin-jsx-a11y`
- TypeScript + plugin
- `experimental.taint` if you want the extra layer
- `.gitignore` for `.env*.local`, only `NEXT_PUBLIC_*` for client data
- CSP in place

### Before going to prod

- `next build` locally — catch build errors
- `next start` — test production behavior
- Lighthouse (incognito) + field data via `useReportWebVitals`
- Bundle analysis (Turbopack or `@next/bundle-analyzer`)
- Optionally: Import Cost, Package Phobia, Bundle Phobia for new deps
