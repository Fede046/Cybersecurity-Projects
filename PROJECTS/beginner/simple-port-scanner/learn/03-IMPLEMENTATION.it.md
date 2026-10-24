# Guida all'implementazione

Questo documento percorre il codice reale, spiegando come funziona dietro le quinte il port scanning asincrono e mettendo in evidenza le parti delicate che fanno funzionare correttamente l'I/O concorrente.

## Walkthrough della struttura dei file
```
simple-port-scanner/
├── src/
│   ├── PortScanner.hpp     # Class definition: member variables, async I/O primitives, method signatures
│   └── PortScanner.cpp     # Implementation: async scan logic, completion handlers, banner grabbing
├── main.cpp                # Entry point: CLI parsing, scanner initialization, blocking run() call
└── CMakeLists.txt          # Build config: C++20 standard, Boost dependency with program_options
```

## Costruire l'interfaccia CLI

### Passo 1: parsing degli argomenti

Cosa stiamo costruendo: un'interfaccia a riga di comando user-friendly con default sensati

Crea o esamina `main.cpp`:
```cpp
// main.cpp:7-17
po::options_description desc("Allowed options");
desc.add_options()
    ("help,h", "produce help message")
    ("dname,i", po::value<std::string>()->default_value("127.0.0.1"), "set domain name or IP address")
    ("ports,p", po::value<std::string>()->default_value("1-1024"), "set a port range from 1 to n")
    ("threads,t", po::value<int>()->default_value(100), "max concurrent threads")
    ("expiry_time,e", po::value<uint8_t>()->default_value(2)->value_name("sec"), "timeout in seconds")
    ("verbose,v", "verbose output");
```

**Perché questo codice funziona:**
- `po::value<T>()->default_value(X)`: parsing dei parametri type-safe con validazione automatica. Se l'utente passa "-t hello", Boost lancia un'eccezione invece di andare in crash.
- Forme brevi e lunghe delle opzioni (`-i` e `--dname`): la convenzione Unix standard fa sembrare il tool professionale.
- `uint8_t` per expiry_time: impone l'intervallo 0-255 secondi. Timeout oltre i 4 minuti non hanno senso per il port scanning.

**Errori comuni qui:**
```cpp
// Wrong - no defaults means required parameters
desc.add_options()
    ("dname", po::value<std::string>(), "IP address");

// User must ALWAYS provide -i, which is annoying for testing localhost

// Right - defaults make tool usable without memorizing flags
desc.add_options()
    ("dname", po::value<std::string>()->default_value("127.0.0.1"), "IP address");
```

### Passo 2: mostrare l'aiuto

Ora dobbiamo fornire un testo di aiuto utile con degli esempi.

In `main.cpp` (righe 23-34):
```cpp
if (vm.count("help")) {
    std::cout << desc << "\n";
    std::cout << "Examples:\n"
          << "  Scan common ports on localhost:\n"
          << "    ./port_scanner -i 127.0.0.1 -p 1-1024\n\n"
          << "  Full TCP port scan:\n"
          << "    ./port_scanner -i 192.168.1.1 -p 65535 -t 200\n\n"
          << "  Postscriptum:\n"
          << "  Scan only systems you own or have explicit permission to test.\n";
    return 0;
}
```

**Cosa sta succedendo:**
1. Controlla se l'utente ha passato il flag `-h` o `--help`
2. Stampa le descrizioni delle opzioni generate automaticamente da `desc`
3. Aggiunge esempi d'uso concreti (fondamentale - le persone imparano dagli esempi, non dalle descrizioni astratte)
4. Include un'avvertenza legale/etica (obbligatoria per gli strumenti di sicurezza)

**Perché lo facciamo così:**
Boost.Program_Options genera le descrizioni automaticamente, ma gli esempi vanno scritti a mano. Gli utenti copiano e incollano gli esempi per imparare, quindi forniamo scenari realistici (porte comuni, scansione completa, timeout personalizzato).

