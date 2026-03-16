# Multi-User Management

## Overview

Managing multiple authenticated Google accounts simultaneously requires:
- Separate OAuth tokens per user
- Isolated sync state per user per calendar
- User-specific API service instances
- Proper data isolation and security

## Architecture

```
Application
├── User 1 (OAuth Token 1)
│   ├── Primary Calendar (sync token)
│   ├── Work Calendar (sync token)
│   └── Family Calendar (sync token)
│
├── User 2 (OAuth Token 2)
│   ├── Primary Calendar (sync token)
│   └── Team Calendar (sync token)
│
└── User 3 (OAuth Token 3)
    └── Primary Calendar (sync token)
```

## User Authentication Manager

### Python Implementation

```python
import os
import json
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/calendar']

class MultiUserAuthManager:
    """Manages OAuth tokens for multiple users."""
    
    def __init__(self, credentials_file, tokens_dir='user_tokens'):
        self.credentials_file = credentials_file
        self.tokens_dir = tokens_dir
        os.makedirs(tokens_dir, exist_ok=True)
    
    def get_user_token_file(self, user_id):
        """Get path to user's token file."""
        return os.path.join(self.tokens_dir, f'{user_id}_token.json')
    
    def load_user_credentials(self, user_id):
        """Load credentials for specific user."""
        token_file = self.get_user_token_file(user_id)
        
        if not os.path.exists(token_file):
            return None
        
        return Credentials.from_authorized_user_file(token_file, SCOPES)
    
    def save_user_credentials(self, user_id, creds):
        """Save credentials for specific user."""
        token_file = self.get_user_token_file(user_id)
        
        with open(token_file, 'w') as token:
            token.write(creds.to_json())
    
    def authorize_user(self, user_id):
        """
        Authorize a new user or refresh expired credentials.
        Returns authorized credentials.
        """
        creds = self.load_user_credentials(user_id)
        
        # Check if we need to refresh or authorize
        if not creds or not creds.valid:
            if creds and creds.expired and creds.refresh_token:
                # Refresh expired token
                creds.refresh(Request())
            else:
                # Need new authorization
                flow = InstalledAppFlow.from_client_secrets_file(
                    self.credentials_file, SCOPES
                )
                creds = flow.run_local_server(port=0)
            
            # Save updated credentials
            self.save_user_credentials(user_id, creds)
        
        return creds
    
    def get_service_for_user(self, user_id):
        """
        Get authorized Calendar API service for specific user.
        
        Returns:
            Authorized Calendar API service instance
        """
        creds = self.authorize_user(user_id)
        return build('calendar', 'v3', credentials=creds)
    
    def list_authorized_users(self):
        """Get list of all authorized user IDs."""
        token_files = os.listdir(self.tokens_dir)
        return [
            f.replace('_token.json', '') 
            for f in token_files 
            if f.endswith('_token.json')
        ]
    
    def revoke_user(self, user_id):
        """Remove user's credentials (logout)."""
        token_file = self.get_user_token_file(user_id)
        
        if os.path.exists(token_file):
            os.remove(token_file)
            print(f"Revoked credentials for user: {user_id}")
```

## Multi-User Sync State Manager

### Complete State Management

