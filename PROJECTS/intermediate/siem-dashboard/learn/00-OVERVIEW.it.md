# SIEM Dashboard

## Che cos'è

Una piattaforma full stack di Security Information and Event Management (SIEM) che acquisisce eventi di log da più tipi di sorgente, li normalizza in uno schema comune, classifica la severità tramite pattern matching ed esegue un motore di correlazione in tempo reale per generare alert. Include una dashboard React per il monitoraggio, le indagini e la riproduzione di scenari di attacco.

Il backend è Flask con MongoDB per la persistenza e Redis Streams per la consegna degli eventi in tempo reale. Il frontend è React con TypeScript, Zustand per lo stato e Server-Sent Events per gli aggiornamenti live.

## Perché è importante

Ogni organizzazione con più di una manciata di server ha bisogno di un monitoraggio centralizzato dei log. Senza di esso, sei cieco agli attacchi finché il danno non è fatto. I SIEM commerciali come Splunk o Microsoft Sentinel costano decine di migliaia all'anno, e la maggior parte dei team di sicurezza fatica ancora con l'alert fatigue, le correlazioni mancate e i workflow di indagine lenti.

Costruire un SIEM da zero ti insegna come funzionano davvero questi sistemi dietro le quinte. Capirai perché le regole di correlazione contano, come la normalizzazione degli eventi previene i silos di dati e perché lo streaming in tempo reale è fondamentale per l'incident response.

**Scenari reali in cui si applica:**
- Un analista SOC deve correlare 20 login SSH falliti da un IP con un login riuscito 30 secondi dopo. Si tratta di un brute force seguito da una compromissione, e il motore di correlazione di questo progetto rileva esattamente questo pattern.
- Un incident responder fa pivot da un IP sorgente sospetto per trovare tutti gli eventi correlati nei log di firewall, autenticazione ed endpoint. La pivot API (`/v1/logs/pivot`) abilita questo workflow.
- Un team di sicurezza vuole testare le regole di rilevamento sui dati storici prima di metterle in produzione. L'endpoint di test delle regole (`/v1/rules/<id>/test`) riproduce gli eventi attraverso una regola senza creare alert reali.

## Cosa imparerai

Questo progetto ti insegna come i sistemi SIEM elaborano, correlano e fanno emergere gli eventi di sicurezza. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**
- La normalizzazione dei log tra formati di sorgente diversi (firewall, IDS, autenticazione, endpoint, DNS, proxy). Ogni tipo di sorgente ha campi diversi e il normalizer in `app/engine/normalizer.py` li mappa su uno schema comune.
- La classificazione della severità tramite pattern matching con regex su indicatori di attacco noti come "brute force", "lateral movement" e "data exfiltration".
- La valutazione delle regole di correlazione con sliding window, inclusi il conteggio a soglia, il rilevamento di sequenze ordinate e l'aggregazione su valori distinti.
- La gestione del ciclo di vita degli alert, dal rilevamento iniziale fino alla presa in carico, all'indagine, alla risoluzione o alla classificazione come falso positivo.

**Competenze tecniche:**
- Costruire una application factory Flask con MongoEngine, Redis e una gestione strutturata degli errori
- Implementare l'autenticazione JWT con hashing delle password Argon2id e una verifica timing safe per prevenire l'enumerazione degli utenti
- Usare Redis Streams con consumer group per una consegna affidabile degli eventi e Server-Sent Events per gli aggiornamenti in tempo reale nel browser
- Scrivere un motore di correlazione multi-thread che valuta le regole su un flusso continuo di eventi
- Costruire un frontend React con gestione dello stato tramite Zustand, TanStack Query per il data fetching e integrazione SSE

**Strumenti e tecniche:**
- Docker Compose per lo sviluppo locale con MongoDB, Redis, Nginx, Flask e Vite in esecuzione insieme
- Playbook di scenari di attacco basati su YAML che simulano tecniche MITRE ATT&CK reali come il brute force (T1110.001) e il DNS tunneling (T1048.003)
- Schemi Pydantic per la validazione delle richieste con formattazione automatica degli errori
- Pipeline di aggregazione MongoDB per il bucketing della timeline, la ripartizione per severità e l'analisi delle sorgenti principali

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**
- Le basi di Python, incluse classi, decorator e type hint. Vedrai decorator come `@endpoint(roles=ADMIN)` e `@_register(SourceType.FIREWALL)` in tutta la codebase.
- Una comprensione di base delle REST API e di HTTP. Il backend espone circa 30 endpoint sotto `/v1/`.
- Familiarità con JSON e con i concetti base dei database. MongoDB memorizza documenti come strutture simili a JSON e lavorerai con l'ORM di MongoEngine.

**Strumenti di cui avrai bisogno:**
- Docker e Docker Compose, per eseguire l'intero stack in locale
- Python 3.14+ (il progetto usa type hint moderni come `str | None`)
- Node.js 24+ e pnpm per il frontend
- Uno strumento per fare richieste HTTP (curl, httpie o Postman) per testare direttamente l'API

**Utili ma non obbligatori:**
- Esperienza con le query e le pipeline di aggregazione di MongoDB
- Familiarità con le strutture dati di Redis, in particolare gli Streams
- Conoscenze di base di React e TypeScript per capire il frontend

## Avvio rapido

Fai girare il progetto in locale:

```bash
# Navigate to the project
cd PROJECTS/intermediate/siem-dashboard

# Start development environment
docker compose -f dev.compose.yml up --build

# The app will be available at http://localhost:8431
# Backend API is at http://localhost:8431/api/v1/
# Direct backend access at http://localhost:5113
# Direct frontend dev server at http://localhost:3959
```

