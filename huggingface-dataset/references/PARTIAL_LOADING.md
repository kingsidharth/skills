# Partial Loading & Streaming Reference

## Streaming

Stream data on-the-fly without downloading or caching. Returns an `IterableDataset`.

```python
from datasets import load_dataset

# From Hub
ds = load_dataset("HuggingFaceFW/fineweb", split="train", streaming=True)
print(next(iter(ds)))  # fetches only what's needed

# From local files (skips Arrow conversion)
ds = load_dataset("json", data_files="path/to/huge/*.jsonl.gz", split="train", streaming=True)

# Column indexing on streams
texts = ds["text"]  # iterate only over the text column
print(next(iter(texts)))
```

**When to use streaming vs regular loading:**
- Streaming: dataset won't fit on disk, you only need one pass, or you want to explore
- Regular: you need random access, multiple passes, or fast indexed lookups

### Shuffle with Streaming

Uses a buffer-based approach (not full shuffle):

```python
ds = ds.shuffle(seed=42, buffer_size=10_000)
```

This fills a buffer of 10K examples, randomly samples from it, and refills. Also shuffles shard order if the dataset has multiple files.

For epoch-level reshuffling:

```python
for epoch in range(num_epochs):
    ds.set_epoch(epoch)  # seed becomes: initial_seed + epoch
    for batch in ds:
        ...
```

### take / skip (Slice a Stream)

```python
first_100 = ds.take(100)      # only first 100 examples
rest = ds.skip(100)            # everything after first 100

# WARNING: take/skip lock shard order — shuffle BEFORE splitting
ds = ds.shuffle(seed=42, buffer_size=10_000)
train = ds.skip(1000)
val = ds.take(1000)
```

### Interleave Multiple Datasets

```python
from datasets import interleave_datasets

en = load_dataset("allenai/c4", "en", split="train", streaming=True)
fr = load_dataset("allenai/c4", "fr", split="train", streaming=True)

# Alternate examples
multi = interleave_datasets([en, fr])

# Weighted sampling
multi = interleave_datasets([en, fr], probabilities=[0.8, 0.2], seed=42)

# Stopping strategies:
# "first_exhausted" (default) — stop when any dataset runs out
# "all_exhausted" — keep going until all datasets seen at least once
# "all_exhausted_without_replacement" — every sample seen exactly once
multi = interleave_datasets([en, fr], stopping_strategy="all_exhausted")
```

### Convert Between Dataset Types

```python
# Dataset → IterableDataset (faster than re-streaming from Hub)
ids = ds.to_iterable_dataset(num_shards=64)
ids = ids.shuffle(seed=42, buffer_size=10_000)

# Use with DataLoader
from torch.utils.data import DataLoader
loader = DataLoader(ids.with_format("torch"), num_workers=4)
# Each worker gets 64/4 = 16 shards
```

## Column Selection (Parquet Pushdown)

Only fetch the columns you need. Unselected columns never leave the server.

```python
ds = load_dataset("big/dataset", split="train", streaming=True,
                  columns=["url", "date"])
```

Works with regular loading too:

```python
ds = load_dataset("big/dataset", split="train", columns=["text", "label"])
```

## Filter Pushdown (Parquet Statistics)

Exploits Parquet row-group statistics to skip data at the storage level:

```python
filters = [("language_score", ">=", 0.99)]
ds = load_dataset("HuggingFaceFW/fineweb", split="train", streaming=True, filters=filters)
```

Best combined with `streaming=True` — without streaming, the full dataset downloads first and then filters.

For ImageFolder with Parquet metadata:

```python
filters = [("label", "=", 0)]
ds = load_dataset("user/imagenet-subset", streaming=True, filters=filters)
```

## Advanced Parquet Tuning

For high-throughput streaming, increase the request buffer and enable prefetching:

```python
import pyarrow.dataset

fragment_scan_options = pyarrow.dataset.ParquetFragmentScanOptions(
    cache_options=pyarrow.CacheOptions(
        prefetch_limit=1,
        range_size_limit=128 * 1024 * 1024  # 128 MiB (default is 32 MiB)
    )
)
ds = load_dataset("big/dataset", streaming=True,
                  fragment_scan_options=fragment_scan_options)
```

For `batch_size` control in the Parquet reader (default = row group size):

```python
ds = load_dataset("parquet", data_files="data.parquet",
                  batch_size=4096)  # override RecordBatch size
```

## Split Slicing

Load only a portion of a split:

```python
# By index (absolute)
ds = load_dataset("repo", split="train[10:20]")

# By percentage
ds = load_dataset("repo", split="train[:10%]")
ds = load_dataset("repo", split="train[-80%:]")

# Combine segments
ds = load_dataset("repo", split="train[:10%]+train[-80%:]")

# Concatenate splits
ds = load_dataset("repo", split="train+test")

# Cross-validated folds
val_splits = [f"train[{k}%:{k+10}%]" for k in range(0, 100, 10)]
train_splits = [f"train[:{k}%]+train[{k+10}%:]" for k in range(0, 100, 10)]
val_ds = load_dataset("repo", split=val_splits)
train_ds = load_dataset("repo", split=train_splits)
```

Programmatic API with `ReadInstruction`:

```python
from datasets import ReadInstruction

ri = ReadInstruction("train", from_=10, to=20, unit="abs")
ds = load_dataset("repo", split=ri)

ri = ReadInstruction("train", to=10, unit="%")
ds = load_dataset("repo", split=ri)

# Equal-sized splits (pct1_dropremainder rounding)
ri = ReadInstruction("train", from_=50, to=52, unit="%", rounding="pct1_dropremainder")
```

## Shard-Level Loading

For datasets with many shard files, glob only what you need:

```python
# Load first 5 shards out of 1024
ds = load_dataset("allenai/c4", data_files="en/c4-train.0000*-of-01024.json.gz")

# Load from a specific subdirectory
ds = load_dataset("allenai/c4", data_dir="en")

# Map data files to specific splits
data_files = {"validation": "en/c4-validation.*.json.gz"}
ds = load_dataset("allenai/c4", data_files=data_files, split="validation")
```

## Inspect Without Downloading

```python
from datasets import load_dataset_builder

builder = load_dataset_builder("cornell-movie-review-data/rotten_tomatoes")
print(builder.info.description)
print(builder.info.features)
print(builder.info.splits)
print(builder.info.download_size)
print(builder.info.dataset_size)
```

## Multiprocessing

Speed up download + preparation with multiple processes (one per shard):

```python
ds = load_dataset("timm/imagenet-1k-wds", num_proc=8)
```

## Batch Iteration on Streams

```python
batched = ds.batch(batch_size=32)
for batch in batched:
    # batch is a dict of lists, 32 examples each
    ...

# Drop incomplete last batch
batched = ds.batch(batch_size=32, drop_last_batch=True)
```

## Training Loop Integration

```python
import torch
from torch.utils.data import DataLoader

ds = load_dataset("big/dataset", split="train", streaming=True)
ds = ds.shuffle(seed=42, buffer_size=10_000)
ds = ds.with_format("torch")

loader = DataLoader(ds, batch_size=32)

for epoch in range(num_epochs):
    ds.set_epoch(epoch)
    for batch in loader:
        ...
```
