# Attendees

## Adding attendees

Add via `attendees[]` in the event body on insert or update:

```ts
attendees: [
  { email: "alice@example.com" },
  { email: "bob@example.com", optional: true },
],
```

Set `sendUpdates: "all"` to send email invitations, `"externalOnly"` for non-Google attendees only, `"none"` for silent add.

## RSVP / response status

`attendees[].responseStatus` values: `needsAction`, `declined`, `tentative`, `accepted`.

Pre-setting `responseStatus` on insert does NOT auto-add the event to the attendee's calendar. To force-add, either update the event on the attendee's calendar setting their response to `accepted`, or use `events.import()` with the same `iCalUID`.

**Response status is the only attendee change that propagates back to the organizer.** All other attendee-side changes are local to their copy.

If >200 attendees, response status is not propagated.

## Event propagation model

1. Organizer creates event → organizer copy lives on organizer's calendar.
2. Each attendee gets an attendee copy.
3. Organizer updates shared properties (summary, time, location, description) → propagated to all attendee copies.
4. Attendee RSVP → propagated back to organizer copy.

### Private (per-copy) properties

NOT synced between copies: reminders, `colorId`, `transparency`, `extendedProperties.private`.

Attendees can modify shared properties on their copy, but those changes are local and may be overwritten on the next organizer update.

## Invitation delivery

How invitations appear depends on the attendee's Google Calendar settings:

- **"From everyone"** — event auto-added to calendar
- **"Only if sender is known"** — auto-added if organizer is a contact, in the same org, or previously interacted. Otherwise attendee must RSVP from email.
- **"When I respond in email"** — event only added after clicking Yes/Maybe/No in email.

## Forcing events onto attendee calendars

Options (all require write access to the attendee's calendar):

1. **Set RSVP programmatically**: update the event on the attendee's calendar, set `responseStatus: "accepted"`.
2. **Import a copy**: `events.import()` with the same `iCalUID` on both organizer's and attendee's calendars. No invitation email sent.
3. **Add organizer to contacts**: if attendee uses "Only if sender is known," adding the organizer's email to their Google Contacts makes future invitations auto-appear.
