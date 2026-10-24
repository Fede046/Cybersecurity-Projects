# Sfide di estensione

Hai costruito il progetto di base. Ora fallo tuo estendendolo con nuove funzionalità.

Queste sfide sono ordinate per difficoltà. Parti dalle più facili per costruire sicurezza, poi affronta quelle più difficili quando vuoi approfondire.

## Sfide facili

### Sfida 1: Aggiungere una nuova regola di detection

**Cosa costruire:**
Aggiungi una regola di detection per gli attacchi di XML External Entity (XXE) injection. I payload XXE contengono stringhe come `<!ENTITY`, `<!DOCTYPE`, `SYSTEM "file:///` e `xmlns:xi` nei parametri URL o nei percorsi.

**Perché è utile:**
XXE (CWE-611) era nella OWASP Top 10 fino al 2021, quando è stata accorpata in A05:2021 Security Misconfiguration. La vulnerabilità XXE di Facebook del 2014 permetteva di leggere file arbitrari dai loro server. Qualsiasi applicazione che esegue il parsing di input XML da parte degli utenti è potenzialmente vulnerabile.

**Cosa imparerai:**
- Come funziona il pattern matching del rule engine
- Scrivere pattern regex che intercettino le varianti di evasione
- Calibrare i punteggi delle regole rispetto a quelle esistenti

**Suggerimenti:**
- Aggiungi una regex compilata a `patterns.py` che copra i payload XXE più comuni
- Aggiungi una voce `_PatternRule` a `_PATTERN_RULES` in `rules.py` con un punteggio appropriato (pensa a dove si colloca XXE tra FILE_INCLUSION a 0.75 e COMMAND_INJECTION a 0.90)
- Testa con il simulatore dev-log: modifica `simulate.py` per generare payload XXE

**Verifica che funzioni:**
```bash
curl "http://localhost:8000/api/test?xml=%3C!DOCTYPE%20foo%20%5B%3C!ENTITY%20xxe%20SYSTEM%20%22file%3A%2F%2F%2Fetc%2Fpasswd%22%3E%5D%3E"
```
Controlla nella dashboard la presenza di un nuovo threat event con il nome della tua regola nel campo matched_rules.

### Sfida 2: Aggiungere la detection delle anomalie nei metodi di richiesta

**Cosa costruire:**
Aggiungi una threshold rule che scatta quando un IP invia un mix insolito di metodi HTTP. Gli utenti normali inviano prevalentemente richieste GET con qualche POST occasionale. Un IP che invia molte richieste PUT, DELETE, PATCH o OPTIONS nell'arco di 5 minuti sta probabilmente sondando l'API alla ricerca di endpoint mal configurati.

**Perché è utile:**
Gli attacchi di enumerazione delle API usano spesso metodi HTTP poco comuni per scoprire endpoint che rispondono in modo diverso (restituendo 200 invece di 405). La violazione di First American Financial del 2019 è stata resa possibile in parte da vulnerabilità di direct object reference scoperte tramite il probing delle API.

