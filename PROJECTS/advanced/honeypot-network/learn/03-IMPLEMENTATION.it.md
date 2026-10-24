# Implementazione

## Struttura dei File

La codebase segue il layout standard dei progetti Go. I tipi pubblici risiedono in `pkg/types/`, mentre tutto il resto si trova in `internal/`. Ciascun servizio di honeypot ha il proprio package denominato secondo la convenzione dei demoni Unix (sshd, httpd, ftpd, smbd, mysqld, redisd). Questo evita conflitti con i package della libreria standard di Go.

```
internal/
├── sshd/           # SSH (most complex: shell, filesystem, commands, keys)
│   ├── server.go   # Accept loop, auth callbacks, channel handling
│   ├── shell.go    # Interactive terminal with x/term, MOTD, command loop
│   ├── commands.go # 25+ fake command handlers
│   ├── filesystem.go # In-memory /etc/passwd, /proc/cpuinfo, etc.
│   └── hostkey.go  # Ed25519 key generation and persistence
├── httpd/          # HTTP (WordPress/phpMyAdmin fakes, scanner detection)
├── ftpd/           # FTP (PASV mode, upload capture, state machine)
├── smbd/           # SMB (NetBIOS framing, negotiate only)
├── mysqld/         # MySQL (binary wire protocol, greeting, auth, queries)
├── redisd/         # Redis (RESP protocol via tidwall/redcon)
├── event/          # Fan-out pub/sub bus + worker pool processor
├── store/          # PostgreSQL (pgxpool) + Redis streaming
├── mitre/          # ATT&CK technique index + sliding window detector
├── intel/          # IOC extraction, SSH/HTTP fingerprinting, STIX, blocklists
├── api/            # Chi REST router + WebSocket handler
├── config/         # YAML + env loading, all constants
├── session/        # Thread-safe tracker + asciicast v2 recorder
├── geo/            # MaxMind GeoIP lookup
├── ratelimit/      # Per-IP token bucket
└── ui/             # Terminal banner, colors, spinner, symbols
```

## Costruzione dell'Honeypot SSH

Il servizio SSH è l'elemento di punta. Crea una configurazione `ssh.ServerConfig` con callback che accettano qualsiasi password o chiave pubblica, mettendosi poi in ascolto delle connessioni.

Quando un client si connette, il server completa l'handshake SSH e gestisce le richieste di canale. La funzione `handleSession` crea un terminale `x/term.Terminal` con un prompt realistico, scrive un banner MOTD di Ubuntu ed entra in un ciclo read-eval-print (REPL).

Ogni comando digitato dall'attaccante viene instradato tramite `DispatchCommand` in commands.go. Questa funzione esegue il pattern matching sul nome del comando e lo delega a gestori specifici. Il comando `ls` legge dal `FakeFS` in memoria, che contiene file realistici come `/etc/passwd` con voci utente plausibili, `/proc/cpuinfo` con informazioni su CPU Intel e `/etc/ssh/sshd_config`.

L'intera sessione (ogni byte di I/O del terminale) viene registrata utilizzando `session.Recorder`, che scrive in formato asciicast v2. Questo formato utilizza JSON delimitato da newline: una riga di intestazione con le dimensioni del terminale, seguita da righe di eventi con tuple `[timestamp, "o", "output data"]`. La dashboard riproduce queste registrazioni utilizzando xterm.js.

## Costruzione dell'Honeypot MySQL

MySQL utilizza un wire protocol binario. Ciascun pacchetto presenta un'intestazione di 4 byte: 3 byte per la lunghezza del payload (little-endian) e 1 byte per il numero di sequenza.

Il server invia un pacchetto di Greeting quando un client si connette. Questo pacchetto include la versione del protocollo (10), la stringa di versione del server ("5.7.42-0ubuntu0.18.04.1"), un ID di connessione e un salt di autenticazione di 20 byte suddiviso in due parti. Il formato del greeting deve essere esatto. I client MySQL verificano specifici offset di byte e rifiutano le connessioni che presentano greeting malformati.

Dopo il greeting, il client invia un pacchetto di autenticazione contenente il nome utente e un hash della password. L'honeypot estrae l'username (stringa length-encoded a un offset noto) e pubblica un evento di login. Risponde sempre con un pacchetto OK per consentire l'accesso al client.

Nella fase di comando, l'honeypot gestisce COM_QUERY (0x03) tramite pattern matching sulle query più comuni. `SELECT @@version_comment` restituisce la stringa di versione. `SHOW DATABASES` restituisce un result set con "information_schema", "mysql", "performance_schema" e "test". La codifica del result set impiega il formato di interi length-encoded di MySQL e i pacchetti di definizione delle colonne, ciascuno con la corretta numerazione di sequenza.

## Costruzione dell'Honeypot FTP

FTP è basato su testo, il che lo rende più semplice rispetto a MySQL, ma introduce la gestione dello stato. Una sessione FTP è una macchina a stati: il client inizia in uno stato non autenticato, invia USER e PASS per autenticarsi, quindi esegue comandi.

