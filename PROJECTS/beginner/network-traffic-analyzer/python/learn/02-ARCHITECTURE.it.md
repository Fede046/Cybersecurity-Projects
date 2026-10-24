# Architettura del sistema

Questo documento analizza come è progettato il sistema e perché sono state prese determinate decisioni architetturali.

## Architettura ad alto livello

```
┌──────────────────────────────────────────────────────────┐
│                   CLI Interface (Typer)                  │
│                      main.py                             │
└────────────────────┬─────────────────────────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌──────────┐
    │Capture │  │Analyze │  │Visualize │
    │ Engine │  │ PCAP   │  │ Charts   │
    └───┬────┘  └───┬────┘  └────┬─────┘
        │           │             │
        │     ┌─────┴──────┐      │
        │     │            │      │
        ▼     ▼            ▼      ▼
    ┌─────────────────────────────────┐
    │    Producer-Consumer Queue      │
    │                                 │
    │  ┌──────────┐   ┌─────────────┐│
    │  │ Producer │──>│    Queue    ││
    │  │ (Scapy)  │   │  (bounded)  ││
    │  └──────────┘   └──────┬──────┘│
    │                        │       │
    │                        ▼       │
    │                  ┌───────────┐ │
    │                  │ Consumer  │ │
    │                  │ (Process) │ │
    │                  └─────┬─────┘ │
    └────────────────────────┼───────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   Statistics    │
                    │   Collector     │
                    │ (Thread-Safe)   │
                    └────────┬────────┘
                             │
         ┌───────────────────┼────────────────┐
         │                   │                │
         ▼                   ▼                ▼
    ┌────────┐         ┌─────────┐      ┌────────┐
    │Console │         │  Export │      │ Charts │
    │ Output │         │JSON/CSV │      │  PNG   │
    └────────┘         └─────────┘      └────────┘
```

### Scomposizione dei componenti

**Interfaccia CLI (main.py)**
- Scopo: fornisce i comandi rivolti all'utente (capture, analyze, export, chart)
- Responsabilità: parsing degli argomenti, instradamento dei comandi, visualizzazione degli errori
- Interfacce: chiama CaptureEngine, analyze_pcap_file e le funzioni di visualizzazione

**Capture Engine (capture.py)**
- Scopo: cattura dei pacchetti in tempo reale con threading producer-consumer
- Responsabilità: gestione dei raw socket, controllo dei privilegi, arresto controllato
- Interfacce: AsyncSniffer di Scapy, Queue per il threading, StatisticsCollector per le metriche

**Analyzer (analyzer.py)**
- Scopo: identificazione dei protocolli ed estrazione dei campi dei pacchetti
- Responsabilità: dissezione dei layer, classificazione dei protocolli, conversione delle strutture dati
- Interfacce: accetta oggetti Packet di Scapy, restituisce dataclass PacketInfo

**Statistics Collector (statistics.py)**
- Scopo: aggregazione thread-safe delle metriche dei pacchetti
- Responsabilità: gestione dei contatori, campionamento della banda, tracciamento degli endpoint
- Interfacce: record_packet() chiamata dal thread consumer, get_statistics() per gli snapshot

**Filter Builder (filters.py)**
- Scopo: costruzione type-safe di espressioni di filtro BPF
- Responsabilità: validazione dell'input, combinazione delle espressioni, generazione della sintassi BPF
- Interfacce: API fluente per concatenare i filtri, build() produce la stringa BPF

**Visualizzazione (visualization.py)**
- Scopo: generare grafici a partire dalle statistiche di cattura
- Responsabilità: creazione delle figure Matplotlib, stile dei grafici, esportazione su file
- Interfacce: accetta CaptureStatistics, produce oggetti Figure

**Export (export.py)**
- Scopo: serializzare i dati di cattura in formati su disco
- Responsabilità: formattazione JSON/CSV, conversione delle strutture dati
- Interfacce: prende CaptureStatistics e liste di PacketInfo, scrive file

**Output (output.py)**
- Scopo: formattazione Rich per la visualizzazione nel terminale
- Responsabilità: generazione di tabelle, barre di avanzamento, output colorato
- Interfacce: singleton Console, funzioni print_* per i diversi tipi di dati

## Flusso dei dati

### Flusso della cattura live dei pacchetti

Analisi passo per passo di cosa succede durante la cattura live:

```
1. User runs command → main.py:capture() (line 110)
   Parses arguments (interface, filter, count, timeout)
   Creates CaptureConfig dataclass

2. Config → CaptureEngine.__init__() (line 46)
   Initializes Queue(maxsize=10000)
   Creates StatisticsCollector
   Sets up threading.Event for shutdown coordination

3. CaptureEngine.start() → AsyncSniffer.start() (line 112)
   Scapy starts producer thread
   Calls _enqueue_packet callback for each packet
   Producer: packet → Queue.put_nowait()

4. Consumer thread _process_packets() runs in parallel (line 72)
   Loop: Queue.get() → extract_packet_info() → record_packet()
   Packet → analyzer.py:extract_packet_info() (line 51)
   PacketInfo → statistics.py:record_packet() (line 47)

5. Statistics update (thread-safe with lock) (line 48-67)
   Increment counters (total_packets, total_bytes)
   Update protocol_distribution dict
   Update endpoint statistics
   Check if bandwidth sample interval elapsed

6. User presses Ctrl+C → GracefulCapture handles signal
   Sets stop_event → consumer thread exits
   Calls sniffer.stop() → producer thread exits
   Returns final CaptureStatistics snapshot

7. Statistics → output.py:print_*() functions (line 84-170)
   Formats Rich tables for protocols, top talkers
   Displays bandwidth graphs
   Shows capture summary panel
```

Esempio con riferimenti al codice:

```python
# Entry point: main.py:159
def capture(interface, filter_expr, count, timeout, output, verbose):
    config = CaptureConfig(
        interface = interface,
        bpf_filter = filter_expr,
        packet_count = count,
        timeout_seconds = timeout,
    )
    
    # Producer-consumer setup: capture.py:112-131
    engine = CaptureEngine(config=config)
    engine.start()  # Spawns threads
    
    # Processing loop: capture.py:72-90
    while not self._stop_event.is_set():
        packet = self._queue.get()
        info = extract_packet_info(packet)  # analyzer.py:51
        self._stats.record_packet(info)     # statistics.py:47
```

### Flusso di analisi di un file PCAP

```
1. User: netanal analyze traffic.pcap
   ↓
2. main.py:analyze() (line 237)
   Validates file exists
   ↓
3. analyzer.py:analyze_pcap_file() (line 162)
   Opens PcapReader (memory efficient iteration)
   ↓
4. For each packet in file:
   extract_packet_info() → PacketInfo
   StatisticsCollector.record_packet()
   ↓
5. Returns CaptureStatistics
   ↓
6. output.py formats and displays
   Protocol table, top talkers, summary
```

## Design pattern

### Pattern producer-consumer

**Che cos'è:**
Separa la generazione dei dati dalla loro elaborazione usando una coda come buffer. I thread producer aggiungono elementi alla coda, i thread consumer li rimuovono e li elaborano. Disaccoppia la velocità di produzione dalla velocità di consumo.

**Dove lo usiamo:**
`capture.py:46-90` implementa il pattern completo. AsyncSniffer è il producer, il loop _process_packets è il consumer.

**Perché lo abbiamo scelto:**
La cattura dei pacchetti deve girare alla velocità della rete senza perdere pacchetti. L'elaborazione (identificazione dei protocolli, aggiornamento delle statistiche, callback opzionali) è più lenta. Il buffering in una coda previene la perdita di pacchetti quando l'elaborazione resta indietro.

**Compromessi:**
- Pro: previene la perdita di pacchetti, disaccoppia le responsabilità, consente il parallelismo
- Contro: usa memoria per il buffer della coda, aggiunge latenza (i pacchetti attendono in coda), richiede la sincronizzazione dei thread

Esempio di implementazione:
```python
# capture.py:64-70 - Producer callback
def _enqueue_packet(self, packet: Packet) -> None:
    try:
        self._queue.put_nowait(packet)
    except Full:
        with self._count_lock:
            self._dropped_packets += 1

# capture.py:72-90 - Consumer loop
def _process_packets(self) -> None:
    while not self._stop_event.is_set():
        try:
            packet = self._queue.get(timeout=0.1)
        except Empty:
            continue
        
        info = extract_packet_info(packet)
        self._stats.record_packet(info)
```

