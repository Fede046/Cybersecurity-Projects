# Binary Analysis Tool: sfide

Sfide di estensione organizzate per livello di difficoltà. Ciascuna si basa sulla codebase esistente — lavorerai con la stessa pipeline basata sui pass, lo stesso type system e gli stessi pattern stabiliti nell'implementazione.

## Sfide facili

### 1. Aggiungere il caricamento di regole YARA personalizzate

**Cosa costruire:** Consentire agli utenti di caricare le proprie regole YARA insieme a quelle integrate. Accetta file `.yar` tramite un nuovo endpoint API e compilali nello scanner.

**Perché è utile:** Ogni team SOC dispone di regole YARA personalizzate per le minacce specifiche del proprio ambiente. Le regole hardcoded non possono coprire gli indicatori specifici di un'organizzazione.

**Cosa imparerai:** Compilazione delle regole YARA, configurazione basata su file, estensione dell'API senza modificare l'interfaccia principale del motore.

**Suggerimenti:**
- Guarda come `YaraScanner::new()` in `yara.rs` compila la stringa delle regole integrate
- `yara-x` supporta la compilazione di più sorgenti di regole in un unico scanner
- Memorizza le regole caricate in una directory e ricompila lo scanner quando le regole cambiano
- Aggiungi un endpoint `GET /api/rules` per elencare le regole attive

**Come verificare che funzioni:**
```bash
# Create a custom rule
cat > /tmp/custom.yar << 'EOF'
rule detect_hello_world {
    strings:
        $hello = "Hello, world"
    condition:
        $hello
}
EOF

# Upload the rule (you'll need to build this endpoint)
curl -X POST -F "rule=@/tmp/custom.yar" http://localhost:3000/api/rules

# Upload a binary containing "Hello, world" and check YARA results
```

### 2. Esportare l'analisi come report PDF

**Cosa costruire:** Aggiungere un pulsante "Download Report" al frontend che generi un riepilogo in PDF dell'analisi. Includi il threat score, i finding chiave di ciascun pass e le mappature delle tecniche MITRE.

**Perché è utile:** Gli analisti devono condividere i finding con stakeholder non tecnici. Un report in PDF è il deliverable standard nell'incident response.

**Cosa imparerai:** Generazione di PDF nel browser o lato server, sintetizzazione dei dati, formattazione di report.

**Suggerimenti:**
- Lato client: usa `jspdf` o `@react-pdf/renderer` per generare i PDF nel browser
- Lato server: aggiungi un endpoint `GET /api/analysis/{slug}/report` che restituisca `application/pdf`
- Dai priorità ai finding più importanti — un PDF di 50 pagine con ogni stringa non è utile
- Includi la suddivisione del threat score in ASCII dalla scheda di panoramica

**Come verificare che funzioni:**
- Carica un binario di test, naviga alla pagina dell'analisi
- Fai clic su "Download Report" e apri il PDF risultante
- Verifica che includa: metadati del file, threat score, principali categorie di scoring, tecniche MITRE

### 3. Aggiungere il rilevamento del tipo di file oltre gli eseguibili

**Cosa costruire:** Quando viene caricato un file non eseguibile (PDF, ZIP, documento di Office), restituire un messaggio di errore utile che identifichi il tipo di file e suggerisca cosa fare invece di un generico errore "unsupported format".

**Perché è utile:** Gli utenti caricheranno ogni genere di file. Dire loro "questo è un PDF, non un eseguibile — prova prima a estrarre gli oggetti incorporati" è più utile di "parse error".

**Cosa imparerai:** Rilevamento dei magic byte per i tipi di file comuni, progettazione di errori orientati all'utente.

**Suggerimenti:**
- Magic byte comuni: PDF (`%PDF`), ZIP/DOCX/XLSX (`PK\x03\x04`), GZIP (`\x1f\x8b`), classe Java (`\xca\xfe\xba\xbe`)
- Aggiungi il rilevamento in `formats::parse_format()` prima del tentativo di parsing con `goblin`
- Restituisci un errore strutturato con il tipo di file rilevato e una guida utile
- I documenti di Office (DOCX, XLSX) sono archivi ZIP — indica che le macro possono essere estratte come target di analisi separati

