# Recommended Tools

## Static analysis

- `ruff` — linting and formatting.
- `pyright` or `mypy` — type checking.
- `vulture` — dead code detection.
- `deptry` — dependency usage checks.
- `import-linter` — import boundary contracts.

## Security

- `bandit` — Python security linting.
- `pip-audit` — dependency vulnerability audit.
- `detect-secrets` or equivalent — secret detection.

## Testing

- `pytest` — test runner.
- `pytest-cov` — coverage.
- `pytest-xdist` — parallel tests, only when tests are isolated.
- `pytest-mock` — mocking, used sparingly.
- lightweight factories or `factory-boy` — fixture creation.

## Database testing

- Testcontainers or Docker Compose test DB.
- Transaction rollback fixtures.
- Explicit test DB guards.
- Isolated object-store prefixes or local fakes.

## Performance

- `py-spy` — sampling profiler.
- `scalene` — CPU/memory profiler.
- `cProfile` / `pstats` — built-in profiling.
- `pytest-benchmark` — benchmark tests.
- `memray` — memory profiling.

## Architecture checks

- `import-linter` contracts for layer direction.
- simple grep checks for forbidden names/patterns.
- CI checks for dependency diff, broad exceptions, and unsafe cleanup.
