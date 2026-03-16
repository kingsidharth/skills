# Multi-Calendar Management

## Overview

Users can have multiple calendars:
- **Primary calendar** - User's main calendar
- **Secondary calendars** - Created by user
- **Subscribed calendars** - Shared calendars from others

Each calendar needs independent sync state tracking.

## Discovering User's Calendars

### List All Calendars

```python
def list_user_calendars(service):
    """
    Get all calendars user has access to.
    
    Returns:
        list: Calendar entries with metadata
    """
    calendar_list = service.calendarList().list().execute()
    
    calendars = []
    for calendar_entry in calendar_list.get('items', []):
        calendars.append({
            'id': calendar_entry['id'],
            'summary': calendar_entry.get('summary', 'Untitled'),
            'primary': calendar_entry.get('primary', False),
            'access_role': calendar_entry.get('accessRole'),
            'background_color': calendar_entry.get('backgroundColor'),
            'selected': calendar_entry.get('selected', False)
        })
    
    return calendars
```

### Node.js Version

```javascript
async function listUserCalendars(calendar) {
  const response = await calendar.calendarList.list();
  
  return response.data.items.map(entry => ({
    id: entry.id,
    summary: entry.summary || 'Untitled',
    primary: entry.primary || false,
    accessRole: entry.accessRole,
    backgroundColor: entry.backgroundColor,
    selected: entry.selected || false
  }));
}
```

### Calendar Access Roles

- `owner` - Full control (create, delete, share)
- `writer` - Can add/edit events
- `reader` - View-only access
- `freeBusyReader` - See free/busy only

## State Management Pattern

### Per-Calendar Sync State

```python
import json
from datetime import datetime, timezone

class CalendarSyncManager:
    """Manages sync state for multiple calendars."""
    
    def __init__(self, state_file='calendar_sync_state.json'):
        self.state_file = state_file
        self.state = self._load_state()
    
    def _load_state(self):
        """Load sync state from file."""
        try:
            with open(self.state_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            return {'calendars': {}}
    
    def _save_state(self):
        """Save sync state to file."""
        with open(self.state_file, 'w') as f:
            json.dump(self.state, f, indent=2)
    
    def get_sync_token(self, calendar_id):
        """Get sync token for specific calendar."""
        return self.state['calendars'].get(calendar_id, {}).get('sync_token')
    
    def save_sync_token(self, calendar_id, sync_token):
        """Save sync token for specific calendar."""
        if calendar_id not in self.state['calendars']:
            self.state['calendars'][calendar_id] = {}
        
        self.state['calendars'][calendar_id]['sync_token'] = sync_token
        self.state['calendars'][calendar_id]['last_sync'] = \
            datetime.now(timezone.utc).isoformat()
        
        self._save_state()
    
    def clear_calendar(self, calendar_id):
        """Clear sync state for calendar (e.g., after 410 error)."""
        if calendar_id in self.state['calendars']:
            del self.state['calendars'][calendar_id]
            self._save_state()
    
    def get_all_calendar_ids(self):
        """Get list of all tracked calendar IDs."""
        return list(self.state['calendars'].keys())
```

### State File Structure

```json
{
  "calendars": {
    "primary": {
      "sync_token": "CPDAlvWDx70CEPDAlvWDx70CGAU=",
      "last_sync": "2025-02-17T10:30:00Z"
    },
    "work@company.com": {
      "sync_token": "XYZ123abc456def==",
      "last_sync": "2025-02-17T10:28:00Z"
    },
    "family@gmail.com": {
      "sync_token": "ABC789xyz123==",
      "last_sync": "2025-02-17T10:25:00Z"
    }
  }
}
```

## Syncing All Calendars

### Complete Multi-Calendar Sync

