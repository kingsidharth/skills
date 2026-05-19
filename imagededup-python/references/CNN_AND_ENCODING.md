# CNN & Encoding Generation

## Hashing method classes

Four hashing classes share an identical API. All produce 64-bit hashes as 16-char hex strings.

```python
from imagededup.methods import PHash, AHash, DHash, WHash
```

| Class | Algorithm | Best for |
|---|---|---|
| `PHash` | Perceptual (DCT-based) | General-purpose near-duplicate detection |
| `AHash` | Average intensity | Fast, simple exact duplicates |
| `DHash` | Gradient difference | Fastest; good for exact duplicates |
| `WHash` | Wavelet transform | Balanced speed/accuracy |

### Constructor

```python
hasher = PHash(verbose=True)  # verbose: show progress bar
```

### `hamming_distance(hash1, hash2)`

Compute hamming distance between two hex hash strings. Pads to 64 bits if needed.

```python
dist = hasher.hamming_distance('abcdef1234567890', 'abcdef1234567891')
```

---

## CNN method

Uses a pre-trained CNN (MobileNetV3 Small by default) to generate 576-dim feature vectors. Compares via cosine similarity.

```python
from imagededup.methods import CNN

cnn = CNN(verbose=True, model_config=None)
```

**Args:**
- `verbose` (bool): Show progress bar. Default `True`.
- `model_config` (CustomModel | None): Optional custom PyTorch model. See [CUSTOM_MODELS.md](CUSTOM_MODELS.md).

### `apply_preprocess(im_arr)`

Apply the model's preprocessing to a numpy image array. Returns a PyTorch tensor.

---

## Encoding generation

Both hashing and CNN classes share these methods.

### `encode_images(image_dir, recursive=False, num_enc_workers=<cpu_count>)`

Generate encodings for all images in a directory.

**Args:**
- `image_dir` (str): Path to image directory.
- `recursive` (bool): Recurse into subdirectories. Default `False`.
- `num_enc_workers` (int): CPU cores for multiprocessing. `0` disables. For CNN on Linux only.

**Returns:** `dict[str, str | np.ndarray]` — filename → hash string (hashing) or numpy array shape `(576,)` (CNN).

**Supported formats:** JPEG, PNG, BMP, MPO, PPM, TIFF, GIF, SVG, PGM, PBM, WEBP.

```python
encodings = hasher.encode_images(image_dir='./images/', recursive=True)
```

### `encode_image(image_file=None, image_array=None)`

Generate encoding for a single image. Provide either `image_file` (path) or `image_array` (numpy). Returns `None` if the image can't be loaded.

```python
h = hasher.encode_image(image_file='photo.jpg')
# or
h = hasher.encode_image(image_array=np.array(Image.open('photo.jpg')))
```