```python
import json
from datetime import datetime, timezone

class MultiUserSyncManager:
    """Manages sync state for multiple users and their calendars."""
    
    def __init__(self, state_file='multi_user_sync_state.json'):
        self.state_file = state_file
        self.state = self._load_state()
    
    def _load_state(self):
        """Load sync state from file."""
        try:
            with open(self.state_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            return {'users': {}}
    
    def _save_state(self):
        """Save sync state to file."""
        with open(self.state_file, 'w') as f:
            json.dump(self.state, f, indent=2)
    
    def get_sync_token(self, user_id, calendar_id):
        """Get sync token for user's calendar."""
        return (self.state['users']
                .get(user_id, {})
                .get('calendars', {})
                .get(calendar_id, {})
                .get('sync_token'))
    
    def save_sync_token(self, user_id, calendar_id, sync_token):
        """Save sync token for user's calendar."""
        # Initialize user if needed
        if user_id not in self.state['users']:
            self.state['users'][user_id] = {'calendars': {}}
        
        # Initialize calendar if needed
        if calendar_id not in self.state['users'][user_id]['calendars']:
            self.state['users'][user_id]['calendars'][calendar_id] = {}
        
        # Save sync token
        self.state['users'][user_id]['calendars'][calendar_id].update({
            'sync_token': sync_token,
            'last_sync': datetime.now(timezone.utc).isoformat()
        })
        
        self._save_state()
    
    def clear_user_calendar(self, user_id, calendar_id):
        """Clear sync state for user's calendar."""
        if (user_id in self.state['users'] and 
            calendar_id in self.state['users'][user_id].get('calendars', {})):
            del self.state['users'][user_id]['calendars'][calendar_id]
            self._save_state()
    
    def clear_user(self, user_id):
        """Clear all sync state for user."""
        if user_id in self.state['users']:
            del self.state['users'][user_id]
            self._save_state()
    
    def get_user_calendar_ids(self, user_id):
        """Get list of calendar IDs being synced for user."""
        return list(
            self.state['users']
            .get(user_id, {})
            .get('calendars', {})
            .keys()
        )
    
    def get_all_users(self):
        """Get list of all user IDs."""
        return list(self.state['users'].keys())
```

### State File Structure

```json
{
  "users": {
    "user_alice": {
      "calendars": {
        "primary": {
          "sync_token": "CPDAlvWDx70CEPDAlvWDx70CGAU=",
          "last_sync": "2025-02-17T10:30:00Z"
        },
        "work@company.com": {
          "sync_token": "XYZ123abc456def==",
          "last_sync": "2025-02-17T10:28:00Z"
        }
      }
    },
    "user_bob": {
      "calendars": {
        "primary": {
          "sync_token": "ABC789xyz123==",
          "last_sync": "2025-02-17T10:25:00Z"
        }
      }
    }
  }
}
```

## Multi-User Event Storage

### Isolated Event Storage

```python
class MultiUserEventStorage:
    """Storage for events across multiple users and calendars."""
    
    def __init__(self):
        # Structure: users -> calendars -> events
        self.data = {}
    
    def add_event(self, user_id, calendar_id, event):
        """Add event for specific user's calendar."""
        # Initialize user
        if user_id not in self.data:
            self.data[user_id] = {}
        
        # Initialize calendar
        if calendar_id not in self.data[user_id]:
            self.data[user_id][calendar_id] = {}
        
        # Add event
        event_id = event['id']
        self.data[user_id][calendar_id][event_id] = event
    
    def remove_event(self, user_id, calendar_id, event_id):
        """Remove event from user's calendar."""
        if (user_id in self.data and 
            calendar_id in self.data[user_id]):
            self.data[user_id][calendar_id].pop(event_id, None)
    
    def get_user_calendar_events(self, user_id, calendar_id):
        """Get all events for user's specific calendar."""
        return (self.data
                .get(user_id, {})
                .get(calendar_id, {}))
    
    def get_all_user_events(self, user_id):
        """Get all events across all calendars for user."""
        all_events = []
        
        for calendar_id, events in self.data.get(user_id, {}).items():
            for event in events.values():
                event_copy = event.copy()
                event_copy['_calendar_id'] = calendar_id
                all_events.append(event_copy)
        
        return all_events
    
    def clear_user_calendar(self, user_id, calendar_id):
        """Clear all events for user's calendar."""
        if user_id in self.data and calendar_id in self.data[user_id]:
            self.data[user_id][calendar_id] = {}
    
    def clear_user(self, user_id):
        """Clear all events for user."""
        if user_id in self.data:
            del self.data[user_id]
```

## Complete Multi-User Sync

### Sync All Users and Calendars

