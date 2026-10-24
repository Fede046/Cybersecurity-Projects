<!-- © AngelaMos | 2026 | 01-CONCEPTS.md -->

# Concetti del C2 Beacon

Questo documento copre le idee fondamentali alla base dell'infrastruttura di Command and Control (C2), come il nostro progetto le implementa, e come si mappano sulla sicurezza offensiva e difensiva del mondo reale. Il tempo di lettura è circa 15-20 minuti. Alla fine dovresti capire perché il C2 funziona nel modo in cui funziona, cosa fa ogni comando del nostro beacon dal punto di vista di MITRE ATT&CK, e come i difensori individuano esattamente questo tipo di traffico.

---

## Architettura di Command and Control (C2)

### Che cos'è

Immagina questo scenario. Un attaccante è già entrato in una rete aziendale, magari tramite un'email di phishing, magari tramite un'appliance VPN senza patch. Ha del codice in esecuzione su una macchina target. E adesso? Ha bisogno di un modo per dire a quel codice cosa fare, e ha bisogno di ricevere indietro i risultati.

È questo che il C2 risolve. Command and Control è il canale di comunicazione tra un attaccante e i sistemi compromessi. L'attaccante esegue un **server** (a volte chiamato team server o listener) su un'infrastruttura che controlla. La macchina compromessa esegue un **implant** (chiamato anche beacon, agent o payload) che raggiunge quel server, chiede istruzioni, le esegue e rimanda indietro i risultati.

In termini concreti:

1. L'attaccante compromette una macchina e distribuisce un binario o uno script beacon.
2. Il beacon avvia una connessione in uscita verso il server C2.
3. Il server accoda i comandi per il beacon (esegui questo comando shell, prendi quel file, fai uno screenshot).
4. Il beacon preleva i comandi accodati, li esegue localmente e rimanda indietro i risultati sullo stesso canale.
5. L'operatore vede i risultati in una dashboard o in un terminale.

Questa fase di un'intrusione, la fase C2, è una delle più critiche. Senza di essa, un attaccante ha un exploit usa e getta. Con essa, ha accesso persistente e interattivo al target. La Cyber Kill Chain di Lockheed Martin colloca il C2 come passo 6 di 7. MITRE ATT&CK dedica un'intera categoria di tattiche a essa (TA0011). Ogni grande violazione di cui hai letto, da SolarWinds (SUNBURST) all'incidente ransomware di Colonial Pipeline, ha coinvolto comunicazioni C2 prolungate.

### Il modello beacon/server

Una domanda naturale: perché il beacon chiama il server? Perché non far connettere il server in entrata verso il target?

La risposta sono i firewall e il NAT. Le reti aziendali bloccano quasi universalmente le connessioni in entrata non sollecitate. Le workstation stanno dietro gateway NAT senza indirizzi IP pubblici. Un server che prova a connettersi alla porta 4444 su una workstation interna verrebbe scartato immediatamente al perimetro.

Ma il traffico in uscita? Quello scorre liberamente nella maggior parte degli ambienti. I dipendenti navigano sul web, chiamano API, guardano video in streaming. Un beacon che fa richieste HTTPS verso quello che sembra un dominio legittimo si confonde perfettamente in quel traffico. Dal punto di vista del firewall, la connessione WebSocket in uscita del beacon appare simile a qualsiasi altro traffico di applicazioni web.

Cobalt Strike, il framework C2 commerciale per red team più usato, ha reso popolare questo modello con il suo concetto di "beacon". Un beacon di Cobalt Strike dorme per un intervallo configurato (default 60 secondi), si sveglia, contatta il team server tramite HTTP/HTTPS, scarica eventuali comandi accodati, li esegue e ripubblica i risultati. Il nome "beacon" deriva da questo schema di check-in periodico, come un faro che lampeggia a intervalli regolari.

Il nostro progetto segue questo stesso modello. Il beacon (`beacon/beacon.py`) avvia una connessione WebSocket verso il server (`backend/app/beacon/router.py`). Il server non prova mai a raggiungere il beacon. È sempre il beacon a chiamare casa.

### Perché gli attaccanti usano il C2

Cinque motivi fondamentali:

**Accesso persistente dopo la compromissione iniziale.** L'exploit iniziale potrebbe essere un evento una tantum, un link di phishing cliccato, una vulnerabilità sfruttata. Il C2 dà all'attaccante un canale duraturo che sopravvive al fatto che il vettore iniziale venga patchato o scoperto. Combinato con i meccanismi di persistence (cron job, chiavi di registro, scheduled task), il beacon torna anche dopo i reboot.

**Esecuzione di comandi senza sessioni interattive.** Un attaccante non ha bisogno di mantenere aperta una sessione SSH o RDP. Il beacon opera in modo asincrono. L'operatore accoda un comando, se ne va, e controlla i risultati ore dopo. Questo è particolarmente utile nelle operazioni di lunga durata in cui un'interazione costante sarebbe sospetta.

**Exfiltration di dati attraverso il canale C2.** I dati rubati escono dalla rete attraverso lo stesso canale cifrato su cui arrivano i comandi. Nessun server FTP separato, nessun upload cloud sospetto. I dati escono dentro il protocollo C2, spesso suddivisi in chunk e codificati per evitare il rilevamento.

**Coordinamento del lateral movement.** Una volta dentro una rete, gli attaccanti saltano da macchina a macchina. Un singolo server C2 può gestire decine o centinaia di beacon in un ambiente compromesso, coordinando azioni come il credential harvesting su una macchina, il lateral movement verso un'altra e lo staging dei dati su una terza.

**Mantenere la stealth tramite canali cifrati.** Il traffico C2 è tipicamente cifrato (TLS, mTLS o cifratura personalizzata). Anche se un difensore cattura i pacchetti, vede blob cifrati. Combina questo con il domain fronting o con un'infrastruttura cloud legittima e il traffico diventa molto difficile da distinguere dalle normali operazioni aziendali.

