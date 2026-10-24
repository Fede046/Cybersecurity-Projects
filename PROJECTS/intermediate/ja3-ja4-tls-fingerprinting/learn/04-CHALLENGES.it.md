<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.md -->

# JA3/JA4 TLS Fingerprinting: sfide

Il modo più veloce per capire questa codebase è modificarla. Queste sfide sono ordinate all'incirca per difficoltà, ciascuna indica i file che tocca, e ciascuna si conclude con il modo in cui dimostreresti che funziona, nello stesso modo in cui il resto del progetto dimostra le cose: un known-answer test, una capture di integrazione o un'esecuzione reale del binario. Leggi prima [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) e [ALGORITHMS.md](./ALGORITHMS.it.md).

## Principiante

Queste ti permettono di prendere confidenza con la pipeline senza toccare la crittografia.

### 1. Aggiungi un tipo di fingerprint a `intel lookup`

Il comando `intel lookup` della CLI accetta un insieme fisso di tipi di fingerprint. Segui `FpKind::from_token` in `model.rs` e il messaggio di errore in `cli.rs`, e assicurati che ogni tipo emesso dall'engine sia accettato (e che il testo di aiuto li elenchi).

*Dimostralo:* esegui `tlsfp intel lookup ja4t 64240_2-1-3-1-1-4_1460_8` e ottieni un verdetto invece di un errore "unknown kind".

### 2. Una nuova colonna di output leggibile

L'output in streaming stampa il JA4 e il JA3 per ogni riga. Aggiungi come colonna il dominio registrato dello SNI (l'eTLD+1), oppure un flag di un carattere quando il fingerprint ha avuto una corrispondenza di intelligence. La formattazione si trova in `write_event` in `cli.rs`.

*Dimostralo:* esegui `tlsfp pcap testdata/pcap/tls-handshake.pcapng` e leggi la nuova colonna.

### 3. Aggiungi un quarto feed al seeding

`seed.rs` compila tre feed CSV nel binario. Aggiungine un quarto (trova un feed pubblico di JA4 o JA3, registra la sua licenza in `NOTICE.md` e nella tabella `intel_source`). Segui le funzioni `load_*` esistenti e la forma di `NewFingerprint`.

*Dimostralo:* `tlsfp intel seed` e poi `tlsfp intel stats` mostrano il nuovo feed e il suo numero di righe.

## Intermedio

Queste richiedono di capire un parser o l'engine di rilevamento.

### 4. Una nuova regola di rilevamento

`detect.rs` esegue sei regole su una finestra temporale, leggendo la tabella `observation` come memoria. Aggiungine una settima. Un buon candidato: **`port_anomaly`**, un handshake TLS il cui JA4 indica HTTP/2 (ALPN `h2`) che arriva su una porta diversa da 443 o 8443, una caratteristica comune dei C2. La regola legge la porta di destinazione dell'evento e l'ALPN già presente nel prefisso; non ha bisogno di una nuova tabella.

*Dimostralo:* aggiungi uno unit test in `tests/detect.rs` che fornisca un `FingerprintEvent` costruito ad hoc e verifichi l'alert, nello stesso modo dei test esistenti sulle regole.

### 5. Decodifica gli handshake DTLS

Il codice JA4 ha già un marcatore di trasporto `d` e una word di versione DTLS, ma la pipeline non gli fornisce mai un handshake DTLS. DTLS viaggia su UDP e incapsula il proprio handshake in modo diverso da TLS (aggiunge numeri di sequenza dei messaggi e offset dei frammenti). Collega un percorso DTLS in `pipeline/decode.rs` e un lettore di record DTLS accanto a quello TLS in `parse/record.rs`, poi instrada verso il `ja4` esistente con `Transport::Dtls`.

*Dimostralo:* cattura un handshake WebRTC o OpenVPN-DTLS, aggiungilo a `testdata/pcap/` e aggiungi un test di integrazione che verifichi un JA4 con trasporto `d`.

### 6. Calibra e misura i limiti

I `ReassemblyLimits` del reassembler e il `PipelineConfig` della pipeline impostano i tetti di memoria. Costruisci con un piccolo script una capture deliberatamente ostile (migliaia di flow, segmenti con buchi permanenti), e osserva i contatori (`-v`) mostrare `segments_dropped` e l'eliminazione dei flow. Poi trova i limiti più piccoli che consentono ancora di calcolare in modo pulito i fingerprint delle capture incluse.

*Dimostralo:* un'esecuzione di `cargo bench` prima e dopo, più i contatori di `-v` sulla tua capture ostile, che mostrino una memoria limitata.

## Avanzato

Queste arrivano fino alla crittografia o all'architettura.

### 7. Vincola un vettore completo di pacchetto protetto QUIC v2

