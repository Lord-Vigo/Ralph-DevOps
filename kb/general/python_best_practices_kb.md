---
name: Python Best Practices KB
kb_type: general
maintained_by: Carl
---

# Python Best Practices KB

Standards and conventions for JamRoom development. Follow these for every IOC — consistency across the codebase reduces review friction and prevents avoidable bugs.

---

## Project Structure

Use `src/` layout. It prevents the common mistake of accidentally importing the local package instead of the installed one, and it is required for reliable PyInstaller bundling.

```
JamRoom/
├── src/
│   └── jamroom/
│       ├── __init__.py
│       ├── main.py           # entry point — calls asyncio.run(app.run())
│       ├── app.py            # Textual App subclass
│       ├── api/
│       │   ├── __init__.py
│       │   ├── server.py     # FastAPI app instance
│       │   └── routes/
│       │       ├── __init__.py
│       │       ├── models.py
│       │       └── generate.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── manager.py    # model lifecycle
│       │   └── runner.py     # subprocess wrapper
│       ├── config/
│       │   ├── __init__.py
│       │   └── settings.py   # AppConfig dataclass
│       └── ui/
│           ├── __init__.py
│           └── screens/
├── tests/
│   ├── conftest.py
│   ├── test_config.py
│   └── test_model_runner.py
├── pyproject.toml
├── requirements.in           # human-maintained, unpinned
├── requirements.txt          # pip-tools generated, pinned
└── jamroom.spec              # PyInstaller spec
```

**Rule:** one module per logical concern. If a module exceeds ~300 lines, consider splitting it.

---

## pyproject.toml (Modern Packaging)

Use `pyproject.toml` exclusively. Do not create or maintain `setup.py` or `setup.cfg`.

```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "jamroom"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "textual>=0.52.0",
    "fastapi>=0.111.0",
    "uvicorn[standard]>=0.29.0",
    "aiofiles>=23.2.1",
    "httpx>=0.27.0",
]

[project.scripts]
jamroom = "jamroom.main:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "TCH"]
ignore = ["E501"]  # line length handled by formatter

[tool.ruff.format]
quote-style = "double"

[tool.mypy]
python_version = "3.11"
strict = true
mypy_path = "src"

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

---

## Dependency Management: pip-tools

Use pip-tools. This is the project standard.

| Tool | File | Purpose |
|---|---|---|
| pip-tools | `requirements.in` | Human-maintained, unpinned, logical dependencies |
| pip-tools | `requirements.txt` | Generated, fully pinned, what gets installed |
| pip-tools | `requirements-dev.in` | Dev-only deps (pytest, ruff, mypy, pip-tools itself) |
| pip-tools | `requirements-dev.txt` | Generated pinned dev deps |

### Workflow

```bash
# Install pip-tools
pip install pip-tools

# After editing requirements.in
pip-compile requirements.in --output-file requirements.txt --strip-extras

# After editing requirements-dev.in
pip-compile requirements-dev.in --output-file requirements-dev.txt --strip-extras

# Sync your venv to the pinned requirements
pip-sync requirements.txt requirements-dev.txt

# Upgrade all deps to latest compatible versions
pip-compile --upgrade requirements.in
```

**Commit both `.in` and `.txt` files.** Never hand-edit `requirements.txt` — regenerate it.

---

## Virtual Environment Setup

```bash
# Create (do this once per machine, never commit the venv directory)
python3.11 -m venv .venv

# Activate
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\activate.bat       # Windows cmd
.venv\Scripts\Activate.ps1       # Windows PowerShell

# Install pinned deps
pip install pip-tools
pip-sync requirements.txt requirements-dev.txt

# Verify
python -m jamroom --version
```

Add `.venv/` to `.gitignore`. Never commit the venv.

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Functions | `snake_case` | `load_model_config()` |
| Variables | `snake_case` | `model_path`, `audio_buffer` |
| Classes | `PascalCase` | `ModelManager`, `AudioCapture` |
| Constants | `UPPER_SNAKE_CASE` | `DEFAULT_CONTEXT_SIZE = 2048` |
| Modules / files | `snake_case` | `model_runner.py`, `audio_capture.py` |
| Packages | `snake_case`, short | `jamroom`, `api`, `ui` |
| Type aliases | `PascalCase` | `ModelId = str` |
| Private members | leading underscore | `_internal_state`, `_cleanup()` |
| Async functions | same as sync | `async def load_model()` — no `async_` prefix |

---

## Module Organization

- One responsibility per module. `model_runner.py` runs models — it does not parse configs.
- Avoid circular imports. If module A imports B and B imports A, extract the shared type into a third module (e.g., `types.py` or `models.py`).
- Keep `__init__.py` files minimal — import only the public API, not implementation details.
- Use `TYPE_CHECKING` guard for imports that are only needed for type hints:

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from jamroom.models.manager import ModelManager  # avoids circular import at runtime
```

---

## Error Handling

### Use specific exceptions — never bare `except`