### Framework C2 del mondo reale

Ecco come si colloca il nostro progetto accanto ai framework usati nelle operazioni reali:

| Framework | Linguaggio | Protocollo | Uso notevole |
|-----------|----------|----------|-------------|
| Cobalt Strike | Java | HTTP/HTTPS/DNS | Il più comune nelle campagne APT. Usato sia dai red team sia da threat actor come APT29, FIN7 e gli operatori del ransomware Conti. |
| Sliver | Go | mTLS/HTTP/DNS/WireGuard | Alternativa open source a Cobalt Strike. Ha guadagnato adozione dopo che le versioni craccate di Cobalt Strike sono diventate pesantemente signaturate. |
| Mythic | Python/Go | HTTP/WebSocket | Framework modulare con agent contribuiti dalla community. Ogni agent è un progetto a sé con le proprie capacità. |
| Brute Ratel | C++ | HTTP/DNS | Progettato specificamente per eludere i prodotti EDR. Ha attirato l'attenzione nel 2022 quando copie craccate sono apparse in giro. |
| Il nostro progetto | Python | WebSocket | Didattico. Mostra il pattern architetturale senza funzionalità di operational security. |

La cosa importante da notare: ognuno di questi framework implementa lo stesso pattern di base. Il server ascolta, il beacon chiama casa, i comandi scorrono verso il basso, i risultati scorrono verso l'alto. Le differenze stanno nella sofisticazione del protocollo, nelle capacità di evasione e nell'operational security. Il nostro progetto elimina la complessità dell'evasione così puoi concentrarti sulla comprensione dell'architettura in sé.

---

## Mapping MITRE ATT&CK

### Cos'è MITRE ATT&CK?

MITRE ATT&CK è una knowledge base di tattiche e tecniche degli avversari basata su osservazioni del mondo reale. Pensala come una tassonomia di tutto ciò che fanno gli attaccanti, organizzata in categorie come Initial Access, Execution, Persistence, Discovery, Collection, Exfiltration e Command and Control.

Ogni tecnica riceve un ID (come T1059 per Command and Scripting Interpreter). I team di sicurezza usano questi ID per classificare gli incidenti, misurare la copertura di detection e comunicare sulle minacce in un linguaggio condiviso. Quando un analista SOC scrive "abbiamo osservato un'esecuzione T1059.001", ogni altro analista sa che intende che è stato usato PowerShell per eseguire comandi.

Il framework vive su [https://attack.mitre.org/](https://attack.mitre.org/) e viene aggiornato regolarmente man mano che nuove tecniche vengono osservate in giro.

Il nostro beacon implementa 10 comandi. Ognuno si mappa su una specifica tecnica ATT&CK. Ecco il mapping completo:

### Tabella comando-tecnica

| Comando | ID tecnica | Tattica | Nome tecnica |
|---------|-------------|--------|----------------|
| `shell` | T1059 | Execution | Command and Scripting Interpreter |
| `sysinfo` | T1082 | Discovery | System Information Discovery |
| `proclist` | T1057 | Discovery | Process Discovery |
| `upload` | T1105 | Command and Control | Ingress Tool Transfer |
| `download` | T1041 | Exfiltration | Exfiltration Over C2 Channel |
| `screenshot` | T1113 | Collection | Screen Capture |
| `keylog_start` | T1056.001 | Collection | Input Capture: Keylogging |
| `keylog_stop` | T1056.001 | Collection | Input Capture: Keylogging |
| `persist` | T1053.003 | Persistence | Scheduled Task/Job: Cron |
| `sleep` | T1029 | Command and Control | Scheduled Transfer |

### Analisi dettagliata

**shell (T1059 - Command and Scripting Interpreter)**

Questa è la capacità C2 più fondamentale. Il beacon riceve una stringa, la passa a `asyncio.create_subprocess_shell()`, cattura stdout e stderr, e li rimanda entrambi indietro. Nella nostra codebase, `handle_shell` in `beacon/beacon.py` fa esattamente questo. Negli attacchi reali, T1059 è presente in quasi ogni intrusione. L'implant SolarWinds SUNBURST usava `cmd.exe` per l'esecuzione dei comandi. La tecnica ha sotto-tecniche per interpreti specifici: T1059.001 (PowerShell), T1059.003 (Windows Command Shell), T1059.004 (Unix Shell). Il nostro beacon usa la shell predefinita del sistema su qualunque OS su cui gira.

**sysinfo (T1082 - System Information Discovery)**

Dopo essere atterrati su una macchina, gli attaccanti hanno bisogno di capire con cosa hanno a che fare. Il nostro `handle_sysinfo` raccoglie hostname, versione dell'OS, architettura, numero di CPU, totali di memoria, partizioni del disco e interfacce di rete usando la libreria `psutil`. Questa è ricognizione. I gruppi APT eseguono abitualmente `systeminfo`, `uname -a` o comandi equivalenti come una delle loro prime azioni. Durante la campagna SolarWinds del 2020, SUNBURST raccoglieva la versione dell'OS, il nome del dominio e i prodotti di sicurezza installati prima che gli operatori decidessero se un target valesse la pena.

**proclist (T1057 - Process Discovery)**

Enumerare i processi in esecuzione dice all'attaccante diverse cose: quali tool di sicurezza sono in esecuzione (c'è un agent EDR? un antivirus?), sotto quale contesto utente girano i processi, e quali servizi sono attivi. Il nostro `handle_proclist` itera `psutil.process_iter()` e restituisce PID, nomi e username per un massimo di 100 processi. Nelle operazioni reali, gli elenchi dei processi aiutano gli attaccanti a decidere in quali processi iniettarsi, quali tool di sicurezza eludere e quali servizi prendere di mira per il furto di credenziali.

