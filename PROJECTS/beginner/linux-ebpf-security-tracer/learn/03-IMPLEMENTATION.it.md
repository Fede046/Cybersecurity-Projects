# Implementazione - Walkthrough del codice

## Struttura dei file

```
src/
├── __init__.py
├── main.py          # 140 lines - CLI orchestration
├── config.py        # 180 lines - Constants and rule definitions
├── loader.py        # 130 lines - BCC loading and ring buffer setup
├── processor.py     # 190 lines - Event parsing and enrichment
├── detector.py      # 280 lines - Detection engine
├── renderer.py      # 250 lines - Output formatting
└── ebpf/
    ├── __init__.py
    ├── process_tracer.c   # 70 lines - execve, clone
    ├── file_tracer.c      # 80 lines - openat, unlinkat, renameat2
    ├── network_tracer.c   # 100 lines - connect, accept4, bind, listen
    ├── privilege_tracer.c # 80 lines - setuid, setgid
    └── system_tracer.c    # 80 lines - ptrace, mount, init_module
```

## Costruire i programmi eBPF

### La struct Event

Ogni programma eBPF condivide lo stesso layout di struct. Ecco la definizione da `process_tracer.c`:

```c
struct event {
    u64 timestamp_ns;
    u32 pid;
    u32 ppid;
    u32 uid;
    u32 gid;
    u32 event_type;
    u32 ret_val;
    char comm[TASK_COMM_LEN];
    char filename[FILENAME_LEN];
    u32 addr_v4;
    u16 port;
    u16 protocol;
    u32 target_uid;
    u32 target_gid;
    u32 ptrace_request;
    u32 target_pid;
};
```

Questa struct è duplicata in ogni file `.c` perché BCC compila ogni file in modo indipendente: nel modello di compilazione di BCC non esiste alcun meccanismo di header condiviso. Il lato Python la rispecchia con una `ctypes.Structure` in `processor.py`.

Decisioni chiave sulle dimensioni:
- `comm` è `TASK_COMM_LEN` (16 byte), la lunghezza massima del nome di un processo nel kernel
- `filename` è di 256 byte, abbastanza per la maggior parte dei percorsi senza toccare il limite di 512 byte dello stack
- I campi di rete usano `u32` per IPv4 e `u16` per la porta, in linea con il layout di `sockaddr_in`

### Aggancio ai tracepoint

La macro `TRACEPOINT_PROBE` è zucchero sintattico di BCC. Quando scrivi:

```c
TRACEPOINT_PROBE(syscalls, sys_enter_execve) {
    // args->filename gives you the first argument
}
```

BCC genera il codice di aggancio. La struct `args` viene generata automaticamente dal file di formato del tracepoint in `/sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/format`. Puoi ispezionarla:

```bash
cat /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/format
```

### Uso del ring buffer

Il pattern reserve/submit evita copie di memoria inutili:

```c
BPF_RINGBUF_OUTPUT(events, 1 << 18);

TRACEPOINT_PROBE(syscalls, sys_enter_execve) {
    struct event *e = events.ringbuf_reserve(sizeof(*e));
    if (!e)
        return 0;

    // Fill the struct directly in ring buffer memory
    e->timestamp_ns = bpf_ktime_get_ns();
    e->pid = bpf_get_current_pid_tgid() >> 32;
    // ...

    events.ringbuf_submit(e, 0);
    return 0;
}
```

Se `ringbuf_reserve` restituisce NULL, il buffer è pieno. Il programma restituisce 0 (richiesto dal verifier) e l'evento viene scartato in silenzio. È una scelta di design deliberata: scartare eventi è meglio che bloccare la syscall o mandare tutto in crash.

### Lettura del contesto del processo

Ottenere il PID del genitore del processo corrente richiede di leggere da `task_struct`:

```c
struct task_struct *task =
    (struct task_struct *)bpf_get_current_task();
bpf_probe_read_kernel(
    &e->ppid, sizeof(e->ppid),
    &task->real_parent->tgid
);
```

`bpf_get_current_task()` restituisce il puntatore alla `task_struct` corrente. Non possiamo dereferenziarlo direttamente (il verifier lo rifiuterebbe), quindi usiamo `bpf_probe_read_kernel()` per copiare in sicurezza il tgid del genitore.

### Parsing degli indirizzi di rete

Il network tracer deve estrarre IP e porta da `sockaddr_in`:

```c
static __always_inline int parse_sockaddr(
    struct event *e, const void *uaddr
) {
    struct sockaddr_in sa = {};
    bpf_probe_read_user(&sa, sizeof(sa), uaddr);

    if (sa.sin_family == AF_INET) {
        e->addr_v4 = sa.sin_addr.s_addr;
        e->port = __builtin_bswap16(sa.sin_port);
        e->protocol = AF_INET;
    }
    return 0;
}
```

L'indirizzo è in network byte order (big-endian), quindi usiamo `__builtin_bswap16` per convertire la porta nell'ordine dei byte dell'host. L'indirizzo IPv4 resta in network order e viene convertito in notazione puntata in Python.

## Costruire il loader Python

### Compilazione con BCC

`loader.py` legge ogni file `.c` e lo passa a BCC:

```python
from bcc import BPF

c_text = src_path.read_text()
bpf = BPF(text=c_text)
bpf["events"].open_ring_buffer(self._callback)
```

BCC compila il codice C usando Clang/LLVM a runtime. Se c'è un errore di sintassi nel codice C, fallisce qui con un errore di compilazione. Il bytecode eBPF compilato viene caricato automaticamente nel kernel e agganciato ai tracepoint dichiarati tramite `TRACEPOINT_PROBE`.

### Gestione dei segnali

Uno shutdown pulito è fondamentale. I programmi eBPF restano agganciati al kernel finché non vengono sganciati esplicitamente. Se il processo Python muore senza cleanup, i programmi continuano a girare (sprecando risorse del kernel) finché gli oggetti BPF non vengono rimossi dal garbage collector.

```python
def _handle_stop(signum, frame):
    self._running = False

signal.signal(signal.SIGINT, _handle_stop)
signal.signal(signal.SIGTERM, _handle_stop)

try:
    while self._running:
        for bpf in self._bpf_objects:
            bpf.ring_buffer_poll(timeout=100)
finally:
    self.cleanup()
```

Il timeout di poll di 100ms significa che il tool controlla la richiesta di shutdown ogni 100ms. È un buon equilibrio tra reattività (Ctrl+C funziona rapidamente) e uso della CPU (niente giri a vuoto in un loop stretto).

## Costruire l'event processor

### Mappatura della struct con ctypes

La struct `RawEvent` rispecchia esattamente il layout C:

```python
class RawEvent(ctypes.Structure):
    _fields_ = [
        ("timestamp_ns", ctypes.c_uint64),
        ("pid", ctypes.c_uint32),
        ("ppid", ctypes.c_uint32),
        ("uid", ctypes.c_uint32),
        ("gid", ctypes.c_uint32),
        ("event_type", ctypes.c_uint32),
        ("ret_val", ctypes.c_uint32),
        ("comm", ctypes.c_char * TASK_COMM_LEN),
        ("filename", ctypes.c_char * MAX_FILENAME_LEN),
        ("addr_v4", ctypes.c_uint32),
        ("port", ctypes.c_uint16),
        ("protocol", ctypes.c_uint16),
        ("target_uid", ctypes.c_uint32),
        ("target_gid", ctypes.c_uint32),
        ("ptrace_request", ctypes.c_uint32),
        ("target_pid", ctypes.c_uint32),
    ]
```

L'ordine e i tipi dei campi devono corrispondere esattamente. Una discrepanza significa che il lato Python legge spazzatura. La callback del ring buffer esegue il cast del puntatore grezzo:

```python
raw = ctypes.cast(
    data, ctypes.POINTER(RawEvent)
).contents
```

### Conversione dei timestamp

I timestamp del kernel da `bpf_ktime_get_ns()` sono nanosecondi monotonici dall'avvio, non l'ora di sistema (wall clock). Per convertirli:

```python
def _boot_time_ns():
    for line in Path("/proc/stat").read_text().splitlines():
        if line.startswith("btime"):
            return int(line.split()[1]) * 1_000_000_000
    return 0

_BOOT_NS = _boot_time_ns()

def _ktime_to_datetime(ktime_ns):
    epoch_ns = _BOOT_NS + ktime_ns
    return datetime.fromtimestamp(
        epoch_ns / 1_000_000_000, tz=timezone.utc
    )
```

`btime` in `/proc/stat` fornisce il boot time in secondi epoch. Aggiungi i nanosecondi del kernel per ottenere l'ora di sistema.

### Conversione IPv4

Il kernel memorizza gli indirizzi IPv4 in network byte order (big endian). Per convertirli in notazione puntata:

```python
def _ipv4_to_str(addr):
    if addr == 0:
        return ""
    return ".".join(
        str((addr >> (i * 8)) & 0xFF)
        for i in range(4)
    )
```

