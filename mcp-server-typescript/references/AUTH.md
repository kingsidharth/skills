# Authentication

Three tiers, from simplest to production-grade. Pick the simplest that fits your threat model.

## 1. Environment variable (local/dev)

Validate at startup, fail fast:

```typescript
const API_KEY = process.env.MY_SERVICE_API_KEY;
if (!API_KEY) {
  console.error("ERROR: MY_SERVICE_API_KEY is required");
  process.exit(1);
}
```

Good for: stdio servers, local dev, single-user. The key authenticates the *server* to an upstream API, not the MCP client to your server.

## 2. Bearer token (simple remote)

Static token checked per-request via middleware:

```typescript
const BEARER_TOKEN = process.env.MCP_AUTH_TOKEN;

app.use("/mcp", (req, res, next) => {
  const auth = req.headers.authorization;
  if (!auth || auth !== `Bearer ${BEARER_TOKEN}`) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }
  next();
});
```

Good for: internal servers behind VPN/Tailscale, small team access. Token transmitted via `Authorization: Bearer <token>` header.

## 3. OAuth 2.1 + PKCE (production remote)

Required by the MCP spec for public remote servers. Clients like Claude.ai and ChatGPT expect this flow.

### Relevant RFCs

- **RFC 9728** — OAuth 2.0 Protected Resource Metadata (`.well-known/oauth-protected-resource`)
- **RFC 8414** — Authorization Server Metadata (`.well-known/oauth-authorization-server`)
- **RFC 7591** — Dynamic Client Registration
- **RFC 8707** — Resource Indicators

### Flow overview

```
Client         Auth Server        MCP Server
  │                │                  │
  │  GET /.well-known/oauth-protected-resource
  ├─────────────────────────────────►│
  │  ◄── 401 + metadata pointer ────┤
  │                │                  │
  │  POST /register (DCR)            │
  ├───────────────►│                  │
  │  ◄── client_id ┤                  │
  │                │                  │
  │  /authorize (PKCE code flow)     │
  │◄──────────────►│                  │
  │  ◄── code ─────┤                  │
  │                │                  │
  │  POST /token   │                  │
  ├───────────────►│                  │
  │  ◄── access_token               │
  │                │                  │
  │  POST /mcp + Bearer token ─────►│
  │  ◄── 200 + result ──────────────┤
```

### Implementation options

**Option A: Use an existing identity provider** (recommended)

Auth0, Okta, Azure AD, Clerk, or any OIDC-compliant provider. Your MCP server is a *resource server* — it validates tokens, it doesn't issue them.

Server-side setup:
1. Serve Protected Resource Metadata at `/.well-known/oauth-protected-resource` pointing to your auth server's metadata URL
2. Validate bearer tokens on every request (issuer, audience, expiry, scopes)

```typescript
import { mcpAuth } from "@anthropic-ai/mcp-auth"; // or mcp-auth.dev SDK

// Mount metadata endpoint
app.use(mcpAuth.protectedResourceMetadataRouter());

// Protect MCP endpoint
app.post("/mcp",
  mcpAuth.bearerAuth("jwt", {
    audience: "https://my-mcp-server.example.com",
    requiredScopes: ["tools:read"],
  }),
  async (req, res) => { /* ... */ }
);
```

**Option B: Self-hosted auth**

You implement the authorization server endpoints yourself. Significantly more work — Dynamic Client Registration, PKCE code exchange, token issuance, token storage. Only do this if you can't use an external provider.

**Option C: Cloudflare Access as OAuth provider**

If deploying on Cloudflare Workers, Access can serve as the identity aggregator. See [REMOTE.md](REMOTE.md) for details.

### Token validation checklist

On every request to your MCP server:
- Verify JWT signature against auth server's JWKS
- Check `iss` matches expected authorization server
- Check `aud` includes your resource identifier
- Check `exp` is in the future
- Check required `scope` values
- Reject if any check fails with 401

### authInfo in tool handlers

The SDK passes validated auth context to tool handlers:

```typescript
server.registerTool(
  "my_tool",
  { /* ... */ },
  async (params, { authInfo }) => {
    // authInfo.token — the raw access token
    // authInfo.claims — decoded JWT claims (sub, scope, etc.)
    const userId = authInfo?.claims?.sub;
    // ...
  }
);
```

### Dynamic Client Registration note

The MCP spec currently requires DCR (RFC 7591). ChatGPT and Claude register a fresh client on each connection. Client Metadata Documents (CMID) are in draft and will eventually replace per-session DCR for known clients — continue supporting DCR until CMID lands.
