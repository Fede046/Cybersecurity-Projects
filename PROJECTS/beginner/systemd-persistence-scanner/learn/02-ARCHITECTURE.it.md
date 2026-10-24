# Architettura del sistema

Questo documento analizza come è progettato sentinel e perché sono state prese certe decisioni architetturali.

## Architettura ad alto livello

```
┌────────────────────────────────────────────────────────────┐
│                     CLI Layer (cobra)                       │
│  cmd/sentinel/main.go → internal/cli/root.go               │
│  Subcommands: scan, baseline save, baseline diff            │
│  Flags: --json, --min-severity, --root, --ignore-file       │
└──────────────────────────┬─────────────────────────────────┘
                           │
              ┌────────────┼────────────────┐
              ▼            ▼                ▼
┌──────────────────┐ ┌───────────┐ ┌──────────────────┐
│  Scanner Registry │ │  Config   │ │  Baseline        │
│  scanner.RunAll() │ │  Ignore   │ │  Save/Load/Diff  │
│  17 modules       │ │  List     │ │  JSON snapshots   │
│  parallel via     │ │  Filter   │ │                   │
│  errgroup         │ │           │ │                   │
└────────┬─────────┘ └─────┬─────┘ └─────────┬─────────┘
         │                 │                  │
         ▼                 ▼                  ▼
┌──────────────────────────────────────────────────────────┐
│              []types.Finding                              │
│  Scanner, Severity, Title, Path, Evidence, MITRE          │
└──────────────────────────┬───────────────────────────────┘
                           │
              ┌────────────┼────────────────┐
              ▼            ▼                ▼
┌──────────────────┐ ┌──────────┐ ┌──────────────────┐
│  Terminal Report  │ │  JSON    │ │  UI (banner,     │
│  Color-coded      │ │  Report  │ │  spinner, colors)│
│  severity groups  │ │  stdout  │ │                   │
└──────────────────┘ └──────────┘ └──────────────────┘
```

### Analisi dei componenti

**CLI Layer (internal/cli/)**
- Scopo: effettuare il parsing dei flag, fare il dispatch verso i workflow di scan o baseline, formattare l'output
- Responsabilità: validazione dei flag, rilevamento dell'hostname, filtraggio per severità, selezione della modalità di output
- Interfacce: chiama scanner.RunAll(), config.LoadIgnoreFile(), baseline.Save/Load/Diff()

**Scanner Registry (internal/scanner/scanner.go)**
- Scopo: raccogliere tutti i moduli scanner ed eseguirli in parallelo
- Responsabilità: registrazione in fase di init, coordinamento delle goroutine, merge dei risultati
- Interfacce: espone Register(), All() e RunAll()

**Pattern Engine (internal/scanner/patterns.go)**
- Scopo: matching di regex centralizzato per contenuti sospetti attraverso tutti gli scanner
- Responsabilità: definizioni dei pattern compilati, ordinamento per severità, API di match a singola funzione
- Interfacce: MatchLine() ritorna (matched, severity, label)

**Config (internal/config/)**
- Scopo: caricare e applicare le regole di ignore per sopprimere i finding known-good
- Responsabilità: parsing di file simil-YAML, filtraggio dei finding per path/scanner/title

**Baseline (internal/baseline/)**
- Scopo: persistere i risultati delle scansioni e calcolare i diff tra gli snapshot
- Responsabilità: serializzazione JSON, deduplicazione tramite chiave composita

**Report (internal/report/)**
- Scopo: renderizzare i finding come output colorato del terminale o JSON strutturato
- Responsabilità: ordinamento per severità, mapping dei colori, troncamento dell'evidenza, statistiche di riepilogo

**UI (internal/ui/)**
- Scopo: presentazione a terminale (banner, spinner, colori, simboli)
- Responsabilità: funzioni di colore ANSI, animazione dello spinner braille, gestione del cursore

## Flusso dei dati

### Flusso del comando scan

Passo per passo di cosa succede quando l'utente esegue `sentinel scan`:

```
1. main.go imports internal/scanner (blank import)
   All 17 scanner init() functions call Register()
   Registry now holds 17 Scanner implementations

2. cobra dispatches to runScan() in cli/scan.go
   Parses --min-severity, --root, --ignore-file flags
   Starts spinner if not in JSON mode

3. scanner.RunAll(root) launches 17 goroutines via errgroup
   Each goroutine calls scanner.Scan(root)
   Each scanner reads files under root, applies MatchLine()
   Findings collected under mutex into shared slice

4. config.LoadIgnoreFile() → ignoreList.Filter(findings)
   Removes any findings matching ignore rules

5. filterBySeverity(findings, minSev)
   Drops findings below the requested threshold

6. types.Tally(filtered) counts per-severity totals

7. report.PrintTerminal() or report.PrintJSON()
   Terminal: sort by severity descending, color-code, print summary
   JSON: encode ScanResult to stdout
```

### Flusso del baseline diff

```
1. baseline save: RunAll() → baseline.Save()
   Serializes findings + hostname + version to JSON file

2. baseline diff: baseline.Load() → RunAll() → baseline.Diff()
   Loads saved snapshot
   Runs fresh scan
   Builds map of known findings by scanner|path|title key
   Returns only findings not present in baseline

3. Apply ignore-list filter and severity filter
   Same pipeline as regular scan
```

## Design pattern

### Il pattern dello Scanner Registry

**Cos'è:**
Un registry centrale a cui gli scanner si aggiungono da soli durante l'inizializzazione del package, disaccoppiando il registry dalla conoscenza delle specifiche implementazioni degli scanner.

**Come funziona:**

Ogni modulo scanner (systemd.go, cron.go, ecc.) chiama Register() nella sua funzione init():

```go
func init() {
    Register(&SystemdScanner{})
}
```

Il registry è semplicemente una slice:

```go
var registry []types.Scanner

func Register(s types.Scanner) {
    registry = append(registry, s)
}
```

L'entry point main.go importa il package scanner con un blank import:

```go
import _ "github.com/CarterPerez-dev/sentinel/internal/scanner"
```

Questo innesca tutte le funzioni init() nel package scanner, popolando il registry prima che main() venga eseguito.

**Perché l'abbiamo scelto:**
Aggiungere un nuovo scanner richiede zero modifiche al codice esistente. Crei il file, implementi l'interfaccia Scanner, chiami Register() in init(). Nessun'istruzione switch, nessuna factory function, nessun file di configurazione. Il compilatore Go gestisce la scoperta (discovery).

**Compromessi:**
- Pro: registrazione zero-touch, impossibile dimenticare di registrare (è nello stesso file dello scanner)
- Contro: effetti collaterali in fase di init, l'ordine di registrazione dipende dall'ordine di sort dei nomi dei file (irrilevante poiché gli scanner vengono eseguiti in parallelo)

### Esecuzione parallela con errgroup

**Cos'è:**
Tutti i 17 scanner vengono eseguiti concorrentemente in goroutine separate, coordinate da golang.org/x/sync/errgroup.

**Come funziona:**

```go
var (
    mu  sync.Mutex
    all []types.Finding
    g   errgroup.Group
)

for _, s := range registry {
    g.Go(func() error {
        results := s.Scan(root)
        mu.Lock()
        all = append(all, results...)
        mu.Unlock()
        return nil
    })
}

_ = g.Wait()
```

Ogni goroutine di scanner legge indipendentemente il proprio insieme di file, applica il pattern matching e raccoglie i finding. Il mutex protegge la slice condivisa dei finding. errgroup.Wait() blocca finché tutte le goroutine non sono completate.

**Perché l'abbiamo scelto:**
Gli scanner sono I/O-bound (leggono file dal disco). Eseguirli in parallelo significa che il tempo totale di scansione è all'incirca il tempo dello scanner più lento, non la somma di tutti i 17. Su un sistema con SSD, questo riduce drasticamente il tempo di scansione.

**Compromessi:**
- Pro: accelerazione quasi lineare per lavoro I/O-bound, coordinamento semplice
- Contro: i finding arrivano in ordine non deterministico (ordinati prima della visualizzazione)

### Pattern matching ordinato per severità

**Cos'è:**
Una singola funzione MatchLine() che testa una riga di testo rispetto a tutti i 16 pattern compilati e ritorna la corrispondenza con la severità più alta.

**Come funziona:**

