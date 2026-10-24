# 02-ARCHITECTURE.md

# Architettura di sistema

Questo documento analizza come è progettato il sistema e perché sono state prese certe decisioni architetturali.

## Architettura ad alto livello

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client Browser                          │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTP/HTTPS
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx Reverse Proxy                         │
│  - Routes /api/* → Backend (FastAPI)                            │
│  - Routes /* → Frontend (Static Files)                          │
│  - Handles CORS, compression, caching                           │
└──────────────┬─────────────────────────┬────────────────────────┘
               │                         │
               ▼                         ▼
┌──────────────────────────┐   ┌─────────────────────────────────┐
│   Backend (FastAPI)      │   │   Frontend (React/Vite)         │
│  ┌────────────────────┐  │   │  ┌───────────────────────────┐  │
│  │     Routes         │  │   │  │    Components             │  │
│  │  /auth, /scans     │  │   │  │  Auth, Scan UI            │  │
│  └─────────┬──────────┘  │   │  └─────────┬─────────────────┘  │
│            ▼              │   │            │                    │
│  ┌────────────────────┐  │   │  ┌───────────────────────────┐  │
│  │    Services        │  │   │  │    TanStack Query         │  │
│  │  Business Logic    │  │   │  │  Server State Mgmt        │  │
│  └─────────┬──────────┘  │   │  └─────────┬─────────────────┘  │
│            ▼              │   │            │                    │
│  ┌────────────────────┐  │   │  ┌───────────────────────────┐  │
│  │   Repositories     │  │   │  │    Zustand Stores         │  │
│  │  Data Access       │  │   │  │  Local State Mgmt         │  │
│  └─────────┬──────────┘  │   │  └───────────────────────────┘  │
│            │              │   │                                 │
│  ┌─────────▼──────────┐  │   └─────────────────────────────────┘
│  │    Scanners        │  │
│  │  RateLimit, Auth,  │  │
│  │  SQLi, IDOR        │  │
│  └────────────────────┘  │
└──────────────┬────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                           │
│  Tables: users, scans, test_results                             │
└─────────────────────────────────────────────────────────────────┘
```

### Scomposizione dei componenti

**Nginx Reverse Proxy**
- Scopo: unico punto di ingresso per tutto il traffico HTTP
- Responsabilità: instradare le richieste in base al path, servire file statici, gestire la terminazione SSL (produzione), comprimere le risposte, mettere in cache gli asset statici
- Interfacce: espone la porta 80 (HTTP) e opzionalmente la 443 (HTTPS), fa da proxy verso il backend sulla porta interna 8000

**FastAPI Backend**
- Scopo: server API REST che fornisce servizi di autenticazione e scansione
- Responsabilità: validare le richieste, imporre l'autenticazione, eseguire scansioni di sicurezza, memorizzare i risultati, restituire risposte JSON
- Interfacce: espone endpoint HTTP su `/auth/*` e `/scans/*`, si connette a PostgreSQL per la persistenza dei dati

**React Frontend**
- Scopo: interfaccia utente per creare scansioni e visualizzare i risultati
- Responsabilità: validazione dei form, comunicazione con le API, gestione dello stato, visualizzazione dei risultati
- Interfacce: comunica con il backend tramite il prefisso `/api`, viene renderizzato nel browser

**PostgreSQL Database**
- Scopo: storage persistente per utenti, scansioni e risultati dei test
- Responsabilità: integrità dei dati, applicazione delle relazioni, ottimizzazione delle query
- Interfacce: accetta connessioni dal backend sulla porta 5432, applica i vincoli di foreign key

**Moduli scanner**
- Scopo: eseguire test di sicurezza contro le API target
- Responsabilità: inviare richieste HTTP, analizzare le risposte, rilevare vulnerabilità, raccogliere evidenze
- Interfacce: ereditano da `BaseScanner`, restituiscono schema `TestResultCreate`

## Flusso dei dati

### Caso d'uso principale: creare ed eseguire una scansione

Walkthrough passo per passo di cosa accade quando un utente invia una nuova scansione:

```
1. User submits form → Frontend validates (Zod schema)
   Input: { targetUrl, authToken, testsToRun, maxRequests }
   Validation happens at frontend/src/lib/validation.ts:42-52

2. Frontend → POST /api/scans/ → Nginx
   Adds Authorization header with JWT from localStorage
   Request routed based on /api prefix

3. Nginx → Backend (routes/scans.py:23-37)
   Proxy passes to http://backend:8000/scans/
   Preserves headers including Authorization

4. Route handler → Dependencies check auth
   @limiter.limit("15/minute") - rate limits this endpoint
   get_current_user() extracts JWT, validates, loads user from DB
   Code at backend/core/dependencies.py:17-49

5. Route → Service layer (services/scan_service.py:23-65)
   ScanService.run_scan(db, user_id, scan_request)
   Creates Scan record in database via repository

6. Service → Scanner modules (scanners/*.py)
   Loops through requested tests (rate_limit, auth, sqli, idor)
   Instantiates appropriate scanner class for each test
   Each scanner inherits from BaseScanner

7. Scanner → Target API
   Makes HTTP requests using requests.Session
   Implements retry logic, rate limiting, timeout handling
   Base logic at backend/scanners/base_scanner.py:40-156

8. Scanner analyzes responses → Returns TestResultCreate
   Detects vulnerabilities based on:
   - Status codes (429 for rate limiting)
   - Response content (SQL errors)
   - Timing differences (blind SQLi)
   - Header patterns (JWT algorithms)

9. Service saves results → Repository → Database
   TestResultRepository.create_test_result() for each scanner output
   Foreign key links results to scan
   Code at backend/repositories/test_result_repository.py:19-50

10. Service → Route → JSON response
    ScanResponse includes full scan with nested test_results
    Frontend receives and redirects to /scans/{id}

11. Frontend fetches full scan → GET /api/scans/{id}
    TanStack Query caches result
    Renders TestResultCard for each result
```

Esempio con riferimenti al codice:

```python
# Step 5: Route handler
@router.post("/", response_model=ScanResponse)
@limiter.limit(settings.API_RATE_LIMIT_SCAN)  # "15/minute"
async def create_scan(
    request: Request,
    scan_request: ScanRequest,
    db: Session = Depends(get_db),
    current_user: UserResponse = Depends(get_current_user),  # Auth check
) -> ScanResponse:
    return ScanService.run_scan(db, current_user.id, scan_request)

# Step 6: Service orchestrates scanners
scanner_mapping: dict[TestType, type[BaseScanner]] = {
    TestType.RATE_LIMIT: RateLimitScanner,
    TestType.AUTH: AuthScanner,
    TestType.SQLI: SQLiScanner,
    TestType.IDOR: IDORScanner,
}

for test_type in scan_request.tests_to_run:
    scanner_class = scanner_mapping.get(test_type)
    scanner = scanner_class(
        target_url=str(scan_request.target_url),
        auth_token=scan_request.auth_token,
        max_requests=scan_request.max_requests,
    )
    result = scanner.scan()  # Execute the test
    results.append(result)

# Step 9: Save to database
for result in results:
    TestResultRepository.create_test_result(
        db=db,
        scan_id=scan.id,
        test_name=result.test_name,
        status=result.status,
        severity=result.severity,
        details=result.details,
        evidence_json=result.evidence_json,
        recommendations_json=result.recommendations_json,
    )
```

### Caso d'uso secondario: flusso di registrazione utente

```
1. User fills registration form → Frontend validates
   Zod schema checks: email format, password strength (8+ chars, uppercase, lowercase, number)
   frontend/src/lib/validation.ts:17-31

2. POST /api/auth/register → Rate limited to 15/minute
   Prevents automated account creation abuse
   backend/routes/auth.py:25-42

3. Service checks if email exists → Repository query
   UserRepository.get_by_email() checks for duplicates
   Returns 400 if email already registered

4. Service hashes password with bcrypt
   core/security.py:11-18 uses bcrypt.gensalt() and hashpw()
   Salt automatically generated per password

5. Repository creates user record
   UserRepository.create_user(email, hashed_password)
   Sets is_active=True, created_at=UTC timestamp

6. Response returns user data (NOT password)
   UserResponse schema excludes hashed_password field
   Frontend redirects to login page
```

## Design pattern

### Repository Pattern

**Cos'è:**
Un livello di astrazione tra la logica di business e l'accesso ai dati. Tutte le query al database passano attraverso classi repository che forniscono interfacce pulite.

**Dove lo usiamo:**
- `repositories/user_repository.py` - Operazioni CRUD sugli utenti
- `repositories/scan_repository.py` - Query sulle scansioni con eager loading
- `repositories/test_result_repository.py` - Operazioni sui risultati dei test

**Perché lo abbiamo scelto:**
Mantiene i service puliti e testabili. I service chiamano `UserRepository.get_by_email(db, email)` invece di scrivere query raw. Se passiamo da SQLAlchemy a un altro ORM o a un database completamente diverso, cambiamo solo le implementazioni dei repository, non la logica dei service.

**Compromessi:**
- Pro: testabile (mock dei repository), manutenibile (query in un unico posto), flessibile (implementazioni sostituibili)
- Contro: un livello di astrazione in più, più file da navigare, può sembrare eccessivo per semplici CRUD

Esempio di implementazione:

```python
# repositories/user_repository.py:12-35
class UserRepository:
    """
    Repository for User database operations
    """
    @staticmethod
    def get_by_id(db: Session, user_id: int) -> User | None:
        return db.query(User).filter(User.id == user_id).first()
    
    @staticmethod
    def get_by_email(db: Session, email: str) -> User | None:
        return db.query(User).filter(User.email == email).first()
    
    @staticmethod
    def create_user(
        db: Session,
        email: str,
        hashed_password: str,
        commit: bool = True
    ) -> User:
        user = User(email=email, hashed_password=hashed_password)
        db.add(user)
        if commit:
            db.commit()
            db.refresh(user)
        return user
```

I service lo usano:

```python
# services/auth_service.py:24-32
existing_user = UserRepository.get_by_email(db, user_data.email)
if existing_user:
    raise HTTPException(status_code=400, detail="Email already registered")

hashed_password = hash_password(user_data.password)

user = UserRepository.create_user(
    db=db,
    email=user_data.email,
    hashed_password=hashed_password,
)
```

### Dependency Injection (il Depends di FastAPI)

**Cos'è:**
Il sistema di dependency injection di FastAPI fornisce automaticamente valori ai parametri dei route handler. Usato per sessioni di database, autenticazione, rate limiting.

**Dove lo usiamo:**
Ogni route handler in `routes/auth.py` e `routes/scans.py` usa dipendenze:

```python
# routes/scans.py:23-37
@router.post("/", response_model=ScanResponse)
@limiter.limit(settings.API_RATE_LIMIT_SCAN)
async def create_scan(
    request: Request,  # Injected by FastAPI
    scan_request: ScanRequest,  # Parsed and validated from request body
    db: Session = Depends(get_db),  # Database session injected
    current_user: UserResponse = Depends(get_current_user),  # Auth check injected
) -> ScanResponse:
    return ScanService.run_scan(db, current_user.id, scan_request)
```

**Perché lo abbiamo scelto:**
Netta separazione delle responsabilità. Il route handler non sa come:
- Ottenere una sessione di database (gestito da `get_db`)
- Validare i token JWT (gestito da `get_current_user`)
- Fare il parsing dei body delle richieste (gestito da Pydantic)

Questo rende i test più semplici: si fa il mock delle dipendenze, non dell'intero ciclo della richiesta.

**Compromessi:**
- Pro: testabile, riutilizzabile, dipendenze esplicite, cleanup automatico (chiusura della sessione)
- Contro: comportamento "magico" per i principianti, il debug può essere complicato se una dipendenza fallisce

L'implementazione della dipendenza `get_current_user` (`core/dependencies.py:17-49`):

```python
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db),
) -> UserResponse:
    """
    FastAPI dependency to extract and verify the current authenticated user
    """
    try:
        payload = decode_token(credentials.credentials)
        email: str | None = payload.get("sub")
        
        if email is None:
            raise HTTPException(status_code=401, detail="Invalid authentication credentials")
        
        user = UserRepository.get_by_email(db, email)
        
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        
        return UserResponse.model_validate(user)
    
    except ValueError:
        raise HTTPException(status_code=401, detail="Invalid authentication credentials")
```

### Template Method (classe astratta BaseScanner)

**Cos'è:**
Definire lo scheletro di un algoritmo in una classe base, lasciando che le sottoclassi sovrascrivano passi specifici. Tutti gli scanner condividono la logica HTTP comune ma implementano il proprio metodo `scan()`.

**Dove lo usiamo:**
`scanners/base_scanner.py` fornisce le funzionalità comuni:

```python
class BaseScanner(ABC):
    def __init__(self, target_url: str, auth_token: str | None = None, max_requests: int | None = None):
        self.target_url = target_url.rstrip("/")
        self.auth_token = auth_token
        self.max_requests = max_requests or settings.DEFAULT_MAX_REQUESTS
        self.session = self._create_session()
        self.last_request_time = 0.0
        self.request_count = 0
    
    def make_request(self, method: str, endpoint: str, **kwargs: Any) -> requests.Response:
        """Common HTTP request logic with retry and rate limiting"""
        self._wait_before_request()
        # ... retry logic, backoff, timeout handling
    
    def get_baseline_timing(self, endpoint: str, samples: int | None = None) -> tuple[float, float]:
        """Statistical baseline for time-based detection"""
        # ... takes samples, calculates mean and stdev
    
    @abstractmethod
    def scan(self) -> TestResultCreate:
        """Must be implemented by specific scanner classes"""
```

Le sottoclassi implementano `scan()`:

```python
# scanners/sqli_scanner.py:25-60
class SQLiScanner(BaseScanner):
    def scan(self) -> TestResultCreate:
        error_based_test = self._test_error_based_sqli()
        if error_based_test["vulnerable"]:
            return self._create_vulnerable_result(...)
        
        boolean_based_test = self._test_boolean_based_sqli()
        if boolean_based_test["vulnerable"]:
            return self._create_vulnerable_result(...)
        
        time_based_test = self._test_time_based_sqli()
        if time_based_test["vulnerable"]:
            return self._create_vulnerable_result(...)
        
        return TestResultCreate(status=ScanStatus.SAFE, ...)
```

**Perché lo abbiamo scelto:**
Elimina la duplicazione del codice. Spaziatura delle richieste, logica di retry, gestione delle sessioni: scritte una sola volta in `BaseScanner`, usate da tutti e quattro i tipi di scanner.

**Compromessi:**
- Pro: principio DRY, comportamento coerente, facile aggiungere nuovi scanner
- Contro: forte accoppiamento con la classe base, l'ereditarietà può essere limitante

## Separazione dei livelli

Il backend usa un'architettura a tre livelli:

```
┌────────────────────────────────────┐
│    Layer 1: Routes                 │
│    - HTTP request/response         │
│    - Validation (Pydantic)         │
│    - Auth checks (dependencies)    │
│    - Rate limiting                 │
└────────────────┬───────────────────┘
                 ↓
┌────────────────────────────────────┐
│    Layer 2: Services               │
│    - Business logic                │
│    - Orchestration                 │
│    - Transaction management        │
│    - Error handling                │
└────────────────┬───────────────────┘
                 ↓
┌────────────────────────────────────┐
│    Layer 3: Repositories           │
│    - Database queries              │
│    - Data access only              │
│    - No business logic             │
└────────────────────────────────────┘
```

### Perché i livelli?

La separazione rende ogni livello testabile in isolamento:
- Testa le route con service mockati
- Testa i service con repository mockati
- Testa i repository contro un vero database di test

Impone inoltre la single responsibility. Le route non scrivono SQL. I service non fanno il parsing degli header HTTP. I repository non implementano regole di business.

### Cosa vive dove

**Livello 1: Routes** (`routes/auth.py`, `routes/scans.py`)
- File: funzioni route handler decorate con `@router.get/post/delete`
- Import: possono importare da service, schema, dipendenze
- Vietato: accesso diretto al database, logica di business, chiamare direttamente i repository

Esempio di route:

```python
# routes/scans.py:67-83
@router.get("/{scan_id}", response_model=ScanResponse)
@limiter.limit(settings.API_RATE_LIMIT_DEFAULT)
async def get_scan(
    request: Request,
    scan_id: int,
    db: Session = Depends(get_db),
    current_user: UserResponse = Depends(get_current_user),
) -> ScanResponse:
    """
    Get a specific scan by ID
    """
    return ScanService.get_scan_by_id(db, scan_id, current_user.id)
```

**Livello 2: Services** (`services/auth_service.py`, `services/scan_service.py`)
- File: classi service con metodi statici
- Import: repository, modelli, schema, utility
- Vietato: codice specifico di HTTP (request, response), query SQL dirette

Esempio di metodo di un service:

```python
# services/scan_service.py:23-65
@staticmethod
def run_scan(db: Session, user_id: int, scan_request: ScanRequest) -> ScanResponse:
    # Create scan record
    scan = ScanRepository.create_scan(
        db=db,
        user_id=user_id,
        target_url=str(scan_request.target_url),
    )
    
    # Map test types to scanner classes
    scanner_mapping: dict[TestType, type[BaseScanner]] = {
        TestType.RATE_LIMIT: RateLimitScanner,
        TestType.AUTH: AuthScanner,
        TestType.SQLI: SQLiScanner,
        TestType.IDOR: IDORScanner,
    }
    
    results: list[TestResultCreate] = []
    
    # Execute each requested test
    for test_type in scan_request.tests_to_run:
        scanner_class = scanner_mapping.get(test_type)
        scanner = scanner_class(...)
        result = scanner.scan()
        results.append(result)
    
    # Save all results
    for result in results:
        TestResultRepository.create_test_result(db=db, scan_id=scan.id, ...)
    
    db.refresh(scan)
    return ScanResponse.model_validate(scan)
```

**Livello 3: Repositories** (`repositories/user_repository.py`, `repositories/scan_repository.py`, `repositories/test_result_repository.py`)
- File: classi repository con metodi statici per le operazioni sul database
- Import: modelli, SQLAlchemy, config
- Vietato: logica di business, gestione HTTP, chiamare altri repository

Esempio di metodo di un repository:

```python
# repositories/scan_repository.py:48-71
@staticmethod
def get_by_user(
    db: Session,
    user_id: int,
    skip: int = 0,
    limit: int | None = None
) -> list[Scan]:
    """
    Get all scans for a user with pagination
    """
    if limit is None:
        limit = settings.DEFAULT_PAGINATION_LIMIT
    
    return (
        db.query(Scan)
        .options(joinedload(Scan.test_results))  # Eager load relationships
        .filter(Scan.user_id == user_id)
        .order_by(Scan.scan_date.desc())
        .offset(skip)
        .limit(limit)
        .all()
    )
```

## Modelli di dati

### Modello User

```python
# models/User.py:12-43
class User(BaseModel):
    """
    Stores authentication credentials and user information
    """
    __tablename__ = "users"
    
    email = Column(
        String(settings.EMAIL_MAX_LENGTH),  # 255 chars
        unique=True,
        nullable=False,
        index=True,  # Fast lookups by email
    )
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True, nullable=False)
```

**Campi spiegati:**
- `id`: chiave primaria auto-incrementante (ereditata da `BaseModel`)
- `email`: identificatore univoco per il login, indicizzato per query di autenticazione veloci
- `hashed_password`: hash bcrypt, mai memorizzato in chiaro, mai restituito nelle risposte API
- `is_active`: flag di soft delete, permette di disabilitare gli account senza perdere dati
- `created_at`, `updated_at`: timestamp ereditati da `BaseModel`

**Relazioni:**
- One-to-many con Scan: `user.scans` restituisce tutte le scansioni create da questo utente
- Definita dalla relationship nel modello Scan: `user = relationship("User", backref="scans")`

### Modello Scan

```python
# models/Scan.py:15-57
class Scan(BaseModel):
    """
    Stores metadata about scans performed on target URLs
    """
    __tablename__ = "scans"
    
    user_id = Column(
        Integer,
        ForeignKey("users.id", ondelete="CASCADE"),  # Delete scans when user deleted
        nullable=False,
        index=True,
    )
    target_url = Column(
        String(settings.URL_MAX_LENGTH),  # 2048 chars
        nullable=False,
    )
    scan_date = Column(
        DateTime(timezone=True),
        default=lambda: datetime.now(UTC),
        nullable=False,
    )
    
    user = relationship("User", backref="scans")
    test_results = relationship(
        "TestResult",
        back_populates="scan",
        cascade="all, delete-orphan",  # Delete results when scan deleted
    )
```

**Campi spiegati:**
- `user_id`: foreign key verso la tabella users, indicizzata per filtrare le scansioni per utente
- `target_url`: URL che è stato scansionato, fino a 2048 caratteri per query string lunghe
- `scan_date`: quando la scansione è stata avviata, datetime timezone-aware in UTC
- `CASCADE`: quando un utente viene eliminato, le sue scansioni vengono eliminate. Quando una scansione viene eliminata, i suoi risultati dei test vengono eliminati

**Relazioni:**
- Many-to-one con User: `scan.user` ottiene l'utente che l'ha creata
- One-to-many con TestResult: `scan.test_results` ottiene tutti i finding di vulnerabilità

**Properties** (calcolate, non memorizzate):

```python
@property
def has_vulnerabilities(self) -> bool:
    return any(result.status == "vulnerable" for result in self.test_results)

@property
def vulnerability_count(self) -> int:
    return sum(1 for result in self.test_results if result.status == "vulnerable")
```

### Modello TestResult

```python
# models/TestResult.py:16-57
class TestResult(BaseModel):
    """
    Stores individual test results for each security scan
    """
    __tablename__ = "test_results"
    
    scan_id = Column(
        Integer,
        ForeignKey("scans.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
    test_name = Column(
        Enum(TestType),  # rate_limit, auth, sqli, idor
        nullable=False,
        index=True,
    )
    status = Column(
        Enum(ScanStatus),  # vulnerable, safe, error
        nullable=False,
        index=True,
    )
    severity = Column(
        Enum(Severity),  # critical, high, medium, low, info
        nullable=False,
        index=True,
    )
    details = Column(Text, nullable=False)
    evidence_json = Column(JSON, nullable=False, default=dict)
    recommendations_json = Column(JSON, nullable=False, default=list)
```

**Campi spiegati:**
- `scan_id`: foreign key verso la tabella scans, indica a quale scansione appartiene questo risultato
- `test_name`: enum che vincola i valori ai tipi di test validi, indicizzato per filtrare per test
- `status`: enum per vulnerable/safe/error, indicizzato per trovare tutte le vulnerabilità
- `severity`: enum per CRITICAL/HIGH/MEDIUM/LOW/INFO, indicizzato per la prioritizzazione
- `details`: descrizione testuale di ciò che è stato trovato
- `evidence_json`: JSON che memorizza codici di risposta, payload, tempi - varia in base al tipo di test
- `recommendations_json`: array di stringhe con i passi di remediation

**Perché le colonne JSON:**
L'evidenza varia in base al tipo di test:
- Test di rate limit: `{"rate_limit_headers": {...}, "bypass_method": "IP spoofing"}`
- Test di SQLi: `{"database_type": "mysql", "payload": "' OR 1=1--", "response_time": "5.23s"}`
- Test di auth: `{"algorithm_variant": "none", "status_code": 200}`

La flessibilità del JSON permette a ogni scanner di memorizzare i dati rilevanti senza modifiche allo schema.

## Architettura di sicurezza

### Threat model

Da cosa ci stiamo proteggendo:

1. **Accesso non autorizzato ai dati delle scansioni** - Gli utenti dovrebbero vedere solo le proprie scansioni. L'attaccante prova a visualizzare la scansione con ID 123 quando ha creato solo la scansione con ID 456. Difesa: il controllo di autorizzazione in `services/scan_service.py:77-80` verifica che `scan.user_id == user_id`.

2. **Furto e replay dei token** - L'attaccante ruba il JWT dal traffico di rete o tramite XSS. Difesa: HTTPS in produzione (imposto da nginx), breve durata del token (24 ore da `config.py:24`), cookie httpOnly (non implementati ma raccomandati).

3. **Tentativi di login con brute force** - L'attaccante prova password comuni contro gli account. Difesa: il rate limiting in `routes/auth.py:49` limita il login a 20/minuto, bcrypt rende lenta la verifica delle password.

4. **SQL injection nei payload dello scanner** - Un utente malevolo crea una scansione con un payload SQLi come URL target sperando di sfruttare il nostro database. Difesa: tutto l'accesso al database usa query parametrizzate tramite SQLAlchemy, mai concatenazione.

5. **Esaurimento delle risorse** - L'attaccante invia ripetutamente scansioni con max_requests=50 per consumare le risorse del backend. Difesa: rate limiting sulla creazione delle scansioni (15/minuto), limiti di timeout sugli scanner, max_requests limitato a 50.

Da cosa NON ci stiamo proteggendo (fuori ambito):

- **Attacchi DDoS** - Il rate limiting a livello applicativo non può fermare flood di rete volumetrici. Richiede difese a livello di infrastruttura (CloudFlare, AWS Shield).
- **Compromissione del database** - Se l'attaccante ottiene accesso diretto al database, può leggere tutti i dati. Richiede hardening dell'infrastruttura, colonne cifrate per i dati sensibili.
- **Server-side request forgery (SSRF)** - Gli scanner fanno richieste verso URL forniti dall'utente. È una funzionalità voluta. Mitigazione: gli scanner girano con accesso di rete limitato, non sulla rete interna.

### Livelli di difesa

Più livelli di sicurezza creano una defense in depth:

```
Layer 1: Network (Nginx)
    ↓ HTTPS, CORS headers, rate limits
Layer 2: Application (FastAPI)
    ↓ JWT validation, endpoint rate limits, input validation
Layer 3: Business Logic (Services)
    ↓ Authorization checks, transaction management
Layer 4: Data Access (Repositories)
    ↓ Parameterized queries, row-level permissions
```

**Perché più livelli?**
Se una difesa fallisce, le altre intercettano l'attacco. Esempio: il rate limit di Nginx viene aggirato tramite IP spoofing, ma il rate limit a livello applicativo (per ID utente) protegge comunque. La validazione JWT viene aggirata in qualche modo, ma il livello service controlla comunque `scan.user_id` prima di restituire i dati.

### Flusso di autenticazione

Ciclo completo di autenticazione JWT:

**1. Registrazione** (`services/auth_service.py:19-41`):
```python
# Hash password with bcrypt
hashed_password = hash_password(user_data.password)
# Bcrypt automatically generates salt, 10 rounds by default

# Store hashed password
user = UserRepository.create_user(
    db=db,
    email=user_data.email,
    hashed_password=hashed_password,
)
```

**2. Login** (`services/auth_service.py:43-71`):
```python
# Verify password
if not verify_password(login_data.password, user.hashed_password):
    raise HTTPException(status_code=401, detail="Invalid email or password")

# Create JWT with expiration
access_token = create_access_token(
    data={"sub": user.email},  # Subject claim
    expires_delta=timedelta(minutes=1440)  # 24 hours
)

return TokenResponse(access_token=access_token, token_type="bearer")
```

**3. Accesso a un endpoint protetto** (`core/dependencies.py:17-49`):
```python
# Extract token from Authorization header
credentials: HTTPAuthorizationCredentials = Depends(security)
# security = HTTPBearer() from fastapi.security

# Decode and verify token
payload = decode_token(credentials.credentials)
email = payload.get("sub")

# Load user from database
user = UserRepository.get_by_email(db, email)
if not user:
    raise HTTPException(status_code=401)

return UserResponse.model_validate(user)
```

**4. Il route handler riceve l'utente autenticato**:
```python
@router.post("/scans/")
async def create_scan(
    current_user: UserResponse = Depends(get_current_user),  # Authenticated
    ...
):
    # current_user is guaranteed to be valid at this point
    return ScanService.run_scan(db, current_user.id, ...)
```

### Strategia di rate limiting

Più implementazioni di rate limit:

**1. A livello Nginx** - Non implementato in dev, ma il nginx di produzione può usare limit_req:
```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
limit_req zone=api_limit burst=20 nodelay;
```

**2. A livello applicativo** - Limiti SlowAPI per endpoint (`backend/factory.py:34-36`):
```python
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

Applicati alle route:
```python
# routes/auth.py:25-28
@router.post("/register", ...)
@limiter.limit(settings.API_RATE_LIMIT_REGISTER)  # "15/minute"
async def register(...):

# routes/auth.py:47-50
@router.post("/login", ...)
@limiter.limit(settings.API_RATE_LIMIT_LOGIN)  # "20/minute"
async def login(...):

# routes/scans.py:23-26
@router.post("/", ...)
@limiter.limit(settings.API_RATE_LIMIT_SCAN)  # "15/minute"
async def create_scan(...):
```

**3. A livello scanner** - Le richieste in uscita verso i target sono distanziate (`base_scanner.py:64-90`):
```python
def _wait_before_request(self, jitter_ms: int | None = None) -> None:
    """
    Implement request spacing to avoid overwhelming target
    """
    required_delay = 1.0 / (self.max_requests / settings.SCANNER_RATE_LIMIT_WINDOW_SECONDS)
    # If max_requests=100 and window=60s, delay = 1.0 / (100/60) = 0.6s between requests
    
    jitter = random.uniform(0, jitter_ms / 1000.0)  # Random variation
    elapsed = time.time() - self.last_request_time
    
    if elapsed < required_delay:
        time.sleep(required_delay - elapsed + jitter)
```

Questo impedisce agli scanner di martellare le API target e di ricevere un ban dell'IP.

## Strategia di storage

### PostgreSQL

**Cosa memorizziamo:**
- Account utente (email, password con hash)
- Metadati delle scansioni (URL target, timestamp)
- Risultati dei test (finding, evidenze, raccomandazioni)

**Perché PostgreSQL:**
- Dati relazionali con foreign key (scans → users, test_results → scans)
- Supporto per colonne JSON per uno storage flessibile delle evidenze
- Transazioni ACID per l'integrità dei dati
- Maturo, ben documentato, ampiamente diffuso

Alternative considerate:
- MongoDB: migliore per dati senza schema, ma abbiamo relazioni chiare e beneficiamo dei vincoli di foreign key
- SQLite: setup più semplice, ma non gestisce bene le scritture concorrenti (più scansioni in esecuzione contemporaneamente)

**Progettazione dello schema:**

```sql
-- Automatically generated by SQLAlchemy from models
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX ix_users_email ON users(email);

CREATE TABLE scans (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_url VARCHAR(2048) NOT NULL,
    scan_date TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX ix_scans_user_id ON scans(user_id);

CREATE TABLE test_results (
    id SERIAL PRIMARY KEY,
    scan_id INTEGER NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
    test_name VARCHAR NOT NULL,  -- Enum: rate_limit, auth, sqli, idor
    status VARCHAR NOT NULL,      -- Enum: vulnerable, safe, error
    severity VARCHAR NOT NULL,    -- Enum: critical, high, medium, low, info
    details TEXT NOT NULL,
    evidence_json JSON NOT NULL,
    recommendations_json JSON NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX ix_test_results_scan_id ON test_results(scan_id);
CREATE INDEX ix_test_results_test_name ON test_results(test_name);
CREATE INDEX ix_test_results_status ON test_results(status);
CREATE INDEX ix_test_results_severity ON test_results(severity);
```

**Indici spiegati:**
- `users.email`: lookup di login veloci (WHERE email = ?)
- `scans.user_id`: filtraggio veloce (WHERE user_id = ?)
- `test_results.scan_id`: join veloci (JOIN scans WHERE scan_id = ?)
- `test_results.status`: query veloci sulle vulnerabilità (WHERE status = 'vulnerable')
- `test_results.severity`: filtraggio veloce per severity (WHERE severity = 'critical')

### Connection pooling

Le connessioni al database sono costose da creare. SQLAlchemy mantiene un pool:

```python
# core/database.py:12-17
engine = create_engine(
    settings.DATABASE_URL,
    pool_pre_ping=True,  # Verify connections before use (handles DB restarts)
    echo=settings.DEBUG,  # Log all SQL when DEBUG=True
)
```

Dimensione predefinita del pool: 5 connessioni, overflow: 10 (fino a 15 in totale).

Ciclo di vita della sessione gestito tramite dipendenza:

```python
# core/database.py:28-36
def get_db() -> Generator[Session, None, None]:
    """
    FastAPI dependency for database sessions
    """
    db = SessionLocal()
    try:
        yield db  # Provide to route handler
    finally:
        db.close()  # Always close, even if exception
```

## Configurazione

### Variabili d'ambiente

Tutta la configurazione vive in `.env` e `config.py`:

```python
# config.py:14-69
class Settings(BaseSettings):
    # Application
    APP_NAME: str = "API Security Tester"
    VERSION: str = "1.0.0"
    DEBUG: bool = False
    
    # Database
    DATABASE_URL: str  # Required, no default
    
    # Security - JWT
    SECRET_KEY: str  # Required, MUST be random in production
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440
    
    # Rate limiting
    API_RATE_LIMIT_LOGIN: str = "20/minute"
    API_RATE_LIMIT_REGISTER: str = "15/minute"
    API_RATE_LIMIT_SCAN: str = "15/minute"
    
    # Scanner limits
    SCANNER_MAX_CONCURRENT_REQUESTS: int = 50
    SCANNER_CONNECTION_TIMEOUT: int = 180
    DEFAULT_MAX_REQUESTS: int = 100
```

**Perché una configurazione centralizzata:**
- Unica fonte di verità per tutte le costanti
- Validazione dei tipi con Pydantic
- Facile da cambiare senza toccare il codice
- Valori diversi per dev/test/prod

**Strategia di configurazione:**
Lo sviluppo usa il file `.env` caricato da Docker Compose. La produzione usa variabili d'ambiente impostate direttamente in Docker, Kubernetes o sulla piattaforma cloud.

Impostazioni critiche che devono essere cambiate per la produzione:
- `SECRET_KEY` - Genera con `openssl rand -hex 32`
- `DEBUG` - Deve essere `false`
- `DATABASE_URL` - Database di produzione, non localhost
- `CORS_ORIGINS` - Dominio effettivo del frontend, non `http://localhost`

## Considerazioni sulle prestazioni

### Colli di bottiglia

Dove questo sistema rallenta sotto carico:

1. **Richieste HTTP dello scanner** - Ogni scansione fa 10-100 richieste HTTP verso API esterne. Se il target è lento (5s di tempo di risposta), le scansioni richiedono minuti. Non si può parallelizzare molto senza sovraccaricare i target. Mitigato dai limiti di timeout (180s da `config.py:51`).

2. **Query al database con relazioni** - Caricare `scan.test_results` genera query N+1 se non si fa eager loading. Con 10 scansioni e 4 risultati ciascuna = 41 query (1 per le scansioni + 40 per i risultati). Risolto con `joinedload()` nei repository:

```python
# repositories/scan_repository.py:50-59
return (
    db.query(Scan)
    .options(joinedload(Scan.test_results))  # Single query with JOIN
    .filter(Scan.user_id == user_id)
    .all()
)
```

3. **Hashing delle password al login** - Bcrypt è intenzionalmente lento (previene il brute force). Ogni login richiede ~100ms. Sotto carico, l'autenticazione diventa un collo di bottiglia. Mitigato dal rate limiting dei tentativi di login e dall'uso del caching (non implementato, ma si potrebbe aggiungere Redis per i token di sessione).

### Ottimizzazioni

Cosa abbiamo fatto per renderlo più veloce:

- **Request pooling** - `BaseScanner` riutilizza `requests.Session()` che mantiene pool di connessioni verso i target. Evita l'overhead dell'handshake TCP a ogni richiesta.

- **Indici del database** - Le foreign key e le colonne interrogate più spesso (email, user_id, status) sono indicizzate. Le query che farebbero table scan diventano lookup su indice.

- **Paginazione delle risposte** - Le query sull'elenco delle scansioni usano LIMIT/OFFSET per evitare di caricare migliaia di record:

```python
# repositories/scan_repository.py:50-71
@staticmethod
def get_by_user(
    db: Session,
    user_id: int,
    skip: int = 0,
    limit: int | None = None  # Default 100 from config
) -> list[Scan]:
    if limit is None:
        limit = settings.DEFAULT_PAGINATION_LIMIT
    
    return (
        db.query(Scan)
        .offset(skip)
        .limit(limit)  # Only load requested page
        .all()
    )
```

- **Colonne enum** - `test_name`, `status`, `severity` usano ENUM di Postgres, non stringhe. Storage più piccolo, confronti più veloci, validità garantita.

### Scalabilità

**Scalabilità verticale** (più CPU/RAM su un singolo server):
- Database: aumentare `max_connections` nella configurazione di Postgres
- Backend: eseguire più worker gunicorn (4 worker nel Dockerfile di produzione)
- Frontend: Nginx è già efficiente, il collo di bottiglia è improbabile qui

Limiti attuali con un singolo server:
- Il database può gestire ~100 connessioni concorrenti
- Il backend con 4 worker gestisce ~400 richieste concorrenti
- Gli scanner sono il vero limite - ogni scansione è di lunga durata (30-60s)

**Scalabilità orizzontale** (più server):
Sfide:
- Gli scanner sono stateless, possono girare su qualsiasi istanza del backend ✓
- Il database richiede una strategia di connection pooling (PgBouncer)
- Serve uno stato di sessione condiviso (Redis) oppure ci si attiene a JWT (stateless) ✓
- È necessario un load balancer (nginx, AWS ALB)

Cosa deve cambiare:
- Aggiungere un load balancer davanti al backend
- Configurare uno session store condiviso o affidarsi solo a JWT
- Gestione del connection pool del database (PgBouncer)
- Considerare una coda di task asincrona (Celery, RQ) per le scansioni di lunga durata

## Decisioni di design

### Decisione 1: scanner sincroni nonostante FastAPI sia async

**Cosa abbiamo scelto:**
Gli scanner usano la libreria sincrona `requests`, non `httpx` async o `aiohttp`.

**Alternative considerate:**
- Client HTTP async `httpx` - Potrebbe eseguire tutti i test in modo concorrente
- `aiohttp` - Benefici simili, API diversa

**Compromessi:**
Pro del sync:
- Codice più semplice, più facile ragionare sui tempi
- Il rilevamento di SQLi time-based richiede un controllo preciso dei tempi
- Il calcolo del timing di baseline richiede richieste sequenziali
- La libreria standard `requests` è collaudata sul campo

Contro del sync:
- Non si possono eseguire più test in modo concorrente all'interno di una scansione
- Blocca l'event loop (mitigato eseguendo in un thread pool)
- Più lento per scansioni con molti test

**Perché abbiamo fatto questa scelta:**
Accuratezza prima della velocità. Il rilevamento di SQL injection blind time-based (`sqli_scanner.py:183-251`) richiede:
1. Stabilire il tempo di risposta di baseline (più campioni)
2. Inviare il payload di delay
3. Misurare se la risposta è più lenta del valore atteso

La concorrenza async introdurrebbe rumore nei tempi. Un ritardo di 5,1s contro 5,3s potrebbe essere jitter di rete, non SQLi. Richieste sequenziali con spaziatura controllata danno segnali più puliti.

### Decisione 2: pattern repository invece di Active Record

**Cosa abbiamo scelto:**
Pattern repository - `UserRepository.get_by_email(db, email)` invece di `User.find_by_email(email)`.

**Alternative considerate:**
- Active Record (stile Django) - I modelli hanno metodi di classe per le query
- Data Mapper (SQL raw) - Scrivere direttamente stringhe SQL

**Compromessi:**
Pro del repository:
- Separazione chiara: i modelli definiscono la struttura, i repository definiscono le query
- Testabile: mock dei repository nei unit test
- Flessibile: si può cambiare ORM senza modificare il codice dei service

Contro del repository:
- Più file, più navigazione
- Livello di astrazione aggiuntivo
- Può sembrare eccessivo per semplici CRUD

**Perché abbiamo fatto questa scelta:**
Testabilità e manutenibilità. Service come `AuthService.login_user()` chiamano `UserRepository.get_by_email()`. Nei test, si fa il mock del repository per restituire un utente finto senza toccare il database.

Se in seguito migriamo da SQLAlchemy a un altro ORM, cambiamo solo le implementazioni dei repository. I service restano invariati.

### Decisione 3: JWT senza refresh token

**Cosa abbiamo scelto:**
Un singolo JWT di lunga durata (24 ore), nessun meccanismo di refresh token.

**Alternative considerate:**
- Access token brevi (15 min) + refresh token (30 giorni)
- Autenticazione basata su sessione con storage lato server

**Compromessi:**
Pro dell'approccio attuale:
- Implementazione più semplice, nessun endpoint di refresh
- Stateless - nessuno storage di sessione necessario
- Funziona subito su più istanze del backend

Contro dell'approccio attuale:
- Non si possono invalidare i token prima della scadenza
- Se il token viene rubato, l'attaccante ha 24 ore di accesso
- Nessun modo di forzare il logout su tutti i dispositivi

**Perché abbiamo fatto questa scelta:**
Semplicità per un progetto educativo. Aggiungere i refresh token richiede:
- Storage dei refresh token (database o Redis)
- Endpoint di refresh con logica di rotazione
- Tracciamento della revoca dei token
- Gestione dei token nel frontend più complessa

Per un'app di produzione, implementeresti i refresh token. Per imparare come funziona JWT, così è più chiaro.

### Decisione 4: Docker Compose per lo sviluppo locale

**Cosa abbiamo scelto:**
Eseguire tutto in container Docker con `dev.compose.yml`, anche per lo sviluppo locale.

**Alternative considerate:**
- Postgres locale + Python locale + Node locale (senza Docker)
- Docker per i servizi, locale per lo sviluppo
- Kubernetes in locale (minikube, kind)

**Compromessi:**
Pro di Docker Compose:
- Ambiente identico per tutti gli sviluppatori
- Avvio dell'intero stack con un solo comando
- L'hot reload funziona comunque con i volume mount
- L'architettura di produzione corrisponde a quella di dev (Docker in entrambi)

Contro di Docker Compose:
- File system più lento su Mac (volume mount)
- Utilizzo di risorse aggiuntivo (overhead dei container)
- Curva di apprendimento per il debug di Docker

**Perché abbiamo fatto questa scelta:**
Il "funziona sulla mia macchina" viene eliminato. Ogni sviluppatore ottiene Postgres 16, Python 3.11, Node 20 indipendentemente dal proprio OS host. Il nuovo membro del team esegue `docker compose up` ed è pronto.

I volume mount preservano l'hot reload:
```yaml
# dev.compose.yml:33-35
volumes:
  - ./backend:/app  # Maps local backend/ to container /app
  # Changes to backend/*.py trigger uvicorn reload
```

## Architettura di deployment

Il deployment di produzione usa container ottimizzati:

```
                    ┌──────────────────┐
Internet ────────>  │  Nginx (Port 80) │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  Backend (8000)  │
                    │  Gunicorn        │
                    │  4 workers       │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  PostgreSQL      │
                    │  (internal only) │
                    └──────────────────┘
```

**Componenti:**

**Container Nginx** - Costruito da `conf/docker/prod/vite.docker`:
- Build multi-stage: compila React, poi serve con nginx
- Serve i file statici da `/usr/share/nginx/html`
- Fa da proxy di `/api` verso il backend
- Compressione Gzip, header di caching

**Container Backend** - Costruito da `conf/docker/prod/fastapi.docker`:
- Esegue gunicorn con 4 worker uvicorn
- Utente non-root per sicurezza
- Nessun volume mount (codice incorporato nell'image)

**Container Database** - Postgres 16 Alpine:
- Non esposto all'host in produzione
- I dati persistono in un volume Docker

**Infrastruttura:**
Setup di produzione minimale:
- Singolo VPS (DigitalOcean Droplet, AWS EC2)
- Orchestrazione con Docker Compose
- SSL tramite Let's Encrypt (certbot) o proxy Cloudflare

Scalare oltre il singolo server:
- Backend: più istanze dietro un load balancer
- Database: read replica, connection pooling
- File statici: CDN (CloudFront, Cloudflare)

## Strategia di gestione degli errori

### Tipi di errore

1. **Errori di validazione** (400) - Pydantic intercetta l'input errato:
```python
# schemas/user_schemas.py:26-38
@field_validator("password")
@classmethod
def validate_password_strength(cls, v: str) -> str:
    if not re.search(r"[A-Z]", v):
        raise ValueError("Password must contain uppercase letter")
    # Pydantic converts to HTTP 422 automatically
```

2. **Errori di autenticazione** (401) - JWT non valido o scaduto:
```python
# core/dependencies.py:33-36
if email is None:
    raise HTTPException(
        status_code=401,
        detail="Invalid authentication credentials"
    )
```

3. **Errori di autorizzazione** (403) - Utente valido, risorsa sbagliata:
```python
# services/scan_service.py:77-80
if scan.user_id != user_id:
    raise HTTPException(
        status_code=403,
        detail="Not authorized to access this scan"
    )
```

4. **Errori di risorsa non trovata** (404) - La risorsa non esiste:
```python
# services/scan_service.py:73-74
if not scan:
    raise HTTPException(status_code=404, detail="Scan not found")
```

5. **Errori dello scanner** - Intercettati e restituiti come `status="error"`:
```python
# services/scan_service.py:52-65
try:
    scanner = scanner_class(...)
    result = scanner.scan()
    results.append(result)
except Exception as e:
    results.append(
        TestResultCreate(
            test_name=test_type,
            status="error",
            severity="info",
            details=f"Scanner error: {str(e)}",
            ...
        )
    )
```

### Meccanismi di recupero

**Perdita della connessione al database:**
- Rilevamento: `pool_pre_ping=True` testa le connessioni prima dell'uso
- Risposta: SQLAlchemy si riconnette automaticamente
- Recupero: la transazione fallita viene annullata con rollback, la richiesta successiva ottiene una nuova connessione

**Timeout dello scanner:**
- Rilevamento: eccezione `requests.Timeout` dopo `SCANNER_CONNECTION_TIMEOUT` secondi
- Risposta: retry con backoff esponenziale (fino a 3 volte)
- Recupero: se tutti i retry falliscono, restituisce un risultato di errore (la scansione continua con gli altri test)

**Rate limit superato (429 dal target):**
- Rilevamento: status code HTTP 429 nella risposta dello scanner
- Risposta: legge l'header `Retry-After`, attende la durata specificata
- Recupero: ritenta la richiesta dopo l'attesa

Codice da `base_scanner.py:92-156`:
```python
def make_request(self, method: str, endpoint: str, **kwargs) -> requests.Response:
    retry_count = 0
    backoff_factor = 2.0
    
    while retry_count <= settings.DEFAULT_RETRY_COUNT:
        try:
            response = self.session.request(method, url, **kwargs)
            
            if response.status_code == 429:
                retry_after = response.headers.get("Retry-After", "60")
                wait_time = int(retry_after) if retry_after.isdigit() else 60
                time.sleep(wait_time)
                retry_count += 1
                continue  # Try again
            
            if response.status_code >= 500:  # Server error
                wait_time = backoff_factor ** retry_count
                time.sleep(wait_time)
                retry_count += 1
                continue
            
            return response  # Success
        
        except (requests.Timeout, requests.ConnectionError):
            if retry_count < settings.DEFAULT_RETRY_COUNT:
                wait_time = backoff_factor ** retry_count
                time.sleep(wait_time)
                retry_count += 1
            else:
                raise  # Give up after retries exhausted
```

## Estensibilità

### Dove aggiungere funzionalità

Vuoi aggiungere un nuovo test di sicurezza (ad es. rilevamento XSS)? Ecco il procedimento:

**1. Crea lo scanner** in `backend/scanners/xss_scanner.py`:
```python
from .base_scanner import BaseScanner
from core.enums import TestType, ScanStatus, Severity

class XSSScanner(BaseScanner):
    def scan(self) -> TestResultCreate:
        # Test for reflected XSS
        test_result = self._test_reflected_xss()
        
        if test_result["vulnerable"]:
            return TestResultCreate(
                test_name=TestType.XSS,  # Need to add to enum
                status=ScanStatus.VULNERABLE,
                severity=Severity.HIGH,
                details="Reflected XSS detected",
                evidence_json=test_result,
                recommendations_json=[...]
            )
        
        return TestResultCreate(test_name=TestType.XSS, status=ScanStatus.SAFE, ...)
```

**2. Aggiungi all'enum** in `backend/core/enums.py:19-25`:
```python
class TestType(str, Enum):
    RATE_LIMIT = "rate_limit"
    AUTH = "auth"
    SQLI = "sqli"
    IDOR = "idor"
    XSS = "xss"  # New test type
```

**3. Registra nel service** in `backend/services/scan_service.py:32-37`:
```python
scanner_mapping: dict[TestType, type[BaseScanner]] = {
    TestType.RATE_LIMIT: RateLimitScanner,
    TestType.AUTH: AuthScanner,
    TestType.SQLI: SQLiScanner,
    TestType.IDOR: IDORScanner,
    TestType.XSS: XSSScanner,  # Register new scanner
}
```

**4. Aggiorna le costanti del frontend** in `frontend/src/config/constants.ts:44-51`:
```typescript
export const SCAN_TEST_TYPES = {
  RATE_LIMIT: 'rate_limit',
  AUTH: 'auth',
  SQLI: 'sqli',
  IDOR: 'idor',
  XSS: 'xss',  // Add new test type
} as const;

export const TEST_TYPE_LABELS: Record<ScanTestType, string> = {
  // ... existing
  [SCAN_TEST_TYPES.XSS]: 'Cross-Site Scripting',
};
```

Non servono modifiche a:
- Schema del database (test_name è un enum, le migrazioni si aggiornano automaticamente)
- Route (passano semplicemente i tipi di test)
- Repository (memorizzano qualsiasi tipo di test venga inviato)

## Limitazioni

Limitazioni architetturali attuali:

1. **Nessuna coda per le scansioni** - Le scansioni girano in modo sincrono nel request handler. Se 10 utenti inviano scansioni contemporaneamente, tutti si bloccano sulle richieste HTTP degli scanner. La soluzione richiede una coda di task asincrona (Celery con Redis, o RQ).

2. **Nessun progresso della scansione in tempo reale** - Il frontend invia la scansione e attende la risposta completa. Le scansioni lunghe (2 minuti) non mostrano alcun progresso. La soluzione richiede una connessione WebSocket o il polling per gli aggiornamenti di progresso.

3. **Un solo target per scansione** - Non si possono scansionare più URL in una singola operazione. La soluzione richiede un ciclo nel livello service e una UI per più input target.

4. **Nessun confronto storico** - Non si possono confrontare i risultati delle scansioni nel tempo ("Era vulnerabile la settimana scorsa?"). La soluzione richiede query aggiuntive e una UI per la visualizzazione dei trend.

5. **Concorrenza limitata negli scanner** - I test girano in sequenza all'interno di una scansione. Si potrebbero eseguire tutti e 4 i test contemporaneamente, ma abbiamo scelto di non farlo per accuratezza dei tempi. Compromesso tra velocità e precisione.

Questi non sono bug: sono compromessi consapevoli. Risolverli richiederebbe modifiche architetturali significative.

## Confronto con sistemi simili

### Burp Suite

Come ci differenziamo:
- Burp è un proxy, noi siamo uno scanner standalone
- Burp ha un'app desktop con GUI, noi siamo basati sul web
- Burp è completo (centinaia di test), noi ci concentriamo su 4 vulnerabilità principali

Perché abbiamo fatto scelte diverse:
Focus educativo. Burp è per pentester professionisti. Questo progetto insegna come funzionano gli scanner implementando tu stesso la logica di base.

### OWASP ZAP

Come ci differenziamo:
- ZAP fa scansione passiva + attiva, noi solo attiva
- ZAP scopre automaticamente gli endpoint, noi testiamo gli URL forniti
- ZAP si integra con CI/CD, noi siamo standalone

Perché abbiamo fatto scelte diverse:
Semplicità. ZAP è potente ma complesso. Questo progetto mostra i fondamentali senza funzionalità travolgenti.

## Evoluzione

### Design versione 1.0 (attuale)

Il design iniziale si è concentrato su:
- Quattro tipi principali di vulnerabilità
- Scanner sincroni per accuratezza
- Pattern repository per una separazione pulita
- Sviluppo Docker-first

### Miglioramenti futuri

Modifiche architetturali pianificate:

1. **Coda di task asincrona** - Spostare la scansione su worker in background
   - Perché: API non bloccante, migliore scalabilità
   - Cosa abilita: progresso in tempo reale, scansioni pianificate

2. **Sistema di plugin** - Caricare gli scanner dinamicamente
   - Perché: estensibilità senza modificare il core
   - Cosa abilita: contributi della community, test personalizzati

3. **Generazione di report** - Export PDF/HTML dei risultati
   - Perché: condividere i finding con i team
   - Cosa abilita: documentazione professionale

4. **Notifiche webhook** - Avvisare quando le scansioni terminano
   - Perché: integrazione con altri strumenti
   - Cosa abilita: notifiche Slack/email, integrazione CI/CD

## Riferimento ai file chiave

Mappa rapida di dove trovare le cose:

- `backend/factory.py` - Application factory, setup dei middleware, registrazione delle route
- `backend/config.py` - Tutte le variabili d'ambiente e la configurazione
- `backend/core/database.py` - Engine del database e gestione delle sessioni
- `backend/core/security.py` - Creazione dei JWT, hashing delle password, validazione dei token
- `backend/core/dependencies.py` - Dipendenze FastAPI (auth, database)
- `backend/models/` - Modelli SQLAlchemy (User, Scan, TestResult)
- `backend/repositories/` - Funzioni di query del database
- `backend/services/` - Orchestrazione della logica di business
- `backend/routes/` - Endpoint API
- `backend/scanners/base_scanner.py` - Funzionalità comuni degli scanner
- `backend/scanners/*_scanner.py` - Singoli test di vulnerabilità
- `frontend/src/hooks/` - Hook React Query per le chiamate API
- `frontend/src/services/` - Funzioni del client API
- `frontend/src/store/` - Gestione dello stato con Zustand
- `conf/nginx/` - Configurazione del reverse proxy Nginx
- `compose.yml` - Docker Compose di produzione
- `dev.compose.yml` - Sviluppo con volume mount

## Prossimi passi

Ora che hai compreso l'architettura:

1. Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per la walkthrough del codice - scopri come ogni scanner rileva le vulnerabilità, come funzionano i flussi di autenticazione e come i dati si muovono attraverso i livelli
2. Prova a modificare gli scanner - cambia i payload SQLi, regola le soglie di timing, aggiungi nuova logica di rilevamento per capire i dettagli dell'implementazione
