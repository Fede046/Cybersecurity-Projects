<!-- © AngelaMos | 2026 | 04-CHALLENGES.md -->

# Sfide di estensione

Hai costruito il progetto di base. Ora rendilo tuo estendendolo con nuove funzionalità.

Queste sfide sono ordinate per difficoltà. Inizia con quelle più facili per prendere confidenza, poi affronta quelle più difficili quando vuoi approfondire.

---

## Sfide facili

### Sfida 1: report in formato CSV

**Cosa costruire:**
Aggiungi un formato di output `csv` che produca un report adatto ai fogli di calcolo.

**Perché è utile:**
I team di compliance vivono nei fogli di calcolo. Un report CSV può essere importato in Excel, Google Sheets o in una piattaforma GRC (Governance, Risk, Compliance) per il tracciamento e i workflow di approvazione. JSON è ottimo per l'automazione, ma quando un auditor chiede delle evidenze, vuole un foglio di calcolo.

**Cosa imparerai:**
- Come funziona lo strategy pattern dei report
- L'escaping CSV (i campi che contengono virgole o virgolette devono essere racchiusi tra virgolette)
- Aggiungere un nuovo valore di opzione al parser della CLI

**Suggerimenti:**
- Crea `src/lib/report_csv.sh` con una funzione `emit_csv_report`
- Aggiungi `csv` come case in `generate_report` in `cisaudit.sh`
- Riga di intestazione: `ID,Section,Title,Level,Scored,Status,Evidence,Remediation`
- Usa una funzione `csv_escape` che racchiuda tra virgolette doppie i campi che contengono virgole o virgolette doppie, e che raddoppi eventuali virgolette doppie interne

**Verifica che funzioni:**
```bash
cisaudit -t testdata/fixtures -f csv -o report.csv
head -5 report.csv
```

### Sfida 2: riepilogo del conteggio dei controlli

**Cosa costruire:**
Aggiungi un flag `--summary` che stampi un riepilogo di una sola riga invece del report completo.

**Perché è utile:**
Le pipeline CI/CD hanno bisogno di una risposta rapida pass/fail, non di un report di più pagine. Una riga di riepilogo come `cisaudit: 82/104 passed (78.8%) Level1=85.2% Level2=68.0%` è facile da analizzare negli script shell e negli aggregatori di log.

**Cosa imparerai:**
- Aggiungere un nuovo flag CLI a `parse_args`
- Cortocircuitare la generazione del report

**Suggerimenti:**
- Aggiungi `OPT_SUMMARY="false"` alle dichiarazioni globali
- Aggiungi un case `--summary` a `parse_args`
- In `main`, dopo `compute_scores`, controlla `OPT_SUMMARY` e stampa la riga singola invece di chiamare `generate_report`

**Verifica che funzioni:**
```bash
cisaudit -t testdata/fixtures --summary
```

### Sfida 3: attivazione/disattivazione dei colori

**Cosa costruire:**
Aggiungi un flag `--no-color` che disabiliti i codici colore ANSI nell'output del terminale.

**Perché è utile:**
Reindirizzare l'output colorato verso un file o un altro tool produce caratteri spazzatura come `[0;32m`. I sistemi di aggregazione dei log, i pager come `less` (senza `-R`) e i log di CI spesso rendono i codici ANSI come testo letterale. Un flag `--no-color` rende l'output pulito per l'uso fuori dal terminale.

**Cosa imparerai:**
- Come funzionano le costanti di colore ANSI in `constants.sh`
- L'inizializzazione condizionale delle variabili globali

**Suggerimenti:**
- Aggiungi `--no-color` a `parse_args`
- Dopo il parsing, se `--no-color` è stato impostato, ridichiara tutte le variabili di colore come stringhe vuote: `RED=""`, `GREEN=""`, ecc.
- In alternativa, rileva `[[ ! -t 1 ]]` (stdout non è un terminale) e disabilita automaticamente i colori

**Verifica che funzioni:**
```bash
cisaudit -t testdata/fixtures --no-color | cat
cisaudit -t testdata/fixtures -f terminal | less
```

---

## Sfide intermedie

### Sfida 4: supporto a RHEL/CentOS

**Cosa costruire:**
Estendi le funzioni di controllo per gestire le distribuzioni basate su Red Hat (RHEL, CentOS, Rocky Linux, AlmaLinux) oltre a Debian/Ubuntu.

