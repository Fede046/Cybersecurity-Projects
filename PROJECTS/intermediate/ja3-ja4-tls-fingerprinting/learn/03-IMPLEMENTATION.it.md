<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# JA3/JA4 TLS Fingerprinting: implementazione

Questo documento percorre il codice. Segue un pacchetto dal momento in cui viene letto dal disco al momento in cui una riga finisce nella tabella `alert`, poi estrae i tre pattern che rendono l'engine sicuro su una rete ostile. Nomina file e funzioni, mai numeri di riga, così resta corretto anche quando il codice si sposta. Leggi prima [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per avere la mappa.

## Il viaggio di un frame, dall'inizio alla fine

Esegui `tlsfp pcap traffic.pcapng --detect`. Ecco cosa succede a un frame che trasporta un ClientHello.

### 1. La sorgente produce un frame

`PcapFileSource::open` (in `pipeline/source.rs`) apre la capture e `next_frame` restituisce un `RawFrame`: un timestamp in nanosecondi, il link type e una slice di byte presa in prestito. La slice prende in prestito il buffer di lettura della sorgente stessa, quindi qui non avviene alcuna copia. Il loop `Pipeline::run` del binario estrae i frame e passa ciascuno a `Pipeline::feed`.

### 2. Il decode rimuove i layer inferiori

`decode_frame` (in `pipeline/decode.rs`) usa `etherparse` per rimuovere gli header Ethernet e IP. Restituisce un valore `Decoded`: un `DecodedSegment` per TCP (con la quadrupla, il numero di sequenza, i flag e il payload), un `DecodedDatagram` per UDP, oppure uno `Skip` se il frame non è qualcosa di cui calcoliamo il fingerprint. Se il segmento è un SYN semplice, è anche qui che vengono percorse le opzioni TCP per JA4T, perché JA4T legge il SYN stesso, non lo stream.

### 3. La flow table riassembla lo stream

La quadrupla del segmento diventa una `FlowKey`, e `FlowKey::from_pair` restituisce anche una `Direction`, così che le due metà della conversazione restino separate. La pipeline cerca (o crea) lo `StreamReassembler` per quella direzione e chiama `push(seq, payload)`.

`StreamReassembler::push` (in `pipeline/flow.rs`) è la parte delicata. Colloca il payload al suo offset di sequenza, lo unisce alla sequenza contigua se riempie il buco, lo parcheggia se arriva in anticipo, e risolve qualsiasi sovrapposizione con i byte già presenti. Restituisce un `PushOutcome` che dice al chiamante se si sono resi disponibili nuovi byte contigui. I byte contigui sono raggiungibili tramite `data()`. È questo che permette a un ClientHello diviso su segmenti riordinati di superare il parsing: il parser vede sempre e solo lo stream pulito e in ordine.

### 4. Il layer di protocollo riconosce un handshake

Con nuovi byte contigui disponibili, `StreamProtocol` (in `pipeline/tls.rs`) ispeziona l'inizio dello stream. Legge l'header del record TLS e, quando è presente un messaggio di handshake completo, lo smista: un ClientHello a `parse_client_hello`, un ServerHello a `parse_server_hello`, un Certificate al lettore dei certificati. Per uno stream in chiaro che invece sembra HTTP, esegue il parsing dell'intestazione della richiesta per JA4H.

### 5. Il parser legge il ClientHello senza copiarlo

`parse_client_hello` (in `parse/hello.rs`) percorre il messaggio con un `Reader` (in `parse/reader.rs`), un cursore che esegue letture con bounds-checking sui byte presi in prestito. Ogni campo, la legacy version, le cipher suite, le estensioni, viene restituito come una slice o un piccolo vettore che prende in prestito il buffer originale (`ClientHello<'pkt>` porta con sé il lifetime). Una lunghezza che supera il buffer restituisce `Err(ParseError)`; il parser non legge mai oltre la fine. Accessor di comodo come `server_name`, `alpn_protocol` e `supported_groups` decodificano le singole estensioni su richiesta.

### 6. Viene calcolato il fingerprint

Il `ClientHello` sottoposto a parsing va a `ja3` (in `ja3.rs`) e a `ja4` (in `ja4.rs`). Ciascuno elimina GREASE, assembla la propria stringa e ne calcola l'hash, JA3 con MD5, JA4 con SHA-256 troncato. [ALGORITHMS.md](./ALGORITHMS.it.md) fornisce la costruzione esatta. I risultati, insieme a SNI, ALPN ed eventuale User-Agent, diventano uno `StreamEvent::ClientHello`, che `event.rs` avvolge con gli indirizzi e il timestamp in un `FingerprintEvent`. Questo è il valore che esce dall'engine.

