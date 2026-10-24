# Password Vault

## Cos'è

Un piccolo password manager da riga di comando scritto in Python. Digiti una volta la master password, e lo strumento conserva tutte le altre password che gli dai all'interno di un unico file cifrato. In seguito, chiedi allo strumento "github" o "email" o "bank" e lui ti restituisce la password.

L'intero progetto conta circa 1.400 righe di codice distribuite in cinque file. Nessun web server, nessuna estensione del browser, nessun account cloud. Un file su disco, una master password nella tua testa.

```
$ pv init
New master password: ************
Confirm master password: ************
Vault created at /home/you/.password-vault/vault.json

$ pv add github
Username for github: alice
Password for github (hidden): ************
URL (optional, press Enter to skip): https://github.com
Notes (optional, press Enter to skip):
Added entry: github

$ pv get github
╭────────────────── github ──────────────────╮
│ username   alice                           │
│ password   hunter2-but-better              │
│ url        https://github.com              │
│ created    2026-05-13T14:22:10+00:00       │
│ updated    2026-05-13T14:22:10+00:00       │
╰────────────────────────────────────────────╯
```

Questo è l'intero strumento. Ha anche `list`, `delete`, `gen` (genera una password casuale robusta) e `change-password` (ruota la master password).

## Perché serve a tutti

La risposta onesta è "serve a te". Ogni sito richiede una password. Ogni password dovrebbe essere diversa. Nessuno riesce a ricordare 200 password diverse, quindi le persone ne riutilizzano una — e nel momento in cui un singolo sito subisce una violazione, l'attaccante prova quella stessa password ovunque. Questo si chiama **credential stuffing** ed è il modo in cui avviene davvero la maggior parte dei furti di account nel 2025.

Un password manager risolve il problema dandoti esattamente una password da ricordare (la master) mentre lui ricorda tutte le altre. L'obiettivo "ogni sito diverso" diventa raggiungibile perché non devi più tenerle tutte a mente.

**Momenti reali in cui le scelte di sicurezza di questo progetto contano:**

