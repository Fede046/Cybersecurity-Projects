# Guida all'implementazione

Questo documento ripercorre il codice vero e proprio. Seguiremo il flusso dei dati dalle righe di log grezze fino agli alert di minaccia con punteggio, spiegheremo l'implementazione di ciascun componente e copriremo le decisioni che stanno dietro al codice.

## Walkthrough della struttura dei file

```
backend/
├── app/
│   ├── api/
│   │   ├── deps.py              # Dependency injection (DB sessions, API key)
│   │   ├── health.py            # GET /health, GET /ready
│   │   ├── ingest.py            # POST /ingest/batch (manual log ingestion)
│   │   ├── models_api.py        # GET /models/status, POST /models/retrain
│   │   ├── stats.py             # GET /stats (time-windowed aggregates)
│   │   ├── threats.py           # GET /threats, GET /threats/{id}
│   │   └── websocket.py         # WS /ws/alerts (real-time alert stream)
│   ├── core/
│   │   ├── alerts/
│   │   │   └── dispatcher.py    # Routes scored events to storage + pub/sub
│   │   ├── detection/
│   │   │   ├── ensemble.py      # Score normalization, fusion, severity
│   │   │   ├── inference.py     # ONNX runtime (3-model inference)
│   │   │   └── rules.py         # ModSecurity-inspired rule engine
│   │   ├── enrichment/
│   │   │   └── geoip.py         # MaxMind GeoLite2 lookups
│   │   ├── features/
│   │   │   ├── aggregator.py    # Redis sorted set windowed features
│   │   │   ├── encoder.py       # Feature dict -> float32 vector
│   │   │   ├── extractor.py     # 23 per-request feature extraction
│   │   │   ├── mappings.py      # Feature order, encoders, constants
│   │   │   ├── patterns.py      # Compiled attack regex patterns
│   │   │   └── signatures.py    # Bot and scanner UA signatures
│   │   └── ingestion/
│   │       ├── parsers.py       # Nginx log line parser (split + regex)
│   │       ├── pipeline.py      # 4-stage async pipeline
│   │       └── tailer.py        # Watchdog file tailer with rotation
│   ├── models/
│   │   ├── Base.py              # TimestampedModel base class
│   │   ├── ModelMetadata.py     # ML model version tracking
│   │   └── ThreatEvent.py       # Stored threat events
│   ├── schemas/                 # Pydantic response models
│   ├── services/
│   │   ├── stats_service.py     # Aggregate query builder
│   │   └── threat_service.py    # CRUD for threat events
│   ├── config.py                # Pydantic Settings
│   └── factory.py               # App factory + lifespan
├── ml/
│   ├── autoencoder.py           # ThreatAutoencoder PyTorch module
│   ├── experiment.py            # MLflow experiment wrapper
│   ├── export_onnx.py           # PyTorch/sklearn -> ONNX export
│   ├── orchestrator.py          # End-to-end training pipeline
│   ├── scaler.py                # IQR-based FeatureScaler
│   ├── splitting.py             # Stratified split + SMOTE
│   ├── train_autoencoder.py     # AE training with early stopping
│   ├── train_classifiers.py     # RF + IF training
│   └── validation.py            # Ensemble validation + quality gates
├── cli/
│   └── main.py                  # Typer CLI (train, retrain, replay)
└── tests/                       # pytest suite
```

## Costruire la pipeline di ingestione dei log

### Step 1: parsing delle righe di log nginx

Il parser trasforma il testo grezzo in dati strutturati. Una riga nginx in formato combined ha questo aspetto:

```
93.184.216.34 - - [15/Mar/2026:09:22:31 +0000] "GET /api/users?id=1 HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
```

La funzione `parse_combined` in `parsers.py` prova prima un percorso veloce basato sullo split di stringhe, ripiegando sulla regex per i casi limite:

```python
def parse_combined(line: str) -> ParsedLogEntry | None:
    if not line:
        return None

    result = _parse_split(line)
    if result is not None:
        return result

    return _parse_regex(line)
```

Il parser basato su split sfrutta il fatto che il formato combined di nginx usa le virgolette come delimitatori. Dividendo su `"` si ottengono segmenti prevedibili:

```python
def _parse_split(line: str) -> ParsedLogEntry | None:
    try:
        parts = line.split('"')
        if len(parts) < 6:
            return None

        prefix = parts[0]
        request_line = parts[1]
        status_size = parts[2]
        referer_raw = parts[3]
        user_agent = parts[5]

        bracket_open = prefix.index("[")
        bracket_close = prefix.index("]")
        ip = prefix[:bracket_open].split()[0]
        timestamp = datetime.strptime(
            prefix[bracket_open + 1:bracket_close],
            _TIMESTAMP_FMT,
        )
        ...
```

