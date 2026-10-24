<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# JA3/JA4 TLS Fingerprinting: concetti

Questo documento spiega le idee su cui è costruito il tool, ciascuna ancorata a qualcosa che è successo davvero. Leggilo prima del codice. Una volta che questi concetti ti sono chiari, l'implementazione è solo una contabilità accurata.

## 1. Il ClientHello è una confessione

Una connessione TLS si apre con il client che invia un `ClientHello`, il primo vero messaggio sulla rete. Viaggia in chiaro, perché le due parti non si sono ancora accordate su una chiave con cui cifrare. Al suo interno, il client elenca esattamente come è configurato:

- le **versioni di TLS** che supporta,
- le **cipher suite** che offre, in uno specifico ordine di preferenza,
- una lista di **estensioni** (SNI, ALPN, supported groups, signature algorithms, key share e molte altre),
- le **curve ellittiche** ("supported groups") che è in grado di usare,
- gli **EC point format** che comprende.

Niente di tutto questo è segreto. Tutto viene scelto dal *software*, non dall'utente. Lo stack TLS di Chrome offre un insieme diverso, in un ordine diverso, rispetto a quello di Firefox, a `crypto/tls` di Go, a `ssl` di Python, alla libreria su misura all'interno di un malware. Due installazioni della stessa versione di Chrome producono ClientHello quasi identici; Chrome e uno script Python ne producono di palesemente diversi.

```
ClientHello
├── legacy_version:        0x0303  (TLS 1.2, frozen here even for TLS 1.3)
├── random:               32 bytes (ignored, it changes every time)
├── cipher_suites:        [0x1301, 0x1302, 0x1303, 0xc02b, ...]
├── compression_methods:  [0x00]
└── extensions:
    ├── 0x0000 server_name           "example.com"
    ├── 0x000a supported_groups      [0x001d, 0x0017, ...]
    ├── 0x000d signature_algorithms  [0x0403, 0x0804, ...]
    ├── 0x0010 alpn                  ["h2", "http/1.1"]
    ├── 0x002b supported_versions    [0x0304, 0x0303]
    └── ...
```

Un **fingerprint** trasforma quella struttura in una stringa breve e stabile. L'intuizione, dovuta a Salesforce nel 2017, è che la *forma* del ClientHello identifica il software anche quando ogni altro identificatore è falsificato. Un attaccante può cambiare gratis il proprio IP, il proprio dominio e il proprio certificato. Cambiare il proprio stack TLS significa ricostruire i propri strumenti.

## 2. JA3: l'originale, e perché è morto

JA3 (chiamato così dai suoi tre autori, John Althouse, Jeff Atkinson, Josh Atkins) costruisce il fingerprint nel modo più ovvio. Prende cinque campi in ordine, versione, cipher, estensioni, supported groups, point format, scrive ciascuno come numeri decimali, li concatena e ne calcola l'MD5:

```
769,47-53-5-10-49161-49162-49171-49172-50-56-19-4,0-10-11,23-24-25,0
                                  │
                                 MD5
                                  ▼
                  ada70206e40642a3e4461f35503241d5
```

Per anni ha funzionato magnificamente. Il profilo di default di Cobalt Strike, Trickbot, Emotet: ognuno aveva un JA3 noto, e feed come abuse.ch SSLBL pubblicavano migliaia di hash JA3 di malware che i difensori potevano bloccare direttamente.

Poi si è rotto, per un motivo specifico e istruttivo. **JA3 calcola l'hash delle estensioni nell'ordine esatto in cui il client le ha inviate.** Nel 2021 Chrome (seguito da altri) ha iniziato a fare qualcosa che la specifica TLS consente esplicitamente: rimescolare l'ordine delle proprie estensioni a ogni singola connessione, come misura deliberata contro l'ossificazione. La lista dei cipher è rimasta la stessa, le estensioni presenti sono rimaste le stesse, ma il loro *ordine* cambiava ogni volta. Dato che JA3 include l'ordine nell'hash, ogni connessione di Chrome ha cominciato a produrre un hash JA3 diverso. Da un giorno all'altro, JA3 è diventato inutile per il fingerprinting dei browser. Una firma che cambia ogni volta non identifica nulla.