**upload (T1105 - Ingress Tool Transfer)**

Caricare file su un host compromesso è il modo in cui gli attaccanti predispongono tool aggiuntivi. Il nostro `handle_upload` riceve un oggetto JSON contenente un nome di file e contenuto codificato in base64, poi lo scrive in `/tmp/`. Nel mondo reale, gli attaccanti caricano exploit di privilege escalation, tool di lateral movement come Mimikatz, implant aggiuntivi o utility di tunnel. La tecnica T1105 è presente ogni volta che vedi un attaccante spingere un binario su un sistema compromesso attraverso il canale C2 invece di far scaricare al target il file da internet.

**download (T1041 - Exfiltration Over C2 Channel)**

Questa è la capacità di furto dei dati. Il nostro `handle_download` legge un file dal disco, lo codifica in base64 e lo rimanda indietro attraverso il WebSocket. Il dettaglio chiave qui: i dati rubati escono attraverso lo stesso canale su cui arrivano i comandi. I difensori che cercano exfiltration tramite protocolli insoliti o upload su cloud storage non se ne accorgerebbero perché i dati sono incorporati nel traffico C2. Il gruppo ransomware DarkSide (responsabile dell'attacco a Colonial Pipeline) ha usato il proprio canale C2 per esfiltrare gigabyte di dati aziendali prima di distribuire il payload ransomware.

**screenshot (T1113 - Screen Capture)**

Fare screenshot permette agli attaccanti di vedere cosa sta facendo un utente in tempo reale. Il nostro `handle_screenshot` usa la libreria `mss` per catturare l'intero schermo, lo converte in PNG e codifica il risultato in base64. Questa tecnica è comune nelle operazioni di spionaggio. APT28 (Fancy Bear) ha usato capacità di screenshot nel loro implant X-Agent per monitorare i target. È usata anche nel crimine finanziario per catturare sessioni bancarie e l'inserimento di credenziali.

**keylog_start / keylog_stop (T1056.001 - Input Capture: Keylogging)**

Il keylogging cattura tutto ciò che un utente digita: password, messaggi, query di ricerca, URL interni. Il nostro beacon usa `pynput` per agganciare gli eventi della tastiera in un thread in background. `handle_keylog_start` avvia la cattura, `handle_keylog_stop` termina il listener e restituisce tutti i tasti catturati. I keylogger sono tra i più vecchi tool offensivi. La differenza tra un keylogger standalone e ciò che abbiamo qui è che il nostro è integrato nel framework C2, quindi i tasti catturati vengono esfiltrati automaticamente attraverso il canale esistente del beacon invece di richiedere un meccanismo di recupero separato.

**persist (T1053.003 - Scheduled Task/Job: Cron)**

La persistence è il modo in cui un attaccante sopravvive ai reboot. Senza di essa, il beacon muore quando la macchina si riavvia e l'attaccante perde l'accesso. Il nostro `handle_persist` scrive una voce cron `@reboot` che rilancia il beacon all'avvio del sistema. La funzione controlla il crontab corrente, verifica che la voce non esista già e la aggiunge. Questa è una tecnica specifica di Linux. L'ID tecnica T1053.003 copre specificamente la persistence basata su cron. Nelle operazioni reali, gruppi APT come TeamTNT hanno usato estesamente la persistence via cron nelle loro campagne di cryptocurrency mining mirate ai server Linux.

**sleep (T1029 - Scheduled Transfer)**

Regolare l'intervallo di sleep è un controllo di operational security. Il nostro `handle_sleep` permette all'operatore di modificare `sleep_interval` e `jitter_percent` al volo. Durante una compromissione iniziale, un attaccante potrebbe usare uno sleep breve (1-5 secondi) per un'interazione rapida. Una volta stabilito l'accesso e desiderando evitare il rilevamento, lo aumenta a minuti o ore. Gli operatori di Cobalt Strike impostano abitualmente intervalli di sleep a 300-900 secondi con jitter del 30-50% durante le operazioni di lunga durata. Il mapping T1029 riflette il fatto che il beacon trasferisce i dati secondo una pianificazione piuttosto che in tempo reale, che è una caratteristica distintiva del beaconing C2.

---

## Codifica del protocollo

### Cifrario XOR

XOR (or esclusivo) è la più semplice operazione binaria reversibile. Dati due bit, XOR restituisce 1 se differiscono e 0 se coincidono:

```
0 XOR 0 = 0
0 XOR 1 = 1
1 XOR 0 = 1
1 XOR 1 = 0
```

La proprietà che rende XOR utile per la codifica: applicare la stessa chiave due volte restituisce i dati originali.

```
data XOR key = encoded
encoded XOR key = data
```

Ecco la matematica concreta con un singolo byte. Diciamo che il byte di dati sia `0x48` (la lettera "H") e il byte della chiave sia `0x63` (la lettera "c"):

```
0x48 = 01001000
0x63 = 01100011
XOR  = 00101011 = 0x2B
```

Applica di nuovo la chiave:

```
0x2B = 00101011
0x63 = 01100011
XOR  = 01001000 = 0x48 = "H"
```

Il nostro progetto implementa questo in `backend/app/core/encoding.py`:

```python
def xor_bytes(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```

La parte `i % len(key)` è critica. Fa ripetere la chiave. Se la chiave è "abc" (3 byte) e i dati sono lunghi 10 byte, la chiave cicla: a,b,c,a,b,c,a,b,c,a. Ogni byte di dati viene messo in XOR con il corrispondente byte della chiave in quella posizione.

**Perché XOR da solo è debole.** Due problemi principali:

Primo, **attacchi known plaintext**. Se un attaccante conosce (o può indovinare) una qualsiasi parte del plaintext, può recuperare la chiave. Poiché il nostro protocollo invia messaggi JSON che iniziano sempre con `{"type":`, un attaccante che cattura l'output codificato può mettere in XOR il plaintext noto con il ciphertext per estrarre la chiave. Una volta che ha la chiave, può decodificare tutto.

Secondo, **analisi delle frequenze**. Con una chiave corta e ripetuta, gli schemi statistici nel plaintext trapelano. La lettera "e" compare circa il 13% delle volte nel testo inglese. Se la chiave si ripete ogni 32 byte, puoi suddividere il ciphertext in 32 flussi e fare l'analisi delle frequenze su ciascuno in modo indipendente. Questa è la classica debolezza del cifrario di Vigenère, e XOR con una chiave ripetuta è essenzialmente Vigenère che opera sui bit invece che sulle lettere.

### Perché XOR + Base64

La nostra pipeline di codifica usa sia XOR sia Base64 per motivi diversi:

- **XOR fornisce un'offuscazione di base.** Non è vera cifratura. Non fermerà un analista determinato. Ma significa che i messaggi del protocollo non stanno sul filo come JSON leggibile. Un osservatore casuale o una regola IDS basica di string-matching non vedrà immediatamente `{"type":"TASK","payload":{"command":"shell"}}` nel traffico.

- **Base64 rende i dati binari sicuri per il trasporto come testo.** Dopo XOR, i dati sono byte arbitrari, inclusi null byte e altri valori che rompono i protocolli di testo. Base64 converte quei byte in un alfabeto ASCII sicuro (A-Z, a-z, 0-9, +, /). Questo è importante perché i frame di testo WebSocket si aspettano dati di testo validi. Senza Base64, dovremmo usare frame WebSocket binari, che è un approccio valido ma aggiunge una complessità di cui non abbiamo bisogno in un progetto per principianti.

- **I framework C2 reali usano cifratura vera e propria.** Cobalt Strike usa AES-256 in modalità CBC. Sliver usa mutual TLS (mTLS) dove sia il server sia il beacon si autenticano con certificati. Brute Ratel usa una combinazione di RC4 e AES. Noi usiamo XOR perché l'obiettivo di questo progetto è insegnare il pattern architetturale, non costruire qualcosa di operativamente sicuro. Capire la codifica XOR ti aiuta a capire cosa fa la vera cifratura a livello concettuale senza la complessità dello scambio di chiavi, dei vettori di inizializzazione e delle modalità del cipher.

### La pipeline di codifica

Ecco il percorso completo che un messaggio compie dal plaintext al formato sul filo e ritorno:

**Codifica (dal beacon al server, o dal server al beacon):**

```
Plaintext JSON --> UTF-8 bytes --> XOR with key --> Base64 --> WebSocket text frame
```

Passo per passo con un esempio reale. Diciamo che il messaggio sia `{"type":"HEARTBEAT","payload":{"id":"abc-123"}}` e la chiave sia `c2-beacon-default-key-change-me`:

1. La stringa JSON diventa byte UTF-8: `7b 22 74 79 70 65 ...`
2. Ogni byte viene messo in XOR con i byte ripetuti della chiave: `7b^63, 22^32, 74^2d, 79^62, ...`
3. Il blob binario risultante viene codificato in Base64 in una stringa ASCII stampabile.
4. Quella stringa viene inviata come frame di testo WebSocket.

**Decodifica (lato ricevente):**

```
WebSocket text frame --> Base64 decode --> XOR with key --> UTF-8 string --> JSON parse
```

Puoi vedere entrambe le direzioni implementate in `backend/app/core/encoding.py`:

```python
def encode(payload: str, key: str) -> str:
    raw = payload.encode("utf-8")
    xored = xor_bytes(raw, key.encode("utf-8"))
    return base64.b64encode(xored).decode("ascii")


def decode(encoded: str, key: str) -> str:
    xored = base64.b64decode(encoded)
    raw = xor_bytes(xored, key.encode("utf-8"))
    return raw.decode("utf-8")
```

Il protocol layer (`backend/app/core/protocol.py`) avvolge queste funzioni con la validazione Pydantic. La funzione `pack` serializza un oggetto `Message` in JSON, poi lo codifica. La funzione `unpack` decodifica, fa il parsing e valida in un solo passo. Se un qualsiasi passaggio fallisce (Base64 errato, XOR produce UTF-8 non valido, JSON malformato, validazione Pydantic fallita), `unpack` solleva un `ValueError`.

---

## Pattern di comportamento del beacon

### Sleep e jitter

Un beacon che chiama casa ogni secondo, al secondo esatto, è banalmente rilevabile. I tool di monitoraggio della rete possono identificare connessioni periodiche con alta confidenza. Se un host fa una richiesta HTTPS verso lo stesso dominio a intervalli di esattamente 1 secondo per ore di fila, è un enorme campanello d'allarme.

Gli intervalli di sleep risolvono parte di questo problema. Invece di comunicazione costante, il beacon attende tra un check-in e l'altro. Il nostro progetto ha come default un intervallo di sleep di 3 secondi, che è aggressivo per un'operazione reale ma va bene per demo interattive.

Il jitter risolve il resto. Il jitter aggiunge casualità all'intervallo di sleep. Il nostro beacon usa il 30% di jitter per impostazione predefinita:

```python
def jittered_sleep() -> float:
    jitter = config.sleep_interval * config.jitter_percent
    return config.sleep_interval + random.uniform(-jitter, jitter)
```

Con un intervallo di 3 secondi e il 30% di jitter, il tempo di sleep effettivo varia da 2,1 secondi a 3,9 secondi. Ogni ciclo è diverso. Lo schema temporale appare irregolare piuttosto che da metronomo.

Per riferimento, Cobalt Strike ha come default un intervallo di sleep di 60 secondi con 0% di jitter. Quel default a 0% è considerato una scarsa operational security, e gli operatori esperti lo cambiano immediatamente. Le impostazioni comuni nel mondo reale sono 300 secondi (5 minuti) con jitter del 30-50% per le operazioni di lunga durata. Durante un engagement attivo in cui l'operatore ha bisogno di risposte rapide, potrebbero usare 5-10 secondi con jitter del 10-20%.

Il trade-off è reattività contro stealth. Uno sleep breve significa che l'operatore ottiene i risultati rapidamente. Uno sleep lungo con jitter elevato rende il beacon più difficile da rilevare ma significa che potresti aspettare 10 minuti perché un comando venga eseguito.

### Heartbeat

Il server ha bisogno di sapere quali beacon sono ancora vivi. Se un beacon tace, è perché l'host è stato spento? Il percorso di rete è cambiato? Il beacon è stato rilevato e ucciso da un EDR?

Il nostro beacon invia messaggi di heartbeat periodici durante il loop di sleep:

```python
async def heartbeat_loop(ws: Any) -> None:
    while True:
        try:
            msg = pack("HEARTBEAT", {"id": config.beacon_id})
            await ws.send(msg)
            await asyncio.sleep(jittered_sleep())
        except Exception:
            break
```

Ogni heartbeat aggiorna il timestamp `last_seen` nel database del server (tramite `registry.update_last_seen` in `backend/app/beacon/registry.py`). La dashboard mostra questo timestamp così l'operatore può vedere a colpo d'occhio quali beacon stanno facendo check-in attivamente e quali sono andati silenti.

Nei framework C2 di produzione, l'heartbeat è spesso implicito piuttosto che esplicito. L'atto stesso di fare check-in per i task funge da heartbeat. Il nostro progetto usa messaggi di heartbeat espliciti perché le connessioni WebSocket sono persistenti, e abbiamo bisogno di un meccanismo per mantenere viva la connessione e aggiornare i timestamp anche quando nessun task è in coda.

### Strategia di riconnessione

Le reti sono inaffidabili. I server si riavviano. I firewall resettano le connessioni. Il beacon deve gestire tutto questo con eleganza.

Il nostro beacon implementa l'exponential backoff:

```
Attempt 1: wait 2 seconds
Attempt 2: wait 4 seconds
Attempt 3: wait 8 seconds
Attempt 4: wait 16 seconds
...continuing to double...
Attempt N: wait up to 300 seconds (5-minute cap)
```

Il codice rilevante in `beacon/beacon.py`:

```python
backoff = config.reconnect_base

while True:
    try:
        async with connect(config.server_url) as ws:
            backoff = config.reconnect_base
            ...
    except (ConnectionRefusedError, websockets.exceptions.ConnectionClosed, OSError):
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, config.reconnect_max)
```

Due cose da notare. Primo, su una connessione riuscita, `backoff` si resetta al valore base (2 secondi). Secondo, il backoff si ferma a `reconnect_max` (300 secondi). Senza il tetto, il backoff crescerebbe indefinitamente e il beacon potrebbe finire per aspettare ore tra un tentativo e l'altro.

Perché l'exponential backoff è importante: se il server è giù e 50 beacon stanno cercando di riconnettersi simultaneamente, non vuoi che tutti e 50 martellino il server ogni 2 secondi. Questo crea rumore di rete e potrebbe sembrare un DDoS. L'exponential backoff distribuisce i tentativi di riconnessione nel tempo, riducendo il carico e riducendo la visibilità.

---

## Meccanismi di persistence

La persistence significa che l'implant sopravvive ai reboot del sistema. Senza persistence, il beacon muore quando la macchina si riavvia, e l'attaccante deve ricompromettere il target da zero. Questo è costoso e rischioso. Ogni tentativo di compromissione è un'occasione per farsi scoprire.

Il nostro progetto implementa un meccanismo di persistence: la persistence basata su cron su Linux. La funzione `handle_persist` in `beacon/beacon.py` aggiunge una voce cron `@reboot` che rilancia lo script del beacon all'avvio del sistema:

```python
beacon_path = os.path.abspath(__file__)
cron_entry = f"@reboot /usr/bin/python3 {beacon_path} &"
```

La funzione legge il crontab corrente, controlla se la voce esiste già (per evitare duplicati), e la aggiunge se necessario.

La persistence via cron è semplice ed efficace sui sistemi Linux, ma è anche semplice da rilevare. Qualsiasi difensore che controlla `crontab -l` la vedrà. Gli attaccanti più sofisticati usano meccanismi più difficili da rilevare.

Ecco le tecniche di persistence comuni organizzate per sistema operativo:

**Linux:**
- Cron job (ciò che usiamo noi) - voci `@reboot` o voci pianificate
- Servizi systemd - un file unit `.service` in `/etc/systemd/system/`
- Modifica del bash profile - aggiungere righe di esecuzione a `~/.bashrc` o `~/.profile`
- Script di init - aggiungere a `/etc/rc.local` o `/etc/init.d/`

**Windows:**
- Registry run key - `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` o l'equivalente HKLM
- Scheduled task - `schtasks.exe /create` con vari trigger
- DLL hijacking - posizionare una DLL malevola in una directory cercata prima di quella legittima
- Servizi Windows - registrarsi come servizio di sistema
- WMI event subscription - attivare l'esecuzione su eventi WMI

**macOS:**
- Launch agent - file plist in `~/Library/LaunchAgents/`
- Launch daemon - file plist in `/Library/LaunchDaemons/`
- Login item - usando la Shared File List o il framework Service Management
- Dylib hijacking - l'equivalente macOS del DLL hijacking

Ognuna di queste si mappa su una diversa sotto-tecnica MITRE ATT&CK sotto T1053 (Scheduled Task/Job), T1547 (Boot or Logon Autostart Execution) o T1574 (Hijack Execution Flow). Il filo conduttore: l'attaccante fa in modo che il proprio codice venga eseguito automaticamente quando scatta un qualche trigger (boot, login, orario pianificato, avvio di un'applicazione).