**Approcci alternativi:**
- Formato man page: più formale ma richiede di mantenere una documentazione separata
- Prompt interattivi: più amichevoli per i principianti ma fastidiosi per chi scrive script e vuole strumenti non interattivi

### Passo 3: passare la configurazione allo scanner

Estrai gli argomenti validati e inizializza lo scanner:
```cpp
// main.cpp:36-40
std::string ip = vm["dname"].as<std::string>();
std::string port = vm["ports"].as<std::string>();
int threads = vm["threads"].as<int>();
uint8_t expiry_time = vm["expiry_time"].as<uint8_t>();

PortScanner scanner;
scanner.set_options(ip, port, threads, expiry_time);
```

Questo pattern (costruttore di default + `set_options`) permette di riusare un oggetto scanner per più scansioni. L'alternativa sarebbe passare tutto al costruttore, ma è meno flessibile per l'uso interattivo.

## Costruire lo scanner principale

### L'algoritmo di scansione

File: `src/PortScanner.cpp`

Il cuore dello scanner è il metodo `scan()`, che implementa un pattern asincrono auto-pianificante:
```cpp
// PortScanner.cpp:123-165
void PortScanner::scan() {
    if (q.empty() || cnt >= MAX_THREADS) return;  // Bail out if no work or at thread limit
    
    uint16_t port = q.front();
    q.pop();
    ++cnt;  // Increment active worker count
    
    auto socket = std::make_shared<tcp::socket>(io);
    auto timer = std::make_shared<boost::asio::steady_timer>(io);
    auto complete = std::make_shared<bool>(false);  // Race condition flag
    
    tcp::endpoint endpoint(this->endpoint.address(), port);
    
    timer->expires_after(std::chrono::seconds(expiry_time));
    
    // Timer handler - races against connection
    timer->async_wait(boost::asio::bind_executor(strand, 
        [this, complete, socket, port](boost::system::error_code ec) {
            if (!ec && !*complete)  {
                *complete = true;
                socket->close();
                printf("%i\t%s\t%s\t%s\n", port, "FILTERED", "NULL", "NULL");
                ++filtered_ports;
                --cnt;
                scan();  // Recursively grab next port
            }
        }));
    
    // Connection handler - races against timer
    socket->async_connect(endpoint, boost::asio::bind_executor(strand,
        [this, socket, timer, port, complete](boost::system::error_code ec) {
            if (*complete) return;  // Lost the race, timer already fired
            *complete = true;
            timer->cancel();  // Won the race, stop timer
            
            std::string service = "---";
            auto banner = std::make_shared<std::string>("---");
            
            // Look up service name
            auto it = basicPorts.find(port);
            if (it != basicPorts.end()) {
                service = it->second;
            }
            
            if (!ec) {
                // Connection succeeded - port is OPEN
                auto buf = std::make_shared<std::array<char, 128>>();
                
                socket->async_read_some(boost::asio::buffer(*buf),
                    boost::asio::bind_executor(strand,
                    [this, port, buf, banner, service](boost::system::error_code ec, std::size_t n) {
                        if (!ec && n > 0) {
                            banner->assign(buf->data(), n);
                        }
                        printf("%i\t%sOPEN%s\t%s\t%s\n", port, GREEN, RESET, service.c_str(), banner->c_str());
                        ++open_ports;
                        --cnt;
                        scan();  // Next port
                    }));
            } else {
                // Connection failed - port is CLOSED
                printf("%i\t%sCLOSED%s\t%s\t%s\n", port, RED, RESET, service.c_str(), banner->c_str());
                ++closed_ports;
                --cnt;
                scan();  // Next port
            }
        }));
}
```

**Le parti chiave spiegate:**

**Guard clause** (`line 123-124`):
```cpp
if (q.empty() || cnt >= MAX_THREADS) return;
```
Questo impedisce di generare infiniti worker. Se la coda è vuota, abbiamo finito. Se siamo al limite di thread, non avviamo un'altra scansione anche se restano porte (i worker già in esecuzione chiameranno prima o poi di nuovo `scan()`).

