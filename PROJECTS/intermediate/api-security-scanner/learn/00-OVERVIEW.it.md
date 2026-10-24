# 00-OVERVIEW.md

# API Security Scanner

## Cos'è

Un'applicazione web full-stack che testa le API alla ricerca di vulnerabilità di sicurezza comuni. Puntala su un qualsiasi endpoint API e controllerà debolezze nel rate limiting, bypass dell'autenticazione, falle di SQL injection e difetti di autorizzazione. Realizzata con FastAPI e React, è pensata per insegnarti come gli attaccanti sondano le API e come difendersi.

## Perché è importante

Le API sono ovunque. La tua app mobile parla con un'API. La tua dashboard web attinge da un'API. Quell'integrazione di terze parti che hai aggiunto la settimana scorsa? API. E sono costantemente sotto attacco. Nel 2023, gli attacchi alle API sono aumentati del 400% su base annua secondo la ricerca di Salt Security. Il breach di Optus nel 2022 ha esposto 10 milioni di record di clienti attraverso un endpoint API non autenticato. Il breach dell'API di T-Mobile nello stesso anno ha fatto trapelare dati su 37 milioni di clienti.

La maggior parte di questi breach si riduce a una manciata di problemi che si ripresentano di continuo. L'assenza di rate limit permette agli attaccanti di martellare gli endpoint. Un'autenticazione difettosa accetta token non validi. La SQL injection funziona ancora perché gli sviluppatori concatenano l'input dell'utente nelle query. Le vulnerabilità IDOR permettono agli utenti di accedere ai dati gli uni degli altri semplicemente cambiando un ID nell'URL.

Questo progetto ti insegna a trovare questi problemi prima che lo facciano gli attaccanti.

**Scenari reali in cui si applica:**
- Team di sicurezza che scansionano le proprie API prima dei rilasci
- Bug bounty hunter che cercano vulnerabilità nelle API pubbliche
- DevOps engineer che verificano che il rate limiting funzioni davvero
- Penetration tester che controllano se la validazione JWT può essere aggirata
- Sviluppatori che imparano da quali attacchi il loro codice deve difendersi

## Cosa imparerai

Questo progetto ti insegna come funziona sotto il cofano il testing di sicurezza delle API. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**
- OWASP API Security Top 10 2023 - non solo l'elenco, ma come appare ogni vulnerabilità nel codice reale e come rilevarla automaticamente
- Tecniche di bypass del rate limiting - come gli attaccanti ruotano gli IP, falsificano gli header e manipolano gli endpoint per eludere i limiti
- Vulnerabilità JWT - perché l'attacco con algoritmo "none" funziona, cosa succede quando le firme non vengono verificate e come testare la presenza di secret deboli
- Rilevamento della SQL injection - tecniche error-based, boolean blind e time-based blind con analisi statistica per ridurre i falsi positivi
- Testing IDOR/BOLA - trovare lacune di autorizzazione che permettono agli utenti di accedere a oggetti a cui non dovrebbero

**Competenze tecniche:**
- Costruire scanner di sicurezza con throttling delle richieste, logica di retry e analisi dei tempi di baseline (vedi `backend/scanners/base_scanner.py:40-90`)
- Implementare un'architettura a livelli con repository, service e route handler che tengono la business logic separata dall'accesso ai dati
- Flussi di autenticazione JWT, dall'hashing delle password alla creazione dei token fino alla validazione sugli endpoint protetti
- Implementazione e rilevamento del rate limiting usando sia l'analisi delle risposte sia l'ispezione degli header
- Gestire richieste HTTP concorrenti con un corretto connection pooling, timeout e recupero dagli errori

**Strumenti e tecniche:**
- FastAPI con async/await per gestire più scansioni in concorrenza senza bloccare
- SQLAlchemy con Alembic per le migrazioni del database e la gestione delle relazioni
- Build Docker multi-stage che separano lo sviluppo (hot reload) dalla produzione (ottimizzata, multi-worker)
- React con TanStack Query per gestire lo stato del server, il refetch automatico e gli aggiornamenti ottimistici
- Configurazione di Nginx come reverse proxy per instradare /api verso il backend e servire i file statici del frontend

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**
- Basi di Python - funzioni, classi, async/await. Leggerai codice come `async def make_request(self, method: str, endpoint: str, **kwargs: Any)` e dovrai capire cosa succede
- Fondamenti di HTTP - cosa significano GET/POST, come funzionano gli header, cosa indicano gli status code. Gli scanner analizzano le risposte per rilevare le vulnerabilità
- Basi di SQL - SELECT, clausole WHERE, cosa fa una JOIN. Vedrai come i payload di SQL injection cercano di manipolare le query
- Concetti di REST API - endpoint, richiesta/risposta, autenticazione. L'intero progetto ruota attorno al testing delle API

**Strumenti di cui avrai bisogno:**
- Docker e Docker Compose - l'intero stack gira in container
- Python 3.11+ - il codice backend usa type hint moderni e pattern matching
- Node.js 20+ - la build del frontend richiede un npm recente
- Conoscenza di PostgreSQL utile - i dati sono memorizzati in Postgres con relazioni tra utenti, scansioni e risultati dei test

