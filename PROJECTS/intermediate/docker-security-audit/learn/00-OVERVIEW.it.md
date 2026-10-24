# Guida alla codebase

Una panoramica della struttura del codice e di cosa fa ciascun package. Inizia da qui se vuoi capire come si incastrano i vari pezzi.

## Struttura delle directory

```
cmd/docksec/          Entry point and CLI commands
internal/
  analyzer/           Security checks for different target types
  benchmark/          CIS Docker Benchmark control definitions
  config/             Runtime configuration and constants# Docker Security Audit Tool (docksec)
```

## Che cos'è

Un tool CLI basato su Go che scansiona gli ambienti Docker alla ricerca di configurazioni di sicurezza errate e le convalida rispetto al CIS Docker Benchmark. Analizza i container in esecuzione, le impostazioni del daemon, le image, i Dockerfile e i file docker-compose per identificare vulnerabilità come container privilegiati, capability pericolose, mount del socket Docker e segreti hardcoded.

## Perché è importante

I container Docker non forniscono sicurezza di default. Una singola configurazione errata può dare a un attaccante il controllo completo del tuo sistema host. Questo tool intercetta questi errori prima che diventino violazioni.

**Scenari reali in cui si applica:**

- **Container escape tramite mount del socket Docker**: nel 2018, il cluster Kubernetes di Tesla è stato compromesso quando gli attaccanti hanno trovato un pod con `/var/run/docker.sock` montato, che ha permesso loro di evadere verso l'host e minare criptovaluta.

- **Privilege escalation tramite capability**: l'exploit "Shocker" del 2014 usava `CAP_DAC_READ_SEARCH` per leggere file arbitrari dal filesystem dell'host, aggirando completamente l'isolamento del container.

- **Esposizione di segreti nelle image**: nel 2019, oltre 100.000 image Docker su Docker Hub contenevano chiavi API, password e chiavi private incorporate, individuabili tramite semplici ricerche testuali.

## Cosa imparerai

Questo progetto ti insegna come funziona davvero la sicurezza dei container dietro le quinte. Costruendolo ed estendendolo, capirai:

**Concetti di sicurezza:**
- **Conformità al CIS Docker Benchmark** - controlli di sicurezza standard del settore che coprono la configurazione dell'host, le impostazioni del daemon, le image e il runtime. Imparerai quali degli oltre 100 controlli contano di più e perché.
- **Il modello delle capability di Linux** - come funzionano i 41 privilegi distinti, quali consentono la container escape (come `CAP_SYS_ADMIN` e `CAP_SYS_PTRACE`) e come sottoporli ad audit in modo programmatico.
- **I confini di isolamento dei namespace** - da cosa proteggono i namespace PID, network, IPC e mount, e come la condivisione dei namespace dell'host infrange quella protezione.
- **Profili di sicurezza (seccomp, AppArmor, SELinux)** - come il filtraggio delle syscall e il mandatory access control prevengono effettivamente gli attacchi, non solo come concetti teorici di defense-in-depth.
- **Tecniche di rilevamento dei segreti** - pattern matching con regex, analisi dell'entropia con l'entropia di Shannon, e perché servono entrambe per intercettare diversi tipi di credenziali trapelate.

**Competenze tecniche:**
- **Interazione con la Docker API** - usare la libreria client ufficiale di Docker per ispezionare container, image e configurazione del daemon senza comandi shell.
- **Analisi statica dei file di build** - fare il parsing dei Dockerfile con un AST (abstract syntax tree) per rilevare problemi di sicurezza a build time, prima ancora che le image vengano eseguite.
- **Scansione concorrente** - usare errgroup e i rate limiter di Go per scansionare centinaia di container in modo efficiente senza sovraccaricare il daemon.
- **Formati di reporting di sicurezza** - generare SARIF (Static Analysis Results Interchange Format) per GitHub Security, JUnit per CI/CD e JSON strutturato per l'automazione.

