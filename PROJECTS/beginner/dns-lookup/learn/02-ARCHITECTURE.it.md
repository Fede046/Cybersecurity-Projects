# Architettura e design

## Architettura del sistema

Il DNS lookup tool segue un'architettura pulita a tre livelli:
```
┌─────────────────────────────────────────────┐
│          User Interface Layer               │
│  (CLI commands, argument parsing)           │
│                                             │
│  File: cli.py                               │
│  Framework: Typer + Rich                    │
└──────────────┬──────────────────────────────┘
               │
               ├─> query()
               ├─> reverse()
               ├─> trace()
               ├─> batch()
               └─> whois()
               │
┌──────────────▼──────────────────────────────┐
│         Business Logic Layer                │
│  (DNS resolution, data processing)          │
│                                             │
│  Files: resolver.py, whois_lookup.py        │
│  Library: dnspython                         │
└──────────────┬──────────────────────────────┘
               │
               ├─> DNS Protocol (UDP:53)
               ├─> WHOIS Protocol (TCP:43)
               │
┌──────────────▼──────────────────────────────┐
│         Presentation Layer                  │
│  (Output formatting, visualization)         │
│                                             │
│  File: output.py                            │
│  Library: Rich                              │
└─────────────────────────────────────────────┘
```

Questa separazione permette di:
- Testare la business logic senza la CLI
- Cambiare formato di output senza modificare la logica di risoluzione
- Aggiungere nuovi comandi senza modificare il resolver principale

## Architettura del flusso dei dati

### Flusso di una query su un singolo dominio
```
User: dnslookup query example.com
            │
            ▼
    cli.py:112-167 (query command)
            │
            ├─> Parse arguments
            ├─> parse_record_types() → [A, AAAA, MX, ...]
            │
            ▼
    resolver.py:213-250 (lookup function)
            │
            ├─> create_resolver() → dns.asyncresolver.Resolver
            ├─> Start timer
            ├─> Create tasks for each record type
            │
            ▼
    resolver.py:191-209 (query_record_type)
            │
            ├─> await resolver.resolve(domain, "A")
            ├─> await resolver.resolve(domain, "AAAA")
            ├─> ... (parallel execution)
            │
            ▼
    asyncio.gather() → collect results
            │
            ▼
    DNSResult object (records + metadata)
            │
            ▼
    output.py:83-127 (print_results_table)
            │
            └─> Rich Table → Terminal
```

### Flusso di una query batch
```
User: dnslookup batch domains.txt
            │
            ▼
    cli.py:266-350 (batch command)
            │
            ├─> Read file line by line
            ├─> Filter comments and empty lines
            │
            ▼
    resolver.py:428-440 (batch_lookup)
            │
            └─> [lookup(d1), lookup(d2), ..., lookup(dn)]
                      │
                      ▼
            asyncio.gather() → parallel execution
                      │
                      ▼
            [DNSResult, DNSResult, DNSResult, ...]
                      │
                      ▼
    output.py:340-377 (print_batch_results)
```

L'ottimizzazione chiave: tutti i domini vengono interrogati in modo concorrente (`resolver.py:432-440`).

### Flusso del DNS trace
```
User: dnslookup trace example.com
            │
            ▼
    cli.py:219-263 (trace command)
            │
            ▼
    resolver.py:293-426 (trace_dns)
            │
            ├─> Start at root servers [.]
            │   └─> Query a.root-servers.net:198.41.0.4
            │       Response: "Refer to .com servers"
            │
            ├─> Query .com TLD server
            │   └─> Get NS records for example.com
            │       Response: "Refer to ns1.example.com"
            │
            ├─> Query authoritative server
            │   └─> ns1.example.com
            │       Response: "A 93.184.216.34" (answer!)
            │
            └─> Build TraceResult with hops
                      │
                      ▼
    output.py:266-310 (print_trace_result)
            └─> Rich Tree visualization
```

Questo imita il funzionamento di un vero resolver ricorsivo.

## Strutture dati principali

### Enum RecordType (`resolver.py:24-33`)
```python
class RecordType(StrEnum):
    A = "A"
    AAAA = "AAAA"
    MX = "MX"
    NS = "NS"
    TXT = "TXT"
    CNAME = "CNAME"
    SOA = "SOA"
    PTR = "PTR"
```

Usare `StrEnum` fornisce type safety pur permettendo il confronto tra stringhe. I valori corrispondono esattamente ai nomi dei tipi di record del protocollo DNS.

### Dataclass DNSRecord (`resolver.py:46-54`)
```python
@dataclass
class DNSRecord:
    record_type: RecordType
    value: str
    ttl: int
    priority: int | None = None
```

Rappresenta un singolo resource record DNS. Il campo `priority` è `None` per la maggior parte dei tipi di record, ma viene valorizzato per i record MX (`resolver.py:148-150`).