**Perché funziona:** il formato combined di nginx mette sempre tra virgolette la request line, il referer e lo user-agent. I segmenti tra le virgolette seguono un pattern fisso. Lo split di stringhe è 3-5 volte più veloce del matching con regex perché evita il backtracking.

**Quando il percorso split fallisce:** se la riga di log ha un formato insolito (virgolette extra nella richiesta, campi malformati), `_parse_split` restituisce `None` e se ne occupa il fallback con regex:

```python
_COMBINED_RE = re.compile(
    r"(?P<ip>\S+) \S+ \S+ "
    r"\[(?P<timestamp>[^\]]+)\] "
    r'"(?P<request>[^"]*)" '
    r"(?P<status>\d{3}) "
    r"(?P<size>\S+) "
    r'"(?P<referer>[^"]*)" '
    r'"(?P<user_agent>[^"]*)"'
)
```

Il risultato è un dataclass `ParsedLogEntry` frozen e con slot. Frozen perché le entry parsate non dovrebbero mai essere mutate. Con slot per efficienza di memoria quando si elaborano migliaia di entry.

### Step 2: la pipeline a 4 stadi

La classe `Pipeline` in `pipeline.py` concatena quattro worker async collegati da code dimensionate:

```
raw_queue (1000) → [parse] → parsed_queue (500) → [features]
→ feature_queue (200) → [detect] → alert_queue (100) → [dispatch]
```

Ogni coda ha una dimensione massima che fornisce backpressure. Se lo stadio di detection rimane indietro, la feature queue si riempie, il che blocca il feature worker, il che fa accumulare la parsed queue, il che alla fine fa accumulare la raw queue. Quando la raw queue è piena, il tailer scarta le righe (loggate come warning).

La pipeline avvia quattro oggetti `asyncio.Task`:

```python
async def start(self) -> None:
    self._tasks = [
        asyncio.create_task(self._parse_worker(), name="parse"),
        asyncio.create_task(self._feature_worker(), name="feature"),
        asyncio.create_task(self._detection_worker(), name="detection"),
        asyncio.create_task(self._dispatch_worker(), name="dispatch"),
    ]
```

Ogni worker segue lo stesso pattern: preleva dalla coda di input, elabora, inserisce nella coda di output, ripete. Lo shutdown usa una poison-pill (`None`) che si propaga a cascata lungo la catena:

```python
async def stop(self) -> None:
    await self.raw_queue.put(None)
    await asyncio.gather(*self._tasks)
```

Quando il parse worker vede `None`, lo inoltra alla parsed queue ed esce. Il feature worker vede `None` sulla parsed queue, lo inoltra ed esce. Questo si propaga a cascata finché tutti i worker non si sono fermati.

### Step 3: file tailing con rilevamento della rotazione

Il `LogTailer` in `tailer.py` osserva il file di log di nginx usando il `PollingObserver` di Watchdog (non inotify, perché i volumi Docker non sempre propagano gli eventi inotify).

L'handler risponde a tre eventi:

- `on_modified`: nuovi dati aggiunti in coda. Legge le nuove righe e le inserisce nella coda.
- `on_moved`: rotazione del log tramite rename (`access.log` -> `access.log.1`). Finisce di leggere il vecchio file, poi riapre il target alla posizione 0.
- `on_created`: rotazione del log in cui un nuovo file compare nel path target. Come `on_moved`.

La persistenza della posizione è il dettaglio critico per il crash recovery:

```python
def _save_position(self) -> None:
    if self._position_path is None or self._file is None:
        return
    try:
        self._position_path.write_text(
            json.dumps({
                "inode": self._inode,
                "offset": self._file.tell(),
            }),
        )
    except OSError:
        logger.debug("Failed to save tailer position")
```

Al riavvio, `_open_target` controlla se l'inode salvato corrisponde al file corrente. Se corrisponde, il tailer riprende dall'offset salvato. Se l'inode è cambiato (il file è stato ruotato), parte dall'inizio del nuovo file.

**Perché il tracciamento dell'inode è importante:** il nome del file `access.log` può puntare a file diversi nel tempo a causa della rotazione. L'inode è l'identità del file nel file system. Se l'inode è cambiato, il file che stavamo leggendo è stato ruotato via ed è stato creato un nuovo file.

**Perché `call_soon_threadsafe`:** Watchdog esegue i propri handler di callback in un thread separato. La coda asyncio appartiene al thread dell'event loop. `call_soon_threadsafe` colma il divario schedulando la chiamata `put_nowait` sul thread dell'event loop.

## Costruire il sistema di estrazione delle feature

### Feature per richiesta

