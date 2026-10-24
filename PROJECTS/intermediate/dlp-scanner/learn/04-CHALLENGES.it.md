# 04-CHALLENGES.md

# Sfide di estensione

Hai costruito uno scanner DLP con scansione di file, database e rete, una pipeline di confidence scoring, la mappatura della compliance e il reporting multi-formato. Queste sfide lo estendono in nuovi territori.

Sono ordinate per difficoltà. Quelle facili richiedono un'ora o due. Quelle avanzate sono impegni di più giorni che ti insegnano competenze usate nei sistemi DLP in produzione.

## Sfide facili

### Sfida 1: aggiungere una nuova regola PII (data di nascita)

**Cosa costruire:** Una regola di rilevamento per le date di nascita nei formati comuni: `MM/DD/YYYY`, `YYYY-MM-DD`, `DD-Mon-YYYY`.

**Perché è importante:** La data di nascita è classificata come PHI tra i 18 identificatori di HIPAA e come dato personale dal GDPR. La violazione di Anthem del 2015 ha esposto 78,8 milioni di record incluse le date di nascita, e la combinazione di data di nascita + nome + codice postale basta a identificare in modo univoco l'87% della popolazione statunitense (ricerca di Latanya Sweeney alla Carnegie Mellon).

**Cosa imparerai:**
- Scrivere pattern regex che corrispondono a più formati di data
- Aggiungere una funzione di validazione che rifiuta le date impossibili (mese 13, giorno 32, 30 febbraio)
- Calibrare il base_score in relazione al tasso di falsi positivi (le date compaiono ovunque)

**Suggerimenti:**
- Crea la regola in un nuovo file `detectors/rules/pii_extended.py` e aggiungi la lista di regole ad `ALL_RULES` in `registry.py`
- Usa un base_score basso (0,10-0,15) perché le stringhe di data sono estremamente comuni
- Keyword di contesto come "date of birth", "dob", "birthday", "born on" dovrebbero fornire la maggior parte del segnale
- Il validatore dovrebbe convertire la stringa trovata in una data reale e rifiutare quelle non valide
- Aggiungi la regola a `RULE_FRAMEWORK_MAP` in `compliance.py` con HIPAA e GDPR

**Verifica che funzioni:** Crea un file di testo con "Patient DOB: 03/15/1987" e "Order date: 03/15/1987". Il primo dovrebbe ottenere un punteggio più alto del secondo grazie alle keyword di contesto.

### Sfida 2: output dei report in HTML

**Cosa costruire:** Un nuovo reporter che genera un file HTML autonomo con una tabella dei finding ordinabile, una codifica a colori per severità e un grafico di riepilogo.

**Perché è importante:** I team di compliance spesso devono condividere i risultati delle scansioni con stakeholder non tecnici che non hanno strumenti da riga di comando. Un report HTML che si apre in un browser è più accessibile di JSON o CSV.

**Cosa imparerai:**
- Implementare il pattern del reporter (rispettando il protocollo esistente)
- Generare template HTML in Python (string template o Jinja2)
- Aggiungere un nuovo formato di output alla CLI senza modificare il codice esistente

**Suggerimenti:**
- Crea `reporters/html_report.py` con una classe `HtmlReporter`
- Aggiungi `"html"` a `REPORTER_MAP` in `engine.py` e a `VALID_FORMATS` in `commands/scan.py`
- Usa CSS inline così che il report sia un singolo file HTML autosufficiente senza dipendenze esterne
- Colora i livelli di severità usando lo stesso schema del reporter da console (rosso per critical, giallo per medium, verde per low)
- Includi in cima una sezione di riepilogo con i conteggi per severità e per framework

**Verifica che funzioni:** Esegui `dlp-scan file ./data -f html -o report.html` e apri il file in un browser. La tabella dovrebbe essere ordinabile cliccando sulle intestazioni delle colonne (aggiungi un minimo di JavaScript per farlo).

### Sfida 3: allowlist per pattern del percorso dei file

**Cosa costruire:** Estendere il sistema di allowlist per sopprimere i finding dei file che corrispondono a pattern glob. Attualmente, `allowlists.file_patterns` esiste nella configurazione ma non viene applicato durante la scansione.

**Perché è importante:** Fixture di test, dati mock e file di seed contengono intenzionalmente PII finte. I team perdono ore a fare il triage dei finding da `tests/fixtures/sample_data.csv` che contengono numeri di carte di credito di test. L'allowlist basata sul percorso elimina questo rumore.

