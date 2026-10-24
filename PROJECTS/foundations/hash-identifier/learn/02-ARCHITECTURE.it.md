# Architettura

Questa pagina riguarda *la forma del codice*. Non cosa fa ogni riga — quello è nella pagina successiva. Qui allarghiamo lo sguardo e osserviamo come i pezzi si incastrano, quali dati fluiscono dove e perché abbiamo scelto questa forma rispetto alle alternative.

## 1. Il quadro generale

L'intero tool è un unico file Python: `hash_identifier.py`. Tutto ciò che viene eseguito vive in quel file. Al suo interno ci sono tre livelli:

```
┌─────────────────────────────────────────────────────────────┐
│  CLI layer  (main, _build_argument_parser, _render_table)   │
│  - reads command-line arguments                             │
│  - prints the colored table to your terminal                │
│  - returns an exit code                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │ calls
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Pure-function layer  (identify)                            │
│  - the actual decision-making                               │
│  - takes a string, returns a list of HashCandidate          │
│  - touches NO files, NO network, NO global state            │
└──────────────────────────┬──────────────────────────────────┘
                           │ uses
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Data layer  (PREFIX_RULES, HEX_LENGTH_RULES, charsets)     │
│  - lookup tables describing what we know about hashes       │
│  - read-only, defined at module load time                   │
└─────────────────────────────────────────────────────────────┘
```

Leggendo dall'alto verso il basso, il livello CLI è l'**esterno** del programma: si occupa dell'essere umano. Il livello delle funzioni pure è il **cervello**: riceve una stringa pulita e restituisce una risposta pulita. Il livello dei dati è la **conoscenza**: ogni "sappiamo X sull'hash Y" vive in una di quelle tabelle, non sparso per il codice.

Questa suddivisione in tre parti è deliberata. Possiamo testare il cervello in isolamento senza mai avviare una CLI — e il file di test fa esattamente questo, chiamando direttamente `identify()`. Possiamo cambiare l'aspetto della tabella (colore, layout, output JSON) senza toccare il cervello. E possiamo aggiungere nuovi formati di hash aggiungendo una riga a una tabella, non aggiungendo una nuova funzione.

## 2. Il flusso dei dati in una singola esecuzione

Ecco cosa succede quando digiti `just run -- 5f4dcc3b5aa765d61d8327deb882cf99`:

```
                                            (your terminal)
                                                  │
                                                  │  "5f4dcc3b5aa765d61d8327deb882cf99"
                                                  ▼
                              ┌─────────────────────────────────────┐
                              │  argparse                           │
                              │  parses sys.argv into args.hash     │
                              └────────────────┬────────────────────┘
                                               │ args.hash = "5f4dcc..."
                                               ▼
                              ┌─────────────────────────────────────┐
                              │  identify(args.hash)                │
                              │                                     │
                              │  text = args.hash.strip()           │
                              │                                     │
                              │  ┌─────────────────────────────┐    │
                              │  │ Step 1: prefix match?       │    │
                              │  └────────────┬────────────────┘    │
                              │  no match     │                     │
                              │               ▼                     │
                              │  ┌─────────────────────────────┐    │
                              │  │ Step 2: special shape?      │    │
                              │  │  (NetNTLM / MySQL5 / DES)   │    │
                              │  └────────────┬────────────────┘    │
                              │  no match     │                     │
                              │               ▼                     │
                              │  ┌─────────────────────────────┐    │
                              │  │ Step 3: hex + length match? │    │
                              │  │  → 32 hex chars → MD5/NTLM..│ ✔  │
                              │  └────────────┬────────────────┘    │
                              │               │                     │
                              │       returns [HashCandidate, ...]  │
                              └────────────────┬────────────────────┘
                                               │
                                               ▼
                              ┌─────────────────────────────────────┐
                              │  _render_table()                    │
                              │  builds a rich.Table, prints it     │
                              └────────────────┬────────────────────┘
                                               │
                                               ▼
                                       (your terminal)

      ┌─────────────────────────────────────────────────────────┐
      │  Candidates for: 5f4dcc3b5aa765d61d8327deb882cf99       │
      │  ╭───────────┬────────────┬───────────────────────────╮ │
      │  │ algorithm │ confidence │ reason                    │ │
      │  ├───────────┼────────────┼───────────────────────────┤ │
      │  │ MD5       │ medium     │ 32 hex chars — most likely│ │
      │  │ NTLM      │ low        │ 32 hex chars — also poss. │ │
      │  │ MD4       │ low        │ 32 hex chars — also poss. │ │
      │  │ RIPEMD-128│ low        │ 32 hex chars — also poss. │ │
      │  ╰───────────┴────────────┴───────────────────────────╯ │
      └─────────────────────────────────────────────────────────┘
```