**Gestione del ciclo di vita con gli shared pointer** (`lines 125-127`):
```cpp
auto socket = std::make_shared<tcp::socket>(io);
auto timer = std::make_shared<boost::asio::steady_timer>(io);
auto complete = std::make_shared<bool>(false);
```
Questi oggetti devono sopravvivere all'operazione asincrona. Catturare gli shared pointer nelle closure delle lambda incrementa i reference count, mantenendo in vita gli oggetti finché i completion handler non terminano. Senza questo, socket/timer potrebbero essere distrutti mentre le operazioni asincrone sono in sospeso (use-after-free).

**Coordinamento della gara con il flag di completamento** (`line 127, 131, 139`):
```cpp
auto complete = std::make_shared<bool>(false);

// In timer handler:
if (!ec && !*complete) {
    *complete = true;  // I won!
    socket->close();
    // ...
}

// In connect handler:
if (*complete) return;  // I lost, timer already won
*complete = true;  // I won!
timer->cancel();
```
Entrambi gli handler controllano e impostano `complete` in modo atomico (protetti dallo strand). Quello che scatta per primo imposta il flag, e quello che perde ritorna subito. Questo evita di elaborare due volte la stessa porta.

**Distribuzione del lavoro con ricorsione in coda** (` lines 136, 151, 158`):
Ogni completion handler termina con `scan()`. Questo implementa un pattern di work-stealing - non appena una porta termina, quel worker prende la porta successiva dalla coda. Non serve alcun dispatcher centrale.

**Perché questa specifica implementazione:**

La gara timer/socket risolve in modo elegante il rilevamento delle porte filtrate. Senza il timer, aspetteremmo per sempre sulle porte filtrate (il firewall scarta i pacchetti, nessuna risposta). Il timer scatta dopo `expiry_time` secondi se il socket non si è connesso, marcando la porta come filtrata.

Le chiamate ricorsive a `scan()` fanno sì che non creiamo mai più operazioni asincrone di `MAX_THREADS`. Avviamo `MAX_THREADS` scansioni, e ogni completamento crea esattamente una nuova scansione, mantenendo una concorrenza costante.

**Errori comuni qui:**
```cpp
// Wrong - would leak if async operation fails
tcp::socket socket(io);  // Stack-allocated
timer->async_wait([&socket](...) {
    socket.close();  // If timer fires after function returns, socket is destroyed, crash!
});

// Right - shared pointer keeps it alive
auto socket = std::make_shared<tcp::socket>(io);
timer->async_wait([socket](...) {  // Captures shared_ptr, extends lifetime
    socket->close();  // Safe even if outer function returned
});
```

## Implementazione della sicurezza

### Banner grabbing

File: `PortScanner.cpp:143-151`
```cpp
auto buf = std::make_shared<std::array<char, 128>>();

socket->async_read_some(boost::asio::buffer(*buf),
    boost::asio::bind_executor(strand,
    [this, port, buf, banner, service](boost::system::error_code ec, std::size_t n) {
        if (!ec && n > 0) {
            banner->assign(buf->data(), n);
        }
        printf("%i\t%sOPEN%s\t%s\t%s\n", port, GREEN, RESET, service.c_str(), banner->c_str());
        // ...
    }));
```

**Cosa previene:**
Nulla - il banner grabbing è una tecnica offensiva, non una difesa. Ma capirlo ti aiuta a mettere in sicurezza i tuoi servizi.

**Come funziona:**
1. Dopo una connessione riuscita, alloca un buffer da 128 byte
2. Chiama `async_read_some`, che ritorna immediatamente
3. Quando arrivano i dati (o si verifica un errore), scatta il completion handler
4. Se sono stati letti dei byte (`n > 0`), li copia nella stringa del banner
5. Stampa il risultato con il contenuto del banner

