# Autenticación JWT en FastAPI

Patrón estándar: password hashing con bcrypt, tokens JWT de corta duración (access token) + refresh token de larga duración.

---

## Instalación

```bash
uv add "python-jose[cryptography]" passlib[bcrypt] python-multipart
```

```toml
# pyproject.toml
[project]
dependencies = [
    "python-jose[cryptography]>=3.3.0",   # JWT
    "passlib[bcrypt]>=1.7.4",              # password hashing
    "python-multipart>=0.0.12",            # form data (para login form OAuth2)
]
```

---

## Configuración y constantes

```python
# config.py
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    secret_key: SecretStr          # SECRET_KEY en .env — mínimo 32 chars random
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7

settings = Settings()
```

---

## Password hashing

```python
# auth/password.py
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(plain_password: str) -> str:
    """Hashea la contraseña con bcrypt. Nunca almacenar texto plano."""
    return pwd_context.hash(plain_password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Compara de forma segura (resistente a timing attacks)."""
    return pwd_context.verify(plain_password, hashed_password)
```

---

## JWT — crear y verificar tokens

```python
# auth/jwt.py
from datetime import datetime, timedelta, timezone
from dataclasses import dataclass
from jose import JWTError, jwt
from pydantic import BaseModel

from ..config import settings

class TokenPayload(BaseModel):
    sub: str          # subject — user id como string
    exp: datetime
    type: str         # "access" o "refresh"

def create_access_token(user_id: int) -> str:
    """Token de corta duración para autenticar requests."""
    now = datetime.now(timezone.utc)
    expire = now + timedelta(minutes=settings.access_token_expire_minutes)
    payload = {
        "sub": str(user_id),
        "exp": expire,
        "type": "access",
        "iat": now,    # issued at
    }
    return jwt.encode(
        payload,
        settings.secret_key.get_secret_value(),
        algorithm=settings.algorithm
    )

def create_refresh_token(user_id: int) -> str:
    """Token de larga duración para obtener nuevos access tokens."""
    now = datetime.now(timezone.utc)
    expire = now + timedelta(days=settings.refresh_token_expire_days)
    payload = {
        "sub": str(user_id),
        "exp": expire,
        "type": "refresh",
        "iat": now,
    }
    return jwt.encode(
        payload,
        settings.secret_key.get_secret_value(),
        algorithm=settings.algorithm
    )

def decode_token(token: str) -> TokenPayload | None:
    """Verifica y decodifica un token. Retorna None si es inválido."""
    try:
        payload = jwt.decode(
            token,
            settings.secret_key.get_secret_value(),
            algorithms=[settings.algorithm]
        )
        return TokenPayload(**payload)
    except JWTError:
        return None
```

---

## Schemas de auth

```python
# api/schemas/auth.py
from pydantic import BaseModel, EmailStr

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class RefreshRequest(BaseModel):
    refresh_token: str
```

---

## Router de autenticación

