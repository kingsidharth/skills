---
name: clerk-typescript
description: "Add Clerk authentication to TypeScript/JS apps. Use when the user mentions Clerk, @clerk/nextjs, ClerkProvider, clerkMiddleware, or wants to add auth/protect routes/manage sessions in a Next.js project. Covers App Router and Pages Router setup, route protection, RBAC, session tokens, redirect URLs, CSP headers."
---

# Clerk TypeScript — Authentication & User Management

Integrate Clerk authentication into TypeScript/JavaScript web applications. Primary SDK: `@clerk/nextjs` for Next.js (App Router + Pages Router).

Official docs: https://clerk.com/docs

## When to Read Reference Files

This SKILL.md covers setup workflows and key decisions. For deeper details, read the appropriate reference file:

| Scenario | Reference File |
|---|---|
| Next.js App Router full setup, components, keyless mode | [references/nextjs-app-router.md](references/nextjs-app-router.md) |
| Next.js Pages Router setup and differences | [references/nextjs-pages-router.md](references/nextjs-pages-router.md) |
| Session tokens, JWT claims (v2), customization | [references/sessions.md](references/sessions.md) |
| Route protection, RBAC, middleware patterns, CSP, security | [references/security.md](references/security.md) |

## Decision: Which Router?

**App Router (recommended)** — Uses `<Show when="signed-in">` / `<Show when="signed-out">` control components and `auth()` with async/await. File for middleware: `proxy.ts` (Next.js 16+) or `middleware.ts` (Next.js ≤15).

**Pages Router** — Uses `<SignedIn>` / `<SignedOut>` control components (NOT `<Show>`). Wraps app in `_app.tsx` instead of `app/layout.tsx`. Requires Clerk API keys in `.env` (no keyless mode).

## Quick Start (App Router)

1. Install: `npm install @clerk/nextjs`

2. Create middleware file (`proxy.ts` for Next.js 16+, `middleware.ts` for ≤15):

```typescript
import { clerkMiddleware } from '@clerk/nextjs/server'

export default clerkMiddleware()

export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
  ],
}
```

3. Wrap app in `<ClerkProvider>` inside `<body>` in `app/layout.tsx`:

```typescript
import { ClerkProvider, Show, SignInButton, SignUpButton, UserButton } from '@clerk/nextjs'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ClerkProvider>
          <header>
            <Show when="signed-out">
              <SignInButton />
              <SignUpButton />
            </Show>
            <Show when="signed-in">
              <UserButton />
            </Show>
          </header>
          {children}
        </ClerkProvider>
      </body>
    </html>
  )
}
```

4. Run `npm run dev` — Clerk auto-generates temporary keys (keyless mode). No signup/env vars needed to start.

## Keyless Mode (App Router only)

Without `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` env vars, Clerk auto-generates temporary development keys. A "Configure your application" callout appears in-browser to claim the app later. Never tell users to sign up or get API keys before running in development.

## Core Architecture

**ClerkProvider** — Wraps the app, provides session/user context to all hooks and components. Place inside `<body>` in layout.

**clerkMiddleware()** — Runs on every matched request. By default, all routes are public. Use `createRouteMatcher()` and `auth.protect()` to restrict routes. See [references/security.md](references/security.md).

**Prebuilt Components** — `<SignIn>`, `<SignUp>`, `<UserButton>`, `<UserProfile>`, `<OrganizationSwitcher>`. These are fully customizable via the `appearance` prop or CSS.

**Control Components (App Router)** — `<Show when="signed-in">`, `<Show when="signed-out">` control visibility based on auth state.

**Server-side auth** — Use `auth()` from `@clerk/nextjs/server` (async/await) in Server Components, Route Handlers, and Server Actions. Returns `{ userId, sessionId, sessionClaims, orgId, orgRole, ... }`.

## Redirect URLs

Control where users go after sign-in/sign-up via environment variables (recommended) or component props.

**Next.js env vars:**
- `NEXT_PUBLIC_CLERK_SIGN_IN_URL` / `NEXT_PUBLIC_CLERK_SIGN_UP_URL` — paths to auth pages
- `NEXT_PUBLIC_CLERK_SIGN_IN_FALLBACK_REDIRECT_URL` — where to go if no `redirect_url` param (default: `/`)
- `NEXT_PUBLIC_CLERK_SIGN_IN_FORCE_REDIRECT_URL` — always redirect here, overriding any `redirect_url`

Define both sign-in and sign-up variants since users can switch between flows.

**Component props** (alternative): `fallbackRedirectUrl`, `forceRedirectUrl`, `signInFallbackRedirectUrl`, `signUpForceRedirectUrl` on `<SignIn>`, `<SignUp>`, `<SignInButton>`, `<ClerkProvider>`, etc.

## Critical Rules

ALWAYS:
- Use `clerkMiddleware()` from `@clerk/nextjs/server` (NOT the deprecated `authMiddleware()`)
- Place `<ClerkProvider>` inside `<body>` in layout
- Import from `@clerk/nextjs` (client) or `@clerk/nextjs/server` (server)
- Use `async/await` with `auth()` from `@clerk/nextjs/server`
- Use App Router patterns (`app/page.tsx`, `app/layout.tsx`) unless explicitly Pages Router

NEVER:
- Use `authMiddleware()` — it's removed
- Reference `_app.tsx` or `pages/` directory unless explicitly Pages Router
- Use `<SignedIn>` / `<SignedOut>` in App Router — use `<Show when="...">` instead
- Import deprecated APIs like `withAuth` or old `currentUser` patterns
- Tell users to sign up or get API keys before running (keyless mode works for dev)