**Cosa succede se lo rimuovi:**
Rileveresti comunque le porte aperte ma non sapresti quale software è in esecuzione. Il banner "SSH-2.0-OpenSSH_7.4" ti dice che si tratta di SSH versione 7.4, che ha CVE note. Senza banner, dovresti connetterti manualmente a ogni porta aperta.

### Rilevamento del filtering basato sul timeout

File: `PortScanner.cpp:128-137`
```cpp
timer->expires_after(std::chrono::seconds(expiry_time));

timer->async_wait(boost::asio::bind_executor(strand, 
    [this, complete, socket, port](boost::system::error_code ec) {
        if (!ec && !*complete)  {
            *complete = true;
            socket->close();
            printf("%i\t%s\t%s\t%s\n", port, "FILTERED", "NULL", "NULL");
            ++filtered_ports;
            --cnt;
            scan();
        }
    }));
```

**Cosa previene:**
Blocchi infiniti sulle porte filtrate. Senza timeout, `async_connect` attende indefinitamente se un firewall scarta i pacchetti.

**Come funziona:**
1. Imposta il timer perché scada dopo `expiry_time` secondi (default 2)
2. Se il timer scatta E la connessione non si è completata (`!*complete`), la porta è filtrata
3. Chiude l'operazione sul socket in sospeso
4. Marca la porta come FILTERED

**Cosa succede se lo rimuovi:**
Lo scanner si bloccherebbe per sempre sulla prima porta filtrata. Scansioneresti la porta 1 (filtrata), aspetteresti in eterno e non arriveresti mai alla porta 2. I timeout sono essenziali per gestire i target che non rispondono.

## Esempio di flusso dei dati

Tracciamo una scansione completa della porta 22 (SSH) su un host in cui è aperta.

### La richiesta parte
```cpp
// Entry point: main.cpp:37-38
PortScanner scanner;
scanner.set_options("192.168.1.100", "22", 100, 2);
```

A questo punto:
- Il resolver DNS traduce "192.168.1.100" in un indirizzo IP (banale per gli IP)
- L'endpoint viene memorizzato come `tcp::endpoint` con l'IP
- La coda contiene una sola voce: `22`

### Lo scanner parte
```cpp
// PortScanner.cpp:111-114
for (int i = 0; i < MAX_THREADS; i++) {
    boost::asio::post(strand, [this]() {
        scan();
    });
}
```

Questo codice pubblica 100 elementi di lavoro (poiché `MAX_THREADS=100`), ma c'è solo 1 porta in coda, quindi 99 ritornano immediatamente alla guard clause. Un worker procede:
```cpp
// PortScanner.cpp:123-127
uint16_t port = 22;  // Popped from queue
q.pop();  // Queue now empty
++cnt;  // cnt = 1

auto socket = std::make_shared<tcp::socket>(io);
auto timer = std::make_shared<boost::asio::steady_timer>(io);
```

### Tentativo di connessione
```cpp
// PortScanner.cpp:128-137
timer->expires_after(std::chrono::seconds(2));
timer->async_wait([...](...) { ... });  // Scheduled, not yet fired

// PortScanner.cpp:138
socket->async_connect(endpoint, [...](...) { ... });  // Begins TCP handshake
```

Sul cavo:
1. Lo scanner invia un pacchetto SYN a 192.168.1.100:22
2. Il target risponde con SYN-ACK (SSH è in ascolto)
3. Lo scanner completa l'handshake con un ACK
4. Connessione stabilita (di solito < 100ms)

### La connessione riesce
```cpp
// PortScanner.cpp:139-151
// Completion handler fires with ec = success
if (*complete) return;  // complete=false, so continue
*complete = true;  // Set flag
timer->cancel();  // Stops timer from firing

auto it = basicPorts.find(22);  // Found: "SSH"
std::string service = "SSH";

// Port is open, try banner grab
auto buf = std::make_shared<std::array<char, 128>>();
socket->async_read_some(boost::asio::buffer(*buf), [...](...) { ... });
```