**Come verificare che funzioni:**
```bash
# Upload a PDF
curl -X PUT -F "file=@test.pdf" http://localhost:3000/api/upload
# Should return: { "error": "Detected PDF file. Binary analysis requires an executable (ELF, PE, or Mach-O)." }
```

## Sfide intermedie

### 4. Aggiungere il disassembly ARM/AArch64

**Cosa costruire:** Estendere il pass di disassembly per supportare le architetture ARM e AArch64. Attualmente, i binari non-x86 ottengono un risultato di disassembly vuoto.

**Perché è utile:** I binari ARM sono ovunque — app Android (librerie native), firmware IoT, binari macOS Apple Silicon. Saltare il disassembly per ARM significa tralasciare un'enorme categoria di target.

**Cosa imparerai:** Nozioni di base sull'instruction set di ARM, disassembly multi-architettura, estensione del sistema di pass.

**Approccio di implementazione:**
1. Aggiungi il crate `capstone` (o `bad64` per AArch64) a `axumortem-engine`
2. Modifica `DisasmPass::run()` per effettuare il dispatch verso un disassembler ARM quando l'architettura è ARM/AArch64
3. La logica di costruzione dei basic block e del CFG dovrebbe essere riutilizzabile — ARM ha lo stesso concetto di istruzioni di branch che creano i confini dei blocchi
4. ARM presenta considerazioni uniche: la modalità Thumb (istruzioni a 16 bit miste a 32 bit), l'esecuzione condizionale su ogni istruzione (ARM32) e il link register invece degli indirizzi di ritorno basati sullo stack

**Casi limite da testare:**
- Passaggio tra modalità Thumb/ARM all'interno di una singola funzione
- Binari AArch64 da Mac Apple Silicon
- Shared library compilate con Android NDK (file `.so`)

**Suggerimenti:**
- Il crate `iced-x86` è progettato esclusivamente per x86. Serve un decoder separato per ARM.
- Istruzioni di branch ARM: `B`, `BL`, `BX`, `BLX`, `CBZ`, `CBNZ`, `TBB`, `TBH`
- Istruzioni di branch AArch64: `B`, `BL`, `BR`, `BLR`, `RET`, `CBZ`, `CBNZ`, `TBZ`, `TBNZ`

### 5. Aggiungere una vista di confronto

**Cosa costruire:** Consentire il caricamento di due binari e il confronto affiancato dei relativi risultati di analisi. Evidenzia le differenze nelle sezioni, negli import, nelle stringhe e nei threat score.

**Perché è utile:** Il diffing è fondamentale per l'analisi dei malware. Il confronto tra un binario sicuramente legittimo e una versione sospettata di essere trojanizzata rivela esattamente cosa è cambiato. È così che sono stati analizzati gli attacchi alla supply chain di CCleaner e 3CX — effettuando il diff tra il binario legittimo e quello compromesso.

**Cosa imparerai:** Algoritmi di diffing, progettazione di interfacce utente per viste di confronto, query di database efficienti per l'analisi multi-binario.

**Approccio di implementazione:**
1. Aggiungi un endpoint `POST /api/compare` che accetta due slug e restituisce un diff
2. Diffing a livello di sezione: confronta nomi delle sezioni, dimensioni, permessi e hash SHA-256
3. Diffing degli import: import aggiunti, rimossi o modificati tra le versioni
4. Diffing delle stringhe: nuove stringhe sospette nel binario modificato
5. Diffing dell'entropia: sezioni che hanno cambiato classificazione di entropia
6. Frontend: vista split-pane con differenze evidenziate a colori

**Casi limite da testare:**
- Confronto tra binari di formati diversi (ELF vs PE)
- Confronto tra binari con diverso numero di sezioni
- Confronto tra un binario stripped e la sua controparte non-stripped

**Suggerimenti:**
- Gli hash SHA-256 delle sezioni rendono banale il confronto a livello di sezione — se l'hash corrisponde, il contenuto è identico
- Inizia con import e stringhe — sono i diff a più alto segnale per l'analisi della supply chain
- Valuta un layout a tre colonne: binario sinistro, indicatore di diff, binario destro

