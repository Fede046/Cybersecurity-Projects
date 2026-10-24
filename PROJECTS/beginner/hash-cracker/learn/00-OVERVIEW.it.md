# Hash Cracker CLI Tool

## Cos'è

Uno strumento da riga di comando multi-thread che recupera password in chiaro a partire da hash crittografici. Supporta MD5, SHA1, SHA256 e SHA512 usando tre strategie di attacco: dictionary attack basati su wordlist, brute force con generazione di tutte le combinazioni possibili e mutazioni basate su regole che trasformano le parole del dizionario in comuni varianti di password come `P@ssw0rd123`.

## Perché è importante

Quando gli attaccanti violano un database, non ottengono le password in chiaro. Ottengono gli hash. La sicurezza di ogni utente in quel database dipende da quanto quegli hash siano resistenti al cracking. Costruire un cracker ti insegna esattamente perché gli hash veloci e senza salt come MD5 sono catastrofici per la memorizzazione delle password, e perché i sistemi moderni usano invece bcrypt o argon2.

**Scenari del mondo reale in cui questo si applica:**
- I penetration tester usano strumenti come hashcat e john the ripper per verificare la robustezza delle password dopo aver ottenuto accesso a dump di hash
- Il breach di LinkedIn del 2012 ha fatto trapelare 6,5 milioni di hash SHA1 senza salt. I ricercatori ne hanno craccato il 90% entro 72 ore
- Il breach di Adobe del 2013 ha esposto 153 milioni di account usando cifratura 3DES (nemmeno hashing) senza salt univoci. Lo stesso blob cifrato appariva milioni di volte perché password identiche producevano ciphertext identico

## Cosa imparerai

**Concetti di sicurezza:**
- Funzioni di hash crittografiche: trasformazioni a senso unico che non possono essere invertite
- Dictionary attack: sfruttare liste di password note provenienti da breach reali
- Brute force attack: ricerca esaustiva attraverso tutte le combinazioni di caratteri possibili
- Mutazioni basate su regole: perché `Password123!` non è una password robusta
- Salting: cosa previene (rainbow table) e cosa non previene (cracking mirato)

**Competenze tecniche:**
- Progettazione di template policy-based in C++23 con vincoli tramite concept
- `std::expected` per una gestione degli errori componibile senza eccezioni
- `std::generator` per la lazy evaluation con le coroutine
- I/O su file memory-mapped con mmap per la lettura zero-copy delle wordlist
- Partizionamento del lavoro multi-thread con `std::jthread` e atomics
- OpenSSL EVP API per il calcolo degli hash

**Strumenti e tecniche:**
- CMake con preset per build riproducibili
- GoogleTest per test unitari e di integrazione
- OpenSSL per le funzioni di hash crittografiche
- Boost.program_options per il parsing degli argomenti CLI

## Prerequisiti

**Conoscenze richieste:**
- Basi di C++: classi, template, lambda, move semantics
- Comprensione di cosa fa una funzione di hash (entra un input, esce un output a lunghezza fissa, non lo puoi invertire)
- Familiarità con la riga di comando

**Strumenti di cui avrai bisogno:**
- GCC 14 o superiore (è richiesto il supporto a C++23)
- CMake 3.25+
- Sistema di build Ninja
- Header di sviluppo di OpenSSL
- Boost.program_options

**Utile ma non richiesto:**
- Esperienza con template e concept
- Familiarità con threading e atomics
- Comprensione dell'I/O memory-mapped

## Quick Start

```bash
cd PROJECTS/beginner/hash-cracker

./install.sh

hashcracker --hash 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8 \
  --wordlist wordlists/10k-most-common.txt
```

Output atteso: lo strumento rileva automaticamente l'hash come SHA256, cerca nel dizionario da 10.000 parole e trova la password `password` in meno di un secondo. Se il tuo terminale lo supporta, vedrai una barra di progresso con velocità ed ETA mentre viene eseguito.

Prova il brute force:
```bash
hashcracker --hash 187ef4436122d1cc2f40dc2b92f0eba0 \
  --bruteforce --charset lower --max-length 4 --type md5
```

Output atteso: genera tutte le combinazioni minuscole fino a 4 caratteri, trova `ab` dopo aver cercato ~350.000 candidati.

## Struttura del progetto

```
hash-cracker/
├── main.cpp                     CLI parsing and dispatch
├── src/
│   ├── config/Config.hpp        Constants, character sets, colors
│   ├── core/
│   │   ├── Concepts.hpp         Hasher and AttackStrategy concepts
│   │   └── Engine.hpp           Template engine (hasher + attack + threading)
│   ├── hash/
│   │   ├── EVPHasher.hpp        Unified OpenSSL EVP hasher template
│   │   ├── HashDetector.hpp     Auto-detect hash type from hex length
│   │   ├── MD5Hasher.hpp        Type alias for EVPHasher<EVP_md5, ...>
│   │   ├── SHA1Hasher.hpp       Type alias for EVPHasher<EVP_sha1, ...>
│   │   ├── SHA256Hasher.hpp     Type alias for EVPHasher<EVP_sha256, ...>
│   │   └── SHA512Hasher.hpp     Type alias for EVPHasher<EVP_sha512, ...>
│   ├── attack/
│   │   ├── DictionaryAttack     mmap wordlist reader with partitioning
│   │   ├── BruteForceAttack     Keyspace generator with partitioning
│   │   └── RuleAttack           Dictionary + mutation rules
│   ├── rules/RuleSet            Mutation transforms via std::generator
│   ├── io/MappedFile            RAII wrapper for mmap
│   ├── threading/ThreadPool     std::jthread work partitioning
│   └── display/Progress         Terminal progress display
├── tests/                       GoogleTest suite (38 tests)
├── wordlists/                   Included 10k common passwords
├── install.sh                   One-command setup
├── Justfile                     Build/test/clean commands
└── CMakeLists.txt               Build configuration
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per imparare come funziona l'hash cracking e perché è importante
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) per vedere come concept, template e threading si integrano tra loro
3. **Esamina il codice** - Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per una dettagliata panoramica del codice
4. **Estendi il progetto** - Leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per aggiungere accelerazione GPU, rainbow table o nuovi algoritmi

## Problemi comuni

**"File not found" quando usi una wordlist**
```
Error: File not found
```
Soluzione: i percorsi sono relativi al punto da cui esegui il comando. Esegui dalla directory root del progetto, oppure usa un percorso assoluto.

**Errore "Invalid hash format"**
```
Error: Invalid hash format
```
Soluzione: l'auto-detector verifica che tutti i caratteri siano esadecimali (0-9, a-f) e che la lunghezza corrisponda a un algoritmo noto (32=MD5, 40=SHA1, 64=SHA256, 128=SHA512). Controlla la presenza di spazi finali o caratteri non esadecimali.

**Il brute force è lento con password lunghe**
Questo è previsto. Il key space cresce esponenzialmente. 6 caratteri minuscoli = 308 milioni di combinazioni. Aggiungi le cifre e diventano 2,2 miliardi. Aggiungi maiuscole e caratteri speciali e ti ritrovi con ore o giorni. Questo è l'intero punto: le password robuste resistono al brute force.
