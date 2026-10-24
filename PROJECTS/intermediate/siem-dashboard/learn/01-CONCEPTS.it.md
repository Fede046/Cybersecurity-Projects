# Concetti di sicurezza fondamentali

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci questa dashboard SIEM. Non sono solo definizioni. Approfondiremo perché sono importanti, come funzionano davvero gli attacchi e come questo progetto implementa le difese.

## Normalizzazione dei log

### Che cos'è

Strumenti di sicurezza diversi producono log in formati completamente diversi. Un log di firewall ha campi come `action`, `protocol` e `bytes_sent`. Un log di IDS ha `signature_id`, `classification` e `priority`. Un log di autenticazione ha `auth_method`, `result` e `failure_reason`. Allo stato grezzo, questi log non sono confrontabili.

La normalizzazione dei log trasforma tutti questi formati diversi in uno schema comune, così puoi interrogarli, correlarli e analizzarli insieme. In questo progetto, il normalizer (`app/engine/normalizer.py`) prende gli eventi grezzi da sette tipi di sorgente e li mappa in un documento `LogEvent` unificato con campi coerenti come `source_ip`, `dest_ip`, `severity`, `event_type` e un dict `normalized` per gli extra specifici della sorgente.

### Perché è importante

Senza normalizzazione, un analista SOC che indaga su un IP sospetto dovrebbe cercare manualmente nell'interfaccia di ciascuno strumento separatamente. Con dati normalizzati, una singola pivot query restituisce ogni connessione del firewall, ogni tentativo di autenticazione, ogni query DNS e ogni esecuzione di processo sugli endpoint provenienti da quell'IP.

La violazione di Target del 2013 è un caso da manuale. Il loro IDS FireEye ha generato alert sul beaconing di un malware, ma il team non è riuscito a correlare quegli alert con i log dei sistemi POS che mostravano l'esfiltrazione dei dati. Gli strumenti esistevano in silos. Un SIEM funzionante con una normalizzazione adeguata avrebbe collegato i puntini.

### Come funziona

Il normalizer usa un pattern a registry. Ogni tipo di sorgente registra una funzione di normalizzazione tramite un decorator:

```
Source Type     Normalizer Function         Extracted Fields
─────────────  ─────────────────────────   ──────────────────────────
firewall       _normalize_firewall()       action, protocol, bytes_sent/received
ids            _normalize_ids()            signature_id, signature_name, priority
auth           _normalize_auth()           auth_method, result, failure_reason
endpoint       _normalize_endpoint()       process_name, command_line, file_path
dns            _normalize_dns()            query, query_type, response_code
proxy          _normalize_proxy()          url, method, status_code, user_agent
generic        _normalize_generic()        message (fallback)
```

Il dispatch è semplice. `normalize()` estrae i campi comuni (indirizzi IP, porte, timestamp, hostname), poi chiama il normalizer specifico della sorgente per estrarre i campi unici di quel tipo di log. Entrambi vengono uniti in un unico documento.

### Insidie comuni

**Errore 1: dare per scontato che tutte le sorgenti abbiano gli stessi campi**
```python
# Bad - will throw KeyError on auth logs
def process_event(event):
    bytes = event["bytes_sent"]  # Only firewall events have this

# Good - use .get() with defaults and check source_type
def process_event(event):
    if event.get("source_type") == "firewall":
        bytes = event.get("bytes_sent", 0)
```

**Errore 2: perdere i dati grezzi**

Il normalizer di questo progetto conserva l'evento originale nel DictField `raw` di `LogEvent`. Questo è fondamentale per la forensics. La normalizzazione per definizione perde informazioni, e durante un'indagine avrai spesso bisogno della riga di log originale.

## Correlazione degli eventi

### Che cos'è

La correlazione è il processo che collega più eventi singoli in un rilevamento di livello più alto. Un singolo login fallito è rumore. Venti login falliti dallo stesso IP in 60 secondi sono un attacco brute force. Un brute force seguito da un login riuscito è una compromissione.

Questo progetto implementa tre strategie di correlazione in `app/engine/correlation.py`:

Le regole **threshold** scattano quando il conteggio degli eventi corrispondenti per una chiave di raggruppamento supera un limite all'interno di una finestra temporale. Esempio: più di 10 eventi `login_failure` dallo stesso `source_ip` in 300 secondi.

