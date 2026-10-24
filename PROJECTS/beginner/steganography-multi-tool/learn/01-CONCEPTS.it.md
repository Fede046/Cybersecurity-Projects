<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# crypha: Concetti

Questo capitolo è la teoria su cui è costruito crypha. Spiega perché occultamento e segretezza sono problemi diversi, come ciascuno dei cinque carrier nasconde i byte, cosa fa la busta cifrata, e come un difensore riesce comunque a scoprire ogni carrier. Ogni affermazione qui è messa in pratica dal codice che leggerai in [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md).

## La steganografia non è crittografia

Queste due parole vengono usate in modo intercambiabile e non dovrebbero esserlo. Risolvono problemi diversi e si combinano.

- **La crittografia fornisce segretezza.** Trasforma un messaggio in modo che, senza la chiave, il contenuto sia illeggibile. Non nasconde il fatto che un messaggio esista. Un blocco PGP, uno zip cifrato, una sessione TLS: tutti annunciano a gran voce "qui c'è un segreto", semplicemente ti negano il contenuto.
- **La steganografia fornisce occultamento.** Nasconde il fatto stesso che un messaggio esista, incorporandolo in qualcosa che sembra insignificante. La steganografia classica non protegge, di per sé, il contenuto: se trovi lo schema di occultamento, leggi il messaggio.

Il punto debole di ciascuna è il punto di forza dell'altra. Il blob opaco della cifratura è un segnale, e i segnali vengono segnalati. Un sistema di data-loss-prevention non ha bisogno di rompere la tua cifratura per fermare un'esfiltrazione; gli basta notare un allegato ad alta entropia da 40 KB che esce dalla rete e metterlo in quarantena. La steganografia sconfigge questo meccanismo non producendo mai, fin dall'inizio, un oggetto sospetto. Il carrier è una foto, una canzone, una fattura.

crypha le sovrappone entrambe, in quest'ordine:

```
message ──> [ compress ] ──> [ encrypt ] ──> envelope bytes ──> [ hide in carrier ] ──> stego file
```

Cifra per la segretezza, poi nascondi per l'occultamento. Se l'occultamento fallisce e qualcuno estrae la busta, ha comunque in mano un ciphertext che non può leggere. Se la cifratura venisse in qualche modo rotta, avrebbe comunque dovuto accorgersi che il payload era lì. I due livelli coprono l'uno le debolezze dell'altro. Questa è l'intera tesi progettuale del tool.

## L'idea di carrier

Un **carrier** è un formato di file con spazio per memorizzare byte che un osservatore casuale, e idealmente uno automatico, non noterà. crypha tratta un carrier come una scatola nera con quattro operazioni: nascondere byte in una copertina, rivelare byte da un file stego, riportare quanti byte una data copertina può contenere, e fiutare se un file sembra anche solo questo tipo di carrier. I byte che memorizza sono sempre la busta opaca; il carrier non sa né si interessa di cosa ci sia dentro.

Esistono due grandi famiglie di carrier, e crypha le implementa entrambe:

- **I carrier a sostituzione** sovrascrivono i bit poco importanti di dati esistenti. I carrier image e audio sostituiscono i bit meno significativi. Il carrier QR sostituisce interi codeword all'interno di un budget di correzione degli errori. Questi modificano leggermente i dati della copertina e sono rilevabili con la statistica.
- **I carrier additivi / strutturali** accodano dati che il formato ignora, oppure usano campi che il formato tollera. Il carrier di testo a larghezza zero accoda rune invisibili. I carrier PDF allegano un file, scrivono chiavi di metadati personalizzate oppure accodano byte dopo il marcatore di fine file. Questi non alterano affatto il contenuto visibile, ma sono trovati banalmente da chiunque guardi i byte grezzi.

Nessuna delle due famiglie è "migliore". Bilanciano capacità, robustezza e stealth in modo diverso, ed è esattamente per questo che un tool didattico ne implementa cinque.

