# 02-ARCHITECTURE.md

# Architettura

Questo documento tratta il design del sistema di credenum: come si collegano i componenti, perché sono strutturati in questo modo e i compromessi alla base delle decisioni di progettazione.

## Architettura di alto livello

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          CLI (harvester.nim)                             │
│  Parse arguments → Validate config → Route to dry-run or scan           │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │ HarvestConfig
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         Runner (runner.nim)                              │
│  Iterate enabled modules → Dispatch to collectors → Aggregate results   │
└───────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┘
        │          │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼          ▼
  ┌──────────┐┌─────────┐┌───────┐┌─────────┐┌─────────┐┌──────────┐
  │ browser  ││  ssh    ││ cloud ││ history ││ keyring ││   git    │
  └──────────┘└─────────┘└───────┘└─────────┘└─────────┘└──────────┘
        │          │          │          │          │          │
        └──────────┴──────────┴──────────┴──────────┴──────────┘
                                   │
                            ┌──────┴──────┐
                            │  apptoken   │
                            └─────────────┘
                                   │
                    All collectors share base.nim utilities
                    All return CollectorResult
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         Report Assembly                                  │
│  Combine CollectorResults → Calculate severity summary → Add metadata   │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ Report
                    ┌──────────┴──────────┐
                    ▼                     ▼
          ┌─────────────────┐   ┌─────────────────┐
          │ terminal.nim    │   │ json.nim         │
          │ Box-drawn color │   │ Structured JSON  │
          │ output to stdout│   │ to stdout/file   │
          └─────────────────┘   └─────────────────┘
                    │                     │
                    ▼                     ▼
             Exit code 0/1         Exit code 0/1
          (0=clean, 1=high/critical findings)