Le regole **sequence** scattano quando un insieme ordinato specifico di tipi di evento si verifica all'interno di una finestra. Esempio: `login_failure` (count >= 5) seguito da `login_success` dallo stesso `source_ip`.

Le regole **aggregation** scattano quando il numero di valori distinti di un campo supera una soglia. Esempio: un `source_ip` si connette a più di 20 valori distinti di `dest_ip` in 60 secondi (comportamento da port scan).

### Perché è importante

Senza correlazione, un SIEM è solo un motore di ricerca di log. La correlazione è ciò che trasforma i dati in rilevamenti. La differenza tra un SIEM che genera alert utili e uno che sommerge gli analisti nel rumore si riduce alla qualità delle sue regole di correlazione.

L'attacco a SolarWinds del 2020 è passato inosservato per mesi anche perché gli strumenti di rilevamento esistenti guardavano i singoli eventi in modo isolato. Gli attaccanti usavano credenziali valide, protocolli standard e strumenti legittimi. Solo la correlazione di più segnali deboli nel tempo (comportamento insolito di un service account + nuova federation trust + query DNS anomale) avrebbe potuto far emergere l'intrusione prima.

### Come funziona

Il motore di correlazione gira come un thread daemon che consuma eventi da uno Stream Redis:

```
Redis Stream (siem:logs)
        │
        ▼
┌───────────────────┐
│ CorrelationEngine │  Daemon thread, reads via XREADGROUP
│   ._run() loop    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  _process_event() │  Iterates all enabled rules
└────────┬──────────┘
         │
    ┌────┴────┬──────────┐
    ▼         ▼          ▼
Threshold  Sequence  Aggregation
evaluator  evaluator  evaluator
    │         │          │
    └────┬────┘──────────┘
         │
    Fire? ──Yes──▶ Alert.create_from_rule()
                         │
                         ▼
                  Redis Stream (siem:alerts)
                         │
                         ▼
                   SSE to browser
```

Ogni evaluator usa un oggetto `CorrelationState` che mantiene in memoria le sliding window. Lo stato è thread-safe (usa `threading.Lock`) e tiene traccia di finestre di eventi per regola e per chiave di raggruppamento. Durante la valutazione, elimina le voci scadute in base a `window_seconds` e controlla se la condizione di attivazione è soddisfatta.

Un meccanismo di cooldown (`CORRELATION_COOLDOWN_SECONDS`, default 300) impedisce alla stessa regola di scattare ripetutamente per la stessa chiave di raggruppamento. Senza di esso, un brute force prolungato genererebbe centinaia di alert identici.

### Attacchi comuni che rileva

1. **Brute Force (T1110.001)** - Regola threshold: conta gli eventi `login_failure` raggruppati per `source_ip`, scatta quando il conteggio è > 10 in 300 secondi
2. **Lateral Movement (T1021.004)** - Regola sequence: `login_success` da un IP esterno seguito da connessioni SSH interne dall'host compromesso
3. **Port Scanning (T1046)** - Regola aggregation: conta i valori distinti di `dest_ip` per `source_ip`, scatta quando ci sono > 20 destinazioni uniche in 60 secondi
4. **DNS Tunneling (T1048.003)** - Regola threshold su query TXT ad alta frequenza verso un singolo dominio, oppure aggregation sui sottodomini distinti interrogati

### Strategie di difesa

Il motore di correlazione stesso è un meccanismo di difesa. Ma ha bisogno di regole ben calibrate. Questo progetto include playbook di scenari che dimostrano gli attacchi e le regole che li rilevano. Il playbook `brute_force_lateral.yml`, ad esempio, simula 20 login SSH falliti seguiti da un login riuscito e da movimento laterale. Una regola threshold su `login_failure` raggruppata per `source_ip` con soglia 10 e finestra 300 intercetterebbe la fase di brute force.

## Classificazione della severità

### Che cos'è

Non tutti gli eventi di sicurezza sono ugualmente urgenti. Un firewall che consente una connessione HTTPS in uscita è informativo. Un processo che esegue `cat /etc/shadow` è ad alta severità. La classificazione della severità assegna un livello di priorità a ogni evento, così gli analisti si concentrano su ciò che conta.

Questo progetto classifica gli eventi in cinque livelli: `critical`, `high`, `medium`, `low` e `info`. Il classificatore in `app/engine/severity.py` usa due approcci: lookup per tipo di evento (veloce, per i tipi di evento noti) e pattern matching con regex (flessibile, per la classificazione basata sul contenuto).

### Come funziona