**Strumenti e tecniche:**
- **Il parser di moby/buildkit** - lo stesso parser che Docker usa per comprendere la sintassi dei Dockerfile, che ti dà accesso ai metadati delle istruzioni e ai numeri di riga per rilevamenti precisi.
- **Mappatura sul CIS Benchmark** - tradurre i controlli di sicurezza in verifiche automatizzate, incluso determinare quali rilevamenti sono "scored" e quali "not scored" nei report di conformità.
- **Attraversamento dei nodi YAML** - navigare i file docker-compose come dati strutturati anziché come testo, gestendo sia il formato mapping sia il formato sequence per la stessa configurazione logica.

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**
- **Basi di Go** - struct, interfacce, goroutine, channel. Leggerai e modificherai codice Go che usa errgroup per la concorrenza e rate limiter per limitare le chiamate API.
- **Fondamenti di Docker** - image vs container, come funzionano i volumi, a cosa servono le porte. Devi conoscere la differenza tra `docker run --privileged` e `--cap-add=NET_ADMIN`.
- **Modello di sicurezza di Linux** - cosa può fare root, concetti di base sui permessi, perché eseguire come UID 0 è pericoloso. Comprendere `/proc` e `/sys` aiuta ma non è obbligatorio.
- **Dimestichezza con la riga di comando** - saper usare con disinvoltura `docker inspect`, leggere output JSON, comprendere gli exit code e la differenza tra stderr e stdout.

**Strumenti di cui avrai bisogno:**
- **Go 1.23+** - il progetto usa i generics e pattern di gestione degli errori più recenti
- **Docker Engine** - in esecuzione in locale o accessibile tramite `DOCKER_HOST`. Docker Desktop su Mac/Windows va benissimo.
- **Make** (opzionale) - per eseguire comodamente i comandi di build e test
- **Git** - per clonare il repository e tracciare le tue modifiche

**Utile ma non obbligatorio:**
- Esperienza con strumenti di scansione di sicurezza come Trivy, Snyk o Anchore
- Familiarità con i PDF dei CIS Benchmark (il tool ti insegna i controlli)
- Conoscenza dei test e dei benchmark in Go

## Avvio rapido

Fai girare il progetto in locale:

```bash
# Clone and navigate
cd PROJECTS/intermediate/docker-security-audit

# Build the binary
go build -o docksec ./cmd/docksec

# Scan all running containers
./docksec scan

# Scan specific targets
./docksec scan --target containers
./docksec scan --target daemon
./docksec scan --target images

# Scan a Dockerfile
./docksec scan --file Dockerfile

# Scan a docker-compose file
./docksec scan --file docker-compose.yml

# Generate JSON output for automation
./docksec scan --output json --output-file results.json

# Generate SARIF for GitHub Security
./docksec scan --output sarif --output-file results.sarif

# Filter by severity
./docksec scan --severity high,critical

# Fail CI builds on findings
./docksec scan --fail-on medium
```

Output atteso: un report nel terminale raggruppato per categoria (Container Runtime, Docker Daemon, ecc.) che mostra i rilevamenti con livelli di gravità, ID dei controlli CIS, descrizioni e passi di remediation. Zero rilevamenti se il tuo ambiente Docker è adeguatamente protetto.

## Struttura del progetto

