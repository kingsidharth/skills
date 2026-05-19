# Updating Events

## update vs patch

`events.update()` — full replacement. You must send the complete event body; omitted fields are cleared.

`events.patch()` — partial merge. Only provided fields are modified; everything else is preserved.

Prefer `patch()` for targeted changes.

## Example: patch

```ts
await calendar.events.patch({
  calendarId: "primary",
  eventId: "eventId",
  sendUpdates: "all",
  requestBody: {
    summary: "Updated title",
    location: "New location",
  },
});
```

## Example: update

```ts
// Fetch the full event first
const res = await calendar.events.get({
  calendarId: "primary",
  eventId: "eventId",
});
const event = res.data;

// Modify
event.summary = "Updated title";

await calendar.events.update({
  calendarId: "primary",
  eventId: "eventId",
  sendUpdates: "all",
  requestBody: event,
});
```

## sendUpdates parameter

Controls email notifications to attendees:

- `"all"` — notify all attendees
- `"externalOnly"` — notify only non-Google attendees
- `"none"` — silent update

## Modifying attendees

Always send the full `attendees[]` array — it's a replace, not a merge. Omitting an attendee removes them from the event.

If >200 attendees, response status is not propagated.

## Etag / conditional updates

Events have an `etag` field. Use `If-Match` header with the etag value to prevent overwriting concurrent changes. Returns `412 Precondition Failed` on conflict.