Il server SSH invia immediatamente il suo banner (requisito del protocollo):
```
SSH-2.0-OpenSSH_7.4p1 Debian-10+deb9u7
```

### Banner ricevuto
```cpp
// PortScanner.cpp:144-151
[](boost::system::error_code ec, std::size_t n) {
    if (!ec && n > 0) {  // Success, read 43 bytes
        banner->assign(buf->data(), 43);  // "SSH-2.0-OpenSSH_7.4p1 Debian-10+deb9u7"
    }
    printf("%i\t%sOPEN%s\t%s\t%s\n", 22, GREEN, RESET, "SSH", "SSH-2.0-OpenSSH_7.4p1...");
    ++open_ports;  // Statistics
    --cnt;  // Active workers now 0
    scan();  // Check queue for more work (empty, so returns immediately)
}
```

Il risultato viene stampato in verde: `22  OPEN  SSH  SSH-2.0-OpenSSH_7.4p1 Debian-10+deb9u7`

## Pattern di gestione degli errori

### Connection refused (porta chiusa)

Quando si scansiona la porta 8080 su un sistema in cui nulla è in ascolto:
```cpp
// PortScanner.cpp:153-158
else {
    // ec = "Connection refused" (ECONNREFUSED)
    printf("%i\t%sCLOSED%s\t%s\t%s\n", port, RED, RESET, service.c_str(), banner->c_str());
    ++closed_ports;
    --cnt;
    scan();
}
```

**Perché questa gestione specifica:**
Connection refused significa che il target ha inviato un pacchetto RST (porta esplicitamente chiusa). È diverso dal timeout (filtrata). La coloriamo in rosso per distinguerla visivamente dalle porte aperte.

**Cosa NON fare:**
```cpp
// Bad: catching and silencing errors
socket->async_connect(endpoint, [](boost::system::error_code ec) {
    // Ignore all errors - terrible idea
});
```

Questo nasconde problemi di rete (fallimento del DNS, route irraggiungibile) che dovrebbero essere segnalati. Controlla sempre i codici di errore.

### Timeout (porta filtrata)

Quando si scansiona la porta 12345 su un host dietro un firewall che scarta i pacchetti:
```cpp
// PortScanner.cpp:129-136
timer->async_wait([](boost::system::error_code ec) {
    if (!ec && !*complete) {  // Timer expired naturally (not cancelled)
        *complete = true;
        socket->close();  // Abort pending connection
        printf("%i\t%s\t%s\t%s\n", port, "FILTERED", "NULL", "NULL");
        ++filtered_ports;
        --cnt;
        scan();
    }
});
```

Il controllo su `ec` è cruciale - se il timer viene cancellato (perché la connessione è riuscita), `ec` è impostato e saltiamo questo handler. Solo la scadenza naturale significa filtrata.

## Ottimizzazioni delle prestazioni

### Prima: scansione sincrona

Questa implementazione ingenua sarebbe disastrosamente lenta:
```cpp
// Don't actually do this
for (int port = 1; port <= 65535; port++) {
    try {
        tcp::socket s(io);
        s.connect(tcp::endpoint(address, port));  // Blocks!
        // If we get here, port is open
    } catch (...) {
        // Port closed or filtered (can't tell which)
    }
}
```

Era lenta perché ogni `connect()` blocca per tutta la durata del timeout. Con un timeout di 2 secondi:
- 65535 porte × 2 secondi = 131.070 secondi = 36 ore (!)

Anche con connessioni da 100ms:
- 65535 porte × 0,1 secondi = 6553 secondi = 1,8 ore

### Dopo: scansione asincrona concorrente
```cpp
// PortScanner.cpp:111-115
for (int i = 0; i < MAX_THREADS; i++) {
    boost::asio::post(strand, [this]() { scan(); });
}
io.run();  // Blocks until all async ops complete
```

