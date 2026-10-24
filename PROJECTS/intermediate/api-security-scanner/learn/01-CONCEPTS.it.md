# 01-CONCEPTS.md

# Concetti di sicurezza fondamentali

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci questo progetto. Non sono semplici definizioni. Approfondiremo perché sono importanti e come funzionano davvero.

## Rate Limiting

### Cos'è

Il rate limiting controlla quante richieste un client può fare alla tua API in una data finestra temporale. È come un buttafuori in un locale che conta quante volte qualcuno cerca di entrare. Se qualcuno sta martellando la porta 100 volte al minuto, qualcosa non va.

Il concetto è semplice: traccia le richieste in base a un identificatore (indirizzo IP, ID utente, API key) e rifiuta le richieste che superano una soglia. Un'implementazione di base potrebbe essere "100 richieste al minuto per indirizzo IP." Quando arriva la richiesta 101 entro quel minuto, restituisci HTTP 429 Too Many Requests.

### Perché è importante

Senza rate limiting, un singolo attaccante può mettere fuori uso l'intero servizio. Nel 2016, la botnet Mirai ha lanciato un attacco DDoS contro il servizio DNS di Dyn inondandolo di richieste. Siti importanti come Twitter, Netflix e Reddit sono andati giù perché Dyn non aveva un rate limiting efficace a livello DNS.

Ma non si tratta solo di denial of service. Il rate limiting blocca anche:
- **Credential stuffing** - attaccanti che provano migliaia di password rubate contro il tuo endpoint di login
- **Data scraping** - concorrenti o malintenzionati che estraggono l'intero catalogo prodotti attraverso la tua API
- **Attacchi brute force** - provare ogni valore possibile per craccare password, API key o trovare ID utente validi
- **Attacchi sui costi** - far lievitare la tua bolletta AWS attivando ripetutamente operazioni costose

L'incidente di scraping di Venmo del 2020 ha esposto i dati delle transazioni di milioni di utenti. Gli attaccanti hanno fatto richieste illimitate all'API pubblica di Venmo e raccolto dati su chi pagava chi. Il rate limiting avrebbe rilevato e bloccato immediatamente questo pattern.

### Come funziona

Guarda come questo progetto rileva il rate limiting in `backend/scanners/rate_limit_scanner.py`:

```python
# Lines 62-145
def _detect_rate_limiting(self, test_request_count: int = 20) -> dict[str, Any]:
    """
    Detect rate limiting by analyzing headers and response patterns
    """
    rate_limit_patterns = RateLimitBypassPayloads.get_header_patterns()
    
    results = {
        "rate_limit_detected": False,
        "rate_limit_headers": {},
        "enforcement_status": None,
    }
    
    for attempt in range(1, test_request_count + 1):
        response = self.make_request("GET", "/")
        
        # Check for standard rate limit headers
        for header_type, pattern in rate_limit_patterns.items():
            for header_name, header_value in headers_lower.items():
                if re.search(pattern, header_name, re.IGNORECASE):
                    results["rate_limit_headers"][header_type] = {
                        "header_name": header_name,
                        "value": header_value,
                    }
                    results["rate_limit_detected"] = True
        
        # Check if we hit the limit
        if response.status_code == 429:
            results["enforcement_status"] = "ACTIVE"
            results["attempts_until_limit"] = attempt
            break
```

Questo codice fa richieste finché non ottiene una risposta 429 o non trova header di rate limit. Gli header seguono pattern come `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. Provengono dalla bozza di standard IETF che la maggior parte delle API segue.

Lo scanner distingue tra tre stati:
1. **Nessun rate limiting** - invia 20 richieste, non viene mai bloccato né vede header
2. **Solo header** - pubblicizza i limiti negli header ma non li applica mai con un 429
3. **Applicazione attiva** - blocca effettivamente le richieste quando i limiti vengono superati

### Attacchi comuni

1. **Spoofing degli header IP** - Molti rate limiter si fidano di header come `X-Forwarded-For` per identificare i client. Gli attaccanti ruotano indirizzi IP falsi in questo header per aggirare i limiti. Lo scanner lo testa in `rate_limit_scanner.py:218-255`.

2. **Attacchi distribuiti** - Usare botnet o cloud provider per attaccare da molti IP reali contemporaneamente, restando sotto i limiti per IP mentre si sovraccarica il servizio nel complesso.

3. **Bypass tramite variazione dell'endpoint** - Alcuni rate limiter sono case sensitive o non considerano le variazioni dell'URL. Prova `/api/users`, `/API/users`, `/api/users/`, `/api/users?`, ecc. Codice in `rate_limit_scanner.py:257-286`.

4. **Esaurimento delle risorse senza limiti** - Anche con il rate limiting, se i limiti sono troppo alti (1000 richieste/minuto), un attaccante può comunque causare danni. L'interruzione di Cloudflare del 2022 è stata causata da un pattern regex che consumava CPU - anche le richieste soggette a rate limiting si sommavano.

### Strategie di difesa

Implementa il rate limiting su più livelli. Questo progetto mostra come rilevarlo, ma ecco come costruirlo:

**Livello 1: edge/CDN**
Usa Cloudflare, AWS WAF o simili per bloccare gli abusi evidenti prima che raggiungano la tua infrastruttura. È economico e veloce perché avviene nelle edge location.

**Livello 2: API gateway**
Kong, AWS API Gateway o nginx possono applicare rate limit per IP, per API key o per utente. Questo intercetta la maggior parte degli abusi.

**Livello 3: livello applicativo**
Il backend di questo progetto usa SlowAPI (vedi `backend/factory.py:34-36`):
```python
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

