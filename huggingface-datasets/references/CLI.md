# CLI

Two CLI tools. Both come with their respective pip packages.

| Tool | Package | Scope |
|---|---|---|
| `hf` | `huggingface_hub` | General Hub operations — download, upload, list, auth |
| `datasets-cli` | `datasets` | Dataset-specific — env info, test loading, delete config |

## Auth

```bash
hf auth login              # interactive
hf auth login --token hf_...
hf auth whoami
hf auth logout
```

Or env:

```bash
export HF_TOKEN=hf_...
```

## Listing & searching

```bash
# Datasets
hf datasets ls                              # recent/trending
hf datasets ls --search "instruction"
hf datasets ls --author Qwen
hf datasets ls --sort downloads --limit 10
hf datasets ls --filter task_categories:text-classification

# Info for one
hf datasets info HuggingFaceFW/fineweb
```

## Downloading

```bash
# Single file
hf download user/repo data/train.parquet --repo-type dataset

# Multiple specific files
hf download user/repo config.json data/train.parquet --repo-type dataset

# Glob patterns
hf download user/repo --repo-type dataset \
    --include "data/*.parquet" --exclude "data/test*"

# To a specific directory (not cache)
hf download user/repo --repo-type dataset --local-dir ./my_copy

# Dry run — check sizes first
hf download user/repo --repo-type dataset --dry-run
hf download user/repo --repo-type dataset --include "*.parquet" --dry-run

# Specific revision
hf download user/repo --repo-type dataset --revision v1.0
```

## Uploading

```bash
# Single file
hf upload user/repo ./local.parquet data/local.parquet --repo-type dataset

# Folder
hf upload user/repo ./data/ . --repo-type dataset --include "*.parquet"

# With commit message
hf upload user/repo ./data/ . --repo-type dataset \
    --commit-message "add Q4 shards"

# Create repo first if needed
hf repo create user/repo --type dataset
```

## Cache management

```bash
hf cache ls                                  # list cached repos + sizes
hf cache scan --verbosity 1
hf cache rm user/repo --repo-type dataset   # delete one
hf cache rm --filter older-than 30d         # bulk cleanup
```

## `datasets-cli`

```bash
datasets-cli --help

# Print system info (helpful for bug reports)
datasets-cli env

# Test that a dataset script / repo loads cleanly
datasets-cli test path/to/my/dataset --save_info

# Delete a config from a dataset on the Hub
datasets-cli delete_from_hub USERNAME/DATASET_NAME CONFIG_NAME \
    --token hf_... --revision main
```

## Common one-liners

```bash
# Peek at a dataset without cloning
hf download user/repo README.md --repo-type dataset --local-dir .
cat README.md

# Download only the README + metadata Parquet
hf download user/repo --repo-type dataset \
    --include "README.md" "metadata.parquet" --local-dir .

# Size check before a big download
hf download HuggingFaceFW/fineweb --repo-type dataset \
    --include "data/CC-MAIN-2024*" --dry-run
```
