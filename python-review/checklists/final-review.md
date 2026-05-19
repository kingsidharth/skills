# Final Review Checklist

## Required sections

- [ ] System map.
- [ ] Highest-risk findings.
- [ ] Pattern findings.
- [ ] Concrete file/function examples.
- [ ] Suggested fixes or applied fixes.
- [ ] Tests added or needed.
- [ ] Final changes needed.
- [ ] Remaining risks.

## Issue classes to report

- [ ] Separation of concerns.
- [ ] Duplicate ways of doing things.
- [ ] Dumpyard modules.
- [ ] Fake services/repositories.
- [ ] Pydantic/serialization boundary issues.
- [ ] State machine reliability.
- [ ] Error handling/retries/idempotency.
- [ ] Test reliability and safety.
- [ ] Performance and scale.
- [ ] Security/data safety.
- [ ] Naming/comments/docs.
- [ ] Shims/v2/backward compatibility debt.

## Final change list rules

Each item should be actionable.

Bad:

- Improve architecture.

Good:

- Move job transition logic from `workers/runner.py` and `routes/jobs.py` into `jobs/state_machine.py`; replace raw string assignments with `transition(job, target_state)`; add workflow tests for invalid transition, stale reclaim, and retry.
