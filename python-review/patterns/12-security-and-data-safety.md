# Security and Data Safety

## Problem

Agents validate shape but miss authority, trust boundaries, and destructive side effects.

## Smells

Auth only in UI; path traversal; unsafe YAML; SQL interpolation; secrets in logs; test cleanup hits dev/prod DB.

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

> What is the worst thing this code can delete or expose if the environment is wrong?
