# Sfide di estensione

Hai costruito il progetto di base. Ora fallo tuo estendendolo con nuove funzionalità.

Queste sfide sono ordinate per difficoltà. Inizia da quelle più facili per prendere confidenza, poi affronta quelle più difficili quando vuoi approfondire.

## Sfide facili

### Sfida 1: aggiungi la visualizzazione del supporto IPv6

**Cosa costruire:**
Migliora l'identificazione dei protocolli per riconoscere esplicitamente e visualizzare i pacchetti IPv6 separatamente da quelli IPv4. Attualmente entrambi vengono mostrati come indirizzi IP ma non vengono distinti.

**Perché è utile:**
L'adozione di IPv6 è in crescita. I tool di monitoraggio della rete devono tenere traccia separatamente del traffico IPv4 e IPv6 per capire l'avanzamento della migrazione e risolvere i problemi dual-stack.

**Cosa imparerai:**
- Lavorare con il layer IPv6 di Scapy
- Estendere l'enum Protocol
- Modificare la logica dell'analyzer

**Suggerimenti:**
- Guarda `analyzer.py:51-103`, dove extract_packet_info() gestisce il layer IP
- Aggiungi un controllo per `packet.haslayer(IPv6)` accanto al controllo IP
- Potresti voler aggiungere Protocol.IPv6 all'enum oppure tracciarlo come metadato
- Non dimenticare di gestire i casi in cui sono presenti sia IPv4 sia IPv6 (tunnel)

**Verifica che funzioni:**
```bash
# Generate IPv6 traffic
ping6 ::1

# Capture and verify IPv6 shows separately
sudo netanal capture -i lo -c 10 --verbose
```

Dovresti vedere indirizzi IPv6 nell'output e statistiche che tengono traccia di entrambe le versioni del protocollo.

### Sfida 2: ricerca dell'OUI degli indirizzi MAC

**Cosa costruire:**
Aggiungi l'identificazione del produttore a partire dagli indirizzi MAC usando l'OUI (i primi 3 byte). Mostra "Apple Inc." invece del solo "a4:83:e7:..."

**Perché è utile:**
Durante l'incident response, conoscere i produttori dei dispositivi aiuta a identificare cosa c'è sulla rete. "Dispositivo Apple sconosciuto" è più utile di un criptico indirizzo MAC.

**Cosa imparerai:**
- Parsing degli indirizzi MAC
- Ricerche in un database OUI
- Strategie di caching

**Approccio di implementazione:**

1. **Scarica il database OUI** dall'IEEE
   - File: http://standards-oui.ieee.org/oui/oui.txt
   - Esegui il parsing in un dizionario: `{"A483E7": "Apple, Inc."}`

2. **Aggiungi una funzione di ricerca** in un nuovo file `netanal/mac_lookup.py`
   ```python
   def lookup_manufacturer(mac_address: str) -> str:
       oui = mac_address.replace(":", "")[:6].upper()
       return OUI_DATABASE.get(oui, "Unknown")
   ```

3. **Integrala con l'output** in `output.py:print_packet()`
   - Mostra il produttore dopo l'indirizzo MAC
   - Esempio: "a4:83:e7:12:34:56 (Apple Inc.)"

**Verifica che funzioni:**
Cattura il traffico sulla tua rete locale. I produttori dovrebbero comparire per i dispositivi riconosciuti.

### Sfida 3: soglia di alert sulla banda

**Cosa costruire:**
Aggiungi l'argomento da riga di comando `--alert-threshold` che attivi un alert visivo quando la banda supera i MB/s specificati.

**Perché è utile:**
Gli alert in tempo reale durante la cattura permettono agli operatori di reagire immediatamente a picchi di traffico, potenziali attacchi o configurazioni errate.

**Cosa imparerai:**
- Pattern di monitoraggio in tempo reale
- Tecniche di notifica nella console
- Confronti con soglie su dati in streaming

**Suggerimenti:**
- Aggiungilo a CaptureConfig in `models.py:135-145`
- Controlla la soglia in `statistics.py:112-127` quando registri i campioni di banda
- Usa `output.py:print_warning()` o print_error() per gli alert
- Valuta l'uso del Panel di Rich per alert ben visibili

