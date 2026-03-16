# Labels & Filters Management

> Labels guide: https://developers.google.com/workspace/gmail/api/guides/labels
> Filters guide: https://developers.google.com/workspace/gmail/api/guides/filter_settings

## Labels

Labels are Gmail's organizational mechanism. They have a many-to-many relationship with messages — a message can have multiple labels, and a label can apply to many messages.

### Creating a Label

**Scope:** `https://www.googleapis.com/auth/gmail.labels`

```python
label_body = {
    'name': 'Projects/Active',           # Use / for nesting
    'labelListVisibility': 'labelShow',   # 'labelShow', 'labelShowIfUnread', 'labelHide'
    'messageListVisibility': 'show',      # 'show' or 'hide'
    'color': {
        'textColor': '#ffffff',
        'backgroundColor': '#4a86e8'
    }
}
label = service.users().labels().create(userId='me', body=label_body).execute()
print(f"Created label: {label['id']} — {label['name']}")
```

```javascript
const label = await gmail.users.labels.create({
  userId: 'me',
  requestBody: {
    name: 'Projects/Active',
    labelListVisibility: 'labelShow',
    messageListVisibility: 'show',
    color: { textColor: '#ffffff', backgroundColor: '#4a86e8' },
  },
});
```

### Listing Labels

```python
results = service.users().labels().list(userId='me').execute()
for label in results.get('labels', []):
    print(f"{label['id']}: {label['name']} ({label['type']})")
```

### Updating a Label

```python
service.users().labels().update(
    userId='me',
    id='Label_42',
    body={
        'name': 'Projects/Completed',
        'color': { 'textColor': '#000000', 'backgroundColor': '#cccccc' }
    }
).execute()
```

### Deleting a Label

```python
service.users().labels().delete(userId='me', id='Label_42').execute()
```

---

## Applying / Removing Labels

### On a Single Message

```python
service.users().messages().modify(
    userId='me',
    id='msg123',
    body={
        'addLabelIds': ['Label_42', 'STARRED'],
        'removeLabelIds': ['INBOX']           # Removing INBOX = archiving
    }
).execute()
```

```javascript
await gmail.users.messages.modify({
  userId: 'me',
  id: 'msg123',
  requestBody: {
    addLabelIds: ['Label_42', 'STARRED'],
    removeLabelIds: ['INBOX'],
  },
});
```

### On an Entire Thread

Applies to **all existing messages** in the thread.

```python
service.users().threads().modify(
    userId='me',
    id='thread456',
    body={
        'addLabelIds': ['Label_42'],
        'removeLabelIds': ['UNREAD']
    }
).execute()
```

**Important:** New messages added to the thread later do NOT inherit the label. You must re-apply after new messages arrive.

### Batch Label Application

```python
# Mark multiple messages as read and archive
msg_ids = ['msg1', 'msg2', 'msg3', 'msg4']
batch = service.new_batch_http_request()
for msg_id in msg_ids:
    batch.add(service.users().messages().modify(
        userId='me', id=msg_id,
        body={'removeLabelIds': ['INBOX', 'UNREAD']}
    ))
batch.execute()
```

### Common Label Operations

| Action | addLabelIds | removeLabelIds |
|--------|-------------|----------------|
| Archive | — | `['INBOX']` |
| Mark read | — | `['UNREAD']` |
| Mark unread | `['UNREAD']` | — |
| Star | `['STARRED']` | — |
| Unstar | — | `['STARRED']` |
| Mark important | `['IMPORTANT']` | — |
| Move to inbox | `['INBOX']` | — |
| Trash | `['TRASH']` | — |
| Apply custom label | `['Label_42']` | — |

---

## Filters (Server-Side Rules)

Filters automatically apply actions to incoming messages that match criteria. They operate on **individual messages**, not threads.

### Creating a Filter

**Scope:** `https://www.googleapis.com/auth/gmail.settings.basic`

```python
filter_body = {
    'criteria': {
        'from': 'notifications@github.com',
        'subject': '',
        'query': '',
        'negatedQuery': '',
        'hasAttachment': False,
        'excludeChats': True,
        'size': 0,
        'sizeComparison': 'unspecified'
    },
    'action': {
        'addLabelIds': ['Label_42'],
        'removeLabelIds': ['INBOX'],       # Archive
        'forward': ''                       # Must be a verified address
    }
}
result = service.users().settings().filters().create(
    userId='me', body=filter_body
).execute()
print(f"Created filter: {result['id']}")
```

```javascript
const filter = await gmail.users.settings.filters.create({
  userId: 'me',
  requestBody: {
    criteria: {
      from: 'notifications@github.com',
      excludeChats: true,
    },
    action: {
      addLabelIds: ['Label_42'],
      removeLabelIds: ['INBOX'],
    },
  },
});
```

### Common Filter Patterns

**Archive and label mailing list emails:**
```python
{
    'criteria': { 'list': 'dev-team@company.com' },
    'action': {
        'addLabelIds': ['Label_DevTeam'],
        'removeLabelIds': ['INBOX']
    }
}
```

**Label large emails with attachments:**
```python
{
    'criteria': {
        'hasAttachment': True,
        'size': 10000000,
        'sizeComparison': 'larger'
    },
    'action': { 'addLabelIds': ['Label_LargeAttachments'] }
}
```

**Use full Gmail search syntax in criteria:**
```python
{
    'criteria': {
        'query': 'from:jira@company.atlassian.net subject:"[PROJ-"'
    },
    'action': {
        'addLabelIds': ['Label_Jira'],
        'removeLabelIds': ['INBOX']
    }
}
```

### Listing Filters

```python
results = service.users().settings().filters().list(userId='me').execute()
for f in results.get('filter', []):
    print(f"Filter {f['id']}: from={f['criteria'].get('from', '')} → labels={f['action'].get('addLabelIds', [])}")
```

### Deleting a Filter

```python
service.users().settings().filters().delete(userId='me', id='filter_id').execute()
```

**Note:** Filters cannot be updated — delete and recreate with new criteria/actions.