```python
from googleapiclient.errors import HttpError

def sync_all_calendars(service, sync_manager, event_storage):
    """
    Sync all calendars user has access to.
    
    Args:
        service: Authorized Calendar API service
        sync_manager: CalendarSyncManager instance
        event_storage: Dict to store events by calendar
    
    Returns:
        dict: Sync results per calendar
    """
    # Get list of calendars
    calendar_list = service.calendarList().list().execute()
    
    results = {}
    
    for calendar_entry in calendar_list.get('items', []):
        calendar_id = calendar_entry['id']
        calendar_name = calendar_entry.get('summary', calendar_id)
        
        # Skip if user hasn't selected this calendar
        if not calendar_entry.get('selected', True):
            print(f"Skipping unselected calendar: {calendar_name}")
            continue
        
        print(f"\nSyncing calendar: {calendar_name}")
        
        try:
            # Get stored sync token for this calendar
            sync_token = sync_manager.get_sync_token(calendar_id)
            
            # Perform sync
            result = sync_calendar_events(service, calendar_id, sync_token)
            
            # Initialize storage for this calendar if needed
            if calendar_id not in event_storage:
                event_storage[calendar_id] = {}
            
            # Process events
            process_sync_results(result, event_storage[calendar_id])
            
            # Save new sync token
            sync_manager.save_sync_token(calendar_id, result['sync_token'])
            
            results[calendar_id] = {
                'success': True,
                'events_count': len(event_storage[calendar_id]),
                'full_sync': result.get('full_sync', False)
            }
            
        except HttpError as error:
            if error.resp.status == 410:
                # Sync token invalidated
                print(f"  Sync token invalidated for {calendar_name}")
                sync_manager.clear_calendar(calendar_id)
                event_storage[calendar_id] = {}  # Clear events
                
                # Retry with full sync
                result = sync_calendar_events(service, calendar_id, None)
                process_sync_results(result, event_storage[calendar_id])
                sync_manager.save_sync_token(calendar_id, result['sync_token'])
                
                results[calendar_id] = {
                    'success': True,
                    'events_count': len(event_storage[calendar_id]),
                    'full_sync': True,
                    'recovered_410': True
                }
            else:
                # Other error
                results[calendar_id] = {
                    'success': False,
                    'error': str(error)
                }
    
    return results
```

### Node.js Version

```javascript
async function syncAllCalendars(calendar, syncManager, eventStorage) {
  const calendarListResponse = await calendar.calendarList.list();
  const results = {};
  
  for (const calendarEntry of calendarListResponse.data.items) {
    const calendarId = calendarEntry.id;
    const calendarName = calendarEntry.summary || calendarId;
    
    // Skip unselected calendars
    if (!calendarEntry.selected) {
      console.log(`Skipping unselected calendar: ${calendarName}`);
      continue;
    }
    
    console.log(`\nSyncing calendar: ${calendarName}`);
    
    try {
      const syncToken = syncManager.getSyncToken(calendarId);
      const result = await syncCalendarEvents(calendar, calendarId, syncToken);
      
      if (!eventStorage[calendarId]) {
        eventStorage[calendarId] = {};
      }
      
      processSyncResults(result, eventStorage[calendarId]);
      syncManager.saveSyncToken(calendarId, result.syncToken);
      
      results[calendarId] = {
        success: true,
        eventsCount: Object.keys(eventStorage[calendarId]).length,
        fullSync: result.fullSync || false
      };
      
    } catch (error) {
      if (error.code === 410) {
        console.log(`  Sync token invalidated for ${calendarName}`);
        syncManager.clearCalendar(calendarId);
        eventStorage[calendarId] = {};
        
        const result = await syncCalendarEvents(calendar, calendarId, null);
        processSyncResults(result, eventStorage[calendarId]);
        syncManager.saveSyncToken(calendarId, result.syncToken);
        
        results[calendarId] = {
          success: true,
          eventsCount: Object.keys(eventStorage[calendarId]).length,
          fullSync: true,
          recovered410: true
        };
      } else {
        results[calendarId] = {
          success: false,
          error: error.message
        };
      }
    }
  }
  
  return results;
}
```

## Selective Calendar Sync

### Sync Specific Calendars Only

