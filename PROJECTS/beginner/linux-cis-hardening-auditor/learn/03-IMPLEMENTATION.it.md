<!-- © AngelaMos | 2026 | 03-IMPLEMENTATION.md -->

# Guida all'implementazione

Questo documento ripercorre il codice effettivo. Tracceremo come vengono registrati i controlli, come le funzioni di controllo ispezionano il sistema, come vengono calcolati i punteggi e come vengono generati i report. Gli snippet di codice provengono direttamente dai file sorgente del progetto.

---

## Panoramica della struttura dei file

```
src/
├── cisaudit.sh              # Entry point and CLI orchestration
├── lib/
│   ├── constants.sh         # All constants and color codes
│   ├── utils.sh             # System inspection helpers
│   ├── registry.sh          # Control registration and result storage
│   ├── engine.sh            # Score aggregation
│   ├── report_terminal.sh   # Terminal output with progress bars
│   ├── report_json.sh       # Machine-readable JSON
│   ├── report_html.sh       # Standalone HTML with embedded CSS
│   └── baseline.sh          # Save/load/diff baselines
├── controls/
│   └── registry_data.sh     # 104 control definitions
└── checks/
    ├── 01_initial_setup.sh  # 20 checks for Section 1
    ├── 02_services.sh       # 16 checks for Section 2
    ├── 03_network.sh        # 20 checks for Section 3
    ├── 04_logging.sh        # 18 checks for Section 4
    ├── 05_access.sh         # 14 checks for Section 5 (cron, SSH)
    ├── 05_access_password.sh # 4 checks for Section 5 (passwords, PAM)
    └── 06_maintenance.sh    # 10 checks for Section 6
```

---

## Costruire il registry dei controlli

### La funzione register_control

Ogni controllo CIS nasce come una chiamata a `register_control` in `registry_data.sh`. La funzione in `registry.sh` memorizza i metadati del controllo in array associativi e ricava il nome della funzione di controllo dall'ID del controllo:

```bash
register_control() {
    local id="$1"
    local section="$2"
    local title="$3"
    local level="$4"
    local scored="$5"
    local description="$6"
    local remediation="$7"

    CTRL_TITLE["$id"]="$title"
    CTRL_SECTION["$id"]="$section"
    CTRL_LEVEL["$id"]="$level"
    CTRL_SCORED["$id"]="$scored"
    CTRL_DESCRIPTION["$id"]="$description"
    CTRL_REMEDIATION["$id"]="$remediation"

    local fn_name="check_${id//\./_}"
    CTRL_CHECK_FN["$id"]="$fn_name"
    REGISTERED_IDS+=("$id")
}
```

L'espressione `${id//\./_}` sostituisce ogni punto con un underscore. L'ID di controllo `"1.1.1"` diventa il nome di funzione `"check_1_1_1"`. L'ID di controllo `"5.2.14"` diventa `"check_5_2_14"`. Questa convenzione di denominazione è imposta dal sistema di registrazione, non dalla documentazione. Se una funzione di controllo non esiste, il motore la salta con un'evidenza che spiega il perché.

### Una chiamata di registrazione

In `registry_data.sh`, ogni definizione di controllo appare così:

```bash
register_control "1.2.1" \
    "Initial Setup" \
    "Ensure /tmp is a separate partition" \
    "1" \
    "yes" \
    "The /tmp directory is a world-writable location used for temporary file storage..." \
    "echo 'tmpfs /tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime 0 0' >> /etc/fstab && mount -o remount /tmp"
```

I sette argomenti posizionali sono: ID, sezione, titolo, livello, scored, descrizione, remediation. La descrizione viene memorizzata ma non mostrata nei report nel terminale (solo in JSON e HTML). La stringa di remediation viene mostrata all'utente quando un controllo fallisce, così sa esattamente quale comando eseguire.

### La funzione record_result

Dopo che una funzione di controllo ha ispezionato il sistema, registra ciò che ha trovato:

```bash
record_result() {
    local id="$1"
    local status="$2"
    local evidence="$3"

    RESULT_STATUS["$id"]="$status"
    RESULT_EVIDENCE["$id"]="$evidence"
    RESULT_ORDER+=("$id")

    case "$status" in
        "$STATUS_PASS") ((TOTAL_PASS++)) || true ;;
        "$STATUS_FAIL") ((TOTAL_FAIL++)) || true ;;
        "$STATUS_WARN") ((TOTAL_WARN++)) || true ;;
        "$STATUS_SKIP") ((TOTAL_SKIP++)) || true ;;
    esac
}
```

