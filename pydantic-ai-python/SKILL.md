---
name: pydantic-ai-python
description: Build AI agents with PydanticAI — tools, structured output, multi-provider, streaming, multi-agent delegation, MCP, capabilities, evals, graphs. Use when working with pydantic_ai, pydantic-ai, Agent, RunContext, tool decorator, output_type, deps_type, capabilities, or any PydanticAI agent pattern in Python.
---

# PydanticAI

Python agent framework by the Pydantic team. Model-agnostic, type-safe, dependency-injected.

## Installation

```bash
uv add pydantic-ai
# Provider extras:
uv add 'pydantic-ai[openai]'    # or anthropic, google, groq, mistral, cohere, bedrock, huggingface
uv add 'pydantic-ai[logfire]'   # observability
```

Python 3.9+. Requires pydantic v2.

## Quick start

```python
from pydantic_ai import Agent

agent = Agent('anthropic:claude-sonnet-4-6', instructions='Be concise.')
result = agent.run_sync('Hello')
print(result.output)
```

## Architecture at a glance

An `Agent` bundles: instructions, tools/toolsets, output type, dependency type, model, model settings, and capabilities.

Agents are **generic**: `Agent[DepsType, OutputType]`. They're designed for reuse — instantiate globally like a FastAPI app.

## Running agents

Five modes: `run()` (async), `run_sync()`, `run_stream()` / `run_stream_sync()`, `run_stream_events()`, `iter()` (graph node iteration).

All return `result.output` typed to your `output_type`.

## Reference files

- [agents-and-dependencies.md](references/agents-and-dependencies.md) — Agent construction, instructions, deps, run modes, conversations
- [tools-and-toolsets.md](references/tools-and-toolsets.md) — Function tools, toolsets, deferred tools, built-in tools, MCP, common tools
- [output-and-streaming.md](references/output-and-streaming.md) — Structured output, streaming, validation, output functions
- [models-and-providers.md](references/models-and-providers.md) — Provider strings, model settings, fallback, custom models
- [capabilities-and-hooks.md](references/capabilities-and-hooks.md) — Composable capabilities, hooks, agent specs (YAML/JSON)
- [multi-agent.md](references/multi-agent.md) — Delegation, handoff, programmatic multi-agent patterns
- [advanced.md](references/advanced.md) — Multimodal input, thinking, graphs, evals, durable execution, embeddings, extensibility, harness