```python
def sync_selected_calendars(service, sync_manager, event_storage, 
                           calendar_ids=None):
    """
    Sync only specified calendars.
    
    Args:
        calendar_ids: List of calendar IDs to sync. If None, syncs all.
    """
    if calendar_ids is None:
        # Sync all calendars
        return sync_all_calendars(service, sync_manager, event_storage)
    
    results = {}
    
    for calendar_id in calendar_ids:
        try:
            # Get calendar info
            calendar_info = service.calendars().get(
                calendarId=calendar_id
            ).execute()
            
            print(f"\nSyncing: {calendar_info.get('summary', calendar_id)}")
            
            # Sync this calendar
            sync_token = sync_manager.get_sync_token(calendar_id)
            result = sync_calendar_events(service, calendar_id, sync_token)
            
            if calendar_id not in event_storage:
                event_storage[calendar_id] = {}
            
            process_sync_results(result, event_storage[calendar_id])
            sync_manager.save_sync_token(calendar_id, result['sync_token'])
            
            results[calendar_id] = {
                'success': True,
                'events_count': len(event_storage[calendar_id])
            }
            
        except HttpError as error:
            results[calendar_id] = {
                'success': False,
                'error': str(error)
            }
    
    return results
```

## Filtering Calendars

### By Access Level

```python
def get_calendars_by_access(service, required_role='writer'):
    """
    Get calendars with specific access level.
    
    Args:
        required_role: 'owner', 'writer', 'reader', or 'freeBusyReader'
    """
    calendar_list = service.calendarList().list().execute()
    
    filtered = []
    for entry in calendar_list.get('items', []):
        if entry.get('accessRole') == required_role:
            filtered.append(entry)
    
    return filtered
```

### By Selection Status

```python
def get_selected_calendars(service):
    """Get only calendars user has selected (visible in UI)."""
    calendar_list = service.calendarList().list().execute()
    
    return [
        entry for entry in calendar_list.get('items', [])
        if entry.get('selected', True)  # Default True if not specified
    ]
```

## Event Storage Pattern

### Organize Events by Calendar

```python
class MultiCalendarEventStorage:
    """Storage for events across multiple calendars."""
    
    def __init__(self):
        self.calendars = {}
    
    def add_event(self, calendar_id, event):
        """Add event to specific calendar."""
        if calendar_id not in self.calendars:
            self.calendars[calendar_id] = {}
        
        event_id = event['id']
        self.calendars[calendar_id][event_id] = event
    
    def remove_event(self, calendar_id, event_id):
        """Remove event from specific calendar."""
        if calendar_id in self.calendars:
            self.calendars[calendar_id].pop(event_id, None)
    
    def get_calendar_events(self, calendar_id):
        """Get all events for specific calendar."""
        return self.calendars.get(calendar_id, {})
    
    def get_all_events(self):
        """Get all events from all calendars."""
        all_events = []
        for calendar_id, events in self.calendars.items():
            for event in events.values():
                # Add calendar_id to each event
                event_copy = event.copy()
                event_copy['_calendar_id'] = calendar_id
                all_events.append(event_copy)
        return all_events
    
    def search_events(self, query):
        """Search events across all calendars."""
        results = []
        query_lower = query.lower()
        
        for calendar_id, events in self.calendars.items():
            for event in events.values():
                summary = event.get('summary', '').lower()
                description = event.get('description', '').lower()
                
                if query_lower in summary or query_lower in description:
                    event_copy = event.copy()
                    event_copy['_calendar_id'] = calendar_id
                    results.append(event_copy)
        
        return results
```

## Adding/Removing Calendars

### Subscribe to Calendar

```python
def subscribe_to_calendar(service, calendar_id):
    """
    Add calendar to user's calendar list.
    
    Args:
        calendar_id: Email of calendar or calendar ID to subscribe to
    """
    calendar_list_entry = {
        'id': calendar_id
    }
    
    created = service.calendarList().insert(
        body=calendar_list_entry
    ).execute()
    
    print(f"Subscribed to: {created.get('summary')}")
    return created
```

### Unsubscribe from Calendar

