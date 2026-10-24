# Architettura - Progettazione del Sistema e Decisioni Tecniche

## Architettura ad Alto Livello

```
┌────────────────────────────────────────────────────────┐
│                      User Space                        │
│                                                        │
│  ┌─────────┐                                           │
│  │  main.py │ CLI entrypoint                           │
│  │  (Typer) │ parses args, wires components            │
│  └────┬─────┘                                          │
│       │                                                │
│       ▼                                                │
│  ┌──────────┐   ┌─────────────┐   ┌────────────────┐  │
│  │loader.py │──▶│processor.py │──▶│  renderer.py   │  │
│  │          │   │             │   │                │  │
│  │ Compiles │   │ Parses raw  │   │ JSON / Live /  │  │
│  │ & loads  │   │ events,     │   │ Table output   │  │
│  │ eBPF C   │   │ enriches    │   │                │  │
│  │ programs │   │ from /proc  │   └────────────────┘  │
│  │          │   │             │                        │
│  │ Sets up  │   │ Filters by  │                        │
│  │ ring buf │   │ severity,   │                        │
│  │ callback │   │ PID, comm   │   ┌────────────────┐  │
│  └──────────┘   │             │──▶│  detector.py   │  │
│                 └─────────────┘   │                │  │
│                                   │ Stateless      │  │
│                                   │ rules +        │  │
│                                   │ stateful       │  │
│                                   │ correlation    │  │
│                                   └────────────────┘  │
├────────────────────────────────────────────────────────┤
│                     Kernel Space                       │
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │              Ring Buffer (shared)            │      │
│  │         BPF_RINGBUF_OUTPUT, 256KB            │      │
│  └──────────┬──────────┬──────────┬─────────────┘      │
│             │          │          │                     │
│  ┌──────────┴───┐ ┌────┴────┐ ┌──┴──────────┐         │
│  │process_tracer│ │file_    │ │network_     │         │
│  │   .c        │ │tracer.c │ │tracer.c     │         │
│  │             │ │         │ │             │         │
│  │ sys_enter_  │ │sys_enter│ │sys_enter_   │         │
│  │ execve      │ │_openat  │ │connect      │         │
│  │ sys_enter_  │ │sys_enter│ │sys_enter_   │         │
│  │ clone       │ │_unlinkat│ │accept4      │         │
│  │             │ │sys_enter│ │sys_enter_   │         │
│  │             │ │_rename  │ │bind/listen  │         │
│  └─────────────┘ └─────────┘ └─────────────┘         │
│  ┌─────────────┐ ┌─────────────┐                      │
│  │privilege_   │ │system_      │                      │
│  │tracer.c     │ │tracer.c     │                      │
│  │             │ │             │                      │
│  │sys_enter_   │ │sys_enter_   │                      │
│  │setuid       │ │ptrace       │                      │
│  │sys_enter_   │ │sys_enter_   │                      │
│  │setgid       │ │mount        │                      │
│  │             │ │sys_enter_   │                      │
│  │             │ │init_module  │                      │
│  └─────────────┘ └─────────────┘                      │
└────────────────────────────────────────────────────────┘
```

## Suddivisione dei Componenti

### main.py - CLI e Orchestrazione
Esegue il parsing degli argomenti da riga di comando tramite Typer e collega insieme loader, processor, detector e renderer. Gestisce lo shutdown basato sui segnali. È il livello più sottile: non contiene logica di business, solo collegamenti.

### loader.py - Ciclo di Vita dei Programmi eBPF
Legge i file `.c` dalla directory `ebpf/`, li compila tramite BCC, li aggancia ai tracepoint del kernel e configura il polling del ring buffer. Gestisce anche la pulizia: stacca i programmi eBPF e libera gli oggetti BPF quando il tool si ferma.

### processor.py - Parsing e Arricchimento degli Eventi
Definisce `RawEvent` (una Structure ctypes che rispecchia la struct C) e `TracerEvent` (una dataclass Python con campi arricchiti). Converte i byte grezzi del ring buffer in oggetti Python strutturati. Arricchisce gli eventi con dati provenienti da `/proc` (nome del processo padre, risoluzione del nome utente). Implementa la logica di filtro.

### detector.py - Motore di Detection
Contiene tutta la logica di detection di sicurezza. Le regole stateless valutano i singoli eventi (ad esempio, "questo è un setuid(0) da parte di un non-root?"). Le regole stateful correlano gli eventi nel tempo usando una finestra scorrevole per PID (ad esempio, "c'è stata una connect prima di questo execve della shell?"). Restituisce oggetti Detection che vengono impressi sugli eventi.

