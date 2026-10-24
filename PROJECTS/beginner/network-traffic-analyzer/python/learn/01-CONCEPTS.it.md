# Concetti di sicurezza fondamentali

Questo documento spiega i concetti di sicurezza che incontrerai costruendo questo progetto. Non si tratta solo di definizioni. Approfondiremo perché sono importanti e come funzionano davvero.

## Cattura dei pacchetti e raw socket

### Che cos'è

Catturare i pacchetti significa leggere i frame di rete direttamente dall'interfaccia di rete prima che il sistema operativo li elabori. Normalmente, il tuo sistema operativo mostra alle applicazioni solo i dati indirizzati a loro. La cattura dei pacchetti ti permette di vedere TUTTO il traffico sul segmento di rete, comprese le comunicazioni di altre macchine.

I raw socket forniscono accesso diretto ai protocolli di rete al di sotto del transport layer. A differenza dei normali socket TCP, in cui il kernel gestisce lo stato della connessione, i raw socket ti permettono di costruire e ispezionare pacchetti a livello IP o inferiore.

### Perché è importante

Durante il breach di DigiNotar del 2011, gli attaccanti hanno emesso certificati SSL fraudolenti per Google e altri siti. Il monitoraggio della rete se ne è accorto perché i certificati falsi comparivano negli handshake TLS visibili ai tool di cattura dei pacchetti. Il certificate pinning non bastava perché gli utenti si fidavano della CA. L'ispezione a livello di pacchetto ha rivelato la contraffazione.

Senza la capacità di catturare i pacchetti, sei cieco rispetto a:
- Quali protocolli girano davvero sulla tua rete (non solo quelli che dovrebbero girare)
- Credenziali non cifrate inviate su HTTP o FTP
- Esfiltrazione di dati tramite DNS tunneling o ICMP
- Movimento laterale tra macchine compromesse

### Come funziona

Lo stack di rete del sistema operativo si presenta così:

```
Application Layer
      ↓
  Socket API
      ↓
Transport Layer (TCP/UDP)
      ↓
  Network Layer (IP)
      ↓
   Link Layer (Ethernet)
      ↓
Physical Network Interface
```

Le applicazioni normali interagiscono a livello di Socket API. Chiamano `socket.connect()` e il kernel gestisce tutto ciò che sta sotto. La cattura dei pacchetti opera a livello di Link Layer, vedendo i frame Ethernet grezzi prima che il kernel li elabori.

Su Linux, questo richiede la capability CAP_NET_RAW. Il kernel controlla questo permesso prima di consentire i socket AF_PACKET. Da `capture.py:354-360`:

```python
def _check_linux_permissions() -> tuple[bool, str]:
    if os.geteuid() == 0:
        return True, "Running as root"
    
    try:
        sock = socket.socket(
            socket.AF_PACKET,
            socket.SOCK_RAW,
            socket.htons(0x0003),
        )
```

Il codice prova a creare un raw packet socket. Se ci riesce, hai CAP_NET_RAW. Se ottieni PermissionError, ti servono privilegi elevati.

### Attacchi comuni

1. **Sniffing in modalità promiscua** - L'attaccante cattura tutto il traffico su un segmento di rete, non solo quello indirizzato a lui. Sulle reti con switch questo richiede l'ARP spoofing per reindirizzare il traffico. Sul wireless, la monitor mode cattura tutti i frame. Difenditi con la cifratura (TLS/SSL) e la segmentazione della rete.

2. **Packet injection** - L'attaccante costruisce pacchetti malevoli e li inietta nella rete. Gli attacchi di predizione dei numeri di sequenza TCP funzionano così. Lo spoofing dello stack TCP/IP di Metasploit si basa sui raw socket. Difenditi con l'egress filtering e il connection tracking sul firewall.

3. **Analisi dei protocolli per la ricognizione** - Gli attaccanti catturano pacchetti per mappare la topologia della tua rete, identificare i servizi e trovare versioni vulnerabili. La ricognizione passiva è difficile da rilevare perché non genera traffico. Difenditi con la cifratura e monitorando catture di pacchetti insolite (tool come ArpON rilevano le interfacce in modalità promiscua).

