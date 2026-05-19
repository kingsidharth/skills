# Versioning & Consistency

## Versioning

Every mutation (add, update, delete, schema change, optimize) creates a new table version. Versions are cheap — they share unchanged data fragments.

```typescript
// Check current version
const v = await table.version();

// List all versions
const versions = await table.listVersions();

// Time travel — checkout specific version (read-only snapshot)
await table.checkout(2);
const oldData = await table.search(query).limit(10).toArray();

// Return to latest
await table.checkoutLatest();

// Restore a version (creates a new version with that snapshot's data)
await table.restore(2);
```

### Version sequence

1. `v1`: `createTable`
2. `v2`: `update`
3. `v3`: `add`
4. `v4`: `restore` (from earlier version)
5. `v5`: `delete`

Read-only operations (`checkout`, `listVersions`, `version`) don't create versions.

System operations (`optimize()`, index builds) also increment versions.

### Cleanup

`optimize()` prunes old versions beyond retention window (7 days default). Configurable:

```python
# Python — set shorter retention
from datetime import timedelta
table.optimize(cleanup_older_than=timedelta(days=1))
```

## Consistency

Control how often reads detect writes from other processes via `read_consistency_interval` on connection:

| Setting | Behavior |
|---|---|
| Unset (default) | No auto cross-process refresh |
| `0` seconds | Check on every read (strongest freshness) |
| Non-zero interval | Check after interval elapses (eventual) |

```typescript
// Strong consistency
const db = await lancedb.connect("data/lancedb", {
  readConsistencyInterval: 0  // check every read
});

// Eventual
const db = await lancedb.connect("data/lancedb", {
  readConsistencyInterval: 5  // seconds
});

// Manual refresh
await table.checkoutLatest();
```

Enterprise: consistency is deployment-configured, not an SDK setting.

## Optimize / compaction

```typescript
await table.optimize();
```

Performs three operations:
1. **Compaction**: merge small fragments into larger ones
2. **Cleanup**: remove files from old versions (beyond retention)
3. **Index update**: add newly-ingested data to existing indexes

Run periodically after bulk ingestion. Compaction may temporarily increase disk usage before cleanup reclaims space.

## Bad vector handling (Python only)

Options: `error` (default), `drop`, `fill` (with fill_value), `null` (if column nullable).
