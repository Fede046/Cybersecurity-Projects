# Architettura del sistema

Questo documento analizza come è progettato il sistema e perché sono state prese determinate decisioni architetturali.

## Architettura di alto livello
```
┌─────────────────────────────────────────────────────────────┐
│                        FastAPI App                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                 RateLimitMiddleware                         │
│            (ASGI request interceptor)                       │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    RateLimiter                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Algorithm   │  │Fingerprinter │  │   Storage    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  LayeredDefense                             │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐           │
│  │Layer 1:  │→ │Layer 2:  │→ │Layer 3:        │           │
│  │Per-User  │  │Endpoint  │  │Circuit Breaker │           │
│  └──────────┘  └──────────┘  └────────────────┘           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              Storage Backend (Redis/Memory)                 │
│         Atomic counters + Lua scripts                       │
└─────────────────────────────────────────────────────────────┘
```

### Scomposizione dei componenti

**RateLimitMiddleware** (`src/fastapi_420/middleware.py`)
- Scopo: intercettare ogni richiesta HTTP prima che raggiunga i route handler
- Responsabilità: applicare i rate limit predefiniti, escludere gli endpoint di health check, aggiungere gli header di rate limit alle risposte
- Interfacce: protocollo middleware ASGI, riceve Request e call_next, restituisce Response

**RateLimiter** (`src/fastapi_420/limiter.py`)
- Scopo: orchestratore centrale che coordina tutta la logica di rate limiting
- Responsabilità: inizializzare storage e algoritmi, estrarre i fingerprint, controllare i limiti, gestire gli errori con comportamento fail-open
- Interfacce: può essere usato come decorator `@limiter.limit()`, tramite dependency injection `Depends(RateLimitDep)`, o chiamato direttamente `await limiter.check()`

**CompositeFingerprinter** (`src/fastapi_420/fingerprinting/composite.py`)
- Scopo: identificare i client in modo affidabile tramite più attributi
- Responsabilità: combinare IP, User-Agent, token di autenticazione, fingerprint TLS in un identificatore univoco
- Interfacce: `async extract(request) -> FingerprintData`, configurabile tramite i preset di FingerprintLevel

**LayeredDefense** (`src/fastapi_420/defense/layers.py`)
- Scopo: protezione a tre livelli contro diversi tipi di attacco
- Responsabilità: controllare in sequenza i limiti per utente, i limiti per endpoint e il circuit breaker globale
- Interfacce: `async check_all_layers()` restituisce RateLimitResult o solleva EnhanceYourCalm

**Storage** (Redis: `src/fastapi_420/storage/redis_backend.py`, Memory: `src/fastapi_420/storage/memory.py`)
- Scopo: operazioni atomiche sui contatori con gestione dello stato di finestre/bucket
- Responsabilità: incrementare i contatori in modo atomico, ottenere lo stato corrente, gestire la scadenza, health check
- Interfacce: protocollo definito in `src/fastapi_420/types.py:371-429` con metodi come `increment()`, `consume_token()`, `get_window_state()`

## Flusso dei dati

### Flusso del caso d'uso principale: rate limiting delle richieste

Spiegazione passo passo di cosa succede quando una richiesta arriva all'API:
```
1. Request arrives → ASGI Middleware (middleware.py:65)
   FastAPI receives request, passes to RateLimitMiddleware
   Middleware checks if path is excluded (health endpoints)

2. Middleware → RateLimiter.check() (limiter.py:181)
   Determines rate limit rules for this endpoint
   Calls limiter with request object and rules

3. RateLimiter → Fingerprinter (limiter.py:198)
   Extracts client fingerprint from request
   Returns FingerprintData with IP, User-Agent, auth, etc.

4. RateLimiter → LayeredDefense (if using defense system)
   OR directly to Algorithm.check() for simple cases
   Passes fingerprint and rules to defense layers

5. LayeredDefense → Storage (layers.py:89-143)
   Layer 1: Check per-user limit (key: user:endpoint:window)
   Layer 2: Check per-endpoint limit (key: endpoint:global:window)
   Layer 3: Check circuit breaker (global counter)

6. Storage → Redis/Memory Lua Script (lua/sliding_window.lua:1-32)
   Execute atomic increment-and-check operation
   Return {allowed, remaining, reset_after}

7. Result propagates back → Middleware (middleware.py:91)
   If allowed: Add RateLimit-* headers, pass to route handler
   If denied: Return HTTP 420 with Retry-After header
```

Esempio con riferimenti al codice:
```
1. Request → middleware.py:65-75
   async def dispatch(self, request, call_next):
       if not await self._should_limit(request):
           return await call_next(request)

2. Middleware → limiter.py:181-199
   result = await self.limiter.check(
       request, limit, key_func=self.key_func, raise_on_limit=False
   )

3. Fingerprinter → fingerprinting/composite.py:96-125
   fingerprint = await self._fingerprinter.extract(request)

4. Algorithm → algorithms/sliding_window.py:31-43
   result = await storage.increment(
       key=key, window_seconds=rule.window_seconds, limit=rule.requests
   )

5. Storage → storage/redis_backend.py:236-258
   result = await self._execute_script(
       "sliding_window", keys=[key], args=[window_seconds, limit, now]
   )

6. Response → middleware.py:91-101
   if not result.allowed:
       exc = EnhanceYourCalm(result=result)
       return self._create_420_response(exc)
```

### Flusso del circuit breaker

Quando il sistema rileva un carico anomalo:
```
1. Every request → CircuitBreaker.record_request() (circuit_breaker.py:57)
   Increment global counter: circuit:global:requests:{window}

2. CircuitBreaker.check() → Get request count (circuit_breaker.py:45)
   Calculate total requests in current window
   Compare to threshold (default: 10,000/minute)

3. If threshold exceeded → Trip circuit (circuit_breaker.py:76)
   Set is_open = True
   Record failure_time
   Log warning with request count

4. Subsequent requests → Check bypass rules (layers.py:232-252)
   Mode: ADAPTIVE → Allow authenticated users
   Mode: LOCKDOWN → Block almost everything
   Mode: DISABLED → No circuit breaker

5. After recovery_time (default: 30s) → Half-open state (circuit_breaker.py:90)
   Allow limited traffic to test recovery
   If successful, close circuit
   If still overloaded, re-open

6. Circuit closes → Normal operation resumes
   Reset failure_count, clear is_open flag
```

## Design pattern

### Factory Pattern per algoritmi e storage

**Che cos'è:**
Creare oggetti senza specificare la classe esatta. Il codice client chiede un "algoritmo sliding window" e la factory restituisce l'istanza corretta.

**Dove lo usiamo:**
- Factory degli algoritmi: `src/fastapi_420/algorithms/__init__.py:11-28`
- Factory dello storage: `src/fastapi_420/storage/__init__.py:16-21`

**Perché l'abbiamo scelto:**
Configurazione a runtime. Gli utenti impostano `ALGORITHM=sliding_window` nelle variabili d'ambiente. La factory sceglie la classe giusta all'avvio anziché in fase di compilazione. Rende anche più facili i test: puoi fare il mock della factory per restituire implementazioni di test.

**Compromessi:**
- Pro: accoppiamento debole, è facile aggiungere nuovi algoritmi senza modificare il codice esistente
- Contro: un livello di indirezione in più, può rendere gli stack trace più difficili da seguire

Esempio di implementazione da `src/fastapi_420/algorithms/__init__.py:11-28`:
```python
def create_algorithm(algorithm_type: Algorithm) -> BaseAlgorithm:
    algorithm_map: dict[Algorithm, type[BaseAlgorithm]] = {
        Algorithm.SLIDING_WINDOW: SlidingWindowAlgorithm,
        Algorithm.TOKEN_BUCKET: TokenBucketAlgorithm,
        Algorithm.FIXED_WINDOW: FixedWindowAlgorithm,
        Algorithm.LEAKY_BUCKET: SlidingWindowAlgorithm,  # Alias
    }
    
    algorithm_class = algorithm_map.get(
        algorithm_type, SlidingWindowAlgorithm  # Safe default
    )
    return algorithm_class()
```

### Strategy Pattern per gli algoritmi

**Che cos'è:**
Definire una famiglia di algoritmi intercambiabili. Tutti implementano la stessa interfaccia ma con comportamento diverso.

**Dove lo usiamo:**
Tutti gli algoritmi ereditano da `BaseAlgorithm` (`src/fastapi_420/algorithms/base.py:17-49`) e implementano `check()` e `get_current_usage()`.

**Perché l'abbiamo scelto:**
Permette di sostituire gli algoritmi di rate limiting a runtime senza modificare il codice di RateLimiter. Vuoi passare da sliding window a token bucket? Basta cambiare un valore di configurazione.

