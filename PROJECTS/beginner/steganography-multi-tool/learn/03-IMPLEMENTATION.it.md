<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# crypha: Implementazione

Questo capitolo percorre il codice. Parte dalla busta condivisa da tutti i carrier, passa per i tre carrier più semplici per costruire l'intuito, e poi dedica la maggior parte della sua lunghezza al pezzo forte: il carrier QR, che reimplementa la geometria dei moduli QR e la decodifica Reed-Solomon a partire da ISO/IEC 18004, perché nessuna libreria Go espone ciò di cui il canale nascosto ha bisogno. Vengono forniti i nomi delle funzioni così puoi trovare ogni pezzo; non ci sono riferimenti a numeri di riga, perché quelli diventano obsoleti nel momento in cui il file cambia.

## La busta: `payload.Pack` e `payload.Unpack`

Tutto ciò che viene nascosto da qualsiasi carrier viene prima costruito da `Pack` in `internal/payload/envelope.go`. `Pack` prende il payload grezzo e una struct `Options` e restituisce i byte della busta incorniciata. Il corpo della funzione è l'ordine delle operazioni del capitolo sui concetti, reso letterale:

```go
if opts.Compress {
    body, _ = compress(body)          // flate, sets flagCompressed
    flags |= flagCompressed
}
header.Write(magic[:])
header.WriteByte(currentVersion)
if len(opts.Passphrase) > 0 {
    flags |= flagEncrypted
    header.WriteByte(flags)
    // salt, params, cipher id, nonce all written into the header ...
    body = aead.Seal(nil, nonce, body, header.Bytes())   // header IS the AAD
} else {
    header.WriteByte(flags)
}
```

La singola riga più importante è `aead.Seal(nil, nonce, body, header.Bytes())`. Il quarto argomento è il dato associato. Passando l'intero header, magic, versione, flag, id del cipher, parametri Argon2id, salt e nonce, `Pack` lega tutti quei metadati al tag di autenticazione del ciphertext. Niente nell'header è segreto, ma niente nell'header può essere cambiato senza invalidare il tag.

`parse` rilegge il layout e, cosa cruciale, registra l'esatta slice dell'header che ha consumato come `p.aad`. `Unpack` poi chiama `aead.Open(nil, p.nonce, body, p.aad)` con quella stessa slice. Se un attaccante ha invertito il flag encrypted, fatto il downgrade del cipher o modificato i parametri KDF, l'`aad` non corrisponde più a ciò che era stato sigillato e `Open` restituisce un errore. crypha lo mappa su `ErrDecrypt`, "decryption failed (wrong passphrase or tampered data)," ed esce con codice diverso da zero. Non restituisce mai spazzatura parzialmente decifrata.

I parametri Argon2id viaggiano nella busta proprio perché `reveal` non abbia bisogno di flag. `parse` legge `params` in una struct `kdfParams`, `Unpack` chiama `deriveKey(passphrase, p.salt, p.params)`, e ne esce esattamente la stessa chiave. `parse` valida anche i parametri prima dell'uso (`p.params.valid()`), perché `argon2.IDKey` va in panic invece di restituire un errore con un costo temporale zero o un rapporto memoria/thread impossibile, e una busta ostile non deve poter innescare quel panic.

## Il carrier image: la trappola NRGBA

`internal/carrier/image/image.go` è breve, e quasi tutta la sua sottigliezza sta in un helper, `toNRGBA`. La versione ingenua di un carrier LSB per immagini decodifica il PNG, modifica i bit bassi e ricodifica. Produce un output corrotto, e il motivo è un vero tranello di Go.

