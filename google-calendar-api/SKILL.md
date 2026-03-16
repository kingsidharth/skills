# Google Calendar API Skill

## Purpose
Comprehensive reference for Google Calendar API integration including authentication, calendar management, event operations, synchronization, and multi-user/multi-calendar support.

## Critical Rules

1. **ALWAYS use sync tokens for incremental updates** - Never poll by repeatedly listing all events
2. **Handle 410 errors** - Invalid sync token requires full re-sync and clearing stored data
3. **Use pagination properly** - nextSyncToken only appears on LAST page of results
4. **Respect quota limits** - Batch requests when possible, avoid unnecessary API calls
5. **Multi-calendar awareness** - Users can have multiple calendars; each needs separate sync state
6. **Multi-auth support** - Track sync tokens per user AND per calendar

## When to Use This Skill

Use this skill when you need to:
- Authenticate with Google Calendar API (OAuth 2.0)
- List calendars a user has access to
- Sync calendar events efficiently (initial + incremental)
- Create, update, or delete events (single or recurring)
- Manage multiple calendars across multiple user accounts
- Handle push notifications for real-time updates
- Implement batch operations for performance
- Work with calendar sharing and ACLs

## Quick Navigation

**Getting Started:**
- `references/authentication.md` - OAuth 2.0 setup, scopes, token management
- `references/quickstart-python.md` - Working Python examples
- `references/quickstart-nodejs.md` - Working Node.js examples (TODO)

**Core Operations:**
- `references/calendars.md` - List, create, update, delete calendars (TODO)
- `references/events.md` - Event CRUD operations, attendees, reminders (TODO)
- `references/recurring-events.md` - Create and manage recurring events (TODO)

**Advanced Patterns:**
- `references/sync-strategy.md` - Incremental sync with sync tokens (CRITICAL)
- `references/multi-calendar.md` - Handle multiple calendars per user
- `references/multi-user.md` - Manage multiple authenticated accounts
- `references/batch-requests.md` - Combine API calls for performance (TODO)
- `references/push-notifications.md` - Real-time change notifications (TODO)

**Reference:**
- `references/api-reference.md` - Key endpoints and resource structures (TODO)
- `references/error-handling.md` - Common errors and recovery strategies (TODO)

## Architecture Overview

```
User Account (OAuth Token)
├── CalendarList (user's visible calendars)
│   ├── Primary Calendar
│   ├── Secondary Calendar 1
│   ├── Secondary Calendar 2
│   └── Subscribed Calendar
│
├── Each Calendar Has:
│   ├── Events (single + recurring)
│   ├── ACL (access control)
│   └── Metadata (timezone, description)
│
└── Settings (user preferences)
```

## Typical End-to-End Workflows

### Workflow 1: Initial Setup & First Sync (Single User, Single Calendar)
```
1. Setup OAuth 2.0 → references/authentication.md
2. Get credentials and authenticate → references/quickstart-python.md or references/quickstart-nodejs.md
3. List user's calendars → references/calendars.md
4. Perform initial full sync → references/sync-strategy.md (Full Sync section)
5. Store sync token per calendar → references/sync-strategy.md (Storage section)
```

### Workflow 2: Incremental Sync (Ongoing)
```
1. Load stored sync token → references/sync-strategy.md (Storage section)
2. Call events.list() with syncToken → references/sync-strategy.md (Incremental Sync)
3. Handle pagination if needed → references/sync-strategy.md (Pagination section)
4. Process changed/deleted events → references/events.md (Event structure)
5. Store new sync token → references/sync-strategy.md (Storage section)
6. Handle 410 errors (token expired) → references/error-handling.md
```

### Workflow 3: Multi-Calendar Support (Single User)
```
1. List all calendars → references/calendars.md (List Calendars)
2. For each calendar:
   a. Check if sync token exists → references/multi-calendar.md
   b. Perform full or incremental sync → references/sync-strategy.md
   c. Store token per calendar → references/multi-calendar.md (State Management)
3. Update UI/storage with all events → references/multi-calendar.md
```

### Workflow 4: Multi-User Support (Calendar App)
```
1. Authenticate each user (separate OAuth tokens) → references/authentication.md
2. For each user:
   a. List their calendars → references/calendars.md
   b. Sync each calendar → references/sync-strategy.md
   c. Store state: user_id → calendar_id → sync_token → references/multi-user.md
3. Isolate user data → references/multi-user.md (Data Isolation)
```

### Workflow 5: Creating Events
```
Single Event: references/events.md (Create Events)
Recurring Event: references/recurring-events.md (Create Recurring)
With Attendees: references/events.md (Attendees section)
Batch Creation: references/batch-requests.md
```

### Workflow 6: Real-Time Sync with Push Notifications
```
1. Set up webhook endpoint → references/push-notifications.md (Setup)
2. Create watch channel per calendar → references/push-notifications.md (Creating Channels)
3. Receive notification → references/push-notifications.md (Handling)
4. Perform incremental sync → references/sync-strategy.md
5. Renew channel before expiration → references/push-notifications.md (Renewal)
```

