# Architettura del sistema

Questo documento analizza come è progettato il port scanner e perché l'I/O asincrono con worker concorrenti offre sia velocità sia chiarezza.

## Architettura di alto livello
```
┌─────────────────────────────────────┐
│      Command Line Interface         │
│   (Boost.Program_Options Parser)    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│        PortScanner Object           │
│   - Configuration Management        │
│   - Work Queue (ports to scan)      │
│   - Thread/Concurrency Control      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│       Boost.Asio io_context         │
│    (Event Loop / Async Runtime)     │
└──────────────┬──────────────────────┘
               │
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐  ┌─────────────┐
│   Socket    │  │    Timer    │
│   (TCP      │  │  (Timeout   │
│ Connection) │  │  Detection) │
└─────────────┘  └─────────────┘
       │               │
       └───────┬───────┘
               ▼
       ┌───────────────┐
       │    Target     │
       │  Host:Port    │
       └───────────────┘
```

### Suddivisione dei componenti

**Interfaccia a riga di comando (main.cpp)**
- Scopo: fare il parsing dell'input dell'utente e inizializzare lo scanner con la configurazione
- Responsabilità: validare gli argomenti, impostare i default, mostrare il testo di aiuto e gli esempi d'uso
- Interfacce: crea e configura un oggetto `PortScanner`, poi chiama `start()` e `run()`

**Controller PortScanner (classe PortScanner)**
- Scopo: orchestrare il processo di scansione e gestire le operazioni concorrenti
- Responsabilità: mantenere la coda di lavoro delle porte da scansionare, imporre i limiti di thread, tenere traccia delle statistiche (conteggi di porte aperte/chiuse/filtrate), fornire la formattazione dei risultati
- Interfacce: espone i metodi `set_options()`, `start()` e `run()`; internamente usa le primitive di Boost.Asio

**io_context di Boost.Asio**
- Scopo: event loop che guida tutte le operazioni asincrone
- Responsabilità: pianificare le operazioni asincrone sui socket e le callback dei timer, eseguire il dispatch dei completion handler quando l'I/O si completa, gestire lo strand di esecuzione per la thread safety
- Interfacce: fornisce le operazioni async_connect, async_read_some e async_wait usate dal nostro scanner

**Coppia socket e timer**
- Scopo: ogni scansione di porta usa un socket (per la connessione) e un timer (per il timeout)
- Responsabilità: il socket tenta la connessione TCP; il timer gareggia contro il socket per rilevare le porte filtrate
- Interfacce: i completion handler scattano quando il socket si connette/fallisce oppure quando il timer scade

## Flusso dei dati

### Flusso di scansione principale

Walkthrough passo per passo di ciò che succede quando esegui `./simplePortScanner -i 192.168.1.1 -p 80-443`:
```
1. main.cpp:12-23 → Parse command line arguments
   Extracts IP (192.168.1.1), port range (80-443), thread count (default 100), timeout (default 2 sec)

2. main.cpp:37-40 → Initialize PortScanner
   Calls set_options() which resolves DNS to IP address endpoint

3. PortScanner.cpp:77-82 → setup_queue()
   Fills queue with ports 80, 81, 82, ... 443 (364 ports total)

4. PortScanner.cpp:109-115 → start()
   Posts MAX_THREADS work items to io_context via strand
   Each work item is a call to scan() function

5. main.cpp:41 → run()
   Calls io.run() which blocks until all async operations complete

6. PortScanner.cpp:123-165 → scan() (called MAX_THREADS times concurrently)
   Pops port from queue, creates socket and timer, races them
   
   IF timeout expires first (line 130-136):
       → Port is FILTERED
       → Print result, decrement counter, recursively call scan() for next port
   
   IF connection succeeds (line 144-151):
       → Port is OPEN
       → Try banner grab (async_read_some)
       → Print result with banner, decrement counter, call scan() again
   
   IF connection fails (line 153-158):
       → Port is CLOSED
       → Print result, decrement counter, call scan() again

7. When queue is empty → io.run() completes → main.cpp:117-120 prints summary
```

Esempio con riferimenti al codice:
```
1. User runs command → main() (main.cpp:6)
   Boost.Program_Options parses to variables

2. Variables → PortScanner.set_options() (PortScanner.cpp:85-95)
   DNS resolution happens: resolver.resolve(domainName, "")
   Stores endpoint for later use

3. PortScanner.start() → Fills queue, posts work (PortScanner.cpp:109-115)
   100 async scan() operations begin

4. Each scan() → Creates socket + timer pair (PortScanner.cpp:123-127)
   Both operations start simultaneously
   Whoever completes first cancels the other

5. Completion handler → Determines port state (PortScanner.cpp:129-165)
   Prints result, decrements active counter, calls scan() to grab next port from queue

6. Queue exhausted → io.run() returns (main.cpp:41)
   Final statistics printed
```

