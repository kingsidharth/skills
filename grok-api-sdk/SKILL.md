---
name: grok-api-sdk
description: "Build applications using the xAI Grok API. Use when the user needs to call Grok models via API — text generation, reasoning, vision/image understanding, image generation, structured outputs, batch processing, streaming, function calling, or any xAI SDK integration. Covers both the Python xAI SDK (xai-sdk) and REST/TypeScript/JavaScript (OpenAI-compatible). Triggers on mentions of 'Grok API', 'xAI API', 'xai-sdk', 'grok-4', 'grok-3', or requests to integrate Grok into applications."
---

# xAI Grok API & SDK Skill

## Quick Reference

| Need | Reference |
|------|-----------|
| Python SDK snippets | [PYTHON.md](references/PYTHON.md) |
| TypeScript / REST snippets | [TYPESCRIPT.md](references/TYPESCRIPT.md) |
| Vision + structured extraction | [VISION_STRUCTURED.md](references/patterns/VISION_STRUCTURED.md) |
| Batch processing | [BATCH.md](references/patterns/BATCH.md) |
| Bounding box / object detection | [BOUNDING_BOXES.md](references/patterns/BOUNDING_BOXES.md) |
| X (Twitter) search | [X_SEARCH.md](references/patterns/X_SEARCH.md) |
| Citations (all + inline) | [CITATIONS.md](references/patterns/CITATIONS.md) |
| Code execution (sandboxed Python) | [CODE_EXECUTION.md](references/patterns/CODE_EXECUTION.md) |

## Connection Setup

**Base URL:** `https://api.x.ai`
**Auth header:** `Authorization: Bearer <XAI_API_KEY>`
**Python SDK:** `pip install xai-sdk` — uses gRPC, covers all features
**OpenAI SDK compatible:** set `base_url="https://api.x.ai/v1"`
**JS/TS AI SDK:** `@ai-sdk/xai` via Vercel AI SDK

## Model Selection

| Model | Type | Use when |
|-------|------|----------|
| `grok-4` | Reasoning (always-on) | Highest quality, complex tasks |
| `grok-4-1-fast-reasoning` | Reasoning, fast | Agentic tool calling, fast + cheap |
| `grok-4-1-fast-non-reasoning` | Non-reasoning, fast | Instant responses, lowest latency |
| `grok-3-mini` | Reasoning (configurable) | Budget tasks, `reasoning_effort` control |
| `grok-imagine-image` | Image generation | Text-to-image, image editing |

**Knowledge cutoff:** November 2024. Use web_search/x_search tools for current data.

## Critical Rules

1. **Reasoning models reject these params** (will error): `presencePenalty`, `frequencyPenalty`, `stop`
2. **`reasoning_effort`** — ONLY `grok-3-mini` supports it (`"low"` / `"high"`). Grok 4 errors.
3. **Encrypted reasoning** — For grok-4: set `include: ["reasoning.encrypted_content"]` in REST, or `use_encrypted_content=True` in SDK. Without this you get no reasoning traces.
4. **Image understanding + storage** — Set `store_messages=False` (SDK) or `"store": false` (REST) when sending images, otherwise requests may fail.
5. **Only one system message** — must be first. `developer` role is alias for `system`.
6. **Structured outputs** — `allOf` not supported. No `minLength`/`maxLength` on strings. No `minItems`/`maxItems` on arrays.
7. **Batch API** — 50% off pricing. No server-side tools, no function calling, no image/video generation models. Text/language models only (vision input IS supported).
8. **Image input limits** — 20 MiB max, `jpg`/`jpeg`/`png` only, no limit on count.
9. **Responses API** is preferred over Chat Completions. Supports stateful conversations (30-day retention).
10. **Timeout** — Always set `timeout=3600` for reasoning models. They can take minutes.

## Responses API (Preferred)

The Responses API supports **stateful conversations** — history stored server-side for 30 days.

**Conversation chaining:** Pass `previous_response_id` instead of resending full history. You're still billed for full context, but cached tokens reduce cost.

**Disable storage:** `store_messages=False` (SDK) or `"store": false` (REST).

See [PYTHON.md](references/PYTHON.md) or [TYPESCRIPT.md](references/TYPESCRIPT.md) for code.

## Structured Outputs

Guarantees model output matches your JSON schema. Supported by all language models.

**Two approaches in Python SDK:**
- `chat.parse(PydanticModel)` → returns `(Response, parsed_model)` — auto parsing
- `response_format=PydanticModel` + `chat.sample()` → manual `Model.model_validate_json(response.content)`

