# Walkthrough dell'implementazione

Questo documento ripercorre il codice vero e proprio della SIEM dashboard, file per file. Seguiremo come un evento di log entra nel sistema, viene normalizzato, classificato, pubblicato su uno stream, correlato rispetto alle regole e fatto emergere come alert nel browser. Ogni sezione fa riferimento a file e funzioni reali, e spiega perché il codice è strutturato in quel modo.

Se non hai ancora letto [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md), fallo prima. Questo documento presume che tu abbia compreso la disposizione ad alto livello dei componenti e il flusso dei dati.

## Bootstrap dell'applicazione

### Il Factory Pattern

`app/__init__.py` contiene `create_app()`, la application factory di Flask. Questa funzione costruisce l'intero backend in un ordine specifico che conta.

```python
def create_app() -> Flask:
    app = Flask(__name__)
    config = get_settings()
    app.config.from_mapping(config.flask_config())

    # 1. Extensions first (database + cache connections)
    init_extensions(app)

    # 2. Error handlers before anything that might raise
    register_error_handlers(app)

    # 3. Rate limiter needs app context
    init_rate_limiter(app)

    # 4. Blueprints register routes
    register_blueprints(app)

    # 5. Streaming infrastructure
    ensure_consumer_group()

    # 6. Clean up orphaned scenario runs from previous crashes
    cleanup_orphaned_runs()

    # 7. Start correlation engine last (needs everything else ready)
    start_correlation_engine(app)

    return app
```

L'ordinamento è deliberato. Non puoi registrare gli error handler dopo che i blueprint hanno iniziato a lanciare errori. Non puoi avviare il correlation engine prima che esistano le connessioni a MongoDB e Redis. E ti servono i consumer group creati prima che il correlation engine provi a leggerne.

Un errore comune quando si costruiscono app Flask: inizializzare le cose al momento dell'import del modulo invece che dentro la factory. Questo rompe il testing (non puoi creare più istanze dell'app con config diverse) e rompe Gunicorn (il modello a fork significa che ogni worker ha bisogno delle proprie connessioni). Tutto qui avviene dentro `create_app()`, il che significa che ogni worker Gunicorn ottiene il proprio setup.

### Configurazione

`app/config.py` usa il `BaseSettings` di Pydantic per caricare la configurazione dalle variabili d'ambiente con default sensati. La classe settings ha circa 60 valori che coprono tutto, dagli URI di MongoDB al tuning del correlation engine.

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # Core
    SECRET_KEY: str = "change-me-in-production"
    DEBUG: bool = False

    # MongoDB
    MONGODB_HOST: str = "mongodb://mongo:27017/siem"

    # Redis
    REDIS_URL: str = "redis://redis:6379/0"

    # Correlation
    CORRELATION_COOLDOWN_SECONDS: int = 300
    STREAM_READ_COUNT: int = 10
    STREAM_BLOCK_MS: int = 2000

    # Rate limiting
    RATELIMIT_DEFAULT: str = "200 per minute"
    RATELIMIT_AUTH: str = "10 per minute"
```

Il metodo `flask_config()` su Settings traduce questi valori nel formato di config di Flask. Nota il default di `SECRET_KEY`. Quella stringa è intenzionalmente ovvia in modo che nessuno la rilasci per errore in produzione. Pydantic la sovrascriverà automaticamente dall'ambiente o dal file `.env`.

La funzione `get_settings()` usa `@lru_cache` per creare un singleton. Questo significa che le impostazioni vengono parsate una sola volta (con tutta la validazione di Pydantic) e poi riutilizzate. Senza la cache, ogni accesso alle impostazioni rifarebbe il parsing delle variabili d'ambiente e la validazione.

### Extensions

`app/extensions.py` inizializza MongoEngine e Redis come singleton a livello di modulo:

```python
db = MongoEngine()
redis_client: Redis | None = None

def init_extensions(app: Flask) -> None:
    db.init_app(app)

    global redis_client
    redis_client = Redis.from_url(
        app.config["REDIS_URL"],
        decode_responses=True,
    )
```

MongoEngine usa il pattern `init_app` di Flask (inizializzazione lazy). Redis è più semplice: basta creare una connessione dall'URL. Il parametro `decode_responses=True` significa che Redis restituisce stringhe Python invece di bytes. Questo conta ovunque a valle perché serializzerai in JSON i dati di Redis e gli oggetti `bytes` non si serializzano in modo pulito.

## Pipeline di ingestione dei log

Questa è la pipeline centrale. Un evento grezzo entra tramite HTTP POST, viene trasformato, memorizzato, pubblicato, correlato e potenzialmente fa scattare un alert. Seguiamo l'intero percorso.

### Step 1: punto di ingresso HTTP

`app/routes/log_routes.py` definisce l'endpoint di ingestione:

```python
@bp.post("/ingest")
@S(body=LogIngestSchema)
@R(status_code=201)
def ingest_log(body: LogIngestSchema):
    return log_ctrl.ingest_event(body)
```

Tre decorator, ciascuno con un compito specifico. `@S(body=LogIngestSchema)` valida il body della richiesta rispetto a uno schema Pydantic. `@R(status_code=201)` serializza il valore di ritorno e imposta il codice di risposta. La route stessa è solo una riga di delega della logica di business.

Nota che questo endpoint non ha il decorator `@endpoint()`, il che significa che non è autenticato. Questo è intenzionale. In un deployment reale, le sorgenti di log (firewall, server, agent) inviano eventi alla SIEM senza credenziali utente. Si autenticherebbero tramite API key o controlli a livello di rete, ma per questo progetto l'endpoint di ingestione è aperto.

### Step 2: validazione dello schema

`app/schemas/log_schemas.py` definisce l'aspetto di una richiesta di ingestione valida:

```python
class LogIngestSchema(BaseModel):
    source_type: SourceType
    event_type: str
    timestamp: datetime | None = None
    source_ip: str | None = None
    source_port: int | None = None
    dest_ip: str | None = None
    dest_port: int | None = None
    protocol: str | None = None
    username: str | None = None
    hostname: str | None = None
    message: str | None = None
    # ... additional optional fields per source type
