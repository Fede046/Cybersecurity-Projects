# Walkthrough dell'implementazione

Questa pagina legge `hash_identifier.py` insieme a te dall'inizio alla fine. Ogni funzionalità di Python viene spiegata la prima volta che compare. Se sei completamente nuovo a Python, leggi questa pagina con il file sorgente aperto in un'altra finestra, così puoi vedere le righe di cui stiamo parlando.

> In tutta questa pagina, diciamo "il file sorgente" per intendere `hash_identifier.py`. Aprilo ora: `code hash_identifier.py`, oppure `nano hash_identifier.py`, o qualunque editor tu usi.

## 1. L'intestazione del file

```python
"""
©AngelaMos | 2026
hash_identifier.py

Identify what kind of hash a string is, by inspecting its shape
...
"""
```

Quel blocco tra tripli apici in cima al file è una **module docstring**. In Python, qualsiasi cosa dentro `"""..."""` è un letterale stringa. Quando il file viene caricato, Python vede questa stringa in cima senza alcun nome associato e la tratta come documentazione per l'intero modulo. Puoi leggerla in seguito con `help(hash_identifier)` o passando il mouse sopra l'import in un IDE.

La prima riga, `©AngelaMos | 2026`, è il marcatore di copyright richiesto in ogni file di questo repo. La seconda riga è il nome del file. Poi una spiegazione più lunga e leggibile di cosa fa il file. Vedrai lo stesso pattern in ogni file di `PROJECTS/foundations/`.

> **Perché una docstring invece di un commento?** Python ha entrambi. Un `# comment` viene eliminato prima che il codice venga eseguito. Una `"""docstring"""` viene memorizzata sul modulo/funzione/classe ed è disponibile a runtime tramite `__doc__`. Tool come Sphinx, mkdocs e l'aiuto al passaggio del mouse del tuo IDE leggono tutti le docstring, non i commenti. Regola pratica: usa le docstring per "cos'è questa cosa e come si usa", e usa i commenti per "perché esiste *questa specifica riga*."

## 2. Import

```python
import argparse
import sys
from dataclasses import dataclass
from typing import Literal

from rich.console import Console
from rich.table import Table
```

Un'istruzione `import` porta nel tuo file codice da un altro file. Python include centinaia di moduli nella sua **libreria standard** (sempre disponibili, nessuna installazione necessaria) e puoi installarne altri da [PyPI](https://pypi.org/) usando `uv add <package>`.

Ci sono due forme di `import`:

- `import argparse` — importa l'intero modulo con il suo nome. Poi fai riferimento alle cose al suo interno come `argparse.ArgumentParser`.
- `from dataclasses import dataclass` — importa solo una cosa dal modulo, così puoi usarla senza prefisso: `dataclass` invece di `dataclasses.dataclass`.

Usa la seconda forma quando ti servono solo una o due cose e hanno nomi descrittivi. Usa la prima forma quando altrimenti tireresti dentro un mucchio di nomi che potrebbero collidere con i tuoi.