E applica limiti a endpoint specifici (`backend/routes/auth.py:25-42`):
```python
@router.post("/register", ...)
@limiter.limit(settings.API_RATE_LIMIT_REGISTER)  # "15/minute" from config
async def register(request: Request, ...):
    return AuthService.register_user(db, user_data)
```

**Difese critiche:**
- Non fidarti di header forniti dal client come `X-Forwarded-For` senza validazione
- Usa più identificatori - IP, ID utente, API key combinati
- Implementa un backoff esponenziale - blocchi più lunghi per violazioni ripetute  
- Restituisci l'header `Retry-After` per dire ai client quando riprovare
- Registra nei log le violazioni del rate limit per il monitoraggio di sicurezza

## Broken Authentication

### Cos'è

L'autenticazione dimostra chi sei. Quando è difettosa, gli attaccanti possono aggirarla, impersonare utenti o accedere ai sistemi senza credenziali valide. Questa è la OWASP API2:2023 - Broken Authentication.

Meccanismi di autenticazione comuni per le API:
- **Session token** - il server genera un token, lo memorizza, lo valida a ogni richiesta
- **Token JWT** - il server firma un token, il client lo rimanda indietro, il server verifica la firma
- **API key** - secret di lunga durata che identificano le applicazioni
- **OAuth 2.0** - autorizzazione delegata con access token

Questo progetto si concentra sui JWT perché sono molto usati e frequentemente configurati in modo errato.

### Perché è importante

L'attacco SolarWinds del 2020 ha usato token di autenticazione rubati per accedere alle reti dei clienti. Una volta dentro, gli attaccanti si sono mossi lateralmente perché l'autenticazione non veniva verificata correttamente a ogni confine tra servizi.

Nel 2018, una vulnerabilità nella funzione "Visualizza come" di Facebook ha permesso agli attaccanti di rubare gli access token di 50 milioni di utenti. I token funzionavano perché Facebook non li validava correttamente quando venivano usati.

La broken authentication è al 2° posto nella API Top 10 di OWASP perché è sia comune sia devastante. Aggira l'autenticazione e spesso ottieni pieno accesso ai dati degli utenti, alle funzioni di amministrazione e ai sistemi interni.

### Come funziona

Un JWT (JSON Web Token) ha tre parti separate da punti:
```
header.payload.signature
```

Esempio:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyQGV4YW1wbGUuY29tIn0.signature_here
```

Decodifica l'header (base64):
```json
{"alg": "HS256", "typ": "JWT"}
```

Decodifica il payload:
```json
{"sub": "user@example.com"}
```

La firma dimostra che il token non è stato manomesso. Viene creata con:
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

Una validazione corretta deve:
1. Verificare la firma usando la secret key del server
2. Controllare che l'algoritmo sia quello che ti aspetti (HS256, RS256, ecc.)
3. Validare il tempo di scadenza, se presente
4. Rifiutare i token con l'algoritmo "none"

Guarda come questo progetto valida i token in `backend/core/security.py:48-62`:
```python
def decode_token(token: str) -> dict[str, str]:
    """
    Decode and verify a JWT token
    """
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]  # Explicitly specify allowed algorithms
        )
        return payload
    except JWTError as e:
        raise ValueError(f"Invalid token: {str(e)}") from e