La funzione `extract_request_features` in `extractor.py` calcola 23 feature da un singolo `ParsedLogEntry`:

```python
def extract_request_features(
    entry: ParsedLogEntry,
    country_code: str = "",
) -> dict[str, int | float | bool | str]:
    full_uri = entry.path
    if entry.query_string:
        full_uri = f"{entry.path}?{entry.query_string}"

    ua_lower = entry.user_agent.lower()
    non_alnum = sum(1 for c in entry.path if not c.isalnum())
    path_len = len(entry.path)

    _, ext = splitext(entry.path)

    return {
        "http_method": entry.method,
        "path_depth": len([s for s in entry.path.split("/") if s]),
        "path_entropy": _shannon_entropy(entry.path),
        ...
    }
```

Ogni feature è scelta deliberatamente per uno specifico segnale di detection:

- `path_entropy` intercetta i payload di attacco che hanno alta casualità rispetto ai normali path URL
- `special_char_ratio` segnala i path con concentrazioni insolite di caratteri non alfanumerici (comuni nei payload di SQL injection e XSS)
- `has_double_encoding` rileva le tecniche di evasione in cui gli attaccanti codificano i payload due volte per aggirare un semplice URL decoding
- `is_known_scanner` confronta le stringhe User-Agent con una lista curata di tool di scanning noti

### Aggregazione a finestra

Il `WindowAggregator` in `aggregator.py` calcola 12 feature comportamentali per IP usando i sorted set di Redis. L'intera operazione avviene in un singolo round-trip in pipeline:

```python
async def record_and_aggregate(
    self, ip, request_id, path, path_depth,
    method, status_code, user_agent, response_size, timestamp,
) -> dict[str, float]:
    prefix = f"ip:{ip}"
    keys = {
        "requests": f"{prefix}:requests",
        "paths": f"{prefix}:paths",
        "statuses": f"{prefix}:statuses",
        ...
    }

    pipe = self._redis.pipeline()

    pipe.zadd(keys["requests"], {request_id: timestamp})
    pipe.zadd(keys["paths"], {_hash_member(path): timestamp})
    ...

    for key in keys.values():
        pipe.zremrangebyscore(key, "-inf", trim_boundary)

    pipe.zcount(keys["requests"], w1m, "+inf")
    pipe.zcount(keys["requests"], w5m, "+inf")
    ...

    results = await pipe.execute()
```

La pipeline impacchetta 30 comandi Redis in un singolo round-trip:
- 7 `ZADD` per registrare la nuova richiesta in tutti i sorted set
- 7 `ZREMRANGEBYSCORE` per eliminare le entry più vecchie di 15 minuti
- 5 `ZCOUNT` per i conteggi di richieste sulle finestre 1m/5m/10m e i conteggi di valori unici
- 4 `ZRANGEBYSCORE` per il recupero dettagliato dei membri (status, dimensioni, metodi, profondità)
- 7 `EXPIRE` per impostare il TTL su tutte le chiavi

**Perché l'hashing MD5 per alcuni membri:** i sorted set usano i membri come identificatori univoci. Per path e user agent vogliamo contare i valori unici. L'hashing fornisce un membro di dimensione fissa che deduplica efficacemente senza memorizzare le stringhe complete in Redis:

```python
def _hash_member(value: str) -> str:
    return hashlib.md5(value.encode(), usedforsecurity=False).hexdigest()[:16]
```

Il flag `usedforsecurity=False` evita i warning di conformità FIPS dato che usiamo MD5 per la deduplicazione, non per la sicurezza crittografica.

Le **statistiche sul tempo tra le richieste** catturano il pattern temporale delle richieste da un singolo IP:

```python
def _inter_request_time_stats(entries):
    if len(entries) < 2:
        return 0.0, 0.0
    timestamps = sorted(score for _, score in entries)
    deltas = [
        (timestamps[i + 1] - timestamps[i]) * 1000
        for i in range(len(timestamps) - 1)
    ]
    mean = sum(deltas) / len(deltas)
    if len(deltas) < 2:
        return mean, 0.0
    variance = sum((d - mean)**2 for d in deltas) / len(deltas)
    return mean, math.sqrt(variance)
```

Gli utenti legittimi hanno una spaziatura irregolare delle richieste (leggono pagine, cliccano link). Gli strumenti automatizzati hanno una spaziatura uniforme. Una bassa deviazione standard rispetto alla media è un'impronta da bot.

### Feature encoding

La funzione `encode_for_inference` in `encoder.py` trasforma il dizionario di feature a tipi misti in un vettore `float32` di 35 elementi:

```python
def encode_for_inference(features):
    vector: list[float] = []

    for name in FEATURE_ORDER:
        raw = features[name]

        if name in BOOLEAN_FEATURES:
            vector.append(1.0 if raw else 0.0)
        elif name in CATEGORICAL_ENCODERS:
            vector.append(float(CATEGORICAL_ENCODERS[name].get(str(raw), 0)))
        elif name == "country_code":
            vector.append(_encode_country(str(raw)))
        else:
            vector.append(float(raw))

    return vector
```

`FEATURE_ORDER` definisce l'ordinamento canonico. Questo ordinamento deve essere identico in fase di training e di inferenza. L'`InferenceEngine` lo valida controllando il campo `feature_names` in `scaler.json` rispetto a `FEATURE_ORDER` al momento del caricamento del modello.

I codici paese usano una codifica ordinale deterministica (A=1, Z=26, due caratteri -> da 1 a 676). Questo evita il one-hot encoding che farebbe esplodere la dimensione delle feature.

## Costruire il motore di detection

### Il rule engine

Il `RuleEngine` in `rules.py` valuta ogni richiesta rispetto a regole di pattern, soglie comportamentali e controlli ausiliari:

```python
class RuleEngine:
    def score_request(self, features, entry) -> RuleResult:
        matched: list[tuple[str, float]] = []

        uri = entry.path
        if entry.query_string:
            uri = f"{entry.path}?{entry.query_string}"

        for rule in _PATTERN_RULES:
            if rule.pattern.search(uri):
                matched.append((rule.name, rule.score))

        if DOUBLE_ENCODED.search(uri):
            matched.append(("DOUBLE_ENCODING", _DOUBLE_ENCODING_SCORE))

        ua_lower = entry.user_agent.lower()
        if any(sig in ua_lower for sig in SCANNER_USER_AGENTS):
            matched.append(("SCANNER_UA", _SCANNER_UA_SCORE))

        for trule in _THRESHOLD_RULES:
            value = features.get(trule.feature_key, 0)
            if isinstance(value, int | float) and value > trule.threshold:
                matched.append((trule.name, trule.score))
```

La logica di scoring applica un boost per le corrispondenze multi-regola:

```python
        scores = sorted([s for _, s in matched], reverse=True)
        threat_score = min(
            scores[0] + _BOOST_PER_ADDITIONAL_RULE * (len(scores) - 1),
            1.0,
        )
```

Una richiesta che corrisponde a SQL injection (0.85) + double encoding (0.40) + scanner UA (0.35) ottiene `0.85 + 0.05 * 2 = 0.95` perché il punteggio di regola più alto (0.85) riceve un boost di 0.05 per ogni corrispondenza aggiuntiva (2 regole in più = +0.10).

### Inferenza ONNX

L'`InferenceEngine` in `inference.py` carica tre sessioni ONNX e le esegue in sequenza su ogni batch:

```python
def predict(self, batch: np.ndarray) -> dict[str, list[float]] | None:
    if not self._loaded:
        return None

    ae_input = self._scale_for_ae(batch)
    ae_reconstructed = self._ae_session.run(None, {"features": ae_input})[0]
    ae_errors = np.mean((ae_input - ae_reconstructed)**2, axis=1)

    rf_result = self._rf_session.run(None, {"features": batch})
    rf_proba = self._extract_rf_proba(rf_result[1])

    if_scores_raw = self._if_session.run(
        None, {"features": batch}
    )[1].flatten()

    return {
        "ae": ae_errors.tolist(),
        "rf": rf_proba.tolist(),
        "if": if_scores_raw.tolist(),
    }
```

**Perché `_scale_for_ae` si applica solo all'autoencoder:** l'autoencoder è stato addestrato su dati normalizzati con RobustScaler. I parametri dello scaler (array center e scale) sono salvati accanto al modello in `scaler.json` e applicati al momento dell'inferenza. Il random forest e l'isolation forest sono stati addestrati su vettori di feature grezzi, quindi ricevono il batch non scalato.

**Perché sessioni ONNX single-thread:** il backend gira sull'event loop single-thread di asyncio. Se ONNX Runtime avvia un proprio thread pool, quei thread competono con l'event loop per la CPU. Impostare `inter_op_num_threads=1` e `intra_op_num_threads=1` mantiene tutto su un solo core ed evita la contesa.

### Score blending nella pipeline

Il detection worker in `pipeline.py` lega insieme regole e ML:

```python
async def _detection_worker(self) -> None:
    while True:
        enriched = await self._feature_queue.get()
        if enriched is None:
            ...
            break
        try:
            rule_result = self._rule_engine.score_request(
                enriched.features, enriched.entry,
            )

            final_score = rule_result.threat_score
            detection_mode = "rules"
            per_model_scores = None

            if (self._inference_engine is not None
                    and self._inference_engine.is_loaded
                    and np is not None):
                per_model_scores = self._score_with_ml(
                    enriched.feature_vector,
                )
                if per_model_scores is not None:
                    ml_fused = fuse_scores(
                        per_model_scores, self._ensemble_weights,
                    )
                    final_score = blend_scores(
                        ml_fused, rule_result.threat_score,
                    )
                    detection_mode = "hybrid"
```

Quando i modelli ML sono disponibili, il flusso è: punteggio delle regole + punteggio fuso dell'ensemble ML -> punteggio finale miscelato. Quando l'ML non è disponibile (cold start), il punteggio delle regole diventa direttamente il punteggio finale. Il campo `detection_mode` su `ScoredRequest` tiene traccia del percorso seguito.

## Costruire la pipeline di training ML

### Il training orchestrator

`TrainingOrchestrator.run()` in `orchestrator.py` esegue l'intera pipeline:

```python
def run(self, X: np.ndarray, y: np.ndarray) -> TrainingResult:
    self._output_dir.mkdir(parents=True, exist_ok=True)

    split = prepare_training_data(X, y)

    with VigilExperiment(self._experiment_name) as experiment:
        experiment.log_params({
            "epochs": self._epochs,
            "batch_size": self._batch_size,
            "n_samples": len(X),
            "n_attack": int(np.sum(y == 1)),
            "n_normal": int(np.sum(y == 0)),
            "n_features": X.shape[1],
        })

        ae_result = self._train_ae(split.X_normal_train)
        rf_result = self._train_rf(split.X_train, split.y_train)
        if_result = self._train_if(split.X_normal_train)

        self._export_models(ae_result, rf_result, if_result)
        ...
```

Nota i dati di training usati da ciascun modello:
- Autoencoder: `X_normal_train` (solo traffico normale, nessun attacco)
- Random Forest: `X_train, y_train` (mix etichettato di normale + attacco, con SMOTE)
- Isolation Forest: `X_normal_train` (solo traffico normale)

L'autoencoder e l'isolation forest imparano che aspetto ha il "normale". Il random forest impara a distinguere il normale dall'attacco. Questo mix di approcci supervisionati e non supervisionati è il cuore della strategia ensemble.

### Training dell'autoencoder

Il training loop in `train_autoencoder.py` usa pattern PyTorch standard con alcune scelte specifiche:

```python
model = ThreatAutoencoder(input_dim=input_dim)
optimizer = torch.optim.AdamW(
    model.parameters(), lr=lr, weight_decay=1e-5, betas=(0.9, 0.999)
)
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, mode="min", factor=0.5, patience=5, min_lr=1e-6
)
```

- **AdamW** invece del semplice Adam: la regolarizzazione tramite weight decay aiuta a impedire che l'autoencoder memorizzi i dati di training. Vogliamo che impari pattern generali del traffico normale, non richieste specifiche.
- **ReduceLROnPlateau**: dimezza il learning rate quando la validation loss ristagna per 5 epoch. Questo previene le oscillazioni attorno a un minimo.
- **Gradient clipping** a `max_norm=1.0`: i tipi di feature misti (alcune 0-1, altre 0-100000) possono causare esplosioni del gradiente. Il clipping stabilizza il training.
- **Early stopping** con patience 10: interrompe il training quando la validation loss non migliora da 10 epoch. Previene l'overfitting.

La calibrazione della soglia avviene dopo il completamento del training:

```python
model.eval()
with torch.no_grad():
    val_errors = model.compute_reconstruction_error(val_tensor)
threshold = float(np.percentile(val_errors.numpy(), percentile))
```

Il 99.5° percentile degli errori di ricostruzione sul validation set diventa la soglia di anomalia. Questo significa che circa lo 0.5% del traffico normale verrà segnalato come anomalo, che è il tasso di falsi positivi di baseline.

### Validazione dell'ensemble e quality gate

Dopo il training, `validate_ensemble` in `validation.py` testa l'intero ensemble su un test set tenuto da parte:

```python
def validate_ensemble(
    model_dir, X_test, y_test,
    ensemble_weights=None, pr_auc_gate=0.85, f1_gate=0.80,
) -> ValidationResult:
    engine = InferenceEngine(model_dir=str(model_dir))
    raw_scores = engine.predict(X_test.astype(np.float32))

    fused = _compute_fused_scores(raw_scores, engine.threshold, weights)
    y_pred = (fused >= BINARY_THRESHOLD).astype(np.int32)

    prec = float(precision_score(y_test, y_pred, zero_division=0))
    rec = float(recall_score(y_test, y_pred, zero_division=0))
    f1_val = float(f1_score(y_test, y_pred, zero_division=0))
    pr_auc_val = float(average_precision_score(y_test, fused))
    ...
```