Il cervello è il riquadro centrale. Tutto ciò che sta sopra e sotto è solo "idraulica": far entrare la stringa, far uscire la tabella.

## 3. La pipeline decisionale a sei passi

Il cervello (`identify()`) è strutturato in **sei passi numerati**. Ogni passo è un'occasione per cortocircuitare e restituire un verdetto. Se un passo trova una corrispondenza, la funzione ritorna immediatamente. Altrimenti, il controllo scende al passo successivo.

Questa forma — "prova prima il segnale più forte, ripiega su quelli più deboli" — si chiama **decision cascade** o **rule pipeline**. Vedrai questo pattern ovunque negli strumenti di sicurezza: filtri antispam, regole IDS, euristiche antivirus, fingerprinting. Condividono tutti lo stesso scheletro.

```
        ┌────────────────────────────────┐
        │ Step 1: PREFIX_RULES?          │ HIGH confidence
        │ Walk the prefix table.         │ ────────►  return
        │ Any prefix start match wins.   │ first match
        └─────────────┬──────────────────┘
                      │ no match
                      ▼
        ┌────────────────────────────────┐
        │ Step 2: special shapes?        │ HIGH/MEDIUM
        │ NetNTLMv2 / NetNTLMv1 (`::`)   │ ────────►  return
        │ MySQL5  (`*` + 40 upper hex)   │ first match
        │ DES crypt  (13 chars)          │
        └─────────────┬──────────────────┘
                      │ no match
                      ▼
        ┌────────────────────────────────┐
        │ Step 3: pure hex?              │ MEDIUM/LOW
        │ If yes, look up length in      │ ────────►  return
        │ HEX_LENGTH_RULES, return all   │ ranked list
        │ candidates ranked by likelihood│
        └─────────────┬──────────────────┘
                      │ not hex
                      ▼
        ┌────────────────────────────────┐
        │ Step 4: generic `$algo$...`?   │ LOW
        │ Looks like a PHC string but    │ ────────►  return
        │ we have no specific rule.      │ generic match
        │ Report it as a generic PHC.    │
        └─────────────┬──────────────────┘
                      │ no
                      ▼
        ┌────────────────────────────────┐
        │ Step 5: shape hint?            │ LOW
        │ Looks like a JWT (eyJ...) or   │ ────────►  return
        │ base64 (`+`, `/`, `=`)?        │ "not a hash"
        │ Tell the user it's not a hash. │
        └─────────────┬──────────────────┘
                      │ no
                      ▼
        ┌────────────────────────────────┐
        │ Step 6: give up.               │ none
        │ Return empty list.             │ ────────►  []
        │ CLI prints "could not identify"│
        └────────────────────────────────┘
```

Qui l'ordine conta, e non è arbitrario. Proviamo sempre **il test più specifico per primo** e **il test più generico per ultimo**:

1. I prefissi PHC sono rivelatori inequivocabili — l'hash stesso nomina l'algoritmo.
2. Anche le forme speciali (NetNTLM, MySQL5, DES crypt) sono forti: hanno strutture distintive.
3. Hex + lunghezza è un segnale che *restringe*, non definitivo — sceglie una famiglia, non un membro.
4. Il fallback PHC generico cattura gli hash che *sembrano* avere forma PHC ma non sono nella nostra tabella.
5. Gli indizi di forma gestiscono il caso comune "ho incollato la cosa sbagliata" (le persone mettono continuamente JWT negli identificatori di hash).
6. Lista vuota = un onesto "non lo so."

