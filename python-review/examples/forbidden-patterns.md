# Forbidden Pattern Examples

## Fake service

```python
class UserService:
    def __init__(self, db):
        self.db = db

    def get_user(self, user_id):
        return self.db.query(User).filter(User.id == user_id).first()
```

Prefer a repository query or use-case function if there is no real service behavior.

## Manual serialization

```python
return {"id": user.id, "created_at": user.created_at.isoformat()}
```

Prefer response schemas at the boundary.

## Unsafe test cleanup

```python
def teardown():
    shutil.rmtree(DATA_DIR)
```

Only delete under a test-owned path after asserting scope.

## Compatibility shim without requirement

```python
if use_new:
    return process_v2(data)
return process_legacy(data)
```

Replace directly unless compatibility is required.

## Noisy comments

```python
# Increment count
count += 1
```

Delete.

## String state mutation

```python
job.status = "done"
```

Use explicit enum and transition function.
