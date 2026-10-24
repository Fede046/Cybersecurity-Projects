# Architettura del sistema

Questo documento descrive come è progettato Portia e perché sono state prese certe decisioni architetturali. Seguiremo le richieste attraverso il sistema e spiegheremo i compromessi.

## Architettura ad alto livello

```
┌──────────────────────────────────────────────────────┐
│                       CLI                            │
│           root.go, scan.go, git.go                   │
└───────────────────────┬──────────────────────────────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
      ┌──────────────┐   ┌──────────────┐
      │   Directory   │   │     Git      │
      │    Source     │   │    Source     │
      │ directory.go  │   │   git.go     │
      └──────┬───────┘   └──────┬───────┘
             │                   │
             └─────────┬─────────┘
                       │
                       ▼ chan types.Chunk
              ┌─────────────────┐
              │    Pipeline     │
              │  pipeline.go    │
              ├─────────────────┤
              │                 │
              │  ┌───────────┐  │
              │  │ Worker 1  │  │
              │  │ detector  │  │
              │  └───────────┘  │
              │  ┌───────────┐  │
              │  │ Worker 2  │  │
              │  │ detector  │  │
              │  └───────────┘  │
              │  ┌───────────┐  │
              │  │ Worker N  │  │
              │  │ detector  │  │
              │  └───────────┘  │
              │                 │
              └────────┬────────┘
                       │
                       ▼ chan types.Finding
              ┌─────────────────┐
              │   Collector     │
              │  dedup + merge  │
              └────────┬────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
      ┌──────────────┐  ┌──────────────┐
      │  HIBP Check  │  │   Reporter   │
      │  (optional)  │  │  term/json/  │
      │  client.go   │  │    sarif     │
      └──────┬───────┘  └──────────────┘
             │                 ▲
             └─────────────────┘
```

## Suddivisione dei componenti

### Livello CLI (`internal/cli/`)

**Scopo:** Analizzare gli argomenti da riga di comando e orchestrare il workflow di scansione.

**Responsabilità:**
- Instradare i comandi `scan`, `git`, `init`, `pyproject`, `config` ai rispettivi handler
- Unire i flag della CLI con i valori del file di configurazione TOML (i flag della CLI hanno la precedenza)
- Creare la Source appropriata, eseguire la Pipeline, controllare opzionalmente HIBP, produrre l'output

**Interfacce:** Usa Cobra per il parsing degli argomenti. Ogni comando è un `cobra.Command` con una funzione `RunE`. La funzione `executeScan` in `scan.go` è condivisa tra i comandi `scan` e `git`.

### Config Loader (`internal/config/`)

**Scopo:** Caricare e unire la configurazione dai file `.portia.toml`.

**Responsabilità:**
- Cercare i file di configurazione in tre posizioni: la directory corrente, `.portia/config.toml`, `~/.config/portia/config.toml`
- Ripiegare su `pyproject.toml` (tabella `[tool.portia]`) quando non viene trovato alcun `.portia.toml`
- Fare il parsing del TOML in una struct `Config` con sezioni per Rules, Scan, Output, HIBP, Allowlist
- Fornire template predefiniti per `portia init` e `portia pyproject`

**Interfacce:** `Load(path string) (*Config, error)` restituisce una configurazione o un errore. Un path vuoto attiva l'auto-discovery.

### Interfaccia Source (`internal/source/`)

**Scopo:** Produrre chunk di testo da vari input (directory, history di git).

**Responsabilità:**
- Percorrere il filesystem o l'albero degli oggetti git
- Saltare file binari, percorsi esclusi, file troppo grandi
- Dividere il contenuto in chunk da 50 righe con metadati su percorso del file e numero di riga
- Inviare i chunk in un channel per il consumo da parte della pipeline

**Interfacce:**
```go
type Source interface {
    Chunks(ctx context.Context, out chan<- types.Chunk) error
    String() string
}
```

**Directory source** (`directory.go`):
- Usa `filepath.WalkDir` per l'attraversamento del filesystem
- Salta `.git`, `node_modules`, `vendor`, `__pycache__`, `.venv`
- Controlla la dimensione del file rispetto a un massimo configurabile (predefinito 1MB)
- Divide i file in segmenti da 50 righe usando uno scanner con buffer

**Git source** (`git.go`):
- Usa go-git v5 per operazioni git in-process
- `scanHistory`: percorre il log dei commit all'indietro, estrae il contenuto dei file dall'albero di ogni commit
- `scanStaged`: legge le voci dell'index di git per la scansione dei soli file in staging
- Supporta i filtri `--branch`, `--since`, `--depth`

### Rule Registry (`internal/rules/`)

**Scopo:** Memorizzare le regole di rilevamento e fornire una ricerca rapida basata su keyword.

