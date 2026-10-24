<!-- © AngelaMos | 2026 | 00-OVERVIEW.md -->

# C2 Beacon / Server

## Che cos'è

Questo è un beacon e un server di Command and Control (C2) a scopo didattico, costruito per mostrare come i framework C2 reali come Cobalt Strike, Sliver e Mythic funzionano sotto il cofano. L'implant beacon si connette al server tramite WebSocket, con il traffico codificato usando XOR e Base64. Ci sono 10 comandi mappati alle tecniche MITRE ATT&CK, un sistema di accodamento dei task e una dashboard operatore in tempo reale in cui puoi selezionare i beacon e impartire comandi tramite una UI in stile terminale.

## Perché è importante

Cobalt Strike viene citato in quasi il 70% degli interventi di incident response. È, con ampio margine, il framework C2 più comunemente incontrato nelle violazioni reali. Quando i difensori trovano beacon di Cobalt Strike in una rete, hanno bisogno di capire come il beacon comunica, come vengono accodati i task e come l'operatore controlla l'implant. Senza questa comprensione, l'incident response è per lo più tirare a indovinare.

L'attacco SolarWinds SUNBURST del 2020 usava un protocollo C2 personalizzato tunnelato su HTTP, con l'implant mascherato da un legittimo aggiornamento del software Orion. Il beacon raccoglieva informazioni di sistema, dormiva fino a due settimane prima di effettuare la sua prima callback, e poi riceveva i task tramite risposte DNS e HTTP costruite per sembrare normale telemetria. La sofisticazione era nel design del protocollo, non nell'exploit in sé.

APT29 (Cozy Bear) è stato documentato mentre usava implant C2 personalizzati con intervalli di sleep con jitter, domain fronting e canali cifrati che si confondono nel normale traffico HTTPS. Il jitter rende più difficile il rilevamento basato sulla rete perché lo schema delle callback appare irregolare anziché meccanico. Questo progetto implementa lo sleep con jitter per lo stesso motivo, così puoi vedere esattamente come funziona quella tecnica e perché un beacon a intervallo fisso è banale da individuare tramite fingerprinting.

Se lavori in un blue team, nell'incident response o nella threat intelligence, capire l'architettura C2 è fondamentale. Non puoi scrivere regole di detection per qualcosa che non capisci. Costruirne uno da zero, anche una versione didattica semplificata, ti dà un modello mentale che la sola lettura di report non fornisce.

## Cosa imparerai

**Concetti di sicurezza:** architettura C2 e il modello beacon/server. Mapping delle tecniche MITRE ATT&CK (T1059 Command Execution, T1082 System Discovery, T1057 Process Discovery, T1105 Ingress Tool Transfer, T1113 Screen Capture, T1056 Input Capture, T1053 Persistence, T1029 Scheduled Transfer). Codifica del protocollo con XOR e Base64. Tecniche di persistence tramite cron. Strategie di detection e come i difensori individuano il traffico C2.

**Competenze tecniche:** programmazione WebSocket su entrambi i lati della connessione. Python async con asyncio, inclusa l'esecuzione di subprocess e la gestione concorrente dei task. React con aggiornamenti della UI in tempo reale, gestione dello stato con Zustand e validazione degli schemi con Zod. Orchestrazione con Docker Compose con reverse proxy Nginx, health check e networking multi-servizio.

**Strumenti:** FastAPI per il server, aiosqlite per il database, psutil per l'enumerazione del sistema, websockets per il transport layer del beacon, Vite per il build del frontend, Zod per la validazione dei tipi a runtime e Pydantic per la modellazione dei dati sul backend.

## Prerequisiti

### Richiesti

- Python 3.13+
- Node.js 22+
- Docker e Docker Compose
- Comprensione di base del networking, di HTTP e di come funziona la comunicazione client-server

### Strumenti richiesti

