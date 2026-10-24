# Architettura del sistema

Questo documento descrive come è progettato AngelusVigil e perché è stata presa ciascuna decisione architetturale. Leggilo prima di immergerti nell'implementazione.

## Architettura di alto livello

```
+----------------------------------------------------------------------+
|                        Docker Compose Stack                          |
|                                                                      |
|  +-----------+    +--------------------------------------------+     |
|  |  nginx    |    |            vigil-backend                   |     |
|  | (external)|    |                                            |     |
|  |           |    |  +--------+  +---------+  +-----------+    |     |
|  | access.log|--->|  | Tailer |->| Pipeline|->| Dispatcher|    |     |
|  |           |    |  +--------+  | (4-stage)|  +-----+----+    |     |
|  +-----------+    |              +-----+----+        |         |     |
|                   |                    |             |         |     |
|                   |              +-----+----+  +-----+------+ |     |
|                   |              |  Redis   |  | PostgreSQL | |     |
|                   |              | (windows |  | (events)   | |     |
|                   |              |  + pub/  |  +------------+ |     |
|                   |              |    sub)  |                  |     |
|                   |              +-----+----+                  |     |
|                   |                    |                       |     |
|                   |              +-----+----+                  |     |
|                   |              | WebSocket|                  |     |
|                   |              |  Relay   |                  |     |
|                   |              +-----+----+                  |     |
|                   |                    |   +---------------+   |     |
|                   |                    |   | FastAPI REST  |   |     |
|                   |                    |   | (6 routers)   |   |     |
|                   +--------------------+---+-------+-------+   |     |
|                                        |           |           |     |
|  +-------------------------------------+-----------+--------+  |     |
|  |        vigil-frontend               |           |        |  |     |
|  |  +-----------+  +----------+        |           |        |  |     |
|  |  | Dashboard |  | Threats  |<-------+<----------+        |  |     |
|  |  | (live)    |  | (table)  | WebSocket   REST API        |  |     |
|  |  +-----------+  +----------+                             |  |     |
|  +----------------------------------------------------------+  |     |
|                                                                 |     |
|  +--------------+  +--------------+                             |     |
|  | GeoIP        |  | MLflow       |                             |     |
|  | Updater      |  | (tracking)   |                             |     |
|  +--------------+  +--------------+                             |     |
+----------------------------------------------------------------------+
```

### Suddivisione dei componenti

**Log Tailer (tailer.py)**
- Scopo: Osservare il file di access log di nginx per nuove righe e passarle alla pipeline
- Responsabilità: Gestire la rotazione dei log (eventi di rename e create), persistere la posizione di lettura per il recupero dopo un crash, inserire le righe grezze nella coda async senza bloccare l'event loop
- Interfaccia: Inserisce righe `str` in `Pipeline.raw_queue` tramite `call_soon_threadsafe`

**Pipeline a 4 stadi (pipeline.py)**
- Scopo: Trasformare le righe di log grezze in candidati minaccia con un punteggio, attraverso quattro stadi sequenziali
- Responsabilità: Fare il parsing delle voci di log, estrarre e codificare le feature, assegnare il punteggio tramite regole + ML, inviare i risultati
- Interfaccia: Accetta righe grezze su `raw_queue`, emette oggetti `ScoredRequest` tramite la callback `on_result`

**Rule Engine (rules.py)**
- Scopo: Fornire un rilevamento immediato basato su pattern, senza modelli ML
- Responsabilità: Valutare su ogni richiesta 9 regole regex, 2 regole a soglia, il rilevamento di double-encoding e di scanner
- Interfaccia: `score_request(features, entry) -> RuleResult`

**Inference Engine (inference.py)**
- Scopo: Eseguire l'ensemble ONNX di 3 modelli per lo scoring basato su ML
- Responsabilità: Caricare le sessioni ONNX, applicare la normalizzazione RobustScaler, calcolare l'errore di ricostruzione dell'autoencoder, estrarre le probabilità RF, restituire i punteggi grezzi per singolo modello
- Interfaccia: `predict(batch) -> dict[str, list[float]]`

**Alert Dispatcher (dispatcher.py)**
- Scopo: Instradare le richieste con punteggio verso lo storage, gli alert in tempo reale e il logging
- Responsabilità: Persistere su PostgreSQL gli eventi di severità MEDIUM+, pubblicare alert JSON su Redis pub/sub, registrare tutti gli eventi come JSON strutturato
- Interfaccia: `dispatch(scored) -> None` (callback async)

**Redis (finestre scorrevoli + pub/sub)**
- Scopo: Doppio ruolo di feature store a finestre e di message broker in tempo reale
- Responsabilità: Memorizzare i sorted set per singolo IP per l'aggregazione comportamentale, inoltrare i messaggi di alert dal backend ai client WebSocket
- Interfaccia: Sorted set per `WindowAggregator`, pub/sub per il relay `AlertDispatcher` -> WebSocket

