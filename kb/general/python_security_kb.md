# Python Security Knowledge Base

Security reference for Python applications, aligned with OWASP ASVS and Secure by Design Framework principles.

---

## Table of Contents

1. [Secure by Design Principles](#secure-by-design-principles)
2. [Input Validation](#input-validation)
3. [Authentication & Authorization](#authentication--authorization)
4. [Password Security](#password-security)
5. [Cryptography](#cryptography)
6. [Secret Management](#secret-management)
7. [Common Vulnerabilities Prevention](#common-vulnerabilities-prevention)
8. [Secure Dependencies](#secure-dependencies)
9. [Secure Logging & Monitoring](#secure-logging--monitoring)
10. [Data Protection](#data-protection)
11. [API Security](#api-security)

---

## Secure by Design Principles

### Core Principles

```python
"""
Secure by Design Framework Principles:

1. MINIMIZE ATTACK SURFACE
   - Remove unnecessary features, ports, services
   - Disable debug modes in production
   - Use allowlists over denylists

2. DEFENSE IN DEPTH
   - Multiple layers of security controls
   - No single point of failure
   - Assume breach mentality

3. SECURE DEFAULTS
   - Most restrictive defaults
   - Fail securely
   - Cleartext by default over encrypted

4. FAIL SECURELY
   - Default deny policies
   - Graceful degradation
   - No information leakage in errors

5. LEAST PRIVILEGE
   - Minimum required permissions
   - Temporary elevated access
   - Regular access reviews
"""

# Example: Secure configuration class
from pydantic import BaseModel, Field
from typing import Optional

class SecurityConfig(BaseModel):
    """Secure configuration with defensive defaults."""
    
    debug: bool = Field(default=False, description="Debug mode")
    allowed_hosts: list[str] = Field(
        default=["localhost"],
        description="Allowed host headers"
    )
    cors_allow_origins: list[str] = Field(
        default=[],
        description="CORS allowed origins (empty = strict)"
    )
    max_request_size: int = Field(
        default=1_048_576,  # 1MB
        ge=0,
        le=10_485_760,
        description="Maximum request size in bytes"
    )
    rate_limit_per_minute: int = Field(
        default=60,
        ge=1,
        le=1000,
        description="Rate limit per minute"
    )
    
    def validate_production(self) -> list[str]:
        """Validate security for production environment."""
        issues = []
        
        if self.debug:
            issues.append("Debug mode must be disabled in production")
        
        if "*" in self.cors_allow_origins:
            issues.append("Wildcard CORS origins not allowed in production")
        
        if len(self.allowed_hosts) > 5:
            issues.append("Consider limiting allowed hosts")
            
        return issues
```

---

## Input Validation

### OWASP Input Validation Rules

```python
"""
OWASP ASVS V5: Validation, Sanitization and Encoding

- Validate all input data (length, type, format, range)
- Use positive allowlists where possible
- Encode output to prevent injection
- Validate on server-side only (client-side is辅助)
"""

from pydantic import BaseModel, field_validator, model_validator
from typing import Any
import re

class ValidatedUserInput(BaseModel):
    """Input validation following OWASP guidelines."""
    
    username: str = Field(..., min_length=3, max_length=50)
    email: str
    age: int | None = Field(None, ge=0, le=150)
    url: str | None = None
    phone: str | None = None
    
    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        """Allowlist validation for username."""
        # Only allow alphanumeric and underscore
        if not re.match(r"^[a-zA-Z][a-zA-Z0-9_]*$", v):
            raise ValueError(
                "Username must start with letter, contain only "
                "alphanumeric and underscore"
            )
        return v
    
    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        """Validate email format."""
        pattern = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
        if not re.match(pattern, v):
            raise ValueError("Invalid email format")
        return v.lower()
    
    @field_validator("url")
    @classmethod
    def validate_url(cls, v: str | None) -> str | None:
        """Validate and sanitize URL."""
        if v is None:
            return v
        
        # Only allow http and https
        if not v.lower().startswith(("http://", "https://")):
            raise ValueError("URL must start with http:// or https://")
        
        # Block dangerous schemes
        dangerous = ("javascript:", "data:", "vbscript:")
        if any(v.lower().startswith(s) for s in dangerous):
            raise ValueError("Dangerous URL scheme not allowed")
        
        return v
    
    @field_validator("phone")
    @classmethod
    def validate_phone(cls, v: str | None) -> str | None:
        """Validate phone number format."""
        if v is None:
            return v
        
        # Remove common formatting characters
        cleaned = re.sub(r"[\s\-\(\)\.\+]", "", v)
        
        if not re.match(r"^\d{7,15}$", cleaned):
            raise ValueError("Invalid phone number format")
        
        return cleaned
```

### Path Validation

```python
from pathlib import Path
import os

def safe_file_access(base_dir: Path, user_provided_path: str) -> Path:
    """Prevent path traversal attacks.
    
    OWASP A01:2021 - Broken Access Control
    """
    # Resolve the path and check it's within base directory
    requested_path = (base_dir / user_provided_path).resolve()
    
    # Ensure the resolved path is within the allowed directory
    try:
        requested_path.relative_to(base_dir)
    except ValueError:
        raise PermissionError("Access denied: path outside allowed directory")
    
    # Additional check for symlinks
    if requested_path.is_symlink():
        raise PermissionError("Symlinks not allowed")
    
    return requested_path

# Usage
def read_user_file(filename: str) -> str:
    """Safely read a file from user's upload directory."""
    base_dir = Path("/var/uploads")
    
    try:
        file_path = safe_file_access(base_dir, filename)
    except PermissionError as e:
        raise ValueError(str(e))
    
    if not file_path.exists():
        raise FileNotFoundError("File not found")
    
    return file_path.read_text()
```

### SQL Injection Prevention

```python
"""
OWASP A03:2021 - Injection
Always use parameterized queries, never string concatenation
"""

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    """Secure database query using parameterization."""
    # GOOD: Parameterized query
    result = await db.execute(
        text("SELECT * FROM users WHERE email = :email"),
        {"email": email}
    )
    return result.fetchone()

# BAD: Never do this!
# query = f"SELECT * FROM users WHERE email = '{email}'"  # SQL Injection!

# For complex queries, use ORM's built-in parameterization
from sqlalchemy import select

async def get_active_users(db: AsyncSession) -> list[User]:
    """Secure query using SQLAlchemy ORM."""
    stmt = select(User).where(User.is_active == True)
    result = await db.execute(stmt)
    return list(result.scalars().all())

# For dynamic WHERE clauses
def build_safe_filter(filters: dict) -> list:
    """Build safe filter conditions."""
    allowed_fields = {"email", "username", "created_at", "is_active"}
    conditions = []
    
    for field, value in filters.items():
        if field not in allowed_fields:
            continue  # Skip unknown fields
        
        if isinstance(value, str):
            # Use ilike for case-insensitive matching
            conditions.append(getattr(User, field).ilike(value))
        else:
            conditions.append(getattr(User, field) == value)
    
    return conditions
```

---

## Authentication & Authorization

### JWT Implementation

```python
import jwt
from datetime import datetime, timedelta, timezone
from typing import Any
from pydantic import BaseModel
from fastapi import HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import secrets

class TokenPayload(BaseModel):
    """JWT token payload."""
    sub: str  # user_id
    exp: datetime
    iat: datetime
    scopes: list[str] = []

class JWTManager:
    """Secure JWT token management."""
    
    def __init__(
        self,
        secret_key: str,
        algorithm: str = "HS256",
        access_token_expire_minutes: int = 30,
        refresh_token_expire_days: int = 7
    ):
        self.secret_key = secret_key
        self.algorithm = algorithm
        self.access_token_expire = timedelta(minutes=access_token_expire_minutes)
        self.refresh_token_expire = timedelta(days=refresh_token_expire_days)
    
    def create_access_token(
        self,
        user_id: str,
        scopes: list[str] | None = None
    ) -> tuple[str, datetime]:
        """Create a new access token."""
        now = datetime.now(timezone.utc)
        expires = now + self.access_token_expire
        
        payload = {
            "sub": user_id,
            "iat": now,
            "exp": expires,
            "type": "access",
            "scopes": scopes or [],
        }
        
        token = jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
        return token, expires
    
    def create_refresh_token(self, user_id: str) -> tuple[str, datetime]:
        """Create a new refresh token."""
        now = datetime.now(timezone.utc)
        expires = now + self.refresh_token_expire
        
        # Use separate secret for refresh tokens
        payload = {
            "sub": user_id,
            "iat": now,
            "exp": expires,
            "type": "refresh",
            "jti": secrets.token_urlsafe(16),  # Unique identifier
        }
        
        token = jwt.encode(
            payload, 
            self.secret_key + "_refresh", 
            algorithm=self.algorithm
        )
        return token, expires
    
    def verify_token(
        self,
        token: str,
        token_type: str = "access"
    ) -> TokenPayload:
        """Verify and decode token."""
        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm]
            )
            
            if payload.get("type") != token_type:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid token type"
                )
            
            return TokenPayload(**payload)
            
        except jwt.ExpiredSignatureError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token has expired"
            )
        except jwt.InvalidTokenError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )

# Usage with FastAPI
security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    jwt_manager: JWTManager = Depends(get_jwt_manager)
) -> User:
    """Dependency to get current authenticated user."""
    token_payload = jwt_manager.verify_token(credentials.credentials)
    
    user = await get_user_by_id(token_payload.sub)
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    
    return user
```

### Role-Based Access Control (RBAC)

```python
from enum import Enum
from functools import wraps
from fastapi import HTTPException, status
from typing import Callable

class Role(str, Enum):
    """Application roles."""
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"
    GUEST = "guest"

class Permission(str, Enum):
    """Granular permissions."""
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    ADMIN = "admin"
    MODERATE = "moderate"

# Role-permission mapping
ROLE_PERMISSIONS: dict[Role, set[Permission]] = {
    Role.ADMIN: {Permission.READ, Permission.WRITE, Permission.DELETE, Permission.ADMIN},
    Role.MODERATOR: {Permission.READ, Permission.WRITE, Permission.MODERATE},
    Role.USER: {Permission.READ, Permission.WRITE},
    Role.GUEST: {Permission.READ},
}

def require_permissions(*required: Permission) -> Callable:
    """Decorator to check permissions."""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, user: User, **kwargs):
            if user.role is None:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="No role assigned"
                )
            
            user_permissions = ROLE_PERMISSIONS.get(user.role, set())
            
            if not all(p in user_permissions for p in required):
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="Insufficient permissions"
                )
            
            return await func(*args, user=user, **kwargs)
        return wrapper
    return decorator

def require_roles(*required: Role) -> Callable:
    """Decorator to check roles."""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, user: User, **kwargs):
            if user.role not in required:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="Insufficient role privileges"
                )
            
            return await func(*args, user=user, **kwargs)
        return wrapper
    return decorator

# Usage in FastAPI endpoints
@app.delete("/users/{user_id}")
@require_permissions(Permission.DELETE)
async def delete_user(
    user_id: int,
    user: User = Depends(get_current_user)
):
    """Delete a user - requires DELETE permission."""
    ...

@app.post("/admin/users")
@require_roles(Role.ADMIN)
async def create_admin_user(
    user_data: UserCreate,
    user: User = Depends(get_current_user)
):
    """Create admin user - requires ADMIN role."""
    ...
```

---

## Password Security

```python
import bcrypt
from pydantic import BaseModel, field_validator
from typing import Any

class PasswordStrength:
    """Password strength validation."""
    
    @staticmethod
    def validate(password: str) -> tuple[bool, str]:
        """Validate password strength.
        
        Returns:
            Tuple of (is_valid, error_message)
        """
        if len(password) < 8:
            return False, "Password must be at least 8 characters"
        
        if len(password) > 128:
            return False, "Password too long (max 128 characters)"
        
        if not any(c.isupper() for c in password):
            return False, "Password must contain uppercase letter"
        
        if not any(c.islower() for c in password):
            return False, "Password must contain lowercase letter"
        
        if not any(c.isdigit() for c in password):
            return False, "Password must contain digit"
        
        # Check for common patterns
        common_patterns = [
            "password", "123456", "qwerty", "admin",
            "letmein", "welcome", "monkey", "dragon"
        ]
        if password.lower() in common_patterns:
            return False, "Password too common"
        
        # Check for keyboard patterns
        keyboard_patterns = [
            "qwerty", "asdfgh", "zxcvbn", "1234567890"
        ]
        lower_pass = password.lower()
        for pattern in keyboard_patterns:
            if pattern in lower_pass or pattern[::-1] in lower_pass:
                return False, "Password contains keyboard pattern"
        
        return True, ""

class PasswordHasher:
    """Secure password hashing using bcrypt."""
    
    @staticmethod
    def hash(password: str) -> str:
        """Hash a password using bcrypt."""
        # Generate salt with work factor 12
        salt = bcrypt.gensalt(rounds=12)
        hashed = bcrypt.hashpw(password.encode("utf-8"), salt)
        return hashed.decode("utf-8")
    
    @staticmethod
    def verify(plain_password: str, hashed_password: str) -> bool:
        """Verify password against hash."""
        try:
            return bcrypt.checkpw(
                plain_password.encode("utf-8"),
                hashed_password.encode("utf-8")
            )
        except Exception:
            # On any error, return False (timing-safe)
            return False
    
    @staticmethod
    def needs_rehash(hashed_password: str) -> bool:
        """Check if password hash needs to be upgraded."""
        # Extract work factor from hash
        # bcrypt hashes look like $2b$12$...
        try:
            parts = hashed_password.split("$")
            if len(parts) >= 3:
                work_factor = int(parts[2])
                return work_factor < 12  # Upgrade if less than 12
        except (ValueError, IndexError):
            pass
        return True

class SecurePassword(BaseModel):
    """Password field with secure handling."""
    
    password: str
    confirm_password: str | None = None
    
    @field_validator("password")
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        is_valid, error = PasswordStrength.validate(v)
        if not is_valid:
            raise ValueError(error)
        return v
    
    @model_validator(mode="after")
    def validate_match(self) -> "SecurePassword":
        if (
            self.confirm_password is not None 
            and self.password != self.confirm_password
        ):
            raise ValueError("Passwords do not match")
        return self
```

---

## Cryptography

```python
"""
Secure cryptographic operations using cryptography library.
NEVER implement your own crypto - use well-tested libraries.
"""

from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
import base64
import os
from typing import Any

class SymmetricEncryption:
    """Fernet symmetric encryption (AES-128-CBC with HMAC)."""
    
    def __init__(self, key: bytes | None = None):
        if key is None:
            key = Fernet.generate_key()
        elif isinstance(key, str):
            key = key.encode()
        
        self.cipher = Fernet(key)
    
    @classmethod
    def derive_key(cls, password: str, salt: bytes | None = None) -> "SymmetricEncryption":
        """Derive key from password using PBKDF2."""
        if salt is None:
            salt = os.urandom(16)
        
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100_000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(password.encode()))
        instance = cls(key)
        instance._salt = salt
        return instance
    
    def encrypt(self, data: str | bytes) -> bytes:
        """Encrypt data."""
        if isinstance(data, str):
            data = data.encode()
        return self.cipher.encrypt(data)
    
    def decrypt(self, encrypted_data: bytes) -> bytes:
        """Decrypt data."""
        return self.cipher.decrypt(encrypted_data)


class AsymmetricEncryption:
    """RSA asymmetric encryption for key exchange."""
    
    @staticmethod
    def generate_keypair(
        key_size: int = 2048,
        public_exponent: int = 65537
    ) -> tuple[bytes, bytes]:
        """Generate RSA keypair."""
        private_key = rsa.generate_private_key(
            public_exponent=public_exponent,
            key_size=key_size,
        )
        
        public_key = private_key.public_key()
        
        # Serialize keys
        private_pem = private_key.private_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PrivateFormat.PKCS8,
            encryption_algorithm=serialization.NoEncryption()
        )
        
        public_pem = public_key.public_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PublicFormat.SubjectPublicKeyInfo
        )
        
        return private_pem, public_pem
    
    @staticmethod
    def encrypt(public_key_pem: bytes, data: bytes) -> bytes:
        """Encrypt with public key."""
        public_key = serialization.load_pem_public_key(public_key_pem)
        
        return public_key.encrypt(
            data,
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )
    
    @staticmethod
    def decrypt(private_key_pem: bytes, encrypted_data: bytes) -> bytes:
        """Decrypt with private key."""
        private_key = serialization.load_pem_private_key(
            private_key_pem,
            password=None
        )
        
        return private_key.decrypt(
            encrypted_data,
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )


class Hasher:
    """Secure hashing for data integrity."""
    
    @staticmethod
    def sha256(data: str | bytes) -> str:
        """Generate SHA-256 hash."""
        from cryptography.hazmat.primitives import hashes
        
        if isinstance(data, str):
            data = data.encode()
        
        from cryptography.hazmat.backends import default_backend
        hash_obj = hashes.Hash(hashes.SHA256(), backend=default_backend())
        hash_obj.update(data)
        return hash_obj.finalize().hex()
    
    @staticmethod
    def generate_salt(length: int = 32) -> bytes:
        """Generate cryptographically secure random salt."""
        return os.urandom(length)
```

---

## Secret Management

```python
"""
OWASP A02:2021 - Cryptographic Failures
OWASP A04:2021 - Insecure Design

Never hardcode secrets! Use environment variables or secrets management.
"""

import os
import json
from typing import Any
from functools import lru_cache

class SecretManager:
    """Secure secret management with multiple backends."""
    
    def __init__(self):
        self._secrets_cache: dict[str, str] = {}
    
    def get(self, key: str, default: str | None = None) -> str | None:
        """Get secret from environment or secrets manager."""
        # Check cache first
        if key in self._secrets_cache:
            return self._secrets_cache[key]
        
        # Check environment variable
        value = os.environ.get(key, default)
        
        if value is not None:
            self._secrets_cache[key] = value
        
        return value
    
    def get_required(self, key: str) -> str:
        """Get secret, raising error if not found."""
        value = self.get(key)
        if value is None:
            raise RuntimeError(f"Required secret '{key}' is not set")
        return value
    
    def get_database_url(self) -> str:
        """Get database URL with proper redaction for logging."""
        url = self.get_required("DATABASE_URL")
        
        # Redact password in URL
        if "@" in url:
            # postgresql://user:pass@host/db -> postgresql://user:***@host/db
            parts = url.split("@")
            user_pass = parts[0].split("://")
            if len(user_pass) > 1:
                user_part = user_pass[1].split(":")
                if len(user_part) > 1:
                    user_pass[1] = f"{user_part[0]}:***"
                    parts[0] = "://".join(user_pass)
            return "@".join(parts)
        
        return url

# Environment variable validation
def validate_environment() -> list[str]:
    """Validate required environment variables."""
    required = [
        "SECRET_KEY",
        "DATABASE_URL",
        "REDIS_URL",
    ]
    
    missing = []
    for var in required:
        if not os.environ.get(var):
            missing.append(var)
    
    return missing

# .env file handling (development only)
from pathlib import Path

def load_dotenv(path: Path | None = None) -> None:
    """Load .env file for development."""
    if path is None:
        path = Path(".env")
    
    if not path.exists():
        return
    
    # Don't load in production
    if os.environ.get("ENV") == "production":
        return
    
    with open(path) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith("#"):
                key, _, value = line.partition("=")
                os.environ.setdefault(key.strip(), value.strip())
```

---

## Common Vulnerabilities Prevention

### SSRF Prevention (Server-Side Request Forgery)

```python
"""
OWASP A10:2021 - Server-Side Request Forgery

Never fetch URLs provided by users without validation.
"""

import socket
import ipaddress
from urllib.parse import urlparse
from functools import lru_cache

class URLValidator:
    """Validate URLs to prevent SSRF attacks."""
    
    # Private IP ranges to block
    BLOCKED_IP_RANGES = [
        "10.0.0.0/8",
        "172.16.0.0/12",
        "192.168.0.0/16",
        "127.0.0.0/8",
        "169.254.0.0/16",  # Link-local
        "0.0.0.0/8",
        "100.64.0.0/10",  # Carrier-grade NAT
        "192.0.0.0/24",
        "192.0.2.0/24",  # TEST-NET-1
        "198.51.100.0/24",  # TEST-NET-2
        "203.0.113.0/24",  # TEST-NET-3
        "fc00::/7",  # IPv6 private
        "fe80::/10",  # IPv6 link-local
    ]
    
    BLOCKED_HOSTS = {
        "localhost",
        "localhost.localdomain",
        "metadata.google.internal",
        "169.254.169.254",  # Cloud metadata
        "metadata.google",
    }
    
    ALLOWED_SCHEMES = {"http", "https"}
    
    @classmethod
    def validate(cls, url: str) -> tuple[bool, str]:
        """Validate URL for safety."""
        try:
            parsed = urlparse(url)
        except Exception:
            return False, "Invalid URL format"
        
        # Check scheme
        if parsed.scheme not in cls.ALLOWED_SCHEMES:
            return False, f"Scheme must be one of {cls.ALLOWED_SCHEMES}"
        
        # Check hostname
        hostname = parsed.hostname
        if hostname is None:
            return False, "Invalid hostname"
        
        # Check blocked hosts
        if hostname.lower() in cls.BLOCKED_HOSTS:
            return False, f"Host '{hostname}' is not allowed"
        
        # Resolve and check IP
        try:
            ip_str = socket.gethostbyname(hostname)
            ip = ipaddress.ip_address(ip_str)
        except (socket.gaierror, ValueError):
            return False, "Could not resolve hostname"
        
        # Check against blocked ranges
        for cidr in cls.BLOCKED_IP_RANGES:
            network = ipaddress.ip_network(cidr, strict=False)
            if ip in network:
                return False, f"IP in blocked range ({cidr})"
        
        return True, ""
    
    @classmethod
    def validate_or_raise(cls, url: str) -> None:
        """Validate URL and raise if invalid."""
        is_valid, error = cls.validate(url)
        if not is_valid:
            raise ValueError(f"URL validation failed: {error}")


def safe_fetch_url(url: str) -> bytes:
    """Safely fetch a URL with SSRF protection."""
    URLValidator.validate_or_raise(url)
    
    # Now safe to fetch
    import httpx
    response = httpx.get(url, timeout=10)
    return response.content
```

### XXE Prevention (XML External Entity)

```python
"""
OWASP A04:2021 - Insecure Design - XXE Prevention

Disable external entity processing when parsing XML.
"""

import defusedxml.ElementTree as ET
from lxml import etree

def safe_parse_xml(xml_string: str) -> etree._Element:
    """Safely parse XML without external entity expansion."""
    
    # Using defusedxml - safe by default
    parser = ET.DefusedXMLParser()
    return ET.fromstring(xml_string, parser=parser)

def safe_parse_lxml(xml_string: str) -> etree._Element:
    """Safely parse XML using lxml with XXE disabled."""
    
    parser = etree.XMLParser(
        no_network=True,  # Block network access
        resolve_entities=False,  # Disable entity resolution
        external_entities=False,  # Disable external entities
        remove_pis=True,  # Remove processing instructions
        remove_comments=True,  # Remove comments
    )
    
    return etree.fromstring(xml_string, parser=parser)
```

### Command Injection Prevention

```python
"""
OWASP A03:2021 - Injection - Command Injection

Never use user input in shell commands!
"""

import subprocess
import shlex
from typing import list

def safe_execute_command(
    command: str,
    args: list[str],
    timeout: int = 30
) -> str:
    """Execute command safely without shell=True."""
    
    # Use list form - never shell=True with user input!
    # BAD: subprocess.run(f"ls {user_input}", shell=True)
    
    # GOOD: List form prevents injection
    result = subprocess.run(
        [command] + args,
        capture_output=True,
        text=True,
        timeout=timeout,
        check=False  # Don't raise on non-zero exit
    )
    
    if result.returncode != 0:
        raise RuntimeError(f"Command failed: {result.stderr}")
    
    return result.stdout


def safe_list_files(directory: str) -> list[str]:
    """List files in directory safely."""
    import os
    
    # Use os.listdir - no shell involved
    # For more control, use pathlib
    from pathlib import Path
    
    base_path = Path("/safe/base/directory")
    target_path = base_path / directory
    
    # Resolve and check it's within base
    resolved = target_path.resolve()
    if not str(resolved).startswith(str(base_path.resolve())):
        raise ValueError("Path traversal detected")
    
    if not resolved.is_dir():
        raise ValueError("Not a directory")
    
    return [str(p.name) for p in resolved.iterdir()]
```

---

## Secure Dependencies

### Dependency Security

```python
"""
OWASP A06:2021 - Vulnerable and Outdated Components

Regularly audit and update dependencies.
"""

import subprocess
import json
from pathlib import Path
from typing import NamedTuple

class SecurityAuditResult(NamedTuple):
    """Results of security audit."""
    vulnerabilities: int
    critical: int
    high: int
    medium: int
    low: int

def audit_dependencies() -> SecurityAuditResult:
    """Run security audit on dependencies."""
    
    # Using pip-audit
    result = subprocess.run(
        ["pip-audit", "--format=json"],
        capture_output=True,
        text=True
    )
    
    if result.returncode not in (0, 1):  # 1 = vulnerabilities found
        raise RuntimeError(f"pip-audit failed: {result.stderr}")
    
    try:
        data = json.loads(result.stdout)
        deps = data.get("dependencies", [])
        
        critical = high = medium = low = 0
        
        for dep in deps:
            for vuln in dep.get("vulns", []):
                severity = vuln.get("severity", "").lower()
                if severity == "critical":
                    critical += 1
                elif severity == "high":
                    high += 1
                elif severity == "medium":
                    medium += 1
                else:
                    low += 1
        
        return SecurityAuditResult(
            vulnerabilities=len(deps),
            critical=critical,
            high=high,
            medium=medium,
            low=low
        )
    except json.JSONDecodeError:
        return SecurityAuditResult(0, 0, 0, 0, 0)

# Requirements file with security
# requirements.in
"""
# Use pip-compile to generate locked requirements
# Always specify version constraints for security
fastapi>=0.100.0,<1.0.0
pydantic>=2.0.0,<3.0.0
python-jose[cryptography]>=3.3.0,<4.0.0
"""

# Security-specific dependencies
SECURITY_DEPS = {
    "authentication": ["python-jose[cryptography]", "passlib[bcrypt]"],
    "encryption": ["cryptography"],
    "validation": ["pydantic>=2.0.0"],
    "api": ["fastapi>=0.100.0", "httpx"],
}
```

---

## Secure Logging & Monitoring

### Security Logging

```python
"""
OWASP A09:2021 - Security Logging Failures

Log security-relevant events without sensitive data exposure.
"""

import logging
import json
from datetime import datetime
from typing import Any
from functools import wraps
import hashlib
import re

class SecurityLogger:
    """Secure logging for security events."""
    
    SENSITIVE_FIELDS = {
        "password", "secret", "token", "api_key", "auth",
        "credit_card", "ssn", "social_security",
        "access_token", "refresh_token", "private_key"
    }
    
    def __init__(self, logger_name: str):
        self.logger = logging.getLogger(logger_name)
    
    @staticmethod
    def sanitize_data(data: dict) -> dict:
        """Remove sensitive fields from data."""
        sanitized = {}
        
        for key, value in data.items():
            key_lower = key.lower()
            
            # Check if field is sensitive
            if any(field in key_lower for field in SecurityLogger.SENSITIVE_FIELDS):
                sanitized[key] = "[REDACTED]"
            elif isinstance(value, dict):
                sanitized[key] = SecurityLogger.sanitize_data(value)
            else:
                sanitized[key] = value
        
        return sanitized
    
    @staticmethod
    def hash_identifier(value: str) -> str:
        """Hash identifiers for logging (preserves uniqueness)."""
        return hashlib.sha256(value.encode()).hexdigest()[:16]
    
    def log_authentication(
        self,
        event: str,
        user_id: str | None,
        success: bool,
        metadata: dict | None = None
    ) -> None:
        """Log authentication event."""
        self.logger.info(
            "auth_event",
            extra={
                "event_type": "authentication",
                "event": event,
                "user_hash": self.hash_identifier(user_id) if user_id else None,
                "success": success,
                "timestamp": datetime.utcnow().isoformat(),
                "metadata": self.sanitize_data(metadata or {})
            }
        )
    
    def log_access_denied(
        self,
        resource: str,
        user_id: str | None,
        reason: str
    ) -> None:
        """Log access denied event."""
        self.logger.warning(
            "access_denied",
            extra={
                "event_type": "access_control",
                "resource": resource,
                "user_hash": self.hash_identifier(user_id) if user_id else None,
                "reason": reason,
                "timestamp": datetime.utcnow().isoformat()
            }
        )
    
    def log_suspicious_activity(
        self,
        description: str,
        ip_address: str | None = None,
        user_id: str | None = None,
        metadata: dict | None = None
    ) -> None:
        """Log suspicious activity for security monitoring."""
        self.logger.warning(
            "suspicious_activity",
            extra={
                "event_type": "security",
                "description": description,
                "ip_hash": self.hash_identifier(ip_address) if ip_address else None,
                "user_hash": self.hash_identifier(user_id) if user_id else None,
                "timestamp": datetime.utcnow().isoformat(),
                "metadata": self.sanitize_data(metadata or {})
            }
        )


def log_function_call(logger: SecurityLogger):
    """Decorator to log function calls securely."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Log function call (sanitize args)
            sanitized_args = SecurityLogger.sanitize_data({
                "args": str(args),
                "kwargs": kwargs
            })
            
            logger.logger.debug(
                f"function_call: {func.__name__}",
                extra=sanitized_args
            )
            
            try:
                result = func(*args, **kwargs)
                logger.logger.debug(f"function_success: {func.__name__}")
                return result
            except Exception as e:
                logger.logger.error(
                    f"function_error: {func.__name__}",
                    extra={"error": str(type(e).__name__)}
                )
                raise
        
        return wrapper
    return decorator
```

---

## Data Protection

### PII Handling

```python
"""
OWASP A04:2021 - Insecure Design - PII Protection

Handle Personally Identifiable Information securely.
"""

from dataclasses import dataclass
from typing import Any
from datetime import datetime, timedelta
import hashlib
import re

class PIIRedactor:
    """Redact PII from data for safe logging/storage."""
    
    PATTERNS = {
        "email": re.compile(r"[\w\.-]+@[\w\.-]+\.\w+"),
        "phone": re.compile(r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b"),
        "ssn": re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),
        "credit_card": re.compile(r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b"),
    }
    
    @classmethod
    def redact(cls, text: str, patterns: list[str] | None = None) -> str:
        """Redact PII from text."""
        if patterns is None:
            patterns = list(cls.PATTERNS.keys())
        
        result = text
        
        for pattern_name in patterns:
            if pattern_name in cls.PATTERNS:
                result = cls.PATTERNS[pattern_name].sub("[REDACTED]", result)
        
        return result
    
    @classmethod
    def redact_dict(cls, data: dict) -> dict:
        """Recursively redact PII from dictionary."""
        sensitive_keys = {"email", "phone", "ssn", "credit_card", "password"}
        
        result = {}
        for key, value in data.items():
            if key.lower() in sensitive_keys:
                result[key] = "[REDACTED]"
            elif isinstance(value, dict):
                result[key] = cls.redact_dict(value)
            elif isinstance(value, str):
                result[key] = cls.redact(value)
            else:
                result[key] = value
        
        return result


@dataclass
class DataRetentionPolicy:
    """Data retention policy configuration."""
    
    # Retention periods in days
    session_data: int = 30
    access_logs: int = 365
    error_logs: int = 90
    user_data: int = 2555  # 7 years
    transaction_data: int = 2555
    
    def should_delete(self, data_type: str, created_at: datetime) -> bool:
        """Check if data should be deleted based on retention policy."""
        retention_days = {
            "session_data": self.session_data,
            "access_logs": self.access_logs,
            "error_logs": self.error_logs,
            "user_data": self.user_data,
            "transaction_data": self.transaction_data,
        }
        
        days = retention_days.get(data_type)
        if days is None:
            return False
        
        cutoff = datetime.utcnow() - timedelta(days=days)
        return created_at < cutoff
```

---

## API Security

### Rate Limiting

```python
"""
OWASP A04:2021 - Insecure Design - Rate Limiting

Implement rate limiting to prevent abuse.
"""

import time
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Callable
from functools import wraps
import asyncio
import httpx

class RateLimitConfig:
    """Rate limiting configuration."""
    
    def __init__(
        self,
        requests: int,
        window_seconds: int,
        block_duration: int = 60
    ):
        self.requests = requests
        self.window_seconds = window_seconds
        self.block_duration = block_duration


class RateLimiter:
    """Token bucket rate limiter."""
    
    def __init__(self, config: RateLimitConfig):
        self.config = config
        self._buckets: dict[str, tuple[int, float, float]] = defaultdict(
            lambda: (config.requests, time.time(), 0.0)
        )
        self._blocks: dict[str, float] = {}
    
    async def check(self, key: str) -> tuple[bool, int, int]:
        """Check if request is allowed.
        
        Returns:
            (is_allowed, current_count, remaining)
        """
        # Check if blocked
        if key in self._blocks:
            if time.time() < self._blocks[key]:
                return False, 0, 0
            else:
                del self._blocks[key]
        
        count, window_start, tokens = self._buckets[key]
        
        # Reset window if expired
        current_time = time.time()
        if current_time - window_start >= self.config.window_seconds:
            self._buckets[key] = (
                self.config.requests,
                current_time,
                float(self.config.requests)
            )
            count = self.config.requests
            tokens = float(self.config.requests)
        
        if tokens >= 1:
            self._buckets[key] = (
                count - 1,
                window_start,
                tokens - 1
            )
            return True, count - 1, int(tokens - 1)
        else:
            # Block user
            self._blocks[key] = current_time + self.config.block_duration
            return False, 0, 0
    
    def get_limit_headers(self, key: str) -> dict[str, str]:
        """Get rate limit headers."""
        count, _, _ = self._buckets[key]
        return {
            "X-RateLimit-Limit": str(self.config.requests),
            "X-RateLimit-Remaining": str(max(0, count - 1)),
            "X-RateLimit-Reset": str(
                int(self._buckets[key][1] + self.config.window_seconds)
            )
        }


# FastAPI integration
from fastapi import Request, HTTPException, status, Depends

rate_limiter = RateLimitConfig(requests=100, window_seconds=60)

async def rate_limit_middleware(request: Request, limiter: RateLimiter):
    """Rate limiting middleware."""
    # Use IP + user agent as key
    key = f"{request.client.host}:{request.headers.get('user-agent', 'unknown')}"
    
    allowed, current, remaining = await limiter.check(key)
    
    if not allowed:
        raise HTTPException(
            status_code=status.HTTP_429_TOO_MANY_REQUESTS,
            detail="Rate limit exceeded",
            headers={"Retry-After": str(limiter.config.block_duration)}
        )


@app.middleware("http")
async def add_rate_limit_headers(request: Request, call_next):
    """Add rate limit headers to response."""
    # ... rate limit check ...
    response = await call_next(request)
    
    # Add headers
    response.headers["X-RateLimit-Limit"] = "100"
    response.headers["X-RateLimit-Remaining"] = "99"
    
    return response
```

### Request/Response Security

```python
"""
OWASP API Security Best Practices

- Validate content-type
- Limit request size
- Sanitize responses
- Implement proper CORS
"""

from fastapi import Request, Response
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
import re

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses."""
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # OWASP Recommendation: Security Headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()"
        
        # Remove server identification
        response.headers.pop("Server", None)
        response.headers.pop("X-Powered-By", None)
        
        return response


class RequestSizeMiddleware(BaseHTTPMiddleware):
    """Limit request body size."""
    
    def __init__(self, app, max_size: int = 1_048_576):  # 1MB
        super().__init__(app)
        self.max_size = max_size
    
    async def dispatch(self, request: Request, call_next):
        content_length = request.headers.get("content-length")
        
        if content_length and int(content_length) > self.max_size:
            return JSONResponse(
                status_code=413,
                content={"detail": "Request too large"}
            )
        
        return await call_next(request)


# Input validation middleware
class InputValidationMiddleware(BaseHTTPMiddleware):
    """Validate incoming request content."""
    
    ALLOWED_CONTENT_TYPES = {
        "application/json",
        "application/x-www-form-urlencoded",
        "multipart/form-data",
    }
    
    async def dispatch(self, request: Request, call_next):
        # Validate content-type
        content_type = request.headers.get("content-type", "").split(";")[0].strip()
        
        if content_type and content_type not in self.ALLOWED_CONTENT_TYPES:
            return JSONResponse(
                status_code=415,
                content={"detail": "Unsupported media type"}
            )
        
        # Validate accept header
        accept = request.headers.get("accept", "")
        if accept and not any(
            allowed in accept 
            for allowed in ("application/json", "*/*")
        ):
            return JSONResponse(
                status_code=406,
                content={"detail": "Not acceptable"}
            )
        
        return await call_next(request)
```

---

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [Secure by Design](https://owasp.org/www-project-secure-by-design/)
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [Python Security Guide](https://python-security.readthedocs.io/)