---

## Detection e difesa

### Come i blue team rilevano il C2

Capire la detection è importante tanto quanto capire l'attacco. Se sai solo come costruire il C2, ti manca metà del quadro. Ecco come i difensori trovano e fermano il traffico dei beacon.

**Firme di rete e analisi del traffico**

Il metodo di detection più diretto. I tool di network security monitoring (NSM) analizzano il traffico alla ricerca di pattern associati al C2:

- **Pattern di beaconing periodici.** Anche con il jitter, l'analisi statistica può identificare connessioni che avvengono a intervalli semi-regolari. Tool come RITA (Real Intelligence Threat Analytics) sono specializzati nel rilevare il beaconing analizzando le distribuzioni della frequenza delle connessioni. Se un host contatta lo stesso server esterno 480 volte in 24 ore a intervalli di circa 3 minuti, è statisticamente identificabile.

- **Connessioni WebSocket insolite.** La maggior parte delle workstation aziendali non mantiene connessioni WebSocket di lunga durata verso server esterni. Una connessione WebSocket che persiste per ore e trasmette messaggi piccoli e regolari è sospetta. Il nostro progetto usa WebSocket in chiaro (lo schema `ws://`), il che significa che il contenuto del traffico è visibile a qualsiasi dispositivo di ispezione inline.