### 6. Aggiungere il rilevamento dei pattern comportamentali

**Cosa costruire:** Creare un nuovo pass di analisi che identifichi pattern comportamentali di alto livello combinando i finding provenienti da import, stringhe e match YARA. Invece di "queste API sono sospette", segnala "questo binario mostra un comportamento da ransomware" o "questo binario implementa una reverse shell".

**Perché è utile:** I singoli indicatori (un import, una stringa, un match YARA) sono rumorosi. I pattern comportamentali sono detection ad alta confidenza che si mappano direttamente su categorie di minacce.

**Cosa imparerai:** Costruzione di un nuovo pass, correlazione di threat intelligence, pattern matching tra i risultati dei diversi pass.

**Approccio di implementazione:**
1. Crea `passes/behavior.rs` implementando `AnalysisPass`
2. Dichiara le dipendenze da `imports`, `strings`, `entropy` e `threat`
3. Definisci i pattern comportamentali come combinazioni di indicatori:
   - **Ransomware**: API di cifratura + enumerazione file + stringhe di note di riscatto + sezioni ad alta entropia
   - **Reverse shell**: API di socket + exec/spawn + stringhe di comandi di shell
   - **Dropper**: alta entropia + pochi import + stringhe `URLDownloadToFile` o `wget`/`curl`
   - **Keylogger**: API di hook della tastiera + scrittura su file + meccanismi di persistenza
   - **Rootkit**: `ptrace` + stringhe di moduli del kernel + percorsi di file nascosti
4. Aggiungi `behavior_result` ad `AnalysisContext`
5. Registra il pass in `AnalysisEngine::new()`
6. Aggiungi una nuova scheda nel frontend

**Suggerimenti:**
- Inizia con 3-4 pattern ed espandi in seguito — non tentare di coprire tutto subito
- Ciascun pattern dovrebbe richiedere almeno 3 indicatori indipendenti per scattare
- I falsi positivi sono peggiori dei falsi negativi per il rilevamento comportamentale

## Sfide avanzate

### 7. Aggiungere il rilevamento della risoluzione dinamica degli import

**Cosa costruire:** Rilevare quando un binario risolve gli import dinamicamente a runtime (utilizzando `GetProcAddress` su Windows o `dlsym` su Linux) incrociando l'analisi delle stringhe con l'analisi degli import.

**Perché è difficile:** I malware nascondono i propri veri import evitandone l'inserimento nella tabella degli import. Memorizzano invece i nomi delle API come stringhe (talvolta offuscate) e li risolvono a runtime. Rilevare questo comportamento richiede la correlazione tra i finding delle stringhe e i pattern di comportamento degli import.

**Modifiche architetturali necessarie:**
1. Estendere `ImportResult` con un campo `dynamic_imports: Vec<DynamicImportCandidate>`
2. Il pass degli import necessita dell'accesso ai risultati delle stringhe (aggiungi "strings" come dipendenza)
3. Incrociare le stringhe che sembrano nomi di API Windows con il database delle API sospette
4. Segnalare la discrepanza: "il binario importa `GetProcAddress` ma non `VirtualAllocEx`, eppure `VirtualAllocEx` compare come stringa"

**Fasi di implementazione:**
1. **Ricerca (1-2 ore):** Studia come `GetProcAddress`/`dlsym` vengono usati nei malware. Documentati sulla ricostruzione della tabella degli import.
2. **Design (1 ora):** Definisci la struct `DynamicImportCandidate`. Decidi come assegnare il punteggio agli import dinamici rispetto a quelli statici.
3. **Implementazione (3-4 ore):** Modifica il pass degli import per incrociare le stringhe. Costruisci la logica di detection. Aggiorna il threat scoring.
4. **Testing (1-2 ore):** Scrivi un binario di test che usi `GetProcAddress`. Verifica che la detection funzioni.