```

`SourceType` è un enum di stringhe: `firewall`, `ids`, `auth`, `endpoint`, `dns`, `proxy`, `generic`. Pydantic valida automaticamente il valore dell'enum. Se fai POST di `"source_type": "foo"`, ottieni un 422 con un messaggio di errore chiaro prima che venga eseguita qualsiasi logica di business.

La maggior parte dei campi è opzionale perché tipi di sorgente diversi forniscono dati diversi. Un log di firewall ha `source_ip` e `dest_ip` ma nessun `username`. Un log di auth ha `username` ma potrebbe non avere `dest_ip`. Il normalizer si occupa di dare senso a qualsiasi campo sia presente.

### Step 3: logica del controller

`app/controllers/log_ctrl.py` orchestra l'ingestione:

```python
def ingest_event(body: LogIngestSchema) -> dict:
    raw = body.model_dump(exclude_none=True)

    # Normalize: common fields + source-specific extraction
    normalized_data = normalize(raw)

    # Classify severity based on event type and content
    severity = classify_severity(raw)

    # Build and save the document
    event = LogEvent(
        source_type=raw["source_type"],
        event_type=raw["event_type"],
        severity=severity,
        timestamp=raw.get("timestamp", datetime.utcnow()),
        source_ip=raw.get("source_ip"),
        dest_ip=raw.get("dest_ip"),
        # ... remaining fields
        raw=raw,
        normalized=normalized_data,
    )
    event.save()

    # Publish to Redis Stream for real-time consumers
    publish_log_event(event)

    return serialize_document(event)
```

Tre operazioni avvengono in sequenza: normalizzare, classificare, persistere. Poi l'evento viene pubblicato sul Redis Stream in modo asincrono. Il dict `raw` viene conservato sul documento accanto al dict `normalized`. Questo doppio storage è importante per la forensics, come trattato in [01-CONCEPTS.it.md](./01-CONCEPTS.it.md).

Il controller chiama `model_dump(exclude_none=True)` sullo schema Pydantic per eliminare i campi che non sono stati forniti. Questo evita di memorizzare un mucchio di valori `null` in MongoDB e mantiene pulito il payload raw.

### Step 4: normalizzazione

`app/engine/normalizer.py` trasforma gli eventi grezzi in una struttura coerente. Il registry pattern qui merita di essere studiato:

```python
NORMALIZERS: dict[str, Callable] = {}

def _register(source_type: SourceType):
    """Decorator to register a normalizer function for a source type."""
    def decorator(fn: Callable) -> Callable:
        NORMALIZERS[source_type.value] = fn
        return fn
    return decorator
```

Ogni tipo di sorgente ottiene la propria funzione di normalizzazione:

```python
@_register(SourceType.FIREWALL)
def _normalize_firewall(data: dict) -> dict:
    return {
        "action": data.get("action"),
        "protocol": data.get("protocol"),
        "bytes_sent": data.get("bytes_sent", 0),
        "bytes_received": data.get("bytes_received", 0),
    }

@_register(SourceType.AUTH)
def _normalize_auth(data: dict) -> dict:
    return {
        "auth_method": data.get("auth_method"),
        "result": data.get("result"),
        "failure_reason": data.get("failure_reason"),
    }

@_register(SourceType.DNS)
def _normalize_dns(data: dict) -> dict:
    return {
        "query": data.get("query"),
        "query_type": data.get("query_type"),
        "response_code": data.get("response_code"),
    }
```

La funzione di dispatch lega il tutto:

```python
def normalize(data: dict) -> dict:
    source_type = data.get("source_type", "generic")
    normalizer = NORMALIZERS.get(source_type, NORMALIZERS["generic"])
    return normalizer(data)
```

Questo pattern rende banale aggiungere nuove sorgenti di log. Scrivi una funzione, appiccica il decorator `@_register` e hai finito. Nessuno switch, nessuna catena if/elif, nessuna modifica al codice esistente. Il dispatcher cerca la funzione nel dict e la chiama.

Una cosa da notare: se un tipo di sorgente non è riconosciuto, si ripiega sul normalizer `generic`, che si limita a prendere il campo `message`. È una scelta di design. Non fallire rumorosamente sui tipi di sorgente sconosciuti significa che il sistema continua a ingerire anche se incontra qualcosa di inatteso. In produzione vorresti loggare un warning, ma scartare gli eventi è peggio che memorizzarli con una normalizzazione minima.

### Step 5: classificazione della severity

`app/engine/severity.py` assegna un livello di severity a ogni evento. Usa un approccio a due livelli:

```python
HIGH_SEVERITY_EVENT_TYPES = frozenset({
    "privilege_escalation", "data_exfiltration",
    "c2_communication", "reverse_shell",
})

MEDIUM_SEVERITY_EVENT_TYPES = frozenset({
    "login_failure", "port_scan",
    "firewall_deny", "ids_alert",
})

CRITICAL_PATTERNS = [
    re.compile(r"privilege.?escalat", re.IGNORECASE),
    re.compile(r"ransomware", re.IGNORECASE),
    re.compile(r"c2.?beacon", re.IGNORECASE),
    re.compile(r"data.?exfil", re.IGNORECASE),
]

HIGH_PATTERNS = [
    re.compile(r"brute.?force", re.IGNORECASE),
    re.compile(r"lateral.?movement", re.IGNORECASE),
    re.compile(r"reverse.?shell", re.IGNORECASE),
    re.compile(r"credential.?dump", re.IGNORECASE),
]
```

La funzione `classify_severity()` controlla prima il tipo di evento (veloce lookup O(1) su frozen set), poi passa al matching con regex sui campi testuali concatenati:

```python
def classify_severity(data: dict) -> str:
    event_type = data.get("event_type", "")

    # Fast path: check event type directly
    if event_type in HIGH_SEVERITY_EVENT_TYPES:
        return "high"
    if event_type in MEDIUM_SEVERITY_EVENT_TYPES:
        return "medium"

    # Slow path: regex against content
    searchable = _build_searchable_text(data)

    for pattern in CRITICAL_PATTERNS:
        if pattern.search(searchable):
            return "critical"
    for pattern in HIGH_PATTERNS:
        if pattern.search(searchable):
            return "high"
    # ... medium and low patterns

    return "info"  # Default
