# Scopes

All scopes are prefixed with `https://www.googleapis.com/auth/`.

Choose the narrowest scope that covers your use case. Users are more likely to approve limited access. You can request multiple scopes in the `scope` array.

## Scope reference

| Scope suffix | Access |
|---|---|
| `calendar` | Full read/write on all calendars + ACLs |
| `calendar.readonly` | Read-only on all calendars |
| `calendar.events` | Read/write events on all calendars |
| `calendar.events.readonly` | Read events on all calendars |
| `calendar.events.owned` | CRUD only on events the user owns |
| `calendar.events.owned.readonly` | Read only owned events |
| `calendar.freebusy` | View free/busy only |
| `calendar.settings.readonly` | Read calendar settings |
| `calendar.calendarlist` | Manage subscribed calendar list |
| `calendar.calendarlist.readonly` | Read subscribed calendar list |
| `calendar.calendars` | Manage calendar metadata + create secondary |
| `calendar.calendars.readonly` | Read calendar metadata |
| `calendar.acls` | Manage sharing permissions |
| `calendar.acls.readonly` | Read sharing permissions |
| `calendar.app.created` | Manage secondary calendars created by the app |
| `calendar.events.public.readonly` | Read events on public calendars |

## Common combinations

**Read-only calendar viewer**: `calendar.readonly`

**Event CRUD (no calendar management)**: `calendar.events`

**Full scheduling app**: `calendar.events` + `calendar.freebusy` + `calendar.calendarlist.readonly`

**App-managed secondary calendars only**: `calendar.app.created`

**Minimal attendee availability check**: `calendar.freebusy`