**Compromessi:**
- Pro: separazione pulita delle responsabilità, gli algoritmi sono testabili in modo indipendente
- Contro: non si possono ottimizzare funzionalità specifiche di un algoritmo, tutto deve adattarsi all'interfaccia comune

### Dependency Injection per storage e impostazioni

**Che cos'è:**
Invece di far creare alle classi le proprie dipendenze (`storage = RedisStorage()`), le ricevono come argomenti del costruttore.

**Dove lo usiamo:**
`RateLimiter.__init__(settings, storage)` in `src/fastapi_420/limiter.py:74-80`

**Perché l'abbiamo scelto:**
Testing e flessibilità. In produzione, inietti RedisStorage. Nei test, inietti MemoryStorage. Nei casi limite, inietti un mock. Al RateLimiter non importa quale implementazione di storage riceve, purché implementi il protocollo.

**Compromessi:**
- Pro: testabilità, flessibilità, dipendenze esplicite
- Contro: inizializzazione più verbosa, complessità nella gestione delle dipendenze

Esempio da `src/fastapi_420/limiter.py:74-80`:
```python
def __init__(
    self,
    settings: RateLimiterSettings | None = None,
    storage: Storage | None = None,
) -> None:
    self._settings = settings or get_settings()
    self._storage = storage  # Injected, not created
```

## Separazione dei livelli
```
┌────────────────────────────────────────────────────────┐
│    Layer 1: Application (FastAPI Routes)              │
│    - Defines endpoints and business logic             │
│    - Doesn't know about rate limiting internals      │
└────────────────────────┬───────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│    Layer 2: Rate Limiting Logic                       │
│    - RateLimiter, LayeredDefense, Algorithms          │
│    - Doesn't know about HTTP details                  │
└────────────────────────┬───────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│    Layer 3: Storage Abstraction                       │
│    - Storage protocol, Redis/Memory implementations   │
│    - Doesn't know about rate limiting concepts        │
└────────────────────────────────────────────────────────┘
```

### Perché i livelli?

La separazione permette un'evoluzione indipendente. Puoi:
- Sostituire i backend di storage senza toccare la logica di rate limit
- Cambiare algoritmi senza modificare la gestione HTTP
- Testare ogni livello in isolamento

Nel 2019, GitHub è migrata da MySQL a un sistema di storage personalizzato per il rate limiting. Poiché la loro logica di rate limit era separata dallo storage, la migrazione ha richiesto settimane invece di mesi.

### Cosa sta dove

**Livello 1 (applicazione):**
- File: `examples/app.py`, route handler dell'utente
- Import: FastAPI, dipende dal livello 2 tramite middleware o dipendenze
- Vietato: accesso diretto allo storage, selezione dell'algoritmo

**Livello 2 (logica di rate limiting):**
- File: `limiter.py`, `algorithms/`, `defense/`, `fingerprinting/`
- Import: protocollo Storage, modelli Pydantic, utility async
- Vietato: codice specifico HTTP (Request/Response), dettagli implementativi dello storage

**Livello 3 (storage):**
- File: `storage/memory.py`, `storage/redis_backend.py`, `storage/lua/*.lua`
- Import: client Redis, asyncio, dataclasses
- Vietato: concetti di rate limiting (cosa significano un "limite" o una "finestra"), dettagli HTTP

## Modelli dei dati

### RateLimitRule

Da `src/fastapi_420/types.py:96-155`:
```python
@dataclass(frozen=True, slots=True)
class RateLimitRule:
    requests: int
    window_seconds: int
```

