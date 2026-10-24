# Guida all'Implementazione

Questo documento percorre il codice reale. Costruiremo le funzionalità chiave passo dopo passo e spiegheremo le decisioni lungo il percorso.

## Walkthrough della Struttura dei File
```
backend/
├── main.py              # Application entry point
├── factory.py           # FastAPI app factory with middleware
├── config.py            # Environment variables and constants
├── core/
│   ├── database.py      # SQLAlchemy engine and session factory
│   ├── security.py      # Password hashing, JWT creation/validation
│   ├── dependencies.py  # FastAPI dependencies (auth, database)
│   └── enums.py         # Type-safe enums for status, severity, test types
├── models/
│   ├── Base.py          # Base model with common fields (id, timestamps)
│   ├── User.py          # User authentication model
│   ├── Scan.py          # Scan metadata model
│   └── TestResult.py    # Individual test results model
├── repositories/
│   ├── user_repository.py         # User database operations
│   ├── scan_repository.py         # Scan database operations
│   └── test_result_repository.py  # Test result database operations
├── routes/
│   ├── auth.py          # Registration and login endpoints
│   └── scans.py         # Scan CRUD endpoints
├── schemas/
│   ├── user_schemas.py        # Pydantic models for user data
│   ├── scan_schemas.py        # Pydantic models for scan data
│   └── test_result_schemas.py # Pydantic models for test results
├── services/
│   ├── auth_service.py  # Authentication business logic
│   └── scan_service.py  # Scan orchestration business logic
└── scanners/
    ├── base_scanner.py      # Common HTTP logic, retry handling
    ├── rate_limit_scanner.py # Rate limiting detection
    ├── auth_scanner.py      # Authentication vulnerability detection
    ├── sqli_scanner.py      # SQL injection detection
    ├── idor_scanner.py      # IDOR/BOLA detection
    └── payloads.py          # Attack payloads organized by type
```

## Costruire l'Autenticazione

### Passo 1: Hashing delle Password con Bcrypt

Cosa stiamo costruendo: Un sistema sicuro di memorizzazione delle password che protegge le credenziali degli utenti anche se il database viene compromesso.

Il codice si trova in `backend/core/security.py:11-28`:
```python
import bcrypt
from jose import JWTError, jwt

def hash_password(password: str) -> str:
    """
    Hash a plain text password using bcrypt
    """
    password_bytes = password.encode("utf-8")  # Convert string to bytes
    salt = bcrypt.gensalt()  # Generate random salt (default 12 rounds)
    hashed = bcrypt.hashpw(password_bytes, salt)
    return hashed.decode("utf-8")  # Convert bytes back to string for storage
```

**Perché questo codice funziona:**
- **Riga 18-19**: Bcrypt richiede input in byte, non stringhe. Il passaggio di encoding/decoding converte tra stringhe Python e array di byte.
- **Riga 19**: `bcrypt.gensalt()` genera un salt unico per ogni password. Il salt è incorporato nell'hash di output, quindi non occorre memorizzarlo separatamente.
- **Riga 20**: `bcrypt.hashpw()` esegue l'hashing vero e proprio. Con i 12 round predefiniti, questo richiede intenzionalmente ~100ms. È proprio questo il punto - rende costoso il brute force.
- **Riga 21**: Riconverti in stringa perché il tipo di colonna `String` di SQLAlchemy si aspetta stringhe, non byte.

La funzione di verifica (`security.py:21-31`):
```python
def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    Verify a plain text password against a hashed password
    """
    password_bytes = plain_password.encode("utf-8")
    hashed_bytes = hashed_password.encode("utf-8")
    return bcrypt.checkpw(password_bytes, hashed_bytes)
```

**Cosa succede:**
1. Converte entrambi gli input in byte
2. `bcrypt.checkpw()` estrae il salt da `hashed_bytes` (è incorporato nell'hash)
3. Calcola l'hash di `password_bytes` con quello stesso salt
4. Confronta il risultato - se coincidono, la password è corretta

**Errori comuni qui:**
```python
# Wrong - storing plaintext
user.password = password  # Database breach = everyone's password leaked

# Wrong - using weak hashing
import hashlib
user.password = hashlib.md5(password.encode()).hexdigest()  # Fast = easy to brute force

# Wrong - forgetting to salt
user.password = hashlib.sha256(password.encode()).hexdigest()  # Rainbow tables break this

# Right - bcrypt with automatic salting
user.hashed_password = hash_password(password)  # Secure
```

### Passo 2: Creazione del Token JWT

Ora dobbiamo creare i token di autenticazione dopo un login riuscito.

In `core/security.py:34-56`:
```python
from datetime import datetime, timedelta
from config import settings

def create_access_token(
    data: dict[str, str],
    expires_delta: timedelta | None = None
) -> str:
    """
    Create a JWT access token
    """
    to_encode = data.copy()  # Don't modify the original dict

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES  # 1440 = 24 hours
        )

    to_encode.update({"exp": expire})  # Add expiration claim

    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,  # MUST be random, 256+ bits
        algorithm=settings.ALGORITHM  # "HS256"
    )
    return encoded_jwt
```

**Parti chiave spiegate:**

**Riga 43** - Copiamo il dict dei dati perché stiamo per modificarlo. Se mutassimo l'originale, il chiamante vedrebbe `{"sub": "user@example.com", "exp": 1234567890}` invece della sola email.

**Riga 45-50** - Calcola quando scade il token. Se non viene passato alcun `expires_delta`, il default è 24 ore da `config.py`. Il claim `exp` fa parte dello standard JWT - le librerie lo controllano automaticamente.

**Riga 52** - Qui avviene la magia. La funzione `jwt.encode()`:
1. Converte il dict `to_encode` in JSON
2. Lo codifica in Base64 come payload
3. Crea un header: `{"alg": "HS256", "typ": "JWT"}`
4. Calcola la firma HMAC-SHA256 usando `SECRET_KEY`
5. Unisce header.payload.signature con dei punti

Il risultato appare così:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIiwiZXhwIjoxNzM4MzY4MDAwfQ.signature_here
```

**Perché lo facciamo in questo modo:**
I JWT sono stateless. Il server non ha bisogno di memorizzare dati di sessione. Quando arriva una richiesta con un JWT, il server verifica la firma e si fida del payload. Questo significa che puoi scalare orizzontalmente - qualsiasi istanza di backend può validare qualsiasi token senza coordinarsi con altre istanze o con uno store di sessione condiviso.

**Approcci alternativi:**
- **Autenticazione basata su sessione**: Memorizza l'ID di sessione in un cookie, cerca i dati dell'utente a ogni richiesta. Più semplice ma richiede uno storage di sessione (Redis, database). Non scala altrettanto facilmente.
- **OAuth 2.0 con refresh token**: Access token di breve durata (15 min) + refresh token di lunga durata. Più sicuro (si possono revocare i refresh token) ma più complesso da implementare.

### Passo 3: Validazione del Token

Quando un endpoint protetto riceve una richiesta, dobbiamo validare il JWT e caricare l'utente.

La funzione di dependency injection in `core/dependencies.py:17-49`:
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from sqlalchemy.orm import Session
from .security import decode_token
from .database import get_db
from repositories.user_repository import UserRepository
from schemas.user_schemas import UserResponse

security = HTTPBearer()  # Extracts "Authorization: Bearer <token>" header

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db),
) -> UserResponse:
    """
    FastAPI dependency to extract and verify the current authenticated user
    """
    try:
        # Decode and verify the JWT
        payload = decode_token(credentials.credentials)
        email: str | None = payload.get("sub")

        if email is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication credentials",
                headers={"WWW-Authenticate": "Bearer"},
            )

        # Load user from database
        user = UserRepository.get_by_email(db, email)

        if not user:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="User not found",
                headers={"WWW-Authenticate": "Bearer"},
            )

        return UserResponse.model_validate(user)

    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        ) from None
```

**Cosa succede:**

**Riga 27** - `HTTPBearer()` è l'extractor integrato di FastAPI per gli header `Authorization: Bearer <token>`. Esegue il parsing dell'header e ci dà la stringa del token in `credentials.credentials`.

**Riga 35** - Chiama `decode_token()` che si trova in `security.py:48-62`:
```python
def decode_token(token: str) -> dict[str, str]:
    """
    Decode and verify a JWT token
    """
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]  # Only accept HS256
        )
        return payload
    except JWTError as e:
        raise ValueError(f"Invalid token: {str(e)}") from e
```

Questo verifica:
- La firma è valida (il token non è stato manomesso)
- L'algoritmo corrisponde a quello atteso (previene l'attacco con algoritmo "none")
- Il token non è scaduto (controlla automaticamente il claim `exp`)