**Cosa imparerai:**
- Collegare la configurazione al comportamento in fase di scansione
- Il matching di pattern glob con `fnmatch`
- La differenza tra soppressione a livello di valore e a livello di file

**Suggerimenti:**
- Il campo `AllowlistConfig.file_patterns` esiste già in `config.py`
- Aggiungi il controllo in `FileScanner._scan_file` prima di eseguire il rilevamento, oppure in `_scan_directory` prima di scansionare il file
- Confronta con il percorso relativo rispetto al target della scansione, non con il percorso assoluto
- Pattern come `test_*`, `*_fixture*` e `mock_*` dovrebbero corrispondere ai nomi dei file

**Verifica che funzioni:** Crea un file `test_data.txt` con un SSN valido. Esegui la scansione con `file_patterns: ["test_*"]` nella configurazione. L'SSN non dovrebbe comparire nei risultati.

## Sfide intermedie

### Sfida 4: scansione incrementale con cache degli hash

**Cosa costruire:** Una cache di scansione che memorizza gli hash SHA-256 dei file scansionati e salta i file non modificati nelle scansioni successive.

**Perché è importante:** Le codebase grandi e i file share contengono milioni di file. Riscansionare i file non modificati spreca tempo. Symantec DLP e Microsoft Purview usano entrambi l'hashing del contenuto per saltare i file non modificati, riducendo il tempo di scansione del 60-90% nelle scansioni ripetute.

**Cosa imparerai:**
- Strategie di caching content-addressable
- SQLite come archivio di metadati incorporato
- L'invalidazione della cache (il problema più difficile dell'informatica, e uno che devi davvero risolvere)

**Approccio di implementazione:**

1. **Crea `cache.py`** con una classe `ScanCache` basata su SQLite
   - Tabella: `(file_path TEXT, content_hash TEXT, scan_time TEXT, finding_count INTEGER)`
   - Calcolo dell'hash: SHA-256 del contenuto del file
   - Lookup: se il file esiste nella cache con lo stesso hash, salta la scansione e carica il conteggio dei finding in cache

2. **Integra con `FileScanner`**
   - Prima di estrarre il testo, controlla la cache
   - Dopo la scansione, memorizza l'hash e il conteggio dei finding
   - Aggiungi un flag `--no-cache` per forzare una riscansione completa

3. **Gestisci i casi limite dell'invalidazione:**
   - E se le regole di rilevamento cambiano tra una scansione e l'altra? (Lo stesso file potrebbe produrre finding diversi con nuove regole)
   - E se la configurazione cambia min_confidence? (I finding prima soppressi potrebbero ora dover essere riportati)
   - E se un file viene eliminato? (Le voci obsolete della cache non dovrebbero comparire nei risultati)

**Suggerimenti:**
- Memorizza nella cache un hash del set di regole attivo e della configurazione. Se uno dei due cambia, invalida l'intera cache
- Usa `aiosqlite` per rispettare il pattern asincrono del database scanner, oppure usa sqlite3 sincrono dato che la scansione dei file è già sincrona
- Il file di cache dovrebbe stare accanto alla configurazione: `.dlp-scanner-cache.db`

**Punti extra:** Aggiungi i sottocomandi `dlp-scan cache stats` e `dlp-scan cache clear`.

### Sfida 5: override della severità per framework di compliance

**Cosa costruire:** Un'opzione di configurazione che sovrascrive la severità in base ai requisiti del framework di compliance. Per esempio, qualsiasi finding PCI-DSS dovrebbe essere almeno "high" indipendentemente dal confidence score, perché PCI-DSS non ha un concetto di dati di carte non cifrati a "bassa severità".

**Perché è importante:** Framework di compliance diversi hanno soglie di severità diverse. Il GDPR tratta gli indirizzi email non cifrati come priorità media per la remediation, ma PCI-DSS tratta qualsiasi PAN non cifrato come un finding bloccante. Gli strumenti DLP in produzione permettono ai team di compliance di configurare soglie minime di severità per framework.

**Cosa imparerai:**
- Aggiungere comportamento guidato dalla configurazione alla pipeline di scoring
- La tensione tra severità basata sulla confidenza e severità basata sulle policy
- Come gli strumenti DLP in produzione bilanciano l'accuratezza del rilevamento con i requisiti di compliance

**Approccio di implementazione:**

1. **Aggiungi alla configurazione:**
   ```yaml
   compliance:
     severity_overrides:
       PCI_DSS: "high"
       HIPAA: "medium"
   ```