Se invertissimo l'ordine — diciamo, controllando la lunghezza prima del prefisso — classificheremmo erroneamente un hash bcrypt come "60 caratteri hex, nessuna regola di lunghezza" perché non guarderemmo mai il suo prefisso `$2b$`. L'ordine codifica la priorità.

## 4. L'oggetto HashCandidate

Il cervello non restituisce una stringa — restituisce una lista di oggetti `HashCandidate`. Un candidato ha tre campi:

```
┌─────────────────────────────────────────────────────────────┐
│  HashCandidate                                              │
│                                                             │
│    algorithm:   str         e.g. "MD5", "bcrypt", "SHA-256" │
│    confidence:  Literal     "high" | "medium" | "low"       │
│    reason:      str         "prefix `$2b$` — bcrypt PHC..." │
│                                                             │
│  frozen=True   ── immutable, can't be mutated after creation│
│  slots=True    ── memory-efficient (no __dict__)            │
└─────────────────────────────────────────────────────────────┘
```

La forma è intenzionale. **`algorithm`** è ciò che hashcat vuole sapere. **`confidence`** dice al lettore umano quanto fidarsi di questa ipotesi. **`reason`** è la *prova* — una spiegazione di una riga del perché il tool ha fatto questa ipotesi. Il campo reason è ciò che rende il tool didattico: l'utente non vede solo "bcrypt" ma "prefix `$2b$` — bcrypt PHC string, 2b variant (current)."

Usiamo `@dataclass(frozen=True, slots=True)` invece di scrivere a mano una classe con `__init__` e `__repr__`:

- **`frozen=True`** significa che una volta costruito un `HashCandidate`, non puoi modificarlo. Se da qualche parte nel codice si provasse `candidate.algorithm = "something else"`, Python solleverebbe `FrozenInstanceError`. Questo rende il flusso dei dati prevedibile: un candidato che esce da `identify()` è lo stesso candidato ovunque compaia in seguito.
- **`slots=True`** è un'ottimizzazione della memoria. Senza slots, ogni istanza si porta dietro un `__dict__` per aggiungere attributi al volo. Non ci serve, quindi lo disattiviamo e risparmiamo memoria.

Entrambi i flag segnalano anche l'*intento* a chi legge: "questo è un value object, non un contenitore mutabile di stato." Quel segnale conta più dei byte risparmiati.

## 5. Le tabelle di dati come fonte di verità

Se volessi aggiungere un nuovo formato di hash a questo tool, non scriveresti nuova logica. Aggiungeresti una riga a una di queste tabelle:

```
PREFIX_RULES: list of (prefix, algorithm, note)
─────────────────────────────────────────────────
("$argon2id$", "Argon2id", "modern PHC string..."),
("$2b$",       "bcrypt",   "bcrypt PHC string..."),
("$6$",        "SHA-512 crypt", "Unix crypt..."),
... ~25 more rows


HEX_LENGTH_RULES: dict of {length_in_hex_chars: [algorithms]}
─────────────────────────────────────────────────────────────
32:  ["MD5", "NTLM", "MD4", "RIPEMD-128"],
40:  ["SHA-1", "RIPEMD-160"],
64:  ["SHA-256", "SHA3-256", "BLAKE2s-256", "RIPEMD-256"],
128: ["SHA-512", "SHA3-512", "BLAKE2b-512", "Whirlpool"],
... etc


HEX_CHARSET, _HEX_UPPER_CHARSET, _DESCRYPT_CHARSET
──────────────────────────────────────────────────
The alphabets used by each format. frozenset for fast lookup.
```

Questo si chiama **data-driven design**. Le regole vivono nei dati, non nel codice. Tre benefici:

1. **Aggiungere un nuovo formato è una riga.** Nessuna nuova funzione, nessuna nuova impalcatura di test da scrivere.
2. **Le regole sono ispezionabili.** Puoi leggere `PREFIX_RULES` e vedere immediatamente ogni formato che il tool conosce.
3. **Le regole sono testabili.** Il file di test itera su `PREFIX_RULES` e conferma che ogni prefisso venga riconosciuto — così i dati e il comportamento non possono disallinearsi.

Quando leggerai la pagina sull'implementazione, nota quanto pochi corpi di funzione abbiano catene *if/elif/elif*. Le decisioni avvengono dentro i lookup nelle tabelle, non dentro i condizionali. Questo è il data-driven design all'opera.

## 6. Due funzioni helper

Il cervello è una grande funzione (`identify`), ma accanto a essa vivono due piccoli helper. Entrambi rispondono a domande sì/no sulla stringa di input:

```
┌──────────────────────────────────────────────────────────┐
│  _is_hex(text) -> bool                                   │
│    "Is every character of text a valid hex digit?"       │
│    Used in step 3 to decide whether to look up length.   │
├──────────────────────────────────────────────────────────┤
│  _is_mysql5(text) -> bool                                │
│    "Does text look like `*` + 40 uppercase hex chars?"   │
│    Used in step 2 for MySQL5 detection.                  │
├──────────────────────────────────────────────────────────┤
│  _is_descrypt(text) -> bool                              │
│    "Is text 13 chars from `./0-9A-Za-z`?"                │
│    Used in step 2 for legacy DES crypt detection.        │
└──────────────────────────────────────────────────────────┘
```

Il trattino basso iniziale (`_is_hex`, non `is_hex`) è una convenzione Python che significa **"questo è privato del modulo."** Dice agli altri sviluppatori: "questo è un dettaglio implementativo di `hash_identifier.py`; non importarlo da altrove." Python non lo *impone* — puoi comunque importare nomi privati — ma ogni linter e ogni revisore ti segnalerà se lo fai.

Gli helper sono minuscoli di proposito. Ognuno è una singola domanda booleana. Li estraiamo da `identify()` non perché siano complicati ma perché dar loro un nome fa leggere `identify()` come l'inglese: "se `_is_hex(text)`, fai il matching hex-lunghezza." Se mettessimo il test inline, l'occhio dovrebbe analizzarlo.

## 7. Il livello CLI

Il livello CLI è la parte con cui l'essere umano interagisce davvero. Fa tre cose:

```
┌────────────────────────────────────────────────────────────┐
│ _build_argument_parser()                                   │
│   Sets up argparse — defines that the program takes one    │
│   positional argument (`hash`) and an optional `--top N`   │
│   flag. Returns a configured parser.                       │
│                                                            │
│   Pulled out of main() so tests can build the parser       │
│   without actually running the CLI.                        │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ _render_table(raw_input, candidates, console)              │
│   Builds a rich.Table object, adds one row per candidate,  │
│   colors the confidence column (green/yellow/cyan),        │
│   and prints it.                                           │
│                                                            │
│   Takes the rich Console as an argument so tests can pass  │
│   a captured-output Console and verify what got printed.   │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ main()                                                     │
│   Parses args, calls identify(), prints the table.         │
│   Returns an exit code:                                    │
│     0 → at least one candidate found                       │
│     1 → no candidates (printed an error message)           │
│                                                            │
│   The exit code lets shell scripts do                      │
│       `if hashid "$x"; then ...`                           │
│   to react to whether identification succeeded.            │
└────────────────────────────────────────────────────────────┘
```

La scelta di design cruciale nel livello CLI è la **dependency injection**. `_render_table` riceve un oggetto `Console` come parametro invece di crearne uno al suo interno. Sembra sofisticato, ma significa solo: "dammi la stampante da usare." Alla funzione non importa se le dai una vera Console del terminale o una Console di test che cattura l'output in una stringa. Questo rende la funzione testabile senza scrivere sul tuo vero terminale durante i test.

## 8. Il file di test rispecchia il cervello

`test_hash_identifier.py` è strutturato per rispecchiare `hash_identifier.py`. Testa ogni comportamento che il tool dichiara di avere:

