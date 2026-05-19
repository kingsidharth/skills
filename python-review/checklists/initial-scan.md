# Initial Scan Checklist

## Structure

- [ ] Identify entry points.
- [ ] Identify domain packages.
- [ ] Search for `utils`, `common`, `shared`, `helpers`.
- [ ] Search for `service`, `manager`, `processor`, `handler`, `wrapper`, `base`.
- [ ] Search for `v2`, `legacy`, `compat`, `shim`.
- [ ] Check import direction and circular import risk.

## Models and serialization

- [ ] Find Pydantic models.
- [ ] Find ORM models.
- [ ] Find manual dict serialization.
- [ ] Find duplicate schemas/DTOs.
- [ ] Check response models and API boundaries.

## Persistence and state

- [ ] Identify DB session creation.
- [ ] Identify transaction ownership.
- [ ] Search for `commit()` locations.
- [ ] Identify job/task states.
- [ ] Check direct string state mutations.
- [ ] Check atomic job claim.

## Tests

- [ ] Identify test DB URL.
- [ ] Check fixtures and cleanup.
- [ ] Search for destructive operations.
- [ ] Check mocks.
- [ ] Check integration/workflow coverage.

## Safety

- [ ] Check object-store prefixes.
- [ ] Check migration scripts.
- [ ] Check shell commands.
- [ ] Check path handling.
- [ ] Check secrets in logs.

## Performance

- [ ] Search for full dataset loads.
- [ ] Search for nested loops.
- [ ] Search for per-row DB/network calls.
- [ ] Check batching/chunking.
- [ ] Check indexes for common queries.