2. **Applica dopo lo scoring:** Nella funzione `_match_to_finding` (o equivalente), dopo aver calcolato la severità dalla confidenza, controlla se uno qualsiasi dei framework di compliance del finding ha una soglia minima di severità, e alzala se necessario

3. **Preserva la confidenza originale:** Il confidence score non dovrebbe cambiare. Cambia solo la classificazione della severità. Questo permette agli analisti di vedere che un finding ha ottenuto 0,35 (normalmente "low") ma è stato elevato a "high" a causa della policy PCI-DSS

**Suggerimenti:**
- Aggiungi `severity_overrides: dict[str, str]` a `ComplianceConfig` in `config.py`
- Usa `SEVERITY_ORDER` da `constants.py` per confrontare numericamente i livelli di severità
- Registra nel log quando una severità viene sovrascritta, così gli analisti capiscono perché un finding a bassa confidenza compare con severità alta

### Sfida 6: scoring euristico basato sui nomi delle colonne del database

**Cosa costruire:** Un'euristica pre-scansione che aumenta la confidenza del rilevamento per le colonne i cui nomi suggeriscono dati sensibili (ad es. `ssn`, `credit_card_number`, `patient_dob`).

**Perché è importante:** I nomi dello schema del database sono segnali di metadati forti. Una colonna chiamata `ssn` in una tabella chiamata `employees` sta quasi certamente memorizzando Social Security Number, anche prima di guardare i dati. L'indagine sulla violazione di Capital One ha rilevato che il bucket S3 compromesso conteneva export CSV con intestazioni di colonna come `SSN` e `AccountNumber`, che sarebbero state banalmente rilevabili con un'analisi dei nomi delle colonne.

**Cosa imparerai:**
- L'introspezione dello schema come segnale di rilevamento
- Combinare segnali di metadati e di contenuto
- Come gli strumenti DLP in produzione usano l'analisi dello schema per dare priorità alla scansione

**Approccio di implementazione:**

1. **Crea un classificatore dei nomi delle colonne** con pattern che mappano i nomi delle colonne sugli ID delle regole:
   ```
   *ssn*, *social_sec* -> PII_SSN
   *credit_card*, *card_num*, *pan* -> FIN_CREDIT_CARD
   *email*, *e_mail* -> PII_EMAIL
   *dob*, *date_of_birth*, *birthday* -> PII_DOB
   ```

2. **Applicalo come boost di contesto** nel database scanner: quando il nome di una colonna corrisponde a un pattern, aggiungi un pre-boost al punteggio base prima di eseguire la normale pipeline di rilevamento

3. **Portalo fino ai finding:** Aggiungi la corrispondenza del nome della colonna come prova aggiuntiva nei metadati del finding

**Suggerimenti:**
- Implementalo in `scanners/db_scanner.py` prima del loop di rilevamento
- Usa `fnmatch` per il matching dei pattern sui nomi delle colonne (come per il filtraggio delle regole)
- Un boost moderato (da +0,15 a +0,25) è appropriato. I nomi delle colonne sono segnali forti ma non definitivi (una colonna chiamata `ssn_backup_old` potrebbe essere vuota o cifrata)

## Sfide avanzate

### Sfida 7: linguaggio di regole personalizzate

**Cosa costruire:** Un formato di definizione delle regole basato su YAML che permetta agli utenti di creare regole di rilevamento senza scrivere Python. Le regole dovrebbero supportare pattern regex, punteggi base, keyword di contesto e tag dei framework di compliance.

**Perché è importante:** Gli strumenti DLP in produzione (Symantec DLP, Netskope) permettono ai team di compliance di definire regole personalizzate tramite editor di policy, perché non tutti i tipi di dati regolamentati sono coperti dalle regole integrate. Gli IBAN europei, i CPF brasiliani, i numeri Aadhaar indiani e gli identificatori specifici di settore hanno tutti bisogno di pattern personalizzati.

**Cosa imparerai:**
- Il design di una DSL (mantenerla abbastanza semplice da essere utile, abbastanza complessa da essere potente)
- La compilazione sicura delle regex (prevenire il ReDoS)
- L'hot reloading delle regole definite dall'utente

**Approccio di implementazione:**

1. **Definisci lo schema delle regole:**
   ```yaml
   rules:
     - id: CUSTOM_BR_CPF
       name: "Brazilian CPF Number"
       pattern: '\b\d{3}\.\d{3}\.\d{3}-\d{2}\b'
       base_score: 0.40
       context_keywords: ["cpf", "cadastro"]
       compliance: ["LGPD"]
       validator: "mod11"
   ```

