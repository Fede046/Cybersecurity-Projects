# Approfondimento sull'architettura

Questo documento fornisce una spiegazione approfondita dell'architettura della Bug Bounty Platform, delle decisioni di design e del ragionamento dietro le scelte tecniche.

---

## Indice

1. [Panoramica](#panoramica)
2. [Architettura ad alto livello](#architettura-ad-alto-livello)
3. [Architettura del backend](#architettura-del-backend)
4. [Architettura del frontend](#architettura-del-frontend)
5. [Progettazione del database](#progettazione-del-database)
6. [Architettura di sicurezza](#architettura-di-sicurezza)
7. [Infrastruttura](#infrastruttura)
8. [Decisioni di design](#decisioni-di-design)

---

## Panoramica

Questa piattaforma è costruita con un'architettura moderna e pronta per la produzione che enfatizza:

1. **Separazione delle responsabilità** - Confini chiari tra i livelli
2. **Type Safety** - Garanzie a compile-time sia nel backend sia nel frontend
3. **Async-First** - Operazioni concorrenti scalabili
4. **Security by Design** - Più livelli di difesa
5. **Developer Experience** - Hot reload, linting, type checking
6. **Pronta per la produzione** - Containerizzata, con migrazioni, monitorata

---

## Architettura ad alto livello

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Browser                        │
│                  (React + TypeScript SPA)                    │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                       Nginx (Reverse Proxy)                  │
│  - Serves static React build                                 │
│  - Proxies /api/* to FastAPI backend                         │
│  - Gzip compression + security headers                       │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
        ▼                                 ▼
┌────────────────────┐          ┌────────────────────┐
│  FastAPI Backend   │◄────────►│  Redis (Cache)     │
│  (Python 3.12+)    │          │  - Sessions        │
│  - REST API        │          │  - Rate limiting   │
│  - JWT Auth        │          └────────────────────┘
│  - Business Logic  │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  PostgreSQL 18     │
│  - Primary data    │
│  - UUID v7 PKs     │
│  - ACID guarantees │
└────────────────────┘
```

### Perché questa architettura?

**Monolite tradizionale vs. microservizi:**
- Questo è un **monolite modulare** - più facile da sviluppare, distribuire ed eseguire il debug rispetto ai microservizi
- I servizi possono essere estratti in microservizi in seguito, se necessario
- Singolo database = transazioni ACID su tutte le entità

**Perché Nginx?**
- Reverse proxy di livello production
- Serving di file statici per la build di React
- La compressione Gzip riduce la banda
- Terminazione SSL/TLS
- Load balancing (se scalato orizzontalmente)

**Perché Redis?**
- Cache in-memory veloce per i dati di sessione
- Rate limiting senza interrogare PostgreSQL
- Può essere usato per il pub/sub (WebSocket) se servirà in seguito

---

## Architettura del backend

### Architettura a livelli

Il backend segue una **rigorosa architettura a livelli**:

```
┌──────────────────────────────────────────────────────────┐
│                    Routes (API Layer)                     │
│  - HTTP request/response handling                         │
│  - Input validation (Pydantic schemas)                    │
│  - Dependency injection                                   │
│  - OpenAPI documentation                                  │
└────────────────────────┬─────────────────────────────────┘
                         │ calls
                         ▼
┌──────────────────────────────────────────────────────────┐
│                  Services (Business Logic)                │
│  - Core business rules                                    │
│  - Orchestrates multiple repositories                     │
│  - Transaction management                                 │
│  - Domain logic                                           │
└────────────────────────┬─────────────────────────────────┘
                         │ calls
                         ▼
┌──────────────────────────────────────────────────────────┐
│               Repositories (Data Access)                  │
│  - CRUD operations                                        │
│  - Query building                                         │
│  - Database interaction                                   │
│  - No business logic                                      │
└────────────────────────┬─────────────────────────────────┘
                         │ operates on
                         ▼
┌──────────────────────────────────────────────────────────┐
│                  Models (Database Entities)               │
│  - SQLAlchemy ORM models                                  │
│  - Relationships                                          │
│  - Database schema definition                             │
└──────────────────────────────────────────────────────────┘
```

### Esempio: flusso di login dell'utente

Seguiamo una richiesta di login attraverso i livelli:

```python
# 1. Route Layer (backend/app/auth/routes.py)
@router.post("/login")
async def login(
    credentials: LoginRequest,
    session: DatabaseSession,
) -> TokenResponse:
    access_token, refresh_token = await authenticate_user(
        session=session,
        email=credentials.email,
        password=credentials.password,
        device_info=credentials.device_info,
    )
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
    )
```

L'handler della route:
- Riceve la richiesta HTTP POST su `/api/v1/auth/login`
- Valida l'input usando il modello Pydantic `LoginRequest`
- Inietta la sessione del database tramite il `Depends()` di FastAPI
- Chiama la funzione di servizio `authenticate_user()`
- Restituisce la risposta come modello Pydantic `TokenResponse`

```python
# 2. Service Layer (backend/app/auth/service.py)
async def authenticate_user(
    session: AsyncSession,
    email: str,
    password: str,
    device_info: str | None,
) -> tuple[str, str]:
    user_repo = UserRepository(session)

    # Find user by email
    user = await user_repo.find_by_email(email)
    if not user:
        raise InvalidCredentialsError()

    # Verify password (timing-safe comparison)
    if not security.verify_password(password, user.password_hash):
        raise InvalidCredentialsError()

    # Create tokens
    access_token = security.create_access_token(user)
    refresh_token = security.create_refresh_token()

    # Store refresh token in database
    token_repo = RefreshTokenRepository(session)
    await token_repo.create_refresh_token(
        user_id=user.id,
        token=refresh_token,
        device_info=device_info,
    )

    await session.commit()

    return access_token, refresh_token
```

Il livello dei servizi:
- Implementa la business logic (regole di autenticazione)
- Coordina più repository (User + RefreshToken)
- Gestisce la verifica della password in modo sicuro
- Crea i token JWT
- Gestisce le transazioni (commit)
- Solleva eccezioni di dominio (`InvalidCredentialsError`)

```python
# 3. Repository Layer (backend/app/user/repository.py)
class UserRepository(BaseRepository[User]):
    async def find_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()
```

Il repository:
- Costruisce la query SQL usando SQLAlchemy
- Esegue la query sul database
- Restituisce il modello ORM oppure None
- Nessuna business logic - puro accesso ai dati

```python
# 4. Model Layer (backend/app/user/models.py)
class User(Base):
    __tablename__ = "users"

    id: Mapped[UUID] = mapped_column(UUID(as_uuid=True), primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    full_name: Mapped[str] = mapped_column(String(255))
    role: Mapped[UserRole] = mapped_column(Enum(UserRole))

    # Relationships
    refresh_tokens: Mapped[list["RefreshToken"]] = relationship(
        back_populates="user",
        cascade="all, delete-orphan",
    )
```

Il modello:
- Definisce lo schema del database
- Mappa le classi Python sulle tabelle del database
- Dichiara le relazioni tra le entità
- Fornisce type hint per tutti i campi

### Perché i livelli?

**Separazione delle responsabilità:**
- Le route non conoscono le query del database
- I servizi non conoscono HTTP
- I repository non implementano regole di business
- I modelli sono pure strutture dati

**Testabilità:**
- Mock dei repository per testare i servizi
- Mock dei servizi per testare le route
- Unit test di ciascun livello in modo indipendente

**Manutenibilità:**
- Cambi il database? Aggiorna solo i repository
- Cambi la business logic? Aggiorna solo i servizi
- Cambi il formato dell'API? Aggiorna solo le route e gli schema

**Type Safety:**
- Ogni livello ha annotazioni di tipo rigorose
- MyPy verifica i tipi a compile-time
- Il refactoring è sicuro e prevedibile

---

## Struttura dei moduli

### Domain-Driven Design (DDD)

Ogni modulo di dominio è autonomo:

```
backend/src/app/user/
├── __init__.py
├── models.py          # User database model
├── repository.py      # UserRepository
├── schemas.py         # Pydantic request/response models
├── routes.py          # API endpoints
├── service.py         # Business logic functions
└── exceptions.py      # Domain-specific exceptions
```

**Perché questa struttura?**
- Tutto il codice relativo agli utenti è in un unico posto
- È facile trovare i file rilevanti
- Può essere estratto in un servizio separato in seguito
- Proprietà e confini chiari

### Modulo core

```
backend/src/app/core/
├── Base.py                # Base model classes (UUIDMixin, etc.)
├── base_repository.py     # Generic repository with CRUD
├── database.py            # Database session management
├── security.py            # Password hashing, JWT creation
├── dependencies.py        # FastAPI dependencies (auth, etc.)
├── exceptions.py          # Base exception classes
├── enums.py               # SafeEnum pattern
├── logging.py             # Structured logging
├── rate_limit.py          # Rate limiting configuration
└── constants.py           # String length constraints
```

Il modulo core fornisce:
- Classi base riutilizzabili
- Utility condivise
- Gestione della connessione al database
- Primitive di sicurezza

---

## Architettura del frontend

### Architettura basata sui componenti

Il modello a componenti di React abilita:
- **Riusabilità:** elementi UI come componenti autonomi
- **Componibilità:** UI complesse a partire da pezzi semplici
- **Manutenibilità:** cambiare un componente senza rompere gli altri

```
frontend/src/
├── routes/                    # File-based routing
│   ├── landing/              # Public landing page
│   ├── login/                # Authentication
│   ├── dashboard/            # User dashboard
│   ├── programs/             # Browse programs
│   │   └── [slug]/          # Dynamic route for program detail
│   ├── company/              # Company dashboard (nested routes)
│   │   ├── programs/        # Manage programs
│   │   ├── inbox/           # Incoming reports
│   │   └── reports/         # Report triage
│   └── admin/                # Admin panel
│
├── components/                # Reusable UI components
│   ├── common/               # Shared components (Button, Card, etc.)
│   ├── forms/                # Form components
│   └── layouts/              # Layout components (Shell, etc.)
│
├── api/                       # Backend integration
│   ├── hooks/                # React Query hooks (useAuth, usePrograms)
│   ├── types/                # TypeScript interfaces
│   └── index.ts              # Axios client configuration
│
├── stores/                    # State management (Zustand)
│   ├── auth.store.ts         # Authentication state
│   ├── shell.ui.store.ts     # UI state (sidebar, modals)
│   └── *.form.store.ts       # Form state
│
└── core/                      # App configuration
    ├── app/                  # App setup (router, providers)
    ├── styles/               # Global styles
    └── config.ts             # Constants and configuration
```

### Strategia di gestione dello stato

Il frontend usa un **approccio ibrido alla gestione dello stato**:

```
┌─────────────────────────────────────────────────────────┐
│              Server State (TanStack Query)               │
│  - API data (users, programs, reports)                   │
│  - Automatic caching and revalidation                    │
│  - Loading/error states                                  │
│  - Optimistic updates                                    │
└─────────────────────────────────────────────────────────┘
                         +
┌─────────────────────────────────────────────────────────┐
│              Client State (Zustand)                      │
│  - UI state (sidebar open/closed, modals)                │
│  - Form state (program creation, report submission)      │
│  - Authentication tokens (persisted to localStorage)     │
└─────────────────────────────────────────────────────────┘
```

**Perché due librerie di gestione dello stato?**

1. **TanStack Query per lo stato del server:**
   - Caching automatico (nessuna necessità di fare il cache manuale delle risposte API)
   - Refetching in background (mantiene i dati aggiornati)
   - Stati di caricamento ed errore già pronti all'uso
   - Aggiornamenti ottimistici per una UX migliore
   - Supporto a paginazione e infinite scroll

2. **Zustand per lo stato del client:**
   - API semplice (meno boilerplate di Redux)
   - Design TypeScript-first
   - Supporto alla persistenza (token di autenticazione)
   - Nessun provider wrapper necessario
   - Re-render minimi

**Esempio: recupero dei programmi con TanStack Query**

```typescript
// api/hooks/usePrograms.ts
export const usePrograms = (params?: ProgramQueryParams) => {
  return useQuery({
    queryKey: ["programs", params],
    queryFn: () => api.get<ProgramListResponse>("/programs", { params }),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};

// In component
function ProgramList() {
  const { data, isLoading, error } = usePrograms({ page: 1, limit: 20 });

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <div>
      {data.items.map(program => (
        <ProgramCard key={program.id} program={program} />
      ))}
    </div>
  );
}
```

TanStack Query automaticamente:
- Mette in cache la risposta (con chiave `["programs", params]`)
- Mostra lo stato di caricamento durante il fetch
- Esegue il refetch al focus della finestra (configurabile)
- Deduplica le richieste concorrenti
- Fornisce la gestione degli errori

**Esempio: stato della UI con Zustand**

```typescript
// stores/shell.ui.store.ts
interface ShellUIStore {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
}

export const useShellUIStore = create<ShellUIStore>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((state) => ({
    sidebarOpen: !state.sidebarOpen
  })),
}));

// In component
function Sidebar() {
  const { sidebarOpen, toggleSidebar } = useShellUIStore();

  return (
    <aside className={sidebarOpen ? "open" : "closed"}>
      <button onClick={toggleSidebar}>Toggle</button>
    </aside>
  );
}
```

Zustand fornisce:
- API semplice basata su hook
- Nessun provider wrapper
- Supporto TypeScript
- Re-render minimi (solo i componenti che usano lo stato modificato)

### Routing basato su file

Usando il file-based routing di React Router 7:

```
routes/
├── landing/
│   └── page.tsx               # → /
├── login/
│   └── page.tsx               # → /login
├── programs/
│   ├── page.tsx               # → /programs
│   └── [slug]/
│       └── page.tsx           # → /programs/:slug
└── company/
    ├── layout.tsx             # Shared layout for /company/*
    ├── programs/
    │   ├── page.tsx           # → /company/programs
    │   └── [id]/
    │       └── page.tsx       # → /company/programs/:id
    └── inbox/
        └── page.tsx           # → /company/inbox
```

**Vantaggi:**
- La struttura delle route rispecchia la struttura dei file
- Code splitting automatico (ogni route è un chunk separato)
- Layout annidati (le route company condividono un layout)
- Route dinamiche con la sintassi `[param]`

---

## Progettazione del database

### Panoramica dello schema

```
┌──────────────┐
│    users     │
│──────────────│
│ id (UUID v7) │◄───┐
│ email        │    │
│ password     │    │
│ role         │    │
└──────────────┘    │
                    │
      ┌─────────────┴──────────────┬────────────────┐
      │                            │                │
      ▼                            ▼                ▼
┌───────────────┐          ┌─────────────┐  ┌────────────┐
│refresh_tokens │          │  programs   │  │  reports   │
│───────────────│          │─────────────│  │────────────│
│ id            │          │ id          │  │ id         │
│ user_id (FK)  │          │ owner_id FK │  │ author_id  │
│ token_hash    │          │ name        │  │ program_id │
│ device_info   │          │ slug        │  │ title      │
│ family_id     │          │ status      │  │ severity   │
└───────────────┘          └─────────────┘  └────────────┘
                                  │
                      ┌───────────┴───────────┐
                      ▼                       ▼
                ┌──────────┐          ┌──────────────┐
                │  assets  │          │ reward_tiers │
                │──────────│          │──────────────│
                │ id       │          │ id           │
                │ program  │          │ program_id   │
                │ type     │          │ severity     │
                │ target   │          │ amount       │
                └──────────┘          └──────────────┘
```

### Chiavi primarie UUID v7

Gli ID auto-increment tradizionali hanno dei problemi:
- Prevedibili (rischio di sicurezza - enumerazione di tutti i record)
- Non globalmente unici (non si possono unire database)
- Richiedono un round-trip al database per essere generati

Gli UUID risolvono questo problema ma hanno i loro difetti:
- UUID v4 è casuale (negativo per l'indicizzazione del database)
- Non ordinabile nel tempo (non si può usare ORDER BY id per ottenere l'ordine cronologico)

**UUID v7 è il meglio dei due mondi:**
- Ordinabile nel tempo (i primi 48 bit sono il timestamp Unix in millisecondi)
- Globalmente unico (nessuna collisione nemmeno tra database diversi)
- Ottimo per gli indici del database (ordine lessicografico = ordine cronologico)
- Sicuro (i bit rimanenti sono casuali)

```python
import uuid_utils as uuid

# Generate UUID v7
user_id = uuid.uuid7()  # → 018d3f54-8c3a-7000-a234-56789abcdef0
                        #    ^^^^^^^^^^^^^^^^ ← timestamp
                        #                     ^^^^^^^^^^^^^^^^^^^ ← random
```

### Pattern SafeEnum

Gli enum tradizionali in SQLAlchemy memorizzano il nome dell'enum:

```python
class Status(enum.Enum):
    ACTIVE = "active"
    PAUSED = "paused"

# Database stores: "ACTIVE" (the Python name)
```

**Problema:** Se rinomini l'enum Python, il database si rompe:

```python
class Status(enum.Enum):
    RUNNING = "active"  # Renamed ACTIVE → RUNNING
    PAUSED = "paused"

# Database still has "ACTIVE", but Python doesn't recognize it!
```

**Soluzione SafeEnum:** Memorizzare il valore, non il nome:

```python
class Status(SafeEnum):
    ACTIVE = "active"
    PAUSED = "paused"

# Database stores: "active" (the value)
```

Ora puoi rinominare in sicurezza:

```python
class Status(SafeEnum):
    RUNNING = "active"  # Value is still "active"
    PAUSED = "paused"

# Database has "active", Python maps it to Status.RUNNING ✓
```

### Soft delete vs hard delete

Alcuni modelli usano i **soft delete** (impostano un timestamp `deleted_at` invece di rimuovere la riga):

```python
class SoftDeleteMixin:
    deleted_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
        default=None,
    )
```

**Quando usare i soft delete:**
- Account utente (requisiti di compliance - conservare l'audit trail)
- Record finanziari (non cancellare mai davvero)
- Report (preservare la cronologia anche se il programma viene eliminato)

**Quando usare gli hard delete:**
- Token di sessione (non serve conservarli dopo il logout)
- Dati temporanei (cache, OTP)
- Richieste di cancellazione GDPR (devono essere eliminati davvero)

---

## Architettura di sicurezza

### Difesa multi-livello

La sicurezza è implementata su più livelli:

```
┌──────────────────────────────────────────────────────┐
│  1. Input Validation (Pydantic)                       │
│     - Type checking                                   │
│     - String length limits                            │
│     - Email/URL format validation                     │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│  2. Authentication (JWT)                              │
│     - Token-based auth                                │
│     - Short-lived access tokens (15 min)              │
│     - Long-lived refresh tokens (7 days)              │
│     - Token versioning                                │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│  3. Authorization (RBAC)                              │
│     - Role-based access control                       │
│     - Resource ownership checks                       │
│     - Admin-only endpoints                            │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│  4. Rate Limiting                                     │
│     - 100 req/min default                             │
│     - 20 req/min for auth endpoints                   │
│     - Per-IP tracking                                 │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│  5. Secure Storage                                    │
│     - Argon2id password hashing                       │
│     - Hashed refresh tokens (not plaintext)           │
│     - Encrypted secrets in env vars                   │
└──────────────────────────────────────────────────────┘
```

### Flusso dei token JWT

```
User Login
   │
   ▼
┌──────────────────────────────────────┐
│  1. POST /api/v1/auth/login          │
│     { email, password }              │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  2. Verify password (Argon2id)       │
│     - Timing-safe comparison         │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  3. Generate tokens                  │
│     - access_token (15 min)          │
│     - refresh_token (7 days)         │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  4. Store refresh token in DB        │
│     - Hashed (not plaintext)         │
│     - With device info and IP        │
│     - Family ID for replay detection │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  5. Return tokens to client          │
│     { access_token, refresh_token }  │
└──────────────────────────────────────┘
```

**Payload dell'access token:**
```json
{
  "sub": "018d3f54-8c3a-7000-a234-56789abcdef0",  // user_id
  "role": "USER",
  "token_version": 1,
  "exp": 1704123456,  // expires in 15 minutes
  "iat": 1704122556
}
```

**Perché access token di breve durata?**
- Se viene rubato, l'attaccante ha solo 15 minuti
- Non c'è modo di revocare gli access token (sono stateless)
- Occorre usare il refresh token per ottenere un nuovo access token

**Flusso del refresh token:**

```
Access token expires (15 min)
   │
   ▼
┌──────────────────────────────────────┐
│  1. POST /api/v1/auth/refresh        │
│     { refresh_token }                │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  2. Verify refresh token in DB       │
│     - Check hash matches             │
│     - Check not expired              │
│     - Check not revoked              │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  3. Token rotation                   │
│     - Delete old refresh token       │
│     - Generate new refresh token     │
│     - Store new token in DB          │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  4. Generate new access token        │
│     - Same user_id                   │
│     - Incremented token_version      │
└────────────┬─────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│  5. Return new tokens                │
│     { access_token, refresh_token }  │
└──────────────────────────────────────┘
```

**La token rotation previene i replay attack:**
- Ogni refresh token è monouso
- Se un attaccante ruba un refresh token, questo diventa invalido dopo un solo utilizzo
- Se un refresh token viene riutilizzato, lo rileviamo (mismatch del family ID) e revochiamo tutti i token di quell'utente

### Token Versioning

Il token versioning consente l'invalidazione istantanea:

```python
class User(Base):
    token_version: Mapped[int] = mapped_column(Integer, default=1)

# When user changes password:
user.token_version += 1
await session.commit()
```

Ora tutti gli access token esistenti diventano invalidi:
- I vecchi token hanno `token_version: 1`
- Il `token_version` corrente dell'utente è `2`
- La verifica del token fallisce: `1 != 2`

Non c'è bisogno di mantenere una blacklist di token!

---

## Infrastruttura

### Architettura Docker Compose

**Production (`compose.yml`):**

```yaml
services:
  nginx:
    build: ./infra/nginx
    ports:
      - "${NGINX_HOST_PORT}:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db
      - redis

  db:
    image: postgres:18-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
```

**Perché questa struttura?**
- **nginx dipende dal backend:** Nginx non può avviarsi finché il backend non è pronto
- **il backend dipende da db + redis:** Il backend ha bisogno di database e cache
- **Volumi per la persistenza dei dati:** I dati di database e Redis sopravvivono ai riavvii dei container

### Build Docker multi-stage

**Dockerfile del backend:**

```dockerfile
# Stage 1: Build dependencies
FROM python:3.12-slim as builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Stage 2: Production image
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY ./src ./src
CMD ["gunicorn", "main:app"]
```

**Vantaggi:**
- Image finale più piccola (nessun build tool)
- Build più veloci (dipendenze in cache nello stage builder)
- Più sicura (nessuna dipendenza di build in produzione)

**Dockerfile del frontend (Nginx):**

```dockerfile
# Stage 1: Build React app
FROM node:22-alpine as builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install
COPY . .
RUN pnpm build

# Stage 2: Serve with Nginx
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
```

**Vantaggi:**
- Nessun Node.js nell'image di produzione
- Nginx è ottimizzato per servire file statici
- Image finale più piccola (~50MB contro ~500MB)

---

## Decisioni di design

### Perché FastAPI?

**Rispetto a Flask:**
- Supporto async/await (vince FastAPI)
- Documentazione OpenAPI automatica (vince FastAPI)
- Validazione dei dati integrata con Pydantic (vince FastAPI)
- Type hint per l'autocompletamento dell'IDE (vince FastAPI)

**Rispetto a Django:**
- Supporto ORM async (pareggio - Django 4.1+ ce l'ha)
- Flessibilità (vince FastAPI - Django è opinionated)
- Pannello di amministrazione (vince Django)
- Batterie incluse (vince Django - ha auth, admin, ecc.)

**Decisione:** FastAPI per il suo design async-first e le funzionalità moderne di Python.

### Perché PostgreSQL?

**Rispetto a MySQL:**
- Supporto JSON (pareggio)
- Ricerca full-text (vince PostgreSQL)
- Conformità ACID (pareggio)
- Indici JSON (vince PostgreSQL)

**Rispetto a MongoDB:**
- Transazioni ACID (vince PostgreSQL)
- Validazione dello schema (pareggio - Postgres ha il JSON schema)
- Join (vince PostgreSQL)
- Flessibilità (vince MongoDB per i dati non strutturati)

**Decisione:** PostgreSQL per le garanzie ACID e la modellazione di dati relazionali.

### Perché React + TypeScript?

**Rispetto a Vue:**
- Dimensione dell'ecosistema (vince React)
- Supporto TypeScript (pareggio)
- Curva di apprendimento (vince Vue - più facile)
- Supporto aziendale (pareggio - React di Meta, Vue indipendente)

**Rispetto a Svelte:**
- Maturità (vince React)
- Mercato del lavoro (vince React)
- Dimensione del bundle (vince Svelte)
- Curva di apprendimento (vince Svelte)

**Decisione:** React + TypeScript per il suo ecosistema maturo e l'adozione nel settore.

### Perché Zustand?

**Rispetto a Redux:**
- Boilerplate (vince Zustand - molto più semplice)
- DevTools (vince Redux - più maturo)
- Middleware (pareggio)
- Dimensione del bundle (vince Zustand)

**Rispetto alla Context API:**
- Prestazioni (vince Zustand - nessun re-render superfluo)
- Persistenza (vince Zustand - integrata)
- DX (vince Zustand - API più semplice)

**Decisione:** Zustand per la sua semplicità e le sue prestazioni.

### Perché TanStack Query?

**Rispetto a Redux Toolkit Query:**
- Flessibilità (vince TanStack - non legato a Redux)
- Gestione della cache (pareggio)
- Dimensione del bundle (vince TanStack)
- Curva di apprendimento (pareggio)

**Rispetto a SWR:**
- Funzionalità (vince TanStack - più completo)
- Dimensione del bundle (vince SWR)
- Paginazione (vince TanStack)
- Community (pareggio)

**Decisione:** TanStack Query per il suo insieme completo di funzionalità e il design indipendente dal framework.

---

## Considerazioni sulle prestazioni

### Indicizzazione del database

Gli indici vengono creati per le colonne interrogate di frequente:

```python
class User(Base):
    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        index=True  # ← Index for fast lookups
    )
```

**Compromesso:**
- Letture più veloci (le query che usano email sono rapide)
- Scritture più lente (l'indice deve essere aggiornato a ogni insert/update)
- Più spazio su disco (l'indice è memorizzato separatamente)

### Connection pooling

Le connessioni al database sono costose da creare. Il connection pooling riutilizza le connessioni:

```python
DB_POOL_SIZE=20          # Max connections in pool
DB_MAX_OVERFLOW=10       # Extra connections when pool is full
DB_POOL_TIMEOUT=30       # Wait 30s for available connection
DB_POOL_RECYCLE=1800     # Recycle connections after 30min
```

**Come funziona:**
1. Arriva una richiesta
2. Il backend prende una connessione dal pool
3. Esegue la query
4. Restituisce la connessione al pool (non la chiude)
5. La richiesta successiva riutilizza la stessa connessione

**Vantaggi:**
- Gestione delle richieste più veloce (nessun overhead di connessione)
- Carico ridotto sul database (meno negoziazioni di connessione)
- Migliore scalabilità (gestione di più richieste concorrenti)

### Caching con Redis

Redis è usato per i dati a cui si accede di frequente:

```python
# Without cache
user = await user_repo.find_by_id(user_id)  # Database query every time

# With cache
user = await redis.get(f"user:{user_id}")
if not user:
    user = await user_repo.find_by_id(user_id)
    await redis.set(f"user:{user_id}", user, ex=300)  # Cache for 5 min
```

**Strategie di invalidazione della cache:**
1. Basata sul tempo (TTL) - scade dopo N secondi
2. Basata sugli eventi - invalidazione su update/delete
3. LRU (Least Recently Used) - Redis rimuove automaticamente le chiavi vecchie

### Code splitting del frontend

Vite divide automaticamente il codice per route:

```typescript
// routes/dashboard/page.tsx is lazy-loaded
const Dashboard = lazy(() => import('./routes/dashboard/page'));
```

**Vantaggi:**
- Caricamento iniziale della pagina più veloce (si carica solo la landing page)
- La navigazione successiva è veloce (i chunk sono in cache)
- Migliore uso della banda (non si carica il pannello admin per gli utenti normali)

---

## Conclusione

Questa architettura dà priorità a:
1. **Developer Experience** - Hot reload, type safety, linting
2. **Sicurezza** - Più livelli di difesa, pattern di autenticazione moderni
3. **Scalabilità** - Operazioni async, connection pooling, caching
4. **Manutenibilità** - Chiara separazione dei livelli, domain-driven design
5. **Pronta per la produzione** - Containerizzazione, migrazioni, monitoring

Ogni decisione di design comporta dei compromessi. Questa architettura favorisce correttezza, sicurezza e manutenibilità rispetto alla pura performance (che può essere ottimizzata in seguito se necessario).

Per maggiori dettagli su argomenti specifici:
- Design pattern: [PATTERNS.it.md](./PATTERNS.it.md)
- Schema del database: [DATABASE.it.md](./DATABASE.it.md)
- Implementazione della sicurezza: [SECURITY.it.md](./SECURITY.it.md)
- Tutorial passo passo: [GETTING-STARTED.it.md](./GETTING-STARTED.it.md)