```

L'helper `_build_searchable_text()` concatena `event_type`, `message` e i campi normalizzati rilevanti (come `signature_name` dagli eventi IDS o `command_line` dagli eventi endpoint) in un'unica stringa. Questo significa che una command line PowerShell contenente "lateral movement" viene classificata con severity high anche se l'event_type è solo "process_execution".

I frozen set sono importanti. L'operatore `in` di Python su un frozenset è O(1) nel caso medio. I pattern regex sono compilati una sola volta al caricamento del modulo, non a ogni chiamata di classificazione. Sono piccole ottimizzazioni, ma quando classifichi migliaia di eventi al secondo si sommano.

Il ritorno di default `"info"` è deliberato. La maggior parte degli eventi in un ambiente di produzione è di routine. Classificare gli sconosciuti con severity high creerebbe alert fatigue e addestrerebbe gli analisti a ignorare gli alert.

### Step 6: pubblicazione sul Redis Stream

`app/core/streaming.py` gestisce l'integrazione con Redis Streams:

```python
def publish_log_event(event: LogEvent) -> str | None:
    """Publish a log event to the Redis log stream."""
    try:
        data = {
            "id": str(event.id),
            "source_type": event.source_type,
            "event_type": event.event_type,
            "severity": event.severity,
            "source_ip": event.source_ip or "",
            "dest_ip": event.dest_ip or "",
            "username": event.username or "",
            "hostname": event.hostname or "",
            "timestamp": event.timestamp.isoformat(),
        }
        stream_id = redis_client.xadd(
            STREAM_LOGS,
            data,
            maxlen=STREAM_MAXLEN,
            approximate=True,
        )
        return stream_id
    except Exception:
        logger.exception("Failed to publish log event")
        return None
```

Diverse cose da notare qui:

Il parametro `maxlen` con `approximate=True` impedisce allo stream di crescere senza limiti. Redis non taglia esattamente a 10000 entry. Invece taglia in blocco quando lo stream supera il limite, il che è più efficiente del trimming esatto. Lo stream potrebbe contenere temporaneamente 10500 entry prima che Redis lo riduca.

I valori `None` vengono convertiti in stringhe vuote perché i campi dei Redis Stream non possono memorizzare `None`. I consumer a valle devono trattare le stringhe vuote come valori mancanti.

Il `try/except` con `return None` significa che un guasto di Redis non manda in crash la pipeline di ingestione. A questo punto l'evento è già persistito in MongoDB. Perdere il publish sullo stream è accettabile. L'evento non farà scattare la correlazione in tempo reale né comparirà nel feed SSE finché non verrà rielaborato, ma non andrà perso.

### Step 7: setup dei consumer group

Prima che il correlation engine possa leggere dallo stream, i consumer group devono esistere:

```python
def ensure_consumer_group() -> None:
    """Create consumer groups if they don't exist."""
    with contextlib.suppress(Exception):
        redis_client.xgroup_create(
            STREAM_LOGS,
            CONSUMER_GROUP,
            id="0",
            mkstream=True,
        )
    with contextlib.suppress(Exception):
        redis_client.xgroup_create(
            STREAM_ALERTS,
            CONSUMER_GROUP,
            id="0",
            mkstream=True,
        )
```

Il `contextlib.suppress(Exception)` gestisce il caso in cui il consumer group esista già (Redis lancia `BUSYGROUP`). Questo rende la funzione idempotente, sicura da chiamare a ogni avvio. Il parametro `mkstream=True` crea lo stream stesso se non esiste. L'`id="0"` significa che il consumer group inizia a leggere dall'inizio dello stream.

## Correlation Engine

Questa è la parte più complessa del backend. L'engine gira in un thread in background, consumando eventi da Redis e valutandoli rispetto alle regole di correlazione.

### Ciclo di vita dell'engine

`app/engine/correlation.py` definisce la classe `CorrelationEngine`:

```python
class CorrelationEngine:
    def __init__(self, app: Flask):
        self.app = app
        self.state = CorrelationState()
        self._thread: threading.Thread | None = None
        self._stop_event = threading.Event()

    def start(self) -> None:
        self._thread = threading.Thread(
            target=self._run,
            daemon=True,
            name="correlation-engine",
        )
        self._thread.start()

    def stop(self) -> None:
        self._stop_event.set()
        if self._thread:
            self._thread.join(timeout=5)
```

L'engine parte come thread daemon. I thread daemon vengono terminati quando il processo principale esce, quindi non c'è il rischio che thread dell'engine orfani mantengano vivo il processo. Lo `_stop_event` fornisce un percorso di shutdown pulito: il loop di `_run()` lo controlla a ogni iterazione.

### Il loop principale

```python
def _run(self) -> None:
    with self.app.app_context():
        while not self._stop_event.is_set():
            try:
                messages = redis_client.xreadgroup(
                    groupname=CONSUMER_GROUP,
                    consumername=CONSUMER_NAME,
                    streams={STREAM_LOGS: ">"},
                    count=STREAM_READ_COUNT,
                    block=STREAM_BLOCK_MS,
                )
                if messages:
                    for stream_name, entries in messages:
                        for entry_id, data in entries:
                            self._process_event(data, entry_id)
                            redis_client.xack(
                                STREAM_LOGS,
                                CONSUMER_GROUP,
                                entry_id,
                            )
            except Exception:
                logger.exception("Correlation engine error")
                time.sleep(1)
```

Analizziamolo:

`self.app.app_context()` fa il push di un application context di Flask. Senza questo, il thread non può accedere a MongoEngine (che è legato all'app context di Flask) né alla config. È un classico tranello di Flask con i thread in background.

`xreadgroup` con `">"` legge solo i messaggi nuovi, non ancora consegnati. Il `block=STREAM_BLOCK_MS` (default 2000ms) rende questa una chiamata bloccante. Il thread dorme fino a 2 secondi in attesa di nuovi eventi invece di fare busy-polling. Quando arrivano eventi, si sveglia immediatamente.

`count=STREAM_READ_COUNT` (default 10) limita quanti eventi vengono letti per chiamata. Questo controlla la dimensione del batch. Leggere 10 eventi alla volta significa che l'engine elabora in piccoli batch, il che mantiene bassa la latenza per evento ammortizzando l'overhead della chiamata a Redis.

Dopo aver elaborato ogni evento, `xack` lo conferma. Questo dice a Redis che l'evento è stato gestito con successo e non necessita di riconsegna. Se l'engine va in crash prima di confermare, Redis riconsegnerà l'evento al prossimo avvio. Questo dà semantica di elaborazione at-least-once.

L'`except Exception` con uno sleep di 1 secondo previene un loop di errori serrato. Se Redis va giù, l'engine logga l'errore e attende un secondo prima di riprovare invece di martellare la connessione morta.

### Caricamento e caching delle regole

```python
def _get_rules(self) -> list:
    """Load enabled rules with caching."""
    now = time.time()
    if now - self._rules_cache_time > RULES_CACHE_TTL:
        self._rules_cache = list(
            CorrelationRule.objects(enabled=True)
        )
        self._rules_cache_time = now
    return self._rules_cache
