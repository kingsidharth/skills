# Vector Search

## Basic vector search

```typescript
const results = await table.search([0.1, 0.2, 0.3])
  .limit(10)
  .toArray();
// Each result includes `_distance` field
```

Auto-detects: if input is a vector → ANN/kNN search. If string → FTS (see fts-hybrid.md).

## Distance metrics

| Metric | Use when | Notes |
|---|---|---|
| `l2` (default) | General purpose | Euclidean distance |
| `cosine` | Unnormalized embeddings | Direction-based, ignores magnitude |
| `dot` | Normalized embeddings | Best performance for normalized vectors |
| `hamming` | Binary vectors | Bit-level comparison, `IVF_FLAT` only |

Match the metric to what your embedding model was trained with. Most modern models use cosine.

```typescript
// Set metric (only when NO vector index exists — otherwise index metric is used)
await table.search(embedding).distanceType("cosine").limit(10).toArray();
```

## Pre-filtering (default)

Filter applied **before** vector search — reduces search space, then finds nearest neighbors in filtered subset.

```typescript
const results = await table.search(embedding)
  .where("category = 'science' AND price > 10")
  .select(["title", "category"])
  .limit(5)
  .toArray();
```

## Post-filtering

Vector search runs first on full dataset, then filter applied to results. May return fewer than `limit` rows.

```typescript
const results = await table.search(embedding)
  .where("label > 1", { prefilter: false })
  .limit(5)
  .toArray();
```

## ANN search (with index)

When a vector index exists, search uses approximate nearest neighbors. Key tuning knobs:

| Parameter | Effect |
|---|---|
| `limit(k)` | Number of results |
| `nprobes(n)` | Partitions to scan (auto-tuned by default) |
| `ef(n)` | HNSW exploration factor — start at 1.5×k, increase for recall |
| `refineFactor(n)` | Rerank n× candidates on full vectors for better accuracy |

```typescript
const results = await table.search(embedding)
  .limit(10)
  .refineFactor(20)  // better recall
  .toArray();
```

### Distance semantics with indexes

| Mode | `_distance` meaning |
|---|---|
| No index / `bypassVectorIndex()` | True distance on full vectors |
| Indexed, no `refineFactor` | Distance on compressed representation |
| Indexed + `refineFactor(≥1)` | Recomputed on full vectors |

## Brute-force / bypass index

```typescript
// Force exhaustive scan (useful for ground-truth recall measurement)
await table.search(embedding).bypassVectorIndex().limit(5).toArray();
```

## Distance range search

```typescript
// Only vectors with distance in [0.1, 0.5)
await table.search(query).distanceRange(0.1, 0.5).toArrow();
```

## Batch search

```typescript
// Multiple query vectors at once — results include `query_index` field
const queries = [embedding1, embedding2, embedding3];
const results = await table.search(queries).limit(5).toArray();
```

## Filtering SQL syntax

LanceDB uses DataFusion SQL expressions:

- Comparison: `>`, `>=`, `<`, `<=`, `=`
- Logical: `AND`, `OR`, `NOT`
- Null checks: `IS NULL`, `IS NOT NULL`
- Set membership: `IN ('a', 'b')`
- Pattern: `LIKE`, `NOT LIKE`
- Type cast: `CAST(col AS float)`
- Regex: `regexp_match(column, pattern)`
- Nested struct access: `` `stats`.`strength` > 3 ``
- Dates: `date_col = date '2021-01-01'`
- Timestamps: `ts_col = timestamp '2021-01-01 00:00:00'`

Backtick-escape column names with special chars, uppercase, or SQL keywords:
`` `CUBE` = 10 AND `nested space`.`inner` < 2 ``

**Best practice**: create scalar indexes on frequently filtered columns for performance.