### Flusso secondario di risoluzione DNS

Prima che avvenga qualsiasi scansione di porte, risolviamo il nome di dominio:
```
1. User provides "-i scanme.nmap.org" → stored as string
2. PortScanner.set_options() calls resolver.resolve(domainName, "")
3. Boost.Asio performs DNS lookup (A or AAAA record)
4. Result converted to tcp::endpoint with IP address
5. All subsequent connections use this cached endpoint
```

Questo avviene in modo sincrono all'avvio. Se il DNS fallisce, il programma va in errore immediatamente, prima che inizi qualsiasi scansione. Per gli indirizzi IP (come 192.168.1.1), la risoluzione è banale e si limita a validarne il formato.

## Design pattern

### I/O asincrono con completion handler

**Che cos'è:**
I/O non bloccante in cui le operazioni ritornano immediatamente e le callback scattano al loro completamento. Invece di aspettare la connessione di un socket (che potrebbe richiedere secondi), avviamo l'operazione e forniamo una funzione da chiamare quando termina.

**Dove lo usiamo:**
Ogni operazione di rete dello scanner:
- `async_connect` per le connessioni TCP (PortScanner.cpp:138)
- `async_read_some` per il banner grabbing (PortScanner.cpp:143)
- `async_wait` per il rilevamento del timeout (PortScanner.cpp:128)

**Perché l'abbiamo scelto:**
Scansionare 65.535 porte in modo sincrono richiederebbe ore. Anche a 100ms per porta (rete locale veloce), sono 1,8 ore. Con l'I/O asincrono e 100 operazioni concorrenti, finiamo in pochi minuti. Il pattern inoltre scala: cambiare il numero di thread è un solo parametro.

**Compromessi:**
- Pro: concorrenza massiccia con pochi thread effettivi, uso efficiente delle risorse, scala fino a migliaia di operazioni simultanee
- Contro: flusso del codice più complesso (callback invece di logica lineare), più difficile da debuggare (gli stack trace mostrano il meccanismo asincrono), richiede la comprensione degli event loop

Esempio di implementazione:
```cpp
// PortScanner.cpp:138-165
socket->async_connect(endpoint, boost::asio::bind_executor(strand, 
    [this, socket, timer, port, complete](boost::system::error_code ec) {
        if (*complete) return;  // Timer already fired, ignore this
        *complete = true;
        timer->cancel();        // Stop the race, we won
        
        if (!ec) {
            // Connection succeeded - port is OPEN
            async_read_some(...);  // Try to grab banner
        } else {
            // Connection failed - port is CLOSED
            print_result(...);
        }
        scan();  // Tail recursion to get next port
    }
));
```

La lambda cattura lo stato condiviso (`socket`, `timer`, il flag `complete`) e viene eseguita più tardi, quando il tentativo di connessione termina. Questo flusso non lineare abilita la concorrenza.

### Coda di lavoro con concorrenza fissa

**Che cos'è:**
Una coda di lavoro in sospeso (porte da scansionare) con un numero fisso di worker che vi attingono. Man mano che ogni worker completa, prende l'elemento successivo. Questo evita di generare 65.535 thread e di sovraccaricare il sistema.

**Dove lo usiamo:**
- Coda: `std::queue<uint16_t> q` (PortScanner.hpp:24) riempita in `setup_queue()` (PortScanner.cpp:77-82)
- Limite di concorrenza: `MAX_THREADS` (default 100) controlla quante scansioni girano simultaneamente
- Prelievo del lavoro: `scan()` estrae dalla coda (PortScanner.cpp:123), elabora, poi chiama sé stessa ricorsivamente per la porta successiva

**Perché l'abbiamo scelto:**
Semplice da capire e da implementare. La coda gestisce naturalmente la distribuzione del lavoro - nessuna logica di scheduling complessa. Quando una scansione termina velocemente (porta chiusa), il worker ne prende immediatamente un'altra. Le scansioni lente (porte aperte con banner grab) non bloccano le altre porte.

**Compromessi:**
- Pro: facile da ragionare, bilanciamento del carico automatico, imposizione semplice del limite di thread
- Contro: non perfettamente efficiente (se le ultime porte sono lente, i worker restano inattivi), non dà priorità alle porte interessanti

### Strand per la thread safety