```

## Analisi dei componenti

### Livello CLI (harvester.nim)

**Scopo:** eseguire il parsing degli argomenti da riga di comando, costruire la configurazione e orchestrare il ciclo di vita della scansione.

**Responsabilità:**
- Eseguire il parsing dei flag CLI usando `parseopt` di Nim (nessuna dipendenza esterna)
- Costruire un `HarvestConfig` con impostazioni convalidate
- Cortocircuitare l'esecuzione per `--help`, `--version` e `--dry-run`
- Chiamare il runner, eseguire il rendering dell'output e determinare l'exit code

**Interfacce:**
- Input: argomenti grezzi da riga di comando
- Output: exit code (0 o 1), output renderizzato su stdout

Il livello CLI è deliberatamente sottile. Non esegue alcuna scansione né alcuna formattazione dell'output. Costruisce la configurazione, chiama il runner, sceglie il formattatore ed esce.

### Runner (runner.nim)

**Scopo:** mappare le categorie di moduli abilitate sulle rispettive implementazioni dei collector e aggregare i risultati.

**Responsabilità:**
- Mantenere la tabella di routing `Category → CollectorProc` tramite `getCollector`
- Iterare su `config.enabledModules` e invocare ciascun collector
- Cronometrare la durata complessiva della scansione
- Sommare i conteggi per gravità nel riepilogo del report

**Interfacce:**
- Input: `HarvestConfig`
- Output: `Report` (metadati + risultati + riepilogo)

Il runner non sa come funzionano internamente i collector. Ottiene un puntatore a funzione da `getCollector`, lo chiama e raccoglie il risultato. Questo significa che aggiungere una nuova categoria di collector richiede solo di aggiungere un case alla tabella di routing e importare il modulo.

### Collector (collectors/*.nim)

**Scopo:** ogni collector scansiona una categoria di esposizione delle credenziali e restituisce i rilevamenti.

**Responsabilità:**
- Scansionare i percorsi di file noti per la categoria
- Analizzare il contenuto e i permessi dei file
- Costruire oggetti `Finding` con la classificazione della gravità
- Restituire un `CollectorResult` con tutti i rilevamenti ed eventuali errori

**Interfaccia condivisa:** ogni collector esporta una proc `collect` con la firma:

```nim
proc collect*(config: HarvestConfig): CollectorResult {.nimcall, raises: [].}
```

Questa firma uniforme è ciò che fa funzionare la tabella di routing. Il runner non ha bisogno di sapere quale collector sta chiamando.

### Utility di base (collectors/base.nim)

**Scopo:** fornire operazioni condivise sul file system e helper per la costruzione dei rilevamenti.

**Responsabilità:**
- Verifiche sicure dell'esistenza di file/directory (intercettando le eccezioni del filesystem)
- Ispezione dei permessi basata su POSIX stat
- Estrazione dei metadati dei file (dimensione, data di modifica, stringa dei permessi)
- Funzioni factory per rilevamenti e credenziali
- Calcolo della gravità basato sui permessi
- Oscuramento dei valori per le anteprime delle credenziali

Il modulo base è l'unico posto che chiama direttamente le syscall POSIX. Tutti i collector passano attraverso `safeFileExists`, `readFileContent`, `getNumericPerms` e i costruttori `makeFinding`/`makeFindingWithCred` invece di usare direttamente `os` e `posix`.

### Renderer dell'output (output/*.nim)

**Scopo:** trasformare un `Report` in un output leggibile dall'uomo o dalla macchina.

**terminal.nim:**
- Esegue il rendering del banner ASCII, delle intestazioni dei moduli con box drawing, dei badge di gravità con colori ANSI, dei dettagli dei rilevamenti e di un footer di riepilogo
- Gestisce il calcolo della lunghezza visiva per le stringhe che contengono sequenze di escape ANSI e caratteri UTF-8 multi-byte
- Rispetta `--quiet` (sopprime il banner) e `--verbose` (mostra i moduli vuoti)

**json.nim:**
- Converte l'intero `Report` in una struttura JSON annidata
- Scrive su stdout e opzionalmente su un file tramite `--output`
- Usa il modulo `json` della stdlib di Nim per la serializzazione

Entrambi i renderer sono consumatori in sola lettura del tipo `Report`. Non modificano i dati né innescano effetti collaterali oltre alla scrittura su stdout/file.

## Flusso dei dati

### Flusso principale: scansione completa

1. L'utente esegue `credenum --modules ssh,cloud --format terminal`
2. `parseCli()` esegue il parsing degli argomenti in `HarvestConfig` con `enabledModules = @[catSsh, catCloud]`
3. `main()` chiama `runCollectors(config)`
4. Il runner itera su `[catSsh, catCloud]`:
   - Recupera `ssh.collect` tramite `getCollector(catSsh)` e lo chiama
   - `ssh.collect` chiama `scanKeys`, `scanConfig`, `scanAuthorizedKeys`, `scanKnownHosts`
   - Ogni sotto-scanner usa `base.nim` per verificare i file, leggerne il contenuto, analizzarne i permessi
   - Restituisce un `CollectorResult` con i rilevamenti e i tempi
   - Lo stesso vale per `cloud.collect` → `scanAws`, `scanGcp`, `scanAzure`, `scanKubernetes`
5. Il runner somma i conteggi per gravità in `Report.summary`
6. `main()` aggiunge il timestamp UTC ai metadati
7. `renderTerminal(report, quiet, verbose)` scrive l'output formattato
8. Se esistono rilevamenti CRITICAL o HIGH, esce con codice 1

### Flusso secondario: dry run

1. L'utente esegue `credenum --dry-run`
2. `parseCli()` imposta `config.dryRun = true`
3. `main()` chiama `renderDryRun(config)`, che stampa l'elenco dei moduli e la directory di destinazione
4. Esce con codice 0 (non avviene alcuna scansione)

### Flusso di costruzione dei rilevamenti

Quando un collector scopre un file di credenziali:

```
File exists? (safeFileExists)
    │
    ▼ yes
Read content (readFileContent)
    │
    ▼
Analyze content (is it a private key? does it contain tokens?)
    │
    ▼
Check permissions (getNumericPerms → isWorldReadable, isGroupReadable)
    │
    ▼
Determine severity (content analysis + permission analysis)
    │
    ▼
Build Finding:
  ├── makeFinding(path, desc, category, severity)        [no credential detail]
  └── makeFindingWithCred(path, desc, category, sev, cred)  [with credential detail]
        │
        ▼
  Finding includes:
    path, category, severity, description,
    permissions (from getPermsString),
    modified (from getModifiedTime),
    size (from getFileSizeBytes),
    credential (optional: source, type, preview, metadata)