**Perché è utile:**
CIS pubblica benchmark separati per le diverse distribuzioni, ma molti controlli sono identici. Gli ambienti enterprise spesso eseguono sia sistemi Debian sia RHEL. Un unico tool di audit che gestisca entrambi riduce il carico di gestione degli strumenti.

**Cosa imparerai:**
- Come la gestione dei pacchetti differisce tra le distribuzioni (dpkg vs rpm)
- Il rilevamento della distribuzione da `/etc/os-release`
- La logica condizionale basata sulla famiglia del sistema operativo

**Approccio di implementazione:**

1. **Modifica `utils.sh`** per aggiungere funzioni compatibili con RHEL:
   ```bash
   package_is_installed() {
       case "$DETECTED_ID" in
           debian|ubuntu)
               run_cmd dpkg-query -W -f='${Status}' "$1" | grep -q "install ok installed"
               ;;
           rhel|centos|rocky|almalinux)
               run_cmd rpm -q "$1" &>/dev/null
               ;;
       esac
   }
   ```

2. **Aggiorna le funzioni di controllo** dove Debian e RHEL differiscono:
   - I percorsi di configurazione PAM differiscono (`/etc/pam.d/common-password` vs `/etc/pam.d/system-auth`)
   - I nomi dei servizi possono differire (`rsyslog` è lo stesso, ma i nomi dei pacchetti variano)
   - Gli strumenti firewall possono differire (`iptables` vs `firewalld`)

3. **Aggiungi test fixture RHEL** in `testdata/fixtures_rhel/` con file di configurazione in stile RHEL

**Suggerimenti:**
- Inizia facendo in modo che `detect_os` imposti una variabile `DETECTED_FAMILY` (`debian` o `rhel`)
- Modifica prima gli helper per pacchetti e servizi, dato che molti controlli dipendono da essi
- Aggiungi file di fixture e test RHEL in modo incrementale, una sezione alla volta

### Sfida 5: report in Markdown

**Cosa costruire:**
Aggiungi un formato di output `markdown` che produca un documento Markdown pulito, adatto a essere incollato in issue GitHub, pagine Confluence o commenti alle pull request.

**Perché è utile:**
I risultati delle revisioni di sicurezza sono spesso documentati in Markdown. Un report di conformità in formato Markdown può essere sottoposto a commit in un repository, allegato a una pull request o incollato in una pagina wiki per la revisione.

**Cosa imparerai:**
- La generazione di tabelle Markdown
- Come strutturare un formato di report per renderlo leggibile sia in forma grezza sia renderizzata

**Suggerimenti:**
- Crea `src/lib/report_markdown.sh` con una funzione `emit_markdown_report`
- Usa tabelle Markdown per la ripartizione per sezione
- Usa blocchi details comprimibili (`<details><summary>`) per i risultati dei singoli controlli
- I controlli falliti dovrebbero essere espansi per impostazione predefinita, quelli superati compressi

**Verifica che funzioni:**
```bash
cisaudit -t testdata/fixtures -f markdown -o report.md
```

### Sfida 6: esecuzione parallela dei controlli

**Cosa costruire:**
Esegui le funzioni di controllo indipendenti in parallelo usando job in background per ridurre il tempo di audit sui sistemi con molti controlli.

**Perché è utile:**
Alcuni controlli comportano operazioni di rete o scansioni di file che richiedono tempo. Eseguire i controlli indipendenti in parallelo può ridurre il tempo di audit sui sistemi grandi da minuti a secondi.

**Cosa imparerai:**
- I job in background di Bash (`&`)
- `wait` per il completamento dei job
- La gestione dello stato condiviso (gli array dei risultati) con scrittori paralleli
- La raccolta dei risultati basata su file temporanei per evitare race condition

**Suggerimenti:**
- I controlli all'interno di sezioni diverse sono indipendenti e possono essere eseguiti in parallelo
- I controlli all'interno della stessa sezione possono condividere lo stato e dovrebbero essere eseguiti in sequenza
- Scrivi i risultati in file temporanei (uno per controllo) e uniscili dopo che tutti i job sono completati
- Usa `mktemp -d` per la directory temporanea dei risultati e ripuliscila in un trap

**Insidie:**
- Gli array associativi di Bash non possono essere scritti in sicurezza da più job in background
- Dovrai serializzare i risultati in file temporanei e caricarli nel processo padre

