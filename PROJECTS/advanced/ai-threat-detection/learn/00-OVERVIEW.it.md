# AngelusVigil: AI Threat Detection

## Cos'è

AngelusVigil è un motore di threat detection in tempo reale che analizza i log del web server nginx usando un ensemble ML di 3 modelli (autoencoder + random forest + isolation forest) per classificare il traffico HTTP come benigno o malevolo. Viene distribuito come sidecar dockerizzato accanto a qualsiasi infrastruttura basata su nginx, senza richiedere alcuna modifica al codice. Il sistema elabora i log attraverso una pipeline async a 4 stadi, estrae vettori di feature a 35 dimensioni, assegna un punteggio al traffico usando sia il rilevamento basato su regole sia quello ML, e mostra gli alert tramite una dashboard React con aggiornamenti live via WebSocket.

## Perché è importante

I Web Application Firewall (WAF) intercettano i pattern di attacco noti, ma non vedono gli attacchi nuovi e faticano con le anomalie comportamentali. Nella violazione di Equifax del 2017 (CVE-2017-5638), gli attaccanti hanno sfruttato una vulnerabilità di Apache Struts che gli strumenti basati su signature non hanno rilevato per mesi. La vulnerabilità Log4Shell del 2021 (CVE-2021-44228) si è diffusa su milioni di server prima che le regole dei WAF venissero aggiornate. La violazione di Capital One del 2019 ha coinvolto un WAF mal configurato che un insider ha sfruttato per esfiltrare 100 milioni di record di clienti.

Il rilevamento basato su ML colma queste lacune imparando come appare il "normale" per il tuo specifico traffico e segnalando le deviazioni. Questo progetto costruisce quel sistema da zero.

**Scenari reali in cui si applica:**
- Una piattaforma SaaS che usa nginx come reverse proxy vuole rilevare SQL injection, path traversal e credential stuffing senza acquistare un WAF commerciale
- Un team DevSecOps ha bisogno di alerting in tempo reale quando i pattern di attacco aumentano contro i propri endpoint API, con abbastanza contesto (GeoIP, vettori di feature, regole corrispondenti) per fare triage rapidamente
- Un team di sicurezza vuole combinare il rilevamento basato su signature (copertura day 0) con modelli ML che migliorano nel tempo man mano che etichettano i veri/falsi positivi del traffico di produzione

## Cosa imparerai

Questo progetto ti insegna come funziona la threat detection basata su ML a livello di infrastruttura. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**
- Anomaly detection usando autoencoder addestrati esclusivamente su traffico normale, così che tutto ciò che il modello non riesce a ricostruire bene risulti sospetto
- Ensemble learning in cui più modelli votano se una richiesta è malevola, riducendo i falsi positivi di ogni singolo modello
- Rilevamento cold-start usando regole ispirate a ModSecurity CRS che forniscono copertura immediata prima che i modelli ML siano addestrati
- Feature engineering per il traffico HTTP, trasformando le righe di log grezze in vettori numerici a 35 dimensioni che catturano la struttura della richiesta, i pattern comportamentali e i segnali temporali

**Competenze tecniche:**
- Costruire pipeline di dati async con backpressure usando `asyncio.Queue` e propagazione dello shutdown tramite poison-pill
- Addestrare ed esportare modelli PyTorch e scikit-learn in formato ONNX per un'inferenza veloce su CPU
- Implementare l'aggregazione a finestra scorrevole con sorted set di Redis per le feature comportamentali per singolo IP
- Scrivere un file tailer basato su Watchdog che gestisce la rotazione dei log di nginx senza perdere righe

