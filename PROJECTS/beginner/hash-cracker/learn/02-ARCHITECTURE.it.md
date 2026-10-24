# Architettura del sistema

Questo documento analizza come è progettato il sistema e perché sono state prese certe decisioni architetturali.

## Architettura ad alto livello

```
┌────────────────────┐
│     CLI Layer      │  (main.cpp)
│  - parse args      │  Boost.program_options
│  - detect hash     │  Auto-type from length
│  - dispatch        │  Template instantiation
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   Engine Layer     │  (Engine.hpp)
│  - create threads  │  std::jthread pool
│  - partition work  │  Per-thread attack slices
│  - coordinate      │  Shared atomics
│  - display         │  Progress thread
└─────────┬──────────┘
          │
    ┌─────┴─────┐
    ▼           ▼
┌────────┐ ┌──────────┐
│ Hasher │ │ Attack   │
│ Policy │ │ Strategy │
└────────┘ └──────────┘
    │           │
    ▼           ▼
┌────────┐ ┌──────────────────────────────────┐
│ EVP    │ │ DictionaryAttack (mmap)          │
│ Hasher │ │ BruteForceAttack (keyspace math) │
│        │ │ RuleAttack (dict + mutations)    │
└────────┘ └──────────────────────────────────┘
```

### Analisi dei componenti

**CLI Layer (main.cpp)**
- Scopo: effettuare il parsing degli argomenti da riga di comando e fare il dispatch verso la corretta istanziazione dell'Engine
- Responsabilità: validazione degli argomenti, rilevamento del tipo di hash, costruzione del charset, output JSON
- Scelta di design chiave: usa una lambda template per evitare il polimorfismo a runtime. Lo `switch` sul tipo di hash crea un percorso di chiamata risolto a compile-time

**Engine Layer (Engine.hpp)**
- Scopo: coordinare hashing, strategia di attacco, threading e visualizzazione del progresso
- Responsabilità: gestione del thread pool, applicazione del salt, raccolta dei risultati, misurazione dei tempi
- Scelta di design chiave: funzione template header-only. I tipi dell'hasher e della strategia di attacco sono parametri template, quindi il compilatore inietta (inline) la funzione di hash direttamente nel loop di cracking

**Hasher Policy (EVPHasher.hpp)**
- Scopo: calcolare gli hash crittografici tramite OpenSSL
- Responsabilità: gestione del contesto EVP, calcolo del digest, codifica esadecimale
- Scelta di design chiave: un singolo template parametrizzato dal puntatore a funzione dell'algoritmo. Tutti e quattro i tipi di hash sono type alias della stessa implementazione

**Strategie di attacco (attack/)**
- Scopo: generare password candidate
- Responsabilità: lettura delle wordlist, generazione del key space, applicazione delle mutazioni, partizionamento dei thread
- Scelta di design chiave: ogni strategia soddisfa il concept `AttackStrategy` e gestisce il proprio partizionamento tramite `create(path, thread_index, total_threads)`

**Progress Display (display/)**
- Scopo: mostrare il progresso del cracking in tempo reale
- Responsabilità: rendering della barra di progresso, calcolo di velocità/ETA, formattazione del risultato
- Scelta di design chiave: gira su un proprio thread, legge atomics condivisi con relaxed ordering, non fa nulla (no-op) quando stdout non è un terminale

## Design fondamentale: template policy-based

La decisione architetturale centrale è risolvere l'hasher e la strategia di attacco a compile-time anziché a runtime. Confronta i due approcci:

**Polimorfismo a runtime (ciò che non abbiamo fatto):**
```cpp
class IHasher {
public:
    virtual std::string hash(std::string_view input) = 0;
    virtual ~IHasher() = default;
};

void crack(IHasher* hasher, ...) {
    for (each candidate) {
        hasher->hash(candidate);  // virtual call every iteration
    }
}
```

Ogni chiamata a `hash()` passa attraverso la vtable, il che significa un salto indiretto (indirect branch). La CPU non può fare l'inline del corpo della funzione, non può ottimizzare oltre il confine della chiamata e paga una penalità di branch prediction. In un loop che gira milioni di volte al secondo, questo si somma.

**Polimorfismo a compile-time (ciò che abbiamo fatto):**
```cpp
template <Hasher H, AttackStrategy A>
auto crack(const CrackConfig& cfg) -> std::expected<CrackResult, CrackError> {
    H hasher;
    // ... hasher.hash(candidate) is a direct call, gets inlined
}
```

Quando scrivi `Engine::crack<SHA256Hasher, DictionaryAttack>(cfg)`, il compilatore genera una versione di `crack` con SHA256Hasher hardcoded. La chiamata a `hash()` diventa una chiamata a funzione diretta che viene inlinata nel loop. Nessuna vtable, nessuna indirezione, nessun overhead.

