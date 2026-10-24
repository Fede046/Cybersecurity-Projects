# 02-ARCHITECTURE.md

# Architettura del sistema

## Pipeline di alto livello

Lo scanner segue una pipeline lineare: la CLI analizza gli argomenti, l'engine orchestra, gli scanner estraggono e rilevano, e i reporter formattano l'output.

```
┌──────────────────────────────────────────────────────────┐
│                     CLI Layer (Typer)                     │
│                                                          │
│  dlp-scan file ./data -f json -o results.json            │
│  dlp-scan db postgres://user:pass@host/db                │
│  dlp-scan network capture.pcap                           │
│  dlp-scan report summary results.json                    │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                    ScanEngine                             │
│                                                          │
│  Loads config ─► Builds DetectorRegistry ─► Selects      │
│  scanner type ─► Runs scan ─► Routes to reporter         │
└──────────────────────┬───────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│ FileScanner  │ │DBScanner │ │NetworkScanner│
│              │ │          │ │              │
│ Walk dirs    │ │ Schema   │ │ PCAP parse   │
│ Extract text │ │ introspect│ │ TCP reassembly│
│ Run detectors│ │ Sample   │ │ DNS exfil    │
│              │ │ rows     │ │ DPI protocol │
│              │ │ Detect   │ │ Detect       │
└──────┬───────┘ └────┬─────┘ └──────┬───────┘
       │              │              │
       └──────────────┼──────────────┘
                      ▼
┌──────────────────────────────────────────────────────────┐
│                 DetectorRegistry                          │
│                                                          │
│  PatternDetector ─► ContextBoost ─► CooccurrenceBoost    │
│       │                                                  │
│       └─► EntropyDetector (parallel)                     │
│                                                          │
│  Rules: PII | Financial | Credentials | Health           │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                    Reporter Layer                         │
│                                                          │
│  ConsoleReporter ─► Rich tables with severity colors     │
│  JsonReporter    ─► Structured JSON with metadata        │
│  SarifReporter   ─► SARIF 2.1.0 for CI/CD pipelines     │
│  CsvReporter     ─► Flat CSV for compliance teams        │
└──────────────────────────────────────────────────────────┘
```

## Scomposizione dei componenti

### Livello CLI

**Scopo:** Analizzare gli argomenti della riga di comando, propagare le opzioni globali, instradare verso il comando di scansione o l'utility di report corretti.

**File:** `cli.py`, `commands/scan.py`, `commands/report.py`

L'app Typer radice in `cli.py` definisce un callback che cattura `--config`, `--verbose` e `--version` nell'oggetto di contesto di Click. I comandi di scansione (`file`, `db`, `network`) sono definiti in `commands/scan.py` e registrati come comandi di primo livello tramite una funzione `register(app)` che chiama `app.command("file")(scan_file)` per ciascuno. Questo evita di annidarli sotto un sottogruppo `scan` pur mantenendo le definizioni dei comandi nel proprio modulo.

Il sottogruppo `report` è un'istanza Typer separata aggiunta tramite `app.add_typer(report_app, name="report")`. Fornisce `convert` (da JSON ad altri formati) e `summary` (stampa una tabella Rich a partire dai risultati JSON).

### ScanEngine

**Scopo:** Unico punto di orchestrazione che collega la configurazione agli scanner e ai reporter.

**File:** `engine.py`

L'engine prende uno `ScanConfig` e costruisce un `DetectorRegistry` scomponendo la configurazione di rilevamento in singoli parametri:

```python
class ScanEngine:
    def __init__(self, config: ScanConfig) -> None:
        self._config = config
        detection = config.detection
        allowlist_vals = detection.allowlists.values
        self._registry = DetectorRegistry(
            enable_patterns=detection.enable_rules,
            disable_patterns=detection.disable_rules,
            allowlist_values=(
                frozenset(allowlist_vals)
                if allowlist_vals else None
            ),
            context_window_tokens=(
                detection.context_window_tokens
            ),
        )
```

L'engine espone `scan_files`, `scan_database` e `scan_network`, ognuno dei quali costruisce lo scanner appropriato, lo esegue e restituisce uno `ScanResult`. La generazione dei report usa un dict `REPORTER_MAP` che mappa le stringhe di formato sulle classi reporter.

### DetectorRegistry

**Scopo:** Hub centrale che carica le regole di rilevamento, le filtra tramite glob di abilitazione/disabilitazione ed esegue l'intera pipeline di scoring sul testo.

**File:** `detectors/registry.py`

Il registry carica tutte le regole da quattro moduli di regole (PII, Financial, Credentials, Health), le filtra usando `fnmatch.fnmatch` contro i pattern di abilitazione/disabilitazione, e incapsula le superstiti in un `PatternDetector`. Quando viene chiamato `detect()`:

1. `PatternDetector` esegue tutti i pattern regex, valida le corrispondenze con i checksum (Luhn, Mod-97, Mod-11) e le filtra contro l'allowlist
2. `apply_context_boost` scansiona una finestra di token attorno a ogni corrispondenza alla ricerca di keyword pertinenti e regola i punteggi in base alla prossimità
3. `_apply_cooccurrence_boost` aggiunge un bonus quando più tipi diversi di PII compaiono entro 500 caratteri l'uno dall'altro
4. `EntropyDetector` trova in modo indipendente le regioni ad alta entropia usando una finestra scorrevole

```
Text Input
    │
    ▼
┌─────────────────────────┐
│    PatternDetector       │
│                          │
│ For each rule:           │
│   regex.finditer(text)   │
│   ─► allowlist filter    │
│   ─► validator (Luhn,    │
│       Mod-97, SSN area)  │
│   ─► base_score + boost  │
└─────────┬───────────────┘
          │
          ▼
┌─────────────────────────┐
│    Context Boost         │
│                          │
│ Token window ±10 tokens  │
│ Keyword proximity search │
│ Distance-weighted boost  │
│ (0.05 to 0.35)           │
└─────────┬───────────────┘
          │
          ▼
┌─────────────────────────┐
│   Co-occurrence Boost    │
│                          │
│ Different rule_ids       │
│ within 500 chars ─► +0.15│
└─────────┬───────────────┘
          │
          ▼
┌─────────────────────────┐
│   Entropy Detector       │
│                          │
│ Sliding 256-byte window  │
│ Shannon H >= 7.2 bits    │
│ Independent matches      │
└─────────┬───────────────┘
          │
          ▼
     DetectorMatch[]
```

### Scanner

**Scopo:** Ogni scanner gestisce una diversa superficie di scansione (file, database, rete) e converte i dati grezzi in testo che il DetectorRegistry può elaborare.

**File:** `scanners/file_scanner.py`, `scanners/db_scanner.py`, `scanners/network_scanner.py`

Tutti gli scanner seguono lo stesso protocollo `Scanner`: un metodo `scan(target: str) -> ScanResult`. Condividono un flusso comune: iterano sui target, estraggono il testo, eseguono il rilevamento, convertono le corrispondenze in finding tramite `match_to_finding` in `scoring.py` (che gestisce in una sola chiamata la classificazione della severità, la ricerca della compliance, la remediation e la redaction), e aggregano il tutto in uno `ScanResult`.

**FileScanner** percorre un albero di directory, applica i filtri di estensione ed esclusione, smista ogni file all'estrattore appropriato in base all'estensione ed esegue il detector su ogni `TextChunk`. La mappatura da estensione a estrattore viene costruita una sola volta da `_build_extension_map`, che itera su tutte le istanze degli estrattori e le indicizza per le loro `supported_extensions`.

**DatabaseScanner** si connette tramite il rilevamento dello schema dell'URI (postgres, mysql, mongodb, sqlite), fa l'introspezione dello schema per trovare le colonne di tipo testo, campiona le righe usando il campionamento nativo del database (TABLESAMPLE BERNOULLI per PostgreSQL, RAND() per MySQL, $sample per MongoDB) e scansiona i valori delle colonne.

**NetworkScanner** legge i file PCAP tramite `read_pcap`, inserisce i pacchetti in un `FlowTracker` per il riassemblaggio TCP, ed elabora il traffico DNS inline tramite `parse_dns` e `DnsExfilDetector`. Il payload di ogni pacchetto viene anche controllato da `detect_base64_payload` alla ricerca di dati codificati. Dopo l'iterazione sui pacchetti, lo scanner riassembla i flussi TCP, identifica il protocollo applicativo tramite `identify_protocol`, estrae il testo tenendo conto del protocollo (`parse_http` per i body HTTP e gli header sensibili, salta TLS/SSH cifrati, decodifica UTF-8 per tutto il resto), ed esegue il rilevamento sul testo estratto.

### Estrattori

**Scopo:** Convertire formati di file binari e strutturati in oggetti `TextChunk` uniformi che portano sia il testo estratto sia una `Location` che descrive da dove proviene.

**File:** `extractors/plaintext.py`, `extractors/pdf.py`, `extractors/office.py`, `extractors/structured.py`, `extractors/archive.py`, `extractors/email.py`

Tutti gli estrattori implementano il protocollo `Extractor`: `extract(path) -> list[TextChunk]` e `supported_extensions -> frozenset[str]`.

```
┌───────────────────────────────────────────────┐
│              Extractor Protocol                │
│  extract(path) -> list[TextChunk]             │
│  supported_extensions -> frozenset[str]        │
└───────────────────────────────────────────────┘
         │
    ┌────┴────┬──────────┬──────────┬──────┐
    ▼         ▼          ▼          ▼      ▼
Plaintext   PDF      Office   Structured  Archive
.txt .log  .pdf    .docx     .csv .json  .zip
.cfg .py           .xlsx     .xml .yaml  .tar.gz
.html .md          .xls      .parquet    .tar.bz2
.ts .go                      .avro
...                          .tsv
```

Il `PlaintextExtractor` divide i file in blocchi da 500 righe per mantenere limitata la memoria. Gli estrattori di formati binari (PDF tramite PyMuPDF, DOCX tramite python-docx, XLSX tramite openpyxl) restituiscono ciascuno un `TextChunk` per pagina/foglio/sezione. L'estrattore di archivi entra ricorsivamente nei file compressi fino a una profondità configurabile con protezione dalle zip bomb (controllo di una soglia sul rapporto di compressione).

### Reporter

**Scopo:** Prendere uno `ScanResult` e serializzarlo nel formato di output richiesto.

**File:** `reporters/console.py`, `reporters/json_report.py`, `reporters/sarif.py`, `reporters/csv_report.py`

Ogni reporter ha un metodo `generate(result) -> str`. Il `ConsoleReporter` ha anche un metodo `display(result)` per l'output nel terminale formattato con Rich, con tabelle colorate per severità.

Il reporter JSON produce un documento strutturato con le sezioni `scan_metadata`, `findings` e `summary`. Il reporter SARIF produce un documento SARIF 2.1.0 con `tool.driver.rules`, mappando i livelli di severità tramite `SARIF_SEVERITY_MAP` (critical/high su "error", medium su "warning", low su "note"). Il reporter CSV appiattisce i finding in righe.

## Modelli di dati

### Modelli principali

```python
@dataclass(frozen=True, slots=True)
class Location:
    source_type: str
    uri: str
    line: int | None = None
    column: int | None = None
    byte_offset: int | None = None
    table_name: str | None = None
    column_name: str | None = None
    sheet_name: str | None = None


@dataclass(slots=True)
class Finding:
    finding_id: str
    rule_id: str
    rule_name: str
    severity: Severity
    confidence: float
    location: Location
    redacted_snippet: str
    compliance_frameworks: list[str]
    remediation: str
    detected_at: datetime


@dataclass(slots=True)
class ScanResult:
    scan_id: str
    tool_version: str
    scan_started_at: datetime
    scan_completed_at: datetime | None
    targets_scanned: int
    findings: list[Finding]
    errors: list[str]
```

`Location` è frozen perché rappresenta un fatto su dove qualcosa è stato trovato. `Finding` è mutabile perché campi come `finding_id` e `detected_at` ricevono default da funzioni factory. `ScanResult` aggrega i finding e fornisce proprietà calcolate (`findings_by_severity`, `findings_by_rule`, `findings_by_framework`) che raggruppano i conteggi per il reporting di riepilogo.

La dataclass `TextChunk` porta il testo estratto abbinato alla sua `Location`, formando il ponte tra estrattori e detector. Ogni frammento di testo sa esattamente da dove proviene, il che permette ai finding di portare con sé informazioni precise sulla posizione attraverso la pipeline.

### Modelli di rilevamento

```python
@dataclass(frozen=True, slots=True)
class DetectionRule:
    rule_id: str
    rule_name: str
    pattern: re.Pattern[str]
    base_score: float
    context_keywords: list[str]
    validator: Callable[[str], bool] | None
    compliance_frameworks: list[str]


@dataclass(frozen=True, slots=True)
class DetectorMatch:
    rule_id: str
    rule_name: str
    start: int
    end: int
    matched_text: str
    score: float
    context_keywords: list[str]
    compliance_frameworks: list[str]
```

`DetectionRule` è una specifica: il pattern regex da trovare, il confidence score base, il validatore di checksum opzionale e le keyword di contesto. `DetectorMatch` è un risultato: cosa è stato trovato, dove nel testo, e il punteggio corrente dopo la validazione. Il campo `score` viene modificato attraverso la pipeline di boost (contesto, co-occorrenza) prima di essere mappato su un livello `Severity` e inserito in un `Finding`.

## Architettura della configurazione

```
┌────────────────────────────────────────────┐
│            .dlp-scanner.yml                │
│                                            │
│  scan:                                     │
│    file: { max_file_size_mb, recursive }   │
│    database: { sample_percentage }         │
│    network: { bpf_filter, max_packets }    │
│  detection:                                │
│    min_confidence, enable_rules,           │
│    disable_rules, allowlists               │
│  compliance: { frameworks }                │
│  output: { format, redaction_style }       │
│  logging: { level, json_output }           │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│         load_config(path) -> ScanConfig    │
│                                            │
│  1. Check CLI --config flag                │
│  2. Search candidates:                     │
│     .dlp-scanner.yml                       │
│     .dlp-scanner.yaml                      │
│     ~/.dlp-scanner.yml                     │
│  3. Parse YAML via ruamel.yaml             │
│  4. Validate with Pydantic 2.x models     │
│  5. Return ScanConfig with defaults        │
└────────────────────────────────────────────┘
```

Ogni valore di configurazione ha un default costante definito in `constants.py`. I modelli Pydantic in `config.py` usano queste costanti come default dei campi, quindi un file di configurazione completamente vuoto produce uno scanner funzionante. I campi a scelta vincolata (`severity_threshold`, `format`, `redaction_style`) usano tipi `Literal` definiti in `constants.py` (ad es. `Literal["critical", "high", "medium", "low"]`), così Pydantic rifiuta i valori non validi al momento del parsing invece di accettare silenziosamente un refuso. Il loader della configurazione usa `ruamel.yaml` (non PyYAML) perché preserva i commenti e gestisce YAML 1.2.

La struttura YAML usa una chiave di primo livello `scan:` per raggruppare la configurazione specifica degli scanner, mentre `detection:`, `compliance:`, `output:` e `logging:` stanno al livello radice. Questo rispecchia il modo in cui gli utenti pensano alla configurazione: "come scansionare" contro "cosa rilevare" contro "come riportare".

## Flusso dei dati: scansione di file

Percorso passo per passo di `dlp-scan file ./data -f json`:

```
1. Typer parses args
   └─► main() callback stores config_path="" and verbose=False in ctx.obj

2. scan_file() receives ctx, target="./data", format="json"
   └─► _run_scan() validates format, loads config, sets logging to WARNING
       (WARNING for machine-readable formats keeps stdout clean)

3. ScanEngine(config) constructs DetectorRegistry
   └─► Registry loads 29 rules from PII/Financial/Credential/Health modules
   └─► Filters through enable_rules=["*"], disable_rules=[]

4. engine.scan_files("./data")
   └─► FileScanner.scan() creates ScanResult, walks directory

5. For each file in ./data/**/*:
   └─► Check extension against include_extensions
   └─► Check path against exclude_patterns
   └─► Check file size against max_file_size_mb
   └─► Select extractor by extension (e.g. .csv -> CsvExtractor)
   └─► extractor.extract(path) -> list[TextChunk]

6. For each TextChunk:
   └─► registry.detect(chunk.text) -> list[DetectorMatch]
       ├─► PatternDetector: regex match + allowlist + validator
       ├─► apply_context_boost: keyword proximity scoring
       ├─► _apply_cooccurrence_boost: multi-PII bonus
       └─► EntropyDetector: high-entropy region detection

7. For each DetectorMatch above min_confidence:
   └─► match_to_finding(match, text, location, redaction_style)
       ├─► score_to_severity(match.score) -> Severity
       ├─► get_frameworks_for_rule(match.rule_id) -> compliance list
       ├─► get_remediation_for_rule(match.rule_id) -> guidance string
       └─► redact(chunk.text, start, end, style) -> snippet
   └─► Append Finding to ScanResult

8. Back in _run_scan():
   └─► engine.generate_report(result, "json")
   └─► JsonReporter().generate(result) -> JSON string
   └─► typer.echo(output) -> stdout
```

## Design pattern

### Polimorfismo basato su Protocol

La codebase usa `typing.Protocol` di Python invece delle abstract base class per i punti di estensione. I protocolli `Extractor`, `Scanner` e `Detector` definiscono interfacce strutturali senza richiedere l'ereditarietà.

```python
class Extractor(Protocol):
    def extract(self, path: str) -> list[TextChunk]: ...

    @property
    def supported_extensions(self) -> frozenset[str]: ...
```

Qualsiasi classe con signature dei metodi corrispondenti soddisfa il protocollo. Questo significa che puoi aggiungere un nuovo estrattore (per esempio, per i file .pptx) senza importare il modulo base. Il type checker verifica la conformità; il runtime non controlla mai l'ereditarietà.

**Perché non le ABC:** Le abstract base class impongono una dipendenza di import e obbligano a catene di `super().__init__()`. I Protocol sono più leggeri e si accordano con la filosofia del duck typing di Python. Poiché gli estrattori sono stateless (nessuno stato condiviso né ciclo di vita), non c'è nulla che una ABC fornirebbe oltre al contratto di tipo.

### Registry pattern

Il `DetectorRegistry` centralizza la gestione delle regole: caricamento, filtraggio ed esecuzione. I singoli moduli di regole (pii.py, financial.py, credentials.py, health.py) esportano ciascuno una lista di oggetti `DetectionRule`. Il registry li unisce in `ALL_RULES`, applica il filtraggio tramite glob e incapsula il risultato in un `PatternDetector`.

Questo mantiene dichiarative le definizioni delle regole. Aggiungere una nuova regola è questione di accodare una `DetectionRule` alla lista appropriata. Il registry gestisce filtraggio ed esecuzione senza che gli autori delle regole debbano capire la pipeline di scoring.

### Pattern di registrazione dei comandi

I comandi CLI sono definiti in `commands/scan.py` come semplici funzioni e registrati sull'app radice tramite una funzione `register(app)`:

```python
def register(app: typer.Typer) -> None:
    app.command("file")(scan_file)
    app.command("db")(scan_db)
    app.command("network")(scan_network)
```

Questo produce comandi di primo livello (`dlp-scan file`, non `dlp-scan scan file`) mantenendo la logica dei comandi fuori da `cli.py`. L'helper `_run_scan` deduplica la logica condivisa (caricamento della configurazione, validazione del formato, instradamento dell'output) tra tutti e tre i tipi di scansione.

## Mappatura della compliance

Il modulo di compliance mappa gli ID delle regole sui framework normativi e sulle indicazioni di remediation usando due dizionari statici:

```
RULE_FRAMEWORK_MAP: rule_id -> [frameworks]
RULE_REMEDIATION_MAP: rule_id -> guidance string
```

Gli ID delle regole corrispondono alle regole di rilevamento effettive (ad es. `FIN_CREDIT_CARD_VISA`, `FIN_CREDIT_CARD_MC`, non un generico `FIN_CREDIT_CARD`). Anche gli indicatori di esfiltrazione di rete (`NET_DNS_EXFIL_*`, `NET_ENCODED_*`) sono mappati. Ogni regola ha una voce di remediation con un testo di indicazioni specifico; le regole sconosciute ripiegano su un default generico.

Quando un `DetectorMatch` viene convertito in un `Finding` tramite `match_to_finding` in `scoring.py`, la funzione chiama `get_frameworks_for_rule` e `get_remediation_for_rule` per arricchire il finding con i metadati di compliance. Se anche la regola di rilevamento stessa porta `compliance_frameworks`, i due insiemi vengono uniti.

Questo design mantiene le regole di rilevamento indipendenti dalla logica di compliance. Il modulo PII non ha bisogno di sapere che HIPAA si interessa agli SSN. È il modulo di compliance a possedere quella mappatura, e può essere aggiornato in modo indipendente quando le normative cambiano.

## Pipeline di redaction

```
matched text
     │
     ▼
 style == "none"?  ─yes─►  raw snippet with context
     │ no
     ▼
 style == "full"?  ─yes─►  [REDACTED] with context
     │ no
     ▼
 _partial_redact()
     │
     ├─ 9+ digit number  ─►  *****6789  (mask all but last 4)
     ├─ email address     ─►  j****@example.com
     └─ generic string    ─►  keep last 25%
     │
     ▼
 _build_snippet()
     │
     └─ ±20 chars context  ─►  "...SSN: *****6789 for..."
```

La redaction parziale è il default perché dà agli analisti abbastanza informazioni per identificare il tipo di dato e la priorità di triage senza esporre l'intero valore sensibile. Le ultime 4 cifre di SSN e carte di credito sono considerate non sensibili da PCI-DSS (puoi stamparle sugli scontrini), quindi la redaction parziale per quei tipi è conforme.

## Architettura dell'analisi di rete

```
┌────────────────────────────────────────────┐
│              PCAP File                     │
│  (.pcap or .pcapng)                        │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│            pcap.read_pcap()                │
│                                            │
│  dpkt.pcap.Reader / dpkt.pcapng.Reader     │
│  Parse Ethernet -> IP -> TCP/UDP           │
│  Yield PacketInfo(src_ip, dst_ip,          │
│       src_port, dst_port, payload,         │
│       tcp_seq, tcp_flags)                  │
└────────────────┬───────────────────────────┘
                 │
         ┌───────┴───────┐
         ▼               ▼
┌─────────────┐  ┌───────────────┐
│FlowTracker  │  │DnsExfilDetector│
│             │  │               │
│Track by     │  │Label length   │
│4-tuple key  │  │check (>50)    │
│             │  │               │
│Reassemble   │  │Subdomain      │
│TCP streams  │  │entropy (>4.0) │
│by seq num   │  │               │
│             │  │QNAME length   │
│Dedup retx   │  │check (>100)   │
└──────┬──────┘  │               │
       │         │TXT volume     │
       ▼         │ratio check    │
┌─────────────┐  └───────┬───────┘
│Protocol ID  │          │
│(DPI)        │          ▼
│             │  ExfilIndicator[]
│HTTP: method │
│  prefix     │
│TLS: \x16\x03│
│SSH: SSH-    │
│SMTP: 220   │
└──────┬──────┘
       │
       ▼
  Reassembled text
  sent to DetectorRegistry
```

Il flow tracker crea chiavi di flusso bidirezionali ordinando le quadruple di andata e di ritorno, così `(A, B, 80, 12345)` e `(B, A, 12345, 80)` si mappano sullo stesso flusso. Il riassemblaggio TCP ordina i segmenti per numero di sequenza e deduplica le ritrasmissioni. Senza riassemblaggio, un numero di carta di credito diviso su due segmenti TCP verrebbe mancato.

