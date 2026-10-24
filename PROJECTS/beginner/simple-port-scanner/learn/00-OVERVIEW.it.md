# Simple Port Scanner

## Che cos'è

Un port scanner TCP concorrente scritto in C++ che sonda gli host target per identificare le porte aperte, chiuse e filtrate. Usa l'I/O asincrono per scansionare più porte simultaneamente e tenta di catturare i banner dei servizi per il fingerprinting.

## Perché è importante

Il port scanning è il primo passo di quasi ogni valutazione della sicurezza di rete e di ogni penetration test. Prima di poter sfruttare un sistema, devi sapere cosa è in ascolto. Questo tool ti insegna come gli attaccanti enumerano i servizi di rete e come i difensori possono rilevare questo tipo di ricognizione.

**Scenari reali in cui si applica:**

- **Ricognizione iniziale nel penetration testing** - Ogni pentest inizia con dei port scan per mappare la superficie d'attacco. Tool come Nmap sono lo standard, ma capire come funzionano dietro le quinte ti rende un tester migliore.

- **Preparazione agli audit di sicurezza** - Prima di un audit di compliance (PCI-DSS, SOC 2), devi verificare quali porte sono esposte. Porte aperte inattese indicano spesso shadow IT o configurazioni errate che fanno fallire gli audit.

- **Incident response e threat hunting** - Quando indaghi su una violazione, scansioni le reti interne per trovare backdoor, canali C2 o artefatti di movimento laterale. Gli attaccanti aprono spesso porte non standard per la persistenza.

## Cosa imparerai

Questo progetto ti insegna come funziona la ricognizione di rete a livello TCP. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**

- **Stati delle porte e loro significato** - La differenza tra porte aperte, chiuse e filtrate ti dice qualcosa sia sul servizio sia sul firewall. Aperta significa che un servizio è in ascolto, chiusa significa che non c'è niente ma l'host ha risposto, filtrata significa che un firewall ha scartato silenziosamente i tuoi pacchetti.

- **Meccanica della connessione TCP** - Il port scanning sfrutta il three-way handshake di TCP. Capire i pacchetti SYN, SYN-ACK e RST è fondamentale per la sicurezza di rete.

- **Banner grabbing per il fingerprinting** - I servizi spesso si annunciano (stringhe di versione SSH, header del server HTTP). Queste informazioni aiutano gli attaccanti a scegliere gli exploit e aiutano i difensori a identificare software obsoleto.

**Competenze tecniche:**

- **Programmazione con I/O asincrono** - Scansionare decine di migliaia di porte in sequenza richiederebbe ore. Questo progetto usa operazioni asincrone per sondare centinaia di porte in modo concorrente, completando scansioni complete in pochi secondi.

- **Pattern di programmazione concorrente** - Gestire più operazioni asincrone con stato condiviso richiede un coordinamento attento. Userai strand executor e shared pointer per prevenire le race condition.

- **Programmazione con i socket di rete** - Le operazioni dirette sui socket TCP ti insegnano cosa succede sotto HTTP e gli altri protocolli applicativi. Questa conoscenza di basso livello è essenziale per il lavoro sulla sicurezza di rete.

**Strumenti e tecniche:**

- **Boost.Asio per l'I/O di rete** - Libreria di I/O asincrono standard del settore, usata in sistemi di produzione. Imparare Asio ti insegna pattern applicabili a qualsiasi applicazione di rete ad alte prestazioni.

- **Rilevamento del filtering basato sul timeout** - Distinguere tra porte chiuse (rifiuto attivo) e porte filtrate (scarto silenzioso) richiede un'analisi dei tempi. Questa tecnica si applica al fingerprinting dei firewall e all'evasione degli IDS.

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**

- **Programmazione C++ di base** - Devi avere familiarità con classi, smart pointer (`std::shared_ptr`) e funzioni lambda. Questo progetto usa funzionalità di C++20 come le structured binding.

- **Fondamenti di networking** - Sapere cosa sono un indirizzo IP e un numero di porta, capire la differenza tra TCP e UDP e avere una comprensione di base dell'handshake TCP (SYN, SYN-ACK, ACK).

