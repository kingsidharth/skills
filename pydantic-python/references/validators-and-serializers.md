# Validators and Serializers

## Field validators

Four modes, usable via `Annotated` pattern or `@field_validator` decorator:

### After (default)

Runs after Pydantic's built-in validation. Receives the already-validated, correctly-typed value. Safest and most common.

```python
# Annotated pattern (preferred for reusable types)
def check_positive(v: int) -> int:
    if v <= 0:
        raise ValueError('must be positive')
    return v

PositiveInt = Annotated[int, AfterValidator(check_positive)]

# Decorator pattern (for model-specific logic)
class Order(BaseModel):
    quantity: int

    @field_validator('quantity')
    @classmethod
    def must_be_positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError('must be positive')
        return v
```

### Before

Runs before Pydantic's internal parsing. Receives raw input (`Any`). Useful for coercion or normalization.

```python
# Ensure a value is always wrapped in a list
EnsureList = Annotated[list[int], BeforeValidator(
    lambda v: v if isinstance(v, list) else [v]
)]
```

### Plain

Replaces Pydantic's validation entirely. No built-in type checking runs after this.

### Wrap

Receives the value and a `handler` callable. You decide whether/when to call the inner validation:

```python
def maybe_validate(v: Any, handler: ValidatorFunctionWrapHandler) -> int:
    if isinstance(v, str) and v == 'skip':
        return 0
    return handler(v)  # run normal validation

FlexInt = Annotated[int, WrapValidator(maybe_validate)]
```

## Model validators

Run across the entire model. Three modes:

### model before

Receives raw input before any field validation. Must return a dict (or the input type). Useful for reshaping payloads.

```python
class Payment(BaseModel):
    @model_validator(mode='before')
    @classmethod
    def flatten_nested(cls, data: Any) -> Any:
        if isinstance(data, dict) and 'details' in data:
            data.update(data.pop('details'))
        return data

    amount: float
    currency: str
```

### model after

Receives the fully constructed model instance. Can cross-validate fields.

```python
class DateRange(BaseModel):
    start: date
    end: date

    @model_validator(mode='after')
    def end_after_start(self) -> 'DateRange':
        if self.end < self.start:
            raise ValueError('end must be >= start')
        return self
```

### model wrap

Like field wrap — receives input and handler, you control if/when full model validation runs.

## Raising validation errors

In validators, raise `ValueError` or `AssertionError` for simple messages. For structured errors with custom types/contexts, raise `PydanticCustomError`:

```python
from pydantic_core import PydanticCustomError

raise PydanticCustomError(
    'too_short',
    '{field_name} must be at least {min_length} chars',
    {'field_name': 'username', 'min_length': 3},
)
```

## Validation context

Pass arbitrary context via `model_validate(data, context={'key': 'val'})`. Access in validators via `info.context`:

```python
@field_validator('price')
@classmethod
def check_price(cls, v: float, info: ValidationInfo) -> float:
    currency = info.context.get('currency', 'USD') if info.context else 'USD'
    # ... apply currency-specific logic
    return v
```

## Field serializers

Customize how a field is serialized. Two modes:

### Plain serializer

Replaces default serialization entirely.

```python
class Event(BaseModel):
    dt: Annotated[datetime, PlainSerializer(lambda v: v.isoformat(), return_type=str)]
```

### Wrap serializer

Receives the value and `handler` (default serializer). Can modify before/after.

```python
@field_serializer('dt')
def serialize_dt(self, v: datetime, _info) -> str:
    return v.strftime('%Y-%m-%d')
```

### Annotated vs decorator

- `Annotated[T, PlainSerializer(...)]` — reusable across models.
- `@field_serializer('field_name')` — model-specific, can access `self`.

## Model serializers

Customize entire model output:

```python
class Response(BaseModel):
    data: dict
    meta: dict

    @model_serializer(mode='wrap')
    def custom_dump(self, handler: SerializerFunctionWrapHandler) -> dict:
        result = handler(self)
        result['_version'] = '2.0'
        return result
```

## Key serialization params

`model_dump()` / `model_dump_json()` accept:
- `by_alias=True` — use alias names.
- `exclude_none=True` — drop `None` values.
- `exclude_unset=True` — drop fields not explicitly set.
- `exclude_defaults=True` — drop fields matching their default.
- `include={'field1', 'field2'}` — whitelist fields.
- `exclude={'field3'}` — blacklist fields.
- `mode='json'` (for `model_dump` only) — produce JSON-compatible Python types.
