# Sfide di Estensione

Hai costruito il progetto di base. Ora fallo tuo estendendolo con nuove funzionalità.

Queste sfide sono ordinate per difficoltà. Inizia con quelle più semplici per acquisire sicurezza, quindi affronta quelle più complesse quando desideri approfondire ulteriormente.

## Sfide Facili

### Sfida 1: Aggiungere un Honeypot Telnet

**Cosa costruire:**
Un servizio Telnet in `internal/telnetd/` che accetta connessioni sulla porta 2323, esegue la negoziazione delle opzioni IAC, presenta un prompt di login e rilascia una finta shell riutilizzando `FakeFS` e `DispatchCommand` di `sshd`.

**Perché è utile:**
Telnet rimane uno dei protocolli più presi di mira su Internet. Mirai e le sue varianti scansionano Telnet sulla porta 23 per impostazione predefinita. Il SANS Internet Storm Center segnala che i tentativi di brute force su Telnet rivaleggiano con quelli su SSH in termini di volume, in particolare contro i dispositivi IoT. Disporre di honeypot sia SSH che Telnet ti consente di confrontare i tool di attacco utilizzati tra i due protocolli.

**Cosa imparerai:**
- Negoziazione delle opzioni Telnet IAC (Interpret As Command)
- Riutilizzo di componenti esistenti tra servizi diversi

**Suggerimenti:**
- La negoziazione Telnet è semplice: invia `IAC WILL ECHO`, `IAC WILL SUPPRESS_GO_AHEAD`, `IAC WONT LINEMODE` al momento della connessione, quindi rimuovi le sequenze IAC dai dati in arrivo
- Riutilizza direttamente `sshd.FakeFS` e `sshd.DispatchCommand`, poiché non hanno dipendenze specifiche per SSH
- `session.Recorder` funziona allo stesso modo poiché sia Telnet che SSH producono I/O di terminale grezzo

**Verifica del funzionamento:**
`telnet localhost 2323` dovrebbe presentare un prompt di login, accettare qualsiasi credenziale e consentirti di eseguire comandi come `ls`, `whoami` e `cat /etc/passwd` con il medesimo output fittizio dell'honeypot SSH.

### Sfida 2: Aggiungere l'Arricchimento GeoIP alla Mappa della Dashboard

**Cosa costruire:**
Scarica il database gratuito MaxMind GeoLite2 e collega `internal/geo/lookup.go` per restituire coordinate reali, paesi e informazioni sull'ASN per ciascun IP di origine. La mappa degli attacchi nella dashboard esegue già il rendering dei marker a partire dai dati `geo` negli eventi; questa sfida fa sì che tali marker appaiano nelle posizioni corrette anziché con coordinate azzerate.

**Perché è utile:**
La distribuzione geografica degli attacchi è una delle prime cose che i team SOC esaminano durante l'analisi dei dati dell'honeypot. GreyNoise pubblica report giornalieri che mostrano come l'attività di scansione si concentri in specifici ASN e paesi. Dati GeoIP reali consentono di correlare le regioni di origine con le tipologie di attacco.

**Cosa imparerai:**
- Formato binario MMDB di MaxMind GeoLite2
- Limiti di accuratezza della geolocalizzazione IP (il livello di città ha un raggio di accuratezza di ~50 km per la maggior parte degli IP)

**Suggerimenti:**
- Registrati per un account MaxMind gratuito per ottenere una chiave di licenza per i download di GeoLite2-City e GeoLite2-ASN
- `internal/geo/lookup.go` possiede già la firma `Resolve(ip)` e il tipo di ritorno `GeoInfo`; devi sostituire il valore di ritorno azzerato con un'effettiva ricerca MMDB
- Il package `oschwald/maxminddb-golang` è già presente in `go.mod`
- Per Docker, aggiungi un init container `geoip-updater` che scarica il database al primo avvio

**Verifica del funzionamento:**
Connettiti all'honeypot SSH da una macchina con un IP pubblico (oppure utilizza una VPN). La mappa degli attacchi della dashboard dovrebbe mostrare un marker nella posizione geografica corretta. Il dettaglio dell'evento dovrebbe mostrare i campi paese, città e ASN.

### Sfida 3: Aggiungere l'Analisi delle Credenziali

**Cosa costruire:**
Un nuovo endpoint API `GET /api/stats/credentials/trending` che traccia quali combinazioni di nome utente/password stanno crescendo in frequenza nel tempo. Memorizza i conteggi dei tentativi di credenziali per ora in PostgreSQL e restituisci le 10 coppie con la crescita più rapida nelle ultime 24 ore.

**Perché è utile:**
Quando una nuova lista di credenziali entra nell'ecosistema delle botnet, gli honeypot rilevano un picco improvviso in combinazioni specifiche di username/password. Nel 2023, l'emergere della variante della botnet Kaiten ha introdotto un nuovo elenco predefinito di credenziali mirato ai DVR Hikvision (`admin/12345`, `admin/888888`). L'analisi delle credenziali di tendenza individua questi cambiamenti in tempo reale.

