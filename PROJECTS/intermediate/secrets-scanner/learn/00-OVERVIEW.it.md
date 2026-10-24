# Portia - Secrets Scanner per codebase

## Cos'è

Portia è un tool CLI in Go che analizza le directory di codice sorgente e la cronologia git alla ricerca di secret trapelati: chiavi API, password, token, chiavi private, connection string. Usa 150 regole di detection, l'analisi dell'entropia di Shannon, un filtro anti falsi positivi a 5 livelli e una verifica opzionale delle violazioni tramite Have I Been Pwned. I file vengono suddivisi in chunk da 50 righe ed elaborati attraverso un worker pool concorrente limitato, con i risultati riportati in formato terminale, JSON o SARIF.

Il nome viene da *Il mercante di Venezia* di Shakespeare, dove Portia è colei che smaschera ciò che è nascosto nel contratto. Questo tool fa lo stesso per la tua codebase.

## Perché è importante

I secret hardcoded nel codice sorgente sono uno degli errori di sicurezza più comuni e più dannosi nello sviluppo software. Non è una questione teorica. Ecco tre incidenti che mostrano esattamente perché è importante:

**Uber, 2016 - Chiavi AWS hardcoded su GitHub**
Due ingegneri di Uber hanno fatto il commit di chiavi di accesso AWS in un repository GitHub privato. Gli attaccanti che hanno ottenuto l'accesso a quel repo hanno usato le chiavi per raggiungere un bucket S3 contenente le informazioni personali di 57 milioni di passeggeri e autisti. Uber ha pagato 148 milioni di dollari in accordi transattivi. Le chiavi sono rimaste nel repo per mesi prima della violazione. Un tool come Portia, eseguito a ogni commit, avrebbe segnalato immediatamente i pattern `AKIA...` grazie alla regola `aws-access-key-id` definita in `internal/rules/builtin.go:22-31`. La regex `\b((?:AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16})\b` intercetta esattamente questo formato.

**CircleCI, gennaio 2023 - Secret nella configurazione CI**
Un attaccante ha compromesso il laptop di un ingegnere di CircleCI e ha usato il cookie di sessione per accedere ai sistemi interni. Ha poi estratto i secret dei clienti memorizzati come variabili d'ambiente nelle pipeline CI/CD. L'advisory (CCI-2023-001) ha esortato tutti i clienti a ruotare ogni secret memorizzato in CircleCI. Il problema di fondo: i secret erano configurati come valori in chiaro nelle configurazioni CI invece di essere recuperati da vault a runtime. Se quei file di configurazione fossero stati analizzati prima del deploy, il problema sarebbe stato individuato. Le regole generiche di Portia per le assegnazioni di `password`, `secret`, `token` e `api_key` puntano esattamente a questo pattern.

**Codecov, aprile 2021 - Attacco alla supply chain tramite token esfiltrati**
Gli attaccanti hanno modificato lo script bash uploader di Codecov per esfiltrare le variabili d'ambiente dagli ambienti CI. Ogni azienda che usava il bash uploader di Codecov ha fatto trapelare qualsiasi token e chiave fosse presente nel proprio ambiente CI, comprese le credenziali per GitHub, AWS e servizi interni. Lo script modificato è rimasto in esecuzione per due mesi prima di essere scoperto. Tra le aziende colpite: Twitch, HashiCorp, Confluent. Portia rileva token per GitHub (`ghp_`, `gho_`, `ghs_`), HashiCorp Vault (`hvs.`, `hvb.`, `hvr.`) e per molti altri provider, tutti definiti in `internal/rules/builtin.go`.

Il filo conduttore: secret presenti in codice o configurazioni dove strumenti automatici avrebbero potuto individuarli. È ciò che costruisce questo progetto.

## Cosa imparerai

Costruire questo progetto ti insegna sia concetti di sicurezza sia pattern di ingegneria reale in Go.

