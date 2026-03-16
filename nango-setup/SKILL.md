---
name: nango-integrations
description: "Build third-party API integrations using Nango — managed OAuth, authenticated proxy requests, and continuous data syncs from 600+ APIs. Use when connecting to Slack, Gmail, Google Docs, Google Calendar, Google Drive, Linear, PostHog, Metabase, or any external API where you need user auth flows, token management, data syncing, or proxy requests. Covers Auth (OAuth 2.0, API keys), Proxy (authenticated pass-through), Functions (syncs + actions), and the AI integration builder."
---

# Nango Integrations Platform

Nango handles OAuth flows, credential storage/refresh, authenticated API requests, and continuous data syncing for 600+ APIs. You write the integration logic as TypeScript Functions; Nango manages auth, retries, rate limits, and caching.

## When to read what

| Task | Read |
|------|------|
| **Any Nango work** | This file (always read first) |
| **Setting up auth, connections, frontend embed** | [AUTH.md](references/AUTH.md) |
| **Making authenticated API requests** | [PROXY.md](references/PROXY.md) |
| **Syncing data continuously from APIs** | [SYNCS.md](references/SYNCS.md) |
| **One-off API actions (write/send/create)** | [ACTIONS.md](references/ACTIONS.md) |
| **Integration configs for specific APIs** | [INTEGRATIONS.md](references/INTEGRATIONS.md) |

## Core concepts

**Integrations** — a configured API provider (e.g., `slack`, `google-mail`). Set up in the dashboard with client ID/secret and scopes.

**Connections** — one user's credentials for one integration. Created after a successful auth flow. Nango keeps tokens refreshed automatically.

**Proxy** — make authenticated API requests through Nango without handling credentials in your code. Nango injects the right tokens.

**Functions** — TypeScript code that runs on Nango's infrastructure. Two types:
- **Syncs** — run on a schedule, fetch data, cache it in Nango. You get webhooks when new data arrives.
- **Actions** — run on demand, perform a single API operation (send message, create issue, etc).

## Architecture for data ingestion

```
External APIs (Slack, Gmail, GCal, GDocs, etc.)
        │
        ▼
┌─────────────────────────────────────────────┐
│  Nango Cloud                                │
│  ┌─────────┐  ┌────────┐  ┌──────────────┐ │
│  │  Auth    │  │ Proxy  │  │  Functions    │ │
│  │ (OAuth)  │  │ (HTTP) │  │ (Syncs/Acts) │ │
│  └────┬────┘  └───┬────┘  └──────┬───────┘ │
│       │            │              │          │
│       └────────────┴──────────────┘          │
│                    │                         │
│            Records Cache (encrypted)         │
└────────────────────┬────────────────────────┘
                     │ webhook + REST API
                     ▼
              Your App (Tasuke)
```

## Quick start

### 1. Install

```bash
npm install @nangohq/node        # backend SDK
npm install @nangohq/frontend     # frontend (optional, for auth embed)
```

### 2. Initialize

```typescript
import { Nango } from '@nangohq/node';

const nango = new Nango({ secretKey: process.env.NANGO_SECRET_KEY });
```

### 3. Trigger auth (frontend)

```typescript
import Nango from '@nangohq/frontend';

const nango = new Nango({ connectSessionToken: '<SESSION-TOKEN>' });
nango.openConnectUI({
  onEvent: (event) => {
    if (event.type === 'connect') {
      // user authorized — connection is now active
      console.log(event.payload.connectionId, event.payload.providerConfigKey);
    }
  },
});
```

### 4. Get connection credentials (backend)

```typescript
const connection = await nango.getConnection('slack', 'user-123');
console.log(connection.credentials); // { access_token, refresh_token, ... }
```

### 5. Make a proxy request

```typescript
const response = await nango.proxy({
  method: 'GET',
  endpoint: '/api/conversations.list',
  providerConfigKey: 'slack',
  connectionId: 'user-123',
});
```

### 6. Fetch synced records

