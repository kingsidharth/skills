# System Map

## Purpose

The agent must prove it understands the system before changing it.

A system map prevents line-level patching that breaks module-level or workflow-level behavior.

## What to map

### Entry points

API routes, CLI commands, worker jobs, cron scripts, notebooks promoted to scripts, tests, fixtures, and migration scripts.

### Data flow

Input source → validation boundary → domain/use-case function → persistence boundary → side effects → response/output → error path.

### Model boundaries

- ORM model: persistence shape.
- Pydantic request model: external input shape.
- Pydantic response model: external output shape.
- Domain object/value object: business meaning.
- Command object: use-case input.
- DTO: cross-boundary transfer, only when needed.

### State machines

List states, allowed transitions, terminal states, retry states, stale/timeout behavior, idempotency keys, and who can mutate state.

### Persistence and side effects

List DB/session ownership, transactions, object storage paths, cache behavior, external API calls, file writes, and destructive operations.

### Test setup

List DB used by tests, fixture creation, cleanup behavior, object-store/filesystem isolation, environment variables, and risky scripts.

## Output template

```md
## System map

### Entry points
- ...

### Data flow
- ...

### Model boundaries
- ...

### State machines
- ...

### Persistence and side effects
- ...

### Test setup
- ...

### Uncertainties
- ...
```

## Red flags

- The agent cannot explain who owns transactions.
- The agent cannot explain where validation ends and business logic begins.
- The agent cannot list lifecycle states.
- The agent cannot say which DB tests use.
- The agent does not know whether destructive operations are scoped.