```

### Attacchi comuni

1. **Attacco con algoritmo none** - JWT supporta un algoritmo chiamato "none" che significa "nessuna firma richiesta." Alcune librerie accettano questi token anche se non sono firmati. Lo scanner lo testa in `auth_scanner.py:168-213`:

```python
def _test_none_algorithm(self) -> dict[str, Any]:
    """
    Test if server accepts JWT with 'none' algorithm
    """
    header, payload, signature = self.auth_token.split(".")
    
    none_variants = ["none", "None", "NONE", "nOnE"]  # Case variations
    
    for variant in none_variants:
        malicious_header = self._base64url_encode(
            json.dumps({"alg": variant, "typ": "JWT"})
        )
        malicious_token = f"{malicious_header}.{payload}."  # No signature
        
        response = self.make_request(
            "GET", "/",
            headers={"Authorization": f"Bearer {malicious_token}"}
        )
        
        if response.status_code == 200:
            return {"vulnerable": True, "algorithm_variant": variant}
```

2. **Autenticazione mancante** - Endpoint che dovrebbero richiedere l'autenticazione non la controllano. Test in `auth_scanner.py:89-131`:
```python
def _test_missing_authentication(self) -> dict[str, Any]:
    """
    Test if endpoint requires authentication
    """
    session_without_auth = self.session.__class__()  # New session, no token
    
    response = session_without_auth.get(self.target_url, timeout=...)
    
    if response.status_code == 200:
        return {"vulnerable": True, "description": "Endpoint accessible without authentication"}
```

3. **Secret key deboli** - Se il secret del JWT è debole ("secret", "password", il nome dell'azienda), gli attaccanti possono forzarlo con brute force e falsificare token validi. Tool come hashcat possono provare milioni di secret al secondo.

4. **Algorithm confusion** - Il server si aspetta RS256 (asimmetrico) ma accetta HS256 (simmetrico). L'attaccante usa la chiave pubblica come secret HMAC per falsificare i token.

### Strategie di difesa

Il progetto implementa una corretta validazione dei JWT. Ecco cosa fare:

**Creazione del token** (`core/security.py:33-56`):
```python
def create_access_token(data: dict[str, str], expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=1440))
    to_encode.update({"exp": expire})
    
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,  # Must be cryptographically random, 256+ bits
        algorithm=settings.ALGORITHM  # "HS256"
    )
    return encoded_jwt
```

**Validazione del token** - Avviene in `core/dependencies.py:17-49`:
```python
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db),
) -> UserResponse:
    try:
        payload = decode_token(credentials.credentials)
        email: str | None = payload.get("sub")
        
        if email is None:
            raise HTTPException(status_code=401, detail="Invalid credentials")
        
        user = UserRepository.get_by_email(db, email)
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        
        return UserResponse.model_validate(user)
```

**Difese critiche:**
- Usa un secret casuale robusto (generalo con `openssl rand -hex 32`)
- Specifica esplicitamente gli algoritmi consentiti, rifiuta "none"
- Verifica le firme prima di fidarti dei dati del payload
- Implementa la scadenza dei token e controllala
- Richiedi l'autenticazione su tutti gli endpoint sensibili
- Usa access token di breve durata (15 minuti) con refresh token

### Insidie comuni

**Errore 1: fidarsi di token non verificati**
```python
# Bad - decodes without verifying signature
import base64, json
header, payload, sig = token.split(".")
data = json.loads(base64.b64decode(payload))
user_id = data["user_id"]  # Attacker controlled!

# Good - verifies signature first
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
user_id = payload["user_id"]  # Safe to use
```

**Errore 2: non controllare la scadenza**
```python
# Bad - no expiration check
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"], options={"verify_exp": False})

# Good - validates expiration
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])  # verify_exp=True by default
```

## SQL Injection

### Cos'è

La SQL injection si verifica quando inserisci dati non fidati direttamente nelle query SQL senza un corretto escaping. L'attaccante invia un input malevolo che cambia la logica della query.

Esempio di codice vulnerabile:
```python
query = f"SELECT * FROM users WHERE email = '{email}'"
```

Se l'attaccante invia `email = "admin'--"`, la query diventa:
```sql
SELECT * FROM users WHERE email = 'admin'--'
```

Il `--` commenta il resto, trasformando una ricerca in un bypass.

### Perché è importante

La SQL injection è nella OWASP Top 10 da due decenni, eppure causa ancora breach. Il breach di Equifax del 2017 ha esposto 147 milioni di record attraverso una vulnerabilità diversa (Apache Struts), ma una volta dentro, gli attaccanti hanno usato la SQL injection per estrarre i dati.

Nel 2019, una SQL injection in Canva ha esposto indirizzi email e password di 137 milioni di utenti. Gli attaccanti hanno manipolato campi di input che venivano concatenati nelle query.

I framework moderni hanno reso la SQLi meno comune, ma si presenta ancora quando gli sviluppatori:
- Concatenano l'input dell'utente nelle query
- Usano male le funzionalità di query raw degli ORM
- Costruiscono SQL dinamico da dati controllati dall'utente
- Si fidano dell'input proveniente da sorgenti interne (che necessita comunque di validazione)

### Come funziona

Ci sono tre tipi principali di SQL injection che questo progetto rileva:

**1. SQLi error-based** - Provocare errori del database che fanno trapelare informazioni

Lo scanner invia payload come `' OR '1'='1` e controlla se nella risposta compaiono messaggi di errore del database (`sqli_scanner.py:51-94`):

```python
def _test_error_based_sqli(self) -> dict[str, Any]:
    """
    Test for error based SQL injection
    """
    error_signatures = SQLiPayloads.get_error_signatures()  # MySQL, Postgres, MSSQL, Oracle
    basic_payloads = SQLiPayloads.BASIC_AUTHENTICATION_BYPASS
    
    for payload in basic_payloads:
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
                        "error_signature": signature
                    }