```typescript
const result = await nango.listRecords({
  providerConfigKey: 'google-mail',
  connectionId: 'user-123',
  model: 'GmailMessage',
});
```

## CLI essentials

```bash
npx nango init          # scaffold nango-integrations folder
npx nango dev           # watch mode — compiles functions, shows errors
npx nango dryrun <sync-name> '<connection-id>'   # test locally
npx nango deploy dev    # deploy to dev environment
npx nango deploy prod   # deploy to production
```

## Environments

Each Nango account has isolated `dev` and `prod` environments. Each has its own secret key, integrations, connections, and deployed functions. Always deploy to `dev` first, test, then promote to `prod`.

## Critical rules

1. **Never store Nango secret keys in client-side code.** The secret key grants full access to all connections. Use it only in backend/server code. For frontend auth flows, generate a short-lived connect session token from your backend.

2. **Use the Proxy instead of raw credentials whenever possible.** `nango.proxy()` handles token injection, refresh, and retries. Fetching credentials with `getConnection()` then making your own HTTP call bypasses Nango's rate-limit handling and retry logic.

3. **Synced records have retention limits.** Records not updated for 30 days get their payload pruned. Syncs not run for 60 days get all records deleted. Always fetch records promptly after webhook notifications and store them in your own database — don't use Nango's cache as your primary data store.

4. **Use cursor-based synchronization for reliability.** Webhooks can be missed. Track the `_nango_metadata.cursor` of the last-fetched record. Pass it on subsequent fetches to get only new/modified records.

5. **Use checkpoints in syncs to avoid re-fetching everything.** Save progress (last timestamp, cursor, page token) with `nango.saveCheckpoint()`. On next run, `nango.getCheckpoint()` resumes from where you left off.

6. **Match your integration ID to the Nango dashboard exactly.** The folder name under `nango-integrations/` must match the integration ID configured in the Nango dashboard (e.g., `slack`, `google-mail`, `linear`).

7. **Use `nango.batchSave()` in syncs, not individual saves.** Batch operations are significantly more efficient and required for Nango's change-detection (additions, updates, deletes) to work correctly.

## Project structure (nango-integrations folder)

```
nango-integrations/
├── .nango/
├── .env                     # NANGO_SECRET_KEY, connection IDs for testing
├── index.ts                 # imports all sync/action files
├── package.json
├── slack/
│   ├── syncs/
│   │   └── slack-messages.ts
│   └── actions/
│       └── slack-send-message.ts
├── google-mail/
│   └── syncs/
│       └── gmail-messages.ts
├── google-calendar/
│   └── syncs/
│       └── gcal-events.ts
├── google-docs/
│   └── syncs/
│       └── gdocs-documents.ts
└── linear/
    └── syncs/
        └── linear-issues.ts
```

## Security

Nango is SOC 2 Type II, GDPR, and HIPAA compliant. Credentials are encrypted at rest with AES-256-GCM. All transit uses TLS 1.2+. Environments are fully isolated.

**IP allowlist** (if your APIs need it): `52.34.139.153`, `54.69.127.183`, `44.247.133.183`, `52.26.211.56`

## AI builder

Generate sync/action code with AI agents (Claude Code, Cursor, Copilot, etc.):

1. Use Nango's MCP: `https://nango.dev/docs/mcp`
2. Install function builder skill: `npx skills add NangoHQ/skills`

The AI builder feeds Nango's interface, best practices, and test patterns to your model. It generates code with proper pagination, error handling, and rate limiting.

## Official docs

- Quickstart: https://nango.dev/docs/getting-started/quickstart
- Auth: https://nango.dev/docs/guides/primitives/auth
- Proxy: https://nango.dev/docs/guides/primitives/proxy
- Functions: https://nango.dev/docs/guides/primitives/functions
- Syncs guide: https://nango.dev/docs/implementation-guides/use-cases/syncs/implement-a-sync
- Integration templates: https://www.nango.dev/templates
- API reference: https://nango.dev/docs/reference/overview
