# 00-OVERVIEW.md

# Credential Enumeration

## Che cos'è

Un tool da riga di comando che scansiona le home directory di Linux alla ricerca di credenziali esposte dopo aver ottenuto l'accesso a un sistema. Verifica 7 categorie di archiviazione delle credenziali: chiavi SSH, dati dei browser, configurazioni dei cloud provider (AWS/GCP/Azure/Kubernetes), cronologia della shell, keyring, archivi di credenziali Git e token applicativi. Ogni rilevamento è classificato per gravità in base ai permessi del file e al rischio di esposizione. È scritto in Nim e compilato in un unico binario statico.

## Perché è importante

Le credenziali lasciate nelle home directory sono uno dei punti d'appoggio più affidabili che gli attaccanti trovano dopo l'accesso iniziale. La violazione di LastPass del 2022-2023 è stata ricondotta alla macchina domestica di un ingegnere DevOps, dove un attaccante ha trovato chiavi SSH e credenziali cloud che davano accesso ai vault di produzione. Nell'incidente di CircleCI (gennaio 2023), il laptop di un ingegnere è stato compromesso e i token di sessione sono stati rubati dall'archiviazione del browser, il che ha dato all'attaccante accesso ai segreti dei clienti in tutta la piattaforma. La violazione di Uber del settembre 2022 è iniziata con il social engineering, ma si è aggravata perché c'erano credenziali hardcoded in script PowerShell su condivisioni di rete interne.

Non sono tecniche di attacco esotiche. MITRE ATT&CK le cataloga come T1552 (Unsecured Credentials) con sotto-tecniche per le credenziali nei file (T1552.001), la bash history (T1552.003) e le chiavi private (T1552.004). Separatamente, T1555 copre le credenziali provenienti dai password store (T1555.001 per i keychain, T1555.003 per i browser web). Queste tecniche compaiono in quasi ogni kill chain post-accesso perché funzionano spesso e in modo affidabile.

Questo progetto ti insegna cosa cercano gli attaccanti, dove cercano e come rilevare quell'esposizione prima che venga sfruttata.

**Scenari reali in cui si applica:**
- Operatori red team che mappano le credenziali disponibili dopo essere atterrati su una workstation Linux
- Difensori blue team che sottopongono ad audit le macchine degli sviluppatori per verificarne l'igiene delle credenziali
- Security engineer che integrano il controllo dell'esposizione delle credenziali nei controlli di conformità CI/CD
- Amministratori di sistema che verificano i permessi delle chiavi SSH sulle macchine della flotta
- Incident responder che determinano a quali credenziali un attaccante potrebbe aver avuto accesso

## Cosa imparerai

**Concetti di sicurezza:**
- Le tecniche di credential access di MITRE ATT&CK (T1552, T1555) e come si mappano su posizioni reali del file system
- Il modello dei permessi dei file di Linux e perché 0644 su una chiave privata SSH è un rilevamento mentre 0600 non lo è
- Come i browser memorizzano le credenziali (logins.json + key4.db di Firefox, il database SQLite Login Data di Chromium)
- Gli schemi di archiviazione delle credenziali cloud in AWS, GCP, Azure e Kubernetes
- La cronologia della shell come fonte di intelligence: segreti trapelati negli export, comandi contenenti credenziali, file .env

**Competenze tecniche:**
- Costruire un'architettura di scanner modulare con collector pluggable in Nim
- Ispezione dei permessi dei file Unix usando le syscall POSIX stat
- Pattern matching per il rilevamento dei segreti nella cronologia della shell e nei file di configurazione
- Classificazione strutturata della gravità basata sull'analisi dei permessi
- Doppio rendering dell'output: terminale colorato con box drawing e JSON strutturato

**Strumenti e tecniche:**
- Programmazione di sistema in Nim con gestione della memoria ORC e garanzie di zero eccezioni (`{.push raises: [].}`)
- Compilazione di binari statici con musl per il deploy senza dipendenze di runtime
- Cross-compilation verso x86_64 e aarch64 tramite zigcc
- Test di integrazione basati su Docker con fixture di credenziali piazzate
- Just come task runner per i workflow di build, test e release

## Prerequisiti

**Conoscenze richieste:**
- Fondamenti di Linux: navigare nel file system, comprendere la struttura della home directory, leggere i permessi dei file con `ls -la`
- Concetti di programmazione di base: funzioni, cicli, condizionali, strutture dati. Nim si legge come Python con le annotazioni di tipo, quindi l'esperienza con Python si trasferisce bene
- Basi di sicurezza: cosa sono le credenziali, perché le credenziali non protette sono pericolose, a cosa servono le chiavi SSH

**Strumenti di cui avrai bisogno:**
- Nim 2.2.0+ con il package manager nimble
- Docker (per i test di integrazione)
- Il task runner Just (opzionale ma consigliato)
- Un sistema Linux (il tool prende di mira specificamente gli archivi di credenziali di Linux)

**Utile ma non obbligatorio:**
- Familiarità con la sintassi di Nim. Se conosci Python, sai leggere Nim. Le differenze significative sono i tipi a compile-time e il layout manuale della memoria
- L'esperienza con il penetration testing o le operazioni red team fornisce il contesto del perché queste posizioni delle credenziali sono importanti
- La comprensione dell'autenticazione dei cloud provider (AWS IAM, service account GCP, Kubernetes RBAC)

