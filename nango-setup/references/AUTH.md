# Nango Auth

## Overview

Nango Auth handles OAuth 2.0, OAuth 1.0a, API keys, basic auth, and custom auth for 600+ APIs. You embed a white-label auth flow in your frontend; Nango stores credentials encrypted, refreshes tokens automatically, and makes them available via API/SDK.

## Flow

```
Your Frontend                    Your Backend                  Nango
    │                                │                            │
    │  1. User clicks "Connect"      │                            │
    │─────────────────────────────▶  │                            │
    │                                │  2. Create connect session  │
    │                                │─────────────────────────▶  │
    │                                │  ◀── session token ────────│
    │  ◀── session token ───────────│                            │
    │                                │                            │
    │  3. Open ConnectUI             │                            │
    │────────────────────────────────────────────────────────────▶│
    │                                │     4. OAuth dance          │
    │  ◀── onEvent(connect) ─────────────────────────────────────│
    │                                │                            │
    │                                │  5. Fetch connection creds  │
    │                                │─────────────────────────▶  │
    │                                │  ◀── tokens ──────────────│
```

## Backend: create connect session

Generate a short-lived session token (30 min) from your backend before opening the auth UI:

```typescript
import { Nango } from '@nangohq/node';

const nango = new Nango({ secretKey: process.env.NANGO_SECRET_KEY });

// Create session — typically in an API route
const session = await nango.createConnectSession({
  end_user: {
    id: 'user-123',           // your internal user ID
    email: 'user@example.com', // optional
    display_name: 'Jane Doe',  // optional
  },
  allowed_integrations: ['slack', 'google-mail', 'google-calendar'],
});

// Return session.token to your frontend
```

## Frontend: open ConnectUI

```typescript
import Nango from '@nangohq/frontend';

const nango = new Nango({ connectSessionToken: sessionToken });

nango.openConnectUI({
  onEvent: (event) => {
    switch (event.type) {
      case 'connect':
        // Authorization succeeded
        const { connectionId, providerConfigKey } = event.payload;
        // Save to your DB, fetch data, etc.
        break;
      case 'close':
        // User closed the modal
        break;
    }
  },
});
```

The ConnectUI is fully white-labeled — it matches your branding. It handles provider-specific guidance, error states, and retry flows.

## Backend: retrieve credentials

```typescript
// Get full connection details including credentials
const connection = await nango.getConnection(
  'slack',       // integration ID (providerConfigKey)
  'user-123'     // connection ID
);

// connection.credentials contains:
// - access_token (for OAuth)
// - refresh_token (for OAuth, if applicable)
// - api_key (for API key auth)
// - raw (provider-specific fields)

// Check connection metadata
console.log(connection.connection_config); // scopes, subdomain, etc.
```

## Backend: list all connections

```typescript
const connections = await nango.listConnections();
// Returns all connections across all integrations

// Filter by integration
const slackConnections = connections.connections.filter(
  c => c.provider_config_key === 'slack'
);
```

## Backend: delete a connection

```typescript
await nango.deleteConnection('slack', 'user-123');
// Soft-deleted immediately, hard-deleted after 31 days
```

## cURL equivalents

```bash
# Get connection
curl -X GET \
  -H "Authorization: Bearer $NANGO_SECRET_KEY" \
  "https://api.nango.dev/connection/user-123?provider_config_key=slack"

# List connections
curl -X GET \
  -H "Authorization: Bearer $NANGO_SECRET_KEY" \
  "https://api.nango.dev/connections"

# Delete connection
curl -X DELETE \
  -H "Authorization: Bearer $NANGO_SECRET_KEY" \
  "https://api.nango.dev/connection/user-123?provider_config_key=slack"
```

## Supported auth methods

| Method | How Nango handles it |
|--------|---------------------|
| OAuth 2.0 | Full flow + automatic token refresh |
| OAuth 1.0a | Full flow (Twitter, etc.) |
| API Key | Stored encrypted, injected on proxy calls |
| Basic Auth | Username/password stored, injected as header |
| Custom | Provider-specific schemes |

## Credential lifecycle

- Tokens are refreshed **automatically** before expiry
- Failed refreshes trigger alerts in the Nango dashboard
- If refresh fails permanently, the connection enters a "needs reconnection" state
- Your app receives a webhook on credential failure — prompt the user to re-auth

## Key gotchas

- The `connectSessionToken` expires in 30 minutes. Generate fresh ones per auth attempt.
- `connectionId` defaults to the end user's `id` if not explicitly set. Use your internal user ID for predictable lookups.
- Changing OAuth scopes on an existing integration **does not** update existing connections. Users must re-authorize to get new scopes.
- Store the `connectionId` and `providerConfigKey` in your database — you need both to retrieve credentials or make proxy calls.
