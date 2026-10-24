# Hash Identifier

## Cos'è

Un piccolo programma Python che guarda una stringa di caratteri dall'aspetto strano e ti dice di che tipo di hash crittografico si tratta probabilmente. Gli dai qualcosa come questo:

```
5f4dcc3b5aa765d61d8327deb882cf99
```

e ti dice "quello è un hash MD5" con una motivazione del perché la pensa così.

È tutto qui il suo compito. Non effettua il crack dell'hash. Non riconverte l'hash in una password. Risponde solo alla domanda "di che tipo di hash si tratta?" — che è la domanda a cui devi rispondere *per prima* prima che qualsiasi altro tool possa esserti d'aiuto.

## Perché serve a qualcuno

La prima cosa che succede quando un attacco reale va a segno è che l'attaccante se ne va con un dump di database pieno di password hash. Gli hash sembrano senza senso, ma non sono casuali — ogni hash porta con sé indizi su come è stato prodotto. Una volta che conosci l'algoritmo (MD5, SHA-256, bcrypt, Argon2, qualunque sia) puoi passare l'hash a un tool di cracking come [hashcat](https://hashcat.net) o [John the Ripper](https://www.openwall.com/john/) e iniziare a tentare di recuperare la password originale.

Ma il punto è questo: hashcat ha bisogno che tu gli dica l'algoritmo. Ha [oltre 400 modalità hash](https://hashcat.net/wiki/doku.php?id=example_hashes), ognuna con un numero diverso. La modalità 0 è MD5. La modalità 100 è SHA-1. La modalità 3200 è bcrypt. Se scegli la modalità sbagliata, hashcat resterà lì per sempre senza trovare nulla. Quindi prima di fare il crack, identifichi. Questo è ciò che fa questo tool.

**Momenti reali in cui lo useresti:**