**Cosa è cambiato:**
- Avviate 100 operazioni asincrone simultaneamente
- Ognuna si completa in modo indipendente e ne avvia un'altra
- Tempo totale = (porte totali / concorrenza) × tempo medio di connessione
- 65535 porte / 100 worker × 0,1 secondi = 66 secondi

**Benchmark:**
- Prima (sincrono): 36 ore per una scansione completa con timeout di 2 secondi
- Dopo (100 thread): ~2 minuti per la stessa scansione
- Miglioramento: 1080× più veloce

Per scansioni su rete locale con latenza inferiore a 10ms:
- Prima: 11 minuti (65535 × 0,01s)
- Dopo: 7 secondi (throughput di 655 porte/sec)
- Miglioramento: 95× più veloce

## Gestione della configurazione

### Parsing dell'intervallo di porte
```cpp
// PortScanner.cpp:26-53
void PortScanner::parse_port(std::string& port) {
    auto t = std::find(port.begin(), port.end(), '-');
    if (t == port.end()) {
        // No dash - single port or max range
        startPort = 1;
        endPort = std::stoi(port);  // "1024" means 1-1024
        return;
    }
    
    // Parse "start-end" format
    std::string s = "", e = "";
    auto it = port.begin();
    while (it != port.end() && *it != '-') {
        s += *it;
        ++it;
    }
    ++it;  // Skip the dash
    while (it != port.end()) {
        e += *it;
        ++it;
    }
    
    int start = std::stoi(s);
    int end = std::stoi(e);
    
    // Validate bounds
    if (start == 0 || end > MAX_PORT || start > end) {
        startPort = 1;
        endPort = MAX_PORT;  // Invalid input = full scan
    } else {
        startPort = static_cast<uint16_t>(start);
        endPort = static_cast<uint16_t>(end);
    }
}
```

**Dettagli importanti:**
- **Validazione dell'input**: il controllo dei limiti garantisce che non scansioniamo la porta 0 (non valida) o porte > 65535 (impossibili)
- **Comportamento di ripiego**: un input non valido (come "5000-100") ricade su una scansione completa invece di andare in crash
- **Parsing delle stringhe**: iterazione manuale sui caratteri invece di regex (più semplice, nessuna dipendenza)

Validiamo presto perché intervalli di porte non validi causano errori strani in seguito (la coda potrebbe essere vuota, o contenere più di 65535 porte se i calcoli vanno in overflow). Fallire subito al momento della configurazione è meglio di misteriosi crash a runtime.

### Risoluzione DNS
```cpp
// PortScanner.cpp:89-92
auto result = resolver.resolve(this->domainName, "");
endpoint = *result.begin();
```

**Come funziona:**
Il resolver di Boost.Asio interroga il DNS per i record A/AAAA. Per "scanme.nmap.org" restituisce 45.33.32.156. Per indirizzi IP come "192.168.1.1", ne valida il formato e ritorna immediatamente.

**Gestione degli errori:**
Se la risoluzione fallisce (il dominio non esiste, il server DNS non è raggiungibile), `resolve()` lancia un'eccezione. È voluto - meglio fallire all'avvio che scansionare silenziosamente l'host sbagliato.

## Insidie comuni di implementazione

### Insidia 1: dimenticare di legare allo strand

**Sintomo:**
Crash casuali, statistiche corrotte, porte scansionate più volte o per niente.

**Causa:**
```cpp
// Wrong - no strand protection
socket->async_connect(endpoint, [this, port](...) {
    ++open_ports;  // RACE CONDITION!
    q.pop();       // CORRUPTS QUEUE!
});
```

Più completion handler vengono eseguiti in modo concorrente, modificando lo stato condiviso (`open_ports`, la coda) senza sincronizzazione. Questo causa data race e comportamento indefinito.

