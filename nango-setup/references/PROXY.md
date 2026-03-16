# Nango Proxy

## Overview

The Proxy lets you make authenticated API requests to external APIs without handling credentials in your code. Nango injects the correct tokens/keys, handles refresh, retries on rate limits, and logs all requests.

## When to use Proxy vs Syncs

| Use Proxy when... | Use Syncs when... |
|---|---|
| One-off reads or writes | Continuously keeping data in sync |
| Real-time responses needed | Background polling is fine |
| Simple API calls | Complex multi-page fetches |
| Actions (send, create, update) | Data ingestion (list, fetch all) |

## Basic usage (Node SDK)

```typescript
import { Nango } from '@nangohq/node';

const nango = new Nango({ secretKey: process.env.NANGO_SECRET_KEY });

// GET request
const response = await nango.proxy({
  method: 'GET',
  endpoint: '/api/conversations.list',
  providerConfigKey: 'slack',
  connectionId: 'user-123',
  params: { limit: '100' },
});

// POST request
const postResponse = await nango.proxy({
  method: 'POST',
  endpoint: '/api/chat.postMessage',
  providerConfigKey: 'slack',
  connectionId: 'user-123',
  data: {
    channel: 'C01234567',
    text: 'Hello from Nango!',
  },
});

// Response has standard shape
console.log(response.data);    // parsed JSON body
console.log(response.status);  // HTTP status
console.log(response.headers); // response headers
```

## cURL equivalent

```bash
# Nango proxies the request, injecting auth headers automatically
curl -X GET \
  -H "Authorization: Bearer $NANGO_SECRET_KEY" \
  -H "Provider-Config-Key: slack" \
  -H "Connection-Id: user-123" \
  "https://api.nango.dev/proxy/api/conversations.list?limit=100"

# POST with body
curl -X POST \
  -H "Authorization: Bearer $NANGO_SECRET_KEY" \
  -H "Provider-Config-Key: slack" \
  -H "Connection-Id: user-123" \
  -H "Content-Type: application/json" \
  -d '{"channel":"C01234567","text":"Hello"}' \
  "https://api.nango.dev/proxy/api/chat.postMessage"
```

## Inside Functions (Syncs & Actions)

Within a Nango Function, use the `nango` helper directly — no need to specify connection or integration:

```typescript
// Inside a sync or action exec() method
const response = await nango.get({
  endpoint: '/v1/messages',
  params: { limit: '50' },
});

const postResult = await nango.post({
  endpoint: '/v1/messages',
  data: { content: 'Hello' },
});

// Also available: nango.put(), nango.patch(), nango.delete()
```

## Proxy options

```typescript
await nango.proxy({
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE',
  endpoint: '/api/path',           // relative to provider's base URL
  providerConfigKey: 'integration-id',
  connectionId: 'connection-id',
  params: { key: 'value' },        // query parameters
  data: { key: 'value' },          // request body (POST/PUT/PATCH)
  headers: { 'X-Custom': 'val' },  // additional headers (merged with auth)
  retries: 3,                       // retry count on failure
  baseUrlOverride: 'https://...',   // override provider's base URL
});
```

## What Nango injects automatically

- **OAuth 2.0**: `Authorization: Bearer <access_token>` header
- **API Key**: header or query param depending on provider config
- **Basic Auth**: `Authorization: Basic <base64>` header
- Nango refreshes expired OAuth tokens before the request — your code never sees a 401 from token expiry

## Rate limit handling

Nango reads provider-specific rate limit headers and automatically retries with backoff. Inside Functions, `nango.get()` / `nango.post()` etc. handle this transparently.

## Pagination inside Functions

```typescript
// Use nango.paginate() for automatic pagination
for await (const page of nango.paginate({
  endpoint: '/api/users.list',
  params: { limit: '200' },
  paginate: {
    type: 'cursor',
    cursor_path_in_response: 'response_metadata.next_cursor',
    cursor_name_in_request: 'cursor',
    limit_name_in_request: 'limit',
  },
})) {
  await nango.batchSave(mapRecords(page), 'SlackUser');
}
```

## Key gotchas

- The `endpoint` is **relative** to the provider's configured base URL. Don't include `https://slack.com` — just `/api/conversations.list`.
- For GraphQL APIs (like Linear), POST to the GraphQL endpoint with the query in `data`.
- Proxy requests from your backend use your Nango secret key. Proxy requests from inside Functions use the connection context automatically.