**Concetti di sicurezza:**
- **Il secret sprawl e perché avviene** - Perché gli sviluppatori continuano a fare l'hardcoding delle credenziali pur sapendo di non doverlo fare, e le soluzioni sistemiche (variabili d'ambiente, integrazioni con vault, hook pre-commit)
- **Entropia di Shannon per l'anomaly detection** - Usare la teoria dell'informazione per distinguere le chiavi API casuali dal normale testo inglese. La matematica è in `internal/rules/entropy.go`.
- **k-Anonymity** - Il protocollo di Troy Hunt per verificare se un secret compare nei database delle violazioni senza rivelare il secret stesso. Implementazione in `internal/hibp/client.go`.
- **Riduzione dei falsi positivi** - Perché una scansione regex ingenua produce migliaia di risultati inutili, e l'approccio di filtraggio a 5 livelli che rende l'output davvero utile. Vedi `internal/engine/filter.go`.
- **SARIF per l'integrazione CI** - Il formato standard di settore che GitHub, Azure DevOps e altre piattaforme CI consumano per mostrare i risultati di sicurezza direttamente nelle pull request.

**Competenze tecniche (specifiche di Go):**
- **Pipeline concorrenti con errgroup** - Worker pool limitati che usano `golang.org/x/sync/errgroup` con comunicazione basata su channel. La pipeline completa è in `internal/engine/pipeline.go`.
- **Framework CLI Cobra** - Costruire una CLI multi-comando con flag persistenti, supporto ai file di configurazione e sottocomandi. Comando root in `internal/cli/root.go`.
- **Libreria go-git** - Attraversamento programmatico della cronologia git, scorrimento dei commit, lettura dei blob e scansione dei file in staging senza invocare `git` da shell. Vedi `internal/source/git.go`.
- **Pattern circuit breaker** - Usare `gobreaker` per prevenire guasti a cascata quando le API esterne (HIBP) non sono disponibili. Configurato in `internal/hibp/client.go`.
- **Caching LRU** - Caching in memoria limitato con `hashicorp/golang-lru` per evitare chiamate API ridondanti. Vedi `internal/hibp/client.go`.
- **Configurazione TOML** - Risoluzione stratificata della configurazione con flag CLI > file di configurazione > valori predefiniti. Supporta sia `.portia.toml` sia `pyproject.toml` (tabella `[tool.portia]`). Vedi `internal/config/config.go` e `internal/cli/root.go`.

**Strumenti e tecniche:**
- **Realizzare regex per la sicurezza** - Scrivere pattern che corrispondono ai formati reali delle credenziali (prefissi delle chiavi AWS, formati dei token GitHub, pattern delle chiavi Stripe) evitando i falsi positivi
- **just task runner** - Usare `Justfile` come alternativa ai Makefile per i workflow di build, test e lint
- **golangci-lint** - Configurazione dell'analisi statica in `.golangci.yml` per imporre la qualità del codice

## Prerequisiti

**Conoscenze richieste:**
- **Basi di Go** - Devi capire goroutine, channel, interfacce e struct. La pipeline li usa tutti intensamente. Se `go func()` o `chan types.Chunk` ti sembrano poco familiari, svolgi prima il Go Tour.
- **Espressioni regolari** - Il motore di detection è costruito sulle regex. Dovresti conoscere i gruppi di cattura, i quantificatori, le classi di caratteri e i gruppi non catturanti. Le regole in `internal/rules/builtin.go` li usano tutti.
- **Fondamenti di git** - Dovresti capire commit, branch, staging e blob. Lo scanner git attraversa programmaticamente queste strutture.

**Strumenti necessari:**
- **Go 1.22+** - Usa `range` sugli interi (ad es. `for range p.workers` in `internal/engine/pipeline.go:53`), che richiede Go 1.22
- **just** - Task runner. Installalo con `cargo install just` o con il tuo package manager
- **git** - Per le funzionalità di scansione della cronologia git

**Utile ma non necessario:**
- **Teoria dell'informazione** - Capire l'entropia aiuta, ma il documento sui concetti la spiega da zero
- **Specifica SARIF** - Lo standard OASIS per i risultati dell'analisi statica. Puoi leggere l'output senza conoscere la specifica.

## Installazione

Tre modi per installare Portia:

**Opzione 1: script di installazione** (Go non necessario)
```bash
curl -fsSL https://raw.githubusercontent.com/CarterPerez-dev/portia/main/install.sh | bash
```

