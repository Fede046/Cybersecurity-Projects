# Architettura

Questo file è la mappa. Alla fine dovresti essere in grado di disegnare il progetto a memoria: quale file contiene cosa, come i livelli dipendono l'uno dall'altro, cosa contiene realmente il file su disco e il flusso passo passo di ogni comando CLI.

## Indice

1. [Il layout a cinque file (e perché)](#1-il-layout-a-cinque-file-e-perché)
2. [Direzione delle dipendenze — chi importa chi](#2-direzione-delle-dipendenze--chi-importa-chi)
3. [Il formato del file vault su disco](#3-il-formato-del-file-vault-su-disco)
4. [Flusso: `pv init`](#4-flusso-pv-init)
5. [Flusso: `pv add`](#5-flusso-pv-add)
6. [Flusso: `pv get` e `pv list`](#6-flusso-pv-get-e-pv-list)
7. [Flusso: `pv change-password`](#7-flusso-pv-change-password)
8. [Flusso: `pv gen` (senza vault)](#8-flusso-pv-gen-senza-vault)
9. [Scritture atomiche + durevoli + sicure per la concorrenza, illustrate](#9-scritture-atomiche--durevoli--sicure-per-la-concorrenza-illustrate)
10. [Ciclo di vita di un `UnlockedVault`](#10-ciclo-di-vita-di-un-unlockedvault)

---

## 1. Il layout a cinque file (e perché)

```
src/password_manager/
├── __init__.py        package entry — re-exports the public API
├── __main__.py        lets `python -m password_manager` work
├── constants.py       every magic number and fixed string
├── crypto.py          Argon2id + AES-256-GCM primitives
├── generator.py       cryptographically secure password generation
├── vault.py           file format, atomic writes, locking, entry CRUD
└── main.py            CLI commands (Typer): init, add, get, list, …
```

Rispetto a `hash-identifier` (un solo file, 680 righe), questo progetto è suddiviso in cinque file sorgente (~1.400 righe in totale). La suddivisione non è decorativa — ogni file ha una ragione precisa di esistere:

| File          | Comunica con          | Non comunica con             | Compito                                                                            |
| ------------- | --------------------- | ---------------------------- | ---------------------------------------------------------------------------------- |
| `constants.py` | Nessuno               | Nulla                        | Unica fonte di verità per numeri, stringhe e parametri configurabili               |
| `crypto.py`   | solo `constants`      | Filesystem, rete, CLI        | Crittografia pura. Byte in ingresso, byte in uscita. Nessun I/O.                   |
| `generator.py`| solo `constants`      | Filesystem, rete, CLI        | Generazione di password casuali. Funzione pura, nessun I/O.                         |
| `vault.py`    | `crypto`, `constants` | Il terminale, la riga di comando | Formato del file, scritture atomiche, file locking, aggiunta/lettura/eliminazione di entry |
| `main.py`     | Tutti i precedenti    | —                            | Livello di collegamento tra la tastiera dell'utente e il resto del codice          |

**Perché questi confini contano:**

- Il file crypto non chiama *nessuna funzione di I/O*. Nessuna lettura di file, nessun `print`, nessun `input`. Questo significa che è banale da testare (basta chiamare `encrypt(b"hello", key)`) ed è impossibile introdurre al livello sbagliato un bug del tipo "stampo la chiave solo per il debug".
- Il file vault non sa nulla del terminale. Solleva eccezioni tipizzate (`VaultNotFoundError`, `WrongPasswordError`, ecc.). Il livello CLI le intercetta e le trasforma in messaggi di errore colorati. Un futuro frontend GUI o web potrebbe essere costruito su `vault.py` senza modificarlo affatto.
- Il file CLI non sa nulla di crittografia. Chiama `UnlockedVault.create(...)` e `vault.add_entry(...)` e `vault.save()`. Se l'anno prossimo sostituissimo Argon2id con qualcosa di più recente, `main.py` non cambierebbe.

Questa si chiama **architettura a livelli** (layered architecture). I livelli inferiori non importano da quelli superiori. La crittografia è in fondo; la CLI è in cima.

---

## 2. Direzione delle dipendenze — chi importa chi

```
                    ┌─────────────────────┐
                    │      main.py        │   the CLI (Typer + Rich)
                    │  (commands, glue)   │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
        ┌───────────────┐  ┌──────────┐  ┌────────────┐
        │  vault.py     │  │ crypto   │  │ generator  │
        │  (file        │  │   .py    │  │   .py      │
        │   format)     │  │          │  │            │
        └───────┬───────┘  └────┬─────┘  └─────┬──────┘
                │               │              │
                └───────┬───────┴──────┬───────┘
                        │              │
                        ▼              ▼
                  ┌───────────────────────┐
                  │     constants.py      │   no imports of our code
                  │ (numbers + strings)   │
                  └───────────────────────┘
```

**Le frecce puntano nella direzione degli import.** `main.py` importa da `vault.py`, `crypto.py`, `generator.py` e `constants.py`. `vault.py` importa da `crypto.py` e `constants.py`. Niente importa nell'altra direzione. Nessun ciclo.

Se mai vedi una freccia che punta nel verso sbagliato (ad esempio `crypto.py` che importa da `vault.py`), è un code smell — di solito significa che un pezzo di logica è finito nel livello sbagliato. Il compilatore/linter non ti fermerà, ma il design comincerà a marcire.

---

## 3. Il formato del file vault su disco

Il vault è un singolo file JSON. Per impostazione predefinita si trova in `~/.password-vault/vault.json` con permessi di file `0600` (solo proprietario).

Ecco come appare il file *approssimativamente* (i campi base64 sono abbreviati):

```json
{
  "version": 1,
  "kdf": {
    "name": "argon2id",
    "salt": "X3lkR1d2hcKLwk0PXfQpPg==",
    "time_cost": 3,
    "memory_cost": 65536,
    "parallelism": 4
  },
  "cipher": {
    "name": "aes-256-gcm",
    "nonce": "8tNTPwoq8uTXkpKt",
    "ciphertext": "Yk7eEVTSfA9wL...<lots more base64>...kw=="
  }
}
```

Qui convivono due livelli di JSON, ed è importante:

**Livello esterno (la busta):** JSON in chiaro contenente i metadati necessari a *decifrare* il livello interno. Chiunque rubi il file può leggerlo — gli dice quale KDF e quale cipher sono stati usati, il salt, il nonce. Niente di tutto ciò è segreto; la sicurezza crittografica dipende dalla *chiave*, non dal nascondere l'algoritmo.

**Livello interno (il ciphertext):** una volta decifrato, è *un altro* documento JSON — un dizionario di entry di credenziali:

```json
{
  "github": {
    "username": "alice",
    "password": "hunter2-but-better",
    "url": "https://github.com",
    "notes": "",
    "created_at": "2026-05-13T14:22:10+00:00",
    "updated_at": "2026-05-13T14:22:10+00:00"
  },
  "email": {
    "username": "alice@example.com",
    "password": "another-secret",
    "url": "",
    "notes": "personal Fastmail",
    "created_at": "2026-05-13T14:30:01+00:00",
    "updated_at": "2026-05-14T09:11:42+00:00"
  }
}
```

Quindi: **una busta JSON che avvolge JSON cifrato.** Noioso, ispezionabile, portabile. Noioso è un bene in sicurezza — meno cose personalizzate in cui sbagliare.

**Perché proprio JSON?**

- Ispezionabile da un essere umano. Puoi fare `cat vault.json` e almeno confermare la struttura. Utile per il debug.
- Banalmente portabile. Ogni linguaggio ha un parser JSON. Se mai volessi scrivere un lettore per questo formato in Rust o Go, lo avresti funzionante in un'ora.
- Compatibile in avanti. Il campo `"version": 1` permette alle versioni future di sapere come leggere i vault di oggi — e ci permette di rifiutare di leggere vault di una versione *futura* che non comprendiamo ancora.

**Perché base64?**

JSON non ha modo di rappresentare byte grezzi. La soluzione standard è base64: un modo di scrivere qualsiasi dato binario come stringa di caratteri ASCII stampabili (`A-Z`, `a-z`, `0-9`, `+`, `/`, `=`). Gonfia i dati di circa il 33% ma ci permette di far transitare i byte attraverso JSON in modo pulito. Salt, nonce e ciphertext sono tutti memorizzati codificati in base64.

---

## 4. Flusso: `pv init`

Questo crea un vault vuoto nuovo di zecca. Segui passo passo cosa succede:

```
user types: `pv init`
   │
   ▼
┌─────────────────────────────────────────────────┐
│ main.init()                                     │
│  - parse --vault flag (or env, or default path) │
│  - exists check: refuse if vault.json exists    │
│  - prompt for master password (twice, confirm)  │
│  - validate: non-empty, >= 8 chars, matches     │
└─────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────┐
│ UnlockedVault.create(path, master)              │
│  - generate fresh 16-byte salt   (secrets)      │
│  - derive 32-byte key from master + salt        │
│    via Argon2id  (~0.5s on modern laptop)       │
│  - build empty entries dict                     │
│  - call self.save()                             │
└─────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────┐
│ vault.save()                                    │
│  - serialize entries (empty {}) to JSON         │
│  - generate fresh 12-byte nonce  (secrets)      │
│  - AES-256-GCM encrypt the inner JSON           │
│  - build outer JSON envelope                    │
│  - atomic write to vault.json.tmp               │
│  - fsync data                                   │
│  - os.replace onto vault.json                   │
│  - fsync parent directory                       │
└─────────────────────────────────────────────────┘
   │
   ▼
   `Vault created at ~/.password-vault/vault.json`
```

Due cose chiave da notare:

1. **Il salt viene generato una sola volta, al momento di `create()`, e non cambia mai per tutta la vita del vault.** Anche dopo che `change-password` ricifra tutto con una nuova chiave, il salt stesso viene rigenerato solo perché la password è cambiata — per una data password, il salt è stabile.
2. **Il nonce viene generato *a ogni salvataggio*, mai riutilizzato.** Il percorso più lento (Argon2id) avviene una volta per sessione; il secondo più lento (cifratura AES-GCM) avviene a ogni salvataggio e usa ogni volta un nonce nuovo.

---

## 5. Flusso: `pv add`

Questo sblocca il vault, aggiunge una entry e la salva di nuovo. Il costo di Argon2id viene pagato *una sola volta* allo sblocco, poi `add` e `save` sono entrambi veloci.

```
user types: `pv add github`
   │
   ▼
┌─────────────────────────────────────────────────┐
│ main.add()                                      │
│  - prompt for master password                   │
└─────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────┐
│ UnlockedVault.unlock(path, master)              │
│  - read vault.json from disk                    │
│  - parse JSON envelope                          │
│  - validate version + algorithm names           │
│  - validate Argon2 parameters (sanity floors)   │
│  - extract salt, KDF params, nonce, ciphertext  │
│  - derive_key(master, salt, params)  ← slow     │
│  - AES-256-GCM decrypt(ciphertext, nonce, key)  │
│      ↓ if auth tag fails: raise                 │
│        WrongPasswordError → CLI exits with msg  │
│  - parse inner JSON → entries dict              │
│  - return UnlockedVault(path, salt, params,     │
│                          key, entries)          │
└─────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────┐
│ main.add() body, inside `with` block            │
│  - prompt for username (visible)                │
│  - if --generate: generate_password(length)     │
│    else: getpass for password (hidden)          │
│  - prompt for url, notes (optional)             │
│  - build Entry(username, password, url, notes,  │
│                created_at=now, updated_at=now)  │
│  - vault.add_entry(name, entry, force=...)      │
│      ↓ if name exists and not force:            │
│        EntryAlreadyExistsError → CLI exits      │
│      ↓ if name is empty or has whitespace:      │
│        ValueError → CLI exits                   │
│  - vault.save()  (atomic, durable, locked)      │
└─────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────┐
│ end of `with` block → vault.__exit__()          │
│  - vault.entries = {}                           │
│  - vault.key = bytes(32)  (zero-filled)         │
│ (best-effort wipe; Python bytes are immutable,  │
│  but we drop the references at minimum)         │
└─────────────────────────────────────────────────┘
   │
   ▼
   `Added entry: github`
```

Nota le **due modalità di fallimento dopo la decifratura**:

- "Password errata" ottiene un messaggio di errore.
- "Il file del vault è corrotto" ottiene lo *stesso* messaggio di errore ("Wrong master password (or vault file is corrupted)").

Questo è voluto. Il fallimento dell'autenticazione GCM significa una di tre cose e non possiamo distinguere quale: password errata, file manomesso, file corrotto. Dal punto di vista dell'utente sono indistinguibili, e *esporre* la differenza aiuta un attaccante (che saprebbe se il suo tentativo era "quasi giusto" oppure "chiave decisamente sbagliata"). Le riduciamo a un unico messaggio onesto.

---

## 6. Flusso: `pv get` e `pv list`

Entrambi seguono lo stesso schema: sblocca, leggi, renderizza, chiudi. Il vault viene sbloccato giusto il tempo di prendere i dati e viene scartato subito dopo il rendering.

```
pv get github
   │
   ├─► prompt master password
   │
   ├─► UnlockedVault.unlock(...)      (slow once, Argon2id)
   │
   ├─► entry = vault.get_entry("github")
   │         ↓ if not found:
   │           EntryNotFoundError → CLI exits 1
   │
   ├─► console.print(rich.Panel(...))    ← colored panel
   │
   └─► end of `with` → wipe key + entries
```

```
pv list
   │
   ├─► prompt master password
   │
   ├─► UnlockedVault.unlock(...)
   │
   ├─► names = vault.names()         (sorted alphabetically)
   │
   ├─► if not names: print "vault is empty" message
   │
   ├─► build a rich.Table with one row per entry
   │      columns: name, username, updated_at
   │      (passwords are NOT shown in `list`)
   │
   ├─► console.print(table)
   │
   └─► end of `with` → wipe key + entries
```

`list` mostra gli username e gli orari di aggiornamento ma **non le password**. L'utente deve chiedere esplicitamente una password con `get <name>`. È un piccolo attrito che riduce la probabilità di far trapelare un intero insieme di credenziali a chiunque stia guardando il tuo terminale.

---

## 7. Flusso: `pv change-password`

Questo è il comando più interessante dal punto di vista crittografico. Ruota la master password in questo modo:

1. Sblocca il vault con la password *vecchia* (costo completo di Argon2id).
2. Genera un *salt nuovo* e deriva una *nuova chiave* dalla *nuova* password.
3. Salva il vault — `save()` cifrerà le entry esistenti con la nuova chiave, usando un nonce nuovo.

```
pv change-password
   │
   ├─► prompt: "Current master password: "
   │
   ├─► UnlockedVault.unlock(path, current_password)
   │     ↓ if wrong: WrongPasswordError → exit 1
   │
   ├─► prompt: "New master password: " (twice, confirm)
   │     ↓ validate: non-empty, >= 8 chars, matches
   │
   ├─► vault.change_master_password(new_password)
   │     - new_salt = secrets.token_bytes(16)
   │     - new_key = derive_key(new, new_salt, defaults)
   │     - self.salt = new_salt
   │     - self.kdf_parameters = defaults()
   │     - self.key = new_key
   │     (only mutates in-memory state — disk untouched)
   │
   ├─► vault.save()
   │     - serializes the SAME entries dict (preserved)
   │     - generates a NEW nonce
   │     - encrypts under the NEW key
   │     - atomic write replaces the old file
   │
   └─► "Master password changed. Vault re-encrypted at <path>"
```

**Perché è interessante:** il file del vault memorizza i parametri KDF e il salt accanto al ciphertext. È *esattamente* per questo che questa operazione è possibile. Se i parametri KDF vivessero solo nel codice, "cambia la mia password" non avrebbe modo di "aggiornare anche i miei parametri Argon2 dai default dell'anno scorso a quelli di quest'anno". Metterli nel file rende possibile il percorso di upgrade. L'argomento `kdf_parameters` di `change_master_password` è l'aggancio per questo.

**Sicurezza in caso di crash:** se il processo muore tra la mutazione dello stato in memoria e il completamento del salvataggio atomico, il *file su disco* ha ancora il vecchio salt e il vecchio ciphertext — pienamente leggibili con la vecchia password. La nuova chiave "vince" solo dopo che `os.replace` è andato a buon fine. Questo è l'intero motivo per cui le scritture atomiche contano per i password manager: una rotazione fallita non deve mai lasciarti chiuso fuori.

---

## 8. Flusso: `pv gen` (senza vault)

Il comando più semplice. Non tocca affatto il vault. Non chiede la master password. Genera semplicemente una password casuale robusta e la stampa.

```
pv gen 32
   │
   ▼
generate_password(length=32,
                  use_lowercase=True,
                  use_uppercase=True,
                  use_digits=True,
                  use_symbols=True)
   │
   ├─► length >= MIN (8)?
   ├─► at least one pool enabled?
   ├─► length >= number of enabled pools?
   │      (need to fit one char from each)
   │
   ├─► required = [secrets.choice(pool) for pool in pools]
   │      one char guaranteed from each enabled pool
   │
   ├─► fill = [secrets.choice(combined) for _ in range(length - len(required))]
   │
   ├─► chars = required + fill
   │
   ├─► _secure_shuffle(chars)
   │      Fisher-Yates with secrets.randbelow
   │      (NOT random.shuffle — predictable)
   │
   └─► return "".join(chars)
```

L'output va su stdout tramite un semplice `print()` (non la console di rich), così è adatto alle pipe:

```bash
pv gen 32 | pbcopy
PASSWORD=$(pv gen 32)
```

Questo è l'unico comando che usa `print` invece di `console.print`. Il motivo è proprio il piping: non vogliamo i codici di escape dei colori di rich dentro la password che viene inviata in pipe a `pbcopy`.

---

## 9. Scritture atomiche + durevoli + sicure per la concorrenza, illustrate

Il metodo `save()` fa più lavoro di quanto si possa immaginare. La versione "scrivi e basta il file" sarebbe una riga; la nostra ne è qualche decina. Ecco perché ogni pezzo è lì.

### Cosa può andare storto con l'approccio ingenuo

```python
# DON'T DO THIS
path.write_bytes(envelope_bytes)
```

Questo ha tre problemi, tutti visti accadere in sistemi reali:

1. **Crash a metà scrittura → file corrotto.** Se il processo muore dopo aver scritto 4096 byte di un file da 6000 byte, il file è scritto a metà. La volta successiva che l'utente prova a sbloccare, il parsing JSON fallisce e pensa che il suo vault sia distrutto.
2. **Perdita di alimentazione → file da 0 byte (o peggio).** Anche se il processo termina, i byte vivono nella page cache del kernel. Il sistema operativo li scriverà su disco *prima o poi*, ma una perdita di alimentazione tra la write e la scrittura su disco fa sì che il file sembri esistere ma non contenga nulla.
3. **Due istanze di `pv` in competizione.** L'utente esegue `pv add github` in un terminale e `pv add email` in un altro simultaneamente. Entrambe sbloccano il vault (lento), entrambe aggiungono la propria entry, entrambe salvano. Quella che salva *per seconda* perde l'entry dell'altra — silenziosamente. Nessun errore.

### Come risolviamo ciascun problema

```
        ┌──────────────────────────────────────────────┐
        │ 1. acquire advisory flock on vault.json.lock │
        │    (POSIX systems — Windows skips this)      │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 2. os.open(vault.json.tmp, …, mode=0600)     │
        │    file created world-unreadable from the    │
        │    very first syscall (no chmod race)        │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 3. os.write(fd, envelope_bytes)              │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 4. os.fsync(fd)                              │
        │    forces kernel page cache → disk.          │
        │    without this, "we wrote it" is a story    │
        │    the page cache tells; a power loss erases │
        │    it.                                       │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 5. os.close(fd)                              │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 6. os.replace(vault.json.tmp, vault.json)    │
        │    atomic rename. after this instant,        │
        │    readers see EITHER the old file OR the    │
        │    new file. never half of either.           │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 7. fsync the parent directory (POSIX)        │
        │    so the rename itself survives power loss. │
        │    without this, an OS crash right after the │
        │    rename can revert the directory entry.    │
        └──────────────────┬───────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────────────┐
        │ 8. release advisory flock                    │
        └──────────────────────────────────────────────┘
```

Ogni passaggio tappa un buco:

| Passaggio | Tappa                                  |
| ---- | -------------------------------------- |
| 1, 8 | Due processi `pv` in competizione      |
| 2    | Breve finestra in cui il file tmp è leggibile da tutti |
| 4    | "Perdita di alimentazione subito dopo la write" fa perdere dati |
| 6    | "Crash a metà scrittura" corrompe il file attivo |
| 7    | "Perdita di alimentazione subito dopo il rename" lo ripristina |

L'advisory lock è interessante: non è imposto dal sistema operativo, ma solo dal codice che vi aderisce. Un processo che ignora `flock` può comunque scrivere il file. Ma *ogni* `save()` nel nostro codice vi aderisce, quindi due invocazioni di `pv` non possono entrare in competizione tra loro. Un editor esterno (vim, `sed`) non usa il lock, ma un utente che modifica a mano il file del vault è già uscito dal contratto del tool.

Windows non ha `fcntl.flock` né l'`fsync` sulle directory, quindi su Windows saltiamo entrambi. NTFS ci offre comunque un `os.replace` atomico. Il compromesso è: su Windows perdiamo la serializzazione tra processi (caso limite raro per un tool single-user) e perdiamo la durabilità assoluta della directory (il journaling di NTFS copre la maggior parte dei casi).

---

## 10. Ciclo di vita di un `UnlockedVault`

Un `UnlockedVault` è un oggetto Python che contiene:

- Il path del file del vault su disco.
- Il salt da 16 byte.
- I parametri Argon2 che sono stati usati.
- La chiave AES da 32 byte (sensibile!).
- Le entry decifrate (anch'esse sensibili! contengono password in chiaro).

Tenere la chiave in memoria significa che le operazioni successive (add, get, delete, save) non devono derivarla di nuovo — altrimenti pagherebbero il costo di Argon2 a ogni salvataggio. Ma significa anche che vogliamo un segnale chiaro di "ho finito con questo".

L'istruzione `with` di Python (un "context manager") è esattamente quel segnale:

```python
with UnlockedVault.unlock(path, master) as vault:
    vault.add_entry("github", entry)
    vault.save()
# at this point, vault.__exit__ has been called
# vault.entries is now {}
# vault.key is now b"\x00" * 32
```

`__enter__` viene eseguito quando il blocco inizia. `__exit__` viene eseguito quando il blocco finisce — *sia normalmente sia per eccezione*. Quindi anche se `vault.save()` solleva un'eccezione, la pulizia avviene comunque.

La pulizia stessa (`vault.close()`) sostituisce il dizionario delle entry con `{}` e la chiave con 32 byte a zero. Questa è una cancellazione **best-effort**. Gli oggetti `bytes` di Python sono immutabili, quindi i byte originali della chiave potrebbero ancora vivere in memoria finché non interviene il garbage collector. Una vera cancellazione alla liberazione in Python richiede `bytearray` più trucchi con `ctypes` che questo progetto didattico evita deliberatamente — la disciplina di "scartare esplicitamente i secret quando si è finito" è l'abitudine più importante.

```
       ┌────────────────────────────────────────────┐
       │  UnlockedVault.unlock(path, master)        │
       │  ─ slow: Argon2id derives 32-byte key      │
       │  ─ AES-GCM decrypts ciphertext             │
       │  ─ returns instance: { path, salt, params, │
       │                       key, entries }       │
       └────────────────────────────────────────────┘
                              │
                              ▼
       ┌────────────────────────────────────────────┐
       │  __enter__ → returns self                  │
       └────────────────────────────────────────────┘
                              │
                              ▼
       ┌────────────────────────────────────────────┐
       │  body of `with` block                      │
       │  ─ get_entry, add_entry, delete_entry      │
       │  ─ save() (fast: key already in memory,    │
       │            just AES-GCM + atomic write)    │
       └────────────────────────────────────────────┘
                              │
                              ▼
       ┌────────────────────────────────────────────┐
       │  __exit__ → close()                        │
       │  ─ self.entries = {}                       │
       │  ─ self.key = b"\x00" * 32                 │
       │  (best-effort wipe; Python bytes are       │
       │   immutable, GC may still hold copies)     │
       └────────────────────────────────────────────┘
```

Questo è lo stesso schema che usa `open()` di Python (`with open("x.txt") as f:`). Il vault è solo una risorsa più sensibile dal punto di vista della sicurezza rispetto a un file handle.

---

## Dove andare dopo

Ora hai in testa la forma del progetto: quale file fa cosa, com'è fatto il file su disco, cosa fa ogni comando passo per passo.

**[03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md)** percorre ogni file sorgente riga per riga. Apri `crypto.py`, `vault.py` e `main.py` in una seconda finestra e leggi insieme.