JA3 però non è morto per i *malware*, ed è per questo che questo tool lo calcola ancora. Gli strumenti dei malware raramente rimescolano le estensioni, i feed pubblici sono ancora espressi in JA3, e osservare JA3 frammentarsi accanto a un JA4 stabile su traffico reale di browser è la dimostrazione più chiara in assoluto del perché esista il suo successore.

> Vale la pena soffermarsi sulla collisione tra Emotet e Cobalt Strike. Entrambi sono comparsi con l'hash JA3 `72a589da586844d7f0818ce684948eea`, non perché siano lo stesso malware, ma perché erano costruiti sulla stessa libreria TLS configurata nello stesso modo. Un fingerprint identifica una *toolchain*. Trattare una corrispondenza di fingerprint come prova di uno specifico attore è un errore; trattarla come "questo proviene da una toolchain nota come sospetta" è corretto.

## 3. JA4: prima ordina, e sopravvivi

JA4, pubblicato da FoxIO nel 2023, risolve il problema dell'ordine con una sola modifica: **ordina le liste dei cipher e delle estensioni prima di calcolarne l'hash.** Se l'insieme delle estensioni offerte da un client è stabile ma il loro ordine è casuale, allora ordinarle elimina la casualità e conserva il segnale. Le estensioni rimescolate di Chrome, una volta ordinate, tornano a essere ogni volta la stessa lista, quindi JA4 è di nuovo stabile.

JA4 ha anche cambiato il formato per renderlo *in parte leggibile da una persona*, cosa che l'MD5 opaco di JA3 non era. Un fingerprint JA4 ha tre parti separate da underscore:

```
t13d1516h2_8daaf6152771_e5627efa2ab1
│            │            │
│            │            └── truncated SHA-256 of the sorted extensions (+ sig algs)
│            └── truncated SHA-256 of the sorted cipher suites
└── 10 readable characters:
    t   transport: t=TCP, q=QUIC, d=DTLS
    13  TLS version: 1.3
    d   SNI present (d=domain, i=no SNI)
    15  cipher count (15)
    16  extension count (16)
    h2  first ALPN value (h2 = HTTP/2)
```

Puoi leggere il prefisso a colpo d'occhio: "TLS 1.3 su TCP, con SNI, 15 cipher, 16 estensioni, che parla HTTP/2". Due client con prefissi diversi sono visibilmente software diversi ancora prima che tu confronti gli hash. Gli hash poi distinguono i client che condividono un prefisso.

Le due scelte di design, l'ordinamento e un prefisso leggibile, sono l'intera ragione per cui JA4 ha avuto successo dove JA3 ha fallito. [ALGORITHMS.md](./ALGORITHMS.it.md) percorre l'esatta costruzione byte per byte.

## 4. GREASE: i client mentono di proposito

Se fai il dump di un vero ClientHello di Chrome, vedrai cipher suite ed estensioni con valori come `0x0a0a`, `0x1a1a`, `0x2a2a`. Non sono reali. Sono **GREASE** (Generate Random Extensions And Sustain Extensibility, RFC 8701), valori riservati casuali che un client inietta deliberatamente per mantenere onesto l'ecosistema, così che middlebox e server non possano iniziare a dare per scontato un insieme fisso di valori e rompere i client futuri che ne aggiungono di nuovi.

GREASE è veleno per il fingerprinting se lo ignori. Il punto centrale di GREASE è che i valori sono *casuali per ogni connessione*, quindi qualsiasi fingerprint che li includa cambia ogni volta, esattamente il problema dell'ordine di JA3 sotto altre vesti. Sia JA3 sia JA4 quindi **eliminano ogni valore GREASE** da ogni lista prima di calcolare l'hash. L'insieme dei valori GREASE è fisso e noto (il pattern `0x?a?a`, sedici valori), quindi eliminarli è un semplice filtro.

```
offered:  [0x1a1a, 0x1301, 0x1302, 0x1303, 0xc02b]   <- 0x1a1a is GREASE
hashed:           [0x1301, 0x1302, 0x1303, 0xc02b]   <- stripped before hashing
```

