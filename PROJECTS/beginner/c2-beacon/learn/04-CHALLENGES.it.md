<!-- © AngelaMos | 2026 | 04-CHALLENGES.md -->

# Sfide di estensione

Queste sfide estendono il progetto C2 beacon con funzionalità reali che si trovano nei framework di command-and-control di produzione. Sono ordinate per difficoltà e ciascuna fa riferimento a file effettivi di questo progetto. Leggi il codice citato prima di iniziare. I suggerimenti (hints) ti indirizzano nella giusta direzione senza svelarti l'intera soluzione.

Ciascuna sfida cita funzioni reali, nomi di classe reali e percorsi di file reali. Se un suggerimento dice "guarda `COMMAND_HANDLERS` in `beacon.py`", vai a leggere quel dizionario. Il codice è la fonte di verità.

---

## Sfide facili

---

### Sfida 1: Aggiungere un comando `pwd`

**Cosa realizzare:** Un nuovo comando che restituisca all'operatore la directory di lavoro corrente (current working directory) del beacon.

**Perché è importante:** Ogni framework C2 reale supporta la consapevolezza della directory. Cobalt Strike, Sliver e Metasploit dispongono tutti dei comandi `pwd` e `cd` perché gli operatori devono sapere dove atterreranno le operazioni sui file. Senza `pwd`, se esegui `download notes.txt`, non hai idea di quale `notes.txt` tu stia prelevando.

**Cosa imparerai:**
- Come i nuovi comandi si propagano attraverso l'intero stack (handler del beacon, enum del backend, tipi del frontend, UI)
- L'intero ciclo di vita di un comando: invio dal frontend, inoltro via WebSocket, esecuzione sul beacon, restituzione del risultato

**Da dove iniziare a leggere:**

Il sistema di dispatch dei comandi si trova in `beacon/beacon.py`. Esamina il dizionario `COMMAND_HANDLERS` alla riga 419. Ogni comando mappa una chiave stringa a una funzione handler asincrona. La funzione `dispatch` alla riga 433 si occupa dell'instradamento: cerca la stringa del comando nel dizionario e chiama l'handler corrispondente.

Sul backend, `backend/app/core/models.py:12-26` definisce `CommandType` come uno `StrEnum`. Ogni comando supportato deve essere elencato qui, altrimenti la validazione Pydantic in `backend/app/ops/router.py:60` lo rifiuterà.

Sul frontend, `frontend/src/core/types.ts:8-19` rispecchia l'enum come schema Zod. La pagina di sessione in `frontend/src/pages/session/index.tsx:19-30` possiede un array `COMMANDS` che controlla quali comandi vengono accettati dal parser di input.

**Suggerimenti:**

1. L'handler del beacon è banale. `os.getcwd()` di Python restituisce la directory di lavoro corrente come stringa. Il tuo handler dovrebbe seguire lo stesso pattern di ritorno degli altri handler: `{"output": ..., "error": None}`.

2. Devi modificare quattro file in questo ordine:
   - Aggiungere `PWD = "pwd"` all'enum `CommandType` in `backend/app/core/models.py`
   - Aggiungere `'pwd'` all'array dell'enum Zod in `frontend/src/core/types.ts`
   - Aggiungere `'pwd'` all'array `COMMANDS` in `frontend/src/pages/session/index.tsx`
   - Aggiungere la funzione handler e la relativa voce in `COMMAND_HANDLERS` in `beacon/beacon.py`

3. Guarda `handle_sysinfo` in `beacon/beacon.py:153` per il pattern. Il tuo handler sarà molto più semplice, circa 3 righe.

**File da modificare:**
- `beacon/beacon.py`
- `backend/app/core/models.py`
- `frontend/src/core/types.ts`
- `frontend/src/pages/session/index.tsx`

**Come testare:**
- Avvia l'intero stack con Docker
- Connetti un beacon
- Digita `pwd` nel terminale della sessione
- Verifica di ricevere un percorso di directory come `/home/yoshi/dev/...`

---

### Sfida 2: Visualizzare l'uptime del beacon

**Cosa realizzare:** Mostrare da quanto tempo ciascun beacon è connesso come nuova colonna nella tabella della dashboard. Visualizzalo come una durata leggibile dall'uomo come "2h 15m" o "3d 6h".

**Perché è importante:** Gli operatori in ingaggi C2 reali devono valutare rapidamente quali beacon siano recenti rispetto a quelli di lunga durata. Un beacon attivo da 5 giorni è più prezioso di uno che si è connesso 30 secondi fa, perché è sopravvissuto a riavvii, scansioni AV e cambi di rete. L'uptime a colpo d'occhio cambia il modo in cui gli operatori danno priorità ai bersagli.

**Cosa imparerai:**
- Leggere dai modelli di dati esistenti senza modificare il backend
- Aritmetica delle date in JavaScript
- Estendere componenti tabella in React

**Da dove iniziare a leggere:**

Il modello `BeaconRecord` contiene già tutto ciò che serve. Guarda `frontend/src/core/types.ts:22-34`. Il campo `first_seen` è una stringa timestamp ISO 8601 impostata quando il beacon si è registrato per la prima volta. Il campo `last_seen` si aggiorna a ogni heartbeat.

La tabella della dashboard si trova in `frontend/src/pages/dashboard/index.tsx`. Il componente `BeaconRow` alla riga 43 esegue il rendering di una riga per beacon. Gli header della tabella si trovano alle righe 120-128. C'è già una funzione `formatRelativeTime` alla riga 14 che converte i timestamp ISO in stringhe relative come "5s ago" o "2h ago". Puoi usare un approccio simile, oppure calcolare `Date.now() - new Date(beacon.first_seen).getTime()` per ottenere l'uptime in millisecondi.

La dashboard ha già un intervallo di tick di 1 secondo alle righe 95-98 che forza i re-render, quindi la tua visualizzazione dell'uptime si aggiornerà automaticamente ogni secondo.

**Suggerimenti:**

1. L'uptime è la differenza tra l'istante attuale e `first_seen`. La sottrazione fornisce millisecondi. Convertili in ore e minuti per la visualizzazione.

2. Aggiungi un nuovo `<th>Uptime</th>` nell'header della tabella e un corrispondente `<td>` in `BeaconRow`. Posizionalo prima della colonna "Last Seen".

3. Scrivi una funzione `formatUptime` che accetti una stringa ISO e restituisca qualcosa come "4h 22m" o "1d 3h". Non complicarla: dividi i millisecondi per le costanti appropriate (1000 per i secondi, 60 per i minuti, ecc.).

