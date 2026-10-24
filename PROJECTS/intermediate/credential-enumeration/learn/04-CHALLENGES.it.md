# 04-CHALLENGES.md

# Sfide

Idee di estensione organizzate per difficoltà. Ogni sfida si basa sulla codebase esistente e insegna specifici concetti di sicurezza o di ingegneria.

## Sfide facili

### 1. Aggiungere un collector per le credenziali dei container

**Cosa costruire:** un nuovo modulo collector che cerchi le credenziali dei container runtime oltre a ciò che il collector apptoken già verifica per Docker.

**Perché è utile:** container runtime come Podman, containerd e gli strumenti di orchestrazione dei container memorizzano dati di autenticazione e configurazione nella home directory. Podman usa `~/.config/containers/auth.json` per l'autenticazione ai registry. Buildah e Skopeo condividono lo stesso archivio di credenziali. Override del kubeconfig specifici per i container possono esistere in `~/.config/containers/containers.conf`.

**Competenze che eserciterai:**
- Aggiungere un nuovo collector all'architettura modulare
- Lavorare con l'enum Category e con il matching esaustivo imposto dal compilatore
- Comprendere l'archiviazione delle credenziali dei container runtime

**Suggerimenti:**
- Segui lo schema: aggiungi il valore di Category, aggiungilo agli array di configurazione, crea il file del collector, collegalo nel runner
- Controlla `~/.config/containers/` per auth.json, registries.conf e containers.conf
- Controlla `~/.config/podman/` se esiste
- La logica di scansione dell'autenticazione Docker in apptoken.nim è un buon riferimento per il parsing dell'autenticazione ai registry

**Come testare:** aggiungi file di fixture di Podman/container in `tests/docker/planted/.config/containers/` e aggiungi verifiche a `validate.sh`.

### 2. Aggiungere il formato di output CSV

**Cosa costruire:** un nuovo renderer dell'output che produca CSV con una riga per rilevamento.

**Perché è utile:** l'output CSV si integra con fogli di calcolo, importazioni nei SIEM e strumenti di analisi dei dati. I team di sicurezza hanno spesso bisogno di aggregare i rilevamenti di più host in un unico dataset.

**Competenze che eserciterai:**
- Aggiungere un nuovo formato di output al sistema dei tipi
- Lavorare con la struttura dati Report
- L'escaping CSV corretto (campi che contengono virgole, virgolette o a capo)

**Suggerimenti:**
- Aggiungi `fmtCsv` all'enum `OutputFormat`
- Crea `output/csv.nim` con le colonne: severity, category, path, permissions, modified, description
- Gestisci l'escaping CSV: racchiudi i campi tra virgolette, raddoppia eventuali virgolette interne
- Il renderer JSON è un buon riferimento strutturale

**Come testare:** esegui con `--format csv` e importa in un foglio di calcolo. Verifica che compaiano tutti i rilevamenti e che i caratteri speciali nelle descrizioni non rompano il parsing.

### 3. Aggiungere il filtro per gravità

**Cosa costruire:** un flag `--min-severity` che filtri l'output per mostrare solo i rilevamenti pari o superiori a un determinato livello di gravità.

**Perché è utile:** nelle pipeline CI potresti voler fallire solo sui rilevamenti CRITICAL. In modalità audit potresti voler vedere tutto, INFO compreso. Il tool attuale mostra tutti i rilevamenti e usa HIGH/CRITICAL solo per gli exit code.

**Competenze che eserciterai:**
- Il parsing degli argomenti CLI in Nim
- Il filtraggio dei dati dopo la raccolta
- Lavorare con enum ordinati (i valori di Severity hanno un ordinamento naturale)

**Suggerimenti:**
- Aggiungi un campo `minSeverity` a `HarvestConfig`
- Esegui il parsing di `--min-severity critical|high|medium|low|info` nel parser della CLI
- Filtra i rilevamenti nel runner dopo la raccolta: `res.findings = res.findings.filterIt(it.severity >= config.minSeverity)`
- Aggiorna il calcolo del riepilogo in modo che conti solo i rilevamenti filtrati
- Aggiorna anche la logica dell'exit code perché rispetti il filtro

