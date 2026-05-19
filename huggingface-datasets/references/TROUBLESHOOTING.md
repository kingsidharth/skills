# Troubleshooting

Common failures and how to get past them.

## Upload

### `401 Unauthorized` on `push_to_hub`

Token lacks write scope. Regenerate from Hub → Settings → Access Tokens with "write" selected.

On macOS, stale keychain entries can shadow a new token:

```bash
git config --global credential.helper osxkeychain
hf auth logout
hf auth login
```

### Lost connection on large upload

Many shards → many commits → rate limit or transient S3 500. **Just re-run `push_to_hub`** — already-uploaded shards are skipped via SHA check.

### `HfHubHTTPError: 429 Too Many Requests`

```
You have exceeded our hourly quotas for action: commit. We invite you to retry later.
```

Upgrade `datasets` to ≥ 2.15.0. Newer versions batch commits. Then wait an hour and retry — existing uploads are preserved.

## Loading

### `FileNotFoundError` with local files

Check that `data_files` paths resolve. Globs need quoting in shell; from Python, they work as-is:

```python
ds = load_dataset("parquet", data_files="data/*.parquet")   # OK
```

Map explicitly if auto-detection fails:

```python
ds = load_dataset("parquet", data_files={"train": "data/train-*.parquet",
                                          "test": "data/test.parquet"})
```

### Slow first load, fast subsequent

Normal — first load converts to Arrow in cache. To avoid re-conversion if cache is wrong:

```python
ds = load_dataset(..., download_mode="force_redownload")
```

### "Wrong splits" on Hub dataset

Auto-detection picked up something unintended. Write an explicit `configs:` block in README YAML. See [DATA_CARD.md](DATA_CARD.md).

## ImageFolder / AudioFolder

- `metadata.csv` / `metadata.jsonl` / `metadata.parquet` — filename must be exact
- `file_name` column is mandatory and case-sensitive
- Paths in `file_name` use forward slashes, relative to the metadata file
- If both subdirs and metadata exist, metadata overrides matching columns

See [FOLDER_BUILDERS.md](FOLDER_BUILDERS.md).

## Pickling

### `TypeError: cannot pickle 'generator' object`

`Dataset.from_generator` wants a **generator function**, not a generator object:

```python
# WRONG
gen = (process(x) for x in source)
ds = Dataset.from_generator(gen)         # fails

# RIGHT
def gen():
    for x in source:
        yield process(x)
ds = Dataset.from_generator(gen)
```

### `.map(num_proc>1)` pickling errors

Child processes pickle the callable and its closure. If something in scope isn't picklable (DB connections, CUDA contexts, lambdas capturing complex state):

- Move the unpicklable object's construction *inside* the map function
- Or provide a stable hash explicitly:

```python
ds = ds.map(fn, new_fingerprint="my_fn_v1", num_proc=8)
```

- Last resort: disable caching for this call:

```python
from datasets import disable_caching
disable_caching()
```

## Performance

### `.filter` is glacially slow on large datasets

- For Parquet sources, prefer `filters=[(col, op, val)]` in `load_dataset` (pushdown — skips unread row groups)
- Increase `batch_size` and use `batched=True`
- Use `num_proc` (but mind the pickling rules above)
- Operate on the IterableDataset form if streaming is an option

### Shuffled dataset iteration is slow

Shuffle creates an indices mapping. Two fixes:

```python
ds = ds.shuffle(seed=42).flatten_indices()     # rewrites contiguously on disk
```

Or switch to iterable form:

```python
ids = ds.to_iterable_dataset(num_shards=128).shuffle(seed=42, buffer_size=1000)
```

### DataLoader with `num_workers > 1` is slow / duplicates data

`IterableDataset` must have enough shards (`num_shards ≥ num_workers`). Check:

```python
ids = load_dataset(..., streaming=True)
print(ids.num_shards)
```

If 1, call `.to_iterable_dataset(num_shards=64)` or `.reshard()`.

For resumable training across crashes, use `StatefulDataLoader` from `torchdata`. See [RESUMABLE.md](RESUMABLE.md).

## GPU / CUDA

### `AttributeError: 'CUDAProperties' object has no attribute 'total_mem'`

```python
# WRONG
torch.cuda.get_device_properties(0).total_mem

# RIGHT
torch.cuda.get_device_properties(0).total_memory
```

## Cache confusion

Two caches exist. Changing `HF_DATASETS_CACHE` doesn't move Hub downloads. See [CACHE.md](CACHE.md) — set `HF_HOME` if you want to relocate everything.

## When stuck

- `datasets-cli env` — dump versions and paths for bug reports
- GitHub issues: https://github.com/huggingface/datasets/issues
- Forums: https://discuss.huggingface.co/c/datasets/10
