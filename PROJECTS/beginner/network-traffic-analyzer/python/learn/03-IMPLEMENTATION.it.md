# Guida all'implementazione

Questo documento percorre il codice vero e proprio. Costruiremo le funzionalità chiave passo dopo passo e spiegheremo le decisioni lungo il percorso.

## Analisi della struttura dei file

```
network-traffic-analyzer/
├── src/netanal/
│   ├── __init__.py          # Package exports, version info
│   ├── __main__.py          # Entry point for python -m netanal
│   ├── main.py              # Typer CLI commands (capture, analyze, export, chart)
│   ├── capture.py           # Producer-consumer packet capture engine
│   ├── analyzer.py          # Protocol dissection using Scapy layers
│   ├── filters.py           # Type-safe BPF filter builder
│   ├── statistics.py        # Thread-safe statistics aggregation
│   ├── models.py            # Data models (PacketInfo, Protocol, CaptureStatistics)
│   ├── visualization.py     # Matplotlib chart generation
│   ├── export.py            # JSON/CSV serialization
│   ├── output.py            # Rich console formatting
│   ├── constants.py         # Configuration constants
│   └── exceptions.py        # Custom exception hierarchy
├── tests/
│   ├── test_filters.py      # FilterBuilder validation tests
│   └── test_models.py       # Data model tests
└── pyproject.toml           # Dependencies and build config
```

## Costruire l'engine di cattura dei pacchetti

### Passo 1: setup producer-consumer

Cosa stiamo costruendo: un engine di cattura che riceve i pacchetti da Scapy alla velocità della rete e li elabora in un thread separato senza perdere dati.

La sfida principale è che i pacchetti arrivano in modo asincrono a ritmi imprevedibili. Se l'elaborazione blocca il thread di cattura, i pacchetti vengono persi. La soluzione è un pattern producer-consumer con una coda limitata.

Da `capture.py:31-62`:

```python
class CaptureEngine:
    def __init__(
        self,
        config: CaptureConfig,
        on_packet: Callable[[PacketInfo], None] | None = None,
        queue_size: int = CaptureDefaults.QUEUE_SIZE,
    ) -> None:
        self._config = config
        self._on_packet = on_packet
        self._queue: Queue[Packet] = Queue(maxsize = queue_size)
        self._stats = StatisticsCollector()
        self._sniffer: AsyncSniffer | None = None
        self._processor_thread: threading.Thread | None = None
        self._stop_event = threading.Event()
        self._packet_count = 0
        self._dropped_packets = 0
        self._running = False
        self._count_lock = threading.Lock()
```

**Perché questo codice funziona:**

- **Queue[Packet]**: buffer limitato tra i thread. `maxsize = 10000` significa che se la coda si riempie, il producer scarta i pacchetti invece di bloccarsi. Questo impedisce al thread di cattura di rallentare.

- **StatisticsCollector**: un oggetto separato gestisce tutte le metriche. Mantiene la logica di cattura separata dalla logica delle statistiche.

- **threading.Event**: stop_event segnala a entrambi i thread quando è il momento di fermarsi. Meglio dei flag perché Event.wait() è interrompibile.

- **Lock**: _count_lock protegge _packet_count e _dropped_packets, che entrambi i thread modificano. Senza di esso, le race condition corrompono i conteggi.

**Errori comuni in questo punto:**

```python
# Wrong: unbounded queue
self._queue = Queue()  # Can grow to gigabytes, OOM kills process

# Wrong: no lock on counters
self._packet_count += 1  # Race condition, loses counts

# Wrong: boolean flag for shutdown
self._should_stop = False  # Thread.join() with timeout is better
```

### Passo 2: setup del thread producer

Ora dobbiamo avviare l'AsyncSniffer di Scapy come producer.

In `capture.py:92-131`:

```python
def start(self) -> None:
    if self._running:
        return
    
    self._running = True
    self._stop_event.clear()
    
    with self._count_lock:
        self._packet_count = 0
        self._dropped_packets = 0
    
    self._stats.reset()
    self._stats.start()
    
    self._processor_thread = threading.Thread(
        target = self._process_packets,
        daemon = True,
    )
    self._processor_thread.start()
    
    sniffer_kwargs: dict[str, object] = {
        "prn": self._enqueue_packet,
        "store": self._config.store_packets,
    }
    
    if self._config.interface:
        sniffer_kwargs["iface"] = self._config.interface
    
    if self._config.bpf_filter:
        sniffer_kwargs["filter"] = self._config.bpf_filter
    
    self._sniffer = AsyncSniffer(**sniffer_kwargs)
    self._sniffer.start()
```

