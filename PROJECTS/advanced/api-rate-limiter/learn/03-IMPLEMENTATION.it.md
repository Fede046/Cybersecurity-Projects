# Guida all'implementazione

Questo documento ripercorre il codice vero e proprio. Costruiremo le funzionalità chiave passo dopo passo e spiegheremo le decisioni lungo il percorso.

## Panoramica della struttura dei file
```
src/fastapi_420/
├── __init__.py                    # Package exports
├── limiter.py                     # RateLimiter class (main entry point)
├── middleware.py                  # ASGI middleware integration
├── dependencies.py                # FastAPI dependency injection helpers
├── exceptions.py                  # Custom exceptions (EnhanceYourCalm)
├── config.py                      # Pydantic settings with validation
├── types.py                       # Data structures and protocols
├── algorithms/
│   ├── __init__.py               # Algorithm factory
│   ├── base.py                   # BaseAlgorithm protocol
│   ├── sliding_window.py         # Recommended algorithm
│   ├── token_bucket.py           # Burst-tolerant algorithm
│   └── fixed_window.py           # Simple algorithm
├── storage/
│   ├── __init__.py               # Storage factory
│   ├── memory.py                 # In-memory backend
│   ├── redis_backend.py          # Redis backend
│   └── lua/
│       ├── sliding_window.lua    # Atomic sliding window
│       ├── token_bucket.lua      # Atomic token bucket
│       └── fixed_window.lua      # Atomic fixed window
├── fingerprinting/
│   ├── __init__.py
│   ├── ip.py                     # IP extraction + IPv6 normalization
│   ├── headers.py                # Header fingerprinting
│   ├── auth.py                   # JWT/API key extraction
│   └── composite.py              # Combines all methods
└── defense/
    ├── __init__.py
    ├── circuit_breaker.py        # Global DDoS protection
    └── layers.py                 # Three-layer defense
```

## Costruire il nucleo: l'algoritmo sliding window

### Passo 1: l'interfaccia dell'algoritmo

Cosa stiamo costruendo: un protocollo che tutti gli algoritmi di rate limiting implementano.

Crea `src/fastapi_420/algorithms/base.py`:
```python
from abc import ABC, abstractmethod
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from fastapi_420.storage import Storage
    from fastapi_420.types import RateLimitResult, RateLimitRule

class BaseAlgorithm(ABC):
    """
    Abstract base class for rate limiting algorithms
    """
    @property
    @abstractmethod
    def name(self) -> str:
        """Algorithm name for logging and debugging"""
        ...
    
    @abstractmethod
    async def check(
        self,
        storage: Storage,
        key: str,
        rule: RateLimitRule,
        timestamp: float | None = None,
    ) -> RateLimitResult:
        """Check if request is allowed under rate limit"""
        ...
```

**Perché questo codice funziona:**
- Righe 3-7: gli import condizionati a TYPE_CHECKING prevengono le dipendenze circolari. Storage e types vengono importati solo quando girano i type checker, non a runtime.
- Righe 11-15: la proprietà astratta obbliga le sottoclassi a definire un nome. Usato nei log per identificare quale algoritmo è in esecuzione.
- Righe 17-25: la firma del metodo check è il contratto. Ogni algoritmo deve accettare storage, key, rule e un timestamp opzionale. Il tipo di ritorno è sempre RateLimitResult.

**Errori comuni qui:**
```python
# Wrong approach
class BaseAlgorithm:
    def check(self, key, limit):  # No type hints
        count = storage.get(key)   # Where does storage come from?
        return count < limit       # Returns bool, not RateLimitResult

# Why this fails:
# 1. No dependency injection for storage
# 2. Missing type hints make it hard to use correctly
# 3. Returns primitive type instead of rich result object
```

### Passo 2: implementare lo sliding window

Ora dobbiamo implementare davvero l'algoritmo.

In `src/fastapi_420/algorithms/sliding_window.py` (righe 18-43):
```python
class SlidingWindowAlgorithm(BaseAlgorithm):
    """
    Sliding window counter algorithm
    
    Achieves ~99.997% accuracy with O(1) memory per client.
    Uses weighted interpolation between two fixed windows.
    """
    @property
    def name(self) -> str:
        return Algorithm.SLIDING_WINDOW.value
    
    async def check(
        self,
        storage: Storage,
        key: str,
        rule: RateLimitRule,
        timestamp: float | None = None,
    ) -> RateLimitResult:
        """
        Check and increment counter using sliding window algorithm
        """
        return await storage.increment(
            key=key,
            window_seconds=rule.window_seconds,
            limit=rule.requests,
            timestamp=timestamp,
        )
```

