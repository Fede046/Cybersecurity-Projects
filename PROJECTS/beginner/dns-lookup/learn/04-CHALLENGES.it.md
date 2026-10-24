# Sfide ed estensioni

## Sfide per principianti

### Sfida 1: aggiungere la validazione DNSSEC

**Obiettivo:** verificare le firme DNSSEC sulle risposte DNS.

**Stato attuale:** il tool recupera i record ma non valida le firme. La riga `resolver.py:200-201` si limita a recuperare i dati senza verifica crittografica.

**Attività di implementazione:**
1. Abilita DNSSEC nel resolver (`resolver.py:91-107`)
```python
   resolver = dns.asyncresolver.Resolver()
   resolver.use_edns(0, dns.flags.DO, 4096)  # Request DNSSEC records
```

2. Controlla i record RRSIG nelle risposte
3. Valida le firme rispetto ai record DNSKEY
4. Percorri la catena di fiducia dalla root fino al dominio di destinazione

**Risorse:**
- RFC 4033, 4034, 4035 (specifiche DNSSEC)
- Esempi di validazione DNSSEC con dnspython
- Testa con `dnssec-tools.org` (dominio firmato)

**Impatto sulla sicurezza:** impedisce di accettare risposte DNS contraffatte. Fondamentale per ambienti ad alta sicurezza.

### Sfida 2: implementare il logging delle query DNS

**Obiettivo:** registrare tutte le query in un file per l'analisi.

**Cosa registrare:**
- Timestamp
- Dominio interrogato
- Tipi di record richiesti
- Server DNS utilizzato
- Tempo di risposta
- Numero di record restituiti

**Implementazione:**
1. Crea un logger in `resolver.py:213-250`
2. Registra prima della query: `logging.info(f"Querying {domain} for {record_types}")`
3. Registra dopo la query: `logging.info(f"Got {len(result.records)} records in {result.query_time_ms}ms")`
4. Aggiungi un flag alla CLI: `--log-file queries.log`

**Idee per l'analisi:**
- Quali domini impiegano più tempo a essere risolti?
- Quali tipi di record falliscono più spesso?
- Analisi delle serie temporali dei tempi di risposta

**Applicazione di sicurezza:** rilevare la ricognizione analizzando i pattern delle query.

### Sfida 3: aggiungere il formato di output CSV

**Obiettivo:** esportare i risultati in CSV per l'analisi con fogli di calcolo.

**Stato attuale:** l'output JSON esiste (`output.py:379-410`), ma il CSV sarebbe più utile per alcuni workflow.

**Implementazione:**
1. Aggiungi la funzione `results_to_csv()` in `output.py`
2. Appiattisci la struttura annidata:
```
   domain,record_type,value,ttl,priority,query_time_ms
   example.com,A,93.184.216.34,86400,,45.2
   example.com,MX,mail.example.com,3600,10,45.2
```
3. Gestisci più record per dominio
4. Aggiungi il flag `--csv` ai comandi

**Caso d'uso:** importare in Excel/Google Sheets per analisi, ordinamento, filtraggio.

## Sfide intermedie

### Sfida 4: enumerazione dei sottodomini

**Obiettivo:** scoprire sottodomini attraverso varie tecniche.

**Tecniche da implementare:**

**1. Brute force** (`cli.py:266-350` come punto di partenza)
```python
common_subdomains = ['www', 'mail', 'ftp', 'admin', 'test', 'dev', 'staging']
for sub in common_subdomains:
    fqdn = f"{sub}.{domain}"
    result = await lookup(fqdn, [RecordType.A])
    if result.records:
        print(f"Found: {fqdn}")
```

**2. Log di Certificate Transparency**
Interroga l'API di crt.sh per trovare domini nei certificati SSL:
```python
import requests
response = requests.get(f"https://crt.sh/?q=%.{domain}&output=json")
```

**3. DNS Zone Transfer** (AXFR)
```python
try:
    zone = dns.zone.from_xfr(dns.query.xfr(nameserver, domain))
    for name in zone:
        print(f"{name}.{domain}")
except:
    print("Zone transfer refused")
```

**Nota di sicurezza:** gli zone transfer sono spesso bloccati. Testa solo su domini di tua proprietà.

**Mondo reale:** l'enumerazione dei sottodomini è il primo passo della ricognizione (MITRE T1590.002).

### Sfida 5: monitoraggio DNS e alerting

**Obiettivo:** monitorare continuamente i domini e inviare alert in caso di modifiche.

**Architettura:**
```
Periodic Queries (every 5 min)
        ↓
Compare to Previous State
        ↓
Detect Changes
        ↓
Send Alerts (email/Slack/webhook)
```