**File da modificare:**
- `frontend/src/pages/dashboard/index.tsx`

**Come testare:**
- Avvia lo stack, connetti un beacon
- Verifica che la colonna uptime mostri inizialmente "0s" o "1s"
- Attendi qualche minuto e verifica che si aggiorni
- Ricarica la pagina e verifica che mostri ancora l'uptime corretto (poiché `first_seen` è persistito in SQLite)

---

### Sfida 3: Aggiungere un comando terminale `clear`

**Cosa realizzare:** Un comando `clear` lato client che svuota l'output del terminale nella pagina della sessione, come l'esecuzione di `clear` in un vero terminale.

**Perché è importante:** Quando esegui dozzine di comandi durante un ingaggio, il terminale si riempie in fretta. Gli operatori hanno bisogno di ripulire la visuale senza perdere la capacità di eseguire nuovi comandi. Questo è anche un buon esercizio per comprendere quali comandi siano puramente lato client rispetto a quelli che viaggiano verso il beacon.

**Cosa imparerai:**
- Intercettazione dei comandi lato client (gestire i comandi prima che raggiungano la rete)
- La differenza tra l'elaborazione dei comandi lato client e lato server
- Manipolazione dello stato in React

**Da dove iniziare a leggere:**

La funzione `handleSubmit` in `frontend/src/pages/session/index.tsx:174` è il punto in cui inizia ogni comando. Chiama `parseInput` (riga 39) per validare il comando, quindi chiama `handleSend` (riga 164) che aggiunge una voce al terminale e invia il task via WebSocket tramite `sendTask`.

L'output del terminale è memorizzato nell'array di stato `entries` alla riga 145. La chiamata `setEntries([])` cancella tutte le voci del terminale.

La funzione `parseInput` alla riga 39 controlla se il comando digitato è presente nell'array `COMMANDS`. Se il comando non viene trovato, restituisce `null` e `handleSubmit` non fa nulla.

**Suggerimenti:**

1. `clear` NON deve essere aggiunto all'array `COMMANDS`, all'enum `CommandType` o ai modelli del backend. Non lascia mai il frontend.

2. Intercetta `clear` all'interno di `handleSubmit` PRIMA della chiamata a `parseInput`. Verifica se `input.trim() === 'clear'`, e in tal caso chiama `setEntries([])`, svuota l'input ed esegui un return anticipato.

3. Potresti voler aggiungere `clear` anche come comando riconosciuto nei suggerimenti di completamento automatico. La funzione `handleInputChange` alla riga 223 filtra l'array `COMMANDS` per i suggerimenti. Puoi creare un array separato `CLIENT_COMMANDS` o inserire il controllo direttamente nel codice.

**File da modificare:**
- `frontend/src/pages/session/index.tsx`

**Come testare:**
- Apri una sessione, esegui qualche comando
- Digita `clear` e premi Invio
- Verifica che il terminale si svuoti
- Esegui nuovi comandi e verifica che appaiano normalmente
- Verifica che non sia stata inviata alcuna richiesta di rete (controlla i frame WebSocket nei DevTools)

---

### Sfida 4: Aggiungere il conteggio delle connessioni nell'Header

**Cosa realizzare:** Visualizzare il numero di operatori connessi nell'header della dashboard. Qualcosa come "2 operators connected" accanto al conteggio dei beacon.

**Perché è importante:** In un ingaggio di gruppo, più operatori possono essere connessi allo stesso server C2. Sapere quante persone sono attive aiuta a coordinarsi. Se vedi "0 operators" mentre ti aspettavi che il tuo compagno di squadra fosse online, qualcosa non va: o la sua connessione è caduta o qualcuno sta intercettando il traffico.

**Cosa imparerai:**
- Aggiungere endpoint REST al backend FastAPI
- Recuperare dati dal backend nei componenti React
- La relazione tra l'OpsManager e il WebSocket degli operatori

**Da dove iniziare a leggere:**

La classe `OpsManager` in `backend/app/ops/manager.py:15` tiene già traccia delle connessioni. La proprietà `connection_count` alla riga 56 restituisce `len(self._connections)`. Il router REST degli operatori in `backend/app/ops/router.py` ha gli endpoint a partire dalla riga 94.

Il componente della dashboard in `frontend/src/pages/dashboard/index.tsx:100` esegue il rendering di una sezione header alle righe 103-112 che mostra già il conteggio dei beacon.

**Suggerimenti:**

1. L'approccio più semplice: aggiungi un nuovo endpoint REST a `backend/app/ops/router.py`. Qualcosa come `GET /api/operators/count` che restituisca `{"count": ops_manager.connection_count}`. Accedi all'ops_manager da `request.app.state.ops_manager`.

2. Sul frontend, effettua il fetch da questo endpoint con uno `useEffect` al montaggio. Salva il risultato nello stato locale. Mostralo accanto al conteggio dei beacon esistente nell'header.

3. Un approccio alternativo che evita il polling: includi il conteggio degli operatori nel payload iniziale del WebSocket `beacon_list` in `backend/app/ops/router.py:46-49`. Aggiungi un campo `operator_count`. In questo modo arriva subito alla connessione senza una richiesta HTTP extra. Tuttavia non si aggiornerà in tempo reale a meno di non trasmettere in broadcast anche i cambi di conteggio.

4. Considera quale approccio offra una migliore precisione in tempo reale. L'endpoint REST si aggiorna solo quando l'utente ricarica o quando effettui il polling. L'approccio WebSocket si aggiorna ogni volta che lo trasmetti in broadcast. Un approccio ibrido funziona bene: invia il conteggio alla connessione iniziale e trasmetti aggiornamenti in broadcast ogni volta che un operatore si connette o si disconnette.

