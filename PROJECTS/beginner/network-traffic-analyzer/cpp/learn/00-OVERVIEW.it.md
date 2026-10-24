# C++ Network Traffic Analyzer

## Che cos'è

Un tool CLI in C++20 che cattura traffico di rete dal vivo o legge file pcap offline, analizza manualmente i frame grezzi Ethernet/IP/TCP/UDP e mostra statistiche in tempo reale in una UI da terminale completamente interattiva. Costruito con libpcap per la cattura dei pacchetti a livello kernel, Boost per il parsing della CLI e FTXUI per la TUI.

## Perché è importante

La visibilità sulla rete è il modo in cui i difensori scoprono gli attaccanti. Se non riesci a vedere cosa attraversa la tua rete, non puoi rilevare intrusioni, esfiltrazione di dati o movimenti laterali. Tool come Wireshark, Zeek e Suricata poggiano tutti sulla stessa base: libpcap.

**Scenari reali in cui si applica:**

- **Incident response:** Durante il breach di Target del 2013, 40 milioni di numeri di carta sono stati esfiltrati attraverso i sistemi POS tramite il malware BlackPOS, che usava normali connessioni TCP verso IP esterni. La visibilità a livello di pacchetto avrebbe segnalato il traffico in uscita inatteso dalle casse dei negozi.

- **Rilevamento di APT:** L'attacco SolarWinds del 2020 (CVE-2020-10148) usava beaconing HTTP — check-in periodici dagli host compromessi verso server controllati dall'attaccante. Il rilevamento di anomalie a livello di pacchetto lo intercetta: host che non hanno mai parlato con IP esterni improvvisamente iniziano a farlo.

- **Baseline dei protocolli:** Non puoi rilevare ciò che è anomalo senza prima sapere cosa è normale. Gli analizzatori di pacchetti stabiliscono distribuzioni di baseline — quanto è DNS rispetto a HTTPS rispetto a SMTP — così che spostamenti inattesi si registrino come alert.

## Cosa imparerai

**Concetti di sicurezza:**

- **Accesso ai raw socket e capability** — Perché la cattura dei pacchetti richiede root o `CAP_NET_RAW`, cosa sono le capability di Linux, e come BPF (Berkeley Packet Filter) permette al kernel di scartare i pacchetti prima ancora che raggiungano lo userspace.

- **Parsing degli header di protocollo** — Come percorrere manualmente la catena frame Ethernet → header IP → header TCP/UDP, usando offset di byte e `reinterpret_cast`. È ciò che fa internamente ogni IDS, engine DPI e firewall.

- **Espressioni di filtro BPF** — Come scrivere e compilare filtri che girano nel kernel (es. `tcp and port 443 and host 192.168.1.1`), e perché il filtraggio lato kernel è di ordini di grandezza più veloce del filtraggio in userspace.

**Pattern C++:**

- **Statistiche protette da mutex** — Aggregazione thread-safe con `std::mutex` e un pattern di snapshot lock-copy-return che previene le data race senza bloccare il thread di rendering.

- **Parsing IP polimorfico** — Una base astratta `IP_class` con le sottoclassi `IPv4` e `IPv6`, costruite a partire dai byte grezzi del pacchetto e che fanno dispatch del parsing del livello di trasporto nel costruttore.

- **RAII per le risorse C** — Avvolgere un handle C-style `pcap_t*` in `std::unique_ptr<pcap_t, decltype(&pcap_close)>` così che l'handle venga rilasciato automaticamente indipendentemente da come la funzione termina.

- **Threading con l'event loop di FTXUI** — Eseguire la cattura dei pacchetti in un thread in background mentre FTXUI gestisce il proprio event loop sul thread principale, coordinati con atomic e un mutex di rendering condiviso.

## Prerequisiti

**Richiesti:**

- **Basi di C++20** — Devi saper leggere codice che usa structured binding, `std::format`, ranges, `std::atomic` e lambda. Se `auto [key, val] : map` ti sembra poco familiare, ripassa prima il C++ moderno.

- **Networking TCP/IP** — Conosci lo stack di livelli Ethernet → IP → TCP/UDP. Capisci cosa sono indirizzi IP e numeri di porta, cosa fa un three-way handshake, come ICMP differisce da TCP.

- **Riga di comando Linux** — Eseguirai comandi, ispezionerai le interfacce di rete con `ip link` e concederai capability con `setcap`. Si dà per scontata la navigazione di base nella shell.

**Tool di cui avrai bisogno:**

- **Linux (Ubuntu/Debian/Arch/Fedora)** — L'engine di cattura usa header specifici di Linux (`netinet/tcp.h`, `netinet/ip.h`). macOS funzionerà con piccole modifiche; Windows no.