### renderer.py - Formattazione dell'Output
Tre modalità di output. `LiveRenderer` usa Rich per lo streaming con codifica a colori. `JsonRenderer` scrive un oggetto JSON per riga su stdout. `TableRenderer` mette in buffer gli eventi e renderizza periodicamente tabelle Rich. `FileRenderer` scrive JSON su file in parallelo a qualsiasi altra modalità di output.

### config.py - Costanti e Metadati delle Regole
Tutti i magic number, i path dei file, le definizioni delle regole di detection, i livelli di severity e le mappature dei tipi di evento vivono qui. Nient'altro è hardcoded altrove. Cambiare la severity di una regola di detection o aggiungere un nuovo path di file sensibile richiede solo di modificare questo file.

### ebpf/*.c - Programmi Kernel-Space
Cinque file C, uno per categoria di syscall. Ognuno definisce un `TRACEPOINT_PROBE` che scatta sul corrispondente evento `syscalls:sys_enter_*`. I programmi catturano i dati dell'evento in una struct condivisa e li inviano al ring buffer. Il codice C è volutamente minimale, tutta la logica di detection resta in Python.

## Flusso dei Dati

### Passo dopo Passo: Dalla Syscall all'Alert

```
1. Process calls execve("/bin/bash")
       │
2. Kernel hits tracepoint syscalls:sys_enter_execve
       │
3. eBPF program (process_tracer.c) fires:
   - Reserves space in ring buffer
   - Fills struct: pid, ppid, uid, comm, filename, timestamp
   - Submits to ring buffer
       │
4. Python callback (on_event in main.py) fires:
   - parse_raw_event() casts raw bytes to RawEvent ctypes struct
   - Converts to TracerEvent dataclass
   - Decodes comm/filename from null-terminated bytes
   - Converts kernel timestamp to wall clock datetime
   - Resolves UID to username via pwd module
       │
5. enrich_event() adds parent process name from /proc
       │
6. detector.evaluate() checks:
   - Stateless: Is the event itself suspicious? No.
   - Stateful: Is this a shell? Yes (bash). Was there a
     recent connect from this PID? Check history deque.
     If yes -> Detection("Reverse Shell", CRITICAL)
       │
7. should_include() applies user's filters:
   - Severity >= minimum? PID matches? Comm matches?
       │
8. renderer.render() outputs:
   [14:30:01] CRITICAL  execve  pid=1234 comm=bash
              /bin/bash [Reverse Shell]
```

## Design Pattern

### Pattern: Semplicità nel Kernel, Complessità in Userspace

I programmi C eBPF fanno il minimo indispensabile: leggono gli argomenti della syscall, riempiono una struct, la inviano al ring buffer. Tutto il lavoro interessante (detection, correlazione, arricchimento, formattazione) avviene in Python.

Perché? I programmi eBPF girano dentro il kernel con vincoli rigidi:
- Limite di stack di 512 byte
- Nessuna allocazione dinamica di memoria
- Nessuna manipolazione di stringhe oltre `bpf_probe_read_*`
- Il verifier rifiuta tutto ciò che è complesso

Spostare la logica in userspace significa anche poter cambiare le regole di detection senza ricompilare i programmi eBPF, e poter fare unit test della logica di detection senza privilegi di root.

### Pattern: Struct di Evento Unica

Tutti e cinque i programmi eBPF usano lo stesso layout `struct event`, anche se non ogni campo è rilevante per ogni tipo di evento. Un evento di processo non ha bisogno di `addr_v4` e un evento di rete non ha bisogno di `filename`, ma condividono la stessa struct.

Sembra uno spreco (la struct è di ~300 byte con campi per lo più a zero nella maggior parte degli eventi), ma ha grandi vantaggi:
- Una sola definizione ctypes `RawEvent` in Python, non cinque
- Un solo callback del ring buffer, non cinque
- Codice più semplice, meno bug

L'alternativa (struct per tipo con union discriminate) risparmierebbe memoria ma aggiungerebbe una complessità che a questa scala non è giustificata.

### Pattern: Correlazione Basata su Deque

Il motore di detection mantiene una `collections.deque` per PID con una lunghezza massima. Gli eventi più vecchi della finestra di correlazione vengono rimossi a ogni valutazione. Questo dà un append O(1) e una scansione O(n) dove n è piccolo (massimo 64 eventi per PID, finestra di 10 secondi).

Per un tool che traccia un tipico server, questo significa ~1000 deque in memoria (una per PID attivo), ciascuna contenente pochi eventi. Memoria totale per la correlazione: al massimo qualche megabyte.

### Trade-off

**Ring buffer vs perf buffer**: Il ring buffer (usato qui) richiede il kernel 5.8+ ma offre garanzie di ordinamento degli eventi e un overhead inferiore grazie alla API zero-copy reserve/submit. Il perf buffer funziona su kernel più vecchi (4.4+) ma ha spreco di allocazione per CPU e nessuna garanzia di ordinamento.

