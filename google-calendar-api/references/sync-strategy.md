# Synchronization Strategy

## Overview

**This is the most important file in the skill.** Proper synchronization is critical for building efficient Calendar applications. Always use incremental sync with sync tokens - never poll by repeatedly fetching all events.

## Core Concept: Sync Tokens

A **sync token** is an opaque string that represents a point-in-time snapshot of calendar data:

```json
{
  "nextSyncToken": "CPDAlvWDx70CEPDAlvWDx70CGAU="
}
```

### How Sync Tokens Work

1. **Initial sync** returns all data + sync token
2. **Incremental sync** uses token to get only changes since last sync
3. **New token** provided after each sync for next iteration

### Benefits

- ✅ Reduces bandwidth (only changed data transferred)
- ✅ Reduces API calls (quota efficient)
- ✅ Faster response times
- ✅ Captures all changes (including deletions)

## Two-Stage Sync Process

### Stage 1: Initial Full Sync

Performed once at the beginning or after sync token invalidation.

#### Python Example

```python
from googleapiclient.discovery import build
from datetime import datetime, timezone, timedelta

def initial_full_sync(service, calendar_id='primary'):
    """
    Perform full sync of calendar events.
    Returns all events and a sync token for future incremental syncs.
    """
    events = []
    page_token = None
    sync_token = None
    
    # Optional: Limit sync to recent events
    # Only sync events from last year
    one_year_ago = datetime.now(timezone.utc) - timedelta(days=365)
    time_min = one_year_ago.isoformat()
    
    print("Performing full sync...")
    
    while True:
        request = service.events().list(
            calendarId=calendar_id,
            maxResults=250,  # Max per page
            singleEvents=True,  # Expand recurring events
            orderBy='startTime',
            timeMin=time_min,  # Optional filter
            pageToken=page_token
        )
        
        response = request.execute()
        
        # Collect events from this page
        events.extend(response.get('items', []))
        
        # Check for more pages
        page_token = response.get('nextPageToken')
        
        if page_token:
            print(f"  Fetched {len(events)} events so far...")
            continue  # More pages to fetch
        else:
            # Last page - contains sync token
            sync_token = response.get('nextSyncToken')
            break
    
    print(f"Full sync complete: {len(events)} events")
    print(f"Sync token: {sync_token[:20]}...")
    
    return {
        'events': events,
        'sync_token': sync_token
    }
```

#### Node.js Example

```javascript
async function initialFullSync(calendar, calendarId = 'primary') {
  const events = [];
  let pageToken = null;
  let syncToken = null;
  
  // Optional: Limit to last year
  const oneYearAgo = new Date();
  oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);
  
  console.log('Performing full sync...');
  
  do {
    const response = await calendar.events.list({
      calendarId: calendarId,
      maxResults: 250,
      singleEvents: true,
      orderBy: 'startTime',
      timeMin: oneYearAgo.toISOString(),
      pageToken: pageToken
    });
    
    events.push(...(response.data.items || []));
    
    pageToken = response.data.nextPageToken;
    
    if (pageToken) {
      console.log(`  Fetched ${events.length} events so far...`);
    } else {
      syncToken = response.data.nextSyncToken;
    }
  } while (pageToken);
  
  console.log(`Full sync complete: ${events.length} events`);
  
  return {
    events: events,
    syncToken: syncToken
  };
}
```

### Stage 2: Incremental Sync

Performed repeatedly to get only changes since last sync.

#### Python Example

```python
def incremental_sync(service, calendar_id, sync_token):
    """
    Perform incremental sync using stored sync token.
    Returns changed/deleted events and new sync token.
    """
    changes = []
    page_token = None
    new_sync_token = None
    
    print("Performing incremental sync...")
    
    while True:
        request = service.events().list(
            calendarId=calendar_id,
            syncToken=sync_token,
            pageToken=page_token
        )
        
        response = request.execute()
        
        # Collect changes from this page
        changes.extend(response.get('items', []))
        
        # Check for pagination
        page_token = response.get('nextPageToken')
        
        if page_token:
            print(f"  Fetched {len(changes)} changes so far...")
            continue  # More pages
        else:
            # Last page - contains new sync token
            new_sync_token = response.get('nextSyncToken')
            break
    
    print(f"Incremental sync complete: {len(changes)} changes")
    
    return {
        'changes': changes,
        'sync_token': new_sync_token
    }
```

#### Node.js Example

```javascript
async function incrementalSync(calendar, calendarId, syncToken) {
  const changes = [];
  let pageToken = null;
  let newSyncToken = null;
  
  console.log('Performing incremental sync...');
  
  do {
    const response = await calendar.events.list({
      calendarId: calendarId,
      syncToken: syncToken,
      pageToken: pageToken
    });
    
    changes.push(...(response.data.items || []));
    
    pageToken = response.data.nextPageToken;
    
    if (pageToken) {
      console.log(`  Fetched ${changes.length} changes so far...`);
    } else {
      newSyncToken = response.data.nextSyncToken;
    }
  } while (pageToken);
  
  console.log(`Incremental sync complete: ${changes.length} changes`);
  
  return {
    changes: changes,
    syncToken: newSyncToken
  };
}
```

