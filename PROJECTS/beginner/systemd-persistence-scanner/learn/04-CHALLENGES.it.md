# Sfide di estensione

Idee per estendere questo progetto, ordinate per difficoltà. Ognuna insegna una competenza diversa. Non sentirti obbligato a farle in ordine.

## Sfide facili

### 1. Aggiungere uno scanner di persistenza Docker

I container Docker possono essere configurati per riavviarsi automaticamente, e i volumi Docker possono montare percorsi dell'host che persistono attraverso i riavvii dei container. Gli attaccanti usano container con `--restart=always` e bind mount su /etc/cron.d come meccanismi di persistenza.

**Cosa costruire:** uno scanner che controlla /etc/docker/daemon.json alla ricerca di configurazioni insicure (come `--insecure-registry`), enumera i Dockerfile nelle posizioni comuni e controlla gli override di docker.service.

**Cosa imparerai:** come l'orchestrazione dei container crea nuove superfici di persistenza che gli scanner tradizionali non vedono.

**Suggerimenti:**
- Inizia controllando se /etc/docker/daemon.json esiste e facendone il parsing alla ricerca di impostazioni sospette
- Controlla /etc/systemd/system/docker.service.d/ per gli override drop-in
- Tecnica MITRE: T1610 (Deploy Container)

**Verifica che funzioni:** crea testdata/docker/daemon.json con una configurazione insicura e verifica che lo scanner la segnali.

### 2. Aggiungere uno scanner di persistenza Cloud-Init

Cloud-init viene eseguito al primo boot (e opzionalmente a ogni boot) nelle VM cloud. Gli script di user data in /var/lib/cloud/instance/scripts/ e il cloud-config in /var/lib/cloud/instance/user-data.txt possono contenere comandi arbitrari.

**Cosa costruire:** scansiona le posizioni di cloud-init alla ricerca di script di user-data contenenti strumenti di rete, payload codificati o catene di download-and-execute.

**Cosa imparerai:** come l'inizializzazione cloud crea un vettore di persistenza unico degli ambienti cloud.

**Suggerimenti:**
- Directory chiave: /var/lib/cloud/instance/, /var/lib/cloud/scripts/per-boot/
- Usa ScanFileForPatterns() poiché gli script di cloud-init sono semplice shell
- Tecnica MITRE: T1078.004 (Valid Accounts: Cloud Accounts)

### 3. Aggiungere exit code basati sulla severità

Le pipeline CI/CD hanno bisogno di un modo per far fallire le build quando esistono finding critical. Al momento sentinel esce sempre con 0.

**Cosa costruire:** exit code 1 quando vengono trovati finding pari o superiori a una severità specificata. Aggiungi un flag `--fail-on` che imposta la soglia.

**Cosa imparerai:** come gli strumenti di sicurezza si integrano nelle pipeline automatizzate. La maggior parte degli scanner commerciali usa gli exit code per far rispettare le policy.

**Suggerimenti:**
- Aggiungi `--fail-on` come flag persistente accanto a `--min-severity`
- Dopo aver stampato l'output, controlla se qualche finding raggiunge la soglia
- Exit code 0 = pulito, 1 = finding sopra la soglia, 2 = errore di scansione

**Verifica che funzioni:**
```bash
sentinel scan --root testdata --fail-on critical; echo $?
```

## Sfide intermedie

### 4. Aggiungere il glob pattern matching alle regole di ignore

Attualmente le regole di ignore corrispondono a percorsi esatti. L'uso nel mondo reale richiede glob pattern come `/etc/cron.d/*` o `/home/*/.*rc`.

**Cosa costruire:** estendi il matcher della ignore-list per supportare i glob pattern della shell usando `filepath.Match()`.

**Cosa imparerai:** come bilanciare espressività e sicurezza nei formati di configurazione. I glob pattern sono più utili delle corrispondenze esatte ma introducono il rischio di sopprimere troppo.

**Approccio di implementazione:**

