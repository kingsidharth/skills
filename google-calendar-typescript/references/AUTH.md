# Auth & OAuth2

## OAuth2 flow (Desktop / CLI apps)

1. Create OAuth 2.0 Client ID (Desktop type) in Google Cloud Console.
2. Download `credentials.json`.
3. Use `@google-cloud/local-auth` or build a custom flow with `google.auth.OAuth2`.

### Using `@google-cloud/local-auth`

```ts
import { authenticate } from "@google-cloud/local-auth";
import { google } from "googleapis";
import path from "node:path";

const auth = await authenticate({
  scopes: ["https://www.googleapis.com/auth/calendar"],
  keyfilePath: path.join(process.cwd(), "credentials.json"),
});
const calendar = google.calendar({ version: "v3", auth });
```

Opens a browser, handles consent, returns an authenticated client. Tokens are NOT persisted by default — store and reload yourself.

### Manual OAuth2 with token persistence

```ts
import { google } from "googleapis";
import fs from "node:fs/promises";

const oauth2Client = new google.auth.OAuth2(
  CLIENT_ID,
  CLIENT_SECRET,
  REDIRECT_URI // "http://localhost:3000/callback" for web
);

// Generate auth URL
const authUrl = oauth2Client.generateAuthUrl({
  access_type: "offline",    // required for refresh token
  prompt: "consent",          // force consent to always get refresh_token
  scope: ["https://www.googleapis.com/auth/calendar"],
});

// After user consents, exchange code for tokens
const { tokens } = await oauth2Client.getToken(code);
oauth2Client.setCredentials(tokens);

// Persist tokens
await fs.writeFile("token.json", JSON.stringify(tokens));

// On next startup, load and set
const saved = JSON.parse(await fs.readFile("token.json", "utf-8"));
oauth2Client.setCredentials(saved);
```

## Token refresh

The `googleapis` client auto-refreshes access tokens if a valid `refresh_token` is present. Listen for new tokens:

```ts
oauth2Client.on("tokens", async (tokens) => {
  const existing = JSON.parse(await fs.readFile("token.json", "utf-8"));
  await fs.writeFile("token.json", JSON.stringify({ ...existing, ...tokens }));
});
```

- `refresh_token` is only returned on first authorization or when `prompt: "consent"` is set.
- Access tokens expire after ~1 hour. The client handles refresh transparently.
- If refresh fails (revoked/expired refresh token), re-run the full OAuth flow.

## Web app flow

Use Web Application client type. Set redirect URI to your callback endpoint. Exchange auth code server-side. Store tokens in DB per user.

## Service accounts

Service accounts can't access personal calendars unless domain-wide delegation is configured. Always use OAuth2 user credentials for personal calendar integrations.

Caution: if a service account creates a calendar, it becomes the data owner — ownership cannot be transferred.

## Multi-account support

Maintain separate OAuth2 clients (or separate stored token sets) per account. Each token set maps to one Google account. Enumerate calendars per account via `calendarList.list()`.
