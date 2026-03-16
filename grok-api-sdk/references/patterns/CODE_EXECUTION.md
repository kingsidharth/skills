# Code Execution Tool

Server-side sandboxed Python execution. Grok writes and runs code in real-time for precise
calculations, data analysis, statistical computing, and visualization.

**Cost:** $5 / 1,000 calls. Token costs billed separately.

## SDK Support

| SDK/API | Tool Name |
|---------|-----------|
| xAI SDK | `code_execution` |
| OpenAI Responses API | `code_interpreter` |
| Vercel AI SDK | `xai.tools.codeExecution()` |

## Python SDK

### Basic calculation

```python
import os
from xai_sdk import Client
from xai_sdk.chat import user
from xai_sdk.tools import code_execution

client = Client(api_key=os.getenv("XAI_API_KEY"))
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[code_execution()],
)
chat.append(user("Calculate compound interest for $10,000 at 5% annually for 10 years"))
response = chat.sample()
print(response.content)
```

### With custom pip packages

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        code_execution(
            pip_packages=["numpy", "pandas", "scipy", "matplotlib"],
        ),
    ],
)
```

### Multi-turn data analysis

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[code_execution()],
    include=["verbose_streaming"],
)

# Turn 1: Initial analysis
chat.append(user("""
Sales data for Q1-Q4: [120000, 135000, 98000, 156000].
Analyze: quarterly trends, growth rates, statistical summary.
"""))

for response, chunk in chat.stream():
    for tool_call in chunk.tool_calls:
        print(f"\nCode: {tool_call.function.name}({tool_call.function.arguments})")
    if chunk.content:
        print(chunk.content, end="", flush=True)

# Turn 2: Follow-up (preserves code context via conversation state)
chat.append(response)
chat.append(user("Now predict Q1 next year using linear regression"))

for response, chunk in chat.stream():
    if chunk.content:
        print(chunk.content, end="", flush=True)
```

### Streaming with verbose tool visibility

```python
chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[code_execution()],
    include=["verbose_streaming"],
)
chat.append(user("Solve: dy/dx = x^2 + y, y(0) = 1, find y(2)"))

is_thinking = True
for response, chunk in chat.stream():
    for tool_call in chunk.tool_calls:
        print(f"\nTool: {tool_call.function.name}")
        print(f"  Args: {tool_call.function.arguments}")
    if response.usage.reasoning_tokens and is_thinking:
        print(f"\rThinking... ({response.usage.reasoning_tokens} tokens)", end="", flush=True)
    if chunk.content and is_thinking:
        print("\n\nResult:")
        is_thinking = False
    if chunk.content:
        print(chunk.content, end="", flush=True)

print(f"\n\nUsage: {response.usage}")
print(f"Tool usage: {response.server_side_tool_usage}")
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
      type: "code_interpreter",
      code_interpreter: {
        container: {
          pip_packages: ["numpy", "pandas", "scipy"],
        },
      },
    },
  ],
  input: [
    {
      role: "user",
      content:
        "Calculate the Sharpe ratio for portfolio returns [0.12, 0.08, -0.03, 0.15] with risk-free rate 0.02",
    },
  ],
});

console.log(response.output_text);
```

## TypeScript — Vercel AI SDK

```typescript
import { xai } from "@ai-sdk/xai";
import { generateText } from "ai";

const result = await generateText({
  model: xai("grok-4-1-fast-reasoning"),
  tools: [
    xai.tools.codeExecution({
      container: { pip_packages: ["numpy", "pandas"] },
    }),
  ],
  prompt: "Perform a t-test: Group A [23,25,28,30], Group B [20,22,24,26]",
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
        "type": "code_interpreter",
        "code_interpreter": {
          "container": {
            "pip_packages": ["numpy", "pandas", "scipy"]
          }
        }
      }
    ],
    "input": [
      {"role": "user", "content": "Calculate compound interest for $10k at 5% for 10 years"}
    ]
  }'
```

## Combining with Other Tools

```python
from xai_sdk.tools import web_search, x_search, code_execution

chat = client.chat.create(
    model="grok-4-1-fast-reasoning",
    tools=[
        web_search(),
        x_search(),
        code_execution(pip_packages=["numpy", "pandas", "matplotlib"]),
    ],
)
chat.append(user(
    "Search for the latest S&P 500 quarterly returns, then calculate the Sharpe ratio "
    "and create a visualization"
))
```

## Common Use Cases

### Financial analysis
```python
"Calculate the Sharpe ratio for portfolio returns [0.12, 0.08, -0.03, 0.15] with risk-free rate 0.02"
```

### Statistical testing
```python
"Perform a t-test comparing Group A [23,25,28,30] and Group B [20,22,24,26], interpret the p-value"
```

### Scientific computing
```python
"Solve this ODE using numerical methods: dy/dx = x^2 + y, y(0) = 1"
```

### Data processing
```python
"""
CSV data with columns date, revenue, costs:
[['2024-01', 50000, 35000], ['2024-02', 55000, 38000], ...]
Calculate monthly profit margins and identify the best month.
"""
```

## Execution Environment

- **Language:** Python only
- **Pre-installed:** numpy, pandas, matplotlib, scipy, and other common packages
- **Custom packages:** Specify via `pip_packages` parameter
- **Sandboxed:** No network access, limited filesystem
- **Stateless:** Context doesn't persist across requests (use conversation chaining for multi-turn)
- **Time/memory limits:** Complex computations may hit constraints

## Best Practices

- **Be specific** — "Calculate correlation matrix and highlight values > 0.7" beats "analyze this data"
- **Provide data inline** — Pass data directly in the prompt; no file upload to sandbox
- **Low temperature** — Use 0.0–0.3 for calculations to avoid creative interpretation
- **Use reasoning models** — `grok-4-1-fast-reasoning` generates better code than non-reasoning variants

## Key Gotchas

- **Not available in Batch API** — Server-side tools are excluded from batch processing
- **Tool name differs by SDK** — `code_execution` (xAI SDK) vs `code_interpreter` (OpenAI/REST)
- **$5/1k calls** — Agent may make multiple code execution calls per request
- **No file I/O** — Can't read uploaded files; pass data in the prompt text
- **Visualizations** — Matplotlib plots are generated but delivery depends on SDK (may be base64 images)
