# Calendars

## Calendar types

**Primary calendar** — auto-created per Google account. ID = user's email address. Cannot be deleted.

**Secondary calendars** — created via `calendars.insert()`. Can be deleted, shared, ownership transferred.

## Calendars vs CalendarList

| Operation | `calendars` | `calendarList` |
|---|---|---|
| `insert` | Creates a new secondary calendar | Subscribes user to existing calendar |
| `delete` | Deletes the calendar permanently | Removes from user's list only |
| `get` | Returns global metadata (title, tz) | Returns metadata + user-specific overrides (color, reminders) |
| `patch`/`update` | Modifies global metadata | Modifies user-specific settings |

The data owner cannot remove a calendar from their CalendarList.

## Listing all calendars

```ts
const res = await calendar.calendarList.list();
for (const cal of res.data.items ?? []) {
  console.log(cal.id, cal.summary, cal.accessRole);
  // accessRole: "owner" | "writer" | "reader" | "freeBusyReader"
}
```

Use `calendarList.list()` to let users pick which calendars to work with. The `accessRole` field determines what operations are allowed.

## Time zones

Use IANA identifiers (`America/Los_Angeles`, `Asia/Kolkata`).

**Calendar timezone** — the default for query results and all-day event boundary matching. Set via `calendars.update()`.

**Event timezone** — attached to `start.timeZone` / `end.timeZone`. For recurring events, a single timezone is required.

Ways to specify event time:

- Offset in dateTime: `2025-06-01T09:00:00-07:00`
- No offset + timeZone field: `2025-06-01T09:00:00` with `timeZone: "America/Los_Angeles"`
- UTC: `2025-06-01T16:00:00Z`

The `timeZone` query parameter on `events.list()` / `events.get()` converts result times. Defaults to calendar timezone if omitted.
