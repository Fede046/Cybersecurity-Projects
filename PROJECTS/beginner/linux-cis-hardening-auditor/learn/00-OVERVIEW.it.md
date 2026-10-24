<!-- © AngelaMos | 2026 | 00-OVERVIEW.md -->

# Linux CIS Hardening Auditor

## Che cos'è

Un tool da riga di comando scritto interamente in Bash che esegue l'audit dei sistemi Linux rispetto a 104 controlli del CIS Debian Linux 12 Benchmark. Verifica l'hardening del filesystem, la minimizzazione dei servizi, i parametri di rete, la configurazione del logging, i controlli di accesso e la manutenzione del sistema. Produce report di conformità con punteggio in formato terminale, JSON o HTML, e può confrontare i risultati con una baseline salvata per tracciare il drift nel tempo.

## Perché è importante

Nel dicembre 2019, un server Citrix configurato male (CVE-2019-19781) ha dato agli attaccanti un punto d'appoggio in centinaia di organizzazioni perché i controlli CIS di base non erano applicati. La patch esisteva. L'advisory esisteva. Ma i sistemi non erano sottoposti ad hardening, e nessun audit automatico ha individuato l'esposizione prima che venisse sfruttata. Agli attaccanti non serviva uno zero-day. Serviva un solo server in cui l'IP forwarding fosse abilitato, il login SSH come root fosse permesso e l'audit logging fosse disattivato.

La violazione di Capital One del 2019 ha esposto 100 milioni di richieste di carte di credito. La causa principale era un WAF configurato male su un'istanza EC2 con permessi IAM eccessivi. Ma l'indagine forense ha rivelato che i controlli di base per l'hardening del sistema, esattamente il tipo di controlli definiti dai benchmark CIS, erano assenti sui server compromessi. Non c'era alcun monitoraggio dell'integrità dei file, nessuna opzione di mount restrittiva su /tmp, e il sottosistema di audit non era abilitato.

I benchmark CIS esistono perché i team di sicurezza continuavano a vedere le stesse configurazioni errate causare violazioni. Il Center for Internet Security ha formalizzato queste lezioni in guide di hardening prescrittive che si mappano direttamente su framework normativi come NIST 800-53, PCI DSS, SOC 2 e HIPAA. Se esegui un audit CIS e ottieni l'85%, puoi dire a un auditor esattamente quale 15% non hai affrontato e perché. Senza un benchmark, stai tirando a indovinare.

**Scenari reali in cui si applica:**
- Un analista SOC deve verificare che i server appena predisposti rispettino lo standard di hardening dell'organizzazione prima di andare in produzione
- Un incident responder vuole valutare rapidamente se un host compromesso avesse dei controlli di sicurezza di base attivi al momento della violazione
- Un team di compliance ha bisogno di prove che i server Linux rispettino i benchmark CIS Level 1 per un audit SOC 2 Type II

## Cosa imparerai

Questo progetto ti insegna come i benchmark CIS si traducono da documenti PDF in controlli di conformità automatizzati. Costruendolo da solo, capisci cosa verifica effettivamente ciascun controllo a livello di sistema.

**Concetti di sicurezza:**
- Struttura del CIS Benchmark: come i controlli sono organizzati in sezioni, livelli e categorie scored vs. unscored
- Hardening dei sistemi Linux: quali parametri del kernel, opzioni di mount, stati dei servizi e permessi dei file contano e perché
- Punteggio di conformità: come quantificare la postura di sicurezza di un sistema come percentuale e tracciarla nel tempo
- Defense in depth: come 104 piccoli controlli si sovrappongono a strati per creare un sistema sottoposto ad hardening

**Competenze tecniche:**
- Architettura Bash: costruire un tool CLI modulare con un registry pattern, funzioni di controllo pluggable e più formattatori di output
- Ispezione del sistema: leggere /proc, fare il parsing dei file di configurazione, verificare gli stati dei servizi systemd e interrogare i valori sysctl senza dipendenze esterne
- Test fixture: eseguire i controlli di conformità su alberi di filesystem fittizi in modo che i test funzionino senza accesso root o un sistema live

**Strumenti e tecniche:**
- `sysctl` e `/proc/sys` per leggere i parametri del kernel
- Configurazione di `modprobe` per disabilitare i moduli del kernel
- Regole di `auditd` per tracciare le system call rilevanti per la sicurezza
- Configurazione PAM per le policy delle password e il blocco degli account
- `iptables` per verificare le policy predefinite del firewall

## Prerequisiti

**Conoscenze richieste:**
- Saper leggere e scrivere con disinvoltura script Bash (funzioni, array, condizionali, manipolazione di stringhe)
- Amministrazione Linux di base: sai a cosa servono `/etc/passwd`, `/etc/ssh/sshd_config` e `systemctl`
- Comprensione dei permessi dei file (proprietario, gruppo, mode bit come 644 e 600)

