# Concetti di sicurezza fondamentali

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci questo progetto. Non sono solo definizioni, ma conoscenze pratiche usate quotidianamente nel penetration testing e nella sicurezza di rete.

## Port scanning TCP

### Che cos'è

Il port scanning è il processo con cui si sonda un host target per determinare quali porte TCP o UDP accettano connessioni. Su ogni numero di porta (0-65535) può esserci un servizio in ascolto. La scansione rivela quale software è in esecuzione su un sistema senza richiedere autenticazione.

Pensalo come controllare ogni porta e ogni finestra di un edificio per vedere quali sono aperte. Le porte 1-1023 sono porte "well-known" assegnate a servizi standard (HTTP sulla 80, SSH sulla 22), mentre le porte più alte possono ospitare qualsiasi cosa.

### Perché è importante

Il port scanning è **ricognizione**, la prima fase della cyber kill chain. Ogni penetration test, ogni vulnerability assessment e molti attacchi reali iniziano mappando ciò che è accessibile. La violazione di Equifax del 2017 è iniziata con una ricognizione che ha trovato un server Apache Struts non aggiornato sulla porta 8080. Trovare quella porta aperta è stato il primo passo.

Nell'attacco della botnet Mirai del 2016 che ha messo fuori uso Dyn DNS, il malware ha scansionato l'intera internet alla ricerca di dispositivi IoT con telnet (porta 23) esposto. Ha trovato centinaia di migliaia di telecamere e router vulnerabili perché la porta 23 non dovrebbe mai essere aperta su dispositivi consumer esposti a internet.

### Come funziona

Il port scanning TCP sfrutta il meccanismo del three-way handshake:
```
Scanner          Target
   |               |
   |----SYN------->|  (attempt connection)
   |               |
   |<--SYN-ACK-----|  (port is OPEN - service listening)
   |               |
   |----RST------->|  (scanner aborts, doesn't complete handshake)
```

oppure
```
Scanner          Target
   |               |
   |----SYN------->|
   |               |
   |<---RST--------|  (port is CLOSED - no service, but host responded)
```

oppure
```
Scanner          Target
   |               |
   |----SYN------->|
   |               |
   |   (silence)   |  (port is FILTERED - firewall dropped packet)
```

Il nostro scanner invia un pacchetto SYN (richiesta di connessione) e interpreta la risposta. La libreria Boost.Asio gestisce i dettagli dell'handshake, ma capire cosa succede sul cavo è fondamentale.

### Attacchi comuni

1. **Ricognizione completa prima dell'exploitation** - Gli attaccanti scansionano intere reti per costruire database di target. Il ransomware WannaCry cercava SMB (porta 445) prima di lanciare l'exploit EternalBlue. Il port scanning identificava le macchine vulnerabili.

2. **Attacchi specifici per servizio** - Trovare MySQL sulla porta 3306 esposto a internet significa che l'attaccante può provare credential stuffing, SQL injection o CVE note specifiche per i database server. Ogni porta aperta restringe la superficie d'attacco a specifiche categorie di exploit.

3. **Fingerprinting di firewall e IDS** - La differenza tra porte chiuse e filtrate rivela le regole del firewall. Se le porte 1-1000 restituiscono RST (chiuse) ma le 1001-2000 vanno in timeout (filtrate), sai che c'è un filtering selettivo. Questa informazione aiuta gli attaccanti a identificare i punti ciechi.

### Strategie di difesa

**Minimizza le porte esposte:** apri solo ciò di cui hai davvero bisogno. Le installazioni di default spesso abilitano servizi non necessari. Esegui `netstat -tuln` su Linux o `netsh interface ipv4 show tcpconnections` su Windows per vedere cosa è in ascolto. Ogni porta aperta è potenziale superficie d'attacco.

