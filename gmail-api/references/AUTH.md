# OAuth 2.0 Authentication & Authorization

> Official guide: https://developers.google.com/workspace/gmail/api/auth/web-server
> OAuth 2.0 overview: https://developers.google.com/identity/protocols/oauth2

## Setup Checklist

1. Create a Google Cloud project at https://console.cloud.google.com
2. Enable the Gmail API: APIs & Services → Library → search "Gmail API" → Enable
3. Configure OAuth Consent Screen (Google Auth Platform → Branding):
   - Set app name, support email, contact info
   - Choose **Internal** (Workspace org only) or **External** (any Google account)
   - Add required scopes under Data Access
4. Create OAuth 2.0 Client credentials (Google Auth Platform → Clients):
   - **Desktop app** — for CLI tools, scripts, desktop applications
   - **Web application** — for server-side web apps (set authorized redirect URIs)
5. Download the credentials JSON file, save as `credentials.json`

---

## Scopes

Request the **narrowest scope(s)** needed. If you change scopes later, delete stored tokens — users must re-consent.

| Scope | Access |
|-------|--------|
| `gmail.readonly` | Read messages, labels, settings, threads |
| `gmail.modify` | Read + write (labels, send, delete) — excludes permanent delete and settings changes |
| `gmail.compose` | Create drafts, send messages |
| `gmail.send` | Send only (no read, no modify) |
| `gmail.labels` | Create/read/update/delete labels |
| `gmail.settings.basic` | Manage filters, forwarding, IMAP/POP settings |
| `gmail.settings.sharing` | Manage delegates and sharing (admin) |
| `gmail.metadata` | Read message metadata (headers, labels) but NOT body content. `format=FULL` and `format=RAW` will fail with this scope. |
| `mail.google.com` | Full access — all Gmail operations. Only use when absolutely necessary. |

All scopes are prefixed with `https://www.googleapis.com/auth/`.

---

## Token Lifecycle

### Access Token

- Short-lived: ~1 hour (`expires_in: 3600` in the token response)
- Sent as `Authorization: Bearer <token>` with every API request
- When expired, use the refresh token to get a new one — no user interaction needed
- **Never refresh on every API call** — cache and reuse until expired

### Refresh Token

- Long-lived (persists until explicitly revoked or expired by policy)
- Used to obtain new access tokens silently
- **Only returned on the first authorization** — unless you force re-consent with `prompt=consent`
- Store securely in a database or encrypted file

### Token Response Shape

```json
{
  "access_token": "ya29.a0AfH6SMB...",
  "expires_in": 3920,
  "refresh_token": "1//0dx...",
  "scope": "https://www.googleapis.com/auth/gmail.readonly",
  "token_type": "Bearer"
}
```

### Getting a Refresh Token

The refresh token is returned ONLY when:
- `access_type=offline` is set in the auth URL, AND
- It's the **first** authorization for this user/client pair — OR
- `prompt=consent` is set (forces the consent screen to show again)

```
https://accounts.google.com/o/oauth2/v2/auth?
  client_id=YOUR_CLIENT_ID&
  redirect_uri=YOUR_REDIRECT_URI&
  response_type=code&
  scope=https://www.googleapis.com/auth/gmail.readonly&
  access_type=offline&
  prompt=consent
```

### Refresh Token Expiration

Refresh tokens can stop working for these reasons:

| Reason | Details |
|--------|---------|
| **Testing mode (7-day expiry)** | OAuth consent screen is "External" + publishing status "Testing" → tokens expire after 7 days. Fix: publish to "In Production". |
| **User revoked access** | User goes to Google Account → Security → Third-party apps → removes your app |
| **Password change** | If the token includes any **Gmail scope**, a password reset revokes it |
| **6 months unused** | Tokens not used for 6 months are revoked by Google |
| **Token limit exceeded** | ~50–100 live refresh tokens per user per OAuth client. Oldest are silently invalidated. |
| **Admin policy** | Workspace admins can restrict scopes (`admin_policy_enforced`) |
| **Session control** | Workspace org session length policies |

### Handling `invalid_grant`

When a refresh fails:
1. Delete the stored token (e.g., remove `token.json`)
2. Re-initiate the OAuth flow (user must re-authorize)
3. Store the new tokens

---

## Development Environments

### Localhost (Desktop App Flow — No Tunnel Needed)

For CLI scripts and desktop apps, the Google client libraries handle localhost redirect automatically.

