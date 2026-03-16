# Nango Syncs

## Overview

Syncs continuously fetch data from external APIs and cache it in Nango. You write the fetch logic as a TypeScript Function; Nango handles scheduling, retries, rate limits, and change detection. Your app receives webhooks when new data arrives and fetches records via API/SDK.

## When to use syncs

- Store a copy of external API data and keep it up to date
- Detect changes in APIs that don't offer webhooks
- Combine polling + webhooks for reliable real-time streams
- Feed data into RAG pipelines, search indexes, or local databases

## Sync file scaffold

```typescript
import { createSync } from 'nango';
import * as z from 'zod';

// Define your data model with Zod
const SlackMessage = z.object({
  id: z.string(),
  channel_id: z.string(),
  user_id: z.string(),
  text: z.string(),
  ts: z.string(),
  thread_ts: z.string().nullable(),
});

export default createSync({
  description: 'Fetches messages from Slack channels',
  version: '1.0.0',
  endpoints: [{ method: 'GET', path: '/slack/messages', group: 'Messages' }],
  frequency: 'every 15 minutes',  // minimum: 'every 15 seconds'
  autoStart: true,                 // start when new connection is created
  trackDeletes: false,             // detect deleted records

  // Optional: enable incremental syncing
  checkpoint: z.object({
    lastTimestamp: z.string(),
  }),

  models: {
    SlackMessage: SlackMessage,
  },

  exec: async (nango) => {
    const checkpoint = await nango.getCheckpoint();

    // Fetch data from API (Nango handles auth automatically)
    const response = await nango.get({
      endpoint: '/api/conversations.history',
      params: {
        channel: 'C01234567',
        oldest: checkpoint?.lastTimestamp || '0',
      },
    });

    const records = response.data.messages.map((msg: any) => ({
      id: msg.ts,
      channel_id: 'C01234567',
      user_id: msg.user,
      text: msg.text,
      ts: msg.ts,
      thread_ts: msg.thread_ts || null,
    }));

    // Save to Nango's cache — triggers change detection
    await nango.batchSave(records, 'SlackMessage');

    // Save progress for next run
    if (records.length > 0) {
      await nango.saveCheckpoint({
        lastTimestamp: records[records.length - 1].ts,
      });
    }

    await nango.log('Sync completed', { count: records.length });
  },
});
```

## File structure

```
nango-integrations/
├── index.ts                          # must import all sync files
├── slack/
│   └── syncs/
│       └── slack-messages.ts         # folder name = integration ID
├── google-mail/
│   └── syncs/
│       └── gmail-messages.ts
```

Register in `index.ts`:
```typescript
import './slack/syncs/slack-messages';
import './google-mail/syncs/gmail-messages';
```

## Checkpoints (incremental syncing)

Without checkpoints: every run fetches the entire dataset. Fine for small datasets.

With checkpoints: save a cursor/timestamp, resume from there next run. Required for large datasets.

```typescript
// First run: checkpoint is null
const checkpoint = await nango.getCheckpoint();

if (checkpoint) {
  // Incremental: only fetch since last sync
  query += ` WHERE modified_at > '${checkpoint.lastModified}'`;
}

// After processing, save progress
await nango.saveCheckpoint({ lastModified: latestRecord.modified_at });
```

## Pagination

```typescript
// Cursor-based pagination (Slack, many REST APIs)
for await (const page of nango.paginate({
  endpoint: '/api/conversations.list',
  params: { limit: '200' },
  paginate: {
    type: 'cursor',
    cursor_path_in_response: 'response_metadata.next_cursor',
    cursor_name_in_request: 'cursor',
    limit_name_in_request: 'limit',
  },
})) {
  const mapped = page.map(mapChannel);
  await nango.batchSave(mapped, 'SlackChannel');
}

// Offset-based pagination
for await (const page of nango.paginate({
  endpoint: '/api/v2/users',
  params: { per_page: '100' },
  paginate: {
    type: 'offset',
    offset_name_in_request: 'page',
    limit_name_in_request: 'per_page',
    response_path: 'users',
  },
})) {
  await nango.batchSave(page.map(mapUser), 'User');
}

// Link-based pagination (GitHub style)
for await (const page of nango.paginate({
  endpoint: '/repos/owner/repo/issues',
  params: { per_page: '100' },
  paginate: {
    type: 'link',
    link_rel_in_response_header: 'next',
    limit_name_in_request: 'per_page',
  },
})) {
  await nango.batchSave(page.map(mapIssue), 'Issue');
}
```

## Consuming synced data (your app)

### Step 1: Set up webhooks

Configure a webhook URL in the Nango dashboard. Nango sends a POST when a sync completes:

```json
{
  "type": "sync",
  "connectionId": "user-123",
  "providerConfigKey": "slack",
  "syncName": "slack-messages",
  "model": "SlackMessage",
  "modifiedAfter": "2025-01-15T10:30:00Z"
}
```

### Step 2: Fetch records

```typescript
// After receiving webhook
const result = await nango.listRecords({
  providerConfigKey: 'slack',
  connectionId: 'user-123',
  model: 'SlackMessage',
  modifiedAfter: webhookPayload.modifiedAfter, // only new/changed records
});

// Each record includes metadata
result.records.forEach(record => {
  console.log(record.id);
  console.log(record._nango_metadata.last_action);   // 'ADDED' | 'UPDATED' | 'DELETED'
  console.log(record._nango_metadata.cursor);          // for cursor-based sync
  console.log(record._nango_metadata.first_seen_at);
  console.log(record._nango_metadata.last_modified_at);
});
```

### Cursor-based fetching (recommended)

More reliable than relying on webhook timestamps alone:

```typescript
// Store the cursor of the last record you processed
let lastCursor = await db.get('nango_cursor:slack:user-123:SlackMessage');

const result = await nango.listRecords({
  providerConfigKey: 'slack',
  connectionId: 'user-123',
  model: 'SlackMessage',
  cursor: lastCursor,  // fetch only records after this cursor
});

// Process records, then save new cursor
if (result.records.length > 0) {
  const newCursor = result.records[result.records.length - 1]._nango_metadata.cursor;
  await db.set('nango_cursor:slack:user-123:SlackMessage', newCursor);
}
```

## nango object reference (inside sync exec)

| Method | Purpose |
|--------|---------|
| `nango.get()` / `.post()` / `.put()` / `.patch()` / `.delete()` | Authenticated HTTP requests |
| `nango.paginate()` | Auto-paginated iteration |
| `nango.batchSave(records, modelName)` | Save records to cache (upsert by `id`) |
| `nango.batchDelete(records, modelName)` | Mark records as deleted |
| `nango.getCheckpoint()` | Get saved checkpoint (null on first run) |
| `nango.saveCheckpoint(data)` | Save checkpoint for next run |
| `nango.log(message, data?)` | Write to Nango's observable logs |
| `nango.getConnection()` | Get current connection details |
| `nango.getMetadata()` | Get per-customer configuration |

## Testing

```bash
# Dry run — executes sync locally, prints records to console (not persisted)
npx nango dryrun slack-messages 'user-123'

# With diagnostics (memory, CPU metrics)
npx nango dryrun slack-messages 'user-123' --diagnostics

# Against a specific environment
npx nango dryrun slack-messages 'user-123' -e prod
```

## Deploying

```bash
npx nango deploy dev                           # all functions
npx nango deploy --sync slack-messages dev     # single sync
npx nango deploy prod                          # to production
```

## Data retention

- Records not updated for **30 days**: payload pruned (metadata retained for change detection)
- Syncs not executed for **60 days**: all records permanently deleted
- Always fetch records promptly and store in your own database