**Correzione:**
```cpp
// Right - strand serializes handlers
socket->async_connect(endpoint, boost::asio::bind_executor(strand,
    [this, port](...) {
        ++open_ports;  // Safe - only one handler runs at a time
        q.pop();       // Safe
    }));
```

**Perché è importante:**
Le data race sono assassini silenziosi. Il tuo programma potrebbe funzionare il 99% delle volte e andare in crash in modo imprevedibile in quell'1% in cui due handler entrano in gara. Usa sempre lo strand per lo stato condiviso.

### Insidia 2: catturare variabili locali per riferimento

**Sintomo:**
Crash da use-after-free, dati spazzatura nei completamenti.

**Causa:**
```cpp
void scan() {
    uint16_t port = q.front();
    socket->async_connect(endpoint, [&port](...) {  // WRONG!
        printf("Port %d\n", port);  // 'port' is destroyed when scan() returns
    });
}
```

La lambda cattura `port` per riferimento, ma `port` è una variabile locale che viene distrutta quando `scan()` ritorna. L'operazione asincrona non si è ancora completata, quindi quando l'handler finalmente viene eseguito, accede a memoria liberata.

**Correzione:**
```cpp
void scan() {
    uint16_t port = q.front();
    socket->async_connect(endpoint, [port](...) {  // Copy by value
        printf("Port %d\n", port);  // Safe - port was copied into the lambda
    });
}
```

**Perché è importante:**
La programmazione asincrona inverte il flusso di controllo. La funzione ritorna molto prima che l'handler venga eseguito. Cattura sempre per valore oppure usa gli shared pointer per gli oggetti con cicli di vita complessi.

## Suggerimenti per il debug

### Problema: "Tutte le porte risultano FILTERED"

**Problema:** ogni porta va in timeout, nulla risulta OPEN o CLOSED.

**Come fare il debug:**
1. Controlla il firewall sulla macchina che esegue la scansione - le connessioni in uscita potrebbero essere bloccate
2. Verifica che il target sia raggiungibile: `ping 192.168.1.100`
3. Fai un test con una porta notoriamente aperta: `telnet scanme.nmap.org 80` dovrebbe connettersi
4. Riduci il numero di thread e aumenta il timeout: `-t 1 -e 10` elimina i problemi di concorrenza e di rete

**Cause comuni:**
- Il firewall dell'host target scarta tutte le connessioni in ingresso (funziona come previsto)
- Un firewall di rete tra te e il target blocca il traffico di port scanning
- L'host target è spento o irraggiungibile
- Stai scansionando da una rete con restrizioni (aziendale, cloud provider) che blocca le scansioni in uscita

### Problema: "Segmentation fault nel completion handler"

**Problema:** crash con uno stack trace negli interni di Boost.Asio.

**Come fare il debug:**
1. Compila con i simboli di debug: `cmake -DCMAKE_BUILD_TYPE=Debug ..`
2. Esegui sotto valgrind: `valgrind --leak-check=full ./simplePortScanner`
3. Cerca catture per riferimento: fai un grep del codice per `[&` per trovarle
4. Verifica l'uso degli shared pointer: socket/timer allocati sullo stack causano questo problema

**Cause comuni:**
- Variabili locali catturate per riferimento (Insidia 2 sopra)
- Oggetti asincroni allocati sullo stack che vengono distrutti mentre le operazioni sono in sospeso
- Double free dovuto alla gestione manuale della memoria (andrebbe usato shared_ptr)

## Estendere il codice

### Aggiungere la scansione UDP

Vuoi scansionare le porte UDP? Ecco il procedimento:

1. **Crea il tipo di socket UDP** in `PortScanner.hpp`
```cpp
   enum class Protocol { TCP, UDP };
   Protocol protocol = Protocol::TCP;
```

2. **Modifica la creazione del socket** in `scan()`
```cpp
   if (protocol == Protocol::UDP) {
       auto socket = std::make_shared<udp::socket>(io);
       // UDP scanning uses sendto instead of connect
   } else {
       auto socket = std::make_shared<tcp::socket>(io);
   }
```