```

Le regole sono messe in cache per 30 secondi (`RULES_CACHE_TTL`). Senza caching, ogni evento farebbe scattare una query MongoDB per tutte le regole abilitate. Con 100 eventi al secondo e 20 regole, sono 100 query al database al secondo solo per caricare le regole. La cache lo riduce a una query ogni 30 secondi.

Il compromesso: quando crei o modifichi una regola, ci vogliono fino a 30 secondi perché abbia effetto. Per una SIEM va bene. Le regole di detection non sono time-critical da distribuire. Se ti servissero aggiornamenti delle regole sotto il secondo, useresti un canale pub/sub di Redis per notificare le modifiche all'engine.

### Elaborazione degli eventi

```python
def _process_event(self, data: dict, entry_id: str) -> None:
    rules = self._get_rules()
    for rule in rules:
        try:
            evaluator = EVALUATORS.get(rule.rule_type)
            if evaluator and self._event_matches_rule(data, rule):
                fired = evaluator(rule, data, self.state)
                if fired:
                    self._create_alert(rule, data, fired)
        except Exception:
            logger.exception(f"Error evaluating rule {rule.id}")
```

Per ogni evento, l'engine itera su tutte le regole abilitate. Per prima cosa controlla se l'evento è rilevante per la regola (l'`event_type` corrisponde? il `source_type` corrisponde?). Poi chiama l'evaluator appropriato in base al tipo di regola.

Il dict `EVALUATORS` mappa i tipi di regola alle funzioni:

```python
EVALUATORS = {
    "threshold": _evaluate_threshold,
    "sequence": _evaluate_sequence,
    "aggregation": _evaluate_aggregation,
}
```

È lo stesso registry pattern usato nel normalizer. Aggiungere un nuovo tipo di regola significa scrivere una funzione evaluator e aggiungere una entry a questo dict.

### Valutazione threshold

L'evaluator threshold conta gli eventi corrispondenti per group key entro una finestra temporale:

```python
def _evaluate_threshold(
    rule: CorrelationRule,
    data: dict,
    state: CorrelationState,
) -> list[str] | None:
    conditions = rule.conditions
    group_key = data.get(conditions.get("group_by", "source_ip"), "")
    window = conditions.get("window_seconds", 300)
    threshold = conditions.get("threshold", 10)

    # Add event to sliding window
    state.add_event(rule.id, group_key, data, window)

    # Check if threshold exceeded
    count = state.get_count(rule.id, group_key)
    if count >= threshold:
        if not state.in_cooldown(rule.id, group_key):
            state.set_cooldown(rule.id, group_key)
            return state.get_event_ids(rule.id, group_key)

    return None
```

La sliding window è mantenuta da `CorrelationState`. Quando aggiungiamo un evento, le entry scadute (più vecchie di `window_seconds`) vengono eliminate. Il conteggio riflette solo gli eventi nella finestra corrente.

Il controllo del cooldown è critico. Senza di esso, una volta superata la soglia, ogni evento successivo farebbe scattare di nuovo la regola. Un attacco brute force di 10 minuti con soglia 10 e finestra 300 genererebbe centinaia di alert senza cooldown. Con il cooldown di default di 300 secondi, ne genera al massimo 2 (uno alla soglia, potenzialmente un altro dopo la scadenza del cooldown se l'attacco continua).

### Valutazione sequence

Le regole sequence sono più complesse. Tracciano insiemi ordinati di tipi di evento:

```python
def _evaluate_sequence(
    rule: CorrelationRule,
    data: dict,
    state: CorrelationState,
) -> list[str] | None:
    conditions = rule.conditions
    steps = conditions.get("steps", [])
    group_key = data.get(conditions.get("group_by", "source_ip"), "")
    window = conditions.get("window_seconds", 300)

    event_type = data.get("event_type", "")

    # Find which step this event matches
    for i, step in enumerate(steps):
        if event_type == step.get("event_type"):
            state.add_sequence_event(
                rule.id, group_key, i, data, window
            )
            break

    # Check if all steps are satisfied in order
    if state.sequence_complete(rule.id, group_key, steps):
        if not state.in_cooldown(rule.id, group_key):
            state.set_cooldown(rule.id, group_key)
            return state.get_sequence_event_ids(rule.id, group_key)

    return None
```

Una regola sequence con step `[login_failure (count >= 5), login_success]` significa: entro la finestra, servono almeno 5 eventi `login_failure` dalla stessa group key, seguiti da almeno un `login_success`. Il metodo `sequence_complete()` controlla sia l'ordinamento sia i requisiti di conteggio per step.

Questo modella pattern di attacco reali. Un brute force seguito da compromissione non è semplicemente "sono successe entrambe le cose". È "prima ci sono stati i fallimenti, poi il successo". Una regola sequence rispetta quell'ordinamento.

### Valutazione aggregation

Le regole aggregation contano valori distinti:

```python
def _evaluate_aggregation(
    rule: CorrelationRule,
    data: dict,
    state: CorrelationState,
) -> list[str] | None:
    conditions = rule.conditions
    group_key = data.get(conditions.get("group_by", "source_ip"), "")
    agg_field = conditions.get("aggregation_field", "dest_ip")
    window = conditions.get("window_seconds", 60)
    threshold = conditions.get("threshold", 20)

    agg_value = data.get(agg_field, "")
    state.add_aggregation_event(
        rule.id, group_key, agg_value, data, window
    )

    distinct_count = state.get_distinct_count(rule.id, group_key)
    if distinct_count >= threshold:
        if not state.in_cooldown(rule.id, group_key):
            state.set_cooldown(rule.id, group_key)
            return state.get_aggregation_event_ids(rule.id, group_key)

    return None
