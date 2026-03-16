# Python Quickstart

## Complete Working Example

This is a fully functional example demonstrating authentication, calendar listing, and event syncing.

### Prerequisites

```bash
pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib
```

### Basic Calendar Access

```python
#!/usr/bin/env python3
"""
Google Calendar API Quickstart
Demonstrates basic authentication and calendar access.
"""

import os
import datetime
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

# If modifying scopes, delete token.json
SCOPES = ['https://www.googleapis.com/auth/calendar.readonly']

def get_credentials():
    """Authenticate and return credentials."""
    creds = None
    
    # token.json stores user's access and refresh tokens
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)
    
    # If no valid credentials, let user log in
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                'credentials.json', SCOPES
            )
            creds = flow.run_local_server(port=0)
        
        # Save credentials for next run
        with open('token.json', 'w') as token:
            token.write(creds.to_json())
    
    return creds

def main():
    """Main function demonstrating Calendar API usage."""
    try:
        # Authenticate
        creds = get_credentials()
        service = build('calendar', 'v3', credentials=creds)
        
        # Get upcoming events
        now = datetime.datetime.now(datetime.timezone.utc).isoformat()
        
        print('Getting the upcoming 10 events...')
        events_result = service.events().list(
            calendarId='primary',
            timeMin=now,
            maxResults=10,
            singleEvents=True,
            orderBy='startTime'
        ).execute()
        
        events = events_result.get('items', [])
        
        if not events:
            print('No upcoming events found.')
            return
        
        # Print events
        print(f'\nFound {len(events)} upcoming events:\n')
        for event in events:
            start = event['start'].get('dateTime', event['start'].get('date'))
            print(f"{start}: {event.get('summary', 'Untitled')}")
    
    except HttpError as error:
        print(f'An error occurred: {error}')

if __name__ == '__main__':
    main()
```

### List All Calendars

```python
def list_calendars(service):
    """List all calendars user has access to."""
    print('\nYour Calendars:')
    print('-' * 50)
    
    calendar_list = service.calendarList().list().execute()
    
    for calendar in calendar_list.get('items', []):
        primary = " (PRIMARY)" if calendar.get('primary') else ""
        access = calendar.get('accessRole', 'unknown')
        
        print(f"{calendar.get('summary')}{primary}")
        print(f"  ID: {calendar['id']}")
        print(f"  Access: {access}")
        print(f"  Timezone: {calendar.get('timeZone', 'N/A')}")
        print()
```

### Complete Sync Implementation