**Riga 36** - Estrae il claim `sub` (subject). È l'identificatore dell'utente che abbiamo inserito nel token durante il login.

**Riga 38-43** - Se `sub` manca, il token è malformato. Restituisce 401 con l'header `WWW-Authenticate` come da standard HTTP.

**Riga 46** - Carica il record completo dell'utente dal database. Potremmo saltare questo passaggio e usare solo l'email del token, ma caricarlo dal DB garantisce che:
- L'utente esista ancora (non sia stato eliminato dopo l'emissione del token)
- L'utente sia ancora attivo (non sia stato disattivato)
- Otteniamo l'oggetto utente completo con tutti i campi

**Riga 48-52** - Se l'utente non esiste nel database, il token non è valido. Questo intercetta gli utenti eliminati con token ancora validi.

**Riga 54** - Converte il modello SQLAlchemy in schema Pydantic. Questo esclude `hashed_password` dalla risposta (lo schema Pydantic non lo include).

**Riga 56-61** - Intercetta `ValueError` da `decode_token()` e la converte in HTTP 401. Il `from None` sopprime il traceback dell'eccezione concatenata - messaggi di errore più puliti per i client.

### Testare l'Autenticazione

Come verificare che funzioni:
```bash
# Register a user
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePass123"
  }'

# Response: {"id": 1, "email": "test@example.com", ...}

# Login
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePass123"
  }'

# Response: {"access_token": "eyJ...", "token_type": "bearer"}

# Use token to access protected endpoint
curl http://localhost:8000/scans/ \
  -H "Authorization: Bearer eyJ..."

# Response: [list of scans]
```

Output atteso: La prima richiesta crea l'utente, la seconda restituisce il JWT, la terza restituisce un array di scan (inizialmente vuoto).

Se vedi `401 Unauthorized`, controlla:
- Il token viene inviato nell'header (non nel query param, non nel body)
- Il formato è esattamente `Bearer <token>` con la B maiuscola
- Il token non è scaduto (24 ore dal login)
- L'utente esiste ancora nel database

## Costruire il Rilevamento di SQL Injection

### Il Problema

Dobbiamo rilevare tre tipi di SQL injection:
1. **Error-based** - Gli errori del database rivelano informazioni
2. **Boolean-based blind** - Le risposte differiscono per condizioni vere/false
3. **Time-based blind** - I ritardi del database rivelano l'injection

### La Soluzione

Usa librerie di payload e analisi delle risposte per rilevare ciascun tipo. Lo scanner si trova in `backend/scanners/sqli_scanner.py`.

### Implementazione

Partiamo dal rilevamento error-based (`sqli_scanner.py:51-94`):
```python
def _test_error_based_sqli(self) -> dict[str, Any]:
    """
    Test for error based SQL injection

    Detects database errors in responses indicating SQLi vulnerability
    """
    error_signatures = SQLiPayloads.get_error_signatures()

    basic_payloads = SQLiPayloads.BASIC_AUTHENTICATION_BYPASS

    for payload in basic_payloads:
        try:
            response = self.make_request("GET", f"/?id={payload}")

            response_text_lower = response.text.lower()

            # Look for database error signatures
            for db_type, signatures in error_signatures.items():
                for signature in signatures:
                    if signature in response_text_lower:
                        return {
                            "vulnerable": True,
                            "database_type": db_type,
                            "payload": payload,
                            "status_code": response.status_code,
                            "error_signature": signature,
                            "response_excerpt": response.text[:500],
                        }

        except Exception:
            continue  # Network errors don't indicate SQLi

    return {
        "vulnerable": False,
        "payloads_tested": len(basic_payloads),
        "description": "No database errors detected",
    }
```

**Parti chiave spiegate:**

**Riga 57** - Carica le signature di errore da `scanners/payloads.py:9-30`. Sono messaggi di errore specifici per database:
```python
ERROR_SIGNATURES = {
    "mysql": [
        "sql syntax",
        "mysql_fetch",
        "mysql error",
    ],
    "postgres": [
        "postgresql",
        "pg_query",
        "syntax error",
    ],
    "mssql": [
        "sqlserver jdbc driver",
        "sqlexception",
    ],
}
```

**Riga 59** - Payload di base come `' OR '1'='1`, `' OR 1=1--`, `admin'--` da `payloads.py:33-48`.

**Riga 63** - Effettua la richiesta con il payload nel query parameter. Se il backend fa:
```python
query = f"SELECT * FROM users WHERE id = {request.args['id']}"
```

E noi inviamo `id=' OR 1=1--`, diventa:
```sql
SELECT * FROM users WHERE id = ' OR 1=1--'
```

Questo è SQL non valido. Il database restituisce un errore: `"You have an error in your SQL syntax"`.