```python
# Bad
try:
    cfg = load_config()
except:
    pass

# Bad
try:
    cfg = load_config()
except Exception:
    pass  # swallows everything including KeyboardInterrupt

# Good
try:
    cfg = load_config()
except FileNotFoundError:
    cfg = AppConfig()  # first run — use defaults
except json.JSONDecodeError as e:
    raise ConfigParseError(f"Config file is corrupted: {e}") from e
```

### Custom exception hierarchy

```python
# src/jamroom/exceptions.py
class JamRoomError(Exception):
    """Base exception for all JamRoom errors."""

class ConfigError(JamRoomError):
    """Configuration is invalid or unreadable."""

class ModelError(JamRoomError):
    """Model loading or inference failure."""

class InferenceTimeoutError(ModelError):
    """Inference exceeded the time limit."""

class SecurityError(JamRoomError):
    """Input failed security validation."""
```

Use `raise NewError(...) from original_error` to preserve the exception chain.

### Error handling rules

- Log the error with `exc_info=True` before re-raising or swallowing.
- Re-raise as a domain exception (e.g., `ModelError`) rather than letting `subprocess.CalledProcessError` leak to the UI layer.
- Never silence errors in async callbacks — they will disappear silently. Always log them.

---

## Testing with pytest

### Basic test structure

```python
# tests/test_model_runner.py
import pytest
from pathlib import Path
from unittest.mock import AsyncMock, patch, MagicMock

from jamroom.models.runner import ModelRunner
from jamroom.exceptions import ModelError

@pytest.fixture
def model_runner(tmp_path: Path) -> ModelRunner:
    return ModelRunner(models_dir=tmp_path)

@pytest.mark.asyncio
async def test_run_raises_on_nonzero_exit(model_runner: ModelRunner) -> None:
    with patch("asyncio.create_subprocess_exec") as mock_exec:
        mock_proc = AsyncMock()
        mock_proc.returncode = 1
        mock_proc.communicate.return_value = (b"", b"model crashed")
        mock_exec.return_value = mock_proc

        with pytest.raises(ModelError, match="model crashed"):
            await model_runner.run("test-model", "hello")
```

### Mocking rules

- Use `unittest.mock.patch` as a context manager or decorator.
- Use `AsyncMock` for coroutines — regular `MagicMock` will not work with `await`.
- Use `tmp_path` (pytest built-in fixture) for any test that touches the filesystem.
- Do not mock at the wrong layer — mock at the boundary (subprocess, file I/O), not internal helpers.

### Pytest configuration

```ini
# pyproject.toml already covers this:
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"   # requires pytest-asyncio
```

### Running tests

```bash
pytest                        # all tests
pytest tests/test_config.py   # single file
pytest -k "test_load"         # tests matching name
pytest -x                     # stop on first failure
pytest --tb=short             # short tracebacks
```

---

## Linting: ruff

ruff is the single lint tool for this project. It replaces flake8, isort, pyupgrade, and black.

```bash
# Check (no changes)
ruff check src/ tests/

# Fix auto-fixable issues
ruff check --fix src/ tests/

# Format (replaces black)
ruff format src/ tests/

# Check formatting without applying
ruff format --check src/ tests/
```

Run both `ruff check` and `ruff format --check` before every commit. The configuration lives in `pyproject.toml` (see above).

**Do not install or use flake8, pylint, isort, or black separately.** ruff covers all of them.

---

## Type Checking: mypy (Strict Mode)

```bash
mypy src/
```

With `strict = true` in `pyproject.toml`, mypy enforces:
- All function arguments and return types must be annotated.
- No implicit `Any`.
- No untyped function bodies.
- `Optional[X]` must be explicitly checked before use.

### Common mypy errors and fixes

| Error | Fix |
|---|---|
| `Missing return type annotation` | Add `-> ReturnType` to the function signature |
| `Argument 1 has incompatible type "str"; expected "Path"` | Pass `Path(value)` |
| `Item "None" of "X \| None" has no attribute "Y"` | Guard with `if value is not None:` |
| `Cannot find implementation for "libname"` | Add a `py.typed` stub or `# type: ignore[import]` with a comment |

---

## Performance

- **Profile before optimizing.** Use `cProfile` to find actual bottlenecks:
  ```bash
  python -m cProfile -s cumtime -m jamroom > profile.txt
  ```
- **Memory profiling for ML models.** Use `tracemalloc` or `memray`:
  ```python
  import tracemalloc
  tracemalloc.start()
  # ... load model ...
  snapshot = tracemalloc.take_snapshot()
  for stat in snapshot.statistics("lineno")[:10]:
      print(stat)
  ```
- **Avoid premature micro-optimization.** String concatenation in a loop, list vs tuple, etc. — only matters if profiling shows it.
- **I/O is almost always the bottleneck**, not Python code. Use async I/O and thread pools for blocking calls.
- **Cache expensive pure computations** with `functools.lru_cache` or `functools.cache`:
  ```python
  from functools import cache

  @cache
  def get_model_metadata(model_path: Path) -> ModelMetadata:
      # parsed once, cached forever per path
      ...
  ```
