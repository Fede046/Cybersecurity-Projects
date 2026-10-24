# Per iniziare: costruisci la tua piattaforma Bug Bounty

Questa guida ti accompagna nella costruzione di una piattaforma simile da zero, spiegando ogni passo e il ragionamento dietro le decisioni di design.

---

## Indice

1. [Prerequisiti](#prerequisiti)
2. [Setup del progetto](#setup-del-progetto)
3. [Sviluppo backend](#sviluppo-backend)
4. [Sviluppo frontend](#sviluppo-frontend)
5. [Progettazione del database](#progettazione-del-database)
6. [Autenticazione](#autenticazione)
7. [Sviluppo delle API](#sviluppo-delle-api)
8. [Testing](#testing)
9. [Deployment](#deployment)

---

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Python:**
- Async/await (`async def`, `await`)
- Type hint (`def func(x: int) -> str`)
- Context manager (`async with`)
- Decorator (`@router.get()`)

**TypeScript/JavaScript:**
- Promise e async/await
- Hook di React (`useState`, `useEffect`)
- Tipi e interfacce TypeScript
- Sintassi ES6+ (arrow function, destructuring)

**Database:**
- Basi di SQL (SELECT, INSERT, UPDATE, DELETE)
- Relazioni (one-to-many, many-to-many)
- Indici e foreign key

**Strumenti:**
- Git (controllo di versione)
- Docker (containerizzazione)
- Basi della riga di comando

---

## Setup del progetto

### Passo 1: crea la struttura del progetto

```bash
mkdir bug-bounty-platform
cd bug-bounty-platform

mkdir -p backend/src/app
mkdir -p frontend/src
mkdir -p infra/nginx
mkdir -p infra/docker
```

### Passo 2: inizializza il backend (Python)

```bash
cd backend

touch pyproject.toml
```

**pyproject.toml:**

```toml
[project]
name = "bug-bounty-backend"
version = "1.0.0"
requires-python = ">=3.12"

dependencies = [
    "fastapi>=0.123.0",
    "uvicorn[standard]>=0.40.0",
    "sqlalchemy>=2.0.0",
    "asyncpg>=0.30.0",
    "alembic>=1.15.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.7.0",
    "pyjwt>=2.10.0",
    "pwdlib[argon2]>=0.2.0",
    "uuid-utils>=0.10.0",
    "python-multipart>=0.0.20",
]

[project.optional-dependencies]
dev = [
    "ruff>=0.9.0",
    "mypy>=1.15.0",
    "pytest>=8.0.0",
    "pytest-asyncio>=0.25.0",
]

[build-system]
requires = ["setuptools>=75.0.0"]
build-backend = "setuptools.build_meta"
```

**Perché queste dipendenze?**
- `fastapi` - Framework web async moderno
- `sqlalchemy` - ORM per le operazioni sul database
- `asyncpg` - Driver PostgreSQL async veloce
- `alembic` - Migrazioni del database
- `pydantic` - Validazione dei dati
- `pyjwt` - Gestione dei token JWT
- `pwdlib[argon2]` - Hashing sicuro delle password
- `uuid-utils` - Supporto agli UUID v7

### Passo 3: inizializza il frontend (React + TypeScript)

```bash
cd ../frontend

pnpm create vite@latest . --template react-ts
pnpm install
```

**Installa le dipendenze aggiuntive:**

```bash
pnpm add react-router-dom @tanstack/react-query zustand axios zod
pnpm add -D sass stylelint @biomejs/biome
```

**Perché queste dipendenze?**
- `react-router-dom` - Routing basato sui file
- `@tanstack/react-query` - Gestione dello stato del server
- `zustand` - Gestione dello stato del client
- `axios` - Client HTTP
- `zod` - Validazione a runtime
- `sass` - Preprocessing CSS
- `biome` - Linting e formattazione

---

## Sviluppo backend

### Passo 1: crea la configurazione di base

**backend/src/config.py:**

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
    )

    APP_NAME: str = "Bug Bounty Platform"
    ENVIRONMENT: str = "development"
    DEBUG: bool = True

    DATABASE_URL: str
    SECRET_KEY: str

    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

settings = Settings()
```

**Perché Pydantic Settings?**
- Validazione dei tipi (DATABASE_URL deve essere una stringa)
- Caricamento delle variabili d'ambiente (legge da .env)
- Valori predefiniti
- Autocompletamento nell'IDE

### Passo 2: configura il database

**backend/src/app/core/database.py:**

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
)
from typing import AsyncGenerator

class DatabaseSessionManager:
    def __init__(self):
        self._engine = None
        self._sessionmaker = None

    def init(self, database_url: str):
        self._engine = create_async_engine(
            database_url,
            echo=True,  # Log SQL queries (disable in production)
            pool_size=20,
            max_overflow=10,
        )
        self._sessionmaker = async_sessionmaker(
            self._engine,
            class_=AsyncSession,
            expire_on_commit=False,
        )

    async def session(self) -> AsyncGenerator[AsyncSession, None]:
        if self._sessionmaker is None:
            raise RuntimeError("Database not initialized")

        async with self._sessionmaker() as session:
            try:
                yield session
            except Exception:
                await session.rollback()
                raise
            finally:
                await session.close()

db_manager = DatabaseSessionManager()
```

**Concetti chiave:**
- `AsyncSession` - Sessione di database async
- `async_sessionmaker` - Factory per creare sessioni
- Context manager (`async with`) - Esegue automaticamente commit/rollback
- Connection pooling - Riutilizzo delle connessioni

### Passo 3: crea i modelli di base

**backend/src/app/core/Base.py:**

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import DateTime, func
from uuid import UUID
from datetime import datetime
import uuid_utils as uuid

class Base(DeclarativeBase):
    pass

class UUIDMixin:
    id: Mapped[UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid7,
    )

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )
```

**Perché i mixin?**
- DRY (Don't Repeat Yourself)
- Ogni modello ottiene automaticamente `id`, `created_at`, `updated_at`
- Si possono aggiungere altri mixin (SoftDeleteMixin, ecc.)

### Passo 4: crea il modello User

**backend/src/app/user/models.py:**

```python
from sqlalchemy import String, Enum
from sqlalchemy.orm import Mapped, mapped_column
from app.core.Base import Base, UUIDMixin, TimestampMixin
from enum import StrEnum

class UserRole(StrEnum):
    USER = "user"
    COMPANY = "company"
    ADMIN = "admin"

class User(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "users"

    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        index=True,
    )
    password_hash: Mapped[str] = mapped_column(String(255))
    full_name: Mapped[str] = mapped_column(String(255))
    role: Mapped[UserRole] = mapped_column(
        Enum(UserRole, values_callable=lambda obj: [e.value for e in obj]),
        default=UserRole.USER,
    )
```

**Punti chiave:**
- `Mapped[str]` - Type hint per SQLAlchemy
- `unique=True, index=True` - Lookup veloci, prevenzione dei duplicati
- `Enum` con values_callable - Memorizza il valore dell'enum, non il nome (pattern SafeEnum)

### Passo 5: crea il repository User

**backend/src/app/user/repository.py:**

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.user.models import User
from uuid import UUID

class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: UUID) -> User | None:
        stmt = select(User).where(User.id == user_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(
        self,
        email: str,
        password_hash: str,
        full_name: str,
        role: str = "user",
    ) -> User:
        user = User(
            email=email,
            password_hash=password_hash,
            full_name=full_name,
            role=role,
        )
        self.session.add(user)
        await self.session.flush()
        await self.session.refresh(user)
        return user
```

**Perché i repository?**
- Separano l'accesso ai dati dalla logica di business
- Query riutilizzabili
- Facili da testare (mock del repository)

### Passo 6: crea gli schema User

**backend/src/app/user/schemas.py:**

```python
from pydantic import BaseModel, EmailStr, Field
from uuid import UUID
from datetime import datetime

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8)
    full_name: str = Field(..., min_length=1, max_length=255)

class UserResponse(BaseModel):
    id: UUID
    email: str
    full_name: str
    role: str
    created_at: datetime

    model_config = {"from_attributes": True}
```

**Perché gli schema Pydantic?**
- Validazione dell'input (formato email, lunghezza password)
- Documentazione OpenAPI automatica
- Type safety
- Serializzazione (modello ORM → JSON)

### Passo 7: crea le route User

**backend/src/app/user/routes.py:**

```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import db_manager
from app.user.schemas import UserResponse
from app.user.repository import UserRepository
from app.core.dependencies import get_current_user
from typing import Annotated

router = APIRouter()

DatabaseSession = Annotated[
    AsyncSession,
    Depends(db_manager.session)
]
CurrentUser = Annotated[User, Depends(get_current_user)]

@router.get("/me", response_model=UserResponse)
async def get_me(user: CurrentUser) -> UserResponse:
    return UserResponse.from_orm(user)
```

**Concetti chiave:**
- `Annotated` - Alias di tipo con dipendenza
- `Depends()` - Dependency injection
- `response_model` - Serializzazione automatica
- Il route handler restituisce un oggetto Python, FastAPI lo converte in JSON

---

## Autenticazione

### Passo 1: hashing delle password

**backend/src/app/core/security.py:**

```python
from pwdlib import PasswordHash
from pwdlib.hashers.argon2 import Argon2Hasher

password_hash = PasswordHash((Argon2Hasher(),))

def hash_password(password: str) -> str:
    return password_hash.hash(password)

def verify_password(password: str, hash: str) -> bool:
    try:
        return password_hash.verify(password, hash)
    except Exception:
        return False
```

**Perché Argon2?**
- Vincitore della Password Hashing Competition (2015)
- Resistente al cracking su GPU
- Memory-hard (costoso da parallelizzare)

### Passo 2: creazione del token JWT

**backend/src/app/core/security.py (continua):**

```python
import jwt
from datetime import datetime, timedelta
from uuid import UUID
from config import settings

def create_access_token(user: User) -> str:
    payload = {
        "sub": str(user.id),
        "role": user.role,
        "token_version": user.token_version,
        "exp": datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        ),
        "iat": datetime.utcnow(),
    }
    return jwt.encode(
        payload,
        settings.SECRET_KEY,
        algorithm=settings.JWT_ALGORITHM,
    )

def decode_access_token(token: str) -> dict:
    try:
        return jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.JWT_ALGORITHM],
        )
    except jwt.ExpiredSignatureError:
        raise UnauthorizedError("Token expired")
    except jwt.InvalidTokenError:
        raise UnauthorizedError("Invalid token")
```

**Payload JWT:**
- `sub` (subject) - ID dell'utente
- `role` - Ruolo dell'utente (per l'autorizzazione)
- `token_version` - Per l'invalidazione istantanea
- `exp` (expiration) - Il token scade dopo 15 minuti
- `iat` (issued at) - Timestamp di creazione

### Passo 3: dipendenza di autenticazione

**backend/src/app/core/dependencies.py:**

```python
from fastapi import Header, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.security import decode_access_token
from app.user.repository import UserRepository
from app.user.models import User
from uuid import UUID

async def get_current_user(
    authorization: str = Header(...),
    session: AsyncSession = Depends(db_manager.session),
) -> User:
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid token")

    token = authorization.replace("Bearer ", "")

    try:
        payload = decode_access_token(token)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

    user_id = UUID(payload["sub"])
    repo = UserRepository(session)
    user = await repo.get_by_id(user_id)

    if not user:
        raise HTTPException(status_code=401, detail="User not found")

    if user.token_version != payload["token_version"]:
        raise HTTPException(status_code=401, detail="Token invalidated")

    return user
```

**Come funziona:**
1. Estrae l'header `Authorization: Bearer <token>`
2. Decodifica il token JWT
3. Ricava l'ID utente dal payload del token
4. Interroga il database per l'utente
5. Verifica che la versione del token corrisponda (per l'invalidazione istantanea)
6. Restituisce l'utente autenticato

---

## Sviluppo frontend

### Passo 1: crea il client API

**frontend/src/api/index.ts:**

```typescript
import axios from 'axios';

const api = axios.create({
  baseURL: '/api',
  headers: {
    'Content-Type': 'application/json',
  },
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      const refreshToken = localStorage.getItem('refresh_token');
      if (refreshToken) {
        try {
          const { data } = await axios.post('/api/auth/refresh', {
            refresh_token: refreshToken,
          });
          localStorage.setItem('access_token', data.access_token);
          localStorage.setItem('refresh_token', data.refresh_token);

          error.config.headers.Authorization = `Bearer ${data.access_token}`;
          return axios(error.config);
        } catch {
          localStorage.removeItem('access_token');
          localStorage.removeItem('refresh_token');
          window.location.href = '/login';
        }
      }
    }
    return Promise.reject(error);
  }
);

export default api;
```

**Cosa fa:**
- Aggiunge automaticamente l'header `Authorization` a tutte le richieste
- Rinnova automaticamente gli access token scaduti
- Reindirizza al login se il refresh fallisce

### Passo 2: crea lo store di autenticazione (Zustand)

**frontend/src/stores/auth.store.ts:**

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AuthStore {
  accessToken: string | null;
  refreshToken: string | null;
  user: User | null;

  setTokens: (access: string, refresh: string) => void;
  setUser: (user: User) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      accessToken: null,
      refreshToken: null,
      user: null,

      setTokens: (access, refresh) => set({
        accessToken: access,
        refreshToken: refresh,
      }),

      setUser: (user) => set({ user }),

      logout: () => set({
        accessToken: null,
        refreshToken: null,
        user: null,
      }),
    }),
    {
      name: 'auth-storage',
    }
  )
);
```

**Perché Zustand?**
- API semplice (nessun boilerplate)
- Persistenza integrata (salva su localStorage)
- Supporto TypeScript
- Nessun provider wrapper necessario

### Passo 3: crea un hook React Query

**frontend/src/api/hooks/useAuth.ts:**

```typescript
import { useMutation } from '@tanstack/react-query';
import api from '../index';
import { useAuthStore } from '../../stores/auth.store';

interface LoginRequest {
  email: string;
  password: string;
}

interface LoginResponse {
  access_token: string;
  refresh_token: string;
}

export const useLogin = () => {
  const setTokens = useAuthStore((state) => state.setTokens);

  return useMutation({
    mutationFn: async (data: LoginRequest) => {
      const response = await api.post<LoginResponse>('/auth/login', data);
      return response.data;
    },
    onSuccess: (data) => {
      setTokens(data.access_token, data.refresh_token);
      localStorage.setItem('access_token', data.access_token);
      localStorage.setItem('refresh_token', data.refresh_token);
    },
  });
};
```

**Utilizzo in un componente:**

```typescript
function LoginForm() {
  const login = useLogin();

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    login.mutate({
      email: 'user@example.com',
      password: 'password123',
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* ... */}
      <button disabled={login.isPending}>
        {login.isPending ? 'Logging in...' : 'Login'}
      </button>
      {login.isError && <p>Login failed</p>}
    </form>
  );
}
```

**Perché TanStack Query?**
- Stati di caricamento/errore automatici (`isPending`, `isError`)
- Aggiornamenti ottimistici
- Logica di retry
- Gestione della cache

---

## Testing

### Testing del backend

**backend/tests/test_user.py:**

```python
import pytest
from httpx import AsyncClient
from app.factory import create_app

@pytest.fixture
async def client():
    app = create_app()
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_register_user(client: AsyncClient):
    response = await client.post("/api/auth/register", json={
        "email": "test@example.com",
        "password": "password123",
        "full_name": "Test User",
    })

    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "id" in data
```

**Esegui i test:**

```bash
pytest backend/tests/
```

### Testing del frontend

**frontend/src/components/LoginForm.test.tsx:**

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { LoginForm } from './LoginForm';

test('renders login form', () => {
  render(<LoginForm />);

  expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /login/i })).toBeInTheDocument();
});

test('submits form with credentials', () => {
  const onSubmit = jest.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  fireEvent.change(screen.getByLabelText(/email/i), {
    target: { value: 'test@example.com' },
  });
  fireEvent.change(screen.getByLabelText(/password/i), {
    target: { value: 'password123' },
  });
  fireEvent.click(screen.getByRole('button', { name: /login/i }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123',
  });
});
```

---

## Deployment

### Passo 1: crea il Docker Compose

**compose.yml:**

```yaml
version: '3.8'

services:
  nginx:
    build: ./infra/nginx
    ports:
      - "8420:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db

  db:
    image: postgres:18-alpine
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Passo 2: esegui il deploy

```bash
cp .env.example .env
just up
```

---

## Prossimi passi

Ora che hai capito le basi:

1. **Leggi la documentazione sull'architettura:** [ARCHITECTURE.it.md](./ARCHITECTURE.it.md)
2. **Impara i design pattern:** [PATTERNS.it.md](./PATTERNS.it.md)
3. **Comprendi il database:** [DATABASE.it.md](./DATABASE.it.md)
4. **Studia le funzionalità di sicurezza:** [SECURITY.it.md](./SECURITY.it.md)

---

## Errori comuni

**1. Non usare correttamente async/await:**

```python
async def bad():
    user_repo.get_by_id(user_id)  # Missing await!

async def good():
    user = await user_repo.get_by_id(user_id)
```

**2. Mescolare codice sync e async:**

```python
async def bad():
    users = session.query(User).all()  # Sync query in async function

async def good():
    stmt = select(User)
    result = await session.execute(stmt)
    users = result.scalars().all()
```

**3. Non gestire gli errori:**

```python
def bad():
    user = await repo.get_by_id(user_id)
    return user.email  # What if user is None?

def good():
    user = await repo.get_by_id(user_id)
    if not user:
        raise UserNotFoundError()
    return user.email
```

**4. Memorizzare le password in chiaro:**

```python
def bad():
    user = User(email=email, password=password)  # Plaintext password!

def good():
    user = User(email=email, password_hash=hash_password(password))
```

---

## Risorse

**Documentazione ufficiale:**
- [FastAPI](https://fastapi.tiangolo.com/)
- [SQLAlchemy](https://docs.sqlalchemy.org/)
- [React](https://react.dev/)
- [TanStack Query](https://tanstack.com/query/latest)

**Libri:**
- "Clean Architecture" by Robert C. Martin
- "Domain-Driven Design" by Eric Evans
- "Designing Data-Intensive Applications" by Martin Kleppmann

**Questo codebase:**
- Esplora il codice in `/backend/src/app/`
- Leggi gli altri documenti di apprendimento in `/learn/`
- Sperimenta modificando il codice

---

Buona fortuna nella costruzione della tua piattaforma!