**Cosa imparerai:**
- Aggregazione di serie temporali in PostgreSQL con `date_trunc`
- Calcolo del tasso di crescita mediante window function

**Suggerimenti:**
- Aggiungi una vista materializzata o tabella `credential_hourly` che aggrega `(username, password, hour, count)` dalla tabella delle credenziali
- Tasso di crescita: confronta la finestra attuale di 6 ore con la precedente finestra di 6 ore, ordinando per rapporto
- Il tipo frontend `CredentialStats` esiste già; estendilo oppure aggiungi un nuovo tipo

**Verifica del funzionamento:**
Esegui Hydra contro l'honeypot SSH con una wordlist personalizzata. L'endpoint delle tendenze dovrebbe mostrare quelle specifiche coppie username/password salire nella classifica nel giro di pochi minuti.

## Sfide Intermedie

### Sfida 4: Aggiungere un Honeypot SMTP

**Cosa costruire:**
Un servizio SMTP in `internal/smtpd/` che implementi quanto basta dell'RFC 5321 per accettare il recapito di email. Gestisci i comandi EHLO, MAIL FROM, RCPT TO, DATA e QUIT. Cattura l'email completa, compresi intestazioni e corpo. Estrai indirizzi del mittente, indirizzi dei destinatari ed eventuali URL o allegati dal corpo del messaggio.

**Applicazione nel mondo reale:**
I server compromessi vengono frequentemente utilizzati come spam relay. Gli attaccanti effettuano scansioni alla ricerca di open relay SMTP, testano attacchi di credential stuffing contro i server di posta e utilizzano i mail server honeypot come bersagli di prova prima di lanciare campagne di phishing. Gli honeypot SMTP catturano template di email di phishing, allegati infetti da malware e l'infrastruttura alla base delle campagne di spam.

**Cosa imparerai:**
- Macchina a stati del protocollo SMTP (RFC 5321)
- Parsing MIME delle email per l'estrazione di allegati e URL
- Estrazione di IOC dal contenuto delle email (domini del mittente, URL incorporati, hash degli allegati)

**Approccio implementativo:**

1. **Creare il gestore del protocollo SMTP**
   - File da creare: `internal/smtpd/server.go`, `internal/smtpd/commands.go`
   - Implementa una macchina a stati basata su righe: INIT → EHLO → MAIL → RCPT → DATA → DONE

2. **Estrarre IOC dalle email catturate**
   - Estrai il dominio del mittente, tutti gli URL presenti nel corpo e gli hash SHA-256 degli allegati
   - Pubblica eventi IOC sul bus affinché il livello di intelligence li acquisisca

3. **Testare i casi limite:**
   - Cosa succede se il client invia STARTTLS? (rispondi con 454 e prosegui in chiaro)
   - Come gestisci i messaggi MIME multi-part con allegati annidati?

**Suggerimenti:**
- SMTP è basato su testo come FTP; il package `ftpd` è un ottimo riferimento strutturale
- La fase DATA termina con una riga contenente solo `.` (RFC 5321 Sezione 4.5.2)
- Cattura l'email grezza su disco (con limite a 5MB) e pubblica un evento file

**Crediti extra:**
Esegui il parsing delle intestazioni `Received:` per tracciare il percorso dell'email attraverso i relay. Estrai l'IP di origine e aggiungilo come IOC.

### Sfida 5: Threat Scoring delle Sessioni

**Cosa costruire:**
Un motore di scoring che assegna un punteggio di minaccia (0-100) a ciascuna sessione in base ai comandi eseguiti, alle tecniche MITRE rilevate, alla durata e ai tentativi di credenziali. Memorizza il punteggio nel record Session ed esponilo nell'API.

**Applicazione nel mondo reale:**
I team SOC che effettuano il triage dei dati degli honeypot devono stabilire le priorità su quali sessioni analizzare per prime. Una sessione in cui l'attaccante si è semplicemente connesso e disconnesso è meno interessante rispetto a una in cui ha eseguito comandi di discovery, scaricato tool e impostato la persistenza. Lo scoring automatizzato consente agli analisti di concentrarsi sulle sessioni di maggior valore.

**Cosa imparerai:**
- Algoritmi di scoring ponderato
- Aggiornamenti del punteggio in tempo reale tramite l'event bus

**Approccio implementativo:**

1. **Definire i pesi dello scoring**
   - Evento di autenticazione: +5 per tentativo (il brute force contribuisce pesantemente)
   - Comandi di discovery (uname, id, cat /etc/passwd): +10 ciascuno
   - Trasferimento di tool (wget, curl verso URL esterni): +20
   - Persistenza (crontab, systemd): +25
   - Tecnica MITRE rilevata: +15 per tecnica univoca
   - Durata della sessione superiore a 30 secondi: +10