3. **Implementa la logica di sondaggio UDP**
```cpp
   // UDP has no connection handshake
   // Send a payload specific to the service (DNS query for port 53)
   // Wait for response or ICMP unreachable
   socket->async_send_to(boost::asio::buffer(probe), endpoint, ...);
```

La scansione UDP è più difficile perché UDP non ha stati di connessione. Devi inviare sonde specifiche per il protocollo e interpretare le risposte per determinare se una porta è aperta.

## Dipendenze

### Perché ogni dipendenza

- **Boost.Asio** (1.70+): framework di I/O asincrono che astrae le API dei socket specifiche del sistema operativo (epoll/kqueue/IOCP). Lo usiamo per `async_connect`, i timer e l'event loop. Alternativa: socket POSIX grezzi, ma richiede di implementare un proprio event loop.

- **Boost.Program_Options** (1.70+): parser degli argomenti CLI con type safety e generazione automatica dell'aiuto. Lo usiamo in `main.cpp` per i flag `-i`, `-p`, `-t`. Alternativa: parsing manuale di `argv`, ma è soggetto a errori e richiede molto boilerplate.

### Sicurezza delle dipendenze

Controlla le vulnerabilità:
```bash
# Boost doesn't have automated CVE scanning, but check your version
dpkg -l | grep libboost  # On Debian/Ubuntu
brew info boost          # On macOS

# Visit https://www.cvedetails.com/vendor/14185/Boost.html
```

Se vedi una CVE di Boost che riguarda Asio (raro), aggiorna:
```bash
sudo apt update && sudo apt upgrade libboost-all-dev
```

La maggior parte delle vulnerabilità di Boost si trova in moduli specifici (Boost.Python, Boost.Beast). Asio è ben verificato e stabile.

## Build e deploy

### Build
```bash
mkdir build && cd build
cmake ..
make
```

Questo produce l'eseguibile `simplePortScanner` nella directory di build. Il processo di build:

1. CMake legge `CMakeLists.txt` e trova le librerie Boost
2. Genera Makefile specifici per la piattaforma (o progetti Ninja/Xcode)
3. Il compilatore viene invocato con il flag `-std=c++20`
4. Effettua il link con Boost.Program_Options e pthread (implicito)

### Sviluppo locale
```bash
# Rebuild after changes
cd build
make

# Run with verbose output to see all scans
./simplePortScanner -i 127.0.0.1 -p 1-100 -v

# Test specific ports
./simplePortScanner -i localhost -p 22,80,443
```

### Deployment in produzione

Per un lavoro di scansione reale:
```bash
# Compile with optimizations
cmake -DCMAKE_BUILD_TYPE=Release ..
make

# Install to system
sudo cp simplePortScanner /usr/local/bin/
```

Differenze principali rispetto allo sviluppo:
- Le build di release sono 3-5× più veloci (ottimizzazioni del compilatore)
- Simboli di debug rimossi (binario più piccolo)
- Asserzioni disabilitate (nessun controllo a runtime)

## Prossimi passi

Hai visto come funzionano l'I/O asincrono, la scansione concorrente e il rilevamento dello stato. Ora:

1. **Prova le sfide** - [04-CHALLENGES.md](./04-CHALLENGES.it.md) contiene idee di estensione come la SYN scan, il rilevamento della versione dei servizi e i formati di output.

2. **Modifica la concorrenza** - Imposta `MAX_THREADS` a 1 e osserva la scansione seriale (lenta). Impostalo a 1000 e guarda l'uso delle risorse impennarsi. Trova il punto ottimale per la tua rete.

3. **Confronta con Nmap** - Esegui `nmap -sT scanme.nmap.org` (TCP connect scan, la stessa del nostro) e confronta i risultati. Nmap ha decenni di gestione dei casi limite che noi non abbiamo.