Crea un account amministratore:

```bash
docker exec -it siem-backend-dev flask admin create \
  --username admin \
  --email admin@example.com
```

Output atteso: `Admin account 'admin' created successfully.`

Testa l'API:

```bash
# Register a regular user
curl -X POST http://localhost:8431/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"analyst1","email":"analyst@test.com","password":"testpass123"}'

# Ingest a test log event
curl -X POST http://localhost:8431/api/v1/logs/ingest \
  -H "Content-Type: application/json" \
  -d '{"source_type":"auth","event_type":"login_failure","source_ip":"10.0.0.1","username":"root"}'
```

## Struttura del progetto

```
siem-dashboard/
├── backend/
│   ├── app/
│   │   ├── __init__.py           # Flask app factory
│   │   ├── config.py             # Pydantic settings (60+ config values)
│   │   ├── extensions.py         # MongoDB and Redis initialization
│   │   ├── cli.py                # Flask CLI for admin management
│   │   ├── core/
│   │   │   ├── auth.py           # JWT + Argon2id authentication
│   │   │   ├── streaming.py      # Redis Streams + SSE generator
│   │   │   ├── errors.py         # Error hierarchy and handlers
│   │   │   ├── rate_limiting.py  # Flask-Limiter setup
│   │   │   ├── serialization.py  # MongoEngine to JSON conversion
│   │   │   └── decorators/       # @endpoint, @S, @R composable decorators
│   │   ├── engine/
│   │   │   ├── normalizer.py     # Multi-format log normalizer
│   │   │   ├── severity.py       # Pattern-based severity classifier
│   │   │   └── correlation.py    # Sliding window correlation engine
│   │   ├── models/               # MongoEngine documents (User, LogEvent, Alert, etc.)
│   │   ├── routes/               # Flask blueprints for each resource
│   │   ├── controllers/          # Business logic handlers
│   │   ├── schemas/              # Pydantic request/response schemas
│   │   └── scenarios/
│   │       ├── playbook.py       # YAML playbook parser
│   │       ├── runner.py         # Threaded scenario executor
│   │       └── playbooks/        # Attack scenario YAML files
│   ├── pyproject.toml            # Python dependencies and tool config
│   └── wsgi.py                   # Gunicorn entry point
├── frontend/
│   ├── src/
│   │   ├── api/hooks/            # TanStack Query hooks for each resource
│   │   ├── api/types/            # Zod schemas and TypeScript types
│   │   ├── core/app/             # Shell layout, routing, protected routes
│   │   ├── core/lib/             # Axios client, error handling, query config
│   │   ├── core/stores/          # Zustand stores (auth, stream, UI)
│   │   ├── core/charts/          # visx chart theme and color constants
│   │   ├── routes/               # Page components (lazy loaded)
│   │   └── config.ts             # API endpoints, query keys, constants
│   └── package.json
├── conf/
│   ├── docker/                   # Dockerfiles for dev and prod
│   └── nginx/                    # Nginx configs for proxying and SSE
├── compose.yml                   # Production Docker Compose
└── dev.compose.yml               # Development Docker Compose
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per conoscere la correlazione dei log, la normalizzazione degli eventi e i principi architetturali dei SIEM
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere come si collegano il motore del backend, il livello di streaming e il frontend
3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per un walkthrough dettagliato del motore di correlazione, del normalizer e della pipeline di streaming
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee come aggiungere il supporto alle regole Sigma, costruire un feed di threat intelligence o implementare playbook SOAR

## Problemi comuni

**Il container del backend continua a riavviarsi**
```
siem-backend-dev | Connection refused: mongodb://mongo:27017/siem
```
Soluzione: MongoDB impiega 10-30 secondi per inizializzarsi. L'healthcheck in `dev.compose.yml` ha `start_period: 30s`, ma a volte non basta sulle macchine più lente. Attendi e controlla `docker compose logs mongo`.

**Gli stream SSE si disconnettono immediatamente**
La configurazione Nginx in `conf/nginx/dev.nginx` ha una gestione speciale per gli endpoint SSE. Se vedi disconnessioni istantanee, verifica che il blocco location con regex `location ~ ^/api(/v1/(logs|alerts)/stream.*)` corrisponda alla tua richiesta. Gli endpoint SSE hanno bisogno di `proxy_buffering off` e di timeout di lettura lunghi.

**Errori di Redis Stream all'avvio**
```
BUSYGROUP Consumer Group name already exists
```
È normale. La funzione `ensure_consumer_group()` in `app/core/streaming.py` racchiude la creazione in `contextlib.suppress(Exception)` perché è idempotente. Il consumer group esiste già da un'esecuzione precedente.

**Il rate limiting blocca le richieste di sviluppo**
Gli endpoint di autenticazione sono limitati a 10 richieste al minuto (`RATELIMIT_AUTH` in `config.py`). Gli endpoint generici ne consentono 200 al minuto. Se stai martellando l'endpoint di login durante i test, aspetta un minuto o aumenta temporaneamente il limite nel tuo file `.env`.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **API Rate Limiter** (avanzato) - Approfondimento sugli algoritmi di rate limiting che questo progetto usa tramite Flask-Limiter, incluse le strategie sliding window e token bucket
- **Network Traffic Analyzer** - Analisi a livello di pacchetto con Scapy, a complemento dell'analisi dei log di livello più alto che esegue questo SIEM
- **Docker Security Auditor** - Scansione di sicurezza dei container che potrebbe alimentare con eventi la pipeline di ingestione di questo SIEM