2. **Integrare con il session tracker**
   - Aggiorna il punteggio in `session.Tracker` a ogni evento
   - Memorizza il punteggio finale quando viene chiamato `End()`
   - Aggiungi il parametro di query `min_score` a `GET /api/sessions`

3. **Aggiungere un indicatore nella dashboard**
   - Codifica a colori delle righe di sessione: verde (0-25), giallo (26-50), arancione (51-75), rosso (76-100)

**Suggerimenti:**
- Poni un limite massimo al punteggio a 100 per prevenire valori spropositati dovuti a lunghe sessioni di brute force
- `session.Tracker` dispone già di `AddTechnique()`; usa un pattern simile per `AddScore(points int)`
- Considera la normalizzazione dei punteggi di brute force: 100 tentativi di login non dovrebbero assegnare 500 punti

**Crediti extra:**
Addestra un semplice modello di regressione logistica su dati di sessione etichettati (contrassegna manualmente 50 sessioni come benigne/sospette/malevole) e confronta le sue classificazioni con lo scorer euristico.

### Sfida 6: Avvisi in Tempo Reale tramite Webhook

**Cosa costruire:**
Un sistema di notifica tramite webhook che invia richieste POST a URL configurati al verificarsi di condizioni specifiche: sessioni ad alta minaccia, rilevamento di brute force, tentativi di download di tool o nuove tecniche MITRE osservate.

**Applicazione nel mondo reale:**
Gli honeypot in produzione devono allertare i team di sicurezza in tempo reale. L'integrazione con Slack, PagerDuty o un SIEM tramite webhook trasforma l'honeypot da un raccoglitore passivo di dati in un sistema attivo di rilevamento. La piattaforma T-Pot di Deutsche Telekom invia avvisi dall'honeypot direttamente alla dashboard del loro SOC attraverso un meccanismo simile.

**Cosa imparerai:**
- Invio di webhook con logica di retry e backoff esponenziale
- Deduplicazione degli avvisi per prevenire allagamenti di notifiche
- Fan-out dall'event bus verso sistemi esterni

**Approccio implementativo:**

1. **Aggiungere la configurazione dei webhook**
   - File da creare: `internal/alert/webhook.go`, `internal/alert/rules.go`
   - Configurazione: URL, secret (per la firma HMAC), tipi di evento su cui attivarsi, periodo di cooldown

2. **Implementare le regole di avviso**
   - Effettua la sottoscrizione all'event bus con filtri per topic
   - Valuta le condizioni: soglia di brute force, rilevamento di tool transfer, punteggio di sessione superiore a N
   - Deduplica: stesso IP + stessa regola = un avviso per periodo di cooldown

3. **Inviare i webhook**
   - Invia tramite POST un payload JSON con dettagli dell'evento, contesto della sessione e mappature MITRE
   - Firma HMAC-SHA256 nell'header `X-Hive-Signature`
   - Riprova 3 volte con backoff esponenziale (1s, 4s, 16s)

**Suggerimenti:**
- Utilizza un canale bufferizzato e una goroutine dedicata per l'invio dei webhook, seguendo lo stesso pattern del processore degli eventi
- La firma HMAC segue lo stesso pattern delle firme dei webhook di GitHub
- Aggiungi un endpoint `GET /api/alerts` per mostrare gli avvisi recenti nella dashboard

**Crediti extra:**
Aggiungi un formattatore specifico per Slack che invia messaggi rich in formato block-kit con IP dell'attaccante, paese, comandi eseguiti e un link alla riproduzione della sessione.

## Sfide Avanzate

### Sfida 7: Distribuzione Multi-Sensore

**Cosa costruire:**
Supporto per l'esecuzione di più istanze di Hive che inviano report a un server centrale di aggregazione. Ciascun sensore si registra presso il server centrale, invia eventi tramite gRPC e il server centrale memorizza e correla i dati tra tutti i sensori.

**Perché è difficile:**
Questo trasforma Hive da uno strumento a singolo nodo in un sistema distribuito. Dovrai gestire la registrazione dei sensori, gli heartbeat, le garanzie di consegna degli eventi, la sincronizzazione dell'orologio per la correlazione tra sensori e una vista unificata della dashboard su tutti i sensori.

**Cosa imparerai:**
- Definizioni di servizi gRPC e streaming RPC
- Coordinamento di sistemi distribuiti (registrazione, heartbeat, rilevamento dei guasti)
- Correlazione degli attacchi tra più sensori

**Modifiche architetturali necessarie:**

```
  Sensor A (SSH, HTTP)      Sensor B (FTP, MySQL)     Sensor C (SMB, Redis)
       │                         │                          │
       └─── gRPC stream ────────┼──── gRPC stream ─────────┘
                                 │
                        ┌────────▼────────┐
                        │  Central Server  │
                        │  ┌────────────┐  │
                        │  │ Correlator │  │
                        │  │ Store      │  │
                        │  │ Dashboard  │  │
                        │  └────────────┘  │
                        └──────────────────┘
```