## Canali uno e due: sostituzione del bit meno significativo

Un canale colore a 8 bit memorizza un valore da 0 a 255. Cambiare il suo bit più basso modifica il valore al massimo di 1, una differenza che nessun occhio può vedere in una fotografia. Un campione audio a 16 bit si colloca su una scala di circa 98 dB di gamma dinamica; invertire il suo bit più basso lo sposta di circa -96 dBFS, sotto il rumore di fondo di qualsiasi ambiente d'ascolto reale. Quindi sia le immagini sia l'audio hanno un bit di riserva per campione che puoi sovrascrivere con il payload, un bit di payload alla volta.

```
original R channel   1 0 1 1 0 0 1 [1]     payload bit = 0
after embedding      1 0 1 1 0 0 1 [0]     value changed 179 -> 178, invisible
                                  ^ least-significant bit carries one payload bit
```

La capacità è semplice. Un'immagine contiene `width x height x 3 / 8` byte, tre canali RGB per pixel a un bit ciascuno, otto bit per byte. Il canale alfa non viene mai toccato, di proposito, perché un pixel con alfa 254/255 invece di 255 appare come un leggero alone su uno sfondo non bianco ed è un indizio diretto per la steganalisi. Un'immagine 640x480 fornisce `640 x 480 x 3 / 8 = 115,200` byte di bit di canale; crypha riserva un prefisso di lunghezza di quattro byte e riporta **115,196** byte di busta utilizzabili. L'audio contiene `samples x channels / 8`; due secondi a 44.1 kHz mono contengono **11,021** byte.

Il problema, ed è grosso, è la fragilità. La steganografia LSB sopravvive a una copia byte per byte e a nient'altro. Ricodifica il PNG come JPEG e la quantizzazione della trasformata discreta del coseno scarta proprio il dettaglio spaziale di basso ordine in cui viveva il tuo payload. Ricodifica il WAV come MP3 e la compressione psicoacustica butta via i bit inudibili, compresi i tuoi. Questo non è un bug di crypha; è la proprietà che definisce la sostituzione in un canale solo lossless. crypha rifiuta del tutto le copertine JPEG proprio per questo motivo.

## Canale tre: iniezione di errori Reed-Solomon nei codici QR

Questo è il pezzo forte, ed è quello che le persone sbagliano. Un codice QR non è solo una bitmap di un URL. È un codice a correzione d'errore sistematico: i dati sono memorizzati in **codeword di dati**, e accanto a questi l'encoder calcola **codeword di correzione d'errore** in modo che uno scanner possa ancora leggere il codice quando una parte è sporca, strappata o coperta da un logo. Al livello di correzione d'errore H, un codice QR può perdere circa il 30% dei suoi codeword ed essere comunque decodificato.

L'idea ingenua, e sbagliata, è "nascondere i dati nei bit di correzione d'errore". L'idea corretta la ribalta. Lasci intatti i codeword di correzione d'errore e **corrompi deliberatamente i codeword di dati**, restando entro il budget di correzione del codice. Reed-Solomon sul campo GF(2^8) può correggere fino a `t = floor((n - k) / 2)` errori in posizione sconosciuta per blocco, dove `n` è il numero totale di codeword in un blocco e `k` sono i codeword di dati. Quindi:

1. Codifica il contenuto di copertura innocuo (diciamo, un URL) in un codice QR pulito.
2. Inietta fino a `t` errori di codeword per blocco nella regione dei dati. Lo schema degli errori è il tuo payload.
3. Il decoder Reed-Solomon di un normale scanner corregge quegli errori, recupera l'URL originale e mostra all'utente un link perfettamente normale. Non saprà mai che lì c'era qualcosa.
4. crypha legge lo stesso codice, ricava di nuovo come avrebbero dovuto essere i dati puliti decodificando RS ogni blocco, fa il diff tra i dati corretti e quelli ricevuti, e rilegge lo schema di errori iniettato come payload.

