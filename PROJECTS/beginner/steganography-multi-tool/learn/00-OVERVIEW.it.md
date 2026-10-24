<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# crypha: Panoramica

## Che cos'è

Un tool di steganografia multi-formato scritto in Go. Gli passi un messaggio o un file, e lui sigilla quel payload in una busta cifrata con passphrase, compressa e con controllo di integrità, poi nasconde la busta dentro un carrier dall'aspetto ordinario: i bit meno significativi di un'immagine o di un file audio, il margine Reed-Solomon di un codice QR, caratteri a larghezza zero in un blocco di testo, oppure la struttura di un PDF. Punta `reveal` sul risultato e lui rileva automaticamente il carrier e ti restituisce il messaggio. Viene distribuito come un singolo binario statico e senza dipendenze, e si usa sia da una CLI scriptabile sia da un wizard guidato da terminale.

Lo scopo del progetto è capire, costruendolo, lo scambio onesto tra due avversari. La steganografia nasconde il fatto che un messaggio esista; la steganalisi è la caccia statistica che lo trova comunque. crypha implementa cinque canali di occultamento molto diversi, ciascuno con la propria capacità, fragilità e storia di rilevamento, e il percorso `learn/` illustra l'analisi che li rompe uno per uno. Niente qui è uno stub. Ogni carrier fa il round-trip di payload testuali e binari casuali sotto test, e il carrier QR è testato in modo differenziale contro encoder e decoder di riferimento.

## Perché è importante

Crittografia e steganografia rispondono a due domande diverse, e le persone le confondono continuamente. La cifratura rende un messaggio illeggibile. La steganografia lo rende inosservabile. La cifratura da sola annuncia comunque che esiste un segreto, e un blob opaco ad alta entropia è di per sé un segnale, esattamente la cosa che uno scanner di data-loss-prevention, una regola di intrusion-detection o un'ispezione di frontiera sono addestrati a segnalare. La steganografia elimina il segnale facendo sembrare il carrier una foto delle vacanze, un memo vocale, una fattura PDF o un codice QR su un poster.

La tecnica non è accademica. Compare in intrusioni reali da entrambi i lati della barricata.

- **Witchetty, 2022.** Il gruppo di spionaggio ha nascosto una backdoor dentro una bitmap di un vecchio logo di Windows ospitata su un servizio cloud pubblico, così il payload arrivava sul bersaglio con l'aspetto di un normale download di un'immagine anziché di un malware (Symantec). L'immagine veniva ancora visualizzata come un logo. Il codice viaggiava nei bit sottostanti.
- **L'exploit kit Stegano, 2016.** Uno script malevolo era nascosto nel canale alfa di banner pubblicitari PNG serviti a milioni di visitatori su siti mainstream (ESET). Una minuscola modifica per pixel alla trasparenza, invisibile sulla pagina, trasportava la logica di redirect.
- **Invoke-PSImage, 2017.** Un tool open-source che impacchetta un intero script PowerShell nei due bit meno significativi dei canali RGB di un PNG. Ha trasformato la steganografia LSB sulle immagini in una primitiva red-team punta-e-clicca, ed è l'antenato diretto del carrier `image` di crypha.
- **Stegoloader / Gatak, 2015.** Questa famiglia di malware estraeva i propri componenti da immagini scaricate a runtime, tenendo il codice eseguibile evidente lontano dal disco, dove uno scanner lo avrebbe cercato (Dell SecureWorks).

I difensori rispondono con la steganalisi. L'attacco chi-quadro, l'analisi RS e la sample-pair analysis stimano ciascuno quanta parte di un'immagine è stata toccata, senza mai aver bisogno della chiave. crypha esiste per insegnare entrambe le mosse. Prima cifra, così un payload scoperto resta illeggibile, poi nasconde il ciphertext, e infine ti dice onestamente come ogni carrier viene scoperto.

**Scenari reali in cui si applica:**
- **Ricerca sui canali nascosti.** Capire come un payload sopravvive, o non riesce a sopravvivere, a una copia, a una ricodifica, a un passaggio di normalizzazione o a un "salva con nome" di un PDF.
- **Rilevamento blue-team.** Sviluppare l'intuito per individuare embedding LSB, rune a larghezza zero e dati accodati dopo l'EOF prima che un tool di esfiltrazione li usi contro di te.
- **Imparare la crittografia applicata.** La busta è una costruzione AEAD completa e corretta: derivazione della chiave Argon2id, ChaCha20-Poly1305, binding dei dati associati e verifica fail-closed, in circa duecento righe leggibili.

## Cosa imparerai