```go
func MatchLine(line string) (matched bool, sev Severity, label string) {
    best := SeverityInfo
    for _, p := range SuspiciousPatterns {
        if p.Pattern.MatchString(line) {
            if !matched || p.Severity > best {
                best = p.Severity
                label = p.Label
            }
            matched = true
        }
    }
    return matched, best, label
}
```

Quando una riga corrisponde a più pattern (un curl reindirizzato in pipe a bash corrisponde sia a NetworkToolPattern sia a DownloadExecPattern), viene riportata solo la severità più alta. Questo previene il doppio conteggio e garantisce che i finding riflettano l'interpretazione più pericolosa.

**Perché l'abbiamo scelto:**
Centralizzare i pattern significa che gli scanner non duplicano le definizioni regex. Aggiungere un nuovo pattern al motore lo applica automaticamente ovunque: systemd, cron, profile, udev e ogni altro scanner che chiama MatchLine().

## Separazione in layer

```
┌────────────────────────────────────┐
│    Layer 1: pkg/types              │
│    - Domain types only             │
│    - No imports from internal      │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    Layer 2: internal/scanner       │
│    - File I/O and pattern matching │
│    - Returns []types.Finding       │
│    - No knowledge of CLI or output │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    Layer 3: internal/cli           │
│    - Orchestrates scan pipeline    │
│    - Applies filters               │
│    - Dispatches to report layer    │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    Layer 4: internal/report + ui   │
│    - Presentation only             │
│    - Terminal colors, JSON encoding│
└────────────────────────────────────┘
```

### Cosa vive dove

**pkg/types:** costanti di Severity, struct Finding, struct ScanResult, interfaccia Scanner. Importato da tutto. Non importa nulla da internal.

**internal/scanner:** tutte le 17 implementazioni degli scanner, il motore di pattern, gli helper del filesystem, il registry. Importa solo pkg/types. Non ha alcuna conoscenza dei flag CLI, del formato di output o del filtraggio.

**internal/config:** caricamento e filtraggio della ignore-list. Importa pkg/types.

**internal/baseline:** persistenza degli snapshot e calcolo del diff. Importa pkg/types.

**internal/cli:** definizioni dei comandi Cobra, parsing dei flag, orchestrazione della scansione. Importa scanner, config, baseline, report, ui.

**internal/report + ui:** formattazione a terminale, encoding JSON, funzioni di colore, spinner. Importa pkg/types e ui.

## Modelli di dati

### Finding

```go
type Finding struct {
    Scanner  string   `json:"scanner"`
    Severity Severity `json:"severity"`
    Title    string   `json:"title"`
    Path     string   `json:"path"`
    Evidence string   `json:"evidence"`
    MITRE    string   `json:"mitre"`
}
```

**Campi:**
- `Scanner`: quale modulo ha prodotto questo finding ("systemd", "cron", "ssh", ecc.)
- `Severity`: enum da Info (0) a Critical (4), si serializza come int nel JSON
- `Title`: descrizione leggibile da un umano di ciò che è stato trovato
- `Path`: percorso assoluto del filesystem al file che contiene il finding
- `Evidence`: la riga o il contenuto effettivo che ha innescato il finding, troncato per la visualizzazione
- `MITRE`: ID di tecnica ATT&CK (ad esempio, "T1543.002")

### ScanResult

```go
type ScanResult struct {
    Version    string        `json:"version"`
    ScanTime   time.Time     `json:"scan_time"`
    Hostname   string        `json:"hostname"`
    Findings   []Finding     `json:"findings"`
    Summary    SeverityCount `json:"summary"`
    DurationMs int64         `json:"duration_ms"`
}
```

Questo è l'output completo di una scansione, usato sia dal renderer del terminale sia dall'encoder JSON. Include i metadati (versione, hostname, timing) insieme ai finding e ai conteggi di severità precalcolati.

### Interfaccia Scanner

```go
type Scanner interface {
    Name() string
    Scan(root string) []Finding
}
```

Ogni modulo scanner implementa questa interfaccia a due metodi. Name() ritorna un identificatore leggibile da un umano. Scan() prende un percorso root del filesystem e ritorna tutti i finding. Il parametro root abilita la scansione di filesystem montati, chroot o directory di fixture di test invece del sistema vivo.

## Decisioni di design

### Perché un package scanner piatto invece di sub-package