Un PNG truecolor standard a 24 bit viene decodificato in `*image.RGBA`, che memorizza il colore con alfa **premoltiplicato**. Quando passi un `*image.RGBA` a `png.Encode`, l'encoder esegue un passaggio di de-moltiplicazione dell'alfa che fa calcoli aritmetici sui valori dei tuoi pixel, e quei calcoli riscrivono esattamente i bit bassi che avevi appena impostato con cura. Il tuo payload viene distrutto in uscita. Un PNG di color-type 6 viene decodificato in `*image.NRGBA` (non premoltiplicato), e per quel tipo `png.Encode` copia direttamente i byte dei pixel senza alcun calcolo sul colore, quindi i bit bassi sopravvivono.

La correzione è convertire sempre la copertina in `*image.NRGBA` prima di toccarla:

```go
func toNRGBA(src stdimage.Image) *stdimage.NRGBA {
    if n, ok := src.(*stdimage.NRGBA); ok && n.Rect.Min == (stdimage.Point{}) {
        return n
    }
    dst := stdimage.NewNRGBA(stdimage.Rect(0, 0, b.Dx(), b.Dy()))
    draw.Draw(dst, dst.Bounds(), src, b.Min, draw.Src)
    return dst
}
```

Per una sorgente opaca, con ogni pixel ad alfa 255, la conversione da RGBA a NRGBA è esattamente lossless, quindi costa solo una copia. `Hide` poi fa scorrere i bit del payload nel bit basso di ogni canale con `pixOffset`, ed è lì che il canale alfa viene protetto:

```go
func pixOffset(slot int) int {
    return bytesPerPixel*(slot/channelsPerPixel) + (slot % channelsPerPixel)
}
```

Il buffer dei pixel NRGBA è disposto come `[R G B A R G B A ...]`, quattro byte per pixel. Mappando lo slot `n` sul pixel `n/3` e sul canale `n%3`, `pixOffset` visita solo R, G e B e salta ogni A. Il byte alfa non viene mai scritto, perché un pixel ad alfa 254 invece di 255 appare come un leggero alone su uno sfondo composito ed è un indizio diretto per la steganalisi.

Prima di tutto questo, `rejectLossy` rifiuta le copertine che non possono fare il round-trip: `*image.Paletted` (modificare un indice di palette salta a un colore completamente diverso) e i tipi a 16 bit come `*image.RGBA64` (otto byte per pixel, un layout diverso). crypha preferisce rifiutare una copertina piuttosto che rovinarla silenziosamente, ed è per questo che il DEMO forza `-depth 8 PNG24:` per il suo gradiente sintetico. Un helper `frame` antepone al payload una lunghezza `uint32` big-endian così che `Reveal` legga prima la lunghezza, poi esattamente quel numero di byte di payload, e si fermi.

## Il carrier di testo a larghezza zero: framing a lunghezza esatta

`internal/carrier/text/text.go` mappa i bit sulle due rune invisibili e viceversa. `Hide` costruisce un frame `[magic][length][payload]`, lo percorre bit per bit con il `bitio.NewReader` condiviso, accoda `zeroRune` (U+200B) per uno 0 e `oneRune` (U+2060) per un 1, e scrive la copertina visibile seguita dalla sequenza invisibile. La lettura è speculare: `extractBits` scorre le rune e raccoglie uno 0 o un 1 per ciascuno dei due caratteri carrier, ignorando tutto il resto.

Il dettaglio difensivo interessante è `parseFrame`. Poiché questo carrier può essere impilato (un testo stego può essere a sua volta usato come copertina), e poiché uno scanner percorre ogni possibile offset del magic in `findFrame`, il parser non deve accettare un frame la cui lunghezza dichiarata non concordi con i bit effettivamente presenti:

```go
remaining := len(bits) - payloadStart
if uint64(length)*bitsPerByte != uint64(remaining) {
    return nil, false        // declared length must consume the run exactly
}
```

Richiedere che il payload consumi la sequenza fino alla sua fine esatta, anziché semplicemente starci dentro, è ciò che impedisce a una corrispondenza spuria del magic più in alto di oscurare il frame reale. Questa esattezza è stata una segnalazione corretta durante l'audit del carrier, ed è la differenza tra un parser robusto e uno che occasionalmente legge i byte sbagliati.