### Strategie di difesa

Questo progetto implementa diverse protezioni:

**Controllo dei privilegi** - Prima di avviare la cattura, il codice verifica i permessi (`capture.py:341-347`). Questo evita messaggi di errore confusi e spiega chiaramente cosa serve. I tool di produzione falliscono subito con errori che indicano cosa fare.

**Filtraggio BPF** - Invece di elaborare ogni pacchetto nello userspace, i filtri BPF girano nel kernel e scartano il traffico irrilevante. In `filters.py:83-92`, il FilterBuilder valida gli input prima di inviare i filtri al kernel. Questo previene attacchi di filter injection in cui un input malevolo potrebbe mandare in crash l'engine di cattura.

**Operazioni in sola lettura** - Questo tool cattura e analizza i pacchetti ma non li modifica né li inietta mai. Il principio del privilegio minimo: la cattura richiede permessi elevati, ma non usiamo quei permessi per nient'altro che la lettura.

## Berkeley Packet Filter (BPF)

### Che cos'è

BPF è una macchina virtuale all'interno del kernel Linux/BSD che filtra in modo efficiente i pacchetti prima che raggiungano lo userspace. Scrivi espressioni di filtro come "tcp port 80" che vengono compilate in bytecode BPF. Il kernel esegue questo bytecode su ogni pacchetto, conservando solo quelli che corrispondono.

### Perché è importante

Senza BPF, la cattura dei pacchetti nello userspace è troppo lenta per le reti ad alta velocità. Ogni pacchetto provoca un context switch dal kernel allo userspace. A 10 Gbps, si tratta di milioni di interrupt al secondo. BPF esegue il filtraggio nel kernel, riducendo i context switch di ordini di grandezza.

La botnet Mirai del 2016 ha sommerso le reti con semplici UDP flood. Gli operatori di rete hanno usato filtri BPF per scartare il traffico d'attacco a livello di kernel, mantenendo operativi i propri tool di monitoraggio. Senza BPF, sarebbero crollati gli stessi tool di cattura.

### Come funziona

BPF compila le espressioni di filtro in bytecode che gira in una macchina virtuale basata su registri. Ecco cosa succede quando scrivi "tcp port 443":

```
Load protocol field from IP header
Compare with TCP (protocol 6)
If not TCP, reject packet
Load destination port from TCP header  
Compare with 443
If not 443, reject packet
Accept packet
```

Questo gira nel kernel per ogni pacchetto prima che lo userspace lo veda. In `filters.py:136-150`, il FilterBuilder crea queste espressioni:

```python
def port(self, port_number: int) -> FilterBuilder:
    _validate_port(port_number)
    self._expressions.append(f"port {port_number}")
    return self

def build(self, operator: Literal["and", "or"] = "and") -> str | None:
    if not self._expressions:
        return None
    return f" {operator} ".join(self._expressions)
```

Le espressioni si combinano con operatori booleani. "tcp and port 443 and host 192.168.1.1" diventa bytecode BPF che verifica tutte e tre le condizioni in modo efficiente.

### Errori comuni

**Errore 1: non validare la sintassi del filtro**
```python
# Bad - passes invalid filter to kernel
filter_expr = user_input  # "tcp port foobar"
sniffer = AsyncSniffer(filter=filter_expr)  # Crashes
```

Il kernel rifiuta la sintassi BPF non valida con errori criptici. Da `filters.py:227-235`:

```python
def validate_bpf_filter(filter_str: str) -> bool:
    try:
        from scapy.arch import compile_filter
        compile_filter(filter_str)
        return True
    except Exception:
        return False
```

Valida prima che inizi la cattura, non quando l'utente ha già aspettato 5 minuti.

**Errore 2: filter injection tramite input non sanificato**

```python
# Bad - user can inject arbitrary filters  
filter_expr = f"host {user_ip}"  # user_ip = "1.2.3.4 or (tcp port 1-65535)"
```

In `filters.py:35-41`, la validazione lo intercetta:

```python
def _validate_ip_address(ip_address: str) -> None:
    try:
        ipaddress.ip_address(ip_address)
    except ValueError as e:
        raise ValidationError(f"Invalid IP address: {ip_address}") from e
```

