# Challenge - Estendi l'eBPF Security Tracer

## Challenge facili

### 1. Aggiungi il supporto IPv6

**Cosa costruire**: Estendi il network tracer in modo che faccia il parsing di `sockaddr_in6` e mostri gli indirizzi IPv6.

**Perché è utile**: Molti servizi moderni e ambienti cloud usano IPv6. Gli attaccanti possono usare IPv6 per aggirare un monitoraggio solo IPv4.

**Cosa imparerai**: La struttura degli indirizzi IPv6, la gestione di più address family in eBPF, l'espansione della struct dell'evento.

**Suggerimenti**:
- Aggiungi un campo `u8 addr_v6[16]` alla struct dell'evento
- Controlla `sa.sin_family == AF_INET6` in `parse_sockaddr`
- Usa `ipaddress.IPv6Address(bytes(addr_v6))` di Python per la conversione
- Non dimenticare di aggiornare la definizione ctypes di `RawEvent`

**Verifica che funzioni**: Esegui `curl -6 http://ipv6.google.com` mentre il tracing è attivo e verifica che l'indirizzo IPv6 compaia nell'output.

### 2. Aggiungi la catena di ancestry dei processi

**Cosa costruire**: Per ogni evento, risali l'albero dei processi (tramite /proc/<pid>/status) per mostrare l'intera ancestry: `bash -> python -> curl`.

**Perché è utile**: Sapere che una `connect` sospetta proviene da `systemd -> sshd -> bash -> python -> nc` racconta una storia molto più ricca del semplice "nc si è connesso da qualche parte."

**Cosa imparerai**: Il filesystem /proc, la risalita dell'albero dei processi, i compromessi di performance dell'enrichment.

**Suggerimenti**:
- Leggi `/proc/<pid>/status` e fai il parsing della riga `PPid:`
- Risali fino al PID 1 o a un errore di lettura
- Usa la cache in modo aggressivo, gli alberi dei processi non cambiano spesso
- Considera una profondità massima (8-10) per evitare casi patologici

**Verifica che funzioni**: Esegui `bash -c "python3 -c 'import os; os.system(\"ls\")'` e verifica che la catena di ancestry compaia.

### 3. Aggiungi un riepilogo con i contatori degli eventi

**Cosa costruire**: Quando il tool termina (Ctrl+C), stampa una tabella riepilogativa con il totale degli eventi per tipo, il totale delle detection per severità e i primi 10 processi per numero di eventi.

**Perché è utile**: Dopo una sessione di tracing, vuoi una panoramica rapida di cosa è successo senza scorrere migliaia di eventi.

**Cosa imparerai**: Aggregazione dei dati, formattazione delle tabelle con Rich, pattern di shutdown pulito.

**Suggerimenti**:
- Aggiungi contatori alla pipeline di elaborazione degli eventi (usa `collections.Counter`)
- Registra una funzione di cleanup che stampa il riepilogo
- Il `TableRenderer` mostra già come usare le tabelle di Rich

**Verifica che funzioni**: Esegui il tracer per 30 secondi su un sistema trafficato, poi premi Ctrl+C e verifica che il riepilogo compaia.

## Challenge intermedie

### 4. Aggiungi la detection consapevole dei container

**Cosa costruire**: Rileva se gli eventi hanno origine dall'interno di un container e includi l'ID del container nell'output. Segnala come CRITICAL i caricamenti di moduli del kernel e le operazioni di mount provenienti dai container.

**Perché è utile**: Le container escape sono un importante vettore di attacco in Kubernetes. Rilevare `mount` o `init_module` dall'interno di un namespace di container è un forte indicatore di un tentativo di escape.

**Cosa imparerai**: I namespace Linux, i cgroup, il rilevamento del container runtime, come i container siano solo processi con isolamento aggiuntivo.

