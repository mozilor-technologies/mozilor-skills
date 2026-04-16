---
name: fastapi-python
description: FastAPI and Python coding standards, scalable project structure, testing patterns, and production best practices.
triggers:
  - "fastapi"
  - "python backend"
  - "python api"
  - "python coding standards"
  - "python testing"
  - "python project structure"
  - "python patterns"
---

# FastAPI & Python — Coding Standards, Patterns & Testing

## 1) Project Structure (Domain-Driven)

Organize by **domain, not file type**. This scales naturally as features grow.

```
project/
├── app/
│   ├── main.py                 # Application factory
│   ├── config.py               # Settings (Pydantic Settings)
│   ├── database.py             # DB engine + session factory
│   ├── dependencies.py         # Shared FastAPI dependencies
│   │
│   ├── auth/                   # Auth domain
│   │   ├── __init__.py
│   │   ├── router.py           # FastAPI routes (thin layer)
│   │   ├── service.py          # Business logic
│   │   ├── repository.py       # DB access only
│   │   ├── models.py           # SQLAlchemy ORM models
│   │   ├── schemas.py          # Pydantic request/response schemas
│   │   └── exceptions.py       # Domain-specific exceptions
│   │
│   ├── users/
│   ├── orders/
│   └── shared/
│       ├── exceptions.py
│       ├── middleware.py
│       └── utils/
│
├── tests/
│   ├── conftest.py             # Shared fixtures
│   ├── unit/                   # Pure logic tests (no I/O)
│   ├── integration/            # DB-level tests
│   └── e2e/                    # Full HTTP cycle
│
├── alembic/                    # DB migrations
├── scripts/                    # Management scripts
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml              # Project metadata + deps (uv/poetry)
└── .env.example
```

## 2) Application Factory Pattern

Never create FastAPI instances at module level for production:

```python
# app/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await database.connect()
    await redis.ping()
    yield
    # Shutdown
    await database.disconnect()
    await redis.close()

def create_app() -> FastAPI:
    app = FastAPI(
        title="My API",
        version="1.0.0",
        lifespan=lifespan,
        docs_url=None if settings.env == "production" else "/docs",
    )
    app.include_router(auth_router, prefix="/api/v1/auth")
    app.include_router(users_router, prefix="/api/v1/users")
    app.add_middleware(RequestIdMiddleware)
    app.add_exception_handler(AppException, app_exception_handler)
    return app

app = create_app()
```

## 3) Layered Separation of Concerns

```python
# router.py — HTTP only, no business logic
@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(
    payload: CreateUserRequest,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    return await service.create_user(payload)

# service.py — business logic, no HTTP/DB framework imports
class UserService:
    def __init__(self, repo: UserRepository, mailer: Mailer):
        self.repo = repo
        self.mailer = mailer

    async def create_user(self, payload: CreateUserRequest) -> UserResponse:
        if await self.repo.exists_by_email(payload.email):
            raise EmailAlreadyExistsError(payload.email)
        user = await self.repo.create(payload)
        await self.mailer.send_welcome(user.email)
        return UserResponse.model_validate(user)

# repository.py — DB access only
class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, payload: CreateUserRequest) -> User:
        user = User(**payload.model_dump())
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user
```

## 4) Configuration Management

```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    env: str = "development"
    debug: bool = False
    database_url: str
    redis_url: str
    jwt_secret: str
    jwt_expire_minutes: int = 30

    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

@lru_cache()
def get_settings() -> Settings:
    return Settings()

settings = get_settings()  # Fails fast on missing required vars
```

## 5) Pydantic Schemas Best Practices

```python
from pydantic import BaseModel, EmailStr, Field, model_validator
from datetime import datetime

class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=2, max_length=100)
    password: str = Field(..., min_length=8)

    @model_validator(mode='after')
    def check_name_not_email(self) -> 'CreateUserRequest':
        if self.name == self.email:
            raise ValueError("name must not equal email")
        return self

class UserResponse(BaseModel):
    id: str
    email: str
    name: str
    created_at: datetime

    model_config = {"from_attributes": True}  # ORM mode
```

## 6) Exception Handling