Il compromesso: ottieni una copia separata della funzione per ogni combinazione hasher/attacco (12 in totale: 4 hasher x 3 attacchi). Questo aumenta leggermente la dimensione del binario. Per uno strumento CLI, è irrilevante.

## I concept come contratti

I concept del C++20 definiscono ciò che un Hasher o una AttackStrategy devono fornire:

```cpp
template <typename T>
concept Hasher = requires(T h, std::string_view input) {
    { h.hash(input) } -> std::same_as<std::string>;
    { T::name() } -> std::convertible_to<std::string_view>;
    { T::digest_length() } -> std::same_as<std::size_t>;
};
```

Questo è un contratto a compile-time. Se scrivi un nuovo hasher che non soddisfa `Hasher`, ottieni un messaggio di errore chiaro a compile-time invece di un misterioso errore del linker o di un crash a runtime. Documenta l'interfaccia senza richiedere ereditarietà.

## Flusso dei dati

### Flusso del Dictionary Attack

```
1. CLI parses --hash and --wordlist
   main.cpp dispatches Engine::crack<SHA256Hasher, DictionaryAttack>(cfg)

2. Engine resolves thread count (hardware_concurrency if 0)
   Creates ThreadPool, spawns N jthreads

3. Each thread calls DictionaryAttack::create(path, thread_id, N)
   create() opens the file with mmap, counts lines,
   computes this thread's byte range [start_offset, end_offset)

4. Thread loop:
   a. attack.next() reads the next word from the mmap region
   b. If salt is set, prepend/append it to the candidate
   c. hasher.hash(candidate) computes the digest
   d. Compare to target hash
   e. If match: set atomic found flag, store result, break
   f. Increment local counter, flush to shared atomic every 1024 iterations

5. Display thread wakes every 100ms, reads shared atomics,
   renders progress bar to terminal

6. All threads join (jthread RAII)
   Engine returns CrackResult or CrackError::Exhausted
```

### Partizionamento del Brute Force

Il key space totale per un charset di dimensione C e lunghezza massima L è:

```
total = C^1 + C^2 + C^3 + ... + C^L
```

Ogni thread riceve una porzione contigua dello spazio di indici piatto. Il thread 0 riceve gli indici `[0, total/N)`, il thread 1 riceve `[total/N, 2*total/N)`, e così via. La conversione di un indice piatto in una stringa candidata usa la decomposizione a radix misto (mixed-radix), simile alla conversione di un numero decimale in una base arbitraria ma con output a lunghezza variabile.

Questo significa che i thread non comunicano mai durante il loop di cracking. Nessuna coda condivisa, nessun work stealing, nessun lock. Ogni thread è completamente indipendente.

## Modello di threading

### Stato condiviso

Solo due valori sono condivisi tra i thread:

```cpp
struct SharedState {
    alignas(64) std::atomic<bool> found{false};
    alignas(64) std::atomic<std::size_t> tested_count{0};
    std::mutex result_mutex;
    std::optional<std::string> result;
};
```

L'`alignas(64)` colloca ogni atomic sulla propria cache line. Senza di esso, le scritture su `tested_count` da parte di un thread invaliderebbero le letture di `found` sugli altri thread perché condividono una cache line di 64 byte. Questo si chiama false sharing e può causare un rallentamento di 5 volte.

### Batching del contatore

Invece di eseguire un incremento atomico su ogni candidato:

```cpp
// Bad: atomic write every iteration (cross-core cache traffic)
state.tested_count.fetch_add(1, std::memory_order_relaxed);
```

Ogni thread mantiene un contatore locale e lo scarica ogni 1024 iterazioni:

```cpp
++local_count;
if ((local_count & 0x3FF) == 0) {
    state.tested_count.fetch_add(local_count, std::memory_order_relaxed);
    local_count = 0;
}
```

Questo riduce il traffico atomico cross-core di 1024 volte. Il display del progresso legge un conteggio approssimato, il che va bene per un aggiornamento della UI ogni 100ms.

## Strategia di gestione degli errori

Tutte le operazioni fallibili ritornano `std::expected<T, CrackError>`. Non ci sono eccezioni nel codebase.

```cpp
enum class CrackError {
    FileNotFound,
    InvalidHash,
    UnsupportedAlgorithm,
    OpenSSLError,
    InvalidConfig,
    Exhausted
};
```

Il tipo di errore è nella firma della funzione. I chiamanti sono costretti a gestirlo:

```cpp
auto attack = DictionaryAttack::create(path, tid, total);
if (!attack.has_value()) { return; }
```

Questo rende i percorsi di errore espliciti e visibili. Non devi mai indovinare se una funzione potrebbe lanciare un'eccezione.

## I/O memory-mapped

DictionaryAttack usa `mmap` invece di `std::ifstream` per leggere le wordlist:

```cpp
auto* mapped = static_cast<const char*>(
    mmap(nullptr, file_size, PROT_READ, MAP_PRIVATE, fd, 0));
madvise(mapped, file_size, MADV_SEQUENTIAL);
```

I contenuti del file vengono mappati direttamente nello spazio di indirizzamento del processo. Leggere una parola è aritmetica dei puntatori (avanzare al newline successivo). Nessuna syscall `read()`, nessuna copia di buffer da kernel a user space. Per una wordlist da 140MB come rockyou.txt, questo conta.

Il wrapper RAII `MappedFile` gestisce la pulizia:

```cpp
class MappedFile {
    ~MappedFile() {
        munmap(data_, size_);
        close(fd_);
    }
};
```

DictionaryAttack contiene un membro `MappedFile`. Il compilatore genera automaticamente le corrette operazioni di move (Rule of Zero).

## Configurazione

Tutti i magic number vivono in `Config.hpp`:

```cpp
namespace config {
constexpr unsigned DEFAULT_THREAD_COUNT = 0;
constexpr std::size_t DEFAULT_MAX_BRUTE_LENGTH = 6;
constexpr int PROGRESS_UPDATE_MS = 100;
constexpr std::string_view CHARSET_LOWER = "abcdefghijklmnopqrstuvwxyz";
// ...
}
```

La configurazione a runtime passa attraverso `CrackConfig`, popolato dal parser CLI e passato all'Engine. Nessuna variabile globale viene mutata dopo lo startup.

## Considerazioni sulle prestazioni

**Hot path**: il loop interno in `Engine.hpp` è hash-and-compare. Ogni microsecondo risparmiato qui si moltiplica per milioni di iterazioni. L'EVPHasher usa un encoder esadecimale basato su lookup-table invece di `std::ostringstream`, evitando un'allocazione sullo heap per ogni hash.

**Collo di bottiglia**: su CPU, il collo di bottiglia è il calcolo dell'hash stesso (gli internals EVP di OpenSSL). Il codice circostante (generazione dei candidati, confronto, aggiornamento del contatore) è trascurabile in confronto. L'accelerazione GPU (CUDA/OpenCL) sarebbe il prossimo passo per guadagni reali di prestazioni.

**Memoria**: mmap significa che la wordlist viene paginata su richiesta dal kernel. L'impronta di memoria effettiva dello strumento è piccola indipendentemente dalla dimensione della wordlist.

## Decisioni di design

| Decisione | Alternativa | Perché così |
|----------|------------|-------------|
| Template policy | Interfacce virtuali | Zero overhead nell'hot loop |
| `std::expected` | Eccezioni | Percorsi di errore espliciti, nessun flusso di controllo nascosto |
| `std::generator` | Ritornare `vector<string>` | Lazy evaluation, terminazione anticipata |
| mmap | `std::ifstream` | Zero-copy, nessuna syscall per riga |
| Partizionamento del lavoro | Coda con work stealing | Zero contesa tra i thread |
| Atomics relaxed | seq_cst | Il progresso approssimato va bene, risparmia il costo delle fence |
| Engine header-only | .cpp compilato | Il template deve comunque stare nell'header |

## Estensibilità

**Aggiungere un nuovo algoritmo di hash:**
1. Aggiungi un type alias in un nuovo header:
   ```cpp
   using SHA3_256Hasher = EVPHasher<EVP_sha3_256, "SHA3-256", 64>;
   ```
2. Aggiungi un case in `HashDetector::detect()` (se la lunghezza del digest è univoca)
3. Aggiungi un case nello switch di dispatch di `main.cpp`
4. Scrivi test rispetto a vettori noti

**Aggiungere una nuova strategia di attacco:**
1. Crea una classe che soddisfa il concept `AttackStrategy` (richiede `next()`, `total()`, `progress()`)
2. Aggiungi una factory `create(path, thread_index, total_threads)`
3. Aggiungi un percorso di dispatch in `main.cpp` ed `Engine.hpp`

Entrambi i punti di estensione richiedono zero modifiche all'Engine stesso.

## Limitazioni

- Solo CPU. Nessuna accelerazione GPU. Uno strumento dedicato come hashcat su una GPU moderna è ~1000 volte più veloce
- Nessun supporto per bcrypt/scrypt/argon2. Questi richiedono librerie diverse e strategie di cracking fondamentalmente diverse (la lentezza intenzionale cambia l'economia del problema)
- Nessun cracking distribuito. Ogni esecuzione usa una sola macchina
- Il flag `--chain-rules` genera un gran numero di candidati per parola, il che può rendere lenti gli attacchi basati su regole su wordlist grandi
- Solo Linux/macOS (mmap è POSIX). Windows richiederebbe invece `CreateFileMapping`