```
clean data block      D0 D1 D2 D3 ...          EC codewords intact
inject payload        D0 D1^p0 D2 D3^p1 ...     up to t corrupted codewords per block
phone scanner         RS-corrects -> D0 D1 D2 D3, reads the URL, sees nothing
crypha                RS-corrects -> D0 D1 D2 D3, XOR-diffs -> p0 p1, reads the payload
```

Il ricevente ha bisogno solo del codice e della versione, non di una copia separata della copertina; la macchina di correzione ricostruisce per lui i dati puliti. Questa è l'estrazione "cieca".

Il costo onesto è la capacità. Ogni codeword corrotto trasporta un byte di payload, e crypha si ferma a metà del budget teorico per avere margine di decodifica, quindi un QR piccolo contiene qualche decina di byte. La copertina di 28 caratteri `https://angelamos.com/crypha` dà una capacità di busta di **52 byte**, che diventano **38 byte** di testo in chiaro dopo il framing e **non bastano** affatto per una busta cifrata. Le percentuali spesso citate del 7/15/25/30% per L/M/Q/H sono la frazione di correzione d'errore sul totale dei codeword, non la dimensione del payload. Non confonderle. QR è un carrier solo in chiaro, e il tool lo dice.

## Canale quattro: testo Unicode a larghezza zero

Unicode contiene caratteri che vengono resi come nulla. crypha ne usa esattamente due:

```
U+200B  ZERO WIDTH SPACE  ->  bit 0
U+2060  WORD JOINER       ->  bit 1
```

Entrambi sono caratteri di formato con la proprietà `Default_Ignorable`, entrambi hanno larghezza di visualizzazione zero, e nessuno dei due causa comportamenti di shaping o di unione, il che esclude caratteri come U+200D (lo zero-width joiner) che fondono visibilmente le emoji. Otto di queste rune codificano un byte di payload. crypha accoda una sequenza incorniciata di queste rune dopo il testo visibile della copertina, così `The quick brown fox.` si legge ancora come `The quick brown fox.` pur trasportando dietro di sé un payload invisibile.

Un mito persistente dice che la normalizzazione Unicode elimina questi caratteri. Non è così. Nessuna delle quattro forme di normalizzazione, NFC, NFD, NFKC, NFKD, ha una mappatura di decomposizione per U+200B o U+2060, quindi tutte e quattro li lasciano esattamente al loro posto. Il mito nasce dalla lettura errata di due algoritmi non correlati: IDNA2003 mappa gli zero-width joiner sul nulla, ma solo all'interno delle etichette dei nomi di dominio, non nel testo dei messaggi; e l'algoritmo confusable-skeleton di UTS #39 rimuove i caratteri ignorabili, ma quello è un controllo di sicurezza sugli identificatori, non una pipeline di testo. Un normale copia-incolla, un round-trip su database o un messaggio di chat preservano il payload.

Il compromesso qui è l'opposto del QR. La capacità è di fatto illimitata e il testo visibile è identico byte per byte, ma il rilevamento è banale. Questo è occultamento da una lettura umana veloce, non un canale nascosto contro una macchina.

## Canale cinque: struttura PDF

Un PDF è un contenitore con spazio libero in diversi punti, e crypha ne usa tre, offerti come `--technique`:

- **attachment** (predefinito): il PDF supporta gli allegati di file incorporati come funzionalità di prima classe. crypha allega la busta come file incorporato lossless. È robusto, sopravvive a un normale salvataggio in Acrobat o Anteprima, ed è il default corretto per un carrier didattico perché il round-trip è pulito e facile da verificare. Il suo stealth è il più debole dei tre: l'allegato compare nel pannello e in `pdfinfo`.
- **metadata**: il dizionario Info del PDF tollera chiavi personalizzate, e la specifica richiede ai lettori di ignorare le chiavi che non riconoscono. crypha memorizza un payload in base64 distribuito su chiavi personalizzate. Sopravvive a una consegna controllata ma è fragile contro un passaggio di "ottimizzazione" o "linearizzazione" che elimina le chiavi non standard.
- **append**: la specifica dice che un lettore torna indietro fino all'ultima tabella di riferimenti incrociati, quindi qualunque cosa dopo il marcatore finale `%%EOF` senza una xref valida viene ignorata. crypha accoda lì la busta incorniciata. È l'opzione più pura di "zero modifiche alla struttura del documento" e sopravvive a copie ingenue, ma qualsiasi riscrittura completa con "salva con nome" la scarta.

Tutte e tre sono strutturali, non statistiche, quindi nessuna cambia un singolo pixel renderizzato del documento. Tutte e tre vengono trovate all'istante da chiunque guardi i byte grezzi.

## La busta cifrata

Qualunque sia il carrier, ogni payload viene prima impacchettato in un'unica busta versionata. Il carrier memorizza sempre e solo questi byte opachi; tutta la crittografia, la compressione, l'integrità e il versioning vivono in un unico posto. Questo è il layout, con l'overhead esatto:

```
plaintext    magic(4) ver(1) flags(1) │ len(4) body(N) │ crc32(4)                              +14 bytes

encrypted    magic(4) ver(1) flags(1) cipher(1) params(9) salt(16) nonce(12) │ len(4) body(N+16) │ crc32(4)
             └───────────────── authenticated as AEAD associated data ───────┘                          +68 bytes
```

Tre idee fanno il lavoro qui.

**Argon2id per la chiave.** Una passphrase non è una chiave. Argon2id (RFC 9106) è una funzione di derivazione della chiave memory-hard: trasforma una passphrase più un salt casuale di 16 byte in una chiave di 32 byte bruciando deliberatamente memoria e tempo, così un attaccante che prova a indovinare le passphrase paga quel costo per ogni tentativo. Il default di crypha è il profilo sicuro per laptop della RFC (64 MiB, tre passate); `--strength high` usa il profilo da 2 GiB. I parametri esatti usati vengono scritti nella busta, così `reveal` riproduce la stessa chiave senza che tu debba dichiarare di nuovo il profilo.

**Cifratura autenticata per segretezza e integrità insieme.** crypha usa di default ChaCha20-Poly1305 (RFC 8439), con AES-256-GCM dietro `--cipher aes256gcm`. ChaCha20 è il default di proposito: è constant-time in software su qualsiasi CPU, mentre l'AES-GCM di Go è constant-time solo con l'hardware AES-NI, e un tool portabile su file non può presumere che il bersaglio ce l'abbia. Entrambe sono costruzioni AEAD, cioè producono un tag di autenticazione di 16 byte insieme al ciphertext. Se un qualsiasi byte del ciphertext viene alterato, `Open` restituisce un errore, non spazzatura.

**L'header è autenticato come dato associato.** Questa è la parte sottile e importante. L'intero header fino al nonce incluso viene passato all'AEAD come "additional authenticated data". Non è cifrato, ma è coperto dal tag. Quindi un attaccante non può invertire il flag "encrypted", fare il downgrade del cipher o manomettere i parametri Argon2id senza rompere l'autenticazione. Una versione sconosciuta viene rifiutata in modo esplicito anziché indovinata. Un byte manomesso o una passphrase sbagliata falliscono in modo chiuso.

Una questione di design che vale la pena nominare: crypha comprime **prima** di cifrare. Comprimere dopo la cifratura è inutile (il ciphertext non si comprime), ma comprimere prima della cifratura è ciò che ha reso possibili gli attacchi CRIME e BREACH su TLS. Quegli attacchi richiedono un attaccante di rete adattivo che possa iniettare testo in chiaro scelto e osservare la lunghezza compressa-poi-cifrata attraverso molte richieste. Un tool offline one-shot su file non ha un oracolo del genere: non c'è nessun attaccante nel ciclo che inietta tentativi. Quindi comprimere-poi-cifrare qui è sicuro, e significa che il ciphertext, e di conseguenza la capacità richiesta al carrier, è più piccolo.