```python
def sync_all_users(auth_manager, sync_manager, event_storage):
    """
    Sync all calendars for all authorized users.
    
    Args:
        auth_manager: MultiUserAuthManager instance
        sync_manager: MultiUserSyncManager instance
        event_storage: MultiUserEventStorage instance
    
    Returns:
        dict: Results per user and calendar
    """
    all_results = {}
    
    # Get all authorized users
    user_ids = auth_manager.list_authorized_users()
    
    for user_id in user_ids:
        print(f"\n{'='*50}")
        print(f"Syncing user: {user_id}")
        print(f"{'='*50}")
        
        try:
            # Get service for this user
            service = auth_manager.get_service_for_user(user_id)
            
            # Get user's calendars
            calendar_list = service.calendarList().list().execute()
            
            user_results = {}
            
            for calendar_entry in calendar_list.get('items', []):
                calendar_id = calendar_entry['id']
                calendar_name = calendar_entry.get('summary', calendar_id)
                
                # Skip unselected calendars
                if not calendar_entry.get('selected', True):
                    continue
                
                print(f"\n  Calendar: {calendar_name}")
                
                try:
                    # Get sync token
                    sync_token = sync_manager.get_sync_token(
                        user_id, calendar_id
                    )
                    
                    # Perform sync
                    result = sync_calendar_events(
                        service, calendar_id, sync_token
                    )
                    
                    # Process events
                    events_key = 'events' if result['full_sync'] else 'changes'
                    for event in result[events_key]:
                        if event.get('status') == 'cancelled':
                            event_storage.remove_event(
                                user_id, calendar_id, event['id']
                            )
                        else:
                            event_storage.add_event(
                                user_id, calendar_id, event
                            )
                    
                    # Save sync token
                    sync_manager.save_sync_token(
                        user_id, calendar_id, result['sync_token']
                    )
                    
                    # Track results
                    event_count = len(
                        event_storage.get_user_calendar_events(
                            user_id, calendar_id
                        )
                    )
                    
                    user_results[calendar_id] = {
                        'success': True,
                        'events_count': event_count,
                        'full_sync': result.get('full_sync', False)
                    }
                    
                    print(f"    ✓ Synced: {event_count} events")
                    
                except HttpError as error:
                    if error.resp.status == 410:
                        # Token invalidated - clear and retry
                        print(f"    ⚠ Sync token invalidated, performing full sync")
                        sync_manager.clear_user_calendar(user_id, calendar_id)
                        event_storage.clear_user_calendar(user_id, calendar_id)
                        
                        # Retry
                        result = sync_calendar_events(service, calendar_id, None)
                        for event in result['events']:
                            if event.get('status') != 'cancelled':
                                event_storage.add_event(
                                    user_id, calendar_id, event
                                )
                        
                        sync_manager.save_sync_token(
                            user_id, calendar_id, result['sync_token']
                        )
                        
                        user_results[calendar_id] = {
                            'success': True,
                            'recovered_410': True
                        }
                    else:
                        user_results[calendar_id] = {
                            'success': False,
                            'error': str(error)
                        }
            
            all_results[user_id] = user_results
            
        except Exception as e:
            all_results[user_id] = {
                'success': False,
                'error': str(e)
            }
    
    return all_results
```

## User Management Operations

### Add New User

```python
def add_user(auth_manager, user_id):
    """
    Add and authorize a new user.
    
    Args:
        user_id: Unique identifier for user (email, username, etc.)
    
    Returns:
        bool: True if successfully authorized
    """
    print(f"Authorizing new user: {user_id}")
    print("A browser window will open for authentication...")
    
    try:
        creds = auth_manager.authorize_user(user_id)
        
        # Verify by getting user info
        service = build('calendar', 'v3', credentials=creds)
        calendar_list = service.calendarList().list(maxResults=1).execute()
        
        print(f"✓ User {user_id} authorized successfully")
        return True
        
    except Exception as e:
        print(f"✗ Failed to authorize user: {e}")
        return False
```

### Remove User

```python
def remove_user(auth_manager, sync_manager, event_storage, user_id):
    """
    Remove user and clean up all data.
    
    Args:
        user_id: User to remove
    """
    print(f"Removing user: {user_id}")
    
    # Revoke OAuth credentials
    auth_manager.revoke_user(user_id)
    
    # Clear sync state
    sync_manager.clear_user(user_id)
    
    # Clear event storage
    event_storage.clear_user(user_id)
    
    print(f"✓ User {user_id} removed completely")
```

### List Users

```python
def list_all_users(auth_manager, event_storage):
    """List all authorized users with their event counts."""
    users = auth_manager.list_authorized_users()
    
    print(f"\nAuthorized Users ({len(users)}):")
    print("-" * 50)
    
    for user_id in users:
        events = event_storage.get_all_user_events(user_id)
        print(f"  {user_id}: {len(events)} total events")
    
    return users
```

## Web Application Pattern

For web apps serving multiple users:

