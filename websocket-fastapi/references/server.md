# Server

FastAPI hub. Single process. In-memory `ConnectionManager` + SQLite for durability.

## Layout

```
app/
├── main.py            # FastAPI app + routes
├── hub.py             # ConnectionManager, fanout
├── protocol.py        # Envelope, channel schemas (see protocol.md)
├── store.py           # SQLite helpers (see sqlite-fanout.md)
└── registry.py        # /register endpoint + connection_id (see connection-id.md)
```

## ConnectionManager

Owns all live sockets. Fanout is `asyncio.Queue` per connection — handlers never await sockets directly.

```python
import asyncio
from collections import defaultdict
from fastapi import WebSocket

class Connection:
    def __init__(self, ws: WebSocket, connection_id: str, role: str):
        self.ws = ws
        self.connection_id = connection_id
        self.role = role  # "worker" | "frontend"
        self.subscriptions: set[str] = set()
        self.outbox: asyncio.Queue[str] = asyncio.Queue(maxsize=1000)

class ConnectionManager:
    def __init__(self):
        self._conns: dict[str, Connection] = {}
        self._by_channel: dict[str, set[str]] = defaultdict(set)

    async def connect(self, ws: WebSocket, connection_id: str, role: str) -> Connection:
        await ws.accept()
        conn = Connection(ws, connection_id, role)
        self._conns[connection_id] = conn
        return conn

    def disconnect(self, connection_id: str):
        conn = self._conns.pop(connection_id, None)
        if not conn:
            return
        for ch in conn.subscriptions:
            self._by_channel[ch].discard(connection_id)

    def subscribe(self, connection_id: str, channel: str):
        self._conns[connection_id].subscriptions.add(channel)
        self._by_channel[channel].add(connection_id)

    async def publish(self, channel: str, frame: str):
        # Enqueue to every subscriber's outbox. Non-blocking.
        for cid in self._by_channel.get(channel, set()):
            conn = self._conns.get(cid)
            if not conn:
                continue
            try:
                conn.outbox.put_nowait(frame)
            except asyncio.QueueFull:
                # Backpressure: drop oldest, log, keep connection
                _ = conn.outbox.get_nowait()
                conn.outbox.put_nowait(frame)

hub = ConnectionManager()
```

## Endpoint

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query
from .protocol import Envelope, validate_payload
from .store import persist, load_since
from .hub import hub

app = FastAPI()

@app.websocket("/ws")
async def ws_endpoint(
    ws: WebSocket,
    connection_id: str = Query(...),
    since: str | None = Query(None),  # replay cursor: last seen envelope id
):
    role = await registry.lookup(connection_id)  # from /register
    if not role:
        await ws.close(code=4401, reason="unknown connection_id")
        return

    conn = await hub.connect(ws, connection_id, role)
    sender = asyncio.create_task(_sender_loop(conn))
    try:
        if since:
            for row in await load_since(since, conn.subscriptions):
                await conn.outbox.put(row)
        await _receiver_loop(conn)
    except WebSocketDisconnect:
        pass
    finally:
        sender.cancel()
        hub.disconnect(connection_id)

async def _sender_loop(conn: Connection):
    while True:
        frame = await conn.outbox.get()
        await conn.ws.send_text(frame)

async def _receiver_loop(conn: Connection):
    while True:
        raw = await conn.ws.receive_text()
        try:
            env = Envelope.model_validate_json(raw)
            validate_payload(env)
        except Exception as e:
            await conn.ws.send_text(_error_frame(None, "validation_failed", str(e)))
            continue

        await persist(env, conn.connection_id)

        if env.type == "command" and env.channel.startswith("worker:"):
            # Mothership → this worker; already routed by channel
            pass
        # Fan out to subscribers
        await hub.publish(env.channel, raw)
```

## Subscribe / unsubscribe

Two control channels, not data channels:

- `sub`: payload `{"channels": ["claims", "status"]}` → hub adds subscriptions, acks.
- `unsub`: same shape.

Handle these before per-channel validation:

```python
if env.channel in ("sub", "unsub"):
    for ch in env.payload["channels"]:
        if env.channel == "sub":
            hub.subscribe(conn.connection_id, ch)
        else:
            hub.unsubscribe(conn.connection_id, ch)
    await conn.ws.send_text(_ack(env.id, "ok"))
    continue
```

## Reject binary

```python
# At endpoint setup
@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket, ...):
    await ws.accept()
    # receive_text() below will raise on binary frames from the client.
    # For belt-and-braces, inspect frame type via receive():
    msg = await ws.receive()
    if "bytes" in msg and msg["bytes"] is not None:
        await ws.close(code=1003, reason="binary frames not accepted")
        return
```

Prefer `receive_text()` throughout — it raises cleanly on binary.

## Heartbeat

Server sends ping envelopes on a timer; see [operations.md](operations.md).