**PostgreSQL (storage degli eventi)**
- Scopo: Storage persistente per gli eventi di minaccia e i metadati dei modelli
- Responsabilità: Memorizzare tutti gli eventi di severità MEDIUM+ con contesto completo (feature, punteggi, geo, regole corrispondenti), memorizzare i metadati di training dei modelli ML e i percorsi degli artifact
- Interfaccia: ORM SQLModel tramite SQLAlchemy async

**React Frontend**
- Scopo: Interfaccia visiva per monitoraggio e investigazione
- Responsabilità: Dashboard live con statistiche e feed degli alert, tabella delle minacce filtrabile, stato dei modelli e controlli di retraining
- Interfaccia: REST API tramite Axios/React Query, WebSocket per gli alert live

## Flusso dei dati

### Flusso primario: dalla riga di log all'alert

Descrizione passo per passo di cosa succede quando nginx scrive una riga di log:

```
1. nginx writes to access.log
       │
       ▼
2. Watchdog PollingObserver fires on_modified
   _LogHandler._read_new_lines()
       │
       ▼
3. Raw line pushed to Pipeline.raw_queue (capacity: 1000)
   via loop.call_soon_threadsafe
       │
       ▼
4. Stage 1 (_parse_worker): parse_combined(line)
   Splits on quotes, extracts IP/method/path/status/UA
   Result: ParsedLogEntry
       │
       ▼
5. Stage 2 (_feature_worker):
   5a. GeoIP lookup → country_code, city, lat, lon
   5b. extract_request_features() → 23 per-request features
   5c. WindowAggregator.record_and_aggregate() → 12 windowed features
       (single Redis pipeline round-trip: 7 ZADD + 7 ZREMRANGEBYSCORE
        + 5 ZCOUNT + 4 ZRANGEBYSCORE + 7 EXPIRE)
   5d. encode_for_inference() → 35-dim float32 vector
   Result: EnrichedRequest
       │
       ▼
6. Stage 3 (_detection_worker):
   6a. RuleEngine.score_request() → RuleResult (score + matched rules)
   6b. If ML models loaded:
       InferenceEngine.predict() → raw {ae, rf, if} scores
       normalize + fuse + blend with rule score
   Result: ScoredRequest (final_score, detection_mode, ml_scores)
       |
       v
7. Stage 4 (_dispatch_worker):
   AlertDispatcher.dispatch(scored)
   7a. Log structured event to stdout
   7b. If severity >= MEDIUM:
       7b-i.  create_threat_event() → INSERT into threat_events
       7b-ii. Publish WebSocketAlert JSON to Redis "alerts" channel
       │
       ▼
8. Redis pub/sub → WebSocket relay → React dashboard AlertFeed
```

### Flusso secondario: retraining del modello

```
1. POST /models/retrain (API key required)
       │
       ▼
2. Fetch stored ThreatEvents from PostgreSQL
   Label via review_label or score thresholds
   Supplement with synthetic data if < 200 samples
       │
       ▼
3. TrainingOrchestrator.run(X, y)
   3a. prepare_training_data() → stratified split + SMOTE
   3b. train_autoencoder() on normal-only data (100 epochs)
   3c. train_random_forest() on labeled data (200 estimators)
   3d. train_isolation_forest() on normal-only data
       │
       ▼
4. Export to ONNX:
   ae.onnx, rf.onnx, if.onnx, scaler.json, threshold.json
       │
       ▼
5. validate_ensemble() on test set
   Quality gates: PR-AUC >= 0.85, F1 >= 0.80
       │
       ▼
6. Log to MLflow (params, metrics, artifacts)
       |
       v
7. Backend restart loads new ONNX models
   Switches detection_mode from "rules" to "hybrid"
```

## Design pattern

### Poison-Pill Shutdown

**Che cos'è:**
Un segnale di shutdown propagato attraverso la catena di code. Quando viene chiamato `Pipeline.stop()`, inserisce `None` in `raw_queue`. Ogni worker controlla la presenza di `None`, lo inoltra alla coda successiva ed esce.

**Dove lo usiamo:**
In tutti e quattro gli stadi della pipeline in `pipeline.py`. La poison pill si propaga a cascata:
`raw_queue → parsed_queue → feature_queue → alert_queue`

**Perché l'abbiamo scelto:**
Shutdown pulito e deterministico. Ogni worker termina l'elemento corrente prima di uscire. Nessuna race condition, nessun task orfano. L'alternativa (cancellare gli oggetti `asyncio.Task`) lascia il lavoro in uno stato incoerente se un worker è nel mezzo di un'elaborazione.