- **Pattern di payload codificati.** I blob codificati in Base64 nei frame WebSocket sono insoliti per le applicazioni legittime. Mentre alcune applicazioni usano Base64 per i dati binari, uno schema prolungato di messaggi Base64 di dimensioni costanti a intervalli regolari risalta.

- **Fingerprinting JA3/JA3S.** JA3 crea un hash dei parametri del TLS Client Hello (cipher suite, estensioni, curve ellittiche). JA3S fa lo stesso per il Server Hello. I framework C2 noti producono fingerprint JA3 distintivi perché usano librerie TLS specifiche con configurazioni specifiche. L'hash JA3 di default di Cobalt Strike è ben documentato e ampiamente signaturato. Il nostro progetto non usa TLS in sviluppo, il che rende questo specifico metodo di detection inapplicabile ma significa anche che il nostro traffico è completamente leggibile da chiunque sia sulla rete.

- **Anomalie DNS.** I framework C2 basati su DNS (come le modalità DNS in Cobalt Strike, Sliver e Brute Ratel) codificano i dati nelle query e nelle risposte DNS. I segni rilevabili includono volumi elevati di query su record TXT, label di sottodominio insolitamente lunghe e traffico DNS verso domini con nomi ad alta entropia. Il nostro progetto non usa il DNS per il C2, ma dovresti conoscere questa tecnica.