```

## Design del sistema dei tipi

### Tipi core (types.nim)

```
                        ┌───────────────┐
                        │    Report     │
                        ├───────────────┤
                        │ metadata      │──── ReportMetadata
                        │ results       │──── seq[CollectorResult]
                        │ summary       │──── array[Severity, int]
                        └───────────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
          ┌─────────────────┐   ┌─────────────────┐
          │ ReportMetadata  │   │ CollectorResult  │
          ├─────────────────┤   ├─────────────────┤
          │ timestamp       │   │ name             │
          │ target          │   │ category         │
          │ version         │   │ findings         │──── seq[Finding]
          │ durationMs      │   │ durationMs       │
          │ modules         │   │ errors           │
          └─────────────────┘   └─────────────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │    Finding       │
                              ├─────────────────┤
                              │ path            │
                              │ category        │──── Category enum
                              │ severity        │──── Severity enum
                              │ description     │
                              │ credential      │──── Option[Credential]
                              │ permissions     │
                              │ modified        │
                              │ size            │
                              └─────────────────┘
                                        │
                              ┌─────────┴────────┐
                              ▼                  ▼
                    ┌──────────────┐    ┌──────────────┐
                    │  Severity    │    │  Credential   │
                    ├──────────────┤    ├──────────────┤
                    │ svInfo       │    │ source       │
                    │ svLow        │    │ credType     │
                    │ svMedium     │    │ preview      │
                    │ svHigh       │    │ metadata     │
                    │ svCritical   │    └──────────────┘
                    └──────────────┘
```

### Perché questa struttura

**Severity come enum con valori stringa:** ogni livello di gravità si mappa direttamente su un'etichetta di visualizzazione (`"info"`, `"low"`, `"medium"`, `"high"`, `"critical"`). Usare un enum invece di stringhe significa che i confronti di gravità sono operazioni su interi, che la gravità può essere usata come indice di array (per colori, etichette e conteggi del riepilogo) e che i valori di gravità non validi vengono intercettati a compile time.

**Finding con Credential opzionale:** non tutti i rilevamenti hanno dettagli sulle credenziali. Una directory SSH con permessi sbagliati è un rilevamento ma non ha una credenziale da mostrare. Un file `.pgpass` ha voci specifiche che vale la pena mostrare. Il tipo `Option[Credential]` rende questo esplicito invece di usare una struct Credential vuota.

**Credential con una Table di metadati:** tipi di credenziali diversi hanno attributi diversi. Una credenziale AWS ha il numero di profili e i conteggi delle chiavi statiche/di sessione. Una chiave SSH ha lo stato di cifratura. Una configurazione Kubernetes ha i conteggi di contesti e utenti. Un campo di metadati `Table[string, string]` gestisce questa variabilità senza bisogno di un tipo Credential diverso per ogni categoria.

**CollectorResult con una seq di errori:** le operazioni sul filesystem possono fallire (permessi negati, symlink rotti, directory mancanti). Invece di interrompersi, i collector intercettano gli errori e li aggiungono all'elenco degli errori. Il renderer dell'output mostra questi errori accanto ai rilevamenti, così l'utente sa cosa non è stato possibile scansionare.

**Report.summary come array[Severity, int]:** usare l'enum Severity come indice di array fornisce una lookup O(1) per i conteggi di gravità e rende naturale l'iterazione: `for sev in Severity: report.summary[sev]`.

### Tipi di configurazione

```nim
HarvestConfig = object
  targetDir: string           # Which home directory to scan
  enabledModules: seq[Category]  # Which collectors to run
  excludePatterns: seq[string]   # Paths/names to skip
  outputFormat: OutputFormat     # terminal, json, or both
  outputPath: string           # File path for JSON output
  dryRun: bool                 # Preview mode
  quiet: bool                  # Suppress banner
  verbose: bool                # Show empty modules
