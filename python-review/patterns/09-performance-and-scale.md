# Performance and Scale

## Problem

Agents write Python that works for toy data and fails at real scale.

## Smells

Full dataset loads; pandas row loops; N+1; per-row network calls; no batching; nested loops; needless materialization.

## Bad pattern

```python
# Representative AI-agent smell.
class GenericService:
    def __init__(self, db):
        self.db = db

    def process(self, data):
        try:
            return self._handle(data)
        except Exception:
            return None
```

## Better direction

- Name the real domain responsibility.
- Keep boundaries explicit.
- Prefer direct functions when no stateful object is needed.
- Use framework features instead of hand-rolled replacements.
- Add tests around behavior before refactoring risky paths.
- Delete the duplicate or old path after replacing it.

## Fix strategy

1. Find concrete examples.
2. Identify the boundary that should own the behavior.
3. Write focused functional/integration/workflow tests if behavior is not covered.
4. Replace the pattern with the smallest clear design.
5. Remove dead code, shims, fake wrappers, and noisy comments.
6. Report remaining risks.

## Review question

> What happens at 10x, 100x, and 1000x input size?


## Python loop guidance

Do not blindly replace loops with `map` or `reduce`.

Prefer:

- loop when side effects or branching are central;
- list comprehension for simple transformations;
- generator expression for streaming;
- `map` when applying one named function across values improves composition;
- `functools.reduce` rarely, only when clearer than a loop;
- vectorized operations for large numeric/tabular data;
- batch APIs when the underlying system supports batching.

## Performance fix examples

- Replace per-row DB inserts with batch insert.
- Replace full-list materialization with chunked iterators.
- Replace repeated ORM → dict → Pydantic → dict conversions with one boundary conversion.
- Push filtering/aggregation into SQL when appropriate.
- Profile before rewriting hot code.
