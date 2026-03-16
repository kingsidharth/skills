# Sending Email, Drafts & Attachments

> Drafts guide: https://developers.google.com/workspace/gmail/api/guides/drafts
> Sending guide: https://developers.google.com/workspace/gmail/api/guides/sending
> Uploads guide: https://developers.google.com/workspace/gmail/api/guides/uploads

## Sending Email

Messages must be RFC 2822 formatted and base64url encoded.

### Python — Simple Send

```python
import base64
from email.message import EmailMessage
from googleapiclient.errors import HttpError

def send_message(service, to, subject, body_text):
    """Send a plain text email."""
    message = EmailMessage()
    message.set_content(body_text)
    message['To'] = to
    message['From'] = 'me'  # Gmail sets this from the authenticated account
    message['Subject'] = subject

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()

    try:
        sent = service.users().messages().send(
            userId='me',
            body={'raw': encoded}
        ).execute()
        print(f"Sent message ID: {sent['id']}")
        return sent
    except HttpError as error:
        print(f"Error sending: {error}")
        return None
```

### Python — Send HTML Email

```python
def send_html_message(service, to, subject, html_body, plain_body=None):
    """Send an HTML email with optional plain text fallback."""
    message = EmailMessage()
    message['To'] = to
    message['From'] = 'me'
    message['Subject'] = subject

    # Set plain text first, then add HTML alternative
    message.set_content(plain_body or 'Please view this email in an HTML-capable client.')
    message.add_alternative(html_body, subtype='html')

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()
    return service.users().messages().send(
        userId='me', body={'raw': encoded}
    ).execute()
```

### Python — Reply to a Thread

To reply within an existing thread, set the `threadId` and proper headers:

```python
def reply_to_message(service, original_msg, reply_body):
    """Reply to an existing message in its thread."""
    headers = {h['name']: h['value'] for h in original_msg['payload']['headers']}

    message = EmailMessage()
    message.set_content(reply_body)
    message['To'] = headers.get('From', '')
    message['From'] = 'me'
    message['Subject'] = 'Re: ' + headers.get('Subject', '')
    message['In-Reply-To'] = headers.get('Message-ID', '')
    message['References'] = headers.get('Message-ID', '')

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()
    return service.users().messages().send(
        userId='me',
        body={
            'raw': encoded,
            'threadId': original_msg['threadId']  # Must match
        }
    ).execute()
```

### Node.js — Send

```javascript
async function sendMessage(gmail, to, subject, body) {
  const message = [
    `To: ${to}`,
    'Content-Type: text/plain; charset=utf-8',
    'MIME-Version: 1.0',
    `Subject: ${subject}`,
    '',
    body,
  ].join('\n');

  const encoded = Buffer.from(message)
    .toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');

  const res = await gmail.users.messages.send({
    userId: 'me',
    requestBody: { raw: encoded },
  });
  return res.data;
}
```

---

## Drafts

A draft is a container with a stable `id`. The inner message `id` changes on every update.

### Creating a Draft

```python
def create_draft(service, to, subject, body_text):
    """Create a draft email."""
    message = EmailMessage()
    message.set_content(body_text)
    message['To'] = to
    message['From'] = 'me'
    message['Subject'] = subject

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()

    draft = service.users().drafts().create(
        userId='me',
        body={'message': {'raw': encoded}}
    ).execute()
    print(f"Draft ID: {draft['id']}")
    return draft
```

### Updating a Draft

The existing message inside the draft is **destroyed and replaced**:

```python
def update_draft(service, draft_id, to, subject, new_body):
    """Replace the content of an existing draft."""
    message = EmailMessage()
    message.set_content(new_body)
    message['To'] = to
    message['Subject'] = subject

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()

    updated = service.users().drafts().update(
        userId='me',
        id=draft_id,
        body={'message': {'raw': encoded}}
    ).execute()
    return updated
```

### Reading a Draft

```python
# Get with format=RAW to get the full MIME for re-editing
draft = service.users().drafts().get(
    userId='me', id=draft_id, format='raw'
).execute()
# draft['message']['raw'] contains the base64url RFC 2822 message
```

### Sending a Draft

```python
sent = service.users().drafts().send(
    userId='me',
    body={'id': draft_id}
).execute()
# Draft is deleted; new message created with SENT label
# sent['id'] is the new message's ID
```

You can also update content while sending:

```python
sent = service.users().drafts().send(
    userId='me',
    body={
        'id': draft_id,
        'message': {'raw': updated_encoded_message}
    }
).execute()
```

### Listing Drafts

```python
results = service.users().drafts().list(userId='me').execute()
for draft in results.get('drafts', []):
    print(f"Draft {draft['id']} — Message {draft['message']['id']}")
```

---

## Attachments

### Sending with Attachments

```python
def send_with_attachment(service, to, subject, body_text, filepath):
    """Send an email with a file attachment."""
    import mimetypes

    message = EmailMessage()
    message['To'] = to
    message['From'] = 'me'
    message['Subject'] = subject
    message.set_content(body_text)

    # Detect MIME type
    mime_type, _ = mimetypes.guess_type(filepath)
    if mime_type is None:
        mime_type = 'application/octet-stream'
    maintype, subtype = mime_type.split('/', 1)

    with open(filepath, 'rb') as f:
        message.add_attachment(
            f.read(),
            maintype=maintype,
            subtype=subtype,
            filename=filepath.split('/')[-1]
        )

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()
    return service.users().messages().send(
        userId='me', body={'raw': encoded}
    ).execute()
```

### Reading Attachments

Small attachments have data inline; large ones require a separate fetch:

```python
def download_attachments(service, message):
    """Download all attachments from a message."""
    attachments = []
    parts = message.get('payload', {}).get('parts', [])

    for part in parts:
        filename = part.get('filename')
        if not filename:
            continue

        body = part.get('body', {})
        if 'data' in body:
            # Small attachment — data is inline
            file_data = base64.urlsafe_b64decode(body['data'])
        elif 'attachmentId' in body:
            # Large attachment — fetch separately
            att = service.users().messages().attachments().get(
                userId='me',
                messageId=message['id'],
                id=body['attachmentId']
            ).execute()
            file_data = base64.urlsafe_b64decode(att['data'])
        else:
            continue

        attachments.append({'filename': filename, 'data': file_data})

    return attachments
```

### Upload Methods

| Method | Max Size | When to Use |
|--------|----------|-------------|
| Simple upload | 5 MB | Small messages |
| Multipart upload | 5 MB | Metadata + small body |
| Resumable upload | 35 MB | Large attachments, unreliable connections |

### Resumable Upload (Python)

```python
from googleapiclient.http import MediaFileUpload

def send_large_attachment(service, to, subject, body_text, filepath):
    message = EmailMessage()
    message['To'] = to
    message['From'] = 'me'
    message['Subject'] = subject
    message.set_content(body_text)

    # Add attachment to MIME
    import mimetypes
    mime_type, _ = mimetypes.guess_type(filepath)
    maintype, subtype = (mime_type or 'application/octet-stream').split('/', 1)
    with open(filepath, 'rb') as f:
        message.add_attachment(f.read(), maintype=maintype, subtype=subtype,
                               filename=filepath.split('/')[-1])

    encoded = base64.urlsafe_b64encode(message.as_bytes()).decode()

    media = MediaFileUpload(filepath, resumable=True)
    return service.users().messages().send(
        userId='me',
        body={'raw': encoded},
        media_body=media
    ).execute()
```
