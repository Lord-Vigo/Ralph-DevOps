# Python Best Practices Knowledge Base

Code quality reference for Python development, covering style, testing, design patterns, performance, logging, documentation, and dependency management.

---

## Table of Contents

1. [Code Style & Formatting](#code-style--formatting)
2. [Testing Practices](#testing-practices)
3. [Design Patterns](#design-patterns)
4. [Performance Optimization](#performance-optimization)
5. [Logging & Observability](#logging--observability)
6. [Error Handling](#error-handling)
7. [Documentation](#documentation)
8. [Dependency Management](#dependency-management)
9. [API Design](#api-design)
10. [Database Practices](#database-practices)

---

## Code Style & Formatting

### PEP 8 Guidelines

```python
"""
PEP 8 Style Guide for Python Code

Key principles:
- Use 4 spaces for indentation
- Maximum line length of 88 characters (Black default)
- Use meaningful variable names
- Constants UPPER_SNAKE_CASE
- Functions lower_snake_case
- Classes CamelCase
- Private attributes _leading_underscore
"""

# Good: Readable, consistent naming
class UserService:
    """Service for managing users."""
    
    MAX_RETRY_COUNT = 3
    DEFAULT_TIMEOUT = 30
    
    def __init__(self, repository: UserRepository):
        self._repository = repository
        self._cache = {}
    
    def get_user_by_id(self, user_id: int) -> User | None:
        """Fetch user by ID from repository."""
        if user_id in self._cache:
            return self._cache[user_id]
        
        user = self._repository.find_one(user_id)
        if user:
            self._cache[user_id] = user
        return user


# Bad: Inconsistent naming
class userservice:  # Class should be CamelCase
    def GET_USER(self):  # Method should be snake_case
        x = 1  # Unclear variable name
        y = 2
```

### Black Code Formatting

```python
# pyproject.toml
[tool.black]
line-length = 88
target-version = ["py311"]
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | build
  | dist
)/
'''

[tool.blackpreview]
# Experimental: Use unstable formatting
string-normalization = true

# Usage: black source/
```

### Ruff Linter

```python
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # pyflakes
    "I",      # isort
    "N",      # pep8-naming
    "UP",     # pyupgrade
    "YTT",    # flake8-2020
    "ASYNC",  # flake8-async
    "C4",     # flake8-comprehensions
    "SIM",    # flake8-simplify
    "RSE",    # flake8-raise
    "PIE",    # flake8-pie
    "T20",    # flake8-print
    "Q",      # flake8-quotes
    "RUF",    # Ruff-specific
]
ignore = [
    "E501",   # Line too long (handled by Black)
    "N802",   # Function name should be lowercase
    "N803",   # Argument name should be lowercase
]

[tool.ruff.lint.isort]
known-first-party = ["app"]

# Run: ruff check source/
```

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

---

## Testing Practices

### Pytest Configuration

```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
minversion = "7.0"
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-ra",           # Show summary for all
    "--strict-markers",
    "--tb=short",
    "--cov=src",
    "--cov-report=term-missing",
    "--cov-report=html",
    "--cov-fail-under=80",
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise NotImplementedError",
    "if TYPE_CHECKING:",
    "if __name__ == .__main__.:",
]
```

### Test Fixtures

```python
import pytest
from datetime import datetime
from unittest.mock import AsyncMock, MagicMock
from sqlalchemy.ext.asyncio import AsyncSession

@pytest.fixture
def mock_db() -> AsyncMock:
    """Create mock database session."""
    return AsyncMock(spec=AsyncSession)

@pytest.fixture
def sample_user() -> dict:
    """Create sample user data."""
    return {
        "id": 1,
        "username": "testuser",
        "email": "test@example.com",
        "is_active": True,
        "created_at": datetime(2024, 1, 1),
    }

@pytest.fixture
def user_repository(mock_db: AsyncMock, sample_user: dict):
    """Create user repository with mocked database."""
    repo = UserRepository(mock_db)
    
    async def mock_find_one(user_id: int):
        if user_id == sample_user["id"]:
            return User(**sample_user)
        return None
    
    mock_db.execute = AsyncMock(side_effect=mock_find_one)
    return repo

@pytest.fixture
async def authenticated_user(
    mock_db: AsyncMock,
    sample_user: dict
) -> User:
    """Create authenticated user fixture."""
    user = User(**sample_user)
    user.is_verified = True
    return user

@pytest.fixture
def override_get_db(mock_db: AsyncMock):
    """Override dependency for database."""
    async def override():
        yield mock_db
    
    return override
```

### Unit Tests

```python
import pytest
from unittest.mock import patch, MagicMock
from datetime import datetime

class TestUserService:
    """Unit tests for UserService."""
    
    @pytest.fixture
    def service(self, user_repository: UserRepository) -> UserService:
        return UserService(repository=user_repository)
    
    def test_get_user_by_id_cached(
        self,
        service: UserService,
        sample_user: dict
    ) -> None:
        """Test that cached users are returned without DB query."""
        # Arrange
        service._cache[1] = User(**sample_user)
        
        # Act
        result = service.get_user_by_id(1)
        
        # Assert
        assert result is not None
        assert result.id == 1
        service._repository.find_one.assert_not_called()
    
    @pytest.mark.asyncio
    async def test_get_user_by_id_not_found(
        self,
        service: UserService
    ) -> None:
        """Test that None is returned for non-existent user."""
        result = service.get_user_by_id(999)
        assert result is None
    
    def test_create_user_validates_email(
        self,
        service: UserService
    ) -> None:
        """Test that invalid email raises validation error."""
        with pytest.raises(ValidationError) as exc_info:
            service.create_user(
                username="test",
                email="invalid-email"
            )
        
        assert "email" in str(exc_info.value).lower()
```

### Integration Tests

```python
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy import text
from datetime import datetime

@pytest.mark.integration
class TestUserAPI:
    """Integration tests for User API endpoints."""
    
    @pytest.fixture
    async def client(self, override_get_db):
        """Create test client with database override."""
        from app.main import app
        from app.dependencies import get_db
        
        app.dependency_overrides[get_db] = override_get_db
        
        async with AsyncClient(
            transport=ASGITransport(app=app),
            base_url="http://test"
        ) as client:
            yield client
    
    @pytest.mark.asyncio
    async def test_create_user(self, client: AsyncClient) -> None:
        """Test user creation endpoint."""
        response = await client.post(
            "/api/users",
            json={
                "username": "newuser",
                "email": "new@example.com",
                "password": "SecurePass123"
            }
        )
        
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "newuser"
        assert "id" in data
        assert "password" not in data
    
    @pytest.mark.asyncio
    async def test_create_user_duplicate_email(
        self,
        client: AsyncClient,
        sample_user: dict
    ) -> None:
        """Test duplicate email is rejected."""
        # Setup: Create user first
        mock_db = MagicMock()
        mock_db.execute = AsyncMock(return_value=MagicMock(
            scalars=MagicMock(first=MagicMock(return_value=sample_user))
        ))
        
        response = await client.post(
            "/api/users",
            json={
                "username": "test",
                "email": sample_user["email"],
                "password": "SecurePass123"
            }
        )
        
        assert response.status_code == 409
    
    @pytest.mark.asyncio
    async def test_health_check(self, client: AsyncClient) -> None:
        """Test health check endpoint."""
        response = await client.get("/health")
        
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"
```

### Mocking

```python
from unittest.mock import AsyncMock, MagicMock, patch
from datetime import datetime

# Async mocking
class MockResponse:
    def __init__(self, status_code: int, data: dict):
        self.status_code = status_code
        self._data = data
    
    def json(self) -> dict:
        return self._data
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, *args):
        pass

@pytest.fixture
def mock_http_client():
    """Mock HTTP client for external API calls."""
    with patch("httpx.AsyncClient") as mock:
        client = AsyncMock()
        client.get = AsyncMock(return_value=MockResponse(200, {"status": "ok"}))
        client.post = AsyncMock(return_value=MockResponse(201, {"id": 1}))
        mock.return_value.__aenter__ = AsyncMock(return_value=client)
        mock.return_value.__aexit__ = AsyncMock(return_value=None)
        yield client

# Property mocking
@pytest.fixture
def mock_datetime():
    """Mock datetime for consistent testing."""
    with patch("app.services.datetime") as mock:
        mock.now = MagicMock(return_value=datetime(2024, 1, 1))
        yield mock
```

---

## Design Patterns

### Repository Pattern

```python
from typing import TypeVar, Generic, Protocol
from abc import ABC, abstractmethod

T = TypeVar("T")

class Repository(Generic[T], Protocol[T]):
    """Repository protocol for data access."""
    
    async def find_one(self, id: int) -> T | None: ...
    async def find_all(self, filters: dict) -> list[T]: ...
    async def create(self, entity: T) -> T: ...
    async def update(self, id: int, entity: T) -> T | None: ...
    async def delete(self, id: int) -> bool: ...

class SQLAlchemyUserRepository:
    """SQLAlchemy implementation of UserRepository."""
    
    def __init__(self, session: AsyncSession):
        self._session = session
    
    async def find_one(self, user_id: int) -> User | None:
        result = await self._session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()
    
    async def find_all(self, filters: dict | None = None) -> list[User]:
        query = select(User)
        if filters:
            for key, value in filters.items():
                query = query.where(getattr(User, key) == value)
        
        result = await self._session.execute(query)
        return list(result.scalars().all())
    
    async def create(self, user: User) -> User:
        self._session.add(user)
        await self._session.commit()
        await self._session.refresh(user)
        return user
    
    async def delete(self, user_id: int) -> bool:
        user = await self.find_one(user_id)
        if user:
            await self._session.delete(user)
            await self._session.commit()
            return True
        return False
```

### Factory Pattern

```python
from abc import ABC, abstractmethod
from typing import Type

class Notification(ABC):
    """Abstract notification class."""
    
    @abstractmethod
    async def send(self, recipient: str, message: str) -> None:
        ...

class EmailNotification(Notification):
    async def send(self, recipient: str, message: str) -> None:
        # Send email
        pass

class SMSNotification(Notification):
    async def send(self, recipient: str, message: str) -> None:
        # Send SMS
        pass

class PushNotification(Notification):
    async def send(self, recipient: str, message: str) -> None:
        # Send push notification
        pass

class NotificationFactory:
    """Factory for creating notification handlers."""
    
    _registry: dict[str, Type[Notification]] = {}
    
    @classmethod
    def register(cls, notification_type: str) -> callable:
        """Decorator to register notification handlers."""
        def decorator(notification_class: Type[Notification]):
            cls._registry[notification_type] = notification_class
            return notification_class
        return decorator
    
    @classmethod
    def create(cls, notification_type: str) -> Notification:
        """Create notification handler by type."""
        if notification_type not in cls._registry:
            raise ValueError(f"Unknown notification type: {notification_type}")
        return cls._registry[notification_type]()

# Registration
@NotificationFactory.register("email")
class EmailNotificationHandler(Notification):
    ...

@NotificationFactory.register("sms")
class SMSNotificationHandler(Notification):
    ...
```

### Service Layer Pattern

```python
from dataclasses import dataclass

@dataclass
class UserCreateRequest:
    """Request to create a new user."""
    username: str
    email: str
    password: str

@dataclass
class UserResponse:
    """Response for user data."""
    id: int
    username: str
    email: str
    is_active: bool

class UserService:
    """Service layer for user business logic."""
    
    def __init__(
        self,
        user_repository: UserRepository,
        password_hasher: PasswordHasher,
        event_bus: EventBus
    ):
        self._repository = user_repository
        self._password_hasher = password_hasher
        self._event_bus = event_bus
    
    async def create_user(self, request: UserCreateRequest) -> UserResponse:
        # Business logic
        existing = await self._repository.find_by_email(request.email)
        if existing:
            raise UserAlreadyExistsError(request.email)
        
        # Hash password
        hashed_password = self._password_hasher.hash(request.password)
        
        # Create user
        user = User(
            username=request.username,
            email=request.email,
            password_hash=hashed_password
        )
        
        created_user = await self._repository.create(user)
        
        # Publish event
        await self._event_bus.publish(UserCreatedEvent(user_id=created_user.id))
        
        return UserResponse(
            id=created_user.id,
            username=created_user.username,
            email=created_user.email,
            is_active=created_user.is_active
        )
```

### Observer Pattern

```python
from abc import ABC, abstractmethod
from typing import Callable
from dataclasses import dataclass
from datetime import datetime

@dataclass
class DomainEvent:
    """Base domain event."""
    occurred_at: datetime = dataclass.field(default_factory=datetime.now)

@dataclass
class UserCreatedEvent(DomainEvent):
    user_id: int
    email: str

class EventBus:
    """Simple event bus for domain events."""
    
    def __init__(self):
        self._handlers: dict[type[DomainEvent], list[Callable]] = {}
    
    def subscribe(
        self, 
        event_type: type[DomainEvent]
    ) -> callable:
        """Decorator to subscribe to events."""
        def decorator(handler: Callable):
            if event_type not in self._handlers:
                self._handlers[event_type] = []
            self._handlers[event_type].append(handler)
            return handler
        return decorator
    
    async def publish(self, event: DomainEvent) -> None:
        """Publish event to all handlers."""
        handlers = self._handlers.get(type(event), [])
        
        for handler in handlers:
            await handler(event)

# Usage
event_bus = EventBus()

@event_bus.subscribe(UserCreatedEvent)
async def send_welcome_email(event: UserCreatedEvent) -> None:
    """Send welcome email when user is created."""
    await email_service.send(
        to=event.email,
        template="welcome"
    )
```

### Dependency Injection

```python
from fastapi import Depends
from typing import Annotated

# Service definitions
class DatabaseService:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string

class CacheService:
    def __init__(self, redis_url: str):
        self.redis_url = redis_url

# Factory functions
def get_database_service() -> DatabaseService:
    return DatabaseService(settings.DATABASE_URL)

def get_cache_service() -> CacheService:
    return CacheService(settings.REDIS_URL)

# FastAPI dependency injection
async def get_user_service(
    db: DatabaseService = Depends(get_database_service),
    cache: CacheService = Depends(get_cache_service)
) -> UserService:
    return UserService(db, cache)

# Usage in endpoint
@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
) -> UserResponse:
    return await service.get_user(user_id)
```

---

## Performance Optimization

### Profiling

```python
import cProfile
import pstats
import io
from functools import wraps
import time

def profile(func):
    """Profile function execution."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        
        result = func(*args, **kwargs)
        
        profiler.disable()
        
        # Print stats
        s = io.StringIO()
        stats = pstats.Stats(profiler, stream=s)
        stats.sort_stats("cumulative")
        stats.print_stats(20)
        print(s.getvalue())
        
        return result
    return wrapper

class Profiler:
    """Context manager for profiling."""
    
    def __init__(self, enabled: bool = True):
        self._enabled = enabled
        self._profiler = None
    
    def __enter__(self):
        if self._enabled:
            self._profiler = cProfile.Profile()
            self._profiler.enable()
        return self
    
    def __exit__(self, *args):
        if self._profiler:
            self._profiler.disable()
            stats = pstats.Stats(self._profiler)
            stats.sort_stats("cumulative")
            stats.print_stats(10)

# Async profiling
import asyncio

async def profile_async(coro):
    """Profile async function."""
    async with asyncio.Profiler() as profiler:
        result = await coro
    
    stats = profiler.get_stats()
    print(f"Total time: {stats.total_time}")
    return result
```

### Caching

```python
from functools import lru_cache
from typing import Callable
import hashlib
import json
import time
import redis
from functools import wraps

def cached(
    ttl: int = 300,
    key_builder: Callable | None = None
):
    """Decorator for caching function results."""
    def decorator(func: Callable) -> Callable:
        cache_store = {}
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Build cache key
            if key_builder:
                cache_key = key_builder(*args, **kwargs)
            else:
                key_data = json.dumps((args, sorted(kwargs.items())), sort_keys=True)
                cache_key = hashlib.md5(key_data.encode()).hexdigest()
            
            # Check cache
            if cache_key in cache_store:
                cached_result, cached_time = cache_store[cache_key]
                if time.time() - cached_time < ttl:
                    return cached_result
            
            # Execute and cache
            result = func(*args, **kwargs)
            cache_store[cache_key] = (result, time.time())
            return result
        
        wrapper.cache_clear = lambda: cache_store.clear()
        return wrapper
    
    return decorator

# Redis caching
class RedisCache:
    """Redis-based caching."""
    
    def __init__(self, redis_client: redis.Redis, prefix: str = "cache"):
        self._redis = redis_client
        self._prefix = prefix
    
    def _key(self, key: str) -> str:
        return f"{self._prefix}:{key}"
    
    async def get(self, key: str) -> bytes | None:
        return await self._redis.get(self._key(key))
    
    async def set(self, key: str, value: bytes, ttl: int = 300) -> None:
        await self._redis.setex(self._key(key), ttl, value)
    
    async def delete(self, key: str) -> None:
        await self._redis.delete(self._key(key))

def cached_async(ttl: int = 300):
    """Async caching decorator with Redis."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            cache_key = f"{func.__name__}:{args}:{kwargs}"
            
            cached = await redis_cache.get(cache_key)
            if cached:
                return json.loads(cached)
            
            result = await func(*args, **kwargs)
            
            await redis_cache.set(
                cache_key,
                json.dumps(result, default=str).encode(),
                ttl
            )
            return result
        return wrapper
    return decorator
```

### Optimizing Collections

```python
from collections import defaultdict
from itertools import islice
import bisect

# Use set for membership testing
def find_duplicates_slow(items: list) -> list:
    """O(n²) - bad for large lists."""
    return [x for i, x in enumerate(items) if x in items[:i]]

def find_duplicates_fast(items: list) -> list:
    """O(n) - using set."""
    seen = set()
    duplicates = []
    for item in items:
        if item in seen:
            duplicates.append(item)
        seen.add(item)
    return duplicates

# Use defaultdict for counting
def count_items(items: list) -> dict:
    """Efficient counting with defaultdict."""
    counts = defaultdict(int)
    for item in items:
        counts[item] += 1
    return dict(counts)

# Use iterators for memory efficiency
def process_large_file(filepath: str):
    """Process large file line by line."""
    with open(filepath) as f:
        for line in f:
            yield line.strip()

# Batch processing
def batched(iterable, n):
    """Batch iterator into groups of n."""
    iterator = iter(iterable)
    while batch := list(islice(iterator, n)):
        yield batch

# Example usage
for batch in batched(process_large_file("huge_file.txt"), 1000):
    process_batch(batch)
```

---

## Logging & Observability

### Structured Logging

```python
import logging
import json
from datetime import datetime
from typing import Any

class StructuredLogger:
    """Structured JSON logger."""
    
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
    
    def _format_message(
        self,
        message: str,
        level: str,
        **kwargs
    ) -> str:
        """Format message as JSON."""
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": level,
            "message": message,
            **kwargs
        }
        return json.dumps(log_data)

    def debug(self, message: str, **kwargs: Any) -> None:
        self.logger.debug(self._format_message(message, "DEBUG", **kwargs))

    def info(self, message: str, **kwargs: Any) -> None:
        self.logger.info(self._format_message(message, "INFO", **kwargs))

    def warning(self, message: str, **kwargs: Any) -> None:
        self.logger.warning(self._format_message(message, "WARNING", **kwargs))

    def error(self, message: str, **kwargs: Any) -> None:
        self.logger.error(self._format_message(message, "ERROR", **kwargs))

    def exception(self, message: str, **kwargs: Any) -> None:
        self.logger.exception(self._format_message(message, "ERROR", **kwargs))

# Usage
logger = StructuredLogger(__name__)

logger.info(
    "user_created",
    user_id=123,
    username="alice",
    ip_address="192.168.1.1"
)
# Output: {"timestamp": "2024-01-01T00:00:00", "level": "INFO", "message": "user_created", "user_id": 123, "username": "alice", "ip_address": "192.168.1.1"}
```

### Logging Configuration

```python
# logging_config.py
import logging
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {
            "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        },
        "json": {
            "()": "app.logging.StructuredFormatter"
        }
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "default",
            "stream": "ext://sys.stdout"
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "formatter": "json",
            "filename": "logs/app.log",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5
        },
        "error_file": {
            "class": "logging.handlers.RotatingFileHandler",
            "formatter": "json",
            "filename": "logs/error.log",
            "level": "ERROR"
        }
    },
    "loggers": {
        "app": {
            "level": "DEBUG",
            "handlers": ["console", "file", "error_file"],
            "propagate": False
        },
        "uvicorn": {
            "level": "INFO",
            "handlers": ["console"]
        }
    },
    "root": {
        "level": "INFO",
        "handlers": ["console"]
    }
}

logging.config.dictConfig(LOGGING_CONFIG)
```

---

## Error Handling

### exception Hierarchies

```python
from fastapi import HTTPException, status
from enum import Enum

class ErrorCode(str, Enum):
    """Application error codes."""
    VALIDATION_ERROR = "VALIDATION_ERROR"
    NOT_FOUND = "NOT_FOUND"
    ALREADY_EXISTS = "ALREADY_EXISTS"
    UNAUTHORIZED = "UNAUTHORIZED"
    FORBIDDEN = "FORBIDDEN"
    INTERNAL_ERROR = "INTERNAL_ERROR"
    RATE_LIMITED = "RATE_LIMITED"

class AppException(Exception):
    """Base application exception."""
    
    def __init__(
        self,
        message: str,
        code: ErrorCode,
        status_code: int = status.HTTP_500_INTERNAL_SERVER_ERROR
    ):
        self.message = message
        self.code = code
        self.status_code = status_code
        super().__init__(self.message)

class NotFoundException(AppException):
    def __init__(self, resource: str, identifier: str | int):
        super().__init__(
            message=f"{resource} not found: {identifier}",
            code=ErrorCode.NOT_FOUND,
            status_code=status.HTTP_404_NOT_FOUND
        )

class ValidationException(AppException):
    def __init__(self, message: str, field: str | None = None):
        super().__init__(
            message=message,
            code=ErrorCode.VALIDATION_ERROR,
            status_code=status.HTTP_400_BAD_REQUEST
        )
        self.field = field

class UnauthorizedException(AppException):
    def __init__(self, message: str = "Authentication required"):
        super().__init__(
            message=message,
            code=ErrorCode.UNAUTHORIZED,
            status_code=status.HTTP_401_UNAUTHORIZED
        )

# Convert to HTTPException for FastAPI
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    """Handle application exceptions."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code.value,
                "message": exc.message
            }
        }
    )
```

### Retry Patterns

```python
import asyncio
from functools import wraps
from typing import Type, Callable

class RetryConfig:
    """Configuration for retry behavior."""
    
    def __init__(
        self,
        max_attempts: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 30.0,
        exponential_base: float = 2.0,
        exceptions: tuple = (Exception,)
    ):
        self.max_attempts = max_attempts
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.exponential_base = exponential_base
        self.exceptions = exceptions

def retry(config: RetryConfig | None = None):
    """Decorator for retrying failed operations."""
    if config is None:
        config = RetryConfig()
    
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def async_wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(config.max_attempts):
                try:
                    return await func(*args, **kwargs)
                except config.exceptions as e:
                    last_exception = e
                    
                    if attempt == config.max_attempts - 1:
                        raise
                    
                    delay = min(
                        config.base_delay * (config.exponential_base ** attempt),
                        config.max_delay
                    )
                    
                    await asyncio.sleep(delay)
            
            raise last_exception
        
        @wraps(func)
        def sync_wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(config.max_attempts):
                try:
                    return func(*args, **kwargs)
                except config.exceptions as e:
                    last_exception = e
                    
                    if attempt == config.max_attempts - 1:
                        raise
                    
                    delay = min(
                        config.base_delay * (config.exponential_base ** attempt),
                        config.max_delay
                    )
                    
                    import time
                    time.sleep(delay)
            
            raise last_exception
        
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper
    
    return decorator

# Usage
@retry(RetryConfig(max_attempts=3, base_delay=1.0))
async def fetch_external_data(url: str) -> dict:
    """Fetch data with retry logic."""
    response = await httpx.get(url)
    response.raise_for_status()
    return response.json()
```

---

## Documentation

### NumPy-Style Docstrings

```python
def calculate_statistics(data, axis=0, ddof=0):
    """Calculate the mean and standard deviation along the specified axis.
    
    Parameters
    ----------
    data : array_like
        Input data. If `data` is not an array, a conversion to a 1-d
        array is attempted.
    axis : int, optional
        Axis along which the variance is calculated. Default is 0.
    ddof : int, optional
        "Delta degrees of freedom": the divisor used in the
        calculation is N - ddof, where N represents the number of
        elements. Default is 0.
    
    Returns
    -------
    mean, std : tuple of arrays
        Mean and standard deviation along the specified axis.
    
    Raises
    ------
    ValueError
        If the input array has an invalid shape or type.
    
    Examples
    --------
    >>> import numpy as np
    >>> data = np.arange(10).reshape(2, 5)
    >>> calculate_statistics(data, axis=1)
    (array([2., 7.]), array([1.41421356, 1.41421356]))
    
    Notes
    -----
    The default `ddof=0` provides a population standard deviation.
    Set `ddof=1` for sample standard deviation.
    """
```

### Class Documentation

```python
class UserRepository:
    """Repository for managing User entities in the database.
    
    This class provides an interface for database operations on User
    entities, abstracting the underlying database implementation.
    
    Parameters
    ----------
    session : AsyncSession
        Database session for executing queries.
    
    Attributes
    ----------
    TABLE_NAME : str
        Name of the database table.
    
    Examples
    --------
    >>> async with async_session_maker() as session:
    ...     repo = UserRepository(session)
    ...     user = await repo.find_one(1)
    ...     if user:
    ...         print(user.username)
    """
    
    TABLE_NAME = "users"
    
    def __init__(self, session: AsyncSession):
        """Initialize repository with database session.
        
        Parameters
        ----------
        session : AsyncSession
            SQLAlchemy async session.
        """
        self._session = session
    
    async def find_one(self, user_id: int) -> User | None:
        """Find a user by ID.
        
        Parameters
        ----------
        user_id : int
            The unique identifier of the user.
        
        Returns
        -------
        User | None
            The User entity if found, None otherwise.
        """
        ...
```

### README Documentation

```markdown
# Project Name

Brief description of the project.

## Features

- Feature 1
- Feature 2  
- Feature 3

## Requirements

- Python 3.11+
- PostgreSQL 14+

## Installation

```bash
pip install -r requirements.txt
```

## Configuration

Copy `.env.example` to `.env` and configure:

```bash
cp .env.example .env
```

## Usage

```python
from app import create_app

app = create_app()
```

## Development

```bash
# Install dev dependencies
pip install -r requirements-dev.txt

# Run tests
pytest

# Run linter
ruff check src/

# Format code
black src/
```

## License

MIT
```

---

## Dependency Management

### pyproject.toml

```toml
[project]
name = "my-app"
version = "1.0.0"
description = "Application description"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [
    {name = "Author Name", email = "author@example.com"}
]
keywords = ["python", "fastapi", "web"]
classifiers = [
    "Programming Language :: Python :: 3.11",
    "Framework :: FastAPI",
]

dependencies = [
    "fastapi>=0.100.0",
    "pydantic>=2.0.0",
    "sqlalchemy[asyncio]>=2.0.0",
    "uvicorn[standard]>=0.23.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.1.0",
    "black>=23.0.0",
    "mypy>=1.5.0",
    "pre-commit>=3.3.0",
]
server = [
    "gunicorn>=21.0.0",
]

[project.scripts]
my-app = "app.main:main"

[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.build_meta"

[tool.setuptools]
packages = ["app"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

### requirements.txt (Generated)

```text
# Generated by pip-compile
fastapi==0.104.1
    --hash=sha256:...
pydantic==2.5.0
    --hash=sha256:...
# ... pinned versions with hashes
```

### Virtual Environments

```bash
# Create venv
python -m venv .venv

# Activate
source .venv/bin/activate

# Install dependencies
pip install -e ".[dev]"

# Generate requirements
pip-compile --output-file=requirements.txt pyproject.toml

# Verify installed packages
pip check

# Export for Docker
pip freeze > requirements.txt
```

---

## API Design

### RESTful Endpoints

```python
from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel, Field
from typing import Annotated

router = APIRouter(prefix="/api/users", tags=["users"])

class UserCreate(BaseModel):
    """Schema for creating a user."""
    username: str = Field(..., min_length=3, max_length=50)
    email: str = Field(..., pattern=r"^[\w\.-]+@[\w\.-]+\.\w+$")
    password: str = Field(..., min_length=8)

class UserResponse(BaseModel):
    """Schema for user response."""
    id: int
    username: str
    email: str
    is_active: bool
    
    class Config:
        from_attributes = True

class UserListResponse(BaseModel):
    """Schema for paginated user list."""
    items: list[UserResponse]
    total: int
    page: int
    page_size: int
    has_next: bool
    has_previous: bool

@router.post("", status_code=status.HTTP_201_CREATED, response_model=UserResponse)
async def create_user(
    user_data: UserCreate,
    service: UserService = Depends(get_user_service)
):
    """Create a new user."""
    try:
        return await service.create_user(user_data)
    except DuplicateEmailError:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Email already registered"
        )

@router.get("", response_model=UserListResponse)
async def list_users(
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=100),
    service: UserService = Depends(get_user_service)
):
    """List all users with pagination."""
    return await service.list_users(page=page, page_size=page_size)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    """Get a user by ID."""
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return user

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    user_data: UserUpdate,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_user)
):
    """Update a user."""
    if current_user.id != user_id and current_user.role != Role.ADMIN:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not authorized to update this user"
        )
    
    try:
        return await service.update_user(user_id, user_data)
    except NotFoundError:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
    current_user: User = Depends(get_current_user)
):
    """Delete a user."""
    if current_user.role != Role.ADMIN:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Only admins can delete users"
        )
    
    if not await service.delete_user(user_id):
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
```

---

## Database Practices

### Async SQLAlchemy Setup

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy import Column, Integer, String, Boolean
import urlparse

Base = declarative_base()

class User(Base):
    """User model."""
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    is_verified = Column(Boolean, default=False)

async def create_engine_and_session(database_url: str):
    """Create async engine and session factory."""
    # Convert postgresql:// to postgresql+asyncpg://
    if database_url.startswith("postgresql://"):
        database_url = database_url.replace(
            "postgresql://", "postgresql+asyncpg://"
        )
    
    engine = create_async_engine(
        database_url,
        echo=settings.DEBUG,
        pool_size=20,
        max_overflow=10,
        pool_pre_ping=True,
        pool_recycle=3600,
    )
    
    async_session_maker = sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False
    )
    
    return engine, async_session_maker

async def get_db_session(async_session_maker):
    """Get database session."""
    async with async_session_maker() as session:
        yield session
```

### Migrations

```python
# alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = postgresql://user:pass@localhost/db

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic
```

### Query Best Practices

```python
from sqlalchemy import select, update, delete
from sqlalchemy.orm import selectinload, joinedload

# Use selectinload for collections (avoids N+1)
async def get_user_with_posts(db: AsyncSession, user_id: int):
    """Get user with posts eagerly loaded."""
    result = await db.execute(
        select(User)
        .options(selectinload(User.posts))
        .where(User.id == user_id)
    )
    return result.scalar_one()

# Use joinedload for single relationships
async def get_post_with_author(db: AsyncSession, post_id: int):
    """Get post with author eagerly loaded."""
    result = await db.execute(
        select(Post)
        .options(joinedload(Post.author))
        .where(Post.id == post_id)
    )
    return result.scalar_one()

# Use update for bulk operations
async def deactivate_inactive_users(db: AsyncSession, days: int):
    """Deactivate users inactive for specified days."""
    await db.execute(
        update(User)
        .where(
            User.last_login < datetime.utcnow() - timedelta(days=days)
        )
        .values(is_active=False)
    )
    await db.commit()

# Batch inserts
async def bulk_create_users(db: AsyncSession, users: list[User]):
    """Bulk insert users."""
    db.add_all(users)
    await db.commit()
```

---

## References

- [PEP 8 Style Guide](https://peps.python.org/pep-0008/)
- [Black Code Formatter](https://black.readthedocs.io/)
- [Ruff Linter](https://docs.astral.sh/ruff/)
- [Pytest Documentation](https://docs.pytest.org/)
- [FastAPI Best Practices](https://fastapi.tiangolo.com/best-practices/)
- [NumPy Docstring Guide](https://numpydoc.readthedocs.io/)