**Perché PR-AUC invece di ROC-AUC come gate primario:** nella threat detection la distribuzione delle classi è fortemente sbilanciata (1-5% di attacchi contro 95-99% di normale). La ROC-AUC può sembrare ottima anche quando il modello ha un alto tasso di falsi positivi perché l'enorme numero di true negative gonfia la metrica. La PR-AUC si concentra su precision e recall per la classe positiva (attacchi), rendendola una metrica più onesta per i problemi di detection sbilanciati.

I quality gate (PR-AUC >= 0.85, F1 >= 0.80) impediscono il deploy di modelli che inonderebbero gli analisti di falsi positivi. Se uno dei due gate fallisce, `passed_gates` è `False` e l'endpoint di retrain riporta il fallimento senza sostituire i modelli.

## Costruire il sistema di alert

### Dispatcher

L'`AlertDispatcher` in `dispatcher.py` riceve oggetti `ScoredRequest` dallo stadio di dispatch della pipeline:

```python
async def dispatch(self, scored: ScoredRequest) -> None:
    severity = classify_severity(scored.final_score)

    logger.info(
        "threat_event severity=%s score=%.2f mode=%s ip=%s path=%s rules=%s",
        severity, scored.final_score, scored.detection_mode,
        scored.entry.ip, scored.entry.path, scored.rule_result.matched_rules,
    )

    if severity in ("HIGH", "MEDIUM"):
        await self._store_event(scored)
        await self._publish_alert(scored, severity)
```

Ogni evento viene loggato (JSON strutturato su stdout). Solo gli eventi di severity MEDIUM e HIGH vengono memorizzati in PostgreSQL e pubblicati sul canale WebSocket. Questo mantiene limitato lo storage garantendo al contempo agli analisti il contesto completo per le minacce non banali.

Il publish pub/sub invia un payload JSON serializzato da Pydantic:

```python
async def _publish_alert(self, scored, severity):
    alert = WebSocketAlert(
        timestamp=scored.entry.timestamp,
        source_ip=scored.entry.ip,
        request_method=scored.entry.method,
        request_path=scored.entry.path,
        threat_score=scored.final_score,
        severity=severity,
        component_scores={
            **scored.rule_result.component_scores,
            **(scored.ml_scores or {}),
        },
    )
    await self._redis.publish(ALERTS_CHANNEL, alert.model_dump_json())
```

### Relay WebSocket sul frontend

L'hook `useAlerts` in `useAlerts.ts` gestisce la connessione WebSocket con stato Zustand e riconnessione con exponential backoff:

```typescript
function connect() {
  const ws = new WebSocket(getWsUrl())
  wsRef.current = ws

  ws.onmessage = (event) => {
    const parsed = WebSocketAlertSchema.safeParse(JSON.parse(event.data))
    if (parsed.success) {
      addAlert({ ...parsed.data, id: crypto.randomUUID() })
    }
  }

  ws.onclose = () => {
    setConnected(false)
    scheduleReconnect()
  }
}

function scheduleReconnect() {
  const delay = Math.min(
    ALERTS.RECONNECT_BASE_MS * 2 ** retryCountRef.current,
    ALERTS.RECONNECT_MAX_MS
  )
  retryCountRef.current += 1
  retryTimerRef.current = setTimeout(connect, delay)
}
```

**Perché la validazione con Zod:** il WebSocket riceve JSON grezzo dal server. `safeParse` valida che la forma corrisponda a `WebSocketAlertSchema` prima di aggiungerlo allo stato. Questo impedisce che messaggi corrotti o inattesi mandino in crash la UI.

**Perché un ring buffer:** l'alert store ha un tetto a `ALERTS.MAX_ITEMS` (100). I nuovi alert vengono anteposti e i vecchi scartati. Questo limita l'uso di memoria e mantiene il feed concentrato sull'attività recente.

**Perché l'exponential backoff:** se il backend si riavvia o la rete ha un singhiozzo, la connessione WebSocket cade. L'exponential backoff (500ms, 1s, 2s, 4s, ... con tetto a 30s) evita di martellare il server durante un'interruzione pur recuperando rapidamente dalle brevi interruzioni.

## Ciclo di vita dell'applicazione

