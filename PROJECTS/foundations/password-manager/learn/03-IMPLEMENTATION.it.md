# Walkthrough dell'implementazione

Apri i file sorgente in un'altra finestra e leggi insieme a questo documento. Li percorreremo in ordine di dipendenza: prima il fondo della torta a strati, per ultimo la cima. L'ordine è:

1. [`constants.py`](#1-constantspy--fonte-unica-di-verità)
2. [`crypto.py`](#2-cryptopy--argon2id--aes-256-gcm)
3. [`generator.py`](#3-generatorpy--password-casuali-sicure)
4. [`vault.py`](#4-vaultpy--formato-file-scritture-atomiche-locking)
5. [`main.py`](#5-mainpy--la-cli)
6. [`__init__.py` e `__main__.py`](#6-__init__py-e-__main__py)
7. [I test](#7-i-test)

Ogni funzionalità di Python viene spiegata la prima volta che compare. Se l'hai già vista, scorri; se non l'hai vista, la spiegazione è proprio lì.

---

## 1. `constants.py` — fonte unica di verità

Apri [`src/password_manager/constants.py`](../src/password_manager/constants.py).

### Perché esiste questo file

Nel primo progetto Python di un principiante, numeri e stringhe vengono sparsi ovunque: `length = 16`, `salt = os.urandom(16)`, `if memory > 65536:`. Sei mesi dopo, nessuno ricorda perché 16, e cambiarlo richiede di cercare in cinque file.

Mettere ogni "numero magico" in un unico file, con un nome e un commento, trasforma il resto del codice in prosa auto-documentante. Invece di `os.urandom(16)`, scriviamo `secrets.token_bytes(SALT_LENGTH_BYTES)` — e il lettore vede subito che tipo di cosa significhi 16 qui.

### In cima al file: gli import

```python
from pathlib import Path
from typing import Final
```

- **`pathlib.Path`** è il tipo di Python orientato agli oggetti per i percorsi del filesystem. Invece di `os.path.join("dir", "file.json")`, scrivi `Path("dir") / "file.json"`. L'operatore `/` è sovraccaricato per i percorsi. È più sicuro perché non c'è alcun quoting/escaping di stringhe.
- **`typing.Final`** è un type hint che marca una variabile come "non riassegnarmi mai." Viene fatto rispettare da **mypy** (il type checker), non da Python stesso. Se scrivi `X: Final[int] = 5` e poi provi `X = 6`, mypy lo segnalerà. Il runtime di Python no — Final è documentazione che il linter comprende.

### Parametri di Argon2id

```python
ARGON2_TIME_COST: Final[int] = 3
ARGON2_MEMORY_KIB: Final[int] = 65536  # 64 MiB
ARGON2_PARALLELISM: Final[int] = 4
SALT_LENGTH_BYTES: Final[int] = 16
```

I tre parametri regolabili spiegati in [01-CONCEPTS.it.md](./01-CONCEPTS.it.md). Il blocco di commenti sopra di essi spiega *perché* ogni valore è quello che è, compresa la divergenza deliberata dalla raccomandazione `parallelism=1` di OWASP, orientata ai server.

Sotto di essi ci sono tre "minimi":

```python
ARGON2_TIME_COST_MIN: Final[int] = 1
ARGON2_PARALLELISM_MIN: Final[int] = 1
ARGON2_MEMORY_KIB_PER_LANE_MIN: Final[int] = 8
```

Non sono manopole di tuning — sono i limiti algoritmici sotto i quali Argon2 stesso si rifiuta di funzionare. Li usiamo in `vault.py` per validare i parametri caricati dal disco, così un file vault corrotto o modificato a mano non può farci chiamare Argon2 con `time_cost=0` e mandarci in crash in profondità dentro la libreria con un errore confuso.

### Parametri di AES-256-GCM

```python
KEY_LENGTH_BYTES: Final[int] = 32   # AES-256 wants 32 bytes
NONCE_LENGTH_BYTES: Final[int] = 12 # GCM-recommended nonce size
```

La dimensione della chiave dice "vogliamo AES-256, non AES-128." La dimensione del nonce è la raccomandazione GCM da [NIST SP 800-38D](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38d.pdf). Non cambiare nessuno dei due.

### Formato del file vault

Il grande blocco di commenti disegna il formato su disco. Poi i nomi delle chiavi JSON vivono come costanti:

```python
VAULT_KEY_VERSION: Final[str] = "version"
VAULT_KEY_KDF: Final[str] = "kdf"
VAULT_KEY_CIPHER: Final[str] = "cipher"

KDF_KEY_NAME: Final[str] = "name"
KDF_KEY_SALT: Final[str] = "salt"
# ... etc
```

Per un progetto da principianti sembra over-engineering, ma ha un beneficio reale: se mai rinomini un campo, lo cambi una volta sola qui, e il linter ti dice subito ogni punto di chiamata da aggiornare. Confrontalo con `"version"` scritto inline in cinque file diversi — un refuso (`"verison"`) potrebbe rompere le cose in silenzio a runtime.

Le costanti dei percorsi file usano `pathlib`:

```python
DEFAULT_VAULT_DIRECTORY: Final[Path] = Path.home() / ".password-vault"
DEFAULT_VAULT_FILENAME: Final[str] = "vault.json"
DEFAULT_VAULT_PATH: Final[Path] = (
    DEFAULT_VAULT_DIRECTORY / DEFAULT_VAULT_FILENAME
)
```

`Path.home()` viene risolto al **momento dell'import**, non al momento della chiamata. Restituisce `/home/yourname` su Linux, `/Users/yourname` su macOS, `C:\Users\yourname` su Windows. Ecco perché funziona su ogni OS senza rami `if os.name == "windows"`.

### Modalità del file

```python
VAULT_FILE_MODE: Final[int] = 0o600
```

`0o600` è la sintassi di Python per il numero ottale 600. Nei permessi dei file Unix significa: il proprietario può leggere e scrivere, nessun altro può fare nulla. Il prefisso `0o` dice a Python "interpreta queste cifre come ottali." (Il semplice `600` sarebbe seicento in decimale — sbagliato.) Lo passiamo a `os.open` in modo che il file venga creato non leggibile dagli altri fin dalla primissima syscall.

### Valori predefiniti del generatore di password

```python
DEFAULT_GENERATED_PASSWORD_LENGTH: Final[int] = 24
MINIMUM_GENERATED_PASSWORD_LENGTH: Final[int] = 8
MINIMUM_MASTER_PASSWORD_LENGTH: Final[int] = 8

LOWERCASE_LETTERS: Final[str] = "abcdefghijklmnopqrstuvwxyz"
UPPERCASE_LETTERS: Final[str] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
DIGITS: Final[str] = "0123456789"
SAFE_SYMBOLS: Final[str] = "!@#$%^&*()-_=+[]{};:,.<>/?"
```

Il pool di simboli esclude deliberatamente alcuni caratteri:

- **Apici** (`'`, `"`, `` ` ``) — confondono il copia-incolla nelle shell.
- **Backslash** (`\`) — metacarattere della shell, le stringhe tra doppi apici lo interpretano.
- **Spazio** — appare invisibile quando viene mostrato.

Questa è una scelta di UX. Se vuoi un random puro con ogni carattere ASCII stampabile, li rimetteresti.

### Prompt della CLI e stringhe dei messaggi

Il resto del file è il testo rivolto all'utente:

```python
PROMPT_MASTER_PASSWORD: Final[str] = "Master password: "
MSG_VAULT_CREATED: Final[str] = "Vault created at {path}"
MSG_ENTRY_ADDED: Final[str] = "Added entry: {name}"
```

`{path}` e `{name}` sono segnaposto per `.format()`. Metterli qui significa:

1. Puoi ritoccare la formulazione in un unico punto senza cercare in `main.py`.
2. Se mai vorrai internazionalizzare lo strumento (versione spagnola, ecc.), distribuiresti un `constants.py` diverso per ogni lingua.

---

## 2. `crypto.py` — Argon2id + AES-256-GCM

Apri [`src/password_manager/crypto.py`](../src/password_manager/crypto.py).

Questo è il livello più basso del progetto. Entrano byte, escono byte. Nessun I/O su file. Nessuna istruzione `print`. Crittografia pura avvolta in funzioni Python amichevoli.

### Import

```python
import secrets
from dataclasses import dataclass

from argon2.low_level import Type, hash_secret_raw
from cryptography.exceptions import InvalidTag
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

from password_manager.constants import (
    ARGON2_MEMORY_KIB,
    ARGON2_PARALLELISM,
    ARGON2_TIME_COST,
    KEY_LENGTH_BYTES,
    NONCE_LENGTH_BYTES,
    SALT_LENGTH_BYTES,
)
```

Tre gruppi, con righe vuote in mezzo. È la convenzione standard di Python ([PEP 8](https://peps.python.org/pep-0008/#imports)):

1. **Standard library** (fornita con Python): `secrets`, `dataclasses`.
2. **Di terze parti** (installate da PyPI): `argon2-cffi`, `cryptography`.
3. **Locale** (il nostro codice): `password_manager.constants`.

`hazmat` in `cryptography.hazmat.primitives.ciphers.aead` è l'abbreviazione di "hazardous materials" (materiali pericolosi) — la libreria usa questo namespace per le primitive che richiedono attenzione per essere usate correttamente. AES-GCM vive qui perché il riuso del nonce è una trappola. La libreria è onesta su questo.

### Eccezioni personalizzate

```python
class CryptoError(Exception):
    """Base class for every cryptography error we raise."""

class WrongPasswordError(CryptoError):
    """Raised when decryption fails."""
```

Due classi, una che eredita dall'altra. Il corpo è solo una docstring — è Python valido; una classe ha bisogno di *qualcosa* nel corpo, e una docstring conta.

**Perché eccezioni personalizzate invece di un semplice `raise Exception`?** Perché il chiamante deve poter gestire errori diversi in modo diverso:

```python
try:
    vault = UnlockedVault.unlock(path, password)
except WrongPasswordError:
    print("Try again")
except VaultNotFoundError:
    print("Run `pv init` first")
```

Se sollevassimo un semplice `Exception` per tutto, il chiamante dovrebbe confrontare i messaggi delle eccezioni come stringhe — fragile e brutto. I tipi personalizzati rendono l'API auto-documentante.

### `@dataclass(frozen=True, slots=True)`

```python
@dataclass(frozen=True, slots=True)
class KdfParameters:
    time_cost: int
    memory_cost: int
    parallelism: int

    @classmethod
    def defaults(cls) -> "KdfParameters":
        return cls(
            time_cost=ARGON2_TIME_COST,
            memory_cost=ARGON2_MEMORY_KIB,
            parallelism=ARGON2_PARALLELISM,
        )
```

Una **dataclass** è una classe con i metodi `__init__`, `__repr__` e `__eq__` generati automaticamente a partire dai campi che dichiari. Senza di essa, scriveresti:

```python
class KdfParameters:
    def __init__(self, time_cost, memory_cost, parallelism):
        self.time_cost = time_cost
        self.memory_cost = memory_cost
        self.parallelism = parallelism

    def __eq__(self, other):
        return (self.time_cost == other.time_cost
                and self.memory_cost == other.memory_cost
                and self.parallelism == other.parallelism)

    def __repr__(self):
        return f"KdfParameters({self.time_cost}, {self.memory_cost}, ...)"
```

Il decoratore `@dataclass` scrive tutto questo per te. I campi con annotazioni di tipo nel corpo della classe diventano gli argomenti del costruttore.

I due flag:

- **`frozen=True`** — le istanze sono immutabili. Dopo `params = KdfParameters(3, 65536, 4)`, non puoi fare `params.time_cost = 5`. Provarci solleverà un'eccezione. Utile quando vuoi che un valore venga passato in giro come un numero o una stringa — nessuno a valle può modificarlo di nascosto.
- **`slots=True`** — risparmia memoria evitando il `__dict__` per istanza. Ottimizzazione minore. Il beneficio maggiore è che impedisce di aggiungere attributi per sbaglio: `params.typo = 5` solleva `AttributeError` invece di creare in silenzio un refuso.

**`@classmethod`**: un metodo il cui primo argomento è la *classe* stessa, non un'istanza. Per convenzione si chiama `cls`. Usato qui per creare un costruttore con un nome più amichevole: `KdfParameters.defaults()` invece di `KdfParameters(3, 65536, 4)`.

### `generate_salt` e `generate_nonce`

```python
def generate_salt() -> bytes:
    return secrets.token_bytes(SALT_LENGTH_BYTES)

def generate_nonce() -> bytes:
    return secrets.token_bytes(NONCE_LENGTH_BYTES)
```

Entrambe sono one-liner. Il punto *non* è il corpo della funzione — il punto è il nome. Avendo `generate_salt` e `generate_nonce` come funzioni con un nome, il codice chiamante è auto-documentante:

```python
nonce = generate_nonce()    # everyone knows what this is for
salt = generate_salt()
```

contro

```python
nonce = secrets.token_bytes(12)   # what's the 12? oh, must be a nonce
salt = secrets.token_bytes(16)    # what's the 16?
```

Le funzioni con nome mantengono inoltre le *dimensioni* di salt e nonce invisibili al resto del codice — le conosce solo `crypto.py`.

### `derive_key` — il passaggio lento

```python
def derive_key(
    master_password: str,
    salt: bytes,
    parameters: KdfParameters | None = None,
) -> bytes:
    if not master_password:
        raise ValueError("master_password must not be empty")

    if parameters is None:
        parameters = KdfParameters.defaults()

    password_bytes = master_password.encode("utf-8")

    return hash_secret_raw(
        secret=password_bytes,
        salt=salt,
        time_cost=parameters.time_cost,
        memory_cost=parameters.memory_cost,
        parallelism=parameters.parallelism,
        hash_len=KEY_LENGTH_BYTES,
        type=Type.ID,
    )
```

Alcune cose degne di nota:

- **`KdfParameters | None = None`** — la sintassi del type hint per "o un `KdfParameters` o `None`." Il `|` è la scrittura più breve di Python 3.10+ per `Optional[KdfParameters]` o `Union[KdfParameters, None]`. Il `= None` fa sì che l'argomento valga `None` per default quando omesso.
- **Prima il controllo sulla password vuota.** Se l'utente (o un chiamante con un bug) passa `""`, rifiutiamo subito. Argon2 girerebbe senza problemi, ma la chiave risultante sarebbe "la chiave derivata da nessun segreto + un salt pubblico" — chiunque rubi il file può rideriviarla.
- **Il controllo `is None`** per `parameters`. Potremmo scrivere `parameters: KdfParameters = KdfParameters.defaults()` come argomento predefinito, ma Python valuta gli argomenti predefiniti **una volta sola, al momento della definizione della funzione**. Per un `KdfParameters` immutabile va in realtà bene, ma il pattern "usa `None` come default, poi sostituiscilo dentro la funzione" è l'abitudine generale più sicura — perché se il default diventasse mai mutabile (come una lista), il bug del default condiviso tra le chiamate morderebbe.
- **`.encode("utf-8")`** — le stringhe in Python vivono come code point Unicode. Le funzioni crittografiche vogliono byte grezzi. UTF-8 è la codifica universalmente corretta per il testo Unicode. Specificala sempre esplicitamente; non affidarti mai al default della piattaforma.
- **`Type.ID`** sceglie specificamente Argon2id — non Argon2d né Argon2i. Vedi [01-CONCEPTS.it.md §6](./01-CONCEPTS.it.md#6-argon2id-nello-specifico-e-perché).
- **Argomenti keyword ovunque.** `secret=password_bytes`, non solo `password_bytes`. È una scelta di design dell'API di sapore sicuro: gli argomenti posizionali in una funzione con sette parametri sono un refuso in attesa di accadere. Nominarli nel punto di chiamata rende gli errori rumorosi.

### `encrypt` e `decrypt`

```python
def encrypt(plaintext: bytes, key: bytes) -> tuple[bytes, bytes]:
    cipher = AESGCM(key)
    nonce = generate_nonce()
    ciphertext = cipher.encrypt(
        nonce=nonce,
        data=plaintext,
        associated_data=None,
    )
    return nonce, ciphertext
```

Tre cose:

1. **`AESGCM(key)`** costruisce un oggetto cipher legato alla chiave. La libreria valida la dimensione della chiave — deve essere 16, 24 o 32 byte. Una chiave di dimensione errata solleva subito un errore.
2. **Un nonce nuovo a ogni chiamata.** La riga `generate_nonce()` è la riga più critica per la sicurezza dell'intero file. Riusare un nonce con la stessa chiave è catastrofico (vedi [01-CONCEPTS.it.md §10](./01-CONCEPTS.it.md#10-nonce-la-cosa-più-pericolosa-di-questa-codebase)).
3. **`associated_data=None`.** AES-GCM ha un parametro opzionale di "dati associati": dati che sono autenticati (a prova di manomissione) ma non cifrati. Utile per gli header dei pacchetti — vuoi che il destinatario rilevi le modifiche all'header, ma l'header stesso è pubblico. Non abbiamo dati di questo tipo, quindi passiamo `None`.

Il tipo di ritorno `tuple[bytes, bytes]` significa "due oggetti bytes restituiti insieme." Il chiamante fa l'unpacking come `nonce, ciphertext = encrypt(data, key)`.

```python
def decrypt(ciphertext: bytes, nonce: bytes, key: bytes) -> bytes:
    cipher = AESGCM(key)
    try:
        return cipher.decrypt(
            nonce=nonce,
            data=ciphertext,
            associated_data=None,
        )
    except InvalidTag as exc:
        raise WrongPasswordError(
            "Decryption failed: wrong master password or corrupted vault"
        ) from exc
```

`InvalidTag` è il segnale della libreria cryptography che il tag di autenticazione non corrispondeva — chiave sbagliata, nonce sbagliato o ciphertext modificato. Lo intercettiamo e lo risolleviamo come il nostro `WrongPasswordError`. Il chiamante non deve sapere che `cryptography.exceptions` esiste.

**`raise WrongPasswordError(...) from exc`** — la parte `from exc` preserva il traceback dell'eccezione originale nell'attributo `__cause__` della nuova eccezione. Se qualcosa esplode in modo inatteso, l'output di debug mostra comunque la causa `InvalidTag` sottostante. Buona pratica quando traduci un tipo di eccezione in un altro.

---

## 3. `generator.py` — password casuali sicure

Apri [`src/password_manager/generator.py`](../src/password_manager/generator.py).

Questo file è più breve di `crypto.py` e più facile da ragionare. Due funzioni pubbliche da seguire con attenzione.

### `generate_password`

```python
def generate_password(
    length: int = DEFAULT_GENERATED_PASSWORD_LENGTH,
    *,
    use_lowercase: bool = True,
    use_uppercase: bool = True,
    use_digits: bool = True,
    use_symbols: bool = True,
) -> str:
```

Il **`*`** nella firma è una funzionalità di Python chiamata **argomenti keyword-only**. Dopo il `*`, ogni argomento deve essere passato per nome. Questo:

```python
generate_password(20, use_symbols=False)    # OK
generate_password(20, True, True, True, False)  # ERROR
```

Costringe i punti di chiamata a essere leggibili. `generate_password(20, True, True, True, False)` è impossibile da capire a colpo d'occhio — dovresti contare i booleani. `generate_password(20, use_symbols=False)` è auto-documentante.

### Validare gli input

```python
if length < MINIMUM_GENERATED_PASSWORD_LENGTH:
    raise PasswordTooShortError(...)

enabled_pools = {
    "lower": LOWERCASE_LETTERS if use_lowercase else "",
    "upper": UPPERCASE_LETTERS if use_uppercase else "",
    "digit": DIGITS if use_digits else "",
    "symbol": SAFE_SYMBOLS if use_symbols else "",
}
enabled_pools = {k: v for k, v in enabled_pools.items() if v}
```

Il `if use_lowercase else ""` è una **espressione condizionale** (a volte chiamata ternaria). `<a> if <cond> else <b>` vale `<a>` quando `<cond>` è truthy, altrimenti `<b>`.

La seconda riga è una **dict comprehension**: `{k: v for k, v in something.items() if condition}` costruisce un nuovo dict a partire dagli elementi filtrati. È l'equivalente per i dict di `[x for x in list if cond]` (una list comprehension). Risultato: manteniamo solo i pool il cui flag era True (perché le stringhe vuote sono falsy in Python).

```python
if not enabled_pools:
    raise ValueError("At least one character pool must be enabled")

if length < len(enabled_pools):
    raise PasswordTooShortError(
        f"length={length} is too small to include one character "
        f"from each of {len(enabled_pools)} enabled pools"
    )
```

Se ogni pool era disabilitato, rifiutiamo. Se l'utente vuole una password più corta del numero di pool abilitati, rifiutiamo anche in quel caso (non riusciremmo a metterne uno per tipo).

### La generazione vera e propria

```python
alphabet = "".join(enabled_pools.values())

required = [secrets.choice(pool) for pool in enabled_pools.values()]
fill_count = length - len(required)
fill = [secrets.choice(alphabet) for _ in range(fill_count)]

chars = required + fill
_secure_shuffle(chars)

return "".join(chars)
```

Tre passaggi:

1. **Caratteri obbligatori.** Per ogni pool abilitato, scegli un carattere. Questo garantisce che la password finale contenga almeno uno per ogni tipo — importante per i siti che impongono regole del tipo "deve contenere una cifra" anche se il campionamento casuale senza questa garanzia sarebbe *tecnicamente* corretto.
2. **Riempi il resto.** Scegli altri `length - len(required)` caratteri dall'alfabeto combinato.
3. **Mescola.** Senza il mescolamento, i caratteri obbligatori sarebbero sempre alle posizioni 0..N-1 — un pattern indovinabile e una debolezza, per quanto piccola.

**`secrets.choice(pool)`** è la versione sicura di `random.choice(pool)`. Sceglie un elemento in modo uniformemente casuale.

**`for _ in range(fill_count)`** — il trattino basso è una convenzione di Python che significa "mi serve una variabile di ciclo ma non userò il suo valore." Come l'identificatore blank di Go.

### `_secure_shuffle`

```python
def _secure_shuffle(items: list[str]) -> None:
    for i in range(len(items) - 1, 0, -1):
        j = secrets.randbelow(i + 1)
        items[i], items[j] = items[j], items[i]
```

Questo implementa lo **shuffle di Fisher-Yates** (chiamato anche shuffle di Knuth). Produce una permutazione uniformemente casuale se la sorgente casuale è uniforme.

Perché non `random.shuffle()`? Perché `random.shuffle()` usa il Mersenne Twister, che è prevedibile. Ci serve un mescolamento che un attaccante non possa prevedere, quindi lo costruiamo da noi sopra `secrets.randbelow`.

La riga `items[i], items[j] = items[j], items[i]` è la sintassi di **tuple-swap** di Python per scambiare due valori senza una variabile temporanea. Funziona perché il lato destro viene valutato per intero prima che avvenga qualsiasi assegnazione a sinistra.

**Il trattino basso iniziale** in `_secure_shuffle`: convenzione per "questo è un helper privato del modulo, non importarmi da fuori di questo modulo." Python non lo impone; è documentazione rivolta agli umani.

---

## 4. `vault.py` — formato file, scritture atomiche, locking

Apri [`src/password_manager/vault.py`](../src/password_manager/vault.py).

Questo è il file più lungo (~1000 righe tra docstring e commenti). Ha più cose in gioco. Affrontalo a pezzi.

### `from __future__ import annotations`

Il primissimo import. È un opt-in del [PEP 563](https://peps.python.org/pep-0563/) che fa sì che i type hint vengano valutati come stringhe invece che come oggetti veri. Il beneficio: le classi possono riferirsi a se stesse nelle proprie annotazioni (`def foo(self) -> UnlockedVault` dall'interno di `UnlockedVault`) senza doverle mettere tra apici. Velocizza anche leggermente il caricamento del modulo. Comportamento predefinito in una futura versione di Python, per ora opt-in.

### Import — a cosa servono, in breve

```python
import base64           # encode raw bytes as ASCII text for JSON
import contextlib       # `contextlib.suppress` and `@contextmanager`
import json             # parse and serialize JSON
import os               # low-level filesystem ops (open/replace/fsync)
from collections.abc import Iterator   # type hint for generator return
from dataclasses import asdict, dataclass, field, replace
from datetime import datetime, UTC     # timestamps in UTC
from pathlib import Path
from types import TracebackType        # type hint for __exit__
from typing import Any, Self           # Any for opaque JSON, Self for method-return types
```

Il blocco `try`/`except ImportError` per `fcntl` è una danza di portabilità: `fcntl` esiste solo su POSIX. Su Windows, `import fcntl` solleverebbe `ImportError`. Lo intercettiamo, impostiamo la variabile a `None` e controlliamo `None` al momento della chiamata. Il commento `# pragma: no cover` dice allo strumento di coverage di saltare quella riga nei report di coverage.

### Eccezioni

Sei classi:

```python
class VaultError(Exception): pass
class VaultNotFoundError(VaultError): pass
class VaultAlreadyExistsError(VaultError): pass
class VaultFormatError(VaultError): pass
class EntryNotFoundError(VaultError): pass
class EntryAlreadyExistsError(VaultError): pass
```

`pass` è la keyword per "questo blocco non ha codice." Una classe con solo `pass` è una classe valida che eredita tutto dal genitore. La usiamo qui perché a contare sono i *nomi* — `except VaultNotFoundError:` permette a un chiamante di gestire quello specifico errore senza intercettare bug non correlati.

### Helper Base64

```python
def _b64encode(data: bytes) -> str:
    return base64.b64encode(data).decode("ascii")

def _b64decode(text: str) -> bytes:
    try:
        return base64.b64decode(text, validate=True)
    except (ValueError, TypeError) as exc:
        raise VaultFormatError(f"Invalid base64 in vault: {exc}") from exc
```

`base64.b64encode` restituisce `bytes`. Facciamo `.decode("ascii")` per ottenere una `str` perché chiavi/valori JSON devono essere stringhe.

`validate=True` è importante: per default `b64decode` ignora in silenzio i caratteri non validi. Con `validate=True`, solleva un errore su input non valido — che è ciò che vogliamo per "sto leggendo un file vault e mi aspetto base64 ben formato."

### `_validate_entry_name`

```python
def _validate_entry_name(name: str) -> None:
    if not name or not name.strip():
        raise ValueError("Entry name cannot be empty or whitespace")
    if name != name.strip():
        raise ValueError(
            "Entry name must not have leading or trailing whitespace"
        )
```

Il controllo sugli spazi iniziali/finali è quello sottile. Senza di esso, `"github"` e `"github "` sarebbero due chiavi diverse, identiche a schermo. Rifiutiamo l'ambiguità al confine.

### `_file_lock` — il context manager

```python
@contextlib.contextmanager
def _file_lock(target_path: Path) -> Iterator[None]:
    if _fcntl is None:
        yield
        return

    lock_path = target_path.with_suffix(target_path.suffix + ".lock")
    target_path.parent.mkdir(parents=True, exist_ok=True)
    fd = os.open(
        lock_path,
        os.O_RDWR | os.O_CREAT,
        VAULT_FILE_MODE,
    )
    try:
        _fcntl.flock(fd, _fcntl.LOCK_EX)
        try:
            yield
        finally:
            _fcntl.flock(fd, _fcntl.LOCK_UN)
    finally:
        os.close(fd)
```

Il decoratore `@contextlib.contextmanager` trasforma una funzione generatore in un context manager (qualcosa utilizzabile con `with`). Il pattern è:

```python
@contextlib.contextmanager
def my_thing():
    # setup
    yield resource    # <-- the `with X as resource:` line gets this
    # teardown
```

Nel nostro caso, il "setup" è "acquisisci il lock" e il "teardown" è "rilascia il lock." Lo `yield` non restituisce un valore utile — marca solo il punto in cui gira il blocco with.

**L'annidamento `try`/`finally`** è ciò che rende la pulizia a prova di bomba: anche se il codice dell'utente dentro `with` solleva un'eccezione, i blocchi `finally` vengono comunque eseguiti, il lock viene rilasciato e il file descriptor viene chiuso.

**`fcntl.LOCK_EX`** è un lock esclusivo — un solo processo alla volta può detenerlo. Se un altro processo lo ha già, ci blocchiamo (attendiamo) finché non viene rilasciato. Per uno strumento CLI monoutente, bloccarsi va bene.

### Dataclass `Entry`

```python
@dataclass(slots=True, frozen=True)
class Entry:
    username: str
    password: str
    url: str = ""
    notes: str = ""
    created_at: str = field(default_factory=_now_iso)
    updated_at: str = field(default_factory=_now_iso)
```

`frozen=True` per lo stesso motivo di `KdfParameters` — una volta costruita, una entry è immutabile.

**`field(default_factory=_now_iso)`** è il modo di dare a un campo di dataclass un valore predefinito *nuovo* a ogni nuova istanza. Se scrivessimo `created_at: str = _now_iso()`, chiameremmo `_now_iso()` una volta sola al momento della definizione della classe e riusiamo lo stesso timestamp per sempre. `default_factory=_now_iso` (nota: si passa la funzione stessa, non la si chiama) dice alla dataclass di chiamarla una volta per ogni invocazione di `Entry()`.

### `Entry.from_dict`

```python
@classmethod
def from_dict(cls, data: dict[str, Any]) -> Entry:
    try:
        username = data["username"]
        password = data["password"]
    except KeyError as exc:
        raise VaultFormatError(
            f"Entry missing required field: {exc}"
        ) from exc
    if not isinstance(username, str) or not isinstance(password, str):
        raise VaultFormatError("Entry username and password must be strings")
    return cls(
        username=username,
        password=password,
        url=data.get("url", ""),
        notes=data.get("notes", ""),
        created_at=data.get("created_at", ""),
        updated_at=data.get("updated_at", ""),
    )
```

Questo è il percorso "deserializza da JSON." I campi obbligatori usano `data["username"]` — una chiave mancante solleva `KeyError`, che intercettiamo e trasformiamo in `VaultFormatError`. I campi opzionali usano `data.get("url", "")` — `dict.get(key, default)` restituisce il default se la chiave manca.

Nota che i timestamp hanno come default `""` invece di "adesso." Se in lettura usassimo "adesso" come default, una vecchia entry che non aveva registrato il proprio orario di creazione sembrerebbe appena creata — fuorviante.

### `UnlockedVault` — la classe grossa

```python
@dataclass(slots=True)
class UnlockedVault:
    path: Path
    salt: bytes
    kdf_parameters: KdfParameters
    key: bytes
    entries: dict[str, Entry]
```

Questa dataclass **non è frozen**. Deve essere mutabile perché possiamo aggiungere/eliminare entry e ruotare la master password. Il campo `key` contiene la chiave AES da 32 byte derivata dalla master password; il campo `entries` contiene le righe di credenziali decifrate.

### `UnlockedVault.create`

```python
@classmethod
def create(
    cls,
    path: Path,
    master_password: str,
    *,
    kdf_parameters: KdfParameters | None = None,
) -> Self:
    if path.exists():
        raise VaultAlreadyExistsError(f"Vault already exists at {path}")

    salt = generate_salt()
    kdf_parameters = kdf_parameters or KdfParameters.defaults()
    key = derive_key(master_password, salt, kdf_parameters)

    vault = cls(
        path=path,
        salt=salt,
        kdf_parameters=kdf_parameters,
        key=key,
        entries={},
    )
    vault.save()
    return vault
```

**`-> Self`** è il modo di Python 3.11 di dire "questo metodo restituisce un'istanza di questa stessa classe." Utile perché i metodi ereditati mantengano i tipi corretti.

**`kdf_parameters or KdfParameters.defaults()`** usa l'`or` a corto circuito di Python. Se `kdf_parameters` è `None` (falsy), il lato destro viene eseguito e produce i default. Se è stato passato un vero `KdfParameters`, è truthy e lo usiamo.

L'argomento `kdf_parameters` è il punto di giunzione che usano i test. I test passano parametri deboli per far terminare Argon2 in millisecondi. I chiamanti in produzione passano `None` e ottengono i veri default.

### `UnlockedVault.unlock`

```python
@classmethod
def unlock(cls, path: Path, master_password: str) -> Self:
    if not path.exists():
        raise VaultNotFoundError(f"No vault at {path}")

    try:
        envelope = json.loads(path.read_text(encoding="utf-8"))
    except (json.JSONDecodeError, UnicodeDecodeError) as exc:
        raise VaultFormatError(f"...") from exc

    salt, kdf_parameters, nonce, ciphertext = _parse_envelope(envelope)

    key = derive_key(master_password, salt, kdf_parameters)
    plaintext_bytes = decrypt(ciphertext, nonce, key)

    try:
        entries_data = json.loads(plaintext_bytes.decode("utf-8"))
    except (json.JSONDecodeError, UnicodeDecodeError) as exc:
        raise VaultFormatError(f"...") from exc

    entries = {
        name: Entry.from_dict(row)
        for name, row in entries_data.items()
    }

    return cls(
        path=path,
        salt=salt,
        kdf_parameters=kdf_parameters,
        key=key,
        entries=entries,
    )
```

Leggi dall'alto al basso:

1. **Controllo che il file esista** — solleva `VaultNotFoundError` se no.
2. **Leggi + fai il parsing dell'envelope JSON.** `Path.read_text(encoding="utf-8")` legge l'intero file come stringa in una sola chiamata.
3. **`_parse_envelope`** estrae i quattro campi che ci servono, validando la versione e i nomi degli algoritmi. Restituisce una tupla, spacchettata in quattro variabili.
4. **Deriva la chiave usando il salt e i parametri presi dal file** — *non* i default di oggi. È la magia che permette ai vecchi vault di continuare a funzionare.
5. **Decifra.** `WrongPasswordError` risale lo stack.
6. **Fai il parsing del JSON decifrato** in un dict `entries_data`.
7. **Converti ogni riga** (un semplice dict) in un'istanza di `Entry` tramite la dict comprehension.

### `UnlockedVault.save`

```python
def save(self) -> None:
    entries_json = json.dumps(
        {name: entry.to_dict() for name, entry in self.entries.items()},
        sort_keys=True,
        indent=2,
    ).encode("utf-8")

    nonce, ciphertext = encrypt(entries_json, self.key)

    envelope = _build_envelope(
        salt=self.salt,
        kdf_parameters=self.kdf_parameters,
        nonce=nonce,
        ciphertext=ciphertext,
    )
    envelope_bytes = json.dumps(envelope, indent=2).encode("utf-8")

    self.path.parent.mkdir(parents=True, exist_ok=True)

    with _file_lock(self.path):
        self._atomic_write(envelope_bytes)
```

Cinque passaggi:

1. **Serializza il dict delle entry in byte JSON.** `sort_keys=True` rende l'output deterministico — comodo per fare il diff dei file cifrati.
2. **Cifra.** Un nonce nuovo viene generato dentro `encrypt()`.
3. **Costruisci il dict envelope.**
4. **Serializza l'envelope in byte.**
5. **Acquisisci il lock, esegui la scrittura atomica.** Il lock viene rilasciato automaticamente quando il blocco `with` termina.

### `UnlockedVault._atomic_write`

```python
def _atomic_write(self, envelope_bytes: bytes) -> None:
    tmp_path = self.path.with_suffix(self.path.suffix + ".tmp")

    fd = os.open(
        tmp_path,
        os.O_WRONLY | os.O_CREAT | os.O_TRUNC,
        VAULT_FILE_MODE,
    )
    try:
        try:
            os.write(fd, envelope_bytes)
            os.fsync(fd)
        finally:
            os.close(fd)

        os.replace(tmp_path, self.path)
    except BaseException:
        with contextlib.suppress(FileNotFoundError):
            os.unlink(tmp_path)
        raise

    if os.name != "nt":
        dir_fd = os.open(self.path.parent, os.O_RDONLY)
        try:
            os.fsync(dir_fd)
        finally:
            os.close(dir_fd)
```

I flag di `os.open`:

- **`os.O_WRONLY`** — apri solo in scrittura.
- **`os.O_CREAT`** — crea se manca.
- **`os.O_TRUNC`** — tronca a zero se esiste.

Il terzo argomento (`VAULT_FILE_MODE = 0o600`) è la modalità del file applicata *al momento della creazione*. Usare `os.open` invece del più ad alto livello `Path.write_bytes` ci permette di impostare la modalità nella stessa syscall, evitando la breve finestra in cui un file appena creato avrebbe altrimenti permessi più ampi (determinati dalla umask).

**`except BaseException`** — intercetta *tutto*, compresi `KeyboardInterrupt` e `SystemExit`. Usiamo qui la cattura più ampia perché vogliamo ripulire il file temporaneo qualunque cosa ci abbia fermato. Il `raise` finale risolleva l'eccezione originale.

**`contextlib.suppress(FileNotFoundError)`** — un one-liner per "ignora questa specifica eccezione se si verifica." Usato qui perché il file temporaneo potrebbe non esistere se siamo andati in crash prima di crearlo.

**L'fsync della directory** alla fine è il secondo di due fsync. Il primo (`os.fsync(fd)`) esegue il flush dei *byte* del file. Il secondo (`os.fsync(dir_fd)`) esegue il flush della *voce* della directory — senza di esso, il rename stesso può andare perso in caso di mancanza di corrente. POSIX richiede entrambi.

`os.name != "nt"` salta l'fsync della directory su Windows perché NTFS non lo supporta. Lì la durabilità è gestita dal journaling di NTFS.

### `UnlockedVault.add_entry`

```python
def add_entry(self, name: str, entry: Entry, *, force: bool = False) -> None:
    _validate_entry_name(name)
    if name in self.entries and not force:
        raise EntryAlreadyExistsError(f"Entry already exists: {name}")
    if name in self.entries:
        old = self.entries[name]
        entry = replace(entry, created_at=old.created_at, updated_at=_now_iso())
    self.entries[name] = entry
```

**`replace(entry, created_at=..., updated_at=...)`** è un helper di dataclasses che costruisce una nuova istanza con alcuni campi modificati. Poiché `Entry` è frozen, non possiamo mutarla sul posto — `replace` crea una copia nuova con le sostituzioni applicate. Questo preserva il `created_at` originale mentre aggiorna `updated_at`.

### `UnlockedVault.change_master_password`

```python
def change_master_password(
    self,
    new_master_password: str,
    *,
    kdf_parameters: KdfParameters | None = None,
) -> None:
    if not new_master_password:
        raise ValueError("new_master_password must not be empty")

    new_salt = generate_salt()
    new_kdf_parameters = kdf_parameters or KdfParameters.defaults()
    new_key = derive_key(new_master_password, new_salt, new_kdf_parameters)

    self.salt = new_salt
    self.kdf_parameters = new_kdf_parameters
    self.key = new_key
```

Nota: **questo metodo muta solo lo stato in memoria.** Non tocca il disco. Il chiamante deve chiamare `save()` dopo. Perché dividere così? Perché tenere gli effetti collaterali su livelli diversi rende i test più puliti — e perché se facessimo il salvataggio dentro, un crash a metà salvataggio lascerebbe il *file* in uno stato indefinito mentre in memoria risulterebbe "rotazione riuscita."

### `close`, `__enter__`, `__exit__`

```python
def close(self) -> None:
    self.entries = {}
    self.key = bytes(KEY_LENGTH_BYTES)

def __enter__(self) -> Self:
    return self

def __exit__(self, exc_type, exc_val, exc_tb) -> None:
    self.close()
```

`__enter__` e `__exit__` sono i metodi speciali di Python che fanno funzionare `with vault as v:`. `__enter__` viene eseguito all'inizio e il suo valore di ritorno è ciò a cui si lega `as v`. `__exit__` viene eseguito alla fine — in modo normale o tramite eccezione.

`bytes(N)` costruisce un oggetto bytes di N byte a zero. Quindi `self.key = bytes(32)` sostituisce la chiave con 32 byte a zero. *Non possiamo* azzerare i byte originali sul posto perché i bytes di Python sono immutabili, ma possiamo abbandonare il riferimento. È una cancellazione best-effort.

### `_build_envelope` e `_parse_envelope`

Sono gli helper "serializza questa dataclass nel dict envelope" e "deserializza il dict envelope in pezzi tipizzati". La parte interessante di `_parse_envelope` è la validazione:

```python
if version != VAULT_FORMAT_VERSION:
    raise VaultFormatError(f"Unsupported vault version: {version} ...")
```

Rifiutiamo i vault di una versione di formato futura. Meglio fallire in modo rumoroso che leggere un vault scritto da una versione futura dello strumento con semantiche sottilmente diverse.

La validazione dei parametri Argon2 rispetto ai minimi algoritmici è il resto della funzione. Un file vault corrotto con `time_cost=0` altrimenti andrebbe in crash in profondità dentro argon2-cffi; lo intercettiamo al confine e mostriamo un errore pulito.

---

## 5. `main.py` — la CLI

Apri [`src/password_manager/main.py`](../src/password_manager/main.py).

Questo file è il collante tra la tastiera dell'utente e il resto del codice. Usa **Typer** per il parsing degli argomenti e **Rich** per l'output colorato.

### Le basi di `Typer`

```python
app = typer.Typer(
    name="pv",
    help="Encrypted password manager (Argon2id + AES-256-GCM)",
    no_args_is_help=True,
    add_completion=False,
)
```

`typer.Typer()` crea una "app" — un registro a cui si agganciano i comandi. Poi ogni comando è una funzione decorata con `@app.command()`:

```python
@app.command()
def init(vault: VaultPath = DEFAULT_VAULT_PATH) -> None:
    """Create a new empty vault at --vault (or PV_VAULT or default path)"""
    ...
```

Il nome della funzione diventa il nome del comando. `pv init` esegue la funzione `init`. La docstring della funzione diventa il testo di `--help` per quel comando. I type hint sui parametri dicono a Typer come fare il parsing.

### `Annotated` e `typer.Option` / `typer.Argument`

```python
VaultPath = Annotated[
    Path,
    typer.Option(
        "--vault",
        "-v",
        help="Path to the vault file",
        envvar="PV_VAULT",
    ),
]
```

**`Annotated[T, metadata]`** è un modo per attaccare metadati extra a un type hint senza cambiarne il tipo sottostante. Typer legge i metadati per costruire il comportamento della CLI; tutto il resto (mypy, runtime) vede solo `Path`.

Definiamo `VaultPath` una volta sola come alias di tipo in modo che ogni comando accetti lo stesso flag `--vault` con la stessa descrizione e lo stesso supporto alla variabile d'ambiente. Un po' DRY.

`envvar="PV_VAULT"` è una comodità di Typer: se `--vault` non viene passato, Typer legge `$PV_VAULT` dall'ambiente. Se manca anche quella, entra in azione il default della funzione (`DEFAULT_VAULT_PATH`).

### Due console, due stream

```python
console = Console()
error_console = Console(stderr=True)
```

**stdout** è per il "risultato" del comando (output di successo, il pannello della password, la tabella delle entry). **stderr** è per diagnostica ed errori. Questa divisione permette agli utenti di fare redirect in modo pulito:

```bash
pv gen 32 | pbcopy           # only the password goes to clipboard
pv get foo 2>/dev/null       # swallow errors, keep the credential panel
```

Se tutto passasse da una sola console, nessuno dei due redirect funzionerebbe in modo pulito.

### `_prompt_master_password`

```python
def _prompt_master_password(prompt: str = PROMPT_MASTER_PASSWORD) -> str:
    return getpass.getpass(prompt)
```

`getpass.getpass(prompt)` legge l'input dal terminale *senza fare l'echo*. La stessa primitiva che usa `sudo`. L'utente digita la password, non vede nulla, preme Invio, e `getpass` restituisce la stringa.

La avvolgiamo in una funzione così abbiamo un solo punto da cambiare se mai vorremo una modalità non interattiva (leggere da stdin senza prompt, per gli script).

### `_unlock_or_exit`

```python
def _unlock_or_exit(path: Path, master_password: str) -> UnlockedVault:
    try:
        return UnlockedVault.unlock(path, master_password)
    except VaultNotFoundError:
        error_console.print(f"[red]{MSG_VAULT_NOT_FOUND.format(path=path)}[/red]")
        raise typer.Exit(code=1) from None
    except WrongPasswordError:
        error_console.print(f"[red]{MSG_WRONG_MASTER_PASSWORD}[/red]")
        raise typer.Exit(code=1) from None
    except VaultFormatError as exc:
        error_console.print(f"[red]Vault file is invalid: {exc}[/red]")
        raise typer.Exit(code=1) from None
    except VaultError as exc:
        error_console.print(f"[red]Vault error: {exc}[/red]")
        raise typer.Exit(code=1) from None
```

Un helper che avvolge `UnlockedVault.unlock` e trasforma ogni tipo di errore nel messaggio e nel codice di uscita giusti. Tutti i comandi CLI lo chiamano, così non devono ripetere il blocco try/except.

**`typer.Exit(code=1)`** è il modo pulito di Typer per uscire con uno stato diverso da zero. Non chiamiamo mai `sys.exit` noi stessi; Typer avvolge l'intera chiamata `app()` in qualcosa che converte `typer.Exit` in una vera uscita.

**`from None`** alla fine di `raise X from None` sopprime il traceback concatenato. Senza di esso, l'utente vedrebbe "While handling X, Y occurred" — utile per il debug ma rumoroso per "sapevamo che poteva fallire e l'abbiamo gestito in modo pulito."

Il markup `[red]...[/red]` è la sintassi dei colori di Rich — viene reso come testo rosso in un terminale che supporta i colori, e come testo semplice in uno che non li supporta.

### Un comando rappresentativo — `add`

```python
@app.command()
def add(
    name: Annotated[str, typer.Argument(help="Entry name (must be unique)")],
    vault: VaultPath = DEFAULT_VAULT_PATH,
    force: Annotated[bool, typer.Option("--force", "-f", help="Overwrite if exists")] = False,
    generate: Annotated[bool, typer.Option("--generate", "-g", help="...")] = False,
    length: Annotated[int, typer.Option("--length", "-n", help="...")] = DEFAULT_GENERATED_PASSWORD_LENGTH,
) -> None:
    """Add (or overwrite with --force) an entry in the vault"""
    master = _prompt_master_password()
    with _unlock_or_exit(vault, master) as unlocked:
        username = input(PROMPT_ENTRY_USERNAME.format(entry=name))

        if generate:
            try:
                password = generate_password(length)
            except PasswordTooShortError as exc:
                error_console.print(f"[red]{exc}[/red]")
                raise typer.Exit(code=1) from None
            console.print(f"[green]Generated password:[/green] {password}")
        else:
            password = _prompt_master_password(f"Password for {name} (hidden): ")

        url = input(PROMPT_ENTRY_URL).strip()
        notes = input(PROMPT_ENTRY_NOTES).strip()

        entry = Entry(username=username, password=password, url=url, notes=notes)

        try:
            unlocked.add_entry(name, entry, force=force)
        except EntryAlreadyExistsError:
            error_console.print(f"[red]{MSG_ENTRY_ALREADY_EXISTS.format(name=name)}[/red]")
            raise typer.Exit(code=1) from None
        except ValueError as exc:
            error_console.print(f"[red]{exc}[/red]")
            raise typer.Exit(code=1) from None

        unlocked.save()
    console.print(f"[green]{MSG_ENTRY_ADDED.format(name=name)}[/green]")
```

Il comando mostra il pattern ricorrente in ogni comando:

1. **Chiedi la master password.**
2. **Apri il blocco `with`** — `_unlock_or_exit` gestisce lo sblocco e gli eventuali errori in fase di sblocco.
3. **Fai il lavoro** — raccogli l'input, chiama `add_entry`, gestisci gli eventuali errori in fase di lavoro.
4. **`save()` dentro il blocco `with`** — ci serve la chiave per salvare.
5. **Il blocco `with` termina** — il vault cancella la propria chiave e le entry.
6. **Stampa il messaggio di successo** — dopo la cancellazione, la entry non è più in memoria, ma il messaggio di successo non ne ha bisogno.

Nota `password = _prompt_master_password(f"Password for {name} (hidden): ")`. Riusiamo lo stesso helper basato su `getpass` che chiede la master password, ma con una stringa di prompt diversa, così anche la password della *entry* viene digitata nascosta. È solo una comodità — la password della entry non è meno segreta della master, ma è una UX più piacevole non fare l'echo.

### `gen` — l'unico comando senza vault

```python
@app.command()
def gen(
    length: Annotated[int, typer.Argument(help="Password length")] = DEFAULT_GENERATED_PASSWORD_LENGTH,
    no_symbols: Annotated[bool, typer.Option("--no-symbols", help="Letters and digits only")] = False,
    no_digits: Annotated[bool, typer.Option("--no-digits", help="Letters and symbols only")] = False,
    no_uppercase: Annotated[bool, typer.Option("--no-uppercase", help="No uppercase letters")] = False,
) -> None:
    try:
        password = generate_password(
            length,
            use_lowercase=True,
            use_uppercase=not no_uppercase,
            use_digits=not no_digits,
            use_symbols=not no_symbols,
        )
    except (PasswordTooShortError, ValueError) as exc:
        error_console.print(f"[red]{exc}[/red]")
        raise typer.Exit(code=1) from None

    print(password)
```

Nota il **semplice `print()` invece di `console.print()`**. `console.print` di Rich aggiungerebbe codici di escape ANSI per il colore se il terminale li supporta. Non li vogliamo dentro una password che potrebbe finire in pipe verso `pbcopy`. Il semplice `print` è lo strumento giusto qui.

I flag sono negativi (`--no-symbols`, `--no-digits`, `--no-uppercase`) perché i default sono *attivi* — la maggior parte delle persone vuole una password forte con tutto. Dire "non voglio X" è il caso raro, che è la semantica giusta per un flag.

### `change-password`

```python
@app.command(name="change-password")
def change_password(vault: VaultPath = DEFAULT_VAULT_PATH) -> None:
    current = _prompt_master_password("Current master password: ")
    with _unlock_or_exit(vault, current) as unlocked:
        new_password = _prompt_master_password_with_confirmation()
        unlocked.change_master_password(new_password)
        unlocked.save()
    console.print(f"[green]{MSG_MASTER_PASSWORD_CHANGED.format(path=vault)}[/green]")
```

Il `name="change-password"` sovrascrive il nome generato automaticamente (`change_password` dal nome della funzione) per usare un trattino invece di un underscore. I trattini sono più convenzionali nelle CLI.

Il flusso:

1. Chiedi la master password **attuale**.
2. Sblocca con essa. Se è sbagliata, `_unlock_or_exit` esce con 1.
3. Chiedi la **nuova** master password (due volte, per conferma).
4. Chiama `change_master_password` — muta solo lo stato in memoria.
5. `save()` — cifra tutto sotto la nuova chiave e scrive in modo atomico.

Se qualcosa fallisce tra i passaggi 4 e 5, il file su disco ha ancora il vecchio salt e il vecchio ciphertext. L'utente non sta peggio di prima.

---

## 6. `__init__.py` e `__main__.py`

Due file minuscoli ma importanti.

### `__init__.py`

Questo rende `password_manager/` un **package** Python. Senza un `__init__.py` (o un `pyproject.toml` che lo dichiari come namespace package), Python non saprebbe di poter guardare dentro la cartella.

Oltre a ciò, il file ri-esporta l'API pubblica:

```python
from password_manager.crypto import (
    CryptoError,
    KdfParameters,
    WrongPasswordError,
)
from password_manager.vault import (
    Entry,
    EntryAlreadyExistsError,
    # ...
)
```

Questo permette ai chiamanti esterni (test, altri strumenti) di scrivere:

```python
from password_manager import UnlockedVault
```

invece di:

```python
from password_manager.vault import UnlockedVault
```

Il beneficio è che possiamo dividere `vault.py` in tre file più avanti (o rinominarlo) senza rompere gli import di nessuno — passano dalla porta principale del package, non dal layout interno.

```python
__version__ = "1.0.0"

__all__ = [
    "CryptoError",
    "Entry",
    # ...
]
```

**`__version__`** è il nome convenzionale per la versione del package. Gli strumenti (`pip`, i sistemi di build) possono leggerlo.

**`__all__`** è l'elenco esplicito dei nomi che `from password_manager import *` porterà dentro. È anche documentazione: "questi sono i nomi che consideriamo pubblici."

### `__main__.py`

```python
from password_manager.main import app

if __name__ == "__main__":
    app()
```

Questo file ti permette di eseguire il package direttamente:

```bash
python -m password_manager init
```

Stesso effetto di `pv init`. Utile quando lo script `pv` non è ancora nel tuo PATH.

Il controllo `if __name__ == "__main__":` è l'idioma standard di Python per "esegui questo codice solo se il file è il punto di ingresso dello script." Se qualcosa importa questo file (cosa che non farebbe, ma l'idioma è universale), la chiamata `app()` non scatta.

---

## 7. I test

Apri i file in `tests/`. Non percorreremo ogni test, indicheremo solo i pattern da cercare.

### `conftest.py`

Pytest tratta `conftest.py` come magico. Tutto ciò che è definito qui è disponibile a ogni file di test nella directory senza un import esplicito.

Qui vivono tre fixture:

- **`vault_path`** — un percorso di vault nuovo e inesistente dentro una directory temporanea. Costruito sulla fixture integrata di pytest `tmp_path`, che viene creata e ripulita automaticamente per ogni test.
- **`master_password`** — una master password di test stabile (`"correct horse battery staple"`).
- **`fresh_vault`** — un `UnlockedVault` vuoto che usa parametri Argon2 veloci.

I parametri Argon2 veloci sono il trucco chiave:

```python
TEST_KDF_PARAMETERS = KdfParameters(
    time_cost=1,
    memory_cost=8,
    parallelism=1,
)
```

Sono sotto le raccomandazioni OWASP — *deliberatamente*. Riducono il tempo di esecuzione dei test da minuti a millisecondi. La correttezza crittografica del codice è la stessa indipendentemente dalla forza dei parametri; Argon2 esegue le stesse operazioni con meno iterazioni.

Nota che non c'è alcun monkey-patching di `KdfParameters.defaults()`. Invece, il test passa esplicitamente `kdf_parameters=TEST_KDF_PARAMETERS` a `UnlockedVault.create`. Ecco perché il codice di produzione fa passare `kdf_parameters` attraverso il costruttore — così i test possono sostituirlo senza inquinare lo stato globale.

### `test_crypto.py`

I test interessanti qui:

- **Round-trip.** Cifra qualcosa, decifralo, verifica di riavere l'originale.
- **Manomissione.** Cifra qualcosa, inverti un byte nel ciphertext, verifica che `decrypt` sollevi `WrongPasswordError`.
- **Chiave sbagliata.** Cifra con la chiave A, prova a decifrare con la chiave B, verifica `WrongPasswordError`.
- **Determinismo del salt.** Stessa password + stesso salt → stessa chiave. Stessa password + salt *diverso* → chiave *diversa*.

I test di manomissione sono quelli critici per la sicurezza. Se l'autenticazione GCM smettesse mai di funzionare, quei test fallirebbero rumorosamente.

### `test_vault.py`

Test end-to-end del vault. I pattern da notare:

- **Ogni test riceve un percorso di vault nuovo.** Il `tmp_path` di pytest rende ogni test completamente isolato.
- **Test di round-trip.** Crea il vault, aggiungi una entry, salva, sblocca con la password giusta, recupera la entry.
- **Test sulle modalità di fallimento.** Password sbagliata, file mancante, JSON corrotto, ciphertext modificato, envelope modificato, campi mancanti, nome di algoritmo sbagliato.
- **Test di validazione di spazi/nomi.** `"github "` viene rifiutato. `""` viene rifiutato.
- **Test su `change_master_password`.** Ruota, salva, riapri con la nuova password, fallisci con la vecchia.

### `test_generator.py`

Test per il generatore di password:

- **Lunghezza.** Il risultato ha esattamente la lunghezza richiesta.
- **Copertura dei pool.** Il risultato contiene almeno un carattere per ogni pool abilitato.
- **Esclusione dei pool.** I pool disabilitati non compaiono mai.
- **Casi di rifiuto.** Lunghezza sotto il minimo, nessun pool abilitato, lunghezza inferiore al numero di pool.
- **Controllo di sanità sulla casualità.** Genera molte password, verifica che non siano tutte uguali. (Non è un *vero* test di casualità — non sarebbe statisticamente significativo con così pochi campioni — solo un controllo di sanità che la funzione non stia restituendo una costante.)

---

## Dove andare dopo

Hai visto ogni file. Sai cosa fa ogni funzione e perché. L'ultimo pezzo è **[04-CHALLENGES.it.md](./04-CHALLENGES.it.md)** — idee di estensione se vuoi continuare.

Dopo di che, la mossa migliore per imparare è scrivere la tua versione *senza guardare questa*. Apri un file vuoto e prova a ricostruire il progetto a memoria. I punti in cui devi andare a cercare qualcosa sono i punti che non hai ancora compreso — torna a quella sezione e rileggila.
