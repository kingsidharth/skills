# Protocol

All WebSocket frames are JSON envelopes. One schema, discriminated by `type` + `channel`.

## Envelope

```python
from datetime import datetime
from typing import Literal
from pydantic import BaseModel, Field
from ulid import ULID

class Envelope(BaseModel):
    id: str = Field(default_factory=lambda: str(ULID()))
    type: Literal["event", "command", "ack", "error"]
    channel: str
    ts: datetime = Field(default_factory=datetime.utcnow)
    payload: dict

    model_config = {"extra": "forbid"}
```

## Types

| `type` | Direction | Meaning |
|---|---|---|
| `event` | any → any | Fire-and-forget state update (status tick, log line) |
| `command` | Mothership → Worker | Request action (new claim, cancel, fetch file) |
| `ack` | receiver → sender | Confirms receipt + processing of a prior `id` |
| `error` | any → any | Validation failure or handler exception; references failing `id` |

`ack` and `error` payloads always include `ref_id` pointing at the original `id`.

## Channels

Flat strings. Colon = namespace scope. Subscriber matches by exact string; no wildcards.

| Channel | Publishers | Subscribers | Payload schema |
|---|---|---|---|
| `claims` | Mothership | Workers, Frontends | `ClaimAssignment` |
| `status` | Workers | Mothership, Frontends | `WorkerStatus` |
| `submissions` | Workers | Mothership, Frontends | `Submission` |
| `worker:<hostname>` | Mothership | one Worker | `WorkerCommand` |
| `claim:<claim_id>` | any | subscribers to that claim | `ClaimEvent` |

## Per-channel payload validation

Register payload models against channels. The hub validates after envelope parse.

```python
from pydantic import BaseModel, HttpUrl

class ClaimAssignment(BaseModel):
    claim_id: str
    required_files: list[HttpUrl]
    metadata: dict

class WorkerStatus(BaseModel):
    worker_id: str
    state: Literal["idle", "working", "error"]
    current_claim_id: str | None = None
    progress: float = 0.0

CHANNEL_SCHEMAS: dict[str, type[BaseModel]] = {
    "claims": ClaimAssignment,
    "status": WorkerStatus,
    # ...
}

def validate_payload(env: Envelope) -> BaseModel:
    # Handle namespaced channels
    base = env.channel.split(":", 1)[0]
    schema = CHANNEL_SCHEMAS.get(env.channel) or CHANNEL_SCHEMAS.get(base)
    if not schema:
        raise ValueError(f"unknown channel: {env.channel}")
    return schema.model_validate(env.payload)
```

## Error envelope

```json
{
  "id": "01HXYZ...",
  "type": "error",
  "channel": "claims",
  "ts": "2026-04-18T12:00:00Z",
  "payload": {
    "ref_id": "01HABC...",
    "code": "validation_failed",
    "detail": "required_files: not a valid URL"
  }
}
```

Codes: `validation_failed`, `unknown_channel`, `handler_error`, `rate_limited`, `duplicate`.

## Ack policy

- Commands **must** be acked. Hub retries unacked commands on reconnect.
- Events are **not** acked by default. Opt-in per channel if ordering matters.
- Ack payload: `{"ref_id": "<original id>", "result": "ok" | "rejected", "detail": "..."}`.

## Dedup

Hub keeps a bounded LRU of last N message `id`s per connection. Duplicate `id` on the same connection → silently drop, send `ack` with `result: "duplicate"`.
