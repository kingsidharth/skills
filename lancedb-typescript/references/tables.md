# Tables & Data Operations

## Table creation

```typescript
// From array of records (schema inferred)
const table = await db.createTable("name", [
  { id: 1, text: "hello", vector: [0.1, 0.2, 0.3] }
], { mode: "overwrite" }); // omit mode to error on existing

// From Arrow table
import * as arrow from "apache-arrow";
const schema = new arrow.Schema([
  new arrow.Field("id", new arrow.Int64()),
  new arrow.Field("vector", new arrow.FixedSizeList(3, new arrow.Field("item", new arrow.Float32()))),
]);
const table = await db.createTable("name", [], { schema });

// Open existing
const table = await db.openTable("name");

// List tables
const names = await db.tableNames();

// Drop table (permanent, not recoverable)
await db.dropTable("name");
```

## Append data

```typescript
await table.add([
  { id: 3, text: "new", vector: [0.7, 0.8, 0.9] }
]);
```

- Each `add()` creates a new version
- For large ingestion, batch into reasonable chunks rather than row-by-row

## Update rows

```typescript
// Update by filter with literal values
await table.update({ where: "id = 2", values: { text: "updated" } });

// Update with SQL expressions
await table.update({ where: "id = 2", valuesSql: { count: "count + 1" } });
```

- Updated rows are moved out of existing indexes — consider reindexing after bulk updates
- Nested column updates not yet supported

## Merge insert (upsert)

Compare incoming rows against existing by key, then choose behavior per match status.

```typescript
// Upsert: update matched + insert new
await table.mergeInsert("id")
  .whenMatchedUpdateAll()
  .whenNotMatchedInsertAll()
  .execute([
    { id: 2, text: "Bobby", count: 21 },
    { id: 3, text: "Charlie", count: 5 }
  ]);
```

Available behaviors:
- `.whenMatchedUpdateAll()` — update keys that exist in target
- `.whenNotMatchedInsertAll()` — insert keys missing from target
- `.whenNotMatchedBySourceDelete(filter?)` — delete target rows absent from source
- Combine for full upsert

**Performance**: create a scalar index on the join column for large tables, otherwise merge scans the full column.

## Delete rows

```typescript
await table.delete("id = 1"); // SQL filter expression
```

- Soft delete — rows excluded from queries but not physically removed
- Run `table.optimize()` to compact and reclaim space
- Default cleanup retention: 7 days

## Schema evolution

```typescript
// Add columns (SQL expression based)
await table.addColumns([
  { name: "power", valueSql: "(stats.strength + stats.magic) / 2.0" }
]);

// Alter columns (rename, change type, set nullable)
await table.alterColumns([
  { path: "name", rename: "full_name" },
  { path: "count", dataType: new arrow.Float32() }
]);

// Drop columns
await table.dropColumns(["power", "temp_col"]);
```

Changing `FixedSizeList` dimensions (e.g. embedding 384→1024) requires a 3-step pattern:
1. `addColumns` with new dimension via `arrow_cast`
2. `dropColumns` old column
3. Rename new column to original name

## Schema types (Arrow mapping)

| JS/TS type | Arrow type | Notes |
|---|---|---|
| `number[]` | `FixedSizeList<Float32>` | Vector column — must be fixed-size |
| `string` | `Utf8` | |
| `number` | `Float64` or `Int64` | Inferred from data |
| `boolean` | `Boolean` | |
| `object` | `Struct` | Nested fields become struct children |
| `Uint8Array` | `Binary` / `FixedSizeBinary` | Binary/blob columns |

## Multimodal data (blobs)

Store images, audio, video as binary columns. Use `Blob` type for large binary data that LanceDB handles with lazy reading.
