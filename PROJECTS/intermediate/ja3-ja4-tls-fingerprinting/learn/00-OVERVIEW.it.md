<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# JA3/JA4 TLS Fingerprinting: panoramica

## Che cos'è

Un sensore passivo di TLS fingerprinting scritto in Rust. Gli dai una packet capture o un'interfaccia di rete live, e lui calcola i fingerprint JA3, JA4, JA4S, JA4H, JA4X e JA4T di ogni handshake che vede, cerca ciascuno di essi in un database locale di threat intelligence e genera alert quando qualcosa non sembra a posto. Non invia mai un pacchetto, non decifra mai i dati applicativi e non ha mai bisogno di una chiave privata. Tutto ciò che sa, lo impara osservando le parti di una connessione che sono visibili prima che inizi la cifratura.

Lo scopo del progetto è capire, costruendolo, come chi difende una rete identifica *quale software* sta comunicando sulla propria rete quando non può vedere *cosa sta dicendo*. Ottieni un sensore funzionante che puoi eseguire su capture reali, un threat database popolato a partire da feed pubblici reali, e una codebase abbastanza piccola da poter essere letta in un weekend.

## Perché è importante

La cifratura ha nascosto il contenuto del traffico di rete, e il settore in gran parte considera la questione risolta. Ciò che la cifratura non ha nascosto è la *negoziazione* della cifratura stessa. Prima che un client e un server TLS si accordino su una chiave, si scambiano un handshake in chiaro, e quell'handshake è modellato dal software del client in un modo che il client non può mascherare facilmente. Un difensore che non può leggere il traffico può comunque riconoscere chi sta parlando.

Questo conta perché gli attaccanti riutilizzano i propri strumenti. Un framework di command-and-control, un credential stealer, un impianto di una botnet: ciascuno è costruito su una specifica libreria TLS, configurata in un modo specifico, e quella configurazione produce lo stesso fingerprint su ogni vittima. Blocca il fingerprint e blocchi la famiglia, indipendentemente da quale IP o dominio la nasconda questa settimana.

Il costo di *non* avere questa visibilità non è ipotetico.

- **SolarWinds / SUNBURST, 2020.** Un attore statale ha distribuito una backdoor tramite un aggiornamento software fidato, e l'impianto mandava beacon verso l'esterno su HTTPS che, a livello di rete, sembrava traffico ordinario. I difensori che osservavano solo IP e domini non avevano nulla in mano; l'infrastruttura malevola era nuova e pulita. Il traffico che *trasportava* il beacon, però, aveva una forma TLS coerente. Il fingerprinting è uno dei pochi segnali passivi che sopravvive a un attaccante che controlla i propri certificati e ruota la propria infrastruttura.
- **Cobalt Strike, ovunque.** Il framework di post-exploitation più comune in assoluto nelle intrusioni reali viene distribuito con profili TLS di default i cui hash JA3 sono di dominio pubblico. Il suo fingerprint di default, `72a589da586844d7f0818ce684948eea`, è così noto che i team maturi generano alert direttamente su di esso. Lo stesso hash compare anche per alcuni campioni di Emotet, perché condividono una libreria TLS sottostante, il che è di per sé una lezione utile: un fingerprint identifica una *toolchain*, non un *threat actor*.
- **La corsa agli armamenti dell'impersonazione dei browser.** Tool come `curl-impersonate` e `utls` esistono proprio per rendere il ClientHello di uno script identico a quello di un vero Chrome, in modo che un filtro basato sui fingerprint lo lasci passare. Ecco perché un singolo fingerprint non basta mai, e perché questo tool ne calcola diversi e li confronta tra loro e con lo User-Agent.

**Scenari reali in cui si applica:**
- **Network security monitoring.** Un sensore su una porta span o su un tap calcola il fingerprint di ogni connessione TLS in uscita e segnala quelle che corrispondono ai feed di malware o che non sono coerenti con il proprio User-Agent.
- **Rilevamento di bot e frodi.** Un servizio web calcola il fingerprint del TLS in ingresso per distinguere un browser reale da un client automatizzato che dichiara di esserlo, senza un CAPTCHA.
- **Threat hunting e forensics.** Un analista passa al tool un `.pcap` catturato durante un incidente e ottiene un inventario ordinato di ogni client che ha comunicato sulla rete, con i relativi verdetti.

## Cosa imparerai