**Riga 65** - Converte la risposta in minuscolo per un confronto case-insensitive. I messaggi di errore variano: "SQL syntax", "Sql Syntax", "sql syntax".

**Riga 68-72** - Controlla ogni signature rispetto alla risposta. Se `"sql syntax"` compare nell'HTML, abbiamo trovato una SQLi.

**Riga 73-79** - Restituisce le evidenze: quale payload ha funzionato, quale tipo di database è stato rilevato, il messaggio di errore effettivo (primi 500 caratteri).

**Riga 81-82** - Gli errori di rete (timeout, connessione rifiutata) non significano SQLi. Passa al payload successivo.

**Riga 84-88** - Nessun errore trovato con tutti i payload = non vulnerabile (almeno a SQLi error-based).

### Rilevamento Boolean-Based Blind

Quando gli errori sono soppressi, controlla se le risposte differiscono per condizioni vere rispetto a condizioni false.

Codice in `sqli_scanner.py:96-164`:
```python
def _test_boolean_based_sqli(self) -> dict[str, Any]:
    """
    Test for boolean based blind SQL injection

    Compares responses from true vs false conditions to detect SQLi
    """
    try:
        # Establish baseline
        baseline_response = self.make_request("GET", "/?id=1")
        baseline_length = len(baseline_response.text)
        baseline_status = baseline_response.status_code

        if baseline_status != 200:
            return {
                "vulnerable": False,
                "description": "Baseline request failed",
                "baseline_status": baseline_status,
            }

        boolean_payloads = SQLiPayloads.BOOLEAN_BASED_BLIND

        # Test true conditions: ' AND 1=1--
        true_payloads = [
            p for p in boolean_payloads
            if "AND '1'='1" in p or "AND 1=1" in p
        ]
        false_payloads = [
            p for p in boolean_payloads
            if "AND '1'='2" in p or "AND 1=2" in p or "AND 1=0" in p
        ]

        true_lengths = []
        for payload in true_payloads:
            response = self.make_request("GET", f"/?id={payload}")
            true_lengths.append(len(response.text))

        false_lengths = []
        for payload in false_payloads:
            response = self.make_request("GET", f"/?id={payload}")
            false_lengths.append(len(response.text))

        # Calculate averages
        avg_true = statistics.mean(true_lengths)
        avg_false = statistics.mean(false_lengths)

        length_diff = abs(avg_true - avg_false)

        # Significant difference indicates SQLi
        if length_diff > 100 and avg_true != avg_false:
            return {
                "vulnerable": True,
                "baseline_length": baseline_length,
                "true_condition_avg_length": avg_true,
                "false_condition_avg_length": avg_false,
                "length_difference": length_diff,
                "confidence": "HIGH" if length_diff > 500 else "MEDIUM",
            }

        return {
            "vulnerable": False,
            "description": "No boolean-based SQLi detected",
            "length_difference": length_diff,
        }
```

**Cosa succede:**

**Riga 105-106** - Ottiene la risposta di baseline con un input normale (`id=1`). Ci serve per fare il confronto.

**Riga 121-128** - Divide i payload in condizioni "sempre vere" e "sempre false".

Esempio di condizione vera: `1' AND 1=1--`
```sql
SELECT * FROM users WHERE id = 1' AND 1=1--'
```
`1=1` è sempre vera, quindi se è vulnerabile, questa restituisce dati.

Esempio di condizione falsa: `1' AND 1=2--`
```sql
SELECT * FROM users WHERE id = 1' AND 1=2--'
```
`1=2` non è mai vera, quindi se è vulnerabile, questa restituisce un risultato vuoto.

**Riga 130-141** - Invia più payload veri e falsi, registra le lunghezze delle risposte.

**Riga 144-147** - Calcola la lunghezza media per vero rispetto a falso. L'approccio statistico riduce i falsi positivi dovuti alla varianza di rete.

**Riga 151** - Se la differenza è significativa (>100 byte) e non nulla, probabilmente è vulnerabile. Una differenza di 500+ byte è ad alta confidenza - probabilmente si vedono record completi contro risultati vuoti.

**Perché funziona:**
Se esiste una SQLi, le condizioni vere restituiscono dati (risposta più lunga), le condizioni false non restituiscono nulla (risposta più corta). Se non c'è SQLi, entrambe vengono trattate come input non valido e restituiscono la stessa pagina di errore.

### Rilevamento Time-Based Blind

La tecnica più sofisticata. Usa le funzioni sleep del database per misurare l'injection.

Codice in `sqli_scanner.py:183-251`:
```python
def _test_time_based_sqli(self, delay_seconds: int = 5) -> dict[str, Any]:
    """
    Test for time based blind SQL injection

    Uses baseline timing comparison with statistical analysis
    for false positive reduction
    """
    try:
        # Establish baseline timing
        baseline_mean, baseline_stdev = self.get_baseline_timing("/")

        threshold = baseline_mean + (3 * baseline_stdev)
        expected_delay_time = baseline_mean + delay_seconds

        all_time_payloads = SQLiPayloads.TIME_BASED_BLIND

        # Group by database type
        delay_payloads = {
            "mysql": [p for p in all_time_payloads if "SLEEP" in p],
            "postgres": [p for p in all_time_payloads if "pg_sleep" in p],
            "mssql": [p for p in all_time_payloads if "WAITFOR" in p],
        }

        for db_type, payloads in delay_payloads.items():
            for payload in payloads:
                delay_times = []

                # Take multiple samples
                for _ in range(3):
                    try:
                        response = self.make_request(
                            "GET",
                            f"/?id={payload}",
                            timeout=delay_seconds + 10,
                        )
                        elapsed = getattr(response, "request_time", 0.0)
                        delay_times.append(elapsed)

                    except Exception:
                        delay_times.append(delay_seconds + 10)  # Assume timeout = worked

                    time.sleep(1)  # Space out requests

                avg_delay = statistics.mean(delay_times)

                # Check if delay matches expected
                if avg_delay >= expected_delay_time - 1:
                    confidence = "HIGH" if avg_delay >= expected_delay_time else "MEDIUM"

                    return {
                        "vulnerable": True,
                        "database_type": db_type,
                        "payload": payload,
                        "baseline_time": f"{baseline_mean:.3f}s",
                        "response_time": f"{avg_delay:.3f}s",
                        "expected_delay": f"{expected_delay_time:.3f}s",
                        "confidence": confidence,
                        "individual_times": [f"{t:.3f}s" for t in delay_times],
                    }
```

**Dettagli importanti:**

