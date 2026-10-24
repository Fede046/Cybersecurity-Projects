# Sfide di Estensione

Queste sfide estendono Portia oltre le sue capacità attuali. Ognuna insegna una competenza diversa. Sono ordinate approssimativamente per difficoltà e si basano sulla codebase esistente senza richiedere refactoring importanti.

## Sfida 1: Integrazione con Pre-commit Hook

**Difficoltà:** Facile | **Tempo:** 1-2 ore | **Insegna:** Git hook, shell scripting, workflow dello sviluppatore

Scrivi uno script che installi Portia come pre-commit hook di Git. Quando uno sviluppatore esegue `git commit`, l'hook deve:

1. Compilare Portia (oppure usare un binario già compilato)
2. Eseguire `portia git --staged` per scansionare solo i file in staging
3. Se vengono trovati secret, stampare i risultati e interrompere il commit
4. Se non vengono trovati secret, permettere al commit di procedere

**Punto di partenza:** Crea uno `scripts/install-hook.sh` che scriva un pre-commit hook in `.git/hooks/pre-commit`. Lo script dell'hook deve chiamare `portia git --staged --format terminal` e controllare l'exit code.

**Suggerimenti:**
- Gli hook di Git devono essere eseguibili (`chmod +x`)
- L'hook deve uscire con 0 per permettere il commit, con un valore diverso da zero per interromperlo
- Dovrai aggiungere il supporto agli exit code alla CLI di Portia (attualmente esce sempre con 0). Aggiungi un flag `--exit-code` che restituisca exit code 1 quando vengono trovati secret. Modifica `executeScan` in `internal/cli/scan.go` in modo che chiami `os.Exit(1)` quando esistono dei finding e il flag è impostato.
- Valuta l'aggiunta di un flag `--quiet` che sopprima il banner e lo spinner per l'uso negli hook

