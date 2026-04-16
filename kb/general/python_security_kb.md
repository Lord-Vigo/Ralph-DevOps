---
name: Python Security KB
kb_type: general
maintained_by: Carl
---

# Python Security KB

Security requirements for JamRoom (LocalAI Music Studio). JamRoom is a local desktop app, but "local" does not mean "safe." Audio files from the internet can be malformed, the REST API accepts network requests (even localhost), and model files are downloaded from external sources. Apply every rule in this document.

---

## Per-IOC Security Checklist

Before implementing any IOC that touches user input, file I/O, subprocess, network, or database — answer every applicable question:

| # | Question | Applies to |
|---|---|---|
| 1 | Is all user input validated before use? | Any IOC |
| 2 | Is path traversal prevented? (resolved path stays inside allowed directory) | File I/O |
| 3 | Is subprocess called with an argument list, not `shell=True`? | subprocess |
| 4 | Is the executable path validated before passing to subprocess? | subprocess |
| 5 | Are all SQL queries parameterized? (no string formatting into SQL) | SQLite |
| 6 | Are downloaded file checksums verified before use? | Model downloads |
| 7 | Is `tempfile` module used (not hardcoded temp paths)? | Temp files |
| 8 | Are dependencies pinned and audited? | All IOCs |

If any applicable question is answered "no," resolve it before marking the IOC done. Document your decision if you consciously accept a risk.

---

## File Path Traversal Prevention

Any user-provided path (file picker, API input, config value) must be resolved to an absolute path and checked to ensure it stays within an allowed directory before use.

### Safe path validation pattern

```python
from pathlib import Path
from jamroom.exceptions import SecurityError

def validate_path_in_dir(user_input: str, allowed_dir: Path) -> Path:
    """
    Resolve user_input to an absolute path and confirm it is inside allowed_dir.
    Raises SecurityError if the path escapes the allowed directory.
    """
    resolved = Path(user_input).resolve()
    allowed = allowed_dir.resolve()
    try:
        resolved.relative_to(allowed)
    except ValueError:
        raise SecurityError(
            f"Path '{resolved}' is outside the allowed directory '{allowed}'"
        )
    return resolved
```

### Where to apply it

- Model file paths supplied via the REST API.
- Audio file paths selected in the TUI or via the API.
- Any config value that holds a file path.
- Paths constructed from user-provided names (e.g., `models_dir / user_model_name`).

### What this prevents

`../../etc/passwd`, symlink traversal, and Windows UNC paths like `\\server\share`. The `resolve()` call expands symlinks and normalizes `..` components before the check.

---

## subprocess Injection Prevention

Never call subprocess with `shell=True` when any argument originates from user input or external data. `shell=True` passes the command through the system shell (`/bin/sh` or `cmd.exe`), which interprets shell metacharacters.

### Unsafe — never do this

```python
# Attacker passes: "; rm -rf ~"
os.system(f"ollama run {user_model_name}")  # catastrophic

subprocess.run(f"whisper {audio_path}", shell=True)  # same problem
```

### Safe pattern — always use this

```python
import asyncio
from pathlib import Path

async def run_model(executable: Path, model_name: str, prompt: str) -> str:
    # All arguments are separate list elements — no shell interpolation
    proc = await asyncio.create_subprocess_exec(
        str(executable),       # validated path — not user string
        "--model", model_name, # validated model name
        "--prompt", prompt,    # arbitrary user text — safe as an element
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        # shell=True is NOT set — this is the default safe behavior
    )
    stdout, stderr = await proc.communicate()
    if proc.returncode != 0:
        raise ModelError(f"Model process failed: {stderr.decode()}")
    return stdout.decode()
```

### Validating executable paths

```python
import shutil

def resolve_executable(name: str, allowed_dir: Path | None = None) -> Path:
    """Find an executable by name, optionally restricted to a specific directory."""
    if allowed_dir is not None:
        candidate = allowed_dir / name
        if candidate.is_file() and os.access(candidate, os.X_OK):
            return candidate.resolve()
        raise SecurityError(f"Executable '{name}' not found in '{allowed_dir}'")

    found = shutil.which(name)
    if found is None:
        raise ModelError(f"Executable '{name}' not found in PATH")
    return Path(found).resolve()
```