**Compromessi:**
- Pro: Svuotamento garantito degli elementi in transito, semplice da implementare, nessuna perdita di dati
- Contro: Lo shutdown non è istantaneo (bisogna attendere che ogni stadio elabori il proprio elemento corrente)

### Sidecar Deployment

**Che cos'è:**
AngelusVigil gira accanto al server nginx di destinazione senza modificarlo. Il backend legge il file di access log di nginx (volume montato in sola lettura) e il frontend fa da proxy per le chiamate API.

**Perché l'abbiamo scelto:**
Zero modifiche al codice dell'applicazione monitorata. Aggiungi AngelusVigil al tuo `compose.yml` e lo punti al volume dei log di nginx. L'applicazione monitorata non sa di essere osservata.

**Compromessi:**
- Pro: Non invasivo, funziona con qualsiasi deployment nginx, può essere aggiunto e rimosso senza influire sul target
- Contro: Limitato alle informazioni presenti nell'access log (niente body delle richieste, niente body delle risposte, nessun header oltre a User-Agent e Referer)

### Factory Pattern con Lifespan

**Che cos'è:**
La funzione `create_app()` costruisce l'applicazione FastAPI, e il context manager async `lifespan` gestisce l'avvio/spegnimento di tutti i sottosistemi.

**Dove lo usiamo:**
`factory.py` è l'unico punto di ingresso dell'applicazione. Il lifespan crea l'engine del database, connette Redis, inizializza GeoIP, costruisce la pipeline, avvia il tailer e al shutdown smonta tutto in ordine inverso.

**Perché l'abbiamo scelto:**
Gestione centralizzata delle risorse. Ogni connessione, file handle e task in background viene creato in un unico punto e ripulito in un unico punto. Questo previene i resource leak e rende la sequenza di avvio/shutdown esplicita e testabile.

## Separazione dei layer

```
+----------------------------------------------+
|  API Layer (api/)                            |
|  - Route handlers, request validation        |
|  - Depends on: Services, Schemas             |
|  - Never touches: Database directly, Redis   |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
|  Service Layer (services/)                   |
|  - Business logic, query building            |
|  - Depends on: Models, SQLAlchemy sessions   |
|  - Never touches: HTTP request objects       |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
|  Core Layer (core/)                          |
|  - Detection, ingestion, features, alerts    |
|  - Depends on: Redis, GeoIP, ONNX           |
|  - Never touches: HTTP, database directly    |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
|  Model Layer (models/)                       |
|  - SQLModel ORM definitions                  |
|  - Depends on: Nothing (leaf node)           |
|  - Never touches: Business logic             |
+----------------------------------------------+
```

### Cosa vive dove

**API Layer (api/):**
- File: `health.py`, `threats.py`, `stats.py`, `models_api.py`, `ingest.py`, `websocket.py`, `deps.py`
- Importa da: `services/`, `schemas/`, `config`
- Vietato: Query dirette al database, operazioni Redis, inferenza ML

**Service Layer (services/):**
- File: `threat_service.py`, `stats_service.py`
- Importa da: `models/`, tipi SQLAlchemy
- Vietato: Gestione di richieste/risposte HTTP, gestione dei WebSocket

**Core Layer (core/):**
- File: Tutto ciò che sta sotto `core/ingestion/`, `core/detection/`, `core/features/`, `core/alerts/`, `core/enrichment/`
- Importa da: client Redis, ONNX runtime, libreria GeoIP
- Vietato: Import di framework HTTP, accesso diretto al database (il dispatcher usa il service layer)

**Model Layer (models/):**
- File: `ThreatEvent.py`, `ModelMetadata.py`, `Base.py`
- Importa da: `sqlmodel`, `uuid`, standard library
- Vietato: Business logic, dipendenze da framework

## Data model

### ThreatEvent

```python
class ThreatEvent(TimestampedModel, table=True):
    __tablename__ = "threat_events"

    source_ip: str              # INET type, indexed
    method: str                 # HTTP method
    path: str                   # Request path
    status_code: int
    response_size: int
    user_agent: str
    threat_score: float         # Indexed, 0.0-1.0
    severity: str               # HIGH/MEDIUM/LOW, indexed
    component_scores: dict      # JSON: per-rule and per-model scores
    matched_rules: list[str]    # JSON array of matched rule names
    country: str | None
    city: str | None
    lat: float | None
    lon: float | None
    feature_vector: list[float] # JSON: full 35-dim vector for replay
    model_version: str | None
    ml_scores: dict | None      # JSON: {ae: 0.7, rf: 0.8, if: 0.3}
    reviewed: bool = False      # Analyst review flag
    review_label: str | None    # "true_positive" or "false_positive"
```

