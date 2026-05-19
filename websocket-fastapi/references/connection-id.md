# Connection ID (REST → WS handshake)

The WebSocket endpoint requires a `connection_id`. Clients obtain it via a one-shot POST, then hold it for the lifetime of the worker/tab.

## Why

- Lets the server map sockets → roles/hostnames without re-negotiating on every reconnect.
- Gives clients a stable identity for `?since=` replay cursors.
- Keeps the WS path dumb — just verifies the id exists.

## Endpoint

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from ulid import ULID
import aiosqlite

app = FastAPI()

class RegisterReq(BaseModel):
    hostname: str
    role: Literal["worker", "frontend"]

class RegisterResp(BaseModel):
    connection_id: str

@app.post("/register", response_model=RegisterResp)
async def register(req: RegisterReq):
    cid = str(ULID())
    await _db.execute(
        "INSERT INTO connections(connection_id, hostname, role, registered_at) "
        "VALUES (?, ?, ?, datetime('now'))",
        (cid, req.hostname, req.role),
    )
    await _db.commit()
    return RegisterResp(connection_id=cid)

async def lookup(connection_id: str) -> str | None:
    row = await (await _db.execute(
        "SELECT role FROM connections WHERE connection_id = ?", (connection_id,)
    )).fetchone()
    return row[0] if row else None
```

## Client side

**Worker:** call once on boot, hold in memory. On process restart, register again — the old `connection_id` is effectively abandoned (prune later).

**Frontend:** call on app mount, store in `sessionStorage` so refreshes reuse the id within a tab session.

```ts
async function getConnectionId(): Promise<string> {
  const cached = sessionStorage.getItem("connection_id");
  if (cached) return cached;
  const r = await fetch("/register", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ hostname: navigator.userAgent, role: "frontend" }),
  });
  const { connection_id } = await r.json();
  sessionStorage.setItem("connection_id", connection_id);
  return connection_id;
}
```

## Rules

- `connection_id` is an opaque ULID. Clients never parse it.
- `hostname` is informational only — humans use it to identify workers. Not an identity key.
- One `connection_id` → one active socket at a time. If a second socket opens with the same id, close the older one (most recent wins).
- Unknown `connection_id` on the WS endpoint → close with code 4401.
