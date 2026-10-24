# Concetti fondamentali di sicurezza

Questo documento spiega i concetti di sicurezza alla base dell'hash cracking. Non si tratta solo di definizioni. Approfondiremo come funzionano davvero gli attacchi, perché esistono certe difese e cosa i breach reali ci insegnano sulla memorizzazione delle password.

## Funzioni di hash crittografiche

### Cosa sono

Una funzione di hash prende in input dati di qualsiasi lunghezza e produce un output a lunghezza fissa (il digest). Lo stesso input produce sempre lo stesso output, ma non puoi risalire dall'output per recuperare l'input. Si tratta di una trasformazione a senso unico, non di cifratura.

```
"password"  → SHA256 → 5e884898da28047151d0e56f8dc...  (always this, every time)
"password1" → SHA256 → 0b14d501a594442a01c6859541bc...  (completely different)
"Password"  → SHA256 → 22ee405817c14cbf9d3c2b92b87c...  (completely different again)
```

Tre proprietà rendono le funzioni di hash utili per la memorizzazione delle password:

1. **Deterministiche**: stesso input, stesso output. Il server può verificare la tua password calcolando l'hash di ciò che digiti e confrontandolo con l'hash memorizzato
2. **Resistenti alla pre-image**: dato un hash, non puoi calcolare l'input che l'ha prodotto. Anche se un attaccante ruba il database degli hash, non ha le password in chiaro
3. **Effetto valanga (avalanche effect)**: cambiare un bit dell'input cambia circa metà dei bit dell'output. "password" e "Password" producono hash completamente non correlati

### Algoritmi supportati da questo strumento

| Algoritmo | Lunghezza del digest | Stato | Utilizzo nel mondo reale |
|-----------|--------------|--------|-----------------|
| MD5 | 128 bit (32 caratteri hex) | Compromesso dal 2004 | Ancora presente in sistemi legacy, WordPress prima della 4.x |
| SHA1 | 160 bit (40 caratteri hex) | Compromesso dal 2017 | Breach di LinkedIn (2012), Git (in fase di abbandono) |
| SHA256 | 256 bit (64 caratteri hex) | Sicuro | Bitcoin, certificati TLS, molti framework web |
| SHA512 | 512 bit (128 caratteri hex) | Sicuro | Default di /etc/shadow su Linux, alcuni sistemi enterprise |

"Compromesso" per MD5 e SHA1 significa che i ricercatori possono generare collisioni (due input diversi che producono lo stesso hash). Per il cracking delle password, il problema più grande è la velocità: questi sono stati progettati per essere veloci, che è esattamente ciò che non vuoi per l'hashing delle password.

### Cosa NON sono le funzioni di hash

Le funzioni di hash non sono cifratura. La cifratura è reversibile con una chiave. L'hashing non è affatto reversibile. Non "decifri" un hash. Indovini degli input, ne calcoli l'hash e controlli se l'output corrisponde. Questo è il cracking.

## Dictionary Attack

### Come funzionano

Un dictionary attack prova ogni parola di una lista di password note. L'attaccante calcola l'hash di ogni parola e lo confronta con l'hash target:

```
Target: 5e884898da28047151d0e56f8dc6292773603d0d...

Hash("123456")   → "e10adc..."  ≠ target
Hash("password") → "5e8848..."  = target  ← found it
```

Questo funziona perché le persone scelgono password prevedibili. Il breach di RockYou nel 2009 ha fatto trapelare 32 milioni di password in chiaro, e le prime 10 erano:

```
1. 123456        6. monkey
2. 12345         7. 1234567
3. 123456789     8. letmein
4. password      9. trustno1
5. iloveyou     10. dragon
```

Queste liste vengono riutilizzate in ogni strumento di cracking. Un dizionario da 14 milioni di parole impiega secondi a esaurirsi contro un hash veloce come SHA256 su hardware moderno.

### Perché sono così efficaci

HaveIBeenPwned tiene traccia di oltre 613 milioni di password provenienti da breach reali. Se la tua password è mai apparsa in un qualsiasi breach, è in un dizionario di cracking. Il riutilizzo delle password peggiora la situazione: la password che usi per un forum usa e getta potrebbe finire in un dizionario usato per craccare il tuo account email.

## Brute Force Attack

### Come funzionano

Il brute force genera ogni possibile combinazione di caratteri fino a una lunghezza massima:

```
Length 1: a, b, c, ..., z                              (26 combinations)
Length 2: aa, ab, ac, ..., zz                           (676 combinations)
Length 3: aaa, aab, ..., zzz                            (17,576 combinations)
Length 4: aaaa, ..., zzzz                               (456,976 combinations)
...
Length 8: aaaaaaaa, ..., zzzzzzzz                       (208 billion combinations)
```

