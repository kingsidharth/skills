# Gmail API Data Structures

> Official reference: https://developers.google.com/workspace/gmail/api/reference/rest

## Message

The fundamental unit. Immutable once created — only `labelIds` can change.

```json
{
  "id": "18a1b2c3d4e5f6g7",
  "threadId": "18a1b2c3d4e5f6g7",
  "labelIds": ["INBOX", "UNREAD", "CATEGORY_PERSONAL"],
  "snippet": "Preview text of the email body...",
  "historyId": "9876543210",
  "internalDate": "1700000000000",
  "sizeEstimate": 12345,
  "payload": {
    "partId": "",
    "mimeType": "multipart/mixed",
    "filename": "",
    "headers": [
      { "name": "From",       "value": "sender@example.com" },
      { "name": "To",         "value": "recipient@example.com" },
      { "name": "Cc",         "value": "cc@example.com" },
      { "name": "Subject",    "value": "Email Subject Line" },
      { "name": "Date",       "value": "Mon, 20 Nov 2023 10:00:00 -0500" },
      { "name": "Message-ID", "value": "<unique-id@mail.gmail.com>" },
      { "name": "In-Reply-To","value": "<parent-msg-id@mail.gmail.com>" },
      { "name": "References", "value": "<root-id@mail.gmail.com>" }
    ],
    "body": { "size": 0 },
    "parts": [
      {
        "partId": "0",
        "mimeType": "multipart/alternative",
        "body": { "size": 0 },
        "parts": [
          {
            "partId": "0.0",
            "mimeType": "text/plain",
            "body": { "size": 1234, "data": "<base64url-encoded-plain-text>" }
          },
          {
            "partId": "0.1",
            "mimeType": "text/html",
            "body": { "size": 5678, "data": "<base64url-encoded-html>" }
          }
        ]
      },
      {
        "partId": "1",
        "mimeType": "application/pdf",
        "filename": "document.pdf",
        "headers": [
          { "name": "Content-Disposition", "value": "attachment; filename=\"document.pdf\"" }
        ],
        "body": { "attachmentId": "ANGjdJ8abc123...", "size": 98765 }
      }
    ]
  }
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Immutable message ID |
| `threadId` | string | Thread this message belongs to |
| `labelIds` | string[] | System + user label IDs applied |
| `snippet` | string | Short plain-text preview (~100 chars) |
| `historyId` | string | Last history record that modified this message |
| `internalDate` | string (epoch ms) | When Google accepted the message (more reliable than `Date` header) |
| `sizeEstimate` | integer | Approximate size in bytes |
| `payload` | MessagePart | Parsed MIME structure (when `format=FULL`) |
| `raw` | string | Entire RFC 2822, base64url encoded (when `format=RAW`) |

### MIME Part Tree Shapes

The `payload.parts` tree is recursive. Common patterns:

| Email Type | MIME Structure |
|------------|---------------|
| Plain text only | `text/plain` — body in `payload.body.data` |
| HTML only | `text/html` — body in `payload.body.data` |
| Plain + HTML | `multipart/alternative` → `[text/plain, text/html]` |
| Body + attachments | `multipart/mixed` → `[multipart/alternative, attachment, ...]` |
| Inline images | `multipart/related` → body + images referenced by `Content-ID` |

### Attachment Handling

- **Small attachments (<~2MB):** Data is inline in `part.body.data` (base64url)
- **Large attachments:** `body.data` is absent; use `body.attachmentId` and fetch separately via `messages.attachments.get`
- **Detect attachments:** Check for `filename` field on a part, or `Content-Disposition: attachment` header

---

## Thread

```json
{
  "id": "18a1b2c3d4e5f6g7",
  "historyId": "9876543210",
  "messages": [
    { "id": "msg1", "threadId": "...", "labelIds": [...], "payload": {...} },
    { "id": "msg2", "threadId": "...", "labelIds": [...], "payload": {...} }
  ]
}
```

Messages are grouped into a thread based on `References`, `In-Reply-To` headers, and matching `Subject`. Labels on a thread = union of labels across all its messages.

---

## Label

```json
{
  "id": "Label_42",
  "name": "Projects/Active",
  "type": "user",
  "messageListVisibility": "show",
  "labelListVisibility": "labelShow",
  "color": { "textColor": "#ffffff", "backgroundColor": "#4a86e8" },
  "messagesTotal": 150,
  "messagesUnread": 12,
  "threadsTotal": 80,
  "threadsUnread": 5
}
```

### System Labels

| ID | Manually Applicable? | Notes |
|----|---------------------|-------|
| `INBOX` | Yes | |
| `SENT` | No | Auto-applied to sent messages |
| `DRAFT` | No | Auto-applied to draft messages |
| `SPAM` | Yes | |
| `TRASH` | Yes | |
| `UNREAD` | Yes | |
| `STARRED` | Yes | |
| `IMPORTANT` | Yes | |
| `CATEGORY_PERSONAL` | Yes | Gmail category tabs |
| `CATEGORY_SOCIAL` | Yes | |
| `CATEGORY_PROMOTIONS` | Yes | |
| `CATEGORY_UPDATES` | Yes | |
| `CATEGORY_FORUMS` | Yes | |

User labels support nested names using `/` (e.g., `Projects/Active`). System label names are reserved — creating a user label with a system label name returns HTTP 400.

---

## Draft

```json
{
  "id": "r-3948576920",
  "message": {
    "id": "18abc123",
    "threadId": "18abc123",
    "labelIds": ["DRAFT"]
  }
}
```

The `draft.id` is stable. The inner `message.id` **changes every time the draft is updated** via `drafts.update`. When sent, the draft is deleted and a new message with `SENT` label is created.

---

## History Record

```json
{
  "id": "12345",
  "messages": [
    { "id": "msg123", "threadId": "thread456" }
  ],
  "messagesAdded": [
    { "message": { "id": "msg123", "threadId": "thread456", "labelIds": ["INBOX", "UNREAD"] } }
  ],
  "messagesDeleted": [
    { "message": { "id": "msg789", "threadId": "thread012" } }
  ],
  "labelsAdded": [
    { "message": { "id": "msg345" }, "labelIds": ["STARRED"] }
  ],
  "labelsRemoved": [
    { "message": { "id": "msg345" }, "labelIds": ["UNREAD"] }
  ]
}
```

Each record captures one atomic change. Used for incremental synchronization via `history.list`.

---

## Filter

```json
{
  "id": "ANe1B...",
  "criteria": {
    "from": "notifications@github.com",
    "to": "",
    "subject": "",
    "query": "",
    "negatedQuery": "",
    "hasAttachment": false,
    "excludeChats": true,
    "size": 0,
    "sizeComparison": "unspecified"
  },
  "action": {
    "addLabelIds": ["Label_42"],
    "removeLabelIds": ["INBOX"],
    "forward": ""
  }
}
```

Filters apply to **individual messages** (not threads). All criteria must match (AND logic). The `query` field accepts full Gmail search syntax.

### Criteria Fields

| Field | Description |
|-------|-------------|
| `from` | Sender display name or email |
| `to` | Recipient (to/cc/bcc) |
| `subject` | Case-insensitive phrase in subject |
| `query` | Gmail advanced search syntax |
| `negatedQuery` | Exclude messages matching this query |
| `hasAttachment` | Boolean |
| `excludeChats` | Exclude chat messages |
| `size` / `sizeComparison` | Filter by message size (`larger`, `smaller`) |

### Action Fields

| Field | Description |
|-------|-------------|
| `addLabelIds` | Labels to add |
| `removeLabelIds` | Labels to remove (e.g., `["INBOX"]` to archive) |
| `forward` | Forward to a verified email address |

---

## Watch Response (Pub/Sub)

```json
{
  "historyId": "9876543210",
  "expiration": "1700000000000"
}
```

## Push Notification Payload

The Pub/Sub `message.data` field (base64url encoded):

```json
{
  "emailAddress": "user@example.com",
  "historyId": "9876543210"
}
```

Use this `historyId` with `history.list` to fetch the actual changes.