## Il carrier audio: WAV e il Close obbligatorio

`internal/carrier/audio` decodifica campioni PCM a 16 bit con `go-audio/wav`, imposta il bit basso di ogni campione nello stesso modo in cui il carrier image imposta l'LSB di un canale, e ricodifica. L'unico requisito non ovvio è che il `Close` dell'encoder WAV è obbligatorio, non una pulizia opzionale: il WAV scrive nel suo header un campo con la dimensione del data-chunk che è noto solo una volta scritti tutti i campioni, quindi `Close` torna all'inizio e lo corregge. crypha scrive su un `writeseeker` in memoria così che l'encoder possa eseguire quel seek anche quando la destinazione finale è un semplice `io.Writer` che non supporta il seek. Una copertina FLAC viene decodificata negli stessi campioni PCM con `mewkiz/flac` e poi riemessa come WAV; l'output FLAC nativo è rimandato perché l'unico encoder FLAC in Go emette frame che i parser rigorosi rifiutano.

## Il pezzo forte: iniezione di errori Reed-Solomon nei QR

Ora l'evento principale. Il carrier QR nasconde un payload sotto forma di errori deliberati e correggibili nei codeword di dati di un codice QR, così un normale scanner del telefono legge la copertina visibile e ripara silenziosamente gli errori iniettati, mentre crypha li rilegge come il segreto. Questo è il carrier che ha richiesto più impegno nel progetto, perché l'ecosistema Go non ti dà quasi nulla con cui lavorare.

### Il problema: `skip2` ti consegna solo una bitmap finita

`github.com/skip2/go-qrcode` può generare un codice QR e, tramite `NewWithForcedVersion`, fissarne la versione e il livello di correzione d'errore. Ma la sua superficie pubblica espone solo `Bitmap() [][]bool`: la matrice finale, mascherata e completamente assemblata con i pattern funzionali già incorporati. I codeword, la struttura dei blocchi e la maschera dati scelta sono tutti non esportati. Per iniettare errori in specifici codeword di dati e rileggerli in seguito, crypha deve smontare quella bitmap finita e ricostruire tutto ciò che la libreria ha nascosto, partendo dalla specifica. Quella ricostruzione vive tra `qr.go`, `matrix.go`, `blocks.go`, `gf.go` e `rs.go`. crypha fissa il livello di correzione d'errore a H e supporta le versioni da 1 a 10.

Ecco l'intera pipeline di `Hide`, poi ogni fase.

```go
code, version, _ := selectVersion(string(coverText), len(payload))  // 1. clean QR
clean, _ := matrixFromBitmap(code.Bitmap(), version)                // 2. bitmap -> matrix
maskID, level, ok := parseFormat(clean)                             // 3. read format info
isFunc := functionModules(version)                                  // 4. function map
order := placementOrder(version, isFunc)                            // 5. zigzag order
serial := readSerial(clean, order, maskID, spec.totalCodewords())   //    unmask + read codewords
dataBlocks, ecBlocks, _ := spec.deinterleave(serial)               // 6. split into blocks
framed := frame(payload)
injectFramed(dataBlocks, spec, framed)                              // 7. inject errors
stego := clean.clone()
writeSerial(stego, order, maskID, spec.interleave(dataBlocks, ecBlocks))  // 8. re-lay + write
renderPNG(stego, out)                                              //    encode PNG
```

### Fase 1: una matrice di copertura pulita

`selectVersion` chiede a `skip2` la versione supportata più piccola (da 1 a 10) la cui capacità a livello H contenga il payload, e in cui il testo di copertura ci stia. Disabilita il bordo della quiet zone della libreria così che sia crypha a controllare la geometria. `matrixFromBitmap` copia il `[][]bool` restituito nel tipo `matrix` di crypha, controllando che le dimensioni corrispondano al `symbolSize` atteso per la versione (21 moduli per la versione 1, crescendo di 4 per versione).

