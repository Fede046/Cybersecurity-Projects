<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# JA3/JA4 TLS Fingerprinting: architettura

Questo documento descrive come è assemblato il tool: i tre crate e perché sono separati, la pipeline di cattura attraverso cui scorre un frame, lo store di intelligence che giudica un fingerprint, e il threat model che ha dato forma a ogni limite. Leggi prima [01-CONCEPTS.md](./01-CONCEPTS.it.md); questo documento presuppone che tu sappia cosa sono i fingerprint.

## La suddivisione in tre crate

Il progetto è un unico workspace Cargo con tre crate in una linea di dipendenze rigorosa. Niente punta all'indietro.

```
   ┌──────────────────────────────────────────────────────────┐
   │  tlsfp-core     the engine                                │
   │  parses TLS, reassembles TCP, decrypts QUIC, hashes.      │
   │  Depends on NOTHING that touches a network, a database,   │
   │  or an async runtime. Forbids `unsafe`. Fuzzable.         │
   └───────────────────────────┬──────────────────────────────┘
                               │  tlsfp-core::FingerprintEvent
   ┌───────────────────────────┴──────────────────────────────┐
   │  tlsfp-intel    the judgement                             │
   │  owns a bundled SQLite database. Matches a fingerprint    │
   │  to a verdict, runs the detection rules, records alerts.  │
   │  Synchronous on purpose. Depends only on tlsfp-core.      │
   └───────────────────────────┬──────────────────────────────┘
                               │  MatchReport + Alert
   ┌───────────────────────────┴──────────────────────────────┐
   │  tlsfp          the binary                                │
   │  the clap CLI and the axum web dashboard. Wires a packet  │
   │  source to the engine to the store to a writer.           │
   └──────────────────────────────────────────────────────────┘
```

Il motivo della suddivisione è la testabilità e il raggio d'impatto. `tlsfp-core` non ha I/O, quindi l'intero engine di fingerprinting gira in modo esatto al byte negli unit test sulle capture incluse nel repository, senza rete, senza orologio e senza database da simulare con mock. Può essere sottoposto a fuzzing in isolamento, cosa importante perché esegue il parsing di input ostile. `tlsfp-intel` aggiunge esattamente una responsabilità, persistenza e giudizio, e dipende solo dal tipo di output dell'engine. Il binario è l'unico crate che sa qualcosa di interfacce, runtime e terminale. Un bug nella dashboard non può raggiungere il parser; una modifica al parser non può rompere lo schema del database.

## La pipeline di cattura

All'interno di `tlsfp-core`, un frame grezzo diventa un fingerprint scorrendo attraverso una sequenza fissa di stadi. Ogni stadio è un modulo separato, così da poter essere compreso e testato da solo.

```
  PacketSource          decode             flow reassembly        protocol            fingerprint
  ────────────          ──────             ───────────────        ────────            ───────────
  pcap file       ┌─> strip Ethernet  ┌─> StreamReassembler  ┌─> watch for a    ┌─> ja3()  ja4()
  pcapng file ────┤   strip IP         │   per (src,dst,      │   TLS record or  │   ja4h() ja4x()
  live (libpcap)  │   keep TCP segment │    sport,dport):     │   an HTTP head   │   ja4t()
  QUIC initial    └─> or UDP datagram  │    order, dedup,     │   in the         └─> StreamEvent
                                       │    resolve overlap,  │   contiguous          + addrs + ts
                      decode.rs        │    bound buffers     │   bytes              = FingerprintEvent
                                       │                      │
                      source.rs        └── flow.rs            └── tls.rs            event.rs
```

**`source.rs`** astrae la provenienza dei frame. `PcapFileSource` legge un file; il `LiveSource` del binario legge un'interfaccia. Entrambi producono un `RawFrame` (un timestamp, un link type e una slice di byte). L'engine non sa né si preoccupa di quale dei due abbia ricevuto.

**`decode.rs`** rimuove da un frame i layer di link e di rete e produce un `DecodedSegment` TCP oppure un `DecodedDatagram` UDP, oppure un motivo di `Skip` se il frame non è interessante. È anche qui che vengono percorse le opzioni del SYN TCP per JA4T, prima del riassemblaggio, perché JA4T legge il SYN stesso, non lo stream che esso apre.

**`flow.rs`** è l'engine di riassemblaggio e il cuore del percorso di cattura. Mantiene una tabella di flow indicizzata dalla quadrupla. Per ogni direzione di ogni flow mantiene un buffer di byte contiguo e un insieme di segmenti fuori ordine parcheggiati. Quando un segmento riempie il buco prima di una sequenza parcheggiata, la sequenza viene unita. Le sovrapposizioni vengono risolte. Questo è lo stadio che fa sì che un ClientHello diviso su tre segmenti riordinati superi correttamente il parsing, ed è lo stadio più accuratamente limitato contro gli abusi (vedi il threat model più sotto).

**`tls.rs`** osserva ogni stream riassemblato alla ricerca di qualcosa di cui calcolare il fingerprint: un flight di handshake TLS completo (ClientHello, ServerHello, Certificate) o l'intestazione di una richiesta HTTP in chiaro. Quando ne riconosce uno, ne esegue il parsing (`parse/`) e calcola il fingerprint.