### 4. Aggiungere avvisi basati sui timestamp

**Cosa costruire:** un flag `--recent <days>` che evidenzi i rilevamenti relativi a file modificati negli ultimi N giorni.

**Perché è utile:** durante l'incident response, ti interessano soprattutto le credenziali a cui si è acceduto o che sono state modificate di recente. Un file `.git-credentials` modificato ieri è più sospetto di uno invariato da due anni.

**Competenze che eserciterai:**
- L'aritmetica delle date in Nim
- Aggiungere indicatori visivi al renderer per il terminale
- La regolazione contestuale della gravità

**Suggerimenti:**
- Esegui il parsing di `--recent 7` nel parser della CLI
- Nel renderer per il terminale, aggiungi un indicatore visivo (colore diverso, marcatore come prefisso) per i rilevamenti in cui la data di modifica rientra nella finestra recente
- Non modificare la gravità in sé, solo la presentazione visiva
- Il campo `modified` di Finding è già una stringa di timestamp ISO 8601

## Sfide intermedie

### 5. Aggiungere il formato di output SARIF

**Cosa costruire:** output in SARIF (Static Analysis Results Interchange Format), il formato standard usato da GitHub Code Scanning, Azure DevOps e altre piattaforme di sicurezza.

**Perché è utile:** SARIF è lo standard del settore per l'output dei tool di sicurezza. Aggiungere il supporto a SARIF significa che i risultati di credenum possono essere caricati su GitHub Code Scanning, mostrati nelle annotazioni delle pull request e importati nelle dashboard di sicurezza.

**Competenze che eserciterai:**
- Implementare un formato di output standard del settore
- Mappare dati specifici del dominio (severity, category) su uno schema standardizzato
- Lavorare con strutture JSON annidate (SARIF è profondamente annidato)

**Approccio di implementazione:**
1. Studia lo schema SARIF 2.1.0 su https://docs.oasis-open.org/sarif/sarif/v2.1.0/
2. Crea `output/sarif.nim`
3. Mappa i concetti di credenum su SARIF: Finding → Result, Category → Rule, Severity → Level
4. I livelli di gravità SARIF sono: error, warning, note, none. Mappa CRITICAL/HIGH → error, MEDIUM → warning, LOW/INFO → note
5. Ogni categoria di collector diventa una "rule" con il proprio ID e la propria descrizione
6. Il campo `physicalLocation` usa il percorso del file di ciascun rilevamento

**Come testare:** carica l'output su GitHub Code Scanning usando `gh api` oppure convalidalo rispetto allo schema SARIF usando un validatore JSON Schema.

### 6. Aggiungere suggerimenti di remediation

**Cosa costruire:** per ogni rilevamento, generare un comando o un'istruzione di remediation specifica.

**Perché è utile:** trovare le credenziali è metà del lavoro. Dire all'utente esattamente come risolvere ogni problema rende il tool operativo invece che solo informativo.

**Competenze che eserciterai:**
- Il pattern matching sui tipi di rilevamento per generare consigli specifici per il contesto
- Il templating di stringhe con percorsi di file e valori reali
- La comprensione delle corrette pratiche di igiene delle credenziali

**Approccio di implementazione:**
1. Aggiungi un campo `remediation` al tipo `Finding` (o una struttura dati parallela)
2. Dopo la raccolta dei rilevamenti, esegui un passaggio di remediation che faccia matching su categoria + pattern della descrizione
3. Genera comandi specifici:
   - Chiave SSH con permessi errati → `chmod 0600 /home/user/.ssh/id_rsa`
   - Chiave SSH non cifrata → `ssh-keygen -p -f /home/user/.ssh/id_rsa` (aggiunge una passphrase)
   - Credenziali AWS leggibili da chiunque → `chmod 0600 /home/user/.aws/credentials`
   - Segreto nella cronologia della shell → `sed -i 'Nd' /home/user/.bash_history` (riga N)
   - .git-credentials in chiaro → `git config --global credential.helper cache` (passa a cache)