Il producer non si blocca mai a causa di un'elaborazione lenta. Il consumer elabora al proprio ritmo. Se la coda si riempie, i pacchetti vengono scartati incrementando un contatore invece di mandare tutto in crash.

### Builder pattern

**Che cos'è:**
Costruisce oggetti complessi passo dopo passo attraverso un'interfaccia fluente. Ogni metodo restituisce self, consentendo il method chaining. La chiamata finale a build() produce il risultato.

**Dove lo usiamo:**
`filters.py:48-175` implementa FilterBuilder per le espressioni BPF.

**Perché lo abbiamo scelto:**
La sintassi BPF è soggetta a errori. Gli utenti possono costruire filtri type-safe con una validazione a ogni passo, invece di una concatenazione di stringhe soggetta a errori.

**Compromessi:**
- Pro: type safety, validazione dell'input, API leggibile, previene l'injection
- Contro: più codice rispetto alle stringhe grezze, richiede di capire il builder

Esempio:
```python
# filters.py:48-175
filter_expr = (
    FilterBuilder()
    .protocol(Protocol.TCP)
    .port(443)
    .host("192.168.1.1")
    .build()
)
# Result: "(tcp) and port 443 and host 192.168.1.1"

# Validates each input:
# filters.py:30-37
def _validate_port(port_number: int) -> None:
    if not PortRange.MIN <= port_number <= PortRange.MAX:
        raise ValidationError(f"Port must be 0-65535, got {port_number}")
```

### Pattern dataclass con slots

**Che cos'è:**
Le dataclass Python con `slots=True` riducono l'uso della memoria memorizzando gli attributi in slot fissi invece che in un dizionario. Le dataclass frozen sono immutabili.

**Dove lo usiamo:**
Tutti i modelli in `models.py:11-159` usano dataclass con slots.

**Perché lo abbiamo scelto:**
Le packet capture generano da migliaia a milioni di oggetti PacketInfo. Gli slots riducono la memoria per oggetto di circa il 40%. L'immutabilità previene modifiche accidentali.

**Compromessi:**
- Pro: minore uso di memoria, sicurezza dell'immutabilità, schema chiaro
- Contro: non è possibile aggiungere attributi dinamicamente, istanziazione leggermente più lenta

Esempio:
```python
# models.py:22-35
@dataclass(frozen = True, slots = True)
class PacketInfo:
    timestamp: float
    src_ip: str
    dst_ip: str
    protocol: Protocol
    size: int
    src_port: int | None = None
    dst_port: int | None = None
    src_mac: str | None = None
    dst_mac: str | None = None
```

Con 1 milione di pacchetti, gli slots fanno risparmiare circa 40MB rispetto agli attributi basati su dizionario.

### Pattern context manager

**Che cos'è:**
Oggetti che implementano `__enter__` e `__exit__` per la preparazione e il rilascio delle risorse. Usati con le istruzioni `with` per garantire il cleanup anche in caso di eccezioni.

**Dove lo usiamo:**
`capture.py:197-230` implementa il context manager GracefulCapture.

**Perché lo abbiamo scelto:**
Garantisce un arresto controllato anche se l'utente preme Ctrl+C o si verificano eccezioni. I signal handler vengono ripristinati correttamente e la cattura si ferma in modo pulito.

**Compromessi:**
- Pro: cleanup garantito, sintassi pulita, sicuro rispetto alle eccezioni
- Contro: boilerplate aggiuntivo, necessità di capire il protocollo `__enter__/__exit__`

Esempio:
```python
# capture.py:197-230
class GracefulCapture:
    def __enter__(self) -> CaptureEngine:
        # Setup: Install signal handlers
        self._original_sigint = signal.signal(signal.SIGINT, self._handle_signal)
        self._engine.start()
        return self._engine
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        # Cleanup: Restore handlers, stop capture
        signal.signal(signal.SIGINT, self._original_sigint)
        self._engine.stop()

# Usage: main.py:159-167
with GracefulCapture(engine) as cap:
    stats = cap.wait()
```

## Separazione in layer