**Analisi comportamentale sugli endpoint**

Anche se il traffico di rete è cifrato e si confonde, le azioni che il beacon compie sull'host sono rilevabili:

- **Processi che generano shell.** Quando il nostro beacon esegue un comando `shell`, crea un processo figlio che esegue un comando di shell. Un processo Python che genera `/bin/sh` o `cmd.exe` è sospetto. I prodotti EDR tracciano le relazioni parent-child tra i processi. Se `python3` genera `bash` che genera `whoami`, quell'albero dei processi viene segnalato.

- **Accesso insolito ai file.** Il comando `download` legge i file. Il comando `upload` scrive in `/tmp/`. Il comando `screenshot` accede alle API di cattura dello schermo. I prodotti EDR monitorano queste operazioni e possono segnalare pattern anomali, specialmente quando provengono da un processo che normalmente non fa queste cose.

- **Hook del keylogger.** Il nostro comando `keylog_start` usa `pynput` per agganciare gli eventi della tastiera. Su Linux, questo comporta la lettura dai device `/dev/input/` o l'uso delle estensioni di input X11. Su Windows, userebbe `SetWindowsHookEx`. I prodotti EDR sorvegliano specificamente l'installazione di input hook perché le applicazioni legittime raramente hanno bisogno di intercettare tutti i tasti premuti.

**Artefatti di detection sugli endpoint**

- **Artefatti di persistence.** I difensori monitorano crontab, servizi systemd, registry run key e scheduled task per modifiche non autorizzate. Una nuova voce cron `@reboot` che punta a uno script Python in una posizione insolita è un ovvio indicatore di compromissione (IOC).

- **File sospetti su disco.** Il nostro beacon sta su disco come un normale file Python. I tool di file integrity monitoring (FIM) notano nuovi file in posizioni inaspettate. I prodotti antivirus ed EDR scansionano i file confrontandoli con firme note ed euristiche comportamentali.

- **Indicatori di process injection.** Gli implant C2 avanzati iniettano il proprio codice in processi legittimi (come `svchost.exe` o `explorer.exe`) per nascondersi. Il nostro beacon non lo fa, il che significa che il processo `python3` che esegue `beacon.py` è visibile negli elenchi dei processi. Questo lo rende più facile da trovare ma significa anche che non stiamo insegnando tecniche di process injection, il che è appropriato per un progetto per principianti.

### Perché il nostro progetto è facile da rilevare

Questo è intenzionale, ed è importante essere onesti sui limiti:

**Niente TLS.** La nostra connessione WebSocket usa `ws://` (testo in chiaro) in sviluppo, non `wss://` (cifrata con TLS). Qualsiasi tool di monitoraggio della rete può leggere i frame WebSocket grezzi. In un engagement reale, tutto il traffico C2 sarebbe cifrato con TLS come minimo.

