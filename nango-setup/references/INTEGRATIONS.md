# Nango Integration Configs

Quick reference for configuring specific APIs in Nango. Each integration needs to be set up in the Nango dashboard with the provider's OAuth credentials.

## Priority integrations (Phase 1)

### Slack

| Field | Value |
|-------|-------|
| Integration ID | `slack` |
| Auth type | OAuth 2.0 |
| Base URL | `https://slack.com` |
| Key scopes | `channels:read`, `channels:history`, `chat:write`, `users:read`, `groups:read`, `im:read`, `files:read` |
| Pagination | Cursor-based (`response_metadata.next_cursor`) |
| Rate limits | Tier 2-4 depending on method (see Slack docs) |

**Sync targets**: channels, messages, users, files, threads
**Action targets**: send message, create channel, upload file, add reaction

**Pagination pattern** (inside sync):
```typescript
for await (const page of nango.paginate({
  endpoint: '/api/conversations.list',
  paginate: {
    type: 'cursor',
    cursor_path_in_response: 'response_metadata.next_cursor',
    cursor_name_in_request: 'cursor',
    limit_name_in_request: 'limit',
  },
  params: { limit: '200', types: 'public_channel,private_channel' },
})) {
  await nango.batchSave(page.map(mapChannel), 'SlackChannel');
}
```

---

### Gmail (google-mail)

| Field | Value |
|-------|-------|
| Integration ID | `google-mail` |
| Auth type | OAuth 2.0 (Google) |
| Base URL | `https://gmail.googleapis.com` |
| Key scopes | `https://www.googleapis.com/auth/gmail.readonly`, `https://www.googleapis.com/auth/gmail.send`, `https://www.googleapis.com/auth/gmail.modify` |
| Pagination | `nextPageToken` / `pageToken` |
| Rate limits | 250 quota units/user/second |

**Sync targets**: messages, threads, labels
**Action targets**: send email, create draft, modify labels

**Incremental sync pattern** (using historyId):
```typescript
const checkpoint = await nango.getCheckpoint();

if (checkpoint?.historyId) {
  // Incremental: get changes since last sync
  const history = await nango.get({
    endpoint: '/gmail/v1/users/me/history',
    params: {
      startHistoryId: checkpoint.historyId,
      historyTypes: 'messageAdded,messageDeleted',
    },
  });
  // Process history changes...
} else {
  // Full sync: list all messages
  for await (const page of nango.paginate({
    endpoint: '/gmail/v1/users/me/messages',
    params: { maxResults: '100' },
    paginate: {
      type: 'cursor',
      cursor_path_in_response: 'nextPageToken',
      cursor_name_in_request: 'pageToken',
      limit_name_in_request: 'maxResults',
    },
  })) {
    // Fetch full message for each ID, then batchSave
  }
}
```

**Note**: Gmail `messages.list` returns only IDs. You must call `messages.get` for each message to get content. Use `format=metadata` for headers only, `format=full` for body.

---

### Google Calendar (google-calendar)

| Field | Value |
|-------|-------|
| Integration ID | `google-calendar` |
| Auth type | OAuth 2.0 (Google) |
| Base URL | `https://www.googleapis.com/calendar` |
| Key scopes | `https://www.googleapis.com/auth/calendar.readonly`, `https://www.googleapis.com/auth/calendar.events` |
| Pagination | `nextPageToken` |
| Incremental | `syncToken` — use it to get only changes since last sync |

**Sync targets**: calendars, events
**Action targets**: create event, update event, delete event

**Incremental sync pattern** (using syncToken):
```typescript
const checkpoint = await nango.getCheckpoint();
const params: Record<string, string> = { maxResults: '250' };

if (checkpoint?.syncToken) {
  params.syncToken = checkpoint.syncToken;
} else {
  // Full sync — optionally set timeMin
  params.timeMin = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000).toISOString();
}

const response = await nango.get({
  endpoint: '/calendar/v3/calendars/primary/events',
  params,
});

await nango.batchSave(response.data.items.map(mapEvent), 'CalendarEvent');

// Save syncToken for next run
if (response.data.nextSyncToken) {
  await nango.saveCheckpoint({ syncToken: response.data.nextSyncToken });
}
```

---

### Google Docs (google-docs)

| Field | Value |
|-------|-------|
| Integration ID | `google-docs` |
| Auth type | OAuth 2.0 (Google) |
| Base URL | `https://docs.googleapis.com` |
| Key scopes | `https://www.googleapis.com/auth/documents.readonly`, `https://www.googleapis.com/auth/drive.readonly` (for listing) |

**Sync targets**: document metadata, document content (text extraction)
**Action targets**: create document, update content

**Note**: Google Docs API doesn't support listing documents. Use the Google Drive API to list docs (`mimeType='application/vnd.google-apps.document'`), then fetch each document's content via the Docs API.

```typescript
// List docs via Drive API
const driveResponse = await nango.get({
  endpoint: 'https://www.googleapis.com/drive/v3/files',
  params: {
    q: "mimeType='application/vnd.google-apps.document'",
    fields: 'files(id,name,modifiedTime)',
    pageSize: '100',
  },
  baseUrlOverride: 'https://www.googleapis.com',
});

// Fetch content for each doc
for (const file of driveResponse.data.files) {
  const doc = await nango.get({
    endpoint: `/v1/documents/${file.id}`,
    params: { fields: 'title,body' },
  });
  // Extract text from doc.data.body.content
}
```

---

### Google Drive (google-drive)

| Field | Value |
|-------|-------|
| Integration ID | `google-drive` |
| Auth type | OAuth 2.0 (Google) |
| Base URL | `https://www.googleapis.com/drive` |
| Key scopes | `https://www.googleapis.com/auth/drive.readonly` |
| Incremental | `changes.list` with `startPageToken` |

**Sync targets**: files metadata, folders, permissions
**Action targets**: upload file, create folder, share file

---

## Phase 2 integrations

### Linear

| Field | Value |
|-------|-------|
| Integration ID | `linear` |
| Auth type | OAuth 2.0 |
| Base URL | `https://api.linear.app` |
| API type | **GraphQL** |
| Key scopes | `read`, `write`, `issues:create` |

**Important**: Linear uses GraphQL. Proxy calls use POST to `/graphql`:

```typescript
const response = await nango.post({
  endpoint: '/graphql',
  data: {
    query: `
      query {
        issues(filter: { updatedAt: { gt: "${since}" } }) {
          nodes {
            id title state { name } assignee { name } updatedAt
          }
          pageInfo { hasNextPage endCursor }
        }
      }
    `,
  },
});
```

---

### PostHog

| Field | Value |
|-------|-------|
| Integration ID | `posthog` |
| Auth type | API Key |
| Base URL | `https://app.posthog.com` (or self-hosted) |

**Sync targets**: events, persons, feature flags, insights

---

### Metabase

| Field | Value |
|-------|-------|
| Integration ID | `metabase` |
| Auth type | API Key or Session token |
| Base URL | Your Metabase instance URL |

**Sync targets**: questions, dashboards, collections, database metadata

---

## Google OAuth setup notes

All Google integrations (Gmail, Calendar, Docs, Drive) share the same Google Cloud project but use different scopes. You can:

1. **Single integration with broad scopes** — one `google` integration covering all services. Simpler but requests more permissions upfront.
2. **Separate integrations per service** — `google-mail`, `google-calendar`, `google-docs`, `google-drive`. More granular consent, better for privacy.

**Recommended**: Use separate integrations. Users only grant permissions for services they actually use.

For all Google integrations, you'll need to go through Google's OAuth consent screen setup and potentially their app verification process for sensitive scopes.