**Cosa imparerai:**
- Come le threshold rule usano le feature a finestra
- Lavorare con i dati Redis di `WindowAggregator`
- La differenza tra pattern rule (regex sull'URI) e threshold rule (condizioni numeriche sulle feature)

**Suggerimenti:**
- `method_entropy_5m` è già calcolata dall'aggregator. Un'entropia superiore a 1.5 (più di 3 metodi distinti con frequenza più o meno uguale) è sospetta per la maggior parte delle applicazioni web
- Aggiungi una `_ThresholdRule` in `rules.py` che abbia come target `method_entropy_5m`
- Assegnale un punteggio intorno a 0.25-0.35, dato che è un segnale comportamentale con potenziale di falsi positivi sui client API legittimi

**Verifica che funzioni:**
Usa il simulatore dev-log per inviare un mix di GET/POST/PUT/DELETE/OPTIONS da un singolo IP e verifica che la regola scatti.

### Sfida 3: Aggiungere una funzionalità di blocklist per paese

**Cosa costruire:**
Aggiungi una lista configurabile di codici paese che aumenti automaticamente il threat score delle richieste provenienti da quei paesi. Molte organizzazioni limitano il traffico proveniente da regioni in cui non hanno clienti.

**Perché è utile:**
Il geo-blocking è un comune livello di difesa. La feature `country_code` esiste già nella pipeline. Questa sfida la fa passare da feature passiva (usata dai modelli ML) a regola attiva (che influenza direttamente i punteggi).

**Cosa imparerai:**
- Come l'enrichment GeoIP alimenta la pipeline di detection
- Aggiungere parametri configurabili tramite Pydantic Settings
- La tensione tra blocco geografico e falsi positivi (VPN, CDN, dipendenti in viaggio)

**Suggerimenti:**
- Aggiungi un'impostazione `blocked_countries` in `config.py` come stringa separata da virgole, convertita in un `set[str]`
- Aggiungi un controllo in `RuleEngine.score_request` che confronti la feature `country_code` con la blocklist
- Assegnale un punteggio intorno a 0.20-0.30, dato che la sola origine geografica non è un indicatore forte

**Verifica che funzioni:**
Imposta `BLOCKED_COUNTRIES=CN,RU` in `.env`, riavvia e verifica che le richieste provenienti da quei codici paese ottengano un aumento del punteggio.

## Sfide intermedie

### Sfida 4: Costruire un ciclo di feedback con active learning

**Cosa costruire:**
Aggiungi endpoint API perché gli analisti possano contrassegnare i threat event come veri positivi o falsi positivi, e usa quelle etichette per avviare automaticamente il retraining quando si accumulano abbastanza nuove etichette.

**Approccio di implementazione:**

1. **Aggiungi endpoint di review** per gestire il feedback degli analisti
   - `PATCH /threats/{id}/review` con body `{"label": "true_positive"}` oppure `{"label": "false_positive"}`
   - Memorizza l'etichetta in `ThreatEvent.review_label` e imposta `reviewed = True`
   - File da modificare: `threats.py`, `threat_service.py`

2. **Aggiungi un trigger automatico di retraining**
   - Tieni il conteggio delle nuove etichette dall'ultimo addestramento
   - Quando il conteggio supera una soglia (ad es. 50 nuove etichette), avvia automaticamente il retraining
   - File da creare: un task in background che controlla periodicamente il conteggio delle etichette

3. **Usa le etichette come dati di training**
   - Modifica l'endpoint di retrain in modo che usi `review_label` come ground truth invece dell'etichettatura basata sul punteggio
   - Gli eventi `true_positive` diventano campioni di attacco, gli eventi `false_positive` diventano campioni normali
   - File da modificare: `models_api.py`

**Cosa imparerai:**
- Active learning: usare il feedback umano per migliorare iterativamente i modelli ML
- Il problema del cold start: le etichette iniziali sono scarse, quindi il sistema deve lavorare con etichette deboli (soglie di punteggio) finché gli analisti non forniscono abbastanza etichette reali
- Come la riduzione dei falsi positivi incide direttamente sulla fiducia degli analisti e sull'alert fatigue

**Suggerimenti:**
- I campi `reviewed` e `review_label` esistono già su `ThreatEvent`. L'infrastruttura c'è, devi solo collegarla
- Valuta di aggiungere un contatore `labels_since_last_train` a `ModelMetadata`
- Attenzione al bilanciamento delle classi: se gli analisti etichettano solo i veri positivi più ovvi, i dati di retraining saranno distorti

**Punti extra:**
Costruisci una semplice pagina di "review queue" nel frontend che mostri gli eventi di severity MEDIUM non ancora esaminati (i più ambigui) e permetta agli analisti di cliccare i pulsanti "True Positive" o "False Positive".

### Sfida 5: Aggiungere l'analisi del body delle richieste tramite i log degli errori

**Cosa costruire:**
Estendi il tailer in modo che osservi anche i log degli errori di nginx, che contengono informazioni sul body delle richieste fallite. Esegui il parsing del formato del log degli errori ed estrai le feature dai body delle POST.

**Applicazione nel mondo reale:**
Molti attacchi di injection (SQL injection, XXE, deserialization) vengono veicolati nei body delle richieste POST. L'access log mostra solo metodo e percorso. I log degli errori spesso includono il payload effettivo che ha causato l'errore, dando al sistema di detection visibilità sul contenuto dell'attacco.

**Cosa imparerai:**
- Il formato del log degli errori di nginx (diverso dal formato combined dell'access log)
- Estendere il tailer per osservare più file
- L'estrazione di feature da testo non strutturato (i body delle richieste contengono contenuto arbitrario)
- Come la violazione di Equifax del 2017 (CVE-2017-5638) usò un payload in un header Content-Type che sarebbe stato visibile nei log degli errori ma non negli access log

**Approccio di implementazione:**

1. **Estendi il tailer**
   - Aggiungi una seconda istanza di `_LogHandler` che osservi il log degli errori di nginx
   - Entrambi gli handler inviano alla stessa `raw_queue` con un prefisso di riga che indica la sorgente (`[access]` vs `[error]`)

2. **Aggiungi un parser per il log degli errori**
   - Esegui il parsing del formato del log degli errori di nginx per estrarre timestamp, livello di errore, IP del client e messaggio di errore (che spesso contiene il body della richiesta)
   - Crea una nuova dataclass `ParsedErrorEntry`

3. **Estrai le feature del body**
   - Calcola entropia, lunghezza e corrispondenze con i pattern di attacco sul testo del body estratto
   - Aggiungile come feature aggiuntive (espandendo il vettore oltre le 35 dimensioni, il che significa fare retraining)

**Suggerimenti:**
- formato del log degli errori di nginx: `2026/03/15 09:22:31 [error] 7#7: *1234 upstream prematurely closed connection while reading response header from upstream, client: 93.184.216.34`
- La parte difficile è che i log degli errori sono molto meno strutturati degli access log. Ti servirà un parsing robusto per i vari formati dei messaggi di errore
- Valuta di mantenere le feature dell'access log e quelle del log degli errori come set di feature separati che vengono concatenati

**Punti extra:**
Aggiungi una feature che correli le voci dell'access log con quelle del log degli errori per timestamp e IP, collegando il contesto completo della richiesta (dall'access log) con il dettaglio dell'errore (dal log degli errori).

### Sfida 6: Aggiungere l'export delle metriche Prometheus

**Cosa costruire:**
Aggiungi un endpoint `/metrics` che esporti statistiche della pipeline, metriche di detection e stato di salute del sistema nel formato di esposizione Prometheus.

**Perché è utile:**
I sistemi di threat detection in produzione hanno bisogno di monitoraggio. Vuoi sapere: quante richieste al secondo sta elaborando la pipeline? Qual è il tasso di detection per severity? Qualche coda si sta accumulando? Quanto è veloce l'inferenza?

**Cosa imparerai:**
- I tipi di metriche Prometheus (counter, gauge, histogram)
- Instrumentare codice Python asincrono senza aggiungere latenza all'hot path
- La differenza tra monitorare il sistema monitorato (metriche delle minacce) e monitorare il sistema di monitoraggio (salute della pipeline)

**Approccio di implementazione:**

1. **Aggiungi la dipendenza prometheus-client**
2. **Instrumenta la pipeline** con counter e histogram
   - `vigil_requests_processed_total` (counter per stadio)
   - `vigil_threats_detected_total` (counter per severity)
   - `vigil_inference_duration_seconds` (histogram)
   - `vigil_queue_depth` (gauge per coda)
3. **Aggiungi l'endpoint `/metrics`** usando `generate_latest()`
4. **Aggiungi una dashboard Grafana** in JSON in `infra/grafana/`

**Suggerimenti:**
- Usa `time.perf_counter()` attorno alla chiamata di inferenza ONNX per misurare la latenza
- Le profondità delle code possono essere campionate periodicamente (ogni 5 secondi) tramite un task in background invece di instrumentare ogni put/get
- Il dizionario `stats` della pipeline traccia già i conteggi di elaborati/errori. Esponili come counter Prometheus

## Sfide avanzate

### Sfida 7: Aggiungere un quarto modello (Transformer)

**Cosa costruire:**
Aggiungi all'ensemble un piccolo modello transformer encoder che elabori sequenze di richieste provenienti dallo stesso IP (non solo richieste singole). Questo modello vede l'ordinamento temporale delle richieste, cosa che gli altri tre modelli non fanno.

**Perché è difficile:**
I modelli attuali elaborano una richiesta alla volta. Un transformer ha bisogno come input di una sequenza di richieste recenti dallo stesso IP. Questo significa mettere in buffer le richieste recenti per singolo IP, fare padding/troncamento a una lunghezza di sequenza fissa e calcolare l'attention lungo la sequenza. Anche la pipeline dei dati di training deve generare sequenze, non campioni singoli.

**Cosa imparerai:**
- L'architettura transformer per l'anomaly detection sequenziale
- I meccanismi di attention e cosa imparano sulle sequenze di richieste
- Integrare un modello di sequenza con modelli per singola richiesta in un ensemble
- Come gestire sequenze di lunghezza variabile (padding, masking)

**Modifiche architetturali necessarie:**

```
                        ┌───────────────────┐
                        │  Existing Models   │
                        │  (per-request)     │
Feature Vector ────────►│  AE + RF + IF      ├───┐
                        └───────────────────┘   │
                                                 │
                        ┌───────────────────┐   ├──► Ensemble
Recent N requests ─────►│  Transformer       │───┘    Fusion
 (from same IP)         │  (per-sequence)    │
                        └───────────────────┘
```

**Passi di implementazione:**

1. **Metti in buffer le richieste recenti per singolo IP** in Redis o in memoria
   - Memorizza gli ultimi 16 vettori di feature per IP
   - Fai padding con zeri se ne sono disponibili meno di 16

2. **Costruisci il modello transformer**
   - Piccolo modello solo encoder: 2 layer, 4 attention head, dimensione di embedding 64
   - Input: (batch, seq_len=16, features=35)
   - Output: anomaly score (un singolo float)
   - Addestra su sequenze di traffico normale, segnala le sequenze che contengono anomalie

3. **Integra con l'ensemble**
   - Aggiungi un quarto peso (ad es. AE 0.30, RF 0.30, IF 0.15, Transformer 0.25)
   - Esporta in ONNX insieme agli altri modelli

4. **Aggiorna la pipeline di training**
   - Genera campioni di sequenza dai dati di training
   - Gestisci il cold start (meno di 16 richieste da un nuovo IP)

**Insidie:**
- La forma dell'input del transformer è (batch, seq, features), non (batch, features). L'export ONNX gestisce la cosa, ma l'inferenza in batch nella pipeline va aggiornata
- L'attention può essere costosa. Profila con attenzione la latenza di inferenza. Un transformer a 2 layer con lunghezza di sequenza 16 dovrebbe essere gestibile (< 5ms per batch da 32)

**Risorse:**
- Vaswani et al., "Attention Is All You Need" (2017), Sezione 3 per l'architettura dell'encoder
- Documentazione di PyTorch `nn.TransformerEncoder` per i dettagli di implementazione

### Sfida 8: Costruire una pipeline distribuita con Kafka

**Cosa costruire:**
Sostituisci la pipeline basata su `asyncio.Queue` con topic Apache Kafka. Ogni stadio della pipeline diventa un consumer group che legge da un topic e produce sul successivo. Questo rimuove il limite di throughput del singolo processo.

**Perché è difficile:**
Kafka cambia in modo fondamentale il modello di concorrenza. Attualmente un solo processo Python gestisce tutto in sequenza. Con Kafka puoi eseguire più worker di parsing, worker di feature e worker di detection in modo indipendente. Ma questo introduce sfide: ordinamento dei messaggi, semantica exactly-once, assegnazione delle partizioni, monitoraggio del consumer lag ed evoluzione dello schema.

**Cosa imparerai:**
- I pattern producer/consumer di Kafka con aiokafka
- Strategie di partizionamento (partizionare per IP sorgente per le feature a finestra)
- Consumer group e rebalancing
- Lo schema registry per far evolvere i formati dei messaggi
- Come sistemi di produzione come la pipeline WAF di Cloudflare gestiscono milioni di richieste al secondo

**Fasi di implementazione:**

**Fase 1: Progettazione dei topic**
- `vigil.raw` - Righe di log grezze (partizionate per hash dell'IP sorgente)
- `vigil.parsed` - Voci di log già parsate (stessa chiave di partizione)
- `vigil.enriched` - Vettori di feature arricchiti
- `vigil.scored` - Risultati finali con punteggio pronti per il dispatch

**Fase 2: Sostituire le code con Kafka**
- Sostituisci `asyncio.Queue.put/get` con produce/consume di Kafka
- Usa Avro o JSON Schema per la serializzazione dei messaggi
- Mantieni l'affinità di partizione per IP sorgente così che le feature a finestra restino coerenti

**Fase 3: Scalare orizzontalmente**
- Esegui 4 worker di parsing, 2 worker di feature, 2 worker di detection
- Monitora il consumer lag per individuare i colli di bottiglia
- Aggiungi l'auto-scaling basato su soglie di lag

**Fase 4: Osservabilità**
- Aggiungi le metriche Kafka all'exporter Prometheus
- Monitora consumer lag, latenza di produzione e distribuzione delle partizioni
- Costruisci una dashboard Grafana che mostri il throughput della pipeline per stadio

**Sfide note:**
1. **Le feature a finestra richiedono affinità di partizione** - Tutte le richieste dallo stesso IP devono andare allo stesso worker di feature perché i sorted set di Redis siano coerenti. Usa l'IP sorgente come chiave di partizione Kafka.
2. **Semantica exactly-once** - Se un worker di detection va in crash a metà elaborazione, il messaggio deve essere rielaborato, non perso. Usa i consumer offset di Kafka con commit manuale dopo l'elaborazione riuscita.

**Criteri di successo:**
- [ ] La pipeline gestisce 5000+ req/s in modo sostenuto (10x l'attuale limite)
- [ ] Aggiungere/rimuovere worker non causa perdita di dati
- [ ] Il consumer lag resta sotto i 1000 messaggi durante il normale funzionamento
- [ ] L'affinità di partizione garantisce che le feature a finestra siano corrette

## Sfide da esperti

### Sfida 9: Costruire un'integrazione SOAR completa

**Cosa costruire:**
Costruisci un livello di Security Orchestration, Automation, and Response (SOAR) che risponda automaticamente alle minacce rilevate. Gli eventi di severity HIGH dovrebbero innescare risposte automatiche: aggiungere l'IP sorgente a una blocklist di nginx, inviare una notifica Slack/Discord, creare un ticket in un sistema di tracciamento e arricchire l'alert con threat intelligence da feed pubblici.

**Prerequisiti:**
Dovresti aver completato prima la Sfida 4 (active learning) e la Sfida 6 (metriche Prometheus) perché questa si basa sui workflow di feedback degli analisti e sull'infrastruttura di monitoraggio.

**Cosa imparerai:**
- I pattern di automazione SOAR usati nei SOC enterprise
- Incident response guidata da playbook (automazione if-this-then-that)
- Integrazione di threat intelligence (AbuseIPDB, VirusTotal, Shodan)
- I rischi del blocco automatico (i falsi positivi tagliano fuori utenti reali)
- Come strumenti come Splunk SOAR e Palo Alto XSOAR implementano questi pattern

**Pianificare questa funzionalità:**

Prima di scrivere codice, rifletti su:
- Quali azioni dovrebbero essere completamente automatizzate e quali richiedere l'approvazione umana?
- Come gestisci i blocchi automatici dovuti a falsi positivi? (Risposta: sblocco automatico dopo un TTL, richiedere la conferma dell'analista per i blocchi permanenti)
- Qual è il blast radius se il modello ML sbaglia e genera falsi positivi di massa?

**Architettura ad alto livello:**

```
Scored Request (HIGH severity)
    │
    ├──► Playbook Engine
    │       │
    │       ├──► Block IP (nginx blocklist update)
    │       ├──► Notify (Slack webhook)
    │       ├──► Enrich (AbuseIPDB lookup)
    │       └──► Ticket (create incident)
    │
    └──► Audit Log (all actions recorded)
```

**Fasi di implementazione:**

**Fase 1: Playbook Engine**
- Definisci i playbook come file YAML che descrivono condizioni di trigger e azioni
- Costruisci un semplice valutatore di regole che associ le richieste con punteggio ai playbook
- Registra nei log tutte le azioni automatiche per l'audit

**Fase 2: Azioni di risposta**
- Blocklist nginx: scrivi gli IP bloccati in un file che nginx include tramite la direttiva `deny`, poi invia SIGHUP per ricaricare
- Notifica Slack: POST a un URL di webhook con i dettagli della minaccia
- Enrichment con AbuseIPDB: GET del punteggio di abuse confidence e dello storico dell'IP

**Fase 3: Protezioni di sicurezza**
- Rate limit sui blocchi automatici (max 10 IP all'ora)
- Sblocco automatico dopo un TTL configurabile (predefinito 1 ora)
- Richiedi la conferma dell'analista per i blocchi che riguardano più di N richieste
- Kill switch: disabilita tutte le risposte automatiche tramite un endpoint API

**Fase 4: Integrazione con la dashboard**
- Mostra lo stato della risposta automatica nella pagina di dettaglio della minaccia
- Visualizza i dati di enrichment (punteggio AbuseIPDB, risultati VirusTotal)
- Pulsante "Undo Block" per gli analisti

**Criteri di successo:**
- [ ] Gli eventi di severity HIGH innescano notifiche Slack automatiche entro 5 secondi
- [ ] Gli IP bloccati vengono sbloccati automaticamente dopo il TTL configurato
- [ ] Tutte le azioni automatiche vengono registrate nei log con il contesto completo
- [ ] Il kill switch disabilita tutta l'automazione con una sola chiamata API
- [ ] I blocchi automatici dovuti a falsi positivi possono essere annullati dagli analisti dalla dashboard

## Combina e abbina

Combina le funzionalità per progetti più grandi:

**Idea di progetto 1: Stack SOC completo**
- Combina la Sfida 4 (active learning) + la Sfida 6 (Prometheus) + la Sfida 9 (SOAR)
- Aggiungi un sistema di case management per tracciare gli incidenti dalla detection alla risoluzione
- Risultato: un Security Operations Center in miniatura in uno stack Docker Compose

**Idea di progetto 2: Detection multi-sorgente**
- Combina la Sfida 5 (log degli errori) + la Sfida 7 (transformer) + la Sfida 8 (Kafka)
- Acquisisci da più sorgenti di log, usa il transformer per rilevare attacchi multi-stadio che si estendono su sequenze di richieste
- Risultato: una piattaforma di threat detection distribuita e multi-sorgente

## Sfide di integrazione nel mondo reale

### Integrare con Elasticsearch e Kibana

**L'obiettivo:**
Invia tutti i threat event con punteggio a Elasticsearch per l'archiviazione a lungo termine e costruisci dashboard Kibana per l'investigazione.

**Cosa ti servirà:**
- Un cluster Elasticsearch 8.x (può girare in locale tramite Docker)
- Kibana per la visualizzazione
- Comprensione dei mapping degli indici di Elasticsearch e delle policy ILM

**Piano di implementazione:**
1. Aggiungi un output Elasticsearch all'`AlertDispatcher` (in parallelo a PostgreSQL, senza sostituirlo)
2. Definisci un mapping dell'indice che includa i campi GeoIP come tipo `geo_point` per le visualizzazioni su mappa
3. Costruisci una dashboard Kibana con:
   - Mappa delle minacce che mostri gli IP sorgente su una mappa del mondo
   - Timeline dei threat score nel tempo
   - Principali tipi di attacco per nome della regola
   - Drill-down dalla severity ai singoli eventi

**Attenzione a:**
- La latenza del bulk indexing di Elasticsearch può rallentare lo stadio di dispatch. Usa operazioni bulk asincrone con un buffer
- Index lifecycle management (ILM) per evitare una crescita illimitata dello storage. Ruota gli indici ogni giorno ed eliminali dopo 30 giorni

### Deploy su Kubernetes

**L'obiettivo:**
Converti lo stack Docker Compose in manifest Kubernetes ed esegui il deploy su un cluster.

**Cosa imparerai:**
- Scrivere Deployment, Service, ConfigMap e Secret Kubernetes
- Persistent volume claim per PostgreSQL e per i dati dei modelli
- Health probe (liveness, readiness, startup) mappate dagli healthcheck di Docker
- Horizontal Pod Autoscaler per il backend basato sulle metriche di profondità delle code

**Passi:**
1. Converti ogni servizio Docker Compose in un Deployment Kubernetes
2. Crea Service per la comunicazione interna (postgres, redis, backend)
3. Usa una ConfigMap per la configurazione non segreta e Secret per le credenziali
4. Crea PersistentVolumeClaim per i dati di postgres e gli artefatti dei modelli
5. Aggiungi una risorsa Ingress per il traffico del frontend
6. Configura l'HPA per il backend basandoti sulle metriche Prometheus (Sfida 6)

**Checklist di produzione:**
- [ ] Tutti i secret sono memorizzati in Kubernetes Secret, non in ConfigMap
- [ ] PostgreSQL usa uno StatefulSet con storage persistente
- [ ] La readiness probe del backend blocca il traffico finché i modelli non sono caricati
- [ ] Per tutti i pod sono impostati resource request e limit
- [ ] Le network policy limitano l'accesso a PostgreSQL e Redis

## Sfide di performance

### Sfida: Gestire 5000 richieste al secondo

**L'obiettivo:**
Fai in modo che la pipeline a processo singolo gestisca 5000 req/s senza perdere log.

**Collo di bottiglia attuale:**
Inferenza ML a ~640 req/s con batch size 32. Lo stadio di detection è il limite.

**Approcci di ottimizzazione:**

**Approccio 1: Aumentare la batch size**
- Come: imposta `BATCH_SIZE=128` e `BATCH_TIMEOUT_MS=100`
- Guadagno: batch più grandi ammortizzano l'overhead della sessione ONNX. Throughput atteso 2-3x
- Compromesso: latenza più alta (fino a 100ms di attesa per un batch pieno contro 50ms)

**Approccio 2: Quantizzare i modelli ONNX**
- Come: usa gli strumenti di quantizzazione di ONNX Runtime per convertire i modelli float32 in int8
- Guadagno: inferenza 2-4x più veloce su CPU con perdita di accuratezza minima
- Compromesso: piccolo degrado di accuratezza, specialmente per l'autoencoder. Valida con i quality gate

**Approccio 3: Saltare l'inferenza ML per il traffico chiaramente benigno**
- Come: se il rule engine assegna 0.0 (nessuna regola corrisponde), salta l'inferenza ML e restituisci direttamente il punteggio della regola
- Guadagno: il 90-95% del traffico è benigno, quindi questo salta l'inferenza per la maggior parte delle richieste
- Compromesso: perdi le detection solo-ML (anomalie che non corrispondono ad alcuna regola). Accettabile se la copertura delle regole è buona

**Fai il benchmark:**
```bash
just devlog-simulate normal 10000 -d 0.0001
```

Metriche target:
- Throughput: 5000+ req/s in modo sostenuto
- Profondità delle code: raw_queue mai piena
- Latenza p99: < 200ms end-to-end

### Sfida: Ridurre l'impronta di memoria

**L'obiettivo:**
Esegui l'intero stack su una macchina con 2GB di RAM (attualmente ne servono 4GB+).

**Prima fai il profiling:**
```bash
docker stats --no-stream
```

**Aree di ottimizzazione comuni:**
- Modelli ONNX: 50-200MB. La quantizzazione (int8) li riduce di 2-4x
- Database GeoIP: ~60MB in mmap. Passa al più piccolo GeoLite2-Country (~5MB) se non serve la precisione a livello di città
- PostgreSQL: riduci `shared_buffers` e `work_mem` in `postgresql.conf`
- Redis: imposta `maxmemory 64mb` con eviction `allkeys-lru` per limitare l'uso di memoria

## Sfide di sicurezza

### Sfida: Aggiungere la mutual authentication TLS per l'API

**Cosa implementare:**
Richiedi certificati client per l'accesso all'API al posto di (o in aggiunta a) l'header X-API-Key.

**Threat model:**
Questo protegge da:
- Furto della chiave API (la chiave è una stringa statica che potrebbe finire nei log, nei file di configurazione o nella cronologia dei commit)
- Replay attack (una chiave API rubata può essere usata da qualsiasi posizione di rete)

**Implementazione:**
- Genera un certificato CA, un certificato server e un certificato client usando `openssl`
- Configura uvicorn in modo che richieda certificati client (oppure metti nginx davanti al backend con `ssl_verify_client on`)
- Valida il CN del subject del certificato client rispetto a una allowlist

**Testare la sicurezza:**
- Prova a raggiungere l'API senza un certificato client (dovresti ottenere 403)
- Prova con un certificato firmato da una CA diversa (dovresti ottenere 403)
- Verifica che la rotazione dei certificati funzioni (emetti nuovi certificati, quelli vecchi funzionano finché non vengono revocati)

### Sfida: Superare il CIS Docker Benchmark

**L'obiettivo:**
Rendi lo stack Docker Compose conforme al CIS Docker Benchmark.

**Lacune attuali:**
- I container girano come root (aggiungi direttive `USER` ai Dockerfile)
- Nessun limite di risorse (aggiungi `mem_limit` e `cpus` a compose.yml)
- Nessun filesystem root in sola lettura (aggiungi `read_only: true` dove possibile)
- Nessuna opzione di sicurezza (aggiungi `security_opt: [no-new-privileges:true]`)

**Remediation:**
Esegui `docker-bench-security` sullo stack e risolvi ogni finding. Concentrati sulle sezioni di configurazione dell'host, configurazione del daemon e runtime dei container.

## Idee per contribuire

Hai finito una sfida? Condividila:

1. **Fai il fork del repo**
2. **Implementa la tua estensione** in un nuovo branch
3. **Documentala** in questa cartella learn
4. **Invia una PR** con:
   - La tua implementazione
   - Test
   - Documentazione learn aggiornata
   - Output di esempio che mostri che funziona

Le buone estensioni potrebbero essere integrate nel progetto principale.

## Sfida te stesso ancora di più

### Costruisci qualcosa di nuovo

Usa i concetti che hai imparato qui per costruire:
- Un **rilevatore di anomalie nel traffico di rete** che analizza file pcap invece dei log HTTP, usando lo stesso approccio a ensemble ma con feature a livello di rete (dimensioni dei pacchetti, distribuzione dei protocolli, pattern di connessione)
- Un **rilevatore di minacce DNS** che monitora i log delle query DNS per tunneling, domini DGA e pattern di risoluzione anomali
- Un **rilevatore di anomalie nel runtime dei container** che osserva le trace delle syscall da Falco e segnala comportamenti insoliti dei processi

### Studia implementazioni reali

Confronta la tua implementazione con i tool di produzione:
- **Suricata** - IDS/IPS open source. Guarda come il loro rule engine gestisce il signature matching a velocità di linea. La loro architettura multi-thread gestisce 10 Gbps+
- **Elastic ML** - Il modulo di anomaly detection di Elasticsearch. Confronta il loro approccio di decomposizione delle serie temporali con il nostro approccio con autoencoder
- **AWS GuardDuty** - La threat detection gestita di Amazon. Leggi la loro documentazione su come combinano ML, threat intelligence e analisi comportamentale

Leggi il loro codice (Suricata è open source), comprendi i loro compromessi, ruba le loro buone idee.

### Scrivi a riguardo

Documenta la tua estensione:
- Un post sul blog che spieghi cosa hai costruito e cosa hai imparato sulla threat detection
- Un confronto tra approcci di detection basati su regole e basati su ML con esempi concreti dalla tua esperienza con questo progetto
- Un'analisi dei tassi di falsi positivi con diverse configurazioni dei pesi dell'ensemble

Insegnare agli altri è il modo migliore per verificare di aver capito.

## Ottenere aiuto

Bloccato su una sfida?

1. **Fai il debug in modo sistematico**
   - Cosa ti aspettavi che succedesse?
   - Cosa è successo davvero?
   - Qual è il caso di test più piccolo che lo riproduce?

2. **Leggi il codice esistente**
   - Il rule engine in `rules.py` mostra il pattern per aggiungere logica di detection
   - La pipeline in `pipeline.py` mostra come comunicano gli stadi
   - L'orchestrator in `orchestrator.py` mostra l'intero flusso di training

3. **Cerca problemi simili**
   - Pattern async di FastAPI: fastapi.tiangolo.com
   - Problemi di ONNX Runtime: github.com/microsoft/onnxruntime
   - Pattern dei sorted set di Redis: redis.io/docs/data-types/sorted-sets

4. **Chiedi aiuto**
   - Scrivi nelle GitHub Discussions
   - Includi: cosa hai provato, cosa è successo, cosa ti aspettavi
   - Mostra il codice rilevante e i messaggi di errore, non solo lo stack trace

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

- [ ] Facile 1: Aggiungere la regola di detection XXE
- [ ] Facile 2: Aggiungere la detection delle anomalie nei metodi
- [ ] Facile 3: Aggiungere la blocklist per paese
- [ ] Intermedia 4: Costruire il ciclo di feedback con active learning
- [ ] Intermedia 5: Aggiungere l'analisi dei log degli errori
- [ ] Intermedia 6: Aggiungere le metriche Prometheus
- [ ] Avanzata 7: Aggiungere il modello transformer
- [ ] Avanzata 8: Costruire la pipeline distribuita con Kafka
- [ ] Esperta 9: Costruire l'integrazione SOAR

Le hai completate tutte? Hai padroneggiato questo progetto. È ora di costruire qualcosa di nuovo o di contribuire alla community.