Il `|| true` dopo l'incremento è necessario perché `((TOTAL_PASS++))` restituisce exit code 1 quando TOTAL_PASS valeva 0 (perché `0` è falsy in contesto aritmetico), e `set -e` terminerebbe lo script. È un tranello comune di Bash che uccide silenziosamente gli script che usano la strict mode.

---

## Scrivere le funzioni di controllo

### Pattern 1: helper condiviso con wrapper per ogni controllo

Molti controlli nella stessa sezione seguono uno schema di verifica identico con parametri diversi. Invece di duplicare la logica, una funzione helper privata fa il lavoro e dei wrapper sottili la chiamano.

I controlli di disabilitazione dei moduli dei filesystem seguono tutti questo schema. L'helper `_check_module_disabled` in `01_initial_setup.sh` gestisce l'intera logica:

```bash
_check_module_disabled() {
    local id="$1"
    local module="$2"

    local status="$STATUS_PASS"
    local evidence=""
    local modprobe_conf="${SYSROOT}/etc/modprobe.d/${module}.conf"

    if run_cmd lsmod | grep -q "^${module} "; then
        status="$STATUS_FAIL"
        evidence="${module} module is currently loaded"
    elif [[ -f "$modprobe_conf" ]] && grep -q "install ${module} /bin/true\|install ${module} /bin/false" "$modprobe_conf"; then
        evidence="${module} disabled via ${modprobe_conf}"
    else
        local found_disabled="false"
        for conf in "${SYSROOT}"/etc/modprobe.d/*.conf; do
            [[ -f "$conf" ]] || continue
            if grep -q "install ${module} /bin/true\|install ${module} /bin/false\|blacklist ${module}" "$conf"; then
                found_disabled="true"
                evidence="${module} disabled via ${conf}"
                break
            fi
        done
        if [[ "$found_disabled" == "false" ]]; then
            status="$STATUS_FAIL"
            evidence="${module} is not disabled"
        fi
    fi

    record_result "$id" "$status" "$evidence"
}
```

La funzione verifica tre cose in ordine: (1) se il modulo è attualmente caricato in memoria, (2) se esiste un file di configurazione modprobe dedicato, (3) se un qualsiasi file di configurazione modprobe contiene una direttiva di disabilitazione o di blacklist per questo modulo. Se nessuna di queste verifiche ha successo, il controllo fallisce.

Ogni controllo avvolge l'helper in una sola riga:

```bash
check_1_1_1() { _check_module_disabled "1.1.1" "cramfs"; }
check_1_1_2() { _check_module_disabled "1.1.2" "freevxfs"; }
check_1_1_3() { _check_module_disabled "1.1.3" "jffs2"; }
```

Lo stesso schema compare in tutta la codebase. I controlli SSH usano `_check_ssh_value` e `_check_ssh_max_int`. I controlli sulle policy delle password usano `_check_login_defs_value`. I controlli sui permessi dei file usano `_check_file_permissions`.

### Pattern 2: controllo complesso autonomo

Alcuni controlli hanno una logica unica che non si generalizza. Il controllo MTA solo locale (`check_2_2_15`) ispeziona più mail transfer agent in ordine di priorità:

```bash
check_2_2_15() {
    local id="2.2.15"
    local status="$STATUS_PASS"
    local evidence=""

    local listening_external=""

    if run_cmd ss -lntp | grep -qE ':25\s' 2>/dev/null; then
        local listeners
        listeners=$(run_cmd ss -lntp | grep -E ':25\s' 2>/dev/null) || true

        if echo "$listeners" | grep -qvE '127\.0\.0\.1:25|::1:25|\[::1\]:25|\*:25'; then
            local bound_addrs
            bound_addrs=$(echo "$listeners" | awk '{print $4}')
            if echo "$bound_addrs" | grep -qvE '^127\.0\.0\.1:|^\[::1\]:|^::1:'; then
                listening_external="true"
            fi
        fi
    fi
    # ... continues with Postfix, Exim, Sendmail checks ...
}
```

Questa funzione verifica prima se qualcosa è in ascolto sulla porta 25 su un'interfaccia non loopback (il test definitivo). Poi ripiega sulla lettura dell'impostazione `inet_interfaces` di Postfix, poi su `dc_local_interfaces` di Exim, poi verifica la presenza di Sendmail. L'approccio a strati gestisce diverse configurazioni di MTA senza fare ipotesi hardcoded su quale MTA sia installato.

