# Agents and Dependencies

## Agent construction

```python
from pydantic_ai import Agent

agent = Agent(
    'openai:gpt-5.2',          # model string (or set at run time)
    deps_type=MyDeps,           # dependency type constraint
    output_type=MyOutput,       # structured output type
    instructions='...',         # static instructions (str or sequence)
    tools=[...],                # list of Tool or plain functions
    toolsets=[...],             # list of Toolset instances
    capabilities=[...],         # list of Capability instances
    model_settings={...},       # default ModelSettings
    retries=3,                  # max output validation retries
    instrument=True,            # auto-instrument with Logfire
)
```

Type: `Agent[DepsType, OutputType]`. Both default to `None` / `str`.

Agents are stateless, reusable. Instantiate globally.

## Instructions

Static (keyword arg):
```python
agent = Agent('...', instructions='You are a helpful assistant.')
```

Dynamic (decorator, has access to deps via `RunContext`):
```python
@agent.instructions
async def add_context(ctx: RunContext[MyDeps]) -> str:
    name = await ctx.deps.db.get_name(ctx.deps.user_id)
    return f"User is {name}"
```

Both static and dynamic instructions are combined into the system prompt.

## Dependencies

Pass runtime data to tools and instructions via dependency injection.

```python
from dataclasses import dataclass
from pydantic_ai import RunContext

@dataclass
class MyDeps:
    db: DatabaseConn
    user_id: int

agent = Agent('...', deps_type=MyDeps)

@agent.tool
async def get_balance(ctx: RunContext[MyDeps]) -> float:
    return await ctx.deps.db.balance(ctx.deps.user_id)

result = await agent.run('...', deps=MyDeps(db=conn, user_id=42))
```

`RunContext` carries: `deps`, `model`, `usage`, `run_step`, `retry`, `tool_name`, `tool_call_id`.

Type-checked: wrong dep type annotation → static type error.

## Running agents

| Method | Sync/Async | Returns |
|--------|-----------|---------|
| `run()` | async | `AgentRunResult` |
| `run_sync()` | sync | `AgentRunResult` |
| `run_stream()` | async ctx mgr | `StreamedRunResult` |
| `run_stream_sync()` | sync ctx mgr | `StreamedRunResultSync` |
| `run_stream_events()` | async ctx mgr | yields `AgentStreamEvent` |
| `iter()` | async ctx mgr | `AgentRun` (graph nodes) |

All accept: `user_prompt`, `deps=`, `model=`, `model_settings=`, `usage_limits=`, `message_history=`.

## Conversations (multi-turn)

Pass `message_history` from a previous run to continue:
```python
r1 = await agent.run('Hello')
r2 = await agent.run('Follow up', message_history=r1.all_messages())
```

`result.new_messages()` returns only messages from the current run.

`result.all_messages()` returns history + new messages.

## Usage limits

```python
from pydantic_ai import UsageLimits

result = await agent.run(
    '...',
    usage_limits=UsageLimits(
        request_limit=10,
        total_tokens_limit=5000,
    ),
)
```

## Model settings at run time

```python
result = await agent.run('...', model_settings={'temperature': 0.5, 'max_tokens': 1000})
```

## Reflection and self-correction

When output validation fails, PydanticAI sends the error back to the LLM and asks it to retry (up to `retries` times). This works for both structured output and tool argument validation.
