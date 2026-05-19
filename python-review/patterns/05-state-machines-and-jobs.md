# State Machines and Jobs

## Problem

Reliable state requires allowed transitions, retries, stale recovery, concurrency, terminal states, and idempotency.

## Smells

Raw string states; direct mutation; no transition map; no stale recovery; non-atomic job claim; retry duplicates side effects.

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

> Can this workflow survive crash, retry, and two workers?


## Better state machine sketch

```python
from enum import Enum

class JobState(str, Enum):
    PENDING = "pending"
    CLAIMED = "claimed"
    RUNNING = "running"
    COMPLETE = "complete"
    FAILED = "failed"
    STALE = "stale"

ALLOWED_TRANSITIONS = {
    JobState.PENDING: {JobState.CLAIMED},
    JobState.CLAIMED: {JobState.RUNNING, JobState.STALE},
    JobState.RUNNING: {JobState.COMPLETE, JobState.FAILED, JobState.STALE},
    JobState.FAILED: {JobState.PENDING},
    JobState.STALE: {JobState.PENDING},
}

def transition(job, target: JobState) -> None:
    if target not in ALLOWED_TRANSITIONS[job.state]:
        raise InvalidTransition(job.state, target)
    job.state = target
```

## Atomic claim requirement

Never claim a job with read-then-write without a lock or atomic update.