### Pattern 3: verifica delle regole di audit

La funzione `_check_audit_rules` in `04_logging.sh` prende un elenco di pattern e verifica che ciascuno compaia in almeno un file di regole di audit:

```bash
_check_audit_rules() {
    local id="$1"
    shift
    local description="$1"
    shift
    local search_patterns=("$@")

    local status="$STATUS_PASS"
    local evidence=""
    local rules_dir="${SYSROOT}/etc/audit/rules.d"
    local audit_rules="${SYSROOT}/etc/audit/audit.rules"
    local missing=()

    for pattern in "${search_patterns[@]}"; do
        local found="false"
        if [[ -d "$rules_dir" ]]; then
            for rule_file in "$rules_dir"/*.rules; do
                [[ -f "$rule_file" ]] || continue
                if grep -q "$pattern" "$rule_file"; then
                    found="true"
                    break
                fi
            done
        fi
        if [[ "$found" == "false" && -f "$audit_rules" ]]; then
            if grep -q "$pattern" "$audit_rules"; then
                found="true"
            fi
        fi
        if [[ "$found" == "false" ]]; then
            missing+=("$pattern")
        fi
    done

    if [[ ${#missing[@]} -gt 0 ]]; then
        status="$STATUS_FAIL"
        evidence="Missing audit rules for: ${missing[*]}"
    else
        evidence="${description} audit rules are configured"
    fi

    record_result "$id" "$status" "$evidence"
}
```

I chiamanti passano elenchi di pattern di lunghezza variabile:

```bash
check_4_1_5() {
    _check_audit_rules "4.1.5" "Time change" \
        "adjtimex" "settimeofday" "clock_settime" "/etc/localtime"
}
```

Se un qualsiasi pattern manca da tutti i file di regole, il controllo fallisce con un'evidenza che elenca esattamente quali pattern non sono stati trovati. Questo fornisce all'utente informazioni concrete su quali regole di audit aggiungere.

---

## L'astrazione SYSROOT

### Come funziona

