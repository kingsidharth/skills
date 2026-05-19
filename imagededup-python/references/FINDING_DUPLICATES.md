# Finding Duplicates

Two methods for bulk/folder deduplication. Both work on all method classes (`PHash`, `AHash`, `DHash`, `WHash`, `CNN`).

## `find_duplicates(...)`

Find duplicate pairs with optional scores.

**Hashing signature:**
```python
from imagededup.methods import PHash

hasher = PHash()
duplicates = hasher.find_duplicates(
    image_dir=None,              # path to image directory
    encoding_map=None,           # OR pre-computed encodings dict
    max_distance_threshold=10,   # int 0-64, max hamming distance
    scores=False,                # return distances with filenames
    outfile=None,                # save results to JSON
    search_method='brute_force_cython',  # or 'bktree' (Windows default)
    recursive=False,
    num_enc_workers=0,           # encoding parallelism
    num_dist_workers=0,          # distance computation parallelism
)
```

**CNN signature:**
```python
from imagededup.methods import CNN

cnn = CNN()
duplicates = cnn.find_duplicates(
    image_dir=None,
    encoding_map=None,
    min_similarity_threshold=0.9,  # float -1.0 to 1.0, cosine sim
    scores=False,
    outfile=None,
    recursive=False,
    num_enc_workers=0,
    num_sim_workers=0,
)
```

**Returns (scores=False):**
```python
{'img1.jpg': ['img2.jpg', 'img3.jpg'], 'img2.jpg': ['img1.jpg'], ...}
```

**Returns (scores=True):**
```python
{'img1.jpg': [('img2.jpg', 3), ('img3.jpg', 5)], ...}
```

The relationship is symmetric: if A is duplicate of B, B is duplicate of A. Images that fail to load are silently skipped.

### Examples

Perceptual hashing with scores saved to JSON:
```python
from imagededup.methods import PHash

phasher = PHash()
duplicates = phasher.find_duplicates(
    image_dir='path/to/images/',
    max_distance_threshold=12,
    scores=True,
    outfile='my_duplicates.json'
)
```

CNN with pre-computed encodings:
```python
from imagededup.methods import CNN

cnn = CNN()
encodings = cnn.encode_images(image_dir='path/to/images/')
duplicates = cnn.find_duplicates(
    encoding_map=encodings,
    min_similarity_threshold=0.85,
    scores=False,
    outfile='my_duplicates.json'
)
```

---

## `find_duplicates_to_remove(...)`

Same args as `find_duplicates` (minus `scores`). Returns a flat `list[str]` of filenames to remove. Does **NOT** delete any files.

```python
to_remove = hasher.find_duplicates_to_remove(
    image_dir='./images/',
    max_distance_threshold=12,
    outfile='remove_list.json'
)
# Returns: ['img2.jpg', 'img5.jpg', ...]
```

**Gotcha — greedy heuristic:** When image A and B are duplicates, and B and C are duplicates, the method may mark only B for removal (since removing B breaks the A↔B and B↔C chains). But it could also mark A and C instead, keeping only B. The outcome depends on traversal order.

For deterministic control, use `find_duplicates` and implement your own selection logic:

```python
duplicates = hasher.find_duplicates(image_dir='./images/', max_distance_threshold=12)

# Custom logic: keep the file with the shortest name
to_remove = set()
for filename, dupes in duplicates.items():
    if filename in to_remove:
        continue
    for dupe in dupes:
        if dupe not in to_remove and len(dupe) > len(filename):
            to_remove.add(dupe)
```

---

## Plotting duplicates

Visualize duplicates found for a given image.

```python
from imagededup.utils import plot_duplicates

plot_duplicates(
    image_dir='./images/',
    duplicate_map=duplicates,     # from find_duplicates()
    filename='query_image.jpg',   # which image's duplicates to show
    outfile='duplicates_plot.png' # optional, save to file
)
```

Accepts both scored and unscored duplicate maps.
