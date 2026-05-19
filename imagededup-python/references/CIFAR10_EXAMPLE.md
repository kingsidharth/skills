# Example: CIFAR-10 Deduplication

End-to-end walkthrough: download CIFAR-10, find duplicates with CNN, plot results, and detect cross-set leakage (test images duplicated in train).

## Setup

```bash
pip install imagededup
wget http://pjreddie.com/media/files/cifar.tgz
tar xzf cifar.tgz

# Merge train + test into one directory for global dedup
mkdir cifar10_images
cp -r cifar/train/* cifar10_images/
cp -r cifar/test/* cifar10_images/
```

## Find all duplicates

```python
from imagededup.methods import CNN

cnn = CNN()
encodings = cnn.encode_images(image_dir='cifar10_images/')
duplicates = cnn.find_duplicates(encoding_map=encodings)
```

## Plot duplicates for a specific image

```python
from imagededup.utils import plot_duplicates
import matplotlib.pyplot as plt

plt.rcParams['figure.figsize'] = (15, 10)

# Sort by number of duplicates, plot the top one
sorted_dupes = sorted(duplicates.items(), key=lambda x: len(x[1]), reverse=True)
plot_duplicates(
    image_dir='cifar10_images/',
    duplicate_map=duplicates,
    filename=sorted_dupes[0][0]
)
```

## Cross-set deduplication (train vs test leakage)

Detect test images that have near-duplicates in the training set — a common source of inflated evaluation metrics.

```python
from pathlib import Path

filenames_test = {p.name for p in Path('cifar/test').glob('*.png')}
filenames_train = {p.name for p in Path('cifar/train').glob('*.png')}

# Test images with duplicates in train
cross_dupes = {}
for k, v in duplicates.items():
    if k in filenames_test:
        train_dupes = [f for f in v if f in filenames_train]
        if train_dupes:
            cross_dupes[k] = train_dupes

cross_dupes = dict(sorted(cross_dupes.items(), key=lambda x: len(x[1]), reverse=True))

print(f"Test images with train duplicates: {len(cross_dupes)}")
plot_duplicates(
    image_dir='cifar10_images/',
    duplicate_map=cross_dupes,
    filename=list(cross_dupes.keys())[0]
)
```

## Within-set deduplication

```python
# Duplicates only within test set
dupes_test = {}
for k, v in duplicates.items():
    if k in filenames_test:
        same_set = [f for f in v if f in filenames_test]
        if same_set:
            dupes_test[k] = same_set

# Duplicates only within train set
dupes_train = {}
for k, v in duplicates.items():
    if k in filenames_train:
        same_set = [f for f in v if f in filenames_train]
        if same_set:
            dupes_train[k] = same_set
```