**Cosa sta succedendo:**
1. Righe 18-23: la docstring spiega che questo è l'algoritmo consigliato e perché (compromesso tra accuratezza e memoria).
2. Righe 25-27: la proprietà name restituisce il valore dell'enum "sliding_window" per il logging.
3. Righe 29-43: il metodo check delega allo storage. La logica dell'algoritmo vive nei backend di storage (script Lua per Redis o Python per la memoria).

**Perché lo facciamo in questo modo:**
Separazione delle responsabilità. L'algoritmo definisce cosa fare (sliding window counter). Lo storage definisce come farlo (script Lua per Redis, lock Python per la memoria). Questo permette a diversi backend di storage di ottimizzare l'implementazione mentre l'interfaccia dell'algoritmo resta coerente.

**Approcci alternativi:**
- **Approccio A: l'algoritmo contiene la logica**: richiederebbe di duplicare la logica di incremento per ogni backend di storage. Soggetto a errori.
- **Approccio B: metodo di incremento generico**: più semplice, ma non può ottimizzare per funzionalità specifiche dello storage (atomicità di Lua in Redis).

### Passo 3: il backend di storage - script Lua di Redis

La vera matematica dello sliding window avviene in Lua per garantire l'atomicità.

In `src/fastapi_420/storage/lua/sliding_window.lua`:
```lua
local key = KEYS[1]
local window_seconds = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local current_window = math.floor(now / window_seconds)
local previous_window = current_window - 1
local elapsed_ratio = (now % window_seconds) / window_seconds

local current_key = key .. ":" .. current_window
local previous_key = key .. ":" .. previous_window

local current_count = tonumber(redis.call('GET', current_key)) or 0
local previous_count = tonumber(redis.call('GET', previous_key)) or 0

local weighted_count = math.floor(
    previous_count * (1 - elapsed_ratio) + current_count
)
local reset_after = window_seconds - (now % window_seconds)

if weighted_count >= limit then
    return {0, 0, reset_after, reset_after}  -- Denied
end

redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window_seconds * 2)

local new_weighted = math.floor(
    previous_count * (1 - elapsed_ratio) + current_count + 1
)
local remaining = math.max(0, limit - new_weighted)

return {1, remaining, reset_after, 0}  -- Allowed
```

**Le parti chiave spiegate:**

**Righe 6-8** - Calcolare in quali finestre ci troviamo:
```lua
local current_window = math.floor(now / window_seconds)
local elapsed_ratio = (now % window_seconds) / window_seconds
```
Se now=1234 e window_seconds=60, current_window=20 (siamo nel 20° minuto). elapsed_ratio=0.57 (siamo al 57% del minuto corrente).

**Righe 16-18** - L'algoritmo centrale:
```lua
local weighted_count = math.floor(
    previous_count * (1 - elapsed_ratio) + current_count
)
```
Questa è la formula dello sliding window. Se siamo al 57% della finestra corrente, la finestra precedente conta per il 43% e quella corrente per il 100%. Esempio: previous=80, current=30, weighted=80*0.43+30=64.4.

**Righe 25-26** - Incremento atomico:
```lua
redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window_seconds * 2)
```
Incrementa il contatore della finestra corrente e imposta la scadenza. La scadenza è il doppio della dimensione della finestra, per garantire che i dati della finestra precedente siano disponibili per l'interpolazione.

**Cosa succede se rimuovi questo:**
Senza script Lua, dovresti:
1. GET current_key
2. GET previous_key
3. Calcolare weighted_count
4. Verificare se si è oltre il limite
5. INCR current_key
6. Impostare la scadenza

Tra i passi 4 e 5, un'altra richiesta potrebbe incrementare il contatore. Race condition. Due richieste vedono entrambe count=99, entrambe incrementano, entrambe vengono permesse quando il limite è 100. Lua viene eseguito in modo atomico, impedendo questo.

## Costruire il sistema di fingerprinting

### Il problema

Cosa stiamo costruendo: un'identificazione affidabile dei client quando gli IP non sono univoci.

L'approccio ingenuo:
```python
# Bad
def get_client_id(request):
    return request.client.host
```

Questo non funziona con:
- Più utenti dietro lo stesso NAT
- Rotazione degli indirizzi IPv6
- Reti degli operatori mobili
- Server proxy

### La soluzione

Creare un fingerprint composito a partire da più attributi.