**Cosa monitorare:**
- Modifiche ai record A/AAAA (infrastruttura spostata)
- Modifiche ai record MX (instradamento email modificato)
- Modifiche ai record NS (provider DNS cambiato)
- Record nuovi/rimossi

**Implementazione:**
1. Memorizza lo stato precedente in un database SQLite
2. Esegui le query a intervalli programmati (usa la libreria `schedule` o cron)
3. Confronta lo stato attuale con quello precedente
4. Genera alert sulle differenze

**Applicazione di sicurezza:** rilevare tentativi di DNS hijacking. Se i record NS cambiano all'improvviso, qualcuno potrebbe aver compromesso l'account presso il registrar.

**Incidente reale:** MyEtherWallet (2018) - un BGP hijack ha modificato il DNS per puntare a un sito di phishing. Il monitoraggio lo avrebbe rilevato.

### Sfida 6: analisi delle prestazioni DNS

**Obiettivo:** misurare e visualizzare le prestazioni dei resolver DNS.

**Metriche da tracciare:**
- Percentili di latenza delle query (p50, p95, p99)
- Tasso di successo per tipo di record
- Frequenza dei timeout
- Confronto tra resolver (8.8.8.8 vs 1.1.1.1 vs ISP)

**Implementazione:**
1. Estendi `DNSResult` con più metriche
2. Interroga gli stessi domini da più resolver
3. Memorizza i risultati in un database time-series (InfluxDB)
4. Visualizza con Grafana

**Da aggiungere a `resolver.py`:**
```python
@dataclass
class DNSResult:
    # ... existing fields ...
    resolver_ip: str | None = None
    tcp_fallback: bool = False
    truncated: bool = False
```

**Domande di analisi:**
- Quale resolver è il più veloce per la tua posizione?
- Le prestazioni variano in base all'ora del giorno?
- Quali domini hanno i tempi di risoluzione più lunghi?

## Sfide avanzate

### Sfida 7: rilevamento del DNS tunneling

**Obiettivo:** identificare il DNS tunneling usato per l'esfiltrazione di dati.

**Caratteristiche del DNS tunneling:**
- Volume elevato di query verso un singolo dominio
- Query/risposte con record TXT di grandi dimensioni
- Lunghezze insolite dei sottodomini
- Sottodomini codificati in Base64/hex
- Intervalli di query regolari

**Implementazione:**

**1. Analisi dell'entropia**
```python
import math
from collections import Counter

def calculate_entropy(subdomain: str) -> float:
    """High entropy suggests encoded data"""
    counter = Counter(subdomain)
    length = len(subdomain)
    return -sum((count/length) * math.log2(count/length) 
                for count in counter.values())

# Legitimate: www.example.com (low entropy)
# Tunneling: aGVsbG8=.attacker.com (high entropy)
```

**2. Rilevamento del volume di query**
```python
# Track queries per domain
query_counts = defaultdict(int)

for domain in queries:
    query_counts[domain] += 1
    if query_counts[domain] > threshold:
        alert(f"Possible tunneling: {domain}")
```

**3. Analisi della lunghezza dei sottodomini**
```python
if len(subdomain) > 63:  # DNS label limit
    alert("Suspicious long subdomain")
```

**Malware reali:** il worm Houdini, Feederbot e DNScat2 usano tutti il DNS tunneling.

**Riferimenti:**
- CWE-406: Insufficient Control of Network Message Volume
- MITRE T1071.004: Application Layer Protocol: DNS

### Sfida 8: database di passive DNS

**Obiettivo:** costruire un database storico delle risoluzioni DNS.

**Architettura:**
```
DNS Queries (from monitoring/batch)
        ↓
Store in Database
        ↓
Query Historical Data
```

**Schema:**
```sql
CREATE TABLE dns_records (
    id INTEGER PRIMARY KEY,
    domain TEXT,
    record_type TEXT,
    value TEXT,
    ttl INTEGER,
    first_seen TIMESTAMP,
    last_seen TIMESTAMP,
    seen_count INTEGER
);

CREATE INDEX idx_domain ON dns_records(domain);
CREATE INDEX idx_value ON dns_records(value);
```

**Query da supportare:**
- Quando example.com si è risolto per la prima volta in 1.2.3.4?
- Quali domini si sono risolti in questo IP?
- Con quale frequenza questo dominio cambia IP?

**Applicazione di sicurezza:** tracciare l'infrastruttura C2 dei malware. I domini malevoli cambiano spesso IP frequentemente.

**Esempi commerciali:** VirusTotal, PassiveTotal, Farsight DNSDB.

