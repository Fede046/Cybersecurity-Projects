<!-- © AngelaMos | 2026 | 02-ARCHITECTURE.md -->

# Architettura: Come si integrano i componenti del sistema C2

Questo documento illustra ciascun componente del progetto C2 beacon, come si collegano tra loro, come fluiscono i dati e perché le decisioni di progettazione sono state prese in questo modo. Alla fine dovresti essere in grado di tracciare un comando dalla dashboard dell'operatore fino al beacon che lo esegue sulla macchina bersaglio, e viceversa.

Tempo di lettura stimato: 20-30 minuti.

---

## Indice dei contenuti

1. [Architettura ad alto livello](#architettura-ad-alto-livello)
2. [Analisi dettagliata dei componenti](#analisi-dettagliata-dei-componenti)
3. [Flusso dei dati](#flusso-dei-dati)
4. [Progettazione del protocollo](#progettazione-del-protocollo)
5. [Schema del database](#schema-del-database)
6. [Decisioni architetturali](#decisioni-architetturali)
7. [Architettura di sicurezza](#architettura-di-sicurezza)
8. [Architettura di deployment](#architettura-di-deployment)
9. [Riferimento ai file chiave](#riferimento-ai-file-chiave)

---

## Architettura ad alto livello

Ecco il quadro completo di ogni parte in movimento e di come comunicano tra loro:

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Network                        │
│                                                              │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │  Nginx   │────>│ Backend  │     │ Frontend │            │
│  │  :47430  │     │  :8000   │     │  :5173   │            │
│  └──────────┘     └──────────┘     └──────────┘            │
│       │                │                                     │
│       │           ┌────┴────┐                               │
│       │           │ SQLite  │                               │
│       │           │  (WAL)  │                               │
│       │           └─────────┘                               │
└───────┼─────────────────────────────────────────────────────┘
        │
   ┌────┴────┐
   │ Beacon  │  (in esecuzione sulla macchina bersaglio, fuori da Docker)
   └─────────┘
```

Cinque componenti, tre reti, un protocollo. Analizziamoli uno per uno.

### Nginx (porta 47430)

Nginx è l'unico punto di ingresso per tutto. Ogni richiesta, che provenga da un beacon connesso tramite WebSocket, da un operatore che visualizza la dashboard o da una chiamata API REST, entra attraverso la porta 47430. Nginx ispeziona il percorso dell'URL e instrada di conseguenza:

- `/api/ws/*` va agli handler WebSocket del backend. Questo copre sia `/api/ws/beacon` (per le connessioni del beacon) sia `/api/ws/operator` (per le connessioni della dashboard dell'operatore). Nginx aggiorna queste richieste a connessioni WebSocket impostando gli header `Upgrade` e `Connection`.
- `/api/*` (non-WebSocket) va agli endpoint REST del backend. Questi servono gli elenchi dei beacon, la cronologia dei task, gli health check e la documentazione OpenAPI.
- Tutto il resto (`/`) va al frontend. In sviluppo ciò significa fare da proxy verso il dev server di Vite sulla porta 5173. In produzione Nginx serve direttamente i file statici precompilati dal disco.

La configurazione di nginx risiede in `infra/nginx/dev.nginx` per lo sviluppo e in `infra/nginx/prod.nginx` per la produzione. La configurazione di base condivisa in `infra/nginx/nginx.conf` definisce gli upstream, le zone di rate limiting, le impostazioni gzip e la mappa di upgrade WebSocket.

Un dettaglio degno di nota: le location WebSocket hanno `proxy_read_timeout 3600s`. Le normali location HTTP usano timeout di 30-60 secondi. Le connessioni WebSocket sono persistenti per loro natura, quindi nginx deve sapere di non terminarle dopo un minuto di apparente inattività.

### Backend (porta 8000)

Il backend è un'applicazione asincrona FastAPI che svolge tre compiti simultaneamente:

1. **Gestione dei beacon.** Accetta connessioni WebSocket dai beacon, convalida i loro messaggi REGISTER, li traccia in memoria e in SQLite e distribuisce i task tramite code asincrone dedicate per ciascun beacon.
2. **Gestione degli operatori.** Accetta connessioni WebSocket dalle dashboard degli operatori, invia loro eventi in tempo reale (beacon connesso, heartbeat, risultato del task) e riceve richieste di invio task.
3. **API REST.** Serve elenchi di beacon e cronologia dei task per qualsiasi client che ne abbia bisogno, oltre agli health check per l'orchestrazione dei container di Docker.

Il backend usa il context manager `lifespan` di FastAPI per inizializzare il database, il registry dei beacon, il task manager e l'operator broadcast manager all'avvio. Tutti e tre i singleton sono collegati ad `app.state` in modo che ogni handler di richiesta possa accedervi. Il pattern dell'app factory in `backend/app/__main__.py` collega tutto insieme: middleware CORS, router per il WebSocket del beacon, WebSocket dell'operatore ed endpoint REST.

### Frontend (porta 5173 in dev)

Il frontend è un'applicazione React creata con Vite, TypeScript e moduli SCSS. Si collega al backend attraverso un unico WebSocket a `/api/ws/operator` e mantiene l'intero stato in uno store Zustand.

Quando la dashboard viene caricata, l'hook `useOperatorSocket` apre una connessione WebSocket. Il server invia immediatamente un messaggio `beacon_list` con tutti i beacon noti. Da quel momento, il frontend riceve eventi in tempo reale: `beacon_connected`, `beacon_disconnected`, `heartbeat`, `task_result` e `task_submitted`. Lo store Zustand in `frontend/src/core/ws.ts` elabora ciascun tipo di evento e aggiorna la corrispondente porzione di stato.

L'operatore può selezionare un beacon dalla dashboard e aprire una pagina di sessione, che fornisce un'interfaccia simile a un terminale per inviare comandi. Quando l'operatore invia un task, il frontend trasmette un messaggio `submit_task` sullo stesso WebSocket. Il server risponde con una conferma `task_submitted` che mappa l'ID locale sull'ID del task generato dal server, e successivamente trasmette in broadcast il `task_result` quando il beacon invia il resoconto.

### SQLite (modalità WAL)

Il database è un singolo file SQLite posizionato in `backend/data/c2.db`. Memorizza tre tabelle: `beacons`, `tasks` e `task_results`. Il backend inizializza lo schema all'avvio in `backend/app/database.py` e abilita la modalità WAL (Write-Ahead Logging) con `PRAGMA journal_mode=WAL`.

La modalità WAL è fondamentale in questo contesto perché il backend gestisce letture concorrenti. Mentre una coroutine scrive il risultato di un task, un'altra potrebbe leggere l'elenco dei beacon per una query dell'operatore. Il WAL consente a lettori e scrittori di operare simultaneamente senza bloccarsi a vicenda. Nella modalità rollback journal predefinita, una scrittura bloccherebbe l'intero database e tutte le letture fino al commit della scrittura.

Il context manager asincrono `get_db()` restituisce ogni volta una nuova connessione `aiosqlite`, con `row_factory` impostato su `aiosqlite.Row` in modo che i risultati delle query vengano restituiti come oggetti simil-dizionario invece che come semplici tuple. Le foreign key vengono abilitate su ogni connessione.

### Beacon (in esecuzione sul target)

Il beacon è uno script Python autonomo posizionato in `beacon/beacon.py` che viene eseguito all'esterno dello stack Docker, su qualsiasi macchina gli venga indicata. Possiede una propria copia delle funzioni di codifica XOR (deliberatamente duplicate in modo che il beacon abbia zero dipendenze dalla codebase del server) e una tabella di dispatch che mappa le stringhe di comando sulle funzioni handler.

All'avvio, il beacon si connette al server tramite WebSocket, invia un messaggio REGISTER con i metadati di sistema (hostname, OS, username, PID, IP interno, architettura), quindi entra nel suo ciclo principale. Invia heartbeat periodici applicando del jitter all'intervallo di sleep, e attende i messaggi TASK dal server. Quando arriva un task, il beacon lo inoltra all'handler appropriato, raccoglie l'output e invia un messaggio RESULT al server.

Se la connessione cade, il beacon usa un backoff esponenziale (partendo da 2 secondi, con tetto massimo a 5 minuti) per riconnettersi. Questo è il comportamento standard per gli impianti reali, perché tempestare un server irraggiungibile alla massima velocità è il modo più rapido per farsi notare dai sistemi di monitoraggio di rete.

---

## Analisi dettagliata dei componenti

### Livelli del Backend

Il backend segue una struttura a livelli che separa gli aspetti del protocollo dalla logica di business e dalla persistenza:

```
backend/app/
├── __main__.py
├── config.py
├── database.py
├── core/
│   ├── encoding.py
│   ├── protocol.py
│   └── models.py
├── beacon/
│   ├── registry.py
│   ├── router.py
│   └── tasking.py
└── ops/
    ├── manager.py
    └── router.py
```

**`__main__.py`** è la factory dell'applicazione. Crea l'istanza FastAPI, configura CORS, imposta il logging, inizializza lo schema del database e istanzia i tre singleton principali (BeaconRegistry, TaskManager, OpsManager) durante l'avvio nel lifespan. La funzione `create_app()` monta tutti i router e definisce l'endpoint di health check. Uvicorn esegue questo file direttamente tramite `app.__main__:app`.

**`config.py`** usa Pydantic Settings per caricare la configurazione dalle variabili d'ambiente e dal file `.env`. Definisce la chiave XOR, il percorso del database, le origini CORS, il livello di log e le opzioni di bind del server. Il decoratore `@lru_cache` su `get_settings()` assicura che il file `.env` venga analizzato esattamente una volta.

**`database.py`** gestisce la definizione dello schema SQLite e fornisce due funzioni: `init_db()` per la creazione una tantum dello schema all'avvio, e `get_db()` come context manager asincrono per connessioni al database per singola richiesta.

**`core/encoding.py`** implementa il cifrario XOR con chiave ripetuta unito all'incapsulamento Base64. La funzione `encode()` accetta una stringa in chiaro e una chiave, esegue lo XOR dei byte UTF-8 con i byte della chiave (facendo scorrere la chiave ciclicamente) e restituisce una stringa ASCII in Base64. La funzione `decode()` inverte il processo. Questo sistema è simmetrico, quindi la stessa chiave funziona in entrambe le direzioni.

**`core/protocol.py`** definisce l'involucro `Message` (un modello Pydantic con un campo `type` e un dizionario `payload`) e l'enum `MessageType` (REGISTER, HEARTBEAT, TASK, RESULT, ERROR). La funzione `pack()` serializza un Message in JSON e lo codifica con XOR+Base64. La funzione `unpack()` decodifica una stringa grezza, analizza il JSON e lo convalida ricostruendo un Message. Se un passaggio fallisce (Base64 errato, JSON errato, schema non valido), solleva un `ValueError`.

**`core/models.py`** definisce i modelli di dati Pydantic utilizzati in tutto il sistema: `BeaconMeta` (payload di registrazione), `BeaconRecord` (record completo nel database), `TaskRequest` (richiesta inoltrata dall'operatore), `TaskRecord` (task salvato nel database) e `TaskResult` (risposta del beacon). Definisce anche `CommandType` come StrEnum con l'elenco di tutti i comandi supportati: shell, sysinfo, proclist, upload, download, screenshot, keylog_start, keylog_stop, persist e sleep.

**`beacon/registry.py`** contiene la classe `BeaconRegistry`. Mantiene un dizionario di `beacon_id -> WebSocket` per le connessioni attive e fornisce metodi per registrare, annullare la registrazione, verificare lo stato attivo, aggiornare i timestamp degli heartbeat e interrogare il database per i record dei beacon. Quando un beacon si registra, esegue un upsert nella tabella `beacons` (INSERT con ON CONFLICT DO UPDATE), così i beacon che si riconnettono aggiornano i propri metadati senza creare duplicati.

**`beacon/router.py`** definisce l'endpoint WebSocket `/ws/beacon`. Questo è il componente più complesso del backend. Quando un beacon si connette, l'handler accetta il WebSocket, attende un messaggio REGISTER, lo valida, registra il beacon, quindi avvia due task asyncio concorrenti: `_send_tasks` (che si blocca sulla coda dei task del beacon e invia messaggi TASK) e `_receive_messages` (che elabora i messaggi in arrivo HEARTBEAT e RESULT). Il pattern `asyncio.wait` con `FIRST_COMPLETED` assicura che se una delle coroutine fallisce (generalmente a causa della disconnessione del WebSocket), l'altra venga annullata in modo pulito.

**`beacon/tasking.py`** contiene la classe `TaskManager`. Mantiene un dizionario di `beacon_id -> asyncio.Queue[TaskRecord]` per l'invio dei task a ciascun beacon. Quando un operatore invia un task, `submit()` lo scrive in SQLite e lo inserisce nella coda. La coroutine `_send_tasks` nel router del beacon chiama `get_next()`, che rimane in attesa finché non compare un task. Quando viene restituito un risultato, `store_result()` lo scrive nella tabella `task_results` e contrassegna il task genitore come completato.

**`ops/manager.py`** contiene la classe `OpsManager`. Traccia un insieme di connessioni WebSocket degli operatori e fornisce un metodo `broadcast()` che invia un evento JSON a tutti gli operatori connessi. Se un invio fallisce (connessione già chiusa), il WebSocket non valido viene rimosso dall'insieme. Questo pattern fan-out fa sì che ogni operatore veda ogni evento in tempo reale.

**`ops/router.py`** definisce due router: `ws_router` per l'endpoint WebSocket `/ws/operator` e `rest_router` per gli endpoint REST (`/beacons`, `/beacons/{id}`, `/beacons/{id}/tasks`). Quando un operatore si connette via WebSocket, l'handler invia immediatamente l'elenco completo dei beacon, quindi entra in un ciclo di ricezione in attesa di messaggi `submit_task`. Gli endpoint REST forniscono gli stessi dati per i client non-WebSocket o per i caricamenti iniziali della pagina.

---

## Flusso dei dati

### Flusso di registrazione del beacon

Ecco cosa accade dal momento in cui un beacon si avvia fino a quando l'operatore lo vede nella propria dashboard:

```
Beacon                    Server                    Operator Dashboard
  │                         │                              │
  │──REGISTER(sysinfo)────>│                              │
  │                         │──INSERT beacon──> SQLite     │
  │                         │                              │
  │                         │──beacon_connected──────────>│
  │                         │                              │
  │<──(connessione attiva)─│                              │
  │                         │                              │
  │──HEARTBEAT────────────>│                              │
  │                         │──UPDATE last_seen─> SQLite   │
  │                         │──heartbeat────────────────>│
```

Passo dopo passo:

1. Il beacon apre un WebSocket verso `/api/ws/beacon` (che nginx inoltra come proxy al backend su `/ws/beacon`).
2. Il beacon invia un messaggio REGISTER contenente i metadati di sistema: hostname, sistema operativo, username, PID, IP interno e architettura. Questo payload passa attraverso la codifica XOR+Base64 prima di essere trasmesso sul canale.
3. L'handler `beacon_websocket` del server in `beacon/router.py` accetta la connessione, estrae il primo messaggio, verifica che sia di tipo REGISTER ed estrae i campi di `BeaconMeta`.
4. Il server chiama `registry.register()`, che memorizza il riferimento WebSocket in memoria ed esegue l'upsert del record del beacon in SQLite. Se questo ID beacon era già stato visto (una riconnessione), i metadati vengono aggiornati e `first_seen` viene preservato.
5. Il server trasmette in broadcast un evento `beacon_connected` a tutte le connessioni WebSocket degli operatori tramite `ops_manager.broadcast()`.
6. Lo store Zustand della dashboard dell'operatore riceve l'evento, chiama `upsertBeacon()`, e la tabella della dashboard si riaggiorna mostrando il nuovo beacon con un indicatore verde "active".
7. Il server avvia le coroutine `_send_tasks` e `_receive_messages`. La connessione rimane ora aperta a tempo indeterminato.
8. Ogni pochi secondi (con jitter), il beacon invia un messaggio HEARTBEAT. Il server aggiorna `last_seen` in SQLite e trasmette l'heartbeat agli operatori, mantenendo aggiornato il timestamp "last seen" sulla dashboard.

### Flusso di esecuzione dei task

Ecco l'intero percorso di andata e ritorno quando un operatore invia un comando:

```
Operatore                 Server                    Beacon
  │                         │                         │
  │──submit_task──────────>│                         │
  │                         │──INSERT task──> SQLite  │
  │                         │──queue.put(task)        │
  │<──task_submitted───────│                         │
  │                         │                         │
  │                         │──TASK(XOR+B64)────────>│
  │                         │                         │──esegui
  │                         │<──RESULT(XOR+B64)──────│
  │                         │──INSERT result─> SQLite │
  │                         │──UPDATE task status     │
  │<──task_result──────────│                         │
```

Passo dopo passo:

1. L'operatore digita un comando nell'interfaccia terminale della sessione e preme invio. La funzione `sendTask()` del frontend invia un messaggio JSON tramite il WebSocket dell'operatore: `{"type": "submit_task", "payload": {"beacon_id": "...", "command": "shell", "args": "whoami", "local_id": "..."}}`. Questo messaggio è in chiaro JSON perché il canale operatore-server viaggia interamente all'interno di Docker.
2. L'handler `operator_websocket` del server in `ops/router.py` riceve il messaggio, crea un `TaskRecord` con un UUID e chiama `task_manager.submit()`.
3. `submit()` esegue due azioni: inserisce il task nella tabella `tasks` in SQLite (così persiste anche se il server si riavvia), e mette il TaskRecord nella `asyncio.Queue` specifica di quel beacon.
4. Il server invia una risposta `task_submitted` all'operatore con la mappatura tra `local_id` e `task_id`. Il frontend la userà per correlare i risultati in seguito.
5. Nel frattempo, in `beacon/router.py`, la coroutine `_send_tasks` per questo beacon era bloccata su `task_manager.get_next(beacon_id)`. Il `queue.put()` la sblocca. La coroutine impacchetta il task in un Message, lo codifica con XOR+Base64 e lo invia al beacon tramite WebSocket.
6. Il beacon riceve il testo grezzo, lo decodifica (Base64 decode, decifratura XOR, parse JSON) e instrada il comando all'handler appropriato. Per un comando `shell`, ciò significa chiamare `asyncio.create_subprocess_shell()` e catturare stdout/stderr.
7. Il beacon impacchetta il risultato (task_id, output, error) in un messaggio RESULT, lo codifica e lo rimanda al server.
8. La coroutine `_receive_messages` del server riceve il RESULT, lo decodifica, crea un oggetto `TaskResult` e chiama `task_manager.store_result()`. Questo scrive il risultato nella tabella `task_results` e aggiorna lo stato del task genitore a `completed`.
9. Il server trasmette un evento `task_result` a tutte le connessioni WebSocket degli operatori.
10. Lo store Zustand della dashboard dell'operatore riceve l'evento, chiama `addTaskResult()`, e il terminale della sessione mostra l'output.

L'intero ciclo di andata e ritorno richiede tipicamente meno di un secondo su una rete locale. Il collo di bottiglia è solitamente l'esecuzione del comando sulla macchina bersaglio.

---

## Progettazione del protocollo

### Tipi di messaggio

Ogni messaggio tra beacon e server segue la stessa struttura di involucro (envelope): una stringa `type` e un dizionario `payload`. Ecco tutti e cinque i tipi di messaggio:

| Tipo | Direzione | Campi del payload | Scopo |
|------|-----------|-------------------|-------|
| REGISTER | Beacon -> Server | hostname, os, username, pid, internal_ip, arch, id | Handshake iniziale quando il beacon si connette per la prima volta |
| HEARTBEAT | Beacon -> Server | id | Segnale di keep-alive, aggiorna il timestamp last_seen |
| TASK | Server -> Beacon | id, command, args | Trasmette un comando che il beacon deve eseguire |
| RESULT | Beacon -> Server | task_id, output, error | Restituisce l'output del comando al server |
| ERROR | Entrambe le direzioni | message | Segnala un errore di protocollo o di esecuzione |

Il modello `Message` in `backend/app/core/protocol.py` usa un `BaseModel` di Pydantic con `type: MessageType` (uno StrEnum) e `payload: dict[str, Any]`. Lo StrEnum garantisce che il campo type sia convalidato rispetto all'insieme noto dei tipi di messaggio durante l'estrazione. Se un beacon invia un messaggio con `"type": "BOGUS"`, la validazione di Pydantic lo intercetta e il server solleva un ValueError.

### Pipeline di codifica

Il canale beacon-server e il canale operatore-server utilizzano strategie di codifica diverse:

```
                    Beacon <-> Server
Stringa JSON -> Byte UTF-8 -> XOR(key) -> Base64 -> Frame di testo WebSocket

                    Server <-> Operatore
Stringa JSON -> Frame di testo WebSocket (nessuna codifica)
```

Il canale beacon-server applica la codifica XOR+Base64 su ogni messaggio. La funzione `encode()` in `backend/app/core/encoding.py` accetta una stringa JSON in chiaro, la converte in byte UTF-8, applica lo XOR a ciascun byte con il byte corrispondente della chiave ripetuta, e codifica in Base64 il risultato in una stringa ASCII adatta a un frame di testo WebSocket. La funzione `decode()` inverte il processo: decodifica Base64, XOR con la stessa chiave, decodifica UTF-8.

Il canale operatore-server invia JSON in chiaro. Nessuna codifica.

Perché questa differenza? Il canale beacon-server simula comunicazioni C2 cifrate. In un framework C2 reale, questo canale userebbe AES-256 o ChaCha20 con una chiave per sessione negoziata tramite uno scambio asimmetrico. Il nostro cifrario XOR è un segnaposto che illustra il concetto senza la complessità della crittografia reale. Mostra dove avviene la codifica nella pipeline, come funzionano le chiavi simmetriche e perché è necessaria la stessa chiave su entrambi i lati.

Il canale operatore-server evita la codifica perché l'operatore è una parte fidata che opera sulla stessa infrastruttura. La dashboard dell'operatore si connette tramite nginx sulla stessa rete Docker. Non c'è una rete non fidata in mezzo, quindi codificare i messaggi aggiungerebbe latenza e complessità senza alcun beneficio di sicurezza. In un C2 di produzione vorresti comunque TLS su questo canale, ma la codifica a livello applicativo che protegge il traffico del beacon non è necessaria per il traffico dell'operatore.

### Perché XOR e non AES

Lo XOR con chiave ripetuta è uno dei cifrari più deboli che esistano. È vulnerabile agli attacchi con testo in chiaro noto, all'analisi delle frequenze e al recupero della chiave se si conosce qualsiasi porzione del testo in chiaro. Lo abbiamo scelto deliberatamente perché:

- È facile da capire. Puoi verificare la codifica a mano con una calcolatrice.
- È simmetrico. La stessa funzione cifra e decifra.
- Dimostra l'intera pipeline. Le funzioni encode/decode/pack/unpack avrebbero le stesse signature anche con AES.
- Sostituirlo in seguito con un cifrario reale richiede di modificare esattamente due funzioni in `encoding.py` e i loro equivalenti nel beacon.

La chiave è configurata in `.env` come `XOR_KEY` e caricata tramite Pydantic Settings in `backend/app/config.py`. Il beacon la legge dalla variabile d'ambiente `C2_XOR_KEY`. Entrambe le parti devono avere la stessa chiave, che è la proprietà fondamentale della crittografia simmetrica.

---

## Schema del database

Il database possiede tre tabelle con relazioni di foreign key:

```
beacons (id PK) <--- tasks (beacon_id FK) <--- task_results (task_id FK)
```

### beacons

```sql
CREATE TABLE IF NOT EXISTS beacons (
    id          TEXT PRIMARY KEY,
    hostname    TEXT NOT NULL,
    os          TEXT NOT NULL,
    username    TEXT NOT NULL,
    pid         INTEGER NOT NULL,
    internal_ip TEXT NOT NULL,
    arch        TEXT NOT NULL,
    first_seen  TEXT NOT NULL,
    last_seen   TEXT NOT NULL
);
```

Ogni riga rappresenta un beacon che si è connesso almeno una volta. L'`id` è un UUID generato dal beacon al primo avvio. `first_seen` viene impostato alla registrazione iniziale e non viene mai modificato. `last_seen` viene aggiornato a ogni heartbeat e alla disconnessione. Le restanti colonne catturano i metadati di sistema raccolti dal beacon durante la registrazione.

La registrazione usa INSERT con ON CONFLICT(id) DO UPDATE, quindi se un beacon si riconnette con lo stesso ID, i metadati vengono aggiornati ma `first_seen` viene preservato dalla riga originale.

### tasks

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id           TEXT PRIMARY KEY,
    beacon_id    TEXT NOT NULL,
    command      TEXT NOT NULL,
    args         TEXT,
    status       TEXT NOT NULL DEFAULT 'pending',
    created_at   TEXT NOT NULL,
    completed_at TEXT,
    FOREIGN KEY (beacon_id) REFERENCES beacons(id)
);
```

Ogni riga rappresenta un task inviato da un operatore per uno specifico beacon. L'`id` è un UUID generato dal server. `command` è uno dei valori dell'enum `CommandType` (shell, sysinfo, proclist, ecc.). `args` è un payload di tipo stringa facoltativo il cui significato dipende dal tipo di comando. Per un comando shell, è la stringa del comando shell. Per un upload, è un blob JSON con nome file e contenuto. Per sleep, è un blob JSON con intervallo e jitter.

`status` inizia come `'pending'` e passa a `'completed'` quando viene restituito un risultato. `created_at` viene impostato al momento dell'invio, `completed_at` viene impostato quando il risultato viene memorizzato.

### task_results

```sql
CREATE TABLE IF NOT EXISTS task_results (
    id         TEXT PRIMARY KEY,
    task_id    TEXT NOT NULL UNIQUE,
    output     TEXT,
    error      TEXT,
    created_at TEXT NOT NULL,
    FOREIGN KEY (task_id) REFERENCES tasks(id)
);
```

Ogni riga è il risultato di un task completato. La foreign key `task_id` è UNIQUE, imponendo una relazione uno-a-uno con la tabella tasks (ogni task produce esattamente un risultato). `output` contiene lo stdout del comando o JSON strutturato, a seconda del tipo di comando. `error` contiene stderr o un messaggio di errore. Entrambi possono essere NULL.

### Perché SQLite con modalità WAL

SQLite è la scelta corretta per questo progetto per vari motivi:

**Singolo server, singolo writer.** Il nostro server C2 è un unico processo. Non c'è un cluster di istanze backend in competizione per l'accesso al database. SQLite gestisce perfettamente i carichi di lavoro a singolo scrittore.

**La modalità WAL gestisce letture concorrenti.** Il backend è asincrono, quindi più coroutine potrebbero leggere dal database contemporaneamente (un operatore che interroga l'elenco dei beacon mentre il server elabora un heartbeat). La modalità WAL permette letture concorrenti senza blocchi, a differenza della modalità rollback journal predefinita. La funzione `init_db()` in `database.py` abilita il WAL con `PRAGMA journal_mode=WAL`.

**Zero configurazione.** SQLite è integrato nel processo Python. Non c'è alcun server di database separato da configurare, nessun connection pooling da ottimizzare, nessuna latenza di rete tra applicazione e database. Il database è un singolo file in `backend/data/c2.db`.

**Adatto allo scopo didattico.** Un framework C2 di produzione che debba supportare più team server, distribuzione geografica o centinaia di beacon concorrenti userebbe PostgreSQL. Per un progetto educativo a server singolo, SQLite mantiene l'attenzione sull'architettura C2 anziché sull'amministrazione del database.

---

## Decisioni architetturali

### Perché WebSockets rispetto all'HTTP Polling

Un sistema C2 richiede comunicazione bidirezionale. Il server deve inviare task ai beacon non appena un operatore li assegna, e i beacon devono rimandare i risultati man mano che completano i comandi. Esistono diversi modi per realizzare ciò:

L'**HTTP polling** prevede che il beacon invii una richiesta GET ogni N secondi chiedendo "hai un task per me?". Funziona, e molti framework C2 reali lo usano perché l'HTTP si confonde con il normale traffico web. Lo svantaggio è la latenza: se l'intervallo di polling è di 30 secondi, un task potrebbe attendere fino a 30 secondi prima che il beacon lo rilevi.

L'**HTTP long polling** migliora la latenza facendo sì che il server tenga aperta la richiesta finché non è disponibile un task. È migliore, ma gestire connessioni tenute aperte aggiunge complessità e ogni richiesta in attesa consuma un thread del server o uno slot di connessione.

I **WebSockets** ci offrono una connessione persistente e full-duplex. Il server può inviare un task al beacon nell'istante esatto in cui arriva, con latenza di polling pari a zero. Il beacon può inviare heartbeat e risultati in qualsiasi momento senza aprire nuove connessioni.

Abbiamo scelto i WebSocket perché semplificano l'architettura. Una connessione, bidirezionale, sempre aperta. Il compromesso riguarda la sicurezza operativa (OPSEC): una connessione WebSocket persistente verso un endpoint fisso (`/api/ws/beacon`) è più facile da rilevare per i sistemi di monitoraggio di rete rispetto a richieste HTTP periodiche che sembrano normale navigazione web. Un C2 di produzione offrirebbe entrambe le opzioni, lasciando la scelta all'operatore in base all'ambiente bersaglio.

### Perché SQLite rispetto a PostgreSQL

Illustrato nella sezione dello schema del database. In sintesi: singolo server, singolo scrittore, il WAL gestisce le letture concorrenti, zero configurazione, adeguato all'ambito educativo.

### Perché Zustand rispetto a React Context

La gestione dello stato del frontend usa Zustand invece di React Context per motivi ben precisi:

**Nessun wrapper Provider.** React Context richiede di racchiudere l'albero dei componenti in un `<Provider>`. Gli store Zustand sono singleton a livello di modulo. Importi `useC2Store` e lo chiami. Questo mantiene pulito l'albero dei componenti ed evita problemi di annidamento dei provider quando si hanno più store.

**Sottoscrizioni basate su selettori.** Con React Context, qualsiasi aggiornamento del valore del contesto riesegue il render di ogni componente che lo consuma. Zustand consente ai componenti di sottoscriversi a porzioni specifiche dello stato. La dashboard si sottoscrive solo all'oggetto beacons. La pagina di sessione si sottoscrive solo ai risultati dei task. Quando un heartbeat aggiorna il `last_seen` di un beacon, la pagina di sessione non esegue il re-render.

**`useShallow` per gli array derivati.** Il selettore `useBeacons` in `frontend/src/core/ws.ts` usa `useShallow` per racchiudere `Object.values(s.beacons)`. Senza `useShallow`, `Object.values()` creerebbe un nuovo riferimento di array a ogni aggiornamento dello store, portando il componente a re-renderizzarsi in un loop infinito anche se i dati effettivi dei beacon non sono cambiati. `useShallow` esegue un confronto shallow degli elementi dell'array e restituisce un nuovo riferimento solo se il contenuto differisce realmente.

**Supporto ai DevTools.** Lo store è racchiuso nel middleware `devtools` di Zustand con `name: 'C2Store'`. Ciò consente di aprire l'estensione del browser Redux DevTools e ispezionare ogni cambio di stato, cosa inestimabile durante il debug dei messaggi WebSocket.

### Perché code di task per singolo beacon

Il `TaskManager` in `backend/app/beacon/tasking.py` mantiene una `asyncio.Queue` separata per ciascun ID beacon. Quando un operatore invia un task per il beacon A, questo entra nella coda del beacon A. La coda del beacon B non ne risente.

Questo design offre molteplici vantaggi:

**Consegna indipendente.** Se il beacon A è lento o disconnesso, i task per il beacon B non vengono bloccati. La coroutine `_send_tasks` di ciascun beacon in `beacon/router.py` rimane in attesa sulla propria coda in modo indipendente.

**Gestione pulita della disconnessione.** Quando un beacon si disconnette, `task_manager.remove_queue(beacon_id)` elimina la coda ed eventuali task pendenti al suo interno. Questo previene perdite di memoria dovute all'accumulo di task per beacon non più attivi.

**Ordinamento semplice.** asyncio.Queue è una struttura FIFO, per cui i task per un dato beacon vengono consegnati nell'ordine esatto in cui sono stati inviati. Non serve alcuna logica di priorità o ordinamento.

L'alternativa sarebbe stata una singola coda condivisa con filtri, o un approccio basato su polling del database in cui la coroutine `_send_tasks` interroga continuamente SQLite. L'approccio con coda per beacon è più semplice, più veloce (nessuna query al database nel percorso critico) e si mappa naturalmente sull'architettura con una coroutine per beacon.

### Perché il pattern a doppia coroutine (send + receive)

L'handler WebSocket del beacon in `backend/app/beacon/router.py` esegue due coroutine in modo concorrente per ciascuna connessione beacon:

- `_send_tasks` rimane in attesa su `task_manager.get_next()` e invia i messaggi TASK al beacon.
- `_receive_messages` rimane in attesa su `ws.receive_text()` ed elabora i messaggi HEARTBEAT e RESULT provenienti dal beacon.

Queste vengono eseguite tramite `asyncio.wait` con `return_when=FIRST_COMPLETED`:

```python
done, pending = await asyncio.wait(
    [send_task, recv_task],
    return_when=asyncio.FIRST_COMPLETED,
)

for task in pending:
    task.cancel()
```

Questo pattern rappresenta il modo standard di gestire la comunicazione WebSocket bidirezionale in asyncio. Ecco perché entrambe le coroutine sono indispensabili:

**`_send_tasks` deve attendere in modo indipendente.** Deve fare `await` sulla coda dei task, operazione che potrebbe richiedere minuti o ore se nessun operatore invia task. Non è possibile intercalare questo processo con la ricezione dei messaggi nella stessa coroutine senza complicata logica di select/poll.

**`_receive_messages` deve attendere in modo indipendente.** Deve fare `await` sul successivo frame WebSocket, che potrebbe essere un heartbeat ogni pochi secondi o un risultato dopo un comando a lunga esecuzione.

**FIRST_COMPLETED gestisce la disconnessione.** Se il beacon si disconnette, `ws.receive_text()` solleva `WebSocketDisconnect`, terminando `_receive_messages`. L'`asyncio.wait` ritorna immediatamente, la coroutine pendente `_send_tasks` viene annullata, e il blocco `finally` nell'handler principale pulisce il registry e la coda dei task. Funziona anche all'inverso: se `_send_tasks` fallisce (ad esempio perché `ws.send_text()` solleva un errore su una connessione interrotta), completa per prima e `_receive_messages` viene annullata.

---

## Architettura di sicurezza

### Modello delle minacce (Threat Model)

Essendo questo un progetto educativo, il modello delle minacce è intenzionalmente limitato. Ecco da cosa ci proteggiamo, cosa simuliamo e cosa lasciamo deliberatamente scoperto:

**Cosa simuliamo di proteggere:**

- **Intercettazione di rete (Eavesdropping).** La codifica XOR+Base64 sul canale beacon-server simula la cifratura del trasporto. Se qualcuno cattura il traffico WebSocket, vedrà stringhe Base64 al posto di JSON in chiaro. Naturalmente, uno XOR con chiave ripetuta di lunghezza nota è banalmente violabile. Il punto è dimostrare dove si colloca la codifica nello stack di protocollo, non offrire riservatezza reale.
- **Integrità dei messaggi.** La funzione `unpack()` convalida i messaggi in arrivo rispetto allo schema Pydantic. Un messaggio malformato o con dati non validi viene rifiutato con un ValueError. Questo impedisce confusione di protocollo basilare, anche se non protegge da un attaccante sofisticato che conosca il formato del protocollo.
- **Identità del beacon.** La chiave XOR condivisa agisce come una forma debole di autenticazione. Se non possiedi la chiave, il tuo messaggio REGISTER non verrà decodificato correttamente e il server lo rifiuterà. Questo simula il concetto di autenticazione tramite chiave pre-condivisa (PSK), anche se l'implementazione effettiva è elementare.

**Da cosa deliberatamente NON proteggiamo:**

- **Reverse engineering.** Il beacon è un semplice script Python in chiaro. Chiunque abbia accesso al file può leggerne il sorgente, estrarre la chiave XOR e comprendere interamente il protocollo.
- **Rilevamento da parte di EDR/AV.** Il beacon non fa alcun tentativo di eludere il rilevamento degli endpoint. Usa librerie standard di Python, nomi di processo standard e chiamate di rete standard.
- **Analisi del traffico.** L'endpoint WebSocket è costantemente `/api/ws/beacon`. L'intervallo dell'heartbeat è prevedibile (anche con il jitter). Un analista di rete potrebbe facilmente identificare l'impronta (fingerprint) di questo traffico.
- **Estrazione della chiave dalla memoria.** La chiave XOR risiede in chiaro nella memoria del processo del beacon. Un dump della memoria la rivela immediatamente.
- **Autenticazione degli operatori.** Chiunque sia in grado di raggiungere l'endpoint `/api/ws/operator` può connettersi come operatore e inviare comandi. Non c'è login, token o controllo degli accessi. Si tratta di una semplificazione voluta.

### Cosa aggiungerebbe un C2 di produzione

Se stessi costruendo un framework C2 reale (per operazioni di red team autorizzate), aggiungeresti livelli che qui abbiamo tralasciato:

**TLS per la cifratura del trasporto.** Tutto il traffico viaggerebbe su HTTPS/WSS. Ciò garantisce riservatezza e integrità reali a livello di trasporto, e mimetizza il traffico C2 nel normale traffico HTTPS.

**Certificate pinning.** Il beacon convaliderebbe il certificato TLS del server rispetto a un hash fissato (pinned), prevenendo attacchi man-in-the-middle anche in presenza di proxy aziendali di intercettazione TLS.

**Scambio asimmetrico di chiavi.** Anziché una chiave XOR pre-condivisa, beacon e server negozierebbero una chiave di sessione usando RSA o ECDH. Questo garantisce la forward secrecy: la compromissione di una chiave di sessione non compromette le comunicazioni passate o future.

**Autenticazione degli operatori.** La dashboard dell'operatore richiederebbe autenticazione (username/password, certificati client o TOTP). Il controllo degli accessi basato sui ruoli (RBAC) limiterebbe quali operatori possono accedere a quali beacon.

**Staging del beacon.** Invece di distribuire l'intero beacon come script Python, un C2 di produzione impiega un piccolo "stager" che scarica il payload completo dal server, lo decifra in memoria e lo esegue senza toccare il disco. Questo riduce la superficie di rilevamento basata su file.

**Profili C2 malleabili.** Strumenti come Cobalt Strike consentono agli operatori di definire "profili" che rimodellano il traffico C2 per farlo assomigliare a servizi web legittimi (chiamate API di Slack, beacon di Google Analytics, ecc.). Questo elude il rilevamento di rete basato su firme.

**Domain fronting o mimetizzazione tramite CDN.** Il beacon si connette a un dominio CDN legittimo (come cloudfront.net) e il CDN instrada la richiesta al server C2 effettivo in base all'header Host. Dal punto di vista della rete, il beacon sta semplicemente comunicando con un CDN.

---

## Architettura di deployment

### Stack di sviluppo

Lo stack di sviluppo è definito in `dev.compose.yml` ed è ottimizzato per le modifiche in tempo reale con hot reload su ogni componente:

```
dev.compose.yml
├── nginx        (nginx:1.27-alpine, porta 47430)
│   └── volumi: configurazione dev.nginx (sola lettura)
├── frontend     (Vite dev server, porta 5173)
│   └── volumi: ./frontend montato, node_modules in volume con nome
└── backend      (uvicorn --reload, porta 8000)
    └── volumi: ./backend montato, .venv in volume con nome
```

**Nginx** usa l'immagine standard `nginx:1.27-alpine` con la configurazione di sviluppo montata. Fa da proxy sia verso il dev server di Vite sia verso il backend FastAPI.

**Frontend** viene compilato da `infra/docker/vite.dev` ed esegue `pnpm dev` con il server HMR (Hot Module Replacement) di Vite sulla porta 5173. La directory dei sorgenti `./frontend` è montata tramite bind mount all'interno del container, così qualsiasi modifica ai file sul tuo host attiva un aggiornamento istantaneo del browser. La directory `node_modules` risiede in un volume Docker con nome (`frontend_modules`) per evitare conflitti tra i binari specifici della piattaforma del tuo sistema operativo host e il Linux del container.

**Backend** viene compilato da `infra/docker/fastapi.dev` ed esegue uvicorn con `--reload` abilitato. La directory dei sorgenti `./backend` è montata tramite bind mount, quindi le modifiche ai file Python innescano un riavvio automatico del server. La directory `.venv` risiede in un volume con nome (`backend_cache`) per lo stesso motivo di isolamento della piattaforma. Le variabili d'ambiente impostano `ENVIRONMENT=development`, `DEBUG=true` e `RELOAD=true`.

Il compose di sviluppo usa la direttiva `depends_on` di Docker con controlli di integrità (health check) per assicurare che il backend sia pronto prima che nginx inizi a inoltrargli traffico. L'health check del backend interroga `http://localhost:8000/health` ogni 10 secondi con un periodo di tolleranza all'avvio di 30 secondi.

### Stack di produzione

Lo stack di produzione è definito in `compose.yml` e differisce dallo sviluppo sotto diversi aspetti:

```
compose.yml
├── nginx        (build multi-stage, porta 47430)
│   └── File statici del frontend compilati incorporati nell'immagine
└── backend      (gunicorn, porta 8000)
    └── Dati SQLite in volume con nome (c2_data)
```

**Nginx** usa una build multi-stage definita in `infra/docker/frontend-builder.prod`. Il primo stage installa pnpm, esegue `pnpm build` e produce i file statici del frontend. Il secondo stage copia tali file in un'immagine nginx con la configurazione di produzione (`prod.nginx`). Non esiste un container frontend separato in produzione. Nginx serve direttamente i file compilati, risultando più veloce e consumando meno risorse.

**Backend** viene compilato da `infra/docker/fastapi.prod` e gira con `ENVIRONMENT=production`, `DEBUG=false`, `RELOAD=false`. Il database SQLite risiede in un volume Docker con nome (`c2_data`) così da persistere tra i riavvii dei container. Vengono impostati limiti alle risorse: il backend riceve fino a 2 CPU e 1GB di RAM, nginx riceve fino a 1 CPU e 256MB di RAM.

La configurazione nginx di produzione (`prod.nginx`) aggiunge header di sicurezza (X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Referrer-Policy, Permissions-Policy), caching aggressivo per le risorse statiche (1 anno per `/assets/` con `immutable`) e limiti di frequenza più stringenti (burst=20 rispetto a burst=50 in dev).

---

## Riferimento ai file chiave

Tabella di consultazione rapida che mappa i concetti sui file di implementazione:

| Concetto | File |
|----------|------|
| Codifica XOR | `backend/app/core/encoding.py` |
| Messaggi di protocollo | `backend/app/core/protocol.py` |
| Modelli di dati | `backend/app/core/models.py` |
| Handler WebSocket del beacon | `backend/app/beacon/router.py` |
| Gestione code dei task | `backend/app/beacon/tasking.py` |
| Registry dei beacon | `backend/app/beacon/registry.py` |
| REST + WebSocket operatore | `backend/app/ops/router.py` |
| Broadcast manager operatore | `backend/app/ops/manager.py` |
| App factory + lifespan | `backend/app/__main__.py` |
| Configurazione (Pydantic Settings) | `backend/app/config.py` |
| Schema database + connessione | `backend/app/database.py` |
| Impianto beacon | `beacon/beacon.py` |
| Store Zustand frontend + WebSocket | `frontend/src/core/ws.ts` |
| Schemi e tipi Zod frontend | `frontend/src/core/types.ts` |
| Configurazione rotte ed endpoint frontend | `frontend/src/config.ts` |
| Configurazione nginx dev | `infra/nginx/dev.nginx` |
| Configurazione nginx prod | `infra/nginx/prod.nginx` |
| Configurazione nginx base condivisa | `infra/nginx/nginx.conf` |
| Docker Compose di sviluppo | `dev.compose.yml` |
| Docker Compose di produzione | `compose.yml` |

---

**Prossimo passo:** [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) esaminerà l'implementazione guidata del codice, mostrando la gestione dei socket, l'esecuzione dei comandi e i meccanismi di persistenza.