**Insidie:**
- I nomi delle API nelle stringhe potrebbero essere codificati con XOR o suddivisi su più stringhe
- `GetProcAddress` è usato legittimamente dai sistemi di plugin — non segnalarlo da solo
- Il cambio di dipendenza (gli import ora dipendono dalle stringhe) modifica l'ordine topologico

### 8. Costruire un sistema di Retrohunt

**Cosa costruire:** Quando vengono aggiunte nuove regole YARA, rieseguire automaticamente la scansione di tutti i binari precedentemente analizzati con le nuove regole e aggiornare i loro threat score.

**Perché è difficile:** Questo coinvolge lo storage, l'elaborazione in background e il sistema di caching. È necessario invalidare selettivamente i risultati in cache (solo le porzioni relative a YARA e threat) senza rieseguire pass costosi come il disassembly.

**Modifiche architetturali necessarie:**
1. Memorizzare i dati binari grezzi (o un riferimento ad essi) oltre ai risultati dell'analisi
2. Costruire un sistema di job in background che elabori le nuove scansioni
3. Implementare la riesecuzione parziale dei pass (rieseguire solo ThreatPass con le nuove regole YARA)
4. Aggiornare lo schema del database per tracciare quale versione delle regole ha prodotto ciascun risultato
5. Aggiungere un endpoint `POST /api/retrohunt` che attivi la nuova scansione

**Fasi di implementazione:**
1. **Ricerca (2-3 ore):** Studia come VirusTotal e altre piattaforme gestiscono il retrohunting. Comprendi le sfide legate alla scalabilità.
2. **Design (2 ore):** Progetta la coda dei job, lo schema di versionamento delle regole e la strategia di riesecuzione parziale.
3. **Implementazione (5-6 ore):** Costruisci il sistema di retrohunt. Si tratta di un'aggiunta rilevante — procedi per fasi.
4. **Testing (2-3 ore):** Carica più binari, aggiungi una nuova regola, attiva il retrohunt, verifica che i punteggi si aggiornino.

**Insidie:**
- La memorizzazione dei binari aumenta notevolmente l'uso del disco — valuta la memorizzazione opzionale rispetto al re-upload
- Il retrohunt su migliaia di binari necessita di un indicatore di progresso e del supporto alla cancellazione
- L'aggiornamento dei threat score modifica i livelli di rischio, il che potrebbe innescare alert nei sistemi a valle

### 9. Aggiungere l'unpacking dei packer basato su firme

**Cosa costruire:** Per i binari rilevati come packed con UPX, spacchettarli automaticamente e rianalizzare il binario decompresso. Mostra sia i risultati dell'analisi della versione packed sia di quella unpacked.

**Perché è difficile:** L'unpacking modifica il binario. È necessario gestire il processo di unpacking in sicurezza, memorizzare entrambe le versioni e presentare un confronto significativo. UPX è il target più semplice poiché lo strumento `upx` può decomprimere la maggior parte dei binari UPX, ma altri packer richiedono unpacker personalizzati.

**Modifiche architetturali necessarie:**
1. Dopo che il pass dell'entropia rileva il packing UPX, invocare lo strumento a riga di comando `upx` con `--decompress`
2. Eseguire l'intera pipeline di analisi sul binario spacchettato
3. Memorizzare entrambi i risultati dell'analisi collegati da una relazione genitore-figlio (parent-child)
4. Frontend: mostrare le schede "packed" e "unpacked" con il relativo confronto

**Fasi di implementazione:**
1. **Ricerca (1-2 ore):** Studia il funzionamento interno di UPX e il comando `upx --decompress`. Comprendi le modalità di fallimento.
2. **Design (1 ora):** Progetta lo schema parent-child, decidi in quale punto della pipeline collocare l'unpacking.
3. **Implementazione (4-5 ore):** Aggiungi il passaggio di unpacking, il secondo pass di analisi, le modifiche allo schema del database e gli aggiornamenti al frontend.
4. **Testing (2-3 ore):** Esegui il packing di binari con UPX, caricali, verifica l'unpacking e la ri-analisi.