Il lifespan in `factory.py` gestisce startup e shutdown:

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    engine = create_async_engine(settings.database_url)
    app.state.session_factory = async_sessionmaker(engine, ...)

    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)

    await redis_manager.connect()
    geoip = GeoIPService(settings.geoip_db_path)

    dispatcher = AlertDispatcher(
        redis_client=redis_manager.client,
        session_factory=app.state.session_factory,
    )

    inference_engine = _load_inference_engine()
    app.state.detection_mode = "hybrid" if inference_engine else "rules"

    pipeline = Pipeline(
        redis_client=redis_client, rule_engine=RuleEngine(),
        geoip=geoip, on_result=dispatcher.dispatch,
        inference_engine=inference_engine, ...
    )
    await pipeline.start()

    tailer = LogTailer(settings.nginx_log_path, pipeline.raw_queue, loop)
    tailer.start()

    yield  # Application runs here

    tailer.stop()
    await pipeline.stop()
    geoip.close()
    await redis_manager.disconnect()
    await engine.dispose()
```

L'ordine di avvio conta: prima il database (creazione dello schema), poi Redis (la pipeline ne dipende), poi GeoIP, poi la pipeline (dipende da tutti e tre), poi il tailer (alimenta la pipeline). Lo shutdown inverte l'ordine: ferma l'input (tailer), svuota la pipeline, chiude le connessioni.

La funzione `_load_inference_engine` tenta di caricare i modelli ONNX e restituisce `None` se non esistono o se onnxruntime non è installato. Questo fallback elegante è ciò che fa funzionare la modalità solo-regole nei deployment nuovi.

## Gestione della configurazione

Pydantic Settings in `config.py` carica da variabili d'ambiente e `.env`:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    ensemble_weight_ae: float = 0.40
    ensemble_weight_rf: float = 0.40
    ensemble_weight_if: float = 0.20

    @model_validator(mode="after")
    def _check_ensemble_weights(self) -> Self:
        total = (
            self.ensemble_weight_ae
            + self.ensemble_weight_rf
            + self.ensemble_weight_if
        )
        if abs(total - 1.0) > 1e-6:
            raise ValueError(
                f"Ensemble weights must sum to 1.0, got {total:.6f}"
            )
        return self
```

Il `model_validator` intercetta le errate configurazioni all'avvio. Se qualcuno imposta i pesi a 0.4/0.4/0.3 (somma 1.1), l'applicazione fallisce immediatamente con un errore chiaro invece di calcolare silenziosamente punteggi sbagliati.

## Strategia di testing

### Unit test

I test del parser verificano sia il percorso veloce sia quello di fallback:

```python
def test_parse_combined_standard_line():
    line = '93.184.216.34 - - [15/Mar/2026:14:22:31 +0000] "GET /api/users HTTP/1.1" 200 1234 "-" "Mozilla/5.0"'
    entry = parse_combined(line)
    assert entry is not None
    assert entry.ip == "93.184.216.34"
    assert entry.method == "GET"
    assert entry.path == "/api/users"
    assert entry.status_code == 200
```

I test di feature extraction verificano segnali di detection specifici:

```python
def test_sqli_payload_detected():
    entry = make_entry(path="/search", query_string="q=1' OR '1'='1")
    features = extract_request_features(entry)
    assert features["has_attack_pattern"] is True
    assert features["query_string_length"] > 0
```

### Test di integrazione

I test end-to-end della pipeline fanno passare una riga di log attraverso tutti e quattro gli stadi e verificano l'output:

```python
async def test_pipeline_processes_log_line():
    pipeline = Pipeline(redis_client=mock_redis, rule_engine=RuleEngine())
    results = []
    pipeline._on_result = lambda scored: results.append(scored)
    await pipeline.start()

    await pipeline.raw_queue.put(SAMPLE_LOG_LINE)
    await pipeline.raw_queue.put(None)
    await asyncio.gather(*pipeline._tasks)

    assert len(results) == 1
    assert results[0].final_score >= 0.0
```

### Eseguire i test

```bash
just test            # Run full suite
just test-v          # Verbose output
just test-cov        # With coverage report
```

## Errori comuni di implementazione

### Errore 1: dimenticare di normalizzare i punteggi ML

**Sintomo:**
I punteggi dell'autoencoder sono 0.001-0.05 mentre i punteggi del random forest sono 0.0-1.0. La fusione dell'ensemble produce punteggi dominati dall'RF perché i punteggi dell'AE sono minuscoli.

**Causa:**
I punteggi grezzi dell'autoencoder sono errori di ricostruzione (MSE), non probabilità. Hanno bisogno di essere normalizzati rispetto alla soglia calibrata.

**Soluzione:**
Normalizza sempre prima di fondere:
```python
per_model["ae"] = normalize_ae_score(raw["ae"][0], engine.threshold)
per_model["rf"] = raw["rf"][0]  # Already a probability
per_model["if"] = normalize_if_score(raw["if"][0])
```

### Errore 2: disallineamento dell'ordine delle feature