```

Il caso d'uso classico: il rilevamento del port scanning. Un IP sorgente si connette a 20+ IP di destinazione distinti in un minuto. Lo stato di aggregazione traccia quali IP di destinazione sono stati visti (usando un set per controlli di appartenenza O(1)) e scatta quando il set supera la soglia.

### Stato thread-safe

`CorrelationState` usa lock di threading perché è accessibile dal thread del correlation engine mentre il thread principale potrebbe leggere lo stato per debug o testing:

```python
class CorrelationState:
    def __init__(self):
        self._lock = threading.Lock()
        self._windows: dict[str, dict[str, list]] = {}
        self._cooldowns: dict[str, dict[str, float]] = {}

    def add_event(self, rule_id, group_key, data, window_seconds):
        with self._lock:
            key = f"{rule_id}:{group_key}"
            now = time.time()

            if key not in self._windows:
                self._windows[key] = []

            # Prune expired
            cutoff = now - window_seconds
            self._windows[key] = [
                e for e in self._windows[key]
                if e["timestamp"] > cutoff
            ]

            # Add new
            self._windows[key].append({
                "timestamp": now,
                "event_id": data.get("id"),
                "data": data,
            })
```

Il lock è a grana grossa: un lock per l'intero oggetto di stato. È più semplice del locking per chiave e abbastanza veloce per un consumo single-threaded. Se dovessi scalare a più thread del correlation engine, vorresti un locking più fine o una struttura dati concorrente lock-free.

La sliding window elimina a ogni inserimento. Le vecchie entry oltre `window_seconds` vengono filtrate. Questo mantiene la memoria limitata. Senza l'eliminazione, lo stato crescerebbe linearmente con il volume degli eventi e non si ridurrebbe mai.

### Creazione degli alert

Quando una regola scatta, l'engine crea un documento Alert:

```python
def _create_alert(self, rule, data, event_ids):
    with self.app.app_context():
        alert = Alert.create_from_rule(
            rule=rule,
            matched_event_ids=event_ids,
            trigger_event=data,
        )

        # Publish to alert stream
        publish_alert(alert)
```

`Alert.create_from_rule()` è un class method sul modello Alert che costruisce e salva il documento, copiando `mitre_tactic`, `mitre_technique`, severity e descrizione della regola. Il campo `matched_event_ids` memorizza i riferimenti ai documenti LogEvent che hanno fatto scattare la regola. Questo permette agli analisti di scendere in profondità da un alert agli eventi grezzi.

Dopo il salvataggio, l'alert viene pubblicato sul Redis Stream `siem:alerts`. La connessione SSE del frontend lo raccoglie e mostra una notifica nel browser.

## Sistema di autenticazione

### Hashing delle password

`app/core/auth.py` implementa il livello di auth:

```python
from pwdlib import PasswordHash
from pwdlib.hashers.argon2 import Argon2Hasher

_hasher = PasswordHash((Argon2Hasher(),))

DUMMY_HASH = _hasher.hash("dummy-password-for-timing")

def hash_password(password: str) -> str:
    return _hasher.hash(password)

def verify_password_timing_safe(password: str, hash: str | None) -> bool:
    """Verify password with constant-time behavior."""
    if hash is None:
        # User doesn't exist - still do the work
        _hasher.verify(password, DUMMY_HASH)
        return False
    return _hasher.verify(password, hash)
```

Il `DUMMY_HASH` viene calcolato una sola volta al caricamento del modulo. Quando `hash` è `None` (l'utente non esiste), la funzione esegue comunque una verifica Argon2id completa rispetto a questo dummy. Il risultato viene scartato, ma il costo in tempo è lo stesso. Questo previene la username enumeration tramite timing side channel.

Perché `pwdlib` invece di `passlib`? `pwdlib` è una libreria più recente con un'API più pulita e migliori default per Argon2id. I parametri di default (memory cost, time cost, parallelism) corrispondono alle attuali raccomandazioni OWASP senza tuning manuale.

### Token JWT

```python
def create_access_token(user: User) -> str:
    config = get_settings()
    now = datetime.utcnow()
    payload = {
        "sub": str(user.id),
        "username": user.username,
        "role": user.role,
        "iat": now,
        "exp": now + timedelta(hours=config.JWT_EXPIRY_HOURS),
    }
    return jwt.encode(payload, config.SECRET_KEY, algorithm="HS256")

def decode_access_token(token: str) -> dict:
    config = get_settings()
    return jwt.decode(token, config.SECRET_KEY, algorithms=["HS256"])
```

Il JWT contiene l'ID dell'utente, lo username e il ruolo. Questo significa che il decorator `@endpoint()` può controllare l'autorizzazione senza una query al database per la maggior parte delle richieste. Il ruolo è incorporato nel token.

Il compromesso: non puoi revocare un JWT prima della sua scadenza. Se cambi il ruolo di un utente o ne disattivi l'account, il vecchio token funziona ancora fino alla scadenza. Il codice mitiga questo caricando l'utente da MongoDB a ogni richiesta autenticata (nel decorator `@endpoint`) e controllando `is_active`. Questo significa che un utente disattivato viene bloccato alla richiesta successiva, non solo alla scadenza del token.

### Il decorator Endpoint

`app/core/decorators/endpoint.py` è il punto di enforcement dell'auth:

```python
def endpoint(roles: str | tuple | None = None):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            token = _extract_token(request)
            if token is None:
                raise AuthenticationError("Missing token")

            payload = decode_access_token(token)
            user = User.objects(id=payload["sub"]).first()

            if user is None or not user.is_active:
                raise AuthenticationError("Invalid or inactive user")

            if roles and user.role not in (
                roles if isinstance(roles, tuple) else (roles,)
            ):
                raise ForbiddenError("Insufficient permissions")

            g.current_user = user
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

`_extract_token()` estrae il JWT dall'header `Authorization: Bearer <token>`. Per gli endpoint SSE controlla anche i query parameter perché l'API `EventSource` del browser non supporta header personalizzati. Questo è annotato nel documento di architettura come un compromesso noto.

Il decorator carica l'utente completo da MongoDB a ogni richiesta. Sembra costoso, ma è necessario. Il JWT potrebbe essere obsoleto (ruolo cambiato, account disattivato). Il controllo sul database garantisce che lo stato corrente sia usato per le decisioni di autorizzazione.

`g.current_user` memorizza l'utente nell'oggetto `g` di Flask, con scope di richiesta. Qualsiasi funzione a valle può accedere all'utente autenticato tramite `g.current_user` senza passarlo come parametro.

## Lo stack di decorator