**Concetti di sicurezza:**
- **Steganografia contro crittografia, e perché le vuoi entrambe.** Occultamento e segretezza sono proprietà indipendenti. crypha le sovrappone: cifra per la segretezza, nascondi per l'occultamento.
- **Cinque canali di occultamento e i loro compromessi.** LSB nei pixel e nei campioni PCM, iniezione di errori Reed-Solomon nei codici QR, Unicode a larghezza zero nel testo e tre tecniche strutturali nei PDF, ciascuno con un diverso profilo di capacità, robustezza e rilevamento.
- **Steganalisi, con onestà.** Come un difensore rileva ogni carrier: gli attacchi chi-quadro e RS sull'LSB, una scansione delle rune in un solo passaggio sul testo a larghezza zero, `strings | tail` su un payload PDF accodato.
- **AEAD fatto correttamente.** Perché l'header è autenticato come dato associato, perché un byte alterato deve far fallire l'apertura anziché decifrare in spazzatura, e perché comprimere-poi-cifrare è sicuro per un tool offline su file mentre sarebbe pericoloso per un protocollo di rete.

**Competenze tecniche:**
- **Un'architettura a plugin in Go.** Una sola interfaccia `Carrier`, un registry che si auto-registra tramite import blank, e un engine che fa dispatch attraverso di esso senza conoscere gli interni di alcun carrier.
- **Reimplementare una specifica.** Il carrier QR ricostruisce il posizionamento dei moduli, l'inversione della maschera, il de-interleaving dei blocchi e la decodifica Reed-Solomon a partire da ISO/IEC 18004, perché la libreria Go disponibile espone solo una bitmap finita.
- **I/O a livello di bit e stranezze dei formati.** Impacchettamento dei bit MSB-first, la conversione obbligatoria in NRGBA che impedisce all'encoder PNG di Go di corrompere i tuoi bit bassi, e il seek-back obbligatorio dell'encoder WAV alla chiusura.
- **Un engine, due frontend.** Un cervello condiviso che una CLI cobra e un wizard bubbletea pilotano entrambi come facce sottili e intercambiabili, così la UI da terminale non contiene alcuna logica dei carrier.

**Tool e tecniche:**
- **`skip2/go-qrcode`** per generare una matrice QR pulita, usata come generatore e come oracolo per i test differenziali, mai a runtime per la decodifica.
- **`golang.org/x/crypto`** per Argon2id e ChaCha20-Poly1305, più `crypto/aes` per l'alternativa AES-256-GCM.
- **`go-audio/wav`** per lettura e scrittura PCM, e **`mewkiz/flac`** per decodificare una copertina FLAC in campioni.
- **`pdfcpu`** per le tecniche lossless basate su allegati e metadati, e il semplice `io` per la tecnica append-after-EOF.

## Prerequisiti

Non ti serve esperienza pregressa di steganografia. Ti serve però una certa dimestichezza con quanto segue.

**Conoscenze richieste:**
- **Basi di Go.** Struct, interfacce, slice ed errori. Se sai leggere un metodo su un valore di interfaccia, sai leggere questo codice.
- **Byte e bit.** Cos'è un byte, cosa significa "il bit meno significativo", e che un `uint32` sono quattro byte big-endian sul filo.
- **Cos'è la cifratura, a grandi linee.** Che una chiave trasforma dati leggibili in dati illeggibili e viceversa. Il capitolo sulla busta in [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) spiega il resto.

**Tool di cui avrai bisogno:**
- **Una toolchain Go**, 1.25 o più recente, solo se compili dai sorgenti. Il binario precompilato non ha bisogno di nulla. Lo script `install.sh` scarica una toolchain per te se manca e hai chiesto di compilare.
- **Nient'altro per eseguirlo.** Niente API key, niente rete, niente servizi. crypha è un tool offline su file, one-shot.

**Utili ma non richiesti:**
- **ImageMagick e ffmpeg**, per sintetizzare copertine usa-e-getta con cui sperimentare. [DEMO.md](../DEMO.md) mostra i comandi esatti.
- Un'occhiata a come sono strutturati i codici QR, se vuoi che il pezzo forte in [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) ti risulti più chiaro.

## Avvio rapido

```bash
# Install (grabs a prebuilt binary, no Go needed):
curl -fsSL https://angelamos.com/crypha/install.sh | bash
# or, with a Go toolchain:
go install github.com/CarterPerez-dev/crypha/cmd/crypha@latest

# Launch the guided wizard:
crypha

# Ask how much a cover can hold:
crypha capacity -i photo.png

# Hide a message in an image, then read it back:
crypha hide -i photo.png -o secret.png --format image -m "meet at noon"
crypha reveal secret.png
```

Output atteso: `capacity` stampa una tabella per carrier con il numero esatto di byte della busta per la copertina che gli hai dato. `hide` stampa una breve ricevuta (file di output, formato, byte del payload, byte della busta, se è stato cifrato e compresso). `reveal` non ha bisogno di `--format`; rileva da solo il carrier, scrive il messaggio recuperato su stdout e stampa su stderr una riga di stato come `revealed 11 bytes via image -> (stdout)`.