**`event.rs`** definisce l'output. Uno `StreamEvent` è il fingerprint più il suo tipo:

```rust
pub enum StreamEvent {
    ClientHello { ja3, ja4, sni, alpn, user_agent, ... },
    ServerHello { ja3s, ja4s, ... },
    Certificate { ja4x },
    HttpRequest { ja4h, method, user_agent, ... },
    TcpSyn     { ja4t },
    TcpSynAck  { ja4ts },
}
```

Avvolto con gli indirizzi di origine e destinazione e un timestamp, diventa un `FingerprintEvent`, l'unico tipo che attraversa il confine in uscita dall'engine. Tutto ciò che sta a valle consuma `FingerprintEvent` e nient'altro.

Il percorso QUIC (`quic.rs`) è un ingresso parallelo nello stesso codice di fingerprinting. Un datagramma UDP sul percorso QUIC viene interpretato come un Initial QUIC, le sue chiavi vengono derivate dal suo Connection ID, il suo payload viene decifrato, i suoi frame CRYPTO vengono riassemblati in un ClientHello, e quel ClientHello passa attraverso lo stesso `ja4()` di uno TCP, uscendone con un marcatore di trasporto `q`.

## Lo store di intelligence

`tlsfp-intel` trasforma un `FingerprintEvent` in un giudizio. Possiede un database SQLite embedded, che qui è la scelta giusta: zero configurazione, un singolo file, transazionale, e in grado di supportare una dashboard che legge mentre un sensore scrive (si apre in modalità WAL con un busy timeout esattamente per questo). Lo store è **sincrono**, deliberatamente. Una ricerca è una singola query indicizzata e una cattura è un semplice loop; avvolgere tutto ciò in un runtime async aggiungerebbe complessità senza dare nulla in cambio. Il runtime async vive solo nel web server, dove esistono davvero lettori concorrenti.

```
   FingerprintEvent
         │
         ▼
   ┌───────────────────────────────────────────────┐
   │  matcher.rs   exact lookup by (kind, value)    │
   │               + JA4 fuzzy match on the         │   intel_fingerprint
   │               (prefix, cipher-hash) columns    │   intel_source
   │               -> MatchReport (verdict, score)  │
   └───────────────────┬───────────────────────────┘
                       │
   ┌───────────────────┴───────────────────────────┐
   │  detect.rs    records the observation, then    │   observation
   │               runs six rules over a time        │   alert
   │               window, raising Alerts            │
   │  signal.rs    the User-Agent / OS heuristics    │
   │               the correlation rules read        │
   └────────────────────────────────────────────────┘
```