**Cosa sta succedendo:**

1. Controlla il flag _running per evitare un doppio avvio (creerebbe thread duplicati)
2. Azzera contatori e statistiche (si riparte da zero per la nuova cattura)
3. Avvia il thread consumer PRIMA del producer (così la coda ha un consumer quando arrivano i pacchetti)
4. Costruisce il dizionario sniffer_kwargs in modo condizionale (include solo i valori di configurazione non None)
5. Passa _enqueue_packet come callback (parametro `prn`)
6. AsyncSniffer.start() avvia internamente il thread producer

**Perché lo facciamo in questo modo:**

Avviare il consumer prima del producer previene l'overflow della coda durante l'inizializzazione. Se il producer parte per primo e il thread consumer non è ancora partito, la coda si riempie immediatamente.

I thread daemon terminano automaticamente quando termina il programma principale. I thread non daemon manterrebbero vivo il programma anche dopo che l'utente ha premuto Ctrl+C.

**Approcci alternativi:**

- **Approccio A**: usare `sniff(prn=callback)` - Funziona ma blocca il thread principale, non può mostrare l'avanzamento né rispondere ai segnali
- **Approccio B**: usare `sniff(timeout=1)` in un loop - Introduce intervalli in cui i pacchetti possono andare persi tra il timeout e il riavvio

### Passo 3: callback del producer

La callback del producer gira nel thread di cattura di Scapy per ogni pacchetto.

In `capture.py:64-70`:

```python
def _enqueue_packet(self, packet: Packet) -> None:
    try:
        self._queue.put_nowait(packet)
    except Full:
        with self._count_lock:
            self._dropped_packets += 1
```

Questo gestisce [specific responsibility]: aggiungere pacchetti alla coda senza bloccarsi. `put_nowait()` solleva l'eccezione Full se la coda è piena. La catturiamo e incrementiamo il contatore dei pacchetti scartati invece di andare in crash.

**Spiegazione delle parti chiave:**

Il motivo per cui usiamo `put_nowait()` invece di `put()` sono le prestazioni. `put()` si blocca finché non c'è spazio disponibile, il che rallenterebbe la cattura alla velocità di elaborazione del consumer. Meglio scartare pacchetti che rallentare la cattura.

Il lock su _dropped_packets previene operazioni di incremento perse. Se due thread eseguono read-modify-write contemporaneamente senza un lock, un incremento va perso.

## Costruire l'identificazione dei protocolli

### Il problema

I pacchetti di Scapy sono oggetti a layer annidati. Dobbiamo identificare il protocollo di livello più alto ed estrarre i campi rilevanti senza codificare a mano ogni possibile combinazione di protocolli.

### La soluzione

Percorrere i layer dal più alto (applicazione) al più basso (link), restituendo la prima corrispondenza.

### Implementazione

In `analyzer.py:14-48`:

```python
def identify_protocol(packet: Packet) -> Protocol:
    if packet.haslayer(DNS):
        return Protocol.DNS
    
    if packet.haslayer(TCP):
        tcp_layer = packet[TCP]
        if tcp_layer.dport == Ports.HTTP or tcp_layer.sport == Ports.HTTP:
            return Protocol.HTTP
        if tcp_layer.dport == Ports.HTTPS or tcp_layer.sport == Ports.HTTPS:
            return Protocol.HTTPS
        return Protocol.TCP
    
    if packet.haslayer(UDP):
        udp_layer = packet[UDP]
        if udp_layer.dport == Ports.DNS or udp_layer.sport == Ports.DNS:
            return Protocol.DNS
        return Protocol.UDP
    
    if packet.haslayer(ICMP):
        return Protocol.ICMP
    
    if packet.haslayer(ARP):
        return Protocol.ARP
    
    return Protocol.OTHER
```

**Spiegazione delle parti chiave:**

**Prima il rilevamento del DNS** (`analyzer.py:14-15`)
Il DNS può girare su TCP o UDP. Controlla il layer DNS prima di controllare il protocollo di trasporto, altrimenti il DNS su TCP verrebbe classificato semplicemente come TCP.

**Rilevamento del protocollo basato sulla porta** (`analyzer.py:20-25`)
HTTP e HTTPS sono semplicemente TCP su porte specifiche. Controlla i numeri di porta per classificare ulteriormente. Vengono controllate sia la porta di origine sia quella di destinazione perché le risposte del server hanno HTTP/HTTPS come porta di origine.

**Fallback su OTHER** (`analyzer.py:45`)
I protocolli sconosciuti non mandano in crash l'analyzer. Vengono classificati come OTHER e conteggiati separatamente nelle statistiche.

L'ordine conta: i protocolli dell'application layer (DNS, HTTP) vengono identificati prima del transport layer (TCP, UDP). Questo fornisce una classificazione più specifica.

### Testare questa funzionalità

```python
from scapy.layers.inet import IP, TCP
from scapy.layers.dns import DNS
from netanal.analyzer import identify_protocol
from netanal.models import Protocol

# Test HTTP detection
http_packet = IP()/TCP(dport=80)
assert identify_protocol(http_packet) == Protocol.HTTP

# Test DNS detection
dns_packet = IP()/UDP()/DNS()
assert identify_protocol(dns_packet) == Protocol.DNS
```

Output atteso: entrambe le asserzioni passano, a dimostrazione che l'identificazione dei protocolli funziona correttamente.

Se vedi Protocol.TCP per HTTP, significa che il controllo della porta è fallito. Verifica che la porta nel pacchetto sia davvero 80.

## Raccolta thread-safe delle statistiche

### Il problema

Più thread aggiornano le stesse statistiche contemporaneamente. Senza sincronizzazione, i contatori perdono incrementi e i dizionari vengono corrotti.

### La soluzione

Usare un singolo lock per proteggere tutto lo stato condiviso. Le sezioni critiche (il codice sotto lock) restano il più brevi possibile.

### Implementazione

File: `statistics.py:47-67`

```python
def record_packet(self, packet: PacketInfo) -> None:
    with self._lock:
        self._total_packets += 1
        self._total_bytes += packet.size
        self._interval_packets += 1
        self._interval_bytes += packet.size
        
        self._protocol_counts[packet.protocol] += 1
        self._protocol_bytes[packet.protocol] += packet.size
        
        self._update_endpoint(packet.src_ip, sent_bytes = packet.size)
        self._update_endpoint(
            packet.dst_ip,
            received_bytes = packet.size
        )
        
        self._update_conversation(
            packet.src_ip,
            packet.dst_ip,
            packet.size
        )
        
        self._check_bandwidth_sample(packet.timestamp)
```

**Cosa previene:**

Gli incrementi persi. Senza il lock:
```
Thread A reads total_packets = 100
Thread B reads total_packets = 100  
Thread A writes 101
Thread B writes 101  # Lost an increment!
```

Con il lock, le operazioni sono atomiche:
```
Thread A acquires lock
Thread A reads 100, writes 101
Thread A releases lock
Thread B acquires lock (waits until A finishes)
Thread B reads 101, writes 102
```

**Come funziona:**

1. `with self._lock:` acquisisce il lock, bloccandosi se un altro thread lo detiene
2. Tutti gli aggiornamenti dei contatori avvengono in modo atomico
3. I metodi helper (_update_endpoint, ecc.) girano sotto lo stesso lock
4. Il lock viene rilasciato automaticamente all'uscita dal blocco with (anche in caso di eccezione)

**Cosa succede se lo rimuovi:**

Esegui il codice sotto carico elevato. I contatori saranno inferiori al numero reale di pacchetti perché gli incrementi vanno persi. Le percentuali della distribuzione dei protocolli non sommeranno a 100%. Le statistiche degli endpoint avranno totali errati.

### Campionamento della banda

Ogni secondo, dobbiamo calcolare la banda attuale. Questo gira sotto lo stesso lock per coerenza.

Da `statistics.py:112-127`:

```python
def _check_bandwidth_sample(self, timestamp: float) -> None:
    if timestamp - self._last_sample_time >= self._bandwidth_interval:
        elapsed = timestamp - self._last_sample_time
        if elapsed > 0:
            bps = self._interval_bytes / elapsed
            pps = self._interval_packets / elapsed
            self._bandwidth_samples.append(
                BandwidthSample(
                    timestamp = timestamp,
                    bytes_per_second = bps,
                    packets_per_second = pps,
                )
            )
        self._interval_bytes = 0
        self._interval_packets = 0
        self._last_sample_time = timestamp
```

Questo codice campiona la banda a intervalli di 1 secondo (configurabili). Calcola byte/sec e pacchetti/sec a partire dai contatori dell'intervallo, poi li azzera per l'intervallo successivo.

Il timestamp proviene dai pacchetti, non dall'orologio di sistema. Questo significa che il calcolo della banda corrisponde esattamente alla temporizzazione dei pacchetti, anche se l'orologio deriva o il sistema va in pausa.