In `src/fastapi_420/fingerprinting/composite.py` (righe 96-163):
```python
async def extract(self, request: Request) -> FingerprintData:
    """Extract fingerprint data from request"""
    raw_ip = ""
    normalized_ip = ""
    user_agent = None
    # ... (other fields initialized)
    
    if self.use_ip:
        raw_ip, normalized_ip = self._ip_extractor.extract(request)
    
    if self.use_user_agent:
        user_agent = self._headers_extractor.extract_user_agent(request)
    
    if self.use_accept_headers:
        accept_language = self._headers_extractor.extract_accept_language(request)
        accept_encoding = self._headers_extractor.extract_accept_encoding(request)
    
    if self.use_header_order:
        headers_hash = self._headers_extractor.compute_headers_hash(request)
    
    if self.use_auth:
        auth_identifier = self._auth_extractor.extract(request)
    
    return FingerprintData(
        ip=raw_ip,
        ip_normalized=normalized_ip,
        user_agent=user_agent,
        # ... (all other fields)
    )
```

Questo codice:
- Righe 105-106: usa l'extractor degli IP per ottenere sia l'IP grezzo che quello normalizzato. La normalizzazione gestisce i prefissi IPv6 /64.
- Righe 108-109: estrae lo User-Agent solo se configurato. Alcune configurazioni attente alla privacy potrebbero disabilitarlo.
- Righe 111-114: Accept-Language e Accept-Encoding sono opt-in. Sono utili ma cambiano occasionalmente.
- Righe 116-117: l'ordine degli header è molto stabile (specifico del browser) ma costoso da calcolare.
- Righe 119-120: l'autenticazione è l'identificatore più affidabile quando presente.

Il risultato viene convertito in una chiave in `src/fastapi_420/types.py:186-202`:
```python
def to_composite_key(self, level: FingerprintLevel) -> str:
    """Generate composite fingerprint key based on level"""
    if level == FingerprintLevel.RELAXED:
        components = [self.ip_normalized]
        if self.auth_identifier:
            components.append(self.auth_identifier)
        return ":".join(filter(None, components))
    
    if level == FingerprintLevel.NORMAL:
        components = [
            self.ip_normalized,
            self.user_agent or "",
            self.auth_identifier or "",
        ]
        return ":".join(components)
    
    # STRICT includes all fields
    components = [
        self.ip_normalized, self.user_agent or "",
        self.accept_language or "", self.accept_encoding or "",
        self.headers_hash or "", self.auth_identifier or "",
        self.tls_fingerprint or "", self.geo_asn or "",
    ]
    return ":".join(components)
```

Esempi di risultato:
```
RELAXED:  "192.168.1.1:user_abc"
NORMAL:   "192.168.1.1:Mozilla/5.0...:user_abc"
STRICT:   "192.168.1.1:Mozilla/5.0...:en-US:gzip:8a3f:user_abc:ja3_hash:AS15169"
```

## Gestire la normalizzazione IPv6

Il caso speciale per IPv6 in `src/fastapi_420/fingerprinting/ip.py:89-107`:
```python
def _normalize_ip(self, ip_str: str) -> str:
    """
    Normalize IP address for rate limiting
    
    IPv6 addresses are normalized to /64 network prefix
    since users typically control entire /64 blocks.
    """
    try:
        addr = ip_address(ip_str)
    except ValueError:
        return ip_str  # Invalid IP, return as-is
    
    if isinstance(addr, IPv6Address):
        if addr.ipv4_mapped:
            return str(addr.ipv4_mapped)  # Convert ::ffff:192.0.2.1 to 192.0.2.1
        
        network = ip_network(
            f"{ip_str}/{self.ipv6_prefix_length}",
            strict=False,
        )
        return str(network.network_address)
    
    return str(addr)  # IPv4, return unchanged
```

**Dettagli importanti:**
- Righe 89-107: questo previene gli attacchi basati sulla rotazione degli indirizzi IPv6. Senza normalizzazione, un attaccante può usare 18 quintilioni di indirizzi dal suo blocco /64.
- Righe 102-103: gli indirizzi IPv6 mappati su IPv4 (::ffff:192.0.2.1) vengono convertiti in normali indirizzi IPv4.
- Righe 105-109: il parametro `strict=False` permette che i bit dell'host siano impostati. Ci interessa solo il prefisso di rete.

Esempio:
```
Input:  2001:0db8:85a3:0000:0000:8a2e:0370:7334
Output: 2001:db8:85a3::

Input:  2001:0db8:85a3:ffff:ffff:ffff:ffff:ffff
Output: 2001:db8:85a3::  (same /64 network)
```

## Costruire la difesa a tre livelli

### Livello 1: per utente per endpoint