**Chiave XOR fissa.** La chiave è impostata nella configurazione e non cambia mai. Se un difensore cattura un messaggio e determina la chiave (cosa semplice dato l'attacco known plaintext descritto sopra), può decodificare tutti i messaggi passati e futuri. I framework C2 reali ruotano le chiavi di cifratura per sessione o per messaggio.

**Nessun process injection o hollowing.** Il nostro beacon gira come proprio processo visibile. Un difensore che esegue `ps aux | grep python` o controlla l'elenco dei processi in un task manager lo vedrà. Gli implant avanzati si iniettano in processi fidati così da ereditarne la reputazione e la visibilità.

**Nessun bypass di AMSI o evasione di EDR.** Su Windows, l'Antimalware Scan Interface (AMSI) permette ai prodotti di sicurezza di ispezionare gli script prima dell'esecuzione. I framework C2 avanzati includono tecniche di bypass di AMSI. Il nostro progetto non affronta affatto questo aspetto.

**Nessun domain fronting o traffic blending.** Il beacon si connette direttamente all'indirizzo del server C2. I setup C2 sofisticati usano il domain fronting (instradare il traffico attraverso un'infrastruttura CDN così che il dominio visibile differisca dalla destinazione effettiva) o imitano i pattern di traffico di applicazioni legittime.

**Il file del beacon sta in chiaro su disco.** Nessun packing, nessuna offuscazione, nessuna esecuzione fileless. Il codice sorgente del beacon è leggibile da chiunque abbia accesso al filesystem.

Tutte queste limitazioni sono deliberate. L'obiettivo di questo progetto è insegnarti come funziona l'architettura C2. Una volta che capisci il pattern server-beacon-operatore, la pipeline di codifica, il modello di task queue e la superficie di detection, hai le basi per capire cosa fanno i framework più avanzati e perché. Insegnare tecniche di evasione in un progetto per principianti darebbe priorità alle cose sbagliate.

---

## Standard e framework di settore

### MITRE ATT&CK

Abbiamo fatto riferimento a MITRE ATT&CK in tutto questo documento, ma merita una sezione a sé.