**Strumenti di cui avrai bisogno:**
- Bash 4+ (incluso in ogni distribuzione Linux moderna)
- Il command runner `just` (opzionale ma consigliato per eseguire i task)
- `shellcheck` per il linting (opzionale, proposto durante l'installazione)

**Utile ma non obbligatorio:**
- Familiarità con i documenti PDF dei CIS Benchmark (leggerli insieme al codice rende i controlli più chiari più rapidamente)
- Esperienza con la configurazione di `auditd` e PAM

## Avvio rapido

```bash
cd PROJECTS/beginner/linux-cis-hardening-auditor
./install.sh
sudo cisaudit
```

Output atteso: un report nel terminale che mostra il banner ASCII CIS, un punteggio di conformità complessivo, una tabella di ripartizione per sezione con i conteggi pass/fail/warn/skip e barre di avanzamento, e i risultati dettagliati per ciascuno dei 104 controlli. I controlli falliti includono le evidenze che spiegano cosa è stato trovato e un comando di remediation.

Per eseguirlo senza accesso root, usa le test fixture:

```bash
cisaudit -t testdata/fixtures -f terminal
```

Questo esegue l'audit completo sul filesystem fittizio in `testdata/fixtures/` e produce lo stesso formato di report. Ogni controllo che può essere valutato dai file di configurazione produrrà un risultato.

## Struttura del progetto

```
linux-cis-hardening-auditor/
├── src/
│   ├── cisaudit.sh              # Entry point: CLI parsing, orchestration
│   ├── lib/
│   │   ├── constants.sh         # Version, colors, status codes, section names
│   │   ├── utils.sh             # Logging, OS detection, file/sysctl helpers
│   │   ├── registry.sh          # Control registration and result recording
│   │   ├── engine.sh            # Score computation per section and level
│   │   ├── report_terminal.sh   # Terminal report with ASCII art and progress bars
│   │   ├── report_json.sh       # Machine-readable JSON report
│   │   ├── report_html.sh       # Standalone HTML report with dark theme
│   │   └── baseline.sh          # Save and diff baselines for drift detection
│   ├── controls/
│   │   └── registry_data.sh     # All 104 control definitions with metadata
│   └── checks/
│       ├── 01_initial_setup.sh  # Filesystem, bootloader, ASLR, core dumps
│       ├── 02_services.sh       # Unnecessary service detection
│       ├── 03_network.sh        # Kernel network params, firewall, protocols
│       ├── 04_logging.sh        # auditd, rsyslog, audit rules
│       ├── 05_access.sh         # cron, SSH configuration
│       ├── 05_access_password.sh # PAM, password policies, account lockout
│       └── 06_maintenance.sh    # File permissions, duplicate UIDs, legacy entries
├── tests/
│   ├── test_runner.sh           # Custom test framework runner
│   ├── test_helpers.sh          # assert_status, assert_evidence_contains
│   └── test_*.sh                # One test file per section + engine/baseline
├── testdata/
│   ├── fixtures/                # Mock filesystem that passes most controls
│   └── fixtures_fail/           # Mock filesystem designed to fail controls
├── install.sh                   # One-command setup with PATH integration
├── Justfile                     # Command runner for audit, test, lint, baseline
└── learn/                       # You are here
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per imparare i benchmark CIS, i principi di hardening dei sistemi e le violazioni reali che hanno motivato questi controlli
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere come il registry pattern, il motore dei controlli e i formattatori dei report si incastrano tra loro
3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per una spiegazione funzione per funzione dei moduli principali
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee come aggiungere il supporto a RHEL, una modalità di auto-remediation e la generazione di playbook Ansible

## Problemi comuni

**Errore "Bash 4+ required"**
```
[✖] Bash 4+ required (found 3.2.57)
```
Soluzione: macOS include Bash 3.2 per motivi di licenza. Installa Bash 5 tramite `brew install bash` ed eseguilo con `/opt/homebrew/bin/bash src/cisaudit.sh`.

**La maggior parte dei controlli mostra SKIP in modalità test**
Soluzione: i controlli che richiedono l'accesso al sistema live (verificare i servizi in esecuzione con `systemctl`, interrogare le regole di `iptables`, ispezionare i moduli del kernel caricati con `lsmod`) verranno saltati quando usi `-t testdata/fixtures`, perché quei comandi non sono disponibili in modalità test. È previsto. I controlli basati su file produrranno comunque risultati PASS/FAIL.

**Punteggio basso su un sistema reale**
Soluzione: la maggior parte delle installazioni Linux non è sottoposta ad hardening CIS di default. Un'installazione Debian appena fatta ottiene tipicamente il 40-60%. È normale. Usa i comandi di remediation nel report per eseguire l'hardening del sistema, poi esegui di nuovo l'audit.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **Firewall Rule Engine** (intermediate) - costruisce la gestione delle regole iptables/nftables che la Sezione 3.3 del CIS verifica
- **Network Traffic Analyzer** (beginner) - cattura e analizza il traffico che i controlli di rete CIS sono progettati per limitare