**Python:**
```python
from google_auth_oauthlib.flow import InstalledAppFlow

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']
flow = InstalledAppFlow.from_client_secrets_file('credentials.json', SCOPES)
creds = flow.run_local_server(port=0)  # port=0 picks random available port
# Opens browser → consent screen → redirects to localhost → captures code
```

**Node.js:**
```javascript
import { authenticate } from '@google-cloud/local-auth';

const auth = await authenticate({
  scopes: ['https://www.googleapis.com/auth/gmail.readonly'],
  keyfilePath: './credentials.json',
});
```

**Console config:** Add `http://localhost` (and/or `http://localhost:PORT`) as an authorized redirect URI in your OAuth client settings.

### Localhost (Web Server Flow — Tunnel Required)

When your app uses a web server auth flow (redirect to a callback URL) but runs locally:

**With Cloudflare Tunnel:**
```bash
# Install cloudflared, then:
cloudflared tunnel --url http://localhost:3000
# Outputs: https://random-name.trycloudflare.com
```
Add `https://random-name.trycloudflare.com/oauth2callback` as an authorized redirect URI.

**With ngrok:**
```bash
ngrok http 3000
# Outputs: https://xxxx.ngrok.io
```
Add `https://xxxx.ngrok.io/oauth2callback` as an authorized redirect URI.

### When to Use Which

| Scenario | Approach |
|----------|----------|
| CLI script / desktop app testing | Localhost redirect (no tunnel) |
| Web app dev (needs real URL redirect) | Cloudflare Tunnel or ngrok |
| Pub/Sub push webhook in dev | Tunnel required — Google can't reach localhost |
| Production | Real server with HTTPS |

---

## Full Auth Flow: Python (Official Quickstart Pattern)

```python
import os
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']

def get_gmail_service():
    creds = None

    # token.json stores access + refresh tokens from prior auth
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)

    # If no valid creds, refresh or re-auth
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                'credentials.json', SCOPES
            )
            creds = flow.run_local_server(port=0)
        # Save for next run
        with open('token.json', 'w') as token:
            token.write(creds.to_json())

    return build('gmail', 'v1', credentials=creds)
```

## Full Auth Flow: Node.js (Official Quickstart Pattern)

```javascript
import path from 'node:path';
import process from 'node:process';
import { authenticate } from '@google-cloud/local-auth';
import { google } from 'googleapis';

const SCOPES = ['https://www.googleapis.com/auth/gmail.readonly'];
const CREDENTIALS_PATH = path.join(process.cwd(), 'credentials.json');

async function getGmailService() {
  const auth = await authenticate({
    scopes: SCOPES,
    keyfilePath: CREDENTIALS_PATH,
  });
  return google.gmail({ version: 'v1', auth });
}
```

## Web Server Auth Flow (Python/Flask)

```python
from google_auth_oauthlib.flow import Flow

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']
REDIRECT_URI = 'https://yourapp.com/oauth2callback'

# Step 1: Generate auth URL
flow = Flow.from_client_secrets_file('credentials.json', scopes=SCOPES)
flow.redirect_uri = REDIRECT_URI
authorization_url, state = flow.authorization_url(
    access_type='offline',
    include_granted_scopes='true',
    prompt='consent'  # Forces refresh token return
)
# Redirect user to authorization_url

# Step 2: Handle callback
flow = Flow.from_client_secrets_file('credentials.json', scopes=SCOPES, state=state)
flow.redirect_uri = REDIRECT_URI
flow.fetch_token(authorization_response=request.url)  # Exchange code for tokens
creds = flow.credentials
# Store creds.refresh_token in your database
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `"This app isn't verified"` | Sensitive scopes, app not verified | Click Advanced → Go to (unsafe) during dev; submit for verification for production |
| `credentials.json not found` | Haven't downloaded OAuth client credentials | Download from Cloud Console → Clients |
| `Token has been expired or revoked` | Access/refresh token invalid | Delete `token.json`, re-authorize. Check if in Testing mode (7-day expiry). |
| `redirect_uri_mismatch` | Redirect URI doesn't exactly match console config | Ensure exact match including port and trailing slash |
| `origin_mismatch` (JS) | JavaScript origin not registered | Add your domain to authorized JS origins |
| `idpiframe_initialization_failed` | 3rd-party cookies blocked | Enable cookies for `accounts.google.com` |
| `invalid_grant` | Many causes (see Token Expiration table) | Delete stored token, re-auth |
| Refresh token not returned | Missing `access_type=offline` or not first auth | Add `prompt=consent` to force re-consent |
