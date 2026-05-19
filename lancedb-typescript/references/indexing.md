# Indexing

## When to index

- **< ~100k vectors**: brute-force kNN is fast enough, no index needed
- **> 100k vectors**: create a vector index for sub-linear search
- **FTS**: always requires an FTS index before keyword search
- **Filtered queries**: scalar indexes on filter columns significantly improve performance

## Vector index types

| Index | Best for | Compression |
|---|---|---|
| `IVF_HNSW_FLAT` | Highest recall, no quantization | ~raw size + graph overhead |
| `IVF_HNSW_SQ` | Best recall/latency trade-off | ~1/4 raw size |
| `IVF_RQ` | Maximum compression | ~1/32 raw size |
| `IVF_PQ` | Higher accuracy at dim ≤ 256 | 1/64 to 1/16 raw size |

If queries frequently use metadata filters (`where(...)`), prefer `IVF_RQ` or `IVF_PQ` — HNSW-backed indexes show higher latency variance under filtered search.

### Create vector index

```typescript
await table.createIndex("vector", {
  config: lancedb.Index.ivfPq({
    distanceType: "cosine",
    numPartitions: 256,    // start: num_rows / 4096
    numSubVectors: 192,    // start: dimension / 8
  })
});

// Or HNSW
await table.createIndex("vector", {
  config: lancedb.Index.ivfHnswSq({
    distanceType: "cosine",
    numPartitions: 1,      // start: num_rows / 1_048_576
    efConstruction: 150,
  })
});
```

Index creation is async — use `waitTimeout` or poll `listIndices()`.

### Tuning starting points

- **HNSW-backed** (`IVF_HNSW_*`): `numPartitions = numRows / 1_048_576`, `efConstruction = 150`
- **IVF_RQ / IVF_PQ**: `numPartitions = numRows / 4096`
- **IVF_PQ**: `numSubVectors = dimension / 8`

### Search configuration

| Parameter | Effect |
|---|---|
| `nprobes(n)` | Partitions to scan (auto-tuned by default) |
| `ef(n)` | HNSW exploration factor — start 1.5×k, up to 10×k for recall |
| `refineFactor(n)` | Rerank on full vectors for quantized indexes |

### Custom index names

Default name: `{column}_idx`. Override with `name` option:

```typescript
await table.createIndex("vector", {
  name: "my_custom_idx",
  config: lancedb.Index.ivfPq({ distanceType: "cosine" })
});
```

## FTS index

```typescript
await table.createIndex("text", { config: lancedb.Index.fts() });
```

See [fts-hybrid.md](fts-hybrid.md) for tokenizer options and query patterns.

## Scalar index

Accelerates filter-only queries and merge-insert joins.

```typescript
// BTree (default for scalar columns)
await table.createIndex("category");

// For array columns
await table.createIndex("tags", { config: lancedb.Index.labelList() });
```

Types: `BTREE` (ordered), `BITMAP` (low-cardinality), `LABEL_LIST` (arrays).

## Binary vector index

Only `IVF_FLAT` + `hamming` supported for binary vectors:

```typescript
await table.createIndex("vector", {
  config: lancedb.Index.ivfFlat({ distanceType: "hamming" })
});
```

Dimension must be multiple of 8. Vectors stored as packed `Uint8Array`.

## Reindexing

New data after index creation is searched via brute-force fallback (always returns correct results, but slower). Run `optimize()` to incrementally reindex:

```typescript
await table.optimize();
```

This performs compaction + cleanup + index update in one call. Default cleanup retention: 7 days.

Enterprise auto-indexes in background. Use `fast_search: true` to skip unindexed rows for speed.

### Check index status

```typescript
const indices = await table.listIndices();
const stats = await table.indexStats("vector_idx");
// stats.numIndexedRows, stats.numUnindexedRows
```