**REST/OpenAI SDK:** Use `response_format: { type: "json_schema", json_schema: { name: "...", schema: {...} } }`

**Supported types:** string, number, integer, boolean, object, array, enum, anyOf.
**NOT supported:** `allOf`, `minLength`/`maxLength`, `minItems`/`maxItems`.

**With tools** — works on Grok 4 family only (grok-4-1-fast, grok-4-fast, non-reasoning variants).

See [VISION_STRUCTURED.md](references/patterns/VISION_STRUCTURED.md) for vision + structured output patterns.

## Image Understanding (Vision)

Send images as input to language models. Images go in user message content as `input_image` blocks.

**SDK:** Use `image()` helper — `user("prompt text", image(image_url=url, detail="high"))`
**REST:** Content array with `{"type": "input_image", "image_url": "...", "detail": "high"}`

Accepts both URLs and `data:image/jpeg;base64,...` data URIs.

## Image Generation

Model: `grok-imagine-image`. Flat per-image pricing. Max 10 per request.

**SDK:** `client.image.sample(prompt=..., model="grok-imagine-image")`
**Editing:** Pass `image_url=` param with source image.
**Batch gen:** `client.image.sample_batch(prompt=..., n=4)`

URLs are temporary — download immediately. OpenAI SDK `images.edit()` NOT supported (use xAI SDK or direct HTTP).

## Batch API

50% off all token types. No rate limit impact. Typically completes within 24 hours.

**Workflow:** Create batch → Add requests → Poll status → Retrieve results

**Limits:** 1 batch/sec creation rate, 100 add-requests calls per 30 seconds, 25MB per request payload, >1M requests may throttle.

**NOT supported in batch:** Server-side tools, function calling, image/video generation.

See [BATCH.md](references/patterns/BATCH.md) for complete examples.

## Tools Pricing

| Tool | Cost / 1k calls |
|------|----------------|
| web_search | $5 |
| x_search | $5 |
| code_execution | $5 |
| attachment_search | $10 |
| collections_search | $2.50 |
| view_image / view_x_video / MCP | Token-based |

## Server-Side Tools (Agentic)

Grok can autonomously use server-side tools. The model decides when and how to call them — you just enable them.

| Tool | SDK Name | REST Name | Cost | Notes |
|------|----------|-----------|------|-------|
| Web Search | `web_search()` | `web_search` | $5/1k calls | Internet search + page browsing |
| X Search | `x_search()` | `x_search` | $5/1k calls | Posts, users, threads on X/Twitter |
| Code Execution | `code_execution()` | `code_interpreter` | $5/1k calls | Sandboxed Python execution |
| Collections Search | `collections_search()` | `collections_search` | $2.50/1k calls | RAG over uploaded docs |
| View Image | — | — | Token-based | Image analysis within search results |
| View X Video | — | — | Token-based | Video analysis within X posts |
| Remote MCP | — | `mcp` | Token-based | Custom MCP servers |

**Key rules:**
- Tools can be **combined** — pass multiple in the `tools` list
- **Not available in Batch API** — server-side tools are excluded
- Agent may make **multiple calls per request** — costs scale with complexity
- **Citations** — automatically returned when using search tools. See [CITATIONS.md](references/patterns/CITATIONS.md).

**Quick example (Python SDK):**
```python
from xai_sdk.tools import web_search, x_search, code_execution

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[web_search(), x_search(), code_execution()],
)
```

See pattern files for full examples: [X_SEARCH.md](references/patterns/X_SEARCH.md), [CODE_EXECUTION.md](references/patterns/CODE_EXECUTION.md), [CITATIONS.md](references/patterns/CITATIONS.md).

## REST Endpoint Quick Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/responses` | POST | Create response (preferred) |
| `/v1/responses/{id}` | GET/DELETE | Retrieve/delete stored response |
| `/v1/chat/completions` | POST | Legacy chat completions |
| `/v1/images/generations` | POST | Generate images |
| `/v1/images/edits` | POST | Edit images |
| `/v1/batches` | POST/GET | Create/list batches |
| `/v1/batches/{id}` | GET | Get batch status |
| `/v1/batches/{id}/requests` | GET/POST | List/add batch requests |
| `/v1/batches/{id}/results` | GET | Get batch results |
| `/v1/batches/{id}:cancel` | POST | Cancel batch |
| `/v1/models` | GET | List models |