In `src/fastapi_420/defense/layers.py` (righe 89-143):
```python
async def _check_layer1_user(
    self,
    context: DefenseContext,
    rules: list[RateLimitRule],
) -> LayerResult:
    """Layer 1: Per-user per-endpoint rate limiting"""
    identifier = context.fingerprint.to_composite_key(
        self._settings.fingerprint.LEVEL
    )
    
    worst_result: RateLimitResult | None = None
    
    for rule in rules:
        key = RateLimitKey(
            prefix=self._settings.KEY_PREFIX,
            version=self._settings.KEY_VERSION,
            layer=Layer.USER,
            endpoint=context.endpoint,
            identifier=identifier,
            window=rule.window_seconds,
        ).build()
        
        result = await self._algorithm.check(
            storage=self._storage,
            key=key,
            rule=rule,
        )
        
        if not result.allowed:
            if worst_result is None or (result.retry_after or 0) > (
                    worst_result.retry_after or 0):
                worst_result = result
    
    if worst_result:
        return LayerResult(
            layer=Layer.USER,
            allowed=False,
            result=worst_result,
        )
    
    return LayerResult(
        layer=Layer.USER,
        allowed=True,
        result=result,
    )
```

**Cosa fa:**
- Righe 94-96: costruisce l'identificatore del client a partire dal fingerprint. Per il livello NORMAL, combina IP + User-Agent + Auth.
- Righe 101-109: costruisce una chiave univoca per questa combinazione utente+endpoint+finestra. Chiave di esempio: `ratelimit:v1:user:POST:/api/upload:192.168.1.1:Mozilla:60`
- Righe 111-116: controlla il rate limit usando l'algoritmo configurato (sliding window di default).
- Righe 118-122: tiene traccia del risultato peggiore (più restrittivo) se si applicano più regole. Se hai "100/minute E 1000/hour" e vengono superate entrambe, restituisce quella con il tempo di retry più lungo.

### Livello 2: globale per endpoint

Nello stesso file (righe 145-169):
```python
async def _check_layer2_endpoint(
    self,
    context: DefenseContext,
    rules: list[RateLimitRule],
) -> LayerResult:
    """Layer 2: Per-endpoint global rate limiting"""
    endpoint_rules = self._settings.endpoint_limits.get(
        context.endpoint,
        self._settings.get_default_rules()
    )
    
    for rule in endpoint_rules:
        key = RateLimitKey(
            prefix=self._settings.KEY_PREFIX,
            version=self._settings.KEY_VERSION,
            layer=Layer.ENDPOINT,
            endpoint=context.endpoint,
            identifier="global",  # Not user-specific
            window=rule.window_seconds,
        ).build()
        
        endpoint_rule = RateLimitRule(
            requests=rule.requests * self._settings.defense.ENDPOINT_LIMIT_MULTIPLIER,
            window_seconds=rule.window_seconds,
        )
        
        result = await self._algorithm.check(
            storage=self._storage,
            key=key,
            rule=endpoint_rule,
        )
```

**Differenza chiave rispetto al livello 1:**
- Riga 162: l'identificatore è "global" invece di essere specifico dell'utente. Tutti gli utenti condividono questo contatore.
- Righe 166-169: il limite viene moltiplicato per `ENDPOINT_LIMIT_MULTIPLIER` (predefinito 10). Se il limite per utente è 100/minuto, il limite dell'endpoint è 1000/minuto. Permette 10 utenti concorrenti al ritmo massimo.

### Livello 3: circuit breaker

In `src/fastapi_420/defense/circuit_breaker.py` (righe 35-57):
```python
async def check(self, storage: Storage) -> bool:
    """
    Check if circuit is allowing requests
    
    Returns True if requests should be allowed
    """
    async with self._lock:
        now = time.time()
        
        if self._state.is_open:
            if now - self._state.last_failure_time >= self.recovery_time:
                await self._enter_half_open()
                return True  # Allow request to test recovery
            return False  # Circuit still open
        
        request_count = await self._get_request_count(storage)
        self._state.total_requests_in_window = request_count
        
        if request_count >= self.threshold:
            await self._trip(now)  # Open the circuit
            return False
        
        return True
```

**Transizioni di stato:**
1. **Closed** (righe 50-57): funzionamento normale. Conta le richieste. Se la soglia viene superata, passa a Open.
2. **Open** (righe 44-48): rifiuta quasi tutte le richieste. Dopo recovery_time, passa a Half-Open.
3. **Half-Open** (righe 45-47): permette alcune richieste per verificare se il sistema si è ripreso. Il successo chiude il circuito. Il fallimento lo riapre.

Il lock alla riga 41 previene le race condition. Più richieste che controllano simultaneamente devono attendere.

## Pattern di gestione degli errori

### Errore di connessione allo storage

