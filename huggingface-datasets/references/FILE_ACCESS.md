# File-Level Access

Download individual files or browse a dataset repo without materializing a `Dataset` object. For listing/filtering datasets, see [DISCOVERY.md](DISCOVERY.md). For resumable downloads, see [RESUMABLE.md](RESUMABLE.md).

## Single file — `hf_hub_download`

```python
from huggingface_hub import hf_hub_download

path = hf_hub_download(
    repo_id="google/fleurs",
    filename="fleurs.py",
    repo_type="dataset",
)
# Returns cache path: ~/.cache/huggingface/hub/datasets--google--fleurs/...
```

Common options:

```python
# Specific revision (branch, tag, commit, PR)
hf_hub_download(..., revision="v1.0")
hf_hub_download(..., revision="refs/pr/3")

# Local directory (preserves subpath)
hf_hub_download(..., local_dir="./my_data")
# Result: ./my_data/data/train.parquet

# Force re-download
hf_hub_download(..., force_download=True)

# Private/gated
hf_hub_download(..., token="hf_...")
```

## Partial repo — `snapshot_download` with glob filters

```python
from huggingface_hub import snapshot_download

# Entire repo
snapshot_download("user/repo", repo_type="dataset")

# Only specific patterns
snapshot_download("user/repo", repo_type="dataset",
                  allow_patterns=["*.parquet", "README.md"])

# Exclude heavy files
snapshot_download("user/repo", repo_type="dataset",
                  ignore_patterns=["*.bin", "*.h5", "*.safetensors"])

# Combine allow + ignore
snapshot_download("user/repo", repo_type="dataset",
                  allow_patterns="data/*.parquet",
                  ignore_patterns="data/test*")

# To a directory (not cache)
snapshot_download("user/repo", repo_type="dataset", local_dir="./my_dataset")
```

Re-running `snapshot_download` is idempotent — already-downloaded files are skipped via SHA check.

## Dry run — size before download

```python
# Single file
info = hf_hub_download("user/repo", filename="big.parquet",
                       repo_type="dataset", dry_run=True)
# DryRunFileInfo: commit_hash, filename, size, is_cached, would_download

# Whole repo
infos = snapshot_download("user/repo", repo_type="dataset", dry_run=True)
total_bytes = sum(i.size for i in infos if i.would_download)
```

## Browse as a filesystem — `HfFileSystem`

Treat the Hub as fsspec. Files are streamed on demand, not pre-downloaded.

```python
from huggingface_hub import HfFileSystem

fs = HfFileSystem()                      # auth via `hf auth login`
# fs = HfFileSystem(token="hf_...")

fs.ls("datasets/user/repo", detail=False)
fs.glob("datasets/user/repo/data/*.parquet")
fs.exists("datasets/user/repo/data/train.parquet")
fs.info("datasets/user/repo/data/train.parquet")   # size, last_commit, blob_id
fs.find("datasets/user/repo")                      # recursive

with fs.open("datasets/user/repo/config.json") as f:
    content = f.read()

for dirpath, dirnames, filenames in fs.walk("datasets/user/repo"):
    ...

url = fs.url("datasets/user/repo/data/train.parquet")
```

### Direct reads from pandas / polars / duckdb

Because `HfFileSystem` is fsspec-compatible, `hf://` URIs work directly:

```python
import pandas as pd

df = pd.read_parquet("hf://datasets/user/repo/data/train.parquet")
df = pd.read_csv("hf://datasets/user/repo/data.csv")
```

See [INTEGRATIONS.md](INTEGRATIONS.md) for Polars, DuckDB, Daft.

## Authentication

```bash
# Option 1 — persistent login
hf auth login

# Option 2 — env var
export HF_TOKEN=hf_...
```

```python
# Option 3 — pass explicitly
hf_hub_download("user/private", filename="data.parquet",
                repo_type="dataset", token="hf_...")
```

## Cache location

Default: `~/.cache/huggingface/hub/`. See [CACHE.md](CACHE.md) for `HF_HOME`, `HF_HUB_CACHE`, and offline mode.

`local_dir` creates a `.cache/huggingface/` metadata folder at its root to enable incremental updates. Safe to delete if you no longer need delta downloads.

## CLI equivalents

```bash
hf download user/repo data/train.parquet --repo-type dataset
hf download user/repo --repo-type dataset --include "*.parquet" --exclude "test*"
hf download user/repo --repo-type dataset --local-dir ./copy
hf download user/repo --repo-type dataset --dry-run
```

See [CLI.md](CLI.md).
