---
name: fastapi
description: Build production-ready FastAPI services following the s3rius/FastAPI-template conventions — async-first, feature-modular routing, SQLAlchemy 2.0, Alembic migrations, Pydantic settings, and pytest fixtures.
---

# FastAPI Patterns

Conventions and patterns for FastAPI projects generated from or aligned with [s3rius/FastAPI-template](https://github.com/s3rius/FastAPI-template).

## When to Use This Skill

- Creating new FastAPI endpoints or routers
- Adding database models or migrations
- Wiring up dependencies (sessions, services)
- Configuring application settings
- Writing tests for FastAPI routes or services
- Setting up lifespan events (startup/shutdown)

## Project Structure

```
{project_name}/
├── web/
│   ├── application.py        # FastAPI app factory
│   ├── lifespan.py           # Startup/shutdown lifecycle
│   └── api/
│       ├── router.py         # Aggregates all feature routers
│       ├── monitoring/       # Health check endpoints
│       ├── users/            # Feature module example
│       │   ├── __init__.py
│       │   ├── views.py      # Endpoint handlers
│       │   └── schema.py     # Pydantic request/response schemas
│       └── dummy/
├── db/
│   ├── base.py               # Declarative base
│   ├── meta.py               # SQLAlchemy metadata
│   ├── models/               # ORM model definitions
│   │   └── users.py
│   └── dependencies.py       # get_db_session() dependency
├── services/                 # External integrations (redis, rabbit, kafka)
├── settings.py               # Pydantic BaseSettings
├── log.py                    # Logging configuration
└── __main__.py               # Entry point
tests/
├── conftest.py               # Shared pytest fixtures
└── test_{feature}.py
```

## Routing

### Central Router Aggregation

All feature routers are registered in `web/api/router.py`:

```python
# web/api/router.py
from fastapi.routing import APIRouter
from {project_name}.web.api import monitoring, users, dummy

api_router = APIRouter()
api_router.include_router(monitoring.router, prefix="/monitoring", tags=["monitoring"])
api_router.include_router(users.router, prefix="/users", tags=["users"])
api_router.include_router(dummy.router, prefix="/dummy", tags=["dummy"])
```

### Feature Module Structure

Each feature module exports a `router` from its `__init__.py`:

```python
# web/api/users/__init__.py
from {project_name}.web.api.users.views import router

__all__ = ["router"]
```

```python
# web/api/users/views.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from {project_name}.db.dependencies import get_db_session
from {project_name}.web.api.users.schema import UserResponse, CreateUserRequest

router = APIRouter()

@router.get("/", response_model=list[UserResponse])
async def get_users(
    db: AsyncSession = Depends(get_db_session),
) -> list[UserResponse]:
    ...

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    payload: CreateUserRequest,
    db: AsyncSession = Depends(get_db_session),
) -> UserResponse:
    ...
```

## Database

### SQLAlchemy 2.0 Models

```python
# db/models/users.py
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column
from {project_name}.db.base import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(200))
    email: Mapped[str] = mapped_column(String(200), unique=True)
```

```python
# db/base.py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

### Session Dependency

```python
# db/dependencies.py
from typing import AsyncGenerator
from fastapi import Request
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db_session(request: Request) -> AsyncGenerator[AsyncSession, None]:
    session: AsyncSession = request.app.state.db_session_factory()
    try:
        yield session
    finally:
        await session.commit()
        await session.close()
```

### DAO Pattern

Database access logic lives in DAO (Data Access Object) classes, not in views:

```python
# db/dao/users_dao.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from {project_name}.db.models.users import User

class UserDAO:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_all(self, limit: int = 100, offset: int = 0) -> list[User]:
        result = await self.session.execute(
            select(User).limit(limit).offset(offset)
        )
        return list(result.scalars().fetchall())

    async def get_by_email(self, email: str) -> User | None:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalars().first()

    async def create(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self.session.add(user)
        await self.session.flush()
        return user
```

Inject DAOs as dependencies in views:

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from {project_name}.db.dependencies import get_db_session
from {project_name}.db.dao.users_dao import UserDAO

async def get_user_dao(db: AsyncSession = Depends(get_db_session)) -> UserDAO:
    return UserDAO(db)

@router.get("/")
async def get_users(dao: UserDAO = Depends(get_user_dao)) -> list[UserResponse]:
    return await dao.get_all()
```

## Settings

Use Pydantic `BaseSettings` with project-prefixed environment variables:

```python
# settings.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    host: str = "127.0.0.1"
    port: int = 8000
    db_host: str = "localhost"
    db_port: int = 5432
    db_user: str = "postgres"
    db_pass: str = "postgres"
    db_base: str = "{project_name}"

    @property
    def db_url(self) -> str:
        return (
            f"postgresql+asyncpg://{self.db_user}:{self.db_pass}"
            f"@{self.db_host}:{self.db_port}/{self.db_base}"
        )

    class Config:
        env_file = ".env"
        env_prefix = "{PROJECT_NAME}_"
        env_file_encoding = "utf-8"

settings = Settings()
```

## Application Factory & Lifespan

```python
# web/application.py
from fastapi import FastAPI
from {project_name}.web.lifespan import lifespan
from {project_name}.web.api.router import api_router

def get_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(api_router, prefix="/api")
    return app
```

```python
# web/lifespan.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from {project_name}.settings import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    engine = create_async_engine(str(settings.db_url))
    app.state.db_engine = engine
    app.state.db_session_factory = async_sessionmaker(engine, expire_on_commit=False)

    yield

    # Shutdown
    await app.state.db_engine.dispose()
```

## Pydantic Schemas

Keep request/response schemas in a `schema.py` file per feature module. Never reuse ORM models directly as response schemas:

```python
# web/api/users/schema.py
from pydantic import BaseModel, EmailStr

class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

    model_config = {"from_attributes": True}
```

## Testing

### conftest.py Fixtures

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from {project_name}.web.application import get_app
from {project_name}.db.base import Base

@pytest.fixture(scope="session")
def anyio_backend() -> str:
    return "asyncio"

@pytest.fixture
async def test_app():
    app = get_app()
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as client:
        yield client

@pytest.fixture
async def dbsession() -> AsyncSession:
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    factory = async_sessionmaker(engine, expire_on_commit=False)
    async with factory() as session:
        yield session
    await engine.dispose()
```

### Writing Tests

```python
# tests/test_users.py
import pytest
from httpx import AsyncClient

@pytest.mark.anyio
async def test_get_users_empty(test_app: AsyncClient) -> None:
    response = await test_app.get("/api/users/")
    assert response.status_code == 200
    assert response.json() == []

@pytest.mark.anyio
async def test_create_user(test_app: AsyncClient) -> None:
    response = await test_app.post("/api/users/", json={"name": "Alice", "email": "alice@example.com"})
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Alice"
    assert "id" in data
```

## Migrations (Alembic)

- Migration files live at the project root under `alembic/versions/`
- Always generate migrations with a descriptive message:
  ```bash
  alembic revision --autogenerate -m "add users table"
  ```
- Never edit the generated migration manually unless correcting autogenerate errors
- Run migrations on startup in CI/CD — never against production without review

## Best Practices

1. **Async everywhere** — all database calls, external service calls, and I/O must be `async`
2. **Never use ORM models as response schemas** — always define separate Pydantic schemas
3. **DAOs for all DB access** — no raw queries in views; views call DAOs only
4. **Settings from environment** — no hardcoded values; use `settings.py` with `.env`
5. **One router per feature** — keep `views.py` focused on a single business domain
6. **Test with real DB fixtures** — use SQLite in-memory for unit tests, avoid mocking the session
7. **Lifespan for resource management** — initialise and dispose engines, pools, and connections in `lifespan.py`
8. **Prefix all env vars** — use `{PROJECT_NAME}_` prefix to avoid collisions