Il classificatore controlla prima i tipi di evento rispetto a dei frozen set:

```
HIGH_SEVERITY_EVENT_TYPES:  privilege_escalation, data_exfiltration,
                            c2_communication, reverse_shell

MEDIUM_SEVERITY_EVENT_TYPES: login_failure, port_scan,
                             firewall_deny, ids_alert
```

Se il tipo di evento non corrisponde, si passa al matching con regex. Il classificatore concatena i campi di testo rilevanti (event_type, message, campi normalizzati come signature_name e command_line) in una stringa ricercabile, poi esegue i pattern regex da critical giù fino a low. Vince il primo match.

I pattern critical includono cose come `privilege.?escalat`, `ransomware` e `c2.?beacon`. I pattern high includono `brute.?force`, `lateral.?movement` e `reverse.?shell`. Le regex usano `re.IGNORECASE` per il matching case insensitive e `?` per i separatori opzionali (così sia "privilege escalation" sia "privilege_escalation" corrispondono).

### Insidie comuni

**Errore: classificare tutto come alta severità**

Questo porta all'alert fatigue. Se tutto è critico, niente lo è. Il classificatore di questo progetto usa `info` come default per gli eventi che non corrispondono a nessun pattern. È voluto. Un tipo di evento sconosciuto senza parole chiave sospette è probabilmente di routine. Meglio perdere un rilevamento a bassa confidenza che seppellire gli alert reali nel rumore.

## Sicurezza di autenticazione e autorizzazione

### Che cos'è

Anche il SIEM stesso deve essere messo in sicurezza. Se un attaccante compromette il SIEM, può sopprimere gli alert, manipolare le regole e coprire le proprie tracce. Questo progetto implementa un'autenticazione basata su JWT con hashing delle password Argon2id e un controllo degli accessi basato sui ruoli.

### Come funziona

Il sistema di autenticazione in `app/core/auth.py` usa `pwdlib` con Argon2id, il vincitore della Password Hashing Competition del 2015. Argon2id è memory-hard, il che significa che resiste agli attacchi di cracking basati su GPU che rendono vulnerabili bcrypt e gli hash basati su SHA.

Un dettaglio cruciale: la funzione `verify_password_timing_safe()` fornisce un comportamento a tempo costante per i lookup falliti. Quando un tentativo di login usa uno username che non esiste, il sistema esegue comunque un confronto di hash con un hash fittizio (`DUMMY_HASH`). Questo previene gli attacchi di timing che potrebbero enumerare gli username validi misurando le differenze nei tempi di risposta.

```
Login attempt with valid username:
  1. Look up user → found
  2. Verify password against stored hash → ~150ms (Argon2id)
  3. Return result

Login attempt with invalid username:
  1. Look up user → not found
  2. Verify password against DUMMY_HASH → ~150ms (same timing)
  3. Return "Invalid username or password"
```

Senza questo, un attaccante potrebbe distinguere "l'utente non esiste" (risposta veloce) da "password sbagliata" (risposta lenta, perché Argon2id è intenzionalmente costoso).

### Modello RBAC

Due ruoli: `analyst` (default) e `admin`. Il decorator `@endpoint(roles=ADMIN)` sulle route di amministrazione controlla i claim del JWT. Gli admin possono gestire gli utenti, cambiare i ruoli e disattivare gli account. Il sistema impedisce di retrocedere l'ultimo admin (controllo `User.count_admins()` in `admin_ctrl.py`).

## Streaming degli eventi in tempo reale

### Che cos'è

Un SIEM che mostra i dati solo al refresh della pagina è quasi inutile durante un incidente in corso. Gli analisti devono vedere gli eventi mentre accadono. Questo progetto usa Redis Streams per la consegna interna degli eventi e Server-Sent Events (SSE) per gli aggiornamenti nel browser.

### Come funziona

Quando un evento di log viene acquisito o viene creato un alert, viene pubblicato su uno Stream Redis tramite `XADD`. Il motore di correlazione legge dallo stream dei log usando `XREADGROUP` con i consumer group (consegna affidabile con acknowledgment). Gli endpoint SSE usano `XREAD` per seguire lo stream ed emettere gli eventi verso i browser connessi.

