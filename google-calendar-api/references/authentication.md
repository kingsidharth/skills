# Authentication & Authorization

## Overview

Google Calendar API uses OAuth 2.0 for authentication. Each user grants permission through a consent flow, and your application receives access tokens to make API calls on their behalf.

## OAuth 2.0 Setup Steps

### 1. Create Google Cloud Project

```
1. Go to: https://console.cloud.google.com/
2. Click "Create Project"
3. Enter project name and click "Create"
```

### 2. Enable Calendar API

```
1. Navigate to: APIs & Services > Library
2. Search for "Google Calendar API"
3. Click "Enable"
```

### 3. Configure OAuth Consent Screen

```
1. Go to: APIs & Services > OAuth consent screen
2. Choose User Type:
   - Internal: Only users in your Google Workspace organization
   - External: Any Google account user
3. Fill in required fields:
   - App name
   - User support email
   - Developer contact information
4. Add scopes (see Scopes section below)
5. Save and continue
```

### 4. Create OAuth 2.0 Credentials

```
1. Go to: APIs & Services > Credentials
2. Click "Create Credentials" > "OAuth client ID"
3. Select application type:
   - Desktop app: For command-line or desktop applications
   - Web application: For web servers
4. Download credentials JSON file
5. Save as "credentials.json" in your project directory
```

## Available Scopes

Choose the minimum scope required for your use case:

### Full Access
```
https://www.googleapis.com/auth/calendar
```
- See, edit, share, and permanently delete all calendars
- **Use when:** Full calendar management application

### Read-Only Access
```
https://www.googleapis.com/auth/calendar.readonly
```
- View all calendars and events
- **Use when:** Calendar viewing/analysis only

### Events Only
```
https://www.googleapis.com/auth/calendar.events
```
- View and edit events on all calendars
- Cannot create/delete calendars
- **Use when:** Event management without calendar admin

```
https://www.googleapis.com/auth/calendar.events.readonly
```
- View events only
- **Use when:** Event viewing/reporting

### Calendar List Management
```
https://www.googleapis.com/auth/calendar.calendarlist
```
- Add, remove calendars from user's list
- **Use when:** Managing calendar subscriptions

```
https://www.googleapis.com/auth/calendar.calendarlist.readonly
```
- View calendar list only

### Specialized Scopes
```
https://www.googleapis.com/auth/calendar.freebusy
```
- View free/busy information only
- **Use when:** Scheduling assistant

```
https://www.googleapis.com/auth/calendar.settings.readonly
```
- View user settings (timezone, etc.)

```
https://www.googleapis.com/auth/calendar.events.owned
```
- Manage only events on calendars you own
- **Use when:** Managing own calendars only

## Token Management

### Access Token
- Short-lived (typically 1 hour)
- Used for API requests
- Include in Authorization header: `Bearer YOUR_ACCESS_TOKEN`

### Refresh Token
- Long-lived (until revoked)
- Used to obtain new access tokens
- Store securely, never expose to client-side code
- Only provided on first authorization (with `access_type=offline`)

### Token Storage Pattern

```python
# Python example
import os
from google.oauth2.credentials import Credentials

TOKEN_FILE = 'token.json'

def get_credentials():
    creds = None
    
    # Load existing token
    if os.path.exists(TOKEN_FILE):
        creds = Credentials.from_authorized_user_file(TOKEN_FILE, SCOPES)
    
    # Check if valid
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            # Refresh the token
            creds.refresh(Request())
        else:
            # Need new authorization
            flow = InstalledAppFlow.from_client_secrets_file(
                'credentials.json', SCOPES
            )
            creds = flow.run_local_server(port=0)
        
        # Save for next run
        with open(TOKEN_FILE, 'w') as token:
            token.write(creds.to_json())
    
    return creds
```

### Multi-User Token Storage

For applications serving multiple users:

```python
# Store tokens per user
def get_user_credentials(user_id):
    token_file = f'tokens/{user_id}_token.json'
    
    if os.path.exists(token_file):
        return Credentials.from_authorized_user_file(token_file, SCOPES)
    
    # User needs to authorize
    return None

def save_user_credentials(user_id, creds):
    token_file = f'tokens/{user_id}_token.json'
    os.makedirs('tokens', exist_ok=True)
    
    with open(token_file, 'w') as token:
        token.write(creds.to_json())
```

## Authorization Flow Types

### Installed Application Flow (Desktop/CLI)

Used for desktop apps and command-line tools:

```python
from google_auth_oauthlib.flow import InstalledAppFlow

flow = InstalledAppFlow.from_client_secrets_file(
    'credentials.json',
    scopes=['https://www.googleapis.com/auth/calendar']
)

# This opens browser for user authorization
creds = flow.run_local_server(port=0)
```

### Web Application Flow

For web servers:

```python
from google_auth_oauthlib.flow import Flow

# Create flow instance
flow = Flow.from_client_secrets_file(
    'credentials.json',
    scopes=['https://www.googleapis.com/auth/calendar'],
    redirect_uri='https://yourapp.com/oauth2callback'
)

# Generate authorization URL
authorization_url, state = flow.authorization_url(
    access_type='offline',
    include_granted_scopes='true'
)

# After user authorizes and returns to redirect_uri:
flow.fetch_token(authorization_response=request.url)
creds = flow.credentials
```

### Service Account (Server-to-Server)

For backend services acting on behalf of many users:

```python
from google.oauth2 import service_account

credentials = service_account.Credentials.from_service_account_file(
    'service-account-key.json',
    scopes=['https://www.googleapis.com/auth/calendar']
)

# Delegate to specific user (requires domain-wide delegation)
delegated_credentials = credentials.with_subject('user@example.com')
```

## Important Parameters

### access_type
```python
authorization_url, state = flow.authorization_url(
    access_type='offline'  # Request refresh token
)
```
- `offline`: Returns refresh token (recommended)
- `online`: No refresh token, must re-authorize when access token expires

### prompt
```python
authorization_url, state = flow.authorization_url(
    prompt='consent'  # Force consent screen
)
```
- `consent`: Always show consent screen (gets new refresh token)
- `select_account`: Let user choose account
- `none`: No interaction (fails if not logged in)

### include_granted_scopes
```python
authorization_url, state = flow.authorization_url(
    include_granted_scopes='true'
)
```
- Enables incremental authorization
- Add scopes without re-requesting already granted ones

## Security Best Practices

### DO:
1. ✅ Store credentials.json securely (never commit to version control)
2. ✅ Use HTTPS for redirect URIs
3. ✅ Validate state parameter to prevent CSRF
4. ✅ Store tokens encrypted at rest
5. ✅ Use minimum required scopes
6. ✅ Implement token refresh before expiration
7. ✅ Provide clear OAuth consent screen descriptions

### DON'T:
1. ❌ Hardcode API keys or secrets in code
2. ❌ Expose credentials.json or tokens in client-side code
3. ❌ Request more scopes than needed
4. ❌ Store tokens in local storage (web apps)
5. ❌ Share credentials between users
6. ❌ Commit token.json to version control

## Testing Authentication

Quick test to verify setup:

```python
from googleapiclient.discovery import build
from google.oauth2.credentials import Credentials

# Assume creds already obtained
creds = Credentials.from_authorized_user_file('token.json', SCOPES)

# Build service
service = build('calendar', 'v3', credentials=creds)

# Test API call
calendar_list = service.calendarList().list().execute()

print(f"Found {len(calendar_list.get('items', []))} calendars")
```

## Troubleshooting

### "Access blocked: This app's request is invalid"
- OAuth consent screen not configured
- Fix: Complete OAuth consent screen setup

### "redirect_uri_mismatch"
- Redirect URI doesn't match configured URI
- Fix: Add exact redirect URI in Google Cloud Console

### "invalid_grant" when refreshing token
- Refresh token revoked or expired
- Fix: Re-authorize user to get new tokens

### "insufficient_scope"
- Requested scope not granted
- Fix: Request appropriate scope during authorization

## Multi-User Considerations

When building apps for multiple users:

1. **Separate token storage per user**
   ```
   tokens/
   ├── user_123_token.json
   ├── user_456_token.json
   └── user_789_token.json
   ```

2. **Associate calendars with users**
   ```python
   user_calendars = {
       "user_123": ["primary", "cal_id_1"],
       "user_456": ["primary", "cal_id_2", "cal_id_3"]
   }
   ```

3. **Handle token expiration gracefully**
   - Check `creds.valid` before API calls
   - Refresh automatically when possible
   - Prompt re-authorization if refresh fails

4. **User privacy**
   - Never mix user data
   - Clear tokens when user disconnects
   - Implement data deletion on user request

## Next Steps

- **For Python examples:** See `quickstart-python.md`
- **For Node.js examples:** See `quickstart-nodejs.md`
- **After authentication:** See `calendars.md` to list calendars
- **Multi-user setup:** See `multi-user.md`