**Fasi di implementazione:**

1. **Fase di ricerca**
   - Approfondisci lo streaming bidirezionale di gRPC in Go
   - Comprendi i compromessi tra push (sensore → server centrale) e pull (server centrale → sensore)
   - Esamina come le architetture degli agenti Elastic Agent e Wazuh gestiscono il coordinamento multi-sensore

2. **Fase di progettazione**
   - Definisci i messaggi protobuf per la registrazione dei sensori, lo streaming degli eventi e l'heartbeat
   - Decidi la semantica di consegna degli eventi: at-least-once (con deduplicazione) oppure at-most-once
   - Pianifica il modo in cui la dashboard distingue gli eventi provenienti da sensori diversi

3. **Fase di implementazione**
   - Inizia con una modalità minima `hive agent` che inoltra gli eventi via gRPC
   - Aggiungi una modalità `hive central` che riceve, memorizza e serve la dashboard
   - Aggiungi la correlazione cross-sensore: lo stesso IP attaccante che colpisce più sensori all'interno di una finestra temporale

4. **Fase di test**
   - Esegui tre istanze locali su porte differenti
   - Simula un attaccante che le scansiona tutte e tre
   - Verifica che la dashboard centrale mostri una cronologia unificata dell'attaccante

**Attenzione a:**
- Disallineamento dell'orologio tra i sensori: usa il timestamp del server centrale per l'ordinamento, non quello del sensore
- Partizioni di rete: effettua il buffering locale degli eventi quando il server centrale non è raggiungibile e rieseguili alla riconnessione

**Risorse:**
- Tutorial gRPC Go ed esempi di streaming bidirezionale
- Documentazione dell'architettura dell'agente Wazuh per pattern di progettazione multi-sensore reali

### Sfida 8: Pipeline di Integrazione SIEM

**Cosa costruire:**
Esporta gli eventi dell'honeypot in formati compatibili con le principali piattaforme SIEM: Elastic Common Schema (ECS) per Elasticsearch, Common Event Format (CEF) per ArcSight/QRadar e syslog (RFC 5424) per qualsiasi raccoglitore syslog.

**Perché è difficile:**
Ciascun SIEM possiede un proprio schema di eventi, convenzioni di denominazione dei campi e requisiti di trasporto. ECS utilizza JSON annidato con nomi di campo specifici come `source.ip`, `event.category` e `threat.technique.id`. CEF impiega un'intestazione delimitata da pipe con estensioni chiave=valore. Eseguire correttamente la mappatura dei campi determina se gli eventi appariranno effettivamente in modo corretto nelle dashboard e nelle regole di correlazione del SIEM.

**Cosa imparerai:**
- Mappatura dei campi dell'Elastic Common Schema (ECS)
- Formattazione degli eventi in CEF e syslog
- Esportazione in streaming degli eventi tramite Redis Streams o Kafka

**Modifiche architetturali necessarie:**

```
  Event Bus ─── Processor ─── Store (PostgreSQL)
                    │
                    ├── Redis Stream (existing)
                    │
                    ├── ECS Formatter ──→ Elasticsearch
                    ├── CEF Formatter ──→ Syslog/QRadar
                    └── Syslog Formatter ──→ rsyslog/Splunk
```

**Fasi di implementazione:**

1. **Fase di ricerca**
   - Leggi la documentazione di riferimento dei campi ECS, concentrandoti sui set di campi `event.*`, `source.*`, `destination.*` e `threat.*`
   - Leggi la specifica CEF (ArcSight Common Event Format)
   - Comprendi il funzionamento dei dati strutturati di syslog (RFC 5424 SD-ELEMENT)

2. **Fase di progettazione**
   - Mappa ciascun `EventType` di Hive nelle categorie ECS e negli ID di classe di evento CEF
   - Scegli il trasporto: HTTP bulk API per Elasticsearch, UDP/TCP per syslog, Kafka per volumi elevati
   - Pianifica la configurazione: gli utenti devono poter selezionare il proprio SIEM e fornire i dettagli di connessione

3. **Fase di implementazione**
   - Crea `internal/export/ecs.go`, `internal/export/cef.go`, `internal/export/syslog.go`
   - Aggiungi un'interfaccia `Exporter` che il processore richiama dopo la persistenza
   - Rendi gli exporter configurabili nel file di configurazione YAML

4. **Fase di test**
   - Avvia un container Elasticsearch e verifica che gli eventi appaiano in Kibana
   - Invia CEF a un collector syslog e verifica il parsing dei campi
   - Test di carico: 1000 eventi/secondo non dovrebbero creare contropressione sul processore

**Attenzione a:**
- ECS richiede valori specifici per `event.kind`: `event` per eventi normali, `alert` per rilevamenti MITRE
- CEF ha un limite di 1023 byte per messaggio syslog; tronca `serviceData` se necessario
- L'API bulk di Elasticsearch restituisce lo stato per singolo elemento; gestisci i fallimenti parziali

