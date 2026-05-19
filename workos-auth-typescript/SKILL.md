---
name: workos-auth-typescript
description: WorkOS AuthKit authentication — Next.js integration, sessions, RBAC roles/permissions, organizations, SSO, invitations, OAuth Connect, Actions webhooks, Stripe add-on. Use when adding auth with WorkOS, authkit-nextjs, or enterprise SSO/SCIM.
---

# WorkOS AuthKit (TypeScript / Next.js)

WorkOS AuthKit is a hosted authentication platform. SDK: `@workos-inc/authkit-nextjs`. Auth flow is redirect-based — user goes to AuthKit hosted UI, returns with an authorization code exchanged for session tokens.

## Quick setup

```bash
npm install @workos-inc/authkit-nextjs
```

```env
WORKOS_API_KEY=sk_...
WORKOS_CLIENT_ID=client_...
WORKOS_COOKIE_PASSWORD="<32+ char password>"
NEXT_PUBLIC_WORKOS_REDIRECT_URI="http://localhost:3000/callback"
```

CLI alternative: `npx workos@latest` — auto-detects framework, installs SDK, writes integration code.

## Integration essentials

See [references/NEXTJS-INTEGRATION.md](references/NEXTJS-INTEGRATION.md) for:
- `AuthKitProvider` wrapper, proxy/middleware setup (page-based vs middleware auth)
- Callback route (`handleAuth`), sign-in endpoint (`getSignInUrl`)
- `withAuth` (server) / `useAuth` (client) for accessing user data
- `ensureSignedIn` for protected routes, `signOut` for ending sessions

## Data model & multi-tenancy

See [references/MODELING.md](references/MODELING.md) for:
- Users, Organizations, Organization Memberships
- B2C (flat users) vs B2B (org-scoped) patterns
- Membership lifecycle: `pending` → `active` → `inactive`
- JIT provisioning, domain verification, Admin Portal SSO setup

## Sessions & tokens

See [references/SESSIONS.md](references/SESSIONS.md) for:
- Access token (JWT) claims: `sub`, `sid`, `org_id`, `role`, `permissions`, `entitlements`
- Refresh token rotation, org switching via refresh
- Sign-out flow (extract `sid`, delete cookie, redirect to logout endpoint)
- Dashboard config: max session length, access token duration, inactivity timeout

## Roles, permissions & RBAC

See [references/RBAC.md](references/RBAC.md) for:
- Environment-level roles + org-level custom roles
- Permission slug conventions (`resource:action`)
- Single vs multiple roles per membership (union of permissions)
- Role assignment via SCIM directory groups or SSO groups
- Role-aware sessions (JWT `role` + `permissions` claims)

## Enterprise auth (SSO)

SSO configured per-organization via Admin Portal. Auth methods: Email+Password, OAuth/Social, Magic Auth (6-digit code), SSO (SAML/OIDC), Passkeys, MFA. Identity linking is automatic by email. Domain verification auto-verifies SSO users.

## Invitations

See [references/INVITATIONS.md](references/INVITATIONS.md) for:
- Invite-to-org and app-wide invitations
- Invite-only signup (disable public signup, invitations bypass)
- Programmatic invitations via API, email acceptance rules

## Actions (webhooks)

See [references/ACTIONS.md](references/ACTIONS.md) for:
- Authentication and user-registration action hooks
- Signature verification (`workos-signature` header)
- Allow/deny responses for custom gating logic

## WorkOS Connect (OAuth/M2M)

See [references/CONNECT.md](references/CONNECT.md) for:
- First-party vs third-party OAuth applications
- Public apps with PKCE, token verification via JWKS
- M2M applications for server-to-server auth

## Stripe add-on

See [references/STRIPE.md](references/STRIPE.md) for:
- Stripe Entitlements in access tokens (feature gating by subscription)
- Stripe Seat Sync (org member count → Stripe billing meter)
- Setup: connect Stripe, set `stripeCustomerId` on organizations
