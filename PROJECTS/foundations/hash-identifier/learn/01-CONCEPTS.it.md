# Concetti

Questa pagina costruisce le idee che ti servono per capire il codice. Partiamo da "cos'è un hash" e arriviamo a "ecco esattamente perché una stringa esadecimale di 32 caratteri è probabilmente MD5." Non è richiesta alcuna conoscenza pregressa di sicurezza.

## 1. Cos'è un hash?

Una **funzione di hash crittografico** prende qualsiasi input — un singolo byte, una password, un file video da 4GB — e produce un output di lunghezza fissa che sembra spazzatura casuale. Lo stesso input dà sempre lo stesso output. Input diversi danno output diversi. E, cosa fondamentale: se hai solo l'output, non puoi tornare indietro fino all'input.

Immaginalo come un frullatore da cucina che va solo in avanti:

```
"password"  ─────► [ MD5 blender ] ─────►  5f4dcc3b5aa765d61d8327deb882cf99
"hello"     ─────► [ MD5 blender ] ─────►  5d41402abc4b2a76b9719d911017c592
"hello!"    ─────► [ MD5 blender ] ─────►  d9014c4624844aa5bac314773d6b689a
                          │
                          └─ change ONE character → totally different output
```

Alcune proprietà che derivano da questo:

- **Deterministica.** Stesso input, stesso output, ogni singola volta. Questo è l'intero motivo per cui gli hash sono utili — puoi memorizzare l'hash di una password, e quando qualcuno fa il login ri-calcoli l'hash di quello che ha digitato e lo confronti.
- **Output di lunghezza fissa.** Che tu faccia l'hash di `"a"` o dell'intera Bibbia, MD5 produce sempre 32 caratteri esadecimali. SHA-256 ne produce sempre 64. Questa lunghezza è il primo grande indizio che usiamo per identificare quale algoritmo è stato usato.
- **A senso unico.** Puoi andare da password → hash, ma non da hash → password. Non c'è un pulsante "un-hash". L'unico modo per capire quale password ha prodotto un hash è *indovinare le password e farne l'hash da solo* finché non ottieni una corrispondenza. Quel tentativo di indovinare è ciò che fa hashcat.
- **Effetto valanga.** Cambia una lettera dell'input e l'intero output cambia. `"password"` e `"Password"` producono hash che non condividono nemmeno un carattere.

Se vuoi giocarci da solo, in una REPL Python (`uv run python`):

```python
>>> import hashlib
>>> hashlib.md5(b"password").hexdigest()
'5f4dcc3b5aa765d61d8327deb882cf99'
>>> hashlib.md5(b"Password").hexdigest()
'dc647eb65e6711e155375218212b3964'
```

La sintassi `b"..."` significa "questi sono byte, non testo" — le funzioni di hash lavorano su byte grezzi, non su caratteri. Non preoccuparti ancora di questa distinzione; nota solo che il cambio di una lettera ha prodotto un hash completamente diverso.

## 2. Perché gli hash esistono (il problema dello storage delle password)

Immagina di gestire un sito web. Gli utenti si registrano con una password. La cosa ingenua da fare è memorizzare le password direttamente nel tuo database:

```
+----------+------------+
| username | password   |
+----------+------------+
| alice    | hunter2    |
| bob      | letmein    |
+----------+------------+
```