**Verifica che funzioni:**
```bash
# Alert on >1 MB/s
sudo netanal capture -i eth0 --alert-threshold 1.0

# Generate traffic to trigger
curl -O https://speed.hetzner.de/100MB.bin
```

L'alert dovrebbe comparire in rosso quando la soglia viene superata.

## Sfide intermedie

### Sfida 4: tracciamento delle connessioni TCP

**Cosa costruire:**
Traccia i three-way handshake TCP e gli stati delle connessioni (SYN, SYN-ACK, ACK, FIN). Visualizza gli handshake incompleti (potenziali SYN flood) e le connessioni di lunga durata.

**Applicazione nel mondo reale:**
Gli attacchi DDoS di tipo SYN flood inviano pacchetti SYN senza completare l'handshake. Rilevare gli handshake incompleti identifica gli attacchi in corso. Le connessioni di lunga durata potrebbero indicare backdoor persistenti.

**Cosa imparerai:**
- La macchina a stati di TCP
- Tracciamento delle tuple di connessione (src_ip, src_port, dst_ip, dst_port)
- Dati in serie temporali con la durata delle connessioni

**Approccio di implementazione:**

1. **Crea un tracker dello stato delle connessioni** in un nuovo file `netanal/tcp_tracker.py`
   ```python
   @dataclass
   class TCPConnection:
       src_ip: str
       src_port: int
       dst_ip: str
       dst_port: int
       state: str  # SYN_SENT, ESTABLISHED, FIN_WAIT, etc.
       start_time: float
       last_seen: float
   
   class TCPTracker:
       def __init__(self):
           self._connections: dict[tuple, TCPConnection] = {}
       
       def process_packet(self, packet: PacketInfo):
           # Extract TCP flags
           # Update connection state based on flags
           # Track in dict keyed by (src_ip, src_port, dst_ip, dst_port)
   ```

2. **Integralo con lo statistics collector**
   - Aggiungi TCPTracker a StatisticsCollector
   - Chiama tracker.process_packet() in record_packet()

3. **Aggiungi un report** che mostri:
   - Handshake incompleti (SYN senza SYN-ACK)
   - Connessioni in ciascuno stato
   - Connessioni di durata più lunga

**Insidie:**
- Flag TCP in Scapy: `packet[TCP].flags` è una bitmask, controlla i flag specifici
- La direzione della connessione conta: (A→B) è diverso da (B→A) 
- Fai scadere le connessioni vecchie per evitare memory leak

**Punti extra:**
Rileva i port scan tracciando molte connessioni da un IP verso porte diverse con solo il flag SYN.

### Sfida 5: correlazione tra query e risposte DNS

**Cosa costruire:**
Abbina le query DNS alle loro risposte. Traccia la latenza delle query, le query fallite e quali domini vengono interrogati di più.

**Applicazione nel mondo reale:**
Il DNS è spesso il primo indicatore di compromissione. Tracciare i pattern delle query rileva il DNS tunneling, il beaconing dei C2 e i malware con DGA (domain generation algorithm).

**Cosa imparerai:**
- Correlazione richiesta/risposta usando i transaction ID
- La struttura del protocollo DNS (flag query vs risposta)
- Rilevamento di pattern basato sul tempo

**Approccio di implementazione:**

1. **Migliora l'estrazione DNS** in `analyzer.py`
   ```python
   def extract_dns_info(packet: Packet) -> dict | None:
       if not packet.haslayer(DNS):
           return None
       
       dns = packet[DNS]
       return {
           "transaction_id": dns.id,
           "query": dns.qr == 0,  # 0 = query, 1 = response
           "domain": dns.qd.qname.decode() if dns.qd else None,
           "response_code": dns.rcode if dns.qr == 1 else None,
           "answers": [str(rr.rdata) for rr in dns.an] if dns.an else []
       }
   ```

2. **Traccia le query** in una nuova classe DNSTracker
   - Memorizza le query in sospeso indicizzate per transaction ID
   - Abbina le risposte alle query
   - Calcola la latenza: response_time - query_time
   - Traccia i fallimenti (NXDOMAIN, SERVFAIL)