```

Le firme di errore sono definite in `scanners/payloads.py:9-30`:
```python
ERROR_SIGNATURES = {
    "mysql": ["sql syntax", "mysql_fetch", "mysql error"],
    "postgres": ["postgresql", "pg_query", "syntax error"],
    "mssql": ["sqlserver jdbc driver", "sqlexception"],
    "oracle": ["ora-", "oracle.jdbc", "pl/sql"],
}
```

**2. SQLi boolean-based blind** - Nessun errore mostrato, ma le risposte differiscono per condizioni vere e false

Invia query con condizioni sempre vere (`1=1`) e sempre false (`1=2`) e confronta le lunghezze delle risposte:

```python
def _test_boolean_based_sqli(self) -> dict[str, Any]:
    """
    Test for boolean based blind SQL injection
    """
    baseline_response = self.make_request("GET", "/?id=1")
    baseline_length = len(baseline_response.text)
    
    # Try true conditions: ' AND 1=1--
    true_payloads = [p for p in boolean_payloads if "AND '1'='1" in p or "AND 1=1" in p]
    true_lengths = []
    for payload in true_payloads:
        response = self.make_request("GET", f"/?id={payload}")
        true_lengths.append(len(response.text))
    
    # Try false conditions: ' AND 1=2--
    false_payloads = [p for p in boolean_payloads if "AND '1'='2" in p]
    false_lengths = []
    for payload in false_payloads:
        response = self.make_request("GET", f"/?id={payload}")
        false_lengths.append(len(response.text))
    
    avg_true = statistics.mean(true_lengths)
    avg_false = statistics.mean(false_lengths)
    length_diff = abs(avg_true - avg_false)
    
    if length_diff > 100:  # Significant difference
        return {"vulnerable": True, "length_difference": length_diff}
```

**3. SQLi time-based blind** - Costringere il database a rallentare, misurare il tempo di risposta

Payload come `'; SELECT SLEEP(5)--` fanno andare in pausa il database. Se la risposta impiega 5+ secondi, è vulnerabile:

```python
def _test_time_based_sqli(self, delay_seconds: int = 5) -> dict[str, Any]:
    """
    Test for time based blind SQL injection
    Uses baseline timing comparison with statistical analysis
    """
    # Establish normal response time
    baseline_mean, baseline_stdev = self.get_baseline_timing("/")
    expected_delay_time = baseline_mean + delay_seconds
    
    delay_payloads = {
        "mysql": [p for p in all_time_payloads if "SLEEP" in p],
        "postgres": [p for p in all_time_payloads if "pg_sleep" in p],
        "mssql": [p for p in all_time_payloads if "WAITFOR" in p],
    }
    
    for db_type, payloads in delay_payloads.items():
        for payload in payloads:
            delay_times = []
            for _ in range(3):  # Multiple samples
                response = self.make_request("GET", f"/?id={payload}", timeout=delay_seconds + 10)
                elapsed = getattr(response, "request_time", 0.0)
                delay_times.append(elapsed)
            
            avg_delay = statistics.mean(delay_times)
            
            if avg_delay >= expected_delay_time - 1:  # Within 1 second of expected
                return {"vulnerable": True, "database_type": db_type, "response_time": f"{avg_delay:.3f}s"}
```

### Attacchi comuni

I payload si trovano in `scanners/payloads.py`. Ecco le tecniche:

1. **Bypass dell'autenticazione** (`payloads.py:33-48`):
```python
BASIC_AUTHENTICATION_BYPASS = [
    "' OR '1'='1",        # Makes WHERE clause always true
    "' OR 1=1--",         # Comments out rest of query
    "admin'--",           # Logs in as admin by ending query early
    ") or '1'='1--",      # Escapes parentheses in complex queries
]
```

2. **Estrazione union-based** (`payloads.py:50-60`):
```python
UNION_BASED = [
    "' UNION SELECT NULL--",                    # Find column count
    "' UNION SELECT username,password FROM users--",  # Extract data
    "' UNION SELECT NULL,version()--",          # Get database version
]
```

3. **Stacked query** (`payloads.py:92-99`):
```python
STACKED_QUERIES = [
    "'; DROP TABLE users--",                    # Delete data
    "'; INSERT INTO users VALUES('hacker','password')--",  # Add accounts
    "'; EXEC xp_cmdshell('whoami')--",         # Run OS commands (MSSQL)
]
```

### Strategie di difesa

Non concatenare mai l'input dell'utente nelle query SQL. Usa query parametrizzate (prepared statement) che trattano i dati come dati, non come codice.

**Codice vulnerabile:**
```python
# BAD - string concatenation
email = request.get("email")
query = f"SELECT * FROM users WHERE email = '{email}'"
result = db.execute(query)
```

**Codice sicuro con SQLAlchemy:**
```python
# GOOD - parameterized query
email = request.get("email")
result = db.query(User).filter(User.email == email).first()

# SQLAlchemy generates safe SQL:
# SELECT * FROM users WHERE email = :email_1
# Parameters: {'email_1': 'user@example.com'}
```

Questo progetto usa il repository pattern per mantenere sicuro l'SQL. Guarda `repositories/user_repository.py:25-35`:
```python
@staticmethod
def get_by_email(db: Session, email: str) -> User | None:
    """
    Get user by email address
    """
    return db.query(User).filter(User.email == email).first()
```

`.filter(User.email == email)` di SQLAlchemy crea automaticamente una query parametrizzata. Il valore dell'email viene passato separatamente dalla stringa SQL, quindi non può iniettare codice.

**Difese critiche:**
- Usa ORM come SQLAlchemy, Django ORM o Prisma
- Quando l'SQL raw è necessario, usa query parametrizzate
- Valida i tipi di input (se ti aspetti un int, convertilo in int e gestisci gli errori)
- Usa account del database con privilegi minimi (non connetterti come admin)
- Disabilita i messaggi di errore in produzione (non far trapelare dettagli del database)

### Insidie comuni

**Errore 1: usare raw() dell'ORM in modo scorretto**
```python
# Bad - raw SQL with concatenation
email = request.get("email")
query = f"SELECT * FROM users WHERE email = '{email}'"
result = db.session.execute(text(query))  # Still vulnerable!

# Good - raw SQL with parameters
result = db.session.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": email}
)
```

**Errore 2: fidarsi dei dati "interni"**
```python
# Bad - assumes data from another service is safe
external_api_result = fetch_from_partner()
user_id = external_api_result["user_id"]
query = f"SELECT * FROM orders WHERE user_id = {user_id}"  # Can still be exploited
```

## IDOR/BOLA (Broken Object Level Authorization)

### Cos'è

IDOR (Insecure Direct Object Reference) e BOLA (Broken Object Level Authorization) sono la stessa cosa. La API Top 10 2023 di OWASP la chiama BOLA e la colloca al 1° posto.

La vulnerabilità: la tua API permette agli utenti di accedere agli oggetti tramite ID, ma non controlla se quell'utente debba avere accesso a quello specifico oggetto.

Esempio:
```
GET /api/users/123/orders
```

L'API controlla se sei autenticato (hai un token valido), ma non controlla se l'utente 123 sei TU. Quindi puoi cambiare l'ID in 124, 125, 126 e visualizzare gli ordini di altri utenti.

### Perché è importante

IDOR è al 1° posto nella OWASP API Security Top 10 2023 perché è ovunque ed è facile da sfruttare. Non servono tool speciali, basta cambiare un numero nell'URL.

Breach reali:
- **USPS Informed Delivery (2018)** - Cambiando gli indirizzi email nelle richieste si potevano visualizzare le scansioni della posta di chiunque
- **T-Mobile (2022)** - Un endpoint API ha fatto trapelare dati su 37 milioni di clienti consentendo l'enumerazione degli ID
- **Parler (2021)** - Gli ID sequenziali dei post hanno permesso ai ricercatori di scaricare l'intera piattaforma (70TB) incrementando gli ID