```

L'oggetto di configurazione viene passato per valore a ogni collector. I collector non lo modificano mai.

```nim
CollectorProc = proc(config: HarvestConfig): CollectorResult {.nimcall, raises: [].}
```

L'alias di tipo `CollectorProc` definisce il contratto che ogni collector deve soddisfare. Il pragma `raises: []` significa che la proc non può sollevare eccezioni. Questo è imposto a compile time da `{.push raises: [].}` all'inizio di ogni file.

## Design pattern

### Strategy Pattern (smistamento dei collector)

Il runner usa lo strategy pattern per lo smistamento dei collector. Ogni Category si mappa su una funzione collector con una firma identica:

```
getCollector(catBrowser) → browser.collect
getCollector(catSsh)     → ssh.collect
getCollector(catCloud)   → cloud.collect
...
```

Si tratta di un'istruzione case invece di una tabella perché il controllo di esaustività dei case di Nim garantisce che ogni Category abbia un handler. Se aggiungi una nuova Category all'enum, il compilatore ti obbliga a gestirla in `getCollector`.

### Factory Pattern (costruzione dei rilevamenti)

Le funzioni `makeFinding` e `makeFindingWithCred` in base.nim sono factory che gestiscono il codice ripetitivo della costruzione di un Finding: recuperare i permessi, la data di modifica e la dimensione del file. I collector chiamano la factory con solo il percorso, la descrizione, la categoria e la gravità. Questo previene incoerenze in cui un collector dimentica di impostare la data di modifica o usa un formato dei permessi diverso.

### Scansione a strati (all'interno dei collector)

Ogni collector usa internamente un approccio di scansione a strati in cui dei sotto-scanner gestiscono aspetti specifici:

```
ssh.collect
  ├── scanKeys         # Private key files
  ├── scanConfig       # SSH client configuration
  ├── scanAuthorizedKeys   # Authorized public keys
  └── scanKnownHosts      # Known host entries
```

I sotto-scanner modificano il `CollectorResult` sul posto tramite un parametro `var` invece di restituire risultati separati. Questo evita l'overhead di allocazione dovuto all'unione di più sequenze.

## Modello di classificazione della gravità

Il modello di gravità ha due input: l'analisi del contenuto e l'analisi dei permessi. Vince la gravità più alta.

```
Content-based severity:
  ┌─────────────────────────────────────────────────┐
  │ Unencrypted SSH key           → HIGH            │
  │ Encrypted SSH key             → INFO            │
  │ Plaintext Git credentials     → HIGH            │
  │ AWS static keys (AKIA)        → HIGH            │
  │ GCP service account key       → HIGH            │
  │ Database password file        → HIGH            │
  │ Secret in shell history       → HIGH            │
  │ Config file (no secrets)      → INFO            │
  └─────────────────────────────────────────────────┘

Permission-based override:
  ┌─────────────────────────────────────────────────┐
  │ World-readable (0o004 bit)    → CRITICAL        │
  │ Group-readable (0o040 bit)    → MEDIUM or HIGH  │
  │ Looser than expected          → LOW             │
  │ Owner-only                    → (no override)   │
  └─────────────────────────────────────────────────┘
