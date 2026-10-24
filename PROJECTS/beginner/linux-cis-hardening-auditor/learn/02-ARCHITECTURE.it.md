<!-- © AngelaMos | 2026 | 02-ARCHITECTURE.md -->

# Architettura del sistema

Questo documento analizza come è progettato cisaudit, come i dati fluiscono attraverso il sistema e perché l'architettura modulare rende semplice aggiungere nuovi controlli, formati di output e versioni del benchmark.

---

## Architettura di alto livello

```
┌──────────────────────────────────────────────────────────┐
│                    cisaudit.sh (entry point)              │
│  Parse CLI args  →  Load modules  →  Orchestrate audit   │
└────────────────┬──────────────────────────┬──────────────┘
                 │                          │
    ┌────────────▼────────────┐  ┌──────────▼──────────────┐
    │   controls/             │  │   lib/                   │
    │   registry_data.sh      │  │   constants.sh           │
    │                         │  │   utils.sh               │
    │   104 register_control  │  │   registry.sh            │
    │   calls defining every  │  │   engine.sh              │
    │   control's metadata    │  │   baseline.sh            │
    └────────────┬────────────┘  │   report_terminal.sh     │
                 │               │   report_json.sh         │
                 │               │   report_html.sh         │
    ┌────────────▼────────────┐  └──────────────────────────┘
    │   checks/               │
    │   01_initial_setup.sh   │
    │   02_services.sh        │
    │   03_network.sh         │
    │   04_logging.sh         │
    │   05_access.sh          │
    │   05_access_password.sh │
    │   06_maintenance.sh     │
    │                         │
    │   check_X_Y_Z()         │
    │   functions that        │
    │   inspect the system    │
    │   and call              │
    │   record_result()       │
    └─────────────────────────┘
```

### Analisi dei componenti

**cisaudit.sh (Entry Point)**
- Esegue il parsing degli argomenti da riga di comando in variabili globali
- Carica (source) tutti i moduli di libreria, le definizioni dei controlli e le funzioni di controllo
- Orchestra l'audit: rileva il sistema operativo, esegue i controlli, calcola i punteggi, emette il report
- Gestisce il salvataggio/confronto della baseline e gli exit code basati sulla soglia

**controls/registry_data.sh (Registry dei controlli)**
- Contiene 104 chiamate `register_control`, ognuna delle quali definisce un controllo del benchmark CIS
- Ogni chiamata specifica: ID del controllo, nome della sezione, titolo, livello, stato scored, descrizione e comando di remediation
- È l'unica fonte di verità su ciò che il tool sottopone ad audit

