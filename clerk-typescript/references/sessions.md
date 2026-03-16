# Session Tokens & JWT Claims

Reference for Clerk session tokens, JWT claims (v2), and customization.

Docs: https://clerk.com/docs/guides/sessions/session-tokens

## Table of Contents

1. [How Sessions Work](#how-sessions-work)
2. [Default Claims (v2)](#default-claims-v2)
3. [Organization Claims](#organization-claims)
4. [Actor Claim (Impersonation)](#actor-claim-impersonation)
5. [Customizing Session Tokens](#customizing-session-tokens)
6. [Size Limitations](#size-limitations)
7. [Session Lifetime](#session-lifetime)
8. [Multi-Session Apps](#multi-session-apps)

## How Sessions Work

When a user authenticates, Clerk generates a short-lived JWT session token. This token authenticates backend requests and contains user/session information.

The token is refreshed automatically by Clerk's frontend SDK. A user is considered "inactive" when the application is closed or stops refreshing.

Key objects in the lifecycle:
- **Client** — the current device/browser
- **Session** — secure auth state of the current user; one Client can hold multiple Sessions
- **User** — the authenticated user with their profile, metadata, and identifiers

## Default Claims (v2)

Version 2 tokens (current) include these claims:

| Claim | Meaning | Example |
|---|---|---|
| `sub` | User ID | `user_123` |
| `sid` | Session ID | `sess_123` |
| `iss` | Issuer (Frontend API URL) | `https://clerk.your-site.com` |
| `exp` | Expiration (Unix timestamp) | `1713158400` |
| `iat` | Issued at (Unix timestamp) | `1713158400` |
| `nbf` | Not valid before (Unix timestamp) | `1713158400` |
| `jti` | Unique token ID | `1234567890` |
| `azp` | Authorized party (Origin header of request) | `https://example.com` |
| `fva` | Factor verification age `[first, second]` in minutes | `[7, -1]` |
| `v` | Token version | `2` |
| `sts` | Session status | `pending` |
| `pla` | Active Plan (`scope:planslug`) | `u:free` or `o:pro` |
| `fea` | Enabled Features with scope | `o:dashboard,o:impersonation` |

The `fva` claim is an array: first element = minutes since first-factor verification, second = minutes since second-factor (or `-1` if no second factor).

## Organization Claims

When the user has an active Organization, the `o` claim is included:

| Claim | Meaning | Example |
|---|---|---|
| `o.id` | Organization ID | `org_123` |
| `o.slg` | Organization slug | `org-slug` |
| `o.rol` | User's role (without `org:` prefix) | `admin` |
| `o.per` | Permissions (comma-separated) | `read,manage` |
| `o.fpm` | Feature-permission bitmask map | `3,2` |

The `o.fpm` is a comma-separated list of integers. Each integer maps to the Feature at the same index in `fea`. When converted to binary, bits correspond to permissions in `o.per` (right to left). Use an SDK to decode this — manual decoding is complex.

**Example**: `fea: o:dashboard,o:teams`, `o.per: manage,read`, `o.fpm: 3,2`
- `3` → binary `11` → dashboard has both `manage` and `read`
- `2` → binary `10` → teams has only `read` (second bit from right)

## Actor Claim (Impersonation)

When an admin impersonates a user, the `act` claim appears:

| Claim | Meaning | Example |
|---|---|---|
| `act.iss` | Referrer | `https://dashboard.clerk.com` |
| `act.sid` | Impersonated session ID | `sess_456` |
| `act.sub` | Impersonator user ID | `user_456` |

## Customizing Session Tokens

Add custom claims via the Clerk Dashboard:

1. Navigate to **Sessions** page
2. Under **Customize session token**, add claims in the Claims editor

Example — expose public metadata in the token:

```json
{
  "metadata": "{{user.public_metadata}}"
}
```

This makes metadata available in `sessionClaims.metadata` without a network request.

Create separate JWT templates for third-party integrations (Supabase, Hasura, etc.) via the Dashboard's **JWT Templates** page.

## Size Limitations

Session tokens are sent with every request, so keep them small. Avoid putting large objects in `publicMetadata` if it's included in the session token. The token is a JWT — excessively large tokens can hit cookie size limits or slow requests.

## Session Lifetime

Configured in Clerk Dashboard under **Sessions**:

**Inactivity timeout** — Session expires after this duration of inactivity (app closed / token not refreshing). Disabled by default. Paid feature in production.

**Maximum lifetime** — Session expires after this duration regardless of activity. Default: 7 days. Custom values require a paid plan in production.

At least one of these must be enabled. Browser cookie limits (Chrome: 400-day max `Max-Age`) may cause sign-out before configured lifetime.

## Multi-Session Apps

Allow multiple accounts signed in simultaneously from the same browser. Enable in Dashboard under **Sessions** → **Multi-session handling**.

For multi-session support in your app, use `<UserButton />` (prebuilt) or wrap the app with a key-based remounting component:

```typescript
function MultisessionAppSupport({ children }: { children: React.ReactNode }) {
  const { session } = useSession()
  return <React.Fragment key={session ? session.id : 'no-users'}>{children}</React.Fragment>
}
```

This forces a full React tree re-render when the active session changes, ensuring all components reflect the correct user.