Una riga vuota separa gli import della libreria standard da quelli di terze parti. È una convenzione [PEP 8](https://peps.python.org/pep-0008/#imports). I linter ti urleranno contro se li mescoli.

Panoramica di ciò che abbiamo appena importato:

| Import       | Cos'è                                                                         | Perché ci serve                                           |
| ------------ | ----------------------------------------------------------------------------- | --------------------------------------------------------- |
| `argparse`   | Parser di argomenti CLI della libreria standard                               | Trasforma `sys.argv` in comodi attributi (`args.hash`)    |
| `sys`        | Libreria standard, comunica con l'interprete Python                           | Usiamo `sys.exit(...)` per impostare l'exit code del programma |
| `dataclass`  | Un decorator che trasforma una classe in un piccolo record                    | Ci evita di scrivere `__init__` per `HashCandidate`       |
| `Literal`    | Un type hint che significa "questo valore è una di queste specifiche stringhe" | Vincola `confidence` a `"high" / "medium" / "low"`       |
| `Console`    | Di terze parti, dalla libreria [`rich`](https://github.com/Textualize/rich)   | La cosa che disegna testo colorato nel terminale          |
| `Table`      | Anch'esso da `rich`                                                           | Costruisce la tabella ASCII colorata che stampiamo all'utente |

## 3. Il tipo Literal

```python
Confidence = Literal["high", "medium", "low"]
```

Questa riga crea un **type alias**. Diamo al tipo `Literal["high", "medium", "low"]` un nome amichevole, `Confidence`, e usiamo quel nome ovunque altrove.

Un tipo `Literal` dice: "il valore di questa cosa deve essere esattamente uno di questi valori specifici — non una stringa qualsiasi." Con mypy (il nostro type checker) attivo, codice come questo:

```python
candidate = HashCandidate(algorithm="MD5", confidence="hgih", reason=...)
                                                       ^^^^^^
                                                       typo!
```

verrebbe segnalato già in fase di scrittura, prima ancora che tu esegua il codice. Senza `Literal`, il tipo sarebbe `str` e `"hgih"` passerebbe inosservato finché un utente non notasse l'output scritto male.

Abbiamo scelto `Literal` invece dell'`Enum` di Python perché per piccoli insiemi fissi di stringhe, `Literal` è più leggero — nessuna definizione di classe separata, nessun attributo `.value` da ricordare. (Per insiemi più grandi o quando ti serve comportamento sui valori, `Enum` è la scelta giusta.)

## 4. La dataclass HashCandidate

```python
@dataclass(frozen=True, slots=True)
class HashCandidate:
    """One possible identification of a hash string ..."""
    algorithm: str
    confidence: Confidence
    reason: str
```

Questo è il record di dati che il cervello restituisce. Vediamo cosa sta succedendo:

- **`class HashCandidate:`** definisce un nuovo tipo chiamato `HashCandidate`. Una classe è un progetto per creare oggetti.
- **`algorithm: str`** dichiara un attributo chiamato `algorithm` di tipo `str` (una stringa). La sintassi due-punti-poi-tipo è una **type annotation**. È opzionale in Python ma usata pesantemente nel codice moderno.
- **`@dataclass(...)`** è un *decorator*. Un decorator è una funzione che avvolge la tua classe e la modifica. Il decorator `@dataclass` guarda i tre attributi che hai dichiarato (`algorithm`, `confidence`, `reason`) e genera automaticamente un metodo `__init__`, un metodo `__repr__` e qualche altro metodo dunder. Senza `@dataclass`, dovresti scrivere:

  ```python
  class HashCandidate:
      def __init__(self, algorithm: str, confidence: Confidence, reason: str):
          self.algorithm = algorithm
          self.confidence = confidence
          self.reason = reason
      def __repr__(self):
          return f"HashCandidate(algorithm={self.algorithm!r}, ...)"
      def __eq__(self, other):
          ...
  ```

  `@dataclass` scrive tutto questo per te.

- **`frozen=True`** rende le istanze immutabili. Dopo che l'oggetto è costruito, `candidate.algorithm = "different"` solleva `FrozenInstanceError`. Questo è ciò che rende `HashCandidate` un **value object** — come un intero o una tupla, non cambia dopo la creazione.

- **`slots=True`** è un'ottimizzazione della memoria. Per default, ogni oggetto Python ha un `__dict__` così puoi aggiungergli attributi arbitrari al volo. Non lo vogliamo — i tre campi sono tutto ciò che avremo mai. `slots=True` dice a Python di allocare un array fisso per i tre campi, saltando il dict. Più veloce, più piccolo, e ora anche `obj.typo = "anything"` fallisce (il che è un bene — intercetta i bug).

Userai spesso entrambi i flag insieme per piccoli record di dati. Insieme dicono "questo è un record, trattalo come tale."

## 5. La tabella PREFIX_RULES

```python
PREFIX_RULES: list[tuple[str, str, str]] = [
    ("$argon2id$", "Argon2id", "modern PHC string, the current standard"),
    ("$argon2i$",  "Argon2i",  "PHC string, side-channel-resistant variant"),
    ...
]
```

Questa è una `list` di `tuple`. Scomponiamola.

Una **list** è il container ordinato di base di Python. La scrivi con le parentesi quadre: `[1, 2, 3]`. Puoi aggiungere elementi, rimuoverli, accedervi per indice (`items[0]`).

Una **tuple** è come una lista ma immutabile. La scrivi con le parentesi tonde: `(1, 2, 3)`. Puoi leggerla ma non puoi modificarla dopo la creazione. Le tuple sono il container giusto quando la *posizione* di ciascun valore ha un significato — come le coordinate `(x, y)`, o qui, "il prefisso, il nome dell'algoritmo e la nota" sempre in quell'ordine.

Quindi `PREFIX_RULES` è una lista di tuple da 3 elementi. Ogni tupla dice "se vedi questo prefisso, l'algoritmo ha questo nome, ed ecco una breve nota a riguardo."

La type annotation `list[tuple[str, str, str]]` dice esattamente questo: "una lista i cui elementi sono tuple di tre stringhe." Questo serve puramente al lettore umano e a mypy — a runtime Python non lo impone.

> **Perché una lista e non un dict?** Un dict ci permetterebbe di cercare per prefisso in tempo O(1). Ma i nostri prefissi non hanno tutti la stessa lunghezza — `$2b$` è di 4 caratteri, `$argon2id$` di 10. Non esiste un'operazione veloce "questa stringa è un prefisso di quella stringa per una qualsiasi chiave nel mio dict", quindi percorriamo la lista. Le prestazioni vanno bene perché la lista è corta (~25 voci) e lo facciamo solo una volta per invocazione del programma.

Nota i raggruppamenti dei commenti — `# Argon2 family`, `# bcrypt and its many variants` — sono l'unico tipo di commento che usiamo molto nei progetti del tier foundations. Danno un nome a *sezioni* di dati correlati così che il lettore possa scorrerle.

L'ordine delle voci conta quando due prefissi potrebbero sovrapporsi. In particolare, `$argon2id$` deve venire prima di `$argon2$` perché `"$argon2id$something".startswith("$argon2$")` sarebbe *anch'esso* vero se `$argon2$` fosse nella nostra tabella. Elenchiamo per primi i prefissi più specifici così che trovino corrispondenza per primi.

## 6. La tabella HEX_LENGTH_RULES

```python
HEX_CHARSET: frozenset[str] = frozenset("0123456789abcdefABCDEF")
_HEX_UPPER_CHARSET: frozenset[str] = frozenset("0123456789ABCDEF")

HEX_LENGTH_RULES: dict[int, list[str]] = {
    16:  ["MySQL323", "CRC-64"],
    32:  ["MD5", "NTLM", "MD4", "RIPEMD-128"],
    40:  ["SHA-1", "RIPEMD-160"],
    ...
}
```

Un **set** è un container non ordinato di valori univoci. Un **frozenset** è un set che non puoi modificare dopo la creazione. Qui usiamo frozenset perché:

1. Non abbiamo mai bisogno di aggiungere/rimuovere caratteri dall'alfabeto esadecimale — è noto e fisso.
2. Il lookup (`c in HEX_CHARSET`) è O(1) — tempo costante. Più veloce di `c in "0123456789abcdef..."` che scansionerebbe la stringa carattere per carattere.
3. Marcarlo come `frozenset` segnala al lettore: "questa è una costante fissa, non provare a modificarla."

Un **dict** (dizionario) è una mappatura da chiavi a valori. Lo scrivi con le parentesi graffe: `{key: value, key: value}`. Il lookup è O(1) sulla chiave.

`HEX_LENGTH_RULES` mappa "lunghezza in caratteri hex" a "lista dei nomi degli algoritmi che producono quella lunghezza." Quindi `HEX_LENGTH_RULES[32]` è `["MD5", "NTLM", "MD4", "RIPEMD-128"]` — i quattro algoritmi che producono un output di 32 caratteri esadecimali.

`_HEX_UPPER_CHARSET` inizia con un trattino basso. Convenzione: **il trattino basso iniziale significa privato del modulo.** Sta dicendo "questo è un dettaglio implementativo, non parte dell'interfaccia pubblica." Python non lo impone, ma ogni linter sì. La variante maiuscola esiste perché MySQL5 stampa il suo hex solo in maiuscolo (formato C `%02X`), quindi usiamo un charset più stretto per evitare falsi positivi su input digitati a mano.

## 7. L'helper `_is_hex`

```python
def _is_hex(text: str) -> bool:
    """Return True iff every character in text is a hex digit and text is non-empty"""
    return bool(text) and all(c in HEX_CHARSET for c in text)
```

Un'istruzione `def` definisce una funzione. Leggendo questa firma:

- `def _is_hex(text: str) -> bool:` dichiara una funzione chiamata `_is_hex` che prende un argomento `text` di tipo `str` e restituisce un `bool` (True o False).
- Il trattino basso iniziale la rende privata del modulo.

Il corpo è una riga. Leggiamola da destra a sinistra:

- `(c in HEX_CHARSET for c in text)` è una **generator expression**. Produce una sequenza di booleani: per ogni carattere `c` in `text`, restituisce `True` se `c` è nel nostro charset esadecimale, altrimenti `False`.
- `all(...)` prende quella sequenza e restituisce `True` solo se *ogni* valore prodotto è True. Equivale a "ogni carattere di text è una cifra esadecimale."
- `bool(text)` vale `False` se `text` è vuoto, `True` altrimenti. Ci serve questa guardia perché `all([])` su una sequenza vuota restituisce True (matematicamente ragionevole, praticamente fastidioso — una stringa vuota non è una stringa esadecimale valida).
- `... and ...` va in cortocircuito: se `bool(text)` è False, non ci preoccupiamo di controllare `all(...)`.

La parola `iff` nella docstring è l'abbreviazione di "if and only if" ("se e solo se"). Gli appassionati di matematica e di informatica la usano continuamente; indica un bicondizionale.

## 8. Rilevamento di MySQL5

```python
_MYSQL5_HEX_BODY_LENGTH = 40
_MYSQL5_TOTAL_LENGTH = _MYSQL5_HEX_BODY_LENGTH + 1


def _is_mysql5(text: str) -> bool:
    """Return True for MySQL5 password format: `*` then 40 UPPERCASE hex chars ..."""
    if len(text) != _MYSQL5_TOTAL_LENGTH or not text.startswith("*"):
        return False
    body = text[1:]
    return all(c in _HEX_UPPER_CHARSET for c in body)
```

Le due costanti in cima derivano da una delle regole di questa codebase: **niente magic number**. Confronta:

```python
if len(text) != 41 or not text.startswith("*"):
    return False
```

con:

```python
if len(text) != _MYSQL5_TOTAL_LENGTH or not text.startswith("*"):
    return False
```

Funzionano entrambi. Solo il secondo dice al lettore *perché* 41 è il numero giusto (sono 40 caratteri hex per il corpo più 1 per l'`*` iniziale).

La funzione fa tre cose:

1. Controlla che la lunghezza totale sia esattamente 41 e che il primo carattere sia `*`. In caso contrario, esce con False.
2. Taglia via l'`*` iniziale usando `text[1:]`. Slicing in Python: `text[start:stop]` ti dà la sottostringa dall'indice `start` (incluso) a `stop` (escluso). Omettere `stop` significa "fino alla fine." Quindi `text[1:]` significa "tutto dall'indice 1 in poi" — cioè scarta il primo carattere.
3. Controlla che ogni carattere del corpo sia nel nostro charset esadecimale solo maiuscolo.

La docstring contiene un avvertimento importante: *non possiamo* usare `body.isupper()` per imporre il maiuscolo, perché `str.isupper()` di Python restituisce False per una stringa senza alcun carattere con maiuscole/minuscole. Quindi `"0123456789ABCDEF...".isupper()` restituirebbe correttamente True, ma un corpo di sole cifre restituirebbe False — rifiutando erroneamente un input valido. Controllare l'appartenenza a `_HEX_UPPER_CHARSET` è il test che corrisponde davvero alla specifica.

Questo è il tipo di trappola sottile che impari solo scottandoti. Vale la pena memorizzarlo: **non usare `.isupper()` / `.islower()` come controllo "questa stringa contiene solo caratteri maiuscoli".**

## 9. Rilevamento di DES crypt

```python
_DESCRYPT_CHARSET: frozenset[str] = frozenset(
    "./0123456789"
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    "abcdefghijklmnopqrstuvwxyz"
)
_DESCRYPT_TOTAL_LENGTH = 13


def _is_descrypt(text: str) -> bool:
    """Return True for traditional 13-char DES crypt (legacy /etc/passwd) ..."""
    return (
        len(text) == _DESCRYPT_TOTAL_LENGTH
        and all(c in _DESCRYPT_CHARSET for c in text)
    )
```

DES crypt è il formato *originale* di password hash di Unix degli anni '70. Nessun prefisso, nessun marcatore di salt, solo 13 caratteri da uno specifico alfabeto di 64 caratteri.

La definizione del charset usa la **concatenazione di letterali stringa**: tre letterali stringa uno accanto all'altro vengono uniti automaticamente da Python in fase di parsing. Quindi questo:

```python
"./0123456789"
"ABCDEFGHIJKLMNOPQRSTUVWXYZ"
"abcdefghijklmnopqrstuvwxyz"
```

è esattamente uguale a:

```python
"./0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
```

Ma la forma spezzata è molto più facile da leggere. Funziona per qualsiasi sequenza di letterali stringa adiacenti — utile quando si mandano a capo stringhe lunghe.

`_is_descrypt` in sé consiste in due controlli: lunghezza giusta, alfabeto giusto. Devono essere entrambi veri (`and`). L'intera espressione è racchiusa tra parentesi solo per il layout visivo — Python ti permette di spezzare le espressioni su più righe se sono dentro `()`, `[]` o `{}`.

## 10. La funzione `identify` — il cervello

Questa è l'attrazione principale. È lunga circa 100 righe e segue la pipeline a sei passi di [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md).

### 10a. Il commento che silenzia pylint

```python
# pylint: disable=too-many-return-statements,too-many-branches
```

Questo commento disattiva due specifici warning di pylint *solo per questa funzione*. Pylint normalmente si lamenta delle funzioni con molte istruzioni `return` o molti rami, perché di solito sono un segnale che dovresti fare refactoring.

Teniamo i warning disattivati perché in questo caso la ramificazione *è* la struttura. Sei passi numerati, ognuno dei quali può restituire un risultato — rifattorizzarli in sei funzioni helper sparpaglierebbe la pipeline per il file e la renderebbe più difficile, non più facile, da leggere. Quindi riconosciamo i warning, spieghiamo perché li teniamo disattivati nel blocco di commento inline soprastante e li disattiviamo esplicitamente.

> **Regola pratica sul silenziare i linter:** non silenziare mai in modo generico. Silenzia sempre il warning *specifico*, sulla funzione *specifica*, con un commento che spieghi il perché. Se ti ritrovi a silenziare lo stesso warning ovunque, è la configurazione del linter a essere sbagliata, non il tuo codice.

### 10b. La firma e la docstring

```python
def identify(raw_input: str) -> list[HashCandidate]:
    """Return ranked candidates for what algorithm produced `raw_input` ..."""
```

Riceve una stringa. Restituisce una lista di oggetti `HashCandidate`. Questo è l'intero contratto. La funzione non solleva eccezioni per input sconosciuti — restituisce invece una lista vuota. (Lanciare eccezioni per "non lo so" costringe ogni chiamante ad avvolgere la chiamata in try/except. Restituire una lista vuota è più pulito.)

La docstring è in **stile Numpy**: ha sezioni etichettate (`Parameters`, `Returns`) con il nome del parametro su una riga a sé e la descrizione indentata sotto. Lo stile Numpy è quello usato dalla maggior parte del Python scientifico. L'altro stile principale è Google ("Args:" e "Returns:" con i due punti), e li vedrai entrambi nel mondo reale. Vanno bene tutti e due; scegline uno e sii coerente.

### 10c. Trim e uscita se vuoto

```python
text = raw_input.strip()

if not text:
    return []
```

`str.strip()` restituisce una *nuova* stringa con gli spazi bianchi iniziali e finali rimossi. (Le stringhe sono immutabili in Python — ogni metodo che "modifica" in realtà restituisce una nuova stringa.) Lo facciamo perché gli hash copiati e incollati dai terminali arrivano spesso con newline finali o spazi iniziali.

*Non* convertiamo il testo in minuscolo. Alcuni formati (MySQL5) sono case-sensitive di proposito.

Il controllo `if not text:` intercetta la stringa vuota. In Python, una stringa vuota è **falsy** — conta come False in un contesto booleano. Quindi `not text` è True quando `text` è vuoto. Lo stesso vale per lista vuota, dict vuoto, `None` e `0`. Abituati a questo — è una delle caratteristiche distintive di Python.

### 10d. Passo 1 — percorrere PREFIX_RULES

```python
for prefix, algorithm, note in PREFIX_RULES:
    if text.startswith(prefix):
        return [
            HashCandidate(
                algorithm=algorithm,
                confidence="high",
                reason=f"prefix `{prefix}` — {note}",
            )
        ]
```

Un ciclo `for ... in ...` itera su ogni elemento di una sequenza. Qui, ogni elemento di `PREFIX_RULES` è una tupla da 3 elementi, quindi la **spacchettiamo** direttamente in tre variabili — `prefix`, `algorithm`, `note` — in un colpo solo. Questo si chiama *tuple unpacking* ed è usato continuamente in Python.

`text.startswith(prefix)` è un metodo di `str` che restituisce True se `text` inizia con `prefix`. Esiste anche `endswith`, tra l'altro.

`f"..."` è una **f-string** (formatted string literal). Qualsiasi cosa dentro `{}` viene valutata e inserita nella stringa. Quindi `f"prefix `{prefix}` — {note}"` produce qualcosa come `"prefix \`$2b$\` — bcrypt PHC string, 2b variant (current)"`. Le f-string sono il modo moderno di formattare le stringhe in Python (3.6+); evita gli stili più vecchi `%` e `.format()` nel codice nuovo.

La funzione restituisce una lista contenente un `HashCandidate`. Usiamo argomenti keyword (`algorithm=algorithm`) invece di quelli posizionali così la chiamata si legge chiaramente anche se non ricordi l'ordine dei parametri.

L'intero controllo `if text.startswith(prefix):` ritorna immediatamente alla prima corrispondenza. Questo va bene perché la tabella è progettata in modo che due prefissi non possano corrispondere allo stesso input (quelli più lunghi e specifici vengono prima).

### 10e. Passo 2 — forme speciali non PHC

NetNTLMv2, NetNTLMv1, MySQL5, DES crypt. Ognuno ha il suo blocco.

```python
if "::" in text and text.count(":") >= 4:
    parts = text.split(":")
    if (len(parts) >= 6 and len(parts[4]) == 32 and _is_hex(parts[4])):
        return [HashCandidate(algorithm="NetNTLMv2", ...)]
    if (len(parts) >= 6 and len(parts[3]) == 48 and _is_hex(parts[3])):
        return [HashCandidate(algorithm="NetNTLMv1", ...)]
```

Qui ci sono alcune funzionalità di Python:

- `"::" in text` restituisce True se la sottostringa `"::"` compare in un punto qualsiasi di `text`. L'operatore `in` funziona su stringhe (controllo di sottostringa), liste (appartenenza), dict (lookup della chiave), set (appartenenza) e qualsiasi iterabile.
- `text.count(":")` restituisce quante volte `:` compare nella stringa.
- `text.split(":")` restituisce una lista di sottostringhe, dividendo su `:`. Quindi `"a:b:c".split(":")` dà `["a", "b", "c"]`.
- `parts[3]` accede alla lista per indice. Python usa l'indicizzazione a base zero, quindi `parts[3]` è il *quarto* elemento.

I record NetNTLMv2 hanno l'aspetto `user::domain:challenge:hmac(32 hex):blob`. Dividiamo su `:`, poi guardiamo la parte all'indice 4 (il campo hmac): se è esattamente di 32 caratteri hex, abbiamo un v2. NetNTLMv1 ha un aspetto simile ma il campo all'indice 3 è di 48 caratteri hex (l'hash LM). Testiamo v2 *per primo* perché il campo distintivo di v2 all'indice 4 è più specifico.

Questo è il passo più disordinato dell'intera funzione. I record NetNTLM non sono stati progettati per essere eleganti — si sono evoluti dai protocolli di autenticazione Microsoft degli anni '90. Il matching sulla forma è il meglio che possiamo fare senza fare il parsing dell'intero protocollo NTLM.

```python
if _is_mysql5(text):
    return [HashCandidate(algorithm="MySQL5", confidence="high", ...)]

if _is_descrypt(text):
    return [HashCandidate(algorithm="DES crypt", confidence="medium", ...)]
```

Chiama i nostri helper. Nota che MySQL5 ottiene confidenza **HIGH** (la forma `*` + 40 hex maiuscoli è essenzialmente unica) mentre DES crypt ottiene **MEDIUM** (una stringa di 13 caratteri `./0-9A-Za-z` potrebbe plausibilmente essere altro — alcuni session ID, alcuni valori codificati). L'onestà è una funzionalità.

### 10f. Passo 3 — lookup hex + lunghezza

```python
if _is_hex(text):
    algorithms = HEX_LENGTH_RULES.get(len(text), [])
    candidates: list[HashCandidate] = []
    for index, algorithm in enumerate(algorithms):
        confidence: Confidence = "medium" if index == 0 else "low"
        label = (
            "most likely candidate at this length"
            if index == 0 else "also possible at this length"
        )
        candidates.append(
            HashCandidate(algorithm=algorithm, confidence=confidence, reason=...)
        )
    return candidates
```

Qui ci sono tre nuove funzionalità di Python:

- **`dict.get(key, default)`** restituisce il valore se la chiave esiste, oppure `default` se no. Quindi `HEX_LENGTH_RULES.get(len(text), [])` restituisce la lista di algoritmi per questa lunghezza, o una lista vuota se non esiste una regola per questa lunghezza. Questo evita un'eccezione `KeyError` che si verificherebbe con `HEX_LENGTH_RULES[len(text)]` su una lunghezza sconosciuta.
- **`enumerate(iterable)`** è un built-in che avvolge un iterabile e produce coppie `(index, value)`. Quindi `for index, algorithm in enumerate(algorithms)` percorre la lista e ci dà sia la posizione sia il valore. Il primo algoritmo ottiene l'indice 0, il secondo l'indice 1, ecc.
- **Espressione ternaria**: `value_if_true if condition else value_if_false`. Quindi `"medium" if index == 0 else "low"` vale `"medium"` per il primo elemento e `"low"` per gli altri. È l'equivalente Python di `cond ? a : b` in C/JavaScript.

Il primo algoritmo per ogni lunghezza è quello più comune nel 2026 — MD5 per la lunghezza 32, SHA-1 per la lunghezza 40, SHA-256 per la lunghezza 64. L'ordinamento della lista in `HEX_LENGTH_RULES` è per diffusione decrescente, quindi "primo" significa davvero "più probabile."

`candidates.append(item)` aggiunge un elemento alla fine della lista. `list[T]` è un tipo generico — `list[HashCandidate]` significa "una lista i cui elementi sono oggetti HashCandidate." Il letterale di lista vuota `[]` non è annotato, quindi scriviamo `candidates: list[HashCandidate] = []` per dire a mypy cosa intendiamo che la lista contenga.

### 10g. Passo 4 — fallback PHC generico

```python
if text.startswith("$"):
    rest = text[1:]
    if "$" in rest:
        algo_name = rest.split("$", 1)[0]
        if algo_name and all(c.isalnum() or c in "-_" for c in algo_name):
            return [HashCandidate(algorithm=f"PHC string ({algo_name})", ...)]
```

Se l'input inizia con `$` ma non ha trovato corrispondenza in nessuna delle nostre regole specifiche, potrebbe comunque essere una stringa PHC di un algoritmo per cui non abbiamo una regola. Proviamo a estrarre il nome dell'algoritmo e lo riportiamo come PHC generico con confidenza LOW.

- `text[1:]` taglia via il `$` iniziale.
- `rest.split("$", 1)` divide su `$` ma una sola volta — il secondo argomento opzionale di `split` è il numero massimo di divisioni. Quindi `"argon2id$v=19$...".split("$", 1)` restituisce `["argon2id", "v=19$..."]`. Senza l'`1`, dividerebbe su ogni `$` e perderemmo informazioni.
- `[0]` prende il primo elemento della lista risultante.
- `c.isalnum()` è un metodo di `str` che restituisce True se `c` è una lettera o una cifra. Lo usiamo per validare che il campo del nome dell'algoritmo contenga solo caratteri ammessi da PHC (alfanumerici più `-` e `_`). Qualsiasi cosa più strana e usciamo invece di inventare un nome di algoritmo dalla spazzatura.

L'intero passo è condizionato al fatto che il nome dell'algoritmo non sia vuoto E che ogni carattere sia ammesso. Meglio non riportare nulla che tirare a indovinare.

### 10h. Passo 5 — indizi di forma

```python
if text.startswith("eyJ"):
    return [HashCandidate(algorithm="JWT (not a hash)", ...)]

if any(c in text for c in "+/=") and len(text) > 8:
    return [HashCandidate(algorithm="Base64 blob (not a hash)", ...)]
```

Le persone incollano continuamente JWT e blob base64 negli identificatori di hash. Invece di restituire un silenzioso "nessuna corrispondenza", segnaliamo cosa probabilmente hanno incollato.

I JWT iniziano sempre con `eyJ` perché l'header JWT è un JSON come `{"alg":"HS256","typ":"JWT"}`, e i byte `{"` codificati in base64 iniziano con `eyI` o `eyJ`. Il prefisso `eyJ` è essenzialmente diagnostico per i JWT.

`any(c in text for c in "+/=")` è una generator expression dentro `any()` — speculare all'`all(...)` che abbiamo usato in `_is_hex`. `any()` restituisce True se *un qualsiasi* elemento della sequenza è True. Insieme: "c'è un qualche carattere di `'+/='` che compare in `text`?" Se sì, l'input contiene caratteri tipici solo di base64 e non può essere un hash esadecimale.

La soglia `len(text) > 8` esiste perché una stringa corta come `"a+b=c"` potrebbe far scattare accidentalmente il controllo base64. Richiediamo una lunghezza sufficiente per essere sicuri che sia davvero base64, non un'espressione matematica.

### 10i. Passo 6 — arrendersi

```python
return []
```

Lista vuota. La CLI stampa "could not identify" quando la vede. È sempre meglio ammettere la sconfitta che mentire con sicurezza.

## 11. Il livello CLI

### 11a. Il parser degli argomenti

```python
def _build_argument_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="hashid", description="...")
    parser.add_argument("hash", help="The hash string to identify ...")
    parser.add_argument("--top", "-n", type=int, default=5, help="...")
    return parser
```

`argparse.ArgumentParser` è il parser CLI della libreria standard. Impostiamo:

- `prog="hashid"` — il nome del programma mostrato nel testo di aiuto.
- `description="..."` — il riepilogo di una riga in cima a `--help`.

Poi aggiungiamo due argomenti:

- **`"hash"`** — un argomento posizionale. L'utente digita `hashid <hash>`. Obbligatorio. Dopo il parsing, `args.hash` contiene la stringa.
- **`"--top", "-n"`** — un flag opzionale con sia una forma lunga (`--top 3`) sia una forma corta (`-n 3`). `type=int` converte la stringa digitata dall'utente in un intero. `default=5` è ciò che ottieni se l'utente non lo passa.

La funzione *restituisce* il parser senza chiamare `.parse_args()`. Perché? Così il file di test può costruire il parser, ispezionarlo, eseguirlo su input di test, ecc., senza eseguire davvero la CLI. **Separare la costruzione dall'esecuzione è un pattern ricorrente per la testabilità.** Ogni volta che ti ritrovi a scrivere `something().run()` in una riga, chiediti se qualcuno ha bisogno di costruire quel qualcosa senza eseguirlo.

### 11b. Il renderer della tabella

```python
def _render_table(raw_input, candidates, console) -> None:
    table = Table(title=f"Candidates for: {raw_input.strip()}", ...)
    table.add_column("algorithm", style="bold white", no_wrap=True)
    table.add_column("confidence", no_wrap=True)
    table.add_column("reason", style="dim")

    confidence_colors: dict[Confidence, str] = {
        "high": "green",
        "medium": "yellow",
        "low": "cyan",
    }
    for candidate in candidates:
        color = confidence_colors[candidate.confidence]
        table.add_row(
            candidate.algorithm,
            f"[{color}]{candidate.confidence}[/{color}]",
            candidate.reason,
        )
    console.print(table)
```

`rich.Table` è una classe della libreria `rich`. Crei una Table, aggiungi colonne, aggiungi righe, poi la stampi. La libreria gestisce tutti i caratteri per disegnare i riquadri, i codici colore, il rilevamento della larghezza del terminale e la gestione di Unicode.

La sintassi `f"[{color}]{candidate.confidence}[/{color}]"` è il markup inline dei colori di `rich`. `[green]text[/green]` colora `text` di verde. Usiamo un dict per cercare il colore di ogni livello di confidenza — tre colori, prevedibili, facili da cambiare in un unico punto se volessimo.

Il tipo di ritorno `-> None` significa che la funzione non restituisce nulla di significativo (ha solo effetti collaterali: stampare sul terminale). Questa è l'annotazione giusta per "questa funzione fa il suo lavoro tramite effetti collaterali."

`console` viene passato come parametro invece di essere creato all'interno. Stessa idea del parser: il test può passare una Console che cattura l'output, la CLI reale passa una Console reale. Dependency injection all'opera.

### 11c. `main()` e la guardia di script

```python
def main() -> int:
    parser = _build_argument_parser()
    args = parser.parse_args()
    console = Console()

    candidates = identify(args.hash)

    if not candidates:
        console.print("[red]No identification possible.[/red] ...")
        return 1

    trimmed = candidates[:args.top]
    _render_table(args.hash, trimmed, console)

    if trimmed[0].confidence == "high":
        console.print("\n[dim]Next step: try the matching cracker ...[/dim]")

    return 0


if __name__ == "__main__":
    sys.exit(main())
```

`main()` è l'entry point. Il corpo si legge come l'inglese:

1. Costruisci il parser.
2. Fai il parsing di `sys.argv` (`parser.parse_args()` usa `sys.argv` per default).
3. Costruisci una Console.
4. Esegui il cervello sull'input dell'utente.
5. Se non ci sono candidati, stampa un errore e restituisci l'exit code 1.
6. Riduci ai primi N risultati (`candidates[:args.top]` è slicing — come prima).
7. Renderizza la tabella.
8. Se il candidato principale ha confidenza HIGH, stampa un suggerimento su cosa fare dopo.
9. Restituisci l'exit code 0.

La funzione restituisce l'exit code come intero. Lo passiamo a `sys.exit()` in fondo.

Il blocco finale:

```python
if __name__ == "__main__":
    sys.exit(main())
```

è il classico idioma degli script Python. `__name__` è una variabile speciale che Python imposta automaticamente:

- Quando esegui `python hash_identifier.py`, Python imposta `__name__ = "__main__"`.
- Quando fai `import hash_identifier` da qualche altra parte, Python imposta `__name__ = "hash_identifier"`.

Quindi `if __name__ == "__main__":` significa "fai questo solo quando il file viene eseguito direttamente, non quando viene importato." Questo permette al file di test di fare `import hash_identifier` e chiamare `identify()` senza avviare accidentalmente la CLI.

`sys.exit(N)` termina il programma con exit code N. Per convenzione l'exit code 0 significa successo; un valore diverso da zero significa fallimento. Gli script di shell (`if hashid "$x"; then ...`) leggono l'exit code per decidere cosa fare dopo.

## 12. Un esempio reale passo per passo

Tracciamo cosa succede per `just run -- '$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQNQy.uK4Of2T7G'`.

```
1. Shell invokes:    python hash_identifier.py '$2b$12$EixZ...'
2. Python runs the file. sys.argv = [".../hash_identifier.py", "$2b$12$EixZ..."]
3. Since __name__ == "__main__", call sys.exit(main()).
4. main() builds parser, calls parser.parse_args().
5. argparse sees positional arg → args.hash = "$2b$12$EixZ..."
   args.top = 5 (default).
6. main() calls identify(args.hash).

7. identify():
   text = "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQNQy.uK4Of2T7G"
   text is non-empty.
   Step 1: walk PREFIX_RULES.
     First row $argon2id$ — does text start with "$argon2id$"?  no.
     ... a few more rows ...
     Row ("$2b$", "bcrypt", ...) — does text start with "$2b$"?  YES!
     return [HashCandidate(algorithm="bcrypt", confidence="high", reason="prefix `$2b$` — bcrypt PHC string, 2b variant (current)")]

8. Back in main(): candidates is non-empty.
9. trimmed = candidates[:5] → just the one candidate.
10. _render_table(args.hash, trimmed, console)
    Build a Table with title "Candidates for: $2b$12$EixZ...".
    Add three columns.
    Build one row: ("bcrypt", "[green]high[/green]", "prefix `$2b$` — ...").
    console.print(table) — rich draws the colored ASCII table to your terminal.
11. Top candidate is HIGH confidence → print the "Next step" nudge.
12. return 0.
13. sys.exit(0) — clean exit.
```

Tempo totale trascorso: ben al di sotto di un millisecondo per il cervello, qualcuno in più perché `rich` faccia il rendering. Quasi tutto il tempo di esecuzione del programma è `rich` che disegna la tabella.

## 13. Il file di test, in breve

Apri `test_hash_identifier.py` se non l'hai ancora fatto. È strutturato come ~25 piccole funzioni `test_*`. Ognuna:

1. Costruisce un input noto.
2. Chiama `identify(input)` o `_is_mysql5(input)` ecc.
3. Fa un'asserzione sull'output (`assert candidates[0].algorithm == "bcrypt"`).

Alcuni interessanti da leggere:

- **`test_every_prefix_rule_is_recognized_with_high_confidence`** (~riga 531) — cicla su `PREFIX_RULES` e conferma che ogni riga produca una corrispondenza con confidenza HIGH. Aggiunge righe automaticamente se aggiungi nuovi prefissi.
- **`test_mysql5_rejects_lowercase_body`** (~riga 188) — conferma la regola "non mentire con sicurezza" dell'implementazione.
- **`test_hash_candidate_is_frozen`** (~riga 481) — usa `pytest.raises` per confermare che modificare una dataclass frozen sollevi `FrozenInstanceError`.

Eseguili: `just test`. L'intera suite è di ~30 test e termina in meno di un secondo.

## 14. Cosa provare dopo

Hai letto il file. Per fissare le conoscenze:

1. Prova `just run -- <hash>` con input strani — stringa vuota (dopo il quoting), sole cifre, stringhe lunghissime, hash con spazi bianchi finali, JWT, base64.
2. Apri `hash_identifier.py` e aggiungi un'istruzione `print()` dentro `identify()` per vedere quale passo trova corrispondenza per ogni input. Poi rimuovila prima di fare commit.
3. Aggiungi un nuovo prefisso a `PREFIX_RULES` — per esempio, scrypt a volte compare con il prefisso `$scrypt$`. Aggiungilo, aggiungi un test, esegui `just test`.
4. Leggi **[04-CHALLENGES.md](./04-CHALLENGES.it.md)** per idee di estensione più difficili.
