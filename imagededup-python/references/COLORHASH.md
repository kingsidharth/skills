# Color Hash (imagehash)

`imagehash.colorhash` analyzes **HSV color distribution** — not spatial structure. From the `ImageHash` library (`pip install ImageHash`).

Useful when images have been structurally modified (rotated, flipped, filtered) but color palette is preserved.

## API

```python
imagehash.colorhash(image, binbits=3)
```

**Args:**
- `image` (PIL.Image): Input image.
- `binbits` (int): Bits per bucket. Default `3`. Higher = more sensitive to color shifts.

**Returns:** `ImageHash` object.

## How it works

Converts to HSV and bins pixels into intensity/saturation/hue buckets:

- First `binbits` bits: fraction of black pixels
- Next `binbits` bits: fraction of gray pixels (low saturation)
- Next `6 × binbits` bits: hue distribution for highly saturated pixels (6 hue bins)
- Next `6 × binbits` bits: hue distribution for mildly saturated pixels (6 hue bins)

Total hash size: `(2 + 6 + 6) × binbits` = `14 × binbits` bits.

## Usage

```python
from PIL import Image
import imagehash

img = Image.open('photo.jpg')
chash = imagehash.colorhash(img, binbits=3)
print(chash)          # hex string
print(type(chash))    # ImageHash
```

### Comparing color hashes

```python
h1 = imagehash.colorhash(img1, binbits=3)
h2 = imagehash.colorhash(img2, binbits=3)
distance = h1 - h2   # hamming distance (int)
h1 == h2              # True if distance == 0
```

### Serialization

```python
ch = imagehash.colorhash(img, binbits=3)
hex_str = str(ch)
restored = imagehash.hex_to_flathash(hex_str, hashsize=3)
assert ch == restored
```

## When to use

- Detecting recolored images
- Finding images with the same color palette regardless of structure
- Supplementing structural hashes for multi-signal matching

## When NOT to use

- Detecting crops, rotations, or structural edits where color is unchanged — use structural hashes or crop-resistant hash instead
- Two completely different images can share a color hash if they happen to have similar color distributions