**Che cos'è:**
Un costrutto di Boost.Asio che serializza l'esecuzione degli handler. Quando più operazioni asincrone si completano, lo strand garantisce che i loro handler non vengano eseguiti simultaneamente. Questo fornisce thread safety senza lock espliciti.

**Dove lo usiamo:**
```cpp
// PortScanner.hpp:23
boost::asio::strand<boost::asio::io_context::executor_type> strand{io.get_executor()};

// All async operations wrapped in bind_executor(strand, ...)
// PortScanner.cpp:111, 129, 139, 144
boost::asio::post(strand, [this]() { scan(); });
boost::asio::bind_executor(strand, [...](...) { ... });
```

**Perché l'abbiamo scelto:**
Più completion handler modificano lo stato condiviso (`cnt`, `q`, contatori delle statistiche). Senza sincronizzazione, le race condition corrompono i dati. Lo strand garantisce che, anche se 100 operazioni girano in modo concorrente, i loro completion handler vengano eseguiti uno alla volta.

**Compromessi:**
- Pro: thread-safe senza lock manuali, nessun rischio di deadlock, codice pulito senza gestione di mutex
- Contro: leggero costo prestazionale dovuto alla serializzazione (trascurabile per il nostro carico di lavoro), tutti gli handler devono essere incapsulati in modo coerente

## Separazione dei livelli

Lo scanner ha tre livelli distinti:
```
┌────────────────────────────────────┐
│    Presentation Layer              │
│    - CLI parsing (main.cpp)        │
│    - Output formatting             │
│    - Color codes for terminal      │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    Business Logic Layer            │
│    - PortScanner class             │
│    - Scanning algorithm            │
│    - State management (counters)   │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    I/O Layer                       │
│    - Boost.Asio runtime            │
│    - Socket operations             │
│    - Timer operations              │
└────────────────────────────────────┘
```

### Perché i livelli?

La separazione delle responsabilità rende ogni componente testabile e sostituibile:

- Vuoi una GUI invece della CLI? Sostituisci il livello di presentazione, mantieni la business logic.
- Vuoi passare da Boost.Asio ai socket POSIX grezzi? Sostituisci il livello di I/O, la business logic resta invariata.
- Vuoi aggiungere tipi di scansione diversi (UDP, SYN scan)? Estendi la business logic senza toccare la presentazione.

### Cosa sta dove

**Livello di presentazione (main.cpp):**
- File: `main.cpp`
- Import: può importare la business logic (classe PortScanner), usa Boost.Program_Options per il parsing della CLI
- Vietato: non deve creare direttamente socket o timer, non deve implementare la logica di scansione

**Livello di business logic (classe PortScanner):**
- File: `src/PortScanner.hpp`, `src/PortScanner.cpp`
- Import: può importare il livello di I/O (Boost.Asio), non può importare il livello di presentazione
- Vietato: non deve gestire il parsing della riga di comando né la formattazione dell'output (restituisce solo dati)

**Livello di I/O (Boost.Asio):**
- File: libreria esterna (Boost)
- Import: libreria standard, API dei socket a livello di sistema operativo
- Vietato: nessuna business logic su porte o scansioni

Questa struttura significa che main.cpp conosce PortScanner, PortScanner conosce Asio, ma Asio non sa nulla di scansioni, e la scansione non sa nulla dei flag della CLI.

## Modelli dei dati

### Voce della coda delle porte
```cpp
// PortScanner.hpp:24
std::queue<std::uint16_t> q;
```

**Campi spiegati:**
- Solo il numero di porta (0-65535), memorizzato come `uint16_t` per risparmiare memoria
- La coda è elaborata in ordine FIFO - le porte vengono scansionate in ordine (80, 81, 82, ...)

**Relazioni:**
- Popolata da `parse_port()`, che converte input dell'utente come "80-443" in singoli numeri di porta
- Consumata da `scan()`, che estrae le porte una alla volta

### Stato dello scanner
```cpp
// PortScanner.hpp:25-29
int cnt = 0;                // Active concurrent scans
int MAX_THREADS = 0;        // Concurrency limit
int open_ports = 0;         // Statistics
int closed_ports = 0;
int filtered_ports = 0;
```

**Campi spiegati:**
- `cnt`: quante operazioni `scan()` sono attualmente in corso. Impedisce di generare troppi worker.
- `MAX_THREADS`: limite di concorrenza configurabile dall'utente. Il default è 100 in main.cpp:15.
- Contatori delle statistiche: incrementati man mano che arrivano i risultati, stampati alla fine per il riepilogo.

