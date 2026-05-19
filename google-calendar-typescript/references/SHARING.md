# Sharing & ACLs

## Calendar sharing

Controlled via the ACL collection. Each entry grants a role to a grantee.

| Role | Access |
|---|---|
| `none` | No access |
| `freeBusyReader` | See free/busy only |
| `reader` | Read events |
| `writer` | Read/write events, see ACLs |
| `owner` | Writer + manage access levels |

The `owner` role is distinct from "data owner." A calendar has one data owner (the creator) but can have multiple users with `owner` role.

Grantees: individual user, group, domain, or public. Up to 6,000 ACL entries per calendar.

Sharing a calendar does NOT auto-add it to the recipient's CalendarList. Call `calendarList.insert()` separately.

Google Workspace domain settings may cap the maximum sharing level for external users.

## Event visibility

Per-event override on shared calendars:

| Visibility | Effect |
|---|---|
| `default` | Follows calendar ACL |
| `public` | Details visible to anyone with `freeBusyReader`+ access |
| `private` | Details visible only to `writer`+ access |

No effect on non-shared calendars.

## Free/busy queries

```ts
const res = await calendar.freebusy.query({
  requestBody: {
    timeMin: "2025-06-01T00:00:00Z",
    timeMax: "2025-06-02T00:00:00Z",
    items: [
      { id: "alice@example.com" },
      { id: "bob@example.com" },
    ],
  },
});
// res.data.calendars["alice@example.com"].busy -> [{start, end}, ...]
```

Requires `calendar.freebusy` scope at minimum.

## Suggesting event times

No built-in "suggest time" endpoint. Build on free/busy:

1. Query free/busy for all intended attendees.
2. Find overlapping free windows matching your duration.
3. Present candidates to the user.