**Bonus:** Fai in modo che funzioni con il framework `pre-commit` (https://pre-commit.com) creando un `.pre-commit-hooks.yaml` nella root del repository.

## Sfida 2: Loader YAML per Regole Personalizzate

**Difficoltà:** Media | **Tempo:** 2-3 ore | **Insegna:** Parsing YAML/TOML, validazione delle regole, estensibilità della configurazione

Aggiungi il supporto per regole di detection definite dall'utente in un file YAML o TOML. Gli utenti dovrebbero poter creare `.portia/rules.yml`:

```yaml
rules:
  - id: "internal-api-key"
    description: "Internal API key format"
    severity: HIGH
    keywords: ["ikey_"]
    pattern: 'ikey_[a-zA-Z0-9]{32}'
    secret_group: 0
    entropy: 3.5
```

**Punto di partenza:** Crea `internal/rules/custom.go` con una funzione `LoadCustomRules(path string) ([]*types.Rule, error)`. Chiamala da `scan.go` dopo aver registrato i builtin.

**Suggerimenti:**
- Usa `gopkg.in/yaml.v3` per il parsing YAML
- Valida il pattern regex chiamando `regexp.Compile` e restituendo un errore chiaro se fallisce
- Valida la severity rispetto ai valori consentiti
- Controlla la presenza di ID di regola duplicati rispetto al registry esistente
- Valuta il supporto di `allowlist` nel formato YAML con pattern per path e per valore

**Trabocchetto:** Le keyword sono fondamentali per le performance. Se una regola personalizzata non ha keyword, eseguirà la sua regex su ogni chunk. Oppure richiedi almeno una keyword, oppure avvisa l'utente che l'assenza di keyword sarà lenta.

## Sfida 3: Scansione Incrementale con Cache

**Difficoltà:** Media | **Tempo:** 2-3 ore | **Insegna:** Hashing, caching su file, ottimizzazione delle performance

Aggiungi un file `.portia-cache/scan.json` che memorizzi gli hash SHA-256 dei file scansionati in precedenza. Nelle scansioni successive, salta i file il cui hash non è cambiato.

**Punto di partenza:** Crea `internal/cache/scan.go` con:
- `type ScanCache struct` che contenga una mappa dal path relativo del file all'hash del file
- `Load(path) (*ScanCache, error)` e `Save(path) error` per la persistenza
- `IsChanged(relPath string, content []byte) bool` che calcoli lo SHA-256 e lo confronti

**Suggerimenti:**
- Memorizza la cache in `.portia-cache/scan.json` nella directory scansionata
- Usa `crypto/sha256` per l'hashing
- La cache dovrebbe includere il numero di regole come metadato. Se le regole cambiano (viene aggiunta una nuova regola), invalida l'intera cache.
- Aggiungi un flag `--no-cache` per forzare una nuova scansione completa
- Aggiungi l'invalidazione della cache in `internal/cli/scan.go` prima di creare la source
- Valuta l'aggiunta della versione di Portia ai metadati della cache, in modo che gli upgrade di versione invalidino la cache

**Impatto sulle performance:** Su una codebase di 10.000 file in cui sono cambiati solo 50 file, questo riduce il tempo di scansione di circa il 99,5%.

## Sfida 4: Integrazione con Git Blame

**Difficoltà:** Media | **Tempo:** 3-4 ore | **Insegna:** API di Git blame, attribuzione, output arricchito

Dopo aver rilevato un secret, esegui `git blame` sul file per determinare chi lo ha committato e quando. Aggiungi queste informazioni al finding.

**Punto di partenza:** La struct `types.Finding` ha già i campi `Author` e `CommitDate`, ma vengono popolati solo durante le scansioni della cronologia git. Per le scansioni di directory, questi campi sono vuoti.

**Suggerimenti:**
- Usa la funzione `git.Blame` di go-git: `blame, err := git.BlameCommit(commit, path)`
- Il risultato del blame ti fornisce SHA del commit, autore e data per ogni riga
- Fai corrispondere il `LineNumber` del finding al risultato del blame per ottenere l'attribuzione
- Questa funzione dovrebbe essere opt-in (flag `--blame`) dato che aggiunge overhead
- Per i file fuori da un repository git, salta il blame silenziosamente
- Aggiungi i dati del blame a tutti e tre i formati dei reporter (terminal, JSON, SARIF)

**Trabocchetto:** `git.Blame` richiede di percorrere l'intera cronologia dei commit per il file. Su repository grandi, questo può essere lento. Valuta il caching dei risultati del blame per file.

## Sfida 5: Scansione Multi-repo

**Difficoltà:** Media | **Tempo:** 3-4 ore | **Insegna:** Gestione della configurazione, I/O concorrente, aggregazione

Aggiungi un comando `portia scan-all` che legga un file di configurazione contenente un elenco di più repository e li scansioni tutti, producendo un report unificato.

```toml
[[repos]]
path = "/home/dev/api-server"
excludes = ["vendor/"]

[[repos]]
path = "/home/dev/frontend"
excludes = ["node_modules/", "dist/"]

[[repos]]
url = "https://github.com/org/service.git"
branch = "main"
depth = 50
```

**Punto di partenza:** Crea `internal/cli/scanall.go` con un nuovo comando cobra.

**Suggerimenti:**
- Esegui il parsing del file di configurazione con `pelletier/go-toml`
- Per le voci `url`, clona in una directory temporanea usando `git.PlainClone`
- Esegui la scansione di ogni repository in modo concorrente usando un errgroup
- Prefissa il `FilePath` di ogni finding con il nome/path del repository per disambiguare
- Valuta un flag `--parallel N` per controllare la concorrenza
- Ripulisci le directory temporanee clonate all'uscita (usa `defer`)

## Sfida 6: GitHub Action

**Difficoltà:** Difficile | **Tempo:** 4-6 ore | **Insegna:** GitHub Actions, Docker, integrazione SARIF, CI/CD

Costruisci una GitHub Action che esegua Portia sulle pull request e carichi i risultati su GitHub Code Scanning.

**Punto di partenza:** Crea `.github/action/action.yml` e un Dockerfile.

**Struttura:**
```
.github/action/
├── action.yml        # Action metadata
├── Dockerfile        # Build Portia in a container
└── entrypoint.sh     # Run Portia and upload SARIF
```

**Suggerimenti:**
- L'`action.yml` dovrebbe accettare gli input: `path` (default `.`), `format` (default `sarif`), `exclude` (opzionale), `hibp` (default false)
- Il Dockerfile dovrebbe essere una build multi-stage: compila Portia in un'immagine Go, copia il binario in un'immagine runtime slim
- `entrypoint.sh` esegue `portia scan --format sarif $INPUT_PATH > results.sarif`, poi carica usando `gh api repos/{owner}/{repo}/code-scanning/sarifs`
- Usa `github.sha` per lo SHA del commit nell'upload SARIF
- L'Action dovrebbe fallire (exit 1) se vengono rilevati finding CRITICAL o HIGH

**Testing:** Crea un `.github/workflows/test-action.yml` che testi l'action su `testdata/fixtures/`.

## Sfida 7: Suggerimenti per la Rotazione dei Secret

**Difficoltà:** Difficile | **Tempo:** 4-6 ore | **Insegna:** API dei provider, indicazioni di remediation, output strutturato

Dopo aver rilevato un secret leaked, fornisci istruzioni di rotazione specifiche per ogni provider.

**Punto di partenza:** Crea `internal/remediation/remediation.go` con una mappa da ID di regola a passaggi di remediation.

**Esempio di output:**
```
CRITICAL  aws-access-key-id  config.py:1

  Rotation steps:
  1. Go to AWS IAM Console → Users → Security credentials
  2. Create a new access key
  3. Update all services using the old key
  4. Deactivate the old key (don't delete yet)
  5. After 24-48 hours with no issues, delete the old key
  6. Run: aws sts get-caller-identity (to verify new key works)

  Documentation: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html
```

**Suggerimenti:**
- Crea una struct `Remediation` con `Steps []string`, `DocURL string`, `CLICommand string`
- Mappa gli ID di regola alle remediation: `aws-access-key-id` → rotazione AWS IAM, `github-pat-classic` → rotazione dalle impostazioni di GitHub, `stripe-live-secret` → rotazione dalla dashboard di Stripe
- Aggiungi un flag `--remediate` alla CLI
- Per il reporter terminal, stampa i passaggi di remediation indentati sotto ogni finding
- Per JSON/SARIF, includi la remediation nelle properties

## Sfida 8: Keyword Matching con Aho-Corasick

**Difficoltà:** Difficile | **Tempo:** 3-4 ore | **Insegna:** Strutture dati trie, algoritmi di string matching, performance

Sostituisci la scansione lineare delle keyword in `MatchKeywords` con un automa di Aho-Corasick per un matching O(n) contro tutte le keyword contemporaneamente.

**Approccio attuale** (`internal/rules/registry.go` MatchKeywords):
Per ogni regola, per ogni keyword, chiama `strings.Contains`. Questo è O(regole * keyword * lunghezza_contenuto).

**Approccio migliore:**
Costruisci un trie da tutte le keyword al momento dell'inizializzazione del registry. Al momento della scansione, fai passare il contenuto nell'automa una sola volta. L'automa riporta quali keyword hanno trovato corrispondenza, e tu le mappi di nuovo sulle regole.

**Punto di partenza:** Usa `github.com/cloudflare/ahocorasick` oppure implementa il tuo.

**Suggerimenti:**
- Costruisci l'automa in `Registry.Register` oppure in un metodo `Finalize()` chiamato dopo che tutte le regole sono state registrate
- L'automa dovrebbe essere case-insensitive (converti tutte le keyword e il contenuto in minuscolo)
- Mappa ogni keyword alla/e sua/e regola/e usando un indice inverso
- Fai un benchmark prima e dopo: `go test -bench=BenchmarkMatchKeywords -benchmem`
- Il miglioramento sarà più evidente su file grandi con molte regole. Su file piccoli con poche regole, l'overhead della costruzione dell'automa potrebbe renderlo più lento.

**Miglioramento atteso:** Su un file di 500 righe con 150 regole con una media di 2 keyword ciascuna, l'approccio attuale esegue circa 300 chiamate a `strings.Contains`. Aho-Corasick fa un solo passaggio sul contenuto. Per codebase grandi con migliaia di file, il vantaggio si accumula.

## Consigli Generali

- **Scrivi prima i test.** Ogni sfida dovrebbe iniziare con un test che fallisce. I pattern di test esistenti in `internal/engine/detector_test.go` e `internal/engine/integration_test.go` sono buoni modelli.
- **Mantieni le modifiche isolate.** Ogni sfida dovrebbe essere implementabile senza modificare la logica di detection di base. Usa interfacce e composizione per estendere anziché modificare.
- **Fai benchmark quando fai affermazioni sulle performance.** I benchmark `testing.B` di Go sono semplici da scrivere. Se affermi che qualcosa è più veloce, dimostralo con i numeri.
- **Controlla il Justfile.** Esegui `just ci` prima di considerare completa qualsiasi sfida. Tutti i test esistenti devono continuare a passare.
