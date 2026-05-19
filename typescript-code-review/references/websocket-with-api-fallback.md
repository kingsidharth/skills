# WebSocket primary, API/polling fallback

**When this applies:** any feature where the server has updates the client should see in real time — chat, presence, notifications, live dashboards, multiplayer state.

The right architecture is **WebSocket as primary**, with **HTTP query/polling as fallback** for connection failures, suspended tabs, mobile background, and initial hydration. Not the other way around.

---

## The architecture

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   useQuery(key)  ──reads──►  cache  ◄──writes── │
│                                  ▲              │
│                                  │              │
│                             invalidate /        │
│                             setQueryData        │
│                                  │              │
│                       ┌──────────┴──────────┐   │
│                       │                     │   │
│                  WebSocket events        Polling│
│                  (primary)               (fallback) │
│                                                 │
└─────────────────────────────────────────────────┘
```

- The **cache** (TanStack Query) is the source of truth for the UI.
- The UI uses `useQuery` and never touches the WebSocket directly.
- The WebSocket pushes events into the cache via `invalidateQueries` or `setQueryData`.
- If the WebSocket disconnects, polling picks up — driven by the same cache.

---

## Rule 1: The UI subscribes to the cache, not the socket

```tsx
// 🚨 component reads from socket directly
function MessageList() {
  const [messages, setMessages] = useState<Message[]>([])
  useEffect(() => {
    const ws = new WebSocket(url)
    ws.onmessage = (e) => setMessages(prev => [...prev, JSON.parse(e.data)])
    return () => ws.close()
  }, [])
  return <ul>{messages.map(...)}</ul>
}
```

Problems: the messages disappear on unmount; no initial fetch; no retry; per-component sockets; no dedup.

```tsx
// ✅ component reads from cache; socket writes to cache
function MessageList() {
  const { data: messages = [] } = useQuery(messagesQuery)
  return <ul>{messages.map(...)}</ul>
}
```

The socket is set up once, app-wide, in a top-level subscription hook (below).

---

## Rule 2: Single app-level subscription that drives the cache

```tsx
function useRealtimeSubscription() {
  const queryClient = useQueryClient()

  useEffect(() => {
    const ws = new WebSocket(WS_URL)

    ws.onmessage = (e) => {
      const event = JSON.parse(e.data) as Event
      handleEvent(queryClient, event)
    }

    return () => ws.close()
  }, [queryClient])
}

// in the root
function App() {
  useRealtimeSubscription()
  return <Routes />
}
```

`handleEvent` is the bridge between server events and cache updates.

---

## Rule 3: Two event shapes — invalidation vs. partial update

**Invalidation event** (server says "X changed, fetch it again"):

```ts
// server: { type: 'invalidate', entity: ['invoices', 'list'] }
function handleEvent(qc: QueryClient, e: InvalidationEvent) {
  qc.invalidateQueries({ queryKey: e.entity })
}
```

Cheap on the wire. The client decides when to fetch. If no observer is mounted, the refetch is deferred until one is.

**Partial update event** (server pushes the new data):

```ts
// server: { type: 'invoice-paid', id: 'inv_123', paidAt: '...' }
function handleEvent(qc: QueryClient, e: InvoicePaidEvent) {
  qc.setQueryData<Invoice>(['invoices', 'detail', e.id], (prev) =>
    prev ? { ...prev, paid: true, paidAt: e.paidAt } : prev,
  )
  // also patch any list views that contain this invoice
  qc.setQueriesData<Invoice[]>(
    { queryKey: ['invoices', 'list'] },
    (list) => list?.map(inv => inv.id === e.id ? { ...inv, paid: true, paidAt: e.paidAt } : inv),
  )
}
```

Pushes the diff. Best for high-frequency, small updates (presence, live counters, single-field changes).

**Default to invalidation events.** They're simpler. Reach for partial updates when the wire cost or perceived latency justifies it.

---

## Rule 4: With WebSocket primary, raise `staleTime`

If the socket pushes updates, polling-on-focus is wasteful — and incorrect, because the socket already has the latest.

```ts
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: Infinity,    // socket invalidates explicitly
      refetchOnWindowFocus: false,
    },
  },
})
```

For specific queries that *should* still refetch on focus (e.g., not WS-backed), override locally.

---

## Rule 5: Polling as a fallback, not as default

When the socket is connected, no polling. When the socket is disconnected, fall back to polling for *critical* data only.

```ts
function useConnectionStatus() {
  const [status, setStatus] = useState<'connected' | 'disconnected'>('disconnected')
  // ...wired up by the realtime subscription
  return status
}