Ad esempio, `0x0100007F` diventa `127.0.0.1` (byte 0 = 127, byte 1 = 0, byte 2 = 0, byte 3 = 1).

## Costruire il detection engine

### Regole stateless

Ogni regola stateless confronta un singolo evento con un pattern. L'implementazione è una serie di controlli condizionali in `_check_stateless()`:

```python
if event.event_type == "setuid":
    if event.target_uid == 0 and event.uid != 0:
        rule = DETECTION_RULES["D001"]
        return Detection(
            rule_id=rule.rule_id,
            name=rule.name,
            severity=rule.severity,
            mitre_id=rule.mitre_id,
            description=rule.description,
        )
```

Le regole sono data-driven. `DETECTION_RULES` in `config.py` contiene i metadati (ID, nome, severity, mappatura MITRE). Il detection engine contiene solo la logica di matching.

### Matching dei percorsi dei file

Le detection basate sui file usano il matching per prefisso rispetto a liste di percorsi curate:

```python
SENSITIVE_READ_PATHS = (
    "/etc/shadow",
    "/etc/gshadow",
    "/etc/sudoers",
    "/etc/master.passwd",
)

def _path_matches(filepath, patterns):
    for pattern in patterns:
        if filepath.startswith(pattern):
            return True
    return False
```

Usare `startswith` invece di una corrispondenza esatta intercetta percorsi come `/etc/shadow-` (backup) e `/etc/sudoers.d/custom`.

### Detection delle scritture tramite flag

L'argomento `flags` della syscall `openat` ci dice se il file è aperto in lettura o in scrittura. Il programma eBPF memorizza i flag in `ret_val`:

```python
O_WRONLY = 1
O_RDWR = 2
O_TRUNC = 512

def _is_write_flags(flags):
    return bool(flags & (O_WRONLY | O_RDWR))
```

Questo distingue la lettura di un file cron (normale) dalla scrittura su uno (tentativo di persistence).

### Correlazione stateful

La detection della reverse shell mantiene una deque per ogni PID:

```python
def _check_stateful(self, event):
    if event.event_type != "execve":
        return None
    if event.comm not in SHELL_BINARIES:
        return None

    hist = self._get_history(event.pid)
    has_connect = any(
        e.event_type == "connect" for e in hist
    )

    if not has_connect:
        ppid_hist = self._history.get(event.ppid)
        if ppid_hist:
            has_connect = any(
                e.event_type == "connect"
                for e in ppid_hist
            )

    if has_connect:
        return Detection(...)
```

Il controllo sul PID del genitore gestisce il caso in cui un processo esegue `connect()` e poi `fork()` + `execve()`. La shell gira come processo figlio, quindi l'evento connect si trova nella cronologia del genitore.

## Costruire il renderer dell'output

### Modalità live con Rich

```python
class LiveRenderer:
    def render(self, event):
        ts = event.timestamp.strftime("%H:%M:%S")
        color = SEVERITY_COLORS.get(
            event.severity, "white"
        )
        sev = Text(f"{event.severity:8s}", style=color)
        # ... build line with Rich Text objects
        self._console.print(line)
```

La classe `Text` di Rich supporta lo stile per singolo segmento. Gli eventi CRITICAL vengono mostrati in rosso grassetto, MEDIUM in giallo, LOW in ciano. I nomi delle detection appaiono in magenta grassetto.

### Modalità JSON

```python
class JsonRenderer:
    def render(self, event):
        d = _event_to_dict(event)
        self._stream.write(json.dumps(d) + "\n")
        self._stream.flush()
```

Un oggetto JSON per riga (formato JSONL). Il `flush()` dopo ogni evento garantisce un output in tempo reale quando si fa il piping verso altri tool.

## Strategia di testing

### Unit test (non serve root)

I test usano una fixture `make_event` che crea istanze di `TracerEvent` senza eBPF:

```python
@pytest.fixture()
def make_event():
    def _make(event_type="execve", pid=1000, ...):
        return TracerEvent(
            timestamp=datetime.now(tz=timezone.utc),
            event_type=event_type,
            pid=pid,
            ...
        )
    return _make
```

Questo ci permette di testare regole di detection, filtraggio e rendering interamente in Python:

```python
def test_setuid_zero_by_nonroot(self, make_event):
    engine = DetectionEngine()
    event = make_event(
        event_type="setuid", uid=1000, target_uid=0,
    )
    result = engine.evaluate(event)
    assert result.detection == "Privilege Escalation"
    assert result.severity == "CRITICAL"
```

