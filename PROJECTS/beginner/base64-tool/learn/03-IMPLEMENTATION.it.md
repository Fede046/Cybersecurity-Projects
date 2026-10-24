# Implementazione

## constants.py — Le fondamenta

Tutto ciò da cui dipendono gli altri moduli vive qui. Nessuna business logic, solo definizioni.

### Enum EncodingFormat (riga 10)

```python
class EncodingFormat(StrEnum):
    BASE64 = "base64"
    BASE64URL = "base64url"
    BASE32 = "base32"
    HEX = "hex"
    URL = "url"
```

`StrEnum` (Python 3.11+) significa che ogni membro è anche una stringa. Puoi confrontare `EncodingFormat.BASE64 == "base64"` e il risultato è `True`. Typer lo usa direttamente per l'opzione `--format`: genera automaticamente le scelte a partire dai valori dell'enum. Non serve codice di validazione separato.

### Set di caratteri (righe 34–46)

```python
BASE64_CHARSET: Final[frozenset[str]] = frozenset("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=")
```

Il set di caratteri validi di ogni encoding è un `frozenset` per avere test di appartenenza in O(1). Il detector controlla `all(c in BASE64_CHARSET for c in stripped)`. Con un frozenset, ogni ricerca di un carattere è un controllo su una hash table invece di una scansione lineare. Per stringhe corte la differenza è trascurabile, ma è comunque la struttura dati giusta.

`Final` di `typing` dice a mypy che si tratta di costanti che non dovrebbero mai essere riassegnate.

### Soglie (righe 24–33)

`PEEL_MAX_DEPTH = 20` limita la ricorsione. In pratica, anche i payload di malware pesantemente offuscati raramente superano i 5–6 layer, ma il limite impedisce a input patologici di girare all'infinito.

`CONFIDENCE_THRESHOLD = 0.6` è il punteggio minimo perché un rilevamento sia considerato valido. Questo valore è stato calibrato per evitare falsi positivi pur intercettando gli encoding legittimi. Troppo basso e stringhe casuali generano falsi rilevamenti. Troppo alto e gli encoding corti o ambigui vengono mancati.

`MIN_INPUT_LENGTH = 4` esclude dal rilevamento gli input banalmente corti. Una stringa di 3 caratteri potrebbe corrispondere per coincidenza a quasi qualsiasi formato.

## encoders.py — Trasformazioni pure

### Funzioni di encode/decode (righe 22–70)

Ogni formato ha una funzione di encode e una di decode dedicate. Sono piccole, mirate e testabili in modo indipendente.

Dettagli di implementazione degni di nota:

**decode_base64 (riga 26)**: usa `validate=True` in `b64.b64decode()`. Senza questo flag, il decoder base64 di Python ignora silenziosamente i caratteri esterni all'alfabeto. Con il flag, qualsiasi carattere non valido solleva `binascii.Error`. Questa è la conformità rigorosa a RFC 4648. Il documento `01-CONCEPTS.md` spiega perché i decoder permissivi creano problemi di sicurezza (attacchi padding oracle).

**decode_base64 e decode_base64url (righe 27, 36)**: entrambe chiamano `"".join(data.split())` prima del decoding. Questo rimuove tutti gli spazi bianchi (newline, spazi, tab). I dati base64 arrivano spesso spezzati a 76 caratteri (standard MIME), e il decoder deve gestirli.

**decode_base32 (riga 45)**: chiama `.upper()` dopo aver rimosso gli spazi bianchi. L'alfabeto di Base32 è solo maiuscolo, ma gli utenti potrebbero incollare testo minuscolo. Normalizzare maiuscole/minuscole prima del decoding è più tollerante senza essere insicuro.

**decode_hex (righe 53–57)**: rimuove i separatori hex comuni (spazio, due punti, trattino, punto). Questo significa che `AA:BB:CC`, `AA BB CC`, `AA-BB-CC` e `AABBCC` vengono tutti decodificati correttamente. I dati hex reali arrivano in molti formati: gli indirizzi MAC usano i due punti, gli hex dump usano gli spazi, alcuni tool usano i trattini.

**encode_url / decode_url (righe 60–70)**: supportano un argomento keyword-only `form`. L'URL encoding standard usa `%20` per gli spazi. Il form encoding (`application/x-www-form-urlencoded`) usa `+`. La distinzione conta nel security testing delle applicazioni web, quindi sono supportati entrambi. L'`*` in `def encode_url(data: bytes, *, form: bool = False)` forza `form` a essere keyword-only, impedendo un uso posizionale accidentale.

### Il registry (righe 73–85)

```python
ENCODER_REGISTRY: dict[EncodingFormat, tuple[EncoderFn, DecoderFn]] = {
    EncodingFormat.BASE64: (encode_base64, decode_base64),
    ...
}
```

In Python le funzioni sono oggetti di prima classe. Questo dizionario mappa ogni formato sulla relativa coppia (encoder, decoder). Il dispatch è una ricerca nel dizionario:

```python
def encode(data: bytes, fmt: EncodingFormat) -> str:
    encoder_fn, _ = ENCODER_REGISTRY[fmt]
    return encoder_fn(data)
```

La voce URL usa delle lambda (`lambda data: encode_url(data)`) perché le funzioni URL hanno un parametro `form` in più che non corrisponde alle firme di tipo `EncoderFn` / `DecoderFn`. Le lambda creano funzioni wrapper con il default `form=False`.

### try_decode (righe 98–107)

```python
def try_decode(data: str, fmt: EncodingFormat) -> bytes | None:
    try:
        return decode(data, fmt)
    except (ValueError, binascii.Error, UnicodeDecodeError, UnicodeEncodeError):
        return None
```

Il detector e il peeler devono tentare il decoding senza andare in crash se l'input non è valido. `try_decode` avvolge `decode` e converte le eccezioni in `None`. La tupla di eccezioni copre tutti i possibili fallimenti di decoding su tutti i formati. Questo è il pattern "errors as values", comune nella programmazione funzionale.

## detector.py — Punteggio di confidenza

### DetectionResult (riga 24)

```python
@dataclass(frozen=True, slots=True)
class DetectionResult:
    format: EncodingFormat
    confidence: float
    decoded: bytes | None
```

Ogni risultato di rilevamento porta con sé la propria confidenza come float da 0.0 a 1.0. Questo è fondamentalmente diverso da un controllo binario "è base64? sì/no". Una stringa come `CAFE` potrebbe essere hex o base64. Il punteggio di confidenza permette al sistema di dire "è hex all'80% di probabilità, base64 al 50%" e ordinare di conseguenza.

### Analisi di uno scorer: _score_base64 (righe 31–67)

Questo è lo scorer più sfumato, perché il set di caratteri di base64 si sovrappone a quello di altri formati.

**Fase 1 — Rifiuto rapido (righe 32–38)**:
- Rimuovi gli spazi bianchi
- Rifiuta se più corta di 4 caratteri
- Rifiuta se un qualsiasi carattere non è in `BASE64_CHARSET`
- Rifiuta se la lunghezza non è divisibile per 4

Questi controlli sono economici. Se uno fallisce, la funzione restituisce subito 0.0 senza tentare un decoding.

**Fase 2 — Punteggio strutturale (righe 40–58)**:
- Parti da 0.4 (base per charset + lunghezza corrispondenti)
- +0.1 per padding valido (0, 1 o 2 caratteri `=`)
- +0.1 se sono presenti `+` o `/` (questi distinguono base64 da hex/base32)
- +0.1 per maiuscole e minuscole mescolate (lettere maiuscole E minuscole)
- penalità di -0.2 se non ci sono maiuscole né caratteri speciali (`+`, `/`, `=`)
- +0.05 per lunghezza >= 8

Quella penalità di -0.2 è cruciale. Una stringa hex come `6132666338` contiene solo lettere minuscole e cifre. Senza la penalità, ottiene come base64 0.4 + 0.1 (padding) + 0.05 (lunghezza) + 0.15 (decode) + 0.15 (stampabile) = 0.85. Con la penalità: 0.65. Nel frattempo lo scorer hex le assegna 0.80. Vince hex.

**Fase 3 — Validazione tramite decode (righe 60–66)**:
- Tenta un vero decoding base64 con `try_decode`
- Se il decoding fallisce, restituisci 0.0 (struttura base64 non valida)
- +0.15 per un decoding riuscito
- +0.15 se l'output decodificato è testo stampabile

### Il problema della disambiguazione tra hex e base64

Questa è stata la sfida di rilevamento più difficile. Considera la stringa hex `6332566a636d563049484268655778765957513d` (che è `c2VjcmV0IHBheWxvYWQ=` codificato in hex, che a sua volta è `secret payload` codificato in base64).

Ogni carattere di quella stringa hex (`0-9`, `a-f`) è anche un carattere base64 valido. La lunghezza della stringa risulta divisibile per 4. Senza un punteggio accurato, entrambi i formati ottengono una confidenza simile.

La soluzione usa due segnali:

1. **Controllo di maiuscole/minuscole mescolate nello scorer base64 (righe 50–55)**: le stringhe hex sono tipicamente tutte minuscole o tutte maiuscole. Il base64 ha quasi sempre maiuscole e minuscole mescolate. Se non ci sono maiuscole né caratteri speciali, base64 riceve una penalità di -0.2.

2. **Bonus per maiuscole/minuscole coerenti nello scorer hex (righe 154–156)**: se tutti i caratteri alfabetici hanno la stessa forma (maiuscola o minuscola), hex riceve un bonus di +0.1.

Questi due aggiustamenti creano una separazione affidabile. La stringa hex ottiene ~0.80 come hex e ~0.50 come base64.

### _score_url (righe 174–192)

Lo scorer URL funziona diversamente dagli altri. Invece di controllare un set di caratteri, usa una regex per trovare i pattern `%XX`:

```python
_URL_PATTERN = re.compile(r"%[0-9a-fA-F]{2}")
```

La confidenza cresce con il rapporto tra caratteri codificati e lunghezza totale. Una stringa composta al 50% da sequenze percent-encoded ottiene un punteggio più alto di una con un singolo `%20`. Questo gestisce l'intero spettro, da "testo per lo più normale con uno spazio codificato" a "payload interamente percent-encoded".

### detect_encoding (righe 206–223)

L'orchestratore. Esegue ogni scorer, filtra in base alla soglia, ordina per confidenza decrescente. Il chiamante ottiene una lista ordinata di tutti i formati possibili sopra 0.6 di confidenza.

`detect_best` (righe 226–228) è una funzione di comodo che restituisce solo il risultato migliore.

## peeler.py — Rimozione ricorsiva dei layer

### Il loop principale (righe 33–73)

La funzione peel è iterativa, non ricorsiva. A ogni iterazione:

1. Chiama `detect_best(current_text)` per trovare il formato più probabile
2. Controlla tre condizioni di uscita: nessun rilevamento, sotto soglia, decode che ha restituito None
3. Registra un `PeelLayer` con la profondità corrente, il formato, la confidenza e le anteprime
4. Decodifica e prova a interpretare il risultato come UTF-8

Il controllo UTF-8 alla riga 65 è la condizione di arresto naturale. Quando arrivi ai dati originali (che potrebbero essere binari), non saranno UTF-8 valido, e il loop si interrompe. Questo impedisce anche al peeler di provare a "rilevare" encoding in spazzatura binaria.

### Perché iterativo invece che ricorsivo

La funzione usa un loop `for` con `max_depth` invece di chiamare se stessa. In Python, le chiamate ricorsive hanno un overhead (nuovi stack frame) e rischiano un `RecursionError` per catene profonde. Un approccio iterativo con uno stato del loop esplicito è più pulito e ha un limite rigido tramite `range(max_depth)`.

### Design di PeelResult (riga 26)

```python
@dataclass(frozen=True, slots=True)
class PeelResult:
    layers: tuple[PeelLayer, ...]
    final_output: bytes
    success: bool
```

`layers` è una tupla, non una lista. Dato che il risultato è frozen, nemmeno i layer dovrebbero essere modificabili. `final_output` è `bytes` invece di `str` perché il contenuto decodificato finale potrebbe essere costituito da dati binari. `success` è `True` se è stato rimosso almeno un layer, `False` se l'input non sembrava affatto codificato.

## formatter.py — Output nel terminale

### stderr per Rich, stdout per i dati (riga 19)

```python
console = Console(stderr=True)
```

Tutto l'output Rich (pannelli, tabelle, colori) va su stderr. Quando l'utente mette l'output in pipe (`b64tool decode "..." | other_tool`), su stdout arrivano solo i dati grezzi. Il controllo `is_piped()` alla riga 22 gestisce questo comportamento: la modalità pipe scrive stringhe grezze, la modalità terminale scrive pannelli Rich.

### Codifica a colori della confidenza (righe 164–169)

```python
def _confidence_color(confidence: float) -> str:
    if confidence >= 0.9:
        return "green"
    if confidence >= 0.7:
        return "yellow"
    return "red"
```

Feedback visivo nell'output di peel. Verde significa che il tool è molto sicuro del formato rilevato. Giallo significa che è probabilmente corretto ma c'è una certa ambiguità. Rosso significa che il rilevamento è marginale (appena sopra la soglia di 0.6).

## cli.py — Mettere tutto insieme

### Applicazione Typer (righe 39–45)

```python
app = typer.Typer(
    name="b64tool",
    help="Multi-format encoding/decoding CLI with recursive layer detection",
    no_args_is_help=True,
    pretty_exceptions_show_locals=False,
)
```

`no_args_is_help=True` mostra il messaggio di aiuto quando l'utente esegue `b64tool` senza sottocomando. `pretty_exceptions_show_locals=False` impedisce a Typer di stampare le variabili locali in caso di crash, cosa che potrebbe esporre dati sensibili in fase di encoding/decoding.

### Type hint Annotated (ovunque)

Ogni argomento e opzione della CLI usa `Annotated[type, typer.Option(...)]`. Questo è il pattern moderno di Typer (che sostituisce il vecchio stile `typer.Option(default, ...)`). L'annotazione di tipo e i metadati della CLI restano insieme, e mypy valida correttamente i tipi.

### Parsing dei passi della catena (righe 264–281)

```python
def _parse_chain_steps(raw: str) -> list[EncodingFormat]:
```

Esegue il parsing di `"base64,hex,url"` in `[EncodingFormat.BASE64, EncodingFormat.HEX, EncodingFormat.URL]`. Ogni passo viene ripulito dagli spazi, convertito in minuscolo e validato rispetto all'enum. I nomi di formato non validi producono un errore chiaro che elenca tutte le opzioni valide. Questa è una funzione privata (con prefisso `_`) perché è un dettaglio implementativo del comando chain.

## utils.py — Risoluzione dell'input

### Tre sorgenti di input (righe 12–43)

Sia `resolve_input_bytes` sia `resolve_input_text` seguono la stessa priorità:

1. **File** (flag `--file`): lettura da disco
2. **Argomento**: stringa diretta sulla riga di comando
3. **Stdin**: input in pipe (solo se stdin non è un TTY)
4. Se nessuno di questi: solleva `typer.BadParameter`

Questo significa che funzionano tutti questi:
```bash
b64tool encode "Hello"
b64tool encode --file data.txt
echo "Hello" | b64tool encode
```

La distinzione tra byte e testo conta. L'encoding opera sui byte (puoi codificare un file binario). Il decoding opera sul testo (la stringa codificata è sempre ASCII-safe). Ecco perché ci sono due funzioni separate invece di una generica.

### is_printable_text (righe 60–68)

Usata dal detector per aumentare la confidenza quando l'output decodificato sembra testo leggibile. La soglia (default 0.8) significa che almeno l'80% dei caratteri deve essere stampabile o uno spazio bianco comune (`\n`, `\r`, `\t`). Questo impedisce che dati binari che per caso si decodificano senza errori ricevano un bonus di "testo stampabile".
