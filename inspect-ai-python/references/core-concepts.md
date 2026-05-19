# Core Concepts

## Architecture

Every Inspect evaluation is a **Task** composed of three parts:

1. **Dataset** — labeled samples with `input` (prompt) and `target` (expected answer)
2. **Solver** — the evaluation pipeline: prompt engineering → model generation → optional refinement
3. **Scorer** — evaluates the solver's output against the target

```
Dataset ──→ Solver(s) ──→ Scorer ──→ Score + Metrics
              ↑
           Model API
```

## Minimal Example

```python
from inspect_ai import Task, task
from inspect_ai.dataset import Sample
from inspect_ai.scorer import exact
from inspect_ai.solver import generate

@task
def hello():
    return Task(
        dataset=[Sample(input="Say Hello World", target="Hello World")],
        solver=[generate()],
        scorer=exact(),
    )
```

Run: `inspect eval hello.py --model openai/gpt-4o`

## The @task Decorator

- Makes a function discoverable by `inspect eval`
- Must return a `Task` object
- Supports parameters for flexibility:

```python
@task
def my_eval(system_prompt="default.txt", grader_model="openai/gpt-4o"):
    return Task(...)
```

Override via CLI: `inspect eval my_eval.py -T system_prompt="alt.txt"`

Or via YAML config: `inspect eval my_eval.py --task-config=config.yaml`

## TaskState Lifecycle

Each sample flows through solvers as a `TaskState`:

```python
class TaskState:
    messages: list[ChatMessage]   # conversation history
    output: ModelOutput           # latest model output
```

Solvers receive `(state, generate)` and must return the modified state. The `generate` function calls the model and appends the assistant response.

## Task Options

Key `Task(...)` parameters beyond dataset/solver/scorer:

- `epochs` — run each sample multiple times
- `sandbox` — sandbox config for untrusted code (e.g. `"docker"`)
- `config` — `GenerateConfig` for model params (temperature, etc.)
- `message_limit`, `token_limit`, `time_limit` — safety limits per sample
- `fail_on_error` — tolerance threshold for sample failures
- `setup` — solver(s) to run before the main solver
- `approval` — tool call approval policies

## Eval from Python

```python
from inspect_ai import eval

logs = eval(
    "my_eval.py",
    model="openai/gpt-4o",
    limit=10,              # only run first 10 samples
    log_dir="./logs",
)
```