### Sfida 9: analisi in tempo reale di flussi di query DNS

**Obiettivo:** analizzare flussi di query DNS live (dai log di router/firewall).

**Sorgenti di input:**
- File pcap da catture di rete
- Syslog dai server DNS
- Dati Netflow/IPFIX
- Log delle query di DNS cloud (AWS Route53, Google Cloud DNS)

**Pattern di rilevamento:**

**1. DGA (Domain Generation Algorithm)**
```python
def is_dga(domain: str) -> bool:
    """DGA domains are often random-looking"""
    labels = domain.split('.')
    if len(labels) < 2:
        return False
    
    sld = labels[-2]  # second-level domain
    
    # High consonant ratio
    consonants = sum(1 for c in sld if c in 'bcdfghjklmnpqrstvwxyz')
    if consonants / len(sld) > 0.7:
        return True
    
    # No dictionary words
    # Low vowel ratio
    # High entropy
    
    return False
```

**2. Rilevamento del fast flux**
Molti IP per un solo dominio, che cambiano frequentemente:
```python
# Track A records over time
if len(unique_ips_for_domain) > 10 and avg_ttl < 300:
    alert("Possible fast flux")
```

**3. C2 beaconing**
Query periodiche regolari:
```python
# Calculate query intervals
intervals = [queries[i+1].time - queries[i].time 
             for i in range(len(queries)-1)]

# If intervals are suspiciously regular
if std_dev(intervals) < threshold:
    alert("Possible C2 beaconing")
```

**Tool da studiare:** Zeek (precedentemente Bro), moduli di analisi DNS di Suricata.

### Sfida 10: DNS firewall / sinkhole

**Obiettivo:** bloccare i domini malevoli fornendo risposte fittizie.

**Architettura:**
```
Client Query
    ↓
Your DNS Server
    ↓
Check Blocklist
    ↓
If Blocked: Return 0.0.0.0
If Allowed: Forward to Upstream Resolver
```

**Implementazione:**

**1. Server DNS di base** (usa la libreria `dnslib`)
```python
from dnslib import DNSRecord, RR, QTYPE, A
from dnslib.server import DNSServer

class BlocklistResolver:
    def __init__(self, blocklist):
        self.blocklist = set(blocklist)
    
    def resolve(self, request, handler):
        qname = str(request.q.qname)
        
        if qname.rstrip('.') in self.blocklist:
            # Return sinkhole IP
            reply = request.reply()
            reply.add_answer(RR(qname, QTYPE.A, rdata=A("0.0.0.0")))
            return reply
        else:
            # Forward to real resolver
            return proxy_request(request)
```

**2. Sorgenti di blocklist**
- Liste di domini di malware (abuse.ch)
- Feed di phishing (PhishTank)
- Liste di ad/tracker (EasyList)
- Feed di threat intelligence

**3. Logging e analytics**
- Traccia i tentativi di query bloccati
- Identifica le macchine infette
- Misura l'efficacia

**Prodotti reali:** Pi-hole, AdGuard Home, Cisco Umbrella.

## Sfide per esperti

### Sfida 11: rilevamento di attacchi di DNS amplification

**Obiettivo:** rilevare quando il tuo server DNS viene usato per attacchi di amplificazione.

**Pattern di attacco:**
1. L'attaccante invia query con IP sorgente falsificato (l'indirizzo della vittima)
2. Il tuo server invia risposte di grandi dimensioni alla vittima
3. La vittima viene inondata

**Segnali di rilevamento:**
```python
# 1. High volume from single IP
queries_per_ip = defaultdict(int)

# 2. Queries for large record types
if record_type in [RecordType.TXT, RecordType.ANY]:
    suspicious_count += 1

# 3. Responses much larger than queries
if response_size / query_size > 50:
    alert("High amplification factor")

# 4. No follow-up queries (victim doesn't actually want data)
if client_ip in one_shot_queries:
    alert("Possible spoofed source")
```

**Mitigazione:**
- Rate limiting delle query per IP
- Disabilitare le query ANY (RFC 8482)
- Response rate limiting (RRL)
- Filtraggio BCP38 (impedisce IP sorgente falsificati)

### Sfida 12: tipi di record DNS personalizzati

**Obiettivo:** aggiungere il supporto per tipi di record più recenti/specializzati.

**Tipi di record da aggiungere:**

**CAA (Certification Authority Authorization)**
```python
# In resolver.py:24-33
class RecordType(StrEnum):
    # ... existing types ...
    CAA = "CAA"  # RFC 8659
```

Specifica quali CA possono emettere certificati per il dominio. Critico per la sicurezza.