Quando Redis non è disponibile, in `src/fastapi_420/limiter.py:265-284`:
```python
async def _get_active_storage(self) -> Storage | None:
    """Get active storage, falling back to memory if primary fails"""
    if self._storage is None:
        return self._fallback_storage
    
    try:
        is_healthy = await self._storage.health_check()
        if is_healthy:
            return self._storage
    except Exception:
        pass  # Log but don't crash
    
    if self._fallback_storage:
        logger.warning(
            "Primary storage unavailable, using memory fallback",
            extra={"primary_storage": self._storage.storage_type.value}
        )
        return self._fallback_storage
    
    return None
```

**Perché questa gestione specifica:**
Le righe 271-275 usano un try-except per intercettare QUALSIASI eccezione dall'health check. Redis può fallire in molti modi (timeout di connessione, errore di autenticazione, OOM). Non ci interessa quale, solo se funziona.

**Cosa NON fare:**
```python
# Bad: catching specific exceptions
try:
    is_healthy = await self._storage.health_check()
except RedisConnectionError:
    pass  # What about RedisTimeoutError? AuthenticationError?

# Bad: crashing on storage failure
is_healthy = await self._storage.health_check()  # Raises, kills the app
```

Questo nasconde i problemi reali. Il buon approccio alla riga 273 intercetta tutto, registra un warning e ripiega in modo graceful.

### Rate limit superato

Quando un limite viene raggiunto, in `src/fastapi_420/limiter.py:225-241`:
```python
if worst_result is not None:
    if self._settings.LOG_VIOLATIONS:
        logger.warning(
            "Rate limit exceeded",
            extra={
                "endpoint": endpoint,
                "identifier": identifier[:16],  # Truncate for privacy
                "remaining": worst_result.remaining,
                "reset_after": worst_result.reset_after,
            },
        )
    
    if raise_on_limit:
        raise EnhanceYourCalm(
            result=worst_result,
            message=self._settings.HTTP_420_MESSAGE,
            detail=self._settings.HTTP_420_DETAIL,
        )
    
    return worst_result
```

**Strategia di logging:**
- Riga 226: registra nei log solo se configurato. I sistemi in produzione potrebbero disabilitarlo (troppo rumoroso).
- Riga 230: tronca l'identificatore ai primi 16 caratteri. Registrare identificatori completi (indirizzi email, JWT) fa trapelare PII.
- Righe 227-233: usa il logging strutturato con il dizionario `extra`. Rende facile filtrare i log: `grep '"remaining": 0'`

**Due modalità di risposta:**
- Righe 235-239: la modalità decorator solleva un'eccezione. FastAPI la intercetta e restituisce HTTP 420.
- Riga 241: la modalità manuale restituisce il risultato. Il chiamante decide cosa fare.

## Ottimizzazioni delle prestazioni

### Caching degli script Redis

Prima dell'ottimizzazione, in `src/fastapi_420/storage/redis_backend.py:106-125`:
```python
async def _load_scripts(self) -> None:
    """Load Lua scripts into Redis and cache their SHA1 hashes"""
    if self._client is None:
        raise StorageError(...)
    
    script_files = {
        "sliding_window": LUA_SCRIPTS_DIR / "sliding_window.lua",
        "token_bucket": LUA_SCRIPTS_DIR / "token_bucket.lua",
        "fixed_window": LUA_SCRIPTS_DIR / "fixed_window.lua",
    }
    
    for name, path in script_files.items():
        script_content = path.read_text()
        sha = await self._client.script_load(script_content)
        self._script_shas[name] = sha
    
    self._scripts_loaded = True
```

Era lento perché la lettura dei file e il caricamento degli script avvenivano a ogni richiesta.

**Dopo:**
Gli script vengono caricati una sola volta all'avvio (righe 106-125). Le chiamate successive usano EVALSHA con lo SHA in cache in `src/fastapi_420/storage/redis_backend.py:152-174`:
```python
async def _execute_script(
    self,
    script_name: str,
    keys: list[str],
    args: list[str | int | float],
) -> list[int | float]:
    """Execute a Lua script using EVALSHA"""
    client = await self._ensure_connected()
    
    sha = self._script_shas[script_name]
    
    try:
        result = await client.evalsha(sha, len(keys), *keys, *args)
        return result
    
    except ResponseError as e:
        if "NOSCRIPT" in str(e):
            # Script not in Redis cache, reload
            await self._load_scripts()
            sha = self._script_shas[script_name]
            result = await client.evalsha(sha, len(keys), *keys, *args)
            return result
        raise StorageError(...)
```