```
Log Ingestion ──XADD──▶ Redis Stream (siem:logs)
                              │
              ┌───────────────┼───────────────┐
              │               │               │
         XREADGROUP      XREADGROUP        XREAD
              │               │               │
     Correlation         (Future          SSE Generator
       Engine           consumers)       ──▶ Browser
              │
         Alert Created ──XADD──▶ Redis Stream (siem:alerts)
                                       │
                                    XREAD
                                       │
                                  SSE Generator
                                  ──▶ Browser
```

Il generatore SSE in `app/core/streaming.py` invia commenti di keepalive (`: keepalive\n\n`) quando non arrivano eventi entro il timeout di blocco. Questo previene i timeout dei proxy e permette al browser di rilevare le connessioni cadute. La configurazione Nginx disabilita il buffering e imposta `proxy_read_timeout 3600s` per gli endpoint SSE.

## Come si collegano questi concetti

```
Raw Log Event
    │
    ▼
Normalization (common schema)
    │
    ▼
Severity Classification (priority label)
    │
    ▼
Persistence (MongoDB) + Publish (Redis Stream)
    │
    ├──▶ SSE to browser (real-time log viewer)
    │
    ▼
Correlation Engine (pattern matching across events)
    │
    ▼
Alert Generation
    │
    ├──▶ SSE to browser (real-time alert feed)
    │
    ▼
Alert Lifecycle (new → acknowledged → investigating → resolved)
    │
    ▼
Forensic Investigation (pivot queries, matched event drill-down)
```

Ogni concetto si basa sul precedente. Senza normalizzazione, la correlazione non può confrontare eventi di sorgenti diverse. Senza classificazione della severità, gli analisti non possono stabilire le priorità. Senza streaming, i tempi di risposta ne risentono. Senza correlazione, i singoli eventi sono rumore privo di significato.

## Standard e framework del settore

### OWASP

Sebbene OWASP si concentri sulla sicurezza delle applicazioni, diverse categorie si applicano al SIEM stesso:
- **A01:2021 Broken Access Control** - Il sistema RBAC con `@endpoint(roles=ADMIN)` impedisce l'accesso non autorizzato alle funzioni di amministrazione. L'helper `_prevent_self_action()` impedisce agli admin di disattivare il proprio account.
- **A02:2021 Cryptographic Failures** - Argon2id per l'hashing delle password, JWT HS256 con scadenza configurabile. Il default di `SECRET_KEY`, "change-me-in-production", è volutamente ovvio.
- **A07:2021 Identification and Authentication Failures** - Verifica delle password timing-safe, rate limiting sugli endpoint di autenticazione (10/minuto), supporto alla disattivazione degli account.

### MITRE ATT&CK

I playbook degli scenari corrispondono direttamente a tecniche MITRE:

- **T1110.001 (Brute Force: Password Guessing)** - `brute_force_lateral.yml` simula 20 tentativi di password SSH su username comuni
- **T1021.004 (Remote Services: SSH)** - Movimento laterale tramite il riuso di chiavi SSH dopo la compromissione iniziale
- **T1048.003 (Exfiltration Over Alternative Protocol: DNS)** - `data_exfiltration.yml` simula il DNS tunneling con query TXT ad alta entropia
- **T1059.001 (Command and Scripting Interpreter: PowerShell)** - `phishing_c2.yml` include l'esecuzione di un download cradle PowerShell codificato
- **T1068 (Exploitation for Privilege Escalation)** - `privilege_escalation.yml` simula la compilazione e l'esecuzione di un exploit del kernel
- **T1071.001 (Application Layer Protocol: Web)** - Beaconing C2 su HTTPS a intervalli regolari

Le regole di correlazione memorizzano i campi `mitre_tactic` e `mitre_technique`, che si propagano agli alert generati per dare contesto all'analista.

### CWE

Common weakness enumeration rilevanti per ciò che questo progetto insegna:
- **CWE-778 (Insufficient Logging)** - L'intero progetto mira a risolvere questo problema. Sette tipi di sorgente con normalizzazione specifica per formato.
- **CWE-223 (Omission of Security-Relevant Information)** - Il normalizer conserva i dati grezzi dell'evento accanto ai campi normalizzati.
- **CWE-779 (Logging of Excessive Data)** - Redis Streams usa `maxlen` con trimming approssimativo (`STREAM_MAXLEN = 10000`) per evitare una crescita illimitata.
- **CWE-307 (Improper Restriction of Excessive Authentication Attempts)** - Rate limiting sugli endpoint di autenticazione più regole di correlazione per il rilevamento del brute force.

## Esempi reali

### Caso di studio 1: la violazione di Target Corporation (2013)

