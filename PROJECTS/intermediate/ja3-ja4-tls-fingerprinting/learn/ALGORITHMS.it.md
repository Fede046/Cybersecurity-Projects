<!-- ©AngelaMos | 2026 -->
<!-- ALGORITHMS.md -->

# JA3/JA4 TLS Fingerprinting: algoritmi

Questo è il riferimento a livello di byte. Per ogni fingerprint fornisce la costruzione esatta, un esempio svolto vincolato a un vettore pubblicato e le regole che contano nei casi limite. Ogni esempio qui presente è anche un test nella codebase, quindi se dubiti di un passaggio puoi eseguirlo. Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per capire *perché* hanno la forma che hanno; questo documento spiega il *come*.

Una nota sugli hash troncati: JA4 e i suoi parenti prendono uno SHA-256 e conservano solo i **primi 12 caratteri hex** (i primi 6 byte). JA3 prende un MD5 completo (32 caratteri hex). Dove questo documento scrive "SHA-256 troncato", intende quei primi 12 caratteri hex.

## JA3 e JA3S

JA3 concatena cinque campi decimali del ClientHello, unisce i valori all'interno di un campo con `-` e i campi con `,`, e calcola l'MD5.

```
field 1: legacy_version          (one number)
field 2: cipher_suites           (GREASE stripped, wire order, joined with -)
field 3: extension_types         (GREASE stripped, wire order, joined with -)
field 4: supported_groups        (GREASE stripped, joined with -)
field 5: ec_point_formats        (joined with -)

joined with commas, then MD5
```

**Esempio svolto** (il vettore originale di Salesforce, `salesforce_client_vector_one` in `ja3.rs`):

```
769,47-53-5-10-49161-49162-49171-49172-50-56-19-4,0-10-11,23-24-25,0
                              │ MD5
                              ▼
              ada70206e40642a3e4461f35503241d5
```

Il `769` iniziale è il campo `legacy_version` in decimale: `769` è `0x0301`, TLS 1.0, che è ciò che conteneva questo vecchio vettore. Un client moderno TLS 1.2 o 1.3 mette qui `771` (`0x0303`), perché TLS 1.3 congela il campo legacy a 1.2 e segnala la versione reale in un'estensione. Qualunque cosa abbia scritto il client, viene resa come un semplice numero decimale. I campi lista sono mantenuti nell'**ordine di trasmissione**, ed è esattamente per questo che il rimescolamento dell'ordine delle estensioni ha rotto JA3.

**JA3S** è la risposta del server: tre campi dal ServerHello (versione, il singolo cipher selezionato, le estensioni), stessa unione con virgole, stesso MD5. Un server sceglie un solo cipher, quindi il campo due è un numero, non una lista.

L'intero motivo per cui questo tool calcola ancora JA3 è che i feed pubblici (`abuse.ch SSLBL` e il set curato di C2) sono espressi in JA3, e l'engine di rilevamento cerca il JA3 insieme al JA4 così da non perdere mai una corrispondenza in un feed.

## JA4 e JA4S

JA4 ha tre sezioni separate da underscore: un prefisso leggibile di 10 caratteri `_` l'hash dei cipher `_` l'hash delle estensioni. La costruzione si trova in `ja4()` e `ja4_prefix()` in `ja4.rs`.

### Il prefisso (10 caratteri)

```
  t    13    d    15    16    h2
  │    │     │    │     │     │
  │    │     │    │     │     └── first ALPN value: first + last char (or "00")
  │    │     │    │     └──────── extension count, two digits, capped at 99
  │    │     │    └────────────── cipher count, two digits, capped at 99
  │    │     └─────────────────── SNI: 'd' if a server_name extension is present, else 'i'
  │    └───────────────────────── TLS version word: 13, 12, 11, 10, or d1/d2/d3 for DTLS
  └────────────────────────────── transport: t=TCP, q=QUIC, d=DTLS
```

Due sottigliezze che il codice gestisce. La **versione** è il valore non-GREASE più alto nell'estensione `supported_versions`, non il campo `legacy_version`, perché un client TLS 1.3 congela il campo legacy a 1.2 (`select_version` in `ja4.rs`). I **caratteri ALPN** sono il primo e l'ultimo byte del primo protocollo ALPN se entrambi sono alfanumerici ASCII (quindi `h2` resta `h2`, `http/1.1` diventa `h1`); in caso contrario, il codice ripiega sul primo e sull'ultimo nibble hex; un ALPN assente è `00` (`alpn_chars` in `ja4.rs`).

### L'hash dei cipher

Prendi le cipher suite, elimina GREASE, formatta ciascuna come quattro cifre hex minuscole, **ordina le stringhe**, uniscile con `,` e calcola lo SHA-256 troncato.

