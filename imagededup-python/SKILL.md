---
name: imagededup-python
description: "Find & remove duplicate/near-duplicate images using perceptual hashing (PHash, AHash, DHash, WHash) and CNN embeddings. Use when deduplicating image datasets or comparing image similarity in python. Libraries: imagededup and imagehash"
---

# Image Deduplication (Python)

Two complementary libraries: **imagededup** (idealo) for batch directory deduplication with hashing + CNN, and **imagehash** (JohannesBuchner) for standalone perceptual hashing including color and crop-resistant hashes.

## Installation

```bash
# imagededup — hashing + CNN dedup (requires torch)
pip install imagededup
uv add imagededup

# imagehash — standalone hashing (lighter, PIL-based)
pip install ImageHash
uv add ImageHash
```

## Method selection

| Goal | Method | Library | Threshold |
|---|---|---|---|
| Fast exact-duplicate detection | `DHash` | imagededup | `max_distance_threshold=10` |
| Robust near-duplicate detection | `PHash` | imagededup | `max_distance_threshold=10-15` |
| Transformation-resistant (rotation, scale) | `CNN` | imagededup | `min_similarity_threshold=0.85-0.95` |
| Color-aware matching (ignores structure) | `colorhash` | imagehash | hamming distance threshold |
| Crop-resistant matching (up to 50%) | `crop_resistant_hash` | imagehash | `bit_error_rate=0.25` |
| Fastest at scale | `DHash` | imagededup | — |
| Best quality (near-dupes + transforms) | `CNN` | imagededup | — |

## Quick start — imagededup

```python
from imagededup.methods import PHash

phasher = PHash()

# Find duplicates in a directory
duplicates = phasher.find_duplicates(
    image_dir='path/to/images/',
    max_distance_threshold=12,
    scores=True
)
# Returns: {'img1.jpg': [('img2.jpg', 3), ...], ...}

# Get flat list of files to remove
to_remove = phasher.find_duplicates_to_remove(
    image_dir='path/to/images/',
    max_distance_threshold=12
)
# Returns: ['img2.jpg', 'img5.jpg', ...]
```

## Quick start — imagehash (colorhash + crop-resistant)

```python
from PIL import Image
import imagehash

img = Image.open('photo.jpg')

# Color hash — analyzes HSV color distribution (ignores structure)
chash = imagehash.colorhash(img, binbits=3)

# Crop-resistant hash — survives up to 50% cropping
crhash = imagehash.crop_resistant_hash(img, min_segment_size=500, segmentation_image_size=1000)
```

## Detailed references

- **[CNN_AND_ENCODING.md](references/CNN_AND_ENCODING.md)**: CNN method, encoding generation, hashing method classes
- **[FINDING_DUPLICATES.md](references/FINDING_DUPLICATES.md)**: `find_duplicates()` and `find_duplicates_to_remove()` — full API for bulk/folder deduplication
- **[CUSTOM_MODELS.md](references/CUSTOM_MODELS.md)**: Plugging custom PyTorch models into CNN dedup
- **[CIFAR10_EXAMPLE.md](references/CIFAR10_EXAMPLE.md)**: End-to-end CIFAR-10 deduplication walkthrough with cross-set detection
- **[COLORHASH.md](references/COLORHASH.md)**: HSV color distribution hashing (imagehash library)
- **[CROP_RESISTANT_HASH.md](references/CROP_RESISTANT_HASH.md)**: Crop-resistant multi-segment hashing (imagehash library)

## Anti-patterns

- **Don't use CNN for exact duplicates** — hashing is orders of magnitude faster and equally accurate for byte-identical or near-identical images.
- **Don't set `max_distance_threshold` too high** — values above 20 produce excessive false positives. Start at 10 and increase gradually.
- **`find_duplicates_to_remove` uses a greedy heuristic** — it may keep different "originals" depending on traversal order. For fine-grained control, use `find_duplicates` and apply your own dedup logic.
- **imagededup's `CNN` downloads a model on first use** — MobileNetV3 by default. Ensure internet access or pre-download.
- **`crop_resistant_hash` returns a multi-hash** — you cannot compare it with `==` against regular `ImageHash` objects. Use its `.matches()` method or convert via `hex_to_multihash`.
