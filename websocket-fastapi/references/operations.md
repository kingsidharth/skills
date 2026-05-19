# Operations

## Tailscale

- Bind Mothership to the tailnet interface or `0.0.0.0` on a Tailscale-only host.
- Clients connect via MagicDNS name: `ws://mothership.tailnet-name.ts.net:8000/ws`.
- No TLS needed inside the tailnet — WireGuard handles transport security. If you want `wss://`, terminate with Caddy on the Mothership box and set `tls internal` or use a Tailscale cert.
- **Do not expose port 8000 on the public internet.** Firewall rule: allow only from `100.64.0.0/10` (Tailscale CGNAT range).

## Heartbeats

The `websockets` library (Python) and browsers handle WS-protocol pings automatically — use them. For Python server:

```python
# uvicorn flags
uvicorn app.main:app --ws-ping-interval 20 --ws-ping-timeout 20
```

Connection is closed if the peer misses the pong window. No app-level heartbeat needed.

If you want app-visible liveness (e.g. "last seen" in UI), emit an `event` on a `heartbeat:<hostname>` channel from the worker every 30s. Separate concern from transport keepalive.

## Backpressure

`ConnectionManager.publish` uses a bounded `asyncio.Queue(maxsize=1000)` per connection. When full:

- **Default policy:** drop oldest, keep newest. Correct for status ticks (latest state wins).
- **For commands:** never drop — bubble up as error, log, alert. Commands are rare enough that a full outbox means the client is effectively dead.

Split queues if mixing: one for events (lossy), one for commands (blocking).

## Graceful shutdown

```python
@app.on_event("shutdown")
async def on_shutdown():
    for conn in list(hub._conns.values()):
        await conn.ws.close(code=1012, reason="server restart")
    await _db.close()
```

Code 1012 ("service restart") signals clients to reconnect immediately.

## Error handling rules

| Situation | Action |
|---|---|
| Malformed JSON | send `error` frame, keep connection |
| Unknown channel | send `error` frame, keep connection |
| Payload validation fails | send `error` frame, keep connection |
| Binary frame received | close with 1003 |
| Unknown `connection_id` | close with 4401 |
| Handler raises | send `error` frame, log with traceback, keep connection |
| Outbox full + command | close with 1011 (server error) |

**Never crash the session on a bad message.** Validation failures are normal; only transport-level issues close the socket.

## Logging

Every envelope in/out gets one structured log line:

```python
logger.info("ws_msg", extra={
    "direction": "in" | "out",
    "connection_id": conn.connection_id,
    "msg_id": env.id,
    "channel": env.channel,
    "type": env.type,
})
```

Payloads are in SQLite — don't duplicate in logs.

## Scale ceiling

Single process, asyncio, SQLite. Rough limits on a modest Hetzner box:

- ~5,000 concurrent connections (file descriptor + memory bound).
- ~1,000 messages/sec sustained write to SQLite in WAL mode.
- Fanout is O(subscribers) per message, all in-process.

When you hit these: swap SQLite → Postgres, swap in-memory fanout → Redis pub/sub. The protocol and client code do not change.
