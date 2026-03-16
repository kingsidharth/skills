# Security — Route Protection, RBAC, CSP

Reference for securing Clerk-powered applications: middleware-based route protection, role-based access control via metadata, and Content-Security-Policy configuration.

Docs:
- https://clerk.com/docs/guides/secure/basic-rbac
- https://clerk.com/docs/guides/secure/session-options
- https://clerk.com/docs/guides/secure/best-practices/csp-headers

## Table of Contents

1. [Protecting Routes with Middleware](#protecting-routes-with-middleware)
2. [Page-Level Protection](#page-level-protection)
3. [RBAC via publicMetadata](#rbac-via-publicmetadata)
4. [CSP Headers](#csp-headers)

## Protecting Routes with Middleware

`clerkMiddleware()` makes all routes public by default. To protect routes, use `createRouteMatcher()` and `auth.protect()`:

```typescript
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

const isProtectedRoute = createRouteMatcher([
  '/dashboard(.*)',
  '/api(.*)',
  '/admin(.*)',
])

export default clerkMiddleware(async (auth, req) => {
  if (isProtectedRoute(req)) {
    await auth.protect()
  }
})

export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
  ],
}
```

Unauthenticated users hitting a protected route are automatically redirected to sign-in.

### Role-Based Middleware Protection

Check session claims in middleware to restrict by role:

```typescript
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'
import { NextResponse } from 'next/server'

const isAdminRoute = createRouteMatcher(['/admin(.*)'])

export default clerkMiddleware(async (auth, req) => {
  if (isAdminRoute(req)) {
    const { sessionClaims } = await auth()
    if (sessionClaims?.metadata?.role !== 'admin') {
      return NextResponse.redirect(new URL('/', req.url))
    }
  }
})
```

## Page-Level Protection

Protect individual pages using the `checkRole()` helper pattern:

### 1. Define roles globally

Create `types/globals.d.ts`:

```typescript
export {}

export type Roles = 'admin' | 'moderator'

declare global {
  interface CustomJwtSessionClaims {
    metadata: {
      role?: Roles
    }
  }
}
```

### 2. Create a role-checking helper

Create `utils/roles.ts`:

```typescript
import { Roles } from '@/types/globals'
import { auth } from '@clerk/nextjs/server'

export const checkRole = async (role: Roles) => {
  const { sessionClaims } = await auth()
  return sessionClaims?.metadata.role === role
}
```

### 3. Protect the page

```typescript
import { checkRole } from '@/utils/roles'
import { redirect } from 'next/navigation'

export default async function AdminDashboard() {
  const isAdmin = await checkRole('admin')
  if (!isAdmin) {
    redirect('/')
  }

  return <p>Admin dashboard</p>
}
```

## RBAC via publicMetadata

This approach is for apps that don't use Clerk Organizations but still need role-based access. It stores roles in `publicMetadata` and exposes them through the session token.

### Setup Steps

1. **Customize session token** — In Clerk Dashboard → Sessions → Customize session token, add:

```json
{
  "metadata": "{{user.public_metadata}}"
}
```

2. **Set a user's role** — In Dashboard → Users → select user → User metadata → Public, set:

```json
{
  "role": "admin"
}
```

3. **Programmatically manage roles** — Use the Backend SDK:

```typescript
'use server'

import { checkRole } from '@/utils/roles'
import { clerkClient } from '@clerk/nextjs/server'

export async function setRole(formData: FormData) {
  const client = await clerkClient()

  if (!checkRole('admin')) {
    return { message: 'Not Authorized' }
  }

  const res = await client.users.updateUserMetadata(
    formData.get('id') as string,
    { publicMetadata: { role: formData.get('role') } }
  )
  return { message: res.publicMetadata }
}

export async function removeRole(formData: FormData) {
  const client = await clerkClient()

  const res = await client.users.updateUserMetadata(
    formData.get('id') as string,
    { publicMetadata: { role: null } }
  )
  return { message: res.publicMetadata }
}
```

Why `publicMetadata`: It's readable on the client but only writable from the server/Dashboard, making it safe for authorization decisions. `privateMetadata` is never exposed to the client. `unsafeMetadata` is writable from the client and should not be used for auth.

### For apps using Organizations

If using Clerk Organizations, use the built-in Roles & Permissions system instead of metadata RBAC. Organization roles are included in session tokens automatically via the `o.rol` and `o.per` claims.

## CSP Headers

Clerk requires specific CSP directives to function. Two approaches:

### Automatic (Next.js SDK ≥6.14.0)

Add `contentSecurityPolicy` to `clerkMiddleware()` options:

**Default mode** — applies reasonable defaults:

```typescript
export default clerkMiddleware(
  async (auth, request) => {
    if (!isPublicRoute(request)) {
      await auth.protect()
    }
  },
  {
    contentSecurityPolicy: {},
  },
)
```

Default mode sets: `connect-src 'self'` + Clerk/Stripe domains, `img-src 'self' https://img.clerk.com`, `script-src 'self' 'unsafe-inline'`, `style-src 'self' 'unsafe-inline'`, `worker-src 'self' blob:`, `frame-src 'self' https://challenges.cloudflare.com` + Stripe.

**Strict mode** — generates a unique nonce per request:

```typescript
export default clerkMiddleware(
  async (auth, request) => {
    if (!isPublicRoute(request)) {
      await auth.protect()
    }
  },
  {
    contentSecurityPolicy: {
      strict: true,
    },
  },
)
```

Strict mode requires the `dynamic` prop on `<ClerkProvider>`:

```typescript
<ClerkProvider dynamic>{children}</ClerkProvider>
```

The nonce is available via the `x-nonce` response header and passed to ClerkProvider automatically.

### Manual Configuration

If not using the automatic option, set these CSP directives:

- `script-src`: your FAPI hostname (e.g. `https://clerk.your-domain.com`) + `https://challenges.cloudflare.com`
- `connect-src`: your FAPI hostname
- `img-src`: `https://img.clerk.com`
- `worker-src`: `'self' blob:`
- `style-src`: `'unsafe-inline'` (Clerk uses runtime CSS-in-JS)
- `frame-src`: `https://challenges.cloudflare.com`

### Additional CSP Directives

Extend the automatic config with additional directives:

```typescript
export default clerkMiddleware(
  handler,
  {
    contentSecurityPolicy: {
      'script-src': ['https://my-analytics.com'],
      'connect-src': ['https://my-api.com'],
    },
  },
)
```

These merge with Clerk's defaults — they don't replace them.