**Risorse:**
- Documentazione di riferimento di Elastic Common Schema
- Guida per sviluppatori ArcSight CEF (HPE/Micro Focus)
- RFC 5424 (The Syslog Protocol)

## Sfide per Esperti

### Sfida 9: Rilevamento delle Anomalie Basato su ML

**Cosa costruire:**
Un livello di machine learning che rileva comportamenti anomali degli attaccanti apprendendo i pattern normali dai dati storici dell'honeypot. Addestra un autoencoder su vettori di feature di sessione (sequenze di comandi, pattern temporali, utilizzo dei protocolli) e contrassegna le sessioni che deviano significativamente dai pattern appresi.

**Prerequisiti:**
Dovresti aver completato prima la Sfida 5 (Threat Scoring delle Sessioni), poiché questa sfida si basa sull'estrazione di feature a livello di sessione.

**Cosa imparerai:**
- Feature engineering dai dati degli eventi di sicurezza
- Architettura ad autoencoder per il rilevamento delle anomalie
- Apprendimento continuo (online learning) da dati in streaming
- Integrazione di modelli ML in Python con un backend in Go

**Pianificazione di questa funzionalità:**

Prima di scrivere codice, rifletti su:
- In che modo questo influisce sulla latenza della pipeline di elaborazione degli eventi?
- Quali feature sono più discriminanti per il rilevamento delle anomalie di sessione?
- Come gestisci il cold start (nessun dato di training al primo deployment)?
- Cosa accade quando il modello subisce drift al mutare dei pattern di attacco?

**Architettura ad alto livello:**

```
  Event Processor
       │
       ├── Existing: GeoIP → MITRE → Store → Stream
       │
       └── New: Feature Extractor
                    │
              ┌─────▼──────┐
              │  Session    │
              │  Features   │
              │  (vector)   │
              └─────┬──────┘
                    │
           ┌────────▼────────┐
           │   ML Service    │
           │  (Python/ONNX)  │
           │                 │
           │  Autoencoder    │
           │  Reconstruction │
           │  Error → Score  │
           └────────┬────────┘
                    │
               anomaly_score
               stored on Session
```

**Fasi di implementazione:**

**Fase 1: Feature Engineering**
- Estrai feature per singola sessione: conteggio dei comandi, comandi univoci, durata della sessione, byte trasferiti, numero di tentativi di autenticazione, numero di tecniche MITRE univoche, tempo intercorso tra i comandi (media/deviazione standard), presenza di tool transfer, presenza di persistenza
- Normalizza le feature nell'intervallo [0, 1] utilizzando il min-max scaling derivato dalle statistiche dei dati di training

**Fase 2: Addestramento del Modello**
- Costruisci un autoencoder in PyTorch: encoder (input → 32 → 16 → 8), decoder (8 → 16 → 32 → input)
- Addestralo sulle sessioni completate della prima settimana di deployment
- Esporta in ONNX per l'inferenza

**Fase 3: Integrazione**
- Crea un microservizio Python che carica il modello ONNX ed espone un endpoint di scoring gRPC o HTTP
- Chiama l'endpoint di scoring dal processore di eventi Go al termine di una sessione
- Memorizza l'anomaly score nel record Session
- Aggiungi un campo `anomaly_score` all'API e alla dashboard

**Fase 4: Apprendimento Continuo**
- Riaddestra settimanalmente sui dati accumulati delle sessioni
- Monitora la distribuzione dell'errore di ricostruzione nel tempo per rilevare il model drift
- Genera un avviso quando il tasso di anomalie supera una determinata soglia (possibile nuovo pattern di attacco)

**Sfide note:**

1. **Problema del cold start**
   - Problema: Nessun dato di addestramento al primo deployment
   - Suggerimento: Inizia con lo scoring basato su regole (Sfida 5) e passa al ML dopo aver raccolto oltre 1000 sessioni. Distribuisci come punto di partenza un modello pre-addestrato basato sui dataset pubblicamente disponibili dell'honeypot Cowrie.

2. **Sbilanciamento delle classi (class imbalance)**
   - Problema: La maggior parte delle sessioni dell'honeypot sono simili tra loro (bot di brute force). Le sessioni realmente inedite sono rare.
   - Suggerimento: Gli autoencoder gestiscono naturalmente questa situazione poiché apprendono il pattern comune. Un elevato errore di ricostruzione significa "inusuale", non necessariamente "malevolo". Combina l'anomaly score con il conteggio dei rilevamenti MITRE per la classificazione finale.

