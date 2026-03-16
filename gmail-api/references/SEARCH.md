# Searching & Filtering Messages

> Official guide: https://developers.google.com/workspace/gmail/api/guides/filtering
> Gmail search operators: https://support.google.com/mail/answer/7190

## Using the `q` Parameter

`messages.list` and `threads.list` accept a `q` parameter with the same syntax as the Gmail web search box:

```
GET /gmail/v1/users/me/messages?q=from:github.com after:2024/01/01 has:attachment
```

---

## Search Operators

### Sender / Recipient

| Operator | Example | Description |
|----------|---------|-------------|
| `from:` | `from:alice@example.com` | Sender address or name |
| `to:` | `to:bob@example.com` | Recipient (to, cc, bcc) |
| `cc:` | `cc:team@example.com` | CC recipient |
| `bcc:` | `bcc:secret@example.com` | BCC recipient |
| `list:` | `list:info@mailing-list.com` | Mailing list (List-ID header) |

### Content

| Operator | Example | Description |
|----------|---------|-------------|
| `subject:` | `subject:"monthly report"` | Subject contains phrase |
| `"exact phrase"` | `"budget proposal"` | Body or subject contains exact phrase |
| `AROUND` | `"budget" AROUND 5 "report"` | Words within N words of each other |

### Attachments

| Operator | Example | Description |
|----------|---------|-------------|
| `has:attachment` | `has:attachment` | Any attachment |
| `has:drive` | `has:drive` | Google Drive attachment |
| `has:document` | `has:document` | Google Docs attachment |
| `has:spreadsheet` | `has:spreadsheet` | Google Sheets attachment |
| `has:presentation` | `has:presentation` | Google Slides attachment |
| `has:youtube` | `has:youtube` | YouTube video |
| `filename:` | `filename:pdf` | Attachment filename contains |
| `filename:` | `filename:report.xlsx` | Specific attachment filename |

### Location & Status

| Operator | Example | Description |
|----------|---------|-------------|
| `in:` | `in:inbox` / `in:sent` / `in:trash` / `in:anywhere` | Location |
| `is:` | `is:unread` / `is:starred` / `is:important` / `is:read` | Status |
| `label:` | `label:work` | Has label (use `-` for spaces: `label:my-label`) |
| `category:` | `category:social` | Gmail tab category |

### Dates

| Operator | Example | Description |
|----------|---------|-------------|
| `after:` | `after:2024/01/01` | After date (yyyy/mm/dd) |
| `before:` | `before:2024/12/31` | Before date |
| `newer_than:` | `newer_than:2d` | Within last N days/months/years (d/m/y) |
| `older_than:` | `older_than:1y` | Older than N units |

**Important:** All dates in `q` are interpreted as midnight **PST**. For other timezones, use epoch seconds:
```
?q=after:1388552400 before:1391230800
```

### Size

| Operator | Example | Description |
|----------|---------|-------------|
| `size:` | `size:5000000` | Larger than N bytes |
| `larger:` | `larger:10M` | Larger than (K, M units) |
| `smaller:` | `smaller:1M` | Smaller than |

### Special

| Operator | Example | Description |
|----------|---------|-------------|
| `rfc822msgid:` | `rfc822msgid:<msg-id@mail.com>` | Find by Message-ID header |
| `deliveredto:` | `deliveredto:user@example.com` | Delivered-to header |

### Logical Operators

| Operator | Example | Description |
|----------|---------|-------------|
| (space) | `from:alice is:unread` | AND (implicit) |
| `OR` | `from:alice OR from:bob` | OR (must be UPPERCASE) |
| `-` | `-from:noreply@` | NOT / exclude |
| `()` | `(from:alice OR from:bob) is:unread` | Grouping |
| `{ }` | `{from:alice from:bob}` | Alternative OR syntax |

---

## API vs. Gmail UI Differences

| Behavior | Gmail UI | Gmail API |
|----------|----------|-----------|
| Alias expansion | Searches `from:primary@` also finds `alias@` | No alias expansion — must search exact address |
| Thread-wide search | Matches if any message in thread matches | Matches individual messages only |
| Chat messages | Included by default | Include `in:chats` or use `excludeChats` in filter criteria |

---

## Code Examples

### Python — Search by Query

```python
def search_messages(service, query, max_results=100):
    """Search for messages matching a Gmail query."""
    results = service.users().messages().list(
        userId='me',
        q=query,
        maxResults=max_results
    ).execute()

    messages = results.get('messages', [])

    # Paginate if needed
    while 'nextPageToken' in results and len(messages) < max_results:
        results = service.users().messages().list(
            userId='me',
            q=query,
            maxResults=max_results,
            pageToken=results['nextPageToken']
        ).execute()
        messages.extend(results.get('messages', []))

    return messages  # Each has only 'id' and 'threadId'
```

### Python — Filter by Label IDs

```python
# Combine label filter with query
results = service.users().messages().list(
    userId='me',
    labelIds=['INBOX', 'UNREAD'],
    q='from:github.com',
    maxResults=50
).execute()
```

### Python — Search Threads

```python
threads = service.users().threads().list(
    userId='me',
    q='subject:"project update" newer_than:7d',
    maxResults=20
).execute()

# Get full thread with all messages
for t in threads.get('threads', []):
    thread = service.users().threads().get(
        userId='me', id=t['id'], format='full'
    ).execute()
    for msg in thread['messages']:
        print(get_header(msg, 'Subject'))
```

### Node.js — Search by Query

```javascript
async function searchMessages(gmail, query, maxResults = 100) {
  const messages = [];
  let pageToken = null;

  do {
    const res = await gmail.users.messages.list({
      userId: 'me',
      q: query,
      maxResults,
      pageToken,
    });
    messages.push(...(res.data.messages || []));
    pageToken = res.data.nextPageToken;
  } while (pageToken && messages.length < maxResults);

  return messages;
}
```

---

## Common Query Patterns

| Goal | Query |
|------|-------|
| Unread emails in inbox | `in:inbox is:unread` |
| Emails from a person in last week | `from:alice@example.com newer_than:7d` |
| Large attachments | `has:attachment larger:10M` |
| PDF attachments from Q1 2024 | `filename:pdf after:2024/01/01 before:2024/04/01` |
| Emails to a mailing list | `list:dev-team@company.com` |
| Starred and unread | `is:starred is:unread` |
| Exclude notifications | `-from:noreply@ -from:no-reply@` |
| Emails in a custom label | `label:projects-active` |
| Exact phrase in subject | `subject:"quarterly review"` |
| All emails except chats | `-in:chats` |
| Emails with Google Drive links | `has:drive` |
