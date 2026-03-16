# Next.js App Router — Clerk Integration

Full reference for `@clerk/nextjs` with App Router.

Docs: https://clerk.com/docs/nextjs/getting-started/quickstart

## Table of Contents

1. [Installation](#installation)
2. [Middleware Setup](#middleware-setup)
3. [ClerkProvider & Layout](#clerkprovider--layout)
4. [Keyless Mode](#keyless-mode)
5. [Control Components](#control-components)
6. [Server-Side Auth](#server-side-auth)
7. [Prebuilt UI Components](#prebuilt-ui-components)
8. [Custom Sign-In/Up Pages](#custom-sign-inup-pages)
9. [Redirect URL Configuration](#redirect-url-configuration)
10. [Integration Options](#integration-options)

## Installation

```bash
npm install @clerk/nextjs
```

Use the project's existing package manager (npm, pnpm, yarn, or bun).

## Middleware Setup

Create `proxy.ts` (Next.js 16+) or `middleware.ts` (Next.js ≤15) in the project root (or `src/` if using the `src` directory). The filename changed but the code is identical.

```typescript
import { clerkMiddleware } from '@clerk/nextjs/server'

export default clerkMiddleware()

export const config = {
  matcher: [
    // Skip Next.js internals and static files
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    // Always run for API routes
    '/(api|trpc)(.*)',
  ],
}
```

By default, `clerkMiddleware()` makes all routes public. To protect routes, pass a handler:

```typescript
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

const isProtectedRoute = createRouteMatcher(['/dashboard(.*)', '/api(.*)'])

export default clerkMiddleware(async (auth, req) => {
  if (isProtectedRoute(req)) {
    await auth.protect()
  }
})

export const config = { /* same matcher */ }
```

## ClerkProvider & Layout

Wrap the entire app in `<ClerkProvider>`, placed inside `<body>`:

```typescript
import type { Metadata } from 'next'
import { ClerkProvider } from '@clerk/nextjs'
import './globals.css'

export const metadata: Metadata = {
  title: 'My App',
  description: 'Built with Clerk',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ClerkProvider>{children}</ClerkProvider>
      </body>
    </html>
  )
}
```

## Keyless Mode

For App Router apps **without** `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` in `.env`, Clerk auto-generates temporary development keys. The app works immediately — no account creation required.

A "Configure your application" callout appears in the browser. Users click it to claim the instance when ready.

When moving to production, add real keys from the Clerk Dashboard API Keys page:

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_...
CLERK_SECRET_KEY=sk_live_...
```

## Control Components

App Router uses `<Show>` for conditional rendering based on auth state:

```typescript
import { Show, SignInButton, SignUpButton, UserButton } from '@clerk/nextjs'

// In a component:
<Show when="signed-out">
  <SignInButton />
  <SignUpButton />
</Show>
<Show when="signed-in">
  <UserButton />
</Show>
```

The deprecated `<SignedIn>` and `<SignedOut>` components are replaced by `<Show>`.

## Server-Side Auth

Use `auth()` from `@clerk/nextjs/server` in Server Components, Route Handlers, and Server Actions. It is async — always `await` it.

```typescript
import { auth } from '@clerk/nextjs/server'

// In a Server Component
export default async function Dashboard() {
  const { userId, sessionClaims } = await auth()

  if (!userId) {
    return <div>Not signed in</div>
  }

  return <div>Welcome, user {userId}</div>
}
```

**Returned properties**: `userId`, `sessionId`, `sessionClaims`, `orgId`, `orgRole`, `orgSlug`, `orgPermissions`, `getToken()`, `protect()`, `redirectToSignIn()`.

Use `currentUser()` from `@clerk/nextjs/server` to get the full User object:

```typescript
import { currentUser } from '@clerk/nextjs/server'

export default async function Profile() {
  const user = await currentUser()
  return <div>Hello, {user?.firstName}</div>
}
```

## Prebuilt UI Components

All imported from `@clerk/nextjs`:

- `<SignIn />` — Full sign-in form. Embed in a page for custom sign-in.
- `<SignUp />` — Full sign-up form.
- `<UserButton />` — Avatar with dropdown for account management.
- `<UserProfile />` — Full-page user profile manager.
- `<OrganizationSwitcher />` — Switch between organizations.
- `<SignInButton />` — Unstyled link/button to sign-in (links to Account Portal by default).
- `<SignUpButton />` — Unstyled link/button to sign-up.

All support the `appearance` prop for theming:

```typescript
<SignIn
  appearance={{
    elements: {
      rootBox: 'mx-auto',
      card: 'bg-gray-50',
    },
  }}
/>
```

## Custom Sign-In/Up Pages

To host sign-in on your own pages instead of the Account Portal:

1. Set env vars:

```env
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
```

2. Create `app/sign-in/[[...sign-in]]/page.tsx`:

```typescript
import { SignIn } from '@clerk/nextjs'

export default function Page() {
  return <SignIn />
}
```

3. Create `app/sign-up/[[...sign-up]]/page.tsx`:

```typescript
import { SignUp } from '@clerk/nextjs'

export default function Page() {
  return <SignUp />
}
```

## Redirect URL Configuration

Redirects after sign-in/sign-up are controlled in priority order:

1. **Force redirect** (env var or prop) — always wins
2. **`redirect_url` query param** — set automatically when navigating from a protected page
3. **Fallback redirect** (env var or prop) — used when no `redirect_url` exists
4. **Default** — `/`

Environment variables (recommended):

```env
# Where to go after auth when no redirect_url present
NEXT_PUBLIC_CLERK_SIGN_IN_FALLBACK_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/onboarding

# Always redirect here (overrides redirect_url)
NEXT_PUBLIC_CLERK_SIGN_IN_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FORCE_REDIRECT_URL=/dashboard
```

Component props (alternative):

```typescript
<SignInButton fallbackRedirectUrl="/dashboard" />
<SignIn forceRedirectUrl="/dashboard" />
```

The deprecated `afterSignIn`, `afterSignUp`, and `redirectUrl` props are replaced by `fallbackRedirectUrl` and `forceRedirectUrl`.

## Integration Options

Clerk provides three integration levels:

1. **Account Portal** (default) — Hosted pages on Clerk servers. Zero-config, works immediately.
2. **Prebuilt components** — Embed `<SignIn />`, `<UserButton />`, etc. in your own pages. Fully customizable appearance, but fixed HTML structure and flow logic.
3. **Custom flows** — Build entirely custom UI using the Clerk API for maximum control. More development effort.

Most apps progress: Account Portal → Prebuilt Components → Custom Flows as they grow.
