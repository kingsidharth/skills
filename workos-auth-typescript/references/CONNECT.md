# WorkOS Connect (OAuth / M2M)

## OAuth Applications

For apps where the actor is a User (web, mobile, desktop, CLI). Uses `authorization_code` flow.

### First-party vs third-party

- **First-party**: apps you control (forums, support portals). No consent screen.
- **Third-party**: apps built by customers/partners. Must be associated with an Organization. Users see consent screen before identity is shared.

### Public apps

For apps that can't securely store secrets (CLI, mobile). Must use PKCE. Configure as Public during creation in Dashboard.

### Token verification

Verify access tokens against JWKS at `https://<subdomain>.authkit.app/oauth2/jwks`:

```ts
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(
  new URL('https://<subdomain>.authkit.app/oauth2/jwks')
);

const { payload } = await jwtVerify(token, JWKS, {
  issuer: 'https://<subdomain>.authkit.app',
  audience: 'client_123456789',
});
```

Also available: Token Introspection API for synchronous validity checks.

### Organization access

Multi-org users prompted to select an org during OAuth. Selected org available as `org_id` claim in access token.

## M2M Applications

For server-to-server auth without user interaction. Uses `client_credentials` flow. Separate from OAuth apps.

## Configuration

Each Connect app needs: redirect URI, name/logo (for third-party consent screen), client credentials (`client_id` + `client_secret`).

## Library support

Standard OIDC libraries work out of the box (Passport.js `openid-client`, OmniAuth, etc.). Discovery URL: `https://<subdomain>.authkit.app`.