2. **Costruisci un loader delle regole** che legge file YAML da una directory `rules/`, compila i pattern regex in modo sicuro (con protezione tramite timeout contro il backtracking catastrofico) e crea oggetti `DetectionRule`

3. **Registra le regole personalizzate** accanto a quelle integrate nel `DetectorRegistry`

4. **Aggiungi riferimenti ai validatori integrati** (mod11, luhn, mod97) che gli utenti possono richiamare per nome invece di scrivere Python

**Insidie:**
- La compilazione delle regex deve essere sicura: un pattern fornito dall'utente come `(a+)+b` causa un backtracking catastrofico. Valuta di usare la libreria `regex` con timeout, oppure valida i pattern contro pattern ReDoS noti
- Le regole personalizzate non dovrebbero poter sovrascrivere o oscurare le regole integrate. Usa prefissi negli ID (`CUSTOM_`) per dar loro un namespace
- Le funzioni di validazione richiamate per nome hanno bisogno di un registry tutto loro

### Sfida 8: monitoraggio dei file in tempo reale

**Cosa costruire:** Una modalità watch che monitora le directory alla ricerca di modifiche ai file usando gli eventi del filesystem e scansiona automaticamente i file nuovi o modificati.

**Perché è importante:** La scansione batch trova i problemi a posteriori. Il monitoraggio in tempo reale intercetta i dati sensibili non appena arrivano su disco. È così che funzionano gli agent DLP sugli endpoint (CrowdStrike Falcon DLP, Digital Guardian): si agganciano agli eventi del filesystem e scansionano in tempo reale.

**Cosa imparerai:**
- Il monitoraggio degli eventi del filesystem con `watchdog` o `inotify`
- Il debouncing degli eventi (un singolo salvataggio di file può generare più eventi)
- La scansione in background senza bloccare l'event loop

**Modifiche architetturali:**

```
┌──────────────────────────────┐
│  FileSystemEventHandler       │
│  (watchdog or inotify)        │
│                               │
│  on_modified -> debounce      │
│  on_created  -> scan_file     │
│  on_moved    -> scan_dest     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  ScanQueue (asyncio.Queue)    │
│                               │
│  Dedup by path               │
│  Rate limit scanning          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  FileScanner.scan(file)       │
│  → Finding → Alert            │
└──────────────────────────────┘
```

**Passi di implementazione:**

1. Aggiungi `watchdog` come dipendenza
2. Crea `commands/watch.py` con un comando `dlp-scan watch ./directory`
3. Implementa un debouncer che raggruppa gli eventi del filesystem in una finestra di 500 ms
4. Usa l'esistente `FileScanner._scan_file` per la scansione dei singoli file
5. Mostra i finding in console in tempo reale (modalità stream, non batch)

**Insidie:**
- Le operazioni di salvataggio degli editor spesso creano file temporanei, ci scrivono e poi li rinominano. Questo genera eventi di creazione, modifica e rinomina. Devi scansionare il file finale, non i file temporanei intermedi
- Le copie di file grandi attivano ripetutamente `on_modified` mentre i dati vengono scritti. Fai debouncing aspettando che la dimensione del file si stabilizzi
- La modalità watch dovrebbe rispettare gli stessi pattern di esclusione e filtri di estensione della scansione batch

### Sfida 9: integrazione con un SIEM tramite syslog

**Cosa costruire:** Un reporter che invia i finding a un SIEM (Splunk, Elastic, QRadar) tramite syslog (RFC 5424) o HTTP Event Collector (Splunk HEC).

**Perché è importante:** I finding DLP sono inutili se restano in un file JSON che nessuno legge. I deployment DLP in produzione inviano gli alert ai SIEM, dove gli analisti SOC ne fanno il triage insieme ai log dei firewall, agli alert EDR e agli eventi di autenticazione. Correlare un finding DLP con un login VPN da una località insolita trasforma un alert di severità media in un incidente.

**Cosa imparerai:**
- La formattazione del protocollo syslog (structured data RFC 5424)
- L'invio dei log via HTTP (Splunk HEC, Elastic Ingest)
- La gestione dell'alert fatigue (batching, deduplicazione, filtraggio per severità)

**Approccio di implementazione:**