**Suggerimenti**:
- Leggi `/proc/<pid>/cgroup` per rilevare l'appartenenza a un container
- I container Docker hanno path cgroup come `/docker/<container_id>`
- I pod Kubernetes hanno path come `/kubepods/pod<uid>/<container_id>`
- Il PID 1 dentro un container corrisponde a un PID normale sull'host
- Aggiungi un campo `container_id` a `TracerEvent`

**Verifica che funzioni**: Esegui `docker run --rm alpine sh -c "ls /etc"` mentre il tracing è attivo e verifica che l'ID del container compaia. Verifica che un `mount` da un container generi una detection CRITICAL.

### 5. Aggiungi l'anomaly detection basata sul rate

**Cosa costruire**: Rileva rate anomali di syscall. Se un processo effettua più di N chiamate `openat` in T secondi (ad es. 100 open in 5 secondi), segnalalo come "Rapid File Scanning."

**Perché è utile**: Gli strumenti automatizzati che cercano credenziali, file sensibili o configurazioni sfruttabili generano pattern distintivi di accesso rapido ai file che l'uso normale non produce.

**Cosa imparerai**: Il calcolo del rate con sliding window, l'anomaly detection basata su soglie, la regolazione del tasso di falsi positivi.

**Suggerimenti**:
- Usa la deque per-PID già presente nel detection engine
- Conta gli eventi di ciascun tipo nella finestra
- Parti con soglie alte per evitare rumore, poi abbassale
- Considera soglie diverse per ogni tipo di evento (openat è naturalmente ad alto volume, ptrace no)
- Aggiungi una regola `D011` a `DETECTION_RULES`

**Verifica che funzioni**: Scrivi uno script che apre 200 file in un ciclo e verifica che la detection scatti. Verifica che un normale `ls` di una directory grande non la attivi.

### 6. Aggiungi l'output Syslog/JSON-over-UDP

**Cosa costruire**: Aggiungi una modalità di output che invia gli eventi a un server syslog remoto oppure come JSON via UDP.

**Perché è utile**: In produzione, alimenteresti un SIEM (Splunk, Elastic, Wazuh) con gli eventi eBPF. UDP/syslog è il punto di integrazione più semplice.

**Cosa imparerai**: La programmazione di rete in Python, il protocollo syslog (RFC 5424), il structured logging per l'integrazione con un SIEM.

**Suggerimenti**:
- `socket.socket(socket.AF_INET, socket.SOCK_DGRAM)` per UDP
- Formato syslog: `<priority>VERSION TIMESTAMP HOSTNAME APP-NAME PROCID MSGID MSG`
- Mappa i livelli di severità alle priorità syslog (CRITICAL -> LOG_CRIT, ecc.)
- Aggiungi l'opzione CLI `--syslog host:port`

**Verifica che funzioni**: Esegui `nc -ul 1514` in un terminale, avvia il tracer con `--syslog localhost:1514` e verifica che gli eventi arrivino.

## Challenge avanzate

### 7. Aggiungi il filtering a livello eBPF

**Cosa costruire**: Sposta il filtering su PID e comm dentro i programmi eBPF, così che gli eventi filtrati non raggiungano mai lo userspace. Attualmente tutti gli eventi passano dal ring buffer e vengono filtrati in Python.

**Perché è utile**: Su un server trafficato che genera più di 50K eventi al secondo, il filtering nello userspace spreca banda del ring buffer. Il filtering a livello eBPF riduce l'overhead di 10-100 volte per i workload filtrati.

**Cosa imparerai**: Le BPF hash map per la configurazione, il passaggio dello stato dei filtri da Python a eBPF, la logica condizionale sicura per il verifier.

**Suggerimenti**:
- Usa `BPF_HASH(pid_filter, u32, u32)` come insieme di PID da includere
- Popola la map da Python: `b["pid_filter"][ctypes.c_uint32(pid)] = ctypes.c_uint32(1)`
- Nel programma eBPF, controlla: `if (pid_filter.lookup(&pid) == NULL) return 0;`
- Per il filtering su comm, usa `BPF_HASH(comm_filter, char[16], u32)`
- Una filter map vuota significa "traccia tutto"