Valida sempre gli input con un controllo di tipo adeguato, non con la concatenazione di stringhe.

## Analisi dei layer di protocollo

### Che cos'è

I protocolli di rete si impilano in layer, ognuno dei quali aggiunge il proprio header con dei metadati. I frame Ethernet contengono pacchetti IP, che contengono segmenti TCP, che contengono richieste HTTP. L'analisi dei protocolli consiste nel sezionare questi layer per estrarre informazioni a ciascun livello.

Il modello OSI definisce sette layer, ma TCP/IP ne usa quattro pratici:

```
Layer 4: Application  (HTTP, DNS, SSH)
Layer 3: Transport    (TCP, UDP)
Layer 2: Network      (IP)
Layer 1: Link         (Ethernet)
```

### Come funziona

In `analyzer.py:14-48`, la funzione identify_protocol percorre i layer:

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
```

Il sistema di layer di Scapy ti permette di verificare `packet.haslayer(TCP)` e di accedere a campi come `packet[TCP].dport`. Ogni layer è un oggetto Python con campi che corrispondono alla specifica del protocollo.

L'estrazione avviene in `analyzer.py:51-103`:

```python
def extract_packet_info(packet: Packet) -> PacketInfo | None:
    if packet.haslayer(Ether):
        ether_layer = packet[Ether]
        src_mac = ether_layer.src
        dst_mac = ether_layer.dst
    
    if packet.haslayer(IP):
        ip_layer = packet[IP]
        src_ip = ip_layer.src
        dst_ip = ip_layer.dst
```

Ogni layer fornisce informazioni diverse. Il link layer fornisce gli indirizzi MAC, il network layer gli IP, il transport layer le porte.

### Attacchi comuni

1. **Protocol tunneling** - Gli attaccanti nascondono traffico malevolo all'interno di protocolli legittimi. Il DNS tunneling esfiltra dati nelle query DNS. L'ICMP tunneling fa girare shell su pacchetti di ping. L'HTTP tunneling aggira i firewall. Il rilevamento richiede l'analisi dei protocolli per individuare pattern insoliti.

2. **Manipolazione degli header** - La manipolazione dei flag TCP (FIN scan, NULL scan, Xmas scan) sonda le porte senza completare gli handshake. Gli attacchi di frammentazione IP sovraccaricano i buffer di riassemblaggio. Difenditi validando la conformità ai protocolli.

3. **Ispezione dei payload cifrati** - Anche il traffico cifrato rivela metadati. Gli handshake TLS mostrano i dettagli dei certificati, lo SNI indica gli hostname di destinazione, le dimensioni e le tempistiche dei pacchetti rivelano il comportamento delle applicazioni. Gli attacchi di traffic analysis funzionano senza decifrare.

### Esempio reale

Le rivelazioni di Snowden del 2013 hanno mostrato il sistema XKEYSCORE della NSA eseguire analisi dei protocolli su larga scala. Catturava metadati da tutti i layer (IP, porte, protocolli, dettagli dei certificati) e li correlava per identificare i bersagli. Non serviva decifrare il traffico per identificare gli utenti e mappare le relazioni di rete.

## Thread safety e concorrenza

### Che cos'è

Il codice thread-safe può essere chiamato da più thread contemporaneamente senza corrompere i dati. Questo richiede primitive di sincronizzazione come lock, code o operazioni atomiche. Senza thread safety, l'accesso concorrente causa race condition in cui il risultato dipende dalla temporizzazione dei thread.

### Perché è importante

La cattura dei pacchetti è intrinsecamente concorrente. I pacchetti arrivano in modo asincrono mentre stai elaborando quelli precedenti. Se perdi pacchetti, ti sfuggono eventi di sicurezza. Se blocchi il thread di cattura, perdi pacchetti. La soluzione è il threading producer-consumer con una coda come buffer.

Da `capture.py:46-62`:

```python
def __init__(
    self,
    config: CaptureConfig,
    on_packet: Callable[[PacketInfo], None] | None = None,
    queue_size: int = CaptureDefaults.QUEUE_SIZE,
) -> None:
    self._queue: Queue[Packet] = Queue(maxsize = queue_size)
    self._stats = StatisticsCollector()
    self._stop_event = threading.Event()
    self._packet_count = 0
    self._dropped_packets = 0
    self._count_lock = threading.Lock()
