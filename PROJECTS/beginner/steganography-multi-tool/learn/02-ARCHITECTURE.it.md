<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# crypha: Architettura

Questo capitolo descrive la forma del codice: come sono separati i pezzi, perché sono separati in quel modo, e come una richiesta scorre da una pressione di tasto fino a un file nascosto. La regola guida è una rigorosa separazione delle responsabilità. I carrier non sanno nulla di crittografia. La busta non sa nulla dei carrier. I frontend non sanno nulla di nessuno dei due. Un unico engine nel mezzo li collega tra loro.

## Un engine, due frontend

crypha si può pilotare in due modi: una CLI cobra scriptabile e un wizard bubbletea guidato. Un design allettante ma sbagliato lascerebbe che ogni frontend faccia l'occultamento per conto proprio. Quella strada duplica la logica importante, e le due copie divergono. crypha invece mette ogni operazione dietro un unico package, `internal/engine`, e rende entrambi i frontend facce sottili sopra di esso.

```
        cobra CLI ───┐
                     ├──>  engine  ──>  carrier registry  ──>  image  audio  qr  text  pdf
     bubbletea TUI ──┘         │
                               └──>  payload envelope (Argon2id · AEAD · flate · CRC32)  ──>  bitio
```

L'engine espone esattamente i verbi che il tool possiede: `Hide`, `Reveal`, `Capacity`, `CapacityAll`, `Catalog`, più due helper di preflight, `Overhead` ed `EnvelopeSize`, che permettono al wizard di mostrare un indicatore di capacità esatto prima di impegnarsi. I comandi cobra in `internal/cli` fanno il parsing dei flag e chiamano queste funzioni. Il wizard bubbletea in `internal/tui` raccoglie gli stessi input attraverso una serie di passi e chiama le stesse funzioni. La TUI non contiene alcuna logica dei carrier; quando le serve la dimensione esatta della busta per il suo indicatore in tempo reale, chiede a `engine.EnvelopeSize`, non mette le mani dentro un carrier. Entrambi i revisori di questo progetto hanno confermato che il wizard è una pura vista. Questo è il design che funziona.

Il vantaggio è che un bug corretto nell'engine è corretto in entrambi i frontend contemporaneamente, e un nuovo carrier compare sia nella CLI sia nel wizard senza che nessuno dei due frontend cambi.

## L'interfaccia Carrier

Ogni canale di occultamento implementa una piccola interfaccia, in `internal/carrier/carrier.go`:

```go
type Carrier interface {
    Format() string
    Hide(cover io.Reader, payload []byte, out io.Writer) error
    Reveal(stego io.Reader) ([]byte, error)
    Capacity(cover io.Reader) (int, error)
    Sniff(stego io.ReadSeeker) bool
}
```

Quattro verbi e un nome. `Hide` legge una copertina e scrive un file stego che trasporta i byte dati. `Reveal` legge un file stego e restituisce i byte. `Capacity` riporta quanti byte una copertina può contenere. `Sniff` è un controllo economico "questo file sembra almeno il mio formato" usato dall'auto-detect. Nota cosa manca: l'interfaccia prende e restituisce `[]byte`, mai un messaggio, mai una passphrase, mai un `Options`. A un carrier viene consegnata la busta finita e lui la memorizza. Non sa se quei byte siano cifrati, compressi o in chiaro, e non gli interessa. Quell'ignoranza è la separazione delle responsabilità resa concreta.

Ogni carrier vive nel proprio package, `image`, `audio`, `qr`, `text`, `pdf`, e dipende solo da `internal/carrier` e da librerie standard o di terze parti. Nessun carrier importa un altro. Puoi leggere, testare o sostituire ciascuno di essi in isolamento.

## Il registry che si auto-registra

I carrier si registrano da soli. Ogni package carrier ha una funzione `init` che chiama `carrier.Register` con un'istanza del proprio tipo:

```go
func init() {
    carrier.Register(qrCarrier{})
}
```

Il registry è una semplice mappa dal nome del formato a `Carrier`, con gli helper `Register`, `Get`, `All`, `Formats` e `Detect`. Il trucco che fa scattare la registrazione è l'aggregatore a import blank `internal/carrier/all/all.go`, che importa ogni package carrier esclusivamente per il suo effetto collaterale:

```go
import (
    _ "github.com/CarterPerez-dev/crypha/internal/carrier/image"
    _ "github.com/CarterPerez-dev/crypha/internal/carrier/audio"
    // ... qr, text, pdf
)
```

L'engine importa `all` una sola volta, anch'esso con un import blank. Quell'unica catena di import esegue l'`init` di ogni carrier, popolando il registry prima che parta `main`. Per aggiungere un sesto carrier scrivi il suo package con un `init` che lo registra, aggiungi una riga ad `all.go`, e l'engine, entrambi i frontend, `formats`, `capacity` e l'auto-detect lo raccolgono tutti senza ulteriori collegamenti. Questo è il pattern a plugin che usano i package `image` e `database/sql` di Go, applicato qui.