**SSHFP (SSH Fingerprint)**
```python
SSHFP = "SSHFP"  # RFC 4255
```

Pubblica nel DNS i fingerprint delle chiavi dei server SSH per la verifica.

**TLSA (DANE/TLSA)**
```python
TLSA = "TLSA"  # RFC 6698
```

DNS-based Authentication of Named Entities. Pubblica gli hash dei certificati TLS.

**DNSKEY, DS, RRSIG** (record DNSSEC)
Per la sfida 1 (validazione DNSSEC).

### Sfida 13: analisi geografica del DNS

**Obiettivo:** determinare dove si trovano i server DNS e analizzarne la distribuzione geografica.

**Implementazione:**

**1. GeoIP lookup**
```python
import geoip2.database

reader = geoip2.database.Reader('GeoLite2-City.mmdb')

for nameserver_ip in result.nameserver_ips:
    response = reader.city(nameserver_ip)
    print(f"{nameserver_ip}: {response.city.name}, {response.country.name}")
```

**2. Geolocalizzazione basata sulla latenza**
Esegui ping da più punti di osservazione e usa calcoli basati sulla velocità della luce per stimare la posizione.

**3. Rilevamento dell'anycast**
Se lo stesso IP corrisponde a posizioni diverse, si tratta di anycast (come i root server, Google DNS, Cloudflare DNS).

**Analisi di sicurezza:**
- L'infrastruttura DNS è geograficamente diversificata? (resilienza)
- Le risposte arrivano da paesi inattesi? (hijacking)
- Analisi della latenza per la selezione ottimale del resolver

### Sfida 14: comunicazione tramite covert channel su DNS

**Obiettivo:** implementare una comunicazione bidirezionale su DNS (solo per ricerca/didattica).

**Come funziona:**

**Client → Server (tramite query)**
```
Encode message in subdomain:
aGVsbG8=.tunnel.attacker.com
```

**Server → Client (tramite risposte)**
```
Encode response in TXT record:
"d29ybGQ="
```

**Implementazione:**
```python
def encode_query(data: bytes, domain: str) -> str:
    """Encode data in subdomain"""
    encoded = base64.b64encode(data).decode()
    # Split into DNS labels (63 char max)
    labels = [encoded[i:i+63] for i in range(0, len(encoded), 63)]
    return '.'.join(labels) + '.' + domain

def decode_response(txt_record: str) -> bytes:
    """Decode data from TXT record"""
    return base64.b64decode(txt_record)
```

**Sfide:**
- Il DNS ha limiti di dimensione (512 byte UDP, 4096 TCP)
- Occorre gestire la frammentazione
- La perdita di pacchetti richiede la ritrasmissione
- Alta latenza (1-2s per richiesta)

**Contromisure da studiare:**
- Analisi dell'entropia (sfida 7)
- Limiti sul volume di query
- Restrizioni sulla lunghezza dei sottodomini
- Monitoraggio della dimensione dei record TXT

**Nota etica:** testa solo su infrastruttura di tua proprietà. Il DNS tunneling senza autorizzazione è illegale.

### Sfida 15: costruire un resolver DNS ricorsivo

**Obiettivo:** implementare da zero un resolver ricorsivo completo (come BIND, Unbound).

**Cosa fa:**
1. Accetta query dai client
2. Itera attraverso la gerarchia DNS (come il comando trace)
3. Mette in cache i risultati
4. Restituisce le risposte

**Componenti principali:**

**1. Cache**
```python
from functools import lru_cache
import time

class DNSCache:
    def __init__(self):
        self.cache = {}
    
    def get(self, key):
        if key in self.cache:
            value, expiry = self.cache[key]
            if time.time() < expiry:
                return value
        return None
    
    def set(self, key, value, ttl):
        self.cache[key] = (value, time.time() + ttl)
```

**2. Risoluzione ricorsiva** (estendi `trace_dns()` da `resolver.py:293-426`)

**3. Loop del server**
```python
from dnslib.server import DNSServer

class RecursiveResolver:
    def resolve(self, request, handler):
        qname = str(request.q.qname)
        
        # Check cache
        cached = self.cache.get(qname)
        if cached:
            return cached
        
        # Perform recursive resolution
        result = trace_and_resolve(qname)
        
        # Cache result
        self.cache.set(qname, result, ttl=300)
        
        return result

resolver = RecursiveResolver()
server = DNSServer(resolver, port=5353)
server.start_thread()
```

**Sfide:**
- Gestire le catene di CNAME
- Implementare il negative caching (NXDOMAIN)
- Gestire timeout e retry
- Validazione DNSSEC
- Gestire il fallback su TCP per risposte di grandi dimensioni