Questa è una catastrofe annunciata. Nel momento in cui qualcuno viola il tuo database — e prima o poi qualcuno lo fa sempre — ogni utente è esposto. Peggio ancora, dato che le persone [riusano le password tra i vari siti](https://www.security.org/digital-safety/password-reuse-statistics/), l'attaccante ora ha le chiavi della banca di Alice, dell'email di Alice e del Netflix di Alice.

La soluzione è non memorizzare mai la password stessa. Memorizzare il suo hash:

```
+----------+----------------------------------+
| username | password_hash                    |
+----------+----------------------------------+
| alice    | 5f4dcc3b5aa765d61d8327deb882cf99 |  ← MD5("hunter2")... if it were
| bob      | 0d107d09f5bbe40cade3de5c71e9e9b7 |     "password" (it isn't)
+----------+----------------------------------+
```

Quando Alice fa il login, fai l'hash di quello che ha appena digitato e lo confronti con l'hash memorizzato. Se corrispondono, entra. Non hai mai conosciuto la sua password e non devi mai farlo.

L'attaccante che ruba questo database ora ha hash, non password. Deve *indovinare* ogni password facendo l'hash di password candidate e confrontandole. Con un hash veloce come MD5, una GPU moderna può provare miliardi di tentativi al secondo. Con un hash lento come bcrypt, può provarne solo migliaia. Questo è l'intero motivo per cui i sistemi moderni usano hash lenti — non perché siano "più sicuri" in qualche modo astratto, ma perché rendono il tentativo di indovinare *costoso*.

## 3. Breach reali che sono ruotati attorno all'identificazione dell'hash

Identificare il formato dell'hash è la *prima mossa* in ogni storia di password trapelate. Finché non sai quale algoritmo ha prodotto gli hash, non succede nient'altro.

**[Breach di LinkedIn del 2012](https://en.wikipedia.org/wiki/2012_LinkedIn_hack)** — 6,5 milioni di hash SHA-1 non salati trapelati. Quaranta caratteri esadecimali ciascuno. I ricercatori hanno identificato il formato in pochi secondi, poi hanno craccato il 90% degli hash in 72 ore perché SHA-1 è veloce e le password non avevano salt (più avanti sul salt). LinkedIn in seguito ha ammesso che [117 milioni di account in più](https://www.theguardian.com/technology/2016/may/19/linkedin-2012-data-breach-hack-117-million-email-password-details) erano stati esposti rispetto a quanto inizialmente dichiarato.

**[Breach di Adobe del 2013](https://krebsonsecurity.com/2013/11/adobe-breach-impacted-at-least-38-million-users/)** — 153 milioni di account, con password memorizzate usando la cifratura 3DES (nemmeno l'hashing) e senza salt univoci. La mancanza di salt univoci significava che password identiche producevano ciphertext identico. I ricercatori potevano vedere, solo guardando il dump, quali 1,9 milioni di account condividevano la password `123456`.

**[Breach di Yahoo del 2016](https://en.wikipedia.org/wiki/Yahoo!_data_breaches)** — 3 miliardi di account. Alcuni hashati con MD5 (catastrofico), alcuni con bcrypt (molto meglio). Il formato misto ha reso l'identificazione il primo compito prima che potesse procedere qualsiasi analisi difensiva.

**[Collection #1 del 2019](https://www.troyhunt.com/the-773-million-record-collection-1-data-reach/)** — 773 milioni di coppie email/password aggregate da breach passati. I ricercatori hanno dovuto ordinare quali hash fossero di quale algoritmo prima di qualsiasi altra cosa.

Il pattern è lo stesso ogni volta: compare il dump → identifica l'algoritmo → decidi se il cracking è fattibile → passa a hashcat.

## 4. I tre segnali che un hash lascia trapelare su se stesso

Questo è il cuore del tool. Una stringa di hash porta con sé fino a tre indizi sulla propria origine: il suo **prefisso**, la sua **lunghezza** e il suo **set di caratteri**. Li usiamo in ordine, dal più forte per primo.

### Segnale 1: prefisso (l'indizio più forte)

I moderni password hash usano un formato auto-descrittivo chiamato **PHC string format** (PHC sta per "Password Hashing Competition", la gara che ci ha dato Argon2 nel 2015). Una stringa PHC inizia con un marcatore che annuncia l'algoritmo:

```
$argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$RdescudvJCsgt3ub+b+dWRWJTmaaJObG
^^^^^^^^^^
 │
 └─ "I am an Argon2id hash. You don't have to guess."
```

```
$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQNQy.uK4Of2T7G.VHvgvWK
^^^^
 │
 └─ "I am a bcrypt hash, variant 2b, cost factor 12."
```

Quando un hash si annuncia così, l'identificazione è essenzialmente gratuita — ti basta confrontare i prefissi. Il tool riporta **confidenza HIGH** per le corrispondenze sul prefisso perché l'hash ti ha letteralmente detto cos'è. La specifica PHC è documentata [qui](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md) se vuoi la grammatica completa.

I vecchi hash *non* lo fanno. MD5 e SHA-1 ti danno semplicemente il digest esadecimale grezzo senza prefisso. Ecco perché il passaggio moderno a PHC è stato una cosa importante — rende il sistema auto-documentante.

Ecco un cheat sheet non esaustivo dei prefissi comuni. La tabella completa è in `hash_identifier.py` sotto `PREFIX_RULES`:

| Prefisso           | Algoritmo           | Dove lo vedi                                    |
| ------------------ | ------------------- | ----------------------------------------------- |
| `$argon2id$`       | Argon2id            | App web moderne, l'attuale gold standard        |
| `$2b$`             | bcrypt              | Il cavallo di battaglia degli ultimi 15 anni    |
| `$6$`              | SHA-512 crypt       | `/etc/shadow` sulla maggior parte delle distro Linux |
| `$apr1$`           | Apache MD5-crypt    | File `.htpasswd` (basic auth di Apache)         |
| `$P$`              | phpass              | WordPress e vecchi forum phpBB                  |
| `pbkdf2_sha256$`   | Django PBKDF2       | Default per le app web Django                   |
| `{SSHA}`           | LDAP salted SHA-1   | Password delle directory LDAP                   |

### Segnale 2: lunghezza (l'indizio secondo più forte)

Le funzioni di hash emettono output di lunghezza fissa. Se non vedi alcun prefisso ma hai 64 caratteri esadecimali, puoi essere fiducioso che provenga da uno della famiglia di hash a 256 bit (SHA-256, SHA3-256, BLAKE2s-256, RIPEMD-256). Ogni algoritmo emette sempre lo stesso numero di byte:

```
algorithm        bytes      hex chars
─────────────────────────────────────
MD5              16           32
SHA-1            20           40
SHA-224          28           56
SHA-256          32           64
SHA-384          48           96
SHA-512          64          128
```

Il motivo per cui il numero di caratteri esadecimali è il doppio del numero di byte: ogni byte è 8 bit, ma un carattere esadecimale rappresenta solo 4 bit (uno di 16 valori possibili: `0-9a-f`). Quindi servono due caratteri esadecimali per byte.

La lunghezza restringe il campo ma raramente individua un vincitore univoco. 32 caratteri esadecimali potrebbero essere MD5, NTLM, MD4 o RIPEMD-128 — producono tutti 128 bit di output. Quindi quando il tool trova una corrispondenza sulla lunghezza, riporta **confidenza MEDIUM** per l'algoritmo più probabile a quella lunghezza e **LOW** per gli altri. "Più probabile" significa "più comune nel mondo reale nel 2026" — MD5 supera di gran lunga RIPEMD-128.

### Segnale 3: charset (usato come controllo di coerenza)

Il set di caratteri dell'hash restringe ulteriormente le cose. Compaiono tre alfabeti comuni:

- **Hex:** solo `0-9a-f` (o `0-9A-F` se maiuscolo). Usato da MD5 grezzo, famiglia SHA, NTLM, ecc.
- **Base64-ish:** `0-9A-Za-z+/=`. Usato da LDAP e alcuni formati di password Java.
- **crypt(3) base64:** un alfabeto peculiare `./0-9A-Za-z` (nota il `.` e il `/` all'inizio, nessun `=` per il padding). Usato da bcrypt e dai vecchi formati crypt di Unix.

Una stringa con un `+` *non* è un hash esadecimale. Una stringa con un `*` seguito da 40 caratteri esadecimali maiuscoli è quasi certamente MySQL5 (e solo MySQL5 — perché MySQL stampa i suoi hash usando lo specificatore di formato C `%02X`, che è solo maiuscolo).

Il charset da solo raramente identifica qualcosa, ma è il fattore decisivo che ci permette di *escludere* le cose. Per esempio, l'helper `_is_mysql5` nel codice si rifiuta di trovare una corrispondenza se il corpo è minuscolo, perché il vero output di MySQL5 è sempre maiuscolo. Meglio dire "non lo so" che mentire con sicurezza.

## 5. Salt (una breve digressione)

Vedrai la parola **salt** ovunque quando leggi sull'hashing delle password. Un salt è una stringa casuale univoca che mescoli nella password prima dell'hashing, poi memorizzi accanto all'hash:

```
hash = bcrypt("hunter2" + random_salt_for_alice)
```

Il punto di un salt è rendere l'hash di ogni utente *diverso*, anche se due utenti hanno scelto la stessa password. Senza salt, puoi ordinare il database per colonna di hash, contare i duplicati e imparare immediatamente quale password è la più popolare (è esattamente così che i ricercatori hanno scoperto che `123456` era la password più popolare di Adobe — nessun cracking richiesto).

I salt sconfiggono anche le **rainbow table**: tabelle di lookup precalcolate che mappano `hash → password` per miliardi di password comuni. Con un salt univoco per utente, ogni voce della rainbow table dovrebbe essere ricalcolata per *ogni salt*, cosa non fattibile.

Una stringa PHC come `$2b$12$EixZaYVK1fsbw1ZfbX3OXe...` contiene il salt proprio lì nella stringa (la parte `EixZaYVK1fsbw1ZfbX3OXe`). Questo non è un problema di sicurezza — il salt deve essere pubblico. Il suo compito è rendere ogni hash univoco, non essere segreto.

## 6. Perché l'identificazione deve venire per prima

I tool di cracking non rilevano automaticamente l'algoritmo — hanno bisogno che tu glielo dica.

```bash
# Hashcat, mode 0 (MD5):
hashcat -m 0 -a 0 5f4dcc3b5aa765d61d8327deb882cf99 wordlist.txt

# Hashcat, mode 3200 (bcrypt):
hashcat -m 3200 -a 0 '$2b$12$EixZaY...' wordlist.txt

# John the Ripper, --format=raw-md5:
john --format=raw-md5 --wordlist=wordlist.txt hashes.txt
```

Scegli la modalità sbagliata e hashcat resterà lì a confrontare il tuo hash bcrypt con output MD5 per sempre senza trovare nulla. Quindi *prima* di fare il crack, identifichi.

Ecco perché esiste un tool come quello in questo progetto (e i più vecchi progetti [`hashid`](https://github.com/psypanda/hashID) e [`hash-identifier`](https://github.com/blackploit/hash-identifier) a cui si ispira). È il primo passo. Il nostro tool è un clone adatto ai principianti di quell'idea, scritto in modo che tu possa leggere ogni riga e capire ogni decisione.

## 7. Il tradeoff che questo tool sta facendo

Potremmo teoricamente ottenere maggiore accuratezza provando ogni algoritmo e calcolando diagnostiche. Non lo facciamo. Prendiamo decisioni dalla *sola forma della stringa*:

- Non eseguiamo mai alcuna funzione di hash.
- Non facciamo mai richieste di rete.
- Non tocchiamo mai il filesystem.
- Non chiamiamo mai alcun tool esterno.

Questo rende il tool **veloce** (istantaneo), **sicuro** (impossibile far trapelare dati) e **banalmente testabile** (ogni test è "dato l'input X, aspettati l'output Y"). È una funzione pura, in senso matematico — lo stesso input dà sempre lo stesso output, nessun effetto collaterale.

Il costo è che a volte riportiamo più candidati con confidenza MEDIUM/LOW. In linea di principio potremmo scegliere un vincitore se provassimo ogni algoritmo contro una wordlist nota — ma quello è un tool diverso. Quel tool è il cracker, e vive in `PROJECTS/beginner/hash-cracker`. L'intero compito del nostro tool è indicarti la *modalità di cracker giusta* in primo luogo.

## 8. Cosa farà e cosa non farà il tool

| Farà                                                  | Non farà                                            |
| ----------------------------------------------------- | --------------------------------------------------- |
| Identificare ~30 formati di hash dal prefisso         | Craccare qualsiasi hash                             |
| Identificare hash esadecimali comuni dalla lunghezza  | Calcolare hash per te                               |
| Riconoscere MySQL5, NetNTLM, DES crypt dalla forma    | Chiamare hashcat o john per te                      |
| Dirti "quello è un JWT" o "quello è base64, non un hash" | Dirti la password                                |
| Stampare candidati ordinati con confidenza e motivazione | Fare richieste di rete                           |
| Girare come CLI one-shot in millisecondi              | Toccare il filesystem                               |

## 9. Dove andare da qui

Ora che sai *cosa* sta facendo il tool e *perché*, leggi **[02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md)** per vedere come il codice è strutturato in una pipeline decisionale a sei passi. Poi **[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md)** percorre insieme a te il vero file Python, funzione per funzione.
