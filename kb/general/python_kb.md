# Python Knowledge Base

A comprehensive reference for Python 3.11+ development, with emphasis on modern patterns and FastAPI applications.

---

## Table of Contents

1. [Data Types & Literals](#data-types--literals)
2. [Control Flow](#control-flow)
3. [Functions](#functions)
4. [Classes & OOP](#classes--oop)
5. [Modules & Imports](#modules--imports)
6. [Error Handling](#error-handling)
7. [Type Hints](#type-hints)
8. [Async Programming](#async-programming)
9. [Collections & Iterables](#collections--iterables)
10. [Context Managers](#context-managers)
11. [Decorators](#decorators)
12. [Working with FastAPI](#working-with-fastapi)

---

## Data Types & Literals

### Primitive Types

```python
# Integer - arbitrary precision
age: int = 25
large_num: int = 1_000_000  # Underscore separator (Python 3.6+)

# Float - IEEE 754 double precision
price: float = 19.99
scientific: float = 1.5e-10

# Boolean
is_active: bool = True
is_deleted: bool = False

# None - singleton
result: str | None = None
```

### String Formatting (Python 3.11+)

```python
name = "Alice"
age = 30

# f-strings (preferred)
message = f"Hello, {name}! You are {age} years old."
formatted = f"Price: ${price:.2f}"  # Float formatting

# Debug formatting (Python 3.8+)
user_info = f"{name=}, {age=}"  # user_info = "name='Alice', age=30"

# Structural pattern matching (Python 3.10+)
status = "active"
match status:
    case "active":
        print("User is active")
    case "pending" | "review":  # Multiple patterns
        print("Awaiting approval")
    case _:
        print("Unknown status")
```

### Collection Literals

```python
# List - mutable, ordered
numbers: list[int] = [1, 2, 3, 4, 5]
matrix: list[list[int]] = [[1, 2], [3, 4]]

# Dict - insertion ordered (Python 3.7+)
config: dict[str, Any] = {"host": "localhost", "port": 8000}

# Set - unordered, unique elements
unique_ids: set[int] = {1, 2, 3, 3}  # {1, 2, 3}

# Tuple - immutable, can be named
point: tuple[int, int] = (10, 20)
named_point: tuple[str, int, int] = ("origin", 0, 0)

# Dictionary unpacking
defaults = {"timeout": 30, "retries": 3}
options = {"timeout": 60, **defaults}  # timeout=60, retries=3
```

---

## Control Flow

### Match Statement (Python 3.10+)

```python
from typing import Any

def process_response(data: Any) -> str:
    match data:
        case {"status": "ok", "data": d}:
            return f"Success: {d}"
        case {"status": "error", "message": msg}:
            return f"Error: {msg}"
        case {"status": status}:
            return f"Unknown status: {status}"
        case _:
            return "Invalid response format"

# Structural pattern matching with classes
class Point:
    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y

def classify_point(p: Point) -> str:
    match p:
        case Point(x=0, y=0):
            return "Origin"
        case Point(x=0, y=y):
            return f"On Y-axis at y={y}"
        case Point(x=x, y=0):
            return f"On X-axis at x={x}"
        case Point():
            return "Elsewhere"
```

### Conditional Expressions

```python
# Ternary operator
status = "active" if user.is_verified else "pending"

# Walrus operator (Python 3.8+) - assignment expressions
if (matched := pattern.search(text)):
    log.info(f"Found: {matched.group()}")

# List comprehension with condition
active_users = [u for u in users if u.is_active]
```

---

## Functions

### Function Definitions

```python
from typing import Callable, Any

def greet(name: str, greeting: str = "Hello") -> str:
    """Greet a user with a custom greeting.
    
    Args:
        name: The name of the user to greet.
        greeting: The greeting word. Defaults to "Hello".
    
    Returns:
        The formatted greeting message.
    """
    return f"{greeting}, {name}!"

# Multiple return values
def divide(a: float, b: float) -> tuple[float, bool]:
    """Returns quotient and success flag."""
    if b == 0:
        return 0.0, False
    return a / b, True

# Callable type hint
def apply_operation(
    func: Callable[[int, int], int], 
    a: int, 
    b: int
) -> int:
    return func(a, b)
```

### *args and **kwargs

```python
def flexible_sum(*args: int, **kwargs: int) -> int:
    """Sum all positional and keyword arguments."""
    total = sum(args)
    for key, value in kwargs.items():
        total += value
    return total

# Usage
result = flexible_sum(1, 2, 3, a=4, b=5)  # 15

# Type hints for unpacked args
def process_items(*items: str) -> list[str]:
    return list(items)

# Combining with positional-only and keyword-only
def configure(
    name: str,           # Positional only
    /,                   # Positional-only delimiter
    age: int,            # Positional or keyword
    *,                   # Keyword-only delimiter
    debug: bool = False, # Keyword only
    verbose: bool = False
) -> dict:
    return {"name": name, "age": age, "debug": debug, "verbose": verbose}
```

### Lambda Functions

```python
# Simple lambda
square = lambda x: x ** 2

# With multiple arguments
full_name = lambda first, last: f"{first} {last}"

# In sorted() with key
users.sort(key=lambda u: u.last_name)

# Higher-order functions
from functools import reduce

numbers = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, numbers))  # [2, 4, 6, 8, 10]
evens = list(filter(lambda x: x % 2 == 0, numbers))  # [2, 4]
total = reduce(lambda acc, x: acc + x, numbers, 0)  # 15
```

---

## Classes & OOP

### Class Definition

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional

@dataclass
class User:
    """Represents a user in the system."""
    id: int
    username: str
    email: str
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.now)
    
    def __post_init__(self) -> None:
        """Validate after initialization."""
        if not self.email or "@" not in self.email:
            raise ValueError("Invalid email address")
    
    def activate(self) -> None:
        """Activate the user account."""
        self.is_active = True
    
    def to_dict(self) -> dict:
        """Convert user to dictionary."""
        return {
            "id": self.id,
            "username": self.username,
            "email": self.email,
            "is_active": self.is_active,
            "created_at": self.created_at.isoformat(),
        }
```

### Inheritance & Method Resolution

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    """Abstract base class for animals."""
    
    def __init__(self, name: str):
        self.name = name
    
    @abstractmethod
    def speak(self) -> str:
        """Return the sound this animal makes."""
        pass
    
    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(name={self.name!r})"


class Dog(Animal):
    def speak(self) -> str:
        return f"{self.name} says woof!"


class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name} says meow!"
```

### Dunder Methods

```python
class Vector:
    """2D vector with rich comparison and arithmetic."""
    
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
    
    def __repr__(self) -> str:
        return f"Vector(x={self.x}, y={self.y})"
    
    def __str__(self) -> str:
        return f"({self.x}, {self.y})"
    
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented
        return self.x == other.x and self.y == other.y
    
    def __add__(self, other: "Vector") -> "Vector":
        return Vector(self.x + other.x, self.y + other.y)
    
    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar)
    
    def __rmul__(self, scalar: float) -> "Vector":
        return self.__mul__(scalar)
    
    def __len__(self) -> int:
        """Return magnitude (rounded)."""
        return int((self.x**2 + self.y**2) ** 0.5)
    
    def __contains__(self, value: float) -> bool:
        """Check if value is in vector components."""
        return value == self.x or value == self.y
```

### Dataclasses (Python 3.7+)

```python
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum

class Priority(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

@dataclass
class Task:
    title: str
    description: str = ""
    priority: Priority = Priority.MEDIUM
    tags: List[str] = field(default_factory=list)
    completed: bool = False
    
    def __post_init__(self) -> None:
        self.tags = list(self.tags)  # Ensure mutable copy
    
    def to_dict(self) -> dict:
        return {
            "title": self.title,
            "description": self.description,
            "priority": self.priority.name,
            "tags": self.tags,
            "completed": self.completed,
        }

# Frozen dataclass (immutable)
@dataclass(frozen=True)
class Config:
    host: str
    port: int
    debug: bool = False
```

---

## Modules & Imports

### Import Patterns

```python
# Standard imports
import json
import os
from pathlib import Path

# From imports (preferred for specific names)
from typing import Optional, Union
from dataclasses import dataclass

# Aliased imports
import numpy as np
from fastapi import FastAPI as FastAPIApp

# Relative imports (within packages)
from . import module  # Same package
from ..package import module  # Parent package
from .module import ClassName  # Specific import

# Lazy imports (defer expensive imports)
def heavy_computation():
    import numpy as np  # Only imported when called
    return np.array([1, 2, 3])
```

### Module Structure

```python
# mypackage/__init__.py
"""Package initialization."""
from .module import MyClass
from .constants import API_VERSION

__all__ = ["MyClass", "API_VERSION"]

# mypackage/module.py
"""Module documentation."""

class MyClass:
    pass
```

### Dynamic Imports

```python
import importlib

def load_handler(handler_name: str):
    """Dynamically load a handler class."""
    module = importlib.import_module(f"handlers.{handler_name}")
    return getattr(module, "Handler")

# Plugin discovery
import pkgutil
def discover_plugins(package_name: str):
    """Find all modules in a package."""
    package = importlib.import_module(package_name)
    plugins = {}
    for _, name, is_pkg in pkgutil.iter_modules(package.__path__):
        if not is_pkg:
            plugins[name] = importlib.import_module(f"{package_name}.{name}")
    return plugins
```

---

## Error Handling

### Exception Hierarchy

```python
# Custom exceptions
class AppError(Exception):
    """Base exception for application errors."""
    
    def __init__(self, message: str, code: str = "UNKNOWN"):
        self.message = message
        self.code = code
        super().__init__(self.message)


class ValidationError(AppError):
    """Raised when input validation fails."""
    
    def __init__(self, message: str, field: str | None = None):
        super().__init__(message, code="VALIDATION_ERROR")
        self.field = field


class NotFoundError(AppError):
    """Raised when a resource is not found."""
    
    def __init__(self, resource: str, identifier: str | int):
        super().__init__(
            f"{resource} not found: {identifier}",
            code="NOT_FOUND"
        )
        self.resource = resource
        self.identifier = identifier
```

### Try/Except Patterns

```python
from typing import Optional

def get_user(user_id: int) -> Optional[User]:
    """Fetch user with proper error handling."""
    try:
        return db.users.find_one({"_id": user_id})
    except ConnectionError as e:
        logger.error(f"Database connection failed: {e}")
        raise
    except Exception as e:
        logger.exception("Unexpected error fetching user")
        raise DatabaseError(str(e)) from e

# Pattern matching in exceptions (Python 3.11+)
try:
    risky_operation()
except Exception as e:
    match e:
        case ValueError(message=msg) if "timeout" in msg:
            logger.warning("Timeout occurred, retrying...")
            retry_operation()
        case TimeoutError():
            logger.error("Operation timed out")
            raise
        case _:
            logger.exception("Unexpected error")
            raise
```

### Context Manager for Errors

```python
from contextlib import contextmanager
import logging

logger = logging.getLogger(__name__)

@contextmanager
def handle_errors(operation_name: str, reraise: bool = True):
    """Context manager for consistent error handling."""
    try:
        yield
    except ValidationError:
        logger.warning(f"Validation failed in {operation_name}")
        if reraise:
            raise
    except PermissionError:
        logger.error(f"Permission denied in {operation_name}")
        if reraise:
            raise
    except Exception:
        logger.exception(f"Unexpected error in {operation_name}")
        if reraise:
            raise

# Usage
with handle_errors("user_creation"):
    create_user(data)
```

---

## Type Hints

### Basic Type Hints

```python
from typing import TypeVar, Generic, Protocol, TypedDict, NotRequired

# Primitive types
name: str = "Alice"
age: int = 30
active: bool = True
price: float = 19.99

# Collections
ids: list[int] = [1, 2, 3]
lookup: dict[str, User] = {}
unique: set[str] = set()
coords: tuple[int, int] = (10, 20)

# Optional and Union
email: str | None = None  # Python 3.10+
optional_name: Optional[str] = None  # Legacy style

# Type alias
UserId = int
UserDict = dict[str, UserId]

# Literal types
Status = Literal["pending", "active", "disabled"]
def set_status(status: Status) -> None: ...

# TypeVar
T = TypeVar("T")
def first(items: list[T]) -> T | None:
    return items[0] if items else None
```

### Generics

```python
from typing import Generic, TypeVar, Iterator

T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

class Repository(Generic[T]):
    """Generic repository for any entity type."""
    
    def __init__(self, collection_name: str):
        self.collection_name = collection_name
    
    def find(self, query: dict) -> list[T]:
        ...
    
    def find_one(self, query: dict) -> T | None:
        ...

class Cache(Generic[K, V]):
    """Generic cache with key-value pairs."""
    
    def __init__(self):
        self._store: dict[K, V] = {}
    
    def get(self, key: K) -> V | None:
        return self._store.get(key)
    
    def set(self, key: K, value: V) -> None:
        self._store[key] = value
```

### Protocol (Structural Subtyping)

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Serializable(Protocol):
    """Protocol for objects that can be serialized."""
    
    def to_dict(self) -> dict:
        ...

@runtime_checkable
class Loggable(Protocol):
    """Protocol for objects that can be logged."""
    
    def __str__(self) -> str:
        ...

def serialize(obj: Serializable) -> dict:
    """Accept any object with to_dict method."""
    return obj.to_dict()

# Usage with structural typing
class User:
    def to_dict(self) -> dict:
        return {"name": "Alice"}

# Works without explicit inheritance!
serialize(User())  # OK - has to_dict method
```

### TypedDict

```python
from typing import TypedDict, NotRequired, Required

class UserDict(TypedDict):
    """Typed dictionary for user data."""
    id: int
    username: str
    email: str
    # Optional fields
    bio: NotRequired[str]
    avatar_url: NotRequired[str]
    # Required field with default
    role: Required[str]  # Must always provide

# Usage
user: UserDict = {
    "id": 1,
    "username": "alice",
    "email": "alice@example.com",
    "role": "admin",
}
```

### Pydantic Integration (FastAPI)

```python
from pydantic import BaseModel, Field, EmailStr, validator
from datetime import datetime

class UserCreate(BaseModel):
    """Schema for creating a new user."""
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(..., min_length=8)
    age: int | None = Field(None, ge=0, le=150)
    
    @validator("username")
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("Username must be alphanumeric")
        return v.lower()
    
    @validator("password")
    def password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain uppercase")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain digit")
        return v
    
    class Config:
        json_schema_extra = {
            "example": {
                "username": "alice",
                "email": "alice@example.com",
                "password": "SecurePass123",
                "age": 30
            }
        }


class UserResponse(BaseModel):
    """Schema for user response (excludes sensitive data)."""
    id: int
    username: str
    email: str
    created_at: datetime
    
    class Config:
        from_attributes = True
```

---

## Async Programming

### Async/Await Basics

```python
import asyncio

async def fetch_data(url: str) -> dict:
    """Fetch data from URL asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def main():
    # Sequential execution
    result1 = await fetch_data("https://api.example.com/1")
    result2 = await fetch_data("https://api.example.com/2")
    
    # Concurrent execution
    results = await asyncio.gather(
        fetch_data("https://api.example.com/1"),
        fetch_data("https://api.example.com/2"),
        fetch_data("https://api.example.com/3"),
    )

# Run the async function
asyncio.run(main())
```

### Asynchronous Generators

```python
async def fetch_all_pages(url: str) -> AsyncIterator[dict]:
    """Fetch all pages asynchronously."""
    page = 1
    while True:
        async with aiohttp.ClientSession() as session:
            async with session.get(f"{url}?page={page}") as response:
                data = await response.json()
                if not data.get("items"):
                    break
                for item in data["items"]:
                    yield item
                page += 1

# Usage
async def process_items():
    async for item in fetch_all_pages("https://api.example.com/items"):
        await process_item(item)
```

### Task Management

```python
import asyncio
from asyncio import Task, TaskGroup

async def worker(id: int, duration: float) -> int:
    """Simulate work with async sleep."""
    await asyncio.sleep(duration)
    return id * 2

async def main():
    # Create tasks
    task1 = asyncio.create_task(worker(1, 1.0))
    task2 = asyncio.create_task(worker(2, 0.5))
    
    # Wait for specific task
    done, pending = await asyncio.wait(
        [task1, task2],
        timeout=2.0,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    # TaskGroup (Python 3.11+) - automatic cleanup
    async with TaskGroup() as tg:
        tg.create_task(worker(1, 1.0))
        tg.create_task(worker(2, 0.5))
    # All tasks complete or exception raised
```

### Async Context Managers

```python
class AsyncDatabaseConnection:
    """Async context manager for database connections."""
    
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self.connection = None
    
    async def __aenter__(self) -> "AsyncDatabaseConnection":
        self.connection = await connect_async(self.connection_string)
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> bool:
        if self.connection:
            await self.connection.close()
        return False  # Don't suppress exceptions

# Usage
async def query_data():
    async with AsyncDatabaseConnection("postgresql://...") as conn:
        result = await conn.execute("SELECT * FROM users")
```

---

## Collections & Iterables

### List Comprehensions

```python
# Basic
squares = [x**2 for x in range(10)]

# With condition
evens = [x for x in range(20) if x % 2 == 0]

# Nested comprehension
matrix = [[i*j for j in range(5)] for i in range(5)]

# Dictionary comprehension
word_lengths = {word: len(word) for word in ["hello", "world"]}

# Set comprehension
unique_chars = {c.lower() for c in "Hello World"}
```

### Itertools

```python
from itertools import (
    chain, cycle, islice, 
    groupby, tee, combinations, permutations
)

# Infinite iterator
counter = count(start=1)
ones = repeat(1)

# Slicing iterators
limited = list(islice(counter, 5))  # [1, 2, 3, 4, 5]

# Chaining
combined = chain([1, 2], [3, 4], [5])  # [1, 2, 3, 4, 5]

# Grouping
data = sorted([("a", 1), ("a", 2), ("b", 3)], key=lambda x: x[0])
for key, group in groupby(data, key=lambda x: x[0]):
    items = list(group)
    # Process items with same key

# Combinations and permutations
list(combinations([1, 2, 3], 2))  # [(1,2), (1,3), (2,3)]
list(permutations([1, 2, 3], 2))  # [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]
```

### Generators

```python
def fibonacci(n: int) -> Generator[int, None, None]:
    """Generate Fibonacci numbers."""
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

def lazy_range(start: int, stop: int) -> Generator[int, None, None]:
    """Lazy range implementation."""
    current = start
    while current < stop:
        yield current
        current += 1

# Generator expression (memory efficient)
sum_of_squares = sum(x**2 for x in range(1_000_000))
```

---

## Context Managers

### Synchronous Context Managers

```python
from contextlib import contextmanager

class DatabaseConnection:
    """Context manager for database connections."""
    
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self.connection = None
    
    def __enter__(self) -> "DatabaseConnection":
        self.connection = create_connection(self.connection_string)
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        if self.connection:
            self.connection.close()
        return False  # Don't suppress exceptions

# Usage
with DatabaseConnection("postgresql://...") as conn:
    result = conn.execute("SELECT * FROM users")

# Generator-based context manager
@contextmanager
def managed_resource(name: str):
    """Context manager using generator."""
    resource = acquire(name)
    try:
        yield resource
    finally:
        release(name)

# suppress specific exceptions
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("nonexistent.txt")
```

---

## Decorators

### Basic Decorators

```python
import functools
import time
from typing import Callable

def timing_decorator(func: Callable) -> Callable:
    """Measure execution time of a function."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f} seconds")
        return result
    return wrapper

def retry(max_attempts: int = 3, delay: float = 1.0) -> Callable:
    """Retry a function on failure."""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay)
            raise RuntimeError("Unreachable")
        return wrapper
    return decorator

# Usage
@timing_decorator
@retry(max_attempts=3, delay=2.0)
def fetch_url(url: str) -> dict:
    ...
```

### Decorators with Arguments

```python
from typing import TypeVar, ParamSpec
import logging

P = ParamSpec("P")
R = TypeVar("R")

def log_calls(logger: logging.Logger) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """Decorator factory that takes a logger argument."""
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            logger.debug(f"Calling {func.__name__} with {args}, {kwargs}")
            try:
                result = func(*args, **kwargs)
                logger.debug(f"{func.__name__} returned {result}")
                return result
            except Exception as e:
                logger.exception(f"{func.__name__} raised {e}")
                raise
        return wrapper
    return decorator

# Usage
logger = logging.getLogger(__name__)

@log_calls(logger)
def process_data(data: dict) -> list:
    ...
```

### Class Decorators

```python
def singleton(cls: type[T]) -> type[T]:
    """Ensure a class has only one instance."""
    instances: dict[type, object] = {}
    
    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance

@singleton
class Config:
    def __init__(self):
        self.settings = {}
```

---

## Working with FastAPI

### FastAPI Application Structure

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel, Field
from typing import Annotated
import uvicorn

app = FastAPI(
    title="My API",
    description="API description",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
)

# Security
security = HTTPBearer()

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Dependencies
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """Validate JWT and return current user."""
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        user = await get_user(user_id)
        if user is None:
            raise HTTPException(status_code=404, detail="User not found")
        return user
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Routes
@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return {"status": "healthy"}

@app.get("/api/users/me", response_model=UserResponse)
async def get_current_user_info(
    current_user: User = Depends(get_current_user)
):
    """Get current authenticated user."""
    return current_user
```

### FastAPI Dependency Injection

```python
from fastapi import Depends
from typing import Annotated

class PaginationParams:
    """Common pagination parameters."""
    
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(50, ge=1, le=100)
    ):
        self.page = page
        self.page_size = page_size
        self.offset = (page - 1) * page_size

# As dependency
@app.get("/items")
async def list_items(
    pagination: PaginationParams = Depends(),
    category: str | None = None
):
    items = await fetch_items(
        offset=pagination.offset,
        limit=pagination.page_size,
        category=category
    )
    return {
        "items": items,
        "page": pagination.page,
        "page_size": pagination.page_size
    }

# With database
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session

@app.get("/users")
async def list_users(
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    result = await db.execute(select(User))
    return result.scalars().all()
```

### Background Tasks

```python
from fastapi import BackgroundTasks

def send_email_notification(email: str, message: str):
    """Background task to send email."""
    # This runs after response is sent
    send_email(email, message)

@app.post("/submit")
async def submit_form(
    form: SubmissionForm,
    background_tasks: BackgroundTasks
):
    # Process immediately
    save_submission(form)
    
    # Schedule background task
    background_tasks.add_task(
        send_email_notification,
        form.email,
        "Thank you for submitting!"
    )
    
    return {"status": "success"}
```

### WebSocket Support

```python
from fastapi import WebSocket, WebSocketDisconnect

class ConnectionManager:
    """Manage WebSocket connections."""
    
    def __init__(self):
        self.active_connections: list[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def broadcast(self, message: dict):
        for connection in self.active_connections:
            await connection.send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            message = {"client_id": client_id, "message": data}
            await manager.broadcast(message)
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

---

## References

- [Python Documentation](https://docs.python.org/3/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [typing — Support for type hints](https://docs.python.org/3/library/typing.html)
- [PEP 484 — Type Hints](https://peps.python.org/pep-0484/)
- [PEP 622 — Structural Pattern Matching](https://peps.python.org/pep-0622/)