Tutti i 17 scanner vivono in `internal/scanner/` come file separati nello stesso package. Un'alternativa sarebbe `internal/scanner/systemd/`, `internal/scanner/cron/`, ecc.

**Cosa abbiamo scelto:** package singolo con un file per scanner.

**Perché:** gli scanner condividono helper (ReadLines, ListFiles, ResolveRoot, FindUserDirs, ScanFileForPatterns), il motore di pattern (MatchLine, tutte le regex compilate) e il registry (Register). Metterli in package separati richiederebbe di esportare tutti questi o di creare un package di utilità condivise. Un singolo package mantiene il codice condiviso non esportato e co-localizzato.

**Compromessi:** il package scanner ha molti file, ma ogni file è autocontenuto e focalizzato. Il pattern di registrazione init() significa che non c'è una "lista di scanner" centrale da mantenere.

### Perché nessuna dipendenza esterna per la logica di scansione

Il package scanner usa solo la libreria standard (os, path/filepath, strings, regexp, bufio) più pkg/types. Le uniche dipendenze esterne sono nei layer CLI e UI (cobra, fatih/color) e nel layer di concorrenza (errgroup).

**Perché:** la logica di scansione deve essere affidabile. Dipendenze esterne nel percorso di detection potrebbero introdurre rischio nella supply chain. Mantenere il motore di detection core privo di dipendenze significa che può essere verificato leggendo il codice della libreria standard di Go.

### Perché regex compilate in variabili a livello di package

Pattern come `ReverseShellPattern` e `DownloadExecPattern` sono compilati una volta all'inizializzazione del package come dichiarazioni `var` con `regexp.MustCompile()`. Un'alternativa sarebbe compilarli al primo utilizzo o passarli come parametri.

**Perché:** le regex vengono compilate esattamente una volta quando il package viene caricato. Ogni successiva chiamata a MatchLine() usa l'automa compilato. Le var a livello di package sono sicure per le letture concorrenti, e tutte le scritture avvengono prima che main() venga eseguito.

## Estensibilità

### Aggiungere un nuovo scanner

1. Crea un nuovo file in internal/scanner/ (ad esempio, `docker.go`)
2. Definisci una struct che implementa types.Scanner:
   ```go
   type DockerScanner struct{}

   func (d *DockerScanner) Name() string { return "docker" }
   func (d *DockerScanner) Scan(root string) []types.Finding { ... }
   ```
3. Registra in init():
   ```go
   func init() { Register(&DockerScanner{}) }
   ```
4. Crea il file di test `docker_test.go` con le fixture in testdata

Nessun altro file deve cambiare. Il registry lo scopre automaticamente.

### Aggiungere un nuovo pattern

Aggiungi a `internal/scanner/patterns.go`:

```go
var NewPattern = regexp.MustCompile(`...`)
```

Aggiungi alla slice SuspiciousPatterns:

```go
{NewPattern, types.SeverityHigh, "description of what this detects"},
```

Ogni scanner che chiama MatchLine() o ScanFileForPatterns() inizierà immediatamente a controllare il nuovo pattern.

## Riferimento ai file chiave

- `cmd/sentinel/main.go` - Entry point, il blank import innesca la registrazione degli scanner
- `pkg/types/types.go` - Tutti i tipi di dominio (Finding, Severity, interfaccia Scanner)
- `internal/scanner/scanner.go` - Registry e RunAll() parallelo
- `internal/scanner/patterns.go` - Tutti i 16 pattern regex compilati e MatchLine()
- `internal/scanner/helpers.go` - Utilità condivise del filesystem e ScanFileForPatterns()
- `internal/cli/root.go` - Comando root di Cobra e flag globali
- `internal/cli/scan.go` - Orchestrazione del sottocomando scan
- `internal/cli/baseline.go` - Sottocomandi baseline save/diff
- `internal/baseline/baseline.go` - Persistenza degli snapshot JSON e diff
- `internal/config/config.go` - Caricamento e filtraggio della ignore-list

## Prossimi passi

Ora che hai compreso l'architettura:
1. Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per una panoramica del motore di pattern, delle implementazioni degli scanner e del codice del baseline diffing
2. Prova a eseguire `sentinel scan --root testdata` e ripercorri l'output fino al codice sorgente dello scanner per vedere l'architettura in azione