1. **Crea `reporters/syslog_reporter.py`** che formatta i finding come messaggi syslog RFC 5424:
   ```
   <134>1 2026-04-08T10:30:00Z scanner dlp-scan - -
   [finding@dlp rule_id="PII_SSN" severity="critical"
   confidence="0.92" uri="employees.csv"] SSN detected
   ```

2. **Aggiungi il supporto a Splunk HEC** come trasporto alternativo: invia via POST payload JSON a `https://splunk:8088/services/collector/event` con un token HEC

3. **Aggiungi alla configurazione:**
   ```yaml
   output:
     siem:
       type: "syslog"        # or "splunk_hec"
       host: "siem.corp.com"
       port: 514
       protocol: "tcp"       # or "udp"
       hec_token: ""         # for Splunk HEC
   ```

4. **Implementa il batching:** Invia i finding in batch da 50 con un intervallo di flush di 5 secondi per evitare di sovraccaricare il SIEM

## Sfide per esperti

### Sfida 10: riduzione dei falsi positivi con il machine learning

**Cosa costruire:** Un ciclo di feedback in cui gli analisti possono marcare i finding come veri positivi o falsi positivi, e un classificatore impara a sopprimere i probabili falsi positivi nelle scansioni future.

**Perché è importante:** La lamentela più grande in assoluto sugli strumenti DLP è il volume di falsi positivi. I deployment di Symantec DLP vedono comunemente tassi di falsi positivi del 40-60% al rollout iniziale. Gli analisti passano ore a scartare finding che corrispondono a pattern SSN ma sono in realtà numeri di serie, ID di batch o codici zip+4. Un classificatore addestrato sul feedback degli analisti può ridurre i falsi positivi del 70-80% mantenendo la recall del rilevamento.