```python
# api/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession

from ..deps import get_session
from ..schemas.auth import TokenResponse, LoginRequest, RefreshRequest
from ...repository.user_repo import UserRepository
from ...auth.password import verify_password
from ...auth.jwt import create_access_token, create_refresh_token, decode_token

router = APIRouter()

@router.post("/token", response_model=TokenResponse)
async def login(
    # OAuth2PasswordRequestForm espera form data: username + password
    form_data: OAuth2PasswordRequestForm = Depends(),
    session: AsyncSession = Depends(get_session),
):
    """Login estándar OAuth2 (Swagger UI lo soporta nativamente)."""
    repo = UserRepository(session)
    user = await repo.get_by_email(form_data.username)  # username = email

    if not user or not verify_password(form_data.password, user.password_hash):
        # CRÍTICO: mismo mensaje para "usuario no existe" y "contraseña incorrecta"
        # Evitar user enumeration
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas",
            headers={"WWW-Authenticate": "Bearer"},
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Cuenta desactivada",
        )

    return TokenResponse(
        access_token=create_access_token(user.id),
        refresh_token=create_refresh_token(user.id),
    )

@router.post("/login", response_model=TokenResponse)
async def login_json(
    credentials: LoginRequest,
    session: AsyncSession = Depends(get_session),
):
    """Login alternativo que acepta JSON (para SPA/apps móviles)."""
    repo = UserRepository(session)
    user = await repo.get_by_email(credentials.email)

    if not user or not verify_password(credentials.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Credenciales incorrectas")

    return TokenResponse(
        access_token=create_access_token(user.id),
        refresh_token=create_refresh_token(user.id),
    )

@router.post("/refresh", response_model=TokenResponse)
async def refresh_token(
    data: RefreshRequest,
    session: AsyncSession = Depends(get_session),
):
    """Obtener nuevo access token usando el refresh token."""
    payload = decode_token(data.refresh_token)

    if not payload or payload.type != "refresh":
        raise HTTPException(status_code=401, detail="Refresh token inválido")

    user_id = int(payload.sub)
    repo = UserRepository(session)
    user = await repo.get_by_id(user_id)

    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="Usuario no encontrado")

    return TokenResponse(
        access_token=create_access_token(user.id),
        refresh_token=create_refresh_token(user.id),  # rotate refresh token
    )
```

---

## Dependencias de autenticación

```python
# api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from .database import get_session
from ..domain.models import User
from ..auth.jwt import decode_token
from ..repository.user_repo import UserRepository

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
) -> User:
    payload = decode_token(token)

    if not payload or payload.type != "access":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="No autenticado",
            headers={"WWW-Authenticate": "Bearer"},
        )

    user = await UserRepository(session).get_by_id(int(payload.sub))
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="Usuario no encontrado")

    return user

async def require_admin(current_user: User = Depends(get_current_user)) -> User:
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Se requieren permisos de administrador",
        )
    return current_user

# Usar en los routers:
# current_user: User = Depends(get_current_user)   ← requiere auth
# admin: User = Depends(require_admin)              ← requiere admin
```

---

## Registro de usuario

```python
# api/routers/users.py
from ..schemas.users import UserCreate, UserResponse
from ...auth.password import hash_password

@router.post("/", response_model=UserResponse, status_code=201)
async def register(
    data: UserCreate,
    session: AsyncSession = Depends(get_session),
):
    repo = UserRepository(session)

    if await repo.email_exists(data.email):
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="Ya existe una cuenta con ese email",
        )

    # Hashear ANTES de almacenar
    user = await repo.create(
        email=data.email,
        username=data.username,
        password_hash=hash_password(data.password),  # nunca almacenar plain
    )

    return user
```

---

## Consideraciones de seguridad

```python
# .env — nunca commitear
SECRET_KEY=tu_clave_super_secreta_de_al_menos_32_caracteres_random
# Generar con: python -c "import secrets; print(secrets.token_hex(32))"

# HTTPS — siempre en producción
# Los JWT viajan en el header Authorization — si es HTTP pueden ser interceptados

# Tokens cortos + refresh:
# - access_token: 15-30 minutos
# - refresh_token: 7-30 días
# - Al logout, invalidar el refresh token en DB (blacklist o tabla de tokens)

# Rate limiting en /auth/token — prevenir brute force
# pip install slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/token")
@limiter.limit("5/minute")  # máx 5 intentos por minuto por IP
async def login(request: Request, ...):
    ...
```

---

## Flujo completo

```
1. Usuario: POST /auth/login  { email, password }
           ↓
2. Server: verifica email existe en DB
           verifica bcrypt.verify(password, hash)
           crea access_token (JWT, 30min) + refresh_token (JWT, 7d)
           ↓
3. Cliente: guarda tokens (memory/httpOnly cookie — NO localStorage)
           ↓
4. Request: GET /products
            Authorization: Bearer <access_token>
           ↓
5. Server: decode JWT, verifica firma, verifica exp
           carga usuario de DB, ejecuta handler
           ↓
6. access_token expirado:
           POST /auth/refresh  { refresh_token }
           → nuevo access_token + nuevo refresh_token (rotation)
```