- **Root o CAP_NET_RAW** — La cattura dei pacchetti lo richiede. Esegui con `sudo` oppure concedi la capability al binario: `sudo setcap cap_net_raw,cap_net_admin=eip ./network-traffic-analyzer`.

- **libpcap, Boost, Ninja, CMake** — Gestiti da `install.sh`.

**Utili ma non richiesti:**

- **Esperienza con Wireshark** — Se hai letto file pcap o scritto display filter BPF, riconoscerai subito i concetti. Non necessaria per costruire il progetto.

- **Background di programmazione di sistema** — Capire il dispatch virtuale, le vtable e l'aritmetica dei puntatori a livello di byte ti aiuterà a seguire `IP.cpp`. Non richiesto, ma accelera la comprensione.

## Avvio rapido

```bash
# Clone the repo
git clone https://github.com/CarterPerez-dev/Cybersecurity-Projects.git
cd PROJECTS/beginner/network-traffic-analyzer/cpp

# Install dependencies and build (one command)
./install.sh

# List available interfaces
just interfaces

# Live capture on eth0
just capture -i eth0

# Capture 100 packets and export
just run -i wlan0 -c 100 --json result.json

# Analyze a pcap file offline
just run --offline traffic.pcap

# Run clang-tidy static analysis
just lint

# Auto-format all source files
just format
```

Output atteso: la TUI si avvia a schermo intero, mostrando una tabella aggiornata in tempo reale dei protocolli di trasporto, dei protocolli applicativi, degli IP principali, delle principali coppie sorgente→destinazione e un grafico della banda. Premi `q` o `Escape` per uscire. All'uscita, i risultati vengono esportati in JSON/CSV se sono stati passati i flag.

## Struttura del progetto

```
cpp/
├── main.cpp                     # Entry point — arg parsing, TUI setup, thread coordination
├── CMakeLists.txt               # Build definition
├── CMakePresets.json            # Debug/release presets with compile_commands.json export
├── Justfile                     # Dev commands (build, run, lint, format, clean)
├── install.sh                   # One-command setup: deps + build
├── .clang-tidy                  # Static analysis config
├── .clang-format                # Code style config
├── include/
│   ├── capture/pcapCapture.hpp  # PcapCapture — libpcap wrapper
│   ├── cli/
│   │   ├── argsParse.hpp        # argsParser — Boost.Program_options wrapper
│   │   └── filter.hpp           # filter struct + filter_type enum
│   ├── packet/
│   │   ├── packet.hpp           # Packet struct, protocol enums
│   │   └── IP.hpp               # IP_class, IPv4, IPv6 declarations
│   ├── stats/protocolStats.hpp  # Stats class, StatsSnapshot, all stat structs
│   └── TUI/view.hpp             # View — FTXUI renderer
└── src/
    ├── capture/pcapCapture.cpp  # Capture engine implementation
    ├── cli/
    │   ├── argsParse.cpp        # CLI option definitions
    │   └── filter.cpp           # BPF filter builder
    ├── packet/
    │   ├── packet.cpp           # Application protocol identification
    │   └── IP.cpp               # IPv4/IPv6 parsing implementation
    ├── stats/protocolStats.cpp  # Statistics aggregation, export
    └── TUI/view.cpp             # TUI layout and rendering
```

## Prossimi passi

1. **Comprendi i concetti** — Leggi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per capire come funzionano libpcap, BPF e il parsing dei protocolli a livello kernel
2. **Studia l'architettura** — Leggi [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) per il modello di threading e il design dei componenti
3. **Percorri il codice** — Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per le analisi riga per riga di ogni componente principale
4. **Estendilo** — Leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per idee su come aggiungere il riassemblaggio degli stream TCP, il rilevamento di anomalie e altro

## Problemi comuni

**Permesso negato:**
```
pcap_open_live failed: eth0: You don't have permission to capture on that device
```
Esegui con `sudo just run` oppure concedi le capability: `sudo setcap cap_net_raw,cap_net_admin=eip ./build/release/network-traffic-analyzer`

**Nessun pacchetto sull'interfaccia wireless:**
Molti driver Wi-Fi non passano tutti i frame in modalità managed. Prova prima `lo` (loopback) per verificare che il tool funzioni, poi prova la tua interfaccia cablata.

**clang-tidy non trova compile_commands.json:**
Esegui prima `just build` (preset debug) — genera `build/debug/compile_commands.json` con `CMAKE_EXPORT_COMPILE_COMMANDS=ON`. Il comando `just lint` legge da `build/release/compile_commands.json`, quindi esegui anche una build release se necessario.