```
docker-security-audit/
├── cmd/
│   └── docksec/
│       └── main.go              # CLI entry point, Cobra commands
├── internal/
│   ├── analyzer/
│   │   ├── analyzer.go          # Analyzer interface
│   │   ├── container.go         # Running container checks
│   │   ├── daemon.go            # Docker daemon config checks
│   │   ├── image.go             # Image metadata checks
│   │   ├── dockerfile.go        # Dockerfile static analysis
│   │   └── compose.go           # docker-compose.yml checks
│   ├── benchmark/
│   │   └── controls.go          # CIS Docker Benchmark control registry
│   ├── config/
│   │   ├── config.go            # Configuration struct and filters
│   │   └── constants.go         # Rate limits, timeouts, thresholds
│   ├── docker/
│   │   └── client.go            # Docker API client wrapper
│   ├── finding/
│   │   └── finding.go           # Finding data model and collection
│   ├── proc/
│   │   ├── capabilities.go      # Linux capabilities parsing
│   │   ├── proc.go              # /proc filesystem reading
│   │   └── security.go          # Security profile inspection
│   ├── report/
│   │   ├── reporter.go          # Reporter interface and factory
│   │   ├── terminal.go          # Human-readable output
│   │   ├── json.go              # Structured JSON output
│   │   ├── sarif.go             # SARIF 2.1.0 format
│   │   └── junit.go             # JUnit XML for CI/CD
│   ├── rules/
│   │   ├── capabilities.go      # 41 Linux capabilities with risk levels
│   │   ├── paths.go             # 200+ sensitive host paths
│   │   └── secrets.go           # 80+ secret patterns and entropy
│   └── scanner/
│       └── scanner.go           # Main scan orchestration
├── Dockerfile                   # Multi-stage build for container deployment
├── go.mod                       # Dependencies
└── go.sum
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per imparare come funzionano davvero a livello di kernel le capability di Linux, i namespace e i profili di sicurezza.

2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere come lo scanner orchestra gli analyzer concorrenti, applica le regole e genera i rilevamenti.

3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per spiegazioni dettagliate dell'implementazione con il codice reale del progetto.

4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee specifiche su come aggiungere funzionalità come il monitoraggio a runtime, l'integrazione con Kubernetes e le policy personalizzate.

## Problemi comuni

**Daemon Docker non accessibile**
```
Error: docker daemon not accessible: Cannot connect to the Docker daemon
```
Soluzione: verifica che Docker sia in esecuzione (`docker ps` funziona), che il tuo utente faccia parte del gruppo `docker`, oppure imposta `DOCKER_HOST` se usi un Docker remoto.

**Nessun rilevamento su container notoriamente vulnerabili**
Soluzione: controlla i flag `--severity` e `--target`. Per impostazione predefinita vengono scansionati tutti i target a tutte le gravità. Usa `--verbose` per vedere quali analyzer vengono eseguiti.

**La build fallisce con "module not found"**
Soluzione: esegui `go mod download` per scaricare le dipendenze. Assicurati di usare Go 1.23 o successivo (`go version`).

**Il file SARIF non compare in GitHub Security**
Soluzione: caricalo in un workflow di GitHub Actions usando `github/codeql-action/upload-sarif@v2`. I file SARIF non vengono caricati automaticamente: richiedono un'integrazione CI esplicita.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:

- **Trivy** - scanner di vulnerabilità completo che include la scansione della configurazione. Questo progetto si concentra specificamente sulla sicurezza Docker a runtime e a build time.
- **Falco** - monitoraggio della sicurezza a runtime tramite eBPF. Completa questo tool rilevando le minacce durante l'esecuzione.
- **Docker Bench Security** - il controllo di sicurezza ufficiale di Docker. Questo progetto lo migliora con output strutturato, integrazione CI/CD e accesso programmatico.
  docker/             Docker SDK wrapper
  finding/            The Finding type and severity levels
  parser/             Dockerfile and compose file parsers
  proc/               Linux /proc filesystem inspection
  report/             Output formatters (terminal, JSON, SARIF, JUnit)
  rules/              Security rule data (capabilities, paths, secrets)
  scanner/            Orchestration layer that ties everything together
```

## cmd/docksec

La CLI è costruita con Cobra. Ogni comando è un file separato.

`main.go` definisce le variabili di versione che vengono sovrascritte a build time:

```go
var (
    version   = "dev"
    commit    = "none"
    buildDate = "unknown"
)
```

Quando esegui `go build -ldflags "-X main.version=1.0.0"`, il compilatore sostituisce il letterale stringa prima di creare il binario.

`scan.go` è dove inizia il lavoro vero e proprio. Crea una Config a partire dai flag, istanzia uno Scanner e chiama Run:

```go
cfg := &config.Config{
    Targets:  targets,
    Files:    files,
    Output:   outputFormat,
    // ...
}
scanner, _ := scanner.New(cfg)
scanner.Run(ctx)
```

## internal/scanner

Questo è il livello di orchestrazione. Crea gli analyzer in base alla configurazione, li esegue in modo concorrente, raccoglie i rilevamenti, li filtra e li invia a un reporter.

Il modello di concorrenza usa `errgroup` con un semaforo:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(s.cfg.Workers)  // Max concurrent goroutines

for _, a := range analyzers {
    a := a
    g.Go(func() error {
        s.limiter.Wait(ctx)  // Rate limit Docker API calls
        findings, _ := a.Analyze(ctx)
        results <- findings
        return nil
    })
}
```

Il rate limiter impedisce di sovraccaricare il daemon Docker. Anche se imposti 50 worker, questi distribuiscono le loro chiamate API nel tempo.

## internal/analyzer

Ogni analyzer implementa questa interfaccia:

```go
type Analyzer interface {
    Name() string
    Analyze(ctx context.Context) (finding.Collection, error)
}
```

Ci sono cinque implementazioni:

| File | Target | Cosa verifica |
|------|--------|----------------|
| container.go | Container in esecuzione | Modalità privilegiata, capability, mount, namespace, profili di sicurezza, limiti delle risorse |
| image.go | Image locali | Istruzione User, segreti nella history, tag delle image di base |
| daemon.go | Daemon Docker | Registry insicuri, ICC, user namespace, funzionalità sperimentali |
| dockerfile.go | File Dockerfile | Istruzione USER, ADD vs COPY, segreti in ENV/ARG, HEALTHCHECK |
| compose.go | File Compose | Le stesse verifiche dei container, ma per le definizioni dei servizi |

L'analyzer dei container è il più complesso. Elenca tutti i container, ispeziona ciascuno ed esegue le verifiche:

```go
func (a *ContainerAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    containers, _ := a.client.ListContainers(ctx, true)

    var findings finding.Collection
    for _, c := range containers {
        info, _ := a.client.InspectContainer(ctx, c.ID)
        findings = append(findings, a.analyzeContainer(info)...)
    }
    return findings, nil
}

func (a *ContainerAnalyzer) analyzeContainer(info types.ContainerJSON) finding.Collection {
    // Each method checks one thing
    findings = append(findings, a.checkPrivileged(target, info)...)
    findings = append(findings, a.checkCapabilities(target, info)...)
    findings = append(findings, a.checkMounts(target, info)...)
    // ...
}
```

Ogni verifica recupera il controllo CIS pertinente per ottenerne i metadati:

```go
func (a *ContainerAnalyzer) checkPrivileged(...) finding.Collection {
    if info.HostConfig.Privileged {
        control, _ := benchmark.Get("5.4")
        f := finding.New("CIS-5.4", control.Title, finding.SeverityCritical, target).
            WithDescription(control.Description).
            WithRemediation(control.Remediation)
        return finding.Collection{f}
    }
    return nil
}
```

## internal/benchmark

Contiene tutti i controlli del CIS Docker Benchmark v1.6.0 come struct Go. Si registrano da soli durante l'init:

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
        Scored:      true,
        Level:       1,
    })
}
```

Il registry globale consente la ricerca per ID:

```go
control, ok := benchmark.Get("5.4")
```

Questo mantiene i metadati delle regole in un unico posto. Se il CIS aggiorna il benchmark, modifichi il registry, non ogni analyzer.

## internal/finding

La struct Finding contiene tutto ciò che riguarda un problema rilevato:

```go
type Finding struct {
    ID          string       // Hash for deduplication
    RuleID      string       // CIS control ID
    Title       string       // Short description
    Description string       // Full explanation
    Severity    Severity     // INFO, LOW, MEDIUM, HIGH, CRITICAL
    Category    string       // Container Runtime, Dockerfile, etc.
    Target      Target       // What was scanned
    Location    *Location    // For files: path and line number
    Remediation string       // How to fix it
    References  []string     // Documentation links
    CISControl  *CISControl  // Original benchmark control
    Timestamp   time.Time    // When discovered
}
```

