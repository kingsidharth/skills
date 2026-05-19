# Next.js Integration

## Provider

Wrap app layout with `AuthKitProvider`:

```tsx
// app/layout.tsx
import { AuthKitProvider } from '@workos-inc/authkit-nextjs/components';

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <AuthKitProvider>{children}</AuthKitProvider>
      </body>
    </html>
  );
}
```

## Proxy (middleware)

Next.js 16 renamed middleware to proxy (`proxy.ts`). Two approaches:

### Complete (`authkitMiddleware`)

**Page-based auth** — protection determined per-page by `withAuth({ ensureSignedIn: true })`:

```ts
// proxy.ts
import { authkitMiddleware } from '@workos-inc/authkit-nextjs';

export default authkitMiddleware();
export const config = { matcher: ['/'] };
```

**Middleware auth** — all routes protected by default, exceptions via allowlist:

```ts
export default authkitMiddleware({
  middlewareAuth: {
    enabled: true,
    unauthenticatedPaths: ['/'],
  },
});
export const config = { matcher: ['/', '/account/:page*'] };
```

### Composable (`authkit`)

For apps with additional proxy logic. Returns `{ session, headers, authorizationUrl }`. You control redirect logic. Must forward `authkitHeaders` on all responses (especially `set-cookie`).

## Callback route

```ts
// app/callback/route.ts
import { handleAuth } from '@workos-inc/authkit-nextjs';
export const GET = handleAuth();
// Custom redirect: handleAuth({ returnPathname: '/dashboard' })
```

Must match `WORKOS_REDIRECT_URI` env var and dashboard redirect URI setting.

## Sign-in endpoint

```ts
// app/login/route.ts
import { getSignInUrl } from '@workos-inc/authkit-nextjs';
import { redirect } from 'next/navigation';

export const GET = async () => {
  const signInUrl = await getSignInUrl();
  return redirect(signInUrl);
};
```

Configure this endpoint in Dashboard → Redirects → Sign-in endpoint.

## Accessing auth data

**Server component** — `withAuth()`:

```tsx
import { withAuth } from '@workos-inc/authkit-nextjs';

export default async function Page() {
  const { user } = await withAuth();
  // user is null if not signed in
}
```

**Client component** — `useAuth()`:

```tsx
'use client';
import { useAuth } from '@workos-inc/authkit-nextjs/components';

export default function Page() {
  const { user, loading } = useAuth();
}
```

## Protected routes

Pass `{ ensureSignedIn: true }` — auto-redirects to AuthKit if no session:

```tsx
// Server
const { user } = await withAuth({ ensureSignedIn: true });

// Client
const { user, loading } = useAuth({ ensureSignedIn: true });
```

## Sign out

```tsx
import { signOut } from '@workos-inc/authkit-nextjs';

// In a server action:
await signOut();
```

Redirects to sign-out redirect URL configured in Dashboard → Redirects.

## Sign up URL

```tsx
import { getSignUpUrl } from '@workos-inc/authkit-nextjs';
const signUpUrl = await getSignUpUrl();
```
