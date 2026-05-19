---
name: websockets-fastapi
description: Build WebSocket hubs with FastAPI, Pydantic, SQLite-backed channels, and in-memory fanout for a Mothership ↔ Workers/Frontends star topology over Tailscale. Use whenever the user mentions websockets, WS channels, real-time push, streaming status updates, pub/sub between mothership and remote workers, or connecting a React frontend via TanStack Query to a WS hub.
---

# WebSockets (FastAPI + SQLite fanout)

Star topology. Mothership is the hub. Workers and Frontends connect as clients. JSON messages only — no binary. No auth (Tailscale-gated network).

## Architecture

```
Worker ──┐
Worker ──┼── WS ──► Mothership (FastAPI hub) ──► SQLite (durable) + in-memory fanout
Worker ──┘                 ▲
                           │
Frontend (React) ──────────┘
```

- **Single Mothership process.** In-memory `ConnectionManager` holds live sockets keyed by `connection_id`.
- **SQLite** stores messages + connection registry. Used for durability, replay on reconnect, and cross-restart history. Not for fanout — fanout is pure in-process `asyncio.Queue`.
- **Channels** are logical topics (e.g. `claims`, `status`, `worker:<hostname>`). Clients subscribe on connect.

## When to use this skill

Any task involving: FastAPI WebSocket endpoints, Pydantic message schemas, channel/topic pub-sub within a single process, SQLite-backed message log, reconnect/replay, or wiring a React client via TanStack Query + a WS subscription hook.

## Quick start

```bash
uv add fastapi uvicorn[standard] pydantic aiosqlite
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Minimal server — see [references/server.md](references/server.md) for the full hub.

## Message contract

Every frame is a JSON envelope. Enforced with Pydantic.

```python
class Envelope(BaseModel):
    id: str              # ULID, client-generated
    type: Literal["event", "command", "ack", "error"]
    channel: str         # e.g. "claims", "status", "worker:hetzner-01"
    ts: datetime
    payload: dict        # validated per channel
```

Full schema, channel registry, and validation patterns → [references/protocol.md](references/protocol.md).

## Routing files

| File | When to read |
|---|---|
| [references/server.md](references/server.md) | Building the FastAPI hub, `ConnectionManager`, endpoints |
| [references/protocol.md](references/protocol.md) | Envelope schema, channel naming, Pydantic validators |
| [references/sqlite-fanout.md](references/sqlite-fanout.md) | Persisting messages, replay-on-reconnect, retention |
| [references/client-python.md](references/client-python.md) | Worker client (reconnect loop, heartbeat, push) |
| [references/client-react.md](references/client-react.md) | React hook + TanStack Query cache sync |
| [references/connection-id.md](references/connection-id.md) | Upgrading from REST: handshake → connection_id |
| [references/operations.md](references/operations.md) | Tailscale, heartbeats, backpressure, errors |

## Core rules

- **JSON only.** Reject binary frames at the endpoint. Files go through B2 URLs inside the payload.
- **Every message has an `id`.** Workers may resend on reconnect; the hub dedupes by `id`.
- **Channels are flat strings**, colon-namespaced for scoping (`worker:<hostname>`, `claim:<claim_id>`).
- **Pydantic validates every inbound frame.** Invalid → send `error` envelope, keep connection open.
- **Heartbeat ping every 20s** from server; close after 2 missed pongs. See operations.md.
- **No auth inside the app** — Tailscale ACLs are the boundary. Do not add token checks unless explicitly asked.

## Upgrading from REST

Client POSTs `/register` with hostname + role → receives `connection_id` → opens `ws://host/ws?connection_id=...`. The connection_id is the stable identity for the session; hostname is the human label. See [references/connection-id.md](references/connection-id.md).