---

## Sfide avanzate

### Sfida 7: modalità di auto-remediation

**Cosa costruire:**
Aggiungi un flag `--fix` che applichi i comandi di remediation per i controlli falliti dopo una conferma.

**Perché è utile:**
Passare dal report di audit a un sistema sottoposto ad hardening richiede attualmente di eseguire manualmente ogni comando di remediation. Una modalità di auto-remediation con richieste di conferma trasforma un processo di hardening manuale di 2 ore in una sessione guidata di 10 minuti.

**Cosa imparerai:**
- Le operazioni pericolose e la UX di conferma
- La remediation idempotente (applicare due volte la stessa correzione non dovrebbe rompere nulla)
- La modalità dry-run per visualizzare in anteprima le modifiche

**Passi di implementazione:**

1. **Aggiungi i flag `--fix` e `--fix-dry-run`** al parser della CLI
2. **Dopo aver eseguito i controlli**, itera sui controlli falliti e mostra il comando di remediation
3. **Chiedi conferma all'utente** per ogni controllo: "Apply fix for 1.1.1 (cramfs)? [y/N/a(ll)]"
4. **Esegui la remediation** con gestione degli errori (alcuni comandi richiedono root, alcuni potrebbero fallire)
5. **Riesegui il controllo specifico** dopo aver applicato la correzione per verificare che venga superato
6. **Stampa un riepilogo** di ciò che è stato corretto e di ciò che richiede ancora un intervento manuale

**Insidie:**
- Alcuni comandi di remediation sono distruttivi (modifica della configurazione di GRUB, modifica di fstab). Richiedi sempre una conferma.
- Alcune remediation richiedono un riavvio per avere effetto (parametri del kernel impostati tramite sysctl.conf)
- Non applicare mai correzioni automatiche in modalità non interattiva (input da pipe). Controlla `[[ -t 0 ]]` prima di chiedere conferma.

### Sfida 8: generazione di playbook Ansible

**Cosa costruire:**
Aggiungi un flag `--emit-playbook` che generi un playbook Ansible per correggere tutti i controlli falliti.

**Perché è utile:**
Negli ambienti infrastructure-as-code, la remediation manuale non persiste. La volta successiva in cui il server viene predisposto da un'immagine di base, tutto l'hardening va perso. Un playbook Ansible cattura la remediation come codice che può essere applicato a ogni server, revisionato nelle pull request e versionato in Git.

**Cosa imparerai:**
- La generazione di YAML da dati strutturati
- La scelta dei moduli Ansible (sysctl, lineinfile, service, mount, modprobe)
- I concetti di gestione della configurazione idempotente

**Modifiche architetturali necessarie:**

```
┌──────────────────────────┐
│  New: playbook_emitter   │
│  Reads RESULT_STATUS[]   │
│  Maps remediation to     │
│  Ansible modules         │
│  Outputs YAML playbook   │
└──────────────────────────┘
```

**Passi di implementazione:**

1. **Crea una mappatura** dalle categorie di controlli CIS ai moduli Ansible:
   - Moduli del kernel (1.1.x) → `community.general.modprobe`
   - Opzioni di mount (1.2.x) → `ansible.posix.mount`
   - Parametri sysctl (3.x.x) → `ansible.posix.sysctl`
   - Configurazione SSH (5.2.x) → `ansible.builtin.lineinfile`
   - Rimozione di pacchetti (2.x.x) → `ansible.builtin.apt` con `state: absent`
   - Permessi dei file (6.x.x) → `ansible.builtin.file`

2. **Genera il playbook** con task solo per i controlli falliti
3. **Includi tag** per ogni sezione CIS, così gli operatori possono eseguire `ansible-playbook --tags section3` per applicare solo l'hardening di rete

**Suggerimenti:**
- Usa un heredoc per l'intestazione del playbook (hosts, become, vars)
- Genera un task per ogni controllo fallito
- Includi l'ID del controllo CIS nel nome del task per la tracciabilità

### Sfida 9: profili di configurazione

**Cosa costruire:**
Supporta profili di benchmark personalizzati che sovrascrivano quali controlli sono abilitati, i loro valori attesi e i loro livelli.

**Perché è utile:**
Le organizzazioni raramente applicano il benchmark CIS così com'è. Un database server potrebbe aver bisogno di impostazioni SSH diverse da un web server. Un host Docker non può disabilitare squashfs. I profili permettono ai team di definire "la nostra baseline CIS per i web server" e "la nostra baseline CIS per i database server" come configurazioni separate.