**Decisioni di design chiave:**
- `feature_vector` è memorizzato come JSON per permettere il retraining dei modelli dagli eventi storici senza dover riestrarre le feature
- `component_scores` memorizza sia i punteggi delle regole sia quelli ML per singolo modello, per la spiegabilità
- `reviewed` e `review_label` supportano il ciclo di active learning: gli analisti marcano gli eventi, e quelle etichette diventano dati di training
- Indice parziale su `reviewed=FALSE` per interrogare in modo efficiente gli eventi non ancora revisionati

### ModelMetadata

```python
class ModelMetadata(TimestampedModel, table=True):
    __tablename__ = "model_metadata"

    model_type: str             # "autoencoder", "rf", "if"
    version: str                # Content-addressable hash
    training_samples: int
    metrics: dict               # JSON: model-specific metrics
    artifact_path: str          # Path to ONNX file
    is_active: bool             # One active per model_type
    mlflow_run_id: str | None
    threshold: float | None     # AE anomaly threshold
    notes: str | None
```

**Decisioni di design chiave:**
- Il flag `is_active` con un indice parziale su `(model_type, is_active=TRUE)` garantisce che per ogni tipo di modello sia attiva una sola versione alla volta
- `version` usa un hashing content-addressable, così output identici del modello producono la stessa stringa di versione

## Architettura di sicurezza

### Threat model

Da cosa ci stiamo proteggendo:
1. **Attacchi alle web application** - SQL injection, XSS, command injection, path traversal, SSRF e altre categorie di attacco OWASP Top 10 rivolte al server nginx monitorato
2. **Reconnaissance e scanning** - Strumenti automatici (nikto, sqlmap, nmap) che cercano vulnerabilità
3. **Anomalie comportamentali** - Credential stuffing, DDoS a livello applicativo e altri attacchi che appaiono normali sulla singola richiesta ma sono anomali nell'aggregato

Da cosa NON ci stiamo proteggendo (fuori ambito):
- Attacchi a livello di rete (SYN flood, IP spoofing) poiché vediamo solo dati a livello HTTP provenienti dagli access log
- Body delle richieste cifrati, poiché gli access log di nginx non contengono dati POST né header non standard
- Minacce interne con accesso diretto al database di AngelusVigil stesso

### Livelli di difesa

```
Layer 1: Rule Engine (immediate, deterministic)
    Pattern matching against known attack signatures
    Behavioral thresholds from windowed features
    ↓
Layer 2: ML Ensemble (learned, probabilistic)
    Autoencoder anomaly detection (unsupervised)
    Random forest classification (supervised)
    Isolation forest outlier detection (unsupervised)
    ↓
Layer 3: Score Blending + Severity Classification
    30% rule weight + 70% ML weight
    HIGH/MEDIUM/LOW severity tiers
    ↓
Layer 4: Alert Dispatch + Human Review
    Persistent storage for forensic analysis
    Real-time WebSocket alerts for immediate triage
    Analyst review labels for active learning
```

### Sicurezza delle API

- Validazione dell'header `X-API-Key` su tutti gli endpoint di mutazione (batch ingest, retrain)
- Le probe di health e readiness non sono autenticate (necessarie per gli healthcheck di Docker)
- L'endpoint WebSocket non è autenticato (trasmette solo alert, non dati grezzi)
- Il container backend monta il volume dei log di nginx in sola lettura, impedendo qualsiasi scrittura verso il sistema monitorato

## Strategia di storage

### PostgreSQL (storage persistente)

**Cosa memorizziamo:**
- Eventi di minaccia (severità MEDIUM+) con contesto completo per l'analisi forense
- Metadati dei modelli e metriche di training per il tracciamento delle versioni

**Perché PostgreSQL:**
Garanzie ACID per la memorizzazione degli eventi di minaccia. Non vuoi perdere un attacco rilevato a causa di una race condition. Il supporto alle colonne JSON gestisce i campi a schema variabile (component_scores, ml_scores, feature_vector) senza richiedere un document database. Il driver async (asyncpg) si integra in modo pulito con il ciclo di vita async di FastAPI.

**Stima dello storage:**
Ogni riga di evento di minaccia pesa circa 2-4 KB, includendo il vettore di 35 float delle feature e i campi JSON. Con un tasso di rilevamento dell'1-5% su traffico moderato (10K richieste/ora), si ottengono 100-500 eventi/ora, ovvero circa 10 GB/mese di storage.

### Redis (stato effimero + messaggistica)

**Cosa memorizziamo:**
- Sorted set a finestra scorrevole per singolo IP (7 set per IP tracciato, TTL di 15 minuti)
- Messaggi pub/sub degli alert in tempo reale (effimeri, nessuna persistenza necessaria)