## Complete Sync Function

Handles both full and incremental sync automatically:

### Python

```python
from googleapiclient.errors import HttpError

def sync_calendar_events(service, calendar_id, stored_sync_token=None):
    """
    Unified sync function that handles both full and incremental sync.
    
    Args:
        service: Authorized Calendar API service
        calendar_id: Calendar ID to sync
        stored_sync_token: Previous sync token (None for full sync)
    
    Returns:
        dict: {
            'events': list of events/changes,
            'sync_token': new sync token to store,
            'full_sync': boolean indicating if full sync was performed
        }
    """
    try:
        if stored_sync_token:
            # Try incremental sync
            result = incremental_sync(service, calendar_id, stored_sync_token)
            result['full_sync'] = False
            return result
        else:
            # Perform full sync
            result = initial_full_sync(service, calendar_id)
            result['full_sync'] = True
            return result
            
    except HttpError as error:
        if error.resp.status == 410:
            # Sync token invalidated - force full sync
            print("Sync token expired. Performing full sync...")
            result = initial_full_sync(service, calendar_id)
            result['full_sync'] = True
            result['token_invalidated'] = True
            return result
        else:
            raise  # Re-raise other errors
```

### Node.js

```javascript
async function syncCalendarEvents(calendar, calendarId, storedSyncToken = null) {
  try {
    if (storedSyncToken) {
      // Try incremental sync
      const result = await incrementalSync(calendar, calendarId, storedSyncToken);
      result.fullSync = false;
      return result;
    } else {
      // Perform full sync
      const result = await initialFullSync(calendar, calendarId);
      result.fullSync = true;
      return result;
    }
  } catch (error) {
    if (error.code === 410) {
      // Sync token invalidated - force full sync
      console.log('Sync token expired. Performing full sync...');
      const result = await initialFullSync(calendar, calendarId);
      result.fullSync = true;
      result.tokenInvalidated = true;
      return result;
    } else {
      throw error;
    }
  }
}
```

## Handling Sync Results

### Processing Events and Changes

```python
def process_sync_results(sync_result, local_storage):
    """
    Process sync results and update local storage.
    
    Args:
        sync_result: Result from sync_calendar_events()
        local_storage: Your local event storage (dict, database, etc.)
    """
    events_key = 'events' if sync_result['full_sync'] else 'changes'
    items = sync_result[events_key]
    
    for item in items:
        event_id = item['id']
        
        # Check if event was deleted or cancelled
        if item.get('status') == 'cancelled':
            # Remove from local storage
            if event_id in local_storage:
                del local_storage[event_id]
                print(f"Deleted event: {event_id}")
        else:
            # Add or update in local storage
            local_storage[event_id] = item
            action = "Updated" if event_id in local_storage else "Added"
            print(f"{action} event: {item.get('summary', 'Untitled')}")
    
    # Store new sync token for next sync
    return sync_result['sync_token']
```

### Detecting Event Changes

```python
def categorize_changes(changes):
    """
    Categorize incremental sync changes.
    
    Returns:
        dict: {
            'created': [...],
            'updated': [...],
            'deleted': [...]
        }
    """
    created = []
    updated = []
    deleted = []
    
    for event in changes:
        if event.get('status') == 'cancelled':
            deleted.append(event)
        elif event.get('created') == event.get('updated'):
            created.append(event)
        else:
            updated.append(event)
    
    return {
        'created': created,
        'updated': updated,
        'deleted': deleted
    }
```

## Storage Patterns

### Simple File Storage (Single Calendar)

```python
import json
import os

SYNC_STATE_FILE = 'sync_state.json'

def load_sync_token():
    """Load sync token from file."""
    if os.path.exists(SYNC_STATE_FILE):
        with open(SYNC_STATE_FILE, 'r') as f:
            state = json.load(f)
            return state.get('sync_token')
    return None

def save_sync_token(sync_token):
    """Save sync token to file."""
    state = {
        'sync_token': sync_token,
        'last_sync': datetime.now(timezone.utc).isoformat()
    }
    with open(SYNC_STATE_FILE, 'w') as f:
        json.dump(state, f, indent=2)
```

### Multi-Calendar Storage

```python
def load_sync_token_for_calendar(calendar_id):
    """Load sync token for specific calendar."""
    if os.path.exists(SYNC_STATE_FILE):
        with open(SYNC_STATE_FILE, 'r') as f:
            state = json.load(f)
            calendars = state.get('calendars', {})
            return calendars.get(calendar_id, {}).get('sync_token')
    return None

def save_sync_token_for_calendar(calendar_id, sync_token):
    """Save sync token for specific calendar."""
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
```

### Database Storage (Recommended for Production)