### Flask Example

```python
from flask import Flask, session, redirect, url_for
from google_auth_oauthlib.flow import Flow

app = Flask(__name__)
app.secret_key = 'your-secret-key'  # Use secure key in production

# Initialize managers
auth_manager = MultiUserAuthManager('credentials.json')
sync_manager = MultiUserSyncManager()
event_storage = MultiUserEventStorage()

@app.route('/login')
def login():
    """Initiate OAuth flow."""
    flow = Flow.from_client_secrets_file(
        'credentials.json',
        scopes=SCOPES,
        redirect_uri=url_for('oauth2callback', _external=True)
    )
    
    authorization_url, state = flow.authorization_url(
        access_type='offline',
        include_granted_scopes='true'
    )
    
    session['state'] = state
    return redirect(authorization_url)

@app.route('/oauth2callback')
def oauth2callback():
    """Handle OAuth callback."""
    state = session['state']
    
    flow = Flow.from_client_secrets_file(
        'credentials.json',
        scopes=SCOPES,
        state=state,
        redirect_uri=url_for('oauth2callback', _external=True)
    )
    
    flow.fetch_token(authorization_response=request.url)
    creds = flow.credentials
    
    # Get user email from token
    service = build('calendar', 'v3', credentials=creds)
    calendar_list = service.calendarList().list(maxResults=1).execute()
    
    # Use primary calendar ID as user identifier
    user_id = calendar_list['items'][0]['id']
    
    # Save credentials
    auth_manager.save_user_credentials(user_id, creds)
    
    # Store in session
    session['user_id'] = user_id
    
    return redirect(url_for('dashboard'))

@app.route('/dashboard')
def dashboard():
    """User dashboard showing their events."""
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    user_id = session['user_id']
    
    # Sync user's calendars
    service = auth_manager.get_service_for_user(user_id)
    sync_user_calendars(service, user_id, sync_manager, event_storage)
    
    # Get user's events
    events = event_storage.get_all_user_events(user_id)
    
    return render_template('dashboard.html', events=events)

@app.route('/logout')
def logout():
    """Log out user."""
    if 'user_id' in session:
        user_id = session.pop('user_id')
        # Optionally clear data
        # remove_user(auth_manager, sync_manager, event_storage, user_id)
    
    return redirect(url_for('login'))
```

## Database Storage (Production)

### SQLAlchemy Models

```python
from sqlalchemy import Column, String, DateTime, Text, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(String(255), primary_key=True)
    email = Column(String(255))
    created_at = Column(DateTime)
    
    # Relationships
    calendars = relationship('Calendar', back_populates='user')

class Calendar(Base):
    __tablename__ = 'calendars'
    
    id = Column(String(255), primary_key=True)
    user_id = Column(String(255), ForeignKey('users.id'))
    name = Column(String(255))
    sync_token = Column(Text)
    last_sync = Column(DateTime)
    
    # Relationships
    user = relationship('User', back_populates='calendars')
    events = relationship('Event', back_populates='calendar')

class Event(Base):
    __tablename__ = 'events'
    
    id = Column(String(255), primary_key=True)
    calendar_id = Column(String(255), ForeignKey('calendars.id'))
    summary = Column(String(500))
    description = Column(Text)
    start_time = Column(DateTime)
    end_time = Column(DateTime)
    data = Column(Text)  # JSON blob of full event
    
    # Relationships
    calendar = relationship('Calendar', back_populates='events')
```

## Security Best Practices

### DO:
1. ✅ Store tokens encrypted at rest
2. ✅ Isolate user data completely
3. ✅ Validate user_id on every request (web apps)
4. ✅ Use HTTPS for all OAuth redirects
5. ✅ Implement proper session management
6. ✅ Provide user data export/deletion
7. ✅ Log access for audit trail

### DON'T:
1. ❌ Mix user data in storage
2. ❌ Expose user tokens in client-side code
3. ❌ Store tokens in cookies (web apps)
4. ❌ Skip token refresh
5. ❌ Share credentials between users
6. ❌ Trust client-provided user_id without validation

## Next Steps

- **Multi-calendar per user:** See `multi-calendar.md`
- **Sync implementation:** See `sync-strategy.md`
- **Authentication details:** See `authentication.md`
- **Error handling:** See `error-handling.md`
