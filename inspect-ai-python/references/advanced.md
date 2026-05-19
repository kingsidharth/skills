# Advanced

## Eval Sets

Run the same task across multiple models or configurations:

```bash
inspect eval task.py --model openai/gpt-4o --model anthropic/claude-sonnet-4-0
```

Logs are grouped in the Log Viewer for side-by-side comparison.

## Error Handling

### Failure Threshold

```python
Task(..., fail_on_error=0.1)  # fail if >10% of samples error
Task(..., fail_on_error=5)    # fail if >5 samples error
Task(..., fail_on_error=True) # fail on any error (default: False)
```

Errors are logged per-sample with full tracebacks. Failed samples get `Score(value="E")`.

### Retries

```bash
inspect eval task.py --max-retries 3        # retry failed API calls
inspect eval task.py --retry-on-error       # retry entire failed samples
```

## Setting Limits

Per-sample safety limits to prevent runaway evals:

```python
Task(
    ...,
    message_limit=50,      # max messages in conversation
    token_limit=100_000,   # max total tokens
    time_limit=300,        # max seconds per sample
    working_limit=60,      # max seconds of active work (excludes wait time)
)
```

CLI: `--message-limit 50 --token-limit 100000 --time-limit 300`

## Typing

Use Python type hints throughout. Key types:

- `TaskState` — solver input/output
- `Score` — scorer output
- `Sample` — dataset record
- `ModelOutput` — model response
- `ChatMessage` variants: `ChatMessageSystem`, `ChatMessageUser`, `ChatMessageAssistant`, `ChatMessageTool`

Metadata typing with Pydantic:

```python
class MyMeta(BaseModel, frozen=True):
    category: str
    difficulty: float

meta = state.metadata_as(MyMeta)
```

## Tracing

Add custom spans to eval logs for debugging:

```python
from inspect_ai.util import trace

with trace("my_step"):
    result = await model.generate(...)
```

Traces appear in the Log Viewer timeline.

## Parallelism

Control concurrent sample execution:

```bash
inspect eval task.py --max-connections 10   # concurrent model API calls
inspect eval task.py --max-samples 5        # concurrent samples
inspect eval task.py --max-subprocesses 10  # concurrent subprocess calls
inspect eval task.py --max-sandboxes 5      # concurrent Docker containers
```

Default parallelism scales with model provider rate limits.

## Interactivity

Interactive mode for development — pause between samples:

```bash
inspect eval task.py --interactive
```

Step through samples, inspect state, modify prompts live.

## Early Stopping

Stop eval early based on running metrics:

```python
from inspect_ai import eval
from inspect_ai.scorer import accuracy

eval("task.py", early_stopping=accuracy(threshold=0.95, window=20))
```

Stops when accuracy over the last 20 samples exceeds 95%.

## Extensions API

Create custom:

- **Model providers** — implement `ModelAPI` for new LLM endpoints
- **Sandbox environments** — implement `SandboxEnvironment` for new container systems
- **Storage** — custom log storage backends

Register via Python entry points in `pyproject.toml`:

```toml
[project.entry-points."inspect_ai"]
my_provider = "my_package:MyModelAPI"
```
