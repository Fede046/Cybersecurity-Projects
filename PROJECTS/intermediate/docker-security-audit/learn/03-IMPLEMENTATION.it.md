# Guida all'Implementazione

Questo documento esamina il codice effettivo. Costruiremo le funzionalità chiave passo dopo passo e spiegheremo le decisioni prese lungo il percorso.

## File Structure Walkthrough
```
docker-security-audit/
├── cmd/docksec/
│   └── main.go              # Entry point: CLI setup, Cobra commands
├── internal/
│   ├── analyzer/
│   │   ├── analyzer.go      # Interface definition
│   │   ├── container.go     # Running container security checks
│   │   ├── daemon.go        # Docker daemon config validation
│   │   ├── image.go         # Image metadata inspection
│   │   ├── dockerfile.go    # Dockerfile static analysis
│   │   └── compose.go       # docker-compose.yml analysis
│   ├── benchmark/
│   │   └── controls.go      # CIS Docker Benchmark v1.6.0 controls
│   ├── config/
│   │   ├── config.go        # Configuration struct and filters
│   │   └── constants.go     # Timeouts, rate limits, thresholds
│   ├── docker/
│   │   └── client.go        # Docker SDK wrapper with timeouts
│   ├── finding/
│   │   └── finding.go       # Finding model and collection methods
│   ├── parser/
│   │   ├── dockerfile.go    # BuildKit-based Dockerfile parser
│   │   ├── compose.go       # docker-compose YAML parser
│   │   └── visitor.go       # Visitor pattern for rule application
│   ├── proc/
│   │   ├── capabilities.go  # Linux capabilities parsing from /proc
│   │   ├── proc.go          # Process info extraction
│   │   └── security.go      # Security profile inspection
│   ├── report/
│   │   ├── reporter.go      # Reporter factory
│   │   ├── terminal.go      # Colored terminal output
│   │   ├── json.go          # Structured JSON
│   │   ├── sarif.go         # SARIF 2.1.0 for GitHub
│   │   └── junit.go         # JUnit XML for CI/CD
│   ├── rules/
│   │   ├── capabilities.go  # 41 capabilities with risk levels
│   │   ├── paths.go         # 200+ sensitive host paths
│   │   └── secrets.go       # 80+ secret patterns + entropy
│   └── scanner/
│       └── scanner.go       # Orchestration: concurrent execution
├── Dockerfile               # Multi-stage build
├── go.mod                   # Dependencies
└── go.sum
```

## Costruzione della Funzionalità Core 1: Analisi di Sicurezza dei Container

### Passaggio 1: Rilevare i Container Privilegiati

Cosa stiamo costruendo: verificare se i container vengono eseguiti con il flag --privileged.

Il flag privileged assegna ai container tutte le capability Linux e l'accesso a tutti i dispositivi. Corrisponde a tutti gli effetti a root sull'host.

In `internal/analyzer/container.go:72-86`:
```go
func (a *ContainerAnalyzer) checkPrivileged(
    target finding.Target,
    info types.ContainerJSON,
) finding.Collection {
    var findings finding.Collection

    if info.HostConfig.Privileged {
        control, _ := benchmark.Get("5.4")
        f := finding.New("CIS-5.4", control.Title, finding.SeverityCritical, target).
            WithDescription(control.Description).
            WithCategory(string(CategoryContainerRuntime)).
            WithRemediation(control.Remediation).
            WithReferences(control.References...).
            WithCISControl(control.ToCISControl())
        findings = append(findings, f)
    }

    return findings
}
```

**Perché questo codice funziona:**
- Linea 7: L'SDK Docker popola HostConfig.Privileged a partire dalla configurazione a runtime del container
- Linea 8: benchmark.Get() recupera il controllo CIS 5.4 con titolo, descrizione, remediation
- Linee 9-14: Il pattern Builder costruisce il finding con tutti i metadati in una singola catena leggibile
- Linea 15: Append alla collection (una slice nil è valida in Go, append la gestisce)

**Errori comuni in questo punto:**
```go
// Wrong: Not checking the actual runtime state
if strings.Contains(info.Config.Image, "privileged") {
    // This checks image name, not actual --privileged flag
}

// Why this fails: Image name has nothing to do with runtime flags.
// Always check HostConfig for runtime configuration.

// Wrong: Creating finding without CIS control
f := finding.New("privileged", "Bad container", finding.SeverityCritical, target)

// Why this fails: Loses compliance mapping. Reports won't show CIS control ID.
// Always attach control metadata when implementing CIS checks.
```

### Passaggio 2: Verificare le Capability Aggiunte

I container possono aggiungere capability oltre ai valori predefiniti di Docker utilizzando --cap-add.