**BCC vs libbpf**: BCC richiede LLVM sull'host e usa ~80MB per tool. libbpf con CO-RE produce binari standalone da ~9MB. Per un tool didattico, l'API Python di BCC e l'esperienza di sviluppo iterativo vincono. In produzione si passerebbe a libbpf.

**Tracepoint vs kprobe**: I tracepoint hanno un ABI stabile, non si rompono tra le versioni del kernel. I kprobe agganciano funzioni arbitrarie del kernel e possono rompersi quando cambiano le API interne. Questo tool usa esclusivamente tracepoint.

## Modelli di Dati

### RawEvent (struct C / ctypes)

| Campo | Tipo | Byte | Scopo |
|-------|------|-------|---------|
| timestamp_ns | u64 | 8 | Clock monotonico del kernel |
| pid | u32 | 4 | ID del processo |
| ppid | u32 | 4 | ID del processo padre |
| uid | u32 | 4 | ID utente |
| gid | u32 | 4 | ID gruppo |
| event_type | u32 | 4 | Enum: EXECVE=1...INIT_MODULE=14 |
| ret_val | u32 | 4 | Valore di ritorno o flag |
| comm | char[16] | 16 | Nome del processo (TASK_COMM_LEN) |
| filename | char[256] | 256 | Path del file o nome del device |
| addr_v4 | u32 | 4 | Indirizzo IPv4 (network order) |
| port | u16 | 2 | Numero di porta (host order) |
| protocol | u16 | 2 | Address family (AF_INET=2) |
| target_uid | u32 | 4 | UID di destinazione per setuid |
| target_gid | u32 | 4 | GID di destinazione per setgid |
| ptrace_request | u32 | 4 | Tipo di operazione ptrace |
| target_pid | u32 | 4 | PID di destinazione per ptrace |
| **Totale** | | **324** | |

### TracerEvent (dataclass Python)

Estende RawEvent con:
- `timestamp` come `datetime` (convertito dai nanosecondi del kernel)
- `username` risolto dall'UID
- `severity`, `detection`, `detection_id`, `mitre_id` dal motore di detection
- dict `extra` per i dati di arricchimento (parent_comm, ecc.)

## Architettura di Sicurezza

### Modello dei Privilegi
Il tool richiede root (CAP_SYS_ADMIN) per caricare i programmi eBPF. Lo verifica all'avvio con `os.geteuid()` ed esce con un messaggio chiaro se non è root.

### Sicurezza di eBPF
Il verifier del kernel garantisce che i programmi eBPF non possano:
- Accedere a memoria al di fuori del proprio stack o delle mappe BPF
- Eseguire loop illimitati
- Chiamare funzioni arbitrarie del kernel
- Mandare in crash il kernel

### Pulizia
I signal handler (SIGINT, SIGTERM) attivano uno shutdown pulito. Il metodo `TracerLoader.cleanup()` chiama `bpf.cleanup()` su ogni oggetto BPF, che stacca i tracepoint e libera le risorse del kernel. Un blocco `try/finally` in `main.py` garantisce che la pulizia venga eseguita anche in caso di eccezioni.

### Validazione dell'Input
Il tool legge dai ring buffer del kernel (fidati) e da /proc (fidato). Non c'è input utente oltre agli argomenti della CLI, che Typer valida tramite le type annotation.

## Configurazione

Tutta la configurazione vive in `config.py` come costanti a livello di modulo:

| Impostazione | Valore | Scopo |
|---------|-------|---------|
| RING_BUFFER_BYTES | 256KB | Dimensione del ring buffer condiviso |
| CORRELATION_WINDOW_SEC | 10 | Finestra scorrevole per la detection stateful |
| MAX_EVENTS_PER_PID | 64 | Numero massimo di eventi nella deque di correlazione |
| MIN_KERNEL_MAJOR/MINOR | 5.8 | Versione minima del kernel |
| SENSITIVE_READ_PATHS | /etc/shadow, ecc. | File che attivano D002 |
| SHELL_BINARIES | sh, bash, ecc. | Binari considerati "shell" |

## Considerazioni sulle Performance

**Dimensionamento del ring buffer**: 256KB è sufficiente per carichi di lavoro tipici. Con frequenze di syscall estreme (>100K/sec), gli eventi possono essere scartati quando `ringbuf_reserve` restituisce NULL. Aumenta `RING_BUFFER_BYTES` per ambienti ad alto throughput.

**Arricchimento degli eventi**: Leggere `/proc/<pid>/comm` per ogni evento aggiunge latenza. Il flag `--no-enrich` lo disabilita per scenari ad alto volume.