**Cosa imparerai:**
- Il parsing di file di configurazione in Bash
- I pattern di override/ereditarietà
- La composizione dei profili (profilo di base + override specifici per ruolo)

**Approccio di implementazione:**

1. **Definisci un formato di profilo** (configurazione YAML o in stile INI):
   ```ini
   [profile]
   name = web-server
   base = cis-debian-12-l1

   [overrides]
   1.1.6 = skip   # squashfs needed for snap
   1.1.8 = skip   # vfat needed for UEFI
   5.2.5 = skip   # X11 forwarding needed for remote IDE

   [thresholds]
   overall = 80
   section.3 = 90
   ```

2. **Carica il profilo** prima di eseguire i controlli
3. **Applica gli override** in `should_run_check` per saltare o modificare i controlli
4. **Supporta soglie per sezione** nella logica degli exit code

---

## Sfide per esperti

### Sfida 10: scansione multi-host con SSH

**Cosa costruire:**
Aggiungi la possibilità di scansionare host remoti tramite SSH, raccogliendo i risultati da più server e generando un report consolidato.

**Perché è utile:**
Un team di sicurezza che gestisce 200 server non può accedere a ciascuno singolarmente. Uno scanner multi-host esegue l'audit da remoto, raccoglie i risultati JSON e produce una dashboard di conformità a livello di flotta che mostra quali server non sono conformi e su quali controlli specifici.

**Prerequisiti:**
Completa prima la Sfida 4 (supporto a RHEL), dato che gli host remoti potrebbero eseguire distribuzioni diverse.

**Cosa imparerai:**
- L'esecuzione remota di comandi basata su SSH
- La gestione di job remoti in parallelo
- L'aggregazione dei risultati da più fonti
- Il punteggio di conformità a livello di flotta

**Fasi di implementazione:**

**Fase 1: esecuzione remota**
- Accetta un file con l'elenco degli host (`--hosts hosts.txt`)
- Connettiti via SSH a ogni host, copia cisaudit, eseguilo con `-f json` e raccogli l'output
- Gestisci l'autenticazione SSH (solo basata su chiave, mai password)

**Fase 2: aggregazione dei risultati**
- Esegui il parsing dei risultati JSON di ogni host
- Calcola i punteggi per host e una media a livello di flotta
- Identifica i controlli che falliscono su più host (problemi sistemici)

**Fase 3: report consolidato**
- Genera un report della flotta che mostri i punteggi per host in una tabella
- Evidenzia i controlli con il tasso di fallimento più alto nella flotta
- Esporta in HTML con un drill-down a livello di host

**Insidie:**
- Le connessioni SSH possono fallire. Gestisci con eleganza timeout e host irraggiungibili
- Host diversi possono avere versioni di Bash diverse. Distribuisci cisaudit come script autonomo
- Applica rate limiting alle connessioni SSH parallele per evitare di sovraccaricare la rete o di far scattare alert dell'IDS

### Sfida 11: monitoraggio continuo della conformità

**Cosa costruire:**
Un servizio e un timer systemd che eseguono cisaudit secondo una pianificazione, memorizzano i risultati e inviano alert quando la conformità regredisce.

**Perché è utile:**
Gli audit puntuali individuano i problemi ma non prevengono il drift. Un server che oggi supera il CIS può essere configurato male domani da un aggiornamento di pacchetti, da una modifica manuale o da un errore della gestione della configurazione. Il monitoraggio continuo individua il drift entro poche ore invece di aspettare il successivo audit trimestrale.

**Cosa imparerai:**
- I file di unità di servizio e timer di systemd
- La memorizzazione dei risultati e l'analisi delle tendenze
- L'integrazione degli alert (email, webhook Slack, syslog)
- I dati di conformità come serie temporali

**Passi di implementazione:**

1. **Crea un timer systemd** che esegua cisaudit ogni giorno:
   ```ini
   [Timer]
   OnCalendar=daily
   Persistent=true
   ```

2. **Memorizza i risultati** in una directory come `/var/lib/cisaudit/` con file JSON marcati con timestamp

3. **Confronta con l'esecuzione precedente** usando il modulo di baseline