**Insidie:**
- Un UPX modificato (in cui i nomi delle sezioni sono cambiati da UPX0/UPX1) manda in errore `upx --decompress`
- Il processo di unpacking deve essere eseguito in un ambiente sandbox (esegue uno strumento su input non fidato)
- Non tutti i binari packed con UPX possono essere spacchettati — gestisci i fallimenti in modo controllato
- Valuta di aggiungere `upx` all'immagine Docker

## Sfide per esperti

### 10. Costruire un cluster di analisi distribuito

**Cosa costruire:** Scalare orizzontalmente AXUMORTEM distribuendo il carico di lavoro di analisi su più istanze di backend tramite una coda di job condivisa.

**Tempo stimato:** 15-25 ore

**Prerequisiti:** Comprensione delle code di messaggi, concetti di sistemi distribuiti, orchestrazione di container.

**Architettura ad alto livello:**

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │   (Nginx/LB)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────v──────┐  ┌───v──────┐  ┌───v──────┐
    │  API Server 1  │  │ API 2    │  │ API 3    │
    │  (web only)    │  │          │  │          │
    └─────────┬──────┘  └───┬──────┘  └───┬──────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────v────────┐
                    │   Job Queue     │
                    │ (Redis/RabbitMQ)│
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────v──────┐  ┌───v──────┐  ┌───v──────┐
    │  Worker 1      │  │ Worker 2 │  │ Worker 3 │
    │  (engine only) │  │          │  │          │
    └─────────┬──────┘  └───┬──────┘  └───┬──────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────v────────┐
                    │   PostgreSQL    │
                    │   (shared)      │
                    └─────────────────┘
```

**Fasi di implementazione:**

1. **Configurazione della coda dei job (3-4 ore):** Aggiungi Redis o RabbitMQ. Modifica la route di upload per accodare i job di analisi invece di eseguirli inline. Aggiungi un endpoint di stato per i job in sospeso.

2. **Processo worker (4-5 ore):** Crea un binario separato (`axumortem-worker`) che prelevi i job dalla coda, esegua il motore e scriva i risultati su PostgreSQL. Il worker riutilizza direttamente il crate `axumortem-engine`.

3. **Polling dei risultati (2-3 ore):** Il server API restituisce lo stato "pending" per le analisi in corso. Aggiungi il supporto a WebSocket o SSE per aggiornamenti di progresso in tempo reale. Il frontend effettua il polling o si iscrive per attendere il completamento.

4. **Scalabilità e affidabilità (4-6 ore):** Gestisci i fallimenti dei worker (timeout dei job, dead letter queue). Aggiungi il monitoraggio dello stato di salute per i worker. Implementa la priorità dei job (prima i binari piccoli). Aggiungi manifest Kubernetes o configurazione Docker Swarm.

**Strategia di testing:**
- Unit test: serializzazione/deserializzazione dei job, operazioni sulla coda
- Test di integrazione: ciclo completo di upload-analisi-recupero attraverso la coda
- Test di carico: invia 100 binari concorrenti, verifica che vengano tutti completati
- Chaos test: termina un worker a metà analisi, verifica che il job venga ritentato

**Sfide note:**
- Trasferimento dei dati binari attraverso la coda (passare percorsi di file rispetto a incorporare i dati)
- Dimensionamento del connection pool del database per più worker
- Arresto controllato (graceful shutdown) — i worker devono completare l'analisi corrente prima di fermarsi

**Criteri di successo:**
- [ ] L'analisi funziona in modo identico sia che venga eseguita inline sia attraverso la coda
- [ ] L'aggiunta/rimozione di worker non richiede il riavvio del server API
- [ ] Le analisi fallite vengono ritentate con backoff esponenziale
- [ ] Il frontend mostra il progresso in tempo reale per le analisi accodate
- [ ] Il sistema gestisce 10 volte il throughput di una singola istanza

### 11. Aggiungere la classificazione basata sul Machine Learning

**Cosa costruire:** Addestrare un classificatore sui vettori di feature estratti da AXUMORTEM (valori di entropia, conteggio degli import, statistiche sulle stringhe, proprietà delle sezioni) per prevedere l'appartenenza a famiglie di malware.

**Tempo stimato:** 20-30 ore

**Prerequisiti:** Concetti base di ML (feature engineering, suddivisione train/test, metriche di classificazione). Familiarità con Python per la pipeline di addestramento.

**Architettura ad alto livello:**

```
┌─────────────────────────────────────────────────┐
│ Training Pipeline (Python)                      │
│                                                 │
│  MalwareBazaar ──> Feature Extraction ──> Model │
│  Dataset             (via engine API)    (ONNX) │
└──────────────────────────┬──────────────────────┘
                           │
                    export .onnx file
                           │