4. Aggiungi l'output della remediation sia al renderer per il terminale sia a quello JSON
5. Valuta un flag `--fix` che applichi automaticamente le correzioni dei permessi (con conferma)

**Credito extra:** genera uno script shell (`--remediate-script fix.sh`) che l'utente possa revisionare ed eseguire.

### 7. Supporto a macOS

**Cosa costruire:** collector specifici per piattaforma per le posizioni di archiviazione delle credenziali di macOS.

**Perché è utile:** macOS memorizza le credenziali in posizioni diverse rispetto a Linux. I percorsi dei browser differiscono (`~/Library/Application Support/`), il Keychain sostituisce i keyring desktop e gli strumenti CLI dei cloud potrebbero usare directory di configurazione diverse.

**Competenze che eserciterai:**
- La gestione del filesystem multipiattaforma
- La compilazione condizionale in Nim (`when defined(macosx)`)
- La comprensione del modello di sicurezza di macOS (Keychain, TCC)

**Approccio di implementazione:**
1. Aggiungi il rilevamento della piattaforma (`when defined(linux)` vs `when defined(macosx)`)
2. Crea costanti di percorso specifiche per piattaforma in config.nim
3. Per i browser su macOS:
   - Firefox: `~/Library/Application Support/Firefox/`
   - Chrome: `~/Library/Application Support/Google/Chrome/`
   - Safari: `~/Library/Cookies/`, `~/Library/Keychains/`
4. Per i keychain di macOS: `~/Library/Keychains/login.keychain-db`
5. Le credenziali cloud usano gli stessi percorsi su entrambe le piattaforme
6. La directory SSH è la stessa (`~/.ssh/`)
7. I file di cronologia sono gli stessi (`~/.bash_history`, `~/.zsh_history`)
8. Specifico di macOS: i file plist in `~/Library/Preferences/` possono contenere token

**Insidia:** Transparency, Consent, and Control (TCC) di macOS può bloccare l'accesso ad alcune directory (ad es. i dati di Safari) a meno che il terminale non abbia il Full Disk Access.

### 8. Modalità watch

**Cosa costruire:** un flag `--watch` che riesegua la scansione a un intervallo configurabile e segnali i rilevamenti nuovi o modificati.

**Perché è utile:** il monitoraggio continuo intercetta le credenziali che compaiono dopo la scansione iniziale. Uno sviluppatore scarica un file `.env`, crea una nuova chiave SSH o configura un nuovo cloud provider mentre lo scanner è in esecuzione in background.

**Competenze che eserciterai:**
- Gli event loop e l'esecuzione periodica in Nim
- Il confronto di dati strutturati (confrontare gli insiemi di rilevamenti tra un'esecuzione e l'altra)
- L'aggiornamento del terminale senza inondare l'output

**Approccio di implementazione:**
1. Memorizza i rilevamenti della scansione precedente come insieme (con chiave percorso + categoria + descrizione)
2. A ogni nuova scansione, confronta i nuovi rilevamenti con l'insieme precedente
3. Segnala solo i rilevamenti nuovi, quelli rimossi e le gravità cambiate
4. Usa `--watch 30` per l'intervallo di scansione in secondi (predefinito 60)
5. Cancella e ridisegna il riepilogo a ogni scansione, aggiungi in coda solo i nuovi rilevamenti

## Sfide avanzate

### 9. Scansione delle credenziali di rete

**Cosa costruire:** estendi il tool per cercare credenziali esposte tramite protocolli di rete: condivisioni di rete montate, export NFS, mount SSHFS e condivisioni SMB.

**Perché è utile:** negli ambienti enterprise, le home directory sono spesso montate via NFS. Le credenziali su una macchina potrebbero essere accessibili da qualsiasi macchina del cluster. Le directory montate in rete hanno una semantica dei permessi diversa (il server NFS può ignorare i controlli dei permessi locali con `no_root_squash`).