**Verifica che funzioni**: Fai il benchmark dell'event rate con e senza filtering a livello eBPF su un processo che genera 1000 processi figli al secondo. Misura la differenza di utilizzo della CPU.

### 8. Costruisci una dashboard in tempo reale

**Cosa costruire**: Una dashboard da terminale che usa il Live display di Rich e mostra: grafico dell'event rate, detection attive, processi principali e un log scorrevole degli eventi, il tutto aggiornato in tempo reale.

**Perché è utile**: Il monitoraggio operativo della sicurezza richiede visibilità a colpo d'occhio. Una dashboard ti permette di osservare il comportamento del sistema durante l'incident response senza leggere le singole righe di log.

**Cosa imparerai**: Rich Live e Layout per il design di TUI, l'aggregazione concorrente dei dati, la gestione del refresh rate.

**Suggerimenti**:
- Usa `rich.live.Live` con `rich.layout.Layout` per una visualizzazione multi-pannello
- Aggiorna ogni 500ms (2 FPS bastano per la leggibilità umana)
- Traccia l'event rate con una finestra mobile di 1 secondo
- Usa `rich.panel.Panel` per ogni sezione
- Considera `rich.progress.SparklineColumn` per la visualizzazione del rate

**Verifica che funzioni**: Esegui la dashboard su un sistema sotto carico (ad es. `stress-ng --cpu 4 --io 4`) e verifica che tutti i pannelli si aggiornino correttamente.

## Challenge expert

### 9. Costruisci un DSL per le regole di detection

**Cosa costruire**: Sostituisci la logica di detection hardcoded in `detector.py` con un rule engine che carica le regole di detection da file YAML, simile al formato delle regole di Falco:

```yaml
- rule: Reverse Shell Detected
  condition:
    sequence:
      - event_type: connect
        within: 10s
      - event_type: execve
        comm_in: [sh, bash, dash, zsh]
    group_by: pid
  severity: CRITICAL
  mitre: T1059.004
  description: Shell execution following outbound connection
```

**Perché è utile**: Le regole di detection hardcoded richiedono modifiche al codice e un nuovo deploy. Un DSL permette ai team di sicurezza di scrivere e modificare le regole senza toccare il codice Python, ed è così che funzionano Falco, Sigma e YARA.

**Cosa imparerai**: Il design di un rule engine, la validazione di schemi YAML, il temporal pattern matching, i principi di design dei DSL.

**Suggerimenti**:
- Inizia con regole stateless (match su un singolo evento) prima di affrontare le sequenze
- Usa Pydantic per la validazione dello schema delle regole
- Supporta gli operatori: `eq`, `in`, `startswith`, `regex`, `gt`, `lt`
- Per le regole di sequenza, riusa la correlazione esistente basata su deque ma rendila configurabile
- Aggiungi l'opzione CLI `--rules-dir` per caricare le regole da una directory
- Considera le priorità delle regole (first match vs best match)

**Verifica che funzioni**: Porta tutte e 10 le regole di detection esistenti in YAML. Verifica che tutti i test esistenti passino ancora con le regole caricate da YAML. Aggiungi una regola personalizzata e verifica che rilevi correttamente.

## Mix and Match

- **Container + Rate Anomaly**: Rileva il rapid file scanning dentro i container (un forte indicatore di reconnaissance del container prima di un tentativo di escape)
- **IPv6 + Syslog**: Monitoraggio full-stack con supporto IPv6 inviato a un SIEM
- **Dashboard + eBPF Filtering**: Dashboard ad alte prestazioni che mostra solo gli eventi filtrati

## Integrazione nel mondo reale

- **Wazuh**: Invia l'output JSON a `ossec.log` di Wazuh oppure usa l'API per l'ingestione di eventi in tempo reale
- **Elastic SIEM**: Invia l'output JSONL a Filebeat, che lo trasferisce a Elasticsearch
- **Grafana/Loki**: Usa promtail per inviare gli eventi, costruisci dashboard per gli event rate e il conteggio delle detection
- **Slack/PagerDuty**: Aggiungi un webhook renderer che invia le detection CRITICAL a Slack o attiva incident su PagerDuty