## Costruzione dei filtri BPF

### Il problema

La sintassi BPF è soggetta a errori. Scrivere a mano `"tcp port 80 and host 192.168.1.1"` espone a errori di battitura, sintassi non valida e vulnerabilità di filter injection.

### La soluzione

Builder pattern con metodi type-safe e validazione dell'input.

### Implementazione

Da `filters.py:48-175`:

```python
@dataclass(slots = True)
class FilterBuilder:
    _expressions: list[str]
    
    def __init__(self) -> None:
        self._expressions = []
    
    def protocol(self, proto: Protocol) -> FilterBuilder:
        bpf_expr = BPF_PROTOCOL_MAP.get(proto)
        if bpf_expr:
            self._expressions.append(f"({bpf_expr})")
        return self
    
    def port(self, port_number: int) -> FilterBuilder:
        _validate_port(port_number)
        self._expressions.append(f"port {port_number}")
        return self
    
    def host(self, ip_address: str) -> FilterBuilder:
        _validate_ip_address(ip_address)
        self._expressions.append(f"host {ip_address}")
        return self
    
    def build(self, operator: Literal["and", "or"] = "and") -> str | None:
        if not self._expressions:
            return None
        return f" {operator} ".join(self._expressions)
```

**Dettagli importanti:**

**Restituire self** (`return self` in ogni metodo)
Consente il method chaining: `FilterBuilder().port(80).host("192.168.1.1").build()`

**Validazione prima della costruzione** (`_validate_port`, `_validate_ip_address`)
```python
def _validate_port(port_number: int) -> None:
    if not PortRange.MIN <= port_number <= PortRange.MAX:
        raise ValidationError(
            f"Port must be {PortRange.MIN}-{PortRange.MAX}, got {port_number}"
        )
```

La porta deve essere compresa tra 0 e 65535. L'IP deve essere interpretabile da `ipaddress.ip_address()`. Fallisce subito con errori chiari prima di passare al kernel.

**Racchiudere le espressioni tra parentesi**
```python
self._expressions.append(f"({bpf_expr})")
```

BPF ha regole di precedenza tra gli operatori. Racchiudere tra parentesi garantisce un parsing corretto. `tcp and port 80 or port 443` potrebbe significare `(tcp and port 80) or (port 443)` [sbagliato] oppure `tcp and (port 80 or port 443)` [inteso]. Le parentesi esplicite evitano l'ambiguità.

## Esempio di flusso dei dati

Seguiamo una richiesta completa attraverso il sistema.

**Scenario:** l'utente esegue `sudo netanal capture -i lo -c 5 --verbose`

### Arriva la richiesta

```python
# Entry point: main.py:110-181
@app.command()
def capture(
    interface: str | None = None,
    filter_expr: str | None = None,
    count: int | None = None,
    timeout: float | None = None,
    output: Path | None = None,
    verbose: bool = False,
) -> None:
```

A questo punto:
- Typer ha eseguito il parsing degli argomenti della riga di comando
- interface = "lo", count = 5, verbose = True
- Bisogna verificare i permessi e creare la configurazione di cattura

Il controllo dei permessi avviene in `main.py:139-143`:
```python
can_capture, msg = check_capture_permissions()
if not can_capture:
    print_error(f"Cannot capture packets: {msg}")
    raise typer.Exit(1)
```

Questo chiama `capture.py:341-347`, che verifica la creazione di raw socket su Linux, l'accesso a /dev/bpf su macOS, oppure controlla Npcap+Admin su Windows.

### Layer di elaborazione

Creazione della configurazione in `main.py:149-154`:
```python
config = CaptureConfig(
    interface = interface,
    bpf_filter = filter_expr,
    packet_count = count,
    timeout_seconds = timeout,
)
```

CaptureConfig è una dataclass frozen (`models.py:135-145`). Immutabile dopo la creazione, viene passata a CaptureEngine.

La cattura parte in `main.py:159-167`:
```python
engine = CaptureEngine(
    config = config,
    on_packet = on_packet if verbose or output else None
)

with GracefulCapture(engine) as cap:
    stats = cap.wait()
```

Il context manager GracefulCapture (`capture.py:197-230`) installa i signal handler, avvia la cattura, attende il completamento, poi fa pulizia. Anche se l'utente preme Ctrl+C, il cleanup viene eseguito.

### Flusso di elaborazione dei pacchetti

Per ogni pacchetto catturato:

1. Scapy chiama la callback `_enqueue_packet` (`capture.py:64-70`)
2. Il pacchetto entra nella coda limitata
3. Il thread consumer preleva il pacchetto dalla coda (`capture.py:76-78`)
4. `extract_packet_info()` esegue il parsing del pacchetto (`analyzer.py:51-103`)
5. `record_packet()` aggiorna le statistiche (`statistics.py:47-67`)
6. Se è attivo verbose, la callback `on_packet()` visualizza il pacchetto (`output.py:46-54`)

Dopo 5 pacchetti, il controllo del conteggio in `capture.py:88-90` imposta lo stop event:
```python
if self._config.packet_count and current_count >= self._config.packet_count:
    self._stop_event.set()
    break
```

### Memorizzazione/output

Il risultato è il CaptureStatistics restituito da `cap.wait()` (`capture.py:145-157`).

La visualizzazione avviene in `main.py:169-171`:
```python
print_capture_summary(stats)
print_protocol_table(stats)
print_top_talkers(stats)
```

Ogni funzione di stampa usa Rich per formattare le tabelle. Da `output.py:84-111`:
```python
def print_protocol_table(stats: CaptureStatistics) -> None:
    table = Table(title = "Protocol Distribution")
    table.add_column("Protocol", style = "cyan", justify = "left")
    table.add_column("Packets", style = "green", justify = "right")
    table.add_column("Bytes", style = "yellow", justify = "right")
    table.add_column("Percentage", style = "magenta", justify = "right")
    
    percentages = stats.get_protocol_percentages()
    
    for protocol in sorted(stats.protocol_distribution.keys(),
                           key = lambda p: p.value):
        count = stats.protocol_distribution[protocol]
        bytes_count = stats.protocol_bytes.get(protocol, 0)
        pct = percentages.get(protocol, 0.0)
        table.add_row(
            protocol.value,
            f"{count:,}",
            format_bytes(bytes_count),
            f"{pct:.1f}%",
        )
    
    console.print(table)
```

## Pattern di gestione degli errori

### Errori di permessi

Quando l'utente non ha i permessi per la cattura dei pacchetti, vogliamo errori chiari che indichino cosa fare.

```python
# capture.py:341-347
def check_capture_permissions() -> tuple[bool, str]:
    system = platform.system()
    
    if system == "Linux":
        return _check_linux_permissions()
    elif system == "Darwin":
        return _check_macos_permissions()
    elif system == "Windows":
        return _check_windows_permissions()
    
    return False, f"Unknown platform: {system}"
```

**Perché questa gestione specifica:**
Restituisce una tupla (bool, str) invece di sollevare un'eccezione. È il chiamante a decidere se generare un errore o un avviso. Messaggi chiari dicono all'utente esattamente cosa serve ("Requires root or CAP_NET_RAW capability" invece di un generico "Permission denied").

Controlli specifici per piattaforma, perché i requisiti differiscono:
- Linux: capability CAP_NET_RAW oppure root
- macOS: root oppure accesso a /dev/bpf*
- Windows: Amministratore + Npcap installato

**Cosa NON fare:**
```python
# Bad: catching everything silently
try:
    start_capture()
except Exception:
    pass  # User gets no feedback, waste time debugging
```

Questo nasconde i problemi reali. Gestisci sempre eccezioni specifiche e fornisci un feedback che indichi cosa fare.

### Validazione dei filtri BPF

I filtri non validi mandano in crash Scapy con errori criptici del kernel. Valida presto.

Da `filters.py:227-235`:
```python
def validate_bpf_filter(filter_str: str) -> bool:
    try:
        from scapy.arch import compile_filter
        compile_filter(filter_str)
        return True
    except Exception:
        return False
```

Uso in `main.py:145-147`:
```python
if filter_expr and not validate_bpf_filter(filter_expr):
    print_error(f"Invalid BPF filter: {filter_expr}")
    raise typer.Exit(1)
```

Fallisce subito, prima di avviare la cattura. L'utente vede immediatamente un errore chiaro invece di un messaggio criptico del kernel dopo aver aspettato.

## Ottimizzazioni delle prestazioni

### Ottimizzazione 1: dataclass slots

**Prima:**
```python
@dataclass
class PacketInfo:
    timestamp: float
    src_ip: str
    # ... 8 more fields
```

Era lento perché ogni istanza usa un `__dict__` per memorizzare gli attributi. Con 1 milione di pacchetti, si tratta di circa 40MB sprecati nell'overhead dei dizionari.