`All` restituisce i carrier ordinati per nome, quindi qualsiasi iterazione su di essi, tabelle di capacità, ordine dell'auto-detect, elenchi dei formati, è deterministica. Il determinismo conta per i test e per un output riproducibile.

## L'engine come dispatcher

`internal/engine/engine.go` è la giuntura. Il suo compito su `Hide` è in tre passi:

```go
func Hide(req HideRequest) (HideResult, error) {
    c, err := ResolveCarrier(req.Format, req.Technique)   // pick the carrier
    env, err := payload.Pack(req.Payload, req.Options)     // build the envelope
    err = c.Hide(req.Cover, env, req.Out)                  // store the opaque bytes
    return HideResult{ /* receipt */ }, nil
}
```

`ResolveCarrier` è dove vive l'unico caso speciale. Quattro dei cinque carrier vengono risolti per nome direttamente dal registry. Il PDF è diverso perché ha delle tecniche: `hide --format pdf --technique attachment|metadata|append`. Quindi `ResolveCarrier` costruisce un carrier PDF tramite `pdf.New(technique)` quando il formato è `pdf`, e rifiuta un flag `--technique` su qualsiasi formato non PDF. Questo tiene il concetto di tecnica confinato all'unico carrier che lo possiede, invece di inquinare l'interfaccia per gli altri quattro.

`Reveal` esegue il percorso inverso: individua il carrier ed estrae la busta, controlla se la busta è cifrata, richiede una passphrase se lo è e non ne è stata fornita alcuna, poi `payload.Unpack`. `Capacity` e `CapacityAll` chiamano semplicemente `Capacity` dei carrier e formattano i risultati in righe. L'engine non tocca mai direttamente bit o cipher; orchestra i due sottosistemi che lo fanno.

## La codifica della busta

La busta viene costruita e analizzata in `internal/payload/envelope.go` da `Pack`, `parse` e `Unpack`. Questo è il layout esatto dei byte, e l'overhead è esatto, non approssimato:

```
plaintext    magic(4) ver(1) flags(1) │ len(4) body(N) │ crc32(4)                              14 bytes overhead

encrypted    magic(4) ver(1) flags(1) cipher(1) params(9) salt(16) nonce(12) │ len(4) body(N+16) │ crc32(4)
             └───────────────── AAD: authenticated, not encrypted ───────────┘                        68 bytes overhead
```

- **magic (4)** è una costante fissa, così `reveal` e l'auto-detect possono rifiutare byte casuali a basso costo prima di fare lavoro vero.
- **ver (1)** parte da 1. `parse` rifiuta esplicitamente qualsiasi altra versione anziché tentare di indovinare un layout che non comprende.
- **flags (1)** oggi trasporta due bit: encrypted e compressed, con spazio per crescere.
- Quando è cifrata, l'header prosegue con **cipher (1)** che identifica ChaCha20-Poly1305 o AES-256-GCM, **params (9)** che contiene tempo, memoria e thread di Argon2id, **salt (16)** per la derivazione della chiave, e **nonce (12)** per l'AEAD.
- **len (4)** è la lunghezza big-endian del body che segue.
- **body (N)** è il payload dopo i passaggi opzionali di compressione e cifratura. Quando è cifrato include il tag di autenticazione di 16 byte.
- **crc32 (4)** copre il body. Sul percorso in chiaro è l'unico controllo di integrità ed è ciò che permette all'auto-detect di confermare "questo è un vero payload crypha". Sul percorso cifrato il tag AEAD garantisce già l'integrità, quindi il CRC è un economico controllo di coerenza dopo la decifratura.

Puoi verificare questi numeri da solo. `crypha capacity -i cover.png --format image` su una copertina 640x480 riporta una capacità di busta di 115196, un massimo in chiaro di 115182 (14 in meno) e un massimo cifrato di 115128 (68 in meno). L'overhead di framing non è una stima arrotondata; sono queste due costanti.

L'ordine delle operazioni è il nocciolo della correttezza. In hide: comprimi se richiesto, poi cifra se richiesto, poi incornicia. In reveal: togli la cornice, poi decifra, poi decomprimi. `Pack` scrive l'header per primo proprio per poter passare i byte dell'header ad `aead.Seal` come dato associato prima di accodare il ciphertext. `parse` registra quella stessa slice dell'header come `aad` così che `Unpack` possa passarla ad `aead.Open`. È così che manomettere la versione o la scelta del cipher rompe l'autenticazione.

## I/O a bit e l'infrastruttura dei carrier