Il progetto usa tre decorator componibili che formano una pipeline: `@endpoint`, `@S` e `@R`. Capire come si compongono è importante.

### @S (validazione dello schema)

`app/core/decorators/schema.py`:

```python
def S(body: type[BaseModel] | None = None,
      query: type[BaseModel] | None = None):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            if body:
                try:
                    parsed = body.model_validate(request.get_json())
                except ValidationError as e:
                    raise AppValidationError(str(e))
                kwargs["body"] = parsed

            if query:
                try:
                    parsed = query.model_validate(request.args.to_dict())
                except ValidationError as e:
                    raise AppValidationError(str(e))
                kwargs["query"] = parsed

            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

`@S` intercetta la richiesta prima che la funzione della route venga eseguita. Parsa il body della richiesta e/o i query parameter rispetto agli schemi Pydantic. I dati validati vengono iniettati come argomenti keyword (`body=`, `query=`). Se la validazione fallisce, solleva un `AppValidationError` che l'error handler converte in una risposta 422.

### @R (serializzazione della risposta)

`app/core/decorators/response.py`:

```python
def R(status_code: int = 200):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            result = fn(*args, **kwargs)
            if isinstance(result, Response):
                return result
            return jsonify(result), status_code
        return wrapper
    return decorator
```

`@R` avvolge il valore di ritorno in `jsonify()` e imposta lo status code. Se la funzione restituisce direttamente una `Response` di Flask (come per lo streaming), passa invariata.

### Ordine di composizione

Quando vedi una route come:

```python
@bp.post("/")
@endpoint(roles=ADMIN)
@S(body=RuleCreateSchema)
@R(status_code=201)
def create_rule(body: RuleCreateSchema):
    return rule_ctrl.create(body)
```

I decorator vengono eseguiti dal basso verso l'alto (prima il più esterno):
1. `@endpoint` controlla l'auth e imposta `g.current_user`
2. `@S` valida il body della richiesta
3. Viene eseguita la funzione della route
4. `@R` serializza la risposta

Se l'auth fallisce, la validazione dello schema non viene mai eseguita. Se la validazione fallisce, la funzione della route non viene mai eseguita. Ogni decorator o passa il controllo al successivo o va in corto circuito con un errore.

## Streaming e SSE

### Generatore SSE

`app/core/streaming.py` implementa il generatore di Server-Sent Events:

```python
def sse_log_stream(last_id: str = "$"):
    """Generator that yields SSE events from the log stream."""
    while True:
        try:
            results = redis_client.xread(
                streams={STREAM_LOGS: last_id},
                count=10,
                block=STREAM_BLOCK_MS,
            )
            if results:
                for stream_name, entries in results:
                    for entry_id, data in entries:
                        last_id = entry_id
                        yield f"data: {json.dumps(data)}\n\n"
            else:
                # No events within block timeout - send keepalive
                yield ": keepalive\n\n"
        except GeneratorExit:
            break
        except Exception:
            logger.exception("SSE stream error")
            yield ": error\n\n"
            time.sleep(1)
```

Dettagli chiave:

Per l'SSE si usa `xread` (non `xreadgroup`). I consumer group forniscono una consegna exactly-once, appropriata per il correlation engine. Ma i client SSE sono browser effimeri che vanno e vengono. Se un browser si disconnette e si riconnette, dovrebbe ricevere gli eventi correnti, non vecchi eventi rigiocati. `xread` con `$` parte dall'evento più recente.

Il commento keepalive (`: keepalive\n\n`) è un commento SSE (inizia con `:`). Il browser lo ignora, ma mantiene viva la connessione HTTP. Senza di esso, proxy e load balancer farebbero andare in timeout la connessione. Il `proxy_read_timeout` di default di Nginx è 60 secondi. La config Nginx di questo progetto lo imposta a 3600 secondi per gli endpoint SSE, ma i keepalive servono comunque come rete di sicurezza.

`GeneratorExit` scatta quando il client si disconnette. È il modo di Python per dire al generatore di ripulire. Senza catturarlo, il generatore loggherebbe un errore spurio a ogni disconnessione del client.

### Route SSE

```python
@bp.get("/stream")
@endpoint()
def stream_logs():
    token = request.args.get("token")  # SSE can't use headers
    return Response(
        sse_log_stream(),
        mimetype="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Nginx hint
        },
    )
```

L'header `X-Accel-Buffering: no` dice a Nginx di disabilitare il buffering della risposta per questo endpoint. Combinato con `proxy_buffering off` nella config Nginx, questo garantisce che gli eventi fluiscano subito verso il browser invece di essere bufferizzati.

## Sistema di playback degli scenari

### Playbook YAML

Il progetto include playbook di scenari di attacco in `app/scenarios/playbooks/`. Ogni file YAML descrive una sequenza di eventi che simulano un attacco reale:

```yaml
# brute_force_lateral.yml
name: "Brute Force with Lateral Movement"
description: "SSH brute force followed by lateral movement"
mitre_techniques:
  - T1110.001
  - T1021.004
steps:
  - delay: 0
    event:
      source_type: auth
      event_type: login_failure
      source_ip: "203.0.113.50"
      username: root
      hostname: web-01
      auth_method: ssh_password
      failure_reason: invalid_password
  # ... 19 more login_failure events with varying usernames
  - delay: 2
    event:
      source_type: auth
      event_type: login_success
      source_ip: "203.0.113.50"
      username: admin
      hostname: web-01
      auth_method: ssh_password
  - delay: 5
    event:
      source_type: auth
      event_type: login_success
      source_ip: "10.0.1.10"  # Internal IP now
      username: admin
      hostname: db-01
      auth_method: ssh_key
```

Il campo `delay` (in secondi) distanzia gli eventi in modo realistico. Un vero brute force non avviene in tempo zero. I delay permettono alle sliding window del correlation engine di funzionare correttamente durante il playback.

### Parser dei playbook

`app/scenarios/playbook.py` carica e valida questi file YAML:

```python
def load_playbook(name: str) -> dict:
    playbook_dir = Path(__file__).parent / "playbooks"
    path = playbook_dir / f"{name}.yml"

    if not path.exists():
        raise NotFoundError(f"Playbook '{name}' not found")

    with open(path) as f:
        playbook = yaml.safe_load(f)

    # Validate structure
    if "steps" not in playbook:
        raise ValidationError("Playbook missing 'steps'")

    return playbook