```

La Queue è thread-safe di default. Il lock protegge le variabili contatore che più thread modificano.

### Come funziona

Il pattern producer-consumer separa la cattura dall'elaborazione:

```
Producer Thread          Consumer Thread
(Scapy AsyncSniffer)     (Processing Loop)
       ↓                        ↓
   Capture packet          Get from queue
       ↓                        ↓
   Put in queue           Analyze packet
       ↓                        ↓
   Repeat                 Update statistics
```

La coda disaccoppia i thread. Il producer non si blocca mai a causa di un'elaborazione lenta. Il consumer elabora al proprio ritmo. La dimensione del buffer determina il compromesso tra uso della memoria e perdita di pacchetti.

In `statistics.py:47-67`, il collector usa un lock per la thread safety:

```python
def record_packet(self, packet: PacketInfo) -> None:
    with self._lock:
        self._total_packets += 1
        self._total_bytes += packet.size
        self._interval_packets += 1
        self._interval_bytes += packet.size
        
        self._protocol_counts[packet.protocol] += 1
        self._protocol_bytes[packet.protocol] += packet.size
```

Il `with self._lock:` garantisce che un solo thread alla volta modifichi le statistiche. Senza di esso, gli incrementi dei contatori entrerebbero in race condition e si perderebbero dei conteggi.

### Errori comuni

**Errore 1: dimenticare di proteggere lo stato condiviso**
```python
# Bad - race condition on packet_count
def _process_packet(self, packet):
    self.packet_count += 1  # Not atomic! 
```

Più thread eseguono read-modify-write sulla stessa variabile. Il thread A legge 100, il thread B legge 100, entrambi scrivono 101. Hai perso un conteggio. Usa lock od operazioni atomiche.

**Errore 2: tenere i lock troppo a lungo**
```python
# Bad - blocks all threads during slow I/O
with self._lock:
    write_to_disk(data)  # File I/O with lock held
```

I lock serializzano l'esecuzione. Tienili solo durante la sezione critica, mai durante l'I/O o calcoli costosi. In `statistics.py:127-143`, il lock protegge la copia dei dati, non il calcolo:

```python
def get_statistics(self) -> CaptureStatistics:
    with self._lock:
        return CaptureStatistics(
            start_time = self._start_time,
            total_packets = self._total_packets,
            protocol_distribution = dict(self._protocol_counts),
        )
```

La copia con dict() avviene all'interno del lock perché tocca dati condivisi. La formattazione e l'elaborazione avvengono fuori dal lock.

## Baseline di rete e rilevamento delle anomalie

### Che cos'è

Una baseline di rete descrive il comportamento normale: rapporti tipici tra protocolli, pattern di banda, coppie di comunicazione. Il rilevamento delle anomalie confronta il traffico attuale con la baseline per identificare le deviazioni. Le deviazioni significative generano alert da investigare.

### Perché è importante

Il worm Stuxnet del 2010 si diffondeva tramite chiavette USB ma comunicava con i server di command and control su HTTP. Le baseline di rete avrebbero segnalato connessioni HTTP insolite da sistemi di controllo industriale che normalmente non accedono mai a internet. Il rilevamento delle anomalie intercetta minacce che i sistemi basati su firme non vedono.

### Come funziona

Questo progetto raccoglie i dati necessari per definire una baseline. In `models.py:95-123`, CaptureStatistics tiene traccia di:

```python
@dataclass(slots = True)
class CaptureStatistics:
    protocol_distribution: dict[Protocol, int] = field(default_factory = dict)
    endpoints: dict[str, EndpointStats] = field(default_factory = dict)
    conversations: dict[tuple[str, str], ConversationStats] = field(default_factory = dict)
    bandwidth_samples: list[BandwidthSample] = field(default_factory = list)