```
┌────────────────────────────────────┐
│    CLI Layer (main.py)             │
│    - Command definitions           │
│    - Argument parsing              │
│    - User interaction              │
└────────────────────────────────────┘
           ↓ calls
┌────────────────────────────────────┐
│    Service Layer                   │
│    - capture.py: CaptureEngine     │
│    - analyzer.py: Protocol logic   │
│    - filters.py: Filter building   │
└────────────────────────────────────┘
           ↓ uses
┌────────────────────────────────────┐
│    Data Layer                      │
│    - statistics.py: Aggregation    │
│    - models.py: Data structures    │
│    - constants.py: Configuration   │
└────────────────────────────────────┘
           ↓ produces
┌────────────────────────────────────┐
│    Output Layer                    │
│    - output.py: Console display    │
│    - visualization.py: Charts      │
│    - export.py: File I/O           │
└────────────────────────────────────┘
```

### Perché i layer?

La separazione delle responsabilità previene un accoppiamento stretto. I comandi CLI non sanno nulla degli internals di Scapy. CaptureEngine non sa nulla della formattazione Rich. Le modifiche alla visualizzazione non influenzano la raccolta delle statistiche.

### Cosa sta dove

**CLI Layer (main.py):**
- File: main.py, __main__.py
- Import: può importare da tutti i layer
- Vietato: uso diretto di Scapy, formattazione Rich (da delegare a output.py), Matplotlib (da delegare a visualization.py)

**Service Layer:**
- File: capture.py, analyzer.py, filters.py
- Import: solo dal data layer, nessuna dipendenza da CLI o output
- Vietato: istruzioni print (restituisci invece i dati), sys.exit() (solleva eccezioni)

**Data Layer:**
- File: statistics.py, models.py, constants.py, exceptions.py
- Import: solo la libreria standard e i type hint
- Vietato: qualsiasi I/O, qualsiasi import di terze parti (tranne che per il type checking)

**Output Layer:**
- File: output.py, visualization.py, export.py
- Import: data layer per i modelli, librerie di formattazione di terze parti
- Vietato: business logic, elaborazione dei pacchetti

## Modelli dei dati

### PacketInfo

```python
# models.py:22-35
@dataclass(frozen = True, slots = True)
class PacketInfo:
    timestamp: float
    src_ip: str
    dst_ip: str
    protocol: Protocol
    size: int
    src_port: int | None = None
    dst_port: int | None = None
    src_mac: str | None = None
    dst_mac: str | None = None
```

