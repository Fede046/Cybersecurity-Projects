# Panoramica dell'implementazione

Questo documento esamina il codice vero e proprio, spiegando come funziona ogni componente e perché è costruito nel modo in cui è.

## L'Hasher unificato

I quattro algoritmi di hash (MD5, SHA1, SHA256, SHA512) usano tutti la stessa OpenSSL EVP API. Le uniche differenze sono quale funzione di digest chiamare e la lunghezza dell'output. Invece di duplicare l'implementazione quattro volte, un singolo template li gestisce tutti:

```cpp
template <auto Algorithm, auto Name, std::size_t DigestLen>
class EVPHasher {
public:
    std::string hash(std::string_view input) const;
    static constexpr std::string_view name() { return Name; }
    static constexpr std::size_t digest_length() { return DigestLen; }
};
```

Ogni hasher concreto è semplicemente un type alias:

```cpp
using MD5Hasher    = EVPHasher<EVP_md5, "MD5", 32>;
using SHA256Hasher = EVPHasher<EVP_sha256, "SHA256", 64>;
```

I parametri template sono risolti a compile-time, quindi il compilatore genera quattro implementazioni separate di `hash()`, ciascuna hardcoded sulla propria specifica funzione di digest. Stesse prestazioni del codice scritto a mano, zero duplicazione.

### L'hot path: il calcolo dell'hash

Il metodo `hash()` è la funzione più critica per le prestazioni nel codebase. Ogni password candidata ci passa attraverso. L'implementazione usa l'interfaccia EVP di OpenSSL con pulizia RAII:

```cpp
std::string EVPHasher::hash(std::string_view input) const {
    auto ctx = std::unique_ptr<EVP_MD_CTX, decltype(&EVP_MD_CTX_free)>(
        EVP_MD_CTX_new(), EVP_MD_CTX_free);

    if (!ctx
        || !EVP_DigestInit_ex(ctx.get(), Algorithm(), nullptr)
        || !EVP_DigestUpdate(ctx.get(), input.data(), input.size())
        || !EVP_DigestFinal_ex(ctx.get(), digest.data(), &len)) {
        return "";
    }

    // Convert digest bytes to hex string
}
```

L'`unique_ptr` con un deleter personalizzato (`EVP_MD_CTX_free`) garantisce che il contesto venga liberato anche se qualcosa fallisce. L'`if` concatenato verifica ogni valore di ritorno di OpenSSL. Ritornare una stringa vuota in caso di fallimento è il corretto comportamento fail-safe: una stringa vuota non corrisponderà mai a un hash target valido, quindi un'esecuzione di cracking degrada in modo controllato invece di produrre risultati sbagliati.

### Codifica esadecimale con una lookup table

L'approccio ingenuo usa `std::ostringstream` con `std::hex` e `std::setw(2)`. Questo crea un oggetto stream allocato sullo heap per ogni singolo hash. A milioni di hash al secondo, sono milioni di allocazioni non necessarie.

Invece, una lookup table precalcolata converte ogni byte in due caratteri esadecimali con zero allocazioni:

```cpp
static constexpr std::array<std::array<char, 2>, 256> HEX_TABLE = [] {
    std::array<std::array<char, 2>, 256> t{};
    constexpr char digits[] = "0123456789abcdef";
    for (int i = 0; i < 256; ++i) {
        t[i] = {digits[i >> 4], digits[i & 0xF]};
    }
    return t;
}();
```

La tabella è calcolata a compile-time (lambda `constexpr`). Per ogni valore di byte 0-255, memorizza i due caratteri esadecimali. Il nibble alto (`i >> 4`) diventa il primo carattere, il nibble basso (`i & 0xF`) diventa il secondo. Il byte `0xAB` mappa su `{'a', 'b'}`.

Convertire l'intero digest è un loop stretto:

```cpp
std::string hex(len * 2, '\0');
for (unsigned int i = 0; i < len; ++i) {
    hex[i * 2]     = HEX_TABLE[digest[i]][0];
    hex[i * 2 + 1] = HEX_TABLE[digest[i]][1];
}
```

Un'allocazione per la stringa di output (dimensione nota in anticipo), due lookup su array per byte. Questo è lo stesso approccio usato da hashcat.

## Auto-rilevamento dell'hash

`HashDetector::detect()` identifica l'algoritmo di hash a partire dalla stringa esadecimale:

```cpp
std::expected<HashType, CrackError> HashDetector::detect(std::string_view hash) {
    // Validate all characters are hexadecimal
    if (!std::ranges::all_of(hash, is_hex)) {
        return std::unexpected(CrackError::InvalidHash);
    }

    // Match length to algorithm
    switch (hash.size()) {
        case config::MD5_HEX_LENGTH:    return HashType::MD5;
        case config::SHA1_HEX_LENGTH:   return HashType::SHA1;
        case config::SHA256_HEX_LENGTH: return HashType::SHA256;
        case config::SHA512_HEX_LENGTH: return HashType::SHA512;
        default: return std::unexpected(CrackError::InvalidHash);
    }
}
```

Questo funziona perché ogni algoritmo produce una lunghezza di digest univoca. MD5 è sempre 32 caratteri esadecimali, SHA1 è sempre 40, SHA256 è 64, SHA512 è 128. La validazione esadecimale intercetta i refusi e l'input non-hash prima che l'esecuzione di cracking sprechi tempo.

## Dictionary Attack con mmap

### Memory mapping del file

Invece di leggere la wordlist riga per riga con `std::ifstream` (che copia i dati dallo spazio kernel allo spazio user a ogni lettura), mappiamo l'intero file nello spazio di indirizzamento del processo:

```cpp
auto* mapped = static_cast<const char*>(
    mmap(nullptr, file_size, PROT_READ, MAP_PRIVATE, fd, 0));
madvise(mapped, file_size, MADV_SEQUENTIAL);
```

Dopo questa chiamata, `mapped` è un puntatore ai contenuti del file in memoria. Leggere una parola è semplice aritmetica dei puntatori. L'hint `MADV_SEQUENTIAL` dice al kernel di fare il prefetch delle pagine oltre la posizione di lettura corrente poiché stiamo scansionando linearmente.

### Partizionamento dei thread

Per N thread, il file viene suddiviso in N range contando i newline:

```cpp
std::size_t lines_per_thread = total_lines / total_threads;
std::size_t my_start_line = thread_index * lines_per_thread + ...;
```

Ogni thread cammina in avanti attraverso il file per trovare l'offset in byte della sua riga iniziale, poi scansiona il proprio range in modo indipendente. Nessun cursore condiviso, nessun lock, nessun coordinamento tra i thread durante il loop di cracking.

### Lettura delle parole

Il metodo `next()` scansiona dall'offset corrente al newline successivo:

```cpp
std::expected<std::string, AttackComplete> DictionaryAttack::next() {
    while (current_offset_ < end_offset_) {
        // Find newline
        while (line_end < end_offset_ && file_.data()[line_end] != '\n') {
            ++line_end;
        }

        // Strip \r for Windows line endings
        if (word_end > line_start && file_.data()[word_end - 1] == '\r') {
            --word_end;
        }

        // Skip empty lines (iterative, not recursive)
        if (word_end > line_start) {
            return std::string(file_.data() + line_start, word_end - line_start);
        }
    }
    return std::unexpected(AttackComplete{});
}
```

Le righe vuote vengono saltate in modo iterativo. Una versione precedente usava la ricorsione (`return next()`), ma una wordlist con molte righe vuote consecutive potrebbe teoricamente mandare in overflow lo stack.

## Generazione del key space per il Brute Force

### Calcolo del key space totale

Per un charset di dimensione C e lunghezza massima L:

```cpp
std::size_t compute_keyspace(std::size_t charset_size, std::size_t max_length) {
    std::size_t total = 0;
    std::size_t power = 1;
    for (std::size_t len = 1; len <= max_length; ++len) {
        power *= charset_size;
        total += power;
    }
    return total;
}
```

Questo calcola `C + C^2 + C^3 + ... + C^L`. Per le lettere minuscole (C=26) ed L=4, è 26 + 676 + 17576 + 456976 = 475.254.

### Conversione da indice a candidato

Ogni candidato ha un indice piatto univoco nel range `[0, total)`. Convertire un indice in una stringa è come convertire un numero in una base a lunghezza variabile:

```cpp
std::string index_to_candidate(std::size_t index) const {
    // Determine which length bucket this index falls in
    std::size_t cumulative = 0;
    std::size_t power = base;
    std::size_t length = 1;
    while (cumulative + power <= index && length < max_length_) {
        cumulative += power;
        ++length;
        power *= base;
    }

    // Convert the offset within that bucket to characters
    std::size_t offset = index - cumulative;
    std::string result(length, charset_[0]);
    for (std::size_t i = length; i > 0; --i) {
        result[i - 1] = charset_[offset % base];
        offset /= base;
    }
    return result;
}
```

Questo è lo stesso algoritmo della conversione di un numero decimale in base-N, tranne che le "cifre" sono caratteri del charset. L'indice 0 mappa sulla prima stringa a singolo carattere, e gli indici aumentano attraverso tutte le stringhe a singolo carattere, poi tutte le stringhe a due caratteri, e così via.

## Mutazioni basate su regole con std::generator

### Regole individuali

Ogni regola è una coroutine che restituisce (yield) le mutazioni in modo lazy:

```cpp
std::generator<std::string> RuleSet::leet_speak(std::string_view word) {
    std::string result(word);
    for (auto& c : result) {
        auto lower = static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
        for (auto [from, to] : LEET_MAP) {
            if (lower == from) { c = to; break; }
        }
    }
    co_yield std::move(result);
}
```

La keyword `co_yield` sospende la funzione e ritorna il valore. Quando il chiamante chiede il valore successivo, l'esecuzione riprende da dove si era interrotta. Per regole semplici come il leet speak, c'è un solo yield. Per append_digits, ci sono 1000 yield (uno per cifra da 0 a 999).

### Composizione delle regole

`apply_all()` delega a ciascuna regola usando `std::ranges::elements_of`:

```cpp
std::generator<std::string> RuleSet::apply_all(std::string_view word) {
    co_yield std::ranges::elements_of(capitalize_first(word));
    co_yield std::ranges::elements_of(uppercase_all(word));
    co_yield std::ranges::elements_of(leet_speak(word));
    co_yield std::ranges::elements_of(append_digits(word));
    co_yield std::ranges::elements_of(prepend_digits(word));
    co_yield std::ranges::elements_of(reverse(word));
    co_yield std::ranges::elements_of(toggle_case(word));
}
```

`elements_of` è una feature del C++23 per delegare ai sub-generator. Senza di essa, scriveresti `for (auto&& s : sub_gen) { co_yield std::move(s); }` per ogni regola, che è prolisso e ha un overhead O(depth) per elemento.

### La leet map

La tabella di sostituzione usa un array constexpr invece di `std::unordered_map`:

```cpp
static constexpr std::array<std::pair<char, char>, 6> LEET_MAP = {{
    {'a', '@'}, {'e', '3'}, {'i', '1'},
    {'o', '0'}, {'s', '$'}, {'t', '7'}
}};
```

Sei voci. Una scansione lineare su 6 elementi è più veloce del calcolo di un hash, della ricerca di un bucket e del dereferenziamento di un puntatore a nodo. L'`unordered_map` allocherebbe anche sullo heap al momento della costruzione, cosa che l'array constexpr evita completamente.

## Il template Engine

L'Engine collega tutto insieme. È una funzione template statica in un header perché deve essere istanziata per ogni combinazione hasher/attacco:

```cpp
template <Hasher H, AttackStrategy A>
auto Engine::crack(const CrackConfig& cfg)
    -> std::expected<CrackResult, CrackError>
```

### La lambda worker

Ogni thread esegue una lambda che crea la propria partizione di attacco, la propria istanza di hasher e cicla finché non trova una corrispondenza o esaurisce i candidati:

```cpp
pool.run([&](unsigned tid, unsigned total, SharedState& state) {
    H hasher;
    auto attack = create_attack();  // partitioned for this thread

    std::size_t local_count = 0;
    while (!state.found.load(std::memory_order_relaxed)) {
        auto candidate = attack->next();
        if (!candidate.has_value()) { break; }

        std::string to_hash = *candidate;
        // Apply salt if configured...

        if (hasher.hash(to_hash) == cfg.target_hash) {
            state.tested_count.fetch_add(local_count, std::memory_order_relaxed);
            state.set_result(std::move(*candidate));
            break;
        }

        ++local_count;
        if ((local_count & 0x3FF) == 0) {
            state.tested_count.fetch_add(local_count, std::memory_order_relaxed);
            local_count = 0;
        }
    }
    state.tested_count.fetch_add(local_count, std::memory_order_relaxed);
});
```