I rilevamenti vengono creati con un builder pattern:

```go
f := finding.New("CIS-5.4", "Privileged container", finding.SeverityCritical, target).
    WithDescription("...").
    WithRemediation("...").
    WithReferences("https://...")
```

L'ID viene generato da un hash di regola, target e posizione. Questo rende i rilevamenti stabili tra una scansione e l'altra:

```go
func (f *Finding) generateID() string {
    data := fmt.Sprintf("%s|%s|%s|%s", f.RuleID, f.Target.Type, f.Target.Name, f.Target.ID)
    hash := sha256.Sum256([]byte(data))
    return hex.EncodeToString(hash[:8])
}
```

Lo stesso problema sullo stesso target produce lo stesso ID. È utile per tracciare la remediation nel tempo.

## internal/rules

Contiene i dati delle regole di sicurezza organizzati per categoria:

### capabilities.go

Mappa tutte le oltre 40 capability di Linux su gravità e descrizioni:

```go
var Capabilities = map[string]CapabilityInfo{
    "CAP_SYS_ADMIN": {
        Severity:    finding.SeverityCritical,
        Description: "Effectively root - mount filesystems, quotas, namespaces...",
    },
    "CAP_NET_RAW": {
        Severity:    finding.SeverityMedium,
        Description: "Craft arbitrary packets, ARP/DNS spoofing, packet sniffing.",
    },
    // ...
}
```

Le mappe di lookup precalcolate rendono veloci le verifiche:

```go
var dangerousCapabilities = func() map[string]struct{} {
    m := make(map[string]struct{})
    for cap, info := range Capabilities {
        if info.Severity >= finding.SeverityHigh {
            m[cap] = struct{}{}
        }
    }
    return m
}()

func IsDangerousCapability(cap string) bool {
    _, exists := dangerousCapabilities[strings.ToUpper(cap)]
    return exists
}
```

### paths.go

Cataloga oltre 200 percorsi sensibili dell'host con gravità e descrizioni:

```go
var SensitiveHostPaths = map[string]PathInfo{
    "/etc/shadow": {
        Severity:    finding.SeverityCritical,
        Description: "Password hashes. Direct credential access.",
    },
    "/var/run/docker.sock": {
        Severity:    finding.SeverityCritical,
        Description: "Docker daemon socket. Full control over Docker, container escape possible.",
    },
    // ...
}
```

Il matching dei percorsi gestisce i prefissi:

```go
func IsSensitivePath(path string) bool {
    if _, exists := sensitivePathLookup[path]; exists {
        return true
    }
    // Check if path is under a sensitive directory
    for sensitivePath := range sensitivePathLookup {
        if strings.HasPrefix(path, sensitivePath+"/") {
            return true
        }
    }
    return false
}
```

Quindi `/etc/shadow` corrisponde, ma corrisponde anche `/etc/foo` (perché `/etc` è sensibile).

### secrets.go

Pattern matching per i segreti nei Dockerfile:

```go
var SecretPatterns = []SecretPattern{
    {
        Type:        SecretTypeAWSKey,
        Pattern:     regexp.MustCompile(`(?i)(AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16}`),
        Description: "AWS Access Key ID",
    },
    {
        Type:        SecretTypeGitHub,
        Pattern:     regexp.MustCompile(`(?i)(ghp|gho|ghu|ghs|ghr)_[A-Za-z0-9_]{36,255}`),
        Description: "GitHub Personal Access Token",
    },
    // 100+ more patterns
}
```

Include anche il calcolo dell'entropia per rilevare stringhe casuali che potrebbero essere segreti:

```go
func CalculateEntropy(s string) float64 {
    freq := make(map[rune]float64)
    for _, c := range s {
        freq[c]++
    }
    var entropy float64
    for _, count := range freq {
        p := count / float64(len(s))
        entropy -= p * math.Log2(p)
    }
    return entropy
}
```