## Avvio rapido

```bash
cd PROJECTS/intermediate/credential-enumeration

bash install.sh

credenum
```

Output atteso: un report colorato nel terminale che mostra i rilevamenti raggruppati per modulo (SSH, browser, cloud, history, keyring, git, apptoken). Ogni rilevamento mostra un badge di gravità, il percorso del file, i permessi e il timestamp di modifica. Il riepilogo in fondo mostra il totale dei rilevamenti per gravità.

Per eseguirlo con output JSON:

```bash
credenum --format json
```

Per scansionare solo moduli specifici:

```bash
credenum --modules ssh,git,cloud
```

Per eseguire la suite di test di integrazione:

```bash
just docker-test
```

Questo costruisce un container Docker con fixture di credenziali piazzate in tutte e 7 le categorie e convalida che lo scanner rilevi ciascuna di esse.

## Struttura del progetto

```
credential-enumeration/
├── src/
│   ├── harvester.nim              # Entry point, CLI parser, main orchestration
│   ├── runner.nim                 # Routes categories to collectors, aggregates results
│   ├── types.nim                  # Core data structures (Finding, Report, Severity, etc)
│   ├── config.nim                 # All constants: paths, patterns, thresholds, colors
│   ├── collectors/
│   │   ├── base.nim               # Shared utilities: file ops, permissions, finding factories
│   │   ├── browser.nim            # Firefox profiles + Chromium variants
│   │   ├── ssh.nim                # Private keys, config, authorized_keys, known_hosts
│   │   ├── git.nim                # .git-credentials, config helpers, GitHub/GitLab tokens
│   │   ├── cloud.nim              # AWS, GCP, Azure, Kubernetes credential files
│   │   ├── history.nim            # Shell history secrets, command patterns, .env files
│   │   ├── keyring.nim            # GNOME Keyring, KDE Wallet, KeePass, pass, Bitwarden
│   │   └── apptoken.nim           # Database creds, dev tokens, infra tokens, Docker auth
│   └── output/
│       ├── terminal.nim           # Box-drawn colored terminal renderer
│       └── json.nim               # Structured JSON output with metadata
├── tests/
│   ├── test_all.nim               # Unit tests (30+ cases)
│   └── docker/
│       ├── Dockerfile             # Multi-stage: nim builder + ubuntu test runtime
│       ├── validate.sh            # Integration test: runs scanner, checks all 7 categories
│       └── planted/               # Credential fixtures (SSH keys, AWS creds, tokens, etc)
├── config.nims                    # Nim compiler switches (ORC, musl, zigcc, cross-compile)
├── credenum.nimble                # Package manifest
├── Justfile                       # Build, test, release, format commands
└── install.sh                     # One-step install: compile + PATH setup
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per conoscere i vettori di esposizione delle credenziali, i permessi dei file di Linux, come browser e cloud provider memorizzano i segreti e le violazioni reali causate da credenziali non protette
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere il design basato sui collector, come funziona la classificazione della gravità e perché il sistema dei tipi è strutturato in quel modo
3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per un'analisi dettagliata del parser della CLI, di ciascun modulo collector, della logica di analisi dei permessi e dei renderer dell'output
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee che vanno dall'aggiunta di nuovi collector alla costruzione di un motore di remediation

## Problemi comuni

**Versione di Nim troppo vecchia**
```
Error: Nim 2.2+ required (found 1.6.x)
```
Soluzione: aggiorna tramite `choosenim stable` o installa da https://nim-lang.org/install.html

**La build statica fallisce (musl non trovato)**
```
Error: musl-gcc not found
```
Soluzione: installa gli strumenti musl. Su Debian/Ubuntu: `apt install musl-tools`. Su Arch: `pacman -S musl`. Lo script di installazione ripiega sul linking dinamico se musl non è disponibile.

**Il test Docker mostra 0 rilevamenti**
```
Results: 0 passed, 30 failed
```
Soluzione: le fixture di credenziali piazzate potrebbero non essere state copiate. Verifica che `tests/docker/planted/` contenga i file di test e ricostruisci con `just docker-build`.

**Binario troppo grande**
La build di debug produce un binario di circa 2MB. Per un binario più piccolo: `just release-small` compila con le ottimizzazioni, rimuove i simboli e comprime con UPX, producendo un binario sotto i 200KB.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **[secrets-scanner](https://github.com/CarterPerez-dev/Cybersecurity-Projects/tree/main/PROJECTS/intermediate/secrets-scanner)** - Scansiona codebase e cronologia git alla ricerca di segreti trapelati usando l'analisi dell'entropia e il pattern matching. Completa questo progetto coprendo il lato repository anziché il lato file system
- **[docker-security-audit](https://github.com/CarterPerez-dev/Cybersecurity-Projects/tree/main/PROJECTS/intermediate/docker-security-audit)** - Scanner per il CIS Docker Benchmark. Si concentra sulla sicurezza dei container, dove container configurati male possono esporre gli stessi file di credenziali che questo tool rileva
- **[api-rate-limiter](https://github.com/CarterPerez-dev/Cybersecurity-Projects/tree/main/PROJECTS/advanced/api-rate-limiter)** - Rate limiting di livello enterprise per FastAPI. Mostra come le credenziali API rubate (il tipo che questo tool trova) vengono usate per abusare degli endpoint API