Il danno non è solo l'esposizione di dati. Nel 2019, dei ricercatori hanno trovato un IDOR in un'app di car sharing che permetteva loro di sbloccare e avviare qualsiasi auto della flotta cambiando l'ID del veicolo.

### Come funziona

Lo scanner testa due pattern:

**1. Enumerazione degli ID** - ID sequenziali o prevedibili rendono facile indovinare valori validi

Estrai gli ID dalle risposte dell'API, poi prova delle variazioni (`idor_scanner.py:119-153`):

```python
def _extract_ids_from_response(self) -> list[Any]:
    """
    Extract potential IDs from API response
    """
    response = self.make_request("GET", "/")
    
    # Look for UUIDs: a1b2c3d4-1234-5678-9abc-def012345678
    uuid_pattern = r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
    uuids = re.findall(uuid_pattern, response.text, re.IGNORECASE)
    
    # Look for numeric IDs: "id": 123
    numeric_id_pattern = r'"id"\s*:\s*(\d+)'
    numeric_ids = re.findall(numeric_id_pattern, response.text)
    
    ids = []
    ids.extend(uuids[:3])
    ids.extend([int(nid) for nid in numeric_ids[:3]])
    
    return ids
```

Poi verifica se quegli ID possono essere manipolati (`idor_scanner.py:155-204`):

```python
def _test_numeric_id_manipulation(self, extracted_ids: list[Any]) -> dict[str, Any]:
    """
    Test numeric ID manipulation for IDOR
    """
    numeric_ids = [id_val for id_val in extracted_ids if isinstance(id_val, int)]
    
    base_id = numeric_ids[0]
    test_ids = [0, -1, 1, 2, 9999, 99999, 999999]  # Common patterns
    
    accessible_unauthorized = []
    
    for test_id in test_ids:
        if test_id == base_id:
            continue
        
        response = self.make_request("GET", f"/{test_id}")
        
        if response.status_code == 200:  # Should be 403 or 404
            accessible_unauthorized.append({
                "id": test_id,
                "status_code": response.status_code,
                "response_length": len(response.text)
            })
    
    if accessible_unauthorized:
        return {"vulnerable": True, "unauthorized_access": accessible_unauthorized}
```

**2. Pattern prevedibili** - ID sequenziali rivelano il conteggio totale e permettono lo scraping

Se ottieni gli ID `[100, 101, 102]` facendo richieste, sai che ci sono almeno 102 oggetti e puoi enumerarli tutti.

```python
def _test_predictable_id_patterns(self) -> dict[str, Any]:
    """
    Test if IDs follow predictable patterns
    """
    ids1 = self._extract_ids_from_response()
    ids2 = self._extract_ids_from_response()
    
    numeric_ids1 = [id for id in ids1 if isinstance(id, int)]
    numeric_ids2 = [id for id in ids2 if isinstance(id, int)]
    
    if len(numeric_ids1) >= 2:
        diff1 = abs(numeric_ids1[1] - numeric_ids1[0])
        
        if len(numeric_ids2) >= 2:
            diff2 = abs(numeric_ids2[1] - numeric_ids2[0])
            
            if diff1 == diff2 and diff1 == 1:  # Increments by 1 each time
                return {
                    "vulnerable": True,
                    "pattern_type": "Sequential IDs",
                    "id_difference": diff1
                }
```

### Attacchi comuni

Payload di attacco in `scanners/payloads.py:179-216`:

1. **Manipolazione degli ID numerici**:
```python
NUMERIC_ID_MANIPULATIONS = [
    0,        # First record
    -1,       # May wrap to last record  
    1,        # Common starting ID
    9999,     # Try common ranges
    999999,   # Higher ranges
]
```

2. **Manipolazione degli ID stringa**:
```python
STRING_ID_MANIPULATIONS = [
    "admin",               # Privileged accounts
    "root",
    "test",
    "../../../etc/passwd", # Path traversal attempts
]
```

3. **Indovinare gli UUID** - Anche se gli UUID sono casuali, implementazioni scadenti potrebbero:
- Usare seed prevedibili
- Accettare UUID composti da soli zeri o soli uno
- Non validare il formato dell'UUID

### Strategie di difesa

La correzione è concettualmente semplice: controlla l'autorizzazione prima di restituire i dati. Ma devi farlo su ogni singolo endpoint.

**Codice vulnerabile:**
```python
@router.get("/api/documents/{doc_id}")
async def get_document(doc_id: int, user: User = Depends(get_current_user)):
    document = db.query(Document).filter(Document.id == doc_id).first()
    return document  # IDOR vulnerability - didn't check ownership
```

