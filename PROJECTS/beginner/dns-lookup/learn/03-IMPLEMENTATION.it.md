# Dettagli implementativi

## Implementazione principale della risoluzione DNS

### Query per un singolo tipo di record (`resolver.py:191-209`)
```python
async def query_record_type(
    domain: str,
    record_type: RecordType,
    resolver: dns.asyncresolver.Resolver,
) -> list[DNSRecord]:
    records = []
    
    try:
        answers = await resolver.resolve(domain, record_type.value)
        
        for rdata in answers:
            value, priority = extract_record_value(rdata, record_type)
            records.append(
                DNSRecord(
                    record_type=record_type,
                    value=value,
                    ttl=answers.rrset.ttl,
                    priority=priority,
                )
            )
    except (dns.resolver.NXDOMAIN, dns.resolver.NoAnswer, 
            dns.resolver.NoNameservers):
        pass  # Return empty list for these
    except dns.exception.Timeout:
        pass  # Also return empty list
    
    return records
```

**Decisioni di design chiave:**

1. **Eccezioni come controllo di flusso**: NXDOMAIN e NoAnswer non sono errori, sono risposte valide. La funzione restituisce una lista vuota invece di sollevare un'eccezione.

2. **TTL dall'rrset**: `answers.rrset.ttl` fornisce il TTL per l'intero resource record set. Tutti i record di un rrset hanno lo stesso TTL.

3. **Estrazione del valore delegata**: la funzione `extract_record_value()` gestisce il parsing specifico per tipo. Questo mantiene pulita la logica della query.

### Query concorrente su più tipi (`resolver.py:213-250`)
```python
async def lookup(
    domain: str,
    record_types: list[RecordType] | None = None,
    nameserver: str | None = None,
    timeout: float = 5.0,
) -> DNSResult:
    if record_types is None:
        record_types = ALL_RECORD_TYPES
    
    resolver = create_resolver(nameserver, timeout)
    result = DNSResult(domain=domain, nameserver=nameserver)
    
    start_time = time.perf_counter()
    
    # Create all tasks upfront
    tasks = [
        query_record_type(domain, rt, resolver) for rt in record_types
    ]
    
    # Execute all concurrently
    query_results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # Process results
    for i, query_result in enumerate(query_results):
        if isinstance(query_result, Exception):
            result.errors.append(f"{record_types[i]}: {query_result}")
        else:
            result.records.extend(query_result)
    
    result.query_time_ms = (time.perf_counter() - start_time) * 1000
    
    return result
```

**Perché `return_exceptions=True`?**

Senza questo flag, se una query solleva un'eccezione, `asyncio.gather()` cancella tutti gli altri task e rilancia l'eccezione. Con il flag, le eccezioni vengono restituite come valori, permettendo risultati parziali.

**Misurazione dei tempi:**
`time.perf_counter()` fornisce una misurazione dei tempi ad alta risoluzione. La differenza viene moltiplicata per 1000 per ottenere i millisecondi.

### Reverse DNS lookup (`resolver.py:253-290`)
```python
async def reverse_lookup(
    ip_address: str,
    nameserver: str | None = None,
    timeout: float = 5.0,
) -> DNSResult:
    resolver = create_resolver(nameserver, timeout)
    result = DNSResult(domain=ip_address, nameserver=nameserver)
    
    start_time = time.perf_counter()
    
    try:
        # resolve_address handles both IPv4 and IPv6
        answers = await resolver.resolve_address(ip_address)
        
        for rdata in answers:
            result.records.append(
                DNSRecord(
                    record_type=RecordType.PTR,
                    value=str(rdata.target).rstrip("."),
                    ttl=answers.rrset.ttl,
                )
            )
    except dns.resolver.NXDOMAIN:
        result.errors.append("No PTR record found")
    except dns.resolver.NoAnswer:
        result.errors.append("No answer from nameserver")
    except dns.resolver.NoNameservers:
        result.errors.append("No nameservers available")
    except dns.exception.Timeout:
        result.errors.append("Query timed out")
    except dns.exception.DNSException as e:
        result.errors.append(str(e))
    
    result.query_time_ms = (time.perf_counter() - start_time) * 1000
    
    return result
```

**Dettagli sul record PTR:**
Il reverse DNS usa le zone speciali `.in-addr.arpa` (IPv4) o `.ip6.arpa` (IPv6). Per l'IP `8.8.8.8`, la query è per `8.8.8.8.in-addr.arpa` (ottetti invertiti).

Il metodo `resolve_address()` gestisce questa conversione automaticamente.

## Implementazione del DNS trace

La funzione trace (`resolver.py:293-426`) è la più complessa. Implementa la risoluzione iterativa, interrogando ogni livello della gerarchia DNS.

### Inizializzazione (`resolver.py:298-314`)
```python
result = TraceResult(domain=domain)

try:
    name = dns.name.from_text(domain)
    rdtype = dns.rdatatype.from_text(record_type)
    
    # Hardcoded root server IPs
    root_servers = [
        ("a.root-servers.net", "198.41.0.4"),
        ("b.root-servers.net", "170.247.170.2"),
        ("c.root-servers.net", "192.33.4.12"),
    ]
    
    current_servers = root_servers
    current_zone = "."
```

**Perché hardcodare i root server?**
Problema di bootstrapping. Per risolvere `a.root-servers.net`, ti serve un resolver DNS funzionante. Questi IP sono essenzialmente la "radice della fiducia" del DNS.

### Ciclo principale di query (`resolver.py:316-420`)
```python
while True:
    server_name, server_ip = current_servers[0]
    
    try:
        # Build DNS query packet
        query = dns.message.make_query(name, rdtype)
        
        # Send UDP packet directly to server
        response = dns.query.udp(query, server_ip, timeout=3.0)
        
        rcode = response.rcode()
        
        if rcode != dns.rcode.NOERROR:
            result.error = f"DNS error: {dns.rcode.to_text(rcode)}"
            break
```

Questo usa API di basso livello di dnspython:
- `dns.message.make_query()`: costruisce il pacchetto di query DNS
- `dns.query.udp()`: invia il pacchetto via UDP

A differenza del `resolver.resolve()` di alto livello, questo dà il pieno controllo su quale server interrogare.

### Gestione delle risposte (`resolver.py:329-348`)
```python
if response.answer:
    # We got the answer!
    for rrset in response.answer:
        for rdata in rrset:
            result.final_answer = str(rdata)
            break
    
    result.hops.append(
        TraceHop(
            zone=current_zone,
            server=server_name,
            server_ip=server_ip,
            response=f"{record_type}: {result.final_answer}",
            is_authoritative=True,
        )
    )
    break  # Done tracing
```

La sezione `answer` contiene la risposta effettiva. Compare solo quando si interrogano i nameserver autoritativi.

### Seguire i referral (`resolver.py:350-404`)
```python
if response.authority:
    ns_records = []
    
    # Extract NS records from authority section
    for rrset in response.authority:
        if rrset.rdtype == dns.rdatatype.NS:
            for rdata in rrset:
                ns_name = str(rdata.target).rstrip(".")
                ns_records.append(ns_name)
            
            # Get the zone these NS records are for
            new_zone = str(rrset.name).rstrip(".")
            if not new_zone:
                new_zone = "."
```

La sezione `authority` contiene i record NS che puntano al livello successivo della gerarchia.

### Risolvere i glue record (`resolver.py:369-403`)
```python
# Check for glue records in additional section
glue_ips = {}
if response.additional:
    for rrset in response.additional:
        if rrset.rdtype == dns.rdatatype.A:
            for rdata in rrset:
                glue_ips[str(rrset.name).rstrip(".")] = rdata.address

# Build list of next servers to query
new_servers = []
for ns in ns_records:
    if ns in glue_ips:
        # Use glue record
        new_servers.append((ns, glue_ips[ns]))
    else:
        # Must resolve NS hostname separately
        try:
            answers = dns.resolver.resolve(ns, "A")
            for rdata in answers:
                new_servers.append((ns, rdata.address))
                break
        except dns.exception.DNSException:
            continue
```

**I glue record risolvono la dipendenza circolare:**
Se interroghi `example.com` e i record NS puntano a `ns1.example.com`, dovresti risolvere `ns1.example.com` per ottenerne l'IP. Ma per risolverlo, devi interrogare... lo stesso server NS. I glue record forniscono direttamente l'IP.

## Implementazione dei comandi CLI

### Parsing degli argomenti (`cli.py:70-87`)
```python
def parse_record_types(types_str: str) -> list[RecordType]:
    if types_str.upper() == "ALL":
        return list(ALL_RECORD_TYPES)
    
    types = []
    for t in types_str.upper().split(","):
        t = t.strip()
        try:
            types.append(RecordType(t))
        except ValueError:
            console.print(
                f"[yellow]Warning:[/yellow] Unknown record type '{t}', skipping"
            )
    
    return types if types else list(ALL_RECORD_TYPES)
```

Questo gestisce input dell'utente come `"A,MX,NS"` o `"all"`. I tipi non validi generano warning ma non causano un fallimento.

### Indicatori di avanzamento (`cli.py:146-154`)
```python
with Progress(
    SpinnerColumn(),
    TextColumn("[progress.description]{task.description}"),
    console=console,
    transient=True,
) as progress:
    progress.add_task(f"Querying {domain}...", total=None)
    result = asyncio.run(lookup(domain, record_types, server, timeout))
```

`transient=True` fa sparire lo spinner al termine. `total=None` crea uno spinner indefinito (non conosciamo in anticipo la durata della query).

### Elaborazione dei file batch (`cli.py:297-312`)
```python
domains = []
with open(file) as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith("#"):
            domains.append(line)

if not domains:
    console.print("[yellow]Warning:[/yellow] No domains found in file")
    raise typer.Exit(0)
```

Formato di file semplice:
- Un dominio per riga
- `#` per i commenti
- Le righe vuote vengono ignorate

**Nota di sicurezza:** nessuna validazione dell'input sui nomi di dominio. Questo si fida dell'input del file. Per file generati dagli utenti, aggiungi una validazione per prevenire input malevoli.

## Implementazione della formattazione dell'output

### Generazione della tabella (`output.py:83-127`)
```python
def print_results_table(result: DNSResult) -> None:
    if not result.records:
        console.print(
            Panel(
                f"[yellow]No records found for {result.domain}[/yellow]",
                title="[yellow]Warning[/yellow]",
                border_style="yellow",
                expand=False,
            )
        )
        return
    
    table = Table(
        title="[bold]DNS Records[/bold]",
        box=box.ROUNDED,
        border_style="blue",
        row_styles=["", "dim"],  # Alternate row shading
        show_header=True,
        header_style="bold cyan",
    )
    
    table.add_column("Type", width=8, no_wrap=True)
    table.add_column("Value", style="green", min_width=30)
    table.add_column("TTL", justify="right", style="dim", width=8)
    
    for record in result.records:
        color = get_record_color(record.record_type)
        value = record.value
        
        # Add priority annotation for MX records
        if record.priority is not None:
            value = f"{value} [dim](priority: {record.priority})[/dim]"
        
        table.add_row(
            f"[{color}]{record.record_type}[/{color}]",
            value,
            format_ttl(record.ttl),
        )
    
    console.print(table)
```

**Funzionalità di formattazione di Rich:**
- `box=box.ROUNDED`: angoli arrotondati
- `row_styles=["", "dim"]`: colori di riga alternati per la leggibilità
- `min_width=30`: impedisce che la colonna sia troppo stretta
- `justify="right"`: allinea a destra la colonna TTL

### Visualizzazione ad albero (`output.py:266-310`)
```python
tree = Tree(
    "[bold blue]:globe_showing_americas: DNS Resolution Path[/bold blue]",
    guide_style="blue",
)

zone_nodes: dict[str, Any] = {}

for hop in result.hops:
    # Create zone node if not exists
    if hop.zone not in zone_nodes:
        if hop.zone == ".":
            zone_display = "[bold yellow][.] Root[/bold yellow]"
        elif hop.zone.endswith("."):
            zone_display = f"[bold yellow][{hop.zone}] TLD[/bold yellow]"
        else:
            zone_display = f"[bold yellow][{hop.zone}.] Authoritative[/bold yellow]"
        
        zone_node = tree.add(zone_display)
        zone_nodes[hop.zone] = zone_node
    else:
        zone_node = zone_nodes[hop.zone]
    
    # Add server under zone
    server_style = "green" if hop.is_authoritative else "cyan"
    server_branch = zone_node.add(
        f"[{server_style}]:arrow_right: {hop.server}[/{server_style}] "
        f"[dim]({hop.server_ip})[/dim]"
    )
    
    # Add response under server
    server_branch.add(f"[dim]{hop.response}[/dim]")

console.print(tree)
```

**Raggruppamento per zona:**
Più hop possono interrogare la stessa zona (provando server diversi). Il dict garantisce un solo nodo di zona con più figli server.

### Serializzazione JSON (`output.py:379-410`)
```python
def results_to_json(results: list[DNSResult] | DNSResult) -> str:
    if isinstance(results, DNSResult):
        results = [results]
    
    data = []
    for result in results:
        record_data = [
            {
                "type": r.record_type.value,
                "value": r.value,
                "ttl": r.ttl,
                "priority": r.priority,
            } for r in result.records
        ]
        
        data.append({
            "domain": result.domain,
            "records": record_data,
            "errors": result.errors,
            "query_time_ms": round(result.query_time_ms, 2),
            "nameserver": result.nameserver,
        })
    
    # Single result: return object. Multiple: return array
    if len(data) == 1:
        return json.dumps(data[0], indent=2)
    
    return json.dumps(data, indent=2)
```

**Output singolo vs batch:**
Una query su un singolo dominio restituisce `{ domain, records }` mentre il batch restituisce `[{ domain, records }, ...]`. Questo è più ergonomico per chi consuma l'output.

## Implementazione WHOIS

### WHOIS lookup (`whois_lookup.py:60-119`)
```python
def lookup_whois(domain: str) -> WhoisResult:
    result = WhoisResult(domain=domain)
    
    try:
        w = whois.whois(domain)
        
        # Check if domain exists
        if w is None or (hasattr(w, "domain_name") and w.domain_name is None):
            result.error = "Domain not found or WHOIS data unavailable"
            return result
        
        # Extract available fields (not all present for all domains)
        result.registrar = w.registrar if hasattr(w, "registrar") else None
        result.creation_date = w.creation_date if hasattr(w, "creation_date") else None
        result.expiration_date = w.expiration_date if hasattr(w, "expiration_date") else None
        result.updated_date = w.updated_date if hasattr(w, "updated_date") else None
        
        # Handle status field (can be string or list)
        if hasattr(w, "status"):
            status = w.status
            if isinstance(status, str):
                result.status = [status]
            elif isinstance(status, list):
                result.status = status
            else:
                result.status = []
```

**Incoerenza dei dati WHOIS:**
Registry di TLD diversi restituiscono campi diversi. La libreria `python-whois` normalizza in parte, ma servono comunque controlli difensivi con `hasattr()`.

**Complessità del campo status:**
Alcuni registry restituiscono una singola stringa di status, altri restituiscono una lista. Normalizza per usare sempre una lista.

### Formattazione delle date (`whois_lookup.py:41-57`)
```python
def format_date(dt: datetime | list | None) -> str:
    if dt is None:
        return "[dim]-[/dim]"
    
    # Some registries return lists of datetimes
    if isinstance(dt, list):
        dt = dt[0] if dt else None
    
    if dt is None:
        return "[dim]-[/dim]"
    
    if isinstance(dt, datetime):
        return dt.strftime("%Y-%m-%d")
    
    return str(dt)
```

**Perché una lista di datetime?**
Alcuni server WHOIS includono versioni normalizzate per fuso orario. Prendiamo la prima.

## Implementazione dei test

### Setup dei test async (`test_resolver.py:1-20`)
```python
import pytest

from dnslookup.resolver import (
    ALL_RECORD_TYPES,
    DNSRecord,
    DNSResult,
    RecordType,
    TraceResult,
    batch_lookup,
    create_resolver,
    lookup,
    reverse_lookup,
    trace_dns,
)
```

I test importano dai moduli reali, non da mock. Questo testa query DNS reali.

### Testare funzioni async (`test_resolver.py:138-146`)
```python
class TestLookup:
    @pytest.mark.asyncio
    async def test_lookup_real_domain(self) -> None:
        result = await lookup("example.com", [RecordType.A])
        assert result.domain == "example.com"
        assert result.query_time_ms > 0
```

Il decorator `@pytest.mark.asyncio` permette funzioni di test async. Il plugin `pytest-asyncio` gestisce l'event loop.

### Testare le condizioni di errore (`test_resolver.py:148-156`)
```python
@pytest.mark.asyncio
async def test_lookup_nonexistent_domain(self) -> None:
    result = await lookup(
        "this-domain-definitely-does-not-exist-12345.com",
        [RecordType.A]
    )
    assert result.domain == "this-domain-definitely-does-not-exist-12345.com"
    assert len(result.records) == 0
```

**Nessun mocking:** usa un dominio che è estremamente improbabile che esista. Più fragile del mocking, ma testa il comportamento reale.

## Pattern implementativi comuni

### Pattern 1: dalle eccezioni ai risultati
```python
try:
    # Operation that might fail
    answers = await resolver.resolve(domain, record_type)
    # Process answers
except ExpectedException:
    # Convert to empty result
    return []
except UnexpectedException as e:
    # Log to errors list
    result.errors.append(str(e))
```

Questo mantiene pulita l'API (nessuna eccezione per i fallimenti attesi) preservando le informazioni sugli errori.

### Pattern 2: accesso difensivo agli attributi
```python
result.registrar = w.registrar if hasattr(w, "registrar") else None
```

Le risposte WHOIS variano. Usa `hasattr()` prima di accedere ad attributi che potrebbero non esistere.

### Pattern 3: type narrowing con isinstance
```python
if isinstance(query_result, Exception):
    result.errors.append(f"{record_types[i]}: {query_result}")
else:
    result.records.extend(query_result)
```

`asyncio.gather(return_exceptions=True)` restituisce le eccezioni come valori. Usa `isinstance()` per distinguerle.

### Pattern 4: delega della formattazione a Rich
```python
# Never mix business logic and formatting
# BAD:
def lookup(...):
    print(f"Querying {domain}")
    result = resolve(domain)
    print_table(result)
    return result

# GOOD:
def lookup(...):
    return resolve(domain)

# Caller handles formatting
result = lookup(domain)
print_table(result)
```

Mantiene il resolver riutilizzabile in contesti non CLI.

## Ottimizzazioni delle prestazioni

### Query concorrenti (`resolver.py:233-242`)

Query in sequenza: `7 types × 50ms = 350ms`
Query concorrenti: `max(50ms) = 50ms`

Accelerazione di 7x per le query su più tipi.

### Concorrenza nel batch (`resolver.py:428-440`)
```python
tasks = [lookup(domain, record_types, nameserver, timeout) for domain in domains]
return await asyncio.gather(*tasks)
```

Per 100 domini:
- In sequenza: `100 × 50ms = 5000ms`
- In concorrenza: `~200-500ms` (limitato dal rate limiting dei server DNS)

Accelerazione di 10-25x.

### Il compromesso dell'assenza di cache

Ogni query arriva al DNS. Questo significa:
- **Più lento**: nessun cache hit
- **Più accurato**: mai dati obsoleti
- **Carico maggiore**: più pacchetti verso i server DNS

Per un tool di ricognizione, l'accuratezza conta più della velocità.

Ora vedi `04-CHALLENGES.md` per scoprire come estendere questo progetto.