```python
#!/usr/bin/env python3
"""
Complete Calendar Sync Example
Demonstrates incremental sync with sync tokens.
"""

import os
import json
from datetime import datetime, timezone, timedelta
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

SYNC_STATE_FILE = 'sync_state.json'

def load_sync_token(calendar_id='primary'):
    """Load stored sync token."""
    if os.path.exists(SYNC_STATE_FILE):
        with open(SYNC_STATE_FILE, 'r') as f:
            state = json.load(f)
            return state.get('calendars', {}).get(calendar_id, {}).get('sync_token')
    return None

def save_sync_token(calendar_id, sync_token):
    """Save sync token for future use."""
    state = {}
    if os.path.exists(SYNC_STATE_FILE):
        with open(SYNC_STATE_FILE, 'r') as f:
            state = json.load(f)
    
    if 'calendars' not in state:
        state['calendars'] = {}
    
    state['calendars'][calendar_id] = {
        'sync_token': sync_token,
        'last_sync': datetime.now(timezone.utc).isoformat()
    }
    
    with open(SYNC_STATE_FILE, 'w') as f:
        json.dump(state, f, indent=2)

def full_sync(service, calendar_id='primary'):
    """Perform initial full sync."""
    print(f"Performing full sync for calendar: {calendar_id}")
    
    events = []
    page_token = None
    sync_token = None
    
    # Sync events from last year
    one_year_ago = datetime.now(timezone.utc) - timedelta(days=365)
    
    while True:
        request = service.events().list(
            calendarId=calendar_id,
            maxResults=250,
            singleEvents=True,
            timeMin=one_year_ago.isoformat(),
            pageToken=page_token
        )
        
        response = request.execute()
        events.extend(response.get('items', []))
        
        page_token = response.get('nextPageToken')
        if page_token:
            print(f"  Fetched {len(events)} events so far...")
        else:
            sync_token = response.get('nextSyncToken')
            break
    
    print(f"Full sync complete: {len(events)} events")
    return events, sync_token

def incremental_sync(service, calendar_id, sync_token):
    """Perform incremental sync using stored token."""
    print(f"Performing incremental sync for calendar: {calendar_id}")
    
    changes = []
    page_token = None
    new_sync_token = None
    
    while True:
        request = service.events().list(
            calendarId=calendar_id,
            syncToken=sync_token,
            pageToken=page_token
        )
        
        response = request.execute()
        changes.extend(response.get('items', []))
        
        page_token = response.get('nextPageToken')
        if page_token:
            print(f"  Fetched {len(changes)} changes so far...")
        else:
            new_sync_token = response.get('nextSyncToken')
            break
    
    print(f"Incremental sync complete: {len(changes)} changes")
    return changes, new_sync_token

def sync_calendar(service, calendar_id='primary'):
    """
    Unified sync function.
    
    Returns:
        tuple: (events/changes, sync_token, was_full_sync)
    """
    stored_sync_token = load_sync_token(calendar_id)
    
    try:
        if stored_sync_token:
            # Try incremental sync
            changes, new_token = incremental_sync(
                service, calendar_id, stored_sync_token
            )
            return changes, new_token, False
        else:
            # Perform full sync
            events, new_token = full_sync(service, calendar_id)
            return events, new_token, True
    
    except HttpError as error:
        if error.resp.status == 410:
            # Sync token invalidated - perform full sync
            print("⚠️  Sync token expired. Performing full sync...")
            events, new_token = full_sync(service, calendar_id)
            return events, new_token, True
        else:
            raise

def main():
    """Main sync demonstration."""
    # Get credentials (from previous example)
    creds = get_credentials()
    service = build('calendar', 'v3', credentials=creds)
    
    # Perform sync
    results, sync_token, full_sync = sync_calendar(service, 'primary')
    
    # Save new sync token
    save_sync_token('primary', sync_token)
    
    # Process results
    if full_sync:
        print(f"\nInitial sync complete. Found {len(results)} events.")
        
        # Show sample events
        print("\nSample events:")
        for event in results[:5]:
            summary = event.get('summary', 'Untitled')
            start = event['start'].get('dateTime', event['start'].get('date'))
            print(f"  {start}: {summary}")
    else:
        print(f"\nIncremental sync complete. {len(results)} changes detected.")
        
        # Categorize changes
        created = []
        updated = []
        deleted = []
        
        for item in results:
            if item.get('status') == 'cancelled':
                deleted.append(item)
            elif item.get('created') == item.get('updated'):
                created.append(item)
            else:
                updated.append(item)
        
        print(f"  Created: {len(created)}")
        print(f"  Updated: {len(updated)}")
        print(f"  Deleted: {len(deleted)}")

if __name__ == '__main__':
    main()
```

### Create Event

```python
def create_event(service, calendar_id='primary'):
    """Create a new event."""
    # Event starts tomorrow at 10 AM
    tomorrow = datetime.now(timezone.utc) + timedelta(days=1)
    start = tomorrow.replace(hour=10, minute=0, second=0, microsecond=0)
    end = start + timedelta(hours=1)
    
    event = {
        'summary': 'Team Meeting',
        'location': 'Conference Room A',
        'description': 'Weekly team sync',
        'start': {
            'dateTime': start.isoformat(),
            'timeZone': 'America/Los_Angeles',
        },
        'end': {
            'dateTime': end.isoformat(),
            'timeZone': 'America/Los_Angeles',
        },
        'attendees': [
            {'email': 'colleague@example.com'},
        ],
        'reminders': {
            'useDefault': False,
            'overrides': [
                {'method': 'email', 'minutes': 24 * 60},
                {'method': 'popup', 'minutes': 10},
            ],
        },
    }
    
    created_event = service.events().insert(
        calendarId=calendar_id,
        body=event,
        sendUpdates='all'  # Send invites to attendees
    ).execute()
    
    print(f"Event created: {created_event.get('htmlLink')}")
    return created_event
```

### Update Event

