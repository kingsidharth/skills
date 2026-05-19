# Review Process

## Principle

Do not start by fixing code. First understand the system, then identify issue classes, then write tests if behavior is not safely captured, then refactor.

## Step 1: Inventory

Inspect:

- `pyproject.toml`, `requirements.txt`, lockfiles.
- App entry points: `main.py`, `app.py`, `asgi.py`, `wsgi.py`, CLI scripts, workers.
- Tests: `tests/`, `conftest.py`, factories, fixtures.
- Migrations: Alembic, Django migrations, custom scripts.
- Config: `.env.example`, settings modules, Docker Compose, CI config.
- Data lifecycle: DB, files, object storage, cache, queues.

Look for duplicate choices:

- multiple config systems;
- multiple loggers;
- multiple HTTP clients;
- multiple serialization paths;
- multiple DB session patterns;
- multiple retry systems.

## Step 2: System map

Write what the system appears to do. Keep it short. Capture uncertainties.

Bad: “The project has routes, services, and models.”

Good: “`POST /sources/{id}/ingest` creates an ingest job. Workers claim pending jobs through `jobs.repository.claim_next`, process images, write artifacts to object storage, then upsert manifests through `samples.repository`. Job state is persisted in `jobs.status`. Unclear: stale-claim recovery and object-store idempotency.”

## Step 3: Pattern scan

Search for issue classes, not one-off style complaints.

Examples:

- Serialization is handwritten in routes despite Pydantic response models.
- Three different modules create DB sessions.
- Job status is mutated directly as strings from five files.
- Tests use the same DB URL as development.
- Agent created `v2` adapters instead of replacing the implementation.

## Step 4: Demonstrate issues

Prefer concrete proof:

- file/function references;
- import graph contradiction;
- duplicate library use;
- failing focused test;
- race condition in job claim;
- destructive fixture path;
- benchmark/profile showing loop or memory issue;
- state transition missing from enum/map.

Avoid vague claims like “bad architecture.”

## Step 5: Test plan

Before refactoring behavior, decide what tests protect intended behavior.

Prefer functional, integration, workflow, regression, and safety tests. Avoid mocks that prove mocks work.

## Step 6: Minimal repair

Change as little as needed to remove the issue class.

Rules:

- Do not add `v2`, `compat`, `legacy`, or shim layers unless explicitly required.
- Do not create a fake service layer.
- Do not create new docs unless asked.
- Do not add noisy comments.
- Do not change public behavior unless the plan says so.
- Do not mix formatting churn with logic.

## Step 7: Final review

Report system map, highest-risk findings, issue class patterns, examples, tests, final change list, and remaining risks.
