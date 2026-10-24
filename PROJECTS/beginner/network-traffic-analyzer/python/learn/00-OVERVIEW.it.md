# Network Traffic Analyzer

## Che cos'è

Un tool di cattura e analisi dei pacchetti basato su Python che intercetta il traffico di rete in tempo reale, identifica i protocolli, tiene traccia dell'uso della banda e genera report visivi. Costruito con Scapy per la cattura dei pacchetti, Rich per l'output nel terminale e Matplotlib per i grafici.

## Perché è importante

La visibilità sulla rete è il fondamento del security monitoring. Se non riesci a vedere cosa succede sulla tua rete, non puoi rilevare intrusioni, esfiltrazione di dati o violazioni delle policy. Questo progetto ti insegna come funziona davvero la cattura dei pacchetti a livello di kernel, non solo come eseguire Wireshark.

**Scenari reali in cui si applica:**

- **Incident response:** durante il breach di Target del 2013, il monitoraggio della rete avrebbe potuto rilevare connessioni insolite tra i sistemi POS e server esterni. L'analisi a livello di pacchetto mostra quali dati stanno lasciando la tua rete e dove stanno andando.

- **Troubleshooting delle prestazioni:** quando le applicazioni rallentano, le packet capture rivelano se il problema è la latenza di rete, le ritrasmissioni o problemi a livello applicativo. I team di rete usano questi tool ogni giorno per diagnosticare problemi di connettività.

- **Baseline di sicurezza:** non puoi rilevare anomalie senza sapere com'è la normalità. Gli analizzatori di pacchetti stabiliscono pattern di traffico di riferimento, mostrando le distribuzioni tipiche dei protocolli, l'uso della banda e i pattern di comunicazione nella tua rete.

## Cosa imparerai

Questo progetto ti insegna come funziona la cattura dei pacchetti di rete a livello di sistema. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**

- **Accesso ai raw socket** - Perché la cattura dei pacchetti richiede privilegi di root/amministratore, cosa fa CAP_NET_RAW su Linux, e come BPF (Berkeley Packet Filter) consente un filtraggio efficiente a livello di kernel senza copiare ogni pacchetto nello userspace.

- **Ispezione dei layer di protocollo** - Come sezionare i pacchetti dal Layer 2 (frame Ethernet con indirizzi MAC) fino al Layer 7 (richieste HTTP), capendo quali informazioni esistono a ogni layer e perché gli attaccanti prendono di mira layer specifici.

- **Definizione di una baseline di rete** - Costruire profili statistici del traffico normale per identificare le anomalie. Terrai traccia delle distribuzioni dei protocolli, dei pattern di banda e del comportamento degli endpoint che i team di sicurezza usano per il rilevamento delle minacce.

**Competenze tecniche:**

- **Pattern di threading producer-consumer** - Implementare un'elaborazione dei pacchetti thread-safe in cui un thread cattura i pacchetti alla velocità della rete mentre un altro li analizza senza perdere dati. Userai Queue e threading.Lock di Python per la sincronizzazione.

- **Filtraggio dei pacchetti a livello di kernel** - Scrivere filtri BPF che girano nel kernel per scartare in modo efficiente i pacchetti indesiderati prima che raggiungano lo userspace. È così che i sistemi di monitoraggio in produzione gestiscono gigabit di traffico senza sovraccaricare la CPU.

- **Raccolta di dati in serie temporali** - Campionare la banda e il tasso di pacchetti a intervalli regolari per costruire grafici che mostrino i pattern di traffico nel tempo. Fondamentale per rilevare attacchi DDoS o trasferimenti di dati insoliti.

**Tool e tecniche:**

- **Manipolazione dei pacchetti con Scapy** - Usare la libreria di packet crafting più potente di Python per catturare e sezionare il traffico di rete. Lavorerai con il sistema di layer di Scapy per estrarre indirizzi IP, porte, tipi di protocollo e dati del payload dai pacchetti grezzi.

- **Interfacce da terminale con Rich** - Costruire dashboard in tempo reale che si aggiornano durante la cattura dei pacchetti, mostrando distribuzioni dei protocolli, top talker e uso della banda con tabelle colorate e indicatori di avanzamento.

- **Visualizzazione con Matplotlib** - Generare grafici a torta della distribuzione dei protocolli, timeline della banda e grafici a barre dei top talker a partire dai dati di cattura dei pacchetti. Le stesse visualizzazioni che gli analisti SOC usano per presentare il comportamento della rete.

## Prerequisiti

Prima di iniziare, dovresti capire:

**Conoscenze richieste:**

- **Basi di Python** - Devi saper leggere codice che usa dataclass, type hint, pattern async/await e context manager. Se `with open() as f:` o `async def function():` ti sembrano poco familiari, ripassa prima i fondamenti di Python.

- **Networking TCP/IP** - Sapere cos'è un indirizzo IP, capire la differenza tra TCP e UDP, riconoscere le porte comuni (80 per HTTP, 443 per HTTPS, 53 per DNS). Dovresti essere in grado di spiegare cosa fa un three-way handshake.