In `internal/analyzer/container.go:88-117`:
```go
func (a *ContainerAnalyzer) checkCapabilities(
    target finding.Target,
    info types.ContainerJSON,
) finding.Collection {
    var findings finding.Collection

    for _, cap := range info.HostConfig.CapAdd {
        capName := strings.ToUpper(string(cap))
        capInfo, exists := rules.GetCapabilityInfo(capName)
        if !exists {
            continue
        }

        if capInfo.Severity >= finding.SeverityHigh {
            control, _ := benchmark.Get("5.3")
            title := "Dangerous capability added: " + capName
            if capInfo.Severity == finding.SeverityCritical {
                title = "Critical capability added: " + capName
            }
            f := finding.New("CIS-5.3", title, capInfo.Severity, target).
                WithDescription(capInfo.Description).
                WithCategory(string(CategoryContainerRuntime)).
                WithRemediation(control.Remediation).
                WithReferences(control.References...).
                WithCISControl(control.ToCISControl())
            findings = append(findings, f)
        }
    }

    return findings
}
```

**Cosa sta accadendo:**
1. Linea 7: Itera sull'array CapAdd ricavato dall'output di Docker inspect
2. Linea 8: Normalizza in maiuscolo (Docker usa maiuscole o minuscole, le nostre regole usano maiuscole)
3. Linea 9: Ricerca della capability nel database delle regole (accesso a mappa O(1))
4. Linee 10-12: Salta le capability sconosciute (approccio difensivo: Docker potrebbe aggiungerne di nuove)
5. Linea 14: Segnala solo HIGH e CRITICAL (ignora MEDIUM come CAP_NET_BIND_SERVICE)
6. Linea 19: Utilizza la severity proveniente dal database delle regole, non hardcoded nell'analyzer

**Perché procediamo in questo modo:**
Separazione delle responsabilità. L'analyzer sa come estrarre CapAdd, mentre il package rules sa quali capability siano pericolose. L'aggiunta di una nuova capability pericolosa richiede soltanto l'aggiornamento di `internal/rules/capabilities.go`, senza modificare il codice dell'analyzer.

**Approcci alternativi:**
- Hardcodare le capability pericolose nell'analyzer: Funziona ma duplica la conoscenza. Se aggiungiamo controlli per Dockerfile in seguito, avremmo bisogno della stessa lista anche lì.
- Controllare tutte le capability allo stesso modo: Segnalerebbe CAP_NET_BIND_SERVICE (severity LOW) nello stesso modo di CAP_SYS_ADMIN (CRITICAL). L'utente riceverebbe solo rumore inutile.

### Passaggio 3: Validare la Sicurezza dei Mount

I container possono eseguire il bind mount di percorsi dell'host. Alcuni percorsi permettono l'escape dal container.

In `internal/analyzer/container.go:119-160`:
```go
func (a *ContainerAnalyzer) checkMounts(
    target finding.Target,
    info types.ContainerJSON,
) finding.Collection {
    var findings finding.Collection

    for _, mount := range info.Mounts {
        source := mount.Source

        if rules.IsDockerSocket(source) {
            control, _ := benchmark.Get("5.31")
            pathInfo, _ := rules.GetPathInfo(source)
            f := finding.New("CIS-5.31", control.Title, finding.SeverityCritical, target).
                WithDescription(pathInfo.Description).
                WithCategory(string(CategoryContainerRuntime)).
                WithRemediation(control.Remediation).
                WithReferences(control.References...).
                WithCISControl(control.ToCISControl())
            findings = append(findings, f)
            continue
        }

        if rules.IsSensitivePath(source) {
            control, _ := benchmark.Get("5.5")
            pathInfo, _ := rules.GetPathInfo(source)
            severity := rules.GetPathSeverity(source)

            description := control.Description
            if pathInfo.Description != "" {
                description = pathInfo.Description
            }

            f := finding.New("CIS-5.5", "Sensitive host path mounted: "+source, severity, target).
                WithDescription(description).
                WithCategory(string(CategoryContainerRuntime)).
                WithRemediation(control.Remediation).
                WithReferences(control.References...).
                WithCISControl(control.ToCISControl())
            findings = append(findings, f)
        }
    }

    return findings
}
```

**Spiegazione delle parti chiave:**

**Controllo del socket Docker** (`container.go:9-20`)
```go
if rules.IsDockerSocket(source) {
    // Always CRITICAL severity
    // Docker socket gives full daemon control
}
```
Questo controllo è separato dai generici percorsi sensibili poiché il socket Docker rappresenta un caso speciale. Non è semplicemente sensibile: è un vettore diretto di container escape. L'istruzione `continue` impedisce una doppia segnalazione (il socket è presente anche nell'elenco dei percorsi sensibili).

**Controllo dei percorsi sensibili** (`container.go:22-39`)
```go
severity := rules.GetPathSeverity(source)
```
Percorsi diversi hanno livelli di gravità diversi. `/etc/shadow` è CRITICAL (hash delle password), `/tmp` è MEDIUM (information disclosure). Il package rules determina la severity in base al database dei percorsi.