**Perché Redis:**
I sorted set con query di range basate sul punteggio sono perfetti per l'aggregazione a finestra scorrevole. `ZCOUNT key <5min_ago> +inf` viene eseguito in tempo O(log N). La pipeline raggruppa tutte le operazioni 7 ZADD + 7 ZREMRANGEBYSCORE + 5 ZCOUNT + 4 ZRANGEBYSCORE + 7 EXPIRE in un unico round-trip.

Il pub/sub fornisce il meccanismo di fan-out per il relay WebSocket. Quando il backend pubblica un alert, ogni client WebSocket connesso lo riceve indipendentemente da quale worker del backend abbia elaborato la richiesta originale. Questo disaccoppia i producer (pipeline) dai consumer (connessioni WebSocket).

**Stima della memoria:**
Ogni IP tracciato usa 7 sorted set con un TTL di 15 minuti. Con 10K IP univoci e 50 membri per set, si ottengono circa 10 MB di memoria Redis.

## Configurazione

### Variabili d'ambiente

```bash
# Server
HOST=0.0.0.0               # Bind address
PORT=8000                   # Backend port
DEBUG=false                 # Debug mode (enables tracebacks in responses)
LOG_LEVEL=INFO              # Python logging level
API_KEY=                    # Required for mutation endpoints

# Database
DATABASE_URL=postgresql+asyncpg://vigil:changeme@localhost:5432/angelusvigil

# Redis
REDIS_URL=redis://localhost:6379

# Paths
NGINX_LOG_PATH=/var/log/nginx/access.log
GEOIP_DB_PATH=/usr/share/GeoIP/GeoLite2-City.mmdb
MODEL_DIR=data/models

# Pipeline Tuning
RAW_QUEUE_SIZE=1000         # Backpressure capacity for raw log lines
PARSED_QUEUE_SIZE=500       # Capacity after parsing
FEATURE_QUEUE_SIZE=200      # Capacity after feature extraction
ALERT_QUEUE_SIZE=100        # Capacity for scored requests awaiting dispatch
BATCH_SIZE=32               # ML inference batch size
BATCH_TIMEOUT_MS=50         # Max wait before sending partial batch

# ML Ensemble
ENSEMBLE_WEIGHT_AE=0.40     # Must sum to 1.0
ENSEMBLE_WEIGHT_RF=0.40
ENSEMBLE_WEIGHT_IF=0.20
AE_THRESHOLD_PERCENTILE=99.5   # Autoencoder anomaly threshold calibration
MLFLOW_TRACKING_URI=file:./mlruns
```

### Strategia di configurazione

La configurazione è gestita tramite Pydantic Settings (`config.py`), che carica da variabili d'ambiente e file `.env`. Un `model_validator` impone all'avvio che i pesi dell'ensemble sommino a 1.0, fallendo subito anziché produrre silenziosamente punteggi sbagliati a runtime.

**Sviluppo:** Il file `.env` fornisce i valori predefiniti. Il compose di sviluppo monta i volumi del codice sorgente per l'hot-reload e usa timeout degli healthcheck più brevi.

**Produzione:** Tutti i segreti (API_KEY, POSTGRES_PASSWORD, GEOIP_LICENSE_KEY) sono obbligatori tramite la sintassi `${VAR:?error}` in `compose.yml`, forzando una configurazione esplicita anziché ricadere su valori predefiniti insicuri.

## Considerazioni sulle performance

### Colli di bottiglia

Dove questo sistema rallenta sotto carico:

1. **Inferenza ML** - L'inferenza ONNX richiede 5-20ms per richiesta senza batching. A 1000 req/s, questo creerebbe un backlog. La pipeline usa il dynamic batching (raccoglie fino a 32 richieste o attende 50ms, a seconda di quale arriva prima) per ammortizzare l'overhead.

2. **Round-trip verso Redis** - Ogni richiesta richiede 7 ZADD + 7 trim + 5 count + 4 range + 7 expire. Il pipelining riduce questo da 30 round-trip a 1, ma a frequenze di richieste molto alte la pipeline Redis stessa diventa un collo di bottiglia.

3. **Lookup GeoIP** - Il database MaxMind mappato in memoria (mmap) è veloce (~100us) ma blocca il thread. Viene avvolto con `asyncio.to_thread()` per evitare di bloccare l'event loop.

### Ottimizzazioni

- **Dynamic batching** nel detection worker: Raccoglie i vettori di feature in batch fino a 32 prima di eseguire l'inferenza ONNX. Questo migliora il throughput da ~35 req/s (senza batching) a ~640 req/s
- **Una sola pipeline Redis** per richiesta: Tutte le 30 operazioni di aggregazione a finestra vengono eseguite in un unico round-trip della pipeline (~1ms in totale contro ~30ms sequenziali)
- **Esecuzione ONNX single-thread**: `inter_op_num_threads=1` e `intra_op_num_threads=1` evita la contesa tra thread nel processo Python async. I thread pool di ONNX si contendono il tempo CPU con l'event loop di asyncio
- **Percorso di parsing veloce**: Parsing con string-split per il formato combined standard di nginx, con ripiego su regex solo per le righe non standard. Il percorso con split è 3-5 volte più veloce