### Cosa non viene testato

I test di integrazione (caricamento dei programmi eBPF, tracciamento di syscall reali) richiedono root e un kernel compatibile. Non possono girare in CI. Il marker `@pytest.mark.integration` li separa, e `just test` li esclude per impostazione predefinita.

## Errori comuni

### Errore: ordine dei campi in ctypes

Se l'ordine di `_fields_` in `RawEvent` non corrisponde alla struct C, ogni campo successivo alla discrepanza legge dati sbagliati. Il sintomo sono valori spazzatura in campi apparentemente casuali. Verifica sempre che l'ordine dei campi corrisponda esattamente.

### Errore: decodifica delle stringhe

Le stringhe del kernel sono array di byte terminati da null. Se dimentichi di fare lo split su `\x00`, in Python otterrai byte spazzatura in coda:

```python
# Wrong: raw.comm.decode() might include garbage
# Right: split on null first
raw.split(b"\x00", 1)[0].decode("utf-8", errors="replace")
```

### Errore: network byte order

Gli indirizzi IPv4 e le porte arrivano dal kernel in network byte order (big-endian). Le porte richiedono `__builtin_bswap16` in C o uno scambio manuale dei byte in Python. Gli indirizzi IPv4 possono essere scomposti byte per byte.

### Errore: stack overflow in BPF

Lo stack eBPF è di 512 byte. La sola `struct event` pesa ~324 byte. Se aggiungi variabili locali, puoi superare il limite. Il pattern reserve/submit evita il problema scrivendo direttamente nella memoria del ring buffer invece di usare struct allocate sullo stack.

## Organizzazione del codice

### Perché nessun header C condiviso?

BCC compila ogni file `.c` in modo indipendente. Non esiste un meccanismo `#include "common.h"`. Ogni file definisce la propria copia di `struct event`. È ridondante ma corrisponde a come BCC funziona in pratica.

### Perché config.py invece di YAML/JSON?

Le costanti Python sono controllate nei tipi da mypy. Sono importabili. Non hanno bisogno di un parser. Per un tool di queste dimensioni, un formato di file di configurazione aggiunge complessità senza benefici.

### Perché un dataclass invece di Pydantic?

`TracerEvent` è un semplice contenitore di dati creato migliaia di volte al secondo. L'overhead di validazione di Pydantic non è giustificato quando la sorgente dei dati è un ring buffer del kernel fidato. Il `dataclass` standard è più leggero e più veloce.

## Estendere il codice

### Aggiungere un nuovo syscall tracer

Per tracciare `mprotect` (cambi di protezione della memoria, utile per rilevare attacchi di JIT spray):

1. Aggiungi `MPROTECT = 15` a `EventType` in `config.py`
2. Aggiungi la mappatura della categoria: `EventType.MPROTECT: "system"`
3. Aggiungi un `TRACEPOINT_PROBE` a `system_tracer.c`:

```c
TRACEPOINT_PROBE(syscalls, sys_enter_mprotect) {
    struct event *e = events.ringbuf_reserve(sizeof(*e));
    if (!e) return 0;
    fill_base(e, 15);
    e->ret_val = args->prot;
    events.ringbuf_submit(e, 0);
    return 0;
}
```

4. Aggiungi una regola di detection se `prot` include `PROT_EXEC` su una regione precedentemente non eseguibile.

## Dipendenze

| Pacchetto | Scopo | Perché questo |
|---------|---------|-------------|
| typer | Framework CLI | Coerente con il repo, generazione automatica dell'help |
| rich | Formattazione del terminale | Output a colori, tabelle, stile del testo |
| bcc | Compilazione e caricamento eBPF | Unico framework eBPF maturo per Python |
| pytest | Testing | Framework di testing standard per Python |
| ruff | Linting | Veloce, completo, sostituisce flake8 |
| mypy | Type checking | Analisi statica per Python |
| yapf | Formattazione | Standard del repo |

BCC è un pacchetto di sistema, non installabile con pip. Installalo tramite `apt install python3-bpfcc` (Debian) o `dnf install python3-bcc` (Fedora).

## Build e deploy

```bash
# Full setup
./install.sh

# Development
uv sync                    # install Python deps
just lint                  # ruff + mypy
just format                # yapf formatting
just test                  # unit tests (no root)
sudo uv run ebpf-tracer    # run the tool
```

Il tool gira in place, non c'è alcuno step di compilazione per il codice Python. I programmi C eBPF vengono compilati da BCC a runtime all'avvio del tool.
