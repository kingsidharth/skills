# Test Data Safety

## Rule

Tests must never delete or mutate dev/prod data.

This is a first-class review area because AI agents often run tests or cleanup scripts without checking the target database or storage path.

## Required checks before running tests

Check:

- `DATABASE_URL` / `TEST_DATABASE_URL`.
- SQLite path.
- Postgres database name.
- Docker Compose service names.
- Alembic environment.
- object-store bucket and prefix.
- filesystem output path.
- cleanup fixtures.
- `DROP`, `DELETE`, `TRUNCATE`, `rm -rf`, `shutil.rmtree`.

## Safe defaults

Tests should create a fresh test database, populate fixtures from scratch, use transaction rollback or per-test DB reset, use test-only object-storage prefixes, reject production/dev DB names, require explicit env var for destructive integration tests, and avoid global cleanup outside test-owned paths.

## Guard examples

```python
from urllib.parse import urlparse

FORBIDDEN_DB_NAMES = {"prod", "production", "dev", "development", "main"}

def assert_test_database_url(url: str) -> None:
    parsed = urlparse(url)
    db_name = parsed.path.rsplit("/", 1)[-1]
    if "test" not in db_name.lower() or db_name.lower() in FORBIDDEN_DB_NAMES:
        raise RuntimeError(f"Refusing to run tests against non-test database: {db_name}")
```

```python
from pathlib import Path
import shutil

def safe_rmtree(path: Path, allowed_root: Path) -> None:
    path = path.resolve()
    allowed_root = allowed_root.resolve()
    if not path.is_relative_to(allowed_root):
        raise RuntimeError(f"Refusing to delete outside test root: {path}")
    if path == allowed_root:
        raise RuntimeError("Refusing to delete the entire test root")
    shutil.rmtree(path)
```

## Review smell

> Test cleanup is broader than test setup.