### Scalabilità

**Scalabilità verticale:**
Il limite di throughput di un singolo processo è circa 640 req/s con il batching. Aggiungere più core CPU non aiuta perché la pipeline è single-thread per design (event loop asyncio). L'uso di memoria scala linearmente con gli IP tracciati in Redis.

**Scalabilità orizzontale:**
Per gestire un throughput più alto, esegui più istanze del backend ognuna delle quali fa il tail di una partizione del flusso di log (oppure usa un log bus condiviso come Kafka). Il pub/sub di Redis distribuisce già gli alert a tutti i client WebSocket indipendentemente da quale backend abbia elaborato l'evento. PostgreSQL gestisce senza problemi le scritture concorrenti da più backend.

## Decisioni di design

### Perché ONNX invece di PyTorch/sklearn nativi

**Cosa abbiamo scelto:**
Esportare tutti e tre i modelli in ONNX e usare ONNX Runtime per l'inferenza.

**Alternative considerate:**
- Inferenza PyTorch nativa: Funziona ma è più lenta del 27% su CPU, e richiede l'intera dipendenza PyTorch nel container backend
- TorchScript: Meglio di PyTorch puro ma richiede comunque il runtime PyTorch. ONNX è indipendente dal framework
- TensorFlow Lite: Buone prestazioni di inferenza ma aggiunge una seconda dipendenza da framework ML accanto a sklearn

**Compromessi:**
ONNX Runtime è più veloce, più leggero e indipendente dal framework. Il passaggio di export aggiunge complessità durante il training, e fare debug dei modelli ONNX è più difficile che con PyTorch nativo. Ma poiché l'inferenza è il percorso critico e il training è raro, ottimizzare la latenza dell'inferenza vince.

### Perché regole + ML invece di solo ML

**Cosa abbiamo scelto:**
Un sistema ibrido che combina i punteggi basati su regole e quelli basati su ML.

**Alternative considerate:**
- Solo ML: Architettura più semplice, ma inutile il primo giorno senza dati di training
- Solo regole: Approccio collaudato (ModSecurity CRS), ma non può rilevare attacchi nuovi o anomalie comportamentali

**Compromessi:**
L'approccio ibrido offre copertura immediata (regole) che migliora nel tempo (ML). Il peso di blend 30/70 dà alle regole abbastanza influenza per intercettare le corrispondenze ad alta confidenza lasciando che sia il ML a dominare per lo scoring più sfumato. Il costo è la complessità architetturale: due percorsi di scoring che devono essere normalizzati e combinati.

### Perché i sorted set di Redis per le feature a finestra

**Cosa abbiamo scelto:**
Sorted set di Redis con timestamp Unix come punteggi, ripuliti tramite `ZREMRANGEBYSCORE`.

**Alternative considerate:**
- Finestre scorrevoli in memoria (`deque` Python con timestamp): Più semplici ma non sopravvivono ai riavvii del processo e non condividono lo stato tra più istanze del backend
- Window function di PostgreSQL: Corrette ma troppo lente. Ogni richiesta richiederebbe una query al database con una window function sugli eventi recenti. A 500 req/s, il database diventa il collo di bottiglia
- Redis Streams: Migliori per l'elaborazione ordinata degli eventi ma peggiori per l'aggregazione ad accesso casuale. I sorted set supportano `ZCOUNT` per il conteggio su intervalli in O(log N)

**Compromessi:**
I sorted set di Redis ci danno query a finestra sotto il millisecondo che sopravvivono ai riavvii del backend e supportano la scalabilità orizzontale. Il costo è la complessità operativa (un altro servizio da eseguire) e il data model a 7 sorted set per IP che richiede una gestione attenta delle chiavi.

## Architettura di deployment

```
+---------------------------------------------+
|       Docker Compose (compose.yml)          |
|                                             |
|  +----------+  +----------+  +-----------+  |
|  | postgres |  |  redis   |  |  geoip    |  |
|  | :5432    |  |  :6379   |  |  updater  |  |
|  +----+-----+  +----+-----+  +-----------+  |
|       |              |                       |
|       +------+-------+                       |
|              |                               |
|       +------+------+                        |
|       |   backend   | :8000 (internal)       |
|       |  (FastAPI)  |                        |
|       +------+------+                        |
|              |                               |
|       +------+------+                        |
|       |  frontend   | :80 (host-mapped)      |
|       |  (nginx +   |                        |
|       |   React)    |                        |
|       +-------------+                        |
+---------------------------------------------+
```

