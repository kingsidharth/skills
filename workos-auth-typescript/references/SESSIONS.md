# Sessions & Tokens

## Token pair

Successful auth returns an **access token** (JWT) and a **refresh token**.

- Access token: short-lived, stored as secure cookie, validated on each request
- Refresh token: stored server-side (DB, cache, or secure httpOnly cookie). Used to get new access tokens.

## Access token JWT claims

| Claim | Description |
|-------|-------------|
| `sub` | WorkOS user ID |
| `sid` | Session ID (used for sign-out) |
| `iss` | `https://api.workos.com/` (or custom auth domain) |
| `org_id` | Selected organization (if applicable) |
| `role` | Role slug of org membership |
| `permissions` | Array of permission slugs from role |
| `entitlements` | Stripe entitlements (if Stripe add-on enabled) |
| `exp` | Expiration timestamp |
| `iat` | Issued-at timestamp |

The Next.js SDK handles token validation and refresh automatically.

For manual validation: use `jose` library, verify against JWKS at `https://api.workos.com/sso/jwks/<clientId>`.

## Refresh token rotation

Refresh tokens may be rotated after use. Always replace old token with newly returned one.

## Switching organizations

Pass `organization_id` to the refresh token endpoint. If authorized, new access token includes the target org's `org_id`, `role`, and `permissions`. If unauthorized, returns an auth error — initiate a new auth flow with `organization_id` param.

## Sign-out flow

1. Extract `sid` from access token JWT
2. Delete app session cookie
3. Redirect browser to `workos.userManagement.getLogoutUrl({ sessionId })`
4. User lands on configured sign-out redirect URL

## Dashboard configuration

- **Max session length**: absolute session expiry
- **Access token duration**: keep short for fast permission propagation
- **Inactivity timeout**: session ends if no refresh within this window
- **Sign-out redirect**: where users go after logout (set in Dashboard → Redirects)

Sign-out redirects support wildcard subdomains (`https://*.example.com`) and wildcard ports on localhost.
