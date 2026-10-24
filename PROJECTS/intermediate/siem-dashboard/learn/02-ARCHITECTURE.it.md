# Architettura del sistema

Come è progettata la dashboard SIEM, perché esiste ciascun componente e come i dati si muovono attraverso il sistema.

## Architettura ad alto livello

```
┌──────────────────────────────────────────────────────────────┐
│                        Browser                                │
│  React 19 + TypeScript + Zustand + TanStack Query            │
│  SSE Listeners (EventSource)                                  │
└──────────────┬────────────────────────────┬──────────────────┘
               │ HTTP/JSON                  │ SSE
               ▼                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     Nginx Reverse Proxy                       │
│  Rate limiting · Gzip · Static assets · SSE passthrough      │
│  proxy_buffering off (for /stream endpoints)                 │
└──────────────┬────────────────────────────┬──────────────────┘
               │                            │
               ▼                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     Flask Backend                             │
│                                                               │
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐      │
│  │ Routes  │→ │ Controllers  │→ │ Engine             │      │
│  │ (auth,  │  │ (validation, │  │ (normalizer,       │      │
│  │  logs,  │  │  business    │  │  severity,         │      │
│  │  alerts │  │  logic)      │  │  correlation)      │      │
│  │  rules) │  │              │  │                    │      │
│  └─────────┘  └──────────────┘  └────────────────────┘      │
│                                                               │
│  ┌──────────────────────┐  ┌──────────────────────────┐      │
│  │ Scenario Runner      │  │ Correlation Engine        │      │
│  │ (daemon threads,     │  │ (daemon thread,           │      │
│  │  playbook playback)  │  │  XREADGROUP consumer)     │      │
│  └──────────────────────┘  └──────────────────────────┘      │
└──────────┬──────────────────────────┬────────────────────────┘
           │                          │
           ▼                          ▼
┌────────────────────┐    ┌────────────────────┐
│    MongoDB 8.0     │    │   Redis 7 Alpine   │
│                    │    │                    │
│  log_events        │    │  siem:logs stream  │
│  alerts            │    │  siem:alerts stream│
│  correlation_rules │    │  rate limit keys   │
│  users             │    │                    │
│  scenario_runs     │    │                    │
└────────────────────┘    └────────────────────┘
```

### Dettaglio dei componenti

**Nginx Reverse Proxy**
Si trova davanti a tutto. Gestisce la terminazione TLS in produzione, il rate limiting sul perimetro (`limit_req_zone` in `conf/nginx/nginx.conf:34-36`), la compressione gzip e la cache degli asset statici. Il dettaglio critico è la configurazione di passthrough per SSE. Gli endpoint API normali usano un proxying con buffering, ma gli endpoint `/stream` richiedono `proxy_buffering off` e un read timeout di 3600s (`conf/nginx/prod.nginx:30-43`). Senza questo, Nginx mette in buffer gli eventi SSE e il browser non riceve nulla finché il buffer non si riempie.

**Flask Backend**
Application factory pattern in `backend/app/__init__.py`. Crea l'app Flask, collega le connessioni a MongoDB e Redis, registra i blueprint, inizializza il rate limiting, crea i consumer group di Redis, ripulisce gli scenario run orfani e avvia il correlation engine. Quell'ordine di avvio è importante. I consumer group devono esistere prima che il correlation engine provi a leggerne.

**MongoDB**
Data store primario per tutti i documenti persistenti. L'ODM MongoEngine mappa le classi Python sulle collection. Cinque collection principali: `log_events`, `alerts`, `correlation_rules`, `users`, `scenario_runs`. Ognuna ha indici specifici definiti nel dict `meta` del modello per le performance delle query.

**Redis**
Due ruoli. Primo, alimenta la pipeline di streaming. Due Redis Streams (`siem:logs` e `siem:alerts`) gestiscono il fanout pub/sub tra l'ingestione dei log, il correlation engine e gli endpoint SSE. Secondo, fa da backend per il rate limiter (`flask-limiter` usa Redis come backend di storage tramite la configurazione `REDIS_URL`).

**React Frontend**
Single page app con React 19, TypeScript e una netta separazione tra il data fetching (hook TanStack Query in `frontend/src/api/hooks/`) e lo stato della UI (store Zustand in `frontend/src/core/stores/`). Gli aggiornamenti in tempo reale arrivano tramite due connessioni SSE gestite dagli hook `useLogStream` e `useAlertStream`.

