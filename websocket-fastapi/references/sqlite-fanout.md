# SQLite persistence + replay

SQLite is the durable log. Fanout stays in-process. SQLite only gets hit on write and on replay.

## Schema

```sql
CREATE TABLE IF NOT EXISTS messages (
    id          TEXT PRIMARY KEY,         -- envelope.id (ULID, sortable)
    type        TEXT NOT NULL,
    channel     TEXT NOT NULL,
    ts          TEXT NOT NULL,            -- ISO8601
    sender_id   TEXT NOT NULL,            -- connection_id
    payload     TEXT NOT NULL,            -- raw JSON
    created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_channel_id ON messages(channel, id);
CREATE INDEX IF NOT EXISTS idx_created_at ON messages(created_at);

CREATE TABLE IF NOT EXISTS connections (
    connection_id TEXT PRIMARY KEY,
    hostname      TEXT NOT NULL,
    role          TEXT NOT NULL,          -- "worker" | "frontend"
    registered_at TEXT NOT NULL,
    last_seen_at  TEXT
);
```

ULIDs are lexicographically sortable by time → `WHERE id > ?` gives you "everything after this cursor" without a separate timestamp column.

## Write path

`aiosqlite` for async. One shared connection; SQLite serializes writes internally.

```python
import aiosqlite
import json

_db: aiosqlite.Connection | None = None

async def init(path: str = "hub.db"):
    global _db
    _db = await aiosqlite.connect(path)
    await _db.execute("PRAGMA journal_mode=WAL")
    await _db.execute("PRAGMA synchronous=NORMAL")
    # apply schema above
    await _db.commit()

async def persist(env: Envelope, sender_id: str):
    await _db.execute(
        "INSERT OR IGNORE INTO messages(id,type,channel,ts,sender_id,payload) VALUES (?,?,?,?,?,?)",
        (env.id, env.type, env.channel, env.ts.isoformat(), sender_id, json.dumps(env.payload)),
    )
    await _db.commit()
```

`INSERT OR IGNORE` handles duplicate `id` from client retries.

## Replay on reconnect

Client passes `?since=<last_id>`. Server returns every message on subscribed channels with `id > since`.

```python
async def load_since(cursor: str, channels: set[str]) -> list[str]:
    if not channels:
        return []
    placeholders = ",".join("?" * len(channels))
    rows = await _db.execute_fetchall(
        f"SELECT id,type,channel,ts,payload FROM messages "
        f"WHERE id > ? AND channel IN ({placeholders}) "
        f"ORDER BY id ASC LIMIT 1000",
        (cursor, *channels),
    )
    return [_row_to_frame(r) for r in rows]
```

Cap replay at 1000 messages. If the client is further behind, it gets the most recent 1000 and a warning envelope — they should reset state rather than stitch.

## Retention

Single-process, single-box. Prune by age:

```python
async def prune(older_than_days: int = 7):
    await _db.execute(
        "DELETE FROM messages WHERE created_at < datetime('now', ?)",
        (f"-{older_than_days} days",),
    )
    await _db.commit()
```

Run as a scheduled task (`asyncio.create_task` loop on startup).

## What NOT to use SQLite for

- **Live fanout.** Every message already goes to in-memory `ConnectionManager.publish()`. Don't poll SQLite for new rows.
- **Cross-process pub/sub.** This setup is single-process by design. If you outgrow it, swap in Redis; the protocol stays the same.
