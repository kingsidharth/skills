# Python Review Skill

## Scope

Use this skill to review, repair, or refactor Python projects where AI-generated or agent-written code may have damaged structure, reliability, performance, security, tests, or maintainability.

This skill is especially useful for:

- FastAPI, Flask, Django, Typer, CLI, worker, queue, data-pipeline, and ML/image-processing projects.
- Python repos with `utils`, `common`, `shared`, `helpers`, `services`, `managers`, `processors`, or wrapper sprawl.
- Projects where AI agents added many files, many abstractions, noisy comments, weak tests, compatibility shims, or multiple ways to do the same thing.
- Reviews where the goal is not only “does it pass tests?” but “does this preserve system shape?”

Do not use this skill as a generic Python style guide. Use it when architecture, behavior, tests, state, safety, and long-run maintainability matter.

---

## What is being tested

The review tests whether the Python project has:

1. Clear separation of concerns.
2. One obvious way to perform each core action.
3. Reliable state transitions and job/workflow lifecycle rules.
4. Explicit ownership of data models, serialization, persistence, errors, configuration, and side effects.
5. Functional, integration, or workflow tests that capture real behavior.
6. No accidental production/dev data destruction through tests.
7. No gratuitous wrappers, fake services, `v2` shims, compatibility layers, or “safe” fallbacks unless explicitly required.
8. No noisy comments, generic docs, or verbal filler.
9. Good enough performance for the expected scale.
10. A final review output that identifies issue classes, concrete examples, suggested fixes, and the final change list.

---

## Initial checks

Start with these checks before editing code.

### 1. Build a system map

Read enough files to explain:

- Entry points: API, CLI, worker, cron, notebook, scripts.
- Data flow: request/input → validation → domain/use case → persistence → side effects → response/output.
- Persistence: database/session ownership, migrations, object storage, files, caches.
- State machines: jobs, tasks, ingestion, retries, lifecycle states.
- Boundaries: API schemas, ORM models, domain objects, repositories, services/functions, workers.
- Test setup: database selection, fixtures, factories, destructive operations, integration tests.

Write a short map before making changes. If the map is uncertain, say where it is uncertain.

### 2. Identify obvious AI-code smells

Search for:

- `utils`, `common`, `shared`, `helpers`, `service`, `manager`, `processor`, `handler`, `wrapper`, `base`, `generic`, `v2`, `compat`, `legacy`, `shim`.
- Broad catches: `except Exception`, `except BaseException`, `contextlib.suppress`.
- Manual serialization: handwritten dicts from ORM/Pydantic objects.
- Test hazards: real DB URLs, destructive fixtures, cleanup scripts, direct `DROP`, `DELETE`, `rm -rf`, object-store deletes.
- Noisy comments and docs generated without need.
- Duplicate libraries for logging, config, HTTP, validation, retries.

### 3. Do not start by rewriting

First produce a review. Then decide whether tests should be written to lock behavior. Then refactor.

### 4. Protect data before tests

Before running tests or scripts, verify they cannot hit a real/dev/prod DB or real object store unless explicitly intended.

Minimum checks:

- Test database URL is isolated.
- Fixtures create and populate from scratch.
- Destructive cleanup is scoped to test resources.
- Object-storage paths are test-prefixed.
- Environment variables cannot silently point to dev/prod.

If unsure, do not run destructive tests.

---

## Review workflow

Use this order.

1. **Inventory**: inspect structure, entry points, dependencies, tests, and config.
2. **System map**: articulate how the system appears to work.
3. **Risk scan**: identify demonstrable issue classes with examples.
4. **Test plan**: decide which behavior needs functional, integration, or workflow tests before code changes.
5. **Test safety check**: confirm tests cannot destroy dev/prod data.
6. **Write/adjust tests**: capture current intended behavior, not accidental implementation.
7. **Refactor/repair**: make the smallest changes that solve the issue class.
8. **Run checks**: lint, type checks, tests, focused integration tests.
9. **Final review**: issue classes, concrete examples, fixes applied/proposed, tests added, final change list, remaining risks.

See `workflow/review-process.md` and `workflow/test-and-refactor-loop.md`.

---

## Final review output format

Use this format.

```md
# Python Review

## System map
- Entry points:
- Data flow:
- Persistence:
- State machines:
- Test setup:
- Uncertainties:

## Highest-risk findings
1. [Issue class] Specific file/function/example
   - Why it matters:
   - Demonstrable evidence:
   - Suggested fix:

## Pattern findings
- Separation of concerns:
- Duplicate ways of doing things:
- State machine reliability:
- Serialization/model boundaries:
- Error handling:
- Tests:
- Performance:
- Security/data safety:
- Naming/comments/docs:

## Tests needed or added
- Functional:
- Integration:
- Workflow/state-machine:
- Regression:

## Final changes needed
1. ...
2. ...
3. ...

## Do not do
- No v2/shims/backward-compat layers unless explicitly required.
- No useless docs.
- No broad wrappers.
- No fake service layer.
```

---

## Library index

### Workflow

- `workflow/review-process.md` — how to conduct the review.
- `workflow/system-map.md` — how the agent should self-articulate what it understood.
- `workflow/test-and-refactor-loop.md` — when to write tests before changing code.
- `workflow/test-data-safety.md` — preventing tests from deleting dev/prod data.

### Pattern library

- `patterns/01-structure-and-separation.md`
- `patterns/02-duplicate-ways-and-dumpyards.md`
- `patterns/03-pydantic-serialization-boundaries.md`
- `patterns/04-fastapi-routing-service-repository.md`
- `patterns/05-state-machines-and-jobs.md`
- `patterns/06-errors-retries-idempotency.md`
- `patterns/07-async-concurrency-resources.md`
- `patterns/08-tests-and-test-smells.md`
- `patterns/09-performance-and-scale.md`
- `patterns/10-naming-comments-docs.md`
- `patterns/11-shims-v2-and-compatibility.md`
- `patterns/12-security-and-data-safety.md`
- `patterns/13-functional-composition.md`

### Checklists

- `checklists/initial-scan.md`
- `checklists/final-review.md`
- `checklists/refactor-safety.md`
- `checklists/python-performance.md`

### References

- `references/research-notes.md` — evidence themes and external research areas to cite when needed.
- `references/recommended-tools.md` — useful static analysis, test, security, and performance tools.

### Templates

- `templates/review-report-template.md`
- `templates/system-map-template.md`
- `templates/test-plan-template.md`
