# Output and Streaming

## Structured output

Set `output_type` to a Pydantic model, dataclass, TypedDict, or primitive:

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent

class CityInfo(BaseModel):
    name: str
    population: int = Field(description='Estimated population')
    country: str

agent = Agent('...', output_type=CityInfo)
result = agent.run_sync('Tell me about Paris')
print(result.output.population)  # typed as CityInfo
```

PydanticAI builds JSON Schema from the model and instructs the LLM to return conforming JSON. Pydantic validates the response; on failure, the error is sent back for retry.

## Union output types

```python
agent = Agent('...', output_type=[CityInfo, CountryInfo])  # union of types
```

## Plain text output

Default `output_type` is `str` — the LLM's text response returned directly.

## Output functions

Functions the LLM can call as a final action (result not sent back to model):

```python
@agent.output_function
async def save_result(ctx: RunContext[MyDeps], data: MyOutput) -> MyOutput:
    await ctx.deps.db.save(data)
    return data
```

## Streamed results

### Text streaming
```python
async with agent.run_stream('...') as response:
    async for text in response.stream_text():
        print(text, end='', flush=True)
```

### Structured streaming
```python
async with agent.run_stream('...') as response:
    async for partial in response.stream_output(debounce_by=0.1):
        print(partial)  # partially validated output
```

### Event streaming
```python
async with agent.run_stream_events('...') as stream:
    async for event in stream:
        # PartStartEvent, PartDeltaEvent, PartEndEvent,
        # FinalResultEvent, AgentRunResultEvent
        print(event)
```

### Sync streaming
```python
with agent.run_stream_sync('...') as response:
    for text in response.stream_text():
        print(text)
```

## Validation and retries

Output validation failures trigger reflection: the validation error is sent back to the LLM as a retry prompt. Controlled by `retries=` on the Agent (default: 1).
