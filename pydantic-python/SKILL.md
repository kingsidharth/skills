---
name: pydantic-python
description: Pydantic v2 data validation, serialization, BaseModel, Field, validators, TypeAdapter, validate_call, ConfigDict, model_dump, model_validate, JSON Schema, custom types, SQLAlchemy integration, pydantic-settings, pydantic-extra-types. Use when writing Python models, schemas, API request/response types, config objects, CLI args, ORM layers, or any data validation. Also trigger when user has manual dict/JSON serialization that should use Pydantic, or raw dataclasses that need validation.
---

# Pydantic (Python)

## Core philosophy: Pydantic as a membrane

Pydantic models are the validation membrane between layers of your application. Every function boundary, API endpoint, queue message, DB row, or config file should pass through a Pydantic model. This eliminates manual serialization and ad-hoc dict construction.

```python
# WRONG — manual serialization everywhere
def process(data: dict) -> dict:
    name = data.get("name", "")
    age = int(data.get("age", 0))
    return {"name": name, "age": age, "valid": True}

# RIGHT — Pydantic as membrane
def process(input: UserInput) -> UserOutput:
    return UserOutput(name=input.name, age=input.age, valid=True)
```

Every function that receives or returns structured data should accept/return a Pydantic model. This applies across all layers: HTTP handlers, background workers, service functions, CLI tools, DB access.

## Installation

```bash
pip install pydantic
# Extras
pip install pydantic[email]           # EmailStr
pip install pydantic-settings         # env/config management
pip install pydantic-extra-types      # phone, currency, timezone, etc.
```

## When to reach for what

| Need | Tool |
|---|---|
| Structured data with validation | `BaseModel` |
| Wrapping a single value (e.g. list, dict) | `RootModel[T]` |
| Validate function args at call site | `@validate_call` |
| Validate a type without a model class | `TypeAdapter(T)` |
| Reusable constrained type | `Annotated[T, Field(...)]` or `Annotated[T, Gt(0)]` |
| App/env config | `pydantic_settings.BaseSettings` |
| ORM ↔ API boundary | `BaseModel` with `ConfigDict(from_attributes=True)` |
| Dynamic schema from runtime data | `create_model(...)` |

## Key model patterns

For full field/model details, see [models-and-fields.md](references/models-and-fields.md).

### BaseModel essentials

- Inherit from `BaseModel`. Fields are type-annotated class attributes.
- `model_validate(data)` — parse dict/object → model instance (class method).
- `model_dump()` — model → dict. Pass `mode='json'` for JSON-safe types.
- `model_dump_json()` — model → JSON string directly (faster than `json.dumps(model_dump())`).
- `model_json_schema()` — generate JSON Schema.
- `model_validate_json(raw)` — parse JSON bytes/str directly (fastest path).
- `model_copy(update={...})` — shallow copy with field overrides.

### Field

`Field()` adds constraints, defaults, aliases, and JSON Schema metadata to fields. Prefer the `Annotated` pattern for reusable types:

```python
PositiveInt = Annotated[int, Field(gt=0)]
ShortStr = Annotated[str, Field(max_length=100)]
```

Key `Field()` params: `default`, `default_factory`, `alias`, `validation_alias`, `serialization_alias`, `gt/ge/lt/le`, `min_length/max_length`, `pattern`, `strict`, `frozen`, `exclude`, `deprecated`.

### ConfigDict

Set on models via `model_config = ConfigDict(...)`. Key options: `strict`, `frozen`, `from_attributes`, `validate_by_name`, `validate_by_alias`, `extra` (`'forbid'|'allow'|'ignore'`), `str_strip_whitespace`, `serialize_by_alias`, `validate_default`.

## Validators and serializers

Four field validator modes: `after` (default, type-safe), `before` (pre-coercion), `plain` (replaces Pydantic's validation), `wrap` (controls whether/how inner validation runs). Model validators run across all fields. See [validators-and-serializers.md](references/validators-and-serializers.md).

## Types

Pydantic validates all standard library types plus its own: `EmailStr`, `HttpUrl`, `AnyUrl`, `SecretStr`, `SecretBytes`, `FilePath`, `DirectoryPath`, `Json[T]`, `ImportString`, `UUID1/4/5`, `conint/confloat/constr` (prefer `Annotated` + `Field` instead). `pydantic-extra-types` adds `PhoneNumber`, `Currency`, `TimeZoneName`, `Color`, `Coordinate`, `MacAddress`, `ISBN`, `PaymentCardNumber`, `SemanticVersion`, `ULID`. See [types-and-config.md](references/types-and-config.md).

## Unions and discriminators

Use `Union[A, B]` with `Discriminator('field_name')` or `Tag('value')` for efficient tagged unions. Pydantic tries each union member left-to-right by default; a discriminator field short-circuits this.

## Serialization

`model_dump()` for Python dict, `model_dump_json()` for JSON string. Control output with `by_alias`, `exclude_none`, `exclude_unset`, `exclude_defaults`, `include`, `exclude`. Custom serializers via `@field_serializer` or `PlainSerializer`/`WrapSerializer` in `Annotated`. See [validators-and-serializers.md](references/validators-and-serializers.md).

## validate_call and TypeAdapter

`@validate_call` decorates any function to validate args from type hints — the membrane pattern for plain functions. `TypeAdapter(T)` wraps any type for standalone `.validate_python()`, `.dump_json()`, `.json_schema()` without a model class. See [validate-call-and-patterns.md](references/validate-call-and-patterns.md).

## SQLAlchemy integration

Keep Pydantic models and SQLAlchemy models separate. Use `ConfigDict(from_attributes=True)` to hydrate Pydantic models from ORM instances. `model_validate(orm_instance)` reads attributes. See [sqlalchemy-integration.md](references/sqlalchemy-integration.md).

## Ecosystem libraries

`clipstick`, `pydantic-argparse` (CLI from models), `redis-om-python` (Redis ODM), `FastDepends` (dependency injection). See [ecosystem.md](references/ecosystem.md).

## Error handling

`ValidationError` contains a list of errors. Use `.errors()` for structured list, `.error_count()`, or `str(e)` for human-readable output. Each error has `type`, `loc` (field path), `msg`, `input`, `ctx`. See [Pydantic docs: Error Handling](https://pydantic.dev/docs/validation/latest/errors/errors/).

## Performance guidance

- `model_validate_json()` is faster than `json.loads()` + `model_validate()` — skips Python dict intermediary.
- Use `TypeAdapter` for hot paths validating non-model types.
- Avoid `model_validate` in tight loops on the same schema — cache the `TypeAdapter` instance.
- `model_dump(mode='json')` avoids a separate JSON encode step.
- Use `Literal`/`Discriminator` on unions to avoid try-each-member overhead.
- `strict=True` at model or field level disables coercion, which is slightly faster and catches bugs.
- Avoid Python-side validators when built-in constraints (`gt`, `max_length`, `pattern`) suffice — Rust-side validation is faster.
