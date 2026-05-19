# Crop-Resistant Hash (imagehash)

`imagehash.crop_resistant_hash` produces a **multi-hash** — one hash per detected image segment. Survives up to ~50% cropping (most other algorithms fail at ~5%). From the `ImageHash` library (`pip install ImageHash`).

Based on the paper "Efficient Cropping-Resistant Robust Image Hashing" (DOI 10.1109/ARES.2014.85).

## API

```python
imagehash.crop_resistant_hash(
    image,
    hash_func=None,
    limit_segments=None,
    segment_threshold=128,
    min_segment_size=500,
    segmentation_image_size=300
)
```

**Args:**
- `image` (PIL.Image): Input image.
- `hash_func` (callable | None): Hash function for each segment. Default uses `average_hash`.
- `limit_segments` (int | None): Max number of segments to hash (largest first). `None` for all.
- `segment_threshold` (int): Brightness threshold for hill/valley segmentation. Default `128`.
- `min_segment_size` (int): Minimum pixels per segment. Default `500`. Lower = more segments (slower, more detailed).
- `segmentation_image_size` (int): Resize image to this dimension for segmentation. Default `300`. Higher = slower but more precise.

**Returns:** `ImageMultihash` — a collection of per-segment hashes.

## How it works

The algorithm partitions the image into bright and dark segments using a watershed-like algorithm, then hashes each segment independently. A cropped image retains most segments, so at least some segment hashes will match the original.

## Usage

```python
from PIL import Image
import imagehash

img = Image.open('photo.jpg')
cr_hash = imagehash.crop_resistant_hash(
    img,
    min_segment_size=500,
    segmentation_image_size=1000
)

# Access individual segment hashes
print(cr_hash.segment_hashes)  # list of ImageHash objects
```

## Comparing crop-resistant hashes

`ImageMultihash` has custom comparison that finds matching segments:

```python
hash_full = imagehash.crop_resistant_hash(img_full)
hash_cropped = imagehash.crop_resistant_hash(img_cropped)

# Equality uses bit_error_rate (default 0.25 = 25% tolerance per segment)
print(hash_full == hash_cropped)  # True if enough segments match
```

### `hash_diff(other, hamming_cutoff=None, bit_error_rate=0.25)`

Fine-grained comparison returning match quality.

- `hamming_cutoff` (int | None): Max hamming distance for a segment match.
- `bit_error_rate` (float): Fraction of bits that can differ (alternative to `hamming_cutoff`). Default `0.25`.
- **Returns:** `(num_matches, sum_hamming_distances)`. Higher `num_matches` = more similar.

```python
matches, distance = hash_cropped.hash_diff(hash_full)
print(f"Matching segments: {matches}, total hamming distance: {distance}")
```

## Serialization

```python
cr = imagehash.crop_resistant_hash(img)
hex_str = str(cr)
restored = imagehash.hex_to_multihash(hex_str)
assert cr == restored
assert str(restored) == hex_str
```

## Gotchas

- **Cannot compare with regular `ImageHash` objects** — `crop_resistant_hash` returns `ImageMultihash`, not `ImageHash`.
- **Segmentation varies slightly between Pillow versions** — Pillow 6 vs ≥7 produces different grayscale rounding, leading to slightly different segments.
- **Slower than standard hashes** — watershed segmentation adds overhead. Use `segmentation_image_size` and `min_segment_size` to control speed/quality tradeoff.
- **`limit_segments`** — use this if you have storage constraints; it hashes only the M largest segments.
