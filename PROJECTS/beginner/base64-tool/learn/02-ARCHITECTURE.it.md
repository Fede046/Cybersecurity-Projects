# Architettura

## Filosofia di design

Ogni modulo di questo progetto ha un solo compito. `encoders.py` trasforma i dati. `detector.py` assegna punteggi ai formati. `peeler.py` orchestra il decoding ricorsivo. `formatter.py` produce l'output. `cli.py` collega l'input dell'utente alle funzioni. Nessun modulo si intromette nelle responsabilità di un altro modulo.

Questo non è over-engineering per un progetto piccolo. È il modo in cui eviti che un progetto piccolo diventi ingestibile. Quando devi aggiungere un nuovo formato di encoding, tocchi `encoders.py` e `detector.py`. Tutto qui. La CLI, il formatter e il peeler non cambiano.

## Grafo delle dipendenze tra moduli

```
cli.py
├── constants.py     (EncodingFormat, ExitCode, PEEL_MAX_DEPTH)
├── encoders.py      (encode, decode, encode_url, decode_url)
├── detector.py      (detect_encoding)
├── peeler.py        (peel)
├── formatter.py     (print_encoded, print_decoded, print_detection, print_peel_result, print_chain_result)
└── utils.py         (resolve_input_bytes, resolve_input_text)

peeler.py
├── constants.py     (CONFIDENCE_THRESHOLD, PEEL_MAX_DEPTH, EncodingFormat)
├── detector.py      (detect_best)
└── utils.py         (safe_bytes_preview, truncate)

detector.py
├── constants.py     (charsets, thresholds, EncodingFormat)
├── encoders.py      (try_decode)
└── utils.py         (is_printable_text)

formatter.py
├── constants.py     (EncodingFormat, PREVIEW_LENGTH)
├── detector.py      (DetectionResult — type only)
├── peeler.py        (PeelResult — type only)
└── utils.py         (safe_bytes_preview)

encoders.py
└── constants.py     (EncodingFormat)

utils.py
└── (no internal deps)

constants.py
└── (no internal deps)
```

Le frecce delle dipendenze puntano sempre verso il basso. `constants.py` e `utils.py` stanno in fondo, con zero dipendenze interne. `cli.py` sta in cima e importa da tutto. Niente nel mezzo punta verso l'alto. Questo è un grafo aciclico diretto (DAG), e se mai crei un import circolare, Python te lo dirà immediatamente.

## Flusso dei dati

### Comando encode

```
User Input (str or file or stdin)
    │
    ▼
resolve_input_bytes()          ← utils.py:12
    │  Converts any input source to raw bytes
    ▼
encode(raw, fmt)               ← encoders.py:88
    │  Dispatches via ENCODER_REGISTRY to format-specific function
    ▼
encode_base64(data) (or other) ← encoders.py:22
    │  Returns encoded string
    ▼
print_encoded(result, fmt)     ← formatter.py:31
    │  Rich panel if terminal, raw stdout if piped
    ▼
Output
```

### Comando decode

```
User Input (str or file or stdin)
    │
    ▼
resolve_input_text()           ← utils.py:29
    │  Converts any input source to stripped text
    ▼
decode(text, fmt)              ← encoders.py:93
    │  Dispatches via ENCODER_REGISTRY to format-specific function
    ▼
decode_base64(data) (or other) ← encoders.py:26
    │  Returns decoded bytes
    ▼
print_decoded(result)          ← formatter.py:44
    │  Safe preview (UTF-8 if possible, hex fallback)
    ▼
Output
```

### Comando detect

```
User Input (str)
    │
    ▼
detect_encoding(text)          ← detector.py:206
    │
    ├──► _score_base64(text)   ← detector.py:31
    ├──► _score_base64url(text)← detector.py:70
    ├──► _score_base32(text)   ← detector.py:97
    ├──► _score_hex(text)      ← detector.py:126
    └──► _score_url(text)      ← detector.py:174
    │
    │  Each scorer returns 0.0–1.0
    │  Results filtered by CONFIDENCE_THRESHOLD (0.6)
    │  Sorted by confidence descending
    ▼
print_detection(results)       ← formatter.py:58
    │  Rich table: format, confidence %, decoded preview
    ▼
Output
```

### Comando peel (la funzionalità di punta)

```
User Input (str)
    │
    ▼
peel(text, max_depth=20)       ← peeler.py:33
    │
    ├──► LOOP (up to max_depth iterations):
    │    │
    │    ├── detect_best(current_text)  ← detector.py:226
    │    │   Returns highest-confidence detection
    │    │
    │    ├── Break if: no detection, below threshold, decode fails
    │    │
    │    ├── Record PeelLayer (depth, format, confidence, previews)
    │    │
    │    └── decoded_bytes → current_text for next iteration
    │        (break if bytes aren't valid UTF-8)
    │
    ▼
PeelResult(layers, final_output, success)
    │
    ▼
print_peel_result(result)      ← formatter.py:94
    │  Layer-by-layer display + final output panel
    ▼
Output
```

### Comando chain

