# Bounding Box / Object Detection with Grok Vision

Use Grok's vision + world knowledge for language-driven object detection. Unlike CNN-based detectors,
Grok can detect objects by semantic description (e.g., "lions that are cubs", "cars that are Teslas",
"text in Japanese only").

**Approach: Two-stage pipeline**
1. Vision model outputs bounding box coordinates as free-form text
2. Language model parses the text into a strict Pydantic/Zod schema

This two-stage approach is more reliable than asking the vision model to output structured JSON directly.

## Schema

### Python

```python
from pydantic import BaseModel

class BoundingBox(BaseModel):
    object_name: str
    y1: float   # top (normalized 0.0-1.0)
    x1: float   # left
    y2: float   # bottom
    x2: float   # right

class BoundingBoxes(BaseModel):
    boxes: list[BoundingBox]
```

### TypeScript (Zod)

```typescript
import { z } from "zod";

const BoundingBox = z.object({
  object_name: z.string(),
  y1: z.number(), // top (normalized 0.0-1.0)
  x1: z.number(), // left
  y2: z.number(), // bottom
  x2: z.number(), // right
});

const BoundingBoxes = z.object({
  boxes: z.array(BoundingBox),
});
```

## Detection Prompt

The prompt instructs Grok to output normalized coordinates relative to image dimensions:

```python
OBJECT_DETECTION_PROMPT = """You are an AI assistant specialized in object detection \
and drawing accurate bounding boxes.
Your task is to generate normalized coordinates for bounding boxes based on given \
instructions and an image.

The coordinates for the bounding boxes should be normalized relative to the width \
and height of the image.

Instructions:
1. Analyze the image carefully.
2. Identify all objects matching the user's query.
3. For each object, provide:
   - object_name: a short descriptive label
   - Bounding box as [y1, x1, y2, x2] where all values are normalized floats (0.0 to 1.0)
     - y1 = top edge, x1 = left edge, y2 = bottom edge, x2 = right edge

User query: {USER_INSTRUCTIONS}
"""
```

## Python — Full Two-Stage Pipeline (Cookbook Pattern)

```python
import os
import base64
from openai import OpenAI
from pydantic import BaseModel
from PIL import Image, ImageDraw, ImageFont

GROK_VISION_MODEL = "grok-4-1-fast-reasoning"
GROK_MODEL = "grok-4-1-fast-reasoning"

client = OpenAI(
    api_key=os.getenv("XAI_API_KEY"),
    base_url="https://api.x.ai/v1",
)

class BoundingBox(BaseModel):
    object_name: str
    y1: float
    x1: float
    y2: float
    x2: float

class BoundingBoxes(BaseModel):
    boxes: list[BoundingBox]

def base64_encode_image(image_path: str) -> str:
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")


def generate_bounding_boxes(
    image_path: str,
    user_query: str,
) -> BoundingBoxes:
    """Two-stage: vision → text coordinates → structured parse."""

    prompt = OBJECT_DETECTION_PROMPT.format(USER_INSTRUCTIONS=user_query)

    # ── Stage 1: Vision model outputs raw coordinate text ──
    completion = client.chat.completions.create(
        model=GROK_VISION_MODEL,
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{base64_encode_image(image_path)}",
                            "detail": "high",
                        },
                    },
                    {"type": "text", "text": prompt},
                ],
            },
        ],
    )

    raw_coords_text = completion.choices[0].message.content
    if not raw_coords_text:
        raise ValueError("Vision model returned empty response")

    # ── Stage 2: Parse raw text into structured BoundingBoxes ──
    completion = client.beta.chat.completions.parse(
        model=GROK_MODEL,
        messages=[
            {
                "role": "user",
                "content": f"Extract the bounding box coordinates from this text:\n{raw_coords_text}",
            },
        ],
        response_format=BoundingBoxes,
    )

    return completion.choices[0].message.parsed


def draw_bounding_boxes(image_path: str, boxes: BoundingBoxes) -> Image.Image:
    """Draw detected bounding boxes on the image."""
    img = Image.open(image_path)
    draw = ImageDraw.Draw(img)
    w, h = img.size

    colors = ["#FF0000", "#00FF00", "#0000FF", "#FFFF00", "#FF00FF", "#00FFFF"]

    for i, box in enumerate(boxes.boxes):
        color = colors[i % len(colors)]
        # Convert normalized coords to pixel coords
        x1_px, y1_px = int(box.x1 * w), int(box.y1 * h)
        x2_px, y2_px = int(box.x2 * w), int(box.y2 * h)

        draw.rectangle([x1_px, y1_px, x2_px, y2_px], outline=color, width=3)
        draw.text(
            (x1_px, max(0, y1_px - 15)),
            box.object_name,
            fill=color,
        )

    return img


# ─── Usage ────────────────────────────────────
boxes = generate_bounding_boxes(
    "lions.jpg",
    "Detect all lions. Label each as adult male, adult female, or cub."
)

annotated = draw_bounding_boxes("lions.jpg", boxes)
annotated.save("lions_annotated.jpg")

for box in boxes.boxes:
    print(f"  {box.object_name}: ({box.x1:.2f},{box.y1:.2f}) → ({box.x2:.2f},{box.y2:.2f})")
```