Le stringhe ad alta entropia (caratteri casuali) sono probabilmente segreti. Le stringhe a bassa entropia (pattern ripetuti) non lo sono.

## internal/parser

### dockerfile.go

Esegue il parsing dei Dockerfile in un AST usando il parser di BuildKit:

```go
func ParseDockerfile(path string) (*DockerfileAST, error) {
    file, _ := os.Open(path)
    result, _ := parser.Parse(file)

    ast := &DockerfileAST{
        Path: path,
        Root: result.AST,
    }
    ast.extractStructure()  // Build Commands and Stages slices
    return ast, nil
}
```

L'AST fornisce helper per le interrogazioni comuni:

```go
ast.HasInstruction("USER")           // Does it set a user?
ast.GetInstructions("ENV")           // All ENV instructions
ast.GetLastInstruction("USER")       // Last USER instruction
ast.FinalStage()                     // For multi-stage builds
```

### compose.go

Esegue il parsing dei file docker-compose.yml usando la libreria YAML:

```go
type ComposeFile struct {
    Path     string
    Services map[string]*Service
    Volumes  map[string]*Volume
    Networks map[string]*Network
}

type Service struct {
    Name        string
    Image       string
    Privileged  bool
    CapAdd      []string
    CapDrop     []string
    Volumes     []VolumeMount
    Ports       []PortMapping
    NetworkMode string
    // ...
}
```

Il parsing gestisce sia la sintassi lunga sia quella breve per volumi e porte:

```go
# Short syntax
volumes:
  - ./host:/container

# Long syntax
volumes:
  - type: bind
    source: ./host
    target: /container
```

## internal/docker

Avvolge il Docker SDK aggiungendo la gestione dei timeout:

```go
type Client struct {
    api *client.Client
}

func (c *Client) InspectContainer(ctx context.Context, containerID string) (types.ContainerJSON, error) {
    // Create derived context with timeout
    inspectCtx, cancel := context.WithTimeout(ctx, config.InspectTimeout)
    defer cancel()

    return c.api.ContainerInspect(inspectCtx, containerID)
}
```

Usa un singleton pattern in modo che l'intero programma condivida un'unica connessione:

```go
var (
    instance *Client
    once     sync.Once
)

func NewClient() (*Client, error) {
    once.Do(func() {
        cli, _ := client.NewClientWithOpts(
            client.FromEnv,
            client.WithAPIVersionNegotiation(),
        )
        instance = &Client{api: cli}
    })
    return instance, nil
}
```

## internal/proc

Legge le informazioni sui processi dal filesystem `/proc`. Funziona direttamente sul kernel dell'host, aggirando le astrazioni di Docker.

```go
func GetProcessInfo(pid int) (*ProcessInfo, error) {
    procPath := fmt.Sprintf("/proc/%d", pid)

    info := &ProcessInfo{PID: pid}
    info.readStatus(procPath)   // /proc/PID/status
    info.readCmdline(procPath)  // /proc/PID/cmdline
    info.readCgroups(procPath)  // /proc/PID/cgroup
    info.readNamespaces(procPath) // /proc/PID/ns/*
    return info, nil
}
```

Può rilevare se un processo si trova in un container verificando i percorsi dei cgroup:

```go
func (p *ProcessInfo) IsInContainer() bool {
    for _, cg := range p.Cgroups {
        if strings.Contains(cg.Path, "docker") ||
           strings.Contains(cg.Path, "containerd") {
            return true
        }
    }
    return false
}
```

Questo package usa la degradazione graduale. Se un file non può essere letto (permessi, sistema non Linux, ecc.), continua con ciò che riesce a ottenere invece di fallire.

## internal/report

Ogni formato di output implementa l'interfaccia Reporter:

```go
type Reporter interface {
    Report(findings finding.Collection) error
}
```

### terminal.go

Output colorato per l'uso interattivo:

```go
func (r *TerminalReporter) Report(findings finding.Collection) error {
    for _, f := range findings {
        color := f.Severity.Color()  // ANSI escape code
        fmt.Printf("%s[%s]%s %s\n", color, f.Severity, reset, f.Title)
    }
    return nil
}
```

### sarif.go

SARIF (Static Analysis Results Interchange Format) per la scheda Security di GitHub:

```go
type SARIFReport struct {
    Schema  string `json:"$schema"`
    Version string `json:"version"`
    Runs    []Run  `json:"runs"`
}
```

GitHub rileva automaticamente i file SARIF e mostra i rilevamenti nella scheda Security.

### junit.go

JUnit XML per l'integrazione CI/CD:

```go
type JUnitTestSuites struct {
    XMLName  xml.Name         `xml:"testsuites"`
    Tests    int              `xml:"tests,attr"`
    Failures int              `xml:"failures,attr"`
    Suites   []JUnitTestSuite `xml:"testsuite"`
}
```

Jenkins, GitLab CI e altri comprendono il formato JUnit per il reporting dei test.

## Aggiungere una nuova verifica

Supponiamo che tu voglia aggiungere una verifica per i container che usano il tag `latest`.

1. Trova l'analyzer pertinente (`container.go` per i container in esecuzione)

2. Aggiungi un metodo di verifica:

```go
func (a *ContainerAnalyzer) checkImageTag(target finding.Target, info types.ContainerJSON) finding.Collection {
    if strings.HasSuffix(info.Config.Image, ":latest") || !strings.Contains(info.Config.Image, ":") {
        control, _ := benchmark.Get("5.27")
        f := finding.New("CIS-5.27", control.Title, finding.SeverityLow, target).
            WithDescription(control.Description)
        return finding.Collection{f}
    }
    return nil
}
```

3. Chiamalo da analyzeContainer:

```go
findings = append(findings, a.checkImageTag(target, info)...)
```

Il controllo CIS dovrebbe già esistere in `benchmark/controls.go`. Se aggiungi una verifica personalizzata, registra prima un nuovo controllo.

## Aggiungere un nuovo pattern di segreti

In `rules/secrets.go`, aggiungi alla slice SecretPatterns:

```go
{
    Type:        SecretTypeAPIKey,
    Pattern:     regexp.MustCompile(`myservice_[A-Za-z0-9]{32}`),
    Description: "MyService API Key",
},
```

L'analyzer dei Dockerfile lo rileverà automaticamente.

## Aggiungere un nuovo formato di output

1. Crea un nuovo file in `internal/report/` (ad es. `csv.go`)

2. Implementa l'interfaccia Reporter:

```go
type CSVReporter struct {
    output io.Writer
}

func (r *CSVReporter) Report(findings finding.Collection) error {
    w := csv.NewWriter(r.output)
    w.Write([]string{"ID", "Severity", "Title", "Target"})
    for _, f := range findings {
        w.Write([]string{f.ID, f.Severity.String(), f.Title, f.Target.String()})
    }
    return w.Flush()
}
```

3. Aggiorna `NewReporter` in `reporter.go` per gestire il nuovo formato:

```go
case "csv":
    return &CSVReporter{output: out}, nil
```

4. Aggiungi il formato alla validazione dei flag CLI in `cmd/docksec/scan.go`

## Strategia di test

La codebase non ha ancora test estesi. Se li aggiungi:

**Unit test** per i package delle regole (capabilities, paths, secrets):
```go
func TestIsDangerousCapability(t *testing.T) {
    tests := []struct{cap string; want bool}{
        {"SYS_ADMIN", true},
        {"NET_BIND_SERVICE", false},
    }
    for _, tt := range tests {
        got := rules.IsDangerousCapability(tt.cap)
        if got != tt.want {
            t.Errorf("IsDangerousCapability(%q) = %v, want %v", tt.cap, got, tt.want)
        }
    }
}
```

**Test di integrazione** per gli analyzer usando mock del Docker SDK o testcontainers.

**Test E2E** eseguendo il binario su configurazioni note come errate.