```
test_bcrypt_prefix_is_recognized            ── covers PREFIX_RULES row $2b$
test_argon2id_prefix_is_recognized          ── covers PREFIX_RULES row $argon2id$
test_apr1_prefix_is_recognized              ── covers PREFIX_RULES row $apr1$
test_sha512_crypt_prefix_is_recognized      ── covers PREFIX_RULES row $6$
test_django_pbkdf2_prefix_is_recognized     ── covers PREFIX_RULES row pbkdf2_sha256$

test_mysql5_format_is_recognized            ── covers Step 2 / _is_mysql5
test_mysql5_rejects_lowercase_body          ── covers the "be honest, don't lie" rule
test_netntlmv2_format_is_recognized         ── covers Step 2 / NetNTLMv2
test_netntlmv1_format_is_recognized         ── covers Step 2 / NetNTLMv1
test_descrypt_format_is_recognized          ── covers Step 2 / _is_descrypt

test_md5_length_returns_md5_first           ── covers Step 3 / 32 hex chars
test_sha1_length_returns_sha1_first         ── covers Step 3 / 40 hex chars
test_sha256_length_returns_sha256_first     ── covers Step 3 / 64 hex chars
test_mysql323_length_returns_mysql323_first ── covers Step 3 / 16 hex chars

test_unknown_phc_string_falls_back_to_generic
                                            ── covers Step 4

test_jwt_input_is_called_out_as_not_a_hash  ── covers Step 5
test_base64_blob_is_called_out_as_not_a_hash── covers Step 5

test_empty_input_returns_no_candidates      ── covers Step 6 (edge case)
test_garbage_returns_no_candidates          ── covers Step 6
test_input_is_trimmed_of_whitespace         ── covers the .strip() at the top

test_hash_candidate_is_frozen               ── covers the @dataclass(frozen=True)

test_every_prefix_rule_is_recognized_with_high_confidence
                                            ── meta-test: iterates over
                                            ── PREFIX_RULES and asserts every
                                            ── row produces a HIGH-confidence
                                            ── match. Keeps data and code in sync.
```

Il meta-test in fondo è il più interessante. È una protezione contro future regressioni: se aggiungi una nuova riga a `PREFIX_RULES` e dimentichi di aggiornare la logica di matching, questo test scatta. Il test cicla sui dati, non su input hardcoded — quindi il test cresce automaticamente man mano che cresce la tabella dei dati.

## 9. Perché qui contano le funzioni pure

Il cervello (`identify()`) è ciò che si chiama una **funzione pura**:

- Dato lo stesso input, restituisce sempre lo stesso output.
- Non modifica nulla al di fuori di sé (nessuna variabile globale, nessun file, nessuna rete).
- Non dipende da nulla al di fuori di sé (nessun orario corrente, nessuna variabile d'ambiente, nessun numero casuale).

Sembra una piccola cosa. È enorme. Le funzioni pure sono:

- **Banalmente testabili.** `assert identify("5f4d...") == [HashCandidate(...)]`. Nessun mocking, nessun setup, nessun teardown.
- **Banalmente parallelizzabili.** Potresti eseguire `identify()` su un milione di hash su 16 core CPU con zero coordinamento, perché nessuna coppia di chiamate può interferire tra loro.
- **Banalmente cacheabili.** Stesso input → stesso output → memoizza liberamente.
- **Banalmente comprensibili.** Puoi leggere `identify()` in isolamento. Non devi sapere in che stato si trova il programma.

La maggior parte dei programmi reali non può essere interamente pura — devono leggere file, inviare pacchetti, scrivere su database. Ma quasi sempre puoi *ritagliare* un nucleo puro e metterci attorno un guscio sottile che si occupa delle cose con effetti collaterali. È esattamente l'architettura qui: cervello puro al centro, guscio CLI con effetti collaterali attorno.

Questo a volte si chiama pattern [Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell). Vale la pena imparare il nome perché, una volta che lo vedi, lo noterai ovunque.

## 10. Prossimo passo

Ora conosci la forma: tre livelli, sei passi, tre tabelle di dati, tre helper, un record `HashCandidate`, un guscio CLI. Leggi **[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md)** come prossima cosa e percorreremo insieme ogni riga di `hash_identifier.py`.