```python
# Using SQLite as example

import sqlite3

def init_sync_db():
    """Initialize sync state database."""
    conn = sqlite3.connect('calendar_sync.db')
    cursor = conn.cursor()
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS sync_tokens (
            user_id TEXT NOT NULL,
            calendar_id TEXT NOT NULL,
            sync_token TEXT NOT NULL,
            last_sync TIMESTAMP NOT NULL,
            PRIMARY KEY (user_id, calendar_id)
        )
    ''')
    
    conn.commit()
    conn.close()

def get_sync_token(user_id, calendar_id):
    """Retrieve sync token for user's calendar."""
    conn = sqlite3.connect('calendar_sync.db')
    cursor = conn.cursor()
    
    cursor.execute('''
        SELECT sync_token FROM sync_tokens
        WHERE user_id = ? AND calendar_id = ?
    ''', (user_id, calendar_id))
    
    result = cursor.fetchone()
    conn.close()
    
    return result[0] if result else None

def save_sync_token_db(user_id, calendar_id, sync_token):
    """Save sync token for user's calendar."""
    conn = sqlite3.connect('calendar_sync.db')
    cursor = conn.cursor()
    
    cursor.execute('''
        INSERT OR REPLACE INTO sync_tokens
        (user_id, calendar_id, sync_token, last_sync)
        VALUES (?, ?, ?, CURRENT_TIMESTAMP)
    ''', (user_id, calendar_id, sync_token))
    
    conn.commit()
    conn.close()
```

## Important Sync Constraints

### Filters Compatible with Sync Tokens

✅ **Allowed in full sync:**
- `timeMin` - Lower bound for event start time
- `timeMax` - Upper bound for event start time
- `maxResults` - Pagination size
- `singleEvents` - Expand recurring events
- `orderBy` - Only with `singleEvents=true`

❌ **NOT allowed with sync tokens:**
- `q` - Free text search
- `iCalUID` - Filter by iCalendar UID
- `orderBy` - In incremental sync
- `privateExtendedProperty`
- `sharedExtendedProperty`
- `updatedMin`

### Critical Rules

1. **Same filters required** - Incremental sync MUST use same filters as initial full sync
2. **PageToken on last page only** - `nextSyncToken` only appears on final page
3. **No result = empty list** - If no changes, still get new sync token with empty items
4. **Includes deletions** - Deleted events appear with `status: 'cancelled'`

## 410 Error Handling

HTTP 410 "Gone" indicates sync token is invalid. Common causes:

- Token expired (typically after several weeks)
- ACL changes affecting calendar access
- Calendar deleted
- Server-side invalidation

### Proper 410 Handling

```python
def sync_with_recovery(service, calendar_id, sync_token, event_storage):
    """
    Sync with automatic 410 recovery.
    """
    try:
        result = sync_calendar_events(service, calendar_id, sync_token)
        
        if result.get('token_invalidated'):
            # Clear local storage - full sync performed
            event_storage.clear()
            print("⚠️  Sync token invalidated - full resync completed")
        
        # Process results
        new_token = process_sync_results(result, event_storage)
        return new_token
        
    except HttpError as error:
        if error.resp.status == 410:
            # Clear everything and start fresh
            event_storage.clear()
            result = initial_full_sync(service, calendar_id)
            new_token = process_sync_results(result, event_storage)
            print("⚠️  Recovered from 410 error with full sync")
            return new_token
        else:
            raise
```

## Pagination During Sync

When large changes occur, incremental sync may paginate:

```python
def incremental_sync_with_large_changes(service, calendar_id, sync_token):
    """
    Handle pagination in incremental sync.
    Server may return pageToken instead of syncToken if many changes.
    """
    all_changes = []
    current_page_token = None
    
    while True:
        params = {
            'calendarId': calendar_id,
            'syncToken': sync_token
        }
        
        if current_page_token:
            # Continue pagination with same syncToken
            params['pageToken'] = current_page_token
        
        response = service.events().list(**params).execute()
        all_changes.extend(response.get('items', []))
        
        # Check what we got back
        next_page_token = response.get('nextPageToken')
        next_sync_token = response.get('nextSyncToken')
        
        if next_page_token:
            # More pages to fetch - continue with pageToken
            current_page_token = next_page_token
            print(f"  Paginating: {len(all_changes)} changes so far...")
        elif next_sync_token:
            # Done - got new sync token
            return {
                'changes': all_changes,
                'sync_token': next_sync_token
            }
        else:
            raise Exception("Response missing both pageToken and syncToken")
```

## Best Practices

1. **Always store sync tokens** - Required for incremental sync
2. **Handle 410 errors** - Clear local data and full sync
3. **Use consistent filters** - Same filters in full and incremental syncs
4. **Check last page** - Only last page has nextSyncToken
5. **Process deletions** - Check `status == 'cancelled'`
6. **Paginate properly** - Continue with pageToken until you get syncToken
7. **Don't skip singleEvents** - Usually want `singleEvents=true`
8. **Set reasonable timeMin** - Don't sync ancient events unnecessarily

## Performance Tips

- Use `maxResults=250` for fewer API calls
- Sync only calendars user actively uses
- Schedule background syncs (every 15-30 minutes)
- Combine with push notifications for real-time updates
- Cache events locally to reduce API dependency

## Next Steps

- **Multi-calendar sync:** See `multi-calendar.md`
- **Multi-user sync:** See `multi-user.md`
- **Real-time updates:** See `push-notifications.md`
- **Error recovery:** See `error-handling.md`