### 7. Lo store giudica e rileva

Di nuovo nel binario, il `FingerprintEvent` va a `tlsfp-intel`. `IntelStore::match_event` esegue il matcher (`matcher.rs`) su ogni fingerprint contenuto nell'evento e restituisce un `MatchReport` per ciascuno che ha trovato corrispondenza. `IntelStore::detect` apre una transazione e chiama `detect::run` (in `detect.rs`), che:

1. registra l'evento come una riga in `observation`,
2. esegue le sei regole, correlando il nuovo fingerprint con ciò che questo IP e questo fingerprint hanno fatto all'interno della finestra temporale,
3. rende persistente ogni `Alert` che genera nella tabella `alert`,
4. esegue il commit, così che l'osservazione e i suoi alert vengano scritti in modo atomico.

### 8. Il writer stampa

Il binario serializza l'evento e gli eventuali report e alert, come riga leggibile o come JSON, e lo scrive su stdout. Per `--report` invece passa l'evento al `ReportBuilder`, che accumula e stampa un unico riepilogo alla fine. Il frame ha finito il suo percorso.

```
RawFrame ─decode─> DecodedSegment ─push─> contiguous bytes ─parse─> ClientHello
   │                                                                     │
   │                                                              ja3()/ja4()
   │                                                                     ▼
   └──────────────────────────────────────────────────────> FingerprintEvent
                                                                         │
                                          match_event + detect (one txn) │
                                                                         ▼
                                                       MatchReport + Alert ─> stdout
```

## Pattern uno: parsing zero-copy con bounds-checking

L'engine esegue il parsing di input binario ostile, quindi la strategia di parsing è la strategia di sicurezza. Due scelte reggono tutto il peso.

**Tutto viene preso in prestito.** `parse_client_hello` restituisce un `ClientHello<'pkt>` che contiene slice del buffer originale del pacchetto. Nessun campo viene copiato all'esterno durante il parsing. Questo mantiene l'hot path privo di allocazioni (i benchmark in `benches/fingerprint.rs` mostrano perché è importante) e significa che un ClientHello con cento estensioni non costa più memoria del pacchetto in cui è arrivato.

**Ogni lettura ha il bounds-checking, e gli sforamenti sono errori, non panic.** Il `Reader` è l'unica cosa che avanza attraverso i byte, e ciascuna delle sue letture controlla prima la lunghezza residua. Un campo di lunghezza troncato, un'estensione che dichiara più byte di quelli contenuti nel record, un certificato che termina a metà di un OID: ognuno di questi restituisce `Err(ParseError)`. I test di `parse/` forniscono esattamente questi input malformati e verificano che producano errori. Poiché l'intero crate imposta `unsafe_code = "forbid"` in `Cargo.toml`, non esiste una via di fuga attraverso cui un bug del parser possa diventare una lettura fuori dai limiti. Questa è la risposta strutturale alla classe di bug di Heartbleed discussa in [01-CONCEPTS.md](./01-CONCEPTS.it.md): in C il bug è una divulgazione di memoria, qui è un `Result::Err`.

## Pattern due: riassemblaggio limitato

`StreamReassembler` è il punto in cui l'input di un avversario potrebbe crescere senza limiti, quindi ogni dimensione è limitata da `ReassemblyLimits`:

- un tetto ai **byte contigui** mantenuti per direzione, così che un singolo stream enorme non possa crescere all'infinito,
- un tetto ai **byte fuori ordine parcheggiati**, così che un attaccante che invia il segmento 2 ma mai il segmento 1 non possa costringere il reassembler a tenere aperto il buco indefinitamente,
- un limite ai **segmenti parcheggiati**, così che molti piccoli segmenti fuori ordine non possano far esplodere la contabilità.

Quando uno stream raggiunge un limite viene marcato come `capped()` e smette di accettare altri dati, invece di crescere. A livello di pipeline, un limite sui flow e un idle timeout limitano il *numero* di flow, eliminando i più vecchi e inattivi quando la tabella è piena. I contatori (`Counters`, riportati con `-v`) espongono `segments_dropped` e `unfinished_tls_streams`, così che un operatore possa vedere quando i limiti sono intervenuti. Il principio di design: una capture ostile deve costare una quantità di memoria *fissa e nota*, mai illimitata. [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) inquadra questo aspetto come threat model; è qui che viene fatto rispettare.