```

Si usa `yaml.safe_load` invece di `yaml.load`. È una considerazione di sicurezza. `yaml.load` può eseguire codice Python arbitrario incorporato nello YAML (tramite i tag `!!python/object`). `safe_load` li rifiuta. In un tool di sicurezza, questo conta più che in molti altri posti.

### Runner threaded

`app/scenarios/runner.py` esegue i playbook in thread in background:

```python
class ScenarioRunner:
    _active_runs: dict[str, threading.Thread] = {}

    @classmethod
    def start(cls, playbook_name: str, run_id: str, app: Flask) -> None:
        thread = threading.Thread(
            target=cls._execute,
            args=(playbook_name, run_id, app),
            daemon=True,
        )
        cls._active_runs[run_id] = thread
        thread.start()

    @classmethod
    def _execute(cls, playbook_name, run_id, app):
        with app.app_context():
            playbook = load_playbook(playbook_name)
            run = ScenarioRun.objects(id=run_id).first()

            try:
                run.update(status="running")
                for step in playbook["steps"]:
                    if run.reload().status == "cancelled":
                        break

                    delay = step.get("delay", 0)
                    if delay > 0:
                        time.sleep(delay)

                    # Ingest the event through the normal pipeline
                    event_data = step["event"]
                    event_data["scenario_run_id"] = run_id
                    _ingest_scenario_event(event_data)

                status = "cancelled" if run.reload().status == "cancelled" else "completed"
                run.update(status=status)
            except Exception as e:
                run.update(status="failed", error=str(e))
            finally:
                cls._active_runs.pop(run_id, None)
```

Gli eventi iniettati dagli scenari passano per la stessa pipeline di ingestione degli eventi reali. Vengono normalizzati, classificati, persistiti, pubblicati sullo stream e correlati. L'unica differenza è il campo `scenario_run_id`, che li tagga per la pulizia e il filtraggio successivi.

Il controllo `run.reload().status == "cancelled"` a ogni step permette di fermare uno scenario in esecuzione. Quando un utente clicca "Cancel" nella UI, lo status viene aggiornato in MongoDB e la successiva iterazione del loop lo rileva.

Il dict `_active_runs` tiene traccia dei thread in esecuzione. La funzione `cleanup_orphaned_runs()` chiamata all'avvio marca come "failed" qualsiasi run con status "running", dato che sono stati interrotti da un riavvio del processo.

## Integrazione col frontend

### Hook SSE

`src/api/hooks/useEventStream.ts` connette il browser agli endpoint SSE:

```typescript
export function useEventStream(
  endpoint: 'logs' | 'alerts',
  enabled: boolean = true
) {
  const { token } = useAuthStore();
  const addEvent = useStreamStore((s) => s.addEvent);
  const addAlert = useStreamStore((s) => s.addAlert);

  useEffect(() => {
    if (!enabled || !token) return;

    const url = `${API_BASE}/${endpoint}/stream?token=${token}`;
    const source = new EventSource(url);

    source.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (endpoint === 'logs') {
        addEvent(data);
      } else {
        addAlert(data);
      }
    };

    source.onerror = () => {
      source.close();
      // Reconnect after delay
      setTimeout(() => {
        // React will re-run the effect
      }, 3000);
    };

    return () => source.close();
  }, [token, endpoint, enabled]);
}
```

Il token viene passato come query parameter perché `EventSource` non supporta header personalizzati. È una limitazione ben nota dell'API SSE. Il token viene comunque trasmesso su HTTPS (assumendo un deployment in produzione), quindi è cifrato in transito.

La logica di riconnessione è semplice: chiudere la connessione e lasciare che il ciclo cleanup/re-run di `useEffect` di React gestisca la riconnessione. In produzione vorresti un exponential backoff, ma per un progetto didattico questo è sufficiente.

### Store Zustand

`src/core/stores/stream.store.ts` gestisce lo stato degli eventi in tempo reale:

```typescript
interface StreamState {
  events: LogEvent[];
  alerts: Alert[];
  addEvent: (event: LogEvent) => void;
  addAlert: (alert: Alert) => void;
  clearEvents: () => void;
}