---

## SQLite Security

If JamRoom uses SQLite (for model metadata, session history, etc.), all queries must use parameterized statements. Never format user data into a SQL string.

### Unsafe — never do this

```python
# SQL injection: user passes: ' OR '1'='1
cursor.execute(f"SELECT * FROM models WHERE name = '{user_input}'")
```

### Safe pattern — always use this

```python
import sqlite3
from pathlib import Path

def get_model_by_name(db_path: Path, name: str) -> dict | None:
    with sqlite3.connect(db_path) as conn:
        conn.row_factory = sqlite3.Row
        cursor = conn.execute(
            "SELECT * FROM models WHERE name = ?",  # ? placeholder
            (name,),                                 # tuple of values — never f-string
        )
        row = cursor.fetchone()
    return dict(row) if row else None

def insert_model(db_path: Path, name: str, path: str, checksum: str) -> None:
    with sqlite3.connect(db_path) as conn:
        conn.execute(
            "INSERT INTO models (name, path, checksum) VALUES (?, ?, ?)",
            (name, path, checksum),
        )
        conn.commit()
```

### Additional SQLite rules

- Use `with sqlite3.connect(...)` (context manager) — auto-commits on success, rolls back on exception.
- Set `conn.row_factory = sqlite3.Row` for dict-like row access.
- Store the DB in the app's user data directory (e.g., `~/.jamroom/jamroom.db`), not the working directory.
- Do not store sensitive user content in the DB if it can be avoided.

---

## Dependency Security

### Pin all dependencies

`requirements.txt` must contain fully pinned versions (generated by pip-tools). Never deploy with unpinned or range-only deps in production.

```
# requirements.txt (generated by pip-compile — do not hand-edit)
fastapi==0.111.0
textual==0.52.1
uvicorn==0.29.0
```

### Periodic auditing

```bash
# Install pip-audit
pip install pip-audit

# Audit installed packages against known CVEs
pip-audit

# Audit from requirements file (without installing)
pip-audit -r requirements.txt

# Audit and output as JSON for CI
pip-audit --format json -r requirements.txt -o audit-results.json
```

Run `pip-audit` before each release and whenever upgrading dependencies. Treat any high-severity finding as a blocker.

### Supply chain rules

- Pin to exact versions in `requirements.txt`.
- Review changelogs when upgrading packages that handle files, network, or parsing.
- Do not install packages from GitHub URLs or local paths in production builds.
- Prefer well-maintained packages with recent releases and active maintainers.

---

## Temp File Safety

Never hardcode temp paths like `/tmp/jamroom_temp.wav`. Use the `tempfile` module — it creates unique, secure names and respects the OS temp directory.

```python
import tempfile
from pathlib import Path

# Temporary file — deleted when the context exits
with tempfile.NamedTemporaryFile(
    suffix=".wav",
    prefix="jamroom_",
    delete=True,
) as tmp:
    tmp.write(audio_data)
    tmp.flush()
    process_audio(Path(tmp.name))
# file is deleted here

# Temporary directory — delete entire dir tree when done
with tempfile.TemporaryDirectory(prefix="jamroom_convert_") as tmp_dir:
    work_path = Path(tmp_dir) / "output.wav"
    run_conversion(output=work_path)
    result = work_path.read_bytes()
# directory deleted here
```

### Temp file rules

- Always use `tempfile.NamedTemporaryFile` or `tempfile.TemporaryDirectory` — never `open("/tmp/something")`.
- Use context managers (`with` statement) to ensure cleanup even if an exception occurs.
- Do not store sensitive audio data in temp files longer than needed.
- On Windows, `NamedTemporaryFile(delete=True)` cannot be opened by another process while open — use `delete=False` and clean up manually if subprocess needs to read the file:

```python
import os

tmp = tempfile.NamedTemporaryFile(suffix=".wav", delete=False)
try:
    tmp.write(audio_data)
    tmp.close()
    await run_model_process(Path(tmp.name))
finally:
    os.unlink(tmp.name)  # always clean up
```

---

## REST API Input Validation

Even though the API only listens on localhost, validate all inputs. A malicious local process or a misconfigured CORS policy could reach the API.

### Use Pydantic models for all request bodies (FastAPI)

```python
from pydantic import BaseModel, Field, field_validator
from pathlib import Path

class GenerateRequest(BaseModel):
    model: str = Field(..., min_length=1, max_length=100, pattern=r"^[\w\-\.]+$")
    prompt: str = Field(..., min_length=1, max_length=8192)
    temperature: float = Field(default=0.8, ge=0.0, le=2.0)
    max_tokens: int = Field(default=256, ge=1, le=4096)

    @field_validator("model")
    @classmethod
    def model_must_be_known(cls, v: str) -> str:
        # validated against the actual loaded model list at runtime
        return v  # defer runtime check to the route handler
```

### Validate path-related API parameters

```python
from fastapi import HTTPException

@router.get("/models/{model_name}/file")
async def get_model_file(model_name: str) -> FileResponse:
    # validate name is safe before constructing any path
    if not re.fullmatch(r"[\w\-\.]+", model_name):
        raise HTTPException(status_code=400, detail="Invalid model name")
    model_path = validate_path_in_dir(model_name, settings.models_dir)
    if not model_path.exists():
        raise HTTPException(status_code=404, detail="Model not found")
    return FileResponse(model_path)
```

### API security rules

- Return generic error messages to the client — do not expose internal paths or stack traces.
- Log the full error internally with `exc_info=True`.
- Do not accept file paths as API parameters if it can be avoided — accept identifiers (model names, IDs) and resolve them server-side.

---

## Model File Integrity

Downloaded model files (`.gguf`, etc.) must be verified against a known checksum before being loaded or executed.

### SHA-256 checksum verification

```python
import hashlib
from pathlib import Path
from jamroom.exceptions import SecurityError

CHUNK_SIZE = 65536  # 64 KB

def verify_checksum(file_path: Path, expected_sha256: str) -> None:
    """
    Compute the SHA-256 hash of file_path and compare to expected_sha256.
    Raises SecurityError if they do not match.
    """
    sha256 = hashlib.sha256()
    with file_path.open("rb") as f:
        while chunk := f.read(CHUNK_SIZE):
            sha256.update(chunk)
    actual = sha256.hexdigest()
    if actual != expected_sha256.lower():
        raise SecurityError(
            f"Checksum mismatch for '{file_path.name}': "
            f"expected {expected_sha256}, got {actual}"
        )
```

### Where to apply it

- After downloading a model file from any URL.
- After unzipping or unpacking a model archive.
- Before loading a model into inference for the first time (if the checksum is stored in the model registry).

### Storing checksums

Store expected checksums in the model registry (SQLite or JSON config) at download time, sourced from the model provider's official manifest. Do not allow the user to supply the checksum alongside the file — that defeats the purpose.

---

## Security Anti-Patterns Reference

| Anti-Pattern | Risk | Correct Approach |
|---|---|---|
| `shell=True` with any external data | Command injection | Argument list, `shell=False` |
| `f"SELECT ... WHERE x = '{val}'"` | SQL injection | Parameterized query with `?` placeholder |
| `open(f"/tmp/{filename}")` | Path traversal, race condition | `tempfile` module |
| `Path(user_input)` without `.resolve()` | Path traversal | `validate_path_in_dir()` |
| Logging user prompt text in production | Privacy / data leak | Log prompt length, not content |
| Unpinned dependencies | Supply chain / CVE exposure | Pinned `requirements.txt` + `pip-audit` |
| Trusting model file without checksum | Tampered/malicious model | Verify SHA-256 after download |
| Exposing stack traces in API responses | Information disclosure | Generic error to client, full log internally |