**Competenze che eserciterai:**
- Rilevare i mount point e i loro tipi (`/proc/mounts` su Linux)
- Comprendere i modelli di permessi NFS rispetto ai permessi del filesystem locale
- La scansione consapevole della rete e la gestione dei timeout

**Approccio di implementazione:**
1. Esegui il parsing di `/proc/mounts` per identificare mount NFS, CIFS, SSHFS e altri mount di rete all'interno del target di scansione
2. Per i mount di rete, regola la gravità: anche i permessi `0600` potrebbero essere aggirati dal file server
3. Scansiona le posizioni comuni delle credenziali di rete: `/etc/fstab` per le credenziali di mount memorizzate, `~/.smbcredentials`
4. Aggiungi un flag `--network` per abilitare questa funzione (disabilitata per impostazione predefinita perché aggiunge latenza)
5. Aggiungi la gestione dei timeout per i percorsi di rete che potrebbero essere lenti o irraggiungibili

### 10. Analisi dell'età delle credenziali

**Cosa costruire:** traccia l'età dei file di credenziali e segnala le credenziali che non sono state ruotate entro una finestra stabilita dalla policy.

**Perché è utile:** un file di credenziali AWS con permessi corretti che non viene ruotato da 18 mesi è comunque un rischio. Molti framework di compliance (SOC 2, PCI DSS) richiedono la rotazione delle credenziali. Questo estende il tool da "è esposta?" a "è obsoleta?"

**Competenze che eserciterai:**
- L'analisi di date e orari rispetto alle soglie della policy
- Definizioni di policy configurabili
- La mappatura sulla compliance (requisiti di rotazione di SOC 2, PCI DSS)

**Approccio di implementazione:**
1. Aggiungi un flag `--max-age <days>` con un valore predefinito di 90 giorni
2. Per ogni rilevamento con un timestamp di modifica, calcola l'età del file
3. Se il file di credenziali è più vecchio della soglia, aggiungi un rilevamento secondario o un flag
4. Gravità per le credenziali obsolete: la data di modifica non dovrebbe cambiare la gravità, ma aggiungi un indicatore `stale` all'output
5. Valuta di leggere la cronologia Git di file come `~/.aws/credentials` per determinare quando è cambiato per l'ultima volta il contenuto (non solo i metadati)

### 11. Modalità agent con reporting remoto

**Cosa costruire:** una modalità daemon che esegua credenum secondo una pianificazione, confronti i risultati con una baseline e invii alert a un endpoint remoto quando compaiono nuovi rilevamenti.

**Perché è utile:** i team di sicurezza che gestiscono flotte di macchine di sviluppatori hanno bisogno di una visibilità continua. Invece di eseguire scansioni ad hoc, la modalità agent fornisce un monitoraggio costante con alert.

**Competenze che eserciterai:**
- Costruire un daemon a lunga esecuzione in Nim
- La gestione della baseline (memorizzare e confrontare i risultati delle scansioni)
- Un client HTTP per inviare i risultati a un endpoint webhook
- La creazione di un file di servizio systemd

**Approccio di implementazione:**
1. Aggiungi un flag `--agent` con `--interval <seconds>` e `--webhook <url>`
2. Alla prima esecuzione, salva il report come file JSON di baseline
3. Nelle esecuzioni successive, confronta il nuovo report con la baseline
4. Se compaiono nuovi rilevamenti o quelli esistenti cambiano gravità, invia il diff al webhook con una POST
5. Supporta più formati di webhook: JSON generico, incoming webhook di Slack, Events API di PagerDuty
6. Scrivi un file di unità systemd per il deploy come servizio di sistema
7. Aggiungi `--baseline <path>` per una gestione esplicita della baseline

**Strategia di test:**
- Esegui unit test sulla logica di confronto (aggiunta, rimozione e cambio di gravità dei rilevamenti)
- Test di integrazione con un server HTTP fittizio che riceve i payload del webhook
- Testa il ciclo di vita del daemon (avvio, scansione, attesa, nuova scansione, arresto)

## Sfida per esperti