La rete di Target è stata violata tramite un'email di phishing inviata a un fornitore di impianti HVAC. Gli attaccanti si sono mossi lateralmente verso i sistemi POS e hanno installato un malware RAM-scraping. L'IDS FireEye di Target ha rilevato il malware e generato alert, ma il team SOC non è riuscito a correlare quegli alert con la catena d'attacco più ampia.

È esattamente il tipo di fallimento che un SIEM con una correlazione adeguata previene. Una regola sequence che cercasse "accesso esterno → furto di credenziali → movimento laterale → accesso ai dati" avrebbe collegato i puntini. Il playbook di scenario `brute_force_lateral.yml` di questo progetto simula una kill chain simile: brute force esterno → raccolta delle credenziali → movimento laterale via SSH → dump del database.

### Caso di studio 2: il DNS tunneling nella campagna APT34/OilRig

APT34 (noto anche come OilRig) ha usato il DNS tunneling per esfiltrare dati da reti compromesse. Codificava i dati rubati nei sottodomini delle query DNS, inviandoli a nameserver controllati dall'attaccante. La tecnica funziona perché la maggior parte dei firewall consente il traffico DNS in uscita senza un'ispezione approfondita.

Il playbook `data_exfiltration.yml` simula esattamente questa tecnica. Genera query TXT ad alta entropia verso sottodomini come `4d5a90000300000004000000ffff0000b800.data.exfil-tunnel.tk`. Una regola di correlazione threshold sulla frequenza delle query DNS, oppure una regola aggregation sui sottodomini distinti per dominio di destinazione, rileverebbe questa attività.

### Caso di studio 3: la catena di consegna Emotet/TrickBot

Le campagne di TA542 seguono uno schema coerente: email di spear-phishing → documento con macro → download cradle PowerShell → payload di secondo stadio → beaconing C2. Il playbook `phishing_c2.yml` riproduce questa catena evento per evento, dal log proxy iniziale del clic sul link di phishing fino ai battiti periodici del beacon C2.

Questo dimostra perché la correlazione per sequenza è importante. Ogni singolo evento della catena potrebbe essere benigno. Scaricare un file Excel è normale. Eseguire PowerShell è normale. L'HTTPS in uscita è normale. Ma la sequenza Excel → cmd.exe → PowerShell → download → beaconing C2 non è normale, e una regola sequence può intercettarla.

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Perché il normalizer conserva i dati grezzi dell'evento oltre ai campi normalizzati? Quale scenario d'indagine richiederebbe i dati grezzi?
2. Qual è la differenza tra una regola threshold e una regola aggregation? Fai un esempio di attacco che ciascuna rileverebbe e l'altra no.
3. Perché `verify_password_timing_safe()` calcola l'hash rispetto a un valore fittizio quando l'utente non esiste? Quale fuga di informazioni previene?
4. Se il cooldown del motore di correlazione è impostato a 300 secondi, cosa succede durante un brute force prolungato di 10 minuti? Quanti alert scatterebbero?
5. Perché il generatore SSE invia commenti di keepalive? Cosa succederebbe in un deployment di produzione dietro Nginx senza di essi?

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. I dettagli dell'implementazione avranno più senso una volta assimilati questi fondamenti.

## Letture di approfondimento

**Essenziali:**
- [MITRE ATT&CK Framework](https://attack.mitre.org/) - La tassonomia attorno a cui sono costruiti gli scenari e le regole di questo progetto. Inizia dalla matrice "Enterprise".
- [Documentazione di Redis Streams](https://redis.io/docs/data-types/streams/) - Capire XADD, XREADGROUP, i consumer group e l'acknowledgment è fondamentale per comprendere il livello di streaming.

**Approfondimenti:**
- [Sigma Rules](https://github.com/SigmaHQ/sigma) - Uno standard aperto per le regole di rilevamento dei SIEM. La Sfida 6 nel documento delle sfide suggerisce di implementare il supporto alle regole Sigma.
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) - Perché è stato scelto Argon2id invece di bcrypt, e la calibrazione raccomandata dei parametri.

**Contesto storico:**
- L'APT1 Report di Mandiant (2013) - Uno dei primi report pubblici di threat intelligence, e un ottimo esempio del tipo di workflow d'indagine che un SIEM rende possibile.
- NIST SP 800-92 (Guide to Computer Security Log Management) - Il documento fondamentale sul perché e sul come raccogliere, conservare e analizzare i log di sicurezza.