**File da modificare:**
- `backend/app/ops/router.py`
- `frontend/src/pages/dashboard/index.tsx`
- Facoltativamente: `frontend/src/core/ws.ts` e `frontend/src/core/types.ts` (se usi l'approccio WebSocket)

---

## Sfide intermedie

---

### Sfida 5: Aggiungere l'autenticazione dell'operatore

**Cosa realizzare:** Richiedere una password prima di consentire connessioni WebSocket dagli operatori. Se la password è errata, chiudi immediatamente la connessione con il codice di stato 4001.

**Perché è importante:** Al momento, chiunque possa raggiungere l'endpoint WebSocket del server C2 può connettersi come operatore e controllare ogni beacon. In un ingaggio reale, ciò significa che un difensore che scopre il server C2 può inviare `shell rm -rf /` a tutti i beacon o semplicemente visualizzare tutti i risultati dei task. Ogni framework C2 di produzione ha l'autenticazione per gli operatori. Cobalt Strike usa password di team. Sliver impiega mutual TLS con certificati operatore.

**Cosa imparerai:**
- Pattern di autenticazione per WebSocket
- Configurazione basata su variabili d'ambiente
- Proteggere un protocollo progettato inizialmente senza autenticazione

**Da dove iniziare a leggere:**

L'handler WebSocket dell'operatore si trova in `backend/app/ops/router.py:25-91`. Il flusso di connessione è: `ops_manager.connect(ws)` accetta il WebSocket alla riga 34, poi invia subito l'elenco dei beacon alle righe 36-49, quindi entra in un ciclo di ricezione dei messaggi alla riga 51.

La classe `Settings` in `backend/app/config.py:17` carica la configurazione dalle variabili d'ambiente usando Pydantic Settings. Il campo `XOR_KEY` alla riga 41 mostra il pattern per aggiungere nuovi valori di configurazione con valori predefiniti.

La connessione WebSocket del frontend viene stabilita in `frontend/src/core/ws.ts:166-167`. La funzione `connect` crea un nuovo `WebSocket` e imposta gli handler di evento. Attualmente, nessun dato di autenticazione viene inviato alla connessione.

**Suggerimenti:**

1. Aggiungi un campo `AUTH_KEY` alla classe `Settings` in `backend/app/config.py`. Assegna un valore predefinito per lo sviluppo, come `"operator-default-key"`. Caricalo dalla variabile d'ambiente `AUTH_KEY`.

2. Nell'handler WebSocket dell'operatore in `router.py:25`, PRIMA di chiamare `ops_manager.connect(ws)`, accetta manualmente la connessione con `await ws.accept()`, quindi attendi il primo messaggio. Se corrisponde a `settings.AUTH_KEY`, procedi normalmente. In caso contrario, chiama `await ws.close(code=4001, reason="Unauthorized")` e termina.

3. Sul frontend, modifica la funzione `connect` in `ws.ts`. Nel callback `ws.onopen` (riga 170), invia la chiave di autenticazione come primo messaggio: `ws.send(JSON.stringify({ type: "auth", key: "..." }))`. Avrai bisogno della chiave disponibile nel frontend, o inserita direttamente nel codice per sviluppo, o caricata da un form di login, o passata tramite variabile d'ambiente usando `import.meta.env` di Vite.

4. Dovrai anche adattare `ops_manager.connect()` in `manager.py:25`. Al momento chiama `await ws.accept()`. Se accetti manualmente nel router prima dell'autenticazione, il manager dovrebbe saltare la chiamata accept. In alternativa, ristruttura il codice in modo che il manager si occupi dell'accettazione e della verifica dell'autenticazione.

5. Fai attenzione al ciclo dei messaggi in `router.py:51`. Dopo l'autenticazione, il primo messaggio "reale" dovrebbe essere un `submit_task`, non il messaggio di auth. Assicurati di non interpretare per errore il messaggio di autenticazione come invio di un task.

**File da modificare:**
- `backend/app/config.py`
- `backend/app/ops/router.py`
- `frontend/src/core/ws.ts`

**Come testare:**
- Imposta `AUTH_KEY=my-secret-key` nel tuo file `.env`
- Aggiorna il frontend affinché invii questa chiave alla connessione
- Avvia lo stack e verifica che la dashboard si carichi regolarmente
- Modifica la chiave del frontend con un valore errato e verifica che il WebSocket si chiuda con 4001
- Controlla i log del server per riscontrare il tentativo non autorizzato

---

### Sfida 6: Aggiungere altri pulsanti Quick Action

**Cosa realizzare:** Espandere il pannello delle azioni rapide nella pagina di sessione con pulsanti aggiuntivi. Considera `whoami`, `uname -a`, `id`, `hostname` e comandi di rete.

**Perché è importante:** Le quick action accelerano i comuni compiti di ricognizione durante un ingaggio. Gli operatori C2 reali eseguono la stessa manciata di comandi su ogni nuovo beacon per orientarsi: chi sono, che sistema è questo, su quale rete mi trovo. Avere pulsanti con un clic per questi comandi è molto più rapido che digitarli ogni volta.

**Cosa imparerai:**
- Come le quick action mappano sui tipi di comando esistenti
- Costruire comandi shell inviati attraverso il tipo di comando `shell`
- Estendere le interfacce dei componenti React

**Da dove iniziare a leggere:**

Il componente `QuickActions` si trova in `frontend/src/pages/session/index.tsx:55-88`. Esegue il rendering di tre pulsanti per `sysinfo`, `proclist` e `screenshot`. Ciascun pulsante chiama `onSend` con un `CommandType`. La prop `onSend` proviene da `handleSend` alla riga 164, che accetta un `CommandType` e `args` facoltativi.

Nota che `sysinfo`, `proclist` e `screenshot` sono tipi di comando a sé stanti, quindi i pulsanti chiamano `onSend('sysinfo')` senza argomenti. Ma per i comandi shell, devi chiamare `handleSend` indicando sia il tipo di comando sia gli argomenti, come `handleSend('shell', 'whoami')`.

**Suggerimenti:**

1. Il componente `QuickActions` accetta attualmente `onSend: (cmd: CommandType) => void`. Devi modificarlo per accettare anche gli argomenti: `onSend: (cmd: CommandType, args?: string) => void`. Questo si adatta alla signature di `handleSend` alla riga 164, che accetta già argomenti opzionali.

2. Aggiungi nuovi pulsanti che chiamino `onSend('shell', 'whoami')`, `onSend('shell', 'uname -a')`, `onSend('shell', 'id')`, ecc.

3. Considera di raggruppare i pulsanti. Potresti creare sezioni: "System Info" per sysinfo/proclist, "Quick Recon" per i comandi shell, "Collection" per screenshot. Usa un'intestazione o un separatore visivo.

4. Potresti anche aggiungere pulsanti per `keylog_start`, `keylog_stop` e `sleep`. Il pulsante sleep potrebbe richiedere un intervallo, ma inizia semplicemente con un valore fisso.

**File da modificare:**
- `frontend/src/pages/session/index.tsx`

**Come testare:**
- Apri una sessione con un beacon attivo
- Fai clic su ciascun nuovo pulsante di azione rapida
- Verifica che il comando e gli argomenti corretti compaiano nel terminale
- Verifica che il beacon restituisca i risultati per ciascuno

---

### Sfida 7: Persistenza della cronologia dei task

**Cosa realizzare:** Quando ti allontani dalla pagina di una sessione e vi ritorni, ricarica la cronologia dei task precedenti dal server affinché il terminale mostri i comandi passati e i relativi risultati.

**Perché è importante:** Al momento, le voci del terminale risiedono nello stato del componente React. Allontanati dalla pagina e svaniranno. In un ingaggio reale che dura ore o giorni, gli operatori passano costantemente da un beacon all'altro. Perdere la cronologia del terminale significa perdere il contesto su ciò che è già stato eseguito, portando a comandi ridondanti o a risultati persi di vista.

**Cosa imparerai:**
- Recuperare dati storici da API REST al montaggio del componente
- Mappare i record del database sullo stato della UI
- La relazione tra i percorsi di dati WebSocket (tempo reale) e REST (storici)

**Da dove iniziare a leggere:**

L'endpoint REST esiste già. Guarda `backend/app/ops/router.py:128-137`. L'endpoint `GET /beacons/{beacon_id}/tasks` chiama `task_manager.get_history(beacon_id, db)` che è implementato in `backend/app/beacon/tasking.py:93-116`. Questa funzione esegue un join tra `tasks` e `task_results` e restituisce righe con le colonne: `id`, `command`, `args`, `status`, `created_at`, `completed_at`, `output`, `error`.

La pagina di sessione in `frontend/src/pages/session/index.tsx:138` contiene la funzione `Component`. Lo stato `entries` alla riga 145 è di tipo `TerminalEntry[]`, dove ogni voce ha `command`, `args`, `result` e `taskId`.

L'interfaccia `TerminalEntry` si trova alla riga 32. Il campo `result` è `TaskResult | null`, e `TaskResult` (da `types.ts:47-54`) ha `id`, `task_id`, `output`, `error` e `created_at`.

**Suggerimenti:**

1. Aggiungi uno `useEffect` nella funzione `Component` che effettui il fetch di `/api/beacons/${id}/tasks` al montaggio. Analizza la risposta trasformandola in oggetti `TerminalEntry[]` e chiama `setEntries(...)` con il risultato.

2. Devi mappare il formato di risposta REST su `TerminalEntry`. La risposta REST include `command`, `args`, `output`, `error` e `id` (l'ID del task). Costruisci ciascuna voce in questo modo:
   ```typescript
   {
     command: row.command,
     args: row.args,
     result: row.output || row.error ? { id: "...", task_id: row.id, output: row.output, error: row.error, created_at: row.created_at } : null,
     taskId: row.id,
   }
   ```

3. La cronologia viene restituita con `ORDER BY created_at DESC` (vedi `tasking.py:113`), quindi vorrai invertirla per la visualizzazione cronologica nel terminale.

4. Fai attenzione a non duplicare le voci. Se il risultato di un task arriva via WebSocket mentre stai caricando la cronologia da REST, potresti ritrovarti con la stessa voce due volte. Considera di deduplicare in base a `taskId` o di caricare la cronologia solo se `entries` è vuoto.

**File da modificare:**
- `frontend/src/pages/session/index.tsx`

**Come testare:**
- Apri una sessione, esegui diversi comandi, attendi i risultati
- Torna alla dashboard
- Fai clic sullo stesso beacon per rientrare nella sessione
- Verifica che i comandi e i risultati precedenti compaiano nel terminale
- Esegui un nuovo comando e verifica che venga aggiunto correttamente in coda alle voci storiche

---

### Sfida 8: Aggiungere la cifratura AES-256

**Cosa realizzare:** Sostituire la codifica XOR con la cifratura AES-256-GCM per il canale di comunicazione beacon-server.

**Perché è importante:** La codifica XOR è banalmente reversibile. Chiunque catturi il traffico e scopra la chiave (che è una stringa statica proveniente da una variabile d'ambiente) può decodificare ogni messaggio. Anche senza la chiave, la codifica XOR è vulnerabile ad attacchi basati su testo in chiaro noto. Se un attaccante sa che il beacon invia un messaggio `HEARTBEAT` ogni pochi secondi, può eseguire lo XOR del testo in chiaro noto con il testo cifrato per recuperare la chiave.

AES-256-GCM offre autenticazione e cifratura: riservatezza (gli attaccanti non possono leggere il messaggio), integrità (gli attaccanti non possono modificare il messaggio senza essere rilevati) e autenticazione (chi decifra può verificare che il messaggio provenga da chi detiene la chiave).

Riferimento MITRE ATT&CK: T1573.001 (Encrypted Channel: Symmetric Cryptography).

**Cosa imparerai:**
- Cifratura simmetrica AES-GCM con IV casuali
- Derivazione della chiave da password tramite PBKDF2 o HKDF
- La differenza tra codifica (XOR, Base64) e cifratura (AES-GCM)
- Aggiornare entrambi i lati di un protocollo di comunicazione simultaneamente

**Da dove iniziare a leggere:**

La codifica lato server risiede in `backend/app/core/encoding.py`. La funzione `encode` alla riga 16 esegue `UTF-8 -> XOR -> Base64`. La funzione `decode` alla riga 24 inverte il flusso. Queste vengono chiamate da `backend/app/core/protocol.py:37-59` nelle funzioni `pack` e `unpack`.

La codifica lato beacon risiede in `beacon/beacon.py:56-78`. Le funzioni `xor_bytes`, `encode` e `decode` rispecchiano esattamente il lato server.

La chiave XOR proviene da `backend/app/config.py:41-44` (impostazione `XOR_KEY`) e `beacon/beacon.py:40-41` (variabile d'ambiente `C2_XOR_KEY`). Entrambe adottano il valore predefinito `"c2-beacon-default-key-change-me"`.

**Suggerimenti:**

1. Usa la libreria `cryptography` di Python. Nello specifico, `cryptography.hazmat.primitives.ciphers.aead.AESGCM`. Questa fornisce un'API pulita: `aesgcm.encrypt(nonce, plaintext, associated_data)` e `aesgcm.decrypt(nonce, ciphertext, associated_data)`.

2. AES-256 richiede una chiave di 32 byte. La chiave XOR attuale è una stringa leggibile dall'uomo. Deriva da essa una corretta chiave a 256 bit usando HKDF o PBKDF2:
   ```python
   from cryptography.hazmat.primitives.kdf.hkdf import HKDF
   from cryptography.hazmat.primitives import hashes
   key = HKDF(
       algorithm=hashes.SHA256(),
       length=32,
       salt=None,
       info=b"c2-beacon-aes",
   ).derive(password.encode("utf-8"))
   ```

3. Ogni messaggio richiede un nonce (IV) univoco di 12 byte. Generalo con `os.urandom(12)`. Preponilo al testo cifrato in modo che il ricevente possa estrarlo. Il formato finale è: `base64(nonce + ciphertext + auth_tag)`.

4. Devi aggiornare sia `encoding.py` (server) sia `beacon.py` (beacon) di pari passo. Se un lato passa ad AES mentre l'altro usa ancora XOR, non potranno comunicare.

5. Le funzioni `pack` e `unpack` di `protocol.py` non necessitano di modifiche strutturali. Chiamano `encode` e `decode`. Se modifichi ciò che quelle funzioni fanno internamente, il livello di protocollo rimane invariato.

6. Scrivi un test. I test esistenti in `backend/tests/test_encoding.py` convalidano il ciclo di andata e ritorno di encode/decode con XOR. Aggiornali per AES-GCM.

**File da modificare:**
- `backend/app/core/encoding.py`
- `beacon/beacon.py`
- `backend/tests/test_encoding.py`

---

## Sfide avanzate

---

### Sfida 9: Supporto multi-operatore con ruoli

**Cosa realizzare:** Aggiungere il controllo degli accessi basato sui ruoli (RBAC) in cui gli operatori sono classificati come "admin" o "viewer". Gli amministratori possono inviare comandi ai beacon. I visualizzatori (viewers) possono osservare la dashboard e vedere i risultati in tempo reale, ma non possono inviare task.

**Perché è importante:** Durante un ingaggio di red team, l'operatore che controlla i beacon non è sempre l'unica persona che osserva. I team lead, chi redige i report e altri operatori potrebbero voler osservare l'attività senza rischiare di eseguire comandi accidentalmente. Cobalt Strike possiede esattamente questa funzionalità: agli operatori può essere concesso l'accesso in "sola lettura" a un team server.

**Cosa imparerai:**
- Autorizzazione basata sui ruoli nelle connessioni WebSocket
- Filtraggio dei messaggi lato server
- Rendering condizionale nel frontend in base ai permessi

**Da dove iniziare a leggere:**

L'`OpsManager` in `backend/app/ops/manager.py:15` memorizza le connessioni come un semplice `set[WebSocket]`. Non ha nozione di chi sia il proprietario di ciascuna connessione o di cosa sia autorizzato a fare.

L'handler WebSocket dell'operatore in `backend/app/ops/router.py:51-77` elabora tutti i messaggi in arrivo allo stesso modo. L'handler `submit_task` alla riga 55 non esegue alcun controllo dei permessi.

**Suggerimenti:**

1. Cambia `OpsManager._connections` da `set[WebSocket]` a `dict[WebSocket, str]` dove il valore è il ruolo ("admin" o "viewer").

2. Quando un operatore si connette, fagli inviare una dichiarazione del ruolo nel suo primo messaggio (o come parte di un messaggio di autenticazione se hai completato la Sfida 5). Salva il ruolo nel dizionario delle connessioni.

3. Nell'handler `operator_websocket` in `router.py:55`, prima di elaborare un messaggio `submit_task`, controlla il ruolo dell'operatore. Se è un viewer, rimanda indietro un messaggio di errore anziché inviare il task.

4. Sul frontend, memorizza il ruolo dell'operatore nello store Zustand. Usalo per disabilitare condizionalmente l'input dei comandi e i pulsanti delle quick action per i viewer. Disabilita i pulsanti visivamente (ingrigiti, attributo `disabled`) affinché i viewer possano vedere cosa è disponibile senza poter interagire.

5. Il metodo `broadcast` in `manager.py:40` dovrebbe continuare a inviare eventi a tutti gli operatori indipendentemente dal ruolo. I viewer hanno bisogno di vedere le connessioni dei beacon, gli heartbeat e i risultati dei task.

**File da modificare:**
- `backend/app/ops/manager.py`
- `backend/app/ops/router.py`
- `frontend/src/core/ws.ts`
- `frontend/src/pages/session/index.tsx`

---

### Sfida 10: Staging del beacon

**Cosa realizzare:** Un deployment a due stadi in cui un piccolo script stager scarica ed esegue il payload completo del beacon dal server C2 a runtime. Lo stager è formato da poche righe di Python. Il codice completo del beacon risiede sul server e viene distribuito su richiesta.

**Perché è importante:** I framework C2 reali usano quasi sempre gli stager per motivi di sicurezza operativa (OPSEC). Uno stager minuscolo (10-20 righe) è molto più difficile da rilevare per un antivirus rispetto a un beacon completo (oltre 500 righe con import come `psutil`, `mss` e `pynput`). Lo stager scarica il payload in memoria, quindi il codice del beacon completo non tocca mai il disco, aggirando le scansioni basate sui file.

Riferimenti MITRE ATT&CK: T1059.006 (Command and Scripting Interpreter: Python), T1105 (Ingress Tool Transfer).

**Cosa imparerai:**
- Consegna di payload a stadi (Staged payload delivery)
- Esecuzione del codice in memoria con `exec()`
- Servire payload di codice dal server C2
- I compromessi di sicurezza tra stager e payload stageless

**Da dove iniziare a leggere:**

Il beacon attuale in `beacon/beacon.py` è uno script autonomo. Importa le proprie dipendenze in alto ed esegue `asyncio.run(main())` alla riga 513. Tutto è autosufficiente.

L'app backend è assemblata in `backend/app/__init__.py` (o `__main__.py`). Gli endpoint REST sono montati tramite i file di router. Guarda come viene usato `rest_router` in `backend/app/ops/router.py:22` per il pattern.

**Suggerimenti:**

1. Aggiungi un nuovo endpoint REST al backend: `GET /api/stage`. Questo endpoint legge il file `beacon/beacon.py` e ne restituisce il contenuto come testo in chiaro. Dovrai configurare il percorso del file del beacon (una variabile d'ambiente o un campo delle impostazioni).

2. Lo stager è un piccolo script Python che fa tre cose:
   - Scarica il payload dal server C2 tramite HTTP
   - Facoltativamente verifica un hash o una firma per prevenire manomissioni
   - Chiama `exec()` per eseguire il payload in memoria

3. Mantieni lo stager il più piccolo possibile. Dovrebbe usare solo moduli della libreria standard (`urllib.request`, `ssl`). Nessun `pip install` richiesto. Più piccolo è lo stager, più è difficile da individuare.

4. Considera di codificare o cifrare il payload durante il transito. Lo stager potrebbe decodificare con XOR il payload prima di eseguirlo. Ciò aggiunge un sottile livello di offuscamento, anche se un analista determinato riuscirà comunque a fare reverse engineering.

5. Pensa alla gestione degli errori. Se il server C2 non è raggiungibile, lo stager dovrebbe riprovare con backoff esponenziale, analogamente alla logica di riconnessione del beacon in `beacon.py:460-510`.

6. Considerazione di sicurezza: `exec()` esegue codice arbitrario. Lo stager dovrebbe verificare che il payload provenga dal vero server C2 e non da un man-in-the-middle. Usa HTTPS per il download o implementa un segreto condiviso che lo stager usa per convalidare il payload.

**File da modificare:**
- Nuovo endpoint REST in `backend/app/ops/router.py` (o un nuovo file di router)
- Nuovo script stager (un file Python piccolo e separato)
- Facoltativamente: `backend/app/config.py` per la configurazione dello staging

---

### Sfida 11: Costruire uno strumento di rilevamento

**Cosa realizzare:** Uno script di monitoraggio di rete che rilevi i pattern di traffico del nostro beacon. Questo script si posiziona dal lato del difensore e ricerca i segnali di comunicazione C2.

**Perché è importante:** Costruire lo strumento di rilevamento per il proprio framework C2 insegna contemporaneamente l'attacco e la difesa. Comprenderai perché certi pattern di traffico siano sospetti e come i difensori individuino realmente i beacon C2 sul campo. Questo è il tipo di pensiero che distingue chi fa sicurezza a livello professionale.

Questa sfida convalida anche il design del tuo C2. Se il tuo strumento di rilevamento intercetta il beacon facilmente, sai che l'OPSEC del beacon è debole.

**Cosa imparerai:**
- Analisi del traffico di rete con Python
- Rilevamento comportamentale (trovare pattern, non firme statiche)
- Come funzionano a livello concettuale i sistemi IDS/IPS come Snort e Suricata
- La prospettiva del difensore sul traffico C2

**Da dove iniziare a leggere:**

Studia i pattern di comunicazione del beacon leggendo `beacon/beacon.py`:
- Il beacon si connette via WebSocket a un URL fisso (riga 38): `ws://localhost:8000/ws/beacon`
- Invia un messaggio `REGISTER` immediatamente alla connessione (riga 468)
- Invia messaggi `HEARTBEAT` a intervalli regolari (righe 443-453) con jitter (righe 97-102)
- L'intervallo dell'heartbeat è predefinito a 3 secondi con il 30% di jitter (righe 42-43)
- Tutti i messaggi sono codificati con XOR e poi con Base64 (righe 62-78)

Questi pattern sono rilevabili:
- Connessioni WebSocket periodiche regolari verso lo stesso endpoint
- Payload codificati in Base64 di dimensioni simili (gli heartbeat hanno sempre all'incirca la stessa lunghezza)
- Un percorso di upgrade WebSocket fisso (`/ws/beacon`)

**Suggerimenti:**

1. Usa `scapy` o `pyshark` per catturare il traffico di rete. Inizia in modo semplice: cattura tutto il traffico TCP sulla porta del server C2 (predefinita 8000) e cerca le richieste di upgrade a WebSocket.

2. Rileva il ritmo degli heartbeat. Registra i timestamp dei messaggi consecutivi provenienti dallo stesso IP di origine. Se arrivano a un intervallo regolare (più o meno il jitter), contrassegnalo come sospetto. Una funzione che calcola la deviazione standard degli intervalli tra i messaggi rivelerà la regolarità.

3. Esamina le dimensioni dei messaggi. I messaggi di heartbeat hanno sempre la stessa lunghezza in chiaro, e lo XOR non modifica la lunghezza, quindi l'output in Base64 avrà la stessa dimensione ogni volta. Se noti molti messaggi di dimensioni identiche dalla stessa sorgente, quello è un forte indicatore di C2.

4. Verifica il percorso di upgrade WebSocket. Una richiesta GET verso `/ws/beacon` è un indizio inequivocabile. In un ingaggio reale, l'operatore lo cambierebbe in qualcosa di innocuo come `/api/v2/health`. Per questa sfida, rileva il percorso predefinito.

5. Valuta di scrivere regole Snort o Suricata al posto di (o in aggiunta a) uno script Python. Una regola Snort che intercetta l'upgrade WebSocket verso `/ws/beacon` richiede una sola riga.

**File da modificare:**
- Nuovo script Python (es. `tools/detect_beacon.py`)

---

### Sfida 12: Aggiungere un canale C2 basato su DNS

**Cosa realizzare:** Un livello di trasporto alternativo in cui il beacon comunica tramite query DNS per record TXT anziché usare i WebSocket. Il beacon codifica comandi e risultati come query e risposte DNS.

**Perché è importante:** Le connessioni WebSocket verso endpoint insoliti sono relativamente facili da rilevare e bloccare. Il traffico DNS, d'altro canto, è consentito attraverso quasi tutti i firewall perché bloccare il DNS compromette la connettività generale. Il C2 basato su DNS viene usato da attori di minaccia sofisticati proprio per la permissività del DNS. Strumenti come `dnscat2`, `iodine` e il DNS beacon di Cobalt Strike sfruttano questo meccanismo.

Riferimento MITRE ATT&CK: T1071.004 (Application Layer Protocol: DNS).

**Cosa imparerai:**
- Fondamenti del protocollo DNS (tipi di query, record TXT, vincoli di codifica)
- Tecniche di tunneling DNS
- Costruire un server DNS personalizzato in Python
- I severi limiti di larghezza di banda del DNS come mezzo di trasporto

**Da dove iniziare a leggere:**

Il trasporto attuale è basato su WebSocket. Il beacon si connette in `beacon/beacon.py:466` usando `websockets.connect()`. I messaggi vengono inviati con `ws.send()` e ricevuti con `ws.recv()`. L'intero trasporto è astratto dietro queste due operazioni.

Il server riceve connessioni WebSocket in `backend/app/beacon/router.py:92`. La funzione `beacon_websocket` gestisce il ciclo di vita del WebSocket.

**Suggerimenti:**

1. Inizia comprendendo i record DNS TXT. Una query DNS per `data.tuodominio.com` può restituire testo arbitrario nel record TXT (fino a 255 byte per stringa, stringhe multiple per record). Il beacon codifica i propri dati nelle etichette del sottodominio e legge le risposte dai record TXT.

2. Costruisci un semplice server DNS usando la libreria `dnslib` di Python. Si mette in ascolto sulla porta UDP 53, riceve query, decodifica le etichette dei sottodomini per estrarre i dati del beacon e risponde con record TXT contenenti i dati del task.

3. Il vincolo di codifica è severo. Le etichette DNS sono limitate a 63 caratteri ciascuna e il nome di dominio totale non può superare i 253 caratteri. La codifica Base32 (non Base64, poiché il DNS non fa distinzione tra maiuscole e minuscole) rappresenta l'approccio standard. Ciò significa che puoi inserire circa 150 byte di dati grezzi per query.

4. I payload di grandi dimensioni (come i risultati di `sysinfo` o `proclist`) devono essere suddivisi in chunk su più query DNS. Implementa un protocollo di sequenziamento: ciascun chunk include un numero di sequenza e un conteggio totale affinché il server possa riassemblarli.

5. Il ciclo principale del beacon passa da "connetti WebSocket, invia/ricevi" a "invia periodicamente query DNS". Ciascun heartbeat diventa una query come `HEARTBEAT.beaconid.tuodominio.com`. Il server risponde con "nessun task" oppure con i dati del task codificati in un record TXT.

6. Questo è un refactoring importante. Considera di astrarre il layer di trasporto nel beacon in modo da poter passare da WebSocket a DNS senza dover riscrivere gli handler dei comandi. Crea un'interfaccia `Transport` con metodi `send()` e `receive()`.

**File da modificare:**
- Refactoring importante di `beacon/beacon.py` (astrazione del trasporto)
- Nuovo script per server DNS
- Nuova implementazione del trasporto DNS per il beacon

---

## Sfide per esperti

---

### Sfida 13: Profilo C2 malleabile (Malleable C2 Profile)

**Cosa realizzare:** Implementare il traffic shaping affinché la comunicazione C2 imiti il normale traffico HTTP. Fai apparire le richieste del beacon come se stessero prelevando risorse da un CDN, da un'API di social media o da un servizio meteo.

**Perché è importante:** I difensori di rete usano l'euristica per segnalare traffico anomalo. Connessioni WebSocket a un IP casuale sulla porta 8000 sono sospette per impostazione predefinita. Ma richieste HTTPS a ciò che sembra `api.weather.com/v2/forecast?lat=40.7&lon=-74.0` si mimetizzano perfettamente nel traffico ordinario. I "profili C2 malleabili" di Cobalt Strike sono file di configurazione che definiscono esattamente l'aspetto che deve avere il traffico. Questa è una delle tecniche di evasione più potenti nei moderni framework C2.

Riferimento MITRE ATT&CK: T1001.003 (Data Obfuscation: Protocol Impersonation).

**Da dove iniziare a leggere:**

Studia il formato attuale dei messaggi. Il beacon invia stringhe codificate in XOR+Base64 tramite WebSocket. Un osservatore di rete vede blob Base64 diretti a un endpoint WebSocket. Questo non è chiaramente traffico web normale.

Guarda come la funzione `pack` del beacon in `beacon/beacon.py:81-86` costruisce i messaggi e come la funzione `unpack` del server in `protocol.py:45-59` li analizza. La trasformazione avviene in questi due punti.

**Suggerimenti:**

1. Fai ricerche sul formato dei profili C2 malleabili di Cobalt Strike. Leggi alcuni profili di esempio per comprendere cosa significhi traffic shaping nella pratica. Concetti chiave: percorsi URI, header HTTP, codifica dei parametri, trasformazioni del body.

2. Sostituisci il trasporto WebSocket con richieste HTTPS che imitino un'API legittima. Ad esempio, il beacon potrebbe inviare gli heartbeat come `GET /api/v2/feed?user=<encoded_beacon_id>&ts=<encoded_heartbeat>` e ricevere i task nel corpo della risposta racchiusi in un finto JSON come `{"status": "ok", "data": {"items": ["<encoded_task>"]}}`.

3. Aggiungi header HTTP realistici: `User-Agent`, `Accept`, `Content-Type`, `Cache-Control`. Copia gli header da una reale richiesta da browser verso il servizio che stai impersonando.

4. Varia i percorsi URI. Non interrogare sempre lo stesso endpoint. Alterna tra `/api/feed`, `/api/profile`, `/api/notifications`, ecc. Ciascuno può trasportare lo stesso payload codificato ma apparire come chiamate API differenti.

5. Il lato server necessita di una trasformazione corrispondente. Per ciascun finto endpoint API, il server estrae i dati reali del C2 dai parametri della richiesta o dal corpo, li elabora e racchiude la risposta nello stesso finto formato API.

---

### Sfida 14: Scrivere una regola YARA

**Cosa realizzare:** Scrivere regole YARA che rilevino il codice sorgente del beacon di questo progetto su disco e i suoi pattern di traffico di rete in una cattura PCAP.

**Perché è importante:** YARA è lo strumento standard per l'identificazione del malware. I team di threat intelligence scrivono regole YARA per dare la caccia al malware noto su endpoint e catture di rete. Scrivere regole YARA per il proprio strumento ti insegna cosa rende identificabile il malware e come ridurre tali indicatori.

**Da dove iniziare a leggere:**

Studia il sorgente del beacon in `beacon/beacon.py` alla ricerca di stringhe e pattern univoci:
- La chiave XOR predefinita alla riga 41: `"c2-beacon-default-key-change-me"`
- La stringa del tipo di messaggio `REGISTER` alla riga 468
- Le chiavi del dizionario `COMMAND_HANDLERS` alle righe 419-430
- Combinazioni di import: `psutil`, `websockets`, `mss`, `pynput` insieme in un unico file è inusuale
- La funzione `collect_system_info` alla riga 105 raccoglie hostname, OS, username, PID, IP interno e architettura, costituendo una fingerprint distintiva

**Suggerimenti:**

1. Installa YARA (`sudo apt install yara` o `pip install yara-python`) e leggi la documentazione sulla sintassi delle regole. Una regola di base richiede: `rule nome { strings: $s1 = "pattern" condition: $s1 }`.

2. Per rilevare il codice sorgente del beacon su disco, cerca combinazioni univoche di stringhe. Qualsiasi singola stringa potrebbe comparire in codice legittimo, quindi combina più stringhe con condizioni `and`. Ad esempio: la chiave XOR E il nome della funzione `collect_system_info` E il nome della variabile `COMMAND_HANDLERS`.

3. Per rilevare il traffico di rete, cattura un PCAP del beacon che comunica con il server (usa `tcpdump` o Wireshark). Cerca la richiesta di upgrade WebSocket verso `/ws/beacon` e i periodici messaggi codificati in Base64.

4. Testa le tue regole sia contro il file del beacon SIA contro una serie di file Python benigni per verificare di non generare falsi positivi. Una regola che fa match su ogni script Python è inutile.

5. Valuta di scrivere regole separate: una per il "sorgente del beacon su disco" (matching di testo) e una per il "traffico del beacon in PCAP" (matching di sequenze di byte rispetto al formato del messaggio codificato).

---

### Sfida 15: Implementare il Process Injection

**Cosa realizzare:** Invece di eseguire il beacon come un visibile processo Python, inietta il codice del beacon all'interno di un altro processo in esecuzione in modo che si nasconda all'interno di un programma legittimo.

**Perché è importante:** Un processo autonomo `python3 beacon.py` è visibile in qualsiasi elenco dei processi. Qualsiasi amministratore di sistema che esegua `ps aux` lo noterà. Il process injection nasconde il beacon all'interno di un processo fidato come `sshd`, `nginx` o `bash`, rendendolo molto più difficile da rilevare.

Riferimento MITRE ATT&CK: T1055 (Process Injection).

**Da dove iniziare a leggere:**

Il beacon attualmente gira come processo autonomo. Guarda `beacon/beacon.py:513-514` dove `asyncio.run(main())` avvia l'intero sistema. La funzione `collect_system_info` alla riga 105 riporta il proprio PID con `os.getpid()`.

**Suggerimenti:**

1. Su Linux, studia l'injection basata su `ptrace`. La system call `ptrace` permette a un processo di controllarne un altro. Puoi agganciarti a un processo bersaglio, allocare memoria nel suo spazio di indirizzamento, scrivere il tuo shellcode o bytecode Python e creare un nuovo thread per eseguirlo.

2. Un approccio più semplice su Linux è l'injection tramite `LD_PRELOAD`. Crea una libreria condivisa che avvii il beacon in un thread in background una volta caricata. Quindi avvia un programma legittimo con `LD_PRELOAD=./beacon.so /usr/bin/un_programma`. Il beacon verrà eseguito all'interno del processo di quel programma.

3. L'approccio specifico di Python: usa `ctypes` per chiamare `dlopen()` e iniettare una libreria condivisa nel processo corrente, oppure usa `/proc/{pid}/mem` per scrivere nello spazio di memoria di un altro processo (richiede privilegi di root).

4. Questa sfida è notevolmente più impegnativa delle altre. Inizia facendo funzionare l'injection con `LD_PRELOAD` prima di cimentarti con `ptrace`. La libreria `ctypes` in Python può interfacciarsi con system call a livello C, ma probabilmente dovrai scrivere un piccolo wrapper in C.

5. Considera le implicazioni etiche. Il process injection è una tecnica utilizzata sia dai red team (test autorizzati) sia dal malware reale. Testalo unicamente in ambienti di laboratorio isolati di tua proprietà. Eseguirlo su sistemi per i quali non possiedi un'autorizzazione esplicita è illegale.

---

## Mettiti ulteriormente alla prova

Questi sono progetti aperti che vanno oltre la modifica di questa codebase. Sviluppano le competenze più ampie essenziali per una carriera nella sicurezza offensiva o difensiva.

**Confronta con framework C2 di produzione.** Installa Sliver (open source, basato su Go) o configura una versione di prova di Cobalt Strike. Eseguili fianco a fianco con questo progetto. Confronta l'architettura: come gestisce Sliver la registrazione dei beacon? In cosa differisce il team server di Cobalt Strike dal nostro `OpsManager`? Quali funzionalità possiedono che noi abbiamo tralasciato? Documenta le differenze.

**Cattura e analizza il tuo stesso traffico.** Avvia Wireshark mentre il beacon comunica con il server. Salva la cattura come file PCAP. Identifica l'handshake WebSocket, il pattern degli heartbeat e gli scambi task/risultato. Scrivi regole Snort o Suricata in grado di rilevare questo traffico su una rete aziendale. Testa tali regole contro il PCAP per verificare che scattino.

**Esegui il deployment su una rete reale.** Esegui il server C2 su un VPS (DigitalOcean, AWS, Linode) e il beacon su una macchina diversa, un altro VPS o una VM locale. Questo ti esporrà a sfide di rete reali: firewall, attraversamento NAT, risoluzione DNS, certificati TLS per connessioni WebSocket sicure e latenza. La maggior parte dello sviluppo locale aggira del tutto questi aspetti.

**Mappa su MITRE ATT&CK.** Leggi le pagine di ATT&CK per ciascuna tecnica implementata dai nostri comandi. Il beacon supporta: T1059.004 (shell), T1082 (sysinfo), T1057 (proclist), T1105 (upload/download), T1113 (screenshot), T1056.001 (keylogging), T1053.003 (persist via cron), T1029 (sleep/jitter per trasferimento pianificato). Per ciascuna tecnica, leggi la sezione "Detection". Cosa cercherebbe un difensore? La nostra implementazione lascia tali artefatti?

**Costruisci un laboratorio con strumenti di detection.** Allestisci uno stack ELK (Elasticsearch, Logstash, Kibana) o Wazuh sulla stessa rete del beacon. Configura Sysmon (Windows) o auditd (Linux) per registrare la creazione dei processi, le connessioni di rete e le modifiche ai file. Esegui il beacon e osserva quali alert vengono generati. Questo ti darà la prospettiva del difensore su ogni comando che esegui.

---

## Monitoraggio del completamento delle sfide

Usa questo elenco per tracciare i tuoi progressi. Spunta ciascuna sfida man mano che la completi.

- [ ] Facile 1: Aggiungere un comando `pwd`
- [ ] Facile 2: Visualizzare l'uptime del beacon
- [ ] Facile 3: Aggiungere un comando terminale `clear`
- [ ] Facile 4: Aggiungere il conteggio delle connessioni nell'Header
- [ ] Intermedia 5: Aggiungere l'autenticazione dell'operatore
- [ ] Intermedia 6: Aggiungere altri pulsanti Quick Action
- [ ] Intermedia 7: Persistenza della cronologia dei task
- [ ] Intermedia 8: Aggiungere la cifratura AES-256
- [ ] Avanzata 9: Supporto multi-operatore con ruoli
- [ ] Avanzata 10: Staging del beacon
- [ ] Avanzata 11: Costruire uno strumento di rilevamento
- [ ] Avanzata 12: Aggiungere un canale C2 basato su DNS
- [ ] Per esperti 13: Profilo C2 malleabile
- [ ] Per esperti 14: Scrivere una regola YARA
- [ ] Per esperti 15: Implementare il Process Injection