**Responsabilità:**
- Memorizzare le regole in una map indicizzata per ID della regola
- Fornire `MatchKeywords(content)` che restituisce solo le regole le cui keyword compaiono nel contenuto
- Supportare l'abilitazione/disabilitazione delle regole
- Mantenere le allowlist globali di percorsi e valori

**Interfacce:** `Register(rule)`, `Get(id)`, `All()`, `MatchKeywords(content)`, `Disable(ids...)`, `Len()`

### Motore di rilevamento (`internal/engine/`)

**Scopo:** Applicare le regole ai chunk e produrre finding.

**Detector** (`detector.go`):
- Prende un chunk, esegue il pre-filtro per keyword tramite il registry
- Per ogni regola corrispondente, scansiona riga per riga con la regex
- Estrae il secret dal capture group
- Valida l'entropia se la regola ha una soglia di entropia
- Passa per FilterFinding per la riduzione dei falsi positivi

**Filter** (`filter.go`):
- `IsPlaceholder` - controlla rispetto ai pattern di GlobalValueAllowlist
- `IsTemplated` - cerca `${...}`, `{{...}}`, `os.getenv()`, `process.env.`
- `IsStopword` - divide il secret sui delimitatori `_-./`, controlla le parti rispetto a oltre 700 stopword
- `IsAllowedPath` - controlla il percorso del file rispetto a GlobalPathAllowlist
- `FilterFinding` - orchestra tutti i controlli, restituisce true se il finding è reale

**Pipeline** (`pipeline.go`):
- Crea un errgroup con una goroutine di source + N goroutine worker + una goroutine collector
- I worker prelevano i chunk dal channel, eseguono il detector, inviano i finding al channel dei finding
- Il collector unisce tutti i finding, deduplica per ruleID+filePath+secret+commitSHA

### Client HIBP (`internal/hibp/`)

**Scopo:** Controllare i secret rilevati rispetto al database di breach di Have I Been Pwned.

**Responsabilità:**
- Calcolo dell'hash SHA-1
- Query all'API con k-anonymity (prefisso di 5 caratteri)
- Cache LRU (10.000 voci) per le ricerche ripetute
- Circuit breaker (5 fallimenti = 60s di cooldown)
**Interfacce:** `Check(ctx, secret) (Result, error)`

### Reporter (`internal/reporter/`)

**Scopo:** Formattare i risultati della scansione per l'output.

**Terminal** (`terminal.go`): Output colorato con colori basati sulla severità (rosso per CRITICAL, giallo per MEDIUM), mascheramento dei secret (mostra i primi/ultimi caratteri), troncamento dello SHA per i commit git, stato dei breach HIBP.

**JSON** (`json.go`): JSON strutturato con un array `findings` e un oggetto `summary`. I secret sono mascherati nell'output.

**SARIF** (`sarif.go`): Output conforme a SARIF v2.1.0 con metadati del tool, definizioni delle regole, risultati con locations e properties.

**Interfacce:** Interfaccia `Reporter` con `Report(w io.Writer, result *types.ScanResult) error`. La funzione factory `New(format) Reporter` restituisce l'implementazione appropriata.

## Flusso dei dati

### Tracciamento: `portia scan ./myproject`

Walkthrough passo passo di cosa succede quando esegui una scansione di directory:

```
1. CLI parses arguments
   root.go:init() → cobra.OnInitialize(initConfig)
   scan.go:runScan() receives path="./myproject"

2. Config loading
   root.go:initConfig() → config.Load(cfgFile)
   Merges CLI flags with TOML config
   Format defaults to "terminal", maxSize defaults to 1MB

3. Registry setup
   scan.go:runScan() → rules.NewRegistry() + rules.RegisterBuiltins(reg)
   Loads 150 rules into the registry map
   Applies disabled rules from config: reg.Disable(cfg.Rules.Disable...)

4. Source creation
   scan.go:runScan() → source.NewDirectory(path, maxSize, excludes)
   Creates Directory struct with path, max file size, exclude patterns

5. Pipeline execution
   scan.go:executeScan() → engine.NewPipeline(reg).Run(ctx, src)

   5a. Source goroutine starts
       Calls src.Chunks(ctx, chunks)
       WalkDir traverses ./myproject
       Skips .git, node_modules, vendor, binary extensions
       Splits each file into 50-line chunks
       Sends each chunk into the chunks channel

   5b. Worker goroutines start (2-16 based on NumCPU)
       Each pulls chunks from the channel
       Calls detector.Detect(chunk):
         - reg.MatchKeywords(chunk.Content) → only rules with matching keywords
         - For each matched rule, scan each line with rule.Pattern regex
         - Extract secret from capture group
         - If rule has entropy threshold, compute Shannon entropy and compare
         - Run FilterFinding: IsPlaceholder → IsTemplated → IsStopword → path allowlist
         - If all checks pass, create Finding and send to findings channel

   5c. Collector goroutine
       Pulls findings from findings channel
       Appends to allFindings slice (mutex-protected)

   5d. Wait for all goroutines (errgroup.Wait)
       Dedup findings by ruleID+filePath+secret+commitSHA

6. HIBP verification (if --hibp flag)
   scan.go:checkHIBP(ctx, result)
   For each finding, calls client.Check(ctx, finding.Secret)
   Updates finding.HIBPStatus and finding.BreachCount

7. Reporter output
   scan.go:executeScan() → reporter.New(format).Report(os.Stdout, result)
   Terminal: colored table with severity, rule, file:line, masked secret
   JSON: structured JSON to stdout
   SARIF: SARIF v2.1.0 JSON to stdout
```