I carrier a sostituzione devono leggere e scrivere singoli bit, non byte, e devono concordare sull'ordine dei bit. `internal/bitio` fornisce un `BitReader` e un `BitWriter` MSB-first: il bit alto di un byte viene emesso per primo. Finché hide e reveal usano lo stesso ordine, qualunque scelta coerente funziona; crypha fissa MSB-first ovunque così che un carrier scritto oggi e letto domani siano d'accordo.

`internal/config` contiene ogni costante usata dal tool: i magic byte, i profili dei parametri KDF, il catalogo dei formati con le relative descrizioni, gli identificatori dei cipher. Niente di magico è hard-coded nel punto di chiamata. `internal/report` rende i risultati in due modi, come tabelle allineate leggibili da umani e, con un flag globale `--json`, come JSON leggibile da macchina in cui `reveal` codifica il payload in base64 così che i dati binari sopravvivano a una pipe.

## L'auto-detect e la trappola dell'oscuramento

Il pezzo più interessante dell'architettura è come funziona `reveal` senza `--format`. Ingenuamente faresti lo sniff di ogni carrier e useresti il primo che dice di sì. È esattamente ciò che fa `carrier.Detect` del registry, e non basta, a causa di un problema di oscuramento (shadowing):

**uno stego QR è un PNG.** Se l'auto-detect fa prima lo sniff del carrier image e il carrier image dice "sì, posso leggere gli LSB da questo PNG", estrarrà allegramente 115 KB di rumore e li restituirà, oscurando il carrier QR che possiede davvero i byte. Lo sniff da solo non può distinguere i due, perché il carrier image può effettivamente leggere bit da qualsiasi PNG; solo che non sono un payload valido.

L'engine risolve il problema in `detect` aggiungendo un passo di validazione. Per ogni carrier in ordine deterministico fa lo sniff, poi esegue davvero il reveal, poi valida la busta estratta rispetto al formato del payload:

```go
func detect(stego []byte) (carrier.Carrier, []byte, error) {
    for _, c := range carrier.All() {
        if !c.Sniff(bytes.NewReader(stego)) {
            continue
        }
        env, err := c.Reveal(bytes.NewReader(stego))
        if err != nil {
            continue
        }
        if payload.Validate(env) == nil {   // does it have our magic + a good CRC?
            return c, env, nil
        }
    }
    return nil, nil, ErrUndetected
}
```

Un carrier vince solo se i byte che estrae portano il magic di crypha e superano il controllo CRC. Quando il carrier image legge rumore da un QR-PNG, `payload.Validate` fallisce sul magic, il ciclo passa oltre, e il carrier QR, che estrae un vero payload incorniciato, vince. Il revisore di correttezza di questa fase ha confermato il comportamento empiricamente: su centinaia di stego QR, nessuno è stato oscurato dal carrier image, e su centinaia di file casuali, nessuno ha prodotto un falso positivo. Il passo `Sniff` resta veloce e agnostico rispetto alla busta; è il passo di validazione che rende corretto il rilevamento.

Ecco perché `reveal` può fare l'auto-detect in sicurezza e perché il DEMO può dire, di uno stego QR, "it must not be mistaken for an ordinary image carrier. It is not."

## Il flusso dati completo

Mettendo tutto insieme, ecco un messaggio da `crypha hide` fino a un file e ritorno:

```
hide -m "meet at noon" --format image --encrypt
  │
  ├─ cli parses flags, resolves the passphrase (‑k / env / no-echo prompt)
  ├─ engine.Hide
  │     ├─ ResolveCarrier("image", "")            -> image carrier
  │     ├─ payload.Pack(msg, {encrypt, compress})
  │     │     compress > Argon2id(salt) > ChaCha20-Poly1305.Seal(header as AAD) > frame
  │     └─ image.Hide(cover, envelope, out)
  │           toNRGBA(cover) > write length prefix + envelope bits into RGB LSBs > png.Encode
  └─ receipt: image, 12 bytes payload, 92 bytes envelope, encrypted, compressed

reveal secret.png
  │
  ├─ engine.Reveal
  │     ├─ detect: sniff each carrier, reveal, payload.Validate -> image wins
  │     ├─ payload.IsEncrypted(env) -> true, so demand a passphrase
  │     └─ payload.Unpack(env, passphrase)
  │           unframe > Argon2id(stored salt+params) > AEAD.Open(header as AAD) > decompress
  └─ "meet at noon" to stdout, status to stderr
```

Ogni freccia attraversa esattamente una giuntura, e ogni giuntura è un confine di package che puoi testare da solo. Il prossimo capitolo percorre il codice che vive dentro questi riquadri, con il carrier QR come pezzo forte.

## Dove andare ora

[03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) è la guida al codice: la trappola NRGBA nel carrier image, il seek-back dell'encoder WAV, e l'intera iniezione QR ricostruita da ISO/IEC-18004 con la decodifica Reed-Solomon, funzione per funzione.