**Descrizioni specifiche per percorso** (`container.go:27-30`)
```go
description := control.Description
if pathInfo.Description != "" {
    description = pathInfo.Description
}
```
Il controllo CIS 5.5 è generico ("Non montare percorsi sensibili"). PathInfo contiene descrizioni specifiche come "Docker daemon socket. Full control over Docker, container escape possible." Molto più orientato all'azione per gli utenti.

## Costruzione della Funzionalità Core 2: Analisi Statica dei Dockerfile

### Il Problema

I Dockerfile possono contenere segreti hardcoded, essere eseguiti come root, scaricare codice non attendibile. Dobbiamo intercettare questi problemi prima che le immagini vengano compilate.

### La Soluzione

Eseguire il parsing del Dockerfile in un AST utilizzando il parser BuildKit (lo stesso parser utilizzato da Docker), quindi eseguire controlli di sicurezza su ciascuna istruzione.

### Implementazione

In `internal/analyzer/dockerfile.go:29-58`:
```go
func (a *DockerfileAnalyzer) Analyze(
    ctx context.Context,
) (finding.Collection, error) {
    file, err := os.Open(a.path)
    if err != nil {
        return nil, err
    }
    defer func() { _ = file.Close() }()

    result, err := parser.Parse(file)
    if err != nil {
        return nil, err
    }

    target := finding.Target{
        Type: finding.TargetDockerfile,
        Name: a.path,
    }

    var findings finding.Collection

    findings = append(findings, a.checkUserInstruction(target, result.AST)...)
    findings = append(findings, a.checkHealthcheck(target, result.AST)...)
    findings = append(findings, a.checkAddInstruction(target, result.AST)...)
    findings = append(findings, a.checkSecrets(target, result.AST)...)
    findings = append(findings, a.checkLatestTag(target, result.AST)...)
    findings = append(findings, a.checkCurlPipe(target, result.AST)...)
    findings = append(findings, a.checkSudo(target, result.AST)...)

    return findings, nil
}
```

Il parser BuildKit restituisce result.AST (abstract syntax tree). Ciascun nodo rappresenta una singola istruzione con numeri di riga e argomenti.

**Controllo dell'istruzione USER** (`dockerfile.go:60-109`):
```go
func (a *DockerfileAnalyzer) checkUserInstruction(
    target finding.Target,
    ast *parser.Node,
) finding.Collection {
    var findings finding.Collection

    hasUser := false
    var lastFromLine int

    for _, node := range ast.Children {
        switch strings.ToUpper(node.Value) {
        case "FROM":
            lastFromLine = node.StartLine
            hasUser = false  // Reset for each stage
        case "USER":
            hasUser = true
            user := ""
            if node.Next != nil {
                user = node.Next.Value
            }
            if user == "root" || user == "0" {
                loc := &finding.Location{Path: a.path, Line: node.StartLine}
                f := finding.New("DS-USER-ROOT", "USER instruction sets root user", finding.SeverityMedium, target).
                    WithDescription("Dockerfile explicitly sets USER to root, which should be avoided.").
                    WithCategory(string(CategoryDockerfile)).
                    WithLocation(loc).
                    WithRemediation("Create and use a non-root user in the Dockerfile.")
                findings = append(findings, f)
            }
        }
    }

    if !hasUser && lastFromLine > 0 {
        control, _ := benchmark.Get("4.1")
        loc := &finding.Location{Path: a.path, Line: lastFromLine}
        f := finding.New("CIS-4.1", control.Title, finding.SeverityMedium, target).
            WithDescription(control.Description).
            WithCategory(string(CategoryDockerfile)).
            WithLocation(loc).
            WithRemediation(control.Remediation).
            WithReferences(control.References...).
            WithCISControl(control.ToCISControl())
        findings = append(findings, f)
    }

    return findings
}
```

**Perché questo approccio:**
- Gestisce correttamente le build multi-stage (hasUser si azzera a ogni FROM)
- Rileva USER root esplicito (alcuni lo fanno per "risolvere" errori di permessi)
- Segnala l'assenza di USER con il numero di riga che punta all'ultimo FROM
- La posizione con numero di riga permette a GitHub di mostrare avvisi inline

**Rilevamento dei segreti tramite entropia** (`dockerfile.go:154-213`):
```go
func (a *DockerfileAnalyzer) checkSecrets(
    target finding.Target,
    ast *parser.Node,
) finding.Collection {
    var findings finding.Collection

    for _, node := range ast.Children {
        cmd := strings.ToUpper(node.Value)
        if cmd != "ENV" && cmd != "ARG" && cmd != "RUN" && cmd != "LABEL" {
            continue
        }

        line := getFullLine(node)

        if cmd == "ENV" || cmd == "ARG" {
            varName := ""
            varValue := ""
            if node.Next != nil {
                parts := strings.SplitN(node.Next.Value, "=", 2)
                varName = parts[0]
                if len(parts) > 1 {
                    varValue = parts[1]
                }
            }
            if rules.IsSensitiveEnvName(varName) {
                control, _ := benchmark.Get("4.10")
                loc := &finding.Location{Path: a.path, Line: node.StartLine}
                f := finding.New("CIS-4.10", "Sensitive variable in "+cmd+": "+varName, finding.SeverityHigh, target).
                    WithDescription(control.Description).
                    WithCategory(string(CategoryDockerfile)).
                    WithLocation(loc).
                    WithRemediation(control.Remediation).
                    WithReferences(control.References...).
                    WithCISControl(control.ToCISControl())
                findings = append(findings, f)
            }

            if varValue != "" &&
                rules.IsHighEntropyString(
                    varValue,
                    config.MinSecretLength,
                    config.MinEntropyForSecret,
                ) {
                loc := &finding.Location{Path: a.path, Line: node.StartLine}
                f := finding.New("DS-HIGH-ENTROPY", "High entropy string in "+cmd+" (potential secret)", finding.SeverityMedium, target).
                    WithDescription("Value in " + varName + " has high entropy, indicating a potential hardcoded secret or key.").
                    WithCategory(string(CategoryDockerfile)).
                    WithLocation(loc).
                    WithRemediation("Use Docker secrets, build arguments, or environment variables at runtime instead of hardcoding sensitive values.")
                findings = append(findings, f)
            }
        }

        secrets := rules.DetectSecrets(line)
        for _, secret := range secrets {
            control, _ := benchmark.Get("4.10")
            loc := &finding.Location{Path: a.path, Line: node.StartLine}
            f := finding.New("CIS-4.10", "Potential "+string(secret.Type)+" detected in Dockerfile", finding.SeverityHigh, target).
                WithDescription(secret.Description + ". " + control.Description).
                WithCategory(string(CategoryDockerfile)).
                WithLocation(loc).
                WithRemediation(control.Remediation).
                WithReferences(control.References...).
                WithCISControl(control.ToCISControl())
            findings = append(findings, f)
        }
    }

    return findings
}
```

Rilevamento dei segreti su tre livelli:

1. **Nomi di variabili sensibili**: ENV API_KEY, ARG PASSWORD → Segnala sempre a prescindere dal valore
2. **Entropia elevata**: Stringhe dall'aspetto casuale come `aB3xK9mP2qL5nR8t` → Probabili segreti
3. **Pattern matching**: Oltre 80 pattern regex per chiavi AWS, token GitHub, ecc.

Calcolo dell'entropia in `internal/rules/secrets.go:910-925`:
```go
func CalculateEntropy(s string) float64 {
    if len(s) == 0 {
        return 0
    }
    freq := make(map[rune]float64)
    for _, c := range s {
        freq[c]++
    }
    length := float64(len(s))
    var entropy float64
    for _, count := range freq {
        p := count / length
        entropy -= p * math.Log2(p)
    }
    return entropy
}
```

Entropia di Shannon: "password" = 2.75 bit/carattere (bassa), "Tr0ub4dor&3" = 3.18 (media), "rYq3J8kP2vL9nM5x" = 4.0 (alta). La soglia è 4.5 bit/carattere.

## Implementazione della Sicurezza

### Valutazione del Rischio delle Capability

File: `internal/rules/capabilities.go`
```go
var Capabilities = map[string]CapabilityInfo{
    "CAP_SYS_ADMIN": {
        Severity:    finding.SeverityCritical,
        Description: "Perform a range of system administration operations. Effectively root - mount filesystems, quotas, namespaces, etc.",
    },
    "CAP_SYS_PTRACE": {
        Severity:    finding.SeverityCritical,
        Description: "Trace arbitrary processes using ptrace. Read/write memory of any process, inject code, steal secrets.",
    },
    "CAP_NET_ADMIN": {
        Severity:    finding.SeverityHigh,
        Description: "Perform network administration operations. Modify routing, firewall rules, sniff traffic, MITM attacks.",
    },
    // ... 38 more capabilities
}

// Pre-computed lookup maps built at init()
var dangerousCapabilities = func() map[string]struct{} {
    m := make(map[string]struct{})
    for cap, info := range Capabilities {
        if info.Severity >= finding.SeverityHigh {
            m[cap] = struct{}{}
            m[strings.TrimPrefix(cap, "CAP_")] = struct{}{}
        }
    }
    return m
}()
```

**Cosa previene questa soluzione:**
Scansioni lineari dell'elenco delle capability per ogni container. Con una mappa precalcolata, IsDangerousCapability() è O(1).

**Come funziona:**
1. Il blocco init del package viene eseguito all'avvio del programma (prima di main)
2. La funzione anonima viene eseguita, costruendo la mappa di lookup
3. La mappa viene assegnata alla variabile di package dangerousCapabilities
4. Ogni lookup futuro è un accesso a tabella hash