La struct `ftpConn` traccia tre stati (init, user-provided, authenticated). Il comando USER memorizza l'username e fa avanzare lo stato. Il comando PASS pubblica un evento di credenziali e fa avanzare lo stato ad authenticated. Dopo l'autenticazione, il client può inviare comandi come PWD, CWD, LIST, STOR e altri.

La modalità PASV rappresenta la parte più interessante. Il client invia PASV e il server apre un nuovo listener TCP su una porta casuale, comunicando poi al client l'IP e la porta in un formato specifico: `227 Entering Passive Mode (h1,h2,h3,h4,p1,p2)`. Il client si connette a questo canale dati per il trasferimento dei file.

Per il caricamento dei file (STOR), l'honeypot accetta fino a 1MB di dati sul canale dati e pubblica un evento di file upload con il relativo contenuto. Questo consente di catturare campioni di malware che gli attaccanti tentano di caricare tramite FTP.

## Motore di Rilevamento MITRE

Il rilevatore in `mitre/detector.go` combina due strategie di rilevamento.

Le regole a singolo evento si attivano immediatamente all'arrivo di un evento corrispondente. Un evento di comando attiva il pattern matching a fronte di categorie di comandi note. I pattern vengono verificati a fronte della stringa del comando convertita in maiuscolo, in modo che `wget http://...` corrisponda al pattern "WGET " in `isToolTransfer()`, mappandosi su T1105.

Le regole multi-evento mantengono uno stato per singolo IP. La struct `ipState` ha due campi: `authHits` (uno slice di timestamp) e `services` (una mappa dei tipi di servizio con il relativo orario di ultimo avvistamento). A ogni evento di autenticazione, il timestamp viene aggiunto ad `authHits`, le voci più vecchie che cadono fuori dalla finestra temporale di 5 minuti vengono eliminate e, se ne rimangono 5 o più, viene rilevata la tecnica T1110 (Brute Force). A ogni evento di connessione viene registrato il tipo di servizio e, se sono stati contattati 3 o più servizi distinti nell'arco di 60 secondi, viene rilevata la tecnica T1046 (Network Service Discovery).

Il rilevatore impiega un mutex per proteggere la mappa di stato per singolo IP, poiché gli eventi provenienti da servizi differenti arrivano su goroutine distinte.

`Detect` restituisce `[]*types.MITREDetection` anziché semplici stringhe con ID di tecnica. Ciascun oggetto è completamente popolato: ID di sessione, ID di tecnica, tattica risolta dall'indice incorporato (`reconnaissance`, `credential-access`, `execution`, ecc.), confidenza (100 per tutte le corrispondenze alle regole), IP di origine, tipo di servizio, il tipo di evento scatenante come evidenza e il timestamp di rilevamento. Il processore utilizza questi oggetti in due modi: gli ID di tecnica vengono aggiunti allo slice `Tags` dell'evento in modo da rimanere nel record dell'evento per il filtraggio, e ogni rilevamento viene persistito nella tabella `mitre_detections` tramite `InsertDetection`, fornendo così alla heatmap MITRE dati reali da renderizzare.

## Esportazione STIX

Il file `intel/stix.go` genera bundle STIX 2.1 in formato JSON. Un bundle contiene un SDO Identity (che rappresenta il sistema honeypot) e un SDO Indicator per ciascun IOC.

Ogni indicatore include un'espressione di pattern STIX. Per gli indirizzi IPv4, il pattern è `[ipv4-addr:value = '1.2.3.4']`. Per gli hash dei file, è `[file:hashes.'SHA-256' = 'abc...']`. Per gli user-agent, il pattern punta a `[network-traffic:extensions.'http-request-ext'.request_header.'User-Agent' = '...']`.

Gli UUID sono generati con UUID v4 tramite `uuid.New()` della libreria google/uuid. Ciascun oggetto riceve un prefisso di tipo: `identity--uuid`, `indicator--uuid`, `bundle--uuid`. UUID v4 è la scelta corretta in questo contesto: STIX 2.1 (Sezione 2.9) specifica che gli identificatori casuali devono utilizzare UUIDv4 e gli identificatori deterministici devono utilizzare UUIDv5. UUID v7 (ordinato temporalmente) non è un formato di identificatore consentito in STIX e causerebbe errori di acquisizione su piattaforme rigorose come OpenCTI.

## Funzionamento Interno dell'Event Bus

Il bus utilizza `sync.RWMutex` di Go per la gestione dei sottoscrittori. La pubblicazione acquisisce un blocco in lettura (consentendo pubblicazioni concorrenti), mentre la sottoscrizione acquisisce un blocco in scrittura. Il percorso di pubblicazione scorre tutti i sottoscrittori, verifica se il set di argomenti del sottoscrittore include il topic dell'evento o il topic jolly "all", e invia sul canale con un `select` non bloccante.

```go
select {
case sub.ch <- ev:
default:
}
```

Il ramo `default` significa che se il canale è pieno, l'evento viene scartato silenziosamente per quel sottoscrittore. Questa è la decisione architetturale chiave: i produttori non vengono mai bloccati da consumatori lenti.