**Riga 192** - Ottiene il timing di baseline da `base_scanner.py:158-177`:
```python
def get_baseline_timing(
    self,
    endpoint: str,
    samples: int | None = None
) -> tuple[float, float]:
    """
    Establish baseline response time for an endpoint

    Takes multiple samples and calculates mean and standard deviation
    """
    if samples is None:
        samples = settings.DEFAULT_BASELINE_SAMPLES  # 10

    times = []

    for _ in range(samples):
        response = self.make_request("GET", endpoint)
        times.append(getattr(response, "request_time", 0.0))
        time.sleep(0.5)  # Space out samples

    return statistics.mean(times), statistics.stdev(times)
```

Questo effettua 10 richieste normali e calcola media e deviazione standard. Risultati di esempio:
- Media: 0.15s
- Stdev: 0.02s

**Riga 194** - Calcola la soglia: `mean + 3*stdev`. Con l'esempio sopra: `0.15 + 3*0.02 = 0.21s`. Qualsiasi risposta oltre 0.21s è "insolitamente lenta" (confidenza del 99,7% se la distribuzione è normale).

**Riga 195** - Ritardo atteso: `baseline + 5s`. Se la baseline è 0.15s e iniettiamo `SLEEP(5)`, ci aspettiamo una risposta di ~5.15s.

**Riga 200-204** - Raggruppa i payload per database. MySQL usa `SLEEP(5)`, Postgres usa `pg_sleep(5)`, MSSQL usa `WAITFOR DELAY '0:0:5'`.

**Riga 211-225** - Prende 3 campioni per ogni payload. Il jitter di rete può causare una varianza di ±0.5s. Fare la media di 3 campioni dà un segnale più pulito.

**Riga 230** - Se il ritardo medio è entro 1 secondo da quello atteso (5.15s ± 1s), è SQLi. La tolleranza di 1 secondo tiene conto dell'overhead di rete.

**Perché funziona:**
Payload come `1'; SELECT SLEEP(5)--` eseguono lo sleep se esiste una SQLi. La risposta impiega 5 secondi in più. Senza SQLi, il payload viene semplicemente trattato come input non valido e restituisce subito.

## Implementazione della Sicurezza

### Validazione della Firma JWT

Fondamentale per prevenire la falsificazione dei token. Il test dell'attacco con algoritmo none in `auth_scanner.py:168-213`:
```python
def _test_none_algorithm(self) -> dict[str, Any]:
    """
    Test if server accepts JWT with 'none' algorithm

    Critical vulnerability: allows unsigned tokens to be accepted
    """
    try:
        header, payload, signature = self.auth_token.split(".")

        none_variants = AuthPayloads.get_jwt_none_variants()
        # ["none", "None", "NONE", "nOnE", "NoNe", "NOne"]

        for variant in none_variants:
            # Create malicious header
            malicious_header = self._base64url_encode(
                json.dumps({"alg": variant, "typ": "JWT"})
            )

            # Remove signature (trailing period means "no signature")
            malicious_token = f"{malicious_header}.{payload}."

            response = self.make_request(
                "GET",
                "/",
                headers={"Authorization": f"Bearer {malicious_token}"}
            )

            if response.status_code == 200:
                return {
                    "vulnerable": True,
                    "vulnerability_type": "JWT None Algorithm",
                    "algorithm_variant": variant,
                    "status_code": response.status_code,
                    "recommendations": [
                        "Reject tokens with 'none' algorithm (all case variations)",
                        "Explicitly verify signature before accepting tokens",
                        "Use allowlist of accepted algorithms",
                    ],
                }

        return {
            "vulnerable": False,
            "description": "None algorithm properly rejected",
        }
```

**Cosa previene:**

JWT normale: `header.payload.signature`
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiJ9.signature
```

JWT malevolo: `header.payload.` (nessuna firma)
```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiJ9.
```

Header malevolo decodificato:
```json
{"alg": "none", "typ": "JWT"}
```

Payload malevolo decodificato:
```json
{"sub": "admin"}
```

Se il server lo accetta, l'attaccante può falsificare token con qualsiasi identità utente.

**Come difendersi:**

La validazione JWT del progetto (`core/security.py:54-57`) specifica esplicitamente gli algoritmi:
```python
payload = jwt.decode(
    token,
    settings.SECRET_KEY,
    algorithms=[settings.ALGORITHM]  # ["HS256"] - none not in this list
)
```

Il parametro `algorithms` è una allowlist. La libreria rifiuterà i token con `alg: none` perché non è in `["HS256"]`.

**Cosa NON fare:**
```python
# BAD - accepts any algorithm
payload = jwt.decode(token, settings.SECRET_KEY)

# BAD - doesn't verify signature
payload = jwt.decode(token, options={"verify_signature": False})

# BAD - allows none
payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256", "none"])
```

### Pattern di Prevenzione IDOR

Controlli di autorizzazione in `services/scan_service.py:67-84`:
```python
@staticmethod
def get_scan_by_id(db: Session, scan_id: int, user_id: int) -> ScanResponse:
    """
    Get scan by ID with authorization check
    """
    scan = ScanRepository.get_by_id(db, scan_id)

    if not scan:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Scan not found",
        )

    # CRITICAL: Check if user owns this scan
    if scan.user_id != user_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not authorized to access this scan",
        )

    return ScanResponse.model_validate(scan)
```

**Perché 404 prima di 403:**
La riga 75-78 restituisce 404 se lo scan non esiste, PRIMA di controllare la proprietà. Questo previene la divulgazione di informazioni.

Se controllassimo prima la proprietà:
- Richiesta dello scan 999 (non esiste) → 404 "Scan not found"
- Richiesta dello scan 123 (esiste, appartiene a qualcun altro) → 403 "Not authorized"

Ora l'attaccante sa che lo scan 123 esiste. Può enumerare tutti gli ID degli scan e scoprire quali sono validi.

Approccio migliore:
- Richiesta dello scan 999 → 404
- Richiesta dello scan 123 → 404

Non si può distinguere tra "non esiste" ed "esiste ma non puoi accedervi". Si divulgano meno informazioni.

**Il controllo di autorizzazione:**
La riga 81-84 confronta `scan.user_id` (proprietario) con `user_id` (richiedente). Se non coincidono, restituisce 403.

Questo avviene a livello di service, NON a livello di route. Ogni percorso di codice che recupera gli scan passa attraverso il service, quindi non possiamo dimenticare il controllo.

## Esempio di Flusso dei Dati

Tracciamo una richiesta completa attraverso il sistema: l'utente crea uno scan che testa la SQLi.

### La Richiesta Arriva

Punto di ingresso: `routes/scans.py:23-37`
```python
@router.post("/", response_model=ScanResponse, status_code=status.HTTP_201_CREATED)
@limiter.limit(settings.API_RATE_LIMIT_SCAN)  # "15/minute"
async def create_scan(
    request: Request,
    scan_request: ScanRequest,  # Pydantic validates this
    db: Session = Depends(get_db),  # Injects database session
    current_user: UserResponse = Depends(get_current_user),  # Validates JWT
) -> ScanResponse:
    """
    Create and execute a new security scan
    """
    return ScanService.run_scan(db, current_user.id, scan_request)
