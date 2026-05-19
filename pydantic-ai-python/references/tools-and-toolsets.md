# Tools and Toolsets

## Function tools

Three registration methods:

**Decorator with context** (`@agent.tool`):
```python
@agent.tool
async def get_weather(ctx: RunContext[MyDeps], city: str) -> str:
    """Get weather for a city."""  # docstring → tool description
    return await ctx.deps.weather_api.get(city)
```

**Decorator without context** (`@agent.tool_plain`):
```python
@agent.tool_plain
def calculate(expression: str) -> float:
    """Evaluate a math expression."""
    return eval(expression)
```

**Via constructor**:
```python
from pydantic_ai import Tool

agent = Agent('...', tools=[Tool(get_weather, takes_ctx=True), calculate])
```

## Schema generation

- Function signature → JSON Schema for tool parameters
- `RunContext` param is excluded from schema
- Docstrings extracted via `griffe` (google, numpy, sphinx styles)
- Parameter descriptions from docstring → parameter schema
- Pydantic validates tool arguments; errors sent back to LLM for retry

## Tool output

Tools can return anything Pydantic can serialize to JSON. For multi-modal returns or metadata, see advanced tool features.

## Toolsets

Collections of tools registered together via `toolsets=` kwarg:

```python
from pydantic_ai.toolsets import FunctionToolset

ts = FunctionToolset()

@ts.tool
def my_tool(x: int) -> str:
    return str(x)

agent = Agent('...', toolsets=[ts])
```

Combine toolsets: all `tools=` and `toolsets=` merge into one combined toolset.

## Deferred tools (human-in-the-loop)

Flag tools as requiring approval before execution:

```python
from pydantic_ai import Tool

tool = Tool(delete_record, defer=True)  # or defer=check_function
```

When deferred, tool call pauses — your app presents it to a human for approval.

## Built-in tools

Provider-native tools executed by the model provider's infrastructure:

- `WebSearchTool` — provider-native web search
- `CodeExecutionTool` — provider-native code execution  
- `WebFetchTool` — provider-native URL fetching
- `FileSearchTool` — vector search over uploaded files (OpenAI)
- `MCPServerTool` — remote MCP servers handled by provider

Passed via `builtin_tools=` parameter. Not all providers support all built-in tools.

## Common tools

SDK-side tool implementations that work across providers:

```python
from pydantic_ai.common_tools import DuckDuckGoSearchTool
agent = Agent('...', tools=[DuckDuckGoSearchTool()])
```

## Provider-adaptive tools (via Capabilities)

`WebSearch` and `MCP` capabilities auto-select native tool when available, fall back to local:
```python
from pydantic_ai.capabilities import WebSearch
agent = Agent('...', capabilities=[WebSearch()])
```

## MCP integration

```python
from pydantic_ai.mcp import MCPServerHTTP

server = MCPServerHTTP('https://my-mcp-server.example.com/sse')
agent = Agent('...', toolsets=[server])
```

Supports `MCPServerHTTP`, `MCPServerStdio`, `MCPServerStreamableHTTP`. Tools from MCP servers appear alongside function tools.

## Third-party tools

Install community tool packages and pass them as toolsets. Check `pydantic_ai.ext` and PyPI for available packages.