Questo progetto insegna come funziona davvero l'identificazione passiva su una rete cifrata. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**
- **Cosa rivela un handshake TLS.** Perché il ClientHello è un fingerprint: le cipher suite, le estensioni e le curve che un client offre sono una firma stabile del software che lo ha costruito.
- **Perché JA4 ha sostituito JA3.** Come una singola scelta di design, ordinare le liste prima di calcolarne l'hash, ha permesso a JA4 di sopravvivere al rimescolamento dell'ordine delle estensioni che ha reso JA3 inutile per i browser moderni.
- **GREASE e rumore intenzionale.** Perché i client iniettano valori casuali nei propri handshake (RFC 8701), e perché un fingerprint deve eliminarli, altrimenti cambia a ogni connessione.
- **Evasione e controlli incrociati.** Come un tool impersona il TLS di un browser, e come l'abbinamento del fingerprint TLS con il fingerprint dello stack TCP e lo User-Agent smaschera la menzogna.
- **QUIC passivo.** Perché un pacchetto initial di QUIC, che è "cifrato", può comunque essere letto da chiunque, dato che la sua chiave è derivata da un valore inviato in chiaro.

**Competenze tecniche:**
- **Parsing sicuro di input binario ostile.** Percorrere i record TLS e il DER X.509 in un linguaggio che vieta `unsafe`, così che una lunghezza malformata sia un errore tipizzato e mai un buffer overrun.
- **Riassemblaggio di stream TCP.** Ricostruire uno stream di byte a partire da segmenti fuori ordine, ritrasmessi e sovrapposti, che è il modo in cui arriva ogni capture reale.
- **Limitare il lavoro non fidato.** Imporre limiti rigidi alla flow table e ai buffer per stream, così che un attaccante non possa far esaurire la memoria al sensore.
- **Progettare un workspace.** Separare un engine che vieta l'I/O da uno store che possiede un database e da un binario che li collega, così che l'engine possa essere sottoposto a fuzzing e benchmark in isolamento.

**Tool e tecniche:**
- **`libpcap`** per la cattura live, e il modello di capability di Linux (`cap_net_raw`) che consente a un binario non privilegiato di aprire un raw socket.
- **`tshark` / Wireshark** come oracolo: anche Wireshark calcola JA3 e JA4, quindi puoi verificare che questo tool sia d'accordo con lui su una capture reale.
- **SQLite** come store di intelligence embedded e a configurazione zero, in modalità WAL così che una dashboard possa leggere mentre un sensore scrive.

## Prerequisiti

Non ti serve un'esperienza pregressa di fingerprinting. Ti serve però una certa dimestichezza con quanto segue.

**Conoscenze richieste:**
- **L'handshake TLS ad alto livello.** Che un client invia un ClientHello, un server risponde con un ServerHello e un certificato, e viene concordata una chiave. Non ti serve la crittografia; ti serve sapere quali messaggi esistono e quali viaggiano in chiaro.
- **Basi di TCP/IP.** Cos'è un segmento, cos'è un SYN, perché i pacchetti arrivano fuori ordine. Il layer di riassemblaggio è il cuore del percorso di cattura.
- **Rust di base, o la disponibilità a leggerlo.** Il codice usa enum, pattern matching, iteratori e `Result`. Se conosci Go, C++ o Python con i type hint, puoi seguirlo.

**Tool di cui avrai bisogno:**
- **Una toolchain Rust**, edizione 2024 (rustc 1.85 o successivo). Lo script `install.sh` ne verifica la presenza.
- **Gli header di sviluppo di `libpcap`** per la cattura live (`libpcap-dev` su Debian o Ubuntu). L'uso solo su file non ne ha bisogno.
- **Una capture da dargli in pasto.** Il repository ne include diverse in `testdata/pcap/`, oppure puoi catturarne una tua con `tcpdump -w out.pcap`.