```

La gravità finale è il massimo tra quella basata sul contenuto e quella basata sui permessi. Una chiave SSH cifrata (INFO dal contenuto) che è leggibile da chiunque (CRITICAL dai permessi) diventa CRITICAL.

## Architettura della configurazione

### Zero configurazione esterna

Il tool non ha file di configurazione, variabili d'ambiente né dotfile. Tutta la configurazione proviene dai flag CLI con valori predefiniti sensati:

| Impostazione | Predefinito | Override |
|---------|---------|----------|
| Directory di destinazione | Home dell'utente corrente | `--target <path>` |
| Moduli abilitati | Tutti e 7 | `--modules ssh,git,...` |
| Pattern di esclusione | Nessuno | `--exclude .cache,vendor` |
| Formato di output | Terminale | `--format json\|both` |
| File di output | Nessuno (solo stdout) | `--output report.json` |

È intenzionale. Un tool di sicurezza che legge la configurazione dal filesystem crea una dipendenza circolare: stai cercando configurazioni esposte mentre ti affidi a una configurazione che potrebbe essere stata essa stessa manomessa.

### Architettura delle costanti

Tutti i target di scansione, i pattern e le soglie sono definiti come costanti a compile time in `config.nim`. Questo significa:

- Nessun parsing della configurazione a runtime
- Nessuna allocazione per elenchi di percorsi o array di pattern
- Il compilatore può fare l'inline e ottimizzare tutte le lookup delle costanti
- Aggiungere un nuovo target di scansione è una modifica di una sola riga a un array di costanti

Le costanti sono organizzate per categoria (SSH, AWS, GCP, browser, ecc.) con raggruppamenti chiari. Anche le costanti della UI (colori, caratteri di box drawing, etichette di gravità) si trovano in config.nim per mantenerle centralizzate.

## Strategia di gestione degli errori

### Nessuna eccezione, mai

Ogni file del progetto inizia con `{.push raises: [].}`. Questo pragma di Nim dice al compilatore che a nessuna proc in questo file è consentito sollevare un'eccezione. Qualsiasi chiamata a una funzione che potrebbe sollevarne una (I/O su file, operazioni sulle stringhe, ecc.) deve essere racchiusa in un try/except.

Questo è imposto a compile time. Se aggiungi una chiamata a `readFile()` senza un try/except, il compilatore la rifiuterà con un errore che mostra esattamente quale funzione potrebbe sollevare un'eccezione e dove.

### Schema di recupero dagli errori

Ogni operazione sul file system segue lo stesso schema:

1. Tenta l'operazione
2. In caso di successo, elabora il risultato
3. In caso di fallimento, restituisci un valore predefinito sicuro oppure aggiungi l'errore all'elenco degli errori

```
safeFileExists()  → returns false on exception
readFileContent() → returns "" on exception
getNumericPerms() → returns -1 on exception
walkDir()         → caught at collector level, added to errors[]
```

I collector non si interrompono mai. Un attraversamento di directory fallito aggiunge un messaggio di errore e continua a scansionare gli altri percorsi. Il report finale mostra sia i rilevamenti sia gli errori, così l'utente sa cosa è stato scansionato e cosa è stato saltato.

### Exit code

| Codice | Significato |
|------|---------|
| 0 | Scansione completata, nessun rilevamento HIGH o CRITICAL |
| 1 | Scansione completata, rilevati rilevamenti HIGH o CRITICAL |

L'exit code viene determinato dopo che tutto l'output è stato renderizzato. Questo permette di usare il tool in script e pipeline CI: `credenum --quiet --format json -o report.json && echo "clean" || echo "findings detected"`.

## Considerazioni sulle prestazioni

### Esecuzione sequenziale dei collector

I collector vengono eseguiti in sequenza, non in parallelo. È una scelta deliberata:

- La maggior parte dei collector si completa in meno di 10ms (il target è una singola home directory, non una scansione di rete)
- L'esecuzione sequenziale produce un ordinamento dell'output deterministico
- Nessun overhead di sincronizzazione tra thread
- Nessun rischio di race TOCTOU sui metadati dei file
- Il tempo totale di scansione è tipicamente inferiore a 100ms per un'intera home directory

Se fosse necessaria l'esecuzione parallela (ad es. per condivisioni di rete o home directory molto grandi), la firma uniforme `CollectorProc` rende semplice la parallelizzazione: avvia ogni collector in un thread e raccogli i risultati.

### Efficienza della memoria

- Tutti i dati costanti (percorsi, pattern, colori) sono compilati nel binario, non allocati a runtime
- Il contenuto dei file viene letto interamente in stringhe (non in streaming) perché i file di credenziali sono piccoli (tipicamente sotto 1KB)
- La scansione della cronologia della shell si ferma a 50.000 righe per prevenire un uso di memoria illimitato su file di cronologia estremamente grandi
- L'attraversamento ricorsivo delle directory (per i file .env e i database .kdbx) è limitato a una profondità di 5 livelli
- I rilevamenti vengono raccolti in sequenze che crescono secondo necessità, senza pre-allocazione

### Dimensione del binario

| Build | Dimensione | Note |
|-------|------|-------|
| Debug | ~2MB | Simboli completi, bounds checking |
| Release | ~500KB | Ottimizzata, LTO, simboli rimossi |
| Release statica | ~400KB | musl, nessuna dipendenza da glibc |
| Compressa | ~150KB | UPX applicato sopra la release statica |

Il binario compresso è abbastanza piccolo da essere trasferito su connessioni lente o incorporato in altri tool.

## Estensibilità

### Aggiungere un nuovo collector

Per aggiungere un nuovo collector (ad es. per le credenziali dei container):

1. Aggiungi un nuovo valore all'enum `Category` in `types.nim`
2. Aggiungi il nome del modulo e la descrizione agli array in `config.nim`
3. Crea `collectors/container.nim` implementando `proc collect*(config: HarvestConfig): CollectorResult`
4. Aggiungi l'import e il ramo del case in `getCollector` di `runner.nim`

Il compilatore ti guiderà: aggiungere un valore all'enum Category senza gestirlo nell'istruzione case produce un errore di compilazione.

### Aggiungere un nuovo formato di output

Per aggiungere un nuovo formato di output (ad es. SARIF):

1. Aggiungi un valore all'enum `OutputFormat`
2. Crea `output/sarif.nim` con una `proc renderSarif*(report: Report, outputPath: string)` 
3. Aggiungi il ramo del case nella funzione main di `harvester.nim`

Il modulo di output riceve il `Report` completo e ha piena libertà su come renderizzarlo.

### Aggiungere nuovi target di scansione

Per scansionare un nuovo file di credenziali all'interno di una categoria esistente:

1. Aggiungi la costante del percorso a `config.nim`
2. Aggiungi la logica di scansione nel collector pertinente
3. Aggiungi una test fixture in `tests/docker/planted/` e una verifica in `validate.sh`

Nessun altro file deve cambiare.

## Limitazioni

**Solo Linux.** Il tool prende di mira i percorsi di archiviazione delle credenziali di Linux. macOS memorizza le credenziali in modo diverso (Keychain, percorsi dei browser diversi). Windows ha un modello completamente diverso (Credential Manager, DPAPI). Supportare altre piattaforme richiederebbe implementazioni dei collector specifiche per piattaforma.

**Sola lettura.** Il tool rileva ma non esegue la remediation. Non correggerà i permessi, non cifrerà le chiavi né ruoterà le credenziali. È intenzionale: un tool di scansione non dovrebbe mai modificare il filesystem che sta ispezionando.

**Percorsi statici.** I percorsi delle credenziali sono compilati nel binario come costanti. Le installazioni non standard (ad es. Firefox installato tramite Flatpak, browser basati su Snap, layout personalizzati della directory HOME) potrebbero memorizzare le credenziali in posizioni diverse che il tool non controlla.

**Nessuna decifratura delle credenziali.** Il tool identifica gli archivi di credenziali cifrati ma non tenta di decifrarli. Verifica se la cifratura è presente e se i permessi dei file espongono i dati cifrati, ma non valuta la robustezza della cifratura né tenta il brute force.

## Riferimento ai file principali

| File | Scopo |
|------|---------|
| `src/harvester.nim` | Entry point, parsing della CLI, ciclo principale |
| `src/runner.nim` | Smistamento dei collector e aggregazione dei risultati |
| `src/types.nim` | Tutte le definizioni dei tipi |
| `src/config.nim` | Tutte le costanti (percorsi, pattern, colori, UI) |
| `src/collectors/base.nim` | Utility condivise per tutti i collector |
| `src/collectors/*.nim` | Un file per ogni categoria di credenziali |
| `src/output/terminal.nim` | Renderer per il terminale con box drawing |
| `src/output/json.nim` | Serializzatore JSON |
| `tests/test_all.nim` | Unit test |
| `tests/docker/` | Infrastruttura dei test di integrazione Docker |
| `config.nims` | Configurazione del compilatore Nim |
| `Justfile` | Comandi di automazione della build |

## Prossimi passi

- Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per vedere come è implementato ciascun componente, con spiegazioni del codice del parser della CLI, dei moduli collector, dell'analisi dei permessi e del rendering dell'output
- Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee di estensione