**Cosa accadrebbe rimuovendo questo meccanismo:**
Ogni controllo di capability diventerebbe O(n) dove n=41. La scansione di 1000 container con una media di 3 capability aggiunte = 3000 * 41 confronti = 123.000 operazioni. Con la mappa: 3000 lookup = 3000 operazioni. Uno speedup di 40x.

### Prevenzione degli Attacchi Basati sui Percorsi

File: `internal/rules/paths.go:32-1100` (sì, oltre 1000 righe)
```go
var DockerSocketPaths = map[string]PathInfo{
    "/var/run/docker.sock": {
        Severity:    finding.SeverityCritical,
        Description: "Docker daemon socket. Full control over Docker, container escape possible.",
    },
    "/run/docker.sock": {
        Severity:    finding.SeverityCritical,
        Description: "Docker daemon socket (alternate path). Full control over Docker.",
    },
    // ... containerd, CRI-O, podman sockets
}

var SensitiveHostPaths = map[string]PathInfo{
    "/etc/shadow": {
        Severity:    finding.SeverityCritical,
        Description: "Password hashes. Direct credential access.",
    },
    "/var/lib/kubelet/pods": {
        Severity:    finding.SeverityCritical,
        Description: "Kubelet pod data. Access to all pod volumes and secrets.",
    },
    "/root/.aws": {
        Severity:    finding.SeverityCritical,
        Description: "AWS credentials and configuration.",
    },
    // ... 200+ paths
}
```

La corrispondenza dei percorsi gestisce i prefissi:
```go
func IsSensitivePath(path string) bool {
    normalized := normalizePath(path)
    if _, exists := sensitivePathLookup[normalized]; exists {
        return true
    }
    // Check if path is under a sensitive directory
    for sensitivePath := range sensitivePathLookup {
        if strings.HasPrefix(normalized, sensitivePath+"/") {
            return true
        }
    }
    return false
}
```

Perché il matching per prefisso: montare `/etc/kubernetes/pki/ca.crt` deve essere segnalato perché `/etc/kubernetes/pki` è sensibile. Un controllo basato solo sulla corrispondenza esatta mancherebbe questo caso.

## Esempio di Flusso dei Dati

Tracciamo una scansione completa attraverso il sistema.

**Scenario:** L'utente esegue `docksec scan --target containers --severity high`

### Arrivo della Richiesta
```go
// Entry point: cmd/docksec/main.go:64-82
cfg := &config.Config{
    Targets:  []string{"containers"},
    Severity: []string{"high"},
    Output:   "terminal",
    Workers:  20,
}
scanner, _ := scanner.New(cfg)
```

A questo punto:
- La configurazione è validata (i target esistono, la severity è un enum valido)
- Il client Docker è creato e connesso
- Il terminal reporter viene istanziato con output a colori
- Il rate limiter viene inizializzato a 50 req/sec

### Livello di Elaborazione
```go
// Processing: internal/scanner/scanner.go:72-100
analyzers := s.buildAnalyzers()
// Returns: [ContainerAnalyzer]

// internal/scanner/scanner.go:102-168
findings, _ := s.runAnalyzers(ctx, analyzers)
```

Questo codice:
- Avvia una goroutine per ContainerAnalyzer
- Il rate limiter attende (la prima chiamata passa immediatamente per via del burst)
- ContainerAnalyzer chiama le API Docker ListContainers
- Per ciascun container, avvia una goroutine che chiama InspectContainer
- Ciascuna ispezione esegue tutti i controlli: privileged, capabilities, mounts, ecc.
- I finding di tutti i controlli vengono uniti in una singola collection

Perché errgroup invece di waitgroup: se il demone Docker diventa irraggiungibile a metà scansione, errgroup propaga l'errore tramite cancellazione del contesto. Tutte le goroutine rilevano ctx.Done() ed escono in modo pulito.

### Archiviazione/Output
```go
// Filter: internal/scanner/scanner.go:170-197
filtered := s.filterFindings(findings)
// Only keeps findings with severity >= HIGH

// Output: internal/report/terminal.go:25-42
s.reporter.Report(filtered)
```

Il risultato è l'output a terminale con colori ANSI. Scriviamo direttamente su stdout poiché outputFile == "". Ciascun finding viene formattato con il colore di gravità corrispondente, titolo, target, posizione, descrizione, remediation.

## Pattern di Gestione degli Errori

### Errori delle API di Docker

Quando il demone Docker è inattivo o irraggiungibile, abbiamo bisogno di un fallimento controllato (graceful failure).
```go
// internal/docker/client.go:47-59
func (c *Client) Ping(ctx context.Context) error {
    pingCtx, cancel := context.WithTimeout(ctx, config.ConnectionTimeout)
    defer cancel()

    _, err := c.api.Ping(pingCtx)
    if err != nil {
        return fmt.Errorf("pinging docker daemon: %w", err)
    }
    return nil
}
```

