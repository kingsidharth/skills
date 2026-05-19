# Cache-First & Perceived Performance

## Pre-Warmed Startup

Desktop apps should feel instant on repeat launches. Store user data locally and render from cache before network responses arrive.

**Architecture:**
1. Hot cache in memory (TanStack Query) — sub-100ms reads
2. Durable cache in IndexedDB — survives restarts
3. Background refresh on window focus and reconnect
4. Prefetch on intent (hover, viewport proximity)

```tsx
// Persist TanStack Query client to IndexedDB
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { get, set, del } from 'idb-keyval';

function createIDBPersister(key: IDBValidKey = 'reactQuery') {
  return {
    persistClient: async (client: PersistedClient) => set(key, client),
    restoreClient: async () => get<PersistedClient>(key),
    removeClient: async () => del(key),
  } satisfies Persister;
}

// Configure QueryClient with long GC time
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { gcTime: 1000 * 60 * 60 * 24 }, // 24h
  },
});
```

**Result:** First launch with empty cache is slow. Every subsequent launch: cached data renders instantly, network refresh happens in background. Spinner count in core flows → zero.

Linear uses this pattern extensively (Linear Sync Engine). Their entire app feels instant because every piece of data has a local shadow.

## Optimistic Updates

Show the result before the server confirms:

```tsx
import { useOptimistic } from 'react';

function FavoriteButton({ itemId, initialState }: Props) {
  const [isFav, setIsFav] = useState(initialState);
  const [optimisticFav, setOptimisticFav] = useOptimistic(isFav);

  const toggle = async () => {
    setOptimisticFav(!optimisticFav);
    try {
      await api.toggleFavorite(itemId, !isFav);
      setIsFav(!isFav);
    } catch {
      // Reverts automatically via useOptimistic
    }
  };

  return <button onClick={toggle}>{optimisticFav ? '★' : '☆'}</button>;
}
```

TanStack Query and SWR both have built-in optimistic update support.

## Prefetching

Warm the cache before the user navigates:

```tsx
// Prefetch on hover
<Link
  to="/dashboard"
  onMouseEnter={() => queryClient.prefetchQuery({ queryKey: ['dashboard'] })}
>
  Dashboard
</Link>

// Prefetch when entering viewport
const ref = useIntersectionObserver(() => {
  queryClient.prefetchQuery({ queryKey: ['feed', page + 1] });
});
```

## Skeleton-First Rendering

When cache is cold, render content-shaped skeletons instead of spinners. Match exact dimensions of final content to prevent layout shift (important for accessibility and perceived speed).

## Idle-Time Work

```ts
requestIdleCallback(() => {
  // Prune expired cache entries
  // Pre-compute derived data
  // Sync pending writes
});
```

## Success Criteria

- Spinner count in core flows: **zero**
- Time to content on navigation: **0–100ms** (from cache)
- Cache hit rate on repeat views: **>90%**
- Optimistic write error rate: **<1%**

> **Ref:** [Never Load](https://johnnyle.io/read/never-load) · [Linear Sync Engine](https://linear.app/blog/scaling-the-linear-sync-engine)