## TypeScript — Two-Stage Pipeline

```typescript
import OpenAI from "openai";
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";
import { readFileSync } from "fs";

const client = new OpenAI({
  apiKey: process.env.XAI_API_KEY,
  baseURL: "https://api.x.ai/v1",
});

const BoundingBox = z.object({
  object_name: z.string(),
  y1: z.number(), x1: z.number(),
  y2: z.number(), x2: z.number(),
});
const BoundingBoxes = z.object({ boxes: z.array(BoundingBox) });

async function detectObjects(imagePath: string, query: string) {
  const b64 = readFileSync(imagePath).toString("base64");
  const dataUri = `data:image/jpeg;base64,${b64}`;

  // Stage 1: Vision → raw text
  const stage1 = await client.chat.completions.create({
    model: "grok-4-1-fast-reasoning",
    messages: [
      {
        role: "user",
        content: [
          { type: "image_url", image_url: { url: dataUri, detail: "high" } },
          { type: "text", text: OBJECT_DETECTION_PROMPT.replace("{USER_INSTRUCTIONS}", query) },
        ],
      },
    ],
  });

  const rawText = stage1.choices[0].message.content!;

  // Stage 2: Parse → structured
  const stage2 = await client.beta.chat.completions.parse({
    model: "grok-4-1-fast-reasoning",
    messages: [
      { role: "user", content: `Extract bounding box coordinates:\n${rawText}` },
    ],
    response_format: zodResponseFormat(BoundingBoxes, "BoundingBoxes"),
  });

  return stage2.choices[0].message.parsed!;
}
```

## Advanced: Language-Driven Detection

Grok's unique advantage over CNN detectors is **semantic specificity**:

```python
# Detect only specific car brands
boxes = generate_bounding_boxes("parking_lot.jpg", "Find all Tesla vehicles")

# Detect animals by behavior
boxes = generate_bounding_boxes("farm.jpg", "Find animals that appear to be sleeping")

# Detect text by language
boxes = generate_bounding_boxes("sign.jpg", "Find all text written in Japanese")

# Detect by condition
boxes = generate_bounding_boxes("shelf.jpg", "Find products with damaged packaging")
```

## Key Gotchas

- **Coordinates are normalized 0.0–1.0** — multiply by image width/height for pixels
- **Two-stage is critical** — direct structured output from vision models is unreliable for coordinates
- **Results are approximate** — this is an LLM, not a precision detector. Good for ~80% accuracy.
- **Complex scenes** — overlapping objects, partial occlusion, and background objects reduce accuracy
- **`detail: "high"`** — always use for detection tasks; `"low"` loses spatial precision
- **Rate limits** — when processing many images, use async with semaphore (see VISION_STRUCTURED.md)