## Challenge di performance

### Benchmark e ottimizzazione

1. Genera carico con `stress-ng --syscall 0 --timeout 60s`
2. Misura il throughput in eventi/secondo
3. Fai il profiling con `py-spy` per trovare i colli di bottiglia in Python
4. Obiettivo: gestire più di 50K eventi/secondo senza perdere eventi

### Tuning del ring buffer

1. Parti da un ring buffer da 256KB
2. Sotto carico, controlla il drop rate (aggiungi una callback per gli eventi persi)
3. Sperimenta con buffer da 512KB, 1MB, 4MB
4. Trova la dimensione minima del buffer che ottiene zero drop per il tuo workload

## Challenge di sicurezza

### Aggiungi il File Integrity Monitoring

Monitora le scritture sui file di sistema critici (`/etc/passwd`, `/etc/sudoers`, `/etc/ssh/sshd_config`) e genera un alert a ogni modifica. È ciò che fanno tool come AIDE e Tripwire, ma in tempo reale.

### Aggiungi una allowlist/denylist di rete

Mantieni un elenco delle connessioni in uscita attese per ogni processo. Genera un alert quando un processo si connette a un IP o a una porta che non è nella sua allowlist. Parti con una modalità di apprendimento che genera automaticamente l'allowlist.

### Aggiungi la detection anti-evasione

Rileva i processi che cercano di eludere il tracing: rinominarsi per sembrare processi di sistema, fare fork rapidi per confondere il tracking basato sul PID, oppure usare `prctl(PR_SET_NAME)` per cambiare il proprio comm.

## Idee per contribuire

- Porta i programmi eBPF da BCC a libbpf per renderli pronti alla produzione
- Aggiungi hook eBPF LSM per l'enforcement (bloccare, non solo rilevare)
- Costruisci una web UI con streaming degli eventi in tempo reale basato su WebSocket
- Aggiungi il supporto al formato di regole Sigma (regole di detection standard del settore)
- Crea un file unit di systemd per l'esecuzione come daemon

## Completamento delle challenge

- [ ] Facile 1: Supporto IPv6
- [ ] Facile 2: Catena di ancestry dei processi
- [ ] Facile 3: Riepilogo con contatori degli eventi
- [ ] Intermedia 4: Detection consapevole dei container
- [ ] Intermedia 5: Anomaly detection basata sul rate
- [ ] Intermedia 6: Output Syslog/UDP
- [ ] Avanzata 7: Filtering a livello eBPF
- [ ] Avanzata 8: Dashboard in tempo reale
- [ ] Expert 9: DSL per le regole di detection

## Ottenere aiuto

**Debug dei programmi eBPF**: Aggiungi `bpf_trace_printk("debug: %d\n", value)` al tuo codice C e leggi l'output con `sudo cat /sys/kernel/debug/tracing/trace_pipe`. È l'equivalente del printf-debugging per eBPF.

**Errori del verifier**: Il verifier eBPF stampa messaggi criptici. Cause comuni: loop non limitato, accesso alla memoria senza bounds check, stack overflow (>512 byte). Riduci le dimensioni delle struct oppure usa le BPF map per i dati di grandi dimensioni.

**Problemi con BCC**: Se BCC non riesce a compilare, controlla che gli header del kernel corrispondano al kernel in esecuzione: `uname -r` dovrebbe corrispondere a una directory in `/lib/modules/`.

**Risorse della community**:
- [iovisor/bcc GitHub Issues](https://github.com/iovisor/bcc/issues) - Domande specifiche su BCC
- [eBPF Slack](https://ebpf.io/slack) - Chat della community
- [Brendan Gregg's Blog](https://www.brendangregg.com/blog/) - Analisi delle performance di eBPF