- Un pentester trova su un server compromesso un file di dump pieno di stringhe come `$2b$12$EixZaYVK1...` e ha bisogno di sapere cosa dare in pasto a hashcat.
- Una sfida CTF ti consegna un hash e zero indizi su quale algoritmo l'abbia prodotto.
- Stai leggendo il writeup di un breach e vuoi capire se le password trapelate erano memorizzate come MD5 veloce e non salato (un disastro) o come bcrypt lento e salato (molto meglio).
- Il [breach di LinkedIn del 2012](https://en.wikipedia.org/wiki/2012_LinkedIn_hack) ha fatto trapelare 6,5 milioni di hash SHA-1 non salati. La prima cosa che qualsiasi ricercatore doveva fare prima di *qualunque* altra cosa era confermare "sì, questi sono SHA-1." Stringhe esadecimali da quaranta caratteri. Facile. Il tool che stai per leggere glielo avrebbe detto in millisecondi.

## Cosa imparerai

**Concetti di sicurezza:**

- Cos'è davvero un hash crittografico (una funzione che trasforma qualsiasi input in un guazzabuglio di lunghezza fissa che non puoi invertire).
- I tre segnali che ogni hash lascia trapelare su se stesso: il suo **prefisso**, la sua **lunghezza** e il suo **set di caratteri**.
- Perché i moderni password hash (`$2b$...`, `$argon2id$...`) *si annunciano* di proposito, e perché i vecchi hash veloci (MD5, SHA-1) no.
- La differenza tra un hash veloce (fatto per la velocità, pessimo per le password) e un hash lento (fatto apposta per resistere al cracking).
- Perché non puoi mai recuperare la password da un hash, ma solo *indovinare* la password e verificare se il suo hash corrisponde.

**Concetti di Python (presumendo che sia la tua prima volta):**

- Come leggere un file Python dall'inizio alla fine e capire cosa sta facendo.
- Cosa fa `import` e dove finisce la libreria standard e dove iniziano i pacchetti di terze parti.
- Funzioni, type hint (`str`, `int`, `list[str]`) e cosa significa `-> bool` dopo la firma di una funzione.
- `@dataclass` — una scorciatoia per creare piccoli oggetti tipo record.
- `frozenset`, `dict`, `list`, `tuple` — i container principali di Python e quando scegliere quale.
- Come un tool a riga di comando inizia davvero a girare (la riga `if __name__ == "__main__"` in fondo).
- Come funziona un file di test e perché ogni funzione del codice principale ha i test accanto.

**Tool che toccherai:**

- [`uv`](https://github.com/astral-sh/uv) — il moderno package manager di Python. Come `pip` ma ~100× più veloce.
- [`just`](https://github.com/casey/just) — un command runner. Invece di memorizzare comandi lunghi, digiti `just test` o `just run`.
- [`rich`](https://github.com/Textualize/rich) — la libreria che stampa alla fine la graziosa tabella colorata.
- [`pytest`](https://pytest.org) — il test runner di Python.
- [`ruff`](https://github.com/astral-sh/ruff) + [`mypy`](https://mypy-lang.org) + [`pylint`](https://pylint.org) — i linter che ti urlano contro se il tuo codice è sbagliato, lento o trasandato.

## Cosa ti serve prima di iniziare

**Conoscenze che dovresti avere:**

- Hai usato un terminale almeno una volta (sai cosa fanno `cd` e `ls`).
- Sai vagamente che "un hash" è una funzione a senso unico. Se no, [01-CONCEPTS.md](./01-CONCEPTS.it.md) ti ci porterà in 10 minuti.
- Sai leggere il codice, o almeno sei disposto a farlo. Spiegheremo ogni funzionalità di Python man mano che la incontriamo.

**Conoscenze che NON ti servono:**

- Alcuna esperienza pregressa con Python. Il senso del tier **foundations** è proprio che si parte da qui.
- Alcuna esperienza pregressa di cybersecurity.
- Alcuna matematica oltre il "contare." In questo progetto non c'è matematica. La crittografia usa la matematica sotto il cofano, ma identificare un hash dalla sua forma no.

**Software che ti serve installato:**

- Python 3.14 o più recente.
- Il tool `uv` (lo script di installazione lo recupererà per te se non ce l'hai).
- Il tool `just` (anch'esso gestito dallo script di installazione).
- Un terminale. Qualsiasi terminale. Su Mac è Terminal.app o iTerm2; su Linux è quello che la tua distro ha incluso; su Windows è WSL2 + Ubuntu (consigliamo caldamente WSL2 invece di Windows nativo).

*Non* ti serve un IDE — un editor di testo va bene. Consigliamo [VS Code](https://code.visualstudio.com) con l'estensione Python, ma `nano`, `vim`, `helix` o qualunque cosa già usi andranno bene.

## Avvio rapido

Dall'interno di `PROJECTS/foundations/hash-identifier/`:

```bash
./install.sh
```

Quello script installerà `uv` e `just` se mancano, creerà un virtual environment (una sandbox Python isolata solo per questo progetto), installerà tutte le dipendenze e verificherà che i test passino. Stampa cosa sta facendo man mano che procede — leggi l'output, non limitarti a chiudere il terminale.

Poi prova il tool:

```bash
just run -- 5f4dcc3b5aa765d61d8327deb882cf99
```

Dovresti vedere una tabella colorata che identifica quella stringa come MD5 (con NTLM, MD4 e RIPEMD-128 come alternative meno probabili — tutte e quattro producono 32 caratteri esadecimali, quindi la sola lunghezza non può distinguerle).

Provane qualcun altro:

```bash
# bcrypt — modern password hash, announces itself with the $2b$ prefix
just run -- '$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQNQy.uK4Of2T7G'

# SHA-256 — 64 hex characters
just run -- e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

# A JWT (this is NOT a hash, but the tool will say so politely)
just run -- eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U

# Total garbage — the tool will say "no idea" rather than guess
just run -- helloworld
```

**Nota sul quoting:** quando un hash inizia con `$`, devi racchiuderlo tra apici singoli (`'$2b$...'`). Senza apici, la tua shell proverà a espandere `$2` come una variabile di shell e spezzerà l'hash. Questa è una cosa della shell, non di Python — ogni shell Unix lo fa.

## Struttura del progetto

```
hash-identifier/
├── hash_identifier.py        the whole tool — one file, ~680 lines
├── test_hash_identifier.py   tests for every behavior the tool claims to have
├── install.sh                one-shot setup script
├── justfile                  shortcuts for run / test / lint / format
├── pyproject.toml            project config: dependencies, linter rules, etc.
├── README.md                 short pointer to this learn/ folder
├── learn/                    you are here
│   ├── 00-OVERVIEW.md        quick start (this file)
│   ├── 01-CONCEPTS.md        what hashes are and how identification works
│   ├── 02-ARCHITECTURE.md    how the code is structured, with diagrams
│   ├── 03-IMPLEMENTATION.md  line-by-line walkthrough of the code
│   └── 04-CHALLENGES.md      extension ideas if you want to go further
└── assets/                   images, screenshots
```

Un solo file di codice è una scelta deliberata. Il tier foundations è pensato per essere leggibile in una sola sessione. I tier intermediate e advanced si suddividono in molti file; foundations no.

## Dove andare dopo

1. **[01-CONCEPTS.md](./01-CONCEPTS.it.md)** — capisci *cos'è* un hash, *perché* l'identificazione è la prima mossa e *come* funzionano davvero gli indizi di prefisso/lunghezza/charset. Leggilo anche se pensi di saperlo già; l'inquadramento conta.
2. **[02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md)** — osserva la pipeline a sei passi che il tool usa per prendere una decisione, disegnata come diagramma di flusso.
3. **[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md)** — leggi `hash_identifier.py` insieme a noi, riga per riga. Ogni funzionalità di Python viene spiegata quando compare per la prima volta.
4. **[04-CHALLENGES.md](./04-CHALLENGES.it.md)** — estensioni che puoi provare da solo una volta assorbito il resto.

## Problemi comuni

**"command not found: just"**
Lo script di installazione dovrebbe configurarlo, ma se non l'ha fatto: `curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash`. Poi chiudi e riapri il terminale così vede il nuovo tool.

**"command not found: uv"**
Stessa idea: `curl -LsSf https://astral.sh/uv/install.sh | sh`, poi riapri il terminale.

**`just run -- $2b$12$...` spezza l'hash**
Hai dimenticato gli apici singoli attorno all'hash. Rilancialo con `just run -- '$2b$12$...'`.

**"ModuleNotFoundError: No module named 'rich'"**
Hai eseguito `python hash_identifier.py` direttamente invece di `just run`. La recipe `just run` usa il virtual environment che ha `rich` installato. Usa `just run`, oppure attiva prima il venv: `source .venv/bin/activate`, *poi* `python hash_identifier.py <hash>`.

**I test falliscono subito dopo l'installazione**
I test dovrebbero passare su un `./install.sh` appena fatto. Se non lo fanno, probabilmente hai un Python più vecchio (esegui `python --version`; ti serve 3.14+). Su Ubuntu, installalo tramite [`deadsnakes`](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa); su Mac usa [Homebrew](https://brew.sh).
