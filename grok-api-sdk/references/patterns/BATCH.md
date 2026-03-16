# Batch API

50% off all token types. No rate limit impact. Completes within 24 hours typically.

**Supports:** Text/language models, vision input, structured outputs.
**Does NOT support:** Server-side tools, function calling, image/video generation models.

> **Note:** xAI Batch API is different from OpenAI's. No JSONL upload — you POST requests directly.

## Python SDK — Complete Workflow

```python
import os
import time
import json
from pydantic import BaseModel, Field
from typing import Optional
from xai_sdk import Client
from xai_sdk.chat import system, user, image

client = Client(api_key=os.getenv("XAI_API_KEY"), timeout=3600)

# ─── 1. Define schema ────────────────────────
class ImageAnalysis(BaseModel):
    description: str = Field(description="1-2 sentence visual description")
    category: str = Field(description="Product category")
    dominant_colors: list[str] = Field(description="Top 3 colors")
    confidence: float = Field(description="0.0-1.0", ge=0.0, le=1.0)

# ─── 2. Create batch ─────────────────────────
batch = client.batch.create(batch_name="image_analysis_batch")
print(f"Batch ID: {batch.batch_id}")

# ─── 3. Build & add requests ─────────────────
items = [
    {"id": "img_001", "url": "https://example.com/shoe.jpg"},
    {"id": "img_002", "url": "https://example.com/laptop.jpg"},
    {"id": "img_003", "url": "https://example.com/mug.jpg"},
]

batch_requests = []
for item in items:
    chat = client.chat.create(
        model="grok-4-1-fast-reasoning",
        batch_request_id=item["id"],
        response_format=ImageAnalysis,         # ← Schema attached per request
    )
    chat.append(system("Analyze the product image and extract structured metadata."))
    chat.append(user(
        "Analyze this product image.",
        image(image_url=item["url"], detail="high"),
    ))
    batch_requests.append(chat)

client.batch.add(batch_id=batch.batch_id, batch_requests=batch_requests)
print(f"Added {len(batch_requests)} requests")

# ─── 4. Poll status ──────────────────────────
while True:
    status = client.batch.get(batch_id=batch.batch_id)
    s = status.state
    print(f"Pending: {s.num_pending} | Done: {s.num_success} | Error: {s.num_error}")
    if s.num_pending == 0:
        break
    time.sleep(15)

# ─── 5. Retrieve results ─────────────────────
all_results, all_errors = [], []
token = None
while True:
    page = client.batch.list_batch_results(
        batch_id=batch.batch_id, limit=100, pagination_token=token,
    )
    all_results.extend(page.succeeded)
    all_errors.extend(page.failed)
    if page.pagination_token is None:
        break
    token = page.pagination_token

# ─── 6. Parse structured responses ───────────
for result in all_results:
    analysis = ImageAnalysis.model_validate_json(result.response.content)
    print(f"[{result.batch_request_id}] {analysis.category}: {analysis.description}")
    print(f"  Colors: {', '.join(analysis.dominant_colors)}")
    print(f"  Tokens: {result.response.usage.total_tokens}")

for err in all_errors:
    print(f"[{err.batch_request_id}] FAILED: {err.error_message}")
```

## REST API — Equivalent Workflow

### 1. Create batch

```bash
curl -X POST https://api.x.ai/v1/batches \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "image_analysis_batch"}'
```

### 2. Add requests with images + schema

```bash
curl -X POST https://api.x.ai/v1/batches/{batch_id}/requests \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "requests": [
      {
        "batch_request_id": "img_001",
        "model": "grok-4-1-fast-reasoning",
        "response_format": {
          "type": "json_schema",
          "json_schema": {
            "name": "ImageAnalysis",
            "schema": {
              "type": "object",
              "properties": {
                "description": {"type": "string"},
                "category": {"type": "string"},
                "dominant_colors": {"type": "array", "items": {"type": "string"}},
                "confidence": {"type": "number"}
              },
              "required": ["description", "category", "dominant_colors", "confidence"],
              "additionalProperties": false
            }
          }
        },
        "messages": [
          {
            "role": "system",
            "content": "Analyze the product image and extract structured metadata."
          },
          {
            "role": "user",
            "content": [
              {"type": "input_image", "image_url": "https://example.com/shoe.jpg", "detail": "high"},
              {"type": "input_text", "text": "Analyze this product image."}
            ]
          }
        ]
      }
    ]
  }'
```