**Esempio svolto** (`foxio_cipher_section_vector` in `ja4.rs`):

```
sorted cipher CSV:
002f,0035,009c,009d,1301,1302,1303,c013,c014,c02b,c02c,c02f,c030,cca8,cca9
                              │ SHA-256, first 12 hex
                              ▼
                        8daaf6152771
```

L'**ordinamento** è l'intera innovazione di JA4. Il client può inviarli in qualsiasi ordine, e un client moderno li rimescola, ma l'ordinamento elimina l'ordine e conserva l'insieme. Ecco perché questo hash è stabile laddove il campo tre di JA3 non lo era.

### L'hash delle estensioni

Prendi i tipi delle estensioni, **rimuovi SNI (`0x0000`) e ALPN (`0x0010`)** perché sono già rappresentati nel prefisso, elimina GREASE, formatta come quattro cifre hex, ordina, unisci con `,`. Poi, se è presente un'estensione `signature_algorithms`, aggiungi `_` e i signature algorithm **nel loro ordine originale** (non ordinati, perché il loro ordine è di per sé significativo e stabile). Calcola lo SHA-256 troncato dell'intero risultato (`ja4_extension_raw` in `ja4.rs`).

**Esempio svolto** (`foxio_extension_section_vector` in `ja4.rs`):

```
sorted exts (SNI + ALPN removed)              sig algs (original order)
0005,000a,000b,000d,0012,0015,0017,001b,    _ 0403,0804,0401,0503,
0023,002b,002d,0033,4469,ff01                  0805,0501,0806,0601
                              │ SHA-256, first 12 hex
                              ▼
                        e5627efa2ab1
```

### Mettere tutto insieme

```
t13d1516h2  _  8daaf6152771  _  e5627efa2ab1
```

Questo è un vero JA4 di Chrome. JA4 ha anche una forma **raw**, in cui le sezioni dei cipher e delle estensioni sono le stringhe CSV stesse invece dei loro hash, per il debug; il tool la emette con `--json`.