## Pattern tre: decifratura passiva di QUIC

`quic.rs` è la parte dell'engine più impegnativa dal punto di vista crittografico, e vale la pena leggerne i commenti per intero. Il flusso:

1. **Individua l'Initial.** `InitialPacket::parse` trova un pacchetto Initial all'interno di un datagramma UDP e legge i campi in chiaro dell'header: la versione, il Destination Connection ID, il token. Un connection ID più lungo di venti byte indica che il pacchetto non è della versione che gestiamo, e viene rifiutato invece di essere interpretato male.
2. **Deriva le chiavi.** `InitialPacket::client_keys` chiama `InitialKeys::client(dcid, version)`. La derivazione è presa direttamente da RFC 9001 (v1) e RFC 9369 (v2): `HKDF-Extract` del connection ID con l'`INITIAL_SALT` della versione, `HKDF-Expand-Label` fino al segreto `"client in"`, poi espansione di quest'ultimo nella chiave AEAD, nell'IV e nella chiave di header protection. Il salt è l'unica cosa che differisce tra v1 e v2, ed è esattamente ciò che fa fallire la verifica del tag alle chiavi derivate con la versione sbagliata.
3. **Apri il pacchetto.** `InitialPacket::open` rimuove la header protection, poi esegue l'open AEAD. Il controllo del tag qui è portante: un osservatore passivo non può distinguere un Initial del client da uno del server tramite alcun campo in chiaro, quindi il fatto che *solo un pacchetto che il client ha effettivamente protetto con queste chiavi supererà la verifica* è ciò che identifica la direzione. Un pacchetto falsificato o del server fallisce la verifica del tag e viene scartato, senza mai essere passato al parser.
4. **Riassembla i frame CRYPTO.** `walk_crypto_frames` itera sui frame decifrati, e un `CryptoAssembler` li ricuce, in base all'offset, nel ClientHello TLS (un ClientHello QUIC può estendersi su più pacchetti Initial). `client_hello()` restituisce i byte assemblati.
5. **Calcola il fingerprint.** Il ClientHello assemblato passa attraverso lo stesso `ja4()` del percorso TCP, con `Transport::Quic`, producendo un fingerprint con prefisso `q`.

Il codice di derivazione delle chiavi è vincolato ai vettori di test delle RFC: `client_initial_keys_match_rfc9001_appendix_a1` e `client_initial_keys_match_rfc9369_appendix_a` derivano le chiavi e le confrontano byte per byte con i valori pubblicati nell'Appendice A. [CONFORMANCE.md](./CONFORMANCE.it.md) registra i confini del supporto a QUIC.

## Come leggere la codebase da solo

Un ordine produttivo, dato il percorso descritto sopra:

1. `parse/reader.rs`, poi `parse/hello.rs`. Il `Reader` racchiude l'intera storia della sicurezza in un piccolo file; `hello.rs` è il parsing più importante.
2. `ja4.rs`, poi `ja3.rs`. Il fingerprint principale, poi quello legacy che ha sostituito. Leggili fianco a fianco per vedere l'ordinamento.
3. `pipeline/flow.rs`. Il reassembler. I limiti sono la parte interessante.
4. `pipeline/mod.rs`. Il loop che collega gli stadi, dove `PipelineConfig` imposta i limiti.
5. `quic.rs`. Tienilo per ultimo; è autonomo e i commenti lo spiegano a sufficienza.

Poi passa a `tlsfp-intel`: `matcher.rs` per il punteggio, `detect.rs` per le regole. Infine `tlsfp/src/cli.rs` per vedere come tutto è collegato.

## Dove andare ora

- [ALGORITHMS.md](./ALGORITHMS.it.md) apre il passo 6 (e il key schedule QUIC del pattern tre) con un dettaglio a livello di byte.
- [CONFORMANCE.md](./CONFORMANCE.it.md) registra cosa ogni stadio accetta, rifiuta e a cosa è vincolato.
- [04-CHALLENGES.md](./04-CHALLENGES.it.md) suggerisce modifiche che ti costringeranno a capire questi pattern modificandoli.
