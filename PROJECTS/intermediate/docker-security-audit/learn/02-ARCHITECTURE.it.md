# Come funziona lo scanner

Questo documento spiega l'architettura di docksec. Non come usarlo, ma come è costruito e perché sono state prese certe decisioni.

## Il quadro generale

Lo scanner segue una pipeline semplice:

```
Config → Docker Client → Analyzers → Findings → Filter → Reporter
```

1. Esegue il parsing dei flag CLI in una struct Config
2. Crea un'unica connessione del client Docker
3. Costruisce un elenco di analyzer in base ai target richiesti
4. Esegue tutti gli analyzer in modo concorrente con un worker pool
5. Raccoglie i rilevamenti da tutti gli analyzer
6. Filtra per gravità, se richiesto
7. Formatta e produce l'output tramite il reporter scelto

## Perché un unico client Docker

Il Docker SDK usa connessioni HTTP su un socket Unix. Creare più client significherebbe più connessioni, il che spreca risorse e può far raggiungere i limiti di connessione.

```go
// internal/docker/client.go
var (
    instance *Client
    once     sync.Once
    initErr  error
)

func NewClient() (*Client, error) {
    once.Do(func() {
        cli, err := client.NewClientWithOpts(
            client.FromEnv,
            client.WithAPIVersionNegotiation(),
        )
        // ...
        instance = &Client{api: cli}
    })
    return instance, initErr
}
```

Il `sync.Once` garantisce che esista un solo client per l'intero programma. Ogni chiamata a `NewClient()` restituisce la stessa istanza. È sicuro perché il client del Docker SDK è thread safe.

L'opzione `WithAPIVersionNegotiation()` è importante. I daemon e i client Docker possono avere versioni dell'API diverse. Senza negoziazione, un client più recente che parla con un daemon più vecchio fallirebbe. La negoziazione sceglie la versione più alta supportata da entrambe le parti.

## Modello di concorrenza

La scansione può essere lenta. Ogni ispezione di un container è un round trip verso il daemon Docker. Con decine di container, la scansione sequenziale richiede troppo tempo.

Lo scanner usa `golang.org/x/sync/errgroup` per l'esecuzione concorrente:

```go
// internal/scanner/scanner.go
func (s *Scanner) runAnalyzers(ctx context.Context, analyzers []analyzer.Analyzer) (finding.Collection, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(s.cfg.Workers)

    results := make(chan finding.Collection, len(analyzers))

    for _, a := range analyzers {
        a := a  // capture loop variable
        g.Go(func() error {
            if err := s.limiter.Wait(ctx); err != nil {
                return err
            }
            findings, err := a.Analyze(ctx)
            // ...
            results <- findings
            return nil
        })
    }
    // ...
}
```

### Perché errgroup invece delle goroutine grezze

Le goroutine grezze richiedono un coordinamento manuale:
- Ti serve un WaitGroup per sapere quando tutte le goroutine hanno finito
- Devi propagare manualmente gli errori
- La cancellazione del context è una tua responsabilità

errgroup gestisce tutto questo:
- `g.Wait()` si blocca finché tutte le goroutine non sono completate
- Se una qualsiasi goroutine restituisce un errore, il context viene cancellato
- `g.SetLimit(n)` limita il numero di goroutine concorrenti (worker pool integrato)

### La cattura della variabile del ciclo

```go
for _, a := range analyzers {
    a := a  // This line is crucial
    g.Go(func() error {
        // use a
    })
}
```

Senza `a := a`, tutte le goroutine condividerebbero la stessa variabile del ciclo. Nel momento in cui vengono eseguite, il ciclo è terminato e `a` punta all'ultimo analyzer. Ogni goroutine analizzerebbe la stessa cosa.

`a := a` crea una nuova variabile con scope limitato a ogni iterazione, catturando il valore corretto.

Nota: Go 1.22 ha corretto questo comportamento per i cicli `for`, ma questo codice supporta Go 1.21+ quindi la cattura è ancora necessaria.

## Rate limiting

Anche con un worker pool, puoi sovraccaricare il daemon Docker con troppe richieste concorrenti. Lo scanner usa un rate limiter a token bucket:

```go
limiter := rate.NewLimiter(
    rate.Limit(config.RateLimitPerSecond),  // 50/sec
    config.RateLimitBurst,                   // burst of 50
)
```

Prima che ogni analyzer venga eseguito, deve acquisire un token:

```go
if err := s.limiter.Wait(ctx); err != nil {
    return err
}
```