```
User Input (str) + --steps "base64,hex,url"
    │
    ▼
resolve_input_bytes()          ← utils.py:12
    │
    ▼
_parse_chain_steps("base64,hex,url")  ← cli.py:264
    │  Validates each format name against EncodingFormat enum
    │  Returns [BASE64, HEX, URL]
    ▼
LOOP over formats:
    │
    ├── encode(current_bytes, fmt)     ← encoders.py:88
    ├── Record (fmt, encoded_string)
    └── encoded_string → bytes for next iteration
    │
    ▼
print_chain_result(steps, final)       ← formatter.py:130
    │  Step-by-step display + final panel
    ▼
Output
```

## Pattern chiave

### Registry pattern (encoders.py:73–85)

Invece di una catena di `if fmt == "base64": ... elif fmt == "base64url": ...`, ogni coppia di encoder e decoder è registrata in un dizionario:

```
ENCODER_REGISTRY: dict[EncodingFormat, tuple[EncoderFn, DecoderFn]]
```

Aggiungere un nuovo formato significa aggiungere una voce al registry e scrivere le due funzioni. Le funzioni di dispatch `encode()` e `decode()` non cambiano mai. Questo è il principio open-closed: aperto all'estensione, chiuso alla modifica.

### Frozen dataclass (detector.py:24, peeler.py:17, peeler.py:26)

Tutti i tipi di risultato usano `@dataclass(frozen=True, slots=True)`. Frozen significa che i campi non possono essere modificati dopo la creazione. Slots significa nessun `__dict__` per istanza, il che usa meno memoria ed è leggermente più veloce. Per dati che scorrono attraverso una pipeline e non dovrebbero mai essere modificati, le frozen dataclass sono lo strumento giusto.

### Output adatto alle pipeline (formatter.py:22–28)

Il tool rileva se stdout è un terminale o una pipe. Quando è in pipe (`echo "data" | b64tool decode | other_tool`), scrive testo grezzo su stdout senza alcuna formattazione Rich. Quando è interattivo, mostra pannelli, tabelle e colori. Questo avviene tramite `is_piped()`, che controlla `sys.stdout.isatty()`.

L'output Rich va su stderr (`Console(stderr=True)` in `formatter.py:19`), così i messaggi diagnostici non contaminano mai i dati in pipe. Questa è una convenzione Unix standard che molti tool CLI sbagliano.

### Architettura degli scorer (detector.py:195–203)

Il rilevamento usa lo stesso registry pattern dell'encoding. Ogni formato ha una funzione scorer con la firma `Callable[[str], float]`. Il dizionario `_SCORERS` mappa `EncodingFormat` sul relativo scorer. Questo significa che aggiungere il rilevamento per un nuovo formato richiede di scrivere una funzione scorer e aggiungere una voce al dizionario.

Ogni scorer segue la stessa struttura:
1. Rifiuto rapido (controllo del charset, controllo della lunghezza)
2. Accumulo di un punteggio di confidenza basato su segnali strutturali
3. Tentativo di decoding effettivo
4. Bonus se l'output decodificato è testo stampabile
5. Restituzione del valore limitato a [0.0, 1.0]

### Type alias con PEP 695 (encoders.py:18–19)

```
type EncoderFn = Callable[[bytes], str]
type DecoderFn = Callable[[str], bytes]
```

Le istruzioni `type` di Python 3.12+ (PEP 695) sostituiscono `TypeAlias` di `typing`. Sono valutate in modo lazy e sono più leggibili. Questi alias documentano il contratto: gli encoder prendono byte e restituiscono stringhe, i decoder prendono stringhe e restituiscono byte.

## Strategia di gestione degli errori

Gli errori sono gestiti su due livelli:

**Livello di modulo**: funzioni come `try_decode()` (`encoders.py:98`) catturano le eccezioni specifiche dell'encoding e restituiscono `None`. Il detector e il peeler usano questo meccanismo per gestire con eleganza i fallimenti di decoding senza andare in crash.

**Livello CLI**: ogni comando (`cli.py`) racchiude il proprio corpo in un try/except. `typer.BadParameter` viene rilanciata (Typer le formatta in modo gradevole). Tutte le altre eccezioni producono un messaggio `[red]Error:[/red]` e l'exit code 1. Questo impedisce che gli stack trace arrivino agli utenti finali.

I moduli intermedi (detector, peeler) non catturano mai eccezioni da soli. Chiamano `try_decode()` e controllano se il risultato è `None`. Questo mantiene la gestione degli errori ai confini, invece di spargerla nella business logic.

## Perché non una classe?

Nessuno dei moduli principali usa classi per il comportamento (solo per i dati: `EncodingFormat`, `DetectionResult`, `PeelLayer`, `PeelResult`). Le funzioni encoder sono funzioni pure. Gli scorer sono funzioni pure. Il peeler è una funzione. Non c'è uno stato mutabile condiviso da incapsulare, quindi non c'è motivo di usare una classe.

Una classe `Encoder` con metodi `encode()` e `decode()` aggiungerebbe un livello di indirezione senza aggiungere valore. Il dizionario del registry ottiene lo stesso polimorfismo con meno cerimonie. Questo è Python idiomatico: usa le classi per i dati e le funzioni per il comportamento, a meno che tu non abbia uno stato da gestire.