### Dataclass DNSResult (`resolver.py:57-65`)
```python
@dataclass
class DNSResult:
    domain: str
    records: list[DNSRecord] = field(default_factory=list)
    errors: list[str] = field(default_factory=list)
    query_time_ms: float = 0.0
    nameserver: str | None = None
```

Aggrega tutte le informazioni su una query. Usare `field(default_factory=list)` previene i bug legati agli argomenti di default mutabili.

### TraceHop e TraceResult (`resolver.py:68-88`)
```python
@dataclass
class TraceHop:
    zone: str              # ".", ".com", "example.com"
    server: str            # "a.root-servers.net"
    server_ip: str         # "198.41.0.4"
    response: str          # Human-readable response
    is_authoritative: bool # Final answer vs referral

@dataclass
class TraceResult:
    domain: str
    hops: list[TraceHop] = field(default_factory=list)
    final_answer: str | None = None
    error: str | None = None
```

Modella il percorso completo di risoluzione attraverso la gerarchia DNS.

## Pattern di interazione tra componenti

### Disaccoppiamento tra resolver e output

Il resolver non importa mai output. Restituisce strutture dati. Il livello CLI chiama entrambi:
```python
# cli.py:155-167
result = asyncio.run(lookup(domain, record_types, server, timeout))

if json_output:
    console.print(results_to_json(result))
else:
    print_header(domain)
    print_results_table(result)
    print_errors(result)
    print_summary(result)
```

Questo permette:
- Formati di output diversi (JSON, tabella, CSV)
- Di testare il resolver senza terminale
- Di usare il resolver in altri progetti

### Strategia di gestione degli errori

Il resolver cattura le eccezioni e le converte in messaggi di errore (`resolver.py:181-189`):
```python
except (dns.resolver.NXDOMAIN, dns.resolver.NoAnswer, 
        dns.resolver.NoNameservers):
    pass  # Expected, not an error
except dns.exception.Timeout:
    pass  # Also expected
```

Gli errori vengono accumulati nella lista `DNSResult.errors` invece di sollevare eccezioni. Questo permette risultati parziali (alcuni tipi di record hanno successo, altri falliscono).

### Modello di esecuzione async

Il progetto usa `asyncio` ovunque per le operazioni DNS I/O-bound. Pattern chiave (`resolver.py:233-242`):
```python
tasks = [query_record_type(domain, rt, resolver) for rt in record_types]
query_results = await asyncio.gather(*tasks, return_exceptions=True)

for i, query_result in enumerate(query_results):
    if isinstance(query_result, Exception):
        result.errors.append(f"{record_types[i]}: {query_result}")
    else:
        result.records.extend(query_result)
```

`return_exceptions=True` impedisce che un singolo fallimento cancelli gli altri task.

## Implementazione del protocollo DNS

### Creare un resolver (`resolver.py:91-107`)
```python
def create_resolver(
    nameserver: str | None = None,
    timeout: float = 5.0,
) -> dns.asyncresolver.Resolver:
    resolver = dns.asyncresolver.Resolver()
    resolver.timeout = timeout
    resolver.lifetime = timeout * 2  # Total query lifetime
    
    if nameserver:
        resolver.nameservers = [nameserver]
    
    return resolver
```

Due valori di timeout:
- `timeout`: timeout per singola query
- `lifetime`: tempo totale inclusi i retry

### Estrarre i valori dei record (`resolver.py:110-158`)

Tipi di record diversi hanno strutture di risposta diverse:
```python
if record_type == RecordType.A or record_type == RecordType.AAAA:
    value = rdata.address  # Simple IP string
elif record_type == RecordType.MX:
    value = str(rdata.exchange).rstrip(".")
    priority = rdata.preference  # MX-specific
elif record_type in (RecordType.NS, RecordType.CNAME, RecordType.PTR):
    value = str(rdata.target).rstrip(".")  # FQDN
```

`.rstrip(".")` rimuove il punto finale dai nomi di dominio completamente qualificati (FQDN).

### Approfondimento sull'implementazione del trace

La funzione trace (`resolver.py:293-426`) implementa la risoluzione DNS iterativa. Sezioni chiave:

**1. Partire dai root server** (`resolver.py:307-314`):
```python
root_servers = [
    ("a.root-servers.net", "198.41.0.4"),
    ("b.root-servers.net", "170.247.170.2"),
    ("c.root-servers.net", "192.33.4.12"),
]
current_servers = root_servers
current_zone = "."
```

**2. Ciclo di query** (`resolver.py:318-420`):
Ogni iterazione interroga un server, elabora la risposta e segue i referral.

**3. Verificare la presenza di una risposta** (`resolver.py:329-348`):
```python
if response.answer:
    for rrset in response.answer:
        for rdata in rrset:
            result.final_answer = str(rdata)
            break
    # Record this hop as authoritative
    result.hops.append(TraceHop(..., is_authoritative=True))
    break  # Done!
```