**Codice corretto:**
```python
@router.get("/api/documents/{doc_id}")
async def get_document(doc_id: int, user: User = Depends(get_current_user)):
    document = db.query(Document).filter(
        Document.id == doc_id,
        Document.owner_id == user.id  # Authorization check
    ).first()
    
    if not document:
        raise HTTPException(status_code=404, detail="Document not found")
    
    return document
```

Questo progetto dimostra una corretta autorizzazione in `services/scan_service.py:67-84`:

```python
@staticmethod
def get_scan_by_id(db: Session, scan_id: int, user_id: int) -> ScanResponse:
    """
    Get scan by ID with authorization check
    """
    scan = ScanRepository.get_by_id(db, scan_id)
    
    if not scan:
        raise HTTPException(status_code=404, detail="Scan not found")
    
    # CRITICAL: Check if user owns this scan
    if scan.user_id != user_id:
        raise HTTPException(status_code=403, detail="Not authorized to access this scan")
    
    return ScanResponse.model_validate(scan)
```

**Difese critiche:**
- Controlla la proprietà dell'oggetto a ogni accesso
- Usa UUID invece di interi sequenziali (ma controlla comunque l'autorizzazione!)
- Implementa la row-level security nel tuo database
- Restituisci 404 per gli accessi non autorizzati (non rivelare l'esistenza con un 403)
- Registra nei log i pattern di accesso sospetti (molti ID provati rapidamente)

### Insidie comuni

**Errore 1: controllare l'autorizzazione solo sulle scritture**
```python
# Bad - checks auth for POST but not GET
@router.post("/api/orders/{order_id}/cancel")
async def cancel_order(order_id: int, user: User = Depends()):
    order = db.get(Order, order_id)
    if order.user_id != user.id:  # Auth check here
        raise HTTPException(403)
    order.status = "cancelled"

@router.get("/api/orders/{order_id}")
async def get_order(order_id: int):
    return db.get(Order, order_id)  # No auth check - vulnerable
```

**Errore 2: fidarsi degli UUID**
```python
# Bad - assumes UUID means authorization not needed
@router.get("/api/documents/{doc_uuid}")
async def get_document(doc_uuid: str):
    return db.query(Document).filter(Document.uuid == doc_uuid).first()
    # Still vulnerable! UUIDs are hard to guess but not impossible

# Good - check authorization even with UUIDs
@router.get("/api/documents/{doc_uuid}")
async def get_document(doc_uuid: str, user: User = Depends()):
    doc = db.query(Document).filter(
        Document.uuid == doc_uuid,
        Document.owner_id == user.id
    ).first()
    if not doc:
        raise HTTPException(404)
    return doc
```

## Come si relazionano questi concetti

Queste vulnerabilità compaiono spesso insieme e si amplificano a vicenda:

```
No Rate Limiting
    ↓
  enables
    ↓
Rapid IDOR Enumeration
    ↓
 combined with
    ↓
Broken Authentication
    ↓
  allows
    ↓
SQL Injection Exploitation
    ↓
   results in
    ↓
Complete Data Breach
```

Esempio reale: un'API con un rate limiting debole permette a un attaccante di:
1. Enumerare rapidamente gli ID utente (IDOR)
2. Provare password rubate su ciascun ID (broken auth)
3. Una volta autenticato, iniettare SQL per fare il dump dell'intero database

La difesa richiede di intervenire su tutti i livelli.

## Standard e framework di settore

### OWASP Top 10

Questo progetto affronta:

- **API1:2023 Broken Object Level Authorization** - Lo scanner IDOR (`idor_scanner.py`) rileva i controlli di autorizzazione mancanti
- **API2:2023 Broken Authentication** - Lo scanner di autenticazione (`auth_scanner.py`) testa le vulnerabilità JWT, l'autenticazione mancante, i token deboli
- **API4:2023 Unrestricted Resource Consumption** - Lo scanner di rate limit (`rate_limit_scanner.py`) rileva rate limit mancanti o aggirabili
- **API8:2023 Security Misconfiguration** - Più scanner intercettano configurazioni errate comuni

### MITRE ATT&CK

Tecniche rilevanti:

- **T1190** - Exploit Public-Facing Application - Tutte queste vulnerabilità si applicano alle API pubbliche
- **T1110** - Brute Force - Il rate limiting previene il credential stuffing e gli attacchi brute force
- **T1212** - Exploitation for Credential Access - La SQL injection può estrarre credenziali
- **T1078** - Valid Accounts - Il bypass dell'autenticazione fornisce all'attaccante un accesso valido

### CWE

Common weakness enumeration coperte:

- **CWE-89** - SQL Injection - Testata da `sqli_scanner.py` con tecniche error-based, boolean blind e time-based
- **CWE-287** - Improper Authentication - Lo scanner di autenticazione rileva autenticazione mancante o difettosa
- **CWE-639** - Authorization Bypass Through User-Controlled Key - Lo scanner IDOR trova problemi di autorizzazione a livello di oggetto
- **CWE-770** - Allocation of Resources Without Limits - Lo scanner di rate limit rileva il consumo illimitato di risorse

## Esempi dal mondo reale

### Caso di studio 1: lo scraping dei dati di Parler (2021)

Cosa è successo: dopo che Parler è stato rimosso dagli app store, dei ricercatori hanno scaricato l'intero contenuto della piattaforma - 70TB inclusi post eliminati, dati degli utenti e metadati con coordinate GPS.

Come ha funzionato l'attacco:
1. Parler usava ID sequenziali per i post
2. Nessun rate limiting sull'API
3. Nessuna autenticazione richiesta per visualizzare i post pubblici
4. I ricercatori hanno scritto uno script che incrementava gli ID dei post da 1 a ~13 milioni
5. Hanno scaricato tutto, compresi i post "eliminati" che in realtà non erano stati eliminati

Quali difese hanno fallito:
- IDOR: gli ID sequenziali hanno reso l'enumerazione banale
- Rate limiting: si potevano fare milioni di richieste senza throttling
- Conservazione dei dati: i post eliminati restavano accessibili

Come si sarebbe potuto prevenire:
- Usare UUID per gli ID dei post (rende l'enumerazione più difficile ma non impossibile)
- Richiedere l'autenticazione per l'accesso all'API
- Implementare un rate limiting aggressivo (100 post all'ora per IP)
- Eliminare davvero i dati quando gli utenti li eliminano
- Registrare e generare alert sui pattern di accesso insoliti

### Caso di studio 2: il breach dell'API di T-Mobile (2022)

Cosa è successo: degli attaccanti hanno sfruttato un endpoint API per accedere ai dati di 37 milioni di clienti, inclusi nomi, indirizzi e numeri di telefono.

Come ha funzionato l'attacco:
1. L'endpoint API accettava ID cliente come parametri
2. Nessun controllo di autorizzazione verificava che il richiedente possedesse quell'ID cliente
3. Gli attaccanti hanno enumerato gli ID per fare scraping dei record dei clienti
4. Il rate limiting era insufficiente per fermare l'attacco

Quali difese hanno fallito:
- IDOR: nessuna autorizzazione a livello di oggetto
- Rate limiting: presente ma impostato troppo alto
- Monitoraggio: l'attacco non è stato rilevato in tempo reale

Come si sarebbe potuto prevenire:
- Controllare l'autorizzazione: `if customer.id != current_user.id: return 403`
- Implementare un rate limiting adeguato (10 richieste/minuto per utente autenticato)
- Monitorare i pattern di accesso sequenziale agli ID
- Richiedere l'autenticazione a più fattori per l'accesso ai dati sensibili
- Usare la row-level security del database come difesa in profondità

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Perché usare UUID invece di ID sequenziali non risolve completamente le vulnerabilità IDOR?
2. Qual è la differenza tra rilevare gli header di rate limiting e rilevare un'applicazione attiva?
3. Come può funzionare la SQL injection time-based blind anche quando non puoi vedere errori del database o differenze nelle risposte?
4. Perché la validazione della firma JWT deve rifiutare esplicitamente l'algoritmo "none"?

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. L'implementazione avrà più senso una volta che questi fondamenti saranno chiari.

## Letture di approfondimento

**Essenziali:**
- OWASP API Security Top 10 2023 - Documento completo con esempi e indicazioni per la remediation
- PortSwigger Web Security Academy - Laboratori interattivi su SQL injection, autenticazione e autorizzazione
- JWT.io - Decodifica i token, comprendine la struttura, scopri le librerie per i diversi linguaggi

**Approfondimenti:**
- "The Web Application Hacker's Handbook" - Capitolo 9 (Attacking Data Stores) per le tecniche di SQL injection
- OWASP Testing Guide v4 - Testing for SQL Injection (metodologia dettagliata)
- Auth0 JWT Handbook - Guida completa alla sicurezza dei JWT

**Contesto storico:**
- "How I Hacked Facebook OAuth to Get Full Access" - Egor Homakov (2013)
- "Exploiting the Auth Token Length Oracle" - spiega gli attacchi di timing sull'autenticazione
- RFC 7519 originale del JWT - Comprendi lo standard e le sue considerazioni di sicurezza