export const useStreamStore = create<StreamState>((set) => ({
  events: [],
  alerts: [],
  addEvent: (event) =>
    set((state) => ({
      events: [...state.events, event].slice(-100),
    })),
  addAlert: (alert) =>
    set((state) => ({
      alerts: [...state.alerts, alert].slice(-50),
    })),
  clearEvents: () => set({ events: [], alerts: [] }),
}));
```

Lo `.slice(-100)` limita il buffer di eventi in memoria a 100 entry. Senza di esso, un flusso di eventi sostenuto farebbe crescere la memoria del browser senza limiti. La UI mostra gli ultimi 100 eventi in tempo reale, e gli utenti possono interrogare l'intera cronologia tramite l'API di ricerca.

### Auth store

`src/core/stores/auth.store.ts` gestisce lo stato di autenticazione con persistenza:

```typescript
export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      login: (token, user) => set({ token, user }),
      logout: () => set({ token: null, user: null }),
    }),
    {
      name: 'siem-auth',
    }
  )
);
```

Il middleware `persist` di Zustand salva il token e l'utente in `localStorage`. Questo significa che aggiornare la pagina non ti disconnette. L'interceptor di Axios in `src/core/lib/api.ts` legge il token da questo store e lo allega a ogni richiesta API:

```typescript
api.interceptors.request.use((config) => {
  const { token } = useAuthStore.getState();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### Hook TanStack Query

`src/api/hooks/` contiene hook personalizzati per ogni risorsa API. Il pattern è coerente:

```typescript
// useAlerts.ts
export function useAlerts(params?: AlertQueryParams) {
  return useQuery({
    queryKey: [QUERY_KEYS.ALERTS, params],
    queryFn: () => api.get('/alerts', { params }).then((r) => r.data),
  });
}

export function useUpdateAlertStatus() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, status }: { id: string; status: string }) =>
      api.patch(`/alerts/${id}/status`, { status }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: [QUERY_KEYS.ALERTS] });
    },
  });
}
```

TanStack Query gestisce caching, deduplicazione e refetching in background. Quando lo status di un alert viene aggiornato, `invalidateQueries` forza un refresh della lista degli alert. Questo mantiene la UI coerente senza gestione manuale dello stato.

## Aggregazioni della dashboard

La pagina dashboard mostra visualizzazioni riepilogative: timeline, ripartizione per severity e top source. Sono alimentate da aggregation pipeline di MongoDB.

### Aggregazione della timeline

`app/controllers/dashboard_ctrl.py`:

```python
def get_timeline(hours: int = 24, bucket_minutes: int = 15):
    cutoff = datetime.utcnow() - timedelta(hours=hours)

    pipeline = [
        {"$match": {"timestamp": {"$gte": cutoff}}},
        {"$group": {
            "_id": {
                "$dateTrunc": {
                    "date": "$timestamp",
                    "unit": "minute",
                    "binSize": bucket_minutes,
                }
            },
            "count": {"$sum": 1},
            "critical": {
                "$sum": {"$cond": [
                    {"$eq": ["$severity", "critical"]}, 1, 0
                ]}
            },
            "high": {
                "$sum": {"$cond": [
                    {"$eq": ["$severity", "high"]}, 1, 0
                ]}
            },
        }},
        {"$sort": {"_id": 1}},
    ]

    return list(LogEvent.objects.aggregate(pipeline))
```

`$dateTrunc` raggruppa i timestamp in intervalli di 15 minuti. All'interno di ogni bucket, la pipeline conta gli eventi totali e li suddivide per severity. Il frontend lo renderizza come stacked area chart.

Questa query usa l'indice su `timestamp` di `LogEvent`. Senza quell'indice, MongoDB scansionerebbe ogni documento. Con milioni di eventi, è la differenza tra millisecondi e minuti.

### Query di pivot

L'API di pivot permette agli analisti di investigare indicatori specifici:

```python
def pivot(field: str, value: str, limit: int = 100):
    """Find all events matching a specific field value."""
    valid_fields = {
        "source_ip", "dest_ip", "username",
        "hostname", "event_type", "source_type",
    }
    if field not in valid_fields:
        raise ValidationError(f"Invalid pivot field: {field}")

    return LogEvent.objects(**{field: value}).order_by(
        "-timestamp"
    ).limit(limit)
```

Questo è il workflow di investigazione. Un analista vede un alert, annota il `source_ip` e fa pivot per trovare ogni evento da quell'IP. La whitelist dei campi validi impedisce query arbitrarie sui campi che potrebbero essere lente o sfruttabili.

## Gestione degli errori

### Gerarchia degli errori

`app/core/errors.py` definisce una gerarchia di errori strutturata:

```python
class AppError(Exception):
    status_code = 500
    error_type = "AppError"

    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

class NotFoundError(AppError):
    status_code = 404
    error_type = "NotFoundError"

class ValidationError(AppError):
    status_code = 422
    error_type = "ValidationError"

class AuthenticationError(AppError):
    status_code = 401
    error_type = "AuthenticationError"

class ForbiddenError(AppError):
    status_code = 403
    error_type = "ForbiddenError"

class ConflictError(AppError):
    status_code = 409
    error_type = "ConflictError"
```

Un singolo error handler di Flask cattura tutte le sottoclassi di `AppError`:

```python
def register_error_handlers(app: Flask):
    @app.errorhandler(AppError)
    def handle_app_error(error: AppError):
        return jsonify({
            "error": error.error_type,
            "message": error.message,
        }), error.status_code
```

Questo ti dà risposte di errore JSON coerenti su tutta l'API. Nessun endpoint restituisce errori HTML. Nessun endpoint restituisce formati di errore diversi. Il frontend può contare sul fatto che `response.data.error` e `response.data.message` siano presenti per tutte le risposte di errore.

### Gestione degli errori nel frontend

`src/core/lib/api.ts` rispecchia questo sul client:

```typescript
class ApiError extends Error {
  constructor(
    public status: number,
    public error: string,
    message: string
  ) {
    super(message);
  }
}

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response) {
      const { status, data } = error.response;
      throw new ApiError(status, data.error, data.message);
    }
    throw new ApiError(0, 'NetworkError', 'Connection failed');
  }
);
```

L'interceptor di Axios trasforma gli errori HTTP in oggetti `ApiError` tipizzati. L'handler globale `onError` di React Query può poi mostrare notifiche toast con il messaggio di errore. Un 401 fa scattare un logout. Un 403 mostra "Insufficient permissions." Un 422 mostra i dettagli dell'errore di validazione.

## Scenari comuni di debug

### "Gli eventi non compaiono in tempo reale"

Controlla questi punti in ordine:
1. La connessione SSE è stabilita? Apri i dev tools del browser, scheda Network, filtra per EventStream. Dovresti vedere una richiesta di lunga durata verso `/api/v1/logs/stream`.
2. Il buffering del proxy Nginx è disabilitato? Controlla `proxy_buffering off` nella config Nginx per gli endpoint SSE.
3. Redis è in esecuzione? `docker compose exec redis redis-cli ping` dovrebbe restituire `PONG`.
4. Gli eventi vengono pubblicati? `docker compose exec redis redis-cli XLEN siem:logs` mostra la lunghezza dello stream.

### "Le regole di correlazione non scattano"

1. Il thread del correlation engine è in esecuzione? Controlla i log del backend per "Correlation engine started."
2. La regola è abilitata? `CorrelationRule.objects(enabled=True)` in una shell Flask.
3. L'evento corrisponde alla regola? La funzione `_event_matches_rule()` controlla `event_type` e `source_type` rispetto alle condizioni della regola.
4. La regola è in cooldown? Dopo essere scattata una volta, non scatterà di nuovo per `CORRELATION_COOLDOWN_SECONDS` (default 300).
5. La cache delle regole è scaduta? Le regole sono in cache per 30 secondi. Le nuove regole impiegano fino a 30 secondi per essere valutate.

### "Errori di autenticazione a ogni richiesta"

1. Controlla la scadenza del JWT. Il default è 24 ore (`JWT_EXPIRY_HOURS` nella config).
2. Controlla che `SECRET_KEY` sia coerente tra i riavvii. Se la chiave cambia, tutti i token esistenti diventano invalidi.
3. Controlla il formato dell'header `Authorization`. Deve essere `Bearer <token>`, non solo il token.

## Cosa c'è dopo

Ora che hai capito come funziona il codice, leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per idee su come estendere questo progetto. Le challenge vanno dall'aggiunta di nuovi normalizer per sorgenti di log (principiante) all'implementazione del supporto per le regole Sigma e dei playbook SOAR (avanzato).
