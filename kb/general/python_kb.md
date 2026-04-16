---
name: Python Patterns & Usage KB
kb_type: general
maintained_by: Carl
---

# Python Patterns & Usage KB

Practical patterns for JamRoom (LocalAI Music Studio). Read this before implementing any IOC that involves async code, file I/O, subprocess calls, configuration, or threading.

---

## Async/Await Patterns (asyncio)

JamRoom uses asyncio as the primary concurrency model. The Textual TUI is async-native; the FastAPI REST layer is async-native. Do not mix blocking calls into the async event loop.

### Basic coroutine pattern

```python
import asyncio

async def fetch_model_status(model_id: str) -> dict:
    await asyncio.sleep(0)  # yield control — never block the loop
    return {"id": model_id, "status": "ready"}
```

### Running the event loop — single entry point

```python
# main.py — only one asyncio.run() call per process
if __name__ == "__main__":
    asyncio.run(main())
```

### Gathering concurrent tasks

```python
results = await asyncio.gather(
    load_model(path_a),
    load_model(path_b),
    return_exceptions=True,  # don't let one failure cancel the rest
)
for r in results:
    if isinstance(r, Exception):
        logger.error("model load failed", exc_info=r)
```

### Async context managers (for resource cleanup)

```python
async with aiofiles.open(audio_path, "rb") as f:
    data = await f.read()
```

### Timeouts

```python
try:
    result = await asyncio.wait_for(run_inference(prompt), timeout=30.0)
except asyncio.TimeoutError:
    raise InferenceTimeoutError(f"Inference exceeded 30s limit")
```

---

## Threading vs asyncio — When to Use Each

| Scenario | Use |
|---|---|
| TUI event loop (Textual) | asyncio |
| FastAPI request handlers | asyncio |
| Model inference (CPU/GPU bound, blocking C extensions) | `asyncio.run_in_executor` → thread pool |
| File watching (watchdog library) | thread (watchdog is thread-based) |
| Subprocess management | asyncio (`asyncio.create_subprocess_exec`) |
| Audio capture/playback (sounddevice callbacks) | thread (sounddevice is callback-based) |

### Running blocking inference in a thread without blocking the event loop

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

_executor = ThreadPoolExecutor(max_workers=2, thread_name_prefix="inference")

async def run_inference_async(prompt: str) -> str:
    loop = asyncio.get_running_loop()
    # blocking call runs in thread, async caller is not blocked
    return await loop.run_in_executor(_executor, _blocking_inference, prompt)

def _blocking_inference(prompt: str) -> str:
    # safe to call blocking C-extension model here
    return model.generate(prompt)
```

Never call `time.sleep()` inside a coroutine — use `await asyncio.sleep()`.

---

## Type Hints and Dataclasses

Use type hints everywhere. Run mypy in strict mode as part of CI.

### Standard type hint patterns

```python
from __future__ import annotations  # enables forward references in all Python 3.x

from typing import Optional, Union
from collections.abc import Sequence, Callable, AsyncIterator

def process_tracks(
    tracks: Sequence[str],
    callback: Callable[[str], None] | None = None,
) -> list[str]:
    ...
```

### Dataclass for structured data (preferred over plain dicts)

```python
from dataclasses import dataclass, field
from pathlib import Path

@dataclass
class ModelConfig:
    name: str
    path: Path
    context_size: int = 2048
    temperature: float = 0.8
    tags: list[str] = field(default_factory=list)

    def __post_init__(self) -> None:
        # validate immediately on construction
        if self.temperature < 0.0 or self.temperature > 2.0:
            raise ValueError(f"temperature must be 0.0–2.0, got {self.temperature}")
        self.path = Path(self.path)  # coerce str → Path if needed
```

### Dataclass JSON serialization (no third-party lib required)

```python
import json
import dataclasses
from pathlib import Path

def config_to_dict(cfg: ModelConfig) -> dict:
    d = dataclasses.asdict(cfg)
    d["path"] = str(cfg.path)  # Path is not JSON-serializable — convert manually
    return d

def config_from_dict(d: dict) -> ModelConfig:
    d["path"] = Path(d["path"])
    return ModelConfig(**d)

# Save
with open(config_path, "w", encoding="utf-8") as f:
    json.dump(config_to_dict(cfg), f, indent=2)

# Load
with open(config_path, "r", encoding="utf-8") as f:
    cfg = config_from_dict(json.load(f))
```

---

## pathlib for File Operations

Use `pathlib.Path` everywhere. Never use `os.path.join()` or string concatenation for paths.

```python
from pathlib import Path

# Constructing paths (cross-platform — works on Linux and Windows)
app_dir = Path.home() / ".jamroom"
models_dir = app_dir / "models"
config_file = app_dir / "config.json"

# Creating directories
models_dir.mkdir(parents=True, exist_ok=True)

# Checking existence
if config_file.exists():
    text = config_file.read_text(encoding="utf-8")

# Iterating files
for model_file in models_dir.glob("*.gguf"):
    print(model_file.name)  # filename only
    print(model_file.stem)  # filename without extension
    print(model_file.suffix)  # ".gguf"

# Resolving to absolute path (important before sandboxing)
resolved = Path(user_input).resolve()

# Safe relative path check (path traversal prevention)
try:
    resolved.relative_to(models_dir.resolve())
except ValueError:
    raise SecurityError("Path escapes the models directory")