## Tracciamento delle Sessioni

Il `session.Tracker` è una mappa in memoria thread-safe da ID di sessione a `types.Session`. Utilizza un `RWMutex` con lock in lettura per le ricerche e lock in scrittura per le mutazioni.

La persistenza è collegata tramite due callback registrate all'avvio in `serve.go`. `SetOnStart` scatta quando `Start()` crea una nuova sessione — il comando serve la collega a `pgStore.InsertSession`, scrivendo il record iniziale (IP, porta, tipo di servizio, ora di inizio) in PostgreSQL prima ancora che l'attaccante si sia autenticato. `SetOnEnd` scatta quando `End()` rimuove la sessione — collegata a `pgStore.UpdateSession`, che scarica lo stato finale: conteggio comandi, tecniche MITRE, threat score, username e timestamp di fine.

Durante una sessione, i servizi chiamano `IncrCommandCount()`, `SetLogin()` e `AddTechnique()` per accumulare lo stato nella struct in memoria. Quando la connessione si chiude, `End()` la rimuove dalla mappa e attiva `onEnd` con il mutex già rilasciato, di modo che la scrittura su database non blocchi mai l'avvio o la terminazione concorrente di altre sessioni.

Il metodo `Active()` del tracker restituisce un'istantanea di tutte le sessioni in corso, utilizzata dal contatore delle "sessioni attive" della dashboard.

## Rate Limiting

Il rate limiter per singolo IP utilizza `golang.org/x/time/rate.Limiter` (algoritmo token bucket). A ciascun IP viene assegnato un proprio limiter, creato alla prima connessione. Una goroutine in background viene eseguita ogni 10 minuti e rimuove i limiter degli IP non visualizzati di recente, prevenendo la crescita della memoria.

Il rate limiter protegge l'honeypot dall'esaurimento delle risorse. Un attaccante che inviasse 10.000 connessioni al secondo genererebbe 10.000 eventi, ciascuno dei quali richiederebbe la ricerca GeoIP, il rilevamento MITRE e l'inserimento nel database. Il limiter pone un tetto a questo flusso a 10 eventi al secondo per singolo IP.

## Architettura Frontend

Il frontend in React impiega un'architettura a livelli:

**Core layer**: Client HTTP Axios con un intercettore di risposte che normalizza gli errori in oggetti tipizzati `ApiError`. Browser router React Router v7. Uno store WebSocket Zustand che mantiene un buffer dei 200 eventi live più recenti e un contatore progressivo degli eventi totali. Lo store WebSocket implementa una riconnessione con backoff esponenziale che parte da 1 secondo e si arresta a un massimo di 30 secondi, in modo che la dashboard si ripristini automaticamente dai riavvii del backend senza richiedere il ricaricamento della pagina.

**API layer**: Definizioni di tipi TypeScript che rispecchiano i tipi Go e hook TanStack Query v5 per ciascun endpoint. Gli hook utilizzano strategie di query con nome definite in `config.ts`: `live` (10s stale, 10s refetch) per il feed degli eventi, `dashboard` (15s) per le statistiche generali, `slow` (60s) per gli aggregati di paesi e credenziali e `static` (stale infinito, nessun refetch) per il catalogo delle tecniche MITRE. Gli errori di query e mutazione vengono notificati automaticamente tramite toast notification di Sonner collegate agli error handler della cache di TanStack Query, assicurando che i fallimenti delle API raggiungano sempre l'utente senza dover gestire gli errori per ciascun hook.

**Component layer**: Componenti UI riutilizzabili (StatCard, ServiceBadge, EventFeed, AttackMap, SessionPlayer) che compongono le pagine. AttackMap utilizza react-leaflet. SessionPlayer utilizza xterm.js per riprodurre le registrazioni asciicast v2 con controlli di riproduzione/pausa/velocità.

**Page layer**: Sei pagine (Dashboard, Events, Sessions, Attackers, MITRE, Intel) che combinano i componenti con i dati forniti dagli hook.

**Tooling**: Biome gestisce sia il linting che la formattazione (sostituendo ESLint + Prettier con un'unica configurazione). Zod v4 valida la conformità delle risposte API ai confini del sistema. Moduli SCSS con token di colore OKLCH (`_tokens.scss`) forniscono il design system — OKLCH offre una manipolazione del colore percettivamente uniforme per la palette di severità delle minacce.

## Build e Distribuzione

Sviluppo:
```bash
just dev-up          # Start all services via Docker Compose
just dev-serve       # Run backend locally (needs Postgres + Redis)
cd frontend && pnpm dev  # Start Vite dev server with proxy
```

Produzione:
```bash
just up -d           # Multi-stage Docker build + nginx reverse proxy
```

La build di produzione compila il binario Go con `CGO_ENABLED=0` all'interno di un container `scratch` (nessun sistema operativo, soltanto il binario), e compila l'applicazione React in file statici serviti da nginx. La configurazione di nginx inoltra via proxy `/api/*` al backend e `/ws/*` con gli header di upgrade per WebSocket.
