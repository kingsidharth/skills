# Duplicate Ways and Dumpyard Modules

## Problem

Agents create `utils`, `common`, `shared`, `helpers`, and `core` when they cannot decide ownership.

## Smells

`utils/db.py`; `utils/serialization.py`; multiple config loaders; multiple DB session factories; duplicate schemas; duplicate retry wrappers.

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

> Is this shared because behavior is the same, or because code looked similar?