```

### Cross-platform path rules

- Always use `pathlib.Path` — it handles `/` vs `\` automatically.
- Never hardcode `/` or `\\` as a separator in path strings.
- Use `Path.home()` not `os.environ["HOME"]` (HOME is not set on Windows).
- Use `Path(sys.executable).parent` to locate the app's own directory when bundled.
- Store paths in config as strings (`str(path)`) — reconstruct as `Path` on load.

---

## subprocess Safety

Use `asyncio.create_subprocess_exec` for async subprocess management. Never use `shell=True` with user-provided input.

### Preferred async subprocess pattern

```python
import asyncio

async def run_model_process(
    executable: Path,
    args: list[str],
) -> tuple[int, str, str]:
    proc = await asyncio.create_subprocess_exec(
        str(executable),
        *args,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    stdout, stderr = await proc.communicate()
    return proc.returncode, stdout.decode(), stderr.decode()
```

### Rules

| Rule | Reason |
|---|---|
| Always use argument list, never `shell=True` | shell=True + user input = injection |
| Validate executable path before use | Prevent running arbitrary binaries |
| Set `stdout=PIPE` and `stderr=PIPE` | Capture output, avoid polluting the TUI |
| Always check `returncode` | Treat non-zero as failure |
| Set a timeout via `asyncio.wait_for` | Prevent hung processes blocking shutdown |
| Use `proc.kill()` in finally blocks | Ensure child processes don't outlive the app |

---

## Config Patterns

Use a dataclass + JSON. Do not use `configparser` / `.ini` files — they lack type safety and nesting.

```python
@dataclass
class AppConfig:
    models_dir: Path = field(default_factory=lambda: Path.home() / ".jamroom" / "models")
    api_port: int = 8765
    log_level: str = "INFO"
    active_model: str | None = None

CONFIG_PATH = Path.home() / ".jamroom" / "config.json"

def load_config() -> AppConfig:
    if not CONFIG_PATH.exists():
        return AppConfig()  # sane defaults
    with CONFIG_PATH.open("r", encoding="utf-8") as f:
        data = json.load(f)
    data["models_dir"] = Path(data["models_dir"])
    return AppConfig(**data)

def save_config(cfg: AppConfig) -> None:
    CONFIG_PATH.parent.mkdir(parents=True, exist_ok=True)
    data = dataclasses.asdict(cfg)
    data["models_dir"] = str(cfg.models_dir)
    with CONFIG_PATH.open("w", encoding="utf-8") as f:
        json.dump(data, f, indent=2)
```

---

## Logging Patterns

Use the standard `logging` module. Apply it consistently — no `print()` statements in production code paths.

### Setup (call once at app startup)

```python
import logging
import sys

def configure_logging(level: str = "INFO") -> None:
    logging.basicConfig(
        level=getattr(logging, level.upper(), logging.INFO),
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S",
        handlers=[
            logging.StreamHandler(sys.stderr),
            logging.FileHandler(Path.home() / ".jamroom" / "jamroom.log", encoding="utf-8"),
        ],
    )
```

### Per-module loggers

```python
# In every module — use __name__, not a hardcoded string
logger = logging.getLogger(__name__)

logger.debug("Loading model: %s", model_path)
logger.info("Model loaded successfully")
logger.warning("Temperature clamped to 2.0 (was %s)", raw_value)
logger.error("Inference failed", exc_info=True)  # exc_info=True captures stack trace
```

### Logging rules

- Never log sensitive data (file contents, user credentials, prompt text in production builds).
- Use `%s` style formatting in log calls — not f-strings (avoids formatting cost when level is filtered).
- Log at `DEBUG` inside loops or hot paths — not `INFO`.
- Always include `exc_info=True` in `logger.error()` calls that catch exceptions.

---

## Entry Point Patterns for PyInstaller Bundling

PyInstaller requires a single importable entry point. Structure the app so `main()` is importable.

```python
# src/jamroom/main.py
def main() -> None:
    import asyncio
    from jamroom.app import JamRoomApp
    asyncio.run(JamRoomApp().run_async())

if __name__ == "__main__":
    main()
```

```toml
# pyproject.toml
[project.scripts]
jamroom = "jamroom.main:main"
```

```python
# jamroom.spec (PyInstaller)
# Always use the importable entry point, not a raw script path
a = Analysis(
    ["src/jamroom/main.py"],
    pathex=["src"],
    ...
)
```

### PyInstaller-specific gotchas

- Use `sys._MEIPASS` to locate bundled data files at runtime:
  ```python
  import sys, pathlib
  def get_resource(relative: str) -> Path:
      base = Path(getattr(sys, "_MEIPASS", Path(__file__).parent))
      return base / relative
  ```
- Do not use `__file__` directly for resource lookup — it breaks in frozen builds.
- Mark all data files explicitly in the `.spec` — PyInstaller won't auto-detect them.

---

## Common Cross-Platform Gotchas

| Issue | Linux | Windows | Fix |
|---|---|---|---|
| Path separator | `/` | `\` | Always use `pathlib.Path` |
| Line endings | `\n` | `\r\n` | Open text files with `newline=""` or let Python handle it |
| Home directory | `$HOME` | `%USERPROFILE%` | Use `Path.home()` |
| Case sensitivity | Paths are case-sensitive | Paths are case-insensitive | Normalize to lowercase before comparing |
| Executable extension | no extension | `.exe` | Check platform or use `shutil.which()` |
| Temp directory | `/tmp` | `%TEMP%` | Use `tempfile.gettempdir()` or `tempfile.mkdtemp()` |
| File locking | Advisory (fcntl) | Mandatory (Win32) | Avoid holding file handles across async yields |
| Subprocess env | inherits shell env | may not inherit PATH | Pass explicit `env=` dict when needed |

```python
import platform

IS_WINDOWS = platform.system() == "Windows"

def model_executable_name(base: str) -> str:
    return f"{base}.exe" if IS_WINDOWS else base
```