function useNotifications() {
  const status = useConnectionStatus()
  return useQuery({
    queryKey: ['notifications'],
    queryFn: fetchNotifications,
    refetchInterval: status === 'disconnected' ? 30_000 : false,
  })
}
```

`refetchInterval: false` (default) → no polling. `refetchInterval: 30_000` → poll every 30 s. Switching dynamically gives you "polls only when WS is down".

---

## Rule 6: Connection state machine

A WebSocket integration must handle:

- **Connecting** — initial / reconnecting.
- **Connected** — passing events to cache.
- **Backoff** — exponential reconnect on close (start 1 s, cap 30 s).
- **Visibility** — close on `document.hidden` if you don't need background updates; reconnect on visible.
- **Online / offline** — listen to `online`/`offline` events.

A robust subscription:

```ts
function useRealtimeSubscription() {
  const queryClient = useQueryClient()
  const [status, setStatus] = useState<ConnStatus>('disconnected')

  useEffect(() => {
    let ws: WebSocket | null = null
    let attempt = 0
    let reconnectTimer: number | null = null
    let cancelled = false

    const connect = () => {
      if (cancelled) return
      ws = new WebSocket(WS_URL)
      ws.onopen = () => { attempt = 0; setStatus('connected') }
      ws.onmessage = (e) => handleEvent(queryClient, JSON.parse(e.data))
      ws.onclose = () => {
        setStatus('disconnected')
        if (cancelled) return
        const delay = Math.min(1000 * 2 ** attempt, 30_000)
        attempt += 1
        reconnectTimer = window.setTimeout(connect, delay)
      }
    }

    connect()

    return () => {
      cancelled = true
      if (reconnectTimer) clearTimeout(reconnectTimer)
      ws?.close()
    }
  }, [queryClient])

  return status
}
```

For production, prefer a battle-tested library: `partysocket` (Cloudflare), Phoenix Channels, Ably/Pusher SDKs, Supabase Realtime — they handle all this and add presence semantics.

---

## Rule 7: Initial state comes from HTTP, not the socket

The socket pushes *changes*. The initial state comes from a normal `useQuery` against an HTTP endpoint. The socket then keeps it fresh.

```tsx
function ChatRoom({ roomId }: Props) {
  // initial fetch via HTTP — handles loading, errors, retries, SSR
  const { data: messages } = useQuery({
    queryKey: ['rooms', roomId, 'messages'],
    queryFn: () => fetchMessages(roomId),
    staleTime: Infinity,
  })

  // subscription handled at app level; messages array updates via setQueryData
  return <MessageList messages={messages ?? []} />
}
```

Don't try to "join the room" via the socket and wait for a full snapshot. HTTP is simpler, observable, cacheable, and has the entire React Query support around it.

---

## Rule 8: Cache invalidation across many views

The `entity` field in events should match query-key shapes so a single message hits all relevant views. Coordinate with the backend on the shape:

```ts
// server emits
{ type: 'invalidate', entity: ['invoices', 'list'] }
{ type: 'invalidate', entity: ['invoices', 'detail', 'inv_123'] }
{ type: 'invalidate', entity: ['invoices'] }   // catch-all
```

Client-side `invalidateQueries({ queryKey: entity })` does prefix matching by default — `['invoices']` invalidates everything under it.

---

## Rule 9: Don't push everything

Two failure modes:

- **Over-pushing**: server pushes every keystroke / every second / every field. Client becomes flooded; renders thrash. Throttle on the server side, or bundle into batched events.
- **Under-pushing**: server pushes once, client misses it during reconnect. Always include a sequence number; on reconnect, the client requests "events since seq N" via HTTP.

---

## Rule 10: Tear down on unmount, but allow shared connection

The WebSocket lives for the app's lifetime, not per-component. If you have multiple subscription hooks, share the connection — most realtime libraries handle this for you. Don't open one socket per component.

---

## When NOT to reach for WebSockets

- Updates < once per minute → polling is fine.
- Mobile heavy, battery-sensitive → server-sent events or polling may be better.
- Cross-region / serverless → sticky connections are expensive; HTTP-level cache invalidation may be enough.

WebSockets are for *real-time, frequent, bidirectional* updates. Use the right tool.

---

## References

- TkDodo — *Using WebSockets with React Query*
- TanStack Query docs — `setQueriesData`, query invalidation
- See also: `tanstack-query/retain-while-refetching.md`, `tanstack-query/essentials.md`
