# Vision + Structured Data Extraction

Combine image understanding with structured outputs to extract typed data from images at scale.

## Pattern: Two-Phase Extraction

For complex schemas, use a two-phase approach:
1. **Phase 1** — Vision model analyzes image and outputs free-form text description
2. **Phase 2** — Language model parses the text into a strict Pydantic/JSON schema

This is more reliable than asking a vision model to directly output JSON, especially for complex nested schemas.

## Python SDK — Single Image Extraction

### Direct parse (simple schemas)

```python
from pydantic import BaseModel, Field
from typing import Optional
from xai_sdk import Client
from xai_sdk.chat import system, user, image

class ProductInfo(BaseModel):
    name: str = Field(description="Product name")
    brand: str = Field(description="Brand name if visible")
    color: str = Field(description="Dominant color")
    condition: str = Field(description="new, used, damaged")

client = Client(api_key=os.getenv("XAI_API_KEY"), timeout=3600)
chat = client.chat.create(model="grok-4-1-fast-reasoning", response_format=ProductInfo)
chat.append(system("Extract product info from the image as structured data."))
chat.append(user("Analyze this product.", image(image_url=url, detail="high")))
response = chat.sample()
result = ProductInfo.model_validate_json(response.content)
```

### Two-phase (complex schemas)

```python
# Phase 1: Vision describes
chat1 = client.chat.create(model="grok-4-1-fast-reasoning")
chat1.append(user(
    "Describe everything you see: colors, patterns, style, fabric, fit, accessories.",
    image(image_url=url, detail="high"),
))
description = chat1.sample().content

# Phase 2: Parse into schema
chat2 = client.chat.create(model="grok-4-1-fast-reasoning", response_format=ClothingAttributes)
chat2.append(user(f"Extract structured clothing attributes from this description:\n{description}"))
response, result = chat2.parse(ClothingAttributes)
```

## Python — Batch Async Extraction (Cookbook Pattern)

From the xAI cookbook "Extracting Structured Data from Fashion Images":

```python
import os
import asyncio
import json
import base64
from pathlib import Path
from typing import Optional
from enum import Enum
from pydantic import BaseModel, Field
from openai import AsyncOpenAI

XAI_API_KEY = os.getenv("XAI_API_KEY")
GROK_MODEL = "grok-4-1-fast-reasoning"

# ─── Schema ───────────────────────────────────
class Color(str, Enum):
    BLACK = "black"
    WHITE = "white"
    RED = "red"
    BLUE = "blue"
    # ... extend as needed

class ClothingAttributes(BaseModel):
    """Structured attributes extracted from a fashion image."""
    category: str = Field(description="e.g. dress, shirt, pants, jacket")
    subcategory: Optional[str] = Field(default=None, description="e.g. maxi dress, blazer")
    colors: list[str] = Field(description="All colors present in the outfit")
    pattern: str = Field(description="solid, striped, plaid, floral, etc.")
    sleeve_length: Optional[str] = Field(default=None, description="short, long, sleeveless, three-quarter")
    neckline: Optional[str] = Field(default=None, description="v-neck, crew, collar, etc.")
    fabric_type: Optional[str] = Field(default=None, description="Best guess: cotton, silk, denim, etc.")
    formality: str = Field(description="casual, smart-casual, formal, athletic")

# ─── Helpers ──────────────────────────────────
def base64_encode_image(image_path: str) -> str:
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")

# ─── Async extraction function ────────────────
async def analyze_image_async(
    client: AsyncOpenAI,
    image_path: str,
    prompt: str,
) -> ClothingAttributes:
    """Extract structured attributes from a single image."""
    b64 = base64_encode_image(image_path)

    completion = await client.beta.chat.completions.parse(
        model=GROK_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a fashion analysis system. Analyze the clothing in the image "
                    "and extract structured attributes. Be precise and factual."
                ),
            },
            {
                "role": "user",
                "content": [
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{b64}",
                            "detail": "high",
                        },
                    },
                    {"type": "text", "text": prompt},
                ],
            },
        ],
        response_format=ClothingAttributes,
    )
    return completion.choices[0].message.parsed

# ─── Batch processing with concurrency ────────
async def process_batch(
    image_paths: list[str],
    max_concurrent: int = 10,
) -> list[dict]:
    """Process many images concurrently with rate limiting."""
    client = AsyncOpenAI(
        api_key=XAI_API_KEY,
        base_url="https://api.x.ai/v1",
    )
    semaphore = asyncio.Semaphore(max_concurrent)
    results = []

    async def process_one(path: str) -> dict:
        async with semaphore:
            try:
                attrs = await analyze_image_async(
                    client, path, "Analyze the clothing in this image."
                )
                return {"path": path, "status": "ok", "attributes": attrs.model_dump()}
            except Exception as e:
                return {"path": path, "status": "error", "error": str(e)}

    tasks = [process_one(p) for p in image_paths]
    results = await asyncio.gather(*tasks)
    return results

# ─── Run ──────────────────────────────────────
image_dir = Path("data/images")
image_paths = sorted(str(p) for p in image_dir.glob("*.jpg"))[:250]

results = asyncio.run(process_batch(image_paths, max_concurrent=10))

succeeded = [r for r in results if r["status"] == "ok"]
failed = [r for r in results if r["status"] == "error"]
print(f"Processed: {len(succeeded)} ok, {len(failed)} errors")
```

## TypeScript — Structured Vision Extraction

```typescript
import OpenAI from "openai";
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const client = new OpenAI({
  apiKey: process.env.XAI_API_KEY,
  baseURL: "https://api.x.ai/v1",
});

const ClothingAttributes = z.object({
  category: z.string(),
  colors: z.array(z.string()),
  pattern: z.string(),
  formality: z.enum(["casual", "smart-casual", "formal", "athletic"]),
});

async function analyzeImage(imageUrl: string) {
  const response = await client.beta.chat.completions.parse({
    model: "grok-4-1-fast-reasoning",
    messages: [
      {
        role: "system",
        content: "Extract clothing attributes from the image.",
      },
      {
        role: "user",
        content: [
          { type: "image_url", image_url: { url: imageUrl, detail: "high" } },
          { type: "text", text: "Analyze the clothing." },
        ],
      },
    ],
    response_format: zodResponseFormat(ClothingAttributes, "ClothingAttributes"),
  });

  return response.choices[0].message.parsed;
}
```

## Key Gotchas

- **`store_messages=False`** — Set when sending images or requests may fail
- **`detail: "high"`** — Use for extraction tasks; `"low"` for quick classification
- **20 MiB max** per image, `jpg`/`jpeg`/`png` only
- **allOf not supported** in schemas — use `anyOf` instead
- **No min/max constraints** on strings or arrays — validate in application code
- **Two-phase is more reliable** for 5+ field schemas with nested objects
