# Push Notifications via Cloud Pub/Sub

> Official guide: https://developers.google.com/workspace/gmail/api/guides/push
> Pub/Sub docs: https://cloud.google.com/pubsub/docs

Gmail uses **Google Cloud Pub/Sub** for real-time mailbox change notifications. Notifications are lightweight — they contain only the user's email address and a `historyId`. You must call `history.list` to get the actual changes.

---

## Setup Steps

### 1. Create a Pub/Sub Topic

In your Google Cloud project:

```bash
# Using gcloud CLI
gcloud pubsub topics create gmail-push

# Or via console: Pub/Sub → Topics → Create Topic
```

Topic name format: `projects/{project-id}/topics/{topic-name}`

**Best practice:** Use a single topic for all Gmail API push notifications in your application.

### 2. Grant Gmail Publish Permissions

Gmail needs permission to publish to your topic:

```bash
gcloud pubsub topics add-iam-policy-binding gmail-push \
  --member="serviceAccount:gmail-api-push@system.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

Or via Console: Topic → Permissions → Add Principal → `gmail-api-push@system.gserviceaccount.com` → Role: Pub/Sub Publisher.

### 3. Create a Subscription

**Push subscription** (Google sends to your webhook):

```bash
gcloud pubsub subscriptions create gmail-push-sub \
  --topic=gmail-push \
  --push-endpoint=https://yourapp.com/pubsub-webhook
```

**Pull subscription** (your app polls for messages):

```bash
gcloud pubsub subscriptions create gmail-pull-sub \
  --topic=gmail-push
```

### 4. Start Watching a Mailbox

```python
watch_request = {
    'topicName': 'projects/my-project/topics/gmail-push',
    'labelIds': ['INBOX'],           # Optional: filter by label
    'labelFilterBehavior': 'INCLUDE'  # 'INCLUDE' or 'EXCLUDE'
}
response = service.users().watch(userId='me', body=watch_request).execute()
# response: { "historyId": "12345", "expiration": "1700000000000" }
```

```javascript
const res = await gmail.users.watch({
  userId: 'me',
  requestBody: {
    topicName: 'projects/my-project/topics/gmail-push',
    labelIds: ['INBOX'],
    labelFilterBehavior: 'INCLUDE',
  },
});
// res.data: { historyId: '12345', expiration: '1700000000000' }
```

**Scope required:** `gmail.readonly`, `gmail.modify`, or `mail.google.com`.

---

## Push vs. Pull Subscriptions

| Aspect | Push | Pull |
|--------|------|------|
| **Delivery** | Google POSTs JSON to your HTTPS endpoint | Your app polls the subscription |
| **Infrastructure** | Publicly reachable HTTPS server | No public endpoint needed |
| **Latency** | Near real-time (seconds) | Depends on poll frequency |
| **Localhost dev** | Requires tunnel (Cloudflare/ngrok) | Works anywhere |
| **Best for** | Production web apps, serverless (Cloud Functions, Cloud Run) | Background workers, scripts, desktop apps |
| **Backpressure** | Return non-2xx to NACK; Pub/Sub retries with backoff | Explicit ACK/NACK control |

---

## Handling Push Webhook

The POST body:

```json
{
  "message": {
    "data": "eyJlbWFpbEFkZHJlc3MiOiAidXNlckBleGFtcGxlLmNvbSIsICJoaXN0b3J5SWQiOiAiMTIzNDU2Nzg5MCJ9",
    "messageId": "2070443601311540",
    "publishTime": "2021-02-26T19:13:55.749Z"
  },
  "subscription": "projects/myproject/subscriptions/mysubscription"
}
```

### Python (Flask)

```python
import base64
import json
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/pubsub-webhook', methods=['POST'])
def pubsub_webhook():
    envelope = request.get_json()
    if not envelope or 'message' not in envelope:
        return 'Bad Request', 400

    # Decode notification
    data = base64.urlsafe_b64decode(envelope['message']['data']).decode('utf-8')
    notification = json.loads(data)

    email_address = notification['emailAddress']
    history_id = notification['historyId']

    # Trigger partial sync for this user
    trigger_partial_sync(email_address, history_id)

    # Return 200 to acknowledge (prevents Pub/Sub retry)
    return '', 200
```

### Node.js (Express)

```javascript
import express from 'express';

const app = express();
app.use(express.json());

app.post('/pubsub-webhook', (req, res) => {
  const message = req.body?.message;
  if (!message?.data) return res.status(400).send('Bad Request');

  const data = JSON.parse(
    Buffer.from(message.data, 'base64').toString('utf-8')
  );

  const { emailAddress, historyId } = data;
  triggerPartialSync(emailAddress, historyId);

  res.status(200).send(); // ACK
});
```

### Pull Subscription (Python)

```python
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = 'projects/my-project/subscriptions/gmail-pull-sub'

def callback(message):
    data = json.loads(message.data.decode('utf-8'))
    email_address = data['emailAddress']
    history_id = data['historyId']

    trigger_partial_sync(email_address, history_id)
    message.ack()

subscriber.subscribe(subscription_path, callback=callback)
```

---

## Watch Renewal

**Critical:** Watch requests expire after ~7 days. Renew proactively.

### Cron / Scheduled Job (run every 3 days)

```python
import time

def renew_watch_if_needed(service, stored_expiration_ms):
    now_ms = int(time.time() * 1000)
    one_day_ms = 24 * 60 * 60 * 1000

    if now_ms > (stored_expiration_ms - one_day_ms):
        response = service.users().watch(userId='me', body={
            'topicName': 'projects/my-project/topics/gmail-push',
            'labelIds': ['INBOX'],
        }).execute()
        new_expiration = int(response['expiration'])
        save_expiration(new_expiration)
        return new_expiration

    return stored_expiration_ms
```

### Stop Watching

```python
service.users().stop(userId='me').execute()
```

```javascript
await gmail.users.stop({ userId: 'me' });
```

---

## Important Limitations

| Limitation | Details |
|-----------|---------|
| **Max notification rate** | 1 event/second per user — excess events are dropped |
| **Payload is minimal** | Only `emailAddress` + `historyId` — no message content |
| **Notifications may be delayed or dropped** | Always have a fallback poll mechanism |
| **Watch expiration** | ~7 days — must renew proactively |
| **No message filtering in notification** | `labelIds` on watch filters which changes trigger notifications, but the notification itself has no details |
| **Be careful of notification loops** | Don't trigger another notification from within your notification handler (e.g., by modifying the same mailbox) |

---

## Localhost Development with Push

For push subscriptions during development, Google must reach your webhook. Options:

**Cloudflare Tunnel:**
```bash
cloudflared tunnel --url http://localhost:3000
# Use the generated URL as your push endpoint
```

**ngrok:**
```bash
ngrok http 3000
# Use the generated URL as your push endpoint
```

**Alternative: Use pull subscription** for local dev — no public endpoint needed. Switch to push in production.