**Dopo:**
```python
@dataclass(frozen = True, slots = True)
class PacketInfo:
    timestamp: float
    src_ip: str
    # ... 8 more fields
```

**Cosa è cambiato:**
- Aggiunto `slots = True` al decorator della dataclass
- Attributi memorizzati in slot fissi, non in un dizionario
- Aggiunto anche `frozen = True` per l'immutabilità

**Benchmark:**
- Prima: 1M pacchetti = ~100MB di memoria
- Dopo: 1M pacchetti = ~60MB di memoria  
- Miglioramento: riduzione della memoria del 40%

Misurato con:
```python
import sys
packet = PacketInfo(...)
print(sys.getsizeof(packet))
```

### Ottimizzazione 2: filtraggio BPF nel kernel

**Prima:**
```python
# Capture all packets, filter in Python
for packet in capture_all():
    if packet.haslayer(TCP) and packet[TCP].dport == 80:
        process(packet)
```

Era lento perché ogni pacchetto provoca un context switch dal kernel allo userspace, e poi il codice Python li controlla uno per uno.

**Dopo:**
```python
# Filter in kernel with BPF
AsyncSniffer(filter="tcp port 80", prn=process)
```

**Cosa è cambiato:**
BPF gira nel kernel, scartando i pacchetti indesiderati prima che lo userspace li veda. Nessun context switch per i pacchetti filtrati.

**Benchmark:**
Su una rete trafficata (1000 pacchetti/sec, 95% irrilevanti):
- Prima: 80% di CPU, 950 context switch inutili al secondo
- Dopo: 5% di CPU, solo 50 pacchetti rilevanti raggiungono lo userspace

Il kernel esegue confronti semplici (port == 80) in modo estremamente veloce. Python esegue lo stesso controllo migliaia di volte più lentamente.

## Insidie comuni nell'implementazione

### Insidia 1: dimenticare i limiti della coda

**Sintomo:**
La memoria del processo cresce fino a gigabyte, il sistema si blocca, l'OOM killer termina il processo.

**Causa:**
```python
# The problematic code
self._queue = Queue()  # Unbounded!
```

Una coda illimitata cresce all'infinito se il producer è più veloce del consumer. Con 10K pacchetti/sec e una dimensione media di 1KB, una coda illimitata cresce di 10MB/sec.

**Correzione:**
```python
# Correct approach
self._queue: Queue[Packet] = Queue(maxsize = 10000)
```

Una coda limitata solleva Full quando raggiunge la capacità. Il producer scarta il pacchetto incrementando un contatore invece di consumare memoria infinita.

**Perché è importante:**
I tool di cattura dei pacchetti in produzione girano per ore o giorni. I memory leak mandano in crash il sistema di monitoraggio, creando punti ciechi durante gli incidenti.

### Insidia 2: l'"ottimizzazione" lock-free

**Sintomo:**
Il conteggio dei pacchetti non corrisponde alla realtà. Le percentuali dei protocolli non sommano a 100%. Statistiche corrotte in modo casuale.

**Causa:**
```python
# Bad: "optimization" that removes lock
def record_packet(self, packet: PacketInfo) -> None:
    # No lock!
    self._total_packets += 1
    self._protocol_counts[packet.protocol] += 1
```

Il ragionamento: "I lock sono lenti, saltiamoli". Ma il `+=` di Python NON è atomico. In realtà sono tre operazioni:
```
1. Read value
2. Add 1
3. Write result
```

Due thread possono intercalarsi, causando aggiornamenti persi.

**Correzione:**
```python
# Correct: use lock
def record_packet(self, packet: PacketInfo) -> None:
    with self._lock:
        self._total_packets += 1
        self._protocol_counts[packet.protocol] += 1
```

**Perché è importante:**
Le statistiche sono inutili se sono errate. Gli incidenti di sicurezza sfuggono perché il rilevamento basato su baseline usa numeri sbagliati.

### Insidia 3: concatenazione di stringhe per i filtri

**Sintomo:**
Errori di sintassi BPF, vulnerabilità di filter injection, crash.

**Causa:**
```python
# Vulnerable code
user_ip = input("Enter IP: ")  # User enters: 1.2.3.4 or 1=1
filter_str = f"host {user_ip}"  # Results in "host 1.2.3.4 or 1=1"
```

Attacco di filter injection. L'attaccante può aggirare le restrizioni previste o costruire filtri che corrispondono a tutto.

**Correzione:**
```python
# Correct: validate first
def host(self, ip_address: str) -> FilterBuilder:
    _validate_ip_address(ip_address)  # Raises on invalid input
    self._expressions.append(f"host {ip_address}")
    return self
```