```

A questo punto:
- `scan_request` è validato da Pydantic (formato URL corretto, tipi di test validi)
- `current_user` è autenticato (il JWT era valido, l'utente esiste nel database)
- `db` è una sessione SQLAlchemy nuova
- Il rate limiting è stato controllato - se l'utente ha superato 15 scan/minuto, la richiesta è stata rifiutata prima che questa funzione venga eseguita

Cosa succede dopo: Si chiama il service layer per orchestrare lo scan.

### Livello di Elaborazione

Il service orchestra gli scanner: `services/scan_service.py:23-65`
```python
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
        scanner_class: type[BaseScanner] | None = scanner_mapping.get(test_type)

        if not scanner_class:
            continue  # Skip unknown test types

        try:
            scanner = scanner_class(
                target_url=str(scan_request.target_url),
                auth_token=scan_request.auth_token,
                max_requests=scan_request.max_requests,
            )

            result = scanner.scan()  # Execute the actual test
            results.append(result)

        except Exception as e:
            # Scanner crashed - return error result instead of failing entire scan
            results.append(
                TestResultCreate(
                    test_name=test_type,
                    status="error",
                    severity="info",
                    details=f"Scanner error: {str(e)}",
                    evidence_json={"error": str(e)},
                    recommendations_json=[
                        "Check target URL is accessible",
                        "Verify authentication token if provided",
                    ],
                )
            )
```

Questo codice:
- Crea subito il record dello scan (riga 26-30) in modo che abbia un ID
- Mappa gli enum dei tipi di test alle classi scanner effettive (riga 33-37)
- Cicla sui test richiesti (riga 42-60)
- Istanzia ogni scanner con l'URL di destinazione e il token di autenticazione
- Chiama `scanner.scan()` che restituisce `TestResultCreate` con i finding
- Intercetta le eccezioni in modo che uno scanner che fallisce non faccia fallire l'intero scan

### Storage/Output

Salva i risultati nel database: `services/scan_service.py:62-65` continua:
```python
    # Save all results
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

    db.refresh(scan)  # Reload scan with test_results relationship populated

    return ScanResponse.model_validate(scan)
```

Il risultato è una risposta JSON contenente lo scan con i risultati dei test annidati:
```json
{
  "id": 42,
  "user_id": 1,
  "target_url": "https://api.example.com/users",
  "scan_date": "2026-02-04T10:30:00Z",
  "created_at": "2026-02-04T10:30:00Z",
  "test_results": [
    {
      "id": 101,
      "scan_id": 42,
      "test_name": "sqli",
      "status": "vulnerable",
      "severity": "critical",
      "details": "Error-based SQL injection detected: mysql",
      "evidence_json": {
        "database_type": "mysql",
        "payload": "' OR 1=1--",
        "error_signature": "sql syntax"
      },
      "recommendations_json": [
        "Use parameterized queries (prepared statements)",
        "Never concatenate user input into SQL queries"
      ]
    }
  ]
}
```

## Pattern di Gestione degli Errori

### Errori del Database con Rollback Automatico

La dependency `get_db()` gestisce la gestione delle transazioni:
```python
# core/database.py:28-36
def get_db() -> Generator[Session, None, None]:
    """
    FastAPI dependency for database sessions
    """
    db = SessionLocal()
    try:
        yield db  # Provide session to route handler
    finally:
        db.close()  # Always close, even if exception raised
```

Se si verifica un'eccezione durante l'elaborazione della richiesta:
1. FastAPI la intercetta
2. Il controllo torna al blocco finally
3. Viene eseguito `db.close()`
4. SQLAlchemy esegue automaticamente il rollback della transazione non committata
5. La connessione viene restituita al pool

Scenario di errore di esempio:
```python
@router.post("/scans/")
async def create_scan(db: Session = Depends(get_db), ...):
    scan = ScanRepository.create_scan(db, ...)  # INSERT INTO scans

    # Something goes wrong here
    raise ValueError("Oops")

    # This never runs
    TestResultRepository.create_test_result(db, ...)
```

Senza gestione esplicita delle transazioni:
- Il record dello scan verrebbe inserito
- Il risultato del test non verrebbe inserito
- Il database resterebbe in uno stato inconsistente

Con la pulizia di `get_db()`:
- ValueError si propaga a FastAPI
- Viene eseguito `db.close()`
- L'INSERT dello scan viene annullato con il rollback
- Il database resta consistente

**Cosa NON fare:**
```python
# Bad - manual session management
db = SessionLocal()
try:
    scan = ScanRepository.create_scan(db, ...)
    db.commit()  # Committed before error could happen
    raise ValueError("Oops")
finally:
    db.close()

# Scan is committed, error occurs after - inconsistent state
```

### Recupero dai Timeout dello Scanner

Gli scanner usano una logica di retry con exponential backoff (`base_scanner.py:92-156`):
```python
def make_request(
    self,
    method: str,
    endpoint: str,
    **kwargs: Any,
) -> requests.Response:
    """
    Make HTTP request with retry logic and rate limit handling
    """
    self._wait_before_request()  # Implement request spacing

    url = urljoin(self.target_url, endpoint)
    retry_count = 0
    backoff_factor = 2.0

    kwargs.setdefault("timeout", settings.SCANNER_CONNECTION_TIMEOUT)

    while retry_count <= settings.DEFAULT_RETRY_COUNT:  # 3 retries
        try:
            start_time = time.time()
            response = self.session.request(method, url, **kwargs)

            # Track timing for time-based detection
            setattr(response, "request_time", time.time() - start_time)

            self.request_count += 1

            # Handle 429 Too Many Requests
            if response.status_code == 429:
                retry_after = response.headers.get("Retry-After", "60")
                wait_time = int(retry_after) if retry_after.isdigit() else 60
                time.sleep(wait_time)
                retry_count += 1
                continue  # Try again after waiting

            # Handle server errors with backoff
            if response.status_code >= 500 and retry_count < settings.DEFAULT_RETRY_COUNT:
                wait_time = backoff_factor ** retry_count  # 1s, 2s, 4s
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
                raise  # Give up after 3 retries

    return response  # Return last response if loop completes