**Schema (`schema.rs`).** Migrazioni versionate costruiscono cinque tabelle: `intel_source` (una riga per feed, con la relativa licenza), `intel_fingerprint` (i fingerprint, con le colonne `part_a`/`part_b` precalcolate per il fuzzy matching di JA4), `observation` (ogni fingerprint che il sensore ha visto, con il suo IP e l'orario) e `alert` (ogni rilevamento che è scattato). Le tabelle observation e alert sono ciò che rende possibili le regole di correlazione: danno al sensore una memoria.

**Seeding (`seed.rs`).** Tre feed sono compilati nel binario come CSV e caricati senza alcuna chiamata di rete: abuse.ch SSLBL (97 JA3 di malware, CC0), la lista JA3 `osx-nix` di Salesforce (157 JA3 di client benigni, BSD-3), e un piccolo set curato di C2 (17 fingerprint). 271 in totale. Un `intel import` opzionale scarica ja4db.com e valida ciascun record in ingresso.

**Matching (`matcher.rs`).** Una corrispondenza esatta è una singola ricerca indicizzata. Un match *fuzzy* di JA4 gestisce il fatto concreto che il JA4 completo di un client potrebbe non essere nel feed, mentre il suo prefisso di capability e cipher sì: il valore viene diviso nelle sue parti di prefisso e di hash dei cipher, e una corrispondenza solo su quelle viene riportata con una confidenza inferiore. Le corrispondenze vengono valutate in un `MatchReport` con un threat score e una confidenza, i due numeri in base ai quali agisce il resto del tool.

**Rilevamento (`detect.rs` + `signal.rs`).** Per ogni evento, l'engine prima registra un'osservazione, poi esegue sei regole, tutte all'interno di un'unica transazione, così che l'osservazione e gli eventuali alert che genera vengano committati insieme:

| Regola | Scatta quando | Gravità |
|------|-----------|----------|
| `known_bad` | il fingerprint corrisponde a una voce malevola di un feed | high / critical |
| `ua_mismatch` | un JA4 non è coerente con lo User-Agent visto dallo stesso IP (**la regola principale**) | high |
| `os_mismatch` | il sistema operativo di un JA4T non è coerente con quello dichiarato dallo User-Agent | medium |
| `first_seen` | questo fingerprint non è mai stato osservato prima | info |
| `fp_rotation` | un IP ha mostrato un numero insolito di fingerprint distinti nella finestra | medium |
| `monoculture` | un fingerprint è comparso da un numero insolito di IP nella finestra | low |

Le regole di correlazione (`ua_mismatch`, `os_mismatch`) sono quelle che intercettano l'evasione, e sono il motivo per cui lo store mantiene una memoria: per sapere che l'IP che ora invia un JA4 di Chrome falsificato ha anche inviato, dieci secondi fa, una richiesta HTTP con uno User-Agent con cui può essere confrontato. `signal.rs` contiene le euristiche che queste regole leggono, per esempio il fatto che uno stack TCP Windows e uno Linux differiscono nelle opzioni del SYN, così che un JA4T possa essere confrontato con uno `User-Agent` che dichiara Windows.

## Il binario: CLI e dashboard

`tlsfp` è l'unico crate che tocca il mondo esterno.

- **`cli.rs`** è l'albero dei comandi clap: `pcap`, `live`, `serve` e i sottocomandi `intel`. Gestisce l'output in streaming, la serializzazione JSON e il collegamento di una sorgente all'engine, allo store.
- **`live.rs`** esegue libpcap su un thread del sistema operativo dedicato e lo collega a un consumer async, perché la lettura bloccante di libpcap non può essere pilotata in modo sicuro dall'interno del runtime async. Riduce anche i privilegi alle due capability necessarie per la cattura.
- **`report.rs`** è il builder forense di `--report`. Invece di produrre in streaming una riga per evento, accumula in-process un quadro di ogni endpoint, fingerprint e mancato riconoscimento, poi stampa un unico riepilogo ordinato, integrando automaticamente intelligence e rilevamento ogni volta che è presente un database.
- **`serve.rs`** è la dashboard axum. Serve gli asset del frontend compilato e trasmette in streaming eventi e alert al browser tramite Server-Sent Events. Il feed live ha tre sorgenti: un file di capture riprodotto (con cadenza controllata, opzionalmente in loop), un'interfaccia live, oppure, di default, un tail del database, così che un sensore separato `tlsfp live --detect` compaia nel browser.

```
   tlsfp live --detect ──writes──> intel.db <──tails── tlsfp serve ──SSE──> browser
                                   (WAL mode lets the reader and writer share the file)
```

Questa è la configurazione di deploy per cui esistono la modalità WAL e il busy timeout: un sensore headless che scrive alert nel database in un processo, e una dashboard che li legge in un altro, sullo stesso file, senza che nessuno dei due blocchi l'altro.

## Il threat model

Un sensore passivo elabora input scelto interamente da un avversario. Chiunque sulla rete monitorata può inviargli qualsiasi byte. L'architettura tratta ogni input come ostile e limita ogni costo.

**Da cosa si difende il tool:**

- **Pacchetti malformati.** Ogni parser dell'engine restituisce un errore tipizzato su input non valido e non legge mai fuori dai limiti, perché l'engine vieta `unsafe`. Una lunghezza troncata, un'estensione che dichiara più byte di quelli contenuti nel record, un certificato che termina a metà di un campo: sono tutti errori, non crash. Il parser JA4X, che percorre il formato più ostile (X.509 DER), è sottoposto a fuzzing con un property test per dimostrarlo.
- **Esaurimento della memoria tramite la flow table.** Un attaccante che apre milioni di flow, o invia segmenti con un buco che non si riempie mai, farebbe crescere senza limiti un reassembler illimitato. La flow table ha un limite rigido sui flow tracciati, un idle timeout che elimina quelli inattivi, e tetti per direzione sia sui byte contigui sia su quelli fuori ordine parcheggiati. Il costo di una capture ostile è fisso e sostenibile.
- **Una decifratura QUIC che rilascia plaintext non verificato.** Il percorso QUIC verifica il tag AEAD prima di fidarsi dei frame CRYPTO decifrati, così un Initial falsificato fallisce la verifica del tag e viene scartato invece di passare spazzatura al parser.

**Cosa il tool esplicitamente non fa:**

- **Non decifra i dati applicativi.** Legge solo ciò che è in chiaro sulla rete: l'handshake, l'intestazione HTTP in chiaro, l'Initial QUIC. Non ha mai una chiave privata e non vede mai i certificati TLS 1.3 o gli header HTTP/2, che sono cifrati.
- **Non fa attribuzione.** Un fingerprint identifica una toolchain, non un attore. Una corrispondenza `known_bad` significa "questo proviene da una toolchain nota come sospetta", non "questo è l'attaccante X". La collisione JA3 tra Emotet e Cobalt Strike ne è il promemoria permanente.
- **Non esegue probing attivo.** Non invia nulla. Un bersaglio non può rilevare il sensore osservando eventuali scansioni, perché non ce ne sono.

## Dove andare ora

- [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) segue un frame lungo tutta questa architettura, da `decode.rs` a una riga nella tabella `alert`.
- [ALGORITHMS.md](./ALGORITHMS.it.md) apre lo stadio di fingerprinting: l'esatta costruzione byte per byte di ogni hash e il key schedule di QUIC.
- [CONFORMANCE.md](./CONFORMANCE.it.md) registra il vettore pubblicato a cui è vincolato ogni stadio e ogni confine deliberato.
