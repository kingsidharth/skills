# Tests and Test Smells

## Problem

Agents write tests that look useful but do not protect behavior.

## Smells

Mocking function under test; asserting mock calls; weakening assertions; snapshots without semantic review; real/dev DB tests.

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

> Can this test fail for the bug we care about?


## Better test example

```python
def test_create_user_rejects_duplicate_email(client, db):
    create_user_in_db(db, email="a@example.com")

    response = client.post("/users", json={"email": "a@example.com"})

    assert response.status_code == 409
    assert count_users(db, email="a@example.com") == 1
```

## Test safety rule

Before running tests, prove they cannot delete dev/prod DB or object-store data.