## Flusso dei dati

### Pipeline di ingestione dei log

Questa è la pipeline centrale. Ogni evento di log, che provenga dallo scenario runner o da una POST esterna, segue lo stesso percorso:

```
1. Raw event arrives
   POST /v1/logs/ingest  (or ScenarioRunner._emit_event)
   │
2. Normalize (app/engine/normalizer.py)
   │  Dispatches to source_type-specific normalizer
   │  Extracts common fields, preserves raw payload
   │
3. Classify severity (app/engine/severity.py)
   │  Event type lookup → regex pattern matching → default to info
   │
4. Persist to MongoDB (app/models/LogEvent.py)
   │  LogEvent.create_event() → document saved with all fields
   │
5. Publish to Redis Stream (app/core/streaming.py)
   │  XADD siem:logs {payload: JSON}
   │  Maxlen ~10000 (approximate trim)
   │
   ├──→ 6a. Correlation Engine reads via XREADGROUP
   │    │   Evaluates all enabled rules against event
   │    │   If rule fires → create Alert → XADD siem:alerts
   │    │   ACK message after processing
   │    │
   │    └──→ SSE /v1/alerts/stream (browser picks up new alerts)
   │
   └──→ 6b. SSE /v1/logs/stream
        │   XREAD with blocking, yields to EventSource
        └── Browser pushes to Zustand StreamStore
```

La cosa importante qui è che i passaggi 4 e 5 avvengono in modo atomico dal punto di vista del controller. L'evento viene persistito prima di arrivare nello stream. Se Redis è giù, l'evento è comunque in MongoDB. Se MongoDB è giù, non viene pubblicato nulla perché `create_event()` genererebbe prima un'eccezione.

### Ciclo di vita degli alert

Gli alert hanno una macchina a stati con cinque status:

```
new → acknowledged → investigating → resolved
                                   → false_positive
```

Quando il correlation engine fa scattare una regola (`app/engine/correlation.py:_process_event`), chiama `Alert.create_from_rule()`, che salva l'alert in MongoDB e lo pubblica nel Redis Stream `siem:alerts`. Il documento dell'alert fa riferimento agli ID degli eventi corrispondenti, così gli analisti possono approfondire cosa ha fatto scattare l'alert.

Le transizioni di stato avvengono tramite `PATCH /v1/alerts/<id>/status` e vengono tracciate con timestamp. Il campo `acknowledged_by` registra quale analista ha preso in carico l'alert, e `resolved_at` segna il momento della chiusura.

### Flusso di autenticazione

```
Register/Login
    │
    ▼
POST /v1/auth/register  or  POST /v1/auth/login
    │                           │
    ▼                           ▼
hash_password()            verify_password_timing_safe()
(Argon2id)                 (constant time, dummy hash for missing users)
    │                           │
    ▼                           ▼
User.create_user()          Validate credentials
    │                           │
    └───────┬───────────────────┘
            ▼
    create_access_token()
    JWT with sub=user_id, username, role, exp
            │
            ▼
    Return {access_token, token_type: "bearer"}
            │
            ▼
    Frontend stores in Zustand (persisted to localStorage)
    Axios interceptor attaches to every request
```

Ogni richiesta successiva passa dal decorator `endpoint()`, che estrae il Bearer token, decodifica il JWT, carica l'utente da MongoDB e lo associa a `g.current_user` di Flask. Il decorator gestisce anche il gating dei ruoli. Passa `roles=["admin"]` e i non-admin ricevono un 403.

## Design pattern

### Application Factory

**Dove si trova:** `backend/app/__init__.py`

