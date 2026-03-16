# Next.js Pages Router — Clerk Integration

Reference for `@clerk/nextjs` with Pages Router. Covers key differences from App Router.

Docs: https://clerk.com/docs/getting-started/quickstart/pages-router

## Key Differences from App Router

| Feature | App Router | Pages Router |
|---|---|---|
| Provider location | `app/layout.tsx` | `pages/_app.tsx` |
| Control components | `<Show when="signed-in">` | `<SignedIn>`, `<SignedOut>` |
| Keyless mode | Supported | **Not supported** — requires API keys |
| CSS layer | Not required | Required for Tailwind v4 |
| Server auth | `auth()` in Server Components | `getAuth(req)` in `getServerSideProps` |

## Setup

### 1. Install

```bash
npm install @clerk/nextjs
```

### 2. Set API Keys (required — no keyless mode)

Get keys from the Clerk Dashboard API Keys page:

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
```

### 3. Add Middleware

Same as App Router — create `proxy.ts` (Next.js 16+) or `middleware.ts` (≤15):

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

### 4. Wrap App in _app.tsx

```typescript
import '@/styles/globals.css'
import { ClerkProvider, SignInButton, SignedIn, SignedOut, UserButton } from '@clerk/nextjs'
import type { AppProps } from 'next/app'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <ClerkProvider
      {...pageProps}
      appearance={{
        cssLayerName: 'clerk',
      }}
    >
      <SignedOut>
        <SignInButton />
      </SignedOut>
      <SignedIn>
        <UserButton />
      </SignedIn>
      <Component {...pageProps} />
    </ClerkProvider>
  )
}

export default MyApp
```

Note: `{...pageProps}` is spread onto `ClerkProvider` — this is required for Pages Router.

### 5. Update globals.css for Tailwind v4

Add the CSS layer declaration at the top of `globals.css`:

```css
@layer theme, base, clerk, components, utilities;
@import 'tailwindcss';
```

The `cssLayerName: 'clerk'` on `<ClerkProvider>` must match the layer name in CSS. This ensures Tailwind utility styles apply after Clerk's styles.

## Control Components

Pages Router uses `<SignedIn>` and `<SignedOut>` — NOT `<Show>`:

```typescript
import { SignedIn, SignedOut, SignInButton, UserButton } from '@clerk/nextjs'

<SignedOut>
  <SignInButton />
</SignedOut>
<SignedIn>
  <UserButton />
</SignedIn>
```

## Server-Side Auth

In Pages Router, use `getAuth()` inside `getServerSideProps`:

```typescript
import { getAuth } from '@clerk/nextjs/server'
import type { GetServerSideProps } from 'next'

export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const { userId } = getAuth(ctx.req)

  if (!userId) {
    return { redirect: { destination: '/sign-in', permanent: false } }
  }

  return { props: { userId } }
}
```

For API routes:

```typescript
import { getAuth } from '@clerk/nextjs/server'
import type { NextApiRequest, NextApiResponse } from 'next'

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  const { userId } = getAuth(req)
  if (!userId) return res.status(401).json({ error: 'Unauthorized' })
  return res.json({ userId })
}
```

## Custom Sign-In/Up Pages (Pages Router)

1. Set env vars:

```env
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
```

2. Create `pages/sign-in/[[...index]].tsx`:

```typescript
import { SignIn } from '@clerk/nextjs'

export default function Page() {
  return <SignIn />
}
```

3. Create `pages/sign-up/[[...index]].tsx`:

```typescript
import { SignUp } from '@clerk/nextjs'

export default function Page() {
  return <SignUp />
}
```
