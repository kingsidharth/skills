# Integrations

The Hub's `hf://` URI scheme is fsspec-compatible. Libraries that speak fsspec can read and write directly — no `datasets` import required.

**URI shape:** `hf://datasets/{owner}/{repo}/{path}` (optionally `@{revision}` before `/`).

```
hf://datasets/user/repo/data/train.parquet
hf://datasets/user/repo@v1.0/data/train.parquet
```

For private/gated repos, set `HF_TOKEN` before importing the library.

## Pandas

```python
import pandas as pd

df = pd.read_parquet("hf://datasets/user/repo/data/train.parquet")
df = pd.read_csv("hf://datasets/user/repo/data.csv")
df = pd.read_json("hf://datasets/user/repo/data.jsonl", lines=True)

# Read a glob via pyarrow backend
df = pd.read_parquet("hf://datasets/user/repo/data/")  # reads all parquet in dir
```

## Polars

Lazy, columnar, fast for large files on cloud storage.

```python
import polars as pl

# Lazy scan — reads schema only, defers compute
lf = pl.scan_parquet("hf://datasets/user/repo/data/train-*.parquet")
df = (lf
      .filter(pl.col("score") >= 0.9)
      .select(["text", "label"])
      .collect())

# Eager
df = pl.read_parquet("hf://datasets/user/repo/data/train.parquet")
df = pl.read_csv("hf://datasets/user/repo/data.csv")
df = pl.read_ndjson("hf://datasets/user/repo/data.jsonl")
```

Polars `scan_parquet` + filter pushdown is the fastest pattern for partial reads of remote Parquet.

## DuckDB

SQL directly over remote Parquet/CSV/JSON.

```python
import duckdb

duckdb.sql("""
  SELECT label, COUNT(*)
  FROM 'hf://datasets/user/repo/data/*.parquet'
  WHERE score >= 0.9
  GROUP BY label
""").show()

# With authentication
duckdb.sql(f"""
  CREATE SECRET hf_token (TYPE huggingface, TOKEN '{token}');
""")
duckdb.sql("SELECT * FROM 'hf://datasets/private/repo/data.parquet' LIMIT 10")

# Materialize to a DuckDB table
con = duckdb.connect("analysis.duckdb")
con.execute("CREATE TABLE t AS SELECT * FROM 'hf://datasets/user/repo/data/*.parquet'")
```

Predicate and projection pushdown work — only needed row groups and columns transfer.

## Daft

Distributed dataframe for very large datasets.

```python
import daft

df = daft.read_parquet("hf://datasets/user/repo/data/*.parquet")
df = df.where(df["score"] >= 0.9).select("text", "label")
df.show()
df.write_parquet("output/")
```

Daft handles remote + local + cloud storage uniformly and scales across cores/nodes.

## PyArrow

```python
import pyarrow.dataset as pads
import pyarrow.parquet as pq

# Directly
table = pq.read_table("hf://datasets/user/repo/data/train.parquet")

# As a multi-file dataset (filter + projection pushdown)
ds = pads.dataset("hf://datasets/user/repo/data/", format="parquet")
table = ds.to_table(columns=["text", "label"],
                     filter=(pads.field("score") >= 0.9))
```

## Converting between formats

### Arrow `Dataset` ↔ Pandas

```python
df = ds.to_pandas()
ds = Dataset.from_pandas(df)
```

### Arrow `Dataset` ↔ Polars

```python
df = ds.to_polars()
ds = Dataset.from_polars(df)
```

### Arrow `Dataset` ↔ PyArrow Table

```python
table = ds.data.table           # zero-copy view into Arrow table
ds = Dataset(table)
```

## Cross-library tips

- **For analytics workflows**, read with Polars/DuckDB/Daft directly from `hf://`. Don't pull through `datasets` unless you need the `map`/`filter`/`with_format` pipeline.
- **For training**, use `datasets` — its `IterableDataset`, shuffling buffer, and DataLoader integration are purpose-built for epoch iteration.
- **For exploration**, DuckDB is the fastest way to answer "what's in this dataset" without downloading it.
