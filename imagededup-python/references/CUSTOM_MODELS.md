# Custom Models

Use `CustomModel` to plug in a different PyTorch feature extractor for CNN-based deduplication.

```python
from imagededup.methods import CNN
from imagededup.utils import CustomModel
```

## Pre-packaged models

Three models ship with imagededup:

| Model | Class | Base |
|---|---|---|
| MobileNetV3 (default) | `MobilenetV3` | torchvision mobilenet_v3_small |
| Vision Transformer | `ViT` | torchvision vit_b_16 (SWAG) |
| EfficientNet | `EfficientNet` | torchvision efficientnet_b4 |

```python
from imagededup.utils.models import ViT, EfficientNet

config = CustomModel(
    name=EfficientNet.name,
    model=EfficientNet(),
    transform=EfficientNet.transform
)
cnn = CNN(model_config=config)

# Use as normal
encodings = cnn.encode_images(image_dir='./images/')
duplicates = cnn.find_duplicates(encoding_map=encodings)
```

## User-defined model

Your model must be a `torch.nn.Module` subclass whose `forward` returns shape `(batch_size, features)`.

```python
import torch
from torchvision.transforms import transforms

class MyExtractor(torch.nn.Module):
    transform = transforms.Compose([transforms.Resize(224), transforms.ToTensor()])
    name = 'my_extractor'

    def __init__(self):
        super().__init__()
        self.backbone = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
        self.backbone.fc = torch.nn.Identity()

    def forward(self, x):
        return self.backbone(x)

config = CustomModel(name=MyExtractor.name, model=MyExtractor(), transform=MyExtractor.transform)
cnn = CNN(model_config=config)
```

It is not necessary to bundle `name` and `transform` as class attributes — they can be passed separately to `CustomModel`.