3. **Riporta le statistiche**:
   - Latenza DNS media
   - Domini più interrogati
   - Percentuale di query fallite
   - Pattern sospetti (troppi domini univoci, domini ad alta entropia)

**Test:**
```bash
# Generate DNS traffic
nslookup google.com
nslookup nonexistent-domain-12345.com

# Capture and analyze
sudo netanal capture -i lo -c 20
```

Dovrebbe mostrare sia le query riuscite sia quelle fallite, con le misure di latenza.

### Sfida 6: istogramma della distribuzione delle dimensioni dei pacchetti

**Cosa costruire:**
Traccia la distribuzione delle dimensioni dei pacchetti e genera un grafico a istogramma. Utile per rilevare pattern di traffico insoliti.

**Applicazione nel mondo reale:**
Le distribuzioni delle dimensioni dei pacchetti rivelano i tipi di applicazione. Il VoIP ha pacchetti piccoli e costanti. I trasferimenti di file hanno pacchetti grandi. Gli attacchi DDoS usano spesso dimensioni specifiche (minuscole per l'amplificazione, grandi per gli attacchi volumetrici).

**Cosa imparerai:**
- Generazione di istogrammi con bin
- Analisi delle distribuzioni statistiche
- Visualizzazioni matplotlib personalizzate

**Approccio di implementazione:**

1. **Aggiungi il tracciamento delle dimensioni** a StatisticsCollector
   ```python
   # In statistics.py
   def __init__(self):
       # ... existing init
       self._size_buckets: dict[int, int] = defaultdict(int)
       self._size_bins = [64, 128, 256, 512, 1024, 1500, 9000]
   
   def _get_size_bucket(self, size: int) -> int:
       for bin_size in self._size_bins:
           if size <= bin_size:
               return bin_size
       return self._size_bins[-1]
   ```

2. **Crea il grafico a istogramma** in `visualization.py`
   ```python
   def create_size_histogram(
       stats: CaptureStatistics,
       title: str = "Packet Size Distribution"
   ) -> Figure:
       # Create bar chart with size bins
       # X-axis: packet size ranges
       # Y-axis: count or percentage
   ```

3. **Aggiungilo alla CLI** nel comando `main.py:chart()`
   - Nuovo tipo di grafico: `--type size-histogram`

**Verifica che funzioni:**
Mescola tipi di traffico diversi e osserva le differenze nella distribuzione:
```bash
# HTTP traffic (varied sizes)
curl http://example.com

# DNS traffic (small consistent sizes)  
for i in {1..10}; do nslookup google.com; done

# Analyze
sudo netanal capture -i lo -c 100
netanal chart captured.pcap --type size-histogram
```

## Sfide avanzate

### Sfida 7: mappatura della geolocalizzazione degli IP

**Cosa costruire:**
Aggiungi una ricerca GeoIP per mostrare paese/città degli IP esterni. Visualizza i top talker con la posizione geografica e genera una visualizzazione su mappa del mondo.

**Perché è difficile:**
Richiede un database GeoIP esterno, la trasformazione delle coordinate per il disegno della mappa e la gestione dei casi limite (IP privati, localhost, posizioni sconosciute).

**Cosa imparerai:**
- Integrazione di un database GeoIP (MaxMind o IP2Location)
- Visualizzazione di dati geografici
- Trasformazioni tra sistemi di coordinate
- Gestione e aggiornamento dei file di database

**Modifiche architetturali necessarie:**

```
New Components:
- netanal/geoip.py → GeoIP lookup class
- Matplotlib basemap → World map plotting
- Database file → GeoLite2 or similar

Modified:
- EndpointStats → Add location fields
- statistics.py → Call GeoIP on new endpoints
- visualization.py → New map chart function
```

**Passi di implementazione:**

1. **Fase di ricerca**
   - Leggi la documentazione di MaxMind GeoLite2
   - Comprendi il formato del database (MMDB)
   - Dai un'occhiata alla libreria Python geoip2

2. **Fase di design**
   - Decidi: database incluso o scaricato dall'utente?
   - Valuta: mettere in cache le ricerche per evitare accessi ripetuti al database
   - Pianifica: cosa fare con gli IP privati (non cercarli)

3. **Fase di implementazione**
   - Inizia con una semplice ricerca del paese
   - Aggiungi città e latitudine/longitudine
   - Crea uno scatter plot su mappa del mondo con matplotlib
   - Dimensiona i punti in base al volume di traffico

4. **Fase di test**
   - Testa con IP pubblici (8.8.8.8, 1.1.1.1)
   - Verifica che gli IP privati saltino la ricerca
   - Controlla il rendering della mappa con traffico reale

**Insidie:**
- MaxMind richiede la registrazione di un account gratuito per scaricare il database
- Gli IP privati (192.168.x.x, 10.x.x.x) dovrebbero saltare la ricerca
- Localhost (127.0.0.1) non ha una posizione geografica
- I file di database sono grandi (~100MB), valuta di aggiungerli al gitignore

**Risorse:**
- MaxMind GeoLite2 (https://dev.maxmind.com/geoip/geolite2-free-geolocation-data)
- Documentazione della libreria Python geoip2
- Tutorial di matplotlib Basemap

### Sfida 8: estrazione dei certificati SSL/TLS

**Cosa costruire:**
Estrai e visualizza le informazioni dei certificati SSL/TLS dagli handshake HTTPS. Mostra il soggetto del certificato, l'emittente, le date di validità e la catena di certificati.

**Perché è difficile:**
Richiede il parsing del protocollo di handshake TLS, la gestione di più versioni di TLS, l'estrazione dei certificati X.509 e la gestione degli handshake frammentati.

**Cosa imparerai:**
- La struttura del protocollo di handshake TLS
- Il formato dei certificati X.509
- L'uso del layer TLS di Scapy
- I concetti di validazione della catena di certificati

**Passi di implementazione:**

**Fase 1: estrazione di base dei certificati** (8-10 ore)
```python
# In analyzer.py, add TLS extraction
from scapy.layers.tls.record import TLS
from scapy.layers.tls.handshake import TLSServerHello, TLSCertificate

def extract_tls_info(packet: Packet) -> dict | None:
    if not packet.haslayer(TLS):
        return None
    
    # Extract certificates from TLS handshake
    # Parse X.509 certificate fields
    # Return dict with subject, issuer, dates
```

**Fase 2: gestione della catena di certificati** (10-12 ore)
- Gestisci più certificati nella catena
- Traccia i certificati root, intermedi e leaf
- Mostra la gerarchia dei certificati

**Fase 3: integrazione e visualizzazione** (6-8 ore)
- Aggiungi al tracciamento delle statistiche
- Crea una tabella di report dei certificati
- Segnala i certificati scaduti o self-signed

**Fase 4: funzionalità avanzate** (8-10 ore)
- Estrazione dello SNI (Server Name Indication)
- Rilevamento delle cipher suite
- Tracciamento della versione TLS
- Alert sui cipher deboli

**Strategia di test:**
```bash
# Generate HTTPS traffic
curl https://google.com
curl https://expired.badssl.com  # Test expired cert

# Capture and analyze
sudo netanal capture -i eth0 --filter "tcp port 443" -c 50
```

**Sfide note:**

1. **Frammentazione TLS**
   - Problema: i certificati si estendono su più segmenti TCP
   - Suggerimento: potrebbe servire il riassemblaggio degli stream TCP (la Sfida 4 aiuta)

2. **Versioni di TLS**
   - Problema: TLS 1.0, 1.1, 1.2 e 1.3 hanno strutture diverse
   - Suggerimento: usa il layer TLS di Scapy, che astrae le versioni

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Estrarre il soggetto e l'emittente del certificato
- [ ] Mostrare le date di validità (not before, not after)
- [ ] Gestire le catene di certificati
- [ ] Visualizzare l'hostname dello SNI
- [ ] Segnalare i certificati scaduti
- [ ] Funzionare con TLS 1.2 e 1.3

### Sfida 9: rilevamento delle anomalie in tempo reale

**Cosa costruire:**
Implementa un rilevamento statistico delle anomalie che generi alert su pattern di traffico insoliti in tempo reale durante la cattura. Rileva picchi di banda, rapporti insoliti tra protocolli e nuove connessioni verso porte strane.

**Tempo stimato:**
2-3 settimane per l'implementazione completa

**Prerequisiti:**
Completa prima la Sfida 4 (tracciamento TCP) e la Sfida 6 (distribuzione delle dimensioni), perché questa si basa su quei pattern.

**Cosa imparerai:**
- Statistical process control
- Calcolo dello Z-score per il rilevamento degli outlier  
- Medie mobili e deviazioni standard
- Generazione di alert in tempo reale
- Modalità di apprendimento della baseline vs modalità di rilevamento delle anomalie

**Pianificare questa funzionalità:**

Prima di scrivere codice, rifletti su:
- Come stabilisci una baseline? (Serve un periodo in "modalità di apprendimento")
- Quali metriche indicano anomalie? (Banda, rapporto tra protocolli, tasso di connessioni)
- Come eviti l'alert fatigue? (Calibrazione delle soglie, periodi di cooldown)
- Qual è la tua tolleranza ai falsi positivi? (Soglie dello Z-score)

**Architettura ad alto livello:**

```
┌─────────────────────────────────────┐
│     Baseline Establishment          │
│   (Learning Mode: 5-10 minutes)     │
│                                     │
│  - Collect normal traffic samples   │
│  - Calculate mean & std dev         │
│  - Store baseline metrics           │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│      Real-Time Detection            │
│   (Detection Mode: Continuous)      │
│                                     │
│  - Compare current vs baseline      │
│  - Calculate Z-scores               │
│  - Trigger alerts on outliers       │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│         Alert Handler               │
│                                     │
│  - Log anomaly details              │
│  - Display terminal alert           │
│  - Optional: webhook notification   │
└─────────────────────────────────────┘
```

**Fasi di implementazione:**

**Fase 1: raccolta della baseline** (12-16 ore)
Crea un nuovo file `netanal/anomaly.py`:
```python
@dataclass
class BaselineMetrics:
    bandwidth_mean: float
    bandwidth_stddev: float
    protocol_ratios: dict[Protocol, float]
    connection_rate_mean: float
    connection_rate_stddev: float
    
class AnomalyDetector:
    def __init__(self, learning_period: int = 300):  # 5 minutes
        self._learning_mode = True
        self._samples: list[float] = []
        # ... more init
    
    def learn(self, stats: CaptureStatistics):
        # Collect samples during learning period
        # Calculate statistics
        pass
    
    def detect(self, stats: CaptureStatistics) -> list[Anomaly]:
        # Compare against baseline
        # Return list of detected anomalies
        pass
```

**Fase 2: rilevamento statistico** (16-20 ore)
Implementa il calcolo dello Z-score:
```python
def calculate_zscore(value: float, mean: float, stddev: float) -> float:
    if stddev == 0:
        return 0
    return (value - mean) / stddev

def is_anomaly(zscore: float, threshold: float = 3.0) -> bool:
    return abs(zscore) > threshold
```

Traccia più metriche:
- Banda (byte/secondo)
- Tasso di pacchetti (pacchetti/secondo)
- Distribuzione dei protocolli (% TCP vs UDP vs altro)
- Tasso di connessioni (nuove connessioni/secondo)
- Porte di destinazione univoche

**Fase 3: generazione degli alert** (8-10 ore)
Crea i tipi di alert:
```python
@dataclass
class Anomaly:
    timestamp: float
    metric: str  # "bandwidth", "protocol_ratio", etc.
    value: float
    expected: float
    severity: str  # "low", "medium", "high"
    description: str

def format_alert(anomaly: Anomaly) -> str:
    # Create human-readable alert message
    # Include what's unusual and by how much
```

**Fase 4: integrazione** (6-8 ore)
- Aggiungi a CaptureEngine
- Chiama il detector durante il campionamento della banda
- Visualizza gli alert in tempo reale
- Registra le anomalie su file

**Strategia di test:**

Testa con anomalie sintetiche:
```python
# Test 1: Bandwidth spike
# Baseline: normal traffic
# Inject: large file download
# Expected: bandwidth anomaly alert

# Test 2: Protocol shift
# Baseline: 80% TCP, 20% UDP
# Inject: UDP flood
# Expected: protocol ratio anomaly

# Test 3: Port scan
# Baseline: connections to normal ports
# Inject: nmap scan
# Expected: high connection rate + unusual ports
```

**Sfide note:**

1. **Problema dell'avvio a freddo**
   - Problema: nessuna baseline alla prima esecuzione
   - Suggerimento: salva su disco le baseline apprese e caricale nelle esecuzioni successive

2. **Concept drift**
   - Problema: il comportamento della rete cambia nel tempo (nuovi servizi, ora del giorno)
   - Suggerimento: implementa baseline adattive che si aggiornano lentamente

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Apprendere la baseline in un periodo di 5-10 minuti
- [ ] Rilevare picchi di banda (>3 deviazioni standard)
- [ ] Rilevare distribuzioni dei protocolli insolite
- [ ] Rilevare anomalie nel tasso di connessioni
- [ ] Visualizzare gli alert in tempo reale
- [ ] Includere un punteggio di confidenza/gravità
- [ ] Evitare falsi positivi eccessivi (<10% durante il traffico normale)
- [ ] Salvare e caricare le baseline apprese

## Sfide sulle prestazioni

### Sfida: gestire traffico a 10 Gbps

**L'obiettivo:**
Ottimizzare l'engine di cattura per gestire traffico a 10 gigabit al secondo senza perdere pacchetti.

**Collo di bottiglia attuale:**
A 10 Gbps, stai elaborando circa 10 milioni di pacchetti al secondo. Il codice attuale perde pacchetti perché:
- La coda si riempie (il buffer da 10K è troppo piccolo)
- Contesa sul lock delle statistiche (milioni di acquisizioni del lock al secondo)
- Il GIL di Python limita il parallelismo
- Le allocazioni di memoria rallentano la garbage collection

**Approcci di ottimizzazione:**

**Approccio 1: più thread di elaborazione**
- Come: usa un thread pool con N worker che prelevano dalla coda
- Guadagno: throughput di elaborazione N volte superiore
- Compromesso: servono statistiche lock-free o un'aggregazione per thread

**Approccio 2: statistiche lock-free**
- Come: usa contatori per thread con aggregazione periodica
- Guadagno: elimina il collo di bottiglia della contesa sul lock
- Compromesso: codice più complesso, consistenza eventuale

**Approccio 3: campionamento**
- Come: elabora 1 pacchetto ogni N
- Guadagno: riduce il carico della CPU in proporzione
- Compromesso: approssimazione statistica, potrebbe perdere eventi rari

**Approccio 4: aggregazione BPF**
- Come: usa eBPF per aggregare nel kernel prima dello userspace
- Guadagno: enorme miglioramento delle prestazioni
- Compromesso: richiede Linux e competenze di programmazione eBPF

**Fai il benchmark:**
```bash
# Generate high-speed traffic with iperf3
iperf3 -s &  # Server
iperf3 -c localhost -b 10G  # Client

# Monitor dropped packets
sudo netanal capture -i lo --verbose | grep "Dropped"
```

Metriche obiettivo:
- Pacchetti scartati: <1% a 10 Gbps
- Uso della CPU: <80% su un sistema a 8 core
- Memoria: <4 GB

## Sfide di sicurezza

### Sfida: aggiungi la cifratura dei PCAP

**Cosa implementare:**
Cifra i file PCAP catturati per proteggere i dati di rete sensibili. Aggiungi un flag `--encrypt` che chieda una password e cifri l'output usando AES-256.

**Threat model:**
Questo protegge da:
- Accessi non autorizzati ai file catturati
- Divulgazione accidentale di traffico sensibile
- Requisiti di compliance (HIPAA, PCI-DSS)

**Implementazione:**
Usa la libreria cryptography:
```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2

def encrypt_pcap(filepath: Path, password: str):
    # Derive key from password using PBKDF2
    # Encrypt file with Fernet (AES-128-CBC + HMAC)
    # Write encrypted output with .enc extension
```

**Testare la sicurezza:**
- Prova ad aprire il file cifrato con Wireshark (dovrebbe fallire)
- Verifica che per la decifratura sia richiesta la password
- Controlla che le password deboli vengano rifiutate

### Sfida: supera la checklist del CIS Benchmark

**L'obiettivo:**
Rendere questo progetto conforme ai controlli CIS (Center for Internet Security) pertinenti per i tool di monitoraggio della rete.

**Lacune attuali:**

**Lacuna 1: nessun audit logging**
- Manca: chi ha eseguito le catture, quando, quali filtri sono stati usati
- Rimedio: aggiungi un audit log in ~/.netanal/audit.log

**Lacuna 2: nessuna documentazione sulla sanificazione dell'input**
- Manca: una documentazione chiara degli input validati
- Rimedio: documenta tutta la validazione in security.md

**Lacuna 3: nessun limite sulle risorse**
- Manca: la memoria è illimitata in caso di configurazione errata
- Rimedio: aggiungi limiti rigidi configurabili

**Lacuna 4: privilege escalation senza logging**
- Manca: nessuna registrazione di quando vengono usati i privilegi di root
- Rimedio: registra tutte le operazioni con privilegi elevati

**Lacuna 5: nessun default sicuro**
- Manca: i default consentono la modalità promiscua e qualsiasi interfaccia
- Rimedio: richiedi la specifica esplicita dell'interfaccia

Ogni lacuna corrisponde a specifici controlli CIS. Implementa le correzioni e documenta la conformità.

## Combina a piacere

Combina le funzionalità per progetti più grandi:

**Idea di progetto 1: dashboard di sicurezza della rete**
- Combina la Sfida 4 (tracciamento TCP) + la Sfida 9 (rilevamento delle anomalie) + la Sfida 7 (geolocalizzazione)
- Aggiungi una web UI che mostri una mappa in tempo reale delle connessioni con le anomalie evidenziate
- Risultato: una dashboard SOC visiva per il monitoraggio della rete

**Idea di progetto 2: monitor di sicurezza DNS**
- Combina la Sfida 5 (tracciamento DNS) + la Sfida 9 (rilevamento delle anomalie)
- Aggiungi il rilevamento dei DGA (analisi dell'entropia dei domini)
- Risultato: un tool di security monitoring specifico per il DNS

**Idea di progetto 3: analisi del traffico cifrato**
- Combina la Sfida 8 (estrazione TLS) + la Sfida 4 (tracciamento TCP)
- Aggiungi il fingerprinting JA3 (identificazione dei client TLS)
- Risultato: rilevare malware tramite i pattern TLS senza decifrare

## Sfide di integrazione nel mondo reale

### Integrazione con un SIEM (Splunk/ELK)

**L'obiettivo:**
Inviare le statistiche di cattura a un SIEM per il logging centralizzato e la correlazione.

**Cosa ti servirà:**
- Un'istanza SIEM (Splunk gratuito o stack ELK)
- HTTP Event Collector (Splunk) o un endpoint Logstash (ELK)
- Formattazione JSON per gli eventi

**Piano di implementazione:**
1. Crea `netanal/siem.py` con un client SIEM
2. Aggiungi l'argomento CLI `--siem-url`
3. Serializza le statistiche in JSON
4. Invia con POST all'endpoint del SIEM ogni N secondi
5. Gestisci con eleganza i fallimenti di connessione

**Attenzione a:**
- Rate limit sull'ingestion del SIEM
- Autenticazione (API key)
- Guasti di rete (metti in coda gli eventi non inviati)
- Privacy dei dati (non inviare i payload dei pacchetti)

**Checklist per la produzione:**
- [ ] TLS per la connessione al SIEM
- [ ] API key da variabile d'ambiente
- [ ] Logica di retry con exponential backoff
- [ ] Coda locale per il funzionamento offline

### Deploy su Kubernetes

**L'obiettivo:**
Impacchettare il tool come container ed eseguirne il deploy come DaemonSet per il monitoraggio della rete a livello di intero cluster.

**Cosa imparerai:**
- Containerizzazione con Docker
- Networking di Kubernetes
- Gestione dei privilegi nei container
- Raccolta distribuita dei dati

**Passi:**

1. **Crea il Dockerfile**
   ```dockerfile
   FROM python:3.14-slim
   RUN apt-get update && apt-get install -y libpcap-dev
   COPY . /app
   WORKDIR /app
   RUN pip install -e .
   CMD ["netanal", "capture"]
   ```

2. **Esegui la build del container**
   ```bash
   docker build -t netanal:latest .
   ```

3. **Crea il manifest Kubernetes**
   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: netanal
   spec:
     template:
       spec:
         hostNetwork: true  # Required for packet capture
         containers:
         - name: netanal
           image: netanal:latest
           securityContext:
             capabilities:
               add: ["NET_RAW", "NET_ADMIN"]
   ```

4. **Esegui il deploy**
   ```bash
   kubectl apply -f netanal-daemonset.yaml
   ```

**Checklist per la produzione:**
- [ ] Utente non root nel container
- [ ] Solo le capability necessarie (NET_RAW)
- [ ] Limiti sulle risorse (CPU/memoria)
- [ ] Aggregazione dei log verso un collector centrale
- [ ] Endpoint di health check

## Ottenere aiuto

Bloccato su una sfida?

1. **Fai il debug in modo sistematico**
   - Cosa ti aspettavi che succedesse?
   - Cosa è successo davvero?
   - Qual è il caso di test più piccolo che lo riproduce?
   - Puoi aggiungere delle istruzioni print per circoscrivere il problema?

2. **Leggi il codice esistente**
   - La Sfida 4 (tracciamento TCP) è simile alla Sfida 5 (tracciamento DNS) - usa gli stessi pattern
   - Il codice di visualizzazione gestisce già più tipi di grafico - estendilo

3. **Cerca problemi simili**
   - "Scapy TCP flags" → trova esempi di parsing dei flag
   - "Python statistical anomaly detection" → trova algoritmi basati sullo Z-score
   - "matplotlib world map" → trova esempi di basemap

4. **Chiedi aiuto**
   - Descrivi cosa stai cercando di costruire
   - Mostra cosa hai provato
   - Spiega cosa è andato storto
   - Includi frammenti di codice pertinenti e messaggi di errore

Non limitarti a incollare "non funziona" con uno stack trace. Spiega come hai capito il problema.

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

**Facili:**
- [ ] Visualizzazione del supporto IPv6
- [ ] Ricerca dell'OUI degli indirizzi MAC
- [ ] Soglia di alert sulla banda

**Intermedie:**
- [ ] Tracciamento delle connessioni TCP
- [ ] Correlazione tra query e risposte DNS
- [ ] Istogramma della distribuzione delle dimensioni dei pacchetti

**Avanzate:**
- [ ] Mappatura della geolocalizzazione degli IP
- [ ] Estrazione dei certificati SSL/TLS
- [ ] Rilevamento delle anomalie in tempo reale

**Prestazioni:**
- [ ] Gestire traffico a 10 Gbps

**Sicurezza:**
- [ ] Cifratura dei PCAP
- [ ] Conformità al CIS Benchmark

**Integrazione:**
- [ ] Integrazione con un SIEM
- [ ] Deploy su Kubernetes

Le hai completate tutte? Hai padroneggiato l'analisi dei pacchetti di rete. Valuta di contribuire con le tue soluzioni al progetto o di costruire qualcosa di nuovo, come un IDS o un tool di network forensics.

## Mettiti ancora più alla prova

### Costruisci qualcosa di nuovo

Usa i concetti che hai imparato qui per costruire:
- **Analizzatore di pacchetti wireless** - Estendilo per monitorare i frame 802.11 e rilevare gli attacchi di deauth
- **Analizzatore di protocolli industriali** - Aggiungi il supporto ai protocolli Modbus/SCADA
- **Tool di network forensics** - Ricostruisci i trasferimenti di file, estrai artefatti dai pcap

### Studia implementazioni reali

Confronta la tua implementazione con i tool di produzione:
- **Wireshark/tshark** - Osserva i display filter rispetto a BPF e i dissector dei protocolli
- **Zeek (ex Bro)** - Studia l'architettura event-driven e l'estensibilità tramite script
- **Suricata** - Esamina l'approccio multi-threading e l'accelerazione su GPU

Leggi il loro codice, capisci i loro compromessi, ruba le loro idee migliori.

### Scrivine

Documenta la tua estensione:
- Un post sul blog che spieghi "Come ho aggiunto il rilevamento delle anomalie a un analizzatore di pacchetti"
- Un tutorial perché altri implementino la tua funzionalità
- Un confronto: il tuo approccio rispetto a come lo fa Wireshark

Insegnare agli altri è il modo migliore per verificare di aver capito.
