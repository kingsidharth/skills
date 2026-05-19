# Python client (Worker)

Worker side. Runs on remote Hetzner/RunPod boxes, connects to Mothership over Tailscale.

Use the `websockets` library (not `aiohttp` — leaner, purpose-built).

```bash
uv add websockets pydantic httpx ulid-py
```

## Skeleton

```python
import asyncio
import json
import socket
import httpx
import websockets
from ulid import ULID

MOTHERSHIP = "http://mothership.tailnet.ts.net:8000"
WS_URL = "ws://mothership.tailnet.ts.net:8000/ws"

class WorkerClient:
    def __init__(self, role: str = "worker"):
        self.hostname = socket.gethostname()
        self.role = role
        self.connection_id: str | None = None
        self.last_seen_id: str | None = None
        self.outbox: asyncio.Queue[dict] = asyncio.Queue()
        self._ws: websockets.WebSocketClientProtocol | None = None

    async def register(self):
        async with httpx.AsyncClient() as c:
            r = await c.post(
                f"{MOTHERSHIP}/register",
                json={"hostname": self.hostname, "role": self.role},
            )
            r.raise_for_status()
            self.connection_id = r.json()["connection_id"]

    async def run(self):
        await self.register()
        while True:
            try:
                await self._session()
            except Exception as e:
                print(f"ws error: {e}; reconnecting in 2s")
                await asyncio.sleep(2)

    async def _session(self):
        url = f"{WS_URL}?connection_id={self.connection_id}"
        if self.last_seen_id:
            url += f"&since={self.last_seen_id}"
        async with websockets.connect(url, ping_interval=20, ping_timeout=20) as ws:
            self._ws = ws
            await self._subscribe(["claims", f"worker:{self.hostname}"])
            await asyncio.gather(self._send_loop(), self._recv_loop())

    async def _subscribe(self, channels: list[str]):
        await self._ws.send(json.dumps({
            "id": str(ULID()),
            "type": "command",
            "channel": "sub",
            "ts": _now_iso(),
            "payload": {"channels": channels},
        }))

    async def _send_loop(self):
        while True:
            msg = await self.outbox.get()
            await self._ws.send(json.dumps(msg))

    async def _recv_loop(self):
        async for raw in self._ws:
            env = json.loads(raw)
            self.last_seen_id = env["id"]
            await self._handle(env)

    async def _handle(self, env: dict):
        # Dispatch on channel + type
        ...

    # Public API for business logic
    async def push_status(self, state: str, progress: float = 0.0):
        await self.outbox.put({
            "id": str(ULID()),
            "type": "event",
            "channel": "status",
            "ts": _now_iso(),
            "payload": {
                "worker_id": self.hostname,
                "state": state,
                "progress": progress,
            },
        })
```

## Key behaviors

- **Reconnect loop** with backoff around `_session()`. Don't catch inside — let the session die, let `run()` restart.
- **`last_seen_id`** persists the replay cursor in memory. For stronger guarantees, write to a local SQLite/file.
- **`ping_interval=20, ping_timeout=20`** matches server heartbeat. `websockets` handles protocol-level pings automatically.
- **Outbox queue** decouples business logic from socket availability. Producing work never blocks on the network.
- **`register()` runs once per worker boot.** The `connection_id` is stable across reconnects until the worker restarts.

## Command handling

```python
async def _handle(self, env: dict):
    if env["type"] == "command" and env["channel"].startswith("worker:"):
        if env["payload"].get("action") == "new_claim":
            await self._ack(env["id"], "ok")
            asyncio.create_task(self._process_claim(env["payload"]))

async def _ack(self, ref_id: str, result: str, detail: str = ""):
    await self.outbox.put({
        "id": str(ULID()),
        "type": "ack",
        "channel": "acks",
        "ts": _now_iso(),
        "payload": {"ref_id": ref_id, "result": result, "detail": detail},
    })
```