**Implementa regole di firewall adeguate:** nega per default il traffico in ingresso, consenti esplicitamente solo i servizi necessari. Il nostro scanner rileva le porte filtrate tramite i timeout (`src/PortScanner.cpp:139-147`), quindi un firewalling adeguato rende la ricognizione più difficile.

**Monitora l'attività di scansione:** più tentativi di connessione verso porte sequenziali da un unico IP sorgente sono una scansione. I sistemi IDS come Snort hanno regole specifiche per il rilevamento dei port scan. Lo schema di pacchetti SYN senza handshake completati salta all'occhio nei log.

**Usa il port knocking o la single packet authorization:** per SSH o altri servizi di amministrazione, richiedi una sequenza segreta di tentativi di connessione prima che la porta risulti aperta. Questo nasconde i servizi critici alle scansioni occasionali.

## Stati delle porte e loro significato

### Che cos'è

Dal punto di vista di uno scanner, ogni porta di un sistema in rete si trova in uno di tre stati: aperta, chiusa o filtrata. Questi stati raccontano storie fondamentalmente diverse sul target.

### Perché è importante

Lo stato rivela sia la configurazione del servizio sia la postura di sicurezza:

- **Aperta**: qualcosa è in ascolto. Questa è la tua superficie d'attacco. Nell'attacco alla supply chain di SolarWinds del 2020, gli attaccanti hanno usato credenziali rubate per accedere a un repository di codice interno. Lo hanno trovato cercando porte di server git aperte (di solito la 443 o porte personalizzate per GitLab/GitHub Enterprise).

- **Chiusa**: l'host è attivo e raggiungibile, ma nulla è in ascolto su quella porta. Le porte chiuse confermano che il target esiste e risponde, aiutando nella mappatura della rete anche quando i servizi non sono esposti.

- **Filtrata**: un firewall o un packet filter si trova tra te e il target. Questo dice agli attaccanti che c'è un'infrastruttura di sicurezza, ma potrebbe anche rivelare debolezze di configurazione se alcune porte sono filtrate e altre no.

### Come funziona

Il nostro scanner implementa il rilevamento dello stato in `src/PortScanner.cpp:123-165`:

**Rilevamento delle porte aperte** (`PortScanner.cpp:138-151`):
```cpp
socket->async_connect(endpoint, [](boost::system::error_code ec) {
    if (!ec) {
        // Connection succeeded = OPEN
        // Try to grab banner
    }
});
```
Se `async_connect` si completa senza errori, l'handshake TCP è riuscito. Qualcosa ha accettato la nostra connessione.

**Rilevamento delle porte chiuse** (`PortScanner.cpp:153-158`):
```cpp
else {
    // Connection failed = CLOSED
    printf("%i\t%sCLOSED%s\t%s\t%s\n", port, RED, RESET, ...);
}
```
Se `async_connect` restituisce rapidamente un errore (di solito "connection refused"), l'host ci ha inviato un pacchetto RST. La porta è chiusa.

**Rilevamento delle porte filtrate** (`PortScanner.cpp:128-137`):
```cpp
timer->async_wait([](boost::system::error_code ec) {
    if (!ec && !*complete) {
        // Timer expired before connection = FILTERED
        printf("%i\t%s\t%s\t%s\n", port, "FILTERED", ...);
    }
});
```
Se entro il timeout (default 2 secondi) non si verificano né un successo né un errore, assumiamo che un firewall abbia scartato i nostri pacchetti. Il tentativo di connessione resta semplicemente in sospeso finché non ci arrendiamo.

### Insidie comuni

**Errore 1: confondere chiusa con filtrata**
```cpp
// Wrong - timing out doesn't mean closed
if (connection_timeout) {
    state = "CLOSED";  // No! This is FILTERED
}

// Right - closed is an active rejection
if (error_code == "connection_refused") {
    state = "CLOSED";
}
```
Chiusa significa che l'host ti ha rifiutato attivamente. Filtrata significa che i tuoi pacchetti sono spariti dentro un firewall. Questa distinzione conta per capire la topologia di rete.