ATT&CK sta per Adversarial Tactics, Techniques, and Common Knowledge. Il framework è mantenuto da The MITRE Corporation ed è liberamente disponibile su [https://attack.mitre.org/](https://attack.mitre.org/).

Il framework è organizzato in una matrice:

- **Le tattiche** sono le colonne. Rappresentano il "perché" di un attacco. Esempi: Initial Access, Execution, Persistence, Privilege Escalation, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, Exfiltration, Command and Control.

- **Le tecniche** sono le celle sotto ogni tattica. Rappresentano il "come". Esempio: sotto Execution, trovi T1059 (Command and Scripting Interpreter), T1053 (Scheduled Task/Job), T1569 (System Services) e molte altre.

- **Le sotto-tecniche** aggiungono specificità. T1059.001 è specificamente PowerShell. T1059.004 è Unix Shell. T1053.003 è Cron.

- **Le procedure** sono esempi del mondo reale di una tecnica usata da uno specifico threat group o famiglia di malware.

Quando costruisci le detection, pensi in termini di copertura ATT&CK. "Possiamo rilevare T1059.001 (esecuzione PowerShell) e T1053.003 (persistence via cron) ma non abbiamo copertura per T1055 (process injection)." Questa gap analysis guida le decisioni di investimento nella sicurezza.

Il nostro progetto tocca queste categorie di tattiche:
- **Execution** (comando shell)
- **Discovery** (sysinfo, proclist)
- **Collection** (screenshot, keylogging)
- **Exfiltration** (download)
- **Persistence** (cron)
- **Command and Control** (upload, sleep e l'intero modello di comunicazione beacon-server)

### La Pyramid of Pain

David Bianco pubblicò la Pyramid of Pain nel 2013 per descrivere come diversi tipi di indicatori influenzino un attaccante quando i difensori li usano per la detection. La piramide ha sei livelli, dal più facile al più difficile da usare per i difensori (e dal meno al più doloroso da cambiare per gli attaccanti):

```
        /\
       /  \
      / TTPs \          <-- Very painful to change
     /--------\
    /  Tools   \        <-- Challenging
   /------------\
  / Network/Host\      <-- Annoying
 /  Artifacts    \
/----------------\
/ Domain Names    \     <-- Simple (buy new domain)
/------------------\
/ IP Addresses      \   <-- Easy (get new IP)
/--------------------\
/ Hash Values         \  <-- Trivial (recompile)
```

Ecco come l'infrastruttura C2 si mappa su questa piramide:

**Hash Value (banale).** Se i difensori rilevano il nostro beacon tramite il suo file hash, l'attaccante ricompila con una piccola modifica e ottiene un nuovo hash. Questo è il metodo di detection di minor valore.

**Indirizzi IP (facile).** Se i difensori bloccano l'indirizzo IP del server C2, l'attaccante avvia un nuovo server su un IP diverso. L'infrastruttura cloud fa sì che questo richieda minuti.

**Nomi di dominio (semplice).** Bloccare il dominio C2 è leggermente più doloroso perché l'attaccante deve registrare nuovi domini e riconfigurare i beacon. Ma i domini sono economici e abbondanti.

**Network Artifact (fastidioso).** È qui che le cose iniziano a contare. Se i difensori rilevano lo specifico pattern dei frame WebSocket del beacon, i payload XOR codificati in Base64 o la distribuzione dell'intervallo di beaconing, l'attaccante deve modificare il protocollo. Questo richiede sforzo di sviluppo. Per il nostro progetto, lo schema di codifica e il formato dei messaggi sono network artifact.

**Tool (impegnativo).** Rilevare il framework C2 stesso (non solo il suo traffico di rete, ma le sue capacità e i suoi pattern di comportamento) costringe l'attaccante a cambiare completamente tool. Passare da Cobalt Strike a Sliver perché Cobalt Strike è troppo pesantemente signaturato è un costo reale.

**TTP (molto doloroso).** Tactics, Techniques, and Procedures sono in cima. Se i difensori rilevano "qualsiasi processo che stabilisce una connessione WebSocket persistente in uscita e periodicamente genera comandi shell", l'attaccante non può semplicemente cambiare tool o dominio. Deve cambiare in modo fondamentale il proprio approccio operativo. Questo è il Sacro Graal della detection engineering.

La lezione: rilevare il nostro beacon tramite la sua chiave XOR o il suo file hash è di basso valore. Rilevarlo tramite il suo pattern comportamentale (connessione WebSocket periodica + generazione di shell + accesso ai file + modifica di cron) è di alto valore perché quei comportamenti persistono indipendentemente dallo schema di codifica o dall'indirizzo IP che l'attaccante usa.

---

## Verifica la tua comprensione

Lavora su queste domande prima di passare alla sezione successiva. Se riesci a rispondere a tutte e cinque senza guardare indietro al testo, hai una solida padronanza dei fondamenti.

**1. Perché i beacon avviano connessioni in uscita invece che il server si connetta in entrata?**

I firewall aziendali bloccano le connessioni in entrata non sollecitate. Le workstation stanno dietro il NAT senza indirizzi IP pubblici. Le connessioni in uscita su porte standard (80, 443) sono tipicamente consentite perché i dipendenti hanno bisogno dell'accesso al web. Facendo chiamare il beacon, il traffico C2 si confonde con il normale traffico web e aggira i controlli di sicurezza perimetrali che filtrano solo le connessioni in entrata.

**2. Qual è lo scopo del jitter nell'intervallo di sleep di un beacon?**

Il jitter aggiunge casualità alla tempistica dei check-in. Senza jitter, un beacon che dorme per esattamente 3 secondi crea un pattern di traffico perfettamente periodico che è banalmente rilevabile dai tool di monitoraggio della rete. Con il 30% di jitter, l'intervallo varia tra 2,1 e 3,9 secondi, rendendo il pattern statisticamente più difficile da identificare come beaconing automatizzato.

**3. In cosa la codifica XOR differisce dalla vera cifratura come AES-256?**

XOR con una chiave ripetuta è vulnerabile agli attacchi known plaintext e all'analisi delle frequenze. Se conosci una qualsiasi porzione del plaintext (e la conosci, perché i messaggi JSON hanno una struttura prevedibile), puoi recuperare la chiave. AES-256 è un block cipher che usa più round di operazioni di sostituzione, permutazione e mixing. È resistente a tutti gli attacchi crittoanalitici noti quando usato correttamente con una gestione appropriata delle chiavi, vettori di inizializzazione casuali e modalità del cipher adeguate. XOR è offuscazione. AES è cifratura. Sono fondamentalmente diversi nelle garanzie di sicurezza.

**4. Cita tre modi in cui un difensore potrebbe rilevare il traffico del nostro beacon su una rete.**

Tre qualsiasi tra: (a) Identificare connessioni WebSocket periodiche con pattern temporali semi-regolari. (b) Rilevare payload codificati in Base64 nei frame di testo WebSocket. (c) Osservare il processo che genera comandi shell dopo aver ricevuto messaggi WebSocket. (d) Monitorare la presenza di nuove voci cron create da processi non standard. (e) Segnalare la connessione WebSocket non cifrata (`ws://`) verso un server esterno. (f) Identificare il processo del beacon per nome negli elenchi dei processi.

**5. Perché il nostro beacon usa l'exponential backoff per la riconnessione?**

Due motivi. Primo, impedisce al beacon di martellare un server temporaneamente giù, cosa che creerebbe rumore di rete visibile e sembrerebbe un'attività di scanning o DDoS. Secondo, se molti beacon perdono la connessione simultaneamente (diciamo che il server si riavvia), l'exponential backoff con un po' di casualità distribuisce i loro tentativi di riconnessione nel tempo invece di farli colpire tutti il server in una volta. Il tetto a 300 secondi garantisce che il beacon alla fine si riconnetta in un tempo ragionevole anche dopo interruzioni prolungate.

---

## Letture di approfondimento

- **MITRE ATT&CK Framework:** [https://attack.mitre.org/](https://attack.mitre.org/) - Il riferimento definitivo per le tattiche e le tecniche degli avversari. Inizia con la matrice Enterprise ed esplora le tecniche che il nostro progetto implementa.

- **The C2 Matrix:** [https://www.thec2matrix.com/](https://www.thec2matrix.com/) - Un foglio di calcolo mantenuto dalla community che confronta decine di framework C2 per funzionalità, protocolli e capacità. Utile per capire il panorama al di là dei framework menzionati in questo documento.

- **Documentazione di Cobalt Strike:** [https://www.cobaltstrike.com/](https://www.cobaltstrike.com/) - Capire Cobalt Strike è importante perché ha stabilito lo standard che la maggior parte dei framework C2 moderni segue. Anche se non lo usi mai, conoscere i suoi concetti (malleable profile, named pipe, spawn-to process) ti dà il vocabolario che il settore usa quotidianamente.

- **La Pyramid of Pain di David Bianco:** Il post originale del blog del 2013 che ha definito come diversi tipi di indicatori influenzino gli avversari. Cerca "David Bianco Pyramid of Pain" per trovare l'originale e molti derivati.

- **SANS Reading Room - C2 Analysis:** SANS pubblica paper di ricerca sulla detection e l'analisi del C2. I loro paper sul network threat hunting e sulla detection del beaconing sono particolarmente rilevanti per ciò che abbiamo costruito qui.

- **Red Canary Threat Detection Report:** Pubblicato annualmente, questo report copre le tecniche ATT&CK più diffuse osservate negli incidenti reali. Fornisce esempi concreti di come le tecniche del nostro progetto si manifestano nelle violazioni reali.
