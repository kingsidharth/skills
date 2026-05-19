# Recurring Events

## Creating a recurring event

Set the `recurrence` field with RFC 5545 rules. The `start`/`end` define the first occurrence; `recurrence` defines the repeat pattern.

```ts
await calendar.events.insert({
  calendarId: "primary",
  requestBody: {
    summary: "Weekly standup",
    start: {
      dateTime: "2025-06-02T09:00:00",
      timeZone: "America/Los_Angeles",
    },
    end: {
      dateTime: "2025-06-02T09:30:00",
      timeZone: "America/Los_Angeles",
    },
    recurrence: ["RRULE:FREQ=WEEKLY;BYDAY=MO;COUNT=12"],
  },
});
```

Recurring events must specify a single `timeZone` (same for start and end).

## RRULE syntax

Core components of `RRULE:`:

| Component | Purpose | Example |
|---|---|---|
| `FREQ` | Repeat frequency (required) | `DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY` |
| `INTERVAL` | Every N intervals | `INTERVAL=2` = every 2 weeks |
| `COUNT` | Total occurrences | `COUNT=10` |
| `UNTIL` | End date (inclusive) | `UNTIL=20251231T235959Z` |
| `BYDAY` | Days of week | `BYDAY=MO,WE,FR` |
| `BYMONTH` | Months | `BYMONTH=1,6` |
| `BYMONTHDAY` | Day of month | `BYMONTHDAY=15` |
| `BYHOUR` | Hours | `BYHOUR=9,17` |

Use either `COUNT` or `UNTIL`, never both.

**RDATE** — add extra dates not covered by RRULE:
`RDATE;VALUE=DATE:20250704,20251225`

**EXDATE** — exclude specific dates:
`EXDATE;VALUE=DATE:20250901`

For timed events use `EXDATE;TZID=America/Los_Angeles:20250901T090000`. For all-day events use `VALUE=DATE`.

The `recurrence` array can contain multiple rules. The final set = union of all RRULE + RDATE, minus EXDATE.

## Common patterns

```
# Every weekday
RRULE:FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR

# First Monday of every month
RRULE:FREQ=MONTHLY;BYDAY=1MO

# Every 2 weeks on Tuesday and Thursday, 10 occurrences
RRULE:FREQ=WEEKLY;INTERVAL=2;BYDAY=TU,TH;COUNT=10

# Yearly on March 15
RRULE:FREQ=YEARLY;BYMONTH=3;BYMONTHDAY=15
```

## Listing instances

```ts
const res = await calendar.events.instances({
  calendarId: "primary",
  eventId: "recurringEventId",
  timeMin: new Date().toISOString(),
  maxResults: 10,
});
```

Returns individual event objects with:

- `recurringEventId` — parent recurring event ID
- `originalStartTime` — when this instance was scheduled (even if rescheduled)
- No `recurrence` field (only the parent has it)

`events.list()` with default settings returns recurring events + exceptions but not plain instances. Set `singleEvents: true` to expand all instances (but then recurring event parents are omitted).

## Modifying a single instance

Fetch the instance, modify, then update:

```ts
const instances = await calendar.events.instances({
  calendarId: "primary",
  eventId: "recurringEventId",
});
const instance = instances.data.items![0];

// Reschedule this one instance
instance.start = { dateTime: "2025-06-02T10:00:00", timeZone: "America/Los_Angeles" };
instance.end = { dateTime: "2025-06-02T10:30:00", timeZone: "America/Los_Angeles" };

await calendar.events.update({
  calendarId: "primary",
  eventId: instance.id!,
  requestBody: instance,
});
```

This creates an **exception** — an instance that differs from the parent.

## Cancelling a single instance

Set `status: "cancelled"` on the instance:

```ts
instance.status = "cancelled";
await calendar.events.update({
  calendarId: "primary",
  eventId: instance.id!,
  requestBody: instance,
});
```

The instance remains accessible via sync with `showDeleted: true`.

## Modifying "this and all following"

This requires two API calls — a split-and-create:

1. **Trim the original**: update the parent's `recurrence` RRULE by setting `UNTIL` to just before the target instance's start.
2. **Create new recurring event**: insert a new event starting at the target instance's time, with the same recurrence pattern but with the desired changes applied.

```ts
// Step 1: Trim original to end before instance #3
await calendar.events.update({
  calendarId: "primary",
  eventId: "recurringEventId",
  requestBody: {
    ...originalEvent,
    recurrence: ["RRULE:FREQ=WEEKLY;UNTIL=20250616T065959Z"],
  },
});

// Step 2: Create new series from instance #3 onward with changes
await calendar.events.insert({
  calendarId: "primary",
  requestBody: {
    summary: "Standup (new room)",
    location: "Room B",  // the change
    start: { dateTime: "2025-06-16T09:00:00", timeZone: "America/Los_Angeles" },
    end: { dateTime: "2025-06-16T09:30:00", timeZone: "America/Los_Angeles" },
    recurrence: ["RRULE:FREQ=WEEKLY;COUNT=9"],
    attendees: originalEvent.attendees,
  },
});
```

Warning: "this and following" resets all exceptions after the split point.

## Modifying the entire series

Update the parent recurring event directly. Changes propagate to all non-exception instances. Do not modify instances individually for series-wide changes — it creates excessive exceptions, slows access, and floods attendees with notifications.

## Deleting a recurring event

`events.delete()` on the parent ID deletes the entire series. To delete a single instance, cancel that instance. To delete "this and following", trim the parent's RRULE.
