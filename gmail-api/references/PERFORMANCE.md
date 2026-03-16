# Batch Requests & Performance

> Batch guide: https://developers.google.com/workspace/gmail/api/guides/batch
> Performance tips: https://developers.google.com/gmail/api/guides/performance

## Batch Requests

Combine multiple API calls into a single HTTP request to reduce network overhead.

### Limits

| Limit | Value |
|-------|-------|
| Max calls per batch (raw HTTP) | 100 |
| Max calls per batch (Python client) | 1000 |
| **Recommended** batch size | ≤ 50 (larger batches may trigger rate limiting) |
| Quota impact | Each inner call counts individually |
| Media uploads in batch | NOT allowed |

### Python — Batch with Callbacks

```python
def fetch_messages_batch(service, message_ids, format='metadata'):
    """Fetch multiple messages in batched requests."""
    results = {}

    def callback(request_id, response, exception):
        if exception:
            print(f"Error for {request_id}: {exception}")
        else:
            results[request_id] = response

    # Process in chunks of 50
    for i in range(0, len(message_ids), 50):
        batch = service.new_batch_http_request(callback=callback)
        for msg_id in message_ids[i:i+50]:
            batch.add(
                service.users().messages().get(
                    userId='me', id=msg_id, format=format
                ),
                request_id=msg_id
            )
        batch.execute()

    return results
```

### Python — Batch Label Modifications

```python
def archive_messages_batch(service, message_ids):
    """Archive multiple messages (remove INBOX label) in batch."""
    def callback(request_id, response, exception):
        if exception:
            print(f"Failed to archive {request_id}: {exception}")

    for i in range(0, len(message_ids), 50):
        batch = service.new_batch_http_request(callback=callback)
        for msg_id in message_ids[i:i+50]:
            batch.add(
                service.users().messages().modify(
                    userId='me', id=msg_id,
                    body={'removeLabelIds': ['INBOX']}
                ),
                request_id=msg_id
            )
        batch.execute()
```

### Raw HTTP Batch (for Node.js / other clients)

The Node.js `googleapis` library lacks a built-in batch method. You must construct raw multipart HTTP:

```
POST /batch/gmail/v1 HTTP/1.1
Host: gmail.googleapis.com
Authorization: Bearer {access_token}
Content-Type: multipart/mixed; boundary=batch_boundary

--batch_boundary
Content-Type: application/http
Content-ID: <item1>

GET /gmail/v1/users/me/messages/msg1?format=metadata

--batch_boundary
Content-Type: application/http
Content-ID: <item2>

GET /gmail/v1/users/me/messages/msg2?format=metadata

--batch_boundary--
```

**Note:** The `Authorization` header goes on the outer request only. Each inner request specifies its own path, verb, and body. The response is also `multipart/mixed` with individual HTTP responses per part.

### Node.js — Batch via fetch

```javascript
async function batchGetMessages(accessToken, messageIds) {
  const boundary = 'batch_gmail_api';
  const parts = messageIds.map((id, i) => [
    `--${boundary}`,
    'Content-Type: application/http',
    `Content-ID: <item${i}>`,
    '',
    `GET /gmail/v1/users/me/messages/${id}?format=metadata`,
    '',
  ].join('\r\n')).join('\r\n');

  const body = `${parts}\r\n--${boundary}--`;

  const res = await fetch('https://gmail.googleapis.com/batch/gmail/v1', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': `multipart/mixed; boundary=${boundary}`,
    },
    body,
  });

  return res.text(); // Parse the multipart response
}
```

---

## Partial Responses (`fields` parameter)

Reduce response size by requesting only needed fields:

```python
# Only get id, labels, and specific headers
result = service.users().messages().get(
    userId='me',
    id='msg123',
    format='metadata',
    metadataHeaders=['From', 'Subject', 'Date'],
    fields='id,threadId,labelIds,payload/headers'
).execute()
```

```python
# List messages — only get IDs (skip threadId, even)
results = service.users().messages().list(
    userId='me',
    q='is:unread',
    fields='messages/id,nextPageToken'
).execute()
```

```javascript
const res = await gmail.users.messages.get({
  userId: 'me',
  id: 'msg123',
  format: 'metadata',
  metadataHeaders: ['From', 'Subject', 'Date'],
  fields: 'id,threadId,labelIds,payload/headers',
});
```

---

## gzip Compression

Reduces bandwidth significantly. Most client libraries enable this automatically.

For raw HTTP, set these headers:

```
Accept-Encoding: gzip
User-Agent: my-app (gzip)
```

---

## Caching Strategy

Messages are **immutable** — once created, only `labelIds` change.

```
┌───────────────────────────────────────────────┐
│               Caching Rules                    │
├───────────────────────────────────────────────┤
│ 1. First fetch:    format=FULL  → cache all   │
│ 2. Sync check:    format=MINIMAL → only       │
│                    labelIds may have changed   │
│ 3. Body content:  NEVER changes → cache once  │
│ 4. Attachments:   NEVER change → cache once   │
│ 5. Headers:       NEVER change → cache once   │
└───────────────────────────────────────────────┘
```

### Python — Cache-Aware Fetch

```python
def get_message_smart(service, msg_id, cache):
    """Fetch a message, using cache when possible."""
    if msg_id in cache:
        # Only check for label changes
        minimal = service.users().messages().get(
            userId='me', id=msg_id, format='minimal'
        ).execute()
        cache[msg_id]['labelIds'] = minimal['labelIds']
        return cache[msg_id]
    else:
        full = service.users().messages().get(
            userId='me', id=msg_id, format='full'
        ).execute()
        cache[msg_id] = full
        return full
```

---

## Rate Limiting & Exponential Backoff

Gmail API has per-user rate limits. When you get HTTP 429 or 403 with rate limit errors:

### Python

```python
import time
import random

def api_call_with_backoff(callable_fn, max_retries=5):
    """Execute an API call with exponential backoff on rate limits."""
    for attempt in range(max_retries):
        try:
            return callable_fn()
        except HttpError as e:
            if e.resp.status in (429, 403) and 'rate' in str(e).lower():
                wait = (2 ** attempt) + random.uniform(0, 1)
                print(f"Rate limited. Retrying in {wait:.1f}s...")
                time.sleep(wait)
            else:
                raise
    raise Exception(f"Failed after {max_retries} retries")

# Usage
result = api_call_with_backoff(
    lambda: service.users().messages().list(userId='me').execute()
)
```

### Node.js

```javascript
async function withBackoff(fn, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if ((err.code === 429 || err.code === 403) && attempt < maxRetries - 1) {
        const wait = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
        console.log(`Rate limited. Retrying in ${(wait/1000).toFixed(1)}s...`);
        await new Promise(r => setTimeout(r, wait));
      } else {
        throw err;
      }
    }
  }
}
```

---

## Performance Checklist

| Practice | Impact |
|----------|--------|
| Batch `messages.get` calls (50 per batch) | Reduces HTTP overhead by ~50x |
| Use `format=MINIMAL` for cached messages | ~95% smaller response |
| Use `fields` parameter | Only fetch what you need |
| Enable gzip | ~60-80% bandwidth reduction |
| Cache message content (immutable) | Eliminates redundant fetches |
| Use `history.list` instead of re-listing | Only fetches changes |
| Implement exponential backoff | Avoids cascading failures from rate limits |
| Don't refresh access token on every call | Avoids latency + throttling |
| Use PATCH for partial updates | Only send changed fields |
| Use `X-HTTP-Method-Override: PATCH` | If your firewall blocks PATCH |