Questo tool tiene la tabella GREASE in un unico posto (`grease.rs`) e applica la stessa eliminazione a ogni fingerprint, perché un fingerprint che si dimentica di eliminare GREASE non è sbagliato occasionalmente, è sbagliato su ogni client moderno.

## 5. JA4+ è una famiglia, e un solo fingerprint non basta mai

JA4 è il fingerprint del client TLS, ma la stessa idea si applica ad altri layer, e i membri della famiglia (collettivamente "JA4+") si verificano a vicenda. Questo tool li calcola tutti:

| Fingerprint | Cosa identifica | Letto da |
|-------------|-------------------|-----------|
| **JA4** | il software del client TLS | il ClientHello |
| **JA4S** | il software del server TLS | il ServerHello |
| **JA4H** | il software del client HTTP | una richiesta HTTP in chiaro |
| **JA4X** | la toolchain che emette il certificato | un certificato X.509 |
| **JA4T** | lo stack TCP/IP del client (il sistema operativo) | il pacchetto SYN |

Il motivo per calcolarne diversi è l'**evasione**. Una classe crescente di tool (`curl-impersonate`, `utls` e altri) esiste proprio per falsificare il ClientHello di un browser. Rendono il JA4 di uno script identico a quello di un vero Chrome, così che un filtro basato solo su JA4 li lasci passare. Ma falsificare un layer non falsifica gli altri. Uno script Python che gira su Linux e impersona il TLS di Chrome emette comunque un SYN TCP *Linux*, quindi il suo JA4T lo tradisce. Invia comunque gli header HTTP nell'ordine tipico di uno script, quindi il suo JA4H lo tradisce. Dichiara comunque `User-Agent: ...Chrome...`, che ora *non è coerente* con il suo stack TCP.

Quell'incoerenza è il segnale più prezioso in assoluto prodotto da questo tool, ed è la regola di rilevamento principale. [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) descrive come le regole li combinano, e [ALGORITHMS.md](./ALGORITHMS.it.md) fornisce ciascuna costruzione.

> Questa è la lezione della corsa agli armamenti dell'impersonazione dei browser. Finché i difensori calcolavano il fingerprint solo del TLS, gli attaccanti dovevano falsificare solo il TLS. La difesa non è un singolo fingerprint migliore; è la *correlazione tra layer che l'attaccante controlla in modo indipendente*. Un attaccante che li falsifica tutti alla perfezione ha, di fatto, ricostruito un vero browser, cosa costosa e rara.

## 6. QUIC passivo: "cifrato" non significa "segreto"

QUIC, il trasporto sotto HTTP/3, porta il proprio handshake TLS all'interno dei suoi primi pacchetti, e quei pacchetti sono cifrati. Sembrerebbe che un osservatore passivo sia tagliato fuori. Non è così, e il motivo è una splendida sottigliezza.

I pacchetti *Initial* di QUIC sono cifrati, ma con una chiave che è **derivata dal Destination Connection ID della connessione, che viene inviato in chiaro nell'header del pacchetto.** La cifratura esiste per sconfiggere i middlebox poco intelligenti che altrimenti altererebbero i pacchetti, non per mantenere segreto l'handshake. Chiunque legga il pacchetto può derivare la stessa chiave, decifrare l'Initial e leggere il ClientHello al suo interno, esattamente come se fosse TCP.

```
QUIC Initial packet
├── header (cleartext):  ... Destination Connection ID = D ...
└── payload (encrypted under a key derived from D)
                          │
        HKDF(D) ──────────┘   anyone can compute this
                          ▼
                  decrypted CRYPTO frames
                          ▼
                  TLS ClientHello  ->  q-transport JA4
```

Questo tool deriva le client initial key dal Connection ID di ciascun pacchetto, seguendo RFC 9001 per QUIC v1 e RFC 9369 per QUIC v2, senza alcun segreto lato server e senza rilevamento della direzione. Riassembla i frame CRYPTO (un ClientHello QUIC può estendersi su più pacchetti Initial) e passa il risultato allo stesso codice JA4 usato dal percorso TCP. Il fingerprint esce con un marcatore di trasporto `q`. [ALGORITHMS.md](./ALGORITHMS.it.md) fornisce il key schedule completo.