## Modello di concorrenza

La pipeline usa il pattern errgroup di Go per la concorrenza strutturata:

```
                    errgroup
                   ┌────────────────────────────────┐
                   │                                │
  Source goroutine │  ──chunks──▶  Worker 1          │
                   │              Worker 2          │
                   │              ...               │
                   │              Worker N          │
                   │              ──findings──▶     │
                   │                    Collector   │
                   │                                │
                   └────────────────────────────────┘
```

**Perché worker limitati?** Il matching regex, legato alla CPU, non beneficia di un parallelismo illimitato. Troppe goroutine in competizione per il tempo di CPU causano overhead di context switching. La formula `min(max(NumCPU, 2), 16)` dà 2 worker sulle macchine single-core e si ferma a 16 sui server grandi.

**Perché errgroup?** Offre due cose: (1) se una qualsiasi goroutine restituisce un errore, il context viene annullato e tutte le goroutine si concludono in modo pulito, e (2) `g.Wait()` si blocca finché tutte le goroutine non sono completate, dandoti un unico punto in cui controllare gli errori.

**Dimensionamento dei channel:** I channel hanno un buffer di `workers * 4`. Questo permette alla source di restare avanti rispetto ai worker (evitando di bloccarsi sugli invii) senza una crescita illimitata della memoria. Se i worker sono lenti, la source si bloccherà una volta riempito il buffer, fornendo un backpressure naturale.

**La danza di detectWg:** I worker condividono un `sync.WaitGroup` separato così da sapere quando tutto il rilevamento è terminato. La goroutine collector gira nello stesso errgroup ma si chiude solo dopo che tutti i worker hanno finito. Questo impedisce al collector di uscire prima del tempo mentre i finding sono ancora in produzione. Vedi `pipeline.go:52-77`.

## Risoluzione della configurazione

La configurazione viene risolta in questo ordine (i valori successivi sovrascrivono i precedenti):

```
1. Defaults (hardcoded)
   Format: "terminal"
   MaxSize: 1MB (1 << 20)
   Workers: min(max(NumCPU, 2), 16)
   HIBP: disabled
   Verbose: false
   NoColor: false

2. Config file (.portia.toml)
   Searched in order:
     .portia.toml (current directory)
     .portia/config.toml
     ~/.config/portia/config.toml
   First found is loaded. Later paths are not checked.
   If none found, falls back to pyproject.toml ([tool.portia] table).

3. CLI flags
   --format, --verbose, --no-color, --exclude, --max-size, --hibp, --config
   These always win over config file values.
```

Questa logica di merge si trova in `internal/cli/root.go:initConfig()`. Il pattern è: controllare se il flag della CLI è stato impostato esplicitamente (non zero/non vuoto) e ripiegare sul valore del file di configurazione solo se il flag non è stato impostato.

## Strategia di matching delle regole

La pipeline di rilevamento è ottimizzata per la velocità. Il matching regex è costoso, quindi l'obiettivo è evitare di eseguire regex su contenuti che non corrisponderanno mai.

```
Content chunk (50 lines of code)
        │
        ▼
┌───────────────────┐
│  Keyword Filter   │  ← O(rules * keywords) string.Contains
│  ~95% eliminated  │
└────────┬──────────┘
         │ Only rules whose keywords appear in this chunk
         ▼
┌───────────────────┐
│  Line-by-Line     │  ← O(lines * matched_rules) regex
│  Regex Matching   │
└────────┬──────────┘
         │ Raw matches with capture groups
         ▼
┌───────────────────┐
│  Secret Extract   │  ← Extract from capture group
│  + Entropy Check  │     Discard if below threshold
└────────┬──────────┘
         │ Validated candidates
         ▼
┌───────────────────┐
│  Filter Chain     │  ← IsPlaceholder → IsTemplated
│  5-layer check    │     → IsStopword → Allowlists
└────────┬──────────┘
         │ Real findings only
         ▼
      Finding
```

Il filtro per keyword è la principale ottimizzazione delle prestazioni. Se un chunk HTML di 50 righe non contiene stringhe come `password`, `secret`, `key`, `token`, `AKIA`, `ghp_`, `sk_live`, ecc., allora zero regole corrisponderanno e zero pattern regex dovranno essere eseguiti su di esso. In pratica, questo elimina la stragrande maggioranza dei chunk.
