# Design Pattern Spiegati

Questo documento spiega i principali design pattern usati nella Bug Bounty Platform, perché sono stati scelti e come migliorano la qualità del codice.

---

## Indice

1. [Dependency Injection](#dependency-injection)
2. [Repository Pattern](#repository-pattern)
3. [Layered Architecture](#layered-architecture)
4. [Factory Pattern](#factory-pattern)
5. [Strategy Pattern](#strategy-pattern)
6. [Singleton Pattern](#singleton-pattern)
7. [Mixin Pattern](#mixin-pattern)
8. [Observer Pattern (Pub/Sub)](#observer-pattern-pubsub)

---

## Dependency Injection

**Che cos'è:** Invece di creare le dipendenze all'interno di una classe, vengono "iniettate" dall'esterno.

**Senza Dependency Injection:**

```python
class UserService:
    def __init__(self):
        self.db = Database()  # Hardcoded dependency
        self.cache = Redis()  # Hardcoded dependency

    def get_user(self, user_id: str):
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            return cached
        user = self.db.query(f"SELECT * FROM users WHERE id = '{user_id}'")
        self.cache.set(f"user:{user_id}", user)
        return user
```

Problemi:
- Difficile da testare (non si possono fare mock di Database o Redis)
- Accoppiamento stretto (UserService dipende da implementazioni specifiche)
- Impossibile sostituire le implementazioni (ad es. usare una cache diversa)

**Con Dependency Injection:**

```python
class UserService:
    def __init__(self, db: Database, cache: Cache):
        self.db = db      # Injected from outside
        self.cache = cache # Injected from outside

    def get_user(self, user_id: str):
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            return cached
        user = self.db.query(f"SELECT * FROM users WHERE id = '{user_id}'")
        self.cache.set(f"user:{user_id}", user)
        return user

# Usage
db = Database()
cache = Redis()
user_service = UserService(db=db, cache=cache)
```

Vantaggi:
- Facile da testare (si iniettano i mock)
- Accoppiamento debole (dipende da interfacce, non da implementazioni)
- Flessibile (si possono iniettare implementazioni diverse)

### Dependency Injection in FastAPI

FastAPI ha una DI integrata che usa `Depends()`:

```python
from fastapi import Depends
from typing import Annotated

# Dependency function
async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with DatabaseSessionManager().session() as session:
        yield session

# Type alias for convenience
DatabaseSession = Annotated[AsyncSession, Depends(get_db_session)]

# Route using dependency
@router.get("/users/{user_id}")
async def get_user(
    user_id: UUID,
    session: DatabaseSession,  # ← Automatically injected
) -> UserResponse:
    repo = UserRepository(session)
    user = await repo.get_by_id(user_id)
    return user
```

**Come funziona:**
1. FastAPI vede il parametro `session: DatabaseSession`
2. Riconosce che `DatabaseSession` è un tipo `Annotated` con `Depends()`
3. Chiama `get_db_session()` per ottenere una sessione
4. Inietta la sessione nel route handler
5. Esegue automaticamente la pulizia quando la richiesta termina

**Esempio reale dal codebase:**

```python
# backend/app/core/dependencies.py

async def get_current_user(
    token: str = Header(...),
    session: DatabaseSession,
) -> User:
    payload = decode_access_token(token)
    user_id = payload["sub"]
    repo = UserRepository(session)
    user = await repo.get_by_id(user_id)
    if not user:
        raise UnauthorizedError()
    return user

# Type alias
CurrentUser = Annotated[User, Depends(get_current_user)]

# Usage in routes
@router.get("/me")
async def get_me(user: CurrentUser) -> UserResponse:
    return user  # User is automatically injected and authenticated!
```

**Perché è potente:**
- I route handler non devono preoccuparsi dell'autenticazione
- Principio DRY (non ripetere la logica di autenticazione in ogni route)
- Facile da testare (mock di `get_current_user`)
- Componibile (le dipendenze possono dipendere da altre dipendenze)

---

## Repository Pattern

**Che cos'è:** Un livello di astrazione tra la business logic e l'archiviazione dei dati.

**Il problema:**

```python
# Business logic mixed with database queries
@router.post("/users")
async def create_user(data: UserCreate, session: DatabaseSession):
    # Database query in route handler
    stmt = select(User).where(User.email == data.email)
    result = await session.execute(stmt)
    existing_user = result.scalar_one_or_none()

    if existing_user:
        raise HTTPException(status_code=400, detail="Email already exists")

    user = User(
        email=data.email,
        password_hash=hash_password(data.password),
    )
    session.add(user)
    await session.commit()
    return user
```

Problemi:
- Il route handler conosce lo schema del database
- Difficile da testare (serve configurare un database)
- Non si può cambiare database facilmente
- Logica delle query duplicata in più route

**La soluzione del Repository Pattern:**

```python
# 1. Create a repository class
class UserRepository(BaseRepository[User]):
    async def find_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, data: UserCreate) -> User:
        user = User(
            email=data.email,
            password_hash=hash_password(data.password),
        )
        self.session.add(user)
        await self.session.flush()
        await self.session.refresh(user)
        return user

# 2. Use repository in route handler
@router.post("/users")
async def create_user(data: UserCreate, session: DatabaseSession):
    repo = UserRepository(session)

    existing_user = await repo.find_by_email(data.email)
    if existing_user:
        raise HTTPException(status_code=400, detail="Email already exists")

    user = await repo.create(data)
    await session.commit()
    return user
```

Vantaggi:
- Il route handler non conosce lo schema del database
- La logica delle query è riutilizzabile
- Facile da testare (mock del repository)
- Si può sostituire l'implementazione del database

### Base Repository generico

Per evitare di ripetere le operazioni CRUD, usiamo un **base repository generico**:

```python
from typing import Generic, TypeVar

ModelT = TypeVar("ModelT", bound=Base)

class BaseRepository(Generic[ModelT]):
    def __init__(self, session: AsyncSession, model: type[ModelT]):
        self.session = session
        self.model = model

    async def get_by_id(self, id: UUID) -> ModelT | None:
        stmt = select(self.model).where(self.model.id == id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_all(self, limit: int = 100, offset: int = 0) -> list[ModelT]:
        stmt = select(self.model).limit(limit).offset(offset)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def create(self, **kwargs) -> ModelT:
        instance = self.model(**kwargs)
        self.session.add(instance)
        await self.session.flush()
        await self.session.refresh(instance)
        return instance

    async def delete(self, id: UUID) -> None:
        stmt = delete(self.model).where(self.model.id == id)
        await self.session.execute(stmt)
```

**Utilizzo:**

```python
# Specific repository extends base
class UserRepository(BaseRepository[User]):
    def __init__(self, session: AsyncSession):
        super().__init__(session, User)

    # Add domain-specific methods
    async def find_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

# Usage
repo = UserRepository(session)
user = await repo.get_by_id(user_id)      # From BaseRepository
user = await repo.find_by_email(email)    # From UserRepository
```

**Perché i generics?**
- Type safety: `repo.get_by_id()` restituisce `User`, non `Any`
- Riutilizzabile: tutti i modelli ottengono il CRUD gratis
- DRY: le operazioni comuni si scrivono una sola volta

---

## Layered Architecture

**Che cos'è:** Organizzare il codice in livelli (layer), dove ogni livello ha una responsabilità specifica.

```
┌─────────────────────────────────────┐
│         Presentation Layer          │  ← Routes (HTTP)
│  - Handle HTTP requests/responses   │
│  - Input validation                 │
│  - Serialize data to JSON           │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│          Business Layer             │  ← Services
│  - Core business logic              │
│  - Orchestrate repositories         │
│  - Transaction management           │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│         Data Access Layer           │  ← Repositories
│  - Database queries                 │
│  - CRUD operations                  │
│  - No business logic                │
└───────────────┬─────────────────────┘
                │
┌───────────────▼─────────────────────┐
│           Database                  │  ← PostgreSQL
└─────────────────────────────────────┘
```

**Regole:**
1. **Dipendenza dall'alto verso il basso:** I livelli superiori dipendono da quelli inferiori, mai il contrario
2. **Responsabilità singola:** Ogni livello ha un solo compito
3. **Nessun salto di livello:** Le route non possono interrogare direttamente il database

**Esempio: creare un report**

```python
# 1. Presentation Layer (routes.py)
@router.post("/reports")
async def create_report(
    data: ReportCreate,
    user: CurrentUser,
    session: DatabaseSession,
) -> ReportResponse:
    report = await submit_vulnerability_report(
        session=session,
        author_id=user.id,
        data=data,
    )
    return ReportResponse.from_orm(report)
```

La route:
- Valida l'input (modello Pydantic `ReportCreate`)
- Autentica l'utente (dipendenza `CurrentUser`)
- Chiama la funzione di service
- Serializza l'output (`ReportResponse`)

```python
# 2. Business Layer (service.py)
async def submit_vulnerability_report(
    session: AsyncSession,
    author_id: UUID,
    data: ReportCreate,
) -> Report:
    # Validate program exists
    program_repo = ProgramRepository(session)
    program = await program_repo.get_by_slug(data.program_slug)
    if not program:
        raise ProgramNotFoundError()

    # Check if program accepts reports
    if program.status != ProgramStatus.ACTIVE:
        raise ProgramNotActiveError()

    # Create report
    report_repo = ReportRepository(session)
    report = await report_repo.create(
        author_id=author_id,
        program_id=program.id,
        title=data.title,
        description=data.description,
        severity=data.severity,
    )

    # Send notification (in real app)
    # await notify_program_owner(program.owner_id, report.id)

    await session.commit()
    return report
```

Il service:
- Implementa le regole di business (il programma deve essere attivo)
- Coordina più repository (Program + Report)
- Gestisce le transazioni (commit)
- Solleva eccezioni di dominio

```python
# 3. Data Access Layer (repository.py)
class ReportRepository(BaseRepository[Report]):
    async def create(
        self,
        author_id: UUID,
        program_id: UUID,
        title: str,
        description: str,
        severity: Severity,
    ) -> Report:
        report = Report(
            author_id=author_id,
            program_id=program_id,
            title=title,
            description=description,
            severity=severity,
            status=ReportStatus.NEW,
        )
        self.session.add(report)
        await self.session.flush()
        await self.session.refresh(report)
        return report
```

Il repository:
- Costruisce il modello Report
- Lo aggiunge alla sessione
- Restituisce l'istanza
- Nessuna business logic

**Vantaggi:**
- **Testabilità:** Si fa il mock di ogni livello in modo indipendente
- **Manutenibilità:** Si modifica un livello senza influenzare gli altri
- **Scalabilità:** Si possono estrarre i livelli in servizi separati se necessario
- **Chiarezza:** È facile trovare dove si trova la logica

---

## Factory Pattern

**Che cos'è:** Una funzione/classe che crea oggetti senza specificarne la classe esatta.

**Esempio reale dal codebase:**

```python
# backend/app/factory.py

def create_app() -> FastAPI:
    app = FastAPI(
        title="Bug Bounty Platform",
        version="1.0.0",
        docs_url="/api/docs",
        redoc_url="/api/redoc",
    )

    # Add middleware
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # Add rate limiting
    app.state.limiter = limiter
    app.add_exception_handler(RateLimitExceeded, rate_limit_handler)

    # Register routers
    app.include_router(auth_router, prefix="/api/v1/auth", tags=["auth"])
    app.include_router(user_router, prefix="/api/v1/users", tags=["users"])
    app.include_router(program_router, prefix="/api/v1/programs", tags=["programs"])
    app.include_router(report_router, prefix="/api/v1/reports", tags=["reports"])
    app.include_router(admin_router, prefix="/api/v1/admin", tags=["admin"])

    return app

# Usage
app = create_app()
```

**Vantaggi:**
- Configurazione centralizzata
- Facile creare più istanze dell'app (testing, ambienti diversi)
- Rispetta il Single Responsibility Principle

**Un altro esempio: la session factory del database**

```python
class DatabaseSessionManager:
    def __init__(self):
        self._engine: AsyncEngine | None = None
        self._sessionmaker: async_sessionmaker | None = None

    def init(self, database_url: str):
        self._engine = create_async_engine(
            database_url,
            pool_size=settings.DB_POOL_SIZE,
            max_overflow=settings.DB_MAX_OVERFLOW,
            pool_timeout=settings.DB_POOL_TIMEOUT,
            pool_recycle=settings.DB_POOL_RECYCLE,
        )
        self._sessionmaker = async_sessionmaker(
            self._engine,
            expire_on_commit=False,
        )

    async def session(self) -> AsyncGenerator[AsyncSession, None]:
        if self._sessionmaker is None:
            raise DatabaseNotInitializedError()

        async with self._sessionmaker() as session:
            try:
                yield session
            except Exception:
                await session.rollback()
                raise
            finally:
                await session.close()

# Usage
db_manager = DatabaseSessionManager()
db_manager.init(settings.DATABASE_URL)

async with db_manager.session() as session:
    # Use session
    pass
```

---

## Strategy Pattern

**Che cos'è:** Definire una famiglia di algoritmi, incapsulare ciascuno di essi e renderli intercambiabili.

**Esempio: hashing delle password**

```python
from abc import ABC, abstractmethod

# Strategy interface
class PasswordHasher(ABC):
    @abstractmethod
    def hash(self, password: str) -> str:
        pass

    @abstractmethod
    def verify(self, password: str, hash: str) -> bool:
        pass

# Concrete strategies
class Argon2Hasher(PasswordHasher):
    def hash(self, password: str) -> str:
        return argon2.hash(password)

    def verify(self, password: str, hash: str) -> bool:
        try:
            return argon2.verify(hash, password)
        except Exception:
            return False

class BcryptHasher(PasswordHasher):
    def hash(self, password: str) -> str:
        return bcrypt.hashpw(password.encode(), bcrypt.gensalt())

    def verify(self, password: str, hash: str) -> bool:
        return bcrypt.checkpw(password.encode(), hash.encode())

# Context
class PasswordService:
    def __init__(self, hasher: PasswordHasher):
        self.hasher = hasher

    def hash_password(self, password: str) -> str:
        return self.hasher.hash(password)

    def verify_password(self, password: str, hash: str) -> bool:
        return self.hasher.verify(password, hash)

# Usage
service = PasswordService(Argon2Hasher())  # Can swap to BcryptHasher()
hash = service.hash_password("secret123")
is_valid = service.verify_password("secret123", hash)
```

**Vantaggi:**
- Facile aggiungere nuovi algoritmi (ad es. PBKDF2)
- Si può cambiare algoritmo senza modificare il codice
- Ogni strategy è testabile in modo indipendente

---

## Singleton Pattern

**Che cos'è:** Garantire che una classe abbia una sola istanza e fornire un punto di accesso globale.

**Esempio: il session manager del database**

```python
class DatabaseSessionManager:
    _instance: "DatabaseSessionManager | None" = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._engine = None
            cls._instance._sessionmaker = None
        return cls._instance

    def init(self, database_url: str):
        if self._engine is not None:
            return  # Already initialized

        self._engine = create_async_engine(database_url)
        self._sessionmaker = async_sessionmaker(self._engine)

    async def session(self):
        async with self._sessionmaker() as session:
            yield session

# Usage - always returns the same instance
db1 = DatabaseSessionManager()
db2 = DatabaseSessionManager()
assert db1 is db2  # True
```

**Quando usarlo:**
- Connessioni al database (costose da creare)
- Logger (un'unica configurazione di logging)
- Configurazione (si carica una volta, si usa ovunque)

**Quando NON usarlo:**
- Nella maggior parte degli altri casi (i singleton sono stato globale = difficili da testare)

---

## Mixin Pattern

**Che cos'è:** Una classe che fornisce metodi ad altre classi ma non è pensata per essere istanziata di per sé.

**Esempio reale dal codebase:**

```python
# backend/app/core/Base.py

class UUIDMixin:
    id: Mapped[UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid7,
    )

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=func.now(),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=func.now(),
        onupdate=func.now(),
    )

class SoftDeleteMixin:
    deleted_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
        default=None,
    )

# Usage: Compose mixins
class User(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "users"

    email: Mapped[str] = mapped_column(String(255))
    password_hash: Mapped[str] = mapped_column(String(255))

class Report(Base, UUIDMixin, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "reports"

    title: Mapped[str] = mapped_column(String(255))
    description: Mapped[str] = mapped_column(Text)
```

**Vantaggi:**
- DRY: non si ripetono `id`, `created_at`, `updated_at` in ogni modello
- Componibile: si combinano a piacere (User non ha bisogno di SoftDeleteMixin)
- Manutenibile: si cambia `UUIDMixin` → tutti i modelli vengono aggiornati

---

## Observer Pattern (Pub/Sub)

**Che cos'è:** Quando un oggetto cambia stato, tutti i suoi dipendenti vengono notificati automaticamente.

**Esempio: un sistema di eventi**

```python
from typing import Callable

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}

    def subscribe(self, event: str, handler: Callable):
        if event not in self._subscribers:
            self._subscribers[event] = []
        self._subscribers[event].append(handler)

    async def publish(self, event: str, data: dict):
        if event not in self._subscribers:
            return

        for handler in self._subscribers[event]:
            await handler(data)

# Event handlers
async def send_email_notification(data: dict):
    print(f"Sending email to {data['email']}")

async def log_event(data: dict):
    print(f"Logging event: {data}")

# Setup
bus = EventBus()
bus.subscribe("user.registered", send_email_notification)
bus.subscribe("user.registered", log_event)

# Trigger event
await bus.publish("user.registered", {
    "email": "user@example.com",
    "user_id": "123",
})
```

**Caso d'uso reale:**

```python
# When a report is submitted
async def submit_report(...):
    report = await report_repo.create(...)

    # Publish event
    await event_bus.publish("report.submitted", {
        "report_id": report.id,
        "program_id": report.program_id,
        "author_id": report.author_id,
    })

    return report

# Multiple handlers react to the event
bus.subscribe("report.submitted", notify_program_owner)
bus.subscribe("report.submitted", update_analytics)
bus.subscribe("report.submitted", log_submission)
```

**Vantaggi:**
- Disaccoppiato: l'invio di un report non sa nulla delle notifiche
- Estensibile: si aggiungono nuovi handler senza modificare il codice
- Testabile: si può testare ogni handler in modo indipendente

---

## Combinare i pattern

Il codice del mondo reale spesso combina più pattern:

```python
# Dependency Injection + Repository + Layered Architecture

# 1. Route (Presentation Layer)
@router.post("/users")
async def create_user(
    data: UserCreate,
    session: DatabaseSession,  # Dependency Injection
) -> UserResponse:
    user = await register_user(session, data)  # Service Layer
    return UserResponse.from_orm(user)

# 2. Service (Business Layer)
async def register_user(
    session: AsyncSession,
    data: UserCreate,
) -> User:
    repo = UserRepository(session)  # Repository Pattern

    existing_user = await repo.find_by_email(data.email)
    if existing_user:
        raise UserAlreadyExistsError()

    password_hash = hash_password(data.password)  # Strategy Pattern
    user = await repo.create(
        email=data.email,
        password_hash=password_hash,
    )

    await session.commit()

    await event_bus.publish("user.registered", {  # Observer Pattern
        "user_id": user.id,
        "email": user.email,
    })

    return user

# 3. Repository (Data Access Layer)
class UserRepository(BaseRepository[User]):  # Generic Pattern
    async def find_by_email(self, email: str) -> User | None:
        ...

    async def create(self, **kwargs) -> User:
        ...
```

Questo combina:
- Dependency Injection (la sessione viene iniettata)
- Repository Pattern (astrazione dell'accesso ai dati)
- Layered Architecture (route → service → repository)
- Strategy Pattern (hash_password può usare algoritmi diversi)
- Observer Pattern (event_bus.publish)
- Generic Pattern (BaseRepository[User])

---

## Conclusione

I design pattern non servono a essere memorizzati, ma a riconoscere problemi comuni e ad applicare soluzioni collaudate.

**Punti chiave:**
1. **Dependency Injection** - Inietta le dipendenze, non codificarle in modo rigido
2. **Repository Pattern** - Astrai l'accesso ai dati dalla business logic
3. **Layered Architecture** - Separa i livelli di presentazione, business e dati
4. **Factory Pattern** - Centralizza la creazione degli oggetti
5. **Strategy Pattern** - Rendi gli algoritmi intercambiabili
6. **Singleton Pattern** - Una sola istanza per le risorse costose
7. **Mixin Pattern** - Componi il comportamento a partire da pezzi riutilizzabili
8. **Observer Pattern** - Disaccoppia i produttori di eventi dai consumatori

Per maggiori informazioni:
- Architettura del sistema: [ARCHITECTURE.it.md](./ARCHITECTURE.it.md)
- Progettazione del database: [DATABASE.it.md](./DATABASE.it.md)
- Implementazione della sicurezza: [SECURITY.it.md](./SECURITY.it.md)
- Tutorial pratico: [GETTING-STARTED.it.md](./GETTING-STARTED.it.md)