**Relazioni:**
- `cnt` protegge la coda di lavoro - se `cnt >= MAX_THREADS`, non partono altre scansioni anche se la coda contiene porte
- Statistiche tracciate in ogni completion handler (PortScanner.cpp:135, 148, 156)

### Mappa delle porte well-known
```cpp
// PortScanner.cpp:3-24
const std::unordered_map<uint16_t, std::string> PortScanner::basicPorts{
    {21, "FTP"},
    {22, "SSH"},
    {80, "HTTP"},
    {443, "HTTPS"},
    ...
};
```

**Campi spiegati:**
- Mappatura costante statica dai numeri di porta ai nomi dei servizi
- Usata solo per la visualizzazione - non influisce sulla logica di scansione

**Relazioni:**
- Consultata nel completion handler (PortScanner.cpp:142) per mostrare il nome del servizio invece del solo numero di porta
- Le porte mancanti vengono visualizzate come "---" (PortScanner.cpp:140)

## Architettura di sicurezza

### Threat model

Da cosa ci proteggiamo:

1. **Interruzioni accidentali della rete** - Scansionare in modo troppo aggressivo potrebbe mandare in crash i sistemi target o gli apparati di rete. I limiti di thread e i timeout evitano di sovraccaricare i target.

2. **Responsabilità legale** - Scansionare reti che non possiedi è spesso illegale (CFAA negli USA). Il tool include avvertenze d'uso per istruire gli utenti sui limiti legali.

3. **Rilevamento da parte di IDS/IPS** - Pur non essendo orientato allo stealth, lo scanner può essere configurato con un numero di thread più basso e timeout più lunghi per ridurre la probabilità di rilevamento.

Da cosa NON ci proteggiamo (fuori ambito):

- **Elusione del rilevamento** - Questo è uno scanner di base. Un IDS avanzato lo intercetterà. Le tecniche stealth (SYN scan, frammentazione, decoy) sono fuori ambito per un progetto per principianti.
- **DoS del sistema target** - Limitiamo i thread ma non implementiamo rate limiting sofisticati o backoff. Una scansione configurata male potrebbe comunque sovraccaricare un target debole.

### Livelli di difesa

Lo scanner in sé è uno strumento di ricognizione, ma capire la defense-in-depth aiuta gli utenti a proteggersi dall'essere scansionati:
```
Layer 1: Firewall (prevents scan completion)
    ↓
Layer 2: IDS (detects scan pattern)
    ↓
Layer 3: Rate limiting (slows attacker)
```

**Perché più livelli?**

Se il firewall fallisce (regola configurata male), l'IDS avvisa il team di sicurezza. Se l'IDS non rileva la scansione (tecnica di evasione), il rate limiting impedisce un'enumerazione rapida. Ogni livello compensa i fallimenti degli altri.

## Configurazione

### Variabili d'ambiente

Questo scanner usa argomenti da riga di comando, non variabili d'ambiente:
```bash
./simplePortScanner \
  -i TARGET          # IP or domain name (default: 127.0.0.1)
  -p PORT_RANGE      # "80" or "1-1024" or "22,80,443" (default: 1-1024)
  -t THREADS         # Max concurrent scans (default: 100)
  -e TIMEOUT         # Seconds to wait before marking filtered (default: 2)
  -v                 # Verbose output (not yet implemented)
  -h                 # Help message
```

### Strategia di configurazione

**Sviluppo:**
Usa un numero basso di thread (`-t 10`) e intervalli di porte piccoli (`-p 80-100`) per testare senza sovraccaricare la tua rete. Scansiona localhost per verificarne il funzionamento.

**Produzione:**
Le scansioni reali usano una concorrenza più alta (`-t 200` o più) per la velocità. Regola il timeout in base alla latenza di rete - le reti locali possono usare 1 secondo, le scansioni su internet richiedono 3-5 secondi. Ottieni sempre il permesso prima di scansionare host esterni.

## Considerazioni sulle prestazioni

### Colli di bottiglia

Dove questo sistema rallenta sotto carico:

1. **La latenza di rete domina** - Anche con un'alta concorrenza, non puoi scansionare più velocemente del round-trip time della rete. Su una connessione con 50ms di latenza, ogni porta richiede almeno 50ms indipendentemente da quanti thread usi.

2. **La risoluzione DNS è sincrona** - La chiamata iniziale `resolver.resolve()` è bloccante. Per i domini con DNS lento, questo ritarda l'inizio della scansione. Mettere in cache gli IP risolti potrebbe aiutare le scansioni ripetute.

### Ottimizzazioni

Cosa abbiamo fatto per renderlo più veloce:

- **I/O asincrono**: il grande guadagno. Una scansione sincrona di 10.000 porte a 100ms l'una = 16 minuti. Asincrona con 100 thread = ~10 secondi.

- **Ottimizzazione con shared pointer** (PortScanner.cpp:125-127): socket e timer creati come `std::shared_ptr`. I completion handler li catturano, garantendo la gestione del ciclo di vita senza pulizia manuale.

### Scalabilità

**Scaling verticale:**
Aumenta MAX_THREADS (fino a ~1000 prima di raggiungere i limiti dei file descriptor sulla maggior parte dei sistemi). Più thread = più scansioni concorrenti = completamento più rapido, ma con rendimenti decrescenti oltre la capacità della rete.

**Scaling orizzontale:**
Suddividi gli intervalli di IP tra più istanze dello scanner. Scansiona 192.168.1.0/24 eseguendo 4 istanze che gestiscono 64 IP ciascuna. Questo parallelizza il collo di bottiglia (la latenza di rete) tra più macchine.

## Decisioni di design

### Decisione 1: connect scan vs SYN scan

**Cosa abbiamo scelto:**
Connect scan TCP completa (three-way handshake completo)

**Alternative considerate:**
- SYN scan (scansione half-open): inviare SYN, leggere SYN-ACK, inviare RST invece di completare l'handshake
- ACK scan: inviare un pacchetto ACK per rilevare le regole del firewall
- UDP scan: inviare pacchetti UDP per controllare i servizi non TCP

**Perché abbiamo scelto la connect scan:**
La SYN scan richiede raw socket, che su Linux richiedono privilegi di root. Questo aggiunge complessità di deployment e rischi di sicurezza. Le connect scan funzionano come utenti non privilegiati e si integrano in modo pulito con l'API di alto livello di Boost.Asio.

**Compromessi:**
- Pro: nessun privilegio speciale necessario, codice più semplice, multipiattaforma (funziona su Windows/Linux/macOS), meno probabilità di mandare in crash stack di rete difettosi
- Contro: più rumorosa (appare chiaramente nei log come connessioni completate), leggermente più lenta (handshake completo vs solo SYN), alcuni sistemi registrano i tentativi di connessione in modo diverso dai SYN

### Decisione 2: filtering basato su timer vs analisi ICMP

**Cosa abbiamo scelto:**
Usare la durata del timeout per dedurre le porte filtrate

**Alternative considerate:**
- Ascoltare i messaggi ICMP "port unreachable" per distinguere le porte chiuse da quelle filtrate
- Inviare più tipi di sonda (SYN, ACK, FIN) e correlare le risposte

**Perché abbiamo scelto i timeout:**
L'ascolto ICMP richiede raw socket (di nuovo, privilegi di root). I packet filter spesso scartano comunque l'ICMP, rendendolo inaffidabile. I timeout funzionano ovunque e gestiscono correttamente il caso comune (il firewall scarta silenziosamente i pacchetti).

**Compromessi:**
- Pro: funziona senza privilegi, gestisce correttamente le porte filtrate, semplice da implementare
- Contro: aggiunge latenza alle scansioni (bisogna attendere l'intero timeout), non può distinguere "filtrata dal firewall" da "rete non raggiungibile", falsi positivi se la rete è semplicemente lenta

### Decisione 3: scan() ricorsiva vs worker pool

**Cosa abbiamo scelto:**
Chiamate ricorsive in coda a `scan()` per la distribuzione del lavoro

**Alternative considerate:**
- Avviare in anticipo N thread worker che ciclano prelevando dalla coda
- Usare una libreria di thread pool con work stealing

**Perché abbiamo scelto la ricorsione:**
Si adatta naturalmente ai completion handler asincroni. Quando una scansione termina, il completion handler chiama semplicemente di nuovo `scan()`. L'event loop di Boost.Asio gestisce lo scheduling.

**Compromessi:**
- Pro: codice minimo, nessuna gestione manuale dei thread, distribuzione automatica del lavoro
- Contro: la profondità dello stack aumenta (anche se la tail call optimization aiuta), minore controllo sul ciclo di vita dei worker, più difficile implementare scheduling avanzati

## Prossimi passi

Ora che hai capito l'architettura:

1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per un walkthrough dettagliato del codice che mostra come si coordinano le operazioni asincrone
2. Prova a modificare il modello di concorrenza - cosa succede se rimuovi lo strand? (Le race condition corromperanno i contatori)
3. Sperimenta con i valori di timeout per vedere come la latenza di rete influisce sulla durata della scansione
