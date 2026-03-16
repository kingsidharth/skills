---
name: gmail-api
description: "Integrate with the Gmail API for reading, sending, syncing, and managing email programmatically. Use when building email clients, automating email workflows, syncing mailboxes, processing attachments, managing labels/filters, or setting up push notifications via Pub/Sub. Covers OAuth 2.0 authentication (including localhost/dev and Cloudflare tunnels), full and partial sync, Pub/Sub push, search/query operators, batch requests, drafts, attachments, labels, filters, and performance best practices. Supports Python and Node.js SDKs."
---

# Gmail API Integration Skill

## When to Read What

| Task | Read |
|------|------|
| **Any Gmail API work** | This file (always read first) |
| **Setting up OAuth / tokens** | [AUTH.md](references/AUTH.md) |
| **Syncing a mailbox** | [SYNC.md](references/patterns/SYNC.md) |
| **Push notifications (Pub/Sub)** | [PUSH.md](references/patterns/PUSH.md) |
| **Searching / filtering messages** | [SEARCH.md](references/patterns/SEARCH.md) |
| **Labels and filters** | [LABELS_FILTERS.md](references/patterns/LABELS_FILTERS.md) |
| **Sending, drafts, attachments** | [COMPOSE.md](references/patterns/COMPOSE.md) |
| **Batch requests & performance** | [PERFORMANCE.md](references/patterns/PERFORMANCE.md) |
| **Data structures (Message, Thread, etc.)** | [DATA_STRUCTURES.md](references/DATA_STRUCTURES.md) |
| **Python code snippets** | [PYTHON.md](references/snippets/PYTHON.md) |
| **Node.js code snippets** | [NODEJS.md](references/snippets/NODEJS.md) |

---

## Critical Rules

1. **Always use the narrowest OAuth scope needed.** Don't request `mail.google.com` (full access) when `gmail.readonly` suffices. Changing scopes later forces users to re-consent — delete stored `token.json` when scopes change.

2. **Store refresh tokens securely and persistently.** The refresh token is only returned on the first authorization (or when `prompt=consent`). Losing it means the user must re-authorize. Store in a database or encrypted file — never in source control.

3. **Never refresh the access token on every API call.** Cache it and only refresh when expired. Refreshing on every call is slow and may be throttled by Google.

4. **Use `historyId` for incremental sync — never re-list the entire mailbox.** Store the `historyId` from your last sync. Call `history.list` with it to get only changes. If it returns 404, fall back to full sync.

5. **Renew Pub/Sub `watch` before it expires (~7 days).** Set up a recurring job (e.g., every 3 days). Missing renewal = missed notifications with no warning.

6. **Batch API calls aggressively.** Group `messages.get` calls (up to 50–100 per batch) instead of making sequential HTTP requests. Use `format=MINIMAL` for already-cached messages.

7. **Messages are immutable — only labels change.** Cache message content locally. On subsequent checks, use `format=MINIMAL` to detect label changes without re-downloading the body.

8. **Use `fields` parameter to request partial responses.** Only fetch the fields you need to minimize bandwidth and latency.

9. **Handle `invalid_grant` gracefully.** Delete stored tokens and re-initiate the OAuth flow. Common causes: Testing mode (7-day token expiry), password change on Gmail-scoped tokens, user revocation, 6-month inactivity.

10. **Implement exponential backoff for rate limits (429/403).** Start at 1s, double each retry, cap at ~32s. Add jitter to avoid thundering herd.

---

## Core Concepts (Quick Reference)

The Gmail API is RESTful. Base URL: `https://gmail.googleapis.com/gmail/v1`

**Resources:**

| Resource | Description | Key Fields |
|----------|-------------|------------|
| **Message** | An email. Immutable once created. | `id`, `threadId`, `labelIds`, `payload` (MIME tree), `historyId`, `internalDate`, `snippet` |
| **Thread** | A conversation — a collection of related messages. | `id`, `messages[]`, `historyId` |
| **Label** | Tag for organizing messages. System labels (INBOX, SENT, DRAFT, SPAM, TRASH, STARRED, UNREAD, IMPORTANT, CATEGORY_*) + user labels. | `id`, `name`, `type`, `color` |
| **Draft** | An unsent message container. Stable ID, but inner message ID changes on update. | `id`, `message` |
| **History** | A change record for incremental sync. | `id`, `messagesAdded`, `messagesDeleted`, `labelsAdded`, `labelsRemoved` |
| **Filter** | A server-side rule applied to incoming messages (not threads). | `id`, `criteria`, `action` |

**Message format options** (on `messages.get`):

| Format | Returns | When to Use |
|--------|---------|-------------|
| `FULL` | Parsed MIME payload + headers + body | First-time fetch, full content needed |
| `MINIMAL` | Only id, threadId, labelIds, snippet, historyId | Sync checks on cached messages |
| `RAW` | Entire RFC 2822 as base64url in `raw` field | Re-parsing, forwarding, archiving |
| `METADATA` | id, threadId, labelIds, snippet + headers (no body) | Header inspection without body |

**Authentication:** OAuth 2.0 only. Access tokens (~1hr), refresh tokens (long-lived). See [AUTH.md](references/AUTH.md).

**Sync strategies:** Full sync (first connect), partial sync via `history.list`, push via Cloud Pub/Sub. See [SYNC.md](references/patterns/SYNC.md) and [PUSH.md](references/patterns/PUSH.md).

**userId:** Always use `"me"` for the authenticated user.

---

## SDK Setup

### Python

```bash
pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib
```

```python
from googleapiclient.discovery import build
service = build('gmail', 'v1', credentials=creds)
```

### Node.js

```bash
npm install googleapis @google-cloud/local-auth
```

```javascript
import { google } from 'googleapis';
const gmail = google.gmail({ version: 'v1', auth });
```

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Refresh token not returned | Set `access_type=offline` AND `prompt=consent` in auth URL |
| Token expires after 7 days | Your OAuth consent screen is in "Testing" mode — publish to "Production" |
| `messages.list` returns only IDs | By design — call `messages.get` per message (use batch) |
| Search doesn't find alias emails | API doesn't expand aliases like the Gmail UI does |
| Labels added to thread don't apply to future messages | Thread label operations only affect *existing* messages |
| `history.list` returns 404 | `historyId` is too old — fall back to full sync |
| Push notifications stop | `watch` expired (7-day max) — renew proactively |
| Slow API calls | You're refreshing the access token every call — cache it |
| Batch request returns 401 | Auth header must be on the outer request, not inner parts |
| `format=FULL` with `gmail.metadata` scope | Scope doesn't permit body access — use `METADATA` format |