- **Dimestichezza con la riga di comando** - Questo è un tool CLI. Eseguirai comandi in un terminale, passerai argomenti, imposterai variabili d'ambiente e leggerai l'output. Si dà per scontata la navigazione di base nella shell (cd, ls, cat).

**Tool di cui avrai bisogno:**

- **Python 3.14+** - Il progetto usa funzionalità moderne di Python come le istruzioni match e i type hint migliorati. Le versioni precedenti non funzioneranno.

- **Accesso root/admin** - La cattura dei pacchetti richiede i permessi per i raw socket. Su Linux ti serve root o la capability CAP_NET_RAW. Su macOS ti serve root o l'accesso ai dispositivi /dev/bpf. Su Windows ti servono privilegi di Amministratore e Npcap installato.

- **Scapy, Rich, Matplotlib** - Installali tramite pip. Scapy esegue la cattura dei pacchetti, Rich rende gradevole l'output nel terminale, Matplotlib genera i grafici.

**Utili ma non obbligatori:**

- **Esperienza con Wireshark** - Se hai usato Wireshark per analizzare file pcap, riconoscerai concetti come le gerarchie di protocolli, le espressioni di filtro e il tracciamento delle conversazioni. Ma non è necessario.

- **Programmazione di sistema** - Capire come funzionano le system call, cosa fa il kernel rispetto allo userspace e perché i context switch sono costosi ti aiuterà ad apprezzare le scelte architetturali. Non è però necessario per costruire il progetto.

## Avvio rapido

Fai girare il progetto in locale:

```bash
# Navigate to the project directory
cd network-traffic-analyzer

# Install dependencies
pip install -e .

# List available network interfaces
sudo netanal interfaces

# Capture 50 packets on your loopback interface
sudo netanal capture -i lo -c 50 --verbose

# Analyze an existing pcap file
netanal analyze traffic.pcap --top-talkers 20

# Generate charts from captured data
netanal chart traffic.pcap --type all -d ./charts/
```

Output atteso: vedrai uno stream di pacchetti in tempo reale che mostra IP di origine/destinazione, protocolli e dimensioni dei pacchetti. Al termine della cattura, ottieni statistiche riassuntive che mostrano la distribuzione dei protocolli, i top talker per volume di traffico e i grafici della banda.

## Struttura del progetto

```
network-traffic-analyzer/
├── src/netanal/
│   ├── capture.py        # Producer-consumer packet capture engine
│   ├── analyzer.py       # Protocol identification and packet parsing
│   ├── filters.py        # BPF filter builder with validation
│   ├── statistics.py     # Thread-safe stats collector
│   ├── models.py         # Data structures (PacketInfo, Protocol enum)
│   ├── visualization.py  # Matplotlib chart generation
│   ├── export.py         # JSON/CSV data export
│   ├── output.py         # Rich console formatting
│   ├── main.py           # Typer CLI command definitions
│   ├── constants.py      # Configuration values
│   └── exceptions.py     # Custom exception hierarchy
├── tests/
│   ├── test_filters.py   # BPF filter builder tests
│   └── test_models.py    # Data model tests
└── pyproject.toml        # Project dependencies and metadata
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per imparare i fondamenti della cattura dei pacchetti, dell'analisi dei protocolli e del monitoraggio di rete
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere il pattern producer-consumer e il design thread-safe
3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per spiegazioni dettagliate del codice con numeri di riga
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee come l'aggiunta del riassemblaggio degli stream TCP o del rilevamento delle anomalie

## Problemi comuni

**Permesso negato durante la cattura dei pacchetti**
```
PermissionError: [Errno 1] Operation not permitted
```
Soluzione: la cattura dei pacchetti richiede privilegi di root. Esegui con `sudo netanal capture` oppure aggiungi la capability CAP_NET_RAW al tuo binario Python su Linux: `sudo setcap cap_net_raw+ep $(which python3)`

**Npcap non installato (solo Windows)**
```
NpcapNotFoundError: Npcap is not installed
```
Soluzione: scarica e installa Npcap da https://npcap.com. È il driver di cattura dei pacchetti di Windows. WinPcap è deprecato e non funzionerà con le versioni moderne di Scapy.

**Nessun pacchetto catturato sull'interfaccia wireless**
```
Total Packets: 0
```
Soluzione: molti adattatori wireless non supportano la modalità promiscua, oppure il tuo sistema operativo la blocca. Prova prima a catturare sull'interfaccia di loopback (`lo` su Linux/Mac, `Loopback Pseudo-Interface 1` su Windows) per verificare che il tool funzioni. Per il wireless, potresti aver bisogno della monitor mode, che richiede tool diversi.

## Progetti correlati

Se lo hai trovato interessante, dai un'occhiata a:

- **Port Scanner** - Si basa sulla programmazione di rete sondando attivamente le porte invece di monitorare passivamente. Usa i raw socket per costruire pacchetti TCP/UDP personalizzati.

- **Intrusion Detection System** - Porta l'analisi dei pacchetti più avanti confrontando il traffico con le firme di attacchi noti. Insegna il pattern matching e la generazione di alert.

- **SSL/TLS Inspector** - Analizza le connessioni cifrate esaminando i certificati e i metadati dell'handshake senza decifrare il payload. Mostra cosa è visibile anche nel traffico cifrato.