**Sintomo:**
Il modello produce punteggi privi di senso. Una richiesta chiaramente benigna ottiene 0.95, un'ovvia SQL injection ottiene 0.02.

**Causa:**
Il vettore di feature al momento dell'inferenza è ordinato diversamente rispetto al training. La feature 0 al training era `http_method`, ma all'inferenza è `path_depth`.

**Soluzione:**
Sia il training sia l'inferenza devono usare `FEATURE_ORDER` da `mappings.py`. L'inference engine lo valida al caricamento controllando l'array `feature_names` in `scaler.json`.

### Errore 3: crescita eccessiva delle chiavi Redis

**Sintomo:**
L'uso di memoria di Redis cresce indefinitamente. Il comando `INFO memory` mostra un `used_memory` in aumento.

**Causa:**
I comandi `EXPIRE` nella pipeline dell'aggregator falliscono silenziosamente, oppure il calcolo del trim boundary è sbagliato, quindi le vecchie entry non vengono mai ripulite.

**Soluzione:**
Verifica che `KEY_TTL = 900` (15 minuti) sia impostato e che `ZREMRANGEBYSCORE` stia eliminando le entry più vecchie della finestra. Controlla il numero di chiavi Redis con `DBSIZE` e ispeziona le singole chiavi con `ZCARD`.

## Suggerimenti per il debug

### La pipeline non elabora i log

**Problema:** la dashboard non mostra attività anche se nginx sta scrivendo i log.

**Come fare debug:**
1. Controlla il tailer: `GET /health` restituisce `pipeline_running: true` e statistiche con `parsed > 0`
2. Controlla che il file di log esista: `docker compose exec backend ls -la /var/log/nginx/access.log`
3. Controlla i permessi del tailer: il container backend deve avere accesso in lettura al volume dei log di nginx
4. Controlla la connettività di Redis: se Redis è giù, il feature worker fallisce silenziosamente e scarta le richieste

### Il training dei modelli fallisce i quality gate

**Problema:** il training si completa ma `passed_gates: false`. L'endpoint di retrain riporta un fallimento.

**Come fare debug:**
1. Controlla le metriche MLflow: guarda `pr_auc` e `f1` nell'output del training
2. Controlla la distribuzione delle classi: se i dati di training sono al 99% normali con pochissimi campioni di attacco, il modello non può imparare a distinguere. Integra con dati di attacco sintetici
3. Controlla la qualità delle feature: se tutte le feature sono zero o costanti, i modelli non hanno nulla da cui imparare. Verifica che la pipeline di feature extraction produca valori non banali

### Gli alert WebSocket non raggiungono la dashboard

**Problema:** il backend logga gli eventi di minaccia ma il feed degli alert nella dashboard è vuoto.

**Come fare debug:**
1. Controlla il pub/sub di Redis: `redis-cli SUBSCRIBE alerts` in un terminale separato dovrebbe mostrare messaggi quando vengono rilevate minacce
2. Controlla la connessione WebSocket: la scheda Network dei dev tools del browser dovrebbe mostrare una connessione WebSocket verso `/ws/alerts`
3. Controlla lo stato della connessione: l'hook `useAlerts` espone `connectionError` che la dashboard mostra come banner

## Build e deploy

### Build

```bash
just build            # Build production Docker images
just rebuild          # Force rebuild without cache
```

### Sviluppo locale

```bash
just dev-up           # Start dev stack (hot-reload enabled)
just dev-logs backend # Follow backend logs
just devlog-up        # Start the synthetic traffic generator
```

### Deploy in produzione

```bash
# Set required environment variables
export POSTGRES_PASSWORD=<strong-password>
export API_KEY=<generated-api-key>
export GEOIP_ACCOUNT_ID=<maxmind-account>
export GEOIP_LICENSE_KEY=<maxmind-key>

just start            # Start production stack (detached)
just logs             # Follow all service logs
```

Differenze principali rispetto al dev:
- Nessun hot-reload (uvicorn gira senza `--reload`)
- Build Docker multi-stage (immagini più piccole, nessuna dipendenza di sviluppo)
- Periodo di avvio dell'healthcheck più lungo (180s per il training iniziale dei modelli)
- Mount esterno del volume dei log di nginx (non il generatore dev-log)

## Prossimi passi

Hai visto come funziona il codice. Ora:

1. **Prova le challenge** - [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) contiene idee di estensione da facili a esperte
2. **Modifica i pesi dell'ensemble** - Cambia `ENSEMBLE_WEIGHT_AE/RF/IF` in `.env` e osserva come cambia il comportamento della detection sulla dashboard
3. **Aggiungi una nuova regola di detection** - Aggiungi una entry a `_PATTERN_RULES` in `rules.py` e testala con il simulatore dev-log