4. **Invia alert** quando:
   - Il punteggio complessivo scende sotto una soglia
   - Un controllo precedentemente superato inizia a fallire (regressione)
   - Compare un nuovo controllo con stato FAIL

5. **Genera un riepilogo settimanale** che mostri la tendenza della conformità

**Criteri di successo:**
- [ ] Il timer systemd esegue cisaudit ogni giorno
- [ ] I risultati sono memorizzati con timestamp
- [ ] Le regressioni fanno scattare degli alert
- [ ] Un comando CLI mostra la tendenza della conformità negli ultimi 30 giorni

---

## Sfide di sicurezza

### Sfida: superare il CIS Level 2

**L'obiettivo:**
Prendi un'installazione Debian 12 appena fatta e usa cisaudit per raggiungere il 100% di conformità al Level 2.

**Lacune attuali su un'installazione nuova:**
- I moduli dei filesystem non sono disabilitati
- /tmp non è una partizione separata
- Il bootloader non ha una password
- La maggior parte dei servizi è nella configurazione predefinita
- Le regole di audit non sono configurate
- SSH consente il login come root
- Le policy delle password usano i valori predefiniti

**Approccio:**
1. Esegui `sudo cisaudit -l 2 -f json -o baseline-fresh.json`
2. Affronta ogni controllo fallito usando i comandi di remediation
3. Riesegui dopo ogni sezione per tracciare i progressi
4. Documenta quali controlli di Level 2 hai dovuto saltare e perché

**Fai attenzione a:**
- Disabilitare vfat rompe il boot UEFI su alcuni sistemi
- Disabilitare squashfs rompe i pacchetti snap
- Impostare policy delle password aggressive può chiuderti fuori se non stai attento
- Alcune regole di audit generano grandi volumi di dati di log

---

## Sfide di integrazione nel mondo reale

### Integrazione con Wazuh

**L'obiettivo:**
Inviare i risultati di cisaudit a un SIEM Wazuh per un monitoraggio centralizzato della conformità.

**Cosa ti servirà:**
- Un server Wazuh (può essere distribuito con Docker)
- La comprensione della raccolta dei log e del motore di regole di Wazuh

**Piano di implementazione:**
1. Emetti i risultati di cisaudit come messaggi in formato syslog
2. Configura l'agent Wazuh per leggere i log di cisaudit
3. Crea regole Wazuh che generino alert per i fallimenti CIS
4. Costruisci una dashboard Wazuh che mostri la postura di conformità tra gli host

### Integrazione con Prometheus/Grafana

**L'obiettivo:**
Esporre i punteggi di cisaudit come metriche Prometheus per la visualizzazione in Grafana.

**Piano di implementazione:**
1. Aggiungi un formato di output `prometheus` che emetta metriche in formato OpenMetrics
2. Scrivi uno script textfile collector che esegua cisaudit e scriva le metriche in `/var/lib/node_exporter/textfile_collector/`
3. Crea una dashboard Grafana con indicatori di conformità per sezione e una linea di tendenza nel tempo

**Metriche da esporre:**
```
cisaudit_score_overall{hostname="web01"} 82.5
cisaudit_score_section{section="Initial Setup"} 88.9
cisaudit_controls_pass{hostname="web01"} 72
cisaudit_controls_fail{hostname="web01"} 18
```

---

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

- [ ] Sfida facile 1: report CSV
- [ ] Sfida facile 2: riepilogo del conteggio dei controlli
- [ ] Sfida facile 3: attivazione/disattivazione dei colori
- [ ] Sfida intermedia 4: supporto a RHEL/CentOS
- [ ] Sfida intermedia 5: report in Markdown
- [ ] Sfida intermedia 6: esecuzione parallela
- [ ] Sfida avanzata 7: auto-remediation
- [ ] Sfida avanzata 8: generazione di playbook Ansible
- [ ] Sfida avanzata 9: profili di configurazione
- [ ] Sfida per esperti 10: scansione SSH multi-host
- [ ] Sfida per esperti 11: monitoraggio continuo
- [ ] Sfida di sicurezza: superare il CIS Level 2
- [ ] Integrazione: Wazuh
- [ ] Integrazione: Prometheus/Grafana

Le hai completate tutte? Hai costruito una piattaforma di compliance di livello production. Valuta di contribuire con dei miglioramenti al progetto o di usare i pattern che hai imparato per costruire strumenti di compliance per altri benchmark (CIS AWS, CIS Kubernetes, CIS Docker).