### Fase 3: leggere le informazioni di formato

Le informazioni di formato sono un campo a 15 bit memorizzato due volte nel codice, che codifica l'id della maschera e il livello di correzione d'errore sotto un codice a correzione d'errore BCH. `parseFormat` legge le 15 celle di modulo elencate in `formatBitCells`, poi recupera il valore inteso come fa un vero decoder, cioè per codeword più vicino:

```go
for d := 0; d < formatCodeCount; d++ {
    dist := bits.OnesCount(uint(stored ^ formatCode(d)))
    if dist < bestDist { bestDist, bestData = dist, d }
}
if bestData < 0 || bestDist > formatMaxDistance { return 0, 0, false }
```

`formatCode` ricalcola la codifica BCH di ciascuno dei 32 possibili valori di formato (inclusa la maschera obbligatoria `0x5412`), e il ciclo sceglie quello più vicino in distanza di Hamming a ciò che è stato letto, rifiutando qualsiasi cosa sia oltre il raggio di correzione di 3 bit del codice. crypha poi pretende che il livello sia H; qualsiasi altro livello significa che questo non è un QR di crypha.

### Fase 4: la mappa funzionale

Il payload può viaggiare solo nei moduli dati, mai nei pattern fissi che uno scanner usa per orientarsi. `functionModules` costruisce una griglia booleana che marca ogni modulo non-dati per la versione data: i tre finder pattern e i loro separatori, le due linee di timing, gli alignment pattern (cercati per versione in `alignmentCenters`, saltando quelli che collidono con un blocco finder) e, dalla versione 7 in su, i due blocchi di informazioni di versione. Questa maschera è ciò che il percorso di posizionamento consulta per sapere quali celle saltare.

Una sottigliezza che vale la pena sottolineare: il blocco finder-più-separatore è largo 8 moduli, non i 7 del solo finder pattern. Sbagliare quel bordo di uno sposta l'intero posizionamento dei codeword e produce spazzatura. Esattamente quell'errore off-by-one, i blocchi finder in alto a destra e in basso a sinistra marcati larghi 8 anziché 9 o 7, è stato uno dei due bug reali scoperti dai test differenziali durante lo sviluppo. Il test che lo scopre è descritto alla fine di questa sezione.

### Fase 5: il posizionamento a zigzag e l'inversione della maschera

I moduli QR vengono riempiti in ordine serpentino: due colonne alla volta, partendo dal basso a destra, serpeggiando in su e poi in giù, saltando i moduli funzionali e la linea di timing verticale. `placementOrder` riproduce esattamente quel percorso, accodando la coordinata di ciascun modulo dati a una slice ordinata finché non ha visitato ogni modulo dati della versione:

```go
x := size - 2; y := size - 1; dirUp := true; xOffset := 1
for i := 0; i < count; i++ {
    order = append(order, point{x: x + xOffset, y: y})
    // toggle between the two columns, step up or down, bounce at the edges,
    // hop over the timing column, and skip any function module
}
```

Con quell'ordinamento in mano, `readSerial` lo percorre un codeword (8 moduli) alla volta e legge i bit, ma con un'inversione della maschera incorporata. Un codice QR applica in XOR una maschera simile a una scacchiera sulla sua regione dati, così che nessuna grande area vuota confonda uno scanner. Per recuperare i veri bit dei codeword devi togliere di nuovo la stessa maschera con uno XOR:

```go
v := m.grid[p.y][p.x]
if maskBit(maskID, p.y, p.x) {
    v = !v            // undo the data mask
}
```

`maskBit` implementa tutte e otto le formule di maschera ISO. Saltare questo passo non è un errore sottile; trasforma ogni codeword in rumore, e una prima versione del carrier che lo ometteva produceva un diff dei moduli privo di significato. `writeSerial` è l'esatto inverso, e riapplica la maschera mentre ridispone i codeword.

### Fase 6: de-interleaving in blocchi

