# Syncing a Mail Client

> Official guide: https://developers.google.com/workspace/gmail/api/guides/sync

## Sync Strategies Overview

| Strategy | When to Use | Mechanism |
|----------|-------------|-----------|
| **Full sync** | First connect, or when partial sync data is unavailable | `messages.list` → batched `messages.get` |
| **Partial sync** | Subsequent updates after initial full sync | `history.list` with saved `historyId` |
| **Push + partial sync** | Real-time updates in production | Pub/Sub notification triggers `history.list` |

---

## Full Sync

Required the first time your client connects, or when `history.list` returns 404.

### Procedure

1. Call `messages.list` to retrieve the first page of message IDs
2. Batch `messages.get` for each returned ID:
   - First-time fetch: `format=FULL` or `format=RAW` — cache the result
   - Previously cached message: `format=MINIMAL` — only `labelIds` may have changed
3. Page through using `nextPageToken`
4. **Store the `historyId`** from the most recent message (first in the list response)

### Python

```python
def full_sync(service):
    """Perform a full mailbox sync. Returns the latest historyId."""
    all_messages = []
    latest_history_id = None

    # Step 1: List all message IDs (paginated)
    results = service.users().messages().list(userId='me', maxResults=500).execute()
    messages = results.get('messages', [])
    all_messages.extend(messages)

    while 'nextPageToken' in results:
        results = service.users().messages().list(
            userId='me',
            maxResults=500,
            pageToken=results['nextPageToken']
        ).execute()
        all_messages.extend(results.get('messages', []))

    # Step 2: Batch-fetch message details
    for i in range(0, len(all_messages), 50):
        batch = service.new_batch_http_request()
        for msg_meta in all_messages[i:i+50]:
            batch.add(
                service.users().messages().get(
                    userId='me', id=msg_meta['id'], format='full'
                ),
                callback=process_message
            )
        batch.execute()

    # Step 3: Store historyId from the most recent message
    if all_messages:
        most_recent = service.users().messages().get(
            userId='me', id=all_messages[0]['id'], format='minimal'
        ).execute()
        latest_history_id = most_recent['historyId']

    return latest_history_id


def process_message(request_id, response, exception):
    if exception:
        print(f"Error: {exception}")
        return
    # Cache response locally (response is a full Message resource)
    save_to_cache(response)
```

### Node.js

```javascript
async function fullSync(gmail) {
  const allMessages = [];
  let pageToken = null;

  // Step 1: List all message IDs
  do {
    const res = await gmail.users.messages.list({
      userId: 'me',
      maxResults: 500,
      pageToken,
    });
    allMessages.push(...(res.data.messages || []));
    pageToken = res.data.nextPageToken;
  } while (pageToken);

  // Step 2: Fetch each message (sequential — see PERFORMANCE.md for batch)
  for (const { id } of allMessages) {
    const msg = await gmail.users.messages.get({
      userId: 'me', id, format: 'full',
    });
    saveToCache(msg.data);
  }

  // Step 3: Get historyId from most recent
  if (allMessages.length > 0) {
    const recent = await gmail.users.messages.get({
      userId: 'me', id: allMessages[0].id, format: 'minimal',
    });
    return recent.data.historyId;
  }
}
```

**Note:** You can also perform full sync using the equivalent `threads.list` / `threads.get` methods if your application is thread-oriented.

---

## Partial Sync (Incremental)

Uses the saved `historyId` to retrieve only changes since last sync.

### Procedure

1. Call `history.list` with `startHistoryId` = your saved value
2. Process change records: `messagesAdded`, `messagesDeleted`, `labelsAdded`, `labelsRemoved`
3. Update local cache accordingly
4. Store the new `historyId` from the response

### Python