**Cosa imparerai:**
- La feature engineering a partire dai segnali di rilevamento (confidenza, keyword di contesto trovate, tipo di regola, tipo di file, pattern del testo circostante)
- L'online learning: aggiornare un modello man mano che arriva nuovo feedback senza riaddestrarlo da zero
- Il compromesso precision-recall nel tooling di sicurezza (un falso negativo è una violazione mancata; un falso positivo è affaticamento dell'analista)

**Fasi di implementazione:**

**Fase 1: raccolta del feedback**
- Aggiungi i comandi `dlp-scan feedback <finding_id> --true-positive` e `--false-positive`
- Memorizza il feedback in un database SQLite: `(finding_id, rule_id, features_json, label, timestamp)`
- Estrai le feature: confidenza, rule_id, estensione del file, keyword di contesto trovate, conteggio delle co-occorrenze, entropia del testo circostante

**Fase 2: classificatore**
- Addestra una regressione logistica o un gradient boosted tree sul feedback accumulato
- Feature: codifica one-hot di rule_id, confidenza numerica, booleano context_found, categoria di file_extension
- Usa scikit-learn con export ONNX per il deployment senza l'intera dipendenza da sklearn

**Fase 3: integrazione**
- Dopo che la pipeline di rilevamento ha prodotto le corrispondenze, esegui il classificatore come post-filtro
- Le corrispondenze classificate come probabili falsi positivi vengono declassate (severità abbassata, oppure spostate in una sezione "soppressi")
- Non sopprimere mai completamente un rilevamento. Mostra sempre i finding soppressi in una sezione separata così che gli analisti possano verificare il classificatore

**Criteri di successo:**
- [ ] La raccolta del feedback funziona e memorizza le feature
- [ ] Il classificatore si addestra su 50+ esempi etichettati
- [ ] Il tasso di falsi positivi scende di almeno il 30% su un test set tenuto da parte
- [ ] Nessun vero positivo viene soppresso completamente (solo declassato)
- [ ] Il modello si riaddestra automaticamente quando il conteggio dei feedback supera delle soglie (100, 500, 1000)

## Sfide di integrazione nel mondo reale

### Integrare con GitHub Code Scanning

**L'obiettivo:** Caricare l'output SARIF su GitHub Code Scanning così che i finding DLP compaiano come annotazioni sulle pull request.

**Cosa imparerai:**
- La API di GitHub Code Scanning
- Il caricamento di SARIF tramite GitHub Actions
- L'integrazione nelle pipeline CI/CD per il tooling di sicurezza

**Passi:**

1. Crea un workflow GitHub Actions che esegue `dlp-scan file . -f sarif -o results.sarif` sulle pull request
2. Carica il file SARIF usando l'action `github/codeql-action/upload-sarif`
3. Configura `on: pull_request` per scansionare solo i file modificati (usa `git diff --name-only` per ottenere l'elenco)
4. Imposta il filtraggio per severità così che solo i finding high/critical blocchino la PR

### Scansionare i bucket AWS S3

**L'obiettivo:** Aggiungere uno scanner S3 che elenca gli oggetti in un bucket, li scarica in una directory temporanea e li scansiona con il file scanner esistente.

**Cosa imparerai:**
- L'integrazione di boto3 per l'elenco e il download degli oggetti S3
- La gestione dei file temporanei per la scansione di oggetti grandi
- La gestione delle credenziali (ruoli IAM contro access key)

**Passi:**

1. Aggiungi il comando `dlp-scan s3 s3://bucket-name/prefix`
2. Usa boto3 per elencare gli oggetti, filtrando per estensione
3. Scarica ogni oggetto in una directory temporanea (usa `tempfile.mkdtemp`)
4. Scansiona con `FileScanner` e rimappa i finding sugli URI S3
5. Elimina i file temporanei dopo la scansione

Questo affronta direttamente lo scenario della violazione di Capital One: PII non cifrate in bucket S3 di cui nessuno conosceva l'esistenza.

## Sfida di prestazioni

### Gestire 1 milione di file

**L'obiettivo:** Fare in modo che il file scanner gestisca una directory con 1 milione di file senza esaurire la memoria né impiegare più di un'ora.

**Collo di bottiglia attuale:** `Path.rglob("*")` genera un elenco di tutti i file prima che inizi la scansione. Con 1 milione di file, questo consuma una quantità significativa di memoria e ritarda il primo risultato della scansione.

**Approcci di ottimizzazione:**

**Approccio 1: attraversamento delle directory in streaming**
- Sostituisci `rglob` con un attraversamento ricorsivo tramite `os.scandir` che produce i file uno alla volta
- Elabora e scarta ogni file prima di leggere il successivo
- La memoria resta costante indipendentemente dalla dimensione della directory

**Approccio 2: estrazione in parallelo**
- Usa `concurrent.futures.ThreadPoolExecutor` per l'estrazione I/O-bound (lettura di file, parsing dei PDF)
- Usa `concurrent.futures.ProcessPoolExecutor` per il rilevamento CPU-bound (matching regex su testi grandi)
- Calibra la dimensione dei pool in base al profiling

**Approccio 3: scansione con priorità**
- Scansiona prima le estensioni ad alto rischio (`.csv`, `.xlsx`, `.sql`) e poi quelle a basso rischio (`.log`, `.txt`)
- Riporta i finding man mano che vengono scoperti (output in streaming) invece di aspettare che la scansione completa termini

**Fai un benchmark:**

```bash
time dlp-scan file /large-directory -f json -o results.json
```

Obiettivo: meno di 60 minuti per 1 milione di file con una dimensione media dei file di 10KB.

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

- [ ] Facile 1: regola per la data di nascita
- [ ] Facile 2: output dei report in HTML
- [ ] Facile 3: allowlist per percorso dei file
- [ ] Intermedia 4: scansione incrementale
- [ ] Intermedia 5: override della severità
- [ ] Intermedia 6: euristica sui nomi delle colonne
- [ ] Avanzata 7: linguaggio di regole personalizzate
- [ ] Avanzata 8: monitoraggio in tempo reale
- [ ] Avanzata 9: integrazione con un SIEM
- [ ] Esperto 10: riduzione dei falsi positivi con ML
- [ ] Integrazione: GitHub Code Scanning
- [ ] Integrazione: scansione dei bucket S3
- [ ] Prestazioni: 1 milione di file

## Studia implementazioni reali

Confronta il tuo lavoro con gli strumenti DLP in produzione:

- **Nightfall AI**: DLP cloud-native con rilevamento basato su ML. Hanno reso open source i loro pattern di rilevamento. Guarda come gestiscono l'estrazione multi-formato
- **truffleHog**: si concentra sul rilevamento delle credenziali nei repository git. Il loro rilevamento basato sull'entropia e i loro pattern regex per le API key sono simili alle regole sulle credenziali di questo progetto
- **detect-secrets**: lo scanner di segreti di Yelp. Confronta la loro architettura a plugin con il pattern del detector registry di questo progetto
- **Microsoft Purview**: DLP enterprise con oltre 300 tipi di informazioni sensibili integrati. La loro documentazione sull'exact data match (EDM) e sui classificatori addestrabili mostra dove si sta dirigendo il settore