La funzione `create_app()` costruisce l'app Flask da zero ogni volta che viene chiamata. Non è solo una convenzione di Flask. Risolve problemi reali: isolamento dei test (ogni test ottiene un'app nuova), flessibilità di configurazione (scambiare i file `.env` tra dev/prod) e problemi di ordine degli import (le estensioni si inizializzano dopo che la configurazione dell'app è impostata).

L'ordine di inizializzazione è deliberato:

1. Caricamento della configurazione (da variabili d'ambiente tramite Pydantic)
2. Setup di CORS
3. Connessioni a MongoDB e Redis
4. Error handler
5. Rate limiter
6. Registrazione dei blueprint
7. Creazione dei consumer group
8. Pulizia degli scenari orfani
9. Avvio del correlation engine

Se sposti il passaggio 7 dopo il passaggio 9, il correlation engine andrà in crash cercando di leggere da un consumer group che non esiste ancora.

### Decorator Stack Pattern

**Dove si trova:** `backend/app/core/decorators/`

Ogni route handler usa lo stesso decorator stack: `@endpoint` → `@S` → `@R`. È una pipeline componibile:

```python
# backend/app/routes/logs.py:32-39
@logs_bp.post("/ingest")
@endpoint(auth_required=False)
@S(LogIngestRequest)
@R(status=201)
def ingest_log() -> Any:
    return log_ctrl.ingest_log()
```

`@endpoint` gestisce l'estrazione del JWT, il caricamento dell'utente, l'enforcement dei ruoli e i confini degli errori. `@S` (Schema) valida i dati della richiesta con Pydantic e salva il risultato in `g.validated`. `@R` (Response) serializza automaticamente il valore di ritorno in JSON con il corretto status code.

I decorator vengono eseguiti dall'esterno verso l'interno: prima endpoint (controllo auth), poi S (validazione), poi il corpo della funzione, poi R (serializzazione). Se l'auth fallisce, la validazione non viene mai eseguita. Se la validazione fallisce, il controller non viene mai eseguito. Questo approccio fail-fast mantiene pulito il codice dei controller.

**Compromessi:**
Il decorator stack è conciso ma può essere confuso da debuggare. Gli stack trace attraversano più livelli di wrapper. Se aggiungi un nuovo decorator, l'ordine conta e sbagliarlo non produce errori evidenti.

### Registry Pattern per i normalizer

**Dove si trova:** `backend/app/engine/normalizer.py`

Ogni source type (firewall, IDS, auth, endpoint, DNS, proxy, generic) ha la propria funzione normalizer registrata tramite il decorator `@_register`:

```python
# backend/app/engine/normalizer.py:16-22
def _register(source_type: SourceType) -> Callable[[NormalizerFn], NormalizerFn]:
    def decorator(fn: NormalizerFn) -> NormalizerFn:
        NORMALIZERS[source_type.value] = fn
        return fn
    return decorator
```

Il dispatcher `normalize()` cerca la funzione giusta nel dict `NORMALIZERS` e ricade su `_normalize_generic` per i tipi sconosciuti. Aggiungere un nuovo source type significa scrivere una funzione e aggiungere il decorator. Nessuno switch statement, nessuna catena if/elif, nessuna modifica alla logica di dispatch.

### Stato thread-safe con i lock

**Dove si trova:** `backend/app/engine/correlation.py` (classe `CorrelationState`)

Il correlation engine gira su un daemon thread ma condivide strutture di stato (sliding window, timestamp di cooldown) a cui si potrebbe accedere durante il test delle regole dal thread principale di Flask. Ogni metodo di `CorrelationState` acquisisce `self._lock` prima di toccare `_windows` o `_cooldowns`. Questo previene le data race ma significa che il correlation engine non può elaborare due eventi in parallelo. Per il throughput a cui mira questo progetto, va bene.

## Separazione dei livelli

```
┌────────────────────────────────────────────────────┐
│    Routes Layer (app/routes/)                       │
│    HTTP concerns only: URL mapping, rate limits     │
│    Does NOT: query databases, process data          │
└───────────────────────┬────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────┐
│    Controller Layer (app/controllers/)               │
│    Business logic: orchestrate models and engine    │
│    Does NOT: parse requests, format responses       │
└───────────────────────┬────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────┐
│    Model Layer (app/models/)                         │
│    Data access: MongoEngine documents, queries      │
│    Does NOT: know about HTTP, Flask request/response│
└───────────────────────┬────────────────────────────┘
                        ▼
┌────────────────────────────────────────────────────┐
│    Engine Layer (app/engine/)                        │
│    Domain logic: normalization, severity, correlation│
│    Does NOT: persist data, know about Flask          │
└────────────────────────────────────────────────────┘
```

### Cosa vive dove

**Routes** (`app/routes/`): Definizioni dei blueprint, pattern degli URL, decorator stack. Ogni funzione di route è un thin wrapper che chiama la corrispondente funzione del controller. Le route importano da controller e schema, mai direttamente dai modelli.

**Controllers** (`app/controllers/`): Coordinamento della business logic. Il controller per l'ingestione dei log (`log_ctrl.ingest_log`) chiama il normalizer, il classificatore di severità, il `create_event` del modello e il publisher dello streaming. I controller accedono a `g.validated` per l'input e a `g.current_user` per il contesto di autenticazione.

**Models** (`app/models/`): Definizioni dei documenti MongoEngine con metodi di query. `BaseDocument` fornisce funzionalità condivise come `get_by_id`, `paginate` e timestamp `updated_at` aggiornati automaticamente. I modelli possono chiamare altri modelli (Alert fa riferimento a LogEvent) ma non importano mai da route o controller.

**Engine** (`app/engine/`): Logica di dominio pura. Il normalizer, il classificatore di severità e il correlation engine vivono qui. Il correlation engine è l'unico componente che raggiunge sia il livello di streaming (per leggere gli eventi) sia il livello dei modelli (per creare gli alert). È un compromesso pragmatico per mantenere il daemon thread autonomo.

## Modelli di dati

### LogEvent

```python
# backend/app/models/LogEvent.py
class LogEvent(BaseDocument):
    meta = {
        "collection": "log_events",
        "ordering": ["-timestamp"],
        "indexes": [
            "timestamp", "source_type", "severity",
            "source_ip", "dest_ip", "username",
            "hostname", "event_type", "scenario_run_id",
        ],
    }
    timestamp       = DateTimeField()
    source_type     = StringField(required=True)   # firewall, ids, auth, etc.
    source_ip       = StringField()
    dest_ip         = StringField()
    source_port     = IntField()
    dest_port       = IntField()
    severity        = StringField(default="info")
    event_type      = StringField()                # login_failure, port_scan, etc.
    raw             = DictField()                  # original payload, untouched
    normalized      = DictField()                  # source-type-specific fields
    hostname        = StringField()
    username        = StringField()
    scenario_run_id = ObjectIdField()              # links to ScenarioRun if simulated
```

Nove indici coprono i pattern di query usati dal log viewer, dalle ricerche pivot e dalle aggregazioni della dashboard. Il campo `raw` preserva l'evento originale esattamente come è stato inviato. Il campo `normalized` contiene i campi specifici del source type estratti dal normalizer. Questo doppio storage significa che puoi sempre tornare ai dati originali se il normalizer aveva un bug.

### Alert

```python
# backend/app/models/Alert.py
class Alert(BaseDocument):
    rule_id             = ObjectIdField(required=True)
    rule_name           = StringField(required=True)
    severity            = StringField(required=True)
    title               = StringField(required=True)   # "{rule_name} [{group_value}]"
    matched_event_ids   = ListField(ObjectIdField())   # references to LogEvent docs
    matched_event_count = IntField(default=0)
    group_value         = StringField()                # the IP, username, etc. that grouped
    status              = StringField(default="new")
    mitre_tactic        = StringField()
    mitre_technique     = StringField()
    acknowledged_by     = StringField()
    acknowledged_at     = DateTimeField()
    resolved_at         = DateTimeField()
```

Gli alert rimandano sia alla regola che li ha generati (`rule_id`) sia agli eventi specifici che hanno fatto match (`matched_event_ids`). Il metodo `get_with_events()` carica i documenti LogEvent referenziati per la vista di dettaglio dell'alert. È un join manuale, dato che MongoDB non fa join relazionali, ma la lista di ID è limitata dalla dimensione della finestra di correlazione, quindi non arriva mai a migliaia di documenti.

### CorrelationRule

```python
# backend/app/models/CorrelationRule.py
class CorrelationRule(BaseDocument):
    name         = StringField(required=True, unique=True)
    rule_type    = StringField(required=True)   # threshold, sequence, aggregation
    conditions   = DictField(required=True)     # type-specific config
    severity     = StringField(required=True)
    enabled      = BooleanField(default=True)
    mitre_tactic    = StringField()
    mitre_technique = StringField()
```

Il campo `conditions` è un dict flessibile la cui forma dipende da `rule_type`. Per le regole threshold contiene `event_filter`, `threshold`, `window_seconds` e `group_by`. Per le regole sequence ha `steps` (una lista ordinata di filtri sugli eventi). Per le regole aggregation aggiunge `aggregation_field` per contare i valori distinti. La validazione di queste forme avviene negli schema Pydantic (`backend/app/schemas/rule.py:60-82`) tramite un `@model_validator` che smista allo schema di condizioni corretto in base a `rule_type`.

## Architettura di sicurezza

### Threat model

Da cosa si difende la piattaforma:

1. **Credential stuffing sull'endpoint di login.** Il rate limiting sia su Nginx (3r/s per gli endpoint di auth) sia su Flask (10/minuto tramite `flask-limiter`) rende impraticabile il brute force. La funzione `verify_password_timing_safe` previene l'enumerazione degli username tramite differenze di timing.

2. **Accesso non autorizzato ai dati del SIEM.** Autenticazione basata su JWT su ogni endpoint API (tranne `/v1/logs/ingest` e le route di auth pubbliche). Il controllo degli accessi basato sui ruoli protegge le operazioni di admin. Il decorator `endpoint()` lo applica in modo uniforme.

3. **Privilege escalation tramite manipolazione dei ruoli.** Il controller `update_role` in `admin_ctrl.py:36-43` controlla `User.count_admins()` prima di consentire un declassamento. Non puoi declassare l'ultimo admin. Gli admin inoltre non possono disattivare o eliminare il proprio account.

4. **Leak dei token SSE.** Le connessioni SSE non possono usare l'header Authorization (limitazione del browser con EventSource), quindi i token vanno nella query string. La funzione `extract_bearer_token()` in `app/core/auth.py:97-102` controlla prima l'header, poi ricade su `request.args.get("token")`. È un compromesso noto. Il token compare nei log di accesso del server ma viene trasmesso su HTTPS.

Cosa è fuori scope:
Gli attacchi a livello di rete (gestiti dall'infrastruttura), la database injection (MongoEngine parametrizza le query), l'XSS (React fa l'escape di default, più gli header di sicurezza di Nginx).

### Defense in Depth

```
Nginx Layer
├── Rate limiting (10r/s API, 3r/s auth)
├── Connection limits (50 per IP in prod)
├── Security headers (X-Frame-Options, X-Content-Type-Options, etc.)
└── Request size limits (10MB max body)
    │
Flask Layer
├── flask-limiter (moving window, 200/min default, 10/min auth)
├── Pydantic validation (all inputs validated before processing)
├── JWT verification (expiry, required claims)
└── Role enforcement (decorator-based RBAC)
    │
Application Layer
├── Argon2id password hashing
├── Constant-time password verification
├── Last-admin protection
└── Self-action prevention (can't deactivate your own account)
```

## Strategia di storage

### MongoDB: stato persistente

Tutti i documenti che devono sopravvivere ai riavvii. Log event, alert, regole di correlazione, utenti, scenario run. MongoEngine fornisce il livello ODM con coercizione dei tipi e validazione.

La classe `BaseDocument` (`app/models/Base.py`) aggiunge `created_at` e `updated_at` a ogni documento e fa l'override di `save()` per aggiornare automaticamente il timestamp. Il metodo di classe `paginate()` fornisce una paginazione coerente basata su offset su tutte le collection.

Strategia degli indici: ogni campo usato in un filtro o in un ordinamento ottiene un indice. LogEvent ha nove indici, il che è aggressivo ma appropriato per una dashboard pesante sulle query. Le performance di scrittura subiscono un piccolo calo, ma le letture (che dominano in un SIEM) restano veloci.

### Redis: stream effimeri e rate limit

Due Redis Streams con maxlen approssimativo di 10.000 voci ciascuno. Gli stream sono effimeri. Riavviare Redis fa perdere i messaggi non elaborati, ma poiché gli eventi vengono prima persistiti in MongoDB, l'unico impatto è che il correlation engine potrebbe perdere alcuni eventi durante la finestra di riavvio. Per un progetto didattico, è un compromesso accettabile. I SIEM di produzione userebbero code persistenti.

Anche i contatori del rate limit vivono in Redis. La strategia `moving-window` di `flask-limiter` memorizza contatori a sliding window indicizzati per IP del client.

## Configurazione

### Variabili d'ambiente

Tutta la configurazione passa da `backend/app/config.py` usando Pydantic Settings:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )
    MONGO_URI: str = "mongodb://mongo:27017/siem"
    REDIS_URL: str = "redis://redis:6379/0"
    SECRET_KEY: str = "change-me-in-production"
    JWT_EXPIRATION_HOURS: int = 24
    CORRELATION_COOLDOWN_SECONDS: int = 300
    RATELIMIT_DEFAULT: str = "200/minute"
    RATELIMIT_AUTH: str = "10/minute"
    # ... ~40 more settings with defaults
```

Ogni impostazione ha un default sensato, quindi il progetto funziona subito con il solo `docker compose up`. Ma i default esplicitamente non sono sicuri per la produzione. Il default di `SECRET_KEY` è `"change-me-in-production"` e Redis non ha password in modalità dev.

### Sviluppo vs produzione

**Sviluppo** (`dev.compose.yml`): Hot reload sia sul frontend sia sul backend. Nessun rate limiting di auth su Nginx. Redis senza password. Porta di MongoDB esposta sull'host. Il frontend gira come dev server Vite con HMR.

**Produzione** (`compose.yml`): Gunicorn con 4 worker. Frontend pre-buildato e servito come file statici da Nginx. Redis richiede `REDIS_PASSWORD`. Nginx aggiunge header di sicurezza, limiti di connessione e caching aggressivo per gli asset statici. Sono impostati limiti di risorse Docker (il backend ottiene 2 CPU, 1GB di RAM).

## Considerazioni sulle performance

### Colli di bottiglia

**Il correlation engine è single-threaded.** Il loop `CorrelationEngine._run()` elabora gli eventi in sequenza. Sotto carico pesante (centinaia di eventi al secondo), il consumer group accumulerà un backlog. L'impostazione `STREAM_READ_COUNT` (default 10) e `STREAM_BLOCK_MS` (default 2000) controllano la dimensione del batch e il comportamento di blocking.

**Aggregazioni MongoDB per la dashboard.** I metodi `timeline_aggregation`, `severity_breakdown` e `top_sources` eseguono tutti aggregation pipeline. Con milioni di log event, diventano lenti. Il raggruppamento `$dateTrunc` nella pipeline della timeline è particolarmente costoso senza un indice composto su `(timestamp, severity)`.

**Le connessioni SSE trattengono i thread.** Ogni client SSE tiene aperto un worker thread di Gunicorn per l'intera durata della connessione. Con 4 worker e 3 client SSE, solo 1 worker è disponibile per le richieste API normali. La produzione richiederebbe più worker oppure una soluzione SSE asincrona.

### Ottimizzazioni già presenti

**Caching delle regole.** Il correlation engine mette in cache le regole abilitate per `CORRELATION_RULE_CACHE_SECONDS` (default 30s) per evitare di interrogare MongoDB a ogni evento. Vedi `CorrelationEngine._get_rules()`.

**Trimming approssimativo dello stream.** `XADD` con `approximate=True` permette a Redis di fare il trim dello stream in modo lazy invece che a ogni scrittura. Supera leggermente il maxlen di 10.000 ma evita l'overhead del trim a ogni scrittura.

**Pulizia delle sliding window.** `CorrelationState.get_window()` elimina le voci scadute a ogni lettura di una finestra, mantenendo la memoria limitata senza un thread di cleanup separato.

## Architettura di deploy

### Topologia Docker Compose

```
                   ┌──────────┐
        :8431 ────→│  Nginx   │
                   └────┬─────┘
                        │
              ┌─────────┴─────────┐
              │                   │
         ┌────▼────┐        ┌────▼────┐
         │ Backend │        │ Static  │
         │ (Flask) │        │ (built  │
         │ :5000   │        │  React) │
         └────┬────┘        └─────────┘
              │
     ┌────────┴────────┐
     │                 │
┌────▼────┐     ┌──────▼─────┐
│ MongoDB │     │   Redis    │
│  :27017 │     │   :6379    │
└─────────┘     └────────────┘
```

Due network Docker isolano il traffico. Il network `frontend` collega Nginx al backend. Il network `backend` collega il backend a MongoDB e Redis. MongoDB e Redis non sono direttamente accessibili dal network frontend.

Limiti di risorse in produzione:
- Nginx: 1 CPU, 256MB di RAM
- Backend: 2 CPU, 1GB di RAM
- MongoDB: 1 CPU, 512MB di RAM
- Redis: 0.5 CPU, 256MB di RAM

### Health check

Ogni servizio ha un health check. Il backend espone `/health`, che restituisce `"1"` (definito in `app/__init__.py`). MongoDB usa `mongosh --eval "db.adminCommand('ping')"`. Redis usa `redis-cli ping`. Nginx dipende dal fatto che il backend sia healthy prima di iniziare ad accettare traffico.

## Decisioni di design

### MongoDB invece di PostgreSQL

MongoDB è stato scelto perché i log event sono semi-strutturati. Source type diversi hanno campi diversi (gli eventi del firewall hanno `action` e `protocol`, gli eventi DNS hanno `query` e `query_type`). Memorizzarli in uno schema relazionale significherebbe una tabella larga e sparsa oppure una tabella separata per ogni source type. I documenti flessibili di MongoDB con il `DictField` per `raw` e `normalized` gestiscono questo caso in modo naturale.

Compromesso: nessun join transazionale. La vista di dettaglio dell'alert richiede una ricerca manuale degli eventi corrispondenti per ID. Con PostgreSQL sarebbe una singola query JOIN.

### Redis Streams invece di Kafka o RabbitMQ

Redis Streams offre abbastanza pub/sub per questo caso d'uso senza aggiungere un'altra dipendenza infrastrutturale. La semantica dei consumer group (`XREADGROUP`, `XACK`) garantisce un'elaborazione exactly-once per il correlation engine. Gli endpoint SSE usano `XREAD` semplice (senza consumer group) dato che si limitano a seguire lo stream per la visualizzazione.

Compromesso: nessuna garanzia di persistenza, nessuna replica multi-nodo, backpressure limitato. Un SIEM di produzione richiederebbe qualcosa di più robusto.

### JWT invece delle sessioni

L'autenticazione stateless significa che il backend non deve memorizzare sessioni in Redis o MongoDB. Ogni richiesta porta con sé la propria prova di identità. Il frontend memorizza il token in Zustand (persistito su localStorage tramite il middleware `persist` in `frontend/src/core/stores/auth.store.ts`).

Compromesso: non puoi invalidare un JWT prima che scada. Se l'account di un utente viene disattivato, può comunque fare richieste finché non scade il claim `exp` del token. Il decorator `endpoint()` mitiga il problema caricando l'utente da MongoDB a ogni richiesta e controllando `is_active`.

### Pydantic per la validazione delle richieste

Ogni endpoint valida l'input tramite schema Pydantic con il decorator `@S`. Questo intercetta i dati errati al confine, prima che raggiungano la logica del controller. Gli schema delle regole di correlazione sono particolarmente interessanti. `RuleCreateRequest` usa un `@model_validator` per smistare la validazione delle condizioni in base a `rule_type`, quindi le regole threshold vengono validate in modo diverso dalle regole sequence.

Compromesso: Pydantic aggiunge tempo di import e un livello di indirezione. Gli endpoint semplici che prendono solo un ID non hanno bisogno di validazione ma attraversano comunque il decorator stack.

## Strategia di gestione degli errori

### Gerarchia degli errori

```python
# backend/app/core/errors.py
AppError (500)
├── NotFoundError (404)
├── ValidationError (422, includes field-level details)
├── AuthenticationError (401)
├── ForbiddenError (403)
└── ConflictError (409)
```

Tutti gli errori personalizzati estendono `AppError`, che porta con sé uno `status_code` e un `message`. La funzione `register_error_handlers()` in `errors.py` collega un singolo error handler di Flask per `AppError` che serializza qualsiasi sottoclasse in una risposta JSON coerente: `{"error": "ErrorClassName", "message": "..."}`.

Il decorator `endpoint()` aggiunge un catch-all per le eccezioni impreviste. Se un controller solleva qualcosa che non è un `AppError`, il decorator registra il traceback tramite structlog e restituisce un generico 500. Questo impedisce che gli stack trace trapelino verso il client.

### Gestione degli errori nel frontend

Il frontend rispecchia questo schema con `ApiError` in `frontend/src/core/lib/errors.ts`. L'interceptor di risposta di Axios trasforma gli errori HTTP in istanze `ApiError` tipizzate con codici come `AUTHENTICATION_ERROR`, `VALIDATION_ERROR`, ecc. La cache di React Query ha error handler globali che mostrano notifiche toast per i fallimenti delle query in background.

## Estensibilità

### Aggiungere un nuovo source type di log

1. Aggiungi il source type all'enum `SourceType` in `backend/app/models/LogEvent.py`
2. Scrivi una funzione normalizer in `backend/app/engine/normalizer.py`:

```python
@_register(SourceType.YOUR_NEW_TYPE)
def _normalize_your_type(raw: dict[str, Any]) -> dict[str, Any]:
    return {
        "your_field": raw.get("your_field"),
        # ... extract source-specific fields
    }
```

3. Facoltativamente aggiungi pattern di severità in `backend/app/engine/severity.py`

Fine. Il registry pattern significa che non servono altre modifiche al codice. L'endpoint di ingest, lo streaming SSE, il correlation engine e il log viewer del frontend funzionano tutti con qualsiasi source type.

### Aggiungere un nuovo tipo di regola di correlazione

1. Aggiungi il tipo all'enum `RuleType` in `backend/app/models/CorrelationRule.py`
2. Aggiungi uno schema Pydantic delle condizioni in `backend/app/schemas/rule.py`
3. Scrivi la funzione evaluator in `backend/app/engine/correlation.py`:

```python
def _evaluate_your_type(rule, event_data, state, rule_id, group_key):
    # Your evaluation logic
    # Return EvaluationResult if fired, None otherwise
```

4. Registrala nel dict `evaluators` dentro `evaluate_rule()`

### Aggiungere un nuovo endpoint API

1. Crea o aggiorna uno schema in `backend/app/schemas/`
2. Scrivi la funzione del controller in `backend/app/controllers/`
3. Aggiungi la route in `backend/app/routes/` con il decorator stack
4. Aggiungi l'hook del frontend in `frontend/src/api/hooks/`
5. Aggiungi il path dell'endpoint a `frontend/src/config.ts`

## Limitazioni

**Nessuno scaling orizzontale.** Il correlation engine usa uno stato in memoria (`CorrelationState`). Eseguire più istanze del backend significa che ogni istanza ha le proprie sliding window e i propri cooldown. Gli eventi verrebbero divisi tra le istanze tramite il consumer group, ma lo stato non sarebbe condiviso. Risolvere il problema richiederebbe di spostare lo stato di correlazione su Redis.

**Nessuna deduplicazione degli eventi.** Se lo stesso evento viene ingerito due volte (ad esempio per un retry di rete), viene memorizzato due volte e può far scattare due volte le regole di correlazione. Un sistema di produzione farebbe l'hash dell'evento raw e controllerebbe i duplicati prima di persistere.

**Nessuna policy di retention dei log.** Gli eventi si accumulano in MongoDB per sempre. Le aggregazioni della dashboard rallenteranno man mano che la collection cresce. Un indice TTL su `timestamp` o un job periodico di pulizia risolverebbe il problema.

**Consumer singolo per la correlazione.** Il `CONSUMER_NAME` è hardcoded a `"engine-1"`. Aggiungere altri consumer richiederebbe una logica di partizionamento per impedire la generazione di alert duplicati.

## Riferimento ai file chiave

Core del backend:
- `backend/app/__init__.py` - Application factory, sequenza di avvio
- `backend/app/config.py` - Tutte le impostazioni via Pydantic
- `backend/app/core/auth.py` - Operazioni su JWT e password
- `backend/app/core/streaming.py` - Pub/sub Redis Streams e SSE
- `backend/app/core/decorators/` - Decorator stack endpoint, S, R
- `backend/app/engine/correlation.py` - Correlation engine e valutazione delle regole
- `backend/app/engine/normalizer.py` - Registry di normalizzazione dei log
- `backend/app/engine/severity.py` - Classificazione della severità
- `backend/app/scenarios/runner.py` - Thread di riproduzione degli scenari

Core del frontend:
- `frontend/src/config.ts` - Endpoint API, query key, route
- `frontend/src/core/lib/api.ts` - Client Axios con interceptor
- `frontend/src/core/stores/auth.store.ts` - Stato di auth con persistenza
- `frontend/src/core/stores/stream.store.ts` - Buffer degli eventi SSE
- `frontend/src/api/hooks/useEventStream.ts` - Gestione della connessione SSE

Infrastruttura:
- `dev.compose.yml` - Setup Docker di sviluppo
- `compose.yml` - Setup Docker di produzione
- `conf/nginx/nginx.conf` - Configurazione principale di Nginx con i rate limit
- `conf/nginx/prod.nginx` - Server block di produzione con passthrough SSE

## Prossimi passi

Ora che hai compreso l'architettura:
1. Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per una guida al codice
2. Prova a modificare il normalizer per aggiungere un nuovo source type e vedere il registry pattern in azione