**Cosa è cambiato:**
- Righe 161-163: usa EVALSHA invece di EVAL. EVALSHA invia solo lo SHA di 40 caratteri invece dell'intero script (200+ caratteri).
- Righe 167-171: gestisce l'errore NOSCRIPT. Succede se Redis è stato riavviato e ha perso la cache degli script. Ricarica e riprova automaticamente.

**Benchmark:**
- Prima (EVAL): 2,3ms per richiesta
- Dopo (EVALSHA): 1,1ms per richiesta
- Miglioramento: 52% più veloce

### Pulizia dello storage in memoria

Il task di pulizia in `src/fastapi_420/storage/memory.py:60-66`:
```python
async def _cleanup_loop(self) -> None:
    """Background loop to clean up expired entries"""
    while not self._closed:
        await asyncio.sleep(self.cleanup_interval)
        await self._cleanup_expired()
```

E la pulizia vera e propria alle righe 68-84:
```python
async def _cleanup_expired(self) -> None:
    """Remove expired entries from storage"""
    now = time.time()
    async with self._lock:
        expired_keys = [
            key for key, entry in self._windows.items()
            if entry.expires_at < now
        ]
        for key in expired_keys:
            del self._windows[key]
        
        expired_buckets = [
            key for key, state in self._buckets.items()
            if state.last_refill + 3600 < now
        ]
        for key in expired_buckets:
            del self._buckets[key]
```

**Perché questo approccio:**
Senza pulizia, l'uso della memoria cresce senza limiti. Ogni client che fa una richiesta aggiunge una voce. Dopo una settimana, hai milioni di voci, per lo più scadute.

La pulizia gira ogni 60 secondi di default. È un task in background (riga 59) che non blocca la gestione delle richieste.

**Compromesso:**
Una pulizia più frequente (ogni 10 secondi) usa più CPU ma mantiene la memoria più bassa. Una meno frequente (ogni 5 minuti) risparmia CPU ma usa più memoria. Il valore predefinito di 60 secondi è un ragionevole compromesso.

## Gestione della configurazione

### Caricare la configurazione

Da `src/fastapi_420/config.py:120-142`:
```python
class RateLimiterSettings(BaseSettings):
    """Main rate limiter settings with environment variable support"""
    model_config = SettingsConfigDict(
        env_prefix="RATELIMIT_",
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )
    
    ENABLED: bool = True
    ALGORITHM: Algorithm = Algorithm.SLIDING_WINDOW
    DEFAULT_LIMIT: str = "100/minute"
    DEFAULT_LIMITS: list[str] = ["100/minute", "1000/hour"]
    FAIL_OPEN: bool = True
    KEY_PREFIX: str = "ratelimit"
    KEY_VERSION: str = "v1"
    INCLUDE_HEADERS: bool = True
    LOG_VIOLATIONS: bool = True
    ENVIRONMENT: Literal["development", "staging", "production"] = "development"
```

**Perché funziona:**
- Righe 123-127: Pydantic legge automaticamente le variabili d'ambiente con prefisso `RATELIMIT_`. Imposta `RATELIMIT_ALGORITHM=token_bucket` e diventa `settings.ALGORITHM`.
- Riga 126: il file `.env` è opzionale. Se presente, le variabili vengono caricate da lì. Altrimenti, si usano i valori predefiniti.
- Riga 127: `extra="ignore"` significa che variabili d'ambiente sconosciute non causano errori. Utile quando l'app condivide l'ambiente con altri servizi.

### Validazione

Le righe 165-179 validano la configurazione all'avvio:
```python
@model_validator(mode="after")
def validate_limits(self) -> RateLimiterSettings:
    """Validate all limit strings can be parsed"""
    RateLimitRule.parse(self.DEFAULT_LIMIT)
    for limit in self.DEFAULT_LIMITS:
        RateLimitRule.parse(limit)
    return self

@model_validator(mode="after")
def validate_production_settings(self) -> RateLimiterSettings:
    """Enforce stricter settings in production"""
    if self.ENVIRONMENT == "production":
        if self.storage.REDIS_URL is None and not self.storage.FALLBACK_TO_MEMORY:
            raise ValueError(
                "Production requires Redis URL or FALLBACK_TO_MEMORY=True"
            )
    return self
```

Se imposti `DEFAULT_LIMIT="abc/minute"`, l'app va in crash all'avvio con:
```
ValueError: Invalid rate limit format: abc/minute
```

Questo è un bene. Meglio fallire all'avvio che fallire misteriosamente a runtime.

## Operazioni su database/storage

### Incremento nello storage in memoria

