---
name: fastapi
description: >-
  FastAPI best practices and conventions with Pydantic v2.
  Use when building REST APIs with FastAPI, handling routes, schemas, dependencies,
  middleware, or configuration. Activate when code imports fastapi or uses
  APIRouter, Depends, BaseModel, or BaseSettings. Follow these guidelines always.
metadata:
  version: "3.0"
---

# FastAPI Best Practices

Modern FastAPI with **Pydantic v2** for building production-ready RESTful APIs.

**Remember**: FastAPI + Pydantic v2 provides automatic validation, serialization, and documentation. Leverage these features — don't fight them.

## Application Factory & Settings

- **Always use application factory pattern** to create FastAPI instances.
- **Use Pydantic v2 `BaseSettings`** for configuration (never raw `os.environ`).
- Load settings with `model_config = SettingsConfigDict(env_file=".env")`.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=False)

    database_url: str
    debug: bool = False
```

## Dependency Injection with Depends()

FastAPI's `Depends()` is a **parameter injection** system. It inspects function signatures, resolves the dependency tree, and provides results as arguments. It is request-scoped by default.

### Core Rules

- **Always use `Annotated[Type, Depends(...)]`** — never `= Depends(...)` as default arguments.
- **Return abstractions, not concretions** — dependency functions should return `Protocol`/`ABC` types, with the concrete implementation hidden inside.
- **Define dependency functions in a dedicated `dependencies/` module** — this is the composition root (the only place that knows about concrete implementations).
- **Dependencies with `yield`** manage resource lifecycle (sessions, connections). Cleanup runs after the response is sent.

### Scoping

| Scope | Mechanism | Use For |
|-------|-----------|---------|
| Per-request | `Depends()` (default) | DB sessions, UoW, repositories |
| Singleton | `@lru_cache` on provider function | Settings, config |
| App-scoped | `lifespan` + `app.state` | Connection pools, HTTP clients |

### Example

```python
from typing import Annotated, AsyncGenerator

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session

SessionDep = Annotated[AsyncSession, Depends(get_session)]

@router.get("/users/{user_id}")
async def get_user(user_id: int, session: SessionDep):
    ...
```

### Dependency Chains (DDD / Layered Architecture)

When using DDD or layered architecture, chain dependencies to maintain layer separation. The composition root (`dependencies/`) wires concrete implementations to abstract interfaces:

```python
# dependencies/uow.py — composition root
async def get_uow(session: SessionDep) -> AsyncGenerator[AbstractUnitOfWork, None]:
    yield SqlAlchemyUnitOfWork(session)

UnitOfWorkDep = Annotated[AbstractUnitOfWork, Depends(get_uow)]

# routes/orders.py — knows only abstractions
@router.post("/orders")
async def create_order(body: CreateOrderRequest, uow: UnitOfWorkDep):
    await create_order_handler(CreateOrderCommand(...), uow)
```

### Testing with Dependency Overrides

```python
app.dependency_overrides[get_uow] = lambda: FakeUnitOfWork()
# Reset after test
app.dependency_overrides.clear()
```

Override at the highest level you want to stub — sub-dependencies below are skipped.

## Routes & Endpoints

- **Type everything** with Pydantic models: request bodies, responses, query params, path params.
- **Use async** for all endpoints and I/O operations.
- **Organize with `APIRouter`** by feature/domain.
- **Routes are thin** — validate input, dispatch to service/handler, return response. No business logic.
- Use status codes from `fastapi.status` module.

```python
from typing import Annotated
from fastapi import Query

@router.get("/items", response_model=ItemListResponse)
async def list_items(
    params: Annotated[ItemQueryParams, Query()],
    session: SessionDep,
):
    return await item_service.list_items(session, params)