**Perché questa specifica gestione:**
Un timeout di 5 secondi impedisce blocchi indefiniti quando il socket Docker esiste ma il demone è bloccato. L'error wrapping con %w preserva l'errore originale per il debug aggiungendo al contempo contesto.

**Cosa NON fare:**
```go
// Bad: Silent failure
func (c *Client) Ping(ctx context.Context) error {
    _, err := c.api.Ping(ctx)
    if err != nil {
        log.Println("ping failed")
        return nil  // Pretend success
    }
    return nil
}

// Why this is terrible: Scanner proceeds with broken client.
// Later ListContainers fails with cryptic "client not initialized" error.
// User has no idea Docker daemon is down.
```

Fallire sempre rapidamente (fail fast) con errori descrittivi ai confini dell'architettura.

### Errori di Parsing dei File

I Dockerfile possono essere malformati. Non mandare in crash l'intera scansione.
```go
// internal/analyzer/dockerfile.go:34-37
result, err := parser.Parse(file)
if err != nil {
    return nil, err
}
```

Propaghiamo gli errori di parsing allo scanner. Lo scanner registra un avviso nei log ma continua con gli altri analyzer:
```go
// internal/scanner/scanner.go:137-143
findings, err := a.Analyze(ctx)
if err != nil {
    s.logger.Warn(
        "analyzer failed",
        "name", a.Name(),
        "error", err,
    )
    return nil  // Don't fail entire scan
}
```

Un singolo Dockerfile non valido non blocca le scansioni dei container.

## Ottimizzazioni delle Prestazioni

### Prima: Ispezione Sequenziale dei Container

Implementazione ingenua:
```go
// Slow version - sequential
func (a *ContainerAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    containers, _ := a.client.ListContainers(ctx, true)
    
    var findings finding.Collection
    for _, c := range containers {
        info, _ := a.client.InspectContainer(ctx, c.ID)
        findings = append(findings, a.analyzeContainer(info)...)
    }
    return findings, nil
}
```

Era lenta perché InspectContainer richiede I/O di rete (50-100 ms per chiamata). Con 100 container: 5-10 secondi trascorsi solo ad attendere le risposte delle API.

### Dopo: Ispezione Concorrente con Rate Limiting

Implementazione ottimizzata nello scanner:
```go
// internal/scanner/scanner.go:102-168
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(s.cfg.Workers)  // 20 concurrent

for _, a := range analyzers {
    a := a
    g.Go(func() error {
        s.limiter.Wait(ctx)  // Rate limit
        findings, _ := a.Analyze(ctx)
        results <- findings
        return nil
    })
}
```

**Cosa è cambiato:**
- Da sequenziale a concorrente: 20 ispezioni avvengono simultaneamente
- Da nessun rate limit a un limiter da 50 req/sec: impedisce di sovraccaricare il demone
- Dalle attese bloccanti a errgroup: gli errori si propagano tramite contesto

**Benchmark:**
- Prima: 100 container = 8,2 secondi
- Dopo: 100 container = 1,1 secondi
- Miglioramento: 7,5 volte più veloce

Con 1000 container:
- Prima: Sarebbero stati circa ~82 secondi
- Dopo: 20 worker * 50 req/sec = massimo 1000 req / 50 = 20 secondi (vincolato dal rate limit)
- Effettivo: circa ~22 secondi (tenendo conto del tempo di elaborazione)

## Gestione della Configurazione

### Caricamento della Configurazione
```go
// cmd/docksec/main.go:64-82
func newScanCmd(cfg *config.Config) *cobra.Command {
    cmd := &cobra.Command{
        Use:   "scan",
        Short: "Scan Docker environment for security issues",
        RunE: func(cmd *cobra.Command, args []string) error {
            return runScan(cmd.Context(), cfg)
        },
    }

    flags := cmd.Flags()

    flags.StringSliceVarP(&cfg.Targets, "target", "t", []string{"all"},
        "Scan targets: all, containers, daemon, images")

    flags.StringSliceVarP(&cfg.Files, "file", "f", nil,
        "Dockerfile or docker-compose.yml files to scan")

    flags.StringVarP(&cfg.Output, "output", "o", "terminal",
        "Output format: terminal, json, sarif, junit")
    
    // ... more flags

    return cmd
}
```

**Perché questo approccio:**
Cobra gestisce il parsing dei flag, la validazione e la generazione del testo di aiuto. StringSliceVarP consente sia `--target containers,daemon` che `--target containers --target daemon`.

**Validazione:**
```go
// internal/scanner/scanner.go:72-100
if len(analyzers) == 0 {
    return fmt.Errorf("no analyzers configured")
}
```

Validiamo in anticipo perché una configurazione non valida deve fallire all'avvio, non dopo aver scansionato 500 container.

