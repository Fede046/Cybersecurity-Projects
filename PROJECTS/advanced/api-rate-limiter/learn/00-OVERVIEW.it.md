# API Rate Limiter (fastapi-420)

## Che cos'è

Una libreria di rate limiting pronta per la produzione per FastAPI che implementa tre algoritmi collaudati sul campo (sliding window, token bucket, fixed window) con storage Redis o in-memory, fingerprinting avanzato dei client e un sistema di difesa a tre livelli che scala dai limiti per singolo utente fino alla protezione DDoS globale.

## Perché è importante

Il 28 febbraio 2018, GitHub ha subito un attacco DDoS da 1,35 Tbps, il più grande mai registrato fino ad allora. L'attacco è durato solo 10 minuti perché i loro sistemi di rate limiting e di filtraggio del traffico sono entrati in azione. Senza un rate limiting adeguato, anche i picchi di traffico legittimo possono mandare giù le API. Quando è stato lanciato l'iPhone 15, i server di attivazione di Apple hanno faticato sotto il carico di milioni di richieste simultanee. Il rate limiting non riguarda solo la sicurezza, riguarda la disponibilità.

**Scenari reali in cui si applica:**
- **Prevenire il credential stuffing** - Nel 2019, Dunkin' Donuts ha subito un attacco di credential stuffing in cui gli attaccanti hanno effettuato milioni di tentativi di login usando credenziali rubate. Limitare gli endpoint di login a 3-5 tentativi al minuto blocca completamente questo schema di attacco.
- **Controllo dei costi delle API** - Se chiami API di terze parti costose o esegui inferenza ML, un singolo client che si comporta male può costarti migliaia di dollari in poche ore. I rate limit proteggono il tuo budget.
- **Distribuzione equa delle risorse** - Nel 2020, gli scalper hanno usato bot per comprare tutto l'inventario di PS5 pochi secondi dopo il lancio. I siti di e-commerce ora usano il rate limiting sugli endpoint di checkout per dare ai clienti reali una possibilità equa.

## Cosa imparerai

Questo progetto ti insegna come funziona davvero il rate limiting in produzione. Costruendolo da solo, capirai:

**Concetti di sicurezza:**
- **Architettura di difesa a tre livelli** - Perché ti servono limiti per utente (fermare l'abuso individuale), limiti per endpoint (prevenire attacchi specifici a un endpoint) e limiti globali (protezione DDoS). Un solo livello non basta. L'attacco DNS a Dyn del 2016 ha aggirato i limiti per utente usando milioni di dispositivi IoT.
- **Client fingerprinting** - Come identificare i client in modo affidabile quando gli IP non sono univoci (NAT, proxy, reti mobili). Imparerai perché fare fingerprinting dei soli indirizzi IP non funziona, soprattutto per IPv6 dove gli utenti controllano interi blocchi /64 (18 quintilioni di indirizzi).
- **Compromessi tra algoritmi** - Perché lo sliding window è consigliato per la produzione (accurato con memoria costante), quando ha senso il token bucket (tolleranza ai burst) e perché il fixed window ha un problema ai confini delle finestre che gli attaccanti sfruttano.

**Competenze tecniche:**
- **Operazioni atomiche con script Lua** - Il rate limiting richiede operazioni atomiche di lettura-modifica-scrittura. Vedrai come gli script Lua di Redis (memorizzati in `src/fastapi_420/storage/lua/`) garantiscono la correttezza anche sotto un pesante carico concorrente.
- **Integrazione del middleware ASGI** - Come agganciarsi al ciclo di vita delle richieste di FastAPI in `src/fastapi_420/middleware.py:37-106` per applicare i rate limit globalmente senza modificare ogni endpoint.
- **Pattern di Python asincrono** - Gestire richieste concorrenti con i lock di asyncio (`src/fastapi_420/storage/memory.py:44`), task di pulizia in background e shutdown graceful.

**Strumenti e tecniche:**
- **Redis per i sistemi distribuiti** - Lo storage in-memory in `src/fastapi_420/storage/memory.py` funziona per istanze singole, ma i sistemi in produzione hanno bisogno di Redis (`src/fastapi_420/storage/redis_backend.py`) per condividere lo stato tra più server.
- **Validazione delle impostazioni con Pydantic** - Il sistema di configurazione in `src/fastapi_420/config.py:120-164` valida le variabili d'ambiente all'avvio, fallendo subito se la configurazione è errata invece di causare errori a runtime.
- **Testare i rate limiter** - La test suite mostra pattern per testare codice dipendente dal tempo usando timestamp espliciti (`tests/test_algorithms.py:143`) invece di usare sleep nei test.

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**
- **async/await di Python** - L'intera codebase è asincrona. Devi conoscere la differenza tra `async def` e `def`, quando usare `await` e cosa fa `asyncio.create_task()`.
- **Basi di FastAPI** - Capire il middleware, la dependency injection con `Depends()` e come funziona il flusso richiesta/risposta. Se non hai mai costruito un'app FastAPI, fallo prima.
- **Header HTTP** - Sapere a cosa servono gli header `X-Forwarded-For`, `User-Agent` e `Authorization`. Il sistema di fingerprinting si basa molto sull'analisi degli header.

**Strumenti che ti serviranno:**
- **Python 3.12+** - Usa i nuovi type hint come `list[str]` invece di `List[str]`
- **Redis (opzionale)** - Per l'uso in produzione. Installalo con Docker: `docker run -d -p 6379:6379 redis:7-alpine`
- **httpx o curl** - Per testare gli endpoint delle API

**Utile ma non richiesto:**
- **Scripting Lua in Redis** - Il progetto include script Lua già scritti, ma capirli aiuta
- **Basi di OAuth/JWT** - L'extractor di autenticazione (`src/fastapi_420/fingerprinting/auth.py`) può fare il parsing dei token JWT

## Avvio rapido

Fai girare il progetto in locale:
```bash
# Clone and navigate
cd PROJECTS/advanced/api-rate-limiter

# Install dependencies
pip install -e . --break-system-packages

# Optional: Start Redis for production-like testing
docker compose -f examples/docker-compose.yml up -d

# Run the example app
python examples/app.py
```

Output atteso: il server si avvia su http://0.0.0.0:8000 con diversi endpoint soggetti a rate limit. Prova a chiamare l'endpoint di login restrittivo:
```bash
# First 3 requests work
curl http://localhost:8000/auth/login -X POST -d "username=test&password=test"

# 4th request gets HTTP 420
curl -i http://localhost:8000/auth/login -X POST -d "username=test&password=test"
```

Vedrai `HTTP/1.1 420 Enhance Your Calm` con gli header `Retry-After` e `RateLimit-*`.

## Struttura del progetto
```
api-rate-limiter/
├── src/fastapi_420/
│   ├── algorithms/           # Rate limiting algorithms
│   │   ├── sliding_window.py # Recommended default (99.997% accurate)
│   │   ├── token_bucket.py   # For burst tolerance
│   │   └── fixed_window.py   # Simple but has boundary issues
│   ├── storage/              # Storage backends
│   │   ├── memory.py         # In-memory (single instance)
│   │   ├── redis_backend.py  # Redis (distributed)
│   │   └── lua/              # Atomic Lua scripts
│   ├── fingerprinting/       # Client identification
│   │   ├── ip.py             # IP extraction with IPv6 /64 normalization
│   │   ├── headers.py        # User-Agent, Accept-* headers
│   │   ├── auth.py           # JWT, API keys, sessions
│   │   └── composite.py      # Combines all methods
│   ├── defense/              # Multi-layer protection
│   │   ├── layers.py         # User/Endpoint/Global limits
│   │   └── circuit_breaker.py# DDoS protection
│   ├── limiter.py            # Main RateLimiter class
│   ├── middleware.py         # ASGI middleware
│   ├── dependencies.py       # FastAPI dependency injection
│   ├── config.py             # Settings with Pydantic
│   └── types.py              # Data structures
├── examples/
│   └── app.py                # Full working example
└── tests/                    # Comprehensive test suite
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per imparare i fondamenti del rate limiting, le differenze tra gli algoritmi e perché il fingerprinting è importante
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere il sistema di difesa a tre livelli e il flusso dei dati
3. **Ripercorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per i dettagli implementativi riga per riga
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee come aggiungere il blocco per geolocalizzazione, l'integrazione di CAPTCHA o algoritmi personalizzati

## Problemi comuni

**"Redis connection failed" ma voglio usare lo storage in memoria**
```
REDIS_URL not set, fallback to memory storage enabled
```
Soluzione: va bene per lo sviluppo. L'impostazione `FALLBACK_TO_MEMORY=True` (predefinita) passa automaticamente allo storage in-memory quando Redis non è disponibile. Per la produzione, imposta la variabile d'ambiente `REDIS_URL`.

**"Il rate limit non funziona per richieste da IP diversi"**
Soluzione: verifica se sei dietro un proxy. Se usi nginx o cloudflare, imposta `TRUST_X_FORWARDED_FOR=True` nelle tue impostazioni in modo che l'extractor degli IP legga l'header `X-Forwarded-For` invece dell'IP del proxy. Vedi `src/fastapi_420/fingerprinting/ip.py:52-68` per la logica di gestione dei proxy.

**"Ricevo HTTP 420 subito alla prima richiesta"**
Controlla la soglia del circuit breaker. Se `CIRCUIT_THRESHOLD` è troppo bassa (tipo 100) e stai facendo load testing, potrebbe scattare il circuit breaker globale. Vedi `src/fastapi_420/defense/circuit_breaker.py:45-55` per la logica della soglia.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **network-traffic-analyzer** - Si basa sull'ispezione dei pacchetti per rilevare attacchi DDoS prima che colpiscano la tua API
- **docker-security-audit** - Mostra come verificare le configurazioni di rate limiting nei deployment containerizzati
- **bug-bounty-platform** - Implementa il rate limiting per gli endpoint di invio per prevenire lo spam
