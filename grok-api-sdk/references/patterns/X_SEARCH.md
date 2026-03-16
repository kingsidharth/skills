# X Search (Twitter Search)

Search X (formerly Twitter) for posts, users, threads. Supports keyword search, semantic search,
handle filtering, date ranges, and image/video understanding within posts.

**Cost:** $5 / 1,000 calls. Token costs billed separately.

## SDK Support

| SDK/API | Tool Name |
|---------|-----------|
| xAI SDK | `x_search` |
| OpenAI Responses API | `x_search` |
| Vercel AI SDK | `xai.tools.xSearch()` |

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `allowed_x_handles` | list[str] | Only posts from these handles (max 10) |
| `excluded_x_handles` | list[str] | Exclude posts from these handles (max 10) |
| `from_date` | ISO8601 str / datetime | Search start date |
| `to_date` | ISO8601 str / datetime | Search end date |
| `enable_image_understanding` | bool | Analyze images in posts |
| `enable_video_understanding` | bool | Analyze videos in posts (X Search only) |

**Cannot combine** `allowed_x_handles` and `excluded_x_handles` in the same request.

## Python SDK

### Basic

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user
from xai_sdk.tools import x_search

client = Client(api_key=os.getenv("XAI_API_KEY"))
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[x_search()],
)
chat.append(user("What are people saying about xAI on X?"))
response = chat.sample()
print(response.content)
print(response.citations)  # List of source URLs
```

### With handle filtering

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        x_search(allowed_x_handles=["elonmusk", "xai"]),
    ],
)
chat.append(user("What is the current status of xAI?"))
```

### With date range

```python
from datetime import datetime

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        x_search(
            from_date=datetime(2025, 10, 1),
            to_date=datetime(2025, 10, 10),
        ),
    ],
)
chat.append(user("What happened with AI regulation this week?"))
```

### With image + video understanding

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        x_search(
            enable_image_understanding=True,
            enable_video_understanding=True,
        ),
    ],
)
chat.append(user("Find recent posts with AI-generated images and describe what you see"))
```

### Streaming with tool call visibility

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[x_search()],
    include=["verbose_streaming"],
)
chat.append(user("What are people saying about xAI?"))

is_thinking = True
for response, chunk in chat.stream():
    for tool_call in chunk.tool_calls:
        print(f"\nTool: {tool_call.function.name}({tool_call.function.arguments})")
    if response.usage.reasoning_tokens and is_thinking:
        print(f"\rThinking... ({response.usage.reasoning_tokens} tokens)", end="", flush=True)
    if chunk.content and is_thinking:
        print("\n\nResponse:")
        is_thinking = False
    if chunk.content:
        print(chunk.content, end="", flush=True)
```

## TypeScript — OpenAI SDK (Responses API)

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.XAI_API_KEY,
  baseURL: "https://api.x.ai/v1",
  timeout: 360_000,
});

const response = await client.responses.create({
  model: "grok-4-1-fast-reasoning",
  tools: [
    {
      type: "x_search",
      x_search: {
        allowed_x_handles: ["elonmusk"],
        from_date: "2025-10-01",
        to_date: "2025-10-31",
      },
    },
  ],
  input: [{ role: "user", content: "What did Elon post about xAI in October?" }],
});

console.log(response.output_text);
```

## TypeScript — Vercel AI SDK

```typescript
import { xai } from "@ai-sdk/xai";
import { generateText } from "ai";

const result = await generateText({
  model: xai("grok-4-1-fast-reasoning"),
  tools: [xai.tools.xSearch({ allowed_x_handles: ["xai"] })],
  prompt: "What are the latest xAI announcements on X?",
});
```

## curl

```bash
curl https://api.x.ai/v1/responses \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -m 3600 \
  -d '{
    "model": "grok-4-1-fast-reasoning",
    "tools": [
      {
        "type": "x_search",
        "x_search": {
          "allowed_x_handles": ["elonmusk"],
          "enable_image_understanding": true
        }
      }
    ],
    "input": [{"role": "user", "content": "What did Elon post recently?"}]
  }'
```

## Combining with Web Search

```python
from xai_sdk.tools import web_search, x_search

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        web_search(),
        x_search(enable_image_understanding=True),
    ],
)
chat.append(user("Compare what the media reports vs what X users say about the new AI regulation"))
```

## Key Gotchas

- **Server-side tool** — Grok decides when and how to search. You cannot force a specific query.
- **$5/1k calls** — Plus token costs. The agent may make multiple search calls per request.
- **Not available in Batch API** — Server-side tools are excluded from batch processing.
- **Image/video tokens** — `view_image` and `view_x_video` are token-based only (no per-call fee).
- **Handle limit** — Max 10 handles in `allowed_x_handles` or `excluded_x_handles`.