- **uv** per la gestione dei pacchetti Python (mai pip). Installazione: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **pnpm** per la gestione dei pacchetti Node (mai npm). Installazione: `corepack enable && corepack prepare pnpm@latest --activate`
- **just** come command runner. Installazione: `cargo install just` o `brew install just`
- **Docker Compose** (incluso in Docker Desktop, oppure installa il plugin separatamente)

### Utili ma non obbligatori

- Familiarità con i WebSocket e il ciclo di vita connect/send/receive/close
- Esperienza con async/await in Python o JavaScript
- Precedente esposizione a React (la dashboard è semplice, ma la conoscenza di React aiuta)

## Avvio rapido

```bash
git clone https://github.com/CarterPerez-dev/Cybersecurity-Projects.git
cd PROJECTS/beginner/c2-beacon
docker compose -f dev.compose.yml up -d
```

Visita http://localhost:47430 nel tuo browser. Dovresti vedere la dashboard operatore con una tabella dei beacon vuota.

In un secondo terminale, avvia un beacon:

```bash
cd PROJECTS/beginner/c2-beacon
just beacon
```

Dovresti vedere un output come questo dal beacon:

```
2026-02-14 10:32:01 INFO: Connecting to ws://localhost:47430/api/ws/beacon
2026-02-14 10:32:01 INFO: Registered as 3a7f1c29-8b42-4e91-a6d3-9f0e5c8d2b17
```

Tornando al browser, il beacon appare nella tabella della dashboard con il suo hostname, OS, username e IP. Clicca la riga per aprire una sessione. Digita `shell whoami` nell'input del terminale e premi Invio. Il comando viene inviato al beacon, eseguito, e l'output appare nella UI del terminale.

Provane qualcuno in più:

```
sysinfo
proclist
shell ls -la /tmp
```

Ogni comando passa attraverso l'intera pipeline: la dashboard operatore invia JSON tramite WebSocket al server, il server accoda un task e lo inoltra (codificato con XOR+Base64) al beacon, il beacon esegue il comando, codifica il risultato, lo rimanda indietro, e il server lo trasmette alla dashboard operatore.

## Struttura del progetto

```
c2-beacon/
├── backend/
│   └── app/
│       ├── core/
│       │   ├── encoding.py      XOR + Base64 encode/decode functions
│       │   ├── models.py        Pydantic models: BeaconRecord, TaskRecord, TaskResult
│       │   └── protocol.py      Message envelope: pack/unpack with type validation
│       ├── beacon/
│       │   ├── registry.py      In-memory beacon registry with aiosqlite persistence
│       │   ├── router.py        WebSocket endpoint for beacon connections
│       │   └── tasking.py       Task queue: create, assign, complete, retrieve
│       ├── ops/
│       │   ├── manager.py       Operator WebSocket connection manager + broadcasting
│       │   └── router.py        Operator WebSocket + REST endpoints
│       ├── config.py            Pydantic Settings: XOR key, ports, CORS, DB path
│       └── database.py          aiosqlite setup with table creation
├── beacon/
│   └── beacon.py                The implant (~514 lines, single file, 10 command handlers)
├── frontend/
│   └── src/
│       ├── core/
│       │   ├── ws.ts            Zustand store + useOperatorSocket hook
│       │   └── types.ts         Zod schemas for all WebSocket message types
│       └── pages/
│           ├── dashboard/       Beacon table with real-time status updates
│           └── session/         Terminal UI with command input and result display
├── infra/
│   ├── docker/                  Dockerfiles for dev (hot reload) and prod
│   └── nginx/                   Reverse proxy configs (WebSocket proxying)
├── dev.compose.yml              3-service dev stack: nginx, backend, frontend
├── compose.yml                  Production compose
├── justfile                     Command runner: just beacon, just dev-up, etc.
└── learn/                       You are here
```

## Come funziona (in breve)