Scarica un binario precompilato per la tua piattaforma (Linux/macOS, amd64/arm64). Se non è disponibile alcun binario, ripiega sulla compilazione dai sorgenti con Go.

**Opzione 2: Go install**
```bash
go install github.com/CarterPerez-dev/portia/cmd/portia@latest
```

Richiede Go 1.24+. Il binario viene posizionato in `$GOPATH/bin` (o `$GOBIN`).

**Opzione 3: build dai sorgenti**
```bash
cd PROJECTS/intermediate/secrets-scanner
go build -o portia ./cmd/portia
```

Questo è il percorso che userai mentre lavori su questo progetto e apporti modifiche al codice.

## Avvio rapido

Metti in funzione il progetto:
```bash
cd PROJECTS/intermediate/secrets-scanner

./portia scan ./testdata/fixtures

./portia scan --format json ./testdata/fixtures

./portia scan --hibp ./testdata/fixtures

./portia git --depth 10 .

./portia git --staged .
```

Output atteso sulle fixture di test: output colorato nel terminale che mostra i secret rilevati con livelli di severità, ID delle regole, percorsi dei file, numeri di riga, punteggi di entropia e valori dei secret mascherati.

Per inizializzare un file di configurazione:
```bash
./portia init
```

Questo crea `.portia.toml` con le impostazioni predefinite. Modificalo per disabilitare regole, impostare esclusioni o abilitare di default il controllo HIBP.

Per i progetti Python puoi usare `pyproject.toml` al suo posto:
```bash
./portia pyproject
```

Questo crea un `pyproject.toml` con una sezione `[tool.portia]`. Portia legge automaticamente da `pyproject.toml` quando non trova alcun `.portia.toml`.

## Struttura del progetto

```
secrets-scanner/
├── cmd/portia/main.go           # Entry point, calls cli.Execute()
├── internal/
│   ├── cli/                     # Cobra commands
│   │   ├── root.go              # Root command, flag definitions, config init
│   │   ├── scan.go              # `portia scan` - directory scanning
│   │   ├── git.go               # `portia git` - git history scanning
│   │   ├── init.go              # `portia init` + `portia pyproject` - create config files
│   │   └── config.go            # `portia config` - show current config
│   ├── config/
│   │   └── config.go            # TOML config loader, defaults, search paths
│   ├── engine/
│   │   ├── detector.go          # Applies rules to chunks, entropy validation
│   │   ├── filter.go            # Stopwords, placeholders, templates, allowlists
│   │   └── pipeline.go          # errgroup worker pool, deduplication
│   ├── hibp/
│   │   └── client.go            # k-anonymity API client, LRU cache, circuit breaker
│   ├── reporter/
│   │   ├── reporter.go          # Reporter interface and factory
│   │   ├── terminal.go          # Colored terminal output with masking
│   │   ├── json.go              # JSON output format
│   │   └── sarif.go             # SARIF v2.1.0 output for CI integration
│   ├── rules/
│   │   ├── builtin.go           # 150 detection rules covering AWS, GitHub, etc.
│   │   ├── entropy.go           # Shannon entropy calculator and charset detection
│   │   └── registry.go          # Rule storage, keyword matching, global allowlists
│   ├── source/
│   │   ├── source.go            # Source interface definition
│   │   ├── directory.go         # Filesystem walker with 50-line chunking
│   │   └── git.go               # Git history scanner using go-git
│   └── ui/                      # Colors, spinner, banner, symbols
├── pkg/types/types.go           # Core types: Finding, Chunk, Rule, Severity
├── testdata/fixtures/            # Test secrets for integration testing
├── Justfile                      # Task runner
└── .golangci.yml                 # Linter configuration
```

## Comandi di sviluppo

