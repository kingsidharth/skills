# Resumable Streaming & Downloads

How to resume a long job after a crash, network failure, or intentional pause — without re-reading data you've already processed.

## The model

An `IterableDataset` has a **resumable position** (cursor). You can snapshot it with `state_dict()` and restore it with `load_state_dict()`. The dataset will skip forward to exactly where you left off on the next iteration.

This works for Hub streams, local file streams, and `DataLoader` workers.

## Stream resume — single worker

```python
from datasets import load_dataset

ds = load_dataset("big/repo", split="train", streaming=True)
ds = ds.shuffle(seed=42, buffer_size=10_000)

it = iter(ds)
for step, ex in enumerate(it):
    train_step(ex)
    if step % 1000 == 0:
        # Snapshot both model and dataset position
        save_checkpoint({
            "model": model.state_dict(),
            "dataset": ds.state_dict(),       # <-- cursor
            "step": step,
        })
```

On restart:

```python
ckpt = load_checkpoint()
ds = load_dataset("big/repo", split="train", streaming=True)
ds = ds.shuffle(seed=42, buffer_size=10_000)   # same config as before
ds.load_state_dict(ckpt["dataset"])             # skip to the cursor
model.load_state_dict(ckpt["model"])

for ex in ds:       # resumes at the exact next example
    train_step(ex)
```

The cursor tracks shard index + row offset. When the dataset is sharded, resume is fast — it jumps directly to the shard and position without replaying from the start.

## Resume with `DataLoader` workers

`StatefulDataLoader` (from `torchdata`) coordinates per-worker cursors:

```python
from torchdata.stateful_dataloader import StatefulDataLoader

ds = load_dataset("big/repo", split="train", streaming=True).with_format("torch")
loader = StatefulDataLoader(ds, batch_size=32, num_workers=4)

for step, batch in enumerate(loader):
    train_step(batch)
    if step % 1000 == 0:
        save_checkpoint({"loader": loader.state_dict(), "step": step})

# Resume
loader.load_state_dict(ckpt["loader"])
```

Plain `DataLoader` cannot resume across workers — the per-worker iterator state is lost on exit. Use `StatefulDataLoader` for any multi-worker resumable setup.

## Epoch resumption

`set_epoch` is part of the resumable state — no extra handling needed:

```python
for epoch in range(start_epoch, num_epochs):
    ds.set_epoch(epoch)
    for ex in ds:
        ...
```

If resuming mid-epoch, the `state_dict` captures both the epoch seed and the in-epoch position.

## What the cursor stores

Approximately:

```python
{
    "examples_iterable": {
        "shard_idx": 7,
        "shard_example_idx": 12345,
        "num_examples_since_previous_state": 1234,  # in shuffle buffer
    },
    "epoch": 3,
    "token_per_repo_id": {...},
}
```

Opaque in practice — treat it as a blob you persist alongside your model checkpoint.

## Requirements & gotchas

- **Config must match on resume.** Same `streaming=True`, same `shuffle(seed, buffer_size)`, same `.map`/`.filter` chain. Changing the pipeline invalidates the cursor.
- **Shard count matters.** With N shards, resume precision = 1 shard. If you have a single shard, you may need to replay from the start of that shard (then skip). Call `.to_iterable_dataset(num_shards=64)` or `.reshard()` when possible.
- **Shuffle buffer is replayed.** After resume, the shuffle buffer refills from the cursor position — the *content* of the buffer post-resume is deterministic given the seed, but not byte-identical to pre-crash if the buffer wasn't full.
- **Map functions must be picklable** for the cursor to round-trip through a checkpoint file.

## Resumable file download

For `hf_hub_download` / `snapshot_download`, resumption is automatic via the cache — re-running the same call skips files already present with matching SHAs.

```python
from huggingface_hub import snapshot_download

# First run — downloads everything
snapshot_download("user/repo", repo_type="dataset", local_dir="./data")

# Crash, then re-run — skips completed files, resumes partial ones
snapshot_download("user/repo", repo_type="dataset", local_dir="./data")
```

To force a retry of a single file:

```python
from huggingface_hub import hf_hub_download

hf_hub_download("user/repo", filename="data/shard-0007.parquet",
                repo_type="dataset", force_download=True)
```

## Discovering where to resume from

If no checkpoint exists but you need to resume *based on data already written downstream* (e.g., processed rows were saved to a Parquet file), use `.skip(n)` after fast-forwarding past processed count:

```python
already_done = count_rows_in_output_parquet()
ds = load_dataset("big/repo", split="train", streaming=True).skip(already_done)
for ex in ds:
    ...
```

This only works if the *order of the stream is deterministic* — same shard order, same shuffle seed, same pipeline. Otherwise use `state_dict` checkpointing.

## Pattern: atomic cursor save

Write-then-rename to avoid corrupted cursor files:

```python
import json, os, tempfile

def save_cursor(path, state):
    fd, tmp = tempfile.mkstemp(dir=os.path.dirname(path))
    with os.fdopen(fd, "w") as f:
        json.dump(state, f)
    os.replace(tmp, path)    # atomic on POSIX
```
