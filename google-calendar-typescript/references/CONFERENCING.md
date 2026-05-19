# Conferencing & Join Links

## Creating a Meet link

```ts
const event = await calendar.events.insert({
  calendarId: "primary",
  conferenceDataVersion: 1,  // required
  requestBody: {
    summary: "Video call",
    start: { dateTime: "2025-06-01T14:00:00Z" },
    end: { dateTime: "2025-06-01T15:00:00Z" },
    conferenceData: {
      createRequest: {
        requestId: crypto.randomUUID(),  // unique per request
        conferenceSolutionKey: { type: "hangoutsMeet" },
      },
    },
  },
});
```

Conference creation is async. The response may have `status.statusCode === "pending"`. Poll with `events.get()` until `"success"`, then read `conferenceData.entryPoints[]`.

Perform a full sync before enabling `conferenceDataVersion: 1` for the first time in an existing app, or you may strip existing conferences.

## Conference solution types

| Type | Value |
|---|---|
| Google Meet | `hangoutsMeet` |
| Hangouts (consumer) | `eventHangout` |
| Classic Hangouts (Workspace, deprecated) | `eventNamedHangout` |

Check `conferenceProperties.allowedConferenceSolutionTypes` on the calendar to see what's supported.

## Reading join links

```ts
const entryPoints = event.data.conferenceData?.entryPoints ?? [];
for (const ep of entryPoints) {
  // ep.entryPointType: "video" | "phone" | "sip" | "more"
  // ep.uri: the join URL
  // ep.label: human-readable label
  console.log(ep.entryPointType, ep.uri);
}
```

Also check `event.data.hangoutLink` for legacy Hangout links.

## Copying a conference to another event

Copy the entire `conferenceData` object from one event to another via `events.patch()`. Both events share the same conference session. Pass `conferenceDataVersion: 1`.

## Third-party conferencing (Zoom etc.)

Third-party links only appear in `conferenceData.entryPoints[]` if the provider has a Calendar add-on installed. Otherwise, look in:

- `event.data.location` — Zoom often sets the join URL here
- `event.data.description` — join link embedded in the body

Parse these fields as fallback when `conferenceData` is empty.