```

**Scenari di retry:**

1. **Timeout**: Attendi 1s, riprova. Ancora timeout? Attendi 2s, riprova. Ancora timeout? Attendi 4s, riprova. Ancora timeout? Solleva un'eccezione.

2. **Connessione rifiutata**: Stessa strategia di exponential backoff.

3. **429 Rate Limited**: Leggi l'header `Retry-After`, attendi quel tempo, riprova. Non serve l'exponential backoff - il server ci ha detto esattamente quanto attendere.

4. **500 Server Error**: Exponential backoff, ma riprova solo se `retry_count < 3`. Dopo 3 tentativi, restituisci la risposta 500 (non sollevare un'eccezione). Lascia che sia il chiamante a decidere come gestirla.

## Ottimizzazioni delle Performance

### Prima: Query Naive al Database (Problema N+1)
```python
# Bad - triggers N+1 queries
@router.get("/scans/")
async def get_scans(db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    scans = db.query(Scan).filter(Scan.user_id == user.id).all()
    # SQL: SELECT * FROM scans WHERE user_id = 1

    for scan in scans:
        print(scan.test_results)  # Each access triggers new query!
        # SQL: SELECT * FROM test_results WHERE scan_id = 42
        # SQL: SELECT * FROM test_results WHERE scan_id = 43
        # ... repeated for each scan

    return scans
```

Con 10 scan, 4 risultati ciascuno = 41 query (1 + 10*4).

### Dopo: Eager Loading con joinedload
```python
# Good - single query with JOIN
def get_by_user(db: Session, user_id: int) -> list[Scan]:
    return (
        db.query(Scan)
        .options(joinedload(Scan.test_results))  # Load relationship in same query
        .filter(Scan.user_id == user_id)
        .all()
    )
    # SQL: SELECT scans.*, test_results.*
    #      FROM scans
    #      LEFT JOIN test_results ON scans.id = test_results.scan_id
    #      WHERE scans.user_id = 1
```

Una singola query carica tutto. Accedere a `scan.test_results` usa dati già caricati, nessuna query aggiuntiva.

**Benchmark:**
- Prima: 41 query, ~205ms (5ms per query)
- Dopo: 1 query, ~15ms
- Miglioramento: **13 volte più veloce**

### Connection Pooling delle Richieste

Lo scanner base riutilizza la sessione HTTP (`base_scanner.py:40-62`):
```python
def __init__(self, target_url: str, auth_token: str | None = None, ...):
    self.target_url = target_url.rstrip("/")
    self.auth_token = auth_token
    self.session = self._create_session()  # Created once

def _create_session(self) -> requests.Session:
    """
    Create persistent HTTP session with proper headers
    """
    session = requests.Session()

    session.headers.update({
        "User-Agent": f"{settings.APP_NAME}/{settings.VERSION}",
        "Accept": "application/json",
    })

    if self.auth_token:
        session.headers.update({"Authorization": f"Bearer {self.auth_token}"})

    return session
```

**Perché è importante:**

Senza sessione (nuova richiesta ogni volta):
```python
# Each call creates new TCP connection
requests.get("https://api.example.com/endpoint1")  # Connect, TLS handshake, request, close
requests.get("https://api.example.com/endpoint2")  # Connect, TLS handshake, request, close
# 2 connections, 2 TLS handshakes
```

Con la sessione:
```python
session = requests.Session()
session.get("https://api.example.com/endpoint1")  # Connect, TLS handshake, request, keep-alive
session.get("https://api.example.com/endpoint2")  # Reuse connection, request
# 1 connection, 1 TLS handshake
```

Un TLS handshake richiede 50-100ms. Su 100 richieste, sono 5-10 secondi risparmiati.

## Gestione della Configurazione

### Caricamento della Config

Tutte le impostazioni vengono caricate dall'ambiente tramite Pydantic:
```python
# config.py:14-69
from functools import lru_cache
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file="../.env",  # Load from .env file
        env_file_encoding="utf-8",
        case_sensitive=True  # DATABASE_URL != database_url
    )

    # Required fields (no default)
    DATABASE_URL: str
    SECRET_KEY: str

    # Optional fields (have defaults)
    APP_NAME: str = "API Security Tester"
    DEBUG: bool = False
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440

    @property
    def cors_origins_list(self) -> list[str]:
        """Convert comma-separated string to list"""
        return [origin.strip() for origin in self.CORS_ORIGINS.split(",")]

@lru_cache
def get_settings() -> Settings:
    """
    Get cached settings instance

    @lru_cache ensures settings are loaded only once
    """
    return Settings()

settings = get_settings()
```

**Validazione:**

Pydantic valida i tipi all'avvio:
```python
# .env contains:
# DEBUG=not_a_boolean

# When Settings() instantiates:
# ValidationError: field required to be bool, got str 'not_a_boolean'
```

L'applicazione va in crash immediatamente con un errore chiaro invece di misteriosi fallimenti a runtime.

**Perché la cache con `@lru_cache`:**

Senza cache:
```python
# Every import creates new Settings instance
from config import settings  # Loads .env
from config import settings  # Loads .env again
```

Con la cache:
```python
# First import loads .env
from config import settings  # Loads .env, caches result

# Subsequent imports reuse cached instance
from config import settings  # Returns cached Settings
```

Avvio più rapido, stato coerente in tutta l'applicazione.

## Operazioni su Database/Storage

### Creare Record con le Transazioni

Metodo del repository con controllo esplicito del commit:
```python
# repositories/scan_repository.py:13-32
@staticmethod
def create_scan(
    db: Session,
    user_id: int,
    target_url: str,
    commit: bool = True  # Allow caller to control commit
) -> Scan:
    """
    Create a new scan
    """
    scan = Scan(
        user_id=user_id,
        target_url=target_url,
        scan_date=datetime.now(UTC),
    )
    db.add(scan)

    if commit:
        db.commit()  # Flush to database
        db.refresh(scan)  # Reload to get auto-generated ID

    return scan
```

**Dettagli importanti:**

- **Gestione delle transazioni**: Il parametro `commit` permette ai chiamanti di raggruppare le operazioni. Crea scan + crea risultati in un'unica transazione.
- **Refresh dopo il commit**: La riga 30 ricarica l'oggetto dal database. Questo popola i campi generati automaticamente come `id`, `created_at`, `updated_at`.
- **Vincoli di foreign key**: PostgreSQL impone che `test_results.scan_id` referenzi `scans.id`. Se creiamo i risultati dei test prima di committare lo scan, il controllo della foreign key fallisce.

### Bulk Insert per le Performance

Creazione dei risultati dei test in batch:
```python
# repositories/test_result_repository.py:52-67
@staticmethod
def bulk_create(
    db: Session,
    test_results: list[TestResult],
    commit: bool = True
) -> list[TestResult]:
    """
    Create multiple test results in bulk
    """
    db.add_all(test_results)  # Add all at once

    if commit:
        db.commit()
        for result in test_results:
            db.refresh(result)  # Refresh each to get IDs

    return test_results
```

**Perché il bulk insert:**

Insert singoli:
```python
for result in results:
    db.add(result)
    db.commit()  # Commit after each - 4 commits for 4 results