**Testing:**
```bash
dig @localhost -p 5353 example.com
```

## Sfide di ricerca

### Sfida 16: machine learning per il rilevamento di domini malevoli

**Obiettivo:** addestrare un modello ML per classificare i domini come malevoli o benigni.

**Feature da estrarre:**
- Lunghezza del dominio
- Entropia dei caratteri
- Frequenze degli n-gram
- Rapporto vocali/consonanti
- TLD (.com vs .xyz vs .tk)
- Età di registrazione (dal WHOIS)
- Numero di sottodomini
- Cambi di IP storici
- Reputazione dell'ASN

**Dati di addestramento:**
- Benigni: i siti top di Alexa/Majestic
- Malevoli: abuse.ch, PhishTank, feed di malware

**Modelli da provare:**
- Random Forest
- XGBoost
- Reti neurali per dati sequenziali

**Valutazione:**
- Precision/recall sul test set
- Tasso di falsi positivi (critico per le blocklist)
- Prestazioni sui domini DGA

**Sfida:** pochi falsi positivi riuscendo comunque a intercettare minacce nuove.

### Sfida 17: analisi della privacy del DNS

**Obiettivo:** misurare l'information leakage dalle query DNS.

**Cosa trapela:**
- Cronologia di navigazione dai log delle query
- Posizione dalla scelta del resolver
- Tipo di dispositivo dai pattern delle query
- Utilizzo delle app dalle query ai CDN

**Esperimenti:**

**1. Ricostruire una sessione di navigazione**
Raccogli le query DNS per 1 ora. Riesci a dedurre:
- Quali siti web sono stati visitati?
- Quali app sono state usate?
- Interessi/demografia dell'utente?

**2. DNS fingerprinting**
Dispositivi diversi fanno query diverse:
- Android interroga android.googleapis.com
- iOS interroga i servizi di apple.com
- Windows interroga microsoft.com

**3. Attacchi di correlazione**
Anche con DoH, l'analisi dei tempi può rivelare i siti:
```python
# Time between query and HTTP request
# Query pattern (images, scripts, API calls)
# Size of responses
```

**Riferimenti:**
- "The Effect of DNS on Tor's Anonymity" (research paper)
- CWE-201: Information Exposure Through Sent Data

### Sfida 18: sviluppare una suite di benchmarking del DNS

**Obiettivo:** creare un toolkit completo per il testing delle prestazioni DNS.

**Benchmark:**

**1. Latenza**
- Misura la latenza delle query p50, p95, p99
- Testa da più posizioni geografiche
- Confronta i resolver (Google, Cloudflare, Quad9)

**2. Throughput**
- Capacità di query al secondo
- Gestione di query concorrenti

**3. Affidabilità**
- Tasso di successo su 24 ore
- Gestione di NXDOMAIN
- Frequenza dei timeout

**4. Privacy**
- Supporto per la query minimization (RFC 7816)
- Validazione DNSSEC
- Supporto DoH/DoT

**5. Resistenza alla censura**
- Le query vengono filtrate?
- I risultati vengono manipolati?

**Tool da costruire:**
- Esecutore automatizzato dei test
- Aggregazione e visualizzazione dei risultati
- Tracciamento storico
- Alerting sul degrado

**Servizio pubblico:** pubblica i risultati come [DNSPerf.com](https://www.dnsperf.com/)

## Conclusione

Ogni sfida sviluppa competenze specifiche:
- **Sfide 1-3**: sviluppo di tool, formati di output
- **Sfide 4-6**: monitoraggio di sicurezza, analisi delle prestazioni
- **Sfide 7-10**: rilevamento delle minacce, tool difensivi
- **Sfide 11-15**: concetti DNS avanzati, covert channel
- **Sfide 16-18**: ricerca, ML, studi di misurazione

**Prossimi passi:**
1. Scegli le sfide adatte al tuo livello di competenza
2. Inizia con le sfide per principianti per imparare la codebase
3. Prosegui con le sfide intermedie focalizzate sulla sicurezza
4. Affronta le sfide avanzate per una competenza approfondita

**Risorse:**
- RFC del DNS (1034, 1035 ed estensioni)
- OWASP Testing Guide (sezioni DNS)
- MITRE ATT&CK Framework (T1590, T1071.004, T1584.002)
- DNSViz per visualizzare DNSSEC
- Wireshark per l'analisi dei pacchetti

Buon hacking! Ricorda: testa solo su sistemi di tua proprietà o per i quali hai il permesso esplicito di effettuare test.