```python
def update_event(service, calendar_id, event_id):
    """Update an existing event."""
    # First, get the event
    event = service.events().get(
        calendarId=calendar_id,
        eventId=event_id
    ).execute()
    
    # Modify event
    event['summary'] = 'Updated Meeting Title'
    event['description'] = 'Updated description'
    
    # Update on server
    updated_event = service.events().update(
        calendarId=calendar_id,
        eventId=event_id,
        body=event,
        sendUpdates='all'
    ).execute()
    
    print(f"Event updated: {updated_event.get('summary')}")
    return updated_event
```

### Delete Event

```python
def delete_event(service, calendar_id, event_id):
    """Delete an event."""
    service.events().delete(
        calendarId=calendar_id,
        eventId=event_id,
        sendUpdates='all'  # Notify attendees
    ).execute()
    
    print(f"Event deleted: {event_id}")
```

### Recurring Event

```python
def create_recurring_event(service, calendar_id='primary'):
    """Create a recurring event."""
    # Starts next Monday at 2 PM
    today = datetime.now(timezone.utc)
    days_ahead = 0 - today.weekday()  # Monday = 0
    if days_ahead <= 0:
        days_ahead += 7
    
    next_monday = today + timedelta(days=days_ahead)
    start = next_monday.replace(hour=14, minute=0, second=0, microsecond=0)
    end = start + timedelta(hours=1)
    
    event = {
        'summary': 'Weekly Team Standup',
        'location': 'Virtual - Zoom',
        'description': 'Weekly sync meeting',
        'start': {
            'dateTime': start.isoformat(),
            'timeZone': 'America/Los_Angeles',
        },
        'end': {
            'dateTime': end.isoformat(),
            'timeZone': 'America/Los_Angeles',
        },
        'recurrence': [
            'RRULE:FREQ=WEEKLY;BYDAY=MO;COUNT=10'  # Every Monday, 10 times
        ],
        'attendees': [
            {'email': 'team@example.com'},
        ],
    }
    
    created_event = service.events().insert(
        calendarId=calendar_id,
        body=event
    ).execute()
    
    print(f"Recurring event created: {created_event.get('summary')}")
    return created_event
```

### Search Events

```python
def search_events(service, query, calendar_id='primary'):
    """Search for events containing query text."""
    # Note: q parameter not compatible with sync tokens
    events_result = service.events().list(
        calendarId=calendar_id,
        q=query,
        maxResults=10,
        singleEvents=True,
        orderBy='startTime'
    ).execute()
    
    events = events_result.get('items', [])
    
    print(f"\nFound {len(events)} events matching '{query}':")
    for event in events:
        start = event['start'].get('dateTime', event['start'].get('date'))
        print(f"  {start}: {event.get('summary', 'Untitled')}")
    
    return events
```

### Free/Busy Query

```python
def check_availability(service, emails, start_time, end_time):
    """
    Check free/busy status for multiple calendars.
    
    Args:
        emails: List of calendar IDs or email addresses
        start_time: datetime object
        end_time: datetime object
    """
    freebusy_query = {
        'timeMin': start_time.isoformat(),
        'timeMax': end_time.isoformat(),
        'items': [{'id': email} for email in emails]
    }
    
    result = service.freebusy().query(body=freebusy_query).execute()
    
    print("\nFree/Busy Status:")
    print("-" * 50)
    
    for calendar_id, calendar_data in result['calendars'].items():
        busy_times = calendar_data.get('busy', [])
        
        print(f"\n{calendar_id}:")
        if busy_times:
            print("  Busy during:")
            for period in busy_times:
                print(f"    {period['start']} - {period['end']}")
        else:
            print("  Free for entire period")
    
    return result
```

### Error Handling

```python
def safe_api_call(func):
    """Decorator for handling common API errors."""
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except HttpError as error:
            if error.resp.status == 404:
                print("Error: Resource not found")
            elif error.resp.status == 403:
                print("Error: Permission denied")
            elif error.resp.status == 410:
                print("Error: Sync token expired")
            elif error.resp.status == 429:
                print("Error: Rate limit exceeded")
            else:
                print(f"HTTP Error: {error}")
            return None
        except Exception as e:
            print(f"Unexpected error: {e}")
            return None
    
    return wrapper

@safe_api_call
def safe_get_event(service, calendar_id, event_id):
    """Safely get an event with error handling."""
    return service.events().get(
        calendarId=calendar_id,
        eventId=event_id
    ).execute()
```