```

## Pydantic v2 Schema Conventions

### Naming Patterns

| Type | Pattern | Example |
|------|---------|---------|
| Request | `{Action}{Entity}Request` | `CreateUserRequest` |
| Response | `{Entity}Response` | `UserResponse` |
| List Response | `{Entity}ListResponse` | `UserListResponse` |
| Query Params | `{Entity}QueryParams` | `ProductQueryParams` |

### Base Schemas (DRY)

```python
class UserBase(BaseModel):
    email: EmailStr
    username: str

class CreateUserRequest(UserBase):
    password: str

class UserResponse(UserBase):
    id: int
    created_at: datetime
```

## Error Handling

- **Custom exception handlers** that convert domain/application exceptions to HTTP responses.
- **Consistent error format**: `{"error": str, "code": str, "details": dict | None}`.
- **Never expose tracebacks in production.**
- Route-level: `HTTPException` for expected HTTP errors. App-level: exception handlers for domain errors.

```python
@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.message, "code": exc.code},
    )
```

## Architecture

### Simple CRUD APIs

```
project/
├── main.py              # App factory
├── config.py            # Pydantic BaseSettings
├── dependencies.py      # Dependency providers
├── routes/              # APIRouter modules by feature
├── services/            # Business logic
├── models/              # SQLAlchemy/DB models
└── schemas/             # Pydantic request/response DTOs
```

### With DDD / Hexagonal Architecture

When complexity warrants it, follow the DDD skill's project structure. FastAPI's role is the **driving adapter** (primary adapter) — it translates HTTP into application commands/queries:

```
contexts/{name}/
├── api/                 # DRIVING ADAPTER (FastAPI)
│   ├── routes/          # Thin: validate → dispatch command/query → return response
│   ├── dependencies/    # COMPOSITION ROOT: wires concrete → abstract
│   ├── schemas/         # Pydantic request/response DTOs (NOT domain models)
│   └── middleware/      # Error translation, auth, logging
├── domain/              # Protocols, aggregates, value objects (no FastAPI imports)
├── application/         # Commands, queries, handlers (no FastAPI imports)
└── infrastructure/      # Concrete implementations (repositories, adapters)
```

**Key rule:** Domain and application layers must never import from FastAPI. FastAPI is an infrastructure concern.

## OpenAPI Documentation

- **Use `tags`** to group related endpoints.
- **Add `summary` and `description`** to all endpoints.
- **Document response models** with `response_model` and `responses`.
- **Include examples** in Pydantic schemas with `json_schema_extra`.

```python
app = FastAPI(
    title="My API",
    version="1.0.0",
    openapi_tags=[
        {"name": "users", "description": "User management"},
    ],
)

class CreateUserRequest(BaseModel):
    email: EmailStr
    username: str
    password: str

    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {"email": "user@example.com", "username": "john_doe", "password": "securepass123"}
            ]
        },
    )
```

## Configuration & Observability

- **Structured logging** with context (request_id, user_id, correlation_id).
- **OpenTelemetry** for tracing and metrics in production.
- **CORS middleware** configured for allowed origins (never `allow_origins=["*"]` in prod).
- **Health check endpoints** (`/health`, `/ready`) for orchestration.

## Prohibited Patterns

- Raw `os.environ` access (use Settings)
- `= Depends(...)` as default arguments (use `Annotated`)
- Global database sessions or mutable state
- Synchronous I/O in async endpoints
- Missing type annotations on route parameters
- Exposing tracebacks in production errors
- Business logic in route handlers (delegate to services/handlers)
- Injecting concrete infrastructure types directly in routes (return abstractions from dependency functions)

## Development Server

```bash
# With uv (recommended)
uv run uvicorn project.main:app --reload --host 0.0.0.0 --port 8000
```

## Testing

- Use `TestClient` (sync) or `httpx.AsyncClient` (async) for integration tests.
- Override dependencies with `app.dependency_overrides` for test isolation.
- Use `pytest` with `pytest-asyncio` for async test support.
- For DDD projects, override at the UoW/repository level to avoid hitting real infrastructure.