Il key space cresce esponenzialmente. Aggiungi le lettere maiuscole e la base passa da 26 a 52. Aggiungi le cifre e diventa 62. Aggiungi i caratteri speciali e diventa 95. Una password di 8 caratteri che usa tutti i tipi di carattere ha 95^8 = 6,6 quadrilioni di combinazioni.

### La matematica della fattibilità

A 3 milioni di hash SHA256 al secondo (ciò che questo strumento raggiunge su CPU):

| Password | Set di caratteri | Combinazioni | Tempo |
|----------|--------------|-------------|------|
| 4 caratteri | minuscole | 456K | 0,15 secondi |
| 6 caratteri | minuscole | 308M | 1,7 minuti |
| 6 caratteri | minuscole + cifre | 2,2B | 12 minuti |
| 8 caratteri | minuscole | 208B | 19 ore |
| 8 caratteri | tutti i caratteri stampabili | 6,6Q | 70.000 anni |

Gli strumenti di cracking su GPU come hashcat raggiungono i 3 miliardi al secondo (1000 volte più veloce), ma anche così 8 caratteri con set di caratteri completi richiedono 25 giorni. Ecco perché la lunghezza della password conta più della complessità.

## Mutazioni basate su regole

### Come funzionano

Gli esseri umani sono prevedibili. Quando sono costretti ad aggiungere una lettera maiuscola, la maggior parte delle persone mette in maiuscolo la prima lettera. Quando sono costretti ad aggiungere un numero, lo accodano. Quando sono costretti ad aggiungere un carattere speciale, usano `!` o `@`. Gli attacchi basati su regole sfruttano questi pattern:

| Regola | Input | Output |
|------|-------|--------|
| Prima lettera maiuscola | password | Password |
| Tutto maiuscolo | password | PASSWORD |
| Sostituzione leet | password | p@$$w0rd |
| Accoda cifre 0-999 | password | password123 |
| Inverti | password | drowssap |
| Inverti maiuscole/minuscole | password | PASSWORD |

Questo trasforma un dizionario da 14 milioni di parole in miliardi di candidati. La combinazione di `capitalize + leet + append digits` trasforma `password` in `P@$$w0rd123`, che soddisfa ogni policy delle password mai scritta e viene comunque craccata in millisecondi.

### Perché i requisiti di complessità non funzionano

L'analisi di Specops del 2021 su 800 milioni di password compromesse ha scoperto che l'83% soddisfaceva i requisiti standard di complessità (8+ caratteri, maiuscola, minuscola, numero, carattere speciale). I pattern più comuni:

```
[Word][Number]        → Password1
[Word][Number][!]     → Password1!
[Season][Year]        → Summer2024
[Name][Birthday]      → Michael1990!
```

I password manager che generano stringhe casuali come `x7$kQ2!mR9pL` sono la difesa vera. Nessun dizionario la contiene, e il brute force di 12 caratteri casuali dall'intero set stampabile è computazionalmente impraticabile.

## Salting delle password

### Cos'è

Un salt è una stringa casuale memorizzata insieme all'hash della password. Prima dell'hashing, il salt viene anteposto o accodato alla password:

```
Without salt:
  User A: SHA256("password")       → 5e884898da...
  User B: SHA256("password")       → 5e884898da...  (identical!)

With salt:
  User A: SHA256("x9f2" + "password") → a1b2c3d4...
  User B: SHA256("k7m1" + "password") → 9z8y7x6w...  (completely different)
```

Il salt non è segreto. È memorizzato in chiaro proprio accanto all'hash nel database. Il suo unico compito è rendere univoco l'hash di ogni utente, anche se usano la stessa password.

### Cosa previene il salt

Le **rainbow table** sono tabelle di lookup precalcolate che mappano gli hash alle password. Senza salt, calcoli SHA256("password") una volta e corrisponde a ogni utente che ha scelto "password". Una rainbow table per SHA256 che copre le password comuni potrebbe occupare 100GB, ma cracca milioni di hash istantaneamente.

Il salt rende inutili le rainbow table. Ogni utente ha un salt diverso, quindi ti servirebbe una rainbow table separata per ogni possibile valore di salt. Con un salt di 16 byte, sono 2^128 tabelle possibili. Non succederà.

Anche il **mass cracking** viene sconfitto. Senza salt, se cracchi un hash, hai craccato ogni utente con quella password. Con il salt, cracchi un utente alla volta perché ogni hash viene calcolato in modo diverso.

### Cosa NON previene il salt

Il salt non rallenta il cracking mirato. Se un attaccante vuole craccare l'hash di uno specifico utente, conosce il salt (è nel database) e lo antepone semplicemente a ogni tentativo. La velocità di cracking è identica. Il nostro flag `--salt` dimostra esattamente questo attacco.