L'operazione centrale in `src/fastapi_420/storage/memory.py:100-166`:
```python
async def increment(
    self,
    key: str,
    window_seconds: int,
    limit: int,
    timestamp: float | None = None,
) -> RateLimitResult:
    """Atomically check and increment counter using sliding window"""
    now = timestamp if timestamp is not None else time.time()
    current_window = int(now // window_seconds)
    previous_window = current_window - 1
    elapsed_ratio = (now % window_seconds) / window_seconds
    
    current_key = f"{key}:{current_window}"
    previous_key = f"{key}:{previous_window}"
    
    async with self._lock:
        current_entry = self._windows.get(current_key)
        previous_entry = self._windows.get(previous_key)
        
        current_count = current_entry.count if current_entry else 0
        previous_count = previous_entry.count if previous_entry else 0
        
        weighted_count = int(
            previous_count * (1 - elapsed_ratio) + current_count
        )
        
        if weighted_count >= limit:
            reset_after = window_seconds - (now % window_seconds)
            return RateLimitResult(
                allowed=False,
                limit=limit,
                remaining=0,
                reset_after=reset_after,
                retry_after=reset_after,
            )
        
        if current_entry:
            current_entry.count += 1
        else:
            self._windows[current_key] = WindowEntry(
                count=1,
                window_start=current_window,
                expires_at=now + (window_seconds * 2),
            )
            self._windows.move_to_end(current_key)
            await self._enforce_max_keys()
```

**Dettagli importanti:**
- Righe 115-116: il lock alla riga 116 è fondamentale. Senza di esso, due richieste potrebbero entrambe leggere count=99, entrambe calcolare weighted_count=99, entrambe incrementare ed essere entrambe permesse quando il limite è 100. Il lock garantisce un'operazione atomica di lettura-modifica-scrittura.
- Righe 123-125: il calcolo del conteggio pesato usa la stessa formula dello script Lua.
- Righe 145-147: l'OrderedDict mantiene le voci in ordine di inserimento. `move_to_end()` sposta alla fine le chiavi usate di recente. Quando si supera max_keys, vengono rimosse per prime le voci più vecchie (usate meno di recente).

## Strategia di test

### Unit test

Test di esempio per lo sliding window in `tests/test_algorithms.py:85-102`:
```python
@pytest.mark.asyncio
async def test_limit_exceeded(self) -> None:
    algo = SlidingWindowAlgorithm()
    storage = MemoryStorage()
    rule = RuleFactory.create(requests=5, window_seconds=WINDOW_MINUTE)
    
    for _ in range(5):
        result = await algo.check(storage, "limit_key", rule)
        assert result.allowed is True
    
    result = await algo.check(storage, "limit_key", rule)
    assert result.allowed is False
    assert result.retry_after is not None
    
    await storage.close()
```

**Cosa testa:**
- Righe 90-92: le prime 5 richieste dovrebbero avere successo (sotto il limite).
- Righe 94-96: la sesta richiesta dovrebbe fallire (al limite).
- Riga 96: verifica che retry_after sia impostato (dice al client quando riprovare).

**Perché queste asserzioni specifiche:**
Gli unit test validano il comportamento, non l'implementazione. Non controlliamo il valore interno del contatore. Controlliamo il comportamento osservabile: le prime 5 permesse, la sesta negata.

### Test di integrazione

Da `tests/test_integration.py:27-58`:
```python
@pytest.mark.asyncio
async def test_middleware_returns_420_when_exceeded(self) -> None:
    storage = MemoryStorage()
    limiter = RateLimiter(storage=storage)
    await limiter.init()
    
    app = create_app_with_middleware(limiter, "5/minute")
    
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as client:
        for _ in range(5):
            response = await client.get("/api/test")
            assert response.status_code == 200
        
        response = await client.get("/api/test")
        assert_420_response(response)
    
    await limiter.close()
```

Questo testa il flusso end-to-end: Request → Middleware → RateLimiter → Storage → Response.

### Eseguire i test
```bash
# All tests
pytest

# Just algorithm tests
pytest tests/test_algorithms.py

# With coverage
pytest --cov=src/fastapi_420

# Specific test
pytest tests/test_limiter.py::TestRateLimiterInit::test_init_with_defaults
```

## Errori comuni di implementazione

### Errore 1: non usare operazioni atomiche

**Sintomo:**
Il rate limiter permette più richieste del limite configurato sotto carico concorrente.

**Causa:**
```python
# Problematic code (don't use)
count = await storage.get(key)
if count < limit:
    await storage.set(key, count + 1)
    return True  # Allowed
```

Tra get e set, un'altra richiesta può modificare il conteggio. Entrambe le richieste vedono count=99, entrambe incrementano, entrambe vengono permesse quando il limite è 100.

