---
name: lancedb-typescript
description: Embedded vector database using LanceDB with TypeScript/JavaScript SDK. Use when building vector search, semantic search, RAG pipelines, hybrid search, full-text search, or multimodal data storage with LanceDB. Trigger on mentions of LanceDB, Lance format, vector similarity search with embedded DB, or vectordb that runs in-process. Also trigger when user needs an embedded alternative to Pinecone/Weaviate/Qdrant.
---

# LanceDB (TypeScript)

Multimodal lakehouse — embedded vector DB built on the Lance columnar format. Runs in-process (like SQLite), supports local disk, S3, GCS, Azure. Apache Arrow under the hood.

## Quick start

```typescript
import * as lancedb from "@lancedb/lancedb";

const db = await lancedb.connect("data/lancedb");

const table = await db.createTable("vectors", [
  { id: 1, text: "hello", vector: [0.1, 0.2, 0.3] },
  { id: 2, text: "world", vector: [0.4, 0.5, 0.6] },
]);

const results = await table.search([0.1, 0.2, 0.3]).limit(5).toArray();
```

**Install**: `npm install @lancedb/lancedb` (native module, uses `napi-rs`)

## Reference routing

Read the relevant reference file before writing code:

| Topic | File | When to read |
|---|---|---|
| Tables & ingestion | [tables.md](references/tables.md) | Creating tables, adding data, schema, updates, deletes, merge-insert/upsert |
| Vector search | [search.md](references/search.md) | Vector similarity, distance metrics, ANN vs brute-force, pre/post filtering |
| Full-text & hybrid search | [fts-hybrid.md](references/fts-hybrid.md) | BM25 keyword search, hybrid vector+FTS, reranking |
| Indexing | [indexing.md](references/indexing.md) | IVF/HNSW index types, FTS indexes, scalar indexes, reindexing |
| Embeddings | [embeddings.md](references/embeddings.md) | Embedding registry, auto-vectorization, supported providers |
| Versioning & consistency | [versioning.md](references/versioning.md) | Time travel, version restore, consistency intervals, optimize/compaction |
| Storage & config | [storage.md](references/storage.md) | S3/GCS/Azure config, storage_options, DynamoDB commit store |

## Key concepts

- **Lance format**: open columnar format optimized for ML — random access, versioning, zero-copy
- **No separate server**: embedded library, connects via local path or object store URI
- **Schema**: Arrow-based. Vector columns are `FixedSizeList<Float32>`. Nested structs supported
- **Versioning**: every mutation creates a new version. `checkout(version)` for time travel
- **Indexing**: optional — brute-force kNN works well up to ~100k vectors. Create ANN indexes for larger datasets
- **Enterprise**: managed service with `db://` URIs, auto-indexing, async background optimization

## Connection patterns

```typescript
// Local embedded
const db = await lancedb.connect("data/lancedb");

// S3
const db = await lancedb.connect("s3://bucket/path", {
  storageOptions: { region: "us-east-1" }
});

// Enterprise
const db = await lancedb.connect("db://your-project", {
  apiKey: "...", region: "us-east-1"
});
```

## TypeScript SDK surface

Core operations map directly to methods on `db` and `table` objects:

- `db.createTable(name, data, opts?)` / `db.openTable(name)` / `db.dropTable(name)`
- `table.add(data)` / `table.update(opts)` / `table.delete(filter)` / `table.mergeInsert(on)`
- `table.search(vector).limit(k).where(filter).toArray()`
- `table.createIndex(column, opts)` / `table.listIndices()`
- `table.schema` / `table.addColumns(transforms)` / `table.alterColumns(alts)` / `table.dropColumns(names)`
- `table.version` / `table.checkout(v)` / `table.checkoutLatest()`
