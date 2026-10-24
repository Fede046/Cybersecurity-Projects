# Le funzionalità di sicurezza spiegate

Questo documento spiega le funzionalità di sicurezza implementate nella Bug Bounty Platform, le minacce che mitigano e le best practice per uno sviluppo sicuro.

---

## Indice

1. [Panoramica sulla sicurezza](#panoramica-sulla-sicurezza)
2. [Autenticazione](#autenticazione)
3. [Autorizzazione](#autorizzazione)
4. [Sicurezza delle password](#sicurezza-delle-password)
5. [Sicurezza dei token](#sicurezza-dei-token)
6. [Validazione dell'input](#validazione-dellinput)
7. [Rate limiting](#rate-limiting)
8. [Vulnerabilità comuni prevenute](#vulnerabilità-comuni-prevenute)
9. [Best practice di sicurezza](#best-practice-di-sicurezza)

---

## Panoramica sulla sicurezza

La sicurezza è implementata in **più livelli**:

```
┌──────────────────────────────────────────────────────────┐
│  Layer 1: Input Validation (Pydantic)                    │
│  - Type checking                                          │
│  - String length limits                                   │
│  - Email/URL format validation                            │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  Layer 2: Authentication (JWT)                            │
│  - Token-based auth                                       │
│  - Short-lived access tokens (15 min)                     │
│  - Token rotation                                         │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  Layer 3: Authorization (RBAC)                            │
│  - Role-based access control                              │
│  - Resource ownership checks                              │
│  - Admin-only endpoints                                   │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  Layer 4: Rate Limiting                                   │
│  - 100 req/min default                                    │
│  - 20 req/min for auth endpoints                          │
│  - Per-IP tracking                                        │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  Layer 5: Secure Storage                                  │
│  - Argon2id password hashing                              │
│  - Hashed refresh tokens                                  │
│  - No plaintext secrets                                   │
└──────────────────────────────────────────────────────────┘
```

**Defense in Depth:** se un livello fallisce, gli altri continuano a proteggere.

---

## Autenticazione

### Autenticazione basata su token JWT

**Perché JWT?**
- Stateless (nessuno storage di sessione sul server)
- Scalabile (funziona su più server)
- Contiene le informazioni dell'utente (nessun lookup sul database a ogni richiesta)

**Struttura del token:**

```json
{
  "sub": "018d3f54-8c3a-7000-a234-56789abcdef0",  // user_id
  "role": "user",
  "token_version": 1,
  "exp": 1704123456,  // expires in 15 minutes
  "iat": 1704122556   // issued at
}
```

**Header:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Sistema a due token

**Access Token:**
- Di breve durata (15 minuti)
- Inviato con ogni richiesta API
- Non può essere revocato (stateless)
- Se rubato, l'attaccante ha solo 15 minuti

**Refresh Token:**
- Di lunga durata (7 giorni)
- Usato solo per ottenere un nuovo access token
- Memorizzato nel database (può essere revocato)
- Monouso (ruotato a ogni refresh)

**Perché due token?**
- Compromesso: sicurezza vs UX
- Access token: veloce (nessun lookup sul DB), ma non revocabile
- Refresh token: lento (lookup sul DB), ma revocabile

### Flusso di login

```
┌──────────────────────────────────────────────────────────┐
│  1. User submits email + password                         │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  2. Verify password (Argon2id)                            │
│     - Timing-safe comparison                              │
│     - No early returns                                    │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  3. Generate access token (JWT)                           │
│     - Signed with SECRET_KEY                              │
│     - Expires in 15 minutes                               │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  4. Generate refresh token                                │
│     - Random 32-byte string                               │
│     - SHA-256 hash stored in DB                           │
│     - Original token sent to client                       │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  5. Return both tokens                                    │
│     { access_token, refresh_token }                       │
└──────────────────────────────────────────────────────────┘
```

**Codice:**

```python
from pwdlib import PasswordHash
import jwt
from datetime import datetime, timedelta
import secrets

async def login(email: str, password: str) -> tuple[str, str]:
    user = await user_repo.find_by_email(email)
    if not user:
        raise InvalidCredentialsError()

    if not verify_password(password, user.hashed_password):
        raise InvalidCredentialsError()

    access_token = create_access_token(user)
    refresh_token = create_refresh_token()

    await refresh_token_repo.create(
        user_id=user.id,
        token_hash=hash_token(refresh_token),
        expires_at=datetime.utcnow() + timedelta(days=7),
    )

    return access_token, refresh_token
```

### Flusso di refresh del token

```
┌──────────────────────────────────────────────────────────┐
│  1. Access token expires (after 15 min)                   │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  2. Client sends refresh token                            │
│     POST /api/v1/auth/refresh                             │
│     { refresh_token }                                     │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  3. Verify refresh token                                  │
│     - Hash token and lookup in DB                         │
│     - Check not expired                                   │
│     - Check not revoked                                   │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  4. Token rotation                                        │
│     - DELETE old refresh token                            │
│     - CREATE new refresh token                            │
│     - Store new hash in DB                                │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  5. Generate new access token                             │
│     - Increment token_version if needed                   │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  6. Return new tokens                                     │
│     { access_token, refresh_token }                       │
└──────────────────────────────────────────────────────────┘
```

**Perché ruotare i refresh token?**
- Ogni refresh token è monouso
- Se rubato, l'attaccante può usarlo una sola volta
- Il token originale diventa invalido
- L'utente nota di essere stato disconnesso

### Rilevamento di attacchi tramite rotazione dei token

**Scenario:** l'attaccante ruba il refresh token

```
Time 0: Legitimate user has token A
Time 1: Attacker steals token A
Time 2: User refreshes → gets token B (token A deleted)
Time 3: Attacker tries to use token A → DETECTED!
```

**Meccanismo di rilevamento:**

```python
async def refresh_token(token: str) -> tuple[str, str]:
    token_hash = hash_token(token)
    stored_token = await refresh_token_repo.find_by_hash(token_hash)

    if not stored_token:
        family_tokens = await refresh_token_repo.find_by_family_id(
            extract_family_id(token)
        )
        if family_tokens:
            await refresh_token_repo.delete_all(family_tokens)
            raise TokenReuseDetectedError()
        raise InvalidTokenError()

    # ...
```

Se viene usato un token eliminato, noi:
1. Troviamo tutti i token della stessa famiglia
2. Li eliminiamo tutti (logout da tutti i dispositivi)
3. Notifichiamo all'utente l'attività sospetta

### Versionamento dei token

**Problema:** come invalidare istantaneamente tutti i token?

**Soluzione:** un campo token version

```python
class User(Base):
    token_version: Mapped[int] = mapped_column(default=0)

def create_access_token(user: User) -> str:
    payload = {
        "sub": str(user.id),
        "token_version": user.token_version,  # ← Include version
        "exp": datetime.utcnow() + timedelta(minutes=15),
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")

async def verify_access_token(token: str) -> User:
    payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
    user = await user_repo.get_by_id(payload["sub"])

    if user.token_version != payload["token_version"]:  # ← Check version
        raise InvalidTokenError("Token invalidated")

    return user
```

**Quando incrementare token_version:**
- L'utente cambia la password
- L'utente segnala il furto di un token
- L'admin reimposta le sessioni dell'utente

```python
def change_password(user: User, new_password: str):
    user.hashed_password = hash_password(new_password)
    user.token_version += 1  # ← Invalidate all tokens
    await session.commit()
```

---

## Autorizzazione

### Role-Based Access Control (RBAC)

**Ruoli:**
- `USER` - Ricercatore normale (può inviare report)
- `COMPANY` - Proprietario del programma (può creare programmi, fare triage dei report)
- `ADMIN` - Amministratore della piattaforma (può accedere al pannello di amministrazione)

**Implementazione:**

```python
from enum import StrEnum

class UserRole(StrEnum):
    USER = "user"
    COMPANY = "company"
    ADMIN = "admin"
    UNKNOWN = "unknown"  # For guest users

class RequireRole:
    def __init__(self, *allowed_roles: UserRole):
        self.allowed_roles = allowed_roles

    async def __call__(self, user: CurrentUser) -> User:
        if user.role not in self.allowed_roles:
            raise ForbiddenError()
        return user

# Usage
AdminOnly = Annotated[User, Depends(RequireRole(UserRole.ADMIN))]

@router.get("/admin/stats")
async def get_stats(user: AdminOnly) -> StatsResponse:
    # Only admins can access
    ...
```

### Proprietà delle risorse

**Esempio:** solo il proprietario del programma può aggiornare il programma

```python
@router.patch("/programs/{slug}")
async def update_program(
    slug: str,
    data: ProgramUpdate,
    user: CurrentUser,
    session: DatabaseSession,
) -> ProgramResponse:
    program = await program_repo.get_by_slug(slug)
    if not program:
        raise NotFoundError()

    if program.company_id != user.id and user.role != UserRole.ADMIN:
        raise ForbiddenError("You don't own this program")

    await program_repo.update(program, data)
    await session.commit()
    return ProgramResponse.from_orm(program)
```

**Matrice di autorizzazione:**

| Azione | USER | COMPANY | ADMIN |
|--------|------|---------|-------|
| Inviare un report | ✓ (a qualsiasi programma) | ✓ | ✓ |
| Creare un programma | ✓ | ✓ | ✓ |
| Fare triage di un report | ✗ | ✓ (propri programmi) | ✓ |
| Eliminare un programma | ✗ | ✓ (propri programmi) | ✓ |
| Vedere il pannello admin | ✗ | ✗ | ✓ |

---

## Sicurezza delle password

### Hashing con Argon2id

**Perché Argon2id?**
- Vincitore della Password Hashing Competition (2015)
- Memory-hard (resiste agli attacchi GPU/ASIC)
- Configurabile (il costo può essere aumentato nel tempo)
- Resistente ai side-channel

**Confronto:**

| Algoritmo | Sicurezza | Velocità | Memoria | Adozione |
|-----------|----------|-------|--------|----------|
| MD5 | ✗✗✗ | Veloce | Bassa | Deprecato |
| SHA-256 | ✗ | Veloce | Bassa | Non usare per le password |
| bcrypt | ✓✓ | Lento | Media | Buono |
| Argon2id | ✓✓✓ | Più lento | Alta | Il migliore |

**Implementazione:**

```python
from pwdlib import PasswordHash
from pwdlib.hashers.argon2 import Argon2Hasher

password_hash = PasswordHash((Argon2Hasher(),))

def hash_password(password: str) -> str:
    return password_hash.hash(password)
    # → $argon2id$v=19$m=65536,t=3,p=4$...

def verify_password(password: str, hash: str) -> bool:
    try:
        return password_hash.verify(password, hash)
    except Exception:
        return False  # Invalid hash or password
```

**Formato dell'hash:**

```
$argon2id$v=19$m=65536,t=3,p=4$salt$hash
│         │   │ │      │  │
│         │   │ │      │  └─ Parallelism (4 threads)
│         │   │ │      └─ Time cost (3 iterations)
│         │   │ └─ Memory cost (64 MiB)
│         │   └─ Version 19
│         └─ Variant (argon2id)
└─ Algorithm
```

### Verifica delle password resistente ai timing attack

**Codice vulnerabile:**

```python
def verify_password_BAD(password: str, hash: str) -> bool:
    if hash_password(password) == hash:
        return True
    else:
        return False  # Early return reveals info!
```

**Timing attack:**
- L'attaccante misura il tempo di risposta
- La password corretta richiede più tempo (lookup sul database, creazione della sessione)
- La password errata ritorna immediatamente
- L'attaccante può fare brute force più velocemente

**Codice sicuro:**

```python
def verify_password_GOOD(password: str, hash: str) -> bool:
    result = password_hash.verify(password, hash)
    # Always takes same time (Argon2id is constant-time)
    return result
```

### Requisiti delle password

**Applicati tramite Pydantic:**

```python
from pydantic import BaseModel, Field, field_validator

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=128)

    @field_validator("password")
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain uppercase letter")
        if not any(c.islower() for c in v):
            raise ValueError("Password must contain lowercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain digit")
        return v
```

**Requisiti:**
- Minimo 8 caratteri
- Massimo 128 caratteri (previene il DoS tramite password molto lunghe)
- Almeno una lettera maiuscola
- Almeno una lettera minuscola
- Almeno una cifra

---

## Sicurezza dei token

### Memorizzazione dei refresh token

**Non memorizzare mai token in chiaro!**

```python
# BAD - plaintext token in database
refresh_token = secrets.token_urlsafe(32)
await db.execute(
    "INSERT INTO refresh_tokens (token) VALUES (?)",
    (refresh_token,)
)

# GOOD - hash token before storing
refresh_token = secrets.token_urlsafe(32)
token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()
await db.execute(
    "INSERT INTO refresh_tokens (token_hash) VALUES (?)",
    (token_hash,)
)
```

**Perché fare l'hash?**
- Se il database viene compromesso, l'attaccante non può usare i token
- Deve possedere il token originale per autenticarsi
- Simile all'hashing delle password

### Chiave segreta JWT

**Genera un secret robusto:**

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
# → x8J9kL2mN4pQ5rS7tU8vW1xY3zA5bC7dE9fG1hI3jK5m
```

**Requisiti:**
- Minimo 32 byte (256 bit)
- Casuale (usa il modulo `secrets`, non `random`)
- Non fare mai commit su Git
- Ruotare periodicamente

**Variabile d'ambiente:**

```bash
# .env
SECRET_KEY=x8J9kL2mN4pQ5rS7tU8vW1xY3zA5bC7dE9fG1hI3jK5m
```

**Se la chiave è compromessa:**
- L'attaccante può falsificare token JWT
- L'attaccante può impersonare qualsiasi utente
- Occorre ruotare la chiave e invalidare tutti i token

### Scadenza dei token

**Access token: 15 minuti**
- Bilancia la sicurezza (finestra breve se rubato) e la UX (refresh non troppo frequente)
- Può essere più breve (5 min) per app ad alta sicurezza
- Può essere più lungo (1 ora) per app a basso rischio

**Refresh token: 7 giorni**
- Impedisce l'accesso indefinito
- L'utente deve rifare il login dopo 7 giorni
- Può essere più breve (1 giorno) per app ad alta sicurezza
- Può essere più lungo (30 giorni) con l'opzione "ricordami"

---

## Validazione dell'input

### Validazione con Pydantic

**Tutto l'input viene validato prima di raggiungere la logica di business:**

```python
from pydantic import BaseModel, EmailStr, Field, HttpUrl

class ProgramCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    slug: str = Field(..., min_length=1, max_length=255, pattern=r'^[a-z0-9-]+$')
    description: str | None = Field(None, max_length=10000)
    website: HttpUrl | None = None

# Usage
@router.post("/programs")
async def create_program(data: ProgramCreate):
    # At this point, data is validated:
    # - name is 1-255 chars
    # - slug matches pattern (lowercase, digits, hyphens)
    # - description is max 10,000 chars
    # - website is valid URL (if provided)
    ...
```

**Vantaggi:**
- Previene la SQL injection (query parametrizzate)
- Previene l'XSS (l'HTML viene sottoposto a escape nel frontend)
- Previene il buffer overflow (limiti sulla lunghezza delle stringhe)
- Messaggi di errore chiari

### Limiti di lunghezza delle stringhe

**Perché limitare le lunghezze?**
- Previene gli attacchi DoS (stringa da 1GB nel database)
- Vincoli sulle colonne del database
- UX (niente nomi da 10.000 caratteri)

**Costanti:**

```python
# config.py
EMAIL_MAX_LENGTH = 255
PASSWORD_HASH_MAX_LENGTH = 255
FULL_NAME_MAX_LENGTH = 255
PROGRAM_NAME_MAX_LENGTH = 255
PROGRAM_SLUG_MAX_LENGTH = 255
REPORT_TITLE_MAX_LENGTH = 255
CWE_ID_MAX_LENGTH = 20
```

**Applicate ai modelli:**

```python
class User(Base):
    email: Mapped[str] = mapped_column(String(EMAIL_MAX_LENGTH))
    full_name: Mapped[str | None] = mapped_column(String(FULL_NAME_MAX_LENGTH))
```

---

## Rate limiting

### Perché il rate limiting?

**Attacchi prevenuti:**
- Tentativi di indovinare la password con brute force
- Abuso delle API (scraping, spam)
- Attacchi DoS

**Implementazione:**

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,  # Rate limit by IP
    default_limits=["100 per minute"],  # Global default
)

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Different limits for different endpoints
@router.post("/auth/login")
@limiter.limit("20/minute")  # Stricter for auth
async def login(...):
    ...

@router.get("/programs")
@limiter.limit("100/minute")  # Relaxed for public endpoints
async def list_programs(...):
    ...
```

**Configurazione:**

```python
# .env
RATE_LIMIT_DEFAULT=100/minute
RATE_LIMIT_AUTH=20/minute
```

**Risposta quando si è soggetti a rate limit:**

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704123456
Retry-After: 60

{
  "error": "Rate limit exceeded. Try again in 60 seconds."
}
```

### Rate limiting per utente

**Più sofisticato: limitare per utente autenticato**

```python
def get_rate_limit_key(request: Request) -> str:
    if request.state.user:
        return f"user:{request.state.user.id}"
    return f"ip:{request.client.host}"

limiter = Limiter(key_func=get_rate_limit_key)
```

**Vantaggi:**
- Gli utenti autenticati hanno una quota separata
- Non si può condividere il rate limit tra IP diversi
- Si possono offrire livelli premium (limiti più alti)

---

## Vulnerabilità comuni prevenute

### SQL Injection

**Codice vulnerabile:**

```python
# BAD - concatenating user input
email = request.args.get("email")
query = f"SELECT * FROM users WHERE email = '{email}'"
result = await db.execute(query)
```

**Attacco:**

```
email=admin@example.com' OR '1'='1
→ SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
→ Returns all users!
```

**Codice sicuro:**

```python
# GOOD - parameterized query
stmt = select(User).where(User.email == email)
result = await session.execute(stmt)
```

SQLAlchemy esegue automaticamente l'escape dei parametri.

### Cross-Site Scripting (XSS)

**Codice vulnerabile:**

```html
<!-- BAD - raw HTML injection -->
<div>{user.bio}</div>
```

**Attacco:**

```
bio=<script>alert('XSS')</script>
→ <div><script>alert('XSS')</script></div>
→ Script executes!
```

**Codice sicuro:**

```jsx
// React automatically escapes
<div>{user.bio}</div>
// → <div>&lt;script&gt;alert('XSS')&lt;/script&gt;</div>
```

**Per il markdown:**

```python
import markdown
from bleach import clean

def render_markdown(text: str) -> str:
    html = markdown.markdown(text)
    return clean(
        html,
        tags=['p', 'br', 'strong', 'em', 'code', 'pre'],
        strip=True,
    )
```

### Cross-Site Request Forgery (CSRF)

**Non vulnerabili perché usiamo token JWT:**
- Il CSRF richiede i cookie (inviati automaticamente)
- I token JWT sono nell'header `Authorization` (che deve essere inviato esplicitamente)
- L'attaccante non può leggere l'header `Authorization` da un altro dominio

**Se si usassero i cookie, servirebbero token CSRF:**

```python
from fastapi_csrf import CsrfProtect

@app.post("/login")
async def login(csrf_protect: CsrfProtect = Depends()):
    await csrf_protect.validate_csrf(request)
    ...
```

### Insecure Direct Object References (IDOR)

**Codice vulnerabile:**

```python
# BAD - no authorization check
@router.get("/reports/{report_id}")
async def get_report(report_id: UUID):
    report = await report_repo.get_by_id(report_id)
    return report  # Anyone can access any report!
```

**Codice sicuro:**

```python
# GOOD - check ownership
@router.get("/reports/{report_id}")
async def get_report(report_id: UUID, user: CurrentUser):
    report = await report_repo.get_by_id(report_id)
    if not report:
        raise NotFoundError()

    if (report.researcher_id != user.id and
        report.program.company_id != user.id and
        user.role != UserRole.ADMIN):
        raise ForbiddenError()

    return report
```

---

## Best practice di sicurezza

### 1. Least Privilege

**Dai agli utenti solo i permessi di cui hanno bisogno:**

```python
# BAD - everyone is admin
user.role = UserRole.ADMIN

# GOOD - specific roles
user.role = UserRole.USER  # Can only submit reports
```

### 2. Defense in Depth

**Più livelli di sicurezza:**
1. Validazione dell'input (Pydantic)
2. Autenticazione (JWT)
3. Autorizzazione (RBAC)
4. Rate limiting
5. Cifratura in transito (HTTPS)
6. Cifratura a riposo (database cifrato)

### 3. Secure Defaults

**Default sicuro, opt-in per ciò che è insicuro:**

```python
# BAD - default allows everything
CORS_ORIGINS = ["*"]

# GOOD - default is restrictive
CORS_ORIGINS = ["https://example.com"]
```

### 4. Security Header

**Aggiungi security header tramite Nginx:**

```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Content-Security-Policy "default-src 'self'";
```

### 5. Messaggi di errore

**Non far trapelare informazioni negli errori:**

```python
# BAD - reveals whether email exists
if not user:
    raise HTTPException(status_code=404, detail="User not found")
if not verify_password(password, user.hashed_password):
    raise HTTPException(status_code=401, detail="Invalid password")

# GOOD - generic error
if not user or not verify_password(password, user.hashed_password):
    raise HTTPException(status_code=401, detail="Invalid credentials")
```

### 6. Logging e monitoraggio

**Registra gli eventi di sicurezza nei log:**

```python
import structlog

logger = structlog.get_logger()

@router.post("/auth/login")
async def login(email: str, password: str):
    user = await user_repo.find_by_email(email)

    if not user:
        logger.warning("login_failed", email=email, reason="user_not_found")
        raise InvalidCredentialsError()

    if not verify_password(password, user.hashed_password):
        logger.warning("login_failed", email=email, reason="invalid_password")
        raise InvalidCredentialsError()

    logger.info("login_success", user_id=user.id, email=email)
    # ...
```

**Monitora:**
- Tentativi di login falliti
- Violazioni del rate limit
- Tentativi di riutilizzo dei token
- Pattern di attività insoliti

### 7. Sicurezza delle dipendenze

**Aggiorna regolarmente le dipendenze:**

```bash
# Check for vulnerabilities
pip-audit

# Update dependencies
pip install --upgrade -r requirements.txt
```

**Usa Dependabot o Renovate:**
- PR automatiche per gli aggiornamenti delle dipendenze
- Includono le patch di sicurezza

### 8. Gestione dei secret

**Non fare mai commit dei secret su Git:**

```bash
# .gitignore
.env
*.pem
*.key
secrets/
```

**Usa le variabili d'ambiente:**

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    SECRET_KEY: str  # Read from .env or environment
    DATABASE_URL: str
```

**In produzione, usa i secret manager:**
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault

---

## Conclusione

La sicurezza **non è un ripensamento**, è progettata in ogni livello:

1. **Autenticazione** - JWT con rotazione dei token
2. **Autorizzazione** - RBAC con proprietà delle risorse
3. **Sicurezza delle password** - Hashing Argon2id
4. **Sicurezza dei token** - Storage con hash, versionamento dei token
5. **Validazione dell'input** - Schema Pydantic
6. **Rate limiting** - Per IP e per utente
7. **Prevenzione delle vulnerabilità** - SQL injection, XSS, CSRF, IDOR

Per maggiori informazioni:
- Architettura del sistema: [ARCHITECTURE.it.md](./ARCHITECTURE.it.md)
- Design pattern: [PATTERNS.it.md](./PATTERNS.it.md)
- Progettazione del database: [DATABASE.it.md](./DATABASE.it.md)
- Tutorial pratico: [GETTING-STARTED.it.md](./GETTING-STARTED.it.md)

**Ricorda:** la sicurezza è un viaggio, non una destinazione. Resta aggiornato sulle best practice e sulle vulnerabilità comuni.
