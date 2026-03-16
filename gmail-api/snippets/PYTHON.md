# Python SDK Quick Reference

> Quickstart: https://developers.google.com/workspace/gmail/api/quickstart/python
> PyDoc: https://googleapis.github.io/google-api-python-client/docs/dyn/gmail_v1.html

## Install

```bash
pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib
```

## Authentication Boilerplate

```python
import os
import base64
from email.message import EmailMessage

from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

SCOPES = ['https://www.googleapis.com/auth/gmail.modify']

def get_service():
    creds = None
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file('credentials.json', SCOPES)
            creds = flow.run_local_server(port=0)
        with open('token.json', 'w') as token:
            token.write(creds.to_json())
    return build('gmail', 'v1', credentials=creds)
```

## API Method Signatures

### Messages

```python
service.users().messages().list(
    userId='me',
    q='from:github.com',          # Gmail search syntax
    labelIds=['INBOX'],            # Filter by label IDs
    maxResults=100,                # Max 500
    pageToken='...',               # Pagination
    fields='messages/id,nextPageToken'  # Partial response
)

service.users().messages().get(
    userId='me',
    id='msg123',
    format='full',                 # full | minimal | raw | metadata
    metadataHeaders=['From', 'Subject'],  # Only with format=metadata
    fields='id,labelIds,payload/headers'  # Partial response
)

service.users().messages().send(
    userId='me',
    body={'raw': base64url_encoded_rfc2822}
)

service.users().messages().modify(
    userId='me',
    id='msg123',
    body={'addLabelIds': ['STARRED'], 'removeLabelIds': ['UNREAD']}
)

service.users().messages().trash(userId='me', id='msg123')
service.users().messages().untrash(userId='me', id='msg123')
service.users().messages().delete(userId='me', id='msg123')  # Permanent!

service.users().messages().attachments().get(
    userId='me', messageId='msg123', id='attachment_id'
)
```

### Threads

```python
service.users().threads().list(userId='me', q='subject:report', maxResults=20)
service.users().threads().get(userId='me', id='thread123', format='full')
service.users().threads().modify(
    userId='me', id='thread123',
    body={'addLabelIds': ['Label_1'], 'removeLabelIds': ['INBOX']}
)
service.users().threads().trash(userId='me', id='thread123')
service.users().threads().delete(userId='me', id='thread123')  # Permanent!
```

### Labels

```python
service.users().labels().list(userId='me')
service.users().labels().get(userId='me', id='Label_42')
service.users().labels().create(userId='me', body={
    'name': 'MyLabel', 'labelListVisibility': 'labelShow',
    'messageListVisibility': 'show'
})
service.users().labels().update(userId='me', id='Label_42', body={...})
service.users().labels().patch(userId='me', id='Label_42', body={...})
service.users().labels().delete(userId='me', id='Label_42')
```

### Drafts

```python
service.users().drafts().list(userId='me')
service.users().drafts().get(userId='me', id='draft123', format='full')
service.users().drafts().create(userId='me', body={'message': {'raw': '...'}})
service.users().drafts().update(userId='me', id='draft123', body={'message': {'raw': '...'}})
service.users().drafts().delete(userId='me', id='draft123')
service.users().drafts().send(userId='me', body={'id': 'draft123'})
```

### History (Sync)

```python
service.users().history().list(
    userId='me',
    startHistoryId='12345',
    historyTypes=['messageAdded', 'messageDeleted', 'labelAdded', 'labelRemoved'],
    labelId='INBOX',        # Optional: only changes to this label
    maxResults=100
)
```

### Watch (Push Notifications)

```python
service.users().watch(userId='me', body={
    'topicName': 'projects/my-project/topics/gmail-push',
    'labelIds': ['INBOX'],
    'labelFilterBehavior': 'INCLUDE'
})
service.users().stop(userId='me')
```

### Filters

```python
service.users().settings().filters().list(userId='me')
service.users().settings().filters().get(userId='me', id='filter123')
service.users().settings().filters().create(userId='me', body={
    'criteria': {'from': 'noreply@github.com'},
    'action': {'addLabelIds': ['Label_1'], 'removeLabelIds': ['INBOX']}
})
service.users().settings().filters().delete(userId='me', id='filter123')
```

### Batch

```python
batch = service.new_batch_http_request(callback=my_callback)
batch.add(request, request_id='unique_id')
batch.execute()
```

## Utility: Extract Header

```python
def get_header(message, header_name):
    """Extract a header value from a message resource."""
    headers = message.get('payload', {}).get('headers', [])
    for h in headers:
        if h['name'].lower() == header_name.lower():
            return h['value']
    return None
```

## Utility: Decode Body

```python
def get_body_text(message):
    """Extract plain text body from a message."""
    payload = message.get('payload', {})
    parts = payload.get('parts', [])

    # Simple message (no parts)
    if not parts and payload.get('body', {}).get('data'):
        return base64.urlsafe_b64decode(payload['body']['data']).decode('utf-8')

    # Multipart — find text/plain
    for part in parts:
        if part['mimeType'] == 'text/plain' and part.get('body', {}).get('data'):
            return base64.urlsafe_b64decode(part['body']['data']).decode('utf-8')
        # Nested multipart/alternative
        for sub in part.get('parts', []):
            if sub['mimeType'] == 'text/plain' and sub.get('body', {}).get('data'):
                return base64.urlsafe_b64decode(sub['body']['data']).decode('utf-8')

    return None
```