### 3. Poll & get results

```bash
# Check status
curl https://api.x.ai/v1/batches/{batch_id} \
  -H "Authorization: Bearer $XAI_API_KEY"

# Get results
curl https://api.x.ai/v1/batches/{batch_id}/results \
  -H "Authorization: Bearer $XAI_API_KEY"
```

## TypeScript — Batch Workflow

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.XAI_API_KEY,
  baseURL: "https://api.x.ai/v1",
});
const BASE = "https://api.x.ai";
const headers = {
  Authorization: `Bearer ${process.env.XAI_API_KEY}`,
  "Content-Type": "application/json",
};

const schema = {
  type: "json_schema" as const,
  json_schema: {
    name: "ImageAnalysis",
    schema: {
      type: "object",
      properties: {
        description: { type: "string" },
        category: { type: "string" },
        dominant_colors: { type: "array", items: { type: "string" } },
        confidence: { type: "number" },
      },
      required: ["description", "category", "dominant_colors", "confidence"],
      additionalProperties: false,
    },
  },
};

// 1. Create batch
const batchRes = await fetch(`${BASE}/v1/batches`, {
  method: "POST", headers,
  body: JSON.stringify({ name: "ts_batch" }),
});
const { batch_id: batchId } = await batchRes.json();

// 2. Add requests
const imageUrls = ["https://example.com/a.jpg", "https://example.com/b.jpg"];
const requests = imageUrls.map((url, i) => ({
  batch_request_id: `img_${i}`,
  model: "grok-4-1-fast-reasoning",
  response_format: schema,
  messages: [
    { role: "system", content: "Analyze the product image." },
    {
      role: "user",
      content: [
        { type: "input_image", image_url: url, detail: "high" },
        { type: "input_text", text: "Extract product metadata." },
      ],
    },
  ],
}));

await fetch(`${BASE}/v1/batches/${batchId}/requests`, {
  method: "POST", headers,
  body: JSON.stringify({ requests }),
});

// 3. Poll
let pending = true;
while (pending) {
  const s = await (await fetch(`${BASE}/v1/batches/${batchId}`, { headers })).json();
  console.log(`${s.state.num_success}/${s.state.num_requests} done`);
  pending = s.state.num_pending > 0;
  if (pending) await new Promise((r) => setTimeout(r, 10_000));
}

// 4. Results
const results = await (await fetch(`${BASE}/v1/batches/${batchId}/results`, { headers })).json();
for (const r of results.succeeded) {
  const data = JSON.parse(r.response.content);
  console.log(`[${r.batch_request_id}]`, data.category, data.dominant_colors);
}
```

## Rate Limits

| Operation | Limit |
|-----------|-------|
| Create batch | 1/sec |
| Add requests | 100 calls/30sec |
| Request payload | 25 MB max |
| Large batches (>1M requests) | May be throttled |

## Text-Only Batch (No Vision)

Same pattern, just omit the image content block:

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    batch_request_id="text_001",
    response_format=SentimentResult,
)
chat.append(system("Classify sentiment."))
chat.append(user("This product is amazing!"))
```

## Key Gotchas

- **Not JSONL** — Unlike OpenAI, xAI uses direct POST requests (not file upload)
- **Pagination** — Results may be paginated; always loop with `pagination_token`
- **Schema per request** — Each batch request can have its own `response_format`
- **Cancel** — `POST /v1/batches/{id}:cancel` (note the colon syntax)
- **Cost tracking** — `batch.cost_breakdown.total_cost_usd_ticks / 1e10` for USD cost