### Workflow 7: Performance Optimization
```
1. Identify bulk operations → references/batch-requests.md
2. Combine into batch request → references/batch-requests.md (Examples)
3. Implement sync tokens → references/sync-strategy.md
4. Use partial responses → references/api-reference.md (Fields parameter)
5. Monitor quotas → references/error-handling.md (Quota errors)
```

## Decision Trees

### "I need to get calendar data"
```
Do you need all events or just recent changes?
├─ All events (first time)
│  └─ Use: references/sync-strategy.md (Full Sync)
└─ Recent changes only
   ├─ Have sync token?
   │  ├─ Yes → Use: references/sync-strategy.md (Incremental Sync)
   │  └─ No → Use: references/sync-strategy.md (Full Sync first)
   └─ Got 410 error?
      └─ Use: references/error-handling.md (Invalidated Token)
```

### "I need to work with events"
```
What operation?
├─ Create
│  ├─ Single event → references/events.md (Create)
│  ├─ Recurring event → references/recurring-events.md (Create)
│  └─ Multiple events → references/batch-requests.md
├─ Update
│  ├─ Single instance → references/events.md (Update)
│  ├─ Recurring series → references/recurring-events.md (Modify All)
│  └─ This and following → references/recurring-events.md (Modify Following)
└─ Delete
   └─ references/events.md (Delete)
```

### "I'm working with multiple calendars"
```
How many users?
├─ Single user, multiple calendars
│  └─ Use: references/multi-calendar.md
└─ Multiple users, multiple calendars each
   └─ Use: references/multi-user.md + references/multi-calendar.md
```

## Common Patterns

### Pattern: Efficient Event Sync
```python
# See references/sync-strategy.md for complete implementation
def sync_calendar(calendar_id, stored_sync_token):
    if stored_sync_token:
        # Incremental sync
        return events_service.list(
            calendarId=calendar_id,
            syncToken=stored_sync_token
        )
    else:
        # Full sync
        return events_service.list(
            calendarId=calendar_id,
            timeMin=one_year_ago  # Optional filter
        )
```

### Pattern: Multi-Calendar State
```python
# See references/multi-calendar.md for complete implementation
sync_state = {
    "primary": {
        "sync_token": "token123",
        "last_sync": "2025-02-17T10:00:00Z"
    },
    "work@example.com": {
        "sync_token": "token456",
        "last_sync": "2025-02-17T09:30:00Z"
    }
}
```

### Pattern: Error Recovery
```python
# See references/error-handling.md for complete implementation
try:
    result = service.events().list(
        calendarId='primary',
        syncToken=stored_token
    ).execute()
except HttpError as error:
    if error.resp.status == 410:
        # Token invalidated - perform full sync
        clear_sync_token(calendar_id)
        return full_sync(calendar_id)
```

## API Base URL
```
https://www.googleapis.com/calendar/v3
```

## Key Resources

- **CalendarList**: `/users/me/calendarList` - User's calendar subscriptions
- **Calendars**: `/calendars/{calendarId}` - Calendar metadata
- **Events**: `/calendars/{calendarId}/events` - Calendar events
- **ACL**: `/calendars/{calendarId}/acl` - Sharing permissions
- **Settings**: `/users/me/settings` - User preferences
- **Colors**: `/colors` - Available color definitions
- **FreeBusy**: `/freeBusy` - Availability queries

## Special Identifiers

- `"primary"` - User's primary calendar
- `"me"` - Authenticated user in endpoints

## Next Steps After Reading This File

1. **New to the API?** Start with `references/authentication.md` → `references/quickstart-python.md`
2. **Need to sync?** Go directly to `references/sync-strategy.md` (most important)
3. **Working with events?** See `references/events.md` and `references/recurring-events.md`
4. **Multiple calendars?** See `references/multi-calendar.md` or `references/multi-user.md`
5. **Performance issues?** Check `references/batch-requests.md` and `references/sync-strategy.md`
6. **Errors?** See `references/error-handling.md`

## File Organization

```
SKILL.md (this file) - Entry point and navigation
└── references/
    ├── authentication.md - OAuth 2.0 and scopes
    ├── quickstart-python.md - Python examples
    ├── quickstart-nodejs.md - Node.js examples (TODO)
    ├── calendars.md - Calendar operations (TODO)
    ├── events.md - Event CRUD (TODO)
    ├── recurring-events.md - Recurring event patterns (TODO)
    ├── sync-strategy.md - ⭐ CRITICAL: Incremental sync
    ├── multi-calendar.md - Multiple calendars per user
    ├── multi-user.md - Multiple user accounts
    ├── batch-requests.md - Performance optimization (TODO)
    ├── push-notifications.md - Real-time updates (TODO)
    ├── api-reference.md - Endpoints and structures (TODO)
    └── error-handling.md - Error recovery (TODO)
```