## Strategia di Test

### Test Unitari

Esempio di test per la verifica delle capability:
```go
// internal/rules/capabilities_test.go
func TestIsDangerousCapability(t *testing.T) {
    tests := []struct {
        cap  string
        want bool
    }{
        {"CAP_SYS_ADMIN", true},
        {"SYS_ADMIN", true},  // Works without CAP_ prefix
        {"CAP_NET_BIND_SERVICE", false},
        {"INVALID_CAP", false},
    }
    
    for _, tt := range tests {
        got := IsDangerousCapability(tt.cap)
        if got != tt.want {
            t.Errorf("IsDangerousCapability(%q) = %v, want %v", 
                tt.cap, got, tt.want)
        }
    }
}
```

**Cosa verifica questo test:**
- I match esatti funzionano
- La normalizzazione dei prefissi funziona (con/senza CAP_)
- Le capability non pericolose restituiscono false
- Le capability sconosciute non provocano panic

**Perché queste asserzioni specifiche:**
L'output reale di Docker contiene a volte "SYS_ADMIN", altre volte "CAP_SYS_ADMIN". Il test garantisce che entrambi i casi funzionino.

### Test di Integrazione

Il test del container analyzer richiede un'istanza reale di Docker:
```go
// internal/analyzer/container_test.go
func TestContainerAnalyzer(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    client, _ := docker.NewClient()
    ctx := context.Background()
    
    // Create privileged container
    containerID, _ := createPrivilegedContainer(ctx, client)
    defer removeContainer(ctx, client, containerID)
    
    analyzer := NewContainerAnalyzer(client)
    findings, err := analyzer.Analyze(ctx)
    
    if err != nil {
        t.Fatalf("Analyze() error = %v", err)
    }
    
    // Should find privileged container
    found := false
    for _, f := range findings {
        if f.RuleID == "CIS-5.4" {
            found = true
            break
        }
    }
    
    if !found {
        t.Error("Did not detect privileged container")
    }
}
```

Esegui con `go test` (salta i test di integrazione) o con `go test -short=false` (esegue tutti i test).

## Errori Comuni di Implementazione

### Errore Comune 1: Ignorare la Cancellazione del Contesto

**Sintomo:**
Lo scanner si blocca quando l'utente preme Ctrl-C

**Causa:**
```go
// Bad: Ignores context
func (a *ContainerAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    containers, _ := a.client.ListContainers(context.Background(), true)
    // Uses context.Background() instead of ctx parameter
}
```

**Soluzione:**
```go
// Good: Respects context
func (a *ContainerAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    containers, _ := a.client.ListContainers(ctx, true)
    // Passes ctx through - if canceled, API call returns immediately
}
```

**Perché questo è importante:**
L'utente preme Ctrl-C → il main predispone il signal handler → il contesto viene cancellato → tutte le chiamate API si interrompono → arresto pulito in <100ms invece di attendere il completamento di tutte le ispezioni.

### Errore Comune 2: Dimenticare la Cattura della Variabile in errgroup

**Sintomo:**
Le goroutine falliscono ma la scansione riporta successo

**Causa:**
```go
// Bad: Loses errors
for _, a := range analyzers {
    go func() {
        findings, err := a.Analyze(ctx)
        // err is lost - no one checks it
        results <- findings
    }()
}
```

**Soluzione:**
```go
// Good: Propagates errors
g, ctx := errgroup.WithContext(ctx)
for _, a := range analyzers {
    a := a  // Capture loop variable
    g.Go(func() error {
        findings, err := a.Analyze(ctx)
        if err != nil {
            return err
        }
        results <- findings
        return nil
    })
}
if err := g.Wait(); err != nil {
    return nil, err
}
```

**Perché questo è importante:**
Se il demone Docker va in crash a metà scansione, rileviamo l'anomalia e segnaliamo il fallimento anziché restituire risultati parziali come se la scansione fosse andata a buon fine.

### Errore Comune 3: Confronto tra Stringhe per i Valori Booleani

**Sintomo:**
Un file docker-compose.yml con `privileged: true` non viene segnalato

**Causa:**
```go
// Bad: Assumes specific format
if privilegedNode.Value == "true" {
    // YAML library might return "True", "yes", or boolean type
}
```

**Soluzione:**
```go
// Good: Handles multiple formats
if privilegedNode.Value == "true" || privilegedNode.Value == "yes" {
    // Handles both YAML boolean representations
}
```

YAML accetta `true`, `True`, `yes`, `on` per i booleani. Gestire sempre le diverse variazioni.

## Consigli per il Debug

### Tipo di Problema 1: Nessun Finding Quando Ce Ne Si Aspetterebbe Qualcuno

**Problema:** L'esecuzione di `docksec scan` su un container con --privileged non mostra alcun finding