**Utili ma non obbligatori:**
- React e TypeScript - il frontend è completamente implementato, ma capirlo aiuta
- Esperienza pregressa di security testing - insegneremo i concetti, ma una precedente familiarità con tool come Burp Suite o OWASP ZAP fornisce contesto
- Familiarità con FastAPI - usiamo pesantemente la dependency injection e la validazione Pydantic

## Avvio rapido

Fai girare il progetto in locale:

```bash
# Clone and navigate (assuming you're already in PROJECTS/intermediate)
cd api-security-scanner

# Copy environment template
cp .env.example .env

# CRITICAL: Edit .env and change SECRET_KEY to something random
# In production this must be a cryptographically secure random string

# Start development environment (includes hot reload)
docker compose -f dev.compose.yml up --build

# Wait for all services to be healthy, then visit:
# - Frontend: http://localhost (or http://localhost:80)
# - Backend API docs: http://localhost:8000/docs
# - Direct API: http://localhost:8000
```

Output atteso: dovresti vedere la pagina di login. Crea un account, poi puoi iniziare a scansionare. Prova a scansionare `https://httpbin.org/get` con tutti e quattro i test abilitati. Vedrai risultati sicuri perché httpbin è progettato per il testing.

## Struttura del progetto

```
api-security-scanner/
├── backend/
│   ├── config.py              # All environment variables and constants
│   ├── core/                  # Infrastructure (database, security, dependencies)
│   ├── models/                # SQLAlchemy models (User, Scan, TestResult)
│   ├── repositories/          # Data access layer, queries isolated here
│   ├── routes/                # FastAPI endpoints (auth, scans)
│   ├── scanners/              # Security testing logic (SQLi, auth, etc)
│   ├── schemas/               # Pydantic validation schemas
│   └── services/              # Business logic layer
├── frontend/
│   └── src/
│       ├── components/        # React UI components
│       ├── hooks/             # TanStack Query hooks for API calls
│       ├── services/          # API client functions
│       └── store/             # Zustand state management
├── conf/
│   ├── docker/                # Dockerfiles for dev and prod
│   └── nginx/                 # Nginx reverse proxy configs
└── compose.yml                # Production Docker Compose
└── dev.compose.yml            # Development with volume mounts
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per imparare i fondamenti di sicurezza alla base di rate limiting, autenticazione, SQL injection e IDOR
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere come è progettato il sistema di scanner, perché usiamo il repository pattern e come i dati fluiscono attraverso i livelli
3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per una spiegazione dettagliata di come funziona ogni scanner, con un'analisi riga per riga del codice reale
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee su cui costruire, dall'aggiunta del rilevamento XSS all'implementazione di plugin di scanner personalizzati

## Problemi comuni

**I container Docker non si avviano**
```
Error: port is already allocated
```
Soluzione: un altro servizio sta usando la porta 80, 5432 o 8000. Controlla `docker ps` e ferma i container in conflitto, oppure modifica `.env` per cambiare `HOST_NGINX_PORT`, `HOST_DB_PORT` o `HOST_BACKEND_PORT`

**Il frontend non riesce a raggiungere il backend**
```
Network Error / Failed to fetch
```
Soluzione: controlla `VITE_API_URL` in `.env`. Per il setup Docker, dovrebbe essere `http://localhost/api`. Se esegui il backend direttamente (non in Docker), usa `http://localhost:8000`

**Le migrazioni del database falliscono**
```
sqlalchemy.exc.OperationalError: could not connect to server
```
Soluzione: il container del database non è ancora pronto. Attendi 10 secondi e riprova. L'healthcheck in `dev.compose.yml:13-17` garantisce che Postgres accetti connessioni prima che il backend si avvii

**Le scansioni vanno in timeout su ogni test**
```
All tests return ERROR status
```
Soluzione: l'URL di destinazione potrebbe bloccare le richieste. Prova prima `https://httpbin.org/get` per verificare che lo scanner funzioni. Controlla `SCANNER_CONNECTION_TIMEOUT` in `config.py:51` - il default è 180 secondi

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **network-traffic-analyzer** [network-traffic-analyzere](https://github.com/CarterPerez-dev/Cybersecurity-Projects/tree/main/PROJECTS/beginner/network-traffic-analyzer) - Approfondisce l'analisi dei pacchetti e la dissezione dei protocolli, e completa questo progetto mostrando cosa succede a livello di rete
- **docker-security-audit** [docker-security-audit](https://github.com/CarterPerez-dev/Cybersecurity-Projects/tree/main/PROJECTS/intermediate/docker-security-audit) - Si concentra sulla sicurezza dei container, ti insegna a scansionare immagini e configurazioni Docker alla ricerca di vulnerabilità
- **bug-bounty-platform** [bug-bounty-platform](https://github.com/CarterPerez-dev/Cybersecurity-Projects/tree/main/PROJECTS/advanced/bug-bounty-platform) - Piattaforma completa per gestire le scoperte di sicurezza, mostra come i team di sicurezza professionali tracciano e risolvono i problemi

