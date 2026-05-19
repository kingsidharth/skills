# React client (Frontend)

TanStack Query owns the cache. WebSocket pushes invalidate or directly update that cache. Don't try to replace Query with a WS store — they compose.

## Approach

- One long-lived WS connection per tab, managed by a context provider.
- Incoming messages mutate the Query cache via `queryClient.setQueryData` or trigger `invalidateQueries`.
- Outgoing messages go through a thin `send()` on the context.

## Provider

```tsx
import { createContext, useContext, useEffect, useRef, useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { ulid } from "ulid";

type Envelope = {
  id: string;
  type: "event" | "command" | "ack" | "error";
  channel: string;
  ts: string;
  payload: Record<string, unknown>;
};

const WSContext = createContext<{
  send: (env: Omit<Envelope, "id" | "ts">) => void;
  status: "connecting" | "open" | "closed";
} | null>(null);

export function WSProvider({
  connectionId,
  url,
  children,
}: {
  connectionId: string;
  url: string;
  children: React.ReactNode;
}) {
  const qc = useQueryClient();
  const wsRef = useRef<WebSocket | null>(null);
  const lastIdRef = useRef<string | null>(null);
  const [status, setStatus] = useState<"connecting" | "open" | "closed">("connecting");

  useEffect(() => {
    let cancelled = false;
    let retry = 0;

    const connect = () => {
      const since = lastIdRef.current ? `&since=${lastIdRef.current}` : "";
      const ws = new WebSocket(`${url}?connection_id=${connectionId}${since}`);
      wsRef.current = ws;
      setStatus("connecting");

      ws.onopen = () => {
        retry = 0;
        setStatus("open");
        ws.send(JSON.stringify({
          id: ulid(),
          type: "command",
          channel: "sub",
          ts: new Date().toISOString(),
          payload: { channels: ["claims", "status", "submissions"] },
        }));
      };

      ws.onmessage = (e) => {
        const env: Envelope = JSON.parse(e.data);
        lastIdRef.current = env.id;
        route(env, qc);
      };

      ws.onclose = () => {
        setStatus("closed");
        if (cancelled) return;
        const delay = Math.min(1000 * 2 ** retry++, 15000);
        setTimeout(connect, delay);
      };
    };

    connect();
    return () => {
      cancelled = true;
      wsRef.current?.close();
    };
  }, [connectionId, url, qc]);

  const send = (env: Omit<Envelope, "id" | "ts">) => {
    wsRef.current?.send(JSON.stringify({
      ...env,
      id: ulid(),
      ts: new Date().toISOString(),
    }));
  };

  return <WSContext.Provider value={{ send, status }}>{children}</WSContext.Provider>;
}

export const useWS = () => {
  const ctx = useContext(WSContext);
  if (!ctx) throw new Error("useWS outside WSProvider");
  return ctx;
};
```

## Cache sync

One place — `route()` — maps channels to cache updates. Keeps component code clean.

```ts
function route(env: Envelope, qc: QueryClient) {
  switch (env.channel) {
    case "status": {
      const s = env.payload as { worker_id: string };
      qc.setQueryData(["worker", s.worker_id], s);
      return;
    }
    case "claims":
      qc.invalidateQueries({ queryKey: ["claims"] });
      return;
    case "submissions":
      qc.invalidateQueries({ queryKey: ["submissions"] });
      return;
  }
  if (env.channel.startsWith("claim:")) {
    const claimId = env.channel.slice("claim:".length);
    qc.invalidateQueries({ queryKey: ["claim", claimId] });
  }
}
```

## Using it in components

Components read via TanStack Query as normal — they don't know WS exists.

```tsx
function WorkerCard({ id }: { id: string }) {
  const { data } = useQuery({
    queryKey: ["worker", id],
    queryFn: () => fetch(`/api/workers/${id}`).then(r => r.json()),
  });
  return <div>{data?.state} — {data?.progress}</div>;
}
```

When a `status` event arrives for worker `id`, `setQueryData` updates the cache → the component re-renders. No WS code in the component.

## Sending commands

```tsx
function CancelButton({ workerHost }: { workerHost: string }) {
  const { send } = useWS();
  return (
    <button onClick={() => send({
      type: "command",
      channel: `worker:${workerHost}`,
      payload: { action: "cancel" },
    })}>
      Cancel
    </button>
  );
}
```

## Notes

- **One WS per tab.** Don't open a socket per hook.
- **Reconnect with exponential backoff** (capped at 15s above).
- **`lastIdRef` enables replay** — on reconnect, the URL includes `?since=<id>` and the server replays missed frames.
- **Browser handles ping/pong** transparently at the protocol level.