# 4 round trips to database
```

Bulk insert:
```python
db.add_all(results)
db.commit()  # Single commit for all 4 results
# 1 round trip to database
```

Con 4 risultati di test, il bulk insert è 4 volte più veloce.

## Errori Comuni di Implementazione

### Errore 1: Dimenticare di Validare l'Algoritmo JWT

**Sintomo:**
Un attaccante invia un token con `"alg": "none"` e ottiene accesso admin.

**Causa:**
```python
# Problematic code - accepts any algorithm
payload = jwt.decode(token, settings.SECRET_KEY)

# Attacker sends:
# eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9.
# (header: {"alg": "none"}, payload: {"sub": "admin"}, no signature)

# Library accepts it because no algorithm restriction
```

**Correzione:**
```python
# Correct - explicitly allow only HS256
payload = jwt.decode(
    token,
    settings.SECRET_KEY,
    algorithms=["HS256"]  # Reject "none" and other algorithms
)
```

**Perché è importante:**
Senza la validazione dell'algoritmo, la sicurezza JWT viene completamente aggirata. L'attaccante può falsificare token con qualsiasi claim.

### Errore 2: SQL Injection nelle Query Raw

**Sintomo:**
Un attaccante invia `email=admin'--` e ottiene accesso admin.

**Causa:**
```python
# Vulnerable - concatenating user input
email = request.get("email")
query = f"SELECT * FROM users WHERE email = '{email}'"
db.execute(text(query))

# Becomes: SELECT * FROM users WHERE email = 'admin'--'
# Comment (--) removes password check
```

**Correzione:**
```python
# Correct - parameterized query
email = request.get("email")
db.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": email}
)

# SQLAlchemy escapes the email value safely
```

Oppure meglio ancora, usa l'ORM:
```python
# Best - ORM automatically parameterizes
db.query(User).filter(User.email == email).first()
```

**Perché è importante:**
La concatenazione di stringhe tratta l'input dell'utente come codice SQL. La parametrizzazione lo tratta come dati.

### Errore 3: Controllo di Autorizzazione Mancante

**Sintomo:**
Un utente può vedere gli scan di altri utenti cambiando l'ID dello scan nell'URL.

**Causa:**
```python
# Vulnerable - no ownership check
@router.get("/scans/{scan_id}")
async def get_scan(scan_id: int, db: Session = Depends(get_db)):
    scan = ScanRepository.get_by_id(db, scan_id)
    if not scan:
        raise HTTPException(404)
    return scan  # Returns any scan, regardless of ownership
```

**Correzione:**
```python
# Correct - verify ownership
@router.get("/scans/{scan_id}")
async def get_scan(
    scan_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    scan = ScanRepository.get_by_id(db, scan_id)
    if not scan:
        raise HTTPException(404)

    if scan.user_id != current_user.id:
        raise HTTPException(403, detail="Not authorized")

    return scan
```

**Perché è importante:**
L'autenticazione dimostra chi sei. L'autorizzazione dimostra a cosa puoi accedere. Servono entrambe.

## Consigli per il Debug

### Tipo di Problema 1: Il Token JWT Sembra Non Valido Ma il Formato è Corretto

**Problema:** Si ricevono errori 401 usando un token che si decodifica correttamente su jwt.io.

**Come fare il debug:**

1. Controlla la scadenza del token:
```python
import jwt
import datetime

token = "eyJ..."
decoded = jwt.decode(token, options={"verify_signature": False})  # Skip verification for debugging
print(decoded)
# {"sub": "user@example.com", "exp": 1704067200}

exp_time = datetime.datetime.fromtimestamp(decoded["exp"])
now = datetime.datetime.now()
print(f"Expires: {exp_time}, Now: {now}, Valid: {exp_time > now}")
```

2. Verifica che la chiave segreta corrisponda:
```python
# In Python shell with access to settings
from config import settings
print(f"SECRET_KEY: {settings.SECRET_KEY}")
# Make sure this matches the key used to create the token
```

3. Controlla che l'algoritmo corrisponda:
```python
# Decode header without verification
import base64
import json

token = "eyJ..."
header = token.split(".")[0]
# Add padding if needed
padding = 4 - (len(header) % 4)
if padding != 4:
    header += "=" * padding

decoded_header = json.loads(base64.urlsafe_b64decode(header))
print(decoded_header)
# {"alg": "HS256", "typ": "JWT"}

# Make sure "alg" matches settings.ALGORITHM
```

**Cause comuni:**
- Token scaduto (controlla il claim `exp`)
- Chiave segreta errata (ambienti dev vs prod)
- Mancata corrispondenza dell'algoritmo (creato con HS256, verificato con RS256)

### Tipo di Problema 2: Lo Scanner Va in Timeout su Tutti i Test

**Problema:** Tutti i risultati dei test restituiscono `status="error"` con messaggi di timeout.

**Come fare il debug:**

1. Verifica che il target sia raggiungibile:
```bash
# From inside backend container
docker exec -it apisec_backend_dev bash
curl -v https://target-api.com/endpoint

# Check:
# - Does connection succeed?
# - What's the response time?
# - Are there redirects?
```

2. Controlla le impostazioni di timeout:
```python
# config.py:51
SCANNER_CONNECTION_TIMEOUT: int = 180  # 3 minutes

# If target is slower, increase this
```

3. Guarda l'errore effettivo:
```python
# services/scan_service.py exception block shows error
except Exception as e:
    details=f"Scanner error: {str(e)}"

# Check what the exception says:
# - "Connection timeout" = target slow or unreachable
# - "Name resolution failed" = DNS issue
# - "SSL certificate verify failed" = TLS problem
```

4. Prova con un target noto e funzionante:
```bash
# Test with httpbin (always responsive)
curl -X POST http://localhost:8000/scans/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "target_url": "https://httpbin.org/get",
    "tests_to_run": ["rate_limit"],
    "max_requests": 10
  }'
```

**Cause comuni:**
- Il target richiede una VPN o è dietro un firewall
- Il target applica rate limiting allo scanner (ironia della sorte)
- Risoluzione DNS che fallisce nella rete Docker
- Problemi con i certificati TLS (self-signed, scaduti)

### Tipo di Problema 3: Errori di Deadlock del Database

**Problema:** Errori `DeadlockDetected` intermittenti sotto carico concorrente.

**Come fare il debug:**

1. Controlla l'ordine delle transazioni:
```sql
-- PostgreSQL query to see locks
SELECT
  pid,
  state,
  query_start,
  state_change,
  query
FROM pg_stat_activity
WHERE state = 'active';
```

