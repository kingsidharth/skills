# Sync

## Incremental sync via syncToken

The recommended pattern for keeping local state in sync with Google Calendar. Two phases:

### Phase 1: Initial full sync

```ts
let pageToken: string | undefined;
let syncToken: string | undefined;
const allEvents: calendar_v3.Schema$Event[] = [];

do {
  const res = await calendar.events.list({
    calendarId: "primary",
    singleEvents: true,
    pageToken,
    timeMin: oneYearAgo.toISOString(),  // optional: scope the initial sync
  });
  allEvents.push(...(res.data.items ?? []));
  pageToken = res.data.nextPageToken ?? undefined;
  syncToken = res.data.nextSyncToken ?? undefined;
} while (pageToken);

// Store all events locally
// Persist syncToken for next run
```

`nextSyncToken` only appears on the last page. Store it durably.

### Phase 2: Incremental sync

```ts
let pageToken: string | undefined;
let syncToken = loadStoredSyncToken();

do {
  const res = await calendar.events.list({
    calendarId: "primary",
    syncToken,
    pageToken,
  });

  for (const event of res.data.items ?? []) {
    if (event.status === "cancelled") {
      // Remove from local store
    } else {
      // Upsert in local store
    }
  }

  pageToken = res.data.nextPageToken ?? undefined;
  if (res.data.nextSyncToken) {
    syncToken = res.data.nextSyncToken;
  }
} while (pageToken);

// Persist new syncToken
```

Incremental results include deleted events (`status: "cancelled"`) so you can remove them locally.

### Handling 410 Gone

The sync token may expire or become invalid. On `410`, wipe local state and redo a full sync:

```ts
try {
  // incremental sync...
} catch (err: any) {
  if (err.code === 410) {
    clearLocalStore();
    performFullSync();
  } else {
    throw err;
  }
}
```

### Constraints

Query parameters used in the initial full sync constrain subsequent incremental syncs. You must use the same parameters (or a subset) across the chain. Using disallowed parameters returns `400`.

## Push notifications (webhooks)

Register a watch channel to receive push notifications when events change, instead of polling:

```ts
const res = await calendar.events.watch({
  calendarId: "primary",
  requestBody: {
    id: crypto.randomUUID(),         // unique channel ID
    type: "web_hook",
    address: "https://yourapp.com/calendar/webhook",
    expiration: String(Date.now() + 7 * 24 * 60 * 60 * 1000),  // max ~30 days
  },
});
// res.data.resourceId — needed to stop the channel
```

When events change, Google sends a POST to your address with headers:

- `X-Goog-Channel-ID` — your channel ID
- `X-Goog-Resource-State` — `sync` (initial), `exists` (change occurred)
- `X-Goog-Resource-ID` — the resource being watched

The notification body is empty — it only tells you *something* changed. You must call `events.list()` with your `syncToken` to get the actual changes.

### Stopping a channel

```ts
await calendar.channels.stop({
  requestBody: {
    id: channelId,
    resourceId: resourceId,
  },
});
```

### Channel expiry

Channels expire. Renew before expiry. Set a timer or cron to re-watch before the expiration timestamp.

## CalendarList sync

Same pattern works for `calendarList.list()` — it also supports `syncToken` for incremental sync of the user's calendar subscriptions. Use this to detect when a user adds or removes a calendar.

## Performance tips

- Use `fields` parameter to request only needed fields: `fields: "items(id,summary,start,end,status)"`.
- Use `showDeleted: true` during sync to catch cancellations.
- Batch requests (up to 50 per batch) for bulk operations.
- Implement exponential backoff on `403` (rate limit) and `429` (quota exceeded).
- Default quota: 1,000,000 queries/day per project.