**Errore 2: non gestire i falsi positivi**
Alcuni host hanno firewall che inviano pacchetti RST per sembrare chiusi anche quando sono filtrati. Una scansione avanzata richiede più tecniche di sondaggio (scansioni SYN, ACK, FIN) per distinguere questi casi limite. Il nostro scanner semplice prende le risposte per quello che sembrano.

## Banner grabbing

### Che cos'è

Il banner grabbing consiste nel connettersi a un servizio e leggere qualunque messaggio iniziale esso invii. Molti protocolli si annunciano immediatamente alla connessione. I server SSH dicono "SSH-2.0-OpenSSH_8.2p1", i web server inviano header HTTP con "Server: Apache/2.4.41", e così via.

### Perché è importante

I banner dei servizi fanno trapelare informazioni sulla versione che gli attaccanti usano per scegliere gli exploit. Se il tuo banner SSH dice "OpenSSH_7.4", posso consultare i database CVE alla ricerca di vulnerabilità note in quella esatta versione. L'attacco Heartbleed del 2014 (CVE-2014-0160) colpiva OpenSSL dalla 1.0.1 alla 1.0.1f. Il banner grabbing diceva agli attaccanti quali server erano vulnerabili.

Gli attacchi a Microsoft Exchange Server del 2021 (ProxyLogon) prendevano di mira specifiche versioni di Exchange. Gli attaccanti scansionavano alla ricerca di server Exchange, catturavano i banner per identificarne la versione, poi lanciavano exploit mirati. Le informazioni sulla versione hanno trasformato una scansione generica in un attacco di precisione.

### Come funziona

Dopo esserci connessi con successo a una porta aperta, proviamo a leggere i dati iniziali:
```cpp
// src/PortScanner.cpp:143-149
socket->async_read_some(boost::asio::buffer(*buf),
    [](boost::system::error_code ec, std::size_t n) {
        if (!ec && n > 0) {
            banner->assign(buf->data(), n);
        }
        printf("%i\tOPEN\t%s\t%s\n", port, service.c_str(), banner->c_str());
    });
```

