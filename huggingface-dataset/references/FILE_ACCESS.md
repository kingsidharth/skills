# File-Level Access Reference

Direct access to individual files in dataset repositories using `huggingface_hub`.

## hf_hub_download — Single File

```python
from huggingface_hub import hf_hub_download

# Download a single file (cached locally)
path = hf_hub_download(
    repo_id="google/fleurs",
    filename="fleurs.py",
    repo_type="dataset"
)
# Returns: '/root/.cache/huggingface/hub/datasets--google--fleurs/snapshots/.../fleurs.py'

# Specific version (branch, tag, commit, or PR)
path = hf_hub_download(repo_id="user/repo", filename="config.json",
                       repo_type="dataset", revision="v1.0")
path = hf_hub_download(repo_id="user/repo", filename="config.json",
                       repo_type="dataset", revision="refs/pr/3")

# To a specific local directory (not cache)
path = hf_hub_download(repo_id="user/repo", filename="data/train.parquet",
                       repo_type="dataset", local_dir="./my_data")
# Result: ./my_data/data/train.parquet (preserves directory structure)

# Force re-download
path = hf_hub_download(repo_id="user/repo", filename="data.parquet",
                       repo_type="dataset", force_download=True)
```

## snapshot_download — Partial Repo with Glob Filters

Download an entire repo or a filtered subset:

```python
from huggingface_hub import snapshot_download

# Entire repo
snapshot_download(repo_id="user/repo", repo_type="dataset")

# Only JSON files
snapshot_download(repo_id="user/repo", repo_type="dataset",
                  allow_patterns="*.json")

# Exclude large binary files
snapshot_download(repo_id="user/repo", repo_type="dataset",
                  ignore_patterns=["*.bin", "*.h5", "*.safetensors"])

# Combine include + exclude
snapshot_download(repo_id="user/repo", repo_type="dataset",
                  allow_patterns=["*.md", "*.json"],
                  ignore_patterns="vocab.json")

# Specific revision
snapshot_download(repo_id="user/repo", repo_type="dataset",
                  revision="refs/pr/1")

# To a local directory
snapshot_download(repo_id="user/repo", repo_type="dataset",
                  local_dir="./my_dataset")
```

## Dry Run — Check Before Downloading

```python
# Programmatic
info = hf_hub_download(repo_id="user/repo", filename="big.parquet",
                       repo_type="dataset", dry_run=True)
# Returns DryRunFileInfo with: commit_hash, filename, size, is_cached, would_download

# For entire repo
infos = snapshot_download(repo_id="user/repo", repo_type="dataset", dry_run=True)
# Returns list of DryRunFileInfo
```

## HfFileSystem — fsspec Interface

Treat the Hub as a filesystem. Files are streamed on demand, not fully downloaded.

```python
from huggingface_hub import HfFileSystem

fs = HfFileSystem()  # uses default token from huggingface-cli login
# Or: fs = HfFileSystem(token="hf_...")

# List directory contents
fs.ls("datasets/user/my-dataset", detail=False)
# Returns: ['datasets/user/my-dataset/.gitattributes', 'datasets/user/my-dataset/README.md', ...]

# Glob pattern matching
files = fs.glob("datasets/user/my-dataset/data/*.parquet")

# Check if file exists
fs.exists("datasets/user/my-dataset/data/train.parquet")

# Get file info (size, commit, etc.)
info = fs.info("datasets/user/my-dataset/data/train.parquet")

# Read a file directly (streams from Hub)
with fs.open("datasets/user/my-dataset/data/config.json") as f:
    content = f.read()

# Walk directory tree
for dirpath, dirnames, filenames in fs.walk("datasets/user/my-dataset"):
    print(dirpath, filenames)

# Find all files recursively
all_files = fs.find("datasets/user/my-dataset")

# Get download URL
url = fs.url("datasets/user/my-dataset/data/train.parquet")
```

### fsspec Integration with Pandas/Polars

Because `HfFileSystem` is fsspec-compatible, pandas and other tools can read directly:

```python
import pandas as pd

# Read Parquet directly from Hub (no explicit download needed)
df = pd.read_parquet("hf://datasets/user/my-dataset/data/train.parquet")

# Read CSV
df = pd.read_csv("hf://datasets/user/my-dataset/data.csv")
```

## CLI Download

```bash
# Single file
hf download user/repo data/train.parquet --repo-type dataset

# Multiple files
hf download user/repo config.json data/train.parquet --repo-type dataset

# With glob patterns
hf download user/repo --repo-type dataset --include "data/*.parquet" --exclude "data/test*"

# To a specific local directory
hf download user/repo --repo-type dataset --local-dir ./my_local_copy

# Dry run (check sizes)
hf download user/repo --repo-type dataset --dry-run
hf download user/repo --repo-type dataset --include "*.parquet" --dry-run
```

## Authentication for Private/Gated Datasets

```python
# Option 1: Login once (persists)
# huggingface-cli login

# Option 2: Environment variable
# export HF_TOKEN=hf_...

# Option 3: Pass token directly
from huggingface_hub import hf_hub_download
path = hf_hub_download(repo_id="user/private-repo", filename="data.parquet",
                       repo_type="dataset", token="hf_...")

# For datasets library
from datasets import load_dataset
ds = load_dataset("user/private-repo", token="hf_...")
# Or set HF_TOKEN env var and it's picked up automatically
```

## Cache Management

Default cache: `~/.cache/huggingface/hub/`

Override with `cache_dir` parameter or `HF_HOME` env var.

For offline use:
```bash
export HF_HUB_OFFLINE=1
# Will only look in cache, no network requests
```

The `local_dir` parameter creates a `.cache/huggingface/` metadata folder at the root. This prevents re-downloading unchanged files. Safe to delete if you no longer need incremental updates.