**Spiegazione dei campi:**
- `timestamp`: tempo Unix epoch della cattura del pacchetto. Float per avere precisione al microsecondo. Usato per i calcoli della banda e per l'analisi delle serie temporali.
- `src_ip/dst_ip`: indirizzi IP come stringhe (IPv4 o IPv6). Non validati a livello di modello (li valida l'analyzer). Usati per il tracciamento degli endpoint.
- `protocol`: enum Protocol (TCP, UDP, ICMP, ecc.). Determinato da analyzer.identify_protocol(). Usato per le statistiche di distribuzione.
- `size`: dimensione totale del pacchetto in byte, inclusi tutti gli header. Usata per i calcoli della banda e del volume di traffico.
- `src_port/dst_port`: opzionali perché ICMP/ARP non hanno porte. None significa non applicabile o non estratto.
- `src_mac/dst_mac`: indirizzi di Layer 2 opzionali. Utili per l'analisi della rete locale, meno rilevanti per il traffico instradato.

**Relazioni:**
- La dataclass frozen previene modifiche accidentali dopo la creazione
- Creato da analyzer.extract_packet_info() a partire da oggetti Packet di Scapy
- Consumato da statistics.StatisticsCollector.record_packet()
- Memorizzato in liste per l'export ma non mantenuto in memoria durante la cattura live (solo le statistiche)

### EndpointStats

```python
# models.py:38-61
@dataclass(slots = True)
class EndpointStats:
    ip_address: str
    packets_sent: int = 0
    packets_received: int = 0
    bytes_sent: int = 0
    bytes_received: int = 0
    
    @property
    def total_packets(self) -> int:
        return self.packets_sent + self.packets_received
    
    @property
    def total_bytes(self) -> int:
        return self.bytes_sent + self.bytes_received
```

**Scopo:** tenere traccia del traffico bidirezionale per ogni indirizzo IP. Usato per identificare i "top talker" e per definire la baseline.

**Relazioni:**
- Mutabile (non frozen) perché i contatori si incrementano per tutta la durata della cattura
- Un'istanza per ogni indirizzo IP univoco osservato
- Memorizzato nel dizionario statistics.StatisticsCollector._endpoints
- Le property consentono di ordinare per volume totale senza memorizzare campi ridondanti

## Architettura di sicurezza

### Threat model

Da cosa ci stiamo proteggendo:

1. **Privilege escalation** - Garantire che la cattura dei pacchetti funzioni solo con i permessi adeguati. Nessun aggiramento della sicurezza del sistema operativo. Verificare esplicitamente le capability prima di tentare la cattura.

2. **Filter injection** - Stringhe di filtro malevole potrebbero mandare in crash il kernel o aggirare le restrizioni previste. Validare tutto l'input dell'utente prima di passarlo al compilatore BPF.

3. **Esaurimento delle risorse** - Code o uso di memoria illimitati potrebbero causare un DoS del sistema di monitoraggio. Usare buffer limitati e limiti ragionevoli.

Da cosa NON ci stiamo proteggendo (fuori scope):

- **Accesso fisico alla rete** - Diamo per scontato che l'attaccante possa collegarsi alla rete. Questo tool non lo impedisce.
- **Ispezione dei payload cifrati** - Analizziamo metadati e header, non il contenuto cifrato. La decifratura TLS richiede proxy MITM.
- **Minacce del quantum computing** - Gli attacchi futuri ai protocolli crittografici non sono affrontati dai tool di cattura dei pacchetti.

### Layer di difesa

```
Layer 1: Privilege Validation
    ↓ (capture.py:341-375)
Layer 2: Input Validation
    ↓ (filters.py:30-66, main.py)
Layer 3: Resource Limits
    ↓ (capture.py:46, constants.py)
Layer 4: Error Handling
    ↓ (exceptions.py, try/except throughout)
```

**Perché più layer?**

Difesa in profondità. Se la validazione dell'input ha un bug, i limiti sulle risorse prevengono il DoS. Se il controllo dei privilegi viene aggirato, il kernel impone comunque i permessi. Ogni layer intercetta vettori d'attacco diversi.

## Strategia di memorizzazione

### Statistiche in memoria

**Cosa memorizziamo:**
- Contatori aggregati (pacchetti totali, byte)
- Distribuzioni per protocollo
- Statistiche per endpoint  
- Campioni di banda (serie temporale)

**Perché in memoria:**
Prestazioni. L'I/O su disco durante una cattura ad alta velocità fa perdere pacchetti. Le statistiche si aggiornano a ogni pacchetto, il che richiede una latenza nell'ordine dei nanosecondi. La RAM la garantisce, il disco no.

**Gestione della memoria:**
```python
# constants.py:36-41
class CaptureDefaults:
    QUEUE_SIZE: Final[int] = 10_000
    BANDWIDTH_SAMPLE_INTERVAL_SECONDS: Final[float] = 1.0
```

La dimensione della coda limita la memoria a circa 10K pacchetti × ~1,5KB = 15MB al massimo. I campioni di banda a 1 al secondo significano 3600 campioni all'ora = circa 100KB all'ora. Le statistiche degli endpoint dipendono dal numero di IP univoci osservati.

### Export su disco

Export opzionale in JSON/CSV per la persistenza:
```python
# export.py:80-107
def export_to_json(
    stats: CaptureStatistics,
    filepath: Path,
    packets: list[PacketInfo] | None = None,
    options: ExportOptions | None = None,
) -> None:
```

Avviene solo su richiesta, non durante la cattura. Separa l'hot path (cattura) dal cold path (analisi).

## Configurazione

### Variabili d'ambiente

```bash
NO_COLOR=1           # Disables colored output for CI/CD environments
CI=1                 # Optimizes output for continuous integration
PYTHONUNBUFFERED=1   # Forces unbuffered stdout for real-time logs
```

### Strategia di configurazione

Le costanti in `constants.py` forniscono valori di default sensati. Gli argomenti da riga di comando sovrascrivono i default. Nessun file di configurazione, per evitare complessità in un tool semplice.

**Sviluppo:**
```python
# constants.py provides overridable defaults
CaptureDefaults.QUEUE_SIZE = 10_000  # Balance memory vs packet loss
```

**Produzione:**
Regola la dimensione della coda in base alla memoria disponibile e al tasso di pacchetti previsto. Una coda da 10K gestisce circa 1-2 Gbps di traffico sostenuto.

## Considerazioni sulle prestazioni

### Colli di bottiglia

Dove questo sistema rallenta sotto carico:

1. **Contesa sulla coda** - Producer e consumer accedono entrambi alla coda. A tassi estremi (10+ Gbps), le operazioni sulla coda diventano un punto di serializzazione. Si mitiga con più code e worker thread.

2. **Lock delle statistiche** - Ogni acquisizione di un pacchetto richiede il lock in record_packet(). A milioni di pacchetti al secondo, la contesa sul lock diventa dominante. Si mitiga con contatori lock-free o statistiche per thread con merge periodico.

### Ottimizzazioni

Cosa abbiamo fatto per renderlo più veloce:

- **Filtraggio BPF nel kernel**: scarta circa il 99% dei pacchetti irrilevanti prima che lo userspace li veda. Passare dal filtraggio nello userspace al filtro BPF ha ridotto l'uso della CPU dall'80% al 5% nei test con un filtro sulla porta 80 su una rete trafficata.

- **Coda limitata con put non bloccante**: usare `put_nowait()` con un contatore esplicito dei pacchetti scartati impedisce al producer di bloccarsi. Il thread di cattura non aspetta mai un consumer lento.

- **Dataclass slots**: riduce la memoria per pacchetto del 40%. Con una coda da 10K, fa risparmiare 6MB. Consente code più grandi con lo stesso budget di memoria.

- **Formattazione delle stringhe minima**: l'output viene formattato solo al momento della visualizzazione, non durante la cattura. `print_packet()` viene chiamata solo se è impostato il flag `--verbose`.

### Scalabilità

**Scalabilità verticale:**
Aggiungi più CPU/RAM a una singola macchina. La cattura dei pacchetti è CPU-bound (parsing dei protocolli) e memory-bound (memorizzazione nella coda). Un sistema a 8 core con 32GB di RAM può gestire circa 5-10 Gbps a seconda del mix di traffico.

**Scalabilità orizzontale:**
Richiede modifiche architetturali:
- Duplicare il traffico verso più host di cattura
- Usare una coda distribuita (Kafka/RabbitMQ) invece della Queue in memoria
- Aggregare le statistiche da più collector
- Il codice attuale non lo supporta senza modifiche

## Decisioni di design

### Decisione 1: AsyncSniffer vs sniff() sincrono

**Cosa abbiamo scelto:**
AsyncSniffer con thread in background

**Alternative considerate:**
- `sniff(prn=callback)` - Scartata perché blocca il thread principale, impedendo l'arresto controllato e la visualizzazione dell'avanzamento
- `sniff(timeout=1)` in un loop - Scartata perché introduce intervalli in cui i pacchetti possono andare persi tra il timeout e il riavvio

**Compromessi:**
Guadagnato: UI reattiva, arresto controllato, elaborazione concorrente
Perso: logica di threading leggermente più complessa, necessità di gestire la coda

### Decisione 2: lock dei thread vs algoritmi lock-free

**Cosa abbiamo scelto:**
`threading.Lock()` per la protezione delle statistiche

**Alternative considerate:**
- Atomic lock-free - Scartati perché Python non ha vere operazioni atomiche (il GIL esiste ma qui non aiuta)
- Nessuna sincronizzazione - Scartata perché causa race condition e corruzione dei dati

**Compromessi:**
Guadagnato: correttezza, semplicità, pattern standard
Perso: un po' di prestazioni a tassi di pacchetti estremi (milioni al secondo), possibile contesa sul lock

### Decisione 3: dataclass vs named tuple

**Cosa abbiamo scelto:**
Dataclass frozen con slots

**Alternative considerate:**
- Named tuple - Scartate perché mancano di type checking, non hanno valori di default, sono più difficili da estendere
- Classi normali - Scartate per il codice boilerplate, l'assenza di un `__repr__` automatico e il maggiore uso di memoria

**Compromessi:**
Guadagnato: type safety, valori di default, meno boilerplate, miglior uso della memoria
Perso: richiede Python 3.10+ per gli slots nelle dataclass

## Prossimi passi

Ora che hai capito l'architettura:

1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per un'analisi guidata del codice che mostra come funziona davvero ogni componente
2. Prova a modificare la dimensione della coda in constants.py e osserva l'impatto sulla perdita di pacchetti sotto carico
3. Segui un singolo pacchetto dalla cattura alle statistiche fino all'output aggiungendo dei print di debug