- La [violazione di Adobe del 2013](https://en.wikipedia.org/wiki/2013_Adobe_breach) ha fatto trapelare 153 milioni di password. Adobe le aveva cifrate con una singola chiave in modalità ECB — nessun salt, nessuna casualità per record — quindi password identiche producevano ciphertext identici. I ricercatori potevano vedere cluster di utenti che avevano tutti scelto `password123` semplicemente guardando il file cifrato. L'uso in questo progetto di un nonce casuale nuovo a ogni salvataggio e di Argon2id con un salt per vault è la correzione diretta a quella classe di errori.
- La [violazione di LastPass del 2022](https://blog.lastpass.com/posts/notice-of-recent-security-incident) ha fatto trapelare backup cifrati dei vault. I vault con master password deboli sono stati craccati offline su larga scala perché l'attaccante aveva tempo illimitato. La difesa — rendere costoso ogni tentativo — è esattamente ciò che ci dà Argon2id. I nostri valori predefiniti portano ogni tentativo a circa mezzo secondo su un laptop moderno, il che rende un attacco da un miliardo di tentativi un'impresa da ~15 anni sulla macchina dell'attaccante.
- Ogni sfida CTF o ogni ingaggio di pentest in cui qualcuno ti consegna "un file cifrato" e chiede "è sicuro a riposo?" — alla fine di questo progetto saprai quali domande fare (quale KDF? quale cipher mode? com'è la situazione del salt? il tag di autenticazione viene verificato?).

## Cosa imparerai

**Idee di sicurezza:**

- Cos'è la **cifratura simmetrica** — una chiave cifra e decifra. Usiamo [AES-256-GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode), lo stesso algoritmo che protegge la connessione HTTPS della tua banca.
- Cosa fa una **key derivation function (KDF)** — trasforma una password umana (corta, debole, prevedibile) in una vera chiave crittografica (32 byte di pura imprevedibilità). Usiamo [Argon2id](https://en.wikipedia.org/wiki/Argon2), il vincitore della Password Hashing Competition del 2015 e l'algoritmo che OWASP raccomanda attualmente per la memorizzazione delle password.
- Perché **salt** e **nonce** sono cose diverse anche se entrambi sono "byte casuali che conservi accanto al ciphertext".
- Cosa significa **cifratura autenticata** e perché "semplicemente cifrato" non basta — senza autenticazione, un attaccante può invertire dei bit nel tuo file in modi prevedibili senza conoscere la chiave.
- La differenza tra `random` (va bene per il lancio di un dado) e `secrets` (l'unica cosa che dovresti usare quando un attaccante vuole prevedere l'output).
- Perché **memorizziamo i parametri del KDF nel file** invece di scriverli semplicemente nel codice, e come questa decisione permetta a `change-password` di aggiornare vecchi vault ai nuovi default anni dopo.
- Perché scrivere un file in modo "atomico" (scrivi su `.tmp`, fsync, rename) conta quando un'interruzione di corrente potrebbe altrimenti lasciarti con zero password.

**Idee su Python (supponendo che sia la tua prima volta):**

- Cos'è un **package** e cosa fa `__init__.py` per esso.
- I **moduli** e come `import` trova effettivamente i file.
- I **type hint** — `str`, `int`, `bytes`, `list[str]`, `dict[str, Entry]`, `Path | None`. Non vengono applicati a runtime, ma sono la documentazione più utile del linguaggio.
- **`@dataclass`** — la scorciatoia per creare classi simili a record senza scrivere `__init__` a mano.
- Il tipo **`Final`** — per dire a Python "questo valore è una costante, non riassegnarlo mai".
- I **context manager** (`with vault.unlock(...) as v:`) — il modo più pulito per dire "predisponi qualcosa, usalo, smontalo anche in caso di errori".
- **Eccezioni e classi di eccezione personalizzate** — definire i propri tipi di errore e intercettarli per categoria.
- **Generator**, **dict comprehension**, **f-string** — idiomi moderni di Python che vedrai ovunque.
- Come funziona `pytest` e perché `conftest.py` è speciale.
- Come funziona una CLI costruita con [Typer](https://typer.tiangolo.com) — trasformare una funzione in un comando semplicemente scrivendo i suoi type hint.

**Strumenti che toccherai:**

- [`uv`](https://github.com/astral-sh/uv) — il moderno gestore di pacchetti Python. Come `pip` ma ~100× più veloce.
- [`just`](https://github.com/casey/just) — un command runner. Invece di memorizzare comandi lunghi, digiti `just test` o `just run`.
- [`typer`](https://typer.tiangolo.com) — il framework CLI.
- [`rich`](https://github.com/Textualize/rich) — la libreria che stampa i bei pannelli e le tabelle colorati.
- [`argon2-cffi`](https://github.com/hynek/argon2-cffi) — il binding Python per l'implementazione di riferimento di Argon2.
- [`cryptography`](https://cryptography.io) — la libreria della Python Cryptographic Authority. Lo standard di riferimento.
- [`pytest`](https://pytest.org) + [`ruff`](https://github.com/astral-sh/ruff) + [`mypy`](https://mypy-lang.org) + [`pylint`](https://pylint.org) — testing e linting.

## Cosa ti serve prima di iniziare

**Conoscenze che dovresti avere:**

- Hai usato un terminale almeno una volta (sai cosa fanno `cd` e `ls`).
- Hai almeno visto le parole "hash" e "cifratura" prima d'ora. Se per te non significano nulla, [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) è pensato per partire da zero — leggilo prima del codice.
- Sai leggere codice, o sei disposto a provarci. Ogni funzionalità di Python viene spiegata la prima volta che compare.

**Conoscenze che NON ti servono:**

- Esperienza pregressa con Python. Il punto centrale del livello **foundations** è che si parte da qui. Questo progetto è però il *più difficile* dei tre progetti foundations — se qualcosa qui ti sembra troppo denso, prova prima [hash-identifier](../../hash-identifier/).
- Alcuna conoscenza pregressa di crittografia. Imparerai cosa sono un KDF, un nonce e un tag di autenticazione in [01-CONCEPTS.it.md](./01-CONCEPTS.it.md). Non serve altra matematica oltre al saper contare — la matematica sta dentro le librerie che chiamiamo.
- Alcun background pregresso di cybersecurity.

**Software da installare:**

- Python 3.13 o più recente (consigliato 3.14).
- Lo strumento `uv` (lo script di installazione lo installerà per te se non ce l'hai).
- Lo strumento `just` (gestito anch'esso dallo script di installazione).
- Un terminale. Mac: Terminal.app o iTerm2. Linux: quello fornito dalla tua distro. Windows: WSL2 + Ubuntu (fortemente consigliato rispetto a Windows nativo — i percorsi di codice per il file locking e fsync sono di stampo POSIX).

Non ti serve un IDE — va bene qualsiasi editor di testo. [VS Code](https://code.visualstudio.com) con l'estensione Python è un buon default.

## Avvio rapido

Dall'interno di `PROJECTS/foundations/password-manager/`:

```bash
./install.sh
```

Quello script installa `uv` e `just` se mancano, crea un ambiente virtuale (una sandbox Python isolata solo per questo progetto), installa tutte le dipendenze ed esegue la suite di test per confermare che tutto funzioni. Leggi l'output mentre scorre — non chiudere semplicemente il terminale.

Poi crea il tuo primo vault:

```bash
just run -- init
```

Ti chiederà una master password, ti chiederà di digitarla di nuovo per conferma e creerà un vault vuoto in `~/.password-vault/vault.json`. Il primo `init` è lento di proposito — è Argon2id che fa il suo lavoro, impiegando deliberatamente circa mezzo secondo per derivare la chiave. Un attaccante che ruba il tuo file vault deve pagare lo stesso costo di mezzo secondo per ogni password che vuole provare.

Aggiungi una entry:

```bash
just run -- add github
```

Ti chiederà lo username, la password (input nascosto) e, opzionalmente, URL e note.

Guarda la entry:

```bash
just run -- get github
```

Vedrai un pannello colorato con ogni campo. La password viene mostrata in chiaro — questo è uno strumento CLI locale, l'utente si fida già del proprio schermo.

Genera una password casuale robusta senza toccare il vault:

```bash
just run -- gen 32
```

Questo stampa una password di 32 caratteri su stdout (nient'altro). Puoi inviarla direttamente alla clipboard con una pipe:

```bash
just run -- gen 32 | pbcopy        # macOS
just run -- gen 32 | xclip -sel c  # Linux
```

Elenca il nome di ogni entry:

```bash
just run -- list
```

Cambia la tua master password (il vault viene ricifrato con una nuova chiave):

```bash
just run -- change-password
```

Elimina una entry:

```bash
just run -- delete github
```

## Struttura del progetto

```
password-manager/
├── src/password_manager/
│   ├── __init__.py        package metadata + re-exports
│   ├── __main__.py        lets `python -m password_manager` work
│   ├── constants.py       every magic number and fixed string
│   ├── crypto.py          Argon2id + AES-256-GCM primitives
│   ├── generator.py       cryptographically secure random passwords
│   ├── vault.py           file format, atomic writes, locking
│   └── main.py            the CLI commands (init, add, get, …)
├── tests/
│   ├── conftest.py        shared pytest fixtures
│   ├── test_crypto.py     round-trip + tamper tests
│   ├── test_generator.py  pool/length/randomness tests
│   └── test_vault.py      end-to-end vault tests
├── install.sh             one-shot setup
├── justfile               shortcuts for run / test / lint / format
├── pyproject.toml         project config + dependencies + linter rules
├── README.md              short pointer to this folder
├── learn/                 you are here
│   ├── 00-OVERVIEW.md     quick start (this file)
│   ├── 01-CONCEPTS.md     KDFs, AES-GCM, salts, nonces, real breaches
│   ├── 02-ARCHITECTURE.md module layout, data flows, file format
│   ├── 03-IMPLEMENTATION.md line-by-line walkthrough
│   └── 04-CHALLENGES.md   extensions if you want to keep going
└── assets/                images, screenshots
```

La suddivisione in cinque file sorgente è l'unico punto in cui questo progetto si allontana dalla semplicità "single-file" di `hash-identifier`. La crittografia richiede confini rigorosi: il file che parla con `secrets.token_bytes` è un file diverso da quello che parla con il tuo filesystem, ed *entrambi* sono diversi dal file che stampa i pannelli colorati. Se mai si insinua un bug, il piccolo file in cui risiede è il primo posto in cui guardare.

## Dove andare dopo

1. **[01-CONCEPTS.it.md](./01-CONCEPTS.it.md)** — le idee di sicurezza. Cos'è un KDF, perché usiamo Argon2id, cosa fa davvero AES-GCM per noi, perché i nonce contano, cosa significa cifratura autenticata. Leggilo prima del codice, anche se pensi di saperlo già. L'inquadramento conta più delle parole.
2. **[02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md)** — come il codice è organizzato in moduli, com'è fatto il file vault su disco e il flusso passo-passo di `init` / `add` / `get` / `change-password`. Con diagrammi per ciascuno.
3. **[03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md)** — leggi con noi ogni file sorgente, in ordine, con ogni funzionalità di Python spiegata la prima volta che compare.
4. **[04-CHALLENGES.it.md](./04-CHALLENGES.it.md)** — idee di estensione (ricerca, export, TOTP, percorso di upgrade del key-stretching) una volta assorbito il resto.

## Problemi comuni

**"command not found: just"**
Lo script di installazione dovrebbe configurarlo, ma se non l'ha fatto: `curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash`. Poi chiudi e riapri il terminale.

**"command not found: uv"**
Stessa idea: `curl -LsSf https://astral.sh/uv/install.sh | sh`, poi riapri il terminale.

**"`pv` è lento su `init`"**
È il punto. Argon2id impiega deliberatamente circa mezzo secondo per chiamata con i valori predefiniti. L'utente lo paga una volta per sessione; un attaccante lo paga a ogni tentativo. Se è dolorosamente lento (diversi secondi), la tua CPU è un po' datata — `constants.py` ha le tre manopole di tuning di Argon2 che puoi abbassare.

**"Wrong master password (or vault file is corrupted)"**
Lo strumento non può distinguere questi due casi — di proposito, vedi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md). Se sei sicuro di aver digitato correttamente la password, prova `ls -la ~/.password-vault/` e controlla che il file `vault.json` abbia dimensione diversa da zero.

**"ModuleNotFoundError: No module named 'argon2'"**
Hai eseguito `python src/password_manager/main.py` direttamente invece di passare da `just run`. La recipe `just run` usa l'ambiente virtuale che ha tutte le dipendenze installate. Usa `just run`, oppure attiva prima il venv: `source .venv/bin/activate`, poi esegui `pv` o `python -m password_manager`.

**I test falliscono subito dopo l'installazione**
I test dovrebbero passare con un `./install.sh` appena eseguito. Se non passano, controlla `python --version` — ti serve la 3.13+. Su Ubuntu, installa un Python più recente tramite [`deadsnakes`](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa); su Mac usa [Homebrew](https://brew.sh).

**"Ho dimenticato la mia master password"**
Non esiste alcun recupero. È voluto — se ci fosse un modo di recuperare la password, lo avrebbe anche chiunque rubi il file. È lo stesso compromesso che fa ogni vero password manager. Scegli qualcosa di memorabile, annota un suggerimento (NON la password) in un luogo fisico sicuro, e usa `change-password` per ruotarla ogni tanto.