Il token bucket funziona così:
- Il bucket contiene fino a 50 token (burst)
- I token si ricaricano a 50 al secondo
- Ogni richiesta consuma un token
- Se non ci sono token disponibili, Wait() si blocca finché non ne compare uno

Questo attenua i picchi. Anche se 50 analyzer partono contemporaneamente, distribuiscono le loro chiamate alla Docker API nel tempo.

## L'interfaccia Analyzer

Tutti gli analyzer implementano la stessa interfaccia:

```go
// internal/analyzer/analyzer.go
type Analyzer interface {
    Name() string
    Analyze(ctx context.Context) (finding.Collection, error)
}
```

Questa astrazione permette allo scanner di trattare tutti gli analyzer in modo uniforme. Non gli importa se un analyzer ispeziona container, esegue il parsing di Dockerfile o interroga il daemon. Tutti accettano un context e restituiscono rilevamenti.

Aggiungere un nuovo analyzer è semplice:
1. Implementa l'interfaccia
2. Aggiungilo a `buildAnalyzers()` nello scanner

Lo scanner non importa mai tipi di analyzer specifici al di là della loro costruzione. Lavora solo con l'interfaccia.

## Come funziona l'analisi dei container

L'analyzer dei container mostra il flusso tipico:

```go
// internal/analyzer/container.go
func (a *ContainerAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    containers, err := a.client.ListContainers(ctx, true)
    if err != nil {
        return nil, err
    }

    var findings finding.Collection
    for _, c := range containers {
        info, err := a.client.InspectContainer(ctx, c.ID)
        if err != nil {
            continue  // Skip failed inspections
        }
        findings = append(findings, a.analyzeContainer(info)...)
    }
    return findings, nil
}
```

1. Elenca tutti i container (compresi quelli fermi)
2. Ispeziona ogni container per ottenerne la configurazione dettagliata
3. Esegue le verifiche di sicurezza sui dati dell'ispezione
4. Aggrega tutti i rilevamenti

Ogni metodo di verifica esamina campi specifici:

```go
func (a *ContainerAnalyzer) checkPrivileged(target finding.Target, info types.ContainerJSON) finding.Collection {
    if info.HostConfig.Privileged {
        control, _ := benchmark.Get("5.4")
        f := finding.New("CIS-5.4", control.Title, finding.SeverityCritical, target).
            WithDescription(control.Description).
            WithRemediation(control.Remediation).
            // ...
        return finding.Collection{f}
    }
    return nil
}
```

La verifica:
1. Esamina uno specifico campo di configurazione
2. Se è configurato male, recupera il controllo CIS per il contesto
3. Crea un rilevamento con tutti i metadati pertinenti

## Il tipo Finding

I rilevamenti contengono tutto il necessario per comprendere e risolvere un problema:

```go
// internal/finding/finding.go
type Finding struct {
    ID          string       // Unique hash for deduplication
    RuleID      string       // CIS control ID (e.g., "CIS-5.4")
    Title       string       // Short description
    Description string       // Full explanation
    Severity    Severity     // INFO, LOW, MEDIUM, HIGH, CRITICAL
    Category    string       // Grouping (Container Runtime, Dockerfile, etc.)
    Target      Target       // What was scanned (container:nginx, image:alpine, etc.)
    Location    *Location    // For files: path, line, column
    Remediation string       // How to fix it
    References  []string     // Links to documentation
    CISControl  *CISControl  // Original benchmark control
    Timestamp   time.Time    // When discovered
}
```

L'ID viene generato da un hash di regola, target e posizione:

```go
func (f *Finding) generateID() string {
    data := fmt.Sprintf("%s|%s|%s|%s", f.RuleID, f.Target.Type, f.Target.Name, f.Target.ID)
    if f.Location != nil {
        data += fmt.Sprintf("|%s:%d", f.Location.Path, f.Location.Line)
    }
    hash := sha256.Sum256([]byte(data))
    return hex.EncodeToString(hash[:8])
}
```

Questo rende i rilevamenti stabili tra una scansione e l'altra. Lo stesso problema produce lo stesso ID, il che è utile per tracciare la remediation nel tempo.

## Registry dei controlli CIS

Il CIS Docker Benchmark ha circa 150 controlli organizzati per sezione. Invece di inserirli hardcoded ovunque, risiedono in un registry centrale:

```go
// internal/benchmark/controls.go
var controlRegistry = make(map[string]Control)

func Register(c Control) {
    controlRegistry[c.ID] = c
}

func Get(id string) (Control, bool) {
    c, ok := controlRegistry[id]
    return c, ok
}
```

I controlli si registrano da soli durante `init()`:

```go
func init() {
    registerHostControls()
    registerDaemonControls()
    registerContainerRuntimeControls()
    // ...
}

func registerContainerRuntimeControls() {
    Register(Control{
        ID:          "5.4",
        Section:     "Container Runtime",
        Title:       "Ensure that privileged containers are not used",
        Severity:    finding.SeverityCritical,
        Description: "Privileged containers have all Linux kernel capabilities...",
        Remediation: "Do not run containers with --privileged flag...",
        // ...
    })
}
```

Quando un analyzer trova un problema, recupera il controllo:

```go
control, _ := benchmark.Get("5.4")
```

Questo mantiene i metadati delle regole in un unico posto. Se il benchmark CIS viene aggiornato, modifichi il registry, non ogni analyzer.

## Astrazione del reporter

I formati di output variano enormemente (colori nel terminale vs JSON vs SARIF XML), ma fanno tutti la stessa cosa: prendono i rilevamenti e producono un output.

```go
// internal/report/reporter.go
type Reporter interface {
    Report(findings finding.Collection) error
}
```

Ogni formato implementa questa interfaccia:
- `TerminalReporter` usa colori ANSI e tabelle
- `JSONReporter` esegue il marshaling in JSON
- `SARIFReporter` produce SARIF per GitHub Security
- `JUnitReporter` produce JUnit XML per la CI

Lo scanner sceglie un reporter all'avvio in base a `--output`:

```go
reporter, err := report.NewReporter(cfg.Output, cfg.OutputFile)
```

Aggiungere un nuovo formato significa implementare Reporter e aggiornare la funzione factory. Il resto della codebase resta invariato.

## Degradazione graduale

Alcune fonti di dati non sono sempre disponibili. Il filesystem `/proc` esiste solo su Linux. Alcuni container potrebbero non avere tutti i campi popolati.

Invece di fallire, lo scanner degrada in modo graduale:

```go
// internal/proc/proc.go
func GetProcessInfo(pid int) (*ProcessInfo, error) {
    // Status is required
    if err := info.readStatus(procPath); err != nil {
        return nil, fmt.Errorf("reading status: %w", err)
    }

    // These are optional - errors ignored
    if err := info.readCmdline(procPath); err != nil {
    }
    if err := info.readCgroups(procPath); err != nil {
    }
    // ...
}
```

Lo schema: richiedi i dati critici, ignora i fallimenti per i dati opzionali. Questo permette allo scanner di funzionare in ambienti limitati (container, sistemi non Linux, permessi ridotti) continuando a fornire valore.

## Filosofia di gestione degli errori

La codebase segue le convenzioni di Go:
- Restituisci gli errori risalendo lo stack delle chiamate
- Avvolgi gli errori con il contesto usando `fmt.Errorf("doing X: %w", err)`
- Lascia che siano i chiamanti a decidere cosa fare con gli errori

I fallimenti degli analyzer non fermano la scansione:

```go
findings, err := a.Analyze(ctx)
if err != nil {
    s.logger.Warn("analyzer failed", "name", a.Name(), "error", err)
    return nil  // Return nil error, not the actual error
}
```

Un analyzer guasto non dovrebbe impedire l'esecuzione degli altri. La scansione continua, registra il fallimento nel log e include tutti i rilevamenti andati a buon fine.

## Timeout

Le chiamate alla Docker API hanno dei timeout per evitare blocchi:

```go
// internal/config/constants.go
const (
    DefaultTimeout    = 30 * time.Second
    InspectTimeout    = 10 * time.Second
    ConnectionTimeout = 5 * time.Second
)

// internal/docker/client.go
func (c *Client) InspectContainer(ctx context.Context, containerID string) (types.ContainerJSON, error) {
    inspectCtx, cancel := context.WithTimeout(ctx, config.InspectTimeout)
    defer cancel()

    info, err := c.api.ContainerInspect(inspectCtx, containerID)
    // ...
}
```

Ogni operazione crea un context derivato con un timeout specifico. Se il daemon Docker è lento o bloccato, la chiamata va in timeout invece di restare bloccata per sempre.

Il `defer cancel()` è importante. Anche se la chiamata si completa prima del timeout, devi chiamare cancel per rilasciare le risorse del timer. Senza di esso, hai un leak di goroutine.