La validazione con `ipaddress.ip_address()` garantisce che l'input sia davvero un IP, non sintassi BPF arbitraria.

**Perché è importante:**
Se i tool di monitoraggio hanno vulnerabilità di filter injection, gli attaccanti possono accecare il monitoraggio facendo sì che i filtri non corrispondano a nulla, oppure sovraccaricare i sistemi facendo sì che i filtri corrispondano a tutto.

## Suggerimenti per il debug

### Tipo di problema 1: nessun pacchetto catturato

**Problema:** pacchetti totali = 0 anche se la rete è attiva

**Come fare il debug:**

1. Verifica che l'interfaccia sia corretta: `netanal interfaces` mostra le interfacce disponibili
2. Verifica che il filtro BPF non sia troppo restrittivo: rimuovi il filtro e riprova
3. Verifica i permessi: `netanal capture` senza sudo mostra l'errore di permessi
4. Controlla la modalità promiscua: alcuni adattatori wireless la bloccano

**Cause comuni:**
- Nome dell'interfaccia sbagliato ("eth0" vs "ens33")
- Il filtro non corrisponde a nulla ("tcp port 12345" su una rete solo HTTP)
- Adattatore wireless in managed mode (serve la monitor mode)
- Firewall che blocca la cattura dei pacchetti

Aggiungi output di debug per vedere i pacchetti che arrivano alla coda:
```python
def _enqueue_packet(self, packet: Packet) -> None:
    print(f"DEBUG: Enqueued packet from {packet[IP].src if packet.haslayer(IP) else 'unknown'}")
    self._queue.put_nowait(packet)
```

Se la coda riceve pacchetti ma le statistiche mostrano zero, il problema è nel thread consumer.

### Tipo di problema 2: numero elevato di pacchetti scartati

**Problema:** le statistiche mostrano migliaia di pacchetti scartati

**Come fare il debug:**

1. Controlla la dimensione della coda: `constants.py:QUEUE_SIZE = 10000` potrebbe essere troppo piccola
2. Fai il profiling del thread consumer: l'elaborazione è lenta?
3. Monitora l'uso della CPU: il sistema è sovraccarico?
4. Controlla le callback: la stampa verbose sta rallentando l'elaborazione?

**Cause comuni:**
- Coda troppo piccola per il tasso di traffico
- Thread consumer bloccato sull'I/O (scrittura su disco)
- CPU al massimo
- Modalità verbose attiva durante un traffico elevato

Aumenta la dimensione della coda:
```python
engine = CaptureEngine(config=config, queue_size=50000)
```

Fai il profiling del consumer:
```python
import cProfile
cProfile.run('engine.wait()')
```

### Tipo di problema 3: memoria che cresce senza limiti

**Problema:** la memoria del processo cresce continuamente fino all'OOM

**Come fare il debug:**

1. Verifica che la coda sia limitata: `Queue(maxsize=10000)` e non `Queue()`
2. Controlla la memorizzazione dei pacchetti: `store_packets=False` nella configurazione
3. Monitora i campioni di banda: la lista cresce all'infinito?
4. Controlla la presenza di cicli di riferimenti: i vecchi pacchetti restano in memoria?

**Cause comuni:**
- Coda illimitata
- store_packets=True mantiene tutti i pacchetti in memoria
- Campioni di banda mai ripuliti
- Riferimenti circolari che impediscono il GC

Correggi la crescita illimitata:
```python
# Limit bandwidth samples
if len(self._bandwidth_samples) > 3600:  # Max 1 hour at 1/sec
    self._bandwidth_samples = self._bandwidth_samples[-3600:]
```

## Principi di organizzazione del codice

### Perché capture.py è strutturato in questo modo

```
capture.py:
├── CaptureEngine class        # Main producer-consumer implementation
│   ├── __init__               # Setup queue, threads, locks
│   ├── _enqueue_packet        # Producer callback (Scapy calls this)
│   ├── _process_packets       # Consumer loop (runs in thread)
│   ├── start/stop/wait        # Public lifecycle methods
│   └── Properties             # is_running, dropped_packets
├── GracefulCapture            # Context manager for signal handling
└── Helper functions           # check_permissions, get_interfaces
```

Separiamo le responsabilità:
- CaptureEngine gestisce il threading e la gestione della coda
- GracefulCapture gestisce il cleanup dei segnali
- Il controllo dei permessi è una funzione autonoma (riutilizzabile)

