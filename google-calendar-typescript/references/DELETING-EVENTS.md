# Deleting Events

## Basic delete

```ts
await calendar.events.delete({
  calendarId: "primary",
  eventId: "eventId",
  sendUpdates: "all",
});
```

## Soft-delete behavior

Deleted events are not immediately removed. They remain accessible via `events.list({ showDeleted: true })` with `status: "cancelled"` until garbage collected.

This matters for sync — incremental sync results include cancelled events so you can remove them from local storage.

## Deleting recurring events

- **Entire series**: `events.delete()` on the parent recurring event ID.
- **Single instance**: set `status: "cancelled"` on the instance via `events.update()`. See [RECURRING-EVENTS.md](RECURRING-EVENTS.md).
- **This and following**: trim the parent's RRULE `UNTIL` to cut off at the target instance. See [RECURRING-EVENTS.md](RECURRING-EVENTS.md).

## sendUpdates parameter

Same as update: `"all"`, `"externalOnly"`, or `"none"`.

## Permissions

Requires write access to the calendar. The event organizer can always delete. Attendees can only remove the event from their own calendar (effectively declining).