┌──────────────────────────v──────────────────────┐
│ Inference (Rust)                                │
│                                                 │
│  AnalysisContext ──> Feature Vector ──> ONNX ──>│
│                         (f32 array)     Runtime │
│                                           │     │
│                                    Classification│
│                                    + Confidence  │
└─────────────────────────────────────────────────┘
```

**Fasi di implementazione:**

1. **Raccolta del dataset (3-4 ore):** Scarica campioni di malware etichettati da MalwareBazaar o VirusTotal. Analizza ciascuno con AXUMORTEM. Esporta i vettori di feature (entropia, conteggio import, conteggio API sospette, conteggio categorie di stringhe, conteggio sezioni, conteggio anomalie, ecc.).

2. **Addestramento del modello (4-6 ore):** Costruisci una pipeline di addestramento in Python. Inizia con un random forest o un gradient boosted tree. L'ingegnerizzazione delle feature (feature engineering) è la parte complessa — decidi quali output dell'analisi diventano feature numeriche. Valuta con precision/recall/F1.

3. **Esportazione del modello (2-3 ore):** Esporta il modello addestrato in formato ONNX. Aggiungi il crate `ort` (ONNX Runtime) al motore.

4. **Integrazione dell'inferenza (5-8 ore):** Crea un nuovo `ClassificationPass` che estragga il vettore di feature da `AnalysisContext`, esegua l'inferenza ONNX e produca un risultato di classificazione (famiglia di malware + punteggio di confidenza). Aggiungi una scheda nel frontend che mostri la classificazione.

**Strategia di testing:**
- Estrazione delle feature: verifica che lo stesso binario produca sempre lo stesso vettore di feature
- Accuratezza del modello: test set di hold-out con etichette note
- Integrazione: upload e classificazione end-to-end

**Sfide note:**
- Normalizzazione delle feature — l'addestramento e l'inferenza devono utilizzare lo stesso ridimensionamento (scaling)
- Sbilanciamento delle classi — i campioni benigni superano di gran lunga i malware nella maggior parte dei dataset
- Drift del modello — il malware si evolve, il modello richiede un riaddestramento periodico
- ONNX Runtime aggiunge circa 50MB all'immagine Docker

**Criteri di successo:**
- [ ] Il modello ottiene un punteggio F1 > 85% sul test set di hold-out
- [ ] La classificazione aggiunge < 100ms al tempo di analisi
- [ ] Il frontend visualizza la previsione della famiglia con la percentuale di confidenza
- [ ] Il tasso di falsi positivi sulle utility di sistema legittime è < 5%

## Combinazioni (Mix and Match)

Combina le sfide per progetti più ampi:

- **Sfide 1 + 8:** Regole YARA personalizzate + retrohunting = una mini piattaforma di threat intelligence
- **Sfide 5 + 9:** Confronto tra binari + unpacking UPX = confronto automatico tra viste packed e unpacked
- **Sfide 6 + 11:** Pattern comportamentali + classificazione ML = rilevamento ibrido (regole + modello)
- **Sfide 4 + 7:** Disassembly ARM + rilevamento import dinamici = analisi completa di malware IoT

## Sfide di integrazione reale

### Integrazione con MISP

Connetti AXUMORTEM a un'istanza [MISP](https://www.misp-project.org/) (Malware Information Sharing Platform). Quando l'analisi rileva una minaccia CRITICAL, crea automaticamente un evento MISP con indicatori di compromissione (IOC): hash dei file, URL C2, catene di API sospette.

### Aggiungere l'arricchimento con VirusTotal

Dopo l'analisi locale, invia l'hash SHA-256 all'API di VirusTotal e mostra i ratio di rilevamento degli antivirus insieme al punteggio di AXUMORTEM. Mostra come lo scoring di AXUMORTEM si confronta con il consenso del settore.

### Costruire un security gate per CI/CD

Crea una modalità CLI per AXUMORTEM che analizzi i binari prodotti da una pipeline di build e interrompa la build con fallimento se il threat score supera una soglia. Utile per intercettare compromissioni della supply chain negli artefatti compilati.

## Sfide sulle prestazioni

### Gestire binari da 50MB in meno di 10 secondi

Esegui il profiling dell'attuale pipeline di analisi. Identifica quali pass sono più lenti per i binari di grandi dimensioni. Ottimizza il percorso critico:
- Il calcolo dell'entropia può usare istruzioni SIMD?
- L'estrazione delle stringhe può essere eseguita in parallelo tra le sezioni?
- Il disassembler può usare più thread per le funzioni indipendenti?

### Ridurre l'uso della memoria del 50%

I binari grandi vengono attualmente caricati interamente in memoria (`Arc<[u8]>`). Implementa l'analisi in streaming dove possibile:
- Il calcolo dell'entropia può funzionare con file mappati in memoria
- L'estrazione delle stringhe può elaborare blocchi (chunks)
- L'hashing delle sezioni funziona già su slice

Esegui il profiling con `heaptrack` o `dhat` per individuare gli hotspot di allocazione.

## Sfide di sicurezza

### Aggiungere la verifica dell'integrità dei binari

Implementa la verifica della firma Authenticode per i binari PE e la verifica codesign per i binari Mach-O. Mostra se la firma è valida, scaduta o proveniente da un certificato noto per essere compromesso (come il certificato rubato a Realtek usato da Stuxnet).

### Costruire un rilevatore di evasione da sandbox

Crea un pass specifico per le tecniche di evasione da sandbox:
- Controlli temporali (delta di `GetTickCount`, loop con `rdtsc`)
- Controlli sull'ambiente (artefatti di VM, presenza di debugger, elenco dei processi)
- Controlli sulle risorse (poca RAM, singola CPU = probabile sandbox)
- Controlli sull'interazione dell'utente (nessun movimento del mouse, nessun documento recente)

Questo è un pass autonomo con una propria categoria di scoring.

### Implementare l'hashing fuzzy SSDEEP

Aggiungi il fuzzy hashing [ssdeep](https://ssdeep-project.github.io/ssdeep/) per identificare binari simili ma non identici. Due varianti della stessa famiglia di malware avranno hash SHA-256 diversi ma hash ssdeep simili. Mostra una sezione "similar binaries" che individui corrispondenze fuzzy nel database.

## Idee per contribuire

Se crei qualcosa di cui sei orgoglioso:

1. Fai il fork del repository, crea un feature branch
2. Segui i pattern di codice esistenti (trait sealed, enum tipizzati, nessun magic number)
3. Aggiungi test per il tuo nuovo pass o per la nuova funzionalità
4. Aggiorna la documentazione di apprendimento se aggiungi funzionalità significative
5. Apri una PR con screenshot di confronto prima/dopo o l'output dei test

## Completamento delle sfide

Traccia i tuoi progressi:

- [ ] **Facile 1:** Caricamento di regole YARA personalizzate
- [ ] **Facile 2:** Esportazione del report in PDF
- [ ] **Facile 3:** Rilevamento del tipo di file
- [ ] **Intermedia 4:** Disassembly ARM/AArch64
- [ ] **Intermedia 5:** Vista di confronto tra binari
- [ ] **Intermedia 6:** Rilevamento dei pattern comportamentali
- [ ] **Avanzata 7:** Risoluzione dinamica degli import
- [ ] **Avanzata 8:** Sistema di retrohunt
- [ ] **Avanzata 9:** Unpacking dei packer
- [ ] **Esperto 10:** Cluster di analisi distribuito
- [ ] **Esperto 11:** Classificazione basata su ML