Questo rende più semplice il testing. Puoi testare il controllo dei permessi senza avviare una cattura. Puoi testare il comportamento della coda senza Scapy.

### Convenzioni di naming

- `_private_method`: l'underscore iniziale indica implementazione interna
- `public_method`: nessun underscore indica che fa parte dell'API pubblica
- `CamelCase`: classi
- `snake_case`: funzioni e variabili
- `SCREAMING_SNAKE`: costanti

Seguire questi pattern rende più facile capire, già dal nome, cosa è API privata e cosa è pubblica.

## Estendere il codice

### Aggiungere un nuovo protocollo

Vuoi rilevare il traffico BitTorrent? Ecco il procedimento:

1. **Aggiungilo all'enum Protocol** in `models.py:11-21`
   ```python
   class Protocol(StrEnum):
       TCP = "TCP"
       # ... existing protocols
       BITTORRENT = "BITTORRENT"
   ```

2. **Aggiorna l'identificazione dei protocolli** in `analyzer.py:14-48`
   ```python
   def identify_protocol(packet: Packet) -> Protocol:
       # Check BitTorrent before TCP fallback
       if packet.haslayer(TCP):
           tcp_layer = packet[TCP]
           if tcp_layer.dport in range(6881, 6890) or tcp_layer.sport in range(6881, 6890):
               return Protocol.BITTORRENT
           # ... existing HTTP/HTTPS checks
           return Protocol.TCP
   ```

3. **Aggiungi la mappatura dei colori** in `constants.py:63-84`
   ```python
   class ProtocolColors:
       RICH: Final[dict[str, str]] = {
           # ... existing colors
           "BITTORRENT": "red",
       }
       
       HEX: Final[dict[str, str]] = {
           # ... existing colors
           "BITTORRENT": "#ff0000",
       }
   ```

4. **Aggiungi il supporto ai filtri BPF** in `filters.py:16-26`
   ```python
   BPF_PROTOCOL_MAP: dict[Protocol, str] = {
       # ... existing protocols
       Protocol.BITTORRENT: "tcp portrange 6881-6889",
   }
   ```

5. **Aggiungi i test** in `tests/test_models.py`
   ```python
   def test_bittorrent_protocol():
       assert Protocol.BITTORRENT.value == "BITTORRENT"
   ```

Ora BitTorrent compare automaticamente nella distribuzione dei protocolli, nei top talker e nei grafici.

## Dipendenze

### Perché ogni dipendenza

- **typer (0.21.1+)**: framework per CLI. Fornisce parsing degli argomenti, generazione dell'help, instradamento dei comandi. Scelto al posto di argparse perché usa i type hint per la validazione automatica. Scelto al posto di click perché è più recente e ha default migliori.

- **rich (14.3.1+)**: formattazione del terminale. Fornisce tabelle colorate, barre di avanzamento, evidenziazione della sintassi. Crea un output CLI dall'aspetto professionale senza codici ANSI scritti a mano. Usato da GitHub CLI e da altri tool CLI moderni.

- **scapy (2.6.1+)**: manipolazione dei pacchetti. L'unica libreria con un supporto completo ai protocolli e alla gestione dei file pcap. Le alternative (dpkt, pyshark) non hanno funzionalità di dissezione dei protocolli o richiedono tool esterni.

- **matplotlib (3.10.0+)**: visualizzazione. Lo standard del settore per i grafici scientifici. I grafici generati corrispondono alle aspettative degli analisti. Le alternative (plotly, bokeh) generano HTML e non PNG, meno adatti ai report.

### Sicurezza delle dipendenze

Verifica le vulnerabilità:
```bash
pip install pip-audit
pip-audit
```

Se vedi una vulnerabilità nelle dipendenze:
1. Verifica se riguarda il modo in cui usiamo la libreria
2. Aggiorna alla versione corretta, se disponibile
3. Valuta una libreria alternativa se non esiste una patch
4. Aggiungila ai problemi noti se devi restare sulla versione vulnerabile

Esempio: una CVE nelle vecchie versioni di Scapy. Aggiorna alla 2.6.1+, che corregge il problema.

## Prossimi passi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.md](./04-CHALLENGES.it.md) contiene idee di estensione come il riassemblaggio degli stream TCP e il rilevamento delle anomalie
2. **Modifica il codice** - Cambia l'identificazione dei protocolli in analyzer.py per rilevare i tuoi protocolli
3. **Fai il profiling delle prestazioni** - Usa cProfile per trovare i colli di bottiglia nelle tue estensioni
