# Agentic Tool Use

## The `.act()` call

Multi-round automatic tool calling. The model decides when to call tools, receives results, and continues until it produces a final text response.

```python
model.act("prompt or chat", [tool1, tool2], on_message=print)
```

## Defining tools

### Option 1: Plain functions (preferred)

Type hints + docstring → auto-extracted name, description, parameter schema.

```python
def add(a: int, b: int) -> int:
    """Given two numbers a and b, returns the sum of them."""
    return a + b
```

### Option 2: ToolFunctionDef

For custom names/descriptions:
```python
from lmstudio import ToolFunctionDef
tool = ToolFunctionDef.from_callable(my_func, name="custom_name", description="...")
```

Tool name, description, and parameter types are all sent to the model — wording affects quality.

## Tools with side effects

Tools can create files, call APIs, run programs:
```python
from pathlib import Path

def create_file(name: str, content: str):
    """Create a file with the given name and content."""
    dest = Path(name)
    if dest.exists():
        return "Error: File already exists."
    dest.write_text(content, encoding="utf-8")
    return "File created."

model.act("Create a hello world file", [create_file])
```

## Error handling (SDK ≥ 1.3.0)

By default, exceptions from tool calls are converted to text and sent back to the model. Override with `handle_invalid_tool_request`:

```python
def _raise_locally(exc, request):
    raise exc

model.act(chat, [divide], handle_invalid_tool_request=_raise_locally)
```

Callback behavior:
- Return `None` → original error text sent to model
- Return a `str` → that string sent instead
- Raise → propagated locally, terminates prediction

## Parallel tool calls (SDK ≥ 1.4.0)

Default: sequential (`max_parallel_tool_calls=1`). Set higher for thread-safe tools:
```python
model.act(chat, tools, max_parallel_tool_calls=4)
model.act(chat, tools, max_parallel_tool_calls=None)  # auto-scale to CPU cores
```

## Progress callbacks for `.act()`

All standard callbacks plus round-aware variants:
- `on_round_start(round_index)` — before each prediction round
- `on_prediction_completed(result)` — after prediction, before tool calls; `result.round_index`
- `on_round_end()` — after tool calls resolved
- `on_prompt_processing_progress(progress, round_index)`
- `on_first_token(round_index)`
- `on_prediction_fragment(fragment, round_index)`
- `on_message(message)` — assistant and tool-result messages (no round index)

## Model selection

Not all models support tool use well. Bigger models perform better. Qwen2.5-7B-Instruct is a solid baseline.
