# Test and Refactor Loop

## Principle

If behavior matters and tests do not capture it, write meaningful tests before refactoring.

Do not write tests to preserve accidental implementation. Write tests to preserve intended behavior.

## When to write tests first

Write tests first when changing state machines, job queues, retries, DB transactions, serialization boundaries, external side effects, API behavior, ingestion/data pipelines, file/object-storage writes, auth, permissions, or performance-sensitive loops.

## Test types

### Functional tests

Use for pure or mostly-pure behavior: hash normalization, validation, state transition rejection, deterministic dedupe keys.

### Integration tests

Use when boundaries matter: API + DB + schema, repository transactions, file write + manifest update, object-store abstraction.

### Workflow tests

Use for lifecycle: pending → claimed → running → complete, crash → stale → reclaim, upload succeeds but DB insert fails → retry does not duplicate.

### Regression tests

Use for confirmed bugs: invalid state cannot be marked complete, dev DB URL rejected in tests, manual serialization no longer drops field aliases.

## Bad tests

Avoid mocking the function being tested, only asserting mock calls, broad snapshot updates, weakened assertions, real dev/prod DBs, and default external-service dependencies.

## Refactor loop

1. Write/verify behavior tests.
2. Run focused tests.
3. Refactor one issue class.
4. Run focused tests again.
5. Run broader checks.
6. Remove dead code created by the refactor.
7. Do not leave `v2`, shims, or duplicated paths unless explicitly required.
