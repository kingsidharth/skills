# Creating Events

## Basic insert

```ts
const event = await calendar.events.insert({
  calendarId: "primary",
  requestBody: {
    summary: "Team standup",
    location: "Conference Room A",
    description: "Daily sync",
    start: {
      dateTime: "2025-06-01T09:00:00",
      timeZone: "America/Los_Angeles",
    },
    end: {
      dateTime: "2025-06-01T09:30:00",
      timeZone: "America/Los_Angeles",
    },
  },
});
console.log(event.data.htmlLink);
```

Requires `calendar` or `calendar.events` scope and write access on the target calendar.

## Timed vs all-day

**Timed events** — use `start.dateTime` + `end.dateTime`.

**All-day events** — use `start.date` + `end.date` (format `YYYY-MM-DD`). End date is exclusive (1-day event on June 1: `start.date: "2025-06-01"`, `end.date: "2025-06-02"`). `timeZone` is ignored for all-day events.

Never mix `date` and `dateTime` in start/end.

## Custom event IDs

Set `id` on insert for idempotent creation. Format: lowercase `a-v` and digits `0-9`, length 5–1024. Duplicate ID → `409`.

## Reminders

```ts
reminders: {
  useDefault: false,
  overrides: [
    { method: "email", minutes: 1440 },  // 24 hours
    { method: "popup", minutes: 10 },
  ],
}
```

`method`: `"email"` or `"popup"`. Reminders are per-user (private property), not shared with attendees. Set `useDefault: true` to inherit from CalendarList defaults.

## Drive attachments

```ts
await calendar.events.patch({
  calendarId: "primary",
  eventId: "existingEventId",
  supportsAttachments: true,
  requestBody: {
    attachments: [{
      fileUrl: "https://drive.google.com/...",
      mimeType: "application/pdf",
      title: "Meeting notes",
    }],
  },
});
```

Perform a full sync before enabling `supportsAttachments` for the first time, or you may strip existing attachments.

## Key event fields

| Field | Notes |
|---|---|
| `id` | Server-generated or client-provided |
| `iCalUID` | Globally unique across calendars. Used for `events.import()`. |
| `htmlLink` | Direct link to event in Google Calendar UI |
| `summary` | Title |
| `description` | Body text (HTML allowed) |
| `location` | Address string (enables Maps integration) |
| `status` | `confirmed`, `tentative`, `cancelled` |
| `visibility` | `default`, `public`, `private` |
| `transparency` | `opaque` (busy) or `transparent` (available) |
| `colorId` | Event color override (1–11) |
| `extendedProperties` | Custom key-value: `shared` (all users) and `private` (per-user) |