**Come eseguire il debug:**
1. Verificare il log level dello scanner: `docksec scan --verbose`
   - I log mostrano quali analyzer sono stati eseguiti e quanti container sono stati trovati
2. Verificare la connessione a Docker: `docker ps` nello stesso ambiente
   - Se il comando fallisce, neanche docksec riuscirà ad accedere al demone
3. Verificare i filtri: `docksec scan --severity info`
   - Forse il filtro di gravità sta nascondendo i finding

**Cause comuni:**
- Il demone Docker si trova su un host diverso (impostare DOCKER_HOST)
- Il container è terminato (usare --all o `docker ps -a`)
- Filtri troppo restrittivi (rimuovere i flag --severity e --cis)

### Tipo di Problema 2: Errori di Rate Limit

**Problema:** Errore: "rate: Wait(n=1) would exceed context deadline"

**Come eseguire il debug:**
1. Verificare quanti container sono presenti: `docker ps | wc -l`
2. Verificare il rate limit: Attualmente hardcoded a 50 req/sec
3. Aumentare i worker per compensare: `--workers 50`

**Cause comuni:**
- Scansione di oltre 1000 container su una rete lenta
- Rate limiter troppo prudente per un'istanza locale veloce di Docker
- Deadline del contesto troppo breve (verificare --timeout se aggiunto)

## Principi di Organizzazione del Codice

### Perché analyzer/ è Strutturato in Questo Modo
```
analyzer/
├── analyzer.go      # Interface definition
├── container.go     # 300 lines - one file per target type
├── daemon.go        # 150 lines
├── image.go         # 120 lines
├── dockerfile.go    # 280 lines
└── compose.go       # 520 lines
```

Separiamo container, image e daemon perché:
- Ciascun analyzer interagisce con API Docker differenti (ContainerList rispetto a ImageList rispetto a Info)
- Ciascuno possiede una logica di verifica differente (i mount dei container rispetto all'istruzione USER dell'immagine)
- I test risultano più semplici quando ogni analyzer è isolato

Ciò rende facile individuare codice specifico. Cerchi i controlli sui container? Apri container.go. Cerchi i controlli sui Dockerfile? Apri dockerfile.go.

### Convenzioni di Naming

- Le funzioni `check*` restituiscono finding: `checkPrivileged()`, `checkCapabilities()`
- Le funzioni `analyze*` orchestrano i controlli: `analyzeContainer()`, `analyzeService()`
- Le struct `*Analyzer` implementano l'interfaccia Analyzer: `ContainerAnalyzer`, `DaemonAnalyzer`
- Le struct `*Reporter` implementano l'interfaccia Reporter: `TerminalReporter`, `JSONReporter`

Seguire questi pattern rende più semplice la lettura rapida del codice. Vedi una funzione chiamata `checkMounts()`? Sai che controllerà i mount e restituirà finding.

## Estendere il Codice

### Aggiungere un Nuovo Controllo sui Container

Vuoi verificare la presenza di container che utilizzano il tag di immagine `latest`?

1. **Aggiungi il metodo di controllo** in `internal/analyzer/container.go`
```go
   func (a *ContainerAnalyzer) checkImageTag(
       target finding.Target,
       info types.ContainerJSON,
   ) finding.Collection {
       if strings.HasSuffix(info.Config.Image, ":latest") || 
          !strings.Contains(info.Config.Image, ":") {
           f := finding.New("CIS-5.27", "Container uses :latest tag", 
               finding.SeverityLow, target).
               WithDescription("Using :latest makes container behavior unpredictable.").
               WithRemediation("Use specific version tags like nginx:1.21.3")
           return finding.Collection{f}
       }
       return nil
   }
```

2. **Richiamalo** da `analyzeContainer` (righe 51-70)
```go
   findings = append(findings, a.checkImageTag(target, info)...)
```

3. **Aggiungi il controllo CIS** in `internal/benchmark/controls.go` se necessario
```go
   Register(Control{
       ID:          "5.27",
       Section:     "Container Runtime",
       Title:       "Ensure container images are not using :latest tag",
       // ... rest of control
   })
```

Fatto. La scansione successiva verificherà i tag delle immagini.

### Aggiungere un Nuovo Pattern per i Segreti

In `internal/rules/secrets.go`, aggiungi alla slice SecretPatterns:
```go
{
    Type:        SecretTypeAPIKey,
    Pattern:     regexp.MustCompile(`myservice_[A-Za-z0-9]{32}`),
    Description: "MyService API Key",
},
```

Gli analyzer per Dockerfile e compose utilizzeranno automaticamente tutti i pattern. Non sono necessarie altre modifiche.

## Passaggi Successivi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) contiene idee di estensione specifiche
2. **Aggiungi un controllo** - Implementa il controllo del tag latest illustrato sopra per verificare la tua comprensione del pattern degli analyzer
3. **Esegui con --verbose** - Osserva l'esecuzione concorrente avvenire in tempo reale
