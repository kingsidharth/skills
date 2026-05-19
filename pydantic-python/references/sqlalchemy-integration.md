# SQLAlchemy Integration

## Architecture: separate models

Keep Pydantic models (API/validation layer) and SQLAlchemy models (persistence layer) as distinct classes. Pydantic acts as the membrane at the boundary — validating data going in, shaping data coming out.

```
HTTP Request → Pydantic Input Model → Service Logic → SQLAlchemy Model → DB
DB → SQLAlchemy Model → Pydantic Output Model → HTTP Response
```

## from_attributes

Enable `ConfigDict(from_attributes=True)` to let `model_validate()` read from ORM instance attributes instead of requiring a dict:

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from pydantic import BaseModel, ConfigDict

class Base(DeclarativeBase):
    pass

class UserRow(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    email: Mapped[str]

class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    email: str

# Usage
row = session.get(UserRow, 1)
user = UserRead.model_validate(row)  # reads row.id, row.name, row.email
```

## Handling reserved names

SQLAlchemy reserves `metadata` on `Base`. Use a trailing underscore on the SA column and an alias on the Pydantic field:

```python
class MyRow(Base):
    __tablename__ = 'items'
    id: Mapped[int] = mapped_column(primary_key=True)
    metadata_: Mapped[dict] = mapped_column('metadata', JSON)

class MyItem(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    metadata: dict[str, str] = Field(alias='metadata_')
```

## Input vs output models

Separate create/update schemas from read schemas:

```python
class UserCreate(BaseModel):
    name: str
    email: EmailStr

class UserUpdate(BaseModel):
    name: str | None = None
    email: EmailStr | None = None

class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    email: str
    created_at: datetime
```

Use `UserCreate` to validate incoming data, construct the ORM row from it, then return `UserRead.model_validate(row)`.

## Writing to DB

Convert Pydantic model → dict → ORM instance:

```python
data = UserCreate(name='Alice', email='[email protected]')
row = UserRow(**data.model_dump())
session.add(row)
session.commit()
```

For updates with partial data:

```python
update = UserUpdate(name='Bob')
for key, val in update.model_dump(exclude_unset=True).items():
    setattr(row, key, val)
session.commit()
```

`exclude_unset=True` ensures only explicitly provided fields are updated.

## Relationship loading

If your Pydantic model references nested models that correspond to SA relationships, ensure the relationships are eagerly loaded (via `joinedload`, `selectinload`, or `lazy='joined'`) before calling `model_validate`. Otherwise, accessing a lazy-loaded relationship outside a session context raises `DetachedInstanceError`.

## SQLModel alternative

SQLModel merges Pydantic and SQLAlchemy into a single class, reducing duplication. Trade-off: tighter coupling between validation and persistence layers. Good for simple CRUD; pure Pydantic + SQLAlchemy gives more control for complex domains.