- **Dimestichezza con la riga di comando** - Compilerai con CMake ed eseguirai lo scanner dal terminale. Una familiarità di base con bash e con i build system aiuta.

**Strumenti di cui avrai bisogno:**

- **CMake 3.31+** - Build system per progetti C++. Installalo tramite il package manager (`apt install cmake` su Ubuntu, `brew install cmake` su macOS).

- **Compilatore C++20** - GCC 10+, Clang 12+ o MSVC 2019+. Il progetto usa funzionalità della libreria standard di C++20.

- **Librerie Boost** - In particolare Boost.Asio per l'I/O asincrono e Boost.Program_options per il parsing della CLI. Installale con `apt install libboost-all-dev` o `brew install boost`.

**Utili ma non obbligatori:**

- **Wireshark o tcpdump** - Gli strumenti di cattura dei pacchetti ti permettono di vedere i pacchetti TCP effettivi che il tuo scanner invia. Guardare i pacchetti SYN che volano aiuta a capire cosa succede sul cavo.

- **Familiarità con Nmap** - Se hai già usato Nmap, riconoscerai concetti come le SYN scan e il rilevamento dei servizi. Questo progetto implementa versioni semplificate di quelle tecniche.

## Avvio rapido

Fai girare il progetto in locale:
```bash
# Clone and navigate
cd PROJECTS/beginner/simple-port-scanner

# Create build directory
mkdir build && cd build

# Configure and build
cmake ..
make

# Run the scanner on localhost
./simplePortScanner -i 127.0.0.1 -p 1-1024

# Scan specific ports with custom settings
./simplePortScanner -i scanme.nmap.org -p 80,443,8080 -t 50 -e 3
```

Output atteso: una tabella che mostra il numero di porta, lo stato (OPEN/CLOSED/FILTERED), il nome del servizio se riconosciuto e l'eventuale banner catturato dal servizio. Le porte aperte appaiono in verde, quelle chiuse in rosso.

## Struttura del progetto
```
simple-port-scanner/
├── src/
│   ├── PortScanner.hpp      # Class definition, member variables, method signatures
│   └── PortScanner.cpp      # Core scanning logic, async operations, banner grabbing
├── main.cpp                 # Entry point, CLI argument parsing with boost::program_options
└── CMakeLists.txt           # Build configuration, dependencies (Boost)
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per conoscere gli stati delle porte TCP, il banner grabbing e le tecniche di ricognizione di rete.

2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere come sono progettati l'I/O asincrono e la scansione concorrente.

3. **Percorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per una spiegazione dettagliata dell'algoritmo di scansione e dei pattern asincroni.

4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee come la scansione UDP, l'OS fingerprinting e le tecniche stealth.

## Problemi comuni

**"boost/asio.hpp: No such file or directory"**
```
fatal error: boost/asio.hpp: No such file or directory
```
Soluzione: installa le librerie di sviluppo di Boost. Su Ubuntu/Debian: `sudo apt install libboost-all-dev`. Su macOS: `brew install boost`. Su Windows, scaricale da boost.org e configura CMake con `-DBOOST_ROOT=C:\path\to\boost`.

**"Connection refused" su tutte le porte**
```
1	CLOSED	---	---
22	CLOSED	SSH	---
80	CLOSED	HTTP	---
```
Soluzione: è normale se stai scansionando una macchina senza servizi in esecuzione. Prova a scansionare `scanme.nmap.org`, che ha porte aperte intenzionalmente a scopo di test, oppure scansiona la tua macchina dopo aver avviato un web server (`python3 -m http.server 8000`).

**Lo scanner si blocca o è molto lento**
Soluzione: il tuo firewall potrebbe applicarti un rate limiting. Riduci il numero di thread (`-t 10` invece del default 100) e aumenta il timeout (`-e 5`). Assicurati inoltre di non scansionare da una rete che blocca le connessioni in uscita.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:

- **packet-sniffer** - Cattura e analizza pacchetti di rete grezzi. Il port scanning ha più senso quando puoi vedere gli scambi SYN/ACK.
- **basic-firewall** - Implementa regole per bloccare i port scan. Capire entrambi i lati (scansione e blocco) ti dà una prospettiva completa sulla sicurezza di rete.
