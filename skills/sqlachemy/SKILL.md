---
name: sqlachemy
description: >-
  SQLAlchemy v2 best practices and conventions.
  Use when working with SQLAlchemy ORM or Core, database models, queries, sessions,
  or migrations. Activate when code imports sqlalchemy, uses Mapped, mapped_column,
  select, or session operations. Follow these guidelines always.
metadata:
  version: "3.0"
---

# SQLAlchemy v2 Best Practices

## Core Principles

- **Always use v2 API patterns** — avoid deprecated v1.x query methods.
- Use `Mapped[T]` and `mapped_column()` for type-safe ORM models.
- Use async sessions only when your application is async (FastAPI, async web servers).
- Use sync sessions for traditional applications, scripts, and simpler use cases.
- Use `select()` statements instead of legacy `Query` interface.

## Model Definition

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, ForeignKey
from typing import Optional

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    email: Mapped[Optional[str]]  # Nullable column

    posts: Mapped[list["Post"]] = relationship(back_populates="user")

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    title: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="posts")
```

## Session Management

### Sync Sessions (Default)

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

engine = create_engine("postgresql://user:pass@localhost/db")
SessionLocal = sessionmaker(engine)

def get_session() -> Session:
    with SessionLocal() as session:
        yield session
```

### Async Sessions (For async applications only)

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

## Query Patterns (v2 Style)

### Select Queries

```python
from sqlalchemy import select

# Simple select
stmt = select(User).where(User.name == "Alice")
result = session.execute(stmt)
user = result.scalar_one_or_none()

# Multiple results
stmt = select(User).where(User.email.isnot(None))
result = session.execute(stmt)
users = result.scalars().all()

# For async, add await:
result = await session.execute(stmt)

# Joins
stmt = select(User).join(User.posts).where(Post.title.like("%SQL%"))
result = session.execute(stmt)
users = result.scalars().unique().all()
```

### Insert/Update/Delete

```python
# Insert
new_user = User(name="Bob", email="bob@example.com")
session.add(new_user)
session.commit()

# Update
stmt = select(User).where(User.id == 1)
user = session.execute(stmt).scalar_one()
user.email = "newemail@example.com"
session.commit()

# Delete
session.delete(user)
session.commit()

# For async, await commit/execute
```

## Relationships and Loading

```python
from sqlalchemy.orm import selectinload, joinedload

# selectinload (separate query — preferred for collections)
stmt = select(User).options(selectinload(User.posts))

# joinedload (single query with JOIN — preferred for single relationships)
stmt = select(User).options(joinedload(User.posts))
```

## DDD Infrastructure Patterns

When using DDD / hexagonal architecture, SQLAlchemy is an **infrastructure concern** — a driven adapter implementing domain-defined repository protocols.

### ORM Models vs Domain Models

ORM models (`infrastructure/persistence/models/`) are **separate** from domain entities (`domain/aggregates/`). Domain models have no SQLAlchemy imports. Use mappers to convert between them.

```python
# infrastructure/persistence/mappers/order.py
class OrderMapper:
    @staticmethod
    def to_domain(model: OrderModel) -> Order:
        return Order.reconstitute(
            id=OrderId(model.id),
            status=OrderStatus(model.status),
            items=[ItemMapper.to_domain(i) for i in model.items],
        )

    @staticmethod
    def to_persistence(entity: Order) -> OrderModel:
        return OrderModel(
            id=entity.id.value,
            status=entity.status.value,
            items=[ItemMapper.to_persistence(i) for i in entity.items],
        )
```

### Repository Implementation

Repositories implement domain-defined Protocols. They use SQLAlchemy sessions internally but return domain models, never ORM models.

```python
# infrastructure/persistence/repositories/order.py
from domain.repositories.order import OrderRepository  # Protocol

class SqlAlchemyOrderRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get(self, order_id: OrderId) -> Order:
        stmt = select(OrderModel).where(OrderModel.id == order_id.value)
        result = await self._session.execute(stmt)
        model = result.scalar_one_or_none()
        if model is None:
            raise OrderNotFound(order_id)
        return OrderMapper.to_domain(model)

    async def add(self, order: Order) -> None:
        model = OrderMapper.to_persistence(order)
        self._session.add(model)

    async def remove(self, order_id: OrderId) -> None:
        stmt = select(OrderModel).where(OrderModel.id == order_id.value)
        model = (await self._session.execute(stmt)).scalar_one()
        await self._session.delete(model)
```

### Unit of Work with SQLAlchemy Session

The session's transaction boundaries map naturally to the Unit of Work pattern.

```python
# infrastructure/persistence/uow.py
class SqlAlchemyUnitOfWork:
    def __init__(self, session_factory: async_sessionmaker) -> None:
        self._session_factory = session_factory

    async def __aenter__(self) -> "SqlAlchemyUnitOfWork":
        self._session = self._session_factory()
        self.orders = SqlAlchemyOrderRepository(self._session)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if exc_type:
            await self._session.rollback()
        await self._session.close()

    async def commit(self) -> None:
        await self._session.commit()
```

## Error Handling

```python
from sqlalchemy.exc import IntegrityError, NoResultFound

try:
    session.commit()  # or await session.commit()
except IntegrityError:
    session.rollback()  # or await session.rollback()
    # Handle constraint violation
```

## Anti-Patterns to Avoid

- `session.query(User).filter_by(...)` — use `select(User).where(...)` (v2 style)
- `name = Column(String(50))` — use `name: Mapped[str] = mapped_column(String(50))`
- Exposing ORM models from repositories (return domain models in DDD)
- Business logic in repositories (repositories are persistence only)
- Creating sessions outside the composition root / dependency provider

## Key Reminders

- Always use `select()` for queries, never `session.query()`.
- Use `Mapped[T]` annotations for all model attributes.
- Prefer `scalar_one()`, `scalar_one_or_none()`, `scalars().all()` for results.
- Use `relationship()` with `back_populates` for bidirectional relationships.
- Use sync sessions by default; only use async if your entire application is async.
- Async requires `await` for `execute()`, `commit()`, `rollback()`, etc.
- In DDD: ORM models are infrastructure, not domain. Map between them explicitly.