2. Cerca i pattern di transazione:
```python
# Problematic pattern - different order
# Thread 1:
UPDATE scans SET ... WHERE id = 1;
UPDATE test_results SET ... WHERE scan_id = 1;

# Thread 2:
UPDATE test_results SET ... WHERE scan_id = 2;
UPDATE scans SET ... WHERE id = 2;

# Thread 1 locks scans.id=1, waits for test_results
# Thread 2 locks test_results.scan_id=2, waits for scans
# = Deadlock
```

3. Correggi con un ordinamento coerente:
```python
# Both threads update in same order = no deadlock
# Always update parent (scans) before child (test_results)
UPDATE scans SET ... WHERE id = ?;
UPDATE test_results SET ... WHERE scan_id = ?;
```

**Cause comuni:**
- Più transazioni che aggiornano le stesse righe in ordini diversi
- Transazioni di lunga durata che mantengono i lock
- Indici mancanti che causano table scan che bloccano molte righe

## Principi di Organizzazione del Codice

### Perché le Route Restano Sottili

Le route in `routes/scans.py` sono volutamente semplici:
```python
@router.post("/", response_model=ScanResponse)
async def create_scan(
    request: Request,
    scan_request: ScanRequest,
    db: Session = Depends(get_db),
    current_user: UserResponse = Depends(get_current_user),
) -> ScanResponse:
    return ScanService.run_scan(db, current_user.id, scan_request)
```

Solo 3 righe:
1. La dependency injection fornisce db e current_user
2. Chiama il service layer
3. Restituisce il risultato

**Tutta la logica di business vive nei service.** Le route gestiscono gli aspetti HTTP:
- Parsing della richiesta
- Validazione con Pydantic
- Controllo dell'autenticazione
- Serializzazione della risposta

Questo rende le route facili da testare:
```python
# Test route with mocked service
def test_create_scan(mock_service):
    mock_service.run_scan.return_value = fake_scan

    response = client.post("/scans/", json={...})

    assert response.status_code == 201
    assert mock_service.run_scan.called
```

Non serve fare il mock di database, scanner o di alcuna logica complessa. Il service è mockato, la route gestisce solo HTTP.

### Convenzioni di Naming

- `*Repository` = Classi di accesso ai dati (UserRepository, ScanRepository)
- `*Service` = Classi di logica di business (AuthService, ScanService)
- `*Scanner` = Implementazioni dei test di sicurezza (SQLiScanner, AuthScanner)
- `*Schema` = Modelli di validazione Pydantic (UserCreate, ScanResponse)
- funzioni `get_*` = Dependency di FastAPI (get_db, get_current_user)
- `_private_method` = Helper interno (non parte dell'interfaccia pubblica)

Seguire questi pattern rende più facile trovare il codice. Devi aggiungere una query al database? Guarda nei repository. Devi cambiare la logica di business? Guarda nei service.

## Estendere il Codice

### Aggiungere un Nuovo Test di Sicurezza (Esempio XSS)

Vuoi aggiungere il rilevamento di XSS (Cross-Site Scripting)? Ecco il processo completo:

**1. Crea lo scanner** `backend/scanners/xss_scanner.py`:
```python
from .base_scanner import BaseScanner
from .payloads import XSSPayloads
from core.enums import TestType, ScanStatus, Severity
from schemas.test_result_schemas import TestResultCreate

class XSSScanner(BaseScanner):
    """
    Tests for Cross-Site Scripting vulnerabilities
    """

    def scan(self) -> TestResultCreate:
        reflected_test = self._test_reflected_xss()

        if reflected_test["vulnerable"]:
            return TestResultCreate(
                test_name=TestType.XSS,
                status=ScanStatus.VULNERABLE,
                severity=Severity.HIGH,
                details=f"Reflected XSS detected: {reflected_test['payload']}",
                evidence_json=reflected_test,
                recommendations_json=[
                    "Encode user input before rendering in HTML",
                    "Use Content-Security-Policy headers",
                    "Validate input on server side",
                ],
            )

        return TestResultCreate(
            test_name=TestType.XSS,
            status=ScanStatus.SAFE,
            severity=Severity.INFO,
            details="No XSS vulnerabilities detected",
            evidence_json=reflected_test,
            recommendations_json=[],
        )

    def _test_reflected_xss(self) -> dict[str, Any]:
        payloads = XSSPayloads.get_basic_payloads()

        for payload in payloads:
            response = self.make_request("GET", f"/?q={payload}")

            # Check if payload appears unencoded in response
            if payload in response.text:
                return {
                    "vulnerable": True,
                    "payload": payload,
                    "status_code": response.status_code,
                }

        return {"vulnerable": False}
```

**2. Aggiungi il valore all'enum** in `backend/core/enums.py:19-26`:
```python
class TestType(str, Enum):
    RATE_LIMIT = "rate_limit"
    AUTH = "auth"
    SQLI = "sqli"
    IDOR = "idor"
    XSS = "xss"  # New test type
```

**3. Registra lo scanner** in `backend/services/scan_service.py:32-38`:
```python
scanner_mapping: dict[TestType, type[BaseScanner]] = {
    TestType.RATE_LIMIT: RateLimitScanner,
    TestType.AUTH: AuthScanner,
    TestType.SQLI: SQLiScanner,
    TestType.IDOR: IDORScanner,
    TestType.XSS: XSSScanner,  # Register new scanner
}
```

**4. Aggiungi i payload** in `backend/scanners/payloads.py:250-280`:
```python
class XSSPayloads:
    BASIC_XSS = [
        "<script>alert('XSS')</script>",
        "<img src=x onerror=alert('XSS')>",
        "<svg/onload=alert('XSS')>",
    ]

    @classmethod
    def get_basic_payloads(cls) -> list[str]:
        return cls.BASIC_XSS
```

**5. Aggiorna il frontend** in `frontend/src/config/constants.ts`:
```typescript
export const SCAN_TEST_TYPES = {
  // ... existing
  XSS: 'xss',
} as const;

export const TEST_TYPE_LABELS: Record<ScanTestType, string> = {
  // ... existing
  [SCAN_TEST_TYPES.XSS]: 'Cross-Site Scripting',
};
```

Tutto qui. Nessuna modifica al database necessaria (l'enum TestType si aggiorna automaticamente). Nessuna modifica alle route (passano i tipi di test così come sono). Solo l'implementazione dello scanner e la registrazione.

## Prossimi Passi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) contiene idee di estensione, dall'aggiunta del rilevamento di stored XSS all'implementazione di plugin scanner personalizzati
2. **Modifica gli scanner** - Cambia le soglie di timing del payload SQLi, aggiungi nuove tecniche di bypass dell'autenticazione, implementa il rilevamento di DOM-based XSS
3. **Leggi i progetti correlati** - Il progetto docker-security-audit si basa sui concetti di scansione dei container, network-traffic-analyzer approfondisce l'analisi dei pacchetti
