# Python SDK Reference (xai-sdk)

## Installation & Setup

```python
pip install xai-sdk
```

```python
import os
from xai_sdk import Client, AsyncClient
from xai_sdk.chat import user, system, image, tool, tool_result
from xai_sdk.tools import web_search, x_search

client = Client(api_key=os.getenv("XAI_API_KEY"), timeout=3600)
```

## Basic Text Generation

```python
chat = client.chat.create(model="grok-4-1-fast-reasoning")
chat.append(system("You are a helpful assistant."))
chat.append(user("Explain quantum computing in one paragraph."))
response = chat.sample()
print(response.content)
print(response.id)  # For conversation chaining
```

## Conversation Chaining (Stateful)

```python
# First turn
chat = client.chat.create(model="grok-4-1-fast-reasoning", store_messages=True)
chat.append(system("You are a helpful assistant."))
chat.append(user("What is 2+2?"))
response = chat.sample()

# Continue — no need to resend history
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    previous_response_id=response.id,
    store_messages=True,
)
chat.append(user("Now multiply that by 10"))
response2 = chat.sample()
```

## Encrypted Reasoning (Grok 4)

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    use_encrypted_content=True,
    store_messages=True,
)
chat.append(system("You are a helpful assistant."))
chat.append(user("Solve this step by step: 17 * 23"))
response = chat.sample()

# Pass encrypted thinking to next turn
chat.append(response)  # SDK auto-adds encrypted content
chat.append(user("Now add 100 to that result"))
response2 = chat.sample()
```

## Image Understanding (Vision)

```python
from xai_sdk.chat import user, image

chat = client.chat.create(model="grok-4-1-fast-reasoning")
chat.append(
    user(
        "Describe what you see in this image.",
        image(image_url="https://example.com/photo.jpg", detail="high"),
    )
)
response = chat.sample()
```

### Base64 image from file

```python
import base64
from pathlib import Path

def load_image_b64(path: str) -> str:
    data = Path(path).read_bytes()
    b64 = base64.b64encode(data).decode("utf-8")
    ext = Path(path).suffix.lower()
    mime = "image/jpeg" if ext in (".jpg", ".jpeg") else "image/png"
    return f"data:{mime};base64,{b64}"

chat.append(
    user("Analyze this.", image(image_url=load_image_b64("photo.jpg"), detail="high"))
)
```

## Structured Outputs — parse()

```python
from pydantic import BaseModel, Field

class Sentiment(BaseModel):
    label: str = Field(description="positive, negative, or neutral")
    confidence: float = Field(description="0.0 to 1.0", ge=0, le=1)
    reasoning: str = Field(description="Brief explanation")

chat = client.chat.create(model="grok-4")
chat.append(system("Analyze sentiment of the given text."))
chat.append(user("This product is absolutely fantastic!"))

response, result = chat.parse(Sentiment)
# result is a Sentiment instance
print(result.label, result.confidence)
```

## Structured Outputs — response_format + sample()

```python
chat = client.chat.create(
    model="grok-4",
    response_format=Sentiment,
)
chat.append(user("This product is terrible."))
response = chat.sample()
result = Sentiment.model_validate_json(response.content)
```

## Structured Outputs — Streaming

```python
chat = client.chat.create(model="grok-4", response_format=Sentiment)
chat.append(user("This is okay I guess."))

for response, chunk in chat.stream():
    print(chunk.content, end="", flush=True)

result = Sentiment.model_validate_json(response.content)
```

## Image Generation

```python
import xai_sdk

client = xai_sdk.Client()

# Basic generation
response = client.image.sample(
    prompt="A mountain landscape at sunset",
    model="grok-imagine-image",
    aspect_ratio="16:9",
    resolution="2k",
)
print(response.url)  # Temporary URL — download immediately

# Image editing
response = client.image.sample(
    prompt="Change the sky to a starry night",
    model="grok-imagine-image",
    image_url="https://example.com/landscape.jpg",
)

# Multiple images from same prompt
responses = client.image.sample_batch(
    prompt="A futuristic cityscape", model="grok-imagine-image", n=4,
)

# Base64 output
response = client.image.sample(
    prompt="A garden", model="grok-imagine-image", image_format="base64",
)
with open("out.jpg", "wb") as f:
    f.write(response.image)
```

## Async Client

```python
import asyncio
from xai_sdk import AsyncClient
from xai_sdk.chat import user

async def main():
    client = AsyncClient(api_key=os.getenv("XAI_API_KEY"), timeout=3600)
    semaphore = asyncio.Semaphore(5)  # Max 5 concurrent

    async def process(prompt: str):
        async with semaphore:
            chat = client.chat.create(model="grok-4-1-fast-reasoning", max_tokens=200)
            chat.append(user(prompt))
            return await chat.sample()

    tasks = [process(p) for p in ["joke", "haiku", "fact"]]
    responses = await asyncio.gather(*tasks)
    for r in responses:
        print(r.content)

asyncio.run(main())
```

## Function Calling

```python
import json
from xai_sdk.chat import user, tool, tool_result

tools = [
    tool(
        name="get_weather",
        description="Get current weather for a location",
        parameters={
            "type": "object",
            "properties": {
                "location": {"type": "string"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
            },
            "required": ["location"],
        },
    ),
]

chat = client.chat.create(model="grok-4-1-fast-reasoning", tools=tools)
chat.append(user("What's the weather in Tokyo?"))
response = chat.sample()

if response.tool_calls:
    chat.append(response)
    for tc in response.tool_calls:
        args = json.loads(tc.function.arguments)
        result = {"temperature": 22, "unit": "celsius"}  # Your function
        chat.append(tool_result(json.dumps(result)))
    response = chat.sample()

print(response.content)
```

## Server-Side Tools (Agentic)

```python
from xai_sdk.tools import web_search, x_search

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[web_search(), x_search()],
)
chat.append(user("What's the latest news about AI regulation?"))
response = chat.sample()
print(response.content)
```

## Retrieve / Delete Stored Response

```python
# Retrieve
response = client.chat.get_stored_completion("<response_id>")

# Delete
client.chat.delete_stored_completion("<response_id>")
```

## Usage Tracking

```python
response = chat.sample()
print(f"Input tokens: {response.usage.prompt_tokens}")
print(f"Output tokens: {response.usage.completion_tokens}")
print(f"Reasoning tokens: {response.usage.reasoning_tokens}")
print(f"Total tokens: {response.usage.total_tokens}")
```