**4. Seguire i referral** (`resolver.py:350-404`):
```python
if response.authority:
    ns_records = []
    for rrset in response.authority:
        if rrset.rdtype == dns.rdatatype.NS:
            for rdata in rrset:
                ns_name = str(rdata.target).rstrip(".")
                ns_records.append(ns_name)
```

**5. Risolvere i glue record** (`resolver.py:374-403`):
I glue record forniscono gli indirizzi IP dei nameserver per evitare dipendenze circolari.
```python
glue_ips = {}
if response.additional:
    for rrset in response.additional:
        if rrset.rdtype == dns.rdatatype.A:
            for rdata in rrset:
                glue_ips[str(rrset.name).rstrip(".")] = rdata.address
```

## Architettura della formattazione dell'output

### Integrazione con la console di Rich

Tutto l'output passa da un'unica istanza di console (`output.py:19`):
```python
console = Console()
```

Questo garantisce uno stile coerente e supporta il rilevamento dei colori.

### Colorazione dei tipi di record (`output.py:22-32`)
```python
RECORD_COLORS: dict[RecordType, str] = {
    RecordType.A: "green",
    RecordType.AAAA: "blue",
    RecordType.MX: "magenta",
    RecordType.NS: "cyan",
    RecordType.TXT: "yellow",
    RecordType.CNAME: "red",
    RecordType.SOA: "white",
    RecordType.PTR: "bright_cyan",
}
```

I colori aiutano a distinguere visivamente i tipi di record nelle query con tipi misti.

### Formattazione del TTL (`output.py:45-61`)

Converte i secondi in un formato leggibile:
```python
if ttl >= 86400:
    days = ttl // 86400
    return f"{days}d"
elif ttl >= 3600:
    hours = ttl // 3600
    return f"{hours}h"
```

Questo rende i valori di TTL più facili da capire a colpo d'occhio.

### Visualizzazione ad albero per i trace (`output.py:266-310`)
```python
tree = Tree(
    "[bold blue]:globe_showing_americas: DNS Resolution Path[/bold blue]",
    guide_style="blue",
)

zone_nodes: dict[str, Any] = {}

for hop in result.hops:
    if hop.zone not in zone_nodes:
        zone_node = tree.add(zone_display)
        zone_nodes[hop.zone] = zone_node
    else:
        zone_node = zone_nodes[hop.zone]
    
    server_branch = zone_node.add(f"[{server_style}]:arrow_right: {hop.server}[/{server_style}]")
```

Raggruppa gli hop per zona per una visualizzazione più chiara.

## Configurazione e gestione delle dipendenze

### Configurazione del progetto (`pyproject.toml:1-36`)
```toml
[project]
name = "dnslookup-cli"
version = "0.1.1"
dependencies = [
    "dnspython>=2.8.0",   # DNS protocol library
    "rich>=14.2.0",       # Terminal formatting
    "typer>=0.20.0",      # CLI framework
    "python-whois>=0.9.6", # WHOIS lookups
]
```

Dipendenze minime mantengono il progetto leggero.

### Strumenti di sviluppo (`pyproject.toml:38-45`)
```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.25.0",  # Test async code
    "pytest-cov>=6.0.0",       # Coverage reporting
    "ruff>=0.9.0",             # Linting
    "mypy>=1.15.0",            # Type checking
]
```

## Automazione dei task (justfile)

Il justfile fornisce comandi comodi (`justfile:17-150`):
```make
run *ARGS:
    uv run dnslookup {{ARGS}}

lookup domain *ARGS:
    uv run dnslookup {{domain}} {{ARGS}}

ci: lint typecheck test
```

Questo semplifica il workflow di sviluppo senza richiedere la conoscenza della sintassi di `uv`.

## Considerazioni di sicurezza nell'architettura

**1. Nessuno stato persistente**: ogni query è indipendente
- Previene il cache poisoning
- Nessuno stato da corrompere
- Semplice da ragionare

**2. Selezione esplicita del server DNS**: l'utente controlla di quale resolver fidarsi
- Si può testare sui nameserver autoritativi
- Utile per validare la propagazione DNS

**3. Applicazione dei timeout**: le operazioni di rete non possono bloccarsi indefinitamente
- `timeout` per query (`resolver.py:99`)
- `lifetime` per l'operazione totale (`resolver.py:100`)

**4. Trasparenza degli errori**: tutti gli errori vengono mostrati all'utente
- Nessun fallimento silenzioso
- L'utente può indagare sui problemi

**5. Nessuna cache locale**: dati freschi a ogni query
- Evita problemi di dati obsoleti
- Carico maggiore ma più accuratezza

Ora vedi `03-IMPLEMENTATION.md` per walkthrough dettagliati del codice.