Aggiungi una passphrase e `reveal` te la chiede prima di decifrare:

```bash
crypha hide -i photo.png -o secret.png --format image -m "coordinates inside" --encrypt --compress
crypha reveal secret.png     # prompts for the passphrase, then decrypts
```

Una passphrase da qualsiasi fonte, il flag `-k`, la variabile d'ambiente `CRYPHA_PASSPHRASE` o il prompt senza eco, significa sempre che il payload è cifrato. crypha non scrive mai silenziosamente in chiaro quando hai chiesto una chiave.

## Struttura del progetto

```
steganography-multi-tool/
├── cmd/crypha/            # tiny main: it only calls cli.Execute()
├── internal/
│   ├── cli/               # cobra: hide reveal capacity formats version tui, plus secure passphrase input
│   ├── tui/               # the bubbletea wizard, a pure view over the engine
│   ├── engine/            # the shared brain both frontends call
│   ├── carrier/           # the Carrier interface + Register/Get/All/Detect registry
│   │   ├── all/           # blank-import aggregator that registers every carrier
│   │   └── image/ audio/ qr/ text/ pdf/
│   ├── payload/           # the encrypted envelope: Argon2id, AEAD, flate, CRC32, framing
│   ├── bitio/             # MSB-first BitReader / BitWriter
│   ├── config/            # every constant: magic bytes, KDF params, the format catalog
│   └── report/            # human tables and --json rendering
├── learn/                 # this teaching track
├── install.sh             # the one-shot curl-able installer
├── .goreleaser.yaml       # cross-platform release binaries
└── justfile               # every recipe
```

La cosa più importante da capire per prima è la giuntura in `internal/engine/engine.go`. Entrambi i frontend chiamano `engine.Hide`, `engine.Reveal` ed `engine.Capacity`; l'engine avvolge il payload in una busta e fa dispatch verso un carrier attraverso il registry. Tutto ciò che sta in `internal/carrier` esiste per memorizzare e recuperare byte opachi, e tutto ciò che sta in `internal/payload` esiste per costruire quei byte. Nessuno dei due sa dell'altro.

## Prossimi passi

1. **Capisci le idee.** Leggi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per steganografia contro crittografia, ciascuno dei cinque canali di occultamento, la busta AEAD e la steganalisi che rompe ogni carrier, con riferimenti a incidenti reali.
2. **Guarda il design.** Leggi [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) per il design un-engine-due-frontend, il registry dei carrier che si auto-registra, il layout esatto dei byte della busta, e come l'auto-detect evita di scambiare un QR-PNG per un'immagine ordinaria.
3. **Percorri il codice.** Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per seguire un payload dal messaggio ai byte nascosti, con l'iniezione Reed-Solomon nel QR ricostruito da ISO/IEC-18004 come pezzo forte.
4. **Estendilo.** Leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per progetti che vanno da un nuovo carrier a modalità LSB a densità variabile, fino a una maggiore resistenza alla steganalisi.

## Problemi comuni

**`hide --format image` rifiuta la copertina**
```
crypha: image cover must be an 8-bit PNG or a 24-bit BMP
```
Soluzione: il carrier image rifiuta di proposito i PNG a palette e a 16 bit, perché modificare gli indici della palette o i byte alti rovinerebbe visibilmente la copertina. ImageMagick per i gradienti sintetici usa di default un PNG a 16 bit; forza il truecolor a 8 bit con `-depth 8 PNG24:cover.png`. Una normale foto salvata come PNG o un BMP a 24 bit funziona e basta.

**`reveal` dice di non aver trovato nulla**
```
crypha: no crypha payload detected; pass a format to force a carrier
```
Soluzione: o il file non contiene un payload crypha, oppure il carrier è stato distrutto. I carrier LSB sopravvivono a una copia byte per byte ma non a una ricodifica; se hai risalvato il PNG stego come JPEG, o il WAV come MP3, il payload è perso. Se conosci il formato, passa `--format` per saltare il rilevamento e ottenere un errore specifico.

**`capacity` su una copertina QR dice "does not fit" per la cifratura**
```
qr   52   38   does not fit
```
Soluzione: è corretto, non è un bug. Un codice QR contiene solo qualche decina di byte nel suo budget di errori correggibili, e una busta cifrata aggiunge 68 byte di overhead. QR è per design un carrier solo in chiaro, e `capacity` te lo dice fin da subito.

## Progetti correlati

Se lo hai trovato interessante, dai un'occhiata a:
- **metadata-scrubber-tool**: l'istinto opposto, togliere i metadati nascosti da un file invece di aggiungerne.
- **binary-analysis-tool**: analisi statica della struttura di un binario, la stessa capacità di "leggere con precisione un formato di file" applicata agli eseguibili.
- **nadezhda** (security-news-scraper): la stessa forma a singolo-binario-statico, un-engine-due-frontend, applicata all'intelligence sulle notizie di sicurezza.