1. Modifica `matchesFinding` in config.go per controllare se il campo path contiene caratteri glob (*, ?, [)
2. Se li contiene, usa `filepath.Match(rule.Path, finding.Path)` invece del confronto esatto
3. Aggiungi casi di test per i pattern con wildcard e punto interrogativo
4. Documenta la sintassi glob supportata nel formato del file di ignore

**Suggerimenti:**
- `filepath.Match()` è sicuro (nessuna ricorsione `**`, nessuna regex injection)
- Considera di aggiungere un campo `pattern:` separato da `path:` se vuoi preservare la retrocompatibilità

### 5. Aggiungere un output di report HTML

Il JSON è ottimo per le macchine. L'output a terminale è ottimo per controlli rapidi. Ma i report di incident response devono essere condivisi con il management e i team legali che non leggono terminali.

**Cosa costruire:** un flag `--html` che genera un report HTML autocontenuto con i finding raggruppati per severità, tabelle ordinabili e sezioni di evidenza espandibili.

**Cosa imparerai:** la generazione di report, il package html/template di Go e come produrre documenti HTML autocontenuti (senza dipendenze esterne CSS/JS).

**Approccio di implementazione:**

1. Crea internal/report/html.go con un template HTML
2. Incorpora il CSS inline in un blocco `<style>`
3. Usa i colori di severità del renderer del terminale come codici esadecimali
4. Includi i metadati della scansione (hostname, timestamp, durata) in un header

**Suggerimenti:**
- Usa `html/template` (non `text/template`) per fare l'auto-escape delle stringhe di evidenza che potrebbero contenere HTML
- Incorpora il template con `//go:embed` per un deploy a singolo binario
- Considera un grafico di riepilogo usando larghezze di barre in puro CSS (nessun JavaScript necessario)

### 6. Aggiungere la raccolta degli hash dei file

Quando un finding fa riferimento a un file sospetto, chi risponde agli incidenti deve verificare che non sia cambiato. Raccogliere gli hash SHA256 al momento della scansione fornisce un registro a prova di manomissione (tamper-evident).

**Cosa costruire:** aggiungi un campo `Hash` a types.Finding. Calcola lo SHA256 per ogni file che produce un finding sopra la severità info.

**Cosa imparerai:** come gli strumenti forensi concatenano le evidenze con gli hash crittografici, e le implicazioni sulle prestazioni del calcolo degli hash dei file durante una scansione.

**Suggerimenti:**
- Usa `crypto/sha256` e `io.Copy()` per il calcolo dell'hash in streaming
- Calcola l'hash solo dei file regolari, non delle directory o dei symlink
- Aggiungi l'hash sia all'output del terminale sia a quello JSON
- Considera un flag `--no-hash` per gli ambienti sensibili alle prestazioni

## Sfide avanzate

### 7. Aggiungere l'integrazione con le regole YARA

YARA è lo standard di settore per il pattern matching nei file. I team di sicurezza scrivono regole YARA per rilevare specifiche famiglie di malware, il tooling dei threat actor e gli indicatori di compromissione. Integrare YARA permetterebbe a sentinel di usare le stesse regole dei prodotti EDR commerciali.

**Cosa costruire:** un flag `--yara-rules` che carica un file di regole YARA e lo applica a ogni file che gli scanner esaminano. Le corrispondenze YARA producono finding con il nome della regola come titolo.

**Cosa imparerai:** l'integrazione CGo (i binding Go di YARA usano CGo), i motori di detection basati su regole e come il settore della sicurezza commerciale affronta il pattern matching.

**Modifiche architetturali necessarie:**

```
Scanner
   │
   ├── MatchLine() (existing regex engine)
   │
   └── MatchYARA() (new YARA engine)
          │
          ▼
      yara.Rules compiled from user-provided .yar file
```

**Passi di implementazione:**

1. Aggiungi la dipendenza github.com/hillu/go-yara/v4
2. Crea internal/scanner/yara.go con la compilazione delle regole e il matching
3. Modifica ScanFileForPatterns per applicare opzionalmente le regole YARA
4. Mappa i metadati delle regole YARA (severity, mitre) sui campi di Finding

**Insidie:**
- CGo significa che la cross-compilazione diventa più difficile e le build sono più lente
- La compilazione delle regole YARA è costosa; compila una volta e riutilizza l'oggetto scanner
- Considera di rendere YARA opzionale (build tag) per mantenere il default a zero dipendenze

### 8. Aggiungere la scansione remota via SSH

Al momento sentinel deve essere distribuito sull'host target. Per una scansione su tutta la fleet, l'esecuzione remota basata su SSH permetterebbe a una singola workstation di scansionare centinaia di server.

**Cosa costruire:** un comando `sentinel remote scan --host user@server` che si connette via SSH al target, copia il binario, lo esegue e fa lo streaming dei risultati JSON indietro.

**Cosa imparerai:** l'integrazione del protocollo SSH in Go, l'auto-deploy del binario e la differenza architetturale tra scansione basata su agent e agentless.

**Passi di implementazione:**

1. Usa golang.org/x/crypto/ssh per la connessione SSH
2. Usa SFTP per copiare il binario di sentinel sull'host remoto
3. Esegui `sentinel scan --json` sull'host remoto
4. Fai il parsing dell'output JSON localmente e renderizzalo
5. Ripulisci il binario remoto dopo la scansione

**Insidie:**
- Il binario deve essere compilato per l'architettura target (GOOS/GOARCH)
- L'autenticazione con chiave SSH dovrebbe essere preferita a quella con password
- Considera un flag `--parallel` per scansionare più host in modo concorrente

### 9. Aggiungere l'analisi della timeline

Invece di una scansione puntuale, costruisci una timeline che mostri quando ogni meccanismo di persistenza è stato installato, correlando i tempi di modifica dei file, le pianificazioni dei cron job e i timestamp delle unit systemd.

**Cosa costruire:** un comando `sentinel timeline` che produce una vista cronologica delle installazioni di persistenza, aiutando chi risponde agli incidenti a ricostruire la sequenza dell'attacco.

**Cosa imparerai:** la ricostruzione forense della timeline, l'analisi dei metadati del filesystem e come la correlazione basata sul tempo rivela pattern di attacco che i singoli finding non mostrano.

**Approccio di implementazione:**

1. Estendi Finding con un campo `Timestamp` (mtime del file)
2. Ordina i finding per timestamp attraverso tutti gli scanner
3. Renderizza come lista cronologica con i delta temporali tra gli eventi
4. Evidenzia i cluster di attività (più modifiche nel giro di minuti suggeriscono un'installazione automatizzata)

## Sfide di sicurezza

### 10. Aggiungere controlli anti-evasione

Gli attaccanti conoscono gli scanner di persistenza. Usano tecniche per eludere la detection:

- Omoglifi Unicode nei nomi dei file (usando la "е" cirillica invece della "e" latina)
- Byte null nel contenuto dei file per rompere il parsing basato sulle righe
- Righe estremamente lunghe per esaurire i motori regex (ReDoS)
- Symlink che puntano fuori dalla root di scansione
- File nascosti (dotfile) in posizioni inaspettate

**Cosa costruire:** fai l'hardening dello scanner contro queste tecniche di evasione.

**Cosa imparerai:** la mentalità avversaria. Costruire uno strumento di sicurezza che può essere esso stesso attaccato ti insegna a pensare come l'attaccante.

**Controlli specifici da aggiungere:**
- Rileva i caratteri non ASCII nei nomi dei file e segnalali
- Segui i symlink con cautela (risolvili e verifica che restino all'interno della root di scansione)
- Imposta una lunghezza massima di riga per il pattern matching
- Scansiona esplicitamente i dotfile (alcuni scanner li saltano)

### 11. Aggiungere il riferimento incrociato ai CIS Benchmark

Mappa i finding di sentinel sui controlli dei CIS Benchmark per la distribuzione target. Un finding come "World-writable unit file" mappa sul controllo CIS 6.1.x (System File Permissions).

**Cosa costruire:** un flag `--cis` che aggiunge gli ID dei controlli CIS accanto agli ID delle tecniche MITRE nell'output.

**Cosa imparerai:** come i framework di compliance si sovrappongono alla threat detection, e perché le organizzazioni hanno bisogno di entrambe le prospettive.

## Sfide di prestazioni

### 12. Benchmark e ottimizzazione del pattern matching

L'attuale motore di pattern esegue tutte le 16 regex su ogni riga. Su sistemi con migliaia di file di configurazione, questo potrebbe diventare un collo di bottiglia.

**Cosa costruire:** aggiungi benchmark usando `testing.B`, fai il profiling del motore di pattern e ottimizza gli hot path.

**Cosa imparerai:** il profiling in Go con pprof, le caratteristiche di prestazione delle regex e quando l'ottimizzazione conta rispetto a quando non conta.

**Approccio:**
- Scrivi benchmark per MatchLine con vari tipi di riga (corrispondenti, non corrispondenti, righe lunghe)
- Fai il profiling con `go test -bench . -cpuprofile cpu.prof`
- Considera: uscita anticipata alla prima corrispondenza quando l'ordinamento per severità è irrilevante, pre-filtraggio con strings.Contains prima della regex, Aho-Corasick per i pattern letterali

**Obiettivo:** scansionare 10.000 file in meno di 1 secondo su un singolo core.

## Idee di contributo

Hai finito una sfida? Condividila:

1. Fai il fork del [repo di sentinel](https://github.com/CarterPerez-dev/sentinel)
2. Implementa la tua estensione in un nuovo branch
3. Aggiungi test e fixture in testdata
4. Invia una PR con la tua implementazione e una descrizione di ciò che rileva

## Metti ulteriormente alla prova te stesso

### Costruisci qualcosa di nuovo

Usa i concetti che hai imparato qui per costruire:
- Uno scanner di persistenza per Windows (chiavi Run, scheduled task, sottoscrizioni WMI, servizi)
- Uno scanner di persistenza per macOS (LaunchAgents, LaunchDaemons, login item, cron)
- Uno scanner di persistenza per Kubernetes (DaemonSet, CronJob, mutating webhook, static pod)

### Studia implementazioni reali

Confronta l'approccio di sentinel con gli strumenti di produzione:
- [Velociraptor](https://github.com/Velocidex/velociraptor) - Usa query VQL per dare la caccia alla persistenza attraverso gli endpoint
- [PEASS-ng/linPEAS](https://github.com/carlospolop/PEASS-ng) - Enumerazione di privilege escalation e persistenza su Linux basata su Bash
- [Autoruns for Linux](https://github.com/microsoft/Autoruns-for-Linux) - La versione di Microsoft dell'enumerazione della persistenza su Linux

Leggi il loro codice, comprendi le loro regole di detection e confronta la copertura. Cosa intercettano che sentinel non intercetta? Cosa intercetta sentinel che a loro sfugge?