### 12. Piattaforma completa per il ciclo di vita delle credenziali

**Cosa costruire:** una dashboard web che aggreghi i risultati di credenum da più host, tracci l'esposizione delle credenziali nel tempo, fornisca visibilità sull'intera flotta e si integri con i workflow di remediation.

**Prerequisiti:** familiarità con un framework web (FastAPI, net/http di Go o simili), progettazione di database e basi di frontend.

**Cosa imparerai:**
- La progettazione di piattaforme per le security operations
- La visibilità delle credenziali sull'intera flotta
- La gestione dei workflow di remediation
- La progettazione di dashboard per le security operations

**Architettura di alto livello:**

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Host A     │  │  Host B     │  │  Host C     │
│  credenum   │  │  credenum   │  │  credenum   │
│  --agent    │  │  --agent    │  │  --agent    │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │ HTTPS/webhook
                        ▼
              ┌─────────────────┐
              │  Aggregation    │
              │  API Server     │
              ├─────────────────┤
              │  PostgreSQL     │
              │  (findings,     │
              │   baselines,    │
              │   hosts)        │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Dashboard      │
              │  - Fleet view   │
              │  - Host detail  │
              │  - Trend charts │
              │  - Remediation  │
              │    tracking     │
              └─────────────────┘
```

**Implementazione per fasi:**

Fase 1 - Fondamenta:
- Definisci lo schema del database (host, scansioni, rilevamenti, baseline)
- Costruisci l'endpoint API che riceve i risultati delle scansioni
- Memorizza e deduplica i rilevamenti

Fase 2 - Funzionalità principali:
- Pagina di panoramica della flotta: host per punteggio di rischio, peggiori trasgressori, modifiche recenti
- Pagina di dettaglio dell'host: elenco completo dei rilevamenti, cronologia, tendenza della gravità
- Vista di confronto: differenze tra scansioni
- Ricerca e filtro su tutti i rilevamenti

Fase 3 - Integrazione:
- Workflow di remediation: assegna i rilevamenti a dei responsabili, traccia la risoluzione, verifica le correzioni
- Regole di alert: un nuovo rilevamento CRITICAL fa scattare PagerDuty/Slack/email
- Vista di compliance: mappa i rilevamenti sui controlli SOC 2 / PCI DSS
- Esportazione: genera report di compliance in PDF/CSV

Fase 4 - Rifinitura:
- Raggruppamento degli host (per team, ambiente, ruolo)
- Tracciamento degli SLA (tempo dal rilevamento alla remediation)
- Token API per l'automazione
- Controllo degli accessi basato sui ruoli

**Criteri di successo:**
- [ ] Gli agent su 3 o più host di test inviano con successo i risultati all'API
- [ ] La dashboard mostra un riepilogo del rischio a livello di flotta
- [ ] I confronti tra rilevamenti individuano correttamente i rilevamenti nuovi, rimossi e modificati
- [ ] Il workflow di remediation traccia almeno un rilevamento dal rilevamento alla risoluzione
- [ ] I grafici di tendenza mostrano le variazioni dell'esposizione nel tempo

## Combina le sfide

Combina le sfide per progetti più grandi:

- **Sfide 3 + 6:** filtro per gravità con suggerimenti di remediation. Filtra da HIGH in su e genera uno script di remediation solo per i rilevamenti urgenti.
- **Sfide 5 + 10:** output SARIF con annotazioni sull'età delle credenziali. Carica su GitHub e segnala le credenziali obsolete come alert di code scanning.
- **Sfide 8 + 11:** modalità watch che alimenta la modalità agent. Monitoraggio locale in tempo reale con alert remoti.
- **Sfide 7 + 9:** supporto a macOS più scansione delle credenziali di rete. Copri le credenziali locali e quelle montate in rete su entrambe le piattaforme.

## Sfide sulle prestazioni

### Esegui il benchmark dei collector

Esegui il profiling del tempo di esecuzione di ciascun collector su home directory di dimensioni diverse. Gli attuali dati di temporizzazione per modulo sono un punto di partenza, ma un profiling più approfondito rivela i colli di bottiglia.

**Cosa misurare:**
- Il tempo per ogni chiamata al file system (stat, readFile, walkDir)
- L'allocazione di memoria per rilevamento
- Il comportamento di scalabilità: come cambia il tempo di scansione con 10 vs 100 vs 1000 file in .ssh/?
- L'impatto dei limiti di profondità sulla scansione della cronologia e dei keyring

**Strumenti:** il flag `--profiler:on` di Nim, `nimprof` o una temporizzazione personalizzata con `getMonoTime()`.

### Ottimizza per home directory grandi

Le home directory degli sviluppatori sui server condivisi possono essere enormi (oltre 100GB, milioni di file). Lo scanner ricorsivo dei .env e lo scanner di KeePass saranno il collo di bottiglia.

**Idee:**
- Usa `inotify` per osservare le modifiche ai file invece di scansioni complete periodiche
- Costruisci un indice dei file alla prima scansione e usa le date di modifica per saltare i file invariati
- Parallelizza gli attraversamenti ricorsivi tra diverse directory di primo livello

## Sfide di sicurezza

### Aggiungi la soppressione dei falsi positivi

Costruisci un formato di file `.credenum-ignore` che permetta agli utenti di contrassegnare i rilevamenti noti come sicuri (ad es. una chiave SSH di test intenzionalmente leggibile da chiunque).

**Requisiti:**
- Supporta esclusioni basate sul percorso (`~/.ssh/test_key`)
- Supporta esclusioni basate su pattern (`*.test.kdbx`)
- Supporta esclusioni basate sulla categoria (`[ssh] id_rsa_test`)
- Anche il file di esclusione stesso dovrebbe essere verificato per i permessi corretti

### Aggiungi la verifica dell'integrità

Calcola l'hash dei file di credenziali e confrontalo con una baseline nota come valida. Se l'hash del contenuto di un file di credenziali cambia senza un evento di rotazione previsto, segnalalo come potenzialmente manomesso.

### Mappatura sulla compliance

Mappa ogni tipo di rilevamento su specifici controlli dei framework di compliance:
- SOC 2 CC6.1 (Logical and Physical Access Controls)
- PCI DSS 8.2 (Authentication Policies)
- CIS Controls 5.2 (Use Unique Passwords)
- NIST 800-53 IA-5 (Authenticator Management)

Produci un report orientato alla compliance che mostri quali controlli hanno rilevamenti.

## Come ottenere aiuto

**Debug dello scanner:** esegui con `--verbose` per vedere tutti i percorsi scansionati, inclusi i moduli con zero rilevamenti. Questo aiuta a capire se un modulo sta cercando nella directory sbagliata o se il percorso di destinazione non è corretto.

**Debug di un collector:** aggiungi istruzioni `echo` temporanee prima di registrare il rilevamento. L'`echo` della stdlib di Nim funziona nei file con `{.push raises: [].}` senza bisogno di racchiuderlo in un try/except, perché `echo` viene trattato in modo speciale dal compilatore.

**Debug della test suite:** esegui `just test` per gli unit test. Se un test specifico fallisce, l'output del test mostra quale asserzione `check` è fallita e i valori attesi rispetto a quelli effettivi. Per i test di integrazione, `docker run --rm -it credenum-test bash` ti dà una shell nel container di test in cui puoi eseguire credenum manualmente.

**Comprendere Nim:** se sei nuovo a Nim, i concetti chiave che compaiono in questa codebase sono:
- Le procedure (`proc`) sono funzioni
- `result` è una variabile di ritorno implicita (equivalente ad assegnare al nome della funzione)
- `{.push raises: [].}` è un pragma del compilatore che impone contratti senza eccezioni
- `{.cast(raises: []).}` sovrascrive il controllo raises per un blocco specifico
- `seq[T]` è un array dinamico, `array[N, T]` ha dimensione fissa
- `Option[T]` è il tipo Maybe/Optional di Nim (da `std/options`)
- `Table[K, V]` è una hash map (da `std/tables`)