La variabile `SYSROOT` ha come valore predefinito `"/"`. Quando si esegue in modalità test (`-t testdata/fixtures`), viene impostata sul percorso della directory di fixture. Ogni funzione di ispezione del sistema antepone `SYSROOT` ai percorsi dei file:

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
```

La funzione `get_sysctl` è il punto in cui la cosa si fa interessante. I valori sysctl vengono normalmente letti con `sysctl -n net.ipv4.ip_forward`, ma questo richiede un kernel in esecuzione. In modalità test, la funzione traduce il nome puntato del parametro sysctl in un percorso `/proc/sys/`:

```bash
get_sysctl() {
    local param="$1"
    local proc_path="${SYSROOT}/proc/sys/${param//\.//}"
    if [[ -f "$proc_path" ]]; then
        cat "$proc_path"
        return 0
    fi

    if run_cmd sysctl -n "$param"; then
        return 0
    fi

    return 1
}
```

L'espressione `${param//\.//}` sostituisce i punti con gli slash. Quindi `net.ipv4.ip_forward` diventa `net/ipv4/ip_forward`, e il percorso completo diventa `testdata/fixtures/proc/sys/net/ipv4/ip_forward`. Quel file contiene `0` (il valore sicuro atteso).

La funzione `run_cmd` blocca esplicitamente l'esecuzione di comandi in modalità test:

```bash
run_cmd() {
    if [[ "$SYSROOT" != "/" ]]; then
        return 1
    fi
    "$@" 2>/dev/null
}
```

Quando `SYSROOT` non è `/`, `run_cmd` restituisce sempre un fallimento. Questo significa che i controlli che si basano su `systemctl`, `dpkg-query`, `lsmod` o `iptables` verranno saltati in modalità test, perché quei comandi non possono essere eseguiti su una directory di fixture. Solo i controlli basati su file producono risultati.

---

## Calcolo dei punteggi

La funzione `compute_scores` in `engine.sh` aggrega i risultati per sezione e per livello:

```bash
compute_scores() {
    local section
    for section in "${SECTION_ORDER[@]}"; do
        SECTION_PASS["$section"]=0
        SECTION_FAIL["$section"]=0
        SECTION_WARN["$section"]=0
        SECTION_SKIP["$section"]=0
    done

    local id
    for id in "${RESULT_ORDER[@]}"; do
        section="${CTRL_SECTION[$id]}"
        case "${RESULT_STATUS[$id]}" in
            "$STATUS_PASS") ((SECTION_PASS["$section"]++)) || true ;;
            "$STATUS_FAIL") ((SECTION_FAIL["$section"]++)) || true ;;
            "$STATUS_WARN") ((SECTION_WARN["$section"]++)) || true ;;
            "$STATUS_SKIP") ((SECTION_SKIP["$section"]++)) || true ;;
        esac
    done
    # ...
}
```

I punteggi sono calcolati come `(pass / (pass + fail)) * 100`. I risultati WARN e SKIP sono esclusi dalla percentuale perché le linee guida di punteggio CIS contano solo i controlli definitivamente superati o falliti. Un controllo che è stato saltato (perché richiede un sistema live) non dovrebbe penalizzare il punteggio.

La chiamata ad `awk` per il calcolo della percentuale aggira la limitazione di Bash alla sola aritmetica intera:

```bash
SCORE_OVERALL=$(awk "BEGIN { printf \"%.1f\", ($TOTAL_PASS / $scored_total) * 100 }")
```

---

## Generazione dei report

### Report nel terminale

Il reporter per il terminale in `report_terminal.sh` produce un output ASCII strutturato con sezioni colorate. La funzione principale concatena quattro passi di rendering:

```bash
emit_terminal_report() {
    _print_banner
    _print_summary_cards
    _print_section_table
    _print_detail_results
    _print_footer
}
```

La funzione della barra di avanzamento `_progress_bar` disegna una barra larga 20 caratteri usando caratteri a blocchi:

```bash
_progress_bar() {
    local percentage="$1"
    local width=20
    local filled
    local empty

    if [[ "$percentage" == "N/A" ]]; then
        printf '%s' "░░░░░░░░░░░░░░░░░░░░"
        return
    fi

    filled=$(awk "BEGIN { printf \"%d\", ($percentage / 100) * $width + 0.5 }")
    empty=$((width - filled))

    local bar=""
    local i
    for (( i = 0; i < filled; i++ )); do
        bar+="█"
    done
    for (( i = 0; i < empty; i++ )); do
        bar+="░"
    done
    printf '%s' "$bar"
}
```

La sezione dei risultati dettagliati mostra lo stato di ogni controllo con evidenza e remediation per i controlli falliti:

```bash
if [[ "$status" == "$STATUS_FAIL" ]]; then
    local evidence="${RESULT_EVIDENCE[$id]:-}"
    local remediation="${CTRL_REMEDIATION[$id]:-}"

    if [[ -n "$evidence" ]]; then
        printf '  %b             Evidence: %s%b\n' "$DIM" "$evidence" "$RESET"
    fi
    if [[ -n "$remediation" ]]; then
        printf '  %b             Fix: %s%b\n' "$YELLOW" "$remediation" "$RESET"
    fi
fi
```

### Report JSON

Il reporter JSON in `report_json.sh` usa `printf` per costruire JSON valido senza `jq`. La funzione `json_escape` gestisce i cinque caratteri che devono essere sottoposti a escape nelle stringhe JSON:

```bash
json_escape() {
    local s="$1"
    s="${s//\\/\\\\}"
    s="${s//\"/\\\"}"
    s="${s//$'\n'/\\n}"
    s="${s//$'\t'/\\t}"
    s="${s//$'\r'/\\r}"
    printf '%s' "$s"
}
```

Il report include i metadati (versione, benchmark, timestamp, hostname, sistema operativo), un oggetto di riepilogo, un array con i punteggi delle sezioni e un array con tutti i risultati dei controlli. Ogni risultato di controllo include ID, sezione, titolo, livello, stato scored, stato del risultato, evidenza e remediation. Questo rende l'output JSON utilizzabile per l'ingestione in SIEM, dashboard o piattaforme di compliance.

### Report HTML

Il reporter HTML in `report_html.sh` genera un documento HTML standalone con CSS incorporato. L'intero report è un unico file autonomo senza dipendenze esterne, il che significa che può essere inviato per email, archiviato o visualizzato offline.

Il CSS usa uno schema di colori scuro con custom property CSS per i colori del tema. I controlli falliti sono resi con `<details open>` in modo da essere espansi per impostazione predefinita, mentre i controlli superati sono compressi. Il report include breakpoint responsive per la visualizzazione su mobile e un foglio di stile di stampa che passa a un tema chiaro.

---

## Confronto con la baseline

Il modulo di baseline in `baseline.sh` permette agli utenti di salvare un'istantanea JSON e confrontare con essa le esecuzioni future.

Il salvataggio è semplice: chiama `emit_json_report` e scrive l'output in un file.

Il caricamento e il confronto usano il matching con regex in puro Bash per estrarre gli ID dei controlli e gli stati dal JSON senza richiedere `jq`:

```bash
load_baseline() {
    local file="$1"
    BASELINE_STATUS=()

    local current_id=""
    local id_pattern='"id":[[:space:]]*"([^"]+)"'
    local status_pattern='"status":[[:space:]]*"([^"]+)"'
    while IFS= read -r line; do
        if [[ "$line" =~ $id_pattern ]]; then
            current_id="${BASH_REMATCH[1]}"
        fi
        if [[ -n "$current_id" && "$line" =~ $status_pattern ]]; then
            BASELINE_STATUS["$current_id"]="${BASH_REMATCH[1]}"
            current_id=""
        fi
    done < "$file"
}
```

L'output del confronto classifica le modifiche come regressioni (da PASS a FAIL), miglioramenti (da FAIL a PASS), invariati, nuovi controlli e controlli rimossi. Le regressioni sono evidenziate in rosso e generano un avviso.

---

## Il framework di test

### Test runner

Il test runner in `test_runner.sh` carica tutti i moduli del progetto, poi individua i file di test con il glob `test_*.sh` (escludendo `test_helpers.sh` e `test_runner.sh`). Per ogni file, lo carica, individua le funzioni che corrispondono a `^test_`, chiama ciascuna di esse e poi le rimuove con unset per evitare collisioni con il file successivo:

```bash
run_test_file() {
    local test_file="$1"
    source "$test_file"

    local test_fns
    test_fns=$(declare -F | awk '{print $3}' | grep '^test_')

    local fn
    for fn in $test_fns; do
        echo "  Running: ${fn}"
        "$fn"
    done

    for fn in $test_fns; do
        unset -f "$fn"
    done
}
```

### Helper di asserzione

La funzione `setup_test` reimposta tutto lo stato dei risultati e imposta `SYSROOT` su una directory di fixture:

```bash
setup_test() {
    local fixtures_dir="$1"
    reset_results
    SYSROOT="$fixtures_dir"
    DETECTED_ID="debian"
    DETECTED_VERSION="12"
}
```

`assert_status` confronta lo stato del risultato registrato per un ID di controllo con il valore atteso:

```bash
assert_status() {
    local id="$1" expected="$2"
    local actual="${RESULT_STATUS[$id]:-UNSET}"
    ((TEST_TOTAL++)) || true
    if [[ "$actual" == "$expected" ]]; then
        ((TEST_PASS++)) || true
    else
        ((TEST_FAIL++)) || true
        echo "  FAIL: ${CURRENT_TEST} — ${id}: expected ${expected}, got ${actual}" >&2
    fi
}
```

`assert_evidence_contains` verifica che la stringa di evidenza contenga una sottostringa attesa. È più robusto del confronto esatto perché le stringhe di evidenza includono spesso dettagli variabili come percorsi di file o valori numerici.

---

## Aggiungere un nuovo controllo

Per aggiungere un nuovo controllo CIS, servono esattamente due modifiche:

**Passo 1: registra il controllo** in `controls/registry_data.sh`:

```bash
register_control "1.6.1" \
    "Initial Setup" \
    "Ensure SELinux or AppArmor is installed" \
    "1" \
    "yes" \
    "Mandatory access control frameworks restrict process capabilities beyond DAC permissions" \
    "apt-get install apparmor apparmor-utils"
```

**Passo 2: scrivi la funzione di controllo** nel file `checks/0X_section.sh` appropriato:

```bash
check_1_6_1() {
    local id="1.6.1"
    local status="$STATUS_PASS"
    local evidence=""

    if package_is_installed "apparmor"; then
        evidence="AppArmor is installed"
    elif package_is_installed "selinux-basics"; then
        evidence="SELinux is installed"
    elif file_exists "/usr/sbin/apparmor_parser"; then
        evidence="AppArmor binary found"
    else
        status="$STATUS_FAIL"
        evidence="Neither AppArmor nor SELinux is installed"
    fi

    record_result "$id" "$status" "$evidence"
}
```

**Passo 3: aggiungi i test** nel file `tests/test_0X_section.sh` appropriato:

```bash
test_selinux_apparmor_pass() {
    CURRENT_TEST="test_selinux_apparmor_pass"
    setup_test "${PROJECT_DIR}/testdata/fixtures"
    # add /usr/sbin/apparmor_parser to fixtures first
    check_1_6_1
    assert_status "1.6.1" "$STATUS_PASS"
}
```

Il motore, i reporter, il modulo di baseline e il motore di punteggio rilevano automaticamente il nuovo controllo. Nessun altro file deve essere modificato.

---

## Errori comuni di implementazione

### Errore 1: incremento aritmetico sotto set -e

**Sintomo:** lo script termina silenziosamente quando incrementa un contatore che vale attualmente 0.

**Causa:**

```bash
((count++))
```

Quando `count` vale 0, `((count++))` viene valutato come 0 (il valore prima dell'incremento), che è falsy nell'aritmetica di Bash. Sotto `set -e`, questo exit code diverso da zero termina lo script.

**Soluzione:**

```bash
((count++)) || true
```

Il `|| true` assorbe l'exit code falso. Questo schema compare in tutta la codebase ovunque venga incrementato un contatore.

### Errore 2: fallimento di grep sotto set -e

**Sintomo:** lo script termina quando grep non trova corrispondenze.

**Causa:** `grep` restituisce exit code 1 quando non trova corrispondenze. Sotto `set -e`, questo viene trattato come un errore fatale.

**Soluzione:**

```bash
local result
result=$(grep "pattern" "$file") || true
```

Il `|| true` garantisce che lo script continui quando grep non trova nulla. La variabile `result` sarà vuota e la funzione di controllo dovrebbe gestire quel caso.

### Errore 3: virgolette mancanti attorno alla variabile nella chiave di un array

**Sintomo:** le ricerche negli array associativi restituiscono un valore vuoto per chiavi che contengono caratteri speciali.

**Causa:**

```bash
RESULT_STATUS[$id]="PASS"
```

Se `$id` contiene caratteri che Bash potrebbe interpretare (spazi, caratteri speciali), la ricerca fallisce.

**Soluzione:**

```bash
RESULT_STATUS["$id"]="PASS"
```

Metti sempre tra virgolette le chiavi degli array associativi. La codebase mette coerentemente tra virgolette tutti gli accessi alle chiavi degli array.

---

## Suggerimenti per il debug

### Un controllo restituisce uno stato inatteso

1. Esegui il controllo specifico in isolamento su una fixture:
   ```bash
   source src/lib/constants.sh src/lib/utils.sh src/lib/registry.sh
   source src/controls/registry_data.sh
   source src/checks/01_initial_setup.sh
   SYSROOT="testdata/fixtures" QUIET="true"
   check_1_1_1
   echo "${RESULT_STATUS[1.1.1]}: ${RESULT_EVIDENCE[1.1.1]}"
   ```
2. Verifica se il file di fixture esiste e contiene il contenuto atteso
3. Verifica che la funzione di controllo stia guardando il percorso di file giusto sotto `SYSROOT`

### Il report JSON non è valido

1. Passalo in pipe a `python3 -m json.tool` per vedere dove fallisce il parser
2. Verifica se qualche stringa di evidenza contiene virgolette o a capo senza escape
3. La funzione `json_escape` gestisce `\`, `"`, `\n`, `\t` e `\r`. Se un nuovo carattere causa problemi, aggiungilo alla funzione

### I test passano in locale ma falliscono in CI

1. Controlla la versione di Bash: `bash --version`. Alcuni ambienti CI includono Bash 4.x mentre un controllo si basa su funzionalità di Bash 5.x
2. Verifica che la working directory sia impostata correttamente. `SCRIPT_DIR` e `PROJECT_DIR` nel test runner dipendono da `${BASH_SOURCE[0]}`
3. Assicurati che i file di fixture abbiano i terminatori di riga corretti (LF, non CRLF)

---

## Prossimi passi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.md](./04-CHALLENGES.it.md) contiene idee di estensione che vanno da nuovi formati di output al supporto multi-distro
2. **Modifica un controllo** - Scegli un controllo, cambia il valore atteso in una fixture e verifica che il controllo fallisca come previsto
3. **Aggiungi un controllo** - Segui il processo in tre passi descritto sopra per aggiungere un controllo del benchmark CIS non ancora implementato