3. **Feature drift**
   - Problema: I pattern di attacco mutano nel corso dei mesi. Un modello addestrato a gennaio potrebbe non rilevare nuove tecniche che compaiono a giugno.
   - Suggerimento: Monitora l'errore medio di ricostruzione per settimana. Se mostra una tendenza al rialzo, il modello necessita di riaddestramento. Implementa il riaddestramento automatico attivato dal rilevamento del drift.

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Estrarre almeno 10 feature per sessione
- [ ] Addestrare un autoencoder che raggiunga un errore medio di ricostruzione < 0,05 sulle sessioni normali
- [ ] Contrassegnare le sessioni con download di tool o persistenza come ad elevata anomalia (errore di ricostruzione > 2 deviazioni standard)
- [ ] Assegnare il punteggio a una sessione in < 10 ms (inferenza ONNX)
- [ ] Gestire il cold start in modo pulito (fallback allo scoring basato su regole)
- [ ] Eseguire il riaddestramento senza downtime (hot-swap del file del modello)

### Sfida 10: Ricostruzione del Playbook dell'Attaccante

**Cosa costruire:**
Un motore di analisi che ricostruisca l'intera narrativa dell'attacco a partire da una sessione: cosa stava cercando di ottenere l'attaccante, quali tecniche ha utilizzato e in quale ordine, e come la sua sessione si confronta con playbook di attacco noti (Mirai, Muhstik, XMRig dropper, Mozi, ecc.).

**Prerequisiti:**
Dovresti aver completato la Sfida 5 (Threat Scoring delle Sessioni) e avere il rilevamento MITRE funzionante. La familiarità con il framework MITRE ATT&CK a livello di tattica è essenziale.

**Cosa imparerai:**
- Modellazione delle sequenze di attacco e analisi delle catene di tattiche (tactic chain analysis)
- Pattern matching rispetto a playbook di threat actor noti
- Generazione automatizzata di report

**Pianificazione di questa funzionalità:**

