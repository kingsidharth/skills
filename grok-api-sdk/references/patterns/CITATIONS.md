# Citations

When using server-side tools (web_search, x_search), Grok returns two types of citation data:

1. **All Citations** — Complete list of all source URLs encountered (always returned)
2. **Inline Citations** — Markdown links embedded in response text at reference points

## All Citations (Default)

Always returned automatically on the response object. Includes every URL the agent visited,
even if not directly referenced in the final answer.

### Python SDK

```python
response = chat.sample()

# List of all source URLs
for url in response.citations:
    print(url)
```

Output:
```
https://x.com/i/status/1975607901571199086
https://x.ai/news
https://docs.x.ai/developers/release-notes
```

### TypeScript / REST

```typescript
const response = await client.responses.create({
  model: "grok-4-1-fast-reasoning",
  tools: [{ type: "web_search" }],
  input: [{ role: "user", content: "Latest xAI news" }],
});

// Citations are on the response object
console.log(response.citations);
```

## Inline Citations

Markdown-style links inserted directly in the response text: `[[N]](url)`.

- `N` = sequential number starting from 1
- Same source reuses the same number
- **Not guaranteed** — the model decides when to cite

### Enabling in Python SDK

```python
from xai_sdk.tools import web_search, x_search

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[web_search(), x_search()],
    include=["inline_citations"],  # ← Enable inline citations
)
chat.append(user("What is xAI?"))
response = chat.sample()

# Response text includes markdown citations
print(response.content)
# "xAI was founded in 2023...[[1]](https://x.ai/news/)[[2]](https://x.com/i/status/...)"
```

### Enabling in REST / OpenAI SDK

Inline citations are returned by default with the Responses API — no extra parameter needed.

```bash
curl https://api.x.ai/v1/responses \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "grok-4-1-fast-reasoning",
    "tools": [{"type": "web_search"}],
    "input": [{"role": "user", "content": "What is xAI?"}]
  }'
```

## Structured Inline Citation Data

Each citation has positional metadata for precise UI rendering.

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"url_citation"` |
| `url` | string | Source URL |
| `start_index` | int | Char position where `[[N]]` starts |
| `end_index` | int | Char position after closing `)` (exclusive) |
| `title` | string | Citation number (e.g. `"1"`) |

### Python SDK — Accessing structured data

```python
for citation in response.inline_citations:
    print(f"Citation [{citation.id}]:")
    print(f"  Position: {citation.start_index} to {citation.end_index}")

    if citation.HasField("web_citation"):
        print(f"  Web URL: {citation.web_citation.url}")
    elif citation.HasField("x_citation"):
        print(f"  X URL: {citation.x_citation.url}")
```

### Python — Extract citation text using position indices

```python
content = response.content

for citation in response.inline_citations:
    citation_text = content[citation.start_index:citation.end_index]
    print(f"Markdown: {citation_text}")
    # "[[1]](https://x.ai/news/)"
```

## Streaming with Inline Citations

Citations appear in real-time during streaming and accumulate on the final response:

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[web_search(), x_search()],
    include=["inline_citations"],
)
chat.append(user("Latest AI news"))

for response, chunk in chat.stream():
    # Markdown citation links appear in real-time
    if chunk.content:
        print(chunk.content, end="", flush=True)

    # Per-chunk citation events
    for citation in chunk.inline_citations:
        print(f"\n  [New citation: {citation.id}]")

# After streaming: all accumulated citations
print("\n\nAll citations:")
for citation in response.inline_citations:
    url = ""
    if citation.HasField("web_citation"):
        url = citation.web_citation.url
    elif citation.HasField("x_citation"):
        url = citation.x_citation.url
    print(f"  [{citation.id}] {url}")
```

## Rendering Citations in a UI

Strip inline markdown and build your own UI components:

```python
import re

content = response.content

# Extract all citation links
pattern = r'\[\[(\d+)\]\]\((https?://[^\)]+)\)'
citations_found = re.findall(pattern, content)

# Remove inline citations from text for clean display
clean_text = re.sub(pattern, '', content)

# Build footnotes
footnotes = {num: url for num, url in citations_found}
print(clean_text)
print("\nSources:")
for num, url in sorted(footnotes.items()):
    print(f"  [{num}] {url}")
```

## Key Gotchas

- **All citations always returned** — no config needed, just `response.citations`
- **Inline citations opt-in** for xAI SDK via `include=["inline_citations"]`; default in Responses API
- **Not every source gets cited inline** — the model decides relevance
- **Position indices** — follow Python slice convention (`start_index` inclusive, `end_index` exclusive)
- **Two citation types** — `web_citation` (from web_search) and `x_citation` (from x_search)
- **Reused numbers** — if the same source is cited again, the original number is reused