```python
def partial_sync(service, start_history_id):
    """Sync changes since the given historyId. Returns new historyId."""
    try:
        results = service.users().history().list(
            userId='me',
            startHistoryId=start_history_id,
            historyTypes=['messageAdded', 'messageDeleted', 'labelAdded', 'labelRemoved']
        ).execute()
    except Exception as e:
        if '404' in str(e):
            # historyId too old — fall back to full sync
            return full_sync(service)
        raise

    for record in results.get('history', []):
        # New messages
        for added in record.get('messagesAdded', []):
            msg_id = added['message']['id']
            msg = service.users().messages().get(
                userId='me', id=msg_id, format='full'
            ).execute()
            save_to_cache(msg)

        # Deleted messages
        for deleted in record.get('messagesDeleted', []):
            remove_from_cache(deleted['message']['id'])

        # Label changes
        for change in record.get('labelsAdded', []):
            update_labels_in_cache(change['message']['id'], add=change['labelIds'])

        for change in record.get('labelsRemoved', []):
            update_labels_in_cache(change['message']['id'], remove=change['labelIds'])

    # Page through if needed
    while 'nextPageToken' in results:
        results = service.users().history().list(
            userId='me',
            startHistoryId=start_history_id,
            pageToken=results['nextPageToken'],
            historyTypes=['messageAdded', 'messageDeleted', 'labelAdded', 'labelRemoved']
        ).execute()
        # Process this page the same way...

    return results['historyId']
```

### Node.js

```javascript
async function partialSync(gmail, startHistoryId) {
  try {
    const res = await gmail.users.history.list({
      userId: 'me',
      startHistoryId,
      historyTypes: ['messageAdded', 'messageDeleted', 'labelAdded', 'labelRemoved'],
    });

    for (const record of (res.data.history || [])) {
      for (const added of (record.messagesAdded || [])) {
        const msg = await gmail.users.messages.get({
          userId: 'me', id: added.message.id, format: 'full',
        });
        saveToCache(msg.data);
      }
      for (const deleted of (record.messagesDeleted || [])) {
        removeFromCache(deleted.message.id);
      }
      for (const change of (record.labelsAdded || [])) {
        updateLabels(change.message.id, { add: change.labelIds });
      }
      for (const change of (record.labelsRemoved || [])) {
        updateLabels(change.message.id, { remove: change.labelIds });
      }
    }

    return res.data.historyId;
  } catch (err) {
    if (err.code === 404) {
      return fullSync(gmail); // historyId too old
    }
    throw err;
  }
}
```

---

## Re-Syncing When Out of Sync

| Symptom | Cause | Resolution |
|---------|-------|------------|
| `history.list` returns 404 | `historyId` too old (purged by Google) | Perform full sync |
| Missing messages | Dropped push notifications | Call `history.list` with last known `historyId` |
| Label mismatches | Didn't process all `labelsAdded`/`labelsRemoved` events | Re-fetch affected messages with `format=MINIMAL` |
| Duplicates | Processing same history event twice | Deduplicate by message `id` (use idempotent cache operations) |
| Stale data after token refresh | Token was revoked silently | Re-authenticate, then full sync |

### Robust Sync Algorithm

```
1. Load saved historyId from database
2. Call history.list(startHistoryId=saved_id)
3. If SUCCESS:
   → Process changes
   → Save new historyId
4. If 404 (historyId too old):
   → Trigger full sync
   → Save new historyId
5. If 401 (auth error):
   → Refresh token, retry
   → If refresh fails, re-authenticate user
6. If 429 (rate limited):
   → Exponential backoff with jitter, retry
7. Optionally: verify a sample of messages with format=MINIMAL
   to catch any drift
```

---

## Recommended Production Architecture

```
┌──────────────────────────────────────────────────┐
│                Your Application                   │
│                                                   │
│  ┌───────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ Full Sync │  │ Partial  │  │ Fallback Poll │ │
│  │ (1st run) │  │  Sync    │  │ (every 10min) │ │
│  └─────┬─────┘  └────┬─────┘  └──────┬────────┘ │
│        │              │               │           │
│        └──────┬───────┘               │           │
│               │                       │           │
│        ┌──────▼───────┐              │           │
│        │  Local Cache  │◄─────────────┘           │
│        │  + historyId  │                          │
│        └───────────────┘                          │
└──────────────────────────────────────────────────┘
          ▲
          │ Pub/Sub notification triggers partial sync
          │
┌─────────┴──────────┐
│  Cloud Pub/Sub     │
│  (push/pull sub)   │
└────────┬───────────┘
         │
┌────────┴───────────┐
│    Gmail API       │
│  (watch request)   │
└────────────────────┘
```

1. **Full sync** on first connect → saves `historyId`
2. **Pub/Sub push** notification arrives → triggers `partial_sync(historyId)`
3. **Fallback poll** every ~10 minutes calls `history.list` as insurance (notifications can be dropped)
4. If `history.list` returns 404 → automatic fallback to full sync