**JA4S** rispecchia tutto questo per il server: il server sceglie un cipher (quindi la sezione dei cipher è quel singolo valore in hex, non l'hash di una lista) e le sue estensioni vengono sottoposte a hash nell'**ordine di trasmissione, non ordinate**, perché un server non rimescola le proprie estensioni (`ja4s` in `ja4.rs`).

## JA4H: il client HTTP

JA4H calcola il fingerprint di una singola richiesta HTTP in chiaro (`ja4h.rs`). Legge il metodo, la versione HTTP, se la richiesta porta cookie e un referer, il numero e l'ordine dei nomi degli altri header, l'accept-language, e i nomi e i valori dei cookie. Il segnale che cattura: una richiesta che omette un accept-language e non invia cookie ha molte più probabilità di provenire da uno script che da una persona, e il prefisso lo rende visibile nei primi caratteri.

Funziona **solo su HTTP in chiaro**. Su HTTPS la richiesta è cifrata; su HTTP/2 gli header sono compressi con HPACK e illeggibili per un osservatore passivo. Quindi JA4H scatta sul semplice traffico `http://`, che su una rete moderna è quello dei client automatizzati e legacy, esattamente la popolazione interessante.

## JA4X: la toolchain dei certificati

JA4X calcola il fingerprint di un certificato X.509 in base a *come è stato costruito*, non a ciò che dice (`ja4x.rs`). Estrae tre liste di object identifier (OID): quelli nel nome dell'emittente, quelli nel nome del soggetto e quelli tra le estensioni, ciascuno nell'ordine in cui è stato scritto nel DER. Ogni lista viene unita con `,` e sottoposta a hash (SHA-256 troncato), producendo tre sezioni:

```
issuer_oids_hash  _  subject_oids_hash  _  extension_oids_hash
```

Due certificati generati dallo stesso software con lo stesso template condividono un JA4X anche quando ogni nome, numero seriale e chiave sono diversi, il che raggruppa i certificati di una stessa famiglia di malware o di una stessa toolchain di CA. In modo passivo funziona solo su **TLS 1.2 e precedenti**, dove il messaggio Certificate viaggia in chiaro; TLS 1.3 lo cifra. L'estrazione degli OID percorre DER controllato dall'attaccante attraverso il lettore minimale in `der.rs`, ed è per questo che `tests/ja4x.rs` include un harness di fuzzing basato su property test: un certificato costruito ad arte deve produrre un errore o un fingerprint, mai un panic.

## JA4T: lo stack TCP

JA4T calcola il fingerprint dello stack TCP/IP (il sistema operativo) a partire da un singolo SYN (`ja4t.rs`). Il formato è composto da quattro parti unite con `_`:

```
window_size _ option_kinds(joined with -) _ MSS _ window_scale
```

Gli option kind sono i *numeri di kind* delle opzioni TCP nell'ordine in cui compaiono nel SYN; un'opzione MSS o window scale mancante viene riportata come `0`.

**Esempi svolti** (`foxio_windows_default_vector` e `foxio_windowed_vector` in `ja4t.rs`):

```
Windows default:  64240_2-1-3-1-1-4_1460_8
Windowed:         65535_2-1-3-1-1-8-4-0-0_1346_6
```

JA4T è il layer che un tool di evasione dimentica. Uno script che impersona il TLS di Chrome viaggia comunque sullo stack TCP del sistema operativo che lo ospita, quindi il suo JA4T dice "Linux" mentre il suo JA4 falsificato e il suo `User-Agent` dicono "Windows Chrome". Quell'incoerenza a tre vie è ciò che le regole di rilevamento `os_mismatch` e `ua_mismatch` trasformano in un alert.

## Eliminazione di GREASE

Ogni fingerprint basato su liste descritto sopra elimina prima GREASE (`grease.rs`). I valori GREASE sono i sedici code point riservati della forma `0x?a?a` (`0x0a0a`, `0x1a1a`, ... `0xfafa`), definiti da RFC 8701. Un client li inietta a caso per mantenere estensibile l'ecosistema, quindi includerli farebbe cambiare il fingerprint a ogni connessione. `is_grease` è un singolo controllo rispetto a quel pattern, applicato in ogni builder di CSV prima dell'ordinamento o dell'hashing.

## Derivazione delle chiavi dell'Initial QUIC

Per leggere il ClientHello all'interno di un pacchetto Initial QUIC, il tool deriva le chiavi di protezione del pacchetto dal suo Destination Connection ID, che è in chiaro (`quic.rs`, `InitialKeys::client`). Lo schedule è HKDF (RFC 5869) con le label e il salt di RFC 9001 (v1) e RFC 9369 (v2):

```
initial_secret = HKDF-Extract(salt = INITIAL_SALT[version], ikm = DCID)
client_secret  = HKDF-Expand-Label(initial_secret, "client in", 32)
  key          = HKDF-Expand-Label(client_secret, <version key label>, 16)
  iv           = HKDF-Expand-Label(client_secret, <version iv label>, 12)
  hp           = HKDF-Expand-Label(client_secret, <version hp label>, 16)
```

L'**unica** cosa che differisce tra v1 e v2 è il salt (e le label di expand), e questo è voluto: le chiavi derivate con il salt della versione sbagliata falliscono la verifica del tag AEAD, ed è così che il tool distingue un pacchetto v1 da uno v2 senza fidarsi di un campo di versione che non ha ancora autenticato. Con le chiavi in mano, `InitialPacket::open` rimuove la header protection ed esegue l'open AEAD. Il controllo del tag è ciò che identifica la direzione *client*: un osservatore passivo non può distinguere un Initial del client da uno del server tramite alcun campo in chiaro, ma solo un pacchetto che il client ha effettivamente protetto con queste chiavi supera la verifica, quindi un open riuscito *è* la prova che si trattava di un Initial del client.

La derivazione è vincolata byte per byte alle RFC: `client_initial_keys_match_rfc9001_appendix_a1` e `client_initial_keys_match_rfc9369_appendix_a` in `quic.rs` derivano le chiavi dal connection ID dell'appendice e verificano i valori di chiave, IV e header protection dell'appendice.

## Scelte di hash, in sintesi

| Fingerprint | Hash | Perché |
|-------------|------|-----|
| JA3 / JA3S | MD5, completo | corrisponde alla definizione originale e a ogni feed JA3 pubblico |
| JA4 / JA4S | SHA-256, primi 12 hex | la definizione di FoxIO; resistente alle collisioni, più breve da leggere |
| JA4H / JA4X / JA4T | SHA-256, primi 12 hex (sezioni) | stessa famiglia, stesso troncamento |

MD5 per JA3 non è una scelta di sicurezza e qui non è una debolezza: un fingerprint è un identificatore, non un autenticatore, e il requisito è riprodurre esattamente gli hash dei feed. JA4 è passato a SHA-256 perché era un taglio netto con il passato e non c'era una base installata con cui restare compatibili.

## Dove andare ora

- [CONFORMANCE.md](./CONFORMANCE.it.md) elenca ogni vettore pubblicato a cui ciascuno di questi è vincolato, e l'esatto perimetro che ogni fingerprint copre e non copre.
- [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) mostra in quale punto della pipeline vengono chiamate queste funzioni.
