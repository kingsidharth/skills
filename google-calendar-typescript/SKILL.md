---
name: google-calendar-typescript
description: Relevant when connecting your app/code with Google Calendar API v3 via TypeScript/JavaScript. Includes OAuth2, token refresh, CRUD on events, recurring events, conference/join links, RSVP, attendees, incremental sync, push notifications, and more.
---

# Google Calendar API — TypeScript

API v3 via `googleapis` npm package. TypeScript + Node.js.

## Auth & setup

- [AUTH.md](references/AUTH.md) — OAuth2 flows (desktop, web, service account), token persistence, refresh handling, multi-account
- [SCOPES.md](references/SCOPES.md) — All Calendar API scopes with access levels, choosing the narrowest scope

## Calendars

- [CALENDARS.md](references/CALENDARS.md) — Primary vs secondary calendars, Calendars vs CalendarList collections, listing/subscribing, time zones

## Events

- [CREATING-EVENTS.md](references/CREATING-EVENTS.md) — `events.insert()`, timed vs all-day, custom IDs, metadata, reminders, Drive attachments
- [UPDATING-EVENTS.md](references/UPDATING-EVENTS.md) — `events.update()` vs `events.patch()`, partial updates, `sendUpdates`
- [DELETING-EVENTS.md](references/DELETING-EVENTS.md) — `events.delete()`, soft-delete behavior, `showDeleted`
- [CONFERENCING.md](references/CONFERENCING.md) — Creating Meet links, reading join URLs, copying conferences, third-party (Zoom) link extraction

## Attendees & sharing

- [ATTENDEES.md](references/ATTENDEES.md) — Adding attendees, RSVP / responseStatus, invitation delivery, event propagation, forcing events onto calendars
- [SHARING.md](references/SHARING.md) — Calendar ACLs, roles, grantees, event visibility, free/busy queries

## Recurring events

- [RECURRING-EVENTS.md](references/RECURRING-EVENTS.md) — RRULE/RDATE/EXDATE syntax, creating series, listing instances, modifying single instance, cancelling, "this and following" split-and-create, series-wide edits

## Sync & performance

- [SYNC.md](references/SYNC.md) — Incremental sync via syncToken, push notifications (webhooks), CalendarList sync, quotas, batching, error handling
