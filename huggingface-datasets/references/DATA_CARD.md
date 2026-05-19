# Data Card & YAML Configuration

Every dataset repo has a `README.md`. YAML front-matter (between `---` markers) controls how the Hub and `datasets` library treat it.

## Structure

```yaml
---
# machine-readable config
license: mit
language: [en]
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
---

# Human-readable docs below
```

## Core tags

| Field | Values |
|---|---|
| `license` | `apache-2.0`, `mit`, `cc-by-4.0`, `cc-by-sa-4.0`, `cc-by-nc-4.0`, `other` |
| `language` | ISO codes: `en`, `es`, `zh`; or `multilingual` |
| `task_categories` | `text-classification`, `image-classification`, `question-answering`, `text-generation`, `object-detection`, `image-to-text`, `automatic-speech-recognition`, ... |
| `size_categories` | `n<1K`, `1K<n<10K`, `10K<n<100K`, `100K<n<1M`, `1M<n<10M`, `10M<n<100M`, `100M<n<1B`, `n>1T` |
| `pretty_name` | Display name |
| `tags` | Custom — `medical`, `finance`, `synthetic`, `curated` |

## Extended (optional)

| Field | Values |
|---|---|
| `annotations_creators` | `expert-generated`, `crowdsourced`, `machine-generated`, `no-annotation` |
| `language_creators` | `found`, `crowdsourced`, `expert-generated`, `machine-generated` |
| `multilinguality` | `monolingual`, `multilingual`, `translation` |
| `source_datasets` | List of source dataset IDs |
| `paperswithcode_id` | PwC slug |

## `configs` — splits and subsets

### Single config, explicit splits

```yaml
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
  - split: validation
    path: "data/val-*.parquet"
```

`config_name` is required even for one config.

### Multiple configs (subsets)

```yaml
configs:
- config_name: english
  data_files: "en/*.parquet"
  default: true
- config_name: spanish
  data_files: "es/*.parquet"
- config_name: french
  data_files: "fr/*.parquet"
```

Load: `load_dataset("user/repo", "spanish")`. `default: true` lets `load_dataset("user/repo")` work without a config name.

### Multiple files per split

```yaml
configs:
- config_name: default
  data_files:
  - split: train
    path:
    - "data/train_part1.parquet"
    - "data/train_part2.parquet"
```

### Per-config builder params

```yaml
configs:
- config_name: tabs
  data_files: "data_tabs.csv"
  sep: "\t"
- config_name: commas
  data_files: "data_commas.csv"
  sep: ","
```

Any kwarg accepted by the underlying builder (CSV, JSON, Parquet) works here.

## Automatic split detection (no `configs` block)

Priority order:

1. Shard filenames: `data/train-00000-of-00003.parquet`
2. Filename keywords: `train.csv`, `my_test.csv`, `validation1.csv` (must be delimited by non-word chars)
3. Directory names: files inside `train/`, `test/`, `validation/`
4. Fallback: single `train` split

Keyword equivalents:

| Canonical | Also matches |
|---|---|
| `train` | `training` |
| `validation` | `valid`, `val`, `dev` |
| `test` | `testing`, `eval`, `evaluation` |

## Recommended repo layouts

Simple:

```
my_dataset/
├── README.md
├── train.parquet
└── test.parquet
```

Sharded:

```
my_dataset/
├── README.md
└── data/
    ├── train-00000-of-00010.parquet
    ├── train-00001-of-00010.parquet
    └── test-00000-of-00002.parquet
```

Multi-config:

```
my_dataset/
├── README.md
├── en/train.parquet
├── en/test.parquet
├── es/train.parquet
└── es/test.parquet
```

ImageFolder:

```
my_dataset/
├── README.md
├── train/metadata.csv
├── train/img_001.png
└── test/metadata.csv
```

## Gating

Enable via dataset Settings → Access requests. Users must accept terms before `load_dataset` works with their token.

## Minimum good card

```markdown
---
license: apache-2.0
task_categories: [image-classification]
language: [en]
pretty_name: My Image Dataset
size_categories: [100K<n<1M]
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
---

# My Image Dataset

## Description
Brief overview and intended use.

## Structure
Fields, splits, types.

## Collection
How data was sourced and processed.

## Considerations
Biases, limitations, ethics.

## Citation
BibTeX.
```