### Complete Application

```python
#!/usr/bin/env python3
"""
Complete Calendar Application
Demonstrates all major features.
"""

import os
import sys
import json
from datetime import datetime, timezone, timedelta
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

SCOPES = ['https://www.googleapis.com/auth/calendar']

class CalendarApp:
    """Main calendar application class."""
    
    def __init__(self):
        self.creds = None
        self.service = None
        self.sync_state_file = 'sync_state.json'
    
    def authenticate(self):
        """Handle authentication."""
        if os.path.exists('token.json'):
            self.creds = Credentials.from_authorized_user_file(
                'token.json', SCOPES
            )
        
        if not self.creds or not self.creds.valid:
            if self.creds and self.creds.expired and self.creds.refresh_token:
                self.creds.refresh(Request())
            else:
                flow = InstalledAppFlow.from_client_secrets_file(
                    'credentials.json', SCOPES
                )
                self.creds = flow.run_local_server(port=0)
            
            with open('token.json', 'w') as token:
                token.write(self.creds.to_json())
        
        self.service = build('calendar', 'v3', credentials=self.creds)
    
    def run(self):
        """Main application loop."""
        self.authenticate()
        
        while True:
            print("\n" + "="*50)
            print("Calendar Application")
            print("="*50)
            print("1. List calendars")
            print("2. Sync events")
            print("3. Create event")
            print("4. Search events")
            print("5. Exit")
            
            choice = input("\nChoice: ").strip()
            
            if choice == '1':
                self.list_calendars()
            elif choice == '2':
                self.sync_events()
            elif choice == '3':
                self.create_event_interactive()
            elif choice == '4':
                self.search_events_interactive()
            elif choice == '5':
                break
    
    def list_calendars(self):
        """List all calendars."""
        calendar_list = self.service.calendarList().list().execute()
        
        print("\nYour Calendars:")
        for idx, cal in enumerate(calendar_list.get('items', []), 1):
            primary = " [PRIMARY]" if cal.get('primary') else ""
            print(f"{idx}. {cal.get('summary')}{primary}")
            print(f"   ID: {cal['id']}")
    
    def sync_events(self):
        """Sync calendar events."""
        results, sync_token, full_sync = sync_calendar(
            self.service, 'primary'
        )
        save_sync_token('primary', sync_token)
        
        if full_sync:
            print(f"\nSynced {len(results)} events")
        else:
            print(f"\n{len(results)} changes detected")
    
    def create_event_interactive(self):
        """Interactive event creation."""
        print("\nCreate New Event")
        summary = input("Title: ")
        description = input("Description: ")
        
        # Simple event tomorrow
        tomorrow = datetime.now(timezone.utc) + timedelta(days=1)
        start = tomorrow.replace(hour=10, minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)
        
        event = {
            'summary': summary,
            'description': description,
            'start': {'dateTime': start.isoformat()},
            'end': {'dateTime': end.isoformat()},
        }
        
        created = self.service.events().insert(
            calendarId='primary',
            body=event
        ).execute()
        
        print(f"\n✓ Event created: {created.get('htmlLink')}")
    
    def search_events_interactive(self):
        """Interactive event search."""
        query = input("\nSearch query: ")
        search_events(self.service, query)

if __name__ == '__main__':
    app = CalendarApp()
    app.run()
```

## Running the Examples

1. **Setup:**
   ```bash
   # Install dependencies
   pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib
   
   # Download credentials.json from Google Cloud Console
   # Place in same directory as script
   ```

2. **First Run:**
   ```bash
   python quickstart.py
   ```
   - Browser opens for authentication
   - Authorize the application
   - `token.json` created for future use

3. **Subsequent Runs:**
   - Uses stored `token.json`
   - Automatically refreshes if expired

## Next Steps

- **Multi-calendar:** See `multi-calendar.md`
- **Multi-user:** See `multi-user.md`
- **Advanced sync:** See `sync-strategy.md`
- **Full API reference:** See `api-reference.md`
