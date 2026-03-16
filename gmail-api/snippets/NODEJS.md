# Node.js SDK Quick Reference

> Quickstart: https://developers.google.com/workspace/gmail/api/quickstart/nodejs
> GitHub: https://github.com/googleapis/google-api-nodejs-client

## Install

```bash
npm install googleapis @google-cloud/local-auth
```

## Authentication Boilerplate

```javascript
import path from 'node:path';
import process from 'node:process';
import fs from 'node:fs';
import { authenticate } from '@google-cloud/local-auth';
import { google } from 'googleapis';

const SCOPES = ['https://www.googleapis.com/auth/gmail.modify'];
const CREDENTIALS_PATH = path.join(process.cwd(), 'credentials.json');
const TOKEN_PATH = path.join(process.cwd(), 'token.json');

async function authorize() {
  // Try loading saved credentials
  if (fs.existsSync(TOKEN_PATH)) {
    const content = fs.readFileSync(TOKEN_PATH, 'utf-8');
    const credentials = JSON.parse(content);
    return google.auth.fromJSON(credentials);
  }

  // Authenticate via browser
  const client = await authenticate({
    scopes: SCOPES,
    keyfilePath: CREDENTIALS_PATH,
  });

  // Save for future use
  if (client.credentials) {
    const keys = JSON.parse(fs.readFileSync(CREDENTIALS_PATH, 'utf-8'));
    const key = keys.installed || keys.web;
    const payload = JSON.stringify({
      type: 'authorized_user',
      client_id: key.client_id,
      client_secret: key.client_secret,
      refresh_token: client.credentials.refresh_token,
    });
    fs.writeFileSync(TOKEN_PATH, payload);
  }

  return client;
}

async function getGmail() {
  const auth = await authorize();
  return google.gmail({ version: 'v1', auth });
}
```

## API Method Signatures

### Messages

```javascript
// List
await gmail.users.messages.list({
  userId: 'me',
  q: 'from:github.com',
  labelIds: ['INBOX'],
  maxResults: 100,
  pageToken: '...',
  fields: 'messages/id,nextPageToken',
});

// Get
await gmail.users.messages.get({
  userId: 'me',
  id: 'msg123',
  format: 'full',         // full | minimal | raw | metadata
  metadataHeaders: ['From', 'Subject'],
  fields: 'id,labelIds,payload/headers',
});

// Send
await gmail.users.messages.send({
  userId: 'me',
  requestBody: { raw: base64urlEncodedMessage },
});

// Modify labels
await gmail.users.messages.modify({
  userId: 'me',
  id: 'msg123',
  requestBody: {
    addLabelIds: ['STARRED'],
    removeLabelIds: ['UNREAD'],
  },
});

// Trash / Untrash / Delete
await gmail.users.messages.trash({ userId: 'me', id: 'msg123' });
await gmail.users.messages.untrash({ userId: 'me', id: 'msg123' });
await gmail.users.messages.delete({ userId: 'me', id: 'msg123' });  // Permanent!

// Get attachment
await gmail.users.messages.attachments.get({
  userId: 'me',
  messageId: 'msg123',
  id: 'attachment_id',
});
```

### Threads

```javascript
await gmail.users.threads.list({ userId: 'me', q: 'subject:report' });
await gmail.users.threads.get({ userId: 'me', id: 'thread123', format: 'full' });
await gmail.users.threads.modify({
  userId: 'me', id: 'thread123',
  requestBody: { addLabelIds: ['Label_1'], removeLabelIds: ['INBOX'] },
});
```

### Labels

```javascript
await gmail.users.labels.list({ userId: 'me' });
await gmail.users.labels.create({
  userId: 'me',
  requestBody: {
    name: 'MyLabel',
    labelListVisibility: 'labelShow',
    messageListVisibility: 'show',
  },
});
await gmail.users.labels.update({
  userId: 'me', id: 'Label_42',
  requestBody: { name: 'Renamed' },
});
await gmail.users.labels.delete({ userId: 'me', id: 'Label_42' });
```

### Drafts

```javascript
await gmail.users.drafts.list({ userId: 'me' });
await gmail.users.drafts.create({
  userId: 'me',
  requestBody: { message: { raw: '...' } },
});
await gmail.users.drafts.update({
  userId: 'me', id: 'draft123',
  requestBody: { message: { raw: '...' } },
});
await gmail.users.drafts.send({
  userId: 'me',
  requestBody: { id: 'draft123' },
});
```

### History (Sync)

```javascript
await gmail.users.history.list({
  userId: 'me',
  startHistoryId: '12345',
  historyTypes: ['messageAdded', 'messageDeleted', 'labelAdded', 'labelRemoved'],
  maxResults: 100,
});
```

### Watch (Push Notifications)

```javascript
await gmail.users.watch({
  userId: 'me',
  requestBody: {
    topicName: 'projects/my-project/topics/gmail-push',
    labelIds: ['INBOX'],
    labelFilterBehavior: 'INCLUDE',
  },
});
await gmail.users.stop({ userId: 'me' });
```

### Filters

```javascript
await gmail.users.settings.filters.list({ userId: 'me' });
await gmail.users.settings.filters.create({
  userId: 'me',
  requestBody: {
    criteria: { from: 'noreply@github.com' },
    action: { addLabelIds: ['Label_1'], removeLabelIds: ['INBOX'] },
  },
});
await gmail.users.settings.filters.delete({ userId: 'me', id: 'filter123' });
```

## Utility: Base64url Encode

```javascript
function encodeMessage(message) {
  return Buffer.from(message)
    .toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');
}
```

## Utility: Extract Header

```javascript
function getHeader(message, name) {
  const headers = message.payload?.headers || [];
  const header = headers.find(h => h.name.toLowerCase() === name.toLowerCase());
  return header?.value || null;
}
```

## Utility: Decode Body

```javascript
function getBodyText(message) {
  const payload = message.payload || {};
  const parts = payload.parts || [];

  // Simple message
  if (!parts.length && payload.body?.data) {
    return Buffer.from(payload.body.data, 'base64').toString('utf-8');
  }

  // Multipart — find text/plain
  for (const part of parts) {
    if (part.mimeType === 'text/plain' && part.body?.data) {
      return Buffer.from(part.body.data, 'base64').toString('utf-8');
    }
    for (const sub of (part.parts || [])) {
      if (sub.mimeType === 'text/plain' && sub.body?.data) {
        return Buffer.from(sub.body.data, 'base64').toString('utf-8');
      }
    }
  }

  return null;
}
```

## Utility: Send Email

```javascript
async function sendEmail(gmail, to, subject, body) {
  const message = [
    `To: ${to}`,
    `Subject: ${subject}`,
    'Content-Type: text/plain; charset=utf-8',
    'MIME-Version: 1.0',
    '',
    body,
  ].join('\n');

  const res = await gmail.users.messages.send({
    userId: 'me',
    requestBody: { raw: encodeMessage(message) },
  });
  return res.data;
}
```

## Note on Batch Requests

The `googleapis` Node.js library does **not** have a built-in batch method. Options:

1. **Raw HTTP multipart** — construct `multipart/mixed` request manually (see PERFORMANCE.md)
2. **Parallel promises** — use `Promise.all` with individual requests (still multiple HTTP calls, but concurrent)
3. **Third-party library** — e.g., `gmail-batch-stream` (unmaintained as of 2024)

```javascript
// Parallel approach (not true batch, but concurrent)
const messageIds = ['msg1', 'msg2', 'msg3'];
const messages = await Promise.all(
  messageIds.map(id =>
    gmail.users.messages.get({ userId: 'me', id, format: 'metadata' })
  )
);
```