Questo progetto usa [`just`](https://github.com/casey/just) come command runner. Esegui `just` senza argomenti per vedere tutti i comandi disponibili.

| Comando | Descrizione |
|---------|-------------|
| `just lint` | Esegue golangci-lint |
| `just lint-fix` | Esegue golangci-lint con auto-fix |
| `just format` | Formatta il codice tramite golangci-lint |
| `just vet` | Esegue `go vet` |
| `just tidy` | Esegue `go mod tidy` |
| `just test` | Esegue tutti i test con il race detector |
| `just test-v` | Esegue i test con output verboso |
| `just cover` | Esegue i test con riepilogo della coverage |
| `just cover-html` | Genera il report di coverage HTML |
| `just ci` | Esegue lint + test (controllo CI completo) |
| `just check` | Esegue lint + vet |
| `just run <args>` | Esegue portia con degli argomenti (ad es. `just run scan .`) |
| `just dev-scan` | Analizza la directory testdata |
| `just dev-git` | Analizza la cronologia git del repo corrente |
| `just dev-json` | Analizza testdata con output JSON |
| `just dev-sarif` | Analizza testdata con output SARIF |
| `just dev-rules` | Elenca tutte le regole di detection |
| `just build` | Build di produzione in `bin/portia` |
| `just install` | `go install` in `$GOPATH/bin` |
| `just info` | Mostra le informazioni su progetto/Go/OS |
| `just clean` | Rimuove gli artefatti di build |

## Prossimi passi

Lavora sui documenti in ordine:

1. **01-CONCEPTS.it.md** - Concetti di sicurezza: secret sprawl, entropia di Shannon, detection basata su regex, filtraggio dei falsi positivi, k-anonymity, SARIF
2. **02-ARCHITECTURE.it.md** - Progettazione del sistema: architettura della pipeline, modello di concorrenza, flusso dei dati, interazioni tra componenti
3. **03-IMPLEMENTATION.it.md** - Walkthrough del codice: ogni file chiave spiegato con riferimenti alle righe
4. **04-CHALLENGES.it.md** - Estensioni: hook pre-commit, regole personalizzate, scansione incrementale, GitHub Actions

## Problemi comuni

**"No secrets detected" su codice reale:**
Il tuo codice potrebbe essere pulito. Prova prima a scansionare `testdata/fixtures/` per confermare che il tool funzioni. Se stai scansionando codice reale, verifica di non escludere troppi percorsi. Usa `--verbose` per vedere quali file vengono analizzati.

**La scansione git è lenta:**
Usa `--depth` per limitare quanti commit analizzare. `--depth 100` analizza gli ultimi 100 commit invece dell'intera cronologia. Per un controllo rapido, usa `--staged` per analizzare solo le modifiche in staging.

**Troppi falsi positivi:**
Modifica `.portia.toml` per aggiungere percorsi o valori alla allowlist. Puoi anche disabilitare regole specifiche con `rules.disable = ["generic-password"]`. Il sistema di filtri (stopword, placeholder, template) gestisce la maggior parte dei casi, ma i pattern specifici del progetto potrebbero richiedere una allowlist personalizzata.

**Il controllo HIBP fallisce:**
L'API HIBP ha dei rate limit. Il circuit breaker in `internal/hibp/client.go` si aprirà dopo 5 fallimenti consecutivi, impedendo ulteriori richieste per 60 secondi. È una scelta progettuale. Se l'API è costantemente non disponibile, la scansione viene comunque completata senza i dati sulle violazioni.

**Errori di build con la versione di Go:**
Questo progetto usa la sintassi `for range p.workers` che richiede Go 1.22+. Controlla la tua versione con `go version`.

## Progetti correlati

Altri progetti in questo repository che completano la scansione dei secret:

- **api-security-scanner** (`PROJECTS/intermediate/api-security-scanner/`) - Analizza le API alla ricerca di misconfiguration di sicurezza. Dove Portia trova i secret nel codice, questo progetto verifica se le API sono adeguatamente protette a runtime.
- **docker-security-audit** (`PROJECTS/intermediate/docker-security-audit/`) - Esegue l'audit di immagini e configurazioni Docker per problemi di sicurezza. Completa Portia controllando gli ambienti container dove i secret sono spesso esposti tramite variabili d'ambiente.
- **api-rate-limiter** (`PROJECTS/advanced/api-rate-limiter/`) - Implementa algoritmi di rate limiting. Rilevante perché il client HIBP di Portia usa lo stesso pattern circuit breaker per gestire con eleganza i rate limit delle API esterne.