**Strumenti e tecniche:**
- ONNX Runtime per il serving di modelli cross-framework (il 27% più veloce dell'inferenza PyTorch nativa)
- Tracciamento degli esperimenti con MLflow per il versionamento dei modelli e il logging delle metriche
- Redis pub/sub per il relay WebSocket in tempo reale su più worker backend
- Orchestrazione con Docker Compose di uno stack di produzione a 5 servizi

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**
- Python async/await (lavorerai con `asyncio.Queue`, `asyncio.Task` e async context manager lungo tutta la pipeline)
- Concetti base di machine learning (cosa significano training/inference, cosa fanno le loss function, la differenza tra apprendimento supervisionato e non supervisionato)
- Fondamenti di HTTP (metodi di richiesta, status code, query string, header, come sono strutturati gli access log di nginx)
- Docker e Docker Compose (l'intero sistema gira come container)

**Strumenti necessari:**
- Docker e Docker Compose v2
- Python 3.13+ (il backend usa la sintassi moderna dei tipi come `dict[str, float]` e `X | None`)
- Node.js 20+ e pnpm (per il frontend React)
- just (command runner, come make ma meglio)
- uv (gestore di pacchetti Python)

**Utile ma non obbligatorio:**
- Esperienza con FastAPI o qualsiasi framework web Python async
- Familiarità con PyTorch o scikit-learn
- Comprensione delle strutture dati di Redis (sorted set, pub/sub)

## Avvio rapido

Fai partire il progetto in locale:

```bash
cd PROJECTS/advanced/ai-threat-detection

# Install just if you don't have it
# https://github.com/casey/just

# One-time setup: install deps and create .env
just setup

# Edit .env with your values (at minimum set POSTGRES_PASSWORD and API_KEY)
# GEOIP credentials are optional for local dev

# Start the dev stack (postgres, redis, backend with hot-reload, frontend)
just dev-up

# In another terminal, start the dev log generator to simulate traffic
just devlog-up

# Generate some mixed traffic (normal + attacks)
just devlog-simulate mixed 200
```

Output atteso: apri `http://localhost:5173` nel tuo browser. Dovresti vedere la dashboard con stat card che mostrano le minacce rilevate, una barra di distribuzione della severity, un feed di alert live che riceve eventi WebSocket, e liste ordinate dei principali IP attaccanti e dei path più colpiti.

Il backend parte in modalità solo regole. Per abilitare il rilevamento ML, addestra i modelli:

```bash
just vigil-train
```

Al termine del training (circa 2 minuti con dati sintetici), riavvia il backend e questo caricherà i modelli ONNX e passerà alla modalità di rilevamento ibrida.

## Struttura del progetto

```
ai-threat-detection/
├── backend/
│   ├── app/
│   │   ├── api/               # FastAPI route handlers (health, threats, stats, models, ws)
│   │   ├── core/
│   │   │   ├── alerts/        # AlertDispatcher: store + publish scored events
│   │   │   ├── detection/     # RuleEngine, InferenceEngine, ensemble scoring
│   │   │   ├── enrichment/    # GeoIP lookups via MaxMind
│   │   │   ├── features/      # Feature extraction, Redis aggregation, encoding
│   │   │   └── ingestion/     # Log tailer, parsers, 4-stage pipeline
│   │   ├── models/            # SQLModel ORM (ThreatEvent, ModelMetadata)
│   │   ├── schemas/           # Pydantic request/response models
│   │   ├── services/          # Database query logic (threat_service, stats_service)
│   │   ├── config.py          # Pydantic Settings (env vars, defaults, validation)
│   │   └── factory.py         # App factory with async lifespan
│   ├── ml/                    # Training pipeline (orchestrator, splitting, export, validation)
│   ├── cli/                   # Typer CLI (train, retrain, replay commands)
│   ├── tests/                 # pytest suite (parsers, features, detection, integration)
│   └── alembic/               # Database migrations
├── frontend/
│   ├── src/
│   │   ├── api/               # React Query hooks, Zod schemas, Axios client
│   │   ├── components/        # AlertFeed, SeverityBadge, MethodBadge, StatCard, ThreatDetail
│   │   ├── core/              # Router config, shell layout, query client
│   │   └── pages/             # Dashboard, Threats (table + filters), Models
│   └── vite.config.ts
├── dev-log/                   # Synthetic nginx traffic generator for testing
├── infra/                     # Dockerfiles, redis.conf, nginx.conf
├── compose.yml                # Production 5-service stack
├── dev.compose.yml            # Development stack with hot-reload
└── justfile                   # Task runner commands
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per imparare anomaly detection, metodi ensemble e feature engineering per il traffico HTTP
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) per vedere come la pipeline a 4 stadi, il windowing su Redis e l'inferenza ONNX si incastrano
3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per una walkthrough dettagliata della pipeline di ingestion, del rule engine, del training ML e del frontend
4. **Estendi il progetto** - Leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per idee come aggiungere nuovi tipi di modello, integrarsi con strumenti SIEM o costruire un workflow di active learning

## Problemi comuni

**Il backend non parte con "database connection refused"**
```
sqlalchemy.exc.OperationalError: connection to server at "localhost" ... refused
```
Soluzione: assicurati che PostgreSQL sia in esecuzione. Se usi Docker, controlla `just dev-ps` per verificare che il container postgres sia healthy. Il backend dipende da `postgres:service_healthy`, quindi attenderà, ma se postgres stesso non è partito, controlla i log con `just dev-logs postgres`.

**Warning "onnxruntime not installed" all'avvio**
È previsto se non hai installato le dipendenze ML. Il sistema ripiega automaticamente sulla modalità solo regole. Per abilitare il rilevamento ML, esegui `cd backend && uv sync --group ml` e poi addestra i modelli con `just vigil-train`.

**Gli alert WebSocket non compaiono sulla dashboard**
Controlla che Redis sia in esecuzione (`just dev-logs redis`). Il relay WebSocket dipende da Redis pub/sub. Verifica anche che il backend stia elaborando i log chiamando `http://localhost:8000/health` e controllando il campo `stats` nella risposta.

## Progetti correlati

Se hai trovato interessante questo progetto, dai un'occhiata a:
- [SIEM Dashboard](../../intermediate/siem-dashboard/) - Costruisci una dashboard SIEM (Security Information and Event Management) con Flask, MongoDB e Redis Streams per l'aggregazione e la correlazione dei log
- [Honeypot Network](../honeypot-network/) - Distribuisci servizi ingannevoli che generano il tipo di traffico di attacco che questo progetto rileva
- [API Security Scanner](../../intermediate/api-security-scanner/) - Scansione attiva delle vulnerabilità che complementa l'approccio di rilevamento passivo di questo progetto