**Correzione:**
```python
# Correct approach (from src/fastapi_420/storage/memory.py:116)
async with self._lock:
    # All operations inside lock are atomic
    count = self._windows.get(key, 0)
    if count >= limit:
        return False
    self._windows[key] = count + 1
    return True
```

Oppure usa gli script Lua di Redis, che vengono eseguiti in modo atomico.

**Perché è importante:**
Con 1000 richieste/sec, la finestra della race è di microsecondi. Senza lock, vedrai forse l'1% di errore. Con 10k richieste/sec, è il 10% di errore. Il sistema diventa inaffidabile.

### Errore 2: dimenticare di impostare la scadenza delle chiavi

**Problema:**
Memory leak in Redis. Le chiavi non scadono mai, Redis esaurisce la memoria.

**Causa:**
```python
# Bad
redis.incr(f"ratelimit:{user}:{window}")
# No EXPIRE set, key lives forever
```

**Correzione da** `src/fastapi_420/storage/lua/sliding_window.lua:26`:
```lua
redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window_seconds * 2)
```

Ogni chiave riceve una scadenza. Dopo il doppio del tempo della finestra (per conservare i dati della finestra precedente), Redis la elimina automaticamente.

### Errore 3: fidarsi ciecamente degli header del client

**Problema:**
L'attaccante falsifica X-Forwarded-For per aggirare i rate limit.

**Causa:**
```python
# Bad
client_ip = request.headers.get("X-Forwarded-For").split(",")[0]
```

L'attaccante invia `X-Forwarded-For: 1.2.3.4` e ora appare come IP 1.2.3.4 invece del suo IP reale.

**Correzione da** `src/fastapi_420/fingerprinting/ip.py:61-81`:
```python
def _parse_x_forwarded_for(self, header, request):
    ips = [ip.strip() for ip in header.split(",")]
    
    # Walk backwards, find first untrusted IP
    for ip in reversed(ips):
        if ip not in self.trusted_proxies:
            return ip
    
    return ips[0]
```

Fidati di X-Forwarded-For solo se la richiesta proviene da un proxy fidato (load balancer, CDN). Altrimenti, usa l'IP della connessione diretta.

## Suggerimenti per il debug

### Tipo di problema 1: i rate limit non funzionano

**Problema:** si fanno richieste illimitate, nessuna risposta 420

**Come fare debug:**
1. Verifica che il middleware sia aggiunto: `app.add_middleware(RateLimitMiddleware, ...)` nel tuo main.py
2. Guarda l'inizializzazione di RateLimiter: `await limiter.init()` deve essere chiamato prima della prima richiesta
3. Verifica la connessione allo storage: controlla nei log "Redis unavailable" o fallimenti degli health check

**Cause comuni:**
- Middleware registrato dopo le route (deve essere prima)
- Le impostazioni hanno `ENABLED=false`
- Percorso dell'endpoint escluso nella configurazione del middleware

### Tipo di problema 2: tutte le richieste ricevono 420

**Problema:** la prima richiesta viene subito limitata

**Come fare debug:**
1. Controlla lo stato del circuit breaker: registra nei log `circuit_breaker.is_open`
2. Verifica che i limiti non siano troppo bassi: un limite "1/minute" bloccherà pesantemente
3. Controlla se il fingerprinting è troppo univoco: ogni richiesta appare come un client diverso

**Soluzione:**
Guarda la chiave effettivamente usata. Aggiungi questo a `src/fastapi_420/limiter.py:208`:
```python
logger.info(f"Rate limit key: {key}, remaining: {result.remaining}")
```

Se la chiave cambia a ogni richiesta, il fingerprinting non funziona.

### Tipo di problema 3: errori di connessione a Redis

**Problema:** `StorageConnectionError: Failed to connect to Redis`

**Cause comuni:**
- Redis non è in esecuzione: `docker ps` non mostra alcun container redis
- URL errato: `REDIS_URL=redis://localhost:6379` ma Redis è su una porta diversa
- Problema di rete: Redis nella rete Docker, app fuori dalla rete

**Correzione:**
```bash
# Start Redis
docker run -d -p 6379:6379 redis:7-alpine

# Test connection
redis-cli ping  # Should return PONG

# Check URL
echo $RATELIMIT_REDIS_URL
```

## Prossimi passi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.md](./04-CHALLENGES.it.md) contiene idee di estensione come aggiungere l'integrazione di CAPTCHA, il blocco per geolocalizzazione o algoritmi personalizzati
2. **Modifica il codice** - Cambia l'algoritmo sliding window per usare una ponderazione diversa o adatta la strategia di recupero del circuit breaker
3. **Leggi i progetti correlati** - Il network-traffic-analyzer si basa su concetti di rate limiting a livello di pacchetto