**Caching degli username**: La risoluzione da UID a username usa una cache dict per evitare chiamate ripetute a `pwd.getpwuid()`.

**Motore di detection**: Le regole stateless sono O(1) per evento. Le regole stateful scansionano la deque, che è limitata a 64 voci, quindi nel caso peggiore sono O(64) confronti.

## Decisioni di Progettazione

### Perché Python, non Go o Rust?

BCC ha binding Python maturi e ben documentati. Esistono binding Go (tramite cilium/ebpf) ma usano libbpf, non BCC. Rust ha libbpf-rs. Per un progetto beginner focalizzato sull'insegnamento dei concetti eBPF, Python permette ai lettori di concentrarsi sui concetti di eBPF e di sicurezza anziché sulla complessità del linguaggio.

### Perché un solo ring buffer, non uno per tracer?

Ogni programma BPF ottiene il proprio `BPF_RINGBUF_OUTPUT`, ma tutti usano lo stesso layout di struct e lo stesso callback Python. Questo mantiene semplice la logica del callback. L'alternativa (callback per tracer con struct per tracer) richiederebbe cinque percorsi di parsing separati.

### Perché tracepoint, non raw_tracepoint?

I raw tracepoint forniscono una struct `bpf_raw_tp_args` con meno astrazioni. Sono leggermente più veloci ma più difficili da usare, devi fare il cast manuale degli argomenti. I tracepoint standard forniscono l'accesso `args->` con campi nominati, molto più leggibile per un progetto didattico.

### Perché Typer per la CLI?

Coerenza con gli altri progetti del repository. Typer fornisce generazione automatica dell'help, validazione dei tipi e shell completion con pochissimo codice.

## Estensibilità

### Aggiungere una Nuova Syscall

1. Aggiungi il tipo di evento all'enum `EventType` in `config.py`
2. Aggiungilo a `EVENT_TYPE_CATEGORIES`
3. Scrivi un `TRACEPOINT_PROBE` nel file `.c` appropriato (oppure creane uno nuovo)
4. Se serve una nuova regola di detection, aggiungila a `DETECTION_RULES` e implementala in `detector.py`

### Aggiungere una Nuova Regola di Detection

1. Aggiungi una voce `DetectionRule` a `DETECTION_RULES` in `config.py`
2. Implementa il controllo in `_check_stateless()` o `_check_stateful()` in `detector.py`
3. Aggiungi un test in `test_detector.py`

### Aggiungere un Nuovo Formato di Output

1. Crea una nuova classe renderer in `renderer.py` con un metodo `render(event)`
2. Aggiungi il nome del formato al tipo literal `OutputFormat` in `config.py`
3. Gestiscilo in `create_renderer()`

## Limitazioni

- **IPv6**: Il network tracer esegue il parsing solo di IPv4 (`sockaddr_in`). Il supporto a IPv6 richiederebbe di gestire `sockaddr_in6` e un campo indirizzo a 128 bit.
- **Consapevolezza dei container**: Nessun rilevamento di ID del container o namespace. Aggiungerlo richiederebbe di leggere `/proc/<pid>/cgroup` oppure di usare gli helper BPF per gli ID dei namespace.
- **Cattura degli argomenti**: Per execve viene catturato solo il primo argomento (filename). La cattura completa di argv richiede di leggere l'array di puntatori, cosa complessa in eBPF a causa dei vincoli del verifier.
- **Tracciamento dei file descriptor**: Il tool non traccia le mappature fd-file, quindi non può correlare un fd di `connect()` con una successiva chiamata `dup2()`.
- **Nessuna persistenza**: Gli eventi non vengono memorizzati. Per l'analisi storica, invia l'output JSON a un file o a un sistema di aggregazione dei log.

## Confronto con i Tool di Produzione

| Funzionalità | Questo Tool | Falco | Tetragon | Tracee |
|---------|-----------|-------|----------|--------|
| Backend eBPF | BCC (Python) | libs (C) | libbpf (Go) | libbpf (Go) |
| Copertura syscall | 14 | 50+ | 30+ | 40+ |
| Regole di detection | 10 | 100+ | Basato su policy | 70+ |
| Enforcement | Solo detection | Solo detection | Detection + blocco | Solo detection |
| Consapevolezza dei container | No | Sì | Sì | Sì |
| Uso di memoria | ~80MB | ~50MB | ~30MB | ~60MB |
| Pronto per la produzione | No (didattico) | Sì | Sì | Sì |

Questo tool è una risorsa didattica. Insegna gli stessi fondamenti che alimentano Falco e Tetragon, ma a una scala in cui ogni riga di codice è leggibile e comprensibile.
