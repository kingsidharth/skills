# Capabilities and Hooks

## Capabilities

Reusable bundles of tools, hooks, instructions, and model settings. Compose agent behavior from pluggable units.

```python
from pydantic_ai import Agent
from pydantic_ai.capabilities import Thinking, WebSearch, MCP

agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[Thinking(), WebSearch(), MCP(server_url='...')],
)
```

### Built-in capabilities

- `Thinking()` — enables extended thinking / chain-of-thought (provider-dependent)
- `WebSearch()` — provider-adaptive web search (native when available, falls back to local)
- `MCP(...)` — provider-adaptive MCP tool access
- `WebFetch()` — provider-adaptive URL fetching
- `ImageGeneration()` — provider-adaptive image generation

### Custom capabilities

Subclass `Capability` to bundle tools, hooks, instructions, and settings:

```python
from pydantic_ai.capabilities import Capability

class MyCapability(Capability):
    def get_tools(self): ...
    def get_instructions(self): ...
    def get_hooks(self): ...
    def get_model_settings(self): ...
```

## Hooks

Lifecycle callbacks on agent runs. Register via decorator or `hooks=` kwarg.

Hook points:
- `@agent.on_model_request` — before each model request
- `@agent.on_model_response` — after each model response
- `@agent.on_tool_call` — before tool execution
- `@agent.on_tool_result` — after tool execution
- `@agent.on_run_start` — at run start
- `@agent.on_run_end` — at run end

```python
@agent.on_tool_call
async def log_tool(ctx, tool_name, args):
    print(f'Calling {tool_name} with {args}')
```

## Agent Specs (YAML/JSON)

Define agents declaratively without code:

```yaml
model: anthropic:claude-sonnet-4-6
instructions: You are a helpful assistant.
output_type:
  type: object
  properties:
    answer: { type: string }
capabilities:
  - type: web_search
  - type: thinking
```

Load with:
```python
from pydantic_ai import Agent

agent = Agent.from_spec('agent_spec.yaml')
```

Supports tools, capabilities, instructions, model settings. Useful for config-driven agent definitions.