**Servizi:**
- **postgres** (18-alpine): Volume persistente, healthcheck tramite `pg_isready`, periodo di avvio di 30s
- **redis** (7.4-alpine): `redis.conf` personalizzato per il tuning, persistenza AOF, healthcheck tramite `redis-cli ping`
- **backend**: Build Docker multi-stage con uv, legge i log di nginx tramite volume in sola lettura, periodo di avvio di 180s (consente il training iniziale dei modelli)
- **frontend**: Build di produzione Vite servita da nginx, fa da proxy di `/api` e `/ws` verso il backend
- **geoip-updater**: Sidecar MaxMind, aggiorna il database GeoLite2-City ogni 168 ore

**Reti:**
- `vigil_network`: Bridge interno che collega tutti i servizi
- `certgames_net`: Rete esterna per l'integrazione con l'applicazione monitorata (CertGames)

**Volumi:**
- `postgres_data`, `redis_data`: Stato persistente
- `geoip_data`: Condiviso tra updater e backend
- `model_data`: Artifact dei modelli ONNX
- `nginx_logs`: Volume esterno dall'nginx monitorato (montato in sola lettura nel backend)

## Strategia di gestione degli errori

### Tipi di errore

1. **Errori di parsing** - Righe di log malformate che non corrispondono al formato combined di nginx. Contati in `stats["parse_errors"]`, registrati e saltati. La pipeline continua a elaborare la riga successiva.

2. **Errori di estrazione delle feature** - Fallimenti del lookup GeoIP, problemi di connessione a Redis durante l'aggregazione a finestra. Registrati con l'IP sorgente per il debug. La richiesta viene scartata dalla pipeline ma non manda in crash il worker.

3. **Errori di inferenza ML** - Fallimenti della sessione ONNX, mancate corrispondenze di shape. Il detection worker ripiega sullo scoring solo con regole per quella richiesta. Se l'inference engine fallisce in modo costante, viene marcato come non caricato.

4. **Errori di dispatch** - Fallimenti di scrittura su PostgreSQL, fallimenti di publish su Redis pub/sub. Registrati e contati. L'evento va perso ma la pipeline continua. I fallimenti di scrittura su PostgreSQL sono rari con il connection pooling; i fallimenti del pub/sub di Redis significano che il relay WebSocket perde quell'alert ma il successivo funziona regolarmente.

### Meccanismi di recupero

**Recupero del log tailer dopo un crash:**
Il tailer persiste `(inode, offset)` in un file JSON di posizione dopo ogni batch di letture. Al riavvio, riprende dalla posizione salvata se l'inode corrisponde (stesso file). Se l'inode è cambiato (il log è stato ruotato), parte dalla posizione 0 del nuovo file.

**Backpressure della pipeline:**
Ogni coda ha una dimensione massima (1000/500/200/100). Se uno stadio a valle resta indietro, la coda a monte si riempie e la chiamata `await queue.put()` blocca il worker a monte. Questo previene l'esaurimento della memoria e propaga la backpressure fino al log tailer, che scarta le righe quando la coda raw è piena (registrato come warning).

**Degradazione graduale:**
Se i modelli ONNX non sono disponibili, il sistema funziona in modalità solo regole. Se Redis è giù, le feature a finestra restituiscono zeri (l'estrazione delle feature intercetta l'eccezione). Se GeoIP non è disponibile, l'enrichment geografico viene saltato. Il fallimento di ciascun sottosistema è isolato dagli altri.

## Estensibilità

### Dove aggiungere funzionalità

**Aggiungere una nuova regola di rilevamento:**
Aggiungi una voce `_PatternRule` a `_PATTERN_RULES` in `rules.py` con una regex compilata e un punteggio. Il rule engine la valuterà automaticamente su ogni richiesta. Non servono registrazioni né collegamenti.

**Aggiungere un nuovo modello ML all'ensemble:**
1. Aggiungi una funzione di training in `ml/`
2. Aggiungi una funzione di export in `ml/export_onnx.py`
3. Carica la nuova sessione ONNX in `inference.py`
4. Aggiungi una funzione di normalizzazione in `ensemble.py`
5. Aggiungi la chiave del modello e il peso alla configurazione dei pesi dell'ensemble
6. Aggiorna il metodo `_score_with_ml` in `pipeline.py`

**Aggiungere una nuova feature:**
1. Calcola la feature in `extractor.py` (per richiesta) o `aggregator.py` (a finestra)
2. Aggiungi il nome della feature a `FEATURE_ORDER` in `mappings.py`
3. Aggiorna `encode_for_inference` in `encoder.py` se la feature richiede una codifica speciale
4. Riesegui il training di tutti i modelli (la dimensione del vettore di feature cambia)

**Aggiungere un nuovo endpoint API:**
Crea un nuovo file router in `api/`, aggiungi i route handler e registralo in `create_app()` in `factory.py`.