Allochiamo un buffer da 128 byte e tentiamo una lettura non bloccante. Se il servizio invia qualcosa immediatamente, lo catturiamo. In caso contrario (molti servizi aspettano prima l'input del client), procediamo senza banner. Questo è un grab passivo - non inviamo sonde specifiche per il protocollo.

Alcuni servizi richiedono che sia tu a parlare per primo il loro protocollo. I server HTTP hanno bisogno che tu invii "GET / HTTP/1.1" prima di rispondere. Il nostro scanner si limita ad ascoltare, il che funziona per i servizi loquaci come SSH, SMTP e FTP che si annunciano da soli.

### Attacchi comuni

1. **Targeting di exploit specifici per versione** - Dopo aver trovato MySQL sulla porta 3306 e catturato "5.5.62-0ubuntu0.14.04.1", gli attaccanti cercano su Exploit-DB vulnerabilità di MySQL 5.5.x. Il banner grabbing trasforma la generica scoperta di porte in una mappatura precisa delle vulnerabilità.

2. **Identificazione di software obsoleto** - Qualsiasi server che annuncia una versione del 2015 è probabilmente vulnerabile a molteplici CVE. Gli attaccanti danno la priorità a questi target. Nel 2017, gli Shadow Brokers hanno fatto trapelare exploit della NSA legati specificamente al rilevamento della versione di Windows ricavata dai banner SMB.

3. **Fingerprinting per il movimento laterale** - All'interno di una rete, il banner grabbing rivela l'infrastruttura. Trovare "VMware ESXi 6.0" sulla porta 443 dice a un attaccante che si tratta di un host di virtualizzazione che vale la pena compromettere (un host ESXi controlla molte VM).

### Strategie di difesa

**Sopprimi le informazioni sulla versione nei banner:** la maggior parte dei servizi ti permette di personalizzare ciò che annunciano. La configurazione di SSH ha `DebianBanner no`, Apache ha `ServerTokens Prod` e Nginx ha `server_tokens off`. Non pubblicizzare la tua versione esatta.

**Usa messaggi di errore generici:** non lasciare che gli errori della tua applicazione rivelino le versioni del framework. "Error 500" è meglio di "Ruby on Rails 5.2.3 - NoMethodError in UsersController#create".

**Implementa la randomizzazione dei banner o degli honeypot:** le configurazioni avanzate randomizzano le stringhe dei banner o pubblicizzano false versioni vulnerabili per far perdere tempo agli attaccanti. Se il tuo banner SSH dichiara di essere del 2012 ma hai tutte le patch applicate, gli scanner ti marcheranno come un bersaglio facile mentre tu rilevi le loro sonde.

## Come si collegano questi concetti

Port scanning, rilevamento dello stato e banner grabbing formano una pipeline di ricognizione:
```
Port Scan
    ↓
Identifies OPEN ports
    ↓
Banner Grab
    ↓
Reveals service versions
    ↓
Vulnerability Mapping
    ↓
Exploitation
```

Ogni passo si basa sul precedente. Non puoi catturare banner senza aver prima trovato porte aperte. Non puoi identificare vulnerabilità senza conoscere le versioni. Capire questa catena aiuta sia gli attaccanti (che la eseguono) sia i difensori (che devono spezzarla).

## Standard e framework del settore

### OWASP Top 10

Questo progetto è collegato a:

- **A05:2021 – Security Misconfiguration** - Le porte aperte non necessarie sono configurazioni errate. Ogni servizio esposto che non ha bisogno di essere pubblico aumenta la superficie d'attacco. Il port scanning identifica questi errori.

- **A01:2021 – Broken Access Control** - I servizi in ascolto su interfacce pubbliche quando dovrebbero essere solo su localhost rappresentano un controllo degli accessi difettoso. La scansione rivela questi difetti architetturali.

### MITRE ATT&CK

Tecniche rilevanti:

- **T1046 - Network Service Discovery** - Il port scanning è esplicitamente elencato come tecnica per scoprire i servizi. Il nostro tool implementa direttamente questo metodo di ricognizione.

- **T1595.001 - Active Scanning: Scanning IP Blocks** - Tool automatizzati scansionano intervalli di IP per trovare target. Questo progetto dimostra come funzionano tali tool a livello di implementazione.

### CWE

Common weakness enumeration trattate:

- **CWE-200 - Exposure of Sensitive Information to an Unauthorized Actor** - Il banner grabbing sfrutta i servizi che rivelano informazioni sulla versione. Correggere la CWE-200 significa ripulire i banner.

- **CWE-1188 - Insecure Default Initialization of Resource** - Le installazioni di default spesso aprono porte non necessarie. I port scan trovano queste esposizioni involontarie.

## Esempi reali

### Caso di studio 1: la botnet Mirai (2016)

La botnet IoT Mirai ha asservito centinaia di migliaia di dispositivi scansionando l'intera internet alla ricerca di telnet (porta 23) e SSH (porta 22) con credenziali di default. La sequenza dell'attacco era:

1. Scansionare indirizzi IP casuali alla ricerca delle porte 23/22 aperte (network service discovery)
2. Catturare i banner telnet per identificare i tipi di dispositivo (alcuni dispositivi IoT annunciano il numero di modello)
3. Provare le credenziali di default (admin/admin, root/root) in base al fingerprint del dispositivo
4. Installare il malware e unirsi alla botnet

L'attacco DDoS a Dyn DNS che ha messo fuori uso Twitter, Netflix e Reddit nell'ottobre 2016 proveniva da dispositivi infettati da Mirai. Tutto è iniziato con il port scanning. L'infezione si è diffusa perché i dispositivi IoT consumer venivano distribuiti con telnet abilitato e password di default non modificabili.

**Come si sarebbe potuto prevenire:** i produttori avrebbero dovuto disabilitare telnet per default, richiedere il cambio della password al primo avvio e implementare un rilevamento dei port scan che mettesse automaticamente in blacklist gli scanner. Gli ISP avrebbero potuto filtrare il traffico in uscita sulla porta 23 dalle reti consumer.

### Caso di studio 2: l'attacco alla supply chain di SolarWinds (2020)

Sebbene la compromissione iniziale sia avvenuta tramite un aggiornamento software trojanizzato, gli attaccanti hanno usato il port scanning durante il movimento laterale all'interno delle reti delle vittime. Dopo aver ottenuto l'accesso iniziale, hanno:

1. Scansionato le reti interne alla ricerca delle comuni porte di gestione (3389 per RDP, 5985 per WinRM)
2. Identificato i server Active Directory (porte 88, 389, 636)
3. Trovato i sistemi di backup e gli strumenti di sicurezza per disabilitarli
4. Mappato l'architettura di rete correlando le porte aperte tra le subnet

Gli attaccanti hanno trascorso mesi all'interno delle reti, scansionando e documentando metodicamente l'infrastruttura prima di esfiltrare i dati. Il port scanning era il loro strumento cartografico.

**Come si sarebbe potuto prevenire:** segmentazione della rete con regole di firewall rigide tra le zone, monitoraggio del port scanning interno (analisi del traffico est-ovest) e adozione di tecnologie di deception (servizi falsi su porte honeypot che generano un alert quando vengono scansionate).

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. **Perché il nostro scanner usa i timeout per rilevare le porte filtrate invece di aspettare semplicemente i pacchetti RST?** (Suggerimento: cosa succede ai pacchetti che colpiscono un firewall configurato con DROP invece di REJECT?)

2. **Se scansioni un web server sulla porta 80 e catturi il banner "Server: Apache/2.4.41 (Ubuntu)", quali informazioni specifiche hai appreso che aiutano un attaccante?** (Pensa al sistema operativo, alla versione del software e alle potenziali vulnerabilità.)

3. **Spiega perché le porte chiuse forniscono comunque informazioni utili per la ricognizione anche se nessun servizio è in esecuzione.** (Cosa ti dice una risposta RST sul target?)

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. L'implementazione avrà più senso una volta che avrai capito cosa significa ogni stato di porta e perché il banner grabbing conta negli attacchi reali.

## Letture di approfondimento

**Essenziali:**

- **RFC 793 - TCP Specification** - La RFC originale di TCP spiega il three-way handshake e il comportamento di RST. La sezione 3.4 tratta l'instaurazione della connessione. Capirla rende chiara la teoria del port scanning.

- **Nmap Network Scanning di Gordon Lyon** - Il libro di riferimento sulle tecniche di port scanning. I capitoli 3-5 trattano i metodi di scansione TCP, incluse le scansioni SYN, connect e ACK. Disponibile gratuitamente su nmap.org/book/.

**Approfondimenti:**

- **IANA Service Name and Transport Protocol Port Number Registry** - L'elenco ufficiale delle porte registrate e dei relativi servizi. Quando il nostro scanner mostra "SSH" per la porta 22, è da qui che proviene quella corrispondenza: iana.org/assignments/service-names-port-numbers/.

- **Documentazione delle regole IDS di Snort** - Le regole 1-1999 trattano il rilevamento delle scansioni. Leggerle mostra quali schemi fanno scattare gli alert e come eludere un IDS di base: snort.org/rules.

**Contesto storico:**

- **Phrack Issue 49 (1996) - The Art of Port Scanning** - L'articolo originale di Fyodor che introduceva le tecniche di stealth scanning. Sebbene datato, spiega la teoria fondamentale che i tool moderni usano ancora.