Dettagli chiave:
- Il memory ordering `relaxed` sul flag `found` è voluto. Non ci serve una visibilità immediata. Se un thread imposta `found` e un altro thread gira per qualche iterazione in più prima di vederlo, va bene. Il costo di un ordering più forte (memory fence a ogni iterazione) non vale la garanzia di fermarsi istantaneamente
- Il contatore locale raggruppa (batch) gli aggiornamenti atomici ogni 1024 iterazioni (`& 0x3FF` è un controllo con bitmask, più veloce del modulo). Il `fetch_add` finale dopo il loop scarica qualsiasi conteggio residuo

## Dispatch della CLI

La funzione main usa una lambda template per fare il dispatch in base al tipo di hash senza polimorfismo a runtime:

```cpp
template <Hasher H>
static auto dispatch_attack(const CrackConfig& cfg)
    -> std::expected<CrackResult, CrackError> {
    if (cfg.bruteforce) return Engine::crack<H, BruteForceAttack>(cfg);
    if (cfg.use_rules) return Engine::crack<H, RuleAttack>(cfg);
    return Engine::crack<H, DictionaryAttack>(cfg);
}

static auto dispatch_hasher(HashType type, const CrackConfig& cfg)
    -> std::expected<CrackResult, CrackError> {
    switch (type) {
        case HashType::MD5:    return dispatch_attack<MD5Hasher>(cfg);
        case HashType::SHA1:   return dispatch_attack<SHA1Hasher>(cfg);
        case HashType::SHA256: return dispatch_attack<SHA256Hasher>(cfg);
        case HashType::SHA512: return dispatch_attack<SHA512Hasher>(cfg);
    }
    return std::unexpected(CrackError::UnsupportedAlgorithm);
}
```

Lo `switch` è l'unico punto in cui viene presa una decisione a runtime. Ogni case istanzia il template Engine con un tipo di hasher concreto. Da quel punto in poi, tutto è risolto a compile-time.

## Strategia di testing

La suite di test ha 38 test organizzati per componente:

**I test dell'Hasher** verificano rispetto ai vettori di risposta nota (known-answer) del NIST. Se `SHA256Hasher::hash("password")` non produce esattamente `5e884898da28...`, l'implementazione è sbagliata. Questi sono i test più importanti perché un sottile bug nell'hashing farebbe fallire silenziosamente l'intero strumento.

**I test di HashDetector** verificano il rilevamento del tipo (basato sulla lunghezza) e la validazione dell'input (rifiuto del non-hex).

**I test di DictionaryAttack** verificano la lettura delle parole, il partizionamento dei thread (due thread insieme leggono tutte le parole), il conteggio totale e la gestione dell'errore file-not-found.

**I test di BruteForceAttack** verificano la matematica del key space, la completezza della generazione dei candidati (tutte le combinazioni prodotte) e il partizionamento dei thread (nessun duplicato, nessun buco).

**I test di RuleSet** verificano ogni regola di mutazione rispetto all'output atteso. Il test `AllRulesProduceMutations` conferma che il conteggio totale supera 2000 (7 regole applicate a "password", con append_digits e prepend_digits che producono 1000 ciascuna).

**I test dell'Engine** sono test di integrazione. `CracksSHA256WithDictionary` esegue l'intera pipeline con 2 thread e verifica che trovi "password". `CracksWithSalt` verifica che l'aggiunta del salt funzioni end-to-end.

## Insidie comuni

**Dimenticare di scaricare il contatore locale**: se un thread trova la password e fa break senza scaricare `local_count`, il conteggio finale dei tentativi sarà sbagliato. Il `fetch_add` dopo il loop gestisce questo.

**Confrontare hash con case diverso**: SHA256 potrebbe produrre esadecimali maiuscoli su alcune piattaforme. Il nostro encoder esadecimale produce sempre minuscoli, e l'hash target viene usato così com'è dalla CLI. Se qualcuno incolla un hash maiuscolo, non corrisponderà. Uno strumento di produzione reale normalizzerebbe entrambi in minuscolo.

**mmap e troncamento del file**: se il file della wordlist viene modificato mentre lo strumento è in esecuzione, il comportamento è indefinito. `MAP_PRIVATE` aiuta (otteniamo uno snapshot copy-on-write), ma per uno strumento che elabora un file una volta ed esce, questa non è una preoccupazione pratica.