**checks/*.sh (Funzioni di controllo)**
- Un file per ogni sezione del benchmark CIS
- Ogni file contiene funzioni `check_X_Y_Z()` che ispezionano il sistema e chiamano `record_result()`
- Le funzioni di controllo leggono i file di configurazione, interrogano i valori sysctl, verificano i permessi dei file e controllano la presenza di pacchetti installati

**lib/ (Librerie core)**
- `constants.sh`: versione, nome del benchmark, codici colore ANSI, codici di stato, nomi delle sezioni
- `utils.sh`: logging, rilevamento del sistema operativo, helper per la lettura dei file, helper sysctl, interrogazioni su servizi/pacchetti
- `registry.sh`: le funzioni `register_control()` e `record_result()`, memorizzazione dei risultati
- `engine.sh`: calcolo dei punteggi per sezione e per livello
- `report_*.sh`: tre formattatori di output (terminale, JSON, HTML)
- `baseline.sh`: salva i risultati come JSON, carica e confronta con una baseline precedente

---

## Flusso dei dati

### Flusso completo dell'audit

Spiegazione passo per passo di ciò che accade quando esegui `sudo cisaudit -l 1 -f json -o report.json`:

```
1. cisaudit.sh starts
   └─ source lib/constants.sh, utils.sh, registry.sh, engine.sh
   └─ source lib/report_*.sh, baseline.sh
   └─ source controls/registry_data.sh (populates CTRL_* arrays)
   └─ source checks/*.sh (defines check_X_Y_Z functions)

2. parse_args() processes CLI arguments
   └─ OPT_LEVEL="1", OPT_FORMAT="json", OPT_OUTPUT="report.json"

3. detect_os() reads /etc/os-release
   └─ DETECTED_ID="debian", DETECTED_VERSION="12"

4. run_checks() iterates REGISTERED_IDS[]
   └─ For each control ID:
       ├─ should_run_check() filters by level and category
       ├─ Calls check_X_Y_Z() function via CTRL_CHECK_FN lookup
       └─ check function calls record_result(id, status, evidence)
           └─ Appends to RESULT_STATUS[], RESULT_EVIDENCE[], RESULT_ORDER[]
           └─ Increments TOTAL_PASS/FAIL/WARN/SKIP counters

5. compute_scores() aggregates results
   └─ Counts pass/fail per section → SCORE_BY_SECTION[]
   └─ Computes SCORE_OVERALL, SCORE_LEVEL1, SCORE_LEVEL2

6. generate_report() dispatches to emit_json_report()
   └─ Iterates RESULT_ORDER[] and SECTION_ORDER[]
   └─ Prints JSON with metadata, summary, sections, controls
   └─ Writes to report.json via OPT_OUTPUT

7. Threshold check: if SCORE_OVERALL < OPT_THRESHOLD, exit 1
```

### Flusso di registrazione dei controlli

Ogni controllo segue lo stesso percorso dalla definizione al risultato:

```
registry_data.sh                    checks/0X_section.sh
       │                                   │
register_control(                    check_1_1_1() {
  "1.1.1",                              status = PASS
  "Initial Setup",                       evidence = ""
  "Ensure cramfs disabled",              # inspect system
  "1", "yes",                            # ...
  "description...",                      record_result(
  "remediation..."                         "1.1.1",
)                                          status,
       │                                   evidence
       ▼                                 )
CTRL_TITLE["1.1.1"]                  }
CTRL_SECTION["1.1.1"]                     │
CTRL_LEVEL["1.1.1"]                       ▼
CTRL_SCORED["1.1.1"]              RESULT_STATUS["1.1.1"]
CTRL_DESCRIPTION["1.1.1"]        RESULT_EVIDENCE["1.1.1"]
CTRL_REMEDIATION["1.1.1"]        RESULT_ORDER += "1.1.1"
CTRL_CHECK_FN["1.1.1"]           TOTAL_PASS++
  = "check_1_1_1"
```

La convenzione di denominazione è automatica. `register_control "1.1.1"` genera il nome di funzione `check_1_1_1` sostituendo i punti con underscore. La funzione di controllo deve esistere con quel nome esatto, altrimenti il motore registra uno SKIP con "Check function check_1_1_1 not implemented."

---

## Design pattern

### Registry Pattern

Il pattern architetturale centrale è un registry dei controlli. Tutti i 104 controlli sono registrati tramite un'unica funzione (`register_control`) in array associativi. Il motore non ha alcuna logica di controllo hardcoded. Itera su `REGISTERED_IDS[]`, recupera il nome della funzione di controllo da `CTRL_CHECK_FN[]` e la chiama dinamicamente.

Questo pattern significa che:
- Aggiungere un nuovo controllo richiede esattamente due modifiche: una chiamata `register_control` e una funzione `check_X_Y_Z`
- Il motore, i reporter e i moduli di baseline sono completamente indipendenti da quali controlli esistano
- I controlli possono essere aggiunti o rimossi senza modificare alcun codice core

```bash
register_control "1.1.1" \
    "Initial Setup" \
    "Ensure mounting of cramfs is disabled" \
    "1" \
    "yes" \
    "The cramfs filesystem type is..." \
    "echo 'install cramfs /bin/true' >> /etc/modprobe.d/cramfs.conf"
```

Questa chiamata popola sei array associativi e un array indicizzato. Il nome della funzione è derivato dall'ID: `check_1_1_1`.

### Strategy Pattern per i reporter

Tre moduli reporter (`report_terminal.sh`, `report_json.sh`, `report_html.sh`) espongono ciascuno un'unica funzione di ingresso: `emit_terminal_report`, `emit_json_report`, `emit_html_report`. La funzione `generate_report` in `cisaudit.sh` smista la chiamata a quella corretta in base a `OPT_FORMAT`.

Ogni reporter legge lo stesso stato globale (RESULT_STATUS, RESULT_EVIDENCE, SCORE_BY_SECTION, ecc.) e lo trasforma in un formato diverso. Non condividono codice tra loro perché ogni formato di output ha una struttura fondamentalmente diversa.

Aggiungere un nuovo formato (CSV, SARIF, Markdown) significa scrivere un nuovo file con una funzione `emit_X_report` e aggiungere un case a `generate_report`. Nient'altro cambia.

### Astrazione SYSROOT

Ogni accesso ai file passa attraverso funzioni helper che antepongono la variabile `SYSROOT`:

```bash
file_exists() {
    [[ -f "${SYSROOT}${1}" ]]
}

read_file() {
    local path="${SYSROOT}${1}"
    if [[ -f "$path" ]]; then
        cat "$path"
    else
        return 1
    fi
}

get_sysctl() {
    local param="$1"
    local proc_path="${SYSROOT}/proc/sys/${param//\.//}"
    if [[ -f "$proc_path" ]]; then
        cat "$proc_path"
        return 0
    fi
    # ...
}
```

Quando `SYSROOT="/"`, queste funzioni accedono al sistema reale. Quando `SYSROOT="testdata/fixtures"`, accedono al filesystem fittizio. Questa singola astrazione permette all'intera test suite di essere eseguita senza root e senza un sistema live.

---

## Separazione dei livelli

```
┌──────────────────────────────────────┐
│  Layer 1: CLI / Orchestration        │
│  cisaudit.sh                         │
│  - Parses arguments                  │
│  - Controls execution order          │
│  - Does NOT inspect the system       │
└──────────────────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│  Layer 2: Control Definitions        │
│  controls/registry_data.sh           │
│  - Declares what to check            │
│  - Contains no check logic           │
│  - Pure metadata                     │
└──────────────────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│  Layer 3: Check Functions            │
│  checks/01_initial_setup.sh ...      │
│  - Inspects the system               │
│  - Calls record_result()             │
│  - Does NOT format output            │
└──────────────────────────────────────┘
           ↓
┌──────────────────────────────────────┐
│  Layer 4: Scoring and Reporting      │
│  lib/engine.sh, report_*.sh          │
│  - Computes scores from results      │
│  - Formats output                    │
│  - Does NOT know what was checked    │
└──────────────────────────────────────┘
```

### Perché i livelli?

- **Testabilità**: le funzioni di controllo possono essere testate su filesystem fittizi senza coinvolgere il livello CLI. Il motore di punteggio può essere testato con dati di risultato sintetici senza eseguire alcun controllo.
- **Estensibilità**: una nuova versione del benchmark CIS significa aggiornare il Layer 2 (definizioni dei controlli) e il Layer 3 (funzioni di controllo). I Layer 1 e 4 restano invariati.
- **Separazione delle responsabilità**: una funzione di controllo non tocca mai la formattazione. Un reporter non tocca mai il filesystem. Ogni modulo ha un solo compito.

---

## Modelli di dati

cisaudit usa gli array associativi di Bash come modello di dati. Non ci sono data store esterni.

### Registry dei controlli (popolato da register_control)

```
CTRL_TITLE["1.1.1"]       = "Ensure mounting of cramfs is disabled"
CTRL_SECTION["1.1.1"]     = "Initial Setup"
CTRL_LEVEL["1.1.1"]       = "1"
CTRL_SCORED["1.1.1"]      = "yes"
CTRL_DESCRIPTION["1.1.1"] = "The cramfs filesystem type is..."
CTRL_REMEDIATION["1.1.1"] = "echo 'install cramfs /bin/true' >> ..."
CTRL_CHECK_FN["1.1.1"]    = "check_1_1_1"
REGISTERED_IDS[]           = ("1.1.1" "1.1.2" "1.1.3" ...)
```

### Memorizzazione dei risultati (popolata da record_result)

```
RESULT_STATUS["1.1.1"]    = "PASS"
RESULT_EVIDENCE["1.1.1"]  = "cramfs disabled via /etc/modprobe.d/cramfs.conf"
RESULT_ORDER[]            = ("1.1.1" "1.1.2" ...)
TOTAL_PASS                = 72
TOTAL_FAIL                = 18
TOTAL_WARN                = 6
TOTAL_SKIP                = 8
```

### Aggregazione dei punteggi (popolata da compute_scores)

```
SECTION_PASS["Initial Setup"]    = 16
SECTION_FAIL["Initial Setup"]    = 2
SCORE_BY_SECTION["Initial Setup"] = "88.9"
SCORE_OVERALL                    = "80.0"
SCORE_LEVEL1                     = "82.5"
SCORE_LEVEL2                     = "75.0"
```

---

## Architettura dei test

### Test basati su fixture

La test suite esegue i controlli su due filesystem fittizi:

```
testdata/
├── fixtures/          # Hardened system (most checks PASS)
│   ├── etc/
│   │   ├── modprobe.d/    # cramfs.conf, dccp.conf, etc.
│   │   ├── ssh/sshd_config
│   │   ├── pam.d/
│   │   ├── audit/rules.d/cis.rules
│   │   ├── sysctl.conf
│   │   ├── fstab
│   │   ├── passwd, shadow, group, gshadow
│   │   └── ...
│   └── proc/sys/          # Simulated /proc values
│       ├── kernel/randomize_va_space  (contains "2")
│       ├── net/ipv4/ip_forward        (contains "0")
│       └── ...
│
└── fixtures_fail/     # Unhardened system (most checks FAIL)
    ├── etc/
    │   ├── modprobe.d/    # empty
    │   ├── ssh/sshd_config (PermitRootLogin yes, etc.)
    │   └── ...
    └── proc/sys/          # Insecure values
        ├── net/ipv4/ip_forward  (contains "1")
        └── ...
```

Ogni parametro `/proc/sys/` è un file di testo semplice che contiene il valore atteso. La funzione `get_sysctl` legge il file in `${SYSROOT}/proc/sys/net/ipv4/ip_forward` invece di chiamare `sysctl -n net.ipv4.ip_forward`. Questo permette alle test fixture di simulare i parametri del kernel senza girare su un kernel reale.

### Framework di test

Il test runner è un framework personalizzato in `test_helpers.sh` con tre funzioni di asserzione:

```bash
assert_status "1.1.1" "$STATUS_PASS"
assert_evidence_contains "1.1.1" "cramfs disabled"
assert_json_valid "$json_output"
```

I test seguono uno schema coerente:

```bash
test_cramfs_disabled_pass() {
    CURRENT_TEST="test_cramfs_disabled_pass"
    setup_test "${PROJECT_DIR}/testdata/fixtures"
    check_1_1_1
    assert_status "1.1.1" "$STATUS_PASS"
}

test_cramfs_disabled_fail() {
    CURRENT_TEST="test_cramfs_disabled_fail"
    setup_test "${PROJECT_DIR}/testdata/fixtures_fail"
    check_1_1_1
    assert_status "1.1.1" "$STATUS_FAIL"
}
```

Ogni test reimposta lo stato dei risultati tramite `setup_test`, imposta `SYSROOT` sulla directory di fixture appropriata, chiama la funzione di controllo e verifica lo stato atteso.

---

## Architettura di sicurezza

### Threat model

**Da cosa ci proteggiamo:**
1. Sistemi Linux configurati male che vanno in produzione senza hardening
2. Drift di configurazione, in cui l'hardening di un sistema regredisce nel tempo (rilevato tramite il confronto con la baseline)
3. Lacune di conformità, in cui un'organizzazione non sa quali controlli CIS supera o fallisce

**Da cosa NON ci proteggiamo:**
- Sfruttamento a runtime dei servizi (questo è un tool di audit, non un IDS)
- Attacchi attivi in corso (cisaudit è un'istantanea in un momento preciso, non un monitoraggio continuo)
- Falsificazione dei risultati (qualcuno potrebbe modificare i file di fixture per simulare un audit superato)

### Difese nel tool stesso

- `set -euo pipefail` in ogni script: fallisce su variabili non definite, errori nelle pipe ed errori non gestiti
- `json_escape` e `html_escape` prevengono l'injection nei report JSON e HTML
- `SYSROOT` ha come valore predefinito `/`, quindi il tool non può sottoporre accidentalmente ad audit una test fixture quando viene eseguito come root senza `-t`
- Il tool non modifica mai il sistema. È di sola lettura per progettazione. Tutte le funzioni di controllo usano `grep`, `cat`, `stat` e `sysctl` per ispezionare senza cambiare lo stato

---

## Decisioni di progettazione

### Perché Bash?

L'audit CIS deve girare su sistemi minimali in cui Python, Ruby o Go potrebbero non essere installati. Bash 4+ è presente in ogni distribuzione Linux moderna. Il tool non ha alcuna dipendenza esterna oltre alle utility GNU standard (grep, awk, sed, stat, date). Questo significa che può girare su un server appena predisposto prima che sia avvenuta qualsiasi gestione della configurazione o installazione di pacchetti.

Il compromesso è che gli array associativi di Bash sono più lenti delle hash table nei linguaggi compilati, e la manipolazione delle stringhe è prolissa. Per 104 controlli, la differenza di prestazioni è trascurabile. L'audit si completa in meno di 2 secondi sui sistemi tipici.

### Perché non usare tool esistenti?

Tool come OpenSCAP e Lynis forniscono funzionalità simili. Questo progetto esiste come risorsa didattica che dimostra come funziona l'audit di conformità partendo dai principi fondamentali. OpenSCAP usa definizioni XML XCCDF/OVAL difficili da leggere. Lynis è uno script monolitico di oltre 7000 righe. cisaudit è intenzionalmente modulare e leggibile, così che ogni decisione di progettazione sia visibile.

### Perché array associativi invece di JSON/SQLite?

Gli array associativi di Bash mantengono il tool privo di dipendenze. Usare `jq` per la manipolazione JSON o `sqlite3` per la memorizzazione aggiungerebbe dipendenze esterne. Il compromesso è che il modello di dati è implicito (gli array devono essere dichiarati e popolati nell'ordine giusto) anziché imposto da uno schema. Per 104 controlli, la semplicità ne vale la pena.

---

## Riferimento ai file principali

Mappa rapida di dove trovare le cose:

- `src/cisaudit.sh` - Entry point della CLI, parsing degli argomenti, orchestrazione
- `src/lib/constants.sh` - Tutte le costanti (versione, colori, codici di stato, nomi delle sezioni)
- `src/lib/registry.sh` - Le funzioni `register_control()` e `record_result()`
- `src/lib/engine.sh` - Logica di calcolo dei punteggi
- `src/lib/utils.sh` - Helper per l'ispezione del sistema (`get_sysctl`, `read_file`, `file_exists`)
- `src/controls/registry_data.sh` - Tutte le 104 definizioni dei controlli
- `src/checks/01_initial_setup.sh` - Controlli su filesystem e bootloader
- `src/checks/03_network.sh` - Controlli sui parametri di rete e sul firewall
- `src/lib/report_terminal.sh` - Report nel terminale con ASCII art
- `src/lib/report_html.sh` - Report HTML standalone con CSS
- `tests/test_helpers.sh` - Funzioni di asserzione per i test

---

## Prossimi passi

Ora che hai compreso l'architettura:
1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per una spiegazione del codice di ciascun modulo
2. Prova a eseguire `cisaudit -t testdata/fixtures -f json | python3 -m json.tool` e a ricondurre l'output al codice