Un codice QR non memorizza i suoi blocchi uno dopo l'altro; ne intercala i codeword così che una macchia fisica danneggi un po' di ciascun blocco anziché distruggerne uno per intero. `blocks.go` contiene le tabelle dei blocchi per versione a livello H (`versionTable`), e `deinterleave` inverte l'interleaving per recuperare i singoli blocchi di dati e di correzione d'errore, usando `blockLayout` per conoscere le lunghezze di dati ed EC di ciascun blocco. `interleave` è il suo inverso per il percorso di scrittura. Queste tabelle sono trascritte dalle tabelle ISO sulla struttura dei blocchi; sono la parte meno affascinante e più soggetta a errori dell'intero carrier, ed è per questo che il test differenziale contro la bitmap pulita di `skip2` conta così tanto.

### Fase 7: iniettare il payload

Questo è l'occultamento vero e proprio, e sono nove righe. `injectFramed` distribuisce i byte del payload incorniciato sui blocchi in round-robin, applicando in XOR ogni byte di payload a un codeword di dati:

```go
for t := 0; t < len(framed); t++ {
    block := t % nb          // spread across blocks round-robin
    slot := t / nb           // next free slot within the block
    if slot >= inject || slot >= len(dataBlocks[block]) {
        return ErrPayloadTooLarge
    }
    dataBlocks[block][slot] ^= framed[t]
}
```

Il budget è `inject := spec.injectPerBlock()`, che è `correctable() / 2`, cioè `(ecPerBlock / 2) / 2`. Il primo dimezzamento è il limite stesso di Reed-Solomon, `t = floor((n - k) / 2)` errori correggibili per blocco. Il secondo dimezzamento è un margine di sicurezza deliberato (`injectionSafetyRatio = 2`): crypha inietta solo metà degli errori che il codice può correggere, così il contenuto visibile viene comunque decodificato con un margine comodo su uno scanner reale. È esattamente per questo che la capacità è onesta e piccola. Un codice di versione 10 ha 8 blocchi, 28 codeword EC per blocco, quindi `correctable` è 14 e `injectPerBlock` è 7; `8 x 7` fa 56 codeword, meno il prefisso di frame di 4 byte, dà la capacità di busta di **52** byte che vedi in `capacity`.

Poiché un codeword di dati corrotto differisce da quello pulito, e quello pulito è recuperabile tramite Reed-Solomon, lo XOR è il canale perfetto: `stego = clean XOR payload`, quindi `payload = clean XOR stego`, e il ricevente recupera `clean` gratuitamente decodificando.

### Fase 8: riscriverlo in output

`interleave` riserializza i blocchi di dati ora corrotti insieme ai blocchi EC intatti, `writeSerial` li ridispone in un clone della matrice pulita (riapplicando la maschera), e `renderPNG` scala ogni modulo in un blocco di 8x8 pixel, aggiunge una quiet zone di 4 moduli e codifica un PNG in scala di grigi. Il risultato viene scansionato come la copertina su qualsiasi telefono.

### Reveal: correggere gli errori per leggerli

`Reveal` inverte la geometria, esattamente la stessa catena da `parseFormat` a `deinterleave`, e poi `extractFramed` fa la parte crittograficamente appagante: recupera i dati puliti di ogni blocco con la decodifica Reed-Solomon, e ne fa il diff in XOR con ciò che è stato ricevuto.

```go
recv := append(append([]byte(nil), dataBlocks[b]...), ecBlocks[b]...)
corrected, err := rsDecode(recv, ec)          // repair the injected errors
clean[b] = corrected[:len(dataBlocks[b])]
// ... then framed[t] = clean[block][slot] ^ dataBlocks[block][slot]
```

`rsDecode`, in `rs.go`, è un decoder Reed-Solomon sistematico su GF(2^8) scritto da zero. Vale la pena capirlo perché è la stessa macchina che esegue anche lo scanner di un telefono:

1. **Sindromi** (`rsSyndromes`). Valuta il blocco ricevuto come polinomio in potenze successive del generatore del campo. Se ogni sindrome è zero, non ci sono errori e il blocco viene restituito così com'è.
2. **Localizzatore di errori** (`rsErrorLocator`). L'algoritmo di Berlekamp-Massey costruisce il polinomio localizzatore degli errori a partire dalle sindromi. Il suo grado è il numero di errori; se supera il budget correggibile, il blocco viene dichiarato non correggibile.
3. **Posizioni degli errori** (`rsErrorPositions`). Una ricerca di Chien valuta il localizzatore in ogni posizione per trovarne le radici, che puntano ai codeword corrotti. Se il numero di radici non corrisponde al grado del localizzatore, fallisce in modo pulito.
4. **Ampiezze degli errori** (`rsSolveMagnitudes`). Con le posizioni note, crypha risolve un piccolo sistema lineare di Vandermonde tramite eliminazione gaussiana sul campo per trovare il valore di ogni errore, poi verifica che la soluzione riproduca le sindromi prima di fidarsene.

L'aritmetica di campo sottostante, in `gf.go`, è il classico trucco delle tabelle log/exp: `init` percorre le potenze del generatore sotto il polinomio primitivo QR `0x11D` per riempire `gfExp` e `gfLog`, così la moltiplicazione diventa una somma di logaritmi e l'inversione diventa una sottrazione. Questo è lo stesso GF(2^8) usato da ogni implementazione QR.

La ricerca di Chien nasconde il secondo dei due bug reali scoperti dai test differenziali: la mappatura da una radice del polinomio localizzatore all'indice di un codeword dipende da una convenzione sulle posizioni, e invertire quella convenzione trova il numero giusto di errori nei posti sbagliati. La correzione è il termine `(len(recv)-1-p)` che vedi sia in `rsErrorPositions` sia nella preparazione delle ampiezze; allinea le posizioni delle radici del localizzatore con l'ordinamento dei codeword usato dall'encoder.

### I due oracoli che lo mantengono onesto

crypha ha reimplementato una specifica, e la reimplementazione di una specifica è esattamente il luogo in cui proliferano bug silenziosi e dall'aspetto plausibile. La difesa è il test differenziale contro due implementazioni di riferimento indipendenti, e nessuna delle due è usata a runtime:

- **Oracolo di generazione: `skip2/go-qrcode`.** Per una data copertina e versione, la ricostruzione della matrice pulita fatta da crypha deve corrispondere bit per bit al `Bitmap()` di `skip2`. Questo valida posizionamento, mascheramento e gestione dei blocchi tutto in una volta. L'off-by-one del blocco finder largo 8 ha fatto fallire questo test in modo eclatante.
- **Oracolo di scansione: `makiuchi-d/gozxing`.** Dopo l'iniezione, il PNG stego deve ancora decodificarsi, tramite un decoder QR completamente indipendente, nel contenuto di copertura originale. Questo dimostra che l'iniezione è rimasta entro il budget di correzione e che il codice si auto-ripara davvero. Se crypha iniettasse anche un solo errore di troppo, `gozxing` non riuscirebbe a recuperare la copertina e il test lo scoprirebbe.

Entrambi gli oracoli sono dipendenze solo di test. A runtime crypha fa la propria introspezione della generazione e la propria decodifica Reed-Solomon; non chiama mai `gozxing` per leggere un codice. Quell'indipendenza è il punto centrale: un bug nel decoder di crypha non può nascondersi dietro lo stesso bug nel suo encoder, perché l'encoder è `skip2` e il controllo di scansione è `gozxing`.

## Dove andare ora

[04-CHALLENGES.it.md](./04-CHALLENGES.it.md) lascia il lettore libero di sperimentare: aggiungi un sesto carrier, costruisci una modalità LSB a densità variabile, implementa un detector chi-quadro contro l'output stesso di crypha, oppure spingi il carrier QR oltre la versione 10.
