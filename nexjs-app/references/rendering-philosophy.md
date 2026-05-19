# Rendering Philosophy (Mental Model)

Why Next.js is shaped the way it is. Read this once to understand why the APIs look like they do.

## The axis: where the static/dynamic boundary lives

Most frameworks put the boundary at the **route** level — a page is static or dynamic, all or nothing.

Next.js puts the boundary at the **component** level. A single page can have:

- A static shell (prerendered, on a CDN)
- A cached function that revalidates independently (`'use cache'` + tags)
- A dynamic section that streams at request time

This is what Partial Prerendering + Cache Components + on-demand revalidation enable together. Not three separate features — one rendering model.

## What the model enables

- **Faster perceived load.** Shell renders instantly; dynamic content streams in behind it. No all-or-nothing wait.
- **Incremental caching.** No up-front commitment ("is this route static or dynamic?"). Cache what makes sense, revalidate what changes.
- **Granular invalidation.** Tag a function, not a route. Invalidate a tag, not a deployment. An expensive query can be cached independently of the rest of the page.

## The trade-off

Finer-grained rendering moves complexity from app code into the hosting platform. In return for less code in your app, the platform has to:

| Requirement | Why |
|---|---|
| **Streaming** | Static and dynamic content arrive in one response, progressively. |
| **Cache coordination (multi-instance)** | `revalidateTag` on one box must reach the others. |
| **Cache consistency** | HTML and RSC payload must stay in sync when regenerated. |
| **PPR shell at CDN latency** | Serving the static shell from the edge while resuming dynamic render at origin often needs platform integration. |

Platforms that deploy Next.js as a plain Node.js process satisfy "functional fidelity" (every feature works). Performance fidelity (shell at edge, sub-second ISR) is a spectrum. The [adapter test suite](https://nextjs.org/docs/app/api-reference/adapters/testing-adapters) is the pass/fail contract.

## Alternative rendering models (for context)

**Build-time prerendering (SSG).** Every page generated at build. Pure CDN deploy. Simple. Any content change = full rebuild + redeploy.

**Route-level boundaries.** Each route picks static or dynamic. Static → CDN; dynamic → server. Simple mental model, but a mostly-static page with one dynamic element (greeting, price) must either be fully dynamic or fetch that element client-side after load.

**Component-level boundaries (Next.js).** The same route delivers both. The cost is infrastructure complexity; the payoff is no client-side waterfalls for small amounts of dynamic content.

## Practical implications

- **Authoring**: put the dynamic thing (`cookies()`, `headers()`, `searchParams`, uncached `fetch`) behind `<Suspense>`. The fallback prerenders; the content streams.
- **Caching**: don't preemptively decide "this route is static." Let components declare `'use cache'` locally.
- **Invalidation**: tag on the way in (`cacheTag`), invalidate on the way out (`updateTag`/`revalidateTag`).
- **Deployment**: the more of the above you use, the more infrastructure matters. If you're self-hosting multi-instance, you own cache coordination (shared handler with `refreshTags`). On verified adapters (Vercel, Bun), it's handled.

## CDN compatibility note

Many CDNs have the primitives (edge compute, KV, blob storage) to support deep Next.js integration, but end-to-end PPR resume is still emerging. Most community adapters today deploy Next.js as a Node.js server without leveraging CDN-specific primitives — you get functional fidelity, less than peak performance fidelity. This will improve over time.
