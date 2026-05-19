# validate_call and Patterns

## @validate_call

Decorates any function to validate arguments from type hints before calling. The membrane pattern for plain functions:

```python
from pydantic import validate_call

@validate_call
def send_email(to: EmailStr, subject: str, body: str = '') -> None:
    ...
```

Arguments are validated against their annotations. `ValidationError` raised on bad input.

### With constraints

Use `Annotated` + `Field` on parameters:

```python
@validate_call
def paginate(
    page: Annotated[int, Field(ge=1)] = 1,
    size: Annotated[int, Field(ge=1, le=100)] = 20,
) -> ...:
    ...
```

### Config

Pass `config=ConfigDict(...)` to customize:

```python
@validate_call(config=ConfigDict(strict=True))
def strict_fn(x: int) -> int:
    return x
```

### Async functions

Works with `async def`. The decorator preserves async nature.

### Access original

The undecorated function is at `.raw_function`:

```python
send_email.raw_function(to='not-validated', ...)
```

### Limitations

- Raises `ValidationError`, not `TypeError` — callers expecting `TypeError` from bad args will break.
- Some overhead from model creation on each call (though schemas are cached).

## Pydantic as membrane — the pattern

### API endpoints (FastAPI)

FastAPI uses Pydantic natively. Request/response models are the membrane:

```python
@app.post("/users", response_model=UserRead)
async def create_user(data: UserCreate) -> UserRead:
    row = UserRow(**data.model_dump())
    session.add(row)
    await session.commit()
    return UserRead.model_validate(row)
```

### Service layer

Internal service functions should also accept/return models:

```python
def calculate_invoice(order: OrderInput) -> InvoiceOutput:
    ...  # business logic
    return InvoiceOutput(...)
```

Not:
```python
def calculate_invoice(items: list, tax_rate: float, ...) -> dict:
    ...
```

### Queue / event consumers

Validate messages at the boundary:

```python
def handle_event(raw: bytes) -> None:
    event = OrderEvent.model_validate_json(raw)
    process(event)
```

### CLI tools

Use `validate_call` or parse args into a model before processing:

```python
@validate_call
def main(input_path: FilePath, output_dir: DirectoryPath, verbose: bool = False):
    ...
```

## Dynamic models

`create_model()` creates model classes at runtime:

```python
from pydantic import create_model

DynamicUser = create_model('DynamicUser', name=(str, ...), age=(int, 0))
```

Field definitions: `(type, default)`, `(type, Field(...))`, or `(type, FieldInfo)`.

Use cases: plugin systems, schema-driven forms, config-driven validation.

## Dataclasses

`pydantic.dataclasses.dataclass` is a drop-in for `@dataclasses.dataclass` that adds Pydantic validation. Use when you want dataclass semantics (no `.model_dump()`, etc.) with validation. For full Pydantic features, prefer `BaseModel`.

## pydantic-settings

`BaseSettings` reads from environment variables, `.env` files, and other sources:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file='.env', env_prefix='APP_')
    db_url: str
    debug: bool = False
    workers: int = 4
```

Env vars are matched case-insensitively. Nested models use `__` separator (e.g. `APP_DB__HOST`).