```python
def unsubscribe_from_calendar(service, sync_manager, calendar_id):
    """
    Remove calendar from user's calendar list.
    Also cleans up sync state.
    """
    # Remove from Calendar API
    service.calendarList().delete(calendarId=calendar_id).execute()
    
    # Clean up sync state
    sync_manager.clear_calendar(calendar_id)
    
    print(f"Unsubscribed from: {calendar_id}")
```

## Performance Considerations

### Parallel Sync (Advanced)

```python
import concurrent.futures

def sync_calendars_parallel(service, sync_manager, event_storage, 
                           calendar_ids):
    """
    Sync multiple calendars in parallel.
    Use with caution - respects quota limits.
    """
    def sync_one_calendar(calendar_id):
        try:
            sync_token = sync_manager.get_sync_token(calendar_id)
            result = sync_calendar_events(service, calendar_id, sync_token)
            
            if calendar_id not in event_storage:
                event_storage[calendar_id] = {}
            
            process_sync_results(result, event_storage[calendar_id])
            sync_manager.save_sync_token(calendar_id, result['sync_token'])
            
            return (calendar_id, True, len(event_storage[calendar_id]))
        except Exception as e:
            return (calendar_id, False, str(e))
    
    # Use thread pool (max 5 concurrent)
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(sync_one_calendar, cal_id): cal_id 
            for cal_id in calendar_ids
        }
        
        results = {}
        for future in concurrent.futures.as_completed(futures):
            cal_id, success, data = future.result()
            results[cal_id] = {
                'success': success,
                'data': data
            }
    
    return results
```

## Calendar Metadata

### Get Calendar Details

```python
def get_calendar_metadata(service, calendar_id):
    """Get detailed calendar information."""
    calendar = service.calendars().get(calendarId=calendar_id).execute()
    
    return {
        'id': calendar['id'],
        'summary': calendar.get('summary'),
        'description': calendar.get('description'),
        'location': calendar.get('location'),
        'timezone': calendar.get('timeZone'),
        'etag': calendar.get('etag')
    }
```

### Update Calendar Settings

```python
def update_calendar_display(service, calendar_id, color=None, selected=None):
    """
    Update how calendar appears in user's list.
    
    Args:
        color: Color ID or hex color
        selected: Boolean - show/hide calendar
    """
    calendar_list_entry = {}
    
    if color:
        calendar_list_entry['colorId'] = color
    if selected is not None:
        calendar_list_entry['selected'] = selected
    
    updated = service.calendarList().patch(
        calendarId=calendar_id,
        body=calendar_list_entry
    ).execute()
    
    return updated
```

## Best Practices

1. **Independent sync per calendar** - Each calendar needs own sync token
2. **Handle access changes** - User permissions can change
3. **Respect selected status** - Don't sync unselected calendars
4. **Store calendar metadata** - Cache names, colors for UI
5. **Clean up on unsubscribe** - Remove sync state when calendar removed
6. **Primary calendar special** - Use "primary" identifier reliably

## Common Patterns

### Find Primary Calendar

```python
def get_primary_calendar_id(service):
    """Get user's primary calendar ID."""
    calendar_list = service.calendarList().list().execute()
    
    for entry in calendar_list.get('items', []):
        if entry.get('primary'):
            return entry['id']
    
    # Fallback - primary is usually user's email
    return 'primary'
```

### Sync Only Owned Calendars

```python
def sync_owned_calendars(service, sync_manager, event_storage):
    """Sync only calendars user owns."""
    calendar_list = service.calendarList().list().execute()
    
    owned_calendars = [
        entry['id'] for entry in calendar_list.get('items', [])
        if entry.get('accessRole') == 'owner'
    ]
    
    return sync_selected_calendars(
        service, sync_manager, event_storage, owned_calendars
    )
```

## Next Steps

- **Multi-user support:** See `multi-user.md`
- **Event operations:** See `events.md`
- **Sync implementation:** See `sync-strategy.md`
- **Error handling:** See `error-handling.md`
