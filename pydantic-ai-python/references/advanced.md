# Advanced Features

## Multimodal input

Pass images, audio, video, or documents alongside text:

```python
from pydantic_ai import Agent, ImageUrl, DocumentUrl, BinaryContent

agent = Agent('anthropic:claude-sonnet-4-6')

# URL-based image
result = await agent.run([
    'Describe this image',
    ImageUrl(url='https://example.com/photo.jpg'),
])

# Base64 binary
result = await agent.run([
    'What is in this image?',
    BinaryContent(data=b'...', media_type='image/png'),
])

# Document (PDF)
result = await agent.run([
    'Summarize this document',
    DocumentUrl(url='https://example.com/doc.pdf'),
])
```

Audio and video supported where the provider allows.

## Thinking (extended reasoning)

Enable via capability or model settings:
```python
from pydantic_ai.capabilities import Thinking

agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[Thinking()])
```

Thinking content appears in message history. Budget controllable via model settings where supported.

## Pydantic Graph

Type-safe graph execution for complex workflows where standard control flow becomes spaghetti.

```python
from pydantic_graph import Graph, GraphRunContext
from dataclasses import dataclass

@dataclass
class MyState:
    counter: int = 0

# Define nodes, edges, and run the graph
# Nodes are typed; graph validates structure at definition time
```

Graphs support persistence (checkpointing), Mermaid diagram generation, and custom state types.

Separate package: `pydantic-graph`.

## Evals (pydantic-evals)

Systematic testing of agent behavior:

```python
from pydantic_evals import Case, Dataset

dataset = Dataset(
    cases=[
        Case(
            name='capital',
            inputs='What is the capital of France?',
            expected_output='Paris',
        ),
    ]
)

async def task(inputs: str) -> str:
    result = await agent.run(inputs)
    return result.output

report = await dataset.evaluate(task)
report.print()
```

Built-in evaluators: exact match, contains, LLM judge, custom. Logfire integration for tracking over time.

Separate package: `pydantic-evals`.

## Durable execution

Survive API failures, app restarts, and long-running workflows:

- **Temporal** — `pydantic_ai.durable_exec.temporal`
- **DBOS** — `pydantic_ai.durable_exec.dbos`
- **Prefect** — `pydantic_ai.durable_exec.prefect`
- **Restate** — `pydantic_ai.durable_exec.restate`

Each wraps the agent run in the respective orchestrator's durable execution model.

## Embeddings

Multi-provider embedding support:

```python
from pydantic_ai.embeddings import EmbeddingModel

model = EmbeddingModel('openai:text-embedding-3-small')
vectors = await model.embed(['hello', 'world'])
```

## Extensibility

Build reusable capability packages. Capabilities bundle tools, hooks, instructions, and model settings into pip-installable units.

## Harness

Pre-built capability library for common agent patterns (code execution, file operations, web browsing). Import from `pydantic_ai.harness`.

Code mode: agents that can write and execute code in a sandboxed environment.

## Observability

OpenTelemetry-based via Pydantic Logfire:
```python
import logfire
logfire.configure()
logfire.instrument_pydantic_ai()
```

Works with any OTel-compatible backend (Datadog, Jaeger, etc.).

## Message history format

Messages are typed dataclasses: `ModelRequest`, `ModelResponse`, `ToolCallPart`, `ToolReturnPart`, `TextPart`, `ThinkingPart`.

Access via `result.all_messages()` or `result.new_messages()`.

## UI event streams

Stream agent events to frontend UIs:
- **AG-UI protocol** — `pydantic_ai.ag_ui`
- **Vercel AI SDK** — `pydantic_ai.ui.vercel_ai`

## Agent2Agent (A2A)

Interoperate with agents from other frameworks via the A2A protocol. Expose PydanticAI agents as A2A servers with `fasta2a`.