**Utili ma non obbligatori:**
- **Wireshark**, per confrontare i fingerprint con una seconda implementazione.
- Una lettura della [specifica JA4+ di FoxIO](https://github.com/FoxIO-LLC/ja4/blob/main/technical_details/JA4.md) e del [post originale di Salesforce su JA3](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/). Puoi anche leggere [`01-CONCEPTS.md`](./01-CONCEPTS.it.md) e consultare le specifiche quando qualcosa non è chiaro.

## Avvio rapido

```bash
cd PROJECTS/intermediate/ja3-ja4-tls-fingerprinting

# Build the binary and seed the threat feeds
cargo build --release
./target/release/tlsfp intel seed

# Fingerprint a vendored capture, one line per handshake
./target/release/tlsfp pcap testdata/pcap/tls-handshake.pcapng

# Read the whole capture and print one ranked forensic summary
./target/release/tlsfp pcap testdata/pcap/tls-handshake.pcapng --report
```

Output atteso: il semplice comando `pcap` stampa una riga per handshake, ciascuna con un JA4 e un JA3 e, per un ClientHello, lo SNI e l'ALPN. Vedrai sia fingerprint con trasporto `t` (TLS su TCP) sia fingerprint con trasporto `q` (TLS dentro QUIC) nella stessa capture, perché il file incluso contiene entrambi. Il comando `--report` invece stampa un unico riepilogo: gli endpoint più attivi, i fingerprint più comuni, eventuali corrispondenze di intelligence ed eventuali alert.

Per osservare un'interfaccia live, concedi al binario le due capability di cui ha bisogno e puntalo su un'interfaccia:

```bash
sudo setcap cap_net_raw,cap_net_admin=eip "$(command -v ./target/release/tlsfp)"
./target/release/tlsfp live eth0 --intel --detect
```

Fermalo con ctrl-c: il primo svuota la cattura e stampa contatori finali affidabili, un secondo esce immediatamente.

## Struttura del progetto

```
ja3-ja4-tls-fingerprinting/
├── crates/
│   ├── tlsfp-core/    # the engine: parses TLS, reassembles TCP, computes the fingerprints
│   ├── tlsfp-intel/   # the store: a bundled SQLite database, matching, and the detection rules
│   └── tlsfp/         # the binary: the CLI and the web dashboard
├── testdata/pcap/     # vendored captures used as integration fixtures
├── frontend/          # the dashboard (Vite + React 19)
└── install.sh         # the one-shot installer
```

Il file più importante da capire per primo è `crates/tlsfp-core/src/ja4.rs`. Tutto ciò che sta nell'engine esiste per fornirgli un ClientHello già sottoposto a parsing; tutto ciò che sta nel resto del tool esiste per agire su ciò che restituisce.

## Prossimi passi

1. **Comprendi le idee.** Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per il ClientHello, JA3 contro JA4, GREASE, l'evasione e QUIC passivo, ciascuno ancorato a un'intrusione reale.
2. **Osserva il design.** Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per la suddivisione in tre crate, la pipeline di cattura, lo store di intelligence e il threat model.
3. **Percorri il codice.** Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per seguire un frame grezzo fino a un alert con punteggio.
4. **Impara gli algoritmi.** Leggi [ALGORITHMS.md](./ALGORITHMS.it.md) per vedere come ogni fingerprint viene costruito byte per byte e come viene decifrato un initial di QUIC.
5. **Verifica il contratto.** Leggi [CONFORMANCE.md](./CONFORMANCE.it.md) per il vettore pubblicato a cui è vincolato ogni fingerprint e per ogni confine di scope deliberato.
6. **Estendilo.** Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per progetti che vanno da "aggiungi un fingerprint" a "costruisci un modello di clustering".

## Problemi comuni

**`tlsfp live` fallisce con un errore di permessi**
```
Error: opening interface eth0: you don't have permission to capture
```
Soluzione: la cattura live apre un raw socket, cosa che un utente non privilegiato non può fare. Concedi una volta per tutte le capability al binario con `sudo setcap cap_net_raw,cap_net_admin=eip "$(command -v tlsfp)"`. Le file capability risiedono sul binario, quindi ripeti la concessione dopo ogni rebuild.

**`intel lookup` dice che non esiste alcun database**
```
no intelligence database at ...; run 'tlsfp intel seed' first
```
Soluzione: i comandi lookup, stats e alerts leggono un database esistente ma non ne creano mai uno. Esegui `tlsfp intel seed` per costruirlo a partire dai tre feed inclusi, operazione che non richiede la rete.

**Una capture mostra meno handshake di quanti me ne aspettassi**
Soluzione: un handshake suddiviso in molti segmenti fuori ordine, o uno il cui ClientHello arriva dopo il limite di byte per stream, potrebbe non produrre un fingerprint. Esegui con `-v` per vedere i contatori, inclusi `tls_miss_rate` e `segments_dropped`. Su un'interfaccia live, i drop del kernel sotto carico producono lo stesso effetto; il tool avvisa quando il kernel ne segnala.

**Il fingerprint non coincide con quello di Wireshark**
Soluzione: verifica di confrontare lo stesso handshake e la stessa versione del fingerprint. JA4 ha una forma raw e una forma hashed; questo tool stampa di default la forma hashed. La forma raw è disponibile nell'output JSON (`--json`) proprio per questo tipo di debug.

## Progetti correlati

Se lo hai trovato interessante, dai un'occhiata a:
- **hsm-emulator**: un progetto avanzato che costruisce l'*altro* lato del TLS, la custodia delle chiavi fornita da un HSM, con la stessa disciplina di vincolare ogni output a un vettore pubblicato.
- **bug-bounty-platform**: un'applicazione completa che gestisce il tipo di traffico che questo sensore osserverebbe, e mostra dove un segnale di fingerprinting confluirebbe in una pipeline di rilevamento più ampia.
