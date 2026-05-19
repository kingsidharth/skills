# Streaming

`streaming=True` returns an `IterableDataset` — data fetched on-the-fly, no cache, forward-only iteration. For resuming a stream after a crash, see [RESUMABLE.md](RESUMABLE.md).

## When to stream

| Stream | Cache (regular load) |
|---|---|
| Dataset > disk | Need random access |
| One-pass exploration | Multi-epoch training on small data |
| Parquet pushdown (columns/filters) | Need `len()` or `ds[i]` |

## Basic stream

```python
from datasets import load_dataset

ds = load_dataset("HuggingFaceFW/fineweb", split="train", streaming=True)
print(next(iter(ds)))

# From local compressed files — skips Arrow conversion
ds = load_dataset("json", data_files="path/*.jsonl.gz", split="train", streaming=True)

# Column indexing on a stream
for txt in ds["text"]:
    ...
```

## Parquet pushdown — don't download what you won't use

Two independent knobs. Combine freely:

```python
# Columns — unselected columns never leave the server
ds = load_dataset("big/repo", split="train", streaming=True,
                  columns=["url", "date"])

# Filters — Parquet row-group statistics skip chunks at storage level
ds = load_dataset("HuggingFaceFW/fineweb", split="train", streaming=True,
                  filters=[("language_score", ">=", 0.99)])

# Together
ds = load_dataset("big/repo", split="train", streaming=True,
                  columns=["text", "score"],
                  filters=[("score", ">=", 0.9), ("lang", "=", "en")])
```

Filter operators: `=`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not in`.

Without `streaming=True`, `filters` still works but the full dataset downloads first. Combine with streaming.

## take / skip

```python
first_100 = ds.take(100)
rest = ds.skip(100)
```

**Gotcha:** `take`/`skip` lock shard order. Shuffle *before* splitting:

```python
ds = ds.shuffle(seed=42, buffer_size=10_000)
val = ds.take(1000)
train = ds.skip(1000)
```

## Shuffle (buffered approximate)

`IterableDataset.shuffle()` fills a buffer, samples randomly, refills. Also shuffles shard order.

```python
ds = ds.shuffle(seed=42, buffer_size=10_000)
```

Reshuffle per epoch:

```python
for epoch in range(num_epochs):
    ds.set_epoch(epoch)   # seed becomes initial_seed + epoch
    for ex in ds:
        ...
```

## Shard / reshard

```python
# For parallel workers
ds.shard(num_shards=4, index=0)         # keep 1/4 of the shards

# Some datasets are single-shard; reshard splits row groups
ds = ds.reshard()      # format-dependent; Parquet reshards via row groups
# If num_shards stays 1 after reshard, fall back to skip+take chunking
```

## Interleave multiple streams

```python
from datasets import interleave_datasets

en = load_dataset("allenai/c4", "en", split="train", streaming=True)
fr = load_dataset("allenai/c4", "fr", split="train", streaming=True)

multi = interleave_datasets([en, fr])                              # alternate
multi = interleave_datasets([en, fr], probabilities=[0.8, 0.2], seed=42)   # weighted
```

Stopping strategies:

| Strategy | Behavior |
|---|---|
| `first_exhausted` (default) | Stop when any source runs out |
| `all_exhausted` | Continue until every source seen ≥1 time (oversampling) |
| `all_exhausted_without_replacement` | Every sample seen exactly once |

## Concatenate

```python
from datasets import concatenate_datasets

combined = concatenate_datasets([stories, wiki])            # vertical (rows)
side_by_side = concatenate_datasets([a, ids], axis=1)       # horizontal (cols, same nrows)
```

## Dataset → IterableDataset

Faster than re-streaming from the Hub when data is already local:

```python
ds = load_dataset("ethz/food101")
ids = ds.to_iterable_dataset(num_shards=64)
ids = ids.shuffle(seed=42, buffer_size=10_000)
```

## PyTorch DataLoader

```python
import torch
from torch.utils.data import DataLoader

ds = load_dataset("big/repo", split="train", streaming=True)
ds = ds.shuffle(seed=42, buffer_size=10_000)
ds = ds.with_format("torch")

loader = DataLoader(ds, batch_size=32, num_workers=4)

for epoch in range(num_epochs):
    ds.set_epoch(epoch)
    for batch in loader:
        ...
```

Each worker gets `num_shards / num_workers` shards — shard count must be ≥ `num_workers`. If it isn't, call `.to_iterable_dataset(num_shards=N)` or `.reshard()`.

## Batch iteration

```python
for batch in ds.batch(batch_size=32):      # dict of lists, 32 each
    ...
ds.batch(batch_size=32, drop_last_batch=True)
```

## Advanced Parquet tuning

For high-throughput streams from object storage, enlarge the Parquet read buffer:

```python
import pyarrow.dataset as pads

fragment_scan_options = pads.ParquetFragmentScanOptions(
    cache_options=pads.CacheOptions(
        prefetch_limit=1,
        range_size_limit=128 * 1024 * 1024,   # 128 MiB (default 32 MiB)
    )
)
ds = load_dataset("big/repo", streaming=True,
                  fragment_scan_options=fragment_scan_options)

# Override RecordBatch size (default = Parquet row group size)
ds = load_dataset("parquet", data_files="data.parquet", batch_size=4096)
```