## Limitazioni

Limitazioni architetturali attuali:

1. **Singola fonte di file di log** - Il tailer osserva un solo file. I deployment multi-server richiederebbero un livello di aggregazione dei log (Fluentd, Filebeat) che alimenti un bus condiviso. Non è difficile da aggiungere, ma non è ancora stato realizzato.

2. **Nessuna analisi del body delle richieste** - Gli access log di nginx non contengono body POST né header personalizzati. Gli attacchi nascosti nei body delle richieste (come l'exploit originale di Struts2 CVE-2017-5638 tramite l'header Content-Type) sono invisibili. Estendere il sistema agli error log di nginx o ai log a livello applicativo colmerebbe questa lacuna.

3. **Il retraining richiede un riavvio** - Dopo il training di nuovi modelli, il backend deve riavviarsi per caricare le nuove sessioni ONNX. Ricaricare a caldo le sessioni ONNX senza downtime richiederebbe un model registry e un meccanismo di scambio delle versioni.

4. **Nessuna pipeline distribuita** - La pipeline a 4 stadi gira in un singolo processo Python. Per un traffico superiore a ~640 req/s, devi partizionare il flusso di log. Un'architettura basata su Kafka con consumer group rimuoverebbe questo limite.

Sono compromessi consapevoli. Risolverli aggiunge una complessità operativa non giustificata per la scala di deployment prevista (sidecar per una singola istanza nginx).

## Confronto con sistemi simili

### vs. ModSecurity + OWASP CRS

ModSecurity è il motore WAF standard del settore. Gira inline (come modulo nginx) e può bloccare le richieste in tempo reale. AngelusVigil gira out-of-band (leggendo i log a posteriori) e può solo rilevare, non bloccare.

Il compromesso: ModSecurity ha zero attrito di deployment per il blocco ma è solo a regole. AngelusVigil aggiunge rilevamento ML e analisi comportamentale al prezzo di essere solo di rilevamento. In pratica, molti team usano entrambi: ModSecurity per bloccare gli attacchi noti, e un sistema come AngelusVigil per rilevare ciò che ModSecurity ha mancato.

### vs. Elastic SIEM / Splunk

Elastic SIEM e Splunk sono SIEM completi che acquisiscono log da molte fonti, correlano gli eventi e forniscono dashboard. AngelusVigil fa una cosa sola: analizzare il traffico HTTP di nginx alla ricerca di minacce.

Il compromesso: i SIEM sono più generali ma richiedono infrastruttura e tuning significativi. AngelusVigil è costruito appositamente per il rilevamento delle minacce HTTP con un ensemble ML addestrato. Useresti AngelusVigil come rilevatore specializzato che alimenta un SIEM con eventi per una correlazione più ampia.

### vs. AWS WAF / Cloudflare WAF

I WAF cloud operano a livello di CDN/load balancer e possono bloccare il traffico prima che raggiunga il tuo server. Usano modelli ML proprietari addestrati sul traffico aggregato di milioni di siti.

Il compromesso: i WAF cloud hanno dataset di training enormi e zero sforzo di deployment, ma sono opachi (non puoi ispezionare i modelli) e costosi su larga scala. AngelusVigil è completamente trasparente, personalizzabile e gratuito, ma richiede la tua infrastruttura e i tuoi dati di training.

## Riferimento ai file chiave

Mappa rapida di dove trovare le cose:

- `backend/app/factory.py` - Ciclo di vita dell'applicazione (avvio/arresto di tutti i sottosistemi)
- `backend/app/config.py` - Tutta la configurazione con valori predefiniti e validazione
- `backend/app/core/ingestion/pipeline.py` - La pipeline async a 4 stadi
- `backend/app/core/ingestion/tailer.py` - Log file tailer basato su Watchdog
- `backend/app/core/detection/rules.py` - Rule engine con 9 pattern + 2 soglie
- `backend/app/core/detection/inference.py` - Inference engine ONNX (3 modelli)
- `backend/app/core/detection/ensemble.py` - Normalizzazione e fusione dei punteggi
- `backend/app/core/features/extractor.py` - 23 feature per richiesta
- `backend/app/core/features/aggregator.py` - 12 feature a finestra basate su Redis
- `backend/ml/orchestrator.py` - Pipeline di training end-to-end
- `frontend/src/pages/dashboard/index.tsx` - Dashboard con feed di alert live
- `frontend/src/api/hooks/useAlerts.ts` - Connessione WebSocket con Zustand

## Prossimi passi

Ora che hai compreso l'architettura:
1. Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per una walkthrough dettagliata del codice
2. Prova a modificare i pesi dell'ensemble in `.env` e osserva come cambia la distribuzione della severità sulla dashboard