[CONFORMANCE.md](./CONFORMANCE.it.md) osserva che il *key schedule* v2 è vincolato all'Appendice A di RFC 9369, mentre il *pacchetto protetto* v2 completo non lo è, perché l'unica trascrizione disponibile aveva dei byte corrotti. Ottieni i byte del pacchetto protetto v2 da una fonte esatta al byte (una capture grezza o un hex dump affidabile, non un copia-incolla passato attraverso uno strumento di riassunto), aggiungilo ai test di `quic.rs` e verifica il ClientHello decifrato.

*Dimostralo:* un nuovo KAT in `quic.rs` che decifri il pacchetto v2 e corrisponda al ClientHello atteso, accanto al test esistente sul pacchetto completo v1.

### 8. Scansione attiva dei fingerprint (un tool separato)

Questo sensore è rigorosamente passivo. Costruisci un binario *separato* (un nuovo crate, così che la garanzia di passività in `tlsfp-core` non venga mai violata) che si connetta attivamente a un host, completi un handshake e calcoli il JA4S del *server*, nel modo in cui fanno gli scanner in stile `tlsd`. Questo è un threat model diverso (ora stai inviando pacchetti e sei rilevabile), quindi deve essere un tool chiaramente separato.

*Dimostralo:* calcola il fingerprint di un server noto (per esempio un host Cloudflare) e conferma che il suo JA4S è stabile tra un'esecuzione e l'altra.

### 9. Una vista a cluster del catalogo

La dashboard mostra i singoli fingerprint. Aggiungi una vista che li **raggruppi in cluster**: raggruppa i JA4 per prefisso condiviso (stessa versione TLS, numero di cipher, numero di estensioni, ALPN), così che un analista veda "questi quaranta fingerprint sono tutti variazioni di un unico stack TLS". I dati sono già nella colonna `part_a` usata dal fuzzy matcher di JA4.

*Dimostralo:* una vista nella dashboard, più una query in `matcher.rs` che restituisca i gruppi di prefissi con il numero dei loro membri.

## Esperto

Queste sono di livello da ricerca e aperte.

### 10. Rileva l'impersonazione tramite l'incoerenza tra layer, statisticamente

Le regole `ua_mismatch` e `os_mismatch` sono euristiche e binarie. Sostituiscile con un modello: per ogni tripla `(JA4, JA4T, User-Agent)` vista sulla rete, apprendi la distribuzione congiunta *normale*, poi assegna un punteggio alle nuove triple in base a quanto sono sorprendenti. Un vero Chrome su Windows è comune; un JA4 di Chrome falsificato con un JA4T Linux è raro e dovrebbe ottenere un punteggio alto. Questa è la versione rigorosa del rilevamento dell'evasione descritto in [01-CONCEPTS.md](./01-CONCEPTS.it.md).

*Dimostralo:* fornisci una capture contenente traffico `curl-impersonate` (generalo tu stesso) e mostra che il modello lo segnala lasciando in pace i browser reali.

### 11. Sopravvivi a un avversario che calcola il fingerprint del tuo sensore

Un attaccante sofisticato sulla rete monitorata potrebbe provare a *rilevare* o *eludere* il sensore costruendo traffico che sfrutta i limiti: flow calibrati per restare appena sotto il timeout di eliminazione, ClientHello frammentati per sconfiggere il riassemblaggio, GREASE in posizioni inattese. Costruisci il threat model del sensore stesso, crea le capture ostili e rendi la pipeline più robusta contro ciascuna di esse. Questo è l'analogo, per un sensore passivo, dell'evasione di un IDS.

*Dimostralo:* un insieme documentato di capture ostili in `testdata/`, ciascuna con un test che verifichi che il sensore calcoli correttamente il fingerprint oppure fallisca in modo sicuro (limitato, nessun crash, nessun handshake mancato che rientrasse nei limiti).

### 12. Encrypted Client Hello (ECH)

L'intero progetto si basa sul fatto che il ClientHello sia in chiaro. **ECH** (Encrypted Client Hello) è l'iniziativa IETF per cambiare questa situazione, cifrando le parti sensibili del ClientHello con una chiave pubblicata nel DNS. Studia la bozza di ECH, determina esattamente cosa un osservatore passivo può e non può ancora vedere (il ClientHello esterno è ancora visibile, ed è a sua volta soggetto a fingerprinting), e scrivi un'analisi di come JA4 si degrada con l'adozione di ECH e di quale segnale sopravvive.

*Dimostralo:* un'analisi scritta fondata sulla specifica di ECH, più una capture di un handshake ECH che mostri di cosa il tool può ancora calcolare il fingerprint a partire dall'hello esterno.

## Una nota sul contribuire a tua volta

Se costruisci qui qualcosa di davvero utile, l'ecosistema JA4+ è giovane e i feed pubblici sono scarni. Un feed con una buona licenza, una vista a cluster o un'analisi rigorosa di ECH sarebbero benvenuti upstream su [FoxIO-LLC/ja4](https://github.com/FoxIO-LLC/ja4) e nella community più ampia. Lo scopo del progetto è imparare come funziona tutto questo; il bonus è che il settore ha ancora problemi aperti che un pomeriggio di lavoro può far avanzare.