```python
# app/shared/exceptions.py
class AppException(Exception):
    def __init__(self, message: str, status_code: int, code: str):
        self.message = message
        self.status_code = status_code
        self.code = code

class NotFoundError(AppException):
    def __init__(self, resource: str):
        super().__init__(f"{resource} not found", 404, "NOT_FOUND")

class EmailAlreadyExistsError(AppException):
    def __init__(self, email: str):
        super().__init__(f"Email {email} already exists", 409, "EMAIL_EXISTS")

# Exception handler
async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": exc.message}},
    )
```

## 7) Testing Patterns

### Fixture Architecture

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@pytest.fixture(scope="session")
def event_loop_policy():
    return asyncio.DefaultEventLoopPolicy()

@pytest.fixture(scope="session")
async def test_engine():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest.fixture
async def session(test_engine) -> AsyncSession:
    async with AsyncSession(test_engine) as sess:
        yield sess
        await sess.rollback()

@pytest.fixture
async def client(session) -> AsyncClient:
    app = create_app()
    app.dependency_overrides[get_db] = lambda: session
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
```

### Unit Tests — Service Layer

```python
# tests/unit/test_user_service.py
import pytest
from unittest.mock import AsyncMock

async def test_create_user_raises_on_duplicate_email():
    repo = AsyncMock()
    repo.exists_by_email.return_value = True
    mailer = AsyncMock()

    service = UserService(repo=repo, mailer=mailer)

    with pytest.raises(EmailAlreadyExistsError):
        await service.create_user(CreateUserRequest(
            email="test@example.com",
            name="Test User",
            password="secret123"
        ))
    mailer.send_welcome.assert_not_called()
```

### Integration Tests — API Layer

```python
# tests/integration/test_users_api.py
async def test_create_user_returns_201(client, session):
    response = await client.post("/api/v1/users", json={
        "email": "new@example.com",
        "name": "New User",
        "password": "password123",
    })
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "new@example.com"
    assert "id" in data
    assert "password" not in data  # never return password
```

## 8) Python Coding Standards

### Style Rules

- Python 3.11+ required; 3.12 preferred for performance
- Use `uv` for dependency management (faster than pip/poetry)
- `ruff` for linting and formatting — replaces flake8, black, isort
- Type all function signatures; run `mypy` in strict mode

```python
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"
select = ["E", "F", "I", "N", "UP", "ANN", "S", "B", "A"]
ignore = ["ANN101"]

[tool.mypy]
python_version = "3.12"
strict = true
```

### Naming Conventions

```python
# Modules and packages: snake_case
user_service.py

# Classes: PascalCase
class UserRepository: ...

# Functions and variables: snake_case
async def get_user_by_id(user_id: str) -> User: ...

# Constants: UPPER_SNAKE_CASE
MAX_RETRY_ATTEMPTS = 3

# Private: single underscore prefix
def _internal_helper(): ...
```

### Async Patterns

```python
# Always use async/await for I/O
async def get_user_with_orders(user_id: str):
    # Parallel fetches
    user, orders = await asyncio.gather(
        user_repo.get(user_id),
        order_repo.get_by_user(user_id),
    )
    return {"user": user, "orders": orders}

# Context managers for resources
async with httpx.AsyncClient() as client:
    response = await client.get(url)
```

## 9) Quality Gates

```bash
# Run all checks before commit
uv run ruff check .           # linting
uv run ruff format --check .  # formatting
uv run mypy src/              # type checking
uv run pytest tests/ -x       # tests (fail fast)
uv run pytest --cov=app       # coverage
```

Coverage targets: unit ≥ 80%, integration ≥ 60%.

## 10) Performance Patterns

- Use `AsyncSession` (SQLAlchemy async) — never block the event loop
- Use `select_in_loading` for related entities instead of lazy loading
- Add `LIMIT` to all list queries — never `fetchall()` unbounded
- Use `background_tasks.add_task()` for non-critical async operations
- Cache expensive computations with `redis` or `functools.lru_cache`

## References

- [FastAPI Best Practices 2026](https://fastlaunchapi.dev/blog/fastapi-best-practices-production-2026)
- [FastAPI Production Structure 2026](https://dev.to/thesius_code_7a136ae718b7/production-ready-fastapi-project-structure-2026-guide-b1g)
- [FastAPI Official Docs](https://fastapi.tiangolo.com)
- [Pydantic v2 Docs](https://docs.pydantic.dev/latest/)