### Fallimenti del salt nel mondo reale

Il breach di LinkedIn del 2012 ha fatto trapelare 6,5 milioni di hash SHA1 senza salt. I ricercatori ne hanno craccato il 90% entro 72 ore. Se LinkedIn avesse usato i salt, il cracking avrebbe richiesto di attaccare ogni hash individualmente invece di craccare l'intero database in una volta.

Il breach di Adobe del 2013 usava la cifratura 3DES (nemmeno hashing) con una singola chiave e senza salt univoci. Poiché password identiche producevano ciphertext identico, i ricercatori potevano identificare le password più comuni semplicemente contando i duplicati, per poi craccarle sulla base dei suggerimenti di password che erano anch'essi trapelati.

## Funzioni di hash lente

### Perché gli hash veloci sono il problema

SHA256 è stato progettato per calcolare l'hash dei dati rapidamente. Questo è ottimo per verificare l'integrità dei file o gli handshake TLS, ma pessimo per la memorizzazione delle password. Una GPU può calcolare 3 miliardi di hash SHA256 al secondo. Questo significa che un attaccante può provare 3 miliardi di password al secondo.

### Come bcrypt e argon2 risolvono il problema

bcrypt, scrypt e argon2 sono funzioni di hash volutamente lente. Aggiungono un "work factor" configurabile che controlla quanto tempo impiega ogni hash:

```
SHA256:  ~3,000,000 hashes/sec (CPU)  ~3,000,000,000 hashes/sec (GPU)
bcrypt:  ~300 hashes/sec (CPU)        ~50,000 hashes/sec (GPU)
argon2:  ~10 hashes/sec (CPU)         GPU-resistant by design
```

bcrypt con cost factor 12 impiega circa 250ms per hash. Un utente che effettua il login aspetta 250ms (impercettibile). Un attaccante che prova 14 milioni di parole del dizionario aspetta 40 giorni. Stessa matematica, esito completamente diverso.

argon2 va oltre richiedendo grandi quantità di memoria per hash, il che limita il parallelismo della GPU. Le GPU hanno migliaia di core ma memoria limitata per core. argon2 sfrutta questo costringendo ogni hash a usare megabyte di RAM, rendendo impraticabile il cracking su GPU.

### Perché non li cracchiamo

Questo strumento cracca solo hash veloci (MD5, SHA1, SHA256, SHA512). È voluto. Craccare bcrypt con un dizionario da 10.000 parole a 300 hash/sec impiega 33 secondi. Lo stesso dizionario contro SHA256 impiega 3 millisecondi. La differenza di velocità è la lezione.

## Standard di settore

La **OWASP Password Storage Cheat Sheet** raccomanda:
- argon2id come scelta primaria
- bcrypt con cost factor 10+ come fallback
- Mai MD5 o la famiglia SHA per le password
- Salt casuale minimo di 16 byte per password

**NIST SP 800-63B** (Digital Identity Guidelines):
- I segreti memorizzati dovrebbero essere sottoposti ad hashing con un salt usando una key derivation function
- Almeno 10.000 iterazioni se si usa PBKDF2
- Verificare le password rispetto a database noti di breach

**MITRE CWE-916**: Use of Password Hash With Insufficient Computational Effort. Assegnato ai sistemi che usano MD5, SHA1 o SHA256 per la memorizzazione delle password senza una key derivation function lenta.

## Metti alla prova la tua comprensione

1. Un attaccante ruba un database con 1 milione di hash SHA256, tutti senza salt. In che modo l'assenza di salt aiuta l'attaccante al di là delle sole rainbow table?

2. Un sito web richiede che le password siano "di almeno 8 caratteri con maiuscola, minuscola, numero e carattere speciale". Perché questo non previene gli attacchi basati su regole?

3. Perché argon2 resiste specificamente all'accelerazione GPU mentre bcrypt vi resiste solo parzialmente?

4. Trovi un hash `5f4dcc3b5aa765d61d8327deb882cf99` in un dump di un breach. Senza eseguire alcuno strumento di cracking, cosa puoi determinare su di esso solo guardandolo?

## Approfondimenti

**Essenziali:**
- [How Passwords Are Stored](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) (OWASP)
- [documentazione di hashcat](https://hashcat.net/wiki/) per comprendere le tecniche di cracking del mondo reale

**Approfondimento tecnico:**
- [Paper su Bcrypt](https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf) di Provos e Mazieres (1999)
- [Specifica di Argon2](https://github.com/P-H-C/phc-winner-argon2/blob/master/argon2-specs.pdf) (vincitore della Password Hashing Competition)
- [Have I Been Pwned](https://haveibeenpwned.com/) per verificare se le password appaiono nei database dei breach
