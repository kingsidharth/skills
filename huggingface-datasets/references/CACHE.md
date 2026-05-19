# Cache Management

Two caches. Know which is which.

| Cache | Location (default) | Contains | Env var |
|---|---|---|---|
| **Hub cache** | `~/.cache/huggingface/hub/` | Raw downloaded files (Parquet, TAR, JSON, models) | `HF_HUB_CACHE` |
| **Datasets cache** | `~/.cache/huggingface/datasets/` | Arrow-format processed datasets + `.map` outputs | `HF_DATASETS_CACHE` |

`HF_HOME` sets the root for both:

```bash
export HF_HOME=/mnt/big_disk/hf
# → /mnt/big_disk/hf/hub   and   /mnt/big_disk/hf/datasets
```

`HF_DATASETS_CACHE` only affects the datasets-specific cache. It does **not** move Hub downloads. On networked/shared setups, set both (or just `HF_HOME`) to avoid surprises.

## Per-call override

```python
ds = load_dataset("user/repo", cache_dir="/path/to/cache")
```

Only affects this call. Env vars are the right tool for persistent changes.

## Download modes

```python
from datasets import load_dataset, DownloadMode

# Re-use cache if present (default)
ds = load_dataset("user/repo")

# Force full re-download
ds = load_dataset("user/repo", download_mode="force_redownload")

# Re-process cached raw files, but don't re-download
ds = load_dataset("user/repo", download_mode="reuse_dataset_if_exists")   # default
ds = load_dataset("user/repo", download_mode="reuse_cache_if_exists")
```

## Clean up Arrow cache

```python
ds.cleanup_cache_files()      # returns count removed
```

Removes stale Arrow files from processing. Safe — re-processing will regenerate.

## Disable caching

### For a single `.map` call

```python
ds = ds.map(fn, load_from_cache_file=False)
```

### Globally

```python
from datasets import disable_caching
disable_caching()
```

With caching disabled, every `.map` re-runs on access. Useful during development, bad for production.

## Offline mode

```bash
export HF_HUB_OFFLINE=1
```

Everything reads from the cache; no network requests. Fails loudly if something is missing instead of silently going out to the net.

## In-memory datasets

For small datasets, skip disk entirely for speed:

```python
import datasets
datasets.config.IN_MEMORY_MAX_SIZE = 1_000_000_000   # 1 GB
# or:
# export HF_DATASETS_IN_MEMORY_MAX_SIZE=1000000000
```

Datasets under the threshold are held fully in RAM. First method takes precedence.

## Inspecting the cache

```bash
hf cache ls             # list cached repos with sizes
hf cache scan           # detailed view
hf cache rm <repo>      # delete a specific repo
```

## Typical setups

**Shared workstation, per-user:**

```bash
# in ~/.bashrc
export HF_HOME=$HOME/.cache/huggingface
```

**GPU cluster with fast scratch disk:**

```bash
export HF_HOME=/scratch/$USER/hf
```

**Avoid re-downloading across experiments:**

```bash
export HF_HUB_CACHE=/mnt/shared/hf_hub      # shared read across users
export HF_DATASETS_CACHE=$HOME/hf_datasets   # per-user processing cache
```