```

La distribuzione dei protocolli mostra il mix di traffico normale. Se la tua rete è di solito al 60% TCP, 30% UDP, 10% altro, uno spostamento improvviso al 90% ICMP indica che qualcosa non va (forse un ping flood).

Le statistiche sugli endpoint tengono traccia di chi parla con chi. Da `statistics.py:82-96`:

```python
def _update_endpoint(
    self,
    ip_address: str,
    sent_bytes: int = 0,
    received_bytes: int = 0,
) -> None:
    if ip_address not in self._endpoints:
        self._endpoints[ip_address] = EndpointStats(
            ip_address = ip_address
        )
    
    endpoint = self._endpoints[ip_address]
    endpoint.bytes_sent += sent_bytes
    endpoint.bytes_received += received_bytes
```

Traccia la banda per IP nel tempo. Una workstation che improvvisamente trasferisce gigabyte merita un'indagine.

### Tecniche di rilevamento

**Rilevamento statistico delle anomalie:**
- Calcola media e deviazione standard per ogni metrica
- Genera un alert quando il valore attuale supera media + 3σ
- Funziona per banda, tassi di pacchetti, rapporti tra protocolli

**Analisi comportamentale:**
- Traccia i grafi di comunicazione (chi parla con chi)
- Genera alert sulle nuove connessioni verso destinazioni insolite
- Rileva il movimento laterale durante i breach

**Analisi delle serie temporali:**
- Campiona la banda ogni secondo (`statistics.py:112-127`)
- Cerca picchi o cali improvvisi
- Gli attacchi DDoS si presentano come aumenti drastici del tasso

## Come si relazionano questi concetti

I concetti si costruiscono l'uno sull'altro a strati:

```
Raw Socket Access
      ↓
 BPF Filtering (efficiency)
      ↓
Protocol Analysis (understanding)
      ↓
Thread-Safe Collection (scale)
      ↓
