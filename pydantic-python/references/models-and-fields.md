# Models and Fields

## BaseModel

Inherit from `BaseModel`. Fields are annotated class attributes. Pydantic validates input data and guarantees the output conforms to declared types.

### Key class methods and properties

- `model_validate(obj)` — validate dict, mapping, or arbitrary object → model. Set `from_attributes=True` in `ConfigDict` to read from object attributes (ORM rows, dataclass instances).
- `model_validate_json(json_str)` — parse JSON bytes/str directly (faster path, skips Python dict).
- `model_dump()` — model → dict. `mode='json'` converts to JSON-safe Python types.
- `model_dump_json()` — model → JSON string.
- `model_json_schema()` — JSON Schema dict.
- `model_copy(update={...})` — shallow copy with overrides (replaces v1 `.copy()`).
- `model_fields` — class-level dict of `FieldInfo` objects.
- `model_rebuild()` — rebuild schema for forward references.
- `model_construct(**fields)` — create instance **without validation** (for trusted data / performance).

### Nested models

Models compose naturally. A field annotated as another `BaseModel` subclass validates recursively.

### RootModel

For wrapping a single value (list, dict, scalar):

```python
class Items(RootModel[list[Item]]):
    pass

items = Items.model_validate([{"name": "a"}, {"name": "b"}])
```

Access via `.root` attribute. Iterable if the root type is iterable.

### Generic models

`BaseModel` can be parameterized with `TypeVar`:

```python
T = TypeVar('T')

class Response(BaseModel, Generic[T]):
    data: T
    status: int

r = Response[list[int]](data=[1, 2], status=200)
```

### Dynamic model creation

`create_model('Name', field_name=(type, default), ...)` creates a model class at runtime. Useful for schema-driven or plugin systems.

### model_config (ConfigDict)

Set `model_config = ConfigDict(...)` on any model. Key options:

- `strict=True` — no coercion (e.g. `"5"` won't become `int(5)`).
- `frozen=True` — immutable instances (hashable).
- `from_attributes=True` — enable `model_validate` from arbitrary objects by reading attributes.
- `extra='forbid'` — raise error on unknown fields. `'allow'` stores them in `__pydantic_extra__`. `'ignore'` silently drops them.
- `validate_by_name=True` — allow field name alongside alias during validation.
- `validate_by_alias=True` — allow alias during validation (True by default).
- `serialize_by_alias=True` — use alias by default in `model_dump()`/`model_dump_json()`.
- `str_strip_whitespace=True` — strip whitespace from all `str` fields.
- `validate_default=True` — validate default values.
- `use_enum_values=True` — store enum `.value` instead of enum member.
- `alias_generator` — callable or `AliasGenerator` to auto-generate aliases (e.g. camelCase).

## Field()

`Field()` is assigned as the default value of a field (or used inside `Annotated`) to configure constraints, metadata, and behavior.

### Constraints by type

**Numeric** (`int`, `float`, `Decimal`): `gt`, `ge`, `lt`, `le`, `multiple_of`.
**String**: `min_length`, `max_length`, `pattern` (regex).
**Collections** (`list`, `set`, `dict`, `frozenset`): `min_length`, `max_length`.
**Decimal**: additionally `max_digits`, `decimal_places`.

### Defaults

- `default=value` or plain assignment.
- `default_factory=callable` for mutable defaults.
- `default_factory` can take a `data` dict argument with already-validated prior fields (field ordering matters).

### Aliases

- `alias='name'` — used for both validation and serialization.
- `validation_alias='name'` — only for input parsing.
- `serialization_alias='name'` — only for output.
- `validation_alias` and `serialization_alias` take priority over `alias`.

### Other Field params

- `frozen=True` — field is immutable after creation.
- `exclude=True` — excluded from `model_dump()` / `model_dump_json()`.
- `deprecated=True` or `deprecated='message'` — emits `DeprecationWarning` on access.
- `strict=True` — per-field strict mode.
- `json_schema_extra` — merge extra keys into generated JSON Schema.

### computed_field

Read-only property included in serialization:

```python
class User(BaseModel):
    first: str
    last: str

    @computed_field
    @property
    def full_name(self) -> str:
        return f"{self.first} {self.last}"
```

Appears in `model_dump()` and JSON Schema but is not an input field.

### The Annotated pattern

Prefer `Annotated[T, Field(...)]` for reusable type definitions. This keeps type hints clean for static analysis:

```python
PositiveInt = Annotated[int, Field(gt=0)]
Email = Annotated[str, Field(pattern=r'^[\w.-]+@[\w.-]+\.\w+$')]

class User(BaseModel):
    age: PositiveInt
    email: Email
```

Constraints from `annotated_types` (`Gt`, `Len`, `Predicate`, etc.) also work inside `Annotated` and are Pydantic-agnostic.

### Private attributes

Use `PrivateAttr` for internal state excluded from validation/serialization. Initialize via `__init_private_attributes__` or `model_post_init`.

```python
class Service(BaseModel):
    url: str
    _client: httpx.Client = PrivateAttr(default_factory=httpx.Client)
```
