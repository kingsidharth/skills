# Data Card & YAML Configuration Reference

## README.md Structure

Every dataset repo has a `README.md` with YAML front-matter (between `---` markers) followed by Markdown documentation.

```yaml
---
# YAML front-matter (machine-readable config)
license: mit
language:
  - en
---

# My Dataset

Human-readable documentation goes here...
```

## YAML Front-Matter Fields

### Core Tags

| Field | Type | Example Values |
|---|---|---|
| `license` | string | `mit`, `apache-2.0`, `cc-by-4.0`, `cc-by-sa-4.0`, `cc-by-nc-4.0`, `other` |
| `language` | list | `en`, `es`, `zh`, `multilingual` |
| `task_categories` | list | `text-classification`, `image-classification`, `question-answering`, `text-generation`, `object-detection`, `image-to-text`, `automatic-speech-recognition` |
| `size_categories` | list | `n<1K`, `1K<n<10K`, `10K<n<100K`, `100K<n<1M`, `1M<n<10M`, `10M<n<100M`, `100M<n<1B` |
| `pretty_name` | string | Human-readable name shown on Hub |
| `tags` | list | Custom tags like `medical`, `finance`, `synthetic`, `curated` |

### Extended Tags (Optional)

| Field | Example Values |
|---|---|
| `annotations_creators` | `expert-generated`, `crowdsourced`, `machine-generated`, `no-annotation` |
| `language_creators` | `found`, `crowdsourced`, `expert-generated`, `machine-generated` |
| `multilinguality` | `monolingual`, `multilingual`, `translation`, `other` |
| `source_datasets` | List of source dataset IDs |
| `paperswithcode_id` | PapersWithCode dataset slug |

## Configs Block — Defining Splits and Subsets

The `configs` field is the primary mechanism for controlling how data is loaded.

### Single Config with Explicit Splits

```yaml
---
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
  - split: validation
    path: "data/val-*.parquet"
---
```

`config_name` is required even for a single configuration.

### Multiple Configs (Subsets)

```yaml
---
configs:
- config_name: english
  data_files: "en/*.parquet"
  default: true
- config_name: spanish
  data_files: "es/*.parquet"
- config_name: french
  data_files: "fr/*.parquet"
---
```

Load: `load_dataset("user/repo", "spanish")`

Set `default: true` on one config so `load_dataset("user/repo")` works without specifying a name.

### Multiple Files Per Split

```yaml
---
configs:
- config_name: default
  data_files:
  - split: train
    path:
    - "data/train_part1.parquet"
    - "data/train_part2.parquet"
  - split: test
    path: "data/test.parquet"
---
```

### Glob Patterns

```yaml
---
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
---
```

### Builder Parameters

Pass format-specific parameters per config:

```yaml
---
configs:
- config_name: tabs
  data_files: "data_tabs.csv"
  sep: "\t"
- config_name: commas
  data_files: "data_commas.csv"
  sep: ","
---
```

Any parameter accepted by the underlying builder (CSV, JSON, Parquet, etc.) can be set here.

## Automatic Split Detection (No YAML Needed)

When no `configs` block is present, the library infers splits:

### Priority Order

1. **Custom shard filenames:** `data/train-00000-of-00003.parquet`, `data/random-00000-of-00003.parquet`
2. **Filename keywords:** `train.csv`, `my_test_file.csv`, `validation1.csv` — the split keyword must be delimited by non-word characters (underscores, dashes, dots, spaces, numbers)
3. **Directory names:** Files in `train/`, `test/`, `validation/` directories
4. **Fallback:** Everything becomes a single `train` split

### Keyword Equivalents

| Canonical | Also Recognized |
|---|---|
| `train` | `training` |
| `validation` | `valid`, `val`, `dev` |
| `test` | `testing`, `eval`, `evaluation` |

### Multiple Files Per Split

Automatic detection works across multiple files:

```
data/train_0.csv
data/train_1.csv
data/train_2.csv
data/test_0.csv
data/test_1.csv
```

All `train_*` files become the `train` split. All `test_*` files become the `test` split.

## Recommended Repository Structure

### Simple Dataset

```
my_dataset/
├── README.md
├── train.parquet
└── test.parquet
```

### Sharded Dataset

```
my_dataset/
├── README.md
└── data/
    ├── train-00000-of-00010.parquet
    ├── train-00001-of-00010.parquet
    ├── ...
    └── test-00000-of-00002.parquet
```

### Multi-Config Dataset

```
my_dataset/
├── README.md
├── en/
│   ├── train.parquet
│   └── test.parquet
├── es/
│   ├── train.parquet
│   └── test.parquet
└── fr/
    ├── train.parquet
    └── test.parquet
```

### ImageFolder Dataset

```
my_dataset/
├── README.md
├── train/
│   ├── metadata.csv
│   ├── img_001.png
│   └── img_002.png
└── test/
    ├── metadata.csv
    ├── img_003.png
    └── img_004.png
```

## Gated Datasets

Require users to share contact info before downloading:

1. Go to dataset Settings on the Hub
2. Enable "Access requests"
3. Users must accept terms before `load_dataset()` works

## Data Card Template

A good data card includes these sections:

```markdown
---
license: apache-2.0
task_categories:
  - image-classification
language:
  - en
pretty_name: My Image Dataset
size_categories:
  - 100K<n<1M
configs:
- config_name: default
  data_files:
  - split: train
    path: "data/train-*.parquet"
  - split: test
    path: "data/test-*.parquet"
---

# My Image Dataset

## Dataset Description
Brief overview and intended use.

## Dataset Structure
Description of fields, splits, and data types.

## Data Collection
How the data was sourced and processed.

## Considerations
Known biases, limitations, or ethical considerations.

## Citation
BibTeX for academic use.
```