```
┌──────────┐    WebSocket     ┌──────────┐    WebSocket     ┌──────────┐
│  Beacon  │ ──XOR+Base64──>  │  Server  │ <──JSON────────  │ Operator │
│ (target) │ <──XOR+Base64──  │ (FastAPI)│ ──JSON────────>  │ (React)  │
└──────────┘                  └──────────┘                  └──────────┘
```

L'implant beacon (`beacon/beacon.py`) si connette al server tramite WebSocket su `/ws/beacon`. Alla connessione, invia un messaggio REGISTER contenente le informazioni di sistema: hostname, OS, username, PID, IP interno e architettura. Il server memorizza questo nel beacon registry (un dict in memoria con backing aiosqlite) e trasmette un evento `beacon_connected` a tutti gli operatori connessi.

Il beacon poi entra in un loop. Invia periodicamente messaggi HEARTBEAT con tempistica jitterata (intervallo base + percentuale di jitter casuale), che mantengono viva la connessione e aggiornano il timestamp "last seen" sul server. Tra un heartbeat e l'altro, attende i messaggi TASK dal server.

Sul lato operatore, la dashboard React si connette tramite un WebSocket separato su `/ws/operator`. Quando digiti un comando nel terminale della sessione, il frontend invia un messaggio JSON al server. Il server crea un TaskRecord, lo accoda, e inoltra il task al beacon target come messaggio TASK codificato con XOR+Base64.

Il beacon riceve il task, lo invia all'handler appropriato (shell, sysinfo, proclist, ecc.), e rimanda indietro un messaggio RESULT con l'output. Il server memorizza il risultato, segna il task come completato, e lo trasmette alla dashboard operatore, che lo renderizza nella UI del terminale.

Due canali WebSocket separati mantengono separate le responsabilità. Il canale beacon usa la codifica XOR+Base64 (simulando comunicazioni C2 cifrate). Il canale operatore usa JSON in chiaro (comunicazione interna fidata).

## Prossimi passi

- [01 - Concepts](01-CONCEPTS.it.md): teoria C2, mapping MITRE ATT&CK per ogni comando, e come i difensori rilevano i beacon
- [02 - Architecture](02-ARCHITECTURE.it.md): design del protocollo, flusso dei dati, decisioni sulla codifica, e perché le cose sono costruite così
- [03 - Implementation](03-IMPLEMENTATION.it.md): spiegazione del codice di ogni componente, dall'implant beacon alla dashboard React
- [04 - Challenges](04-CHALLENGES.it.md): estensioni ed esercizi più difficili (cifratura AES, DNS tunneling, tecniche di evasione)

## Problemi comuni

**"WebSocket closed before established" nella console del browser:** è React StrictMode che esegue un doppio mount in sviluppo. Il primo WebSocket si connette e si disconnette immediatamente, poi il secondo si connette e resta. Innocuo. Lo vedrai ogni volta che ricarichi la pagina in modalità dev.

**Il beacon non riesce a connettersi al server:** assicurati che i container Docker siano effettivamente in esecuzione. Esegui `just dev-ps` per controllare. Il beacon si connette tramite Nginx sulla porta 47430, quindi tutti e tre i servizi (nginx, backend, frontend) devono essere in salute. Se il backend risulta unhealthy, controlla i suoi log con `just dev-logs backend`.

**"No beacons connected" sulla dashboard:** devi eseguire l'implant beacon separatamente. Non si avvia automaticamente con Docker. Apri un secondo terminale ed esegui `just beacon`. Il beacon dovrebbe registrarsi entro pochi secondi e apparire nella tabella della dashboard.

**I comandi restituiscono output vuoto:** alcuni comandi richiedono condizioni specifiche. `screenshot` ha bisogno di un display server (fallirà in ambienti headless o container Docker). `keylog_start` ha bisogno di pynput, che richiede una sessione X11 o Wayland. `persist` funziona solo su Linux. Il comando `shell` funziona ovunque.