## Steganalisi: come viene scoperto ogni carrier

Un tool didattico che ti mostrasse solo come nascondere mentirebbe per omissione. Ecco come un difensore rompe ogni carrier. Il threat model di crypha presume che il difensore sia competente, ed è esattamente per questo che il payload viene sempre cifrato per primo.

**L'LSB su immagini e audio viene scoperto con la statistica.** L'embedding LSB lascia un'impronta debole ma misurabile, anche se è invisibile. Gli attacchi classici:
- **L'attacco chi-quadro** (Westfeld e Pfitzmann, 1999) sfrutta il fatto che l'embedding LSB sequenziale porta coppie di valori, come 178 e 179, verso frequenze uguali. Misura quanto ciascuna di queste coppie si avvicina a una divisione 50/50 e segnala le immagini in cui sono sospettosamente bilanciate.
- **L'analisi RS** (Fridrich, Goljan e Du, 2001) inverte gli LSB a gruppi e osserva come risponde una misura di "levigatezza" dell'immagine. Immagini pulite e immagini con embedding rispondono in modo diverso, e il metodo stima la frazione di payload incorporato, non solo la sua presenza.
- **La sample-pair analysis** (Dumitrescu, Wu e Wang, 2002) fa qualcosa di simile con un modello statistico preciso delle coppie di campioni.
- **StegExpose** (Boehm, 2014) li fonde in un unico detector per PNG e BMP.

La difesa pratica contro tutti questi è fare embedding in modo sparso e casuale anziché riempire sequenzialmente ogni bit basso, ed è per questo che crypha usa di default un singolo bit per canale e tratta una modalità a 2 bit come un'estensione documentata, non come default.

**Il testo a larghezza zero viene scoperto in un solo passaggio.** Non c'è alcuna sottigliezza statistica. `unicode.Is(unicode.Cf, r)` sulle rune, o `grep -P '\xe2\x80\x8b'` sui byte, o semplicemente notare che il numero di caratteri visibili è molto inferiore al numero totale di rune, trova ogni byte nascosto. Non c'è alcuna negabilità contro una scansione automatica, ed è di nuovo per questo che il payload è cifrato.

**L'iniezione QR si nasconde a uno scanner, non a un analista.** Uno scanner del telefono non può davvero vedere il payload; il suo decoder Reed-Solomon lo cancella come rumore. Ma un analista che sospetta di un codice QR e ricodifica il contenuto visibile in un QR pulito può fare il diff con l'immagine sospetta, la stessa operazione che esegue il `reveal` di crypha, e recuperare lo schema iniettato. L'occultamento è robusto contro il consumatore previsto e trasparente per un investigatore motivato.

**I trucchi strutturali sui PDF vengono scoperti leggendo il file.** `pdfinfo` e il pannello degli allegati espongono un allegato. `exiftool` scarica le chiavi di metadati personalizzate. `strings file.pdf | tail` rivela i byte accodati dopo `%%EOF`. Nessuna di queste tecniche è statisticamente nascosta; si nascondono a una persona che apre il documento, non a una persona che lo ispeziona.

Il riassunto onesto: i carrier di crypha offrono un occultamento di livello didattico, e la cifratura offre una segretezza di livello production. Contro un ispettore competente l'occultamento spesso fallirà, e quando succede, è la cifratura che continua a proteggere il messaggio. Quella stratificazione è il punto.

## Dove andare ora

[02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) trasforma queste idee in struttura: l'interfaccia `Carrier`, il registry che si auto-registra, la codifica esatta della busta, e la logica di auto-detect che impedisce che un QR-PNG venga scambiato per un'immagine ordinaria.