Baseline Establishment (detection)
```

Ti servono i raw socket per vedere i pacchetti. BPF rende il tutto efficiente. L'analisi dei protocolli estrae il significato. La thread safety consente l'elaborazione in tempo reale. Le baseline rendono possibile il security monitoring.

## Standard e framework del settore

### OWASP Top 10

Questo progetto affronta:

- **A01:2021 - Broken Access Control** - La cattura dei pacchetti richiede un controllo esplicito dei privilegi. Il codice verifica CAP_NET_RAW su Linux e Amministratore su Windows, e fallisce in modo chiaro quando i permessi sono insufficienti (`capture.py:341-375`).

- **A04:2021 - Insecure Design** - Il pattern producer-consumer con code limitate previene l'esaurimento delle risorse. La dimensione della coda limita l'uso della memoria anche sotto packet flood (`capture.py:46-47`).

### MITRE ATT&CK

Tecniche rilevanti:

- **T1040 - Network Sniffing** - Questo tool implementa la tecnica usata dagli attaccanti. Capire come funziona la cattura dei pacchetti aiuta a rilevare quando gli avversari installano degli sniffer. Cerca interfacce in modalità promiscua ed esecuzioni insolite di processi di cattura.

- **T1071 - Application Layer Protocol** - Il codice di identificazione dei protocolli mostra come rilevare il traffico di command and control nascosto in HTTP/HTTPS. Framework C2 come Cobalt Strike usano DNS o HTTP come canali occulti.

- **T1048 - Exfiltration Over Alternative Protocol** - Il DNS tunneling e l'esfiltrazione ICMP emergono nelle distribuzioni dei protocolli. Il rilevamento basato su baseline segnala pattern di utilizzo dei protocolli insoliti.

### CWE

Common weakness enumeration coperte:

- **CWE-362 - Concurrent Execution using Shared Resource with Improper Synchronization** - Il progetto dimostra pattern di locking corretti per le statistiche condivise. Race condition nei contatori dei pacchetti produrrebbero metriche errate (`statistics.py:47-67`).

- **CWE-400 - Uncontrolled Resource Consumption** - Una coda limitata con dimensione configurabile previene l'esaurimento della memoria durante i picchi di traffico. I sistemi di produzione hanno bisogno di meccanismi di backpressure (`capture.py:77-80`).

## Esempi reali

### Caso di studio 1: breach di Anthem Health Insurance (2015)

Gli attaccanti hanno compromesso la rete di Anthem ed esfiltrato 78,8 milioni di record nell'arco di diversi mesi. Il monitoraggio della rete aveva rilevato connessioni insolite dal database verso l'esterno, ma gli alert erano stati ignorati. Una corretta cattura dei pacchetti e un'analisi basata su baseline avrebbero segnalato:

- Server di database che avviavano connessioni HTTPS in uscita (comportamento insolito)
- Grandi trasferimenti di dati fuori orario (anomalia di banda)
- Connessioni verso domini registrati di recente (rilevamento basato sulla reputazione)

Il breach è costato 115 milioni di dollari in risarcimenti. La visibilità sulla rete tramite l'analisi dei pacchetti non è facoltativa negli ambienti con dati sensibili.

### Caso di studio 2: attacco alla supply chain di SolarWinds (2020)

La backdoor SUNBURST comunicava tramite DNS per il command and control. Risolveva sottodomini di avsvmcloud.com per ricevere istruzioni. Le difese tradizionali non se ne sono accorte perché:

- Il DNS in uscita è consentito su tutte le reti
- La cifratura TLS nascondeva il payload del callback HTTP
- A comunicare era un software legittimo (Orion)

Tuttavia, l'analisi a livello di pacchetto ha rivelato delle anomalie:

- Volume insolito di query DNS da parte dei server
- Sottodomini ad alta entropia (dall'aspetto casuale)
- Risposte DNS con TTL sospettosamente lunghi

I tool di monitoraggio della rete che usavano tecniche di cattura dei pacchetti alla fine hanno identificato i sistemi compromessi analizzando i pattern dei metadati DNS, non il contenuto dei payload.

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere:

1. Perché la cattura dei pacchetti richiede privilegi elevati, e di quale specifica capability del kernel ha bisogno su Linux? Come concederesti questa capability senza dare a un programma privilegi di root completi?

2. Spiega come il filtraggio BPF migliora le prestazioni della cattura dei pacchetti rispetto al filtraggio nello userspace. Perché è fondamentale per le reti ad alta velocità? Cosa succede a 10 Gbps senza BPF?

3. Nel pattern producer-consumer usato da questo progetto, cosa succederebbe se il thread consumer si bloccasse per 10 secondi? Come previene la Queue la perdita di dati? Qual è il compromesso tra dimensione della coda e uso della memoria?

Se queste domande non ti sono chiare, rileggi le sezioni pertinenti. L'implementazione avrà più senso una volta che questi fondamenti ti saranno chiari.

## Letture di approfondimento

**Essenziali:**

- **"The TCP/IP Guide" di Charles Kozierok** - Riferimento completo sui protocolli. Leggi le sezioni sul framing Ethernet, sul routing IP, sulla gestione delle connessioni TCP e sulla gestione dei datagrammi UDP. Questi sono i protocolli che sezionerai nelle packet capture.

- **"Building an IDS" (SANS Reading Room)** - Spiega i pattern architetturali del monitoraggio di rete. Il pattern producer-consumer, il signature matching e i concetti di analisi statistica si applicano direttamente a questo progetto.

**Approfondimenti:**

- **"The BSD Packet Filter: A New Architecture for User-level Packet Capture" (McCanne & Jacobson, 1993)** - Il paper originale su BPF. Spiega il design della macchina virtuale e perché il filtraggio a livello di kernel è necessario. Leggilo quando vorrai capire gli internals di BPF.

- **Documentazione delle API PCAP (tcpdump.org)** - Scapy avvolge libpcap/WinPcap/Npcap. Capire la C API sottostante aiuta a fare il debug dei problemi di cattura e spiega le scelte di design di Scapy.

**Contesto storico:**

- **"A Look Back at 'Security Problems in the TCP/IP Protocol Suite'" (Bellovin, 1989)** - Mostra che molti attacchi di rete hanno decenni. Difetti di design dei protocolli del 1989 influenzano ancora oggi la sicurezza. Conoscere la storia evita di ripetere gli stessi errori.