Il detector di esfiltrazione DNS gira indipendentemente dai detector basati su regex. Analizza le query DNS alla ricerca di segnali di codifica: entropia simile a base64 nelle label dei sottodomini, label anormalmente lunghe, QNAME lunghi e rapporti sospetti nel volume di query TXT. La campagna APT di OilRig ha usato esattamente questi pattern per esfiltrare documenti rubati tramite DNS tunneling verso l'infrastruttura C2.

## Strategia di gestione degli errori

Gli errori vengono raccolti, non lanciati. Ogni scanner accoda i messaggi di errore a `ScanResult.errors` e continua a scansionare i target rimanenti. La CLI controlla `result.errors` dopo il completamento della scansione ed esce con codice 1 se si sono verificati errori, ma i risultati parziali vengono comunque riportati.

Questo approccio "raccogli e continua" significa che un singolo PDF corrotto in una directory di 10.000 file non interrompe la scansione. L'indagine sulla violazione di Equifax ha rilevato che gli strumenti di scansione che fallivano su singoli file spesso lasciavano intere directory non scansionate, ed è per questo che i moderni strumenti DLP trattano i fallimenti di estrazione come avvisi invece che come errori fatali.

## Considerazioni sulle prestazioni

**La scansione dei file** è I/O-bound. Lo scanner elabora i file in sequenza per evitare di sovraccaricare l'I/O del disco. L'estrazione del testo per i formati binari (PDF, Office) può essere CPU-intensive, ma questi file sono tipicamente una piccola frazione del totale.

**Il rilevamento** scala linearmente con la lunghezza del testo moltiplicata per il numero di regole. Con 29 regole e un chunk di testo medio di 500 righe, un singolo passaggio di rilevamento richiede microsecondi. Il detector di entropia è più costoso a causa della sua finestra scorrevole, quindi viene eseguito solo quando è abilitato e solo sui chunk di testo di alto livello (non sulle singole corrispondenze regex).

**La memoria** resta limitata grazie al chunking. L'estrattore di testo semplice legge 500 righe alla volta. L'estrazione degli archivi applica limiti di profondità e controlli sul rapporto anti zip bomb.

## Riferimento ai file chiave

- `cli.py` - Entry point, opzioni globali, app Typer
- `engine.py` - Orchestrazione, collega la configurazione agli scanner e ai reporter
- `config.py` - Modelli Pydantic, caricamento YAML, ricerca della configurazione
- `constants.py` - Tutti i numeri magici, le soglie, i literal di tipo
- `models.py` - Finding, Location, ScanResult, TextChunk
- `compliance.py` - Mappatura regola-framework, classificazione della severità
- `scoring.py` - Conversione condivisa da match a finding per tutti gli scanner
- `redaction.py` - Strategie di redaction parziale/completa/nessuna
- `detectors/registry.py` - Caricamento delle regole, filtraggio, pipeline di scoring
- `detectors/pattern.py` - Matching regex con allowlist e validazione dei checksum
- `detectors/context.py` - Boost di prossimità delle keyword, boost di co-occorrenza
- `detectors/entropy.py` - Rilevamento dell'entropia di Shannon, finestra scorrevole
- `detectors/rules/` - Definizioni delle regole (pii, financial, credentials, health)
- `extractors/` - Estrazione del testo da oltre 14 formati di file
- `scanners/` - Implementazioni delle scansioni di file, database e rete
- `network/` - Parsing PCAP, tracciamento dei flussi, DPI, esfiltrazione DNS
- `reporters/` - Output console, JSON, SARIF, CSV
- `commands/` - Implementazioni dei comandi CLI (scan, report)

## Prossimi passi

Ora che hai capito l'architettura:
1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per la panoramica del codice
2. Prova a modificare una regola di rilevamento in `detectors/rules/pii.py` per vedere come risponde la pipeline di scoring
