# Types and Config

## Built-in types

Pydantic validates all Python standard library types: `int`, `float`, `bool`, `str`, `bytes`, `list`, `tuple`, `set`, `frozenset`, `dict`, `None`, `date`, `datetime`, `time`, `timedelta`, `Decimal`, `UUID`, `Path`, `Enum`, `IPv4Address/IPv6Address`, `Pattern`, `Sequence`, `Mapping`, etc.

Strictness and coercion rules vary by type. In lax mode (default), `"5"` becomes `int(5)`. In strict mode, only exact types pass.

## Pydantic-specific types

From `pydantic`:

- `EmailStr` — validated email (requires `pydantic[email]`).
- `NameEmail` — `"Name <email>"` format.
- `HttpUrl`, `AnyUrl`, `AnyHttpUrl`, `FileUrl`, `FtpUrl` — URL types with scheme validation.
- `SecretStr`, `SecretBytes` — values hidden from `repr()`/logs, accessed via `.get_secret_value()`.
- `FilePath`, `DirectoryPath`, `NewPath` — validated filesystem paths.
- `Json[T]` — accepts a JSON string, parses it, then validates against `T`.
- `ImportString` — dotted import path, resolves to the actual Python object.
- `conint`, `confloat`, `constr`, `conlist`, `conset` — constrained types (legacy; prefer `Annotated` + `Field`).
- `PositiveInt`, `NegativeInt`, `NonNegativeInt`, `NonPositiveInt`, `StrictInt`, etc.
- `FiniteFloat`, `StrictFloat`, `StrictBool`, `StrictStr`, `StrictBytes`.
- `PastDate`, `FutureDate`, `PastDatetime`, `FutureDatetime`, `AwareDatetime`, `NaiveDatetime`.
- `Base64Bytes`, `Base64Str`, `Base64UrlBytes`, `Base64UrlStr`.
- `UUID1`, `UUID3`, `UUID4`, `UUID5`.
- `ByteSize` — parses `"1.5GB"` → bytes as int.

## pydantic-extra-types

Install: `pip install pydantic-extra-types`.

- `PhoneNumber` — E.164 validated (requires `phonenumbers`).
- `Currency` — ISO 4217 currency codes.
- `TimeZoneName` — IANA timezone strings.
- `Color` — CSS color strings, RGB tuples, hex codes.
- `Coordinate`, `Latitude`, `Longitude` — geographic coordinates.
- `MacAddress` — MAC address validation.
- `ISBN` — ISBN-10/ISBN-13.
- `PaymentCardNumber` — Luhn-validated card numbers.
- `CountryAlpha2`, `CountryAlpha3`, `CountryNumericCode` — ISO 3166.
- `LanguageAlpha2`, `LanguageName` — ISO 639.
- `SemanticVersion` — semver strings.
- `ULID` — Universally Unique Lexicographically Sortable Identifier.
- `PendulumDateTime` — pendulum datetime integration.
- `RoutingNumber` — ABA routing numbers.
- `ScriptCode` — ISO 15924 script codes.

## Custom types via Annotated

Compose validators, serializers, and JSON Schema overrides:

```python
TruncatedFloat = Annotated[
    float,
    AfterValidator(lambda x: round(x, 1)),
    PlainSerializer(lambda x: f'{x:.1e}', return_type=str),
    WithJsonSchema({'type': 'string'}, mode='serialization'),
]
```

## Custom types via __get_pydantic_core_schema__

For third-party types or complex validation, implement `__get_pydantic_core_schema__` as a classmethod or as `Annotated` metadata:

```python
class MyType:
    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler):
        return core_schema.no_info_plain_validator_function(cls._validate)

    @classmethod
    def _validate(cls, v):
        # custom validation logic
        return cls(v)
```

For types you don't control, use `Annotated` with a marker class that implements `__get_pydantic_core_schema__`.

## TypeAdapter

Wraps any type for standalone validation/serialization without a model:

```python
ta = TypeAdapter(list[int])
ta.validate_python(['1', '2', '3'])  # [1, 2, 3]
ta.dump_json([1, 2, 3])              # b'[1,2,3]'
ta.json_schema()                     # {'type': 'array', 'items': {'type': 'integer'}}
```

Cache `TypeAdapter` instances — construction builds the schema (expensive). Validation calls are fast.

## Strict mode

Enable per-field (`Field(strict=True)`), per-model (`ConfigDict(strict=True)`), or per-call (`model_validate(data, strict=True)`). In strict mode, no coercion happens — `"5"` is not a valid `int`.

## Unions

Default: left-to-right matching ("smart" mode tries best match first). For tagged unions, use a discriminator:

```python
class Cat(BaseModel):
    type: Literal['cat']
    meows: int

class Dog(BaseModel):
    type: Literal['dog']
    barks: int

Pet = Annotated[Cat | Dog, Discriminator('type')]
```

Also supports callable discriminators and `Tag()` for types without a common field.