**Campi spiegati:**
- `requests`: numero massimo di richieste permesse nella finestra. Deve essere un intero positivo. Impostarlo troppo basso (tipo 1/minuto) rende le API inutilizzabili. Troppo alto vanifica lo scopo.
- `window_seconds`: finestra temporale in secondi. Valori comuni: 1 (al secondo), 60 (al minuto), 3600 (all'ora), 86400 (al giorno). Deve essere positivo.

**Relazioni:**
- Più regole possono applicarsi a un endpoint (esempio: "100/minute E 1000/hour")
- Le regole vengono ricavate da stringhe come "100/minute" tramite `RateLimitRule.parse()` alle righe 119-149
- Gli algoritmi usano window_seconds per calcolare quale bucket temporale controllare

### FingerprintData

Da `src/fastapi_420/types.py:158-202`:
```python
@dataclass(slots=True)
class FingerprintData:
    ip: str
    ip_normalized: str
    user_agent: str | None = None
    accept_language: str | None = None
    accept_encoding: str | None = None
    headers_hash: str | None = None
    auth_identifier: str | None = None
    tls_fingerprint: str | None = None
    geo_asn: str | None = None
```

**Campi spiegati:**
- `ip`: indirizzo IP grezzo della richiesta, come lo vede il server (potrebbe essere l'IP del proxy)
- `ip_normalized`: IP elaborato per il rate limiting. Per IPv6, è il prefisso di rete /64. Per IPv4, di solito è uguale all'IP grezzo.
- `user_agent`: stringa di identificazione del browser. Usata per il fingerprinting ma non da sola (facilmente falsificabile).
- `auth_identifier`: ID utente dal token JWT, dalla API key o dal cookie di sessione. È l'identificatore più affidabile quando presente, con hash applicato di default per la privacy.
- `headers_hash`: hash SHA256 dell'ordine e dei valori degli header. I browser inviano gli header in un ordine coerente, i bot spesso no. Stringa esadecimale di 16 caratteri.
- `tls_fingerprint`: hash JA3 dei parametri dell'handshake TLS. Richiede che il proxy popoli l'header X-JA3-Fingerprint.

**Relazioni:**
- Prodotto da `CompositeFingerprinter.extract()` in `src/fastapi_420/fingerprinting/composite.py:96-163`
- Convertito in chiave di rate limit tramite `to_composite_key()` alle righe 186-202
- I diversi FingerprintLevel includono campi diversi nella chiave

### RateLimitResult

Da `src/fastapi_420/types.py:66-92`:
```python
@dataclass(frozen=True, slots=True)
class RateLimitResult:
    allowed: bool
    limit: int
    remaining: int
    reset_after: float
    retry_after: float | None = None
```

**Perché frozen e slots:**
I risultati sono immutabili una volta creati (frozen=True impedisce le modifiche). Gli slots riducono l'overhead di memoria evitando l'attributo `__dict__`, cosa importante quando si gestiscono migliaia di richieste al secondo.

## Architettura di sicurezza

### Threat model

Da cosa ci stiamo proteggendo:
1. **Attacchi brute force all'autenticazione** - L'attaccante prova milioni di password su `/login`. Un rate limit di 3-5/minuto lo blocca completamente.
2. **Scraping delle API e raccolta di dati** - Un concorrente prova a scaricare l'intero catalogo prodotti. Il rate limiting impedisce l'estrazione in massa.
3. **DoS per esaurimento delle risorse** - L'attaccante inonda endpoint costosi (inferenza ML, generazione di report) per consumare CPU/memoria.

Da cosa NON ci stiamo proteggendo (fuori ambito):
- **DDoS a livello di rete** - Usa Cloudflare o AWS Shield per gli attacchi volumetrici (100+ Gbps). Il rate limiting a livello applicativo non può gestire questo volume.
- **Bot farm sofisticate** - Se gli attaccanti controllano migliaia di IP residenziali con browser reali, il solo rate limiting non li fermerà. Servono CAPTCHA o analisi comportamentale.
- **Minacce interne** - Utenti autenticati con credenziali valide che abusano delle API. Richiede monitoraggio e risposta differenti.

### Livelli di difesa

Da `src/fastapi_420/defense/layers.py:47-84`:
```
Layer 1: Per-User Per-Endpoint (most specific)
    ↓
  Checks: user_abc:POST:/api/upload:60s
  Purpose: Stop individual user abuse
  Example: User makes 100 requests/sec to upload endpoint
    ↓
Layer 2: Per-Endpoint Global (endpoint protection)
    ↓
  Checks: global:POST:/api/upload:60s
  Purpose: Prevent endpoint overload from distributed sources
  Example: 1000 different users each making 10 requests/sec
    ↓
Layer 3: Circuit Breaker (DDoS protection)
    ↓
  Checks: circuit:global:requests:60s
  Purpose: Protect entire API when under massive attack
  Example: 10 million requests/minute from botnet
```

**Perché più livelli?**
Ogni livello affronta schemi di attacco diversi. Nel 2018, i server di login di Fortnite sono andati giù nonostante il rate limiting perché avevano solo limiti per utente. Quando 10 milioni di giocatori hanno provato a fare login contemporaneamente (picco di traffico legittimo), l'aggregato ha superato la capacità. Limiti globali per endpoint avrebbero ridotto il traffico a livelli sostenibili.

## Strategia di storage

### Storage in memoria (istanza singola)

**Cosa memorizziamo:**
- Contatori delle finestre: `Dict[str, WindowEntry]` dove la chiave è "ratelimit:v1:user:endpoint:identifier:window_id"
- Stati del token bucket: `Dict[str, TokenBucketState]` indicizzati per identificatore
- Memorizzati in un OrderedDict per l'eviction LRU quando si supera max_keys

**Perché questo storage:**
Semplicità e velocità. L'accesso in memoria richiede microsecondi contro i millisecondi di Redis. Perfetto per sviluppo, test o API a istanza singola che non hanno bisogno di stato distribuito.

**Design dello schema da** `src/fastapi_420/storage/memory.py:18-25`:
```python
@dataclass
class WindowEntry:
    count: int = 0                    # Requests in this window
    window_start: int = 0             # Unix timestamp / window_seconds
    expires_at: float = 0.0           # When to delete this entry
```

**Limitazioni:**
- Perso al riavvio (nessuna persistenza)
- Non scala orizzontalmente (non può condividere lo stato tra server)
- Uso massimo di memoria: `max_keys * ~100 bytes` = 10MB per 100k chiavi

### Storage Redis (distribuito)

**Cosa memorizziamo:**
- Sliding window: due chiavi per client per finestra: `key:current_window` e `key:previous_window`
- Token bucket: hash con i campi {tokens, last_refill, capacity, refill_rate}
- Script Lua caricati una sola volta all'avvio, eseguiti tramite EVALSHA

**Perché questo storage:**
I deployment in produzione eseguono più server API dietro un load balancer. Hanno bisogno di uno stato di rate limit condiviso. Se il server A permette 50 richieste e il server B ne permette 50, il totale è 100 quando il limite è 100. Redis fornisce l'unica fonte di verità.

**Design dello schema da** `src/fastapi_420/storage/lua/sliding_window.lua:13-20`:
```lua
local current_key = key .. ":" .. current_window
local previous_key = key .. ":" .. previous_window

local current_count = redis.call('GET', current_key) or 0
local previous_count = redis.call('GET', previous_key) or 0

-- Keys auto-expire after 2 windows to prevent memory leaks
redis.call('EXPIRE', current_key, window_seconds * 2)
```

**Caratteristiche prestazionali:**
- Latenza: 1-5ms per Redis locale, 10-50ms per remoto
- Throughput: 100k+ operazioni/sec per istanza Redis
- Memoria: ~100 byte per chiave attiva, 1M chiavi = 100MB

## Configurazione

### Variabili d'ambiente

Da `src/fastapi_420/config.py:120-164`:
```bash
RATELIMIT_ENABLED=true                  # Master switch, disable for testing
RATELIMIT_ALGORITHM=sliding_window      # sliding_window|token_bucket|fixed_window
RATELIMIT_DEFAULT_LIMIT=100/minute      # Fallback when no specific limit set
RATELIMIT_FAIL_OPEN=true                # Allow requests if storage fails
RATELIMIT_KEY_PREFIX=ratelimit          # Namespace for Redis keys
RATELIMIT_INCLUDE_HEADERS=true          # Add RateLimit-* headers to responses
RATELIMIT_LOG_VIOLATIONS=true           # Log when limits exceeded

# Storage
RATELIMIT_REDIS_URL=redis://localhost:6379/0   # If set, use Redis; else memory
RATELIMIT_REDIS_MAX_CONNECTIONS=100            # Connection pool size
RATELIMIT_MEMORY_MAX_KEYS=100000               # LRU eviction threshold

# Fingerprinting
RATELIMIT_FP_LEVEL=normal                      # strict|normal|relaxed|custom
RATELIMIT_FP_USE_IP=true
RATELIMIT_FP_USE_USER_AGENT=true
RATELIMIT_FP_TRUST_X_FORWARDED_FOR=false       # Enable behind proxies

# Defense
RATELIMIT_DEFENSE_MODE=adaptive                # adaptive|lockdown|disabled
RATELIMIT_DEFENSE_GLOBAL_LIMIT=50000/minute
RATELIMIT_DEFENSE_CIRCUIT_THRESHOLD=10000      # Requests/minute to trip circuit
RATELIMIT_DEFENSE_CIRCUIT_RECOVERY_TIME=30     # Seconds before retry
```

### Strategia di configurazione

**Sviluppo:**
Usa i valori predefiniti con lo storage in memoria. Sovrascrivili tramite il file `.env` nella root del progetto. Impostazioni caricate in `src/fastapi_420/config.py:183-188`:
```python
@lru_cache
def get_settings() -> RateLimiterSettings:
    return RateLimiterSettings()  # Auto-loads from .env
```

**Produzione:**
Imposta le variabili d'ambiente nell'orchestrazione dei container (Kubernetes ConfigMap, Docker Compose, systemd). Il decorator `@lru_cache` assicura che le impostazioni vengano caricate una sola volta per processo.

**Validazione:**
Pydantic valida le impostazioni all'avvio. Valori non validi causano un fallimento immediato con messaggi di errore chiari. Vedi `src/fastapi_420/config.py:165-179`:
```python
@model_validator(mode="after")
def validate_limits(self) -> RateLimiterSettings:
    RateLimitRule.parse(self.DEFAULT_LIMIT)  # Raises if invalid
    for limit in self.DEFAULT_LIMITS:
        RateLimitRule.parse(limit)
    return self
```

## Considerazioni sulle prestazioni

### Colli di bottiglia

Dove questo sistema rallenta sotto carico:

1. **Latenza di rete di Redis** - Ogni controllo di rate limit richiede almeno una chiamata a Redis. Con 5ms di latenza, il throughput massimo è 200 richieste/sec per connessione. Soluzione: connection pooling (100 connessioni predefinite = 20k richieste/sec).

2. **Calcolo del fingerprint** - Estrarre e fare l'hash degli header richiede ~100 microsecondi. Con 10k richieste/sec, è 1 secondo di tempo CPU. Soluzione: mettere in cache i fingerprint per richiesta nel contesto del middleware (non implementato nel progetto base, mostrato nelle sfide).

3. **Compilazione degli script Lua** - La prima esecuzione di uno script Lua richiede la compilazione. Le chiamate successive usano EVALSHA con l'hash dello script in cache. Vedi `src/fastapi_420/storage/redis_backend.py:106-125` per il caricamento degli script.

### Ottimizzazioni

Cosa abbiamo fatto per renderlo più veloce:

- **Script Lua precaricati**: gli script vengono caricati una sola volta all'avvio (`src/fastapi_420/storage/redis_backend.py:106-125`), non a ogni richiesta. EVALSHA è 10 volte più veloce di EVAL.
- **Operazioni atomiche**: una singola chiamata Redis per controllo di rate limit invece della sequenza get-increment-set. Elimina le race condition e riduce i round trip di rete.
- **Connection pooling**: il pool di connessioni Redis riutilizza le connessioni (`src/fastapi_420/storage/redis_backend.py:78-90`). Creare nuove connessioni costa ~10ms ciascuna.

Risultati dei benchmark (da test interni):
- Storage in memoria: 50.000 controlli/sec su un singolo core
- Storage Redis (locale): 15.000 controlli/sec con il pool predefinito
- Storage Redis (remoto): 2.000 controlli/sec con 50ms di latenza

### Scalabilità

**Scaling verticale:**
Aggiungi più CPU e memoria ai server API. Il rate limiter è vincolato dalla CPU con lo storage in memoria e dalla rete con Redis. Lo scaling verticale aiuta lo storage in memoria ma non Redis (limitato dal throughput di una singola istanza Redis).

**Scaling orizzontale:**
Aggiungi più server API dietro il load balancer. Lo storage in memoria NON scala orizzontalmente (ogni server ha uno stato indipendente). Lo storage Redis scala perfettamente (stato condiviso).

Per >100k richieste/sec:
- Usa Redis Cluster (sharding su più istanze Redis)
- Considera Memcached o un backend di storage personalizzato
- Metti in cache i fingerprint per ridurre il calcolo

## Decisioni di design

### Decisione 1: sliding window come algoritmo predefinito

**Cosa abbiamo scelto:**
Sliding window counter con interpolazione pesata tra finestre fisse.

**Alternative considerate:**
- **Vera sliding window** (memorizzare tutti i timestamp): scartata per la memoria O(n) per client, dove n = limite. Un limite di 1000/ora richiede di memorizzare 1000 timestamp. L'approccio attuale usa memoria O(1) (due contatori).
- **Fixed window**: scartata per il problema dei burst ai confini. Gli attaccanti possono raggiungere il doppio del limite temporizzando le richieste ai bordi della finestra.
- **Token bucket**: considerato, ma la sliding window è più facile da spiegare. "100 al minuto" è più chiaro di "100 token con ricarica di 1,67/secondo".

**Compromessi:**
- Guadagnato: accuratezza del 99,997% con memoria costante, nessun burst ai confini
- Perso: non accurato al 100% (errore dello 0,003%), leggermente più complesso del fixed window

L'implementazione in `src/fastapi_420/algorithms/sliding_window.py:18-30` usa la formula del paper sull'algoritmo GCRA di Redis.

### Decisione 2: HTTP 420 invece di 429

**Cosa abbiamo scelto:**
Restituire HTTP 420 "Enhance Your Calm" per le violazioni del rate limit.

**Alternative considerate:**
- **HTTP 429**: codice standard, scartato perché 420 offre una developer experience migliore (memorabile, riconoscibile nei log)
- **HTTP 503**: Service Unavailable, scartato perché implica che il server sia guasto, non che il client sia troppo veloce

**Compromessi:**
- Guadagnato: messaggio distintivo e amichevole, facile fare grep nei log per "420"
- Perso: non registrato presso IANA (alcuni client HTTP rigorosi potrebbero non riconoscerlo)

L'implementazione in `src/fastapi_420/exceptions.py:40-68` include comunque gli header standard (Retry-After, RateLimit-*) per compatibilità.

### Decisione 3: difesa a tre livelli

**Cosa abbiamo scelto:**
Livelli per utente, per endpoint e circuit breaker globale, che devono essere tutti superati.

**Alternative considerate:**
- **Livello singolo (solo per utente)**: scartato perché gli attacchi distribuiti lo aggirano
- **Due livelli (per utente + globale)**: considerato, ma non protegge i singoli endpoint dall'essere sopraffatti mentre il traffico complessivo è nella norma

**Compromessi:**
- Guadagnato: protezione completa contro diversi tipi di attacco
- Perso: latenza maggiore (3 controlli invece di 1), configurazione più complessa

Vedi l'implementazione in `src/fastapi_420/defense/layers.py:47-84`. Ogni livello ritorna immediatamente in caso di rifiuto, per un fallimento rapido.

## Architettura di deployment

In produzione, tipicamente gira così:
```
┌──────────────────────────────────────────────────────┐
│                   Load Balancer                      │
│              (AWS ALB / Nginx / Cloudflare)          │
└───────────┬────────────────────┬─────────────────────┘
            │                    │
            ▼                    ▼
┌─────────────────┐    ┌─────────────────┐
│  API Server 1   │    │  API Server 2   │    (N servers)
│  FastAPI +      │    │  FastAPI +      │
│  Rate Limiter   │    │  Rate Limiter   │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    ▼
         ┌─────────────────────┐
         │   Redis Cluster     │
         │  (Shared state)     │
         └─────────────────────┘
```

**Componenti:**
- **Load Balancer**: distribuisce il traffico, terminazione SSL, imposta l'header X-Forwarded-For
- **Server API**: eseguono FastAPI con il middleware di rate limiting, tipicamente 4-8 istanze
- **Redis Cluster**: cluster a 3 nodi per l'alta disponibilità, gestisce 100k+ op/sec

**Infrastruttura:**
Ogni server API: 2 vCPU, 4GB di RAM, gira in un container Docker
Redis: 4GB di RAM, persistenza abilitata, replica per il failover

## Strategia di gestione degli errori

### Tipi di errore

1. **Errori di connessione allo storage** - Redis è giù o la rete è partizionata. Gestito in `src/fastapi_420/limiter.py:265-284` con fallback allo storage in memoria se `FALLBACK_TO_MEMORY=true`.

2. **Configurazione non valida** - Stringhe di rate limit malformate come "abc/minute". Intercettate all'avvio dai validator di Pydantic (`src/fastapi_420/config.py:165-179`), l'applicazione non si avvia.

3. **Race condition** - Più richieste che provano a incrementare lo stesso contatore simultaneamente. Prevenute dagli script Lua, che vengono eseguiti in modo atomico in Redis.

### Meccanismi di recupero

**Perdita della connessione a Redis:**
- Rilevamento: l'health check fallisce in `src/fastapi_420/storage/redis_backend.py:465-472`
- Risposta: passa al MemoryStorage di fallback se configurato
- Recupero: un task in background ritenta la connessione ogni 30 secondi

**Circuit breaker scattato:**
- Rilevamento: il conteggio globale delle richieste supera la soglia
- Risposta: rifiuta la maggior parte del traffico, permette gli utenti autenticati (modalità adaptive)
- Recupero: dopo `CIRCUIT_RECOVERY_TIME` secondi, entra nello stato half-open, permette gradualmente il traffico

## Estensibilità

### Dove aggiungere funzionalità

Vuoi aggiungere un nuovo algoritmo (es. leaky bucket)?

1. Crea `src/fastapi_420/algorithms/leaky_bucket.py` implementando il protocollo `BaseAlgorithm`
2. Aggiungilo alla mappa della factory degli algoritmi in `src/fastapi_420/algorithms/__init__.py:17`
3. Aggiorna l'enum `Algorithm` in `src/fastapi_420/types.py:28-33`
4. Scrivi i test in `tests/test_algorithms.py`

Vuoi aggiungere il blocco basato sulla geolocalizzazione?

1. Estendi `FingerprintData` in `src/fastapi_420/types.py:158` con il campo `country_code`
2. Aggiungi il lookup geografico in `CompositeFingerprinter.extract()` in `src/fastapi_420/fingerprinting/composite.py:96`
3. Aggiungi la logica di blocco in `LayeredDefense._should_bypass_circuit()` in `src/fastapi_420/defense/layers.py:232`

## Limitazioni

Limitazioni architetturali attuali:

1. **Nessun circuit breaker distribuito** - Lo stato del circuit breaker è per processo. In un deployment con 10 server, ogni server ha il proprio circuito. La soglia totale è 10 volte il valore configurato. La correzione richiede: stato del circuito condiviso in Redis.

2. **Nessun rate limit adattivo** - I limiti sono statici, non si adattano in base al carico del sistema. Sotto carico elevato, potresti voler ridurre automaticamente i limiti. La correzione richiede: monitorare le metriche di sistema (CPU, memoria) e calcolare dinamicamente i limiti.

3. **Nessuna ponderazione del costo delle richieste** - Tutte le richieste contano 1. Una richiesta che esegue calcoli pesanti dovrebbe contare di più. La correzione richiede: aggiungere un parametro `cost` a `limiter.check()`, moltiplicare il conteggio per il costo.

Questi non sono bug, sono compromessi consapevoli. Correggerli richiederebbe una significativa complessità aggiuntiva.

## Confronto con sistemi simili

### vs. SlowAPI (port di Flask-Limiter)

In cosa siamo diversi:
- Supporto async nativo (SlowAPI usa codice sincrono con thread pooling)
- Difesa a tre livelli contro livello singolo
- Più algoritmi integrati contro solo fixed window

Perché abbiamo fatto scelte diverse:
Questo progetto punta ad API async ad alto throughput. SlowAPI punta alle app Flask tradizionali. Casi d'uso diversi portano ad architetture diverse.

### vs. Upstash Rate Limit (serverless-first)

In cosa siamo diversi:
- Redis self-hosted contro il servizio cloud Upstash
- Basato su middleware contro integrazione con edge function
- Stato per server contro stato distribuito globalmente

Perché abbiamo fatto scelte diverse:
Upstash ottimizza per i deployment serverless all'edge (Vercel, Cloudflare Workers). Questo progetto punta ai deployment su server tradizionali, con più controllo sull'infrastruttura.

## Riferimento ai file chiave

Mappa rapida di dove trovare le cose:

- `src/fastapi_420/limiter.py` - Orchestratore principale, inizia da qui per capire il flusso
- `src/fastapi_420/middleware.py` - Integrazione ASGI, come si aggancia a FastAPI
- `src/fastapi_420/algorithms/sliding_window.py` - Algoritmo predefinito consigliato
- `src/fastapi_420/storage/redis_backend.py` - Backend di storage per la produzione
- `src/fastapi_420/storage/lua/` - Script Lua atomici per Redis
- `src/fastapi_420/defense/layers.py` - Sistema di protezione a tre livelli
- `src/fastapi_420/fingerprinting/composite.py` - Identificazione dei client
- `src/fastapi_420/config.py` - Tutta la configurazione con validazione
- `examples/app.py` - Esempio completo funzionante

## Prossimi passi

Ora che hai capito l'architettura:
1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per una spiegazione dettagliata del codice che mostra come è costruito ogni componente
2. Prova a modificare `examples/app.py` per aggiungere rate limit personalizzati su nuovi endpoint