## 7. La cattura passiva e il problema del riassemblaggio

Tutto quanto sopra presuppone che tu possa vedere un ClientHello pulito. In una capture reale non puoi. TCP consegna uno *stream* di byte spezzettato in segmenti che arrivano fuori ordine, vengono ritrasmessi e a volte si sovrappongono. Un ClientHello abbastanza grande da estendersi su due segmenti non supererà il parsing se guardi uno solo dei due segmenti.

Quindi, prima di qualsiasi fingerprinting, il tool deve fare ciò che TCP stesso fa all'endpoint: **riassemblare ciascuna direzione di ciascuna conversazione in uno stream di byte contiguo.** Tiene traccia di ogni flow (una quadrupla di indirizzo e porta di origine e di destinazione), mette in buffer i segmenti fuori ordine finché il buco che li precede non si riempie, e risolve le sovrapposizioni. Solo quando una sequenza contigua di byte contiene un record TLS completo viene eseguito il parser.

È anche qui che risiede la **sicurezza del sensore stesso**. Un tool passivo elabora per definizione byte controllati dall'attaccante: chiunque sulla rete monitorata può inviargli ciò che vuole. Per questo il layer di riassemblaggio è limitato in ogni dimensione: un numero massimo di flow tracciati, un idle timeout che elimina quelli inattivi, un tetto ai byte fuori ordine in buffer per ciascuna direzione. Senza questi limiti, un attaccante potrebbe aprire milioni di flow, o inviare segmenti con un buco permanente che non si riempie mai, e guardare la memoria del sensore salire finché l'host non muore. I limiti trasformano un input ostile illimitato in un costo fisso e sostenibile. [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) mostra i limiti esatti.

## 8. Qui la memory safety è una proprietà di sicurezza

Questo tool è scritto in Rust, e l'engine di fingerprinting vieta `unsafe` del tutto. Non è una preferenza stilistica; è una risposta diretta a ciò che questo codice fa.

Ogni parser in `tlsfp-core` legge **input binario controllato dall'attaccante**: un record TLS dalla rete, un certificato X.509 da un server ostile, un pacchetto QUIC da chissà dove. Questa è esattamente la categoria di codice che produce le peggiori vulnerabilità nel mondo C. **Heartbleed (CVE-2014-0160)** era un bug di parsing TLS, un campo di lunghezza considerato affidabile senza bounds-checking, che permetteva a un attaccante di leggere la memoria del server. Un sensore di fingerprinting non è *altro che* parsing di TLS e di certificati, quindi in C sarebbe una fabbrica di Heartbleed.

In Rust, un campo di lunghezza che dichiara più byte di quelli esistenti produce un `ParseError` tipizzato, non una lettura fuori dai limiti. Il parser dei certificati di JA4X, che percorre il formato più favorevole agli attaccanti in assoluto (X.509 DER), ha un harness di fuzzing basato su property test che gli lancia contro certificati casuali e troncati proprio per dimostrare che restituisce errori invece di andare in panic. La sicurezza è la caratteristica che permette a un sensore passivo di stare su una rete ostile senza diventare esso stesso la vulnerabilità. [CONFORMANCE.md](./CONFORMANCE.it.md) registra esattamente cosa accettano e cosa rifiutano i parser.

## Letture di approfondimento

- [Specifiche tecniche JA4+ di FoxIO](https://github.com/FoxIO-LLC/ja4/tree/main/technical_details): le definizioni ufficiali di JA4, JA4S, JA4H, JA4X, JA4T.
- [Salesforce, "TLS Fingerprinting with JA3 and JA3S" (2017)](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/): l'originale.
- [RFC 8701](https://www.rfc-editor.org/rfc/rfc8701.html): GREASE.
- [RFC 9001](https://www.rfc-editor.org/rfc/rfc9001.html) e [RFC 9369](https://www.rfc-editor.org/rfc/rfc9369.html): l'uso di TLS in QUIC, v1 e v2.
- [abuse.ch SSLBL](https://sslbl.abuse.ch/): uno dei feed pubblici da cui questo tool attinge i dati iniziali.