Prima di scrivere codice, rifletti su:
- Come definisci un "playbook" in modo sufficientemente flessibile da corrispondere a comportamenti varianti?
- Quale granularità di corrispondenza è utile? Corrispondenza esatta dei comandi? Sequenza di tecniche? Ordine delle tattiche?
- Come gestisci le corrispondenze parziali (l'attaccante ha completato 3 passaggi su 5)?

**Architettura ad alto livello:**

```
  Completed Session
       │
       ▼
  ┌─────────────┐
  │  Technique   │
  │  Sequence    │
  │  Extractor   │
  └──────┬──────┘
         │
    [T1110, T1082, T1105, T1053.003]
         │
  ┌──────▼──────┐     ┌──────────────────┐
  │  Playbook   │◄────│  Known Playbooks │
  │  Matcher    │     │  (Mirai, Muhstik │
  │             │     │   XMRig, Mozi)   │
  └──────┬──────┘     └──────────────────┘
         │
  ┌──────▼──────┐
  │  Narrative  │
  │  Generator  │
  │             │
  │  "Attacker  │
  │   performed │
  │   brute     │
  │   force..." │
  └─────────────┘
```

**Fasi di implementazione:**

**Fase 1: Definizioni dei Playbook**
- Definisci 5-10 playbook di attacco noti come sequenze ordinate di ID di tecniche MITRE
- Mirai: T1110 → T1082 → T1105 → T1053.003 (brute force → discovery → download bot → cron persistence)
- XMRig dropper: T1110 → T1105 → T1496 (brute force → download miner → resource hijacking)
- Includi percorsi ramificati: gli attaccanti possono saltare passaggi o eseguirli in ordine sparso

**Fase 2: Corrispondenza delle Sequenze**
- Implementa il matching di sottosequenze con tolleranza per passaggi mancanti o riordinati
- Assegna un punteggio alle corrispondenze come percentuale di completamento del playbook
- Gestisci corrispondenze parziali multiple (una sessione potrebbe corrispondere al 60% a Mirai e al 40% a XMRig)

**Fase 3: Generazione della Narrativa**
- Report basato su template: "Questa sessione corrisponde al playbook [Mirai] con una confidenza del [80%]. L'attaccante ha eseguito l'autenticazione tramite brute force (T1110), discovery di sistema (T1082) e scaricato un payload da [url] (T1105). La sessione non ha raggiunto la fase di persistenza."
- Includi una vista cronologica: timestamp + comando + tecnica MITRE + tattica

**Fase 4: API e Dashboard**
- Aggiungi l'endpoint `GET /api/sessions/{id}/playbook`
- Aggiungi una scheda "Playbook" alla pagina di dettaglio della sessione che mostra il playbook corrispondente, il punteggio di confidenza, i passaggi completati/rimanenti e il riepilogo narrativo

**Sfide note:**

1. **Flessibilità della sequenza**
   - Problema: Gli attaccanti raramente seguono i playbook alla lettera. Possono eseguire comandi in ordini differenti, usare nomi di tool diversi per la stessa tecnica o saltare dei passaggi.
   - Suggerimento: Usa gli ID delle tecniche (non i comandi) per il matching e consenti interruzioni nella sequenza. Un approccio basato sulla "longest common subsequence" gestisce il riordinamento.

2. **Corrispondenze multiple**
   - Problema: Una sessione può corrispondere parzialmente a diversi playbook.
   - Suggerimento: Ordina in base alla percentuale di completamento e presenta le prime 3 corrispondenze.

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Definire almeno 5 playbook di attacco noti
- [ ] Associare le sessioni ai playbook indicando la percentuale di completamento
- [ ] Generare riepiloghi narrativi leggibili
- [ ] Visualizzare l'analisi del playbook nella dashboard
- [ ] Gestire le sessioni che non corrispondono ad alcun playbook noto (classificandole come "inedite")

## Sfide di Integrazione nel Mondo Reale

### Distribuzione su un VPS

**L'obiettivo:**
Eseguire Hive su un VPS esposto pubblicamente per catturare traffico reale di attaccanti.

**Cosa imparerai:**
- Deployment in produzione con Docker Compose
- Configurazione del firewall per le porte dell'honeypot
- Gestione dei log e monitoraggio dello spazio su disco

**Passaggi:**
1. Esegui il provisioning di un VPS da 5-10 $/mese (DigitalOcean, Vultr, Hetzner)
2. Configura il firewall: consenti l'accesso alle porte dell'honeypot (22, 80, 21, 445, 3306, 6379) da qualsiasi origine, limita la porta della dashboard (3000) esclusivamente al tuo indirizzo IP
3. Sposta il reale servizio SSH su una porta non standard (es. 22222) prima di avviare l'honeypot sulla porta 22
4. Esegui il deployment con `docker compose up -d`
5. Imposta la rotazione dei log e il monitoraggio del disco (i dati dell'honeypot crescono rapidamente)

**Checklist di produzione:**
- [ ] Servizio SSH reale spostato su una porta non standard con autenticazione esclusivamente a chiavi
- [ ] Dashboard accessibile solo dal tuo IP (regola di firewall)
- [ ] Backup automatizzato del database (cron job con pg_dump)
- [ ] Avvisi sull'utilizzo del disco (warning all'80%, critico al 90%)
- [ ] Rotazione dei log per i log dei container Docker
- [ ] Riavvio automatico in caso di crash (Docker restart policy: unless-stopped)

**Attenzione a:**
- NON eseguire l'honeypot sulla porta 22 mentre il tuo SSH reale è ancora attivo su di essa. Rimarrai chiuso fuori.
- Alcuni provider di VPS vietano gli honeypot nei propri Termini di Servizio (ToS). Verifica prima di procedere al deployment.
- Aspettati oltre 10.000 eventi al giorno su un IP pubblico. Pianifica lo storage di conseguenza.

### Integrare con OpenCTI

**L'obiettivo:**
Inviare i bundle STIX di Hive a un'istanza di OpenCTI per la correlazione della threat intelligence.

**Cosa ti servirà:**
- Un'istanza OpenCTI (deployment Docker o cloud)
- Token API di OpenCTI con permessi di scrittura
- Comprensione dei tipi di oggetti STIX 2.1

**Piano di implementazione:**
1. Configura un connettore OpenCTI che esporti periodicamente i bundle STIX tramite `GET /api/iocs/export/stix`
2. Invia tramite POST il bundle STIX all'endpoint di importazione STIX di OpenCTI
3. Mappa i punteggi di confidenza di Hive sulla scala di confidenza di OpenCTI
4. Crea etichette OpenCTI per i dati originati dall'honeypot

**Attenzione a:**
- OpenCTI deduplica in base allo STIX ID. Hive genera identificatori casuali UUID v4, quindi riesportare la stessa lista di IOC produce un nuovo bundle con ID diversi — OpenCTI creerà oggetti duplicati. Per una vera idempotenza, passa il generatore STIX a UUID v5 con un namespace fisso inizializzato con `ioc.Type + ":" + ioc.Value`, ottenendo così lo stesso ID per il medesimo IOC a ogni esportazione.
- I bundle di grandi dimensioni (oltre 1000 indicatori) dovrebbero essere suddivisi in batch da 100 per l'API di importazione di OpenCTI.
- Rate limit sull'API di OpenCTI: inserisci un ritardo di 1 secondo tra l'invio dei vari batch.

## Sfide sulle Prestazioni

### Gestire 10.000 Eventi al Secondo

**L'obiettivo:**
Fare in modo che Hive sostenga 10.000 eventi al secondo derivanti da attacchi di brute force concorrenti su tutti e sei i servizi, senza scartare eventi né degradare la reattività verso l'attaccante.

**Collo di bottiglia attuale:**
Il pool di worker del processore di eventi (4 goroutine) gestisce la ricerca GeoIP, il rilevamento MITRE, fino a cinque scritture sequenziali in PostgreSQL (eventi, rilevamenti, credenziali, upsert dell'attaccante, upsert degli IOC) e la pubblicazione su Redis per ciascun evento. A circa 2–5 ms per evento, a seconda di quanti percorsi di scrittura vengono attivati, il limite massimo è di circa 800–2.000 eventi/secondo su tutti i worker.

**Approcci di ottimizzazione:**

**Approccio 1: Inserimenti in batch nel database**
- Come: Esegui il buffering degli eventi per 100 ms o 100 eventi (a seconda di quale condizione si verifichi per prima), quindi esegui una INSERT in batch
- Guadagno: PostgreSQL gestisce gli inserimenti in batch in modo da 10 a 20 volte più efficiente rispetto agli inserimenti singoli
- Compromesso: Fino a 100 ms di ritardo prima che gli eventi compaiano nel database

**Approccio 2: Incrementare la dimensione del pool di worker**
- Come: Rendi configurabile il numero di worker, impostando come valore predefinito `runtime.NumCPU()`
- Guadagno: Scalabilità lineare fino al limite del connection pool del database
- Compromesso: Più connessioni al database, potenziale rischio di esaurimento delle connessioni

**Approccio 3: GeoIP asincrono con cache**
- Come: Salva in cache i risultati GeoIP in una cache LRU (la maggior parte degli attaccanti invia centinaia di eventi dallo stesso IP)
- Guadagno: Elimina le ricerche MMDB ridondanti per gli IP ripetuti
- Compromesso: Utilizzo di memoria della cache (~100 byte per voce, trascurabile per 10.000 IP)

**Esegui il benchmark:**
```bash
for i in $(seq 1 100); do
    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
        root@localhost -p 2222 &
done
```

Metriche target:
- Latenza di elaborazione degli eventi: < 5ms p99
- Lag WebSocket della dashboard: < 100ms dall'evento al browser
- Zero eventi scartati nella pipeline del processore

## Sfide sulla Sicurezza

### Rendere l'Honeypot Resistente alle Evasioni (Escape)

**Cosa implementare:**
Misure di difesa in profondità che impediscano a un attaccante di utilizzare l'honeypot come punto di pivot o di scoprire che si tratta di un honeypot.

**Modello di minaccia:**
Questo protegge da:
- Attaccanti che sfruttano l'accesso di rete in uscita dell'honeypot per scansionare altri host
- Fingerprinting dell'honeypot tramite analisi temporale o comportamenti di sistema mancanti
- Esaurimento delle risorse (fork bomb, flood di memoria)

**Implementazione:**
1. Isolamento di rete: configura le regole di rete di Docker per bloccare le connessioni in uscita dal container di backend (ad eccezione di quelle verso PostgreSQL e Redis)
2. Limiti di risorse: imposta limiti di CPU e memoria in `compose.yml` per il container di backend
3. Anti-fingerprinting: aggiungi ritardi casuali (10-50 ms) alle risposte ai comandi per prevenire il rilevamento dell'honeypot basato sul timing
4. Limiti di connessione: poni un tetto massimo alle connessioni concorrenti per servizio (es. 50 SSH, 100 HTTP)

**Testare la sicurezza:**
- Prova a eseguire `ping` o `curl` verso un host esterno dalla shell dell'honeypot SSH (dovrebbe sembrare funzionante ma non produrre alcun traffico di rete reale)
- Esegui una fork bomb nella shell SSH e verifica che il container sopravviva
- Utilizza strumenti di tipo `honeypot-detector` (shodan honeyscore, honeypot-hunter) contro l'istanza in esecuzione

## Studiare Implementazioni Reali

Confronta la tua implementazione con strumenti di produzione:
- **Cowrie** (honeypot SSH/Telnet in Python) — Studia la loro profondità di emulazione dei comandi e il formato di riproduzione delle sessioni
- **T-Pot** (piattaforma multi-honeypot di Deutsche Telekom) — Studia la loro orchestrazione Docker e l'integrazione con ELK
- **HoneyDB** (aggregazione dati honeypot della community) — Studia il design delle loro API per condividere la threat intelligence tra sensori
- **Dionaea** (honeypot multiprotocollo basato su C) — Studia il loro approccio alla cattura di binari malware e allo sfruttamento di SMB

Leggi il loro codice, comprendi i loro compromessi, fai tue le loro buone idee.

## Completamento delle Sfide

Traccia i tuoi progressi:

- [ ] Facile 1: Honeypot Telnet
- [ ] Facile 2: Arricchimento GeoIP
- [ ] Facile 3: Analisi delle Credenziali
- [ ] Intermedia 4: Honeypot SMTP
- [ ] Intermedia 5: Threat Scoring delle Sessioni
- [ ] Intermedia 6: Avvisi tramite Webhook
- [ ] Avanzata 7: Distribuzione Multi-Sensore
- [ ] Avanzata 8: Pipeline di Integrazione SIEM
- [ ] Esperto 9: Rilevamento delle Anomalie Basato su ML
- [ ] Esperto 10: Ricostruzione del Playbook dell'Attaccante
