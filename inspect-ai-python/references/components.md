# Components: Datasets, Solvers, Scorers

---

## Datasets

### Sample

The core data type. Required field: `input`. Optional: `target`, `choices`, `id`, `metadata`, `sandbox`, `files`, `setup`.

```python
from inspect_ai.dataset import Sample

Sample(
    input="What is 2+2?",
    target="4",
    id="math_001",
    metadata={"difficulty": "easy"},
)
```

For multiple choice, `target` is a letter (A/B/C/D) and `choices` is the list of options.

### Loading Datasets

```python
from inspect_ai.dataset import csv_dataset, json_dataset, hf_dataset

# Direct read (columns must be input/target)
ds = csv_dataset("data.csv")
ds = json_dataset("data.jsonl")

# HuggingFace
ds = hf_dataset("hellaswag", split="validation", sample_fields=record_to_sample, trust=True)
```

### Field Mapping

When column names don't match `input`/`target`:

```python
from inspect_ai.dataset import FieldSpec

ds = json_dataset("data.jsonl", FieldSpec(
    input="question",
    target="answer",
    id="qid",
    metadata=["category"],
))
```

Or use a custom function:

```python
def record_to_sample(record):
    return Sample(
        input=record["question"],
        target=chr(ord("A") + int(record["label"])),
        choices=record["options"],
    )
```

### Filtering & Shuffling

```python
ds = json_dataset("data.jsonl").filter(lambda s: s.metadata["difficulty"] == "hard")
ds = ds.shuffle(seed=42)
```

### Sample Files

Map local files into sandbox environments:

```python
Sample(
    input="Find the flag",
    target="CTF{found}",
    files={"/shared/flag.txt": "flag.txt"},
    setup="setup.sh",  # runs in sandbox before eval
)
```

---

## Solvers

Solvers transform `TaskState`. They can be chained or composed.

### Built-in Solvers

| Solver | Purpose |
|---|---|
| `generate()` | Call model, append response |
| `system_message("prompt.txt")` | Prepend system message |
| `user_message("text")` | Append user message |
| `chain_of_thought()` | Rewrite prompt for step-by-step reasoning |
| `prompt_template("tmpl.txt")` | Template substitution with `{prompt}` placeholder |
| `multiple_choice()` | Present choices, extract answer letter |
| `self_critique()` | Model critiques its own output, then regenerates |

### Chaining

```python
solver=[
    system_message("You are a security expert."),
    chain_of_thought(),
    generate(),
    self_critique(),
]
```

### Custom Solvers

```python
from inspect_ai.solver import solver, TaskState, Generate

@solver
def my_solver():
    async def solve(state: TaskState, generate: Generate):
        state.messages.append(ChatMessageUser(content="Think step by step."))
        return await generate(state)
    return solve
```

### Composite Solvers

Wrap chains in `@solver` for reuse:

```python
from inspect_ai.solver import chain

@solver
def critique_pipeline(system="system.txt"):
    return chain(
        system_message(system),
        generate(),
        self_critique(),
    )
```

### Solver Independence

Tasks can accept alternate solvers via CLI: `inspect eval task.py --solver other_solver.py`

---

## Scorers

Evaluate whether solver output matches the target.

### Built-in Scorers

| Scorer | Use Case |
|---|---|
| `exact()` | Normalized exact match |
| `includes()` | Target appears anywhere in output |
| `match()` | Target at start/end of output |
| `pattern(regex)` | Regex extraction |
| `answer()` | Extract from "ANSWER: ..." format |
| `f1()` | Token-level F1 score |
| `choice()` | For `multiple_choice()` solver |
| `math()` | Mathematical equivalence (requires `sympy`) |
| `model_graded_qa()` | LLM judges answer quality |
| `model_graded_fact()` | LLM checks if output contains target fact |
| `perplexity()` | Per-token NLL from logprobs |

### Model Grading

```python
scorer=model_graded_fact()                          # uses eval model as grader
scorer=model_graded_fact(model="openai/gpt-4o")     # explicit grader model
scorer=model_graded_qa(template="custom_rubric.txt")
```

### Custom Scorers

```python
from inspect_ai.scorer import scorer, Score, Target, accuracy

@scorer(metrics=[accuracy()])
def my_scorer():
    async def score(state: TaskState, target: Target):
        answer = state.output.completion
        return Score(value="C" if target.text in answer else "I")
    return score
```

`Score.value` is typically `"C"` (correct) / `"I"` (incorrect) / `"P"` (partial), or a numeric float.

### Metrics

Default metrics: `accuracy()` and `stderr()`. Custom:

```python
Task(..., scorer=my_scorer(), metrics=[mean(), stderr()])
```

### Epochs

Run each sample N times and reduce:

```python
Task(..., epochs=3)  # default reducer: majority vote
Task(..., epochs=Epochs(3, reducer="mean"))
```

### Multiple Scorers

```python
Task(..., scorer=[exact(), model_graded_fact()])
```

### Rescoring

```bash
inspect score --log logs/eval.json --scorer new_scorer.py
```
