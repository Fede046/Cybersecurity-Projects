# Binary Analysis Tool: architettura

Questo modulo tratta il design di sistema di AXUMORTEM: come funziona la pipeline basata sui pass, come interagiscono i componenti e il ragionamento dietro ogni decisione di design.

## Architettura ad alto livello

```
                         ┌─────────────────────────────────────┐
                         │            Nginx (port 22784)       │
                         │    static files + reverse proxy     │
                         └──────────────┬──────────────────────┘
                                        │
                           ┌────────────┴────────────┐
                           │                         │
                      GET /index.html           POST /api/*
                      GET /assets/*             GET /api/*
                           │                         │
                           v                         v
                  ┌─────────────────┐     ┌─────────────────────┐
                  │  React Frontend │     │  Axum Backend (:3000)│
                  │  (static build) │     │                     │
                  └─────────────────┘     │  ┌───────────────┐  │
                                          │  │ Upload Route  │  │
                                          │  └───────┬───────┘  │
                                          │          │          │
                                          │  ┌───────v───────┐  │
                                          │  │AnalysisEngine │  │
                                          │  │(spawn_blocking)│  │
                                          │  └───────┬───────┘  │
                                          │          │          │
                                          │  ┌───────v───────┐  │
                                          │  │  PostgreSQL   │  │
                                          │  │  (port 5432)  │  │
                                          │  └───────────────┘  │
                                          └─────────────────────┘
```

Il frontend è una build statica di React servita da Nginx. Tutte le richieste API passano tramite proxy attraverso Nginx fino al backend Axum sulla porta 3000. Il motore di analisi esegue il lavoro CPU-intensive su un thread pool bloccante (`spawn_blocking`) per evitare di affamare il runtime asincrono. I risultati vengono persistiti su PostgreSQL, così upload ripetuti dello stesso binario restituiscono istantaneamente i risultati in cache.

## Analisi dei componenti

### Motore di analisi (crate `axumortem-engine`)

La libreria core. Nessuna conoscenza di HTTP, database o questioni di frontend. Riceve byte grezzi in input e produce risultati di analisi strutturati.

**Responsabilità:**
- Registrare e ordinare i pass di analisi
- Eseguire i pass in ordine di dipendenza
- Fornire un contesto condiviso in cui i pass memorizzano i risultati
- Calcolare gli hash dei file

**Tipi chiave:**
- `AnalysisEngine` — orchestratore di primo livello
- `AnalysisContext` — contenitore di stato mutabile passato tra i pass
- `PassManager` — risoluzione delle dipendenze ed esecuzione sequenziale
- `AnalysisPass` — trait che ogni pass implementa

### Server HTTP (crate `axumortem`)

Un sottile wrapper che espone il motore via HTTP. Gestisce gli upload dei file, la persistenza nel database e il recupero dei risultati.

**Responsabilità:**
- Accettare upload di file multipart
- Delegare l'analisi al motore (su un thread bloccante)
- Persistere i risultati su PostgreSQL
- Recuperare i risultati in cache tramite slug
- Endpoint di health check

**Route:**
- `PUT /api/upload` — carica un binario, ricevi uno slug
- `GET /api/analysis/{slug}` — recupera i risultati completi dell'analisi
- `GET /health` — controllo di liveness

### Frontend (`frontend/`)

SPA React 19 con risultati di analisi a schede. Nessuna business logic — renderizza qualunque cosa l'API restituisca.

**Responsabilità:**
- Upload dei file tramite drag-and-drop con indicatore di progresso
- Visualizzare i risultati dell'analisi su sei schede
- Renderizzare le visualizzazioni del CFG usando il layout a grafo di dagre
- Validare le risposte dell'API con schemi Zod

## Pipeline basata sui pass

La pipeline di analisi è il fulcro architetturale. Ogni capacità di analisi è un "pass" — un'unità isolata che legge dal contesto condiviso, fa il proprio lavoro e scrive i risultati indietro.

### Il trait AnalysisPass

Ogni pass implementa questo trait:

```rust
pub trait AnalysisPass: Sealed + Send + Sync {
    fn name(&self) -> &'static str;
    fn dependencies(&self) -> &[&'static str];
    fn run(
        &self,
        ctx: &mut AnalysisContext,
    ) -> Result<(), EngineError>;
}
```

`name()` restituisce un identificatore univoco. `dependencies()` restituisce i nomi dei pass che devono essere eseguiti prima. `run()` fa il lavoro vero e proprio, leggendo gli input da `ctx` e scrivendo gli output di nuovo in `ctx`.

Il supertrait `Sealed` impedisce al codice esterno di implementare `AnalysisPass`. Questo è il sealed trait pattern — solo il crate `axumortem-engine` può definire i pass:

```rust
mod private {
    pub trait Sealed {}
}

pub trait AnalysisPass: private::Sealed + Send + Sync {
    // ...
}
```

### L'AnalysisContext

Il contesto è il contenitore di stato condiviso che fluisce attraverso la pipeline:

```rust
pub struct AnalysisContext {
    source: BinarySource,
    pub sha256: String,
    pub file_name: String,
    pub file_size: u64,
    pub format_result: Option<FormatResult>,
    pub import_result: Option<ImportResult>,
    pub string_result: Option<StringResult>,
    pub entropy_result: Option<EntropyResult>,
    pub disassembly_result: Option<DisassemblyResult>,
    pub threat_result: Option<ThreatResult>,
}
```

Ogni pass verifica le proprie dipendenze tramite `Option`. Se `EntropyPass` ha bisogno dei dati del formato, chiama `ctx.format_result.as_ref().ok_or_else(...)`. Se la dipendenza non c'è, il pass fallisce con un errore `MissingDependency`.

`BinarySource` supporta due modalità:

```rust
pub enum BinarySource {
    Mapped(Mmap),
    Buffered(Arc<[u8]>),
}
```

File memory-mapped per i binari grandi analizzati da disco. Array di byte in buffer per i binari ricevuti via HTTP. Entrambi implementano `AsRef<[u8]>`, quindi ai pass non importa quale modalità sia attiva.

### Ordinamento topologico

Il PassManager ordina i pass usando l'algoritmo di Kahn (ordinamento topologico basato su BFS). Questo garantisce che ogni pass venga eseguito dopo le proprie dipendenze, indipendentemente dall'ordine di registrazione:

```rust
fn topological_order(
    passes: &[Box<dyn AnalysisPass>],
) -> Vec<usize> {
    let name_to_idx: HashMap<&str, usize> = passes
        .iter()
        .enumerate()
        .map(|(i, p)| (p.name(), i))
        .collect();

    let n = passes.len();
    let mut in_degree = vec![0usize; n];
    let mut adjacency: Vec<Vec<usize>> = vec![vec![]; n];

    for (idx, pass) in passes.iter().enumerate() {
        for dep_name in pass.dependencies() {
            if let Some(&dep_idx) = name_to_idx.get(dep_name) {
                adjacency[dep_idx].push(idx);
                in_degree[idx] += 1;
            }
        }
    }

    let mut queue: VecDeque<usize> = in_degree
        .iter()
        .enumerate()
        .filter(|&(_, deg)| *deg == 0)
        .map(|(i, _)| i)
        .collect();

    let mut order = Vec::with_capacity(n);

    while let Some(node) = queue.pop_front() {
        order.push(node);
        for &neighbor in &adjacency[node] {
            in_degree[neighbor] -= 1;
            if in_degree[neighbor] == 0 {
                queue.push_back(neighbor);
            }
        }
    }

    assert_eq!(
        order.len(), n,
        "cycle detected in pass dependencies"
    );

    order
}
```

L'algoritmo costruisce una lista di adiacenza dalle dichiarazioni di dipendenza, calcola gli in-degree, poi elabora per primi i nodi con in-degree zero. Se l'ordine finale non include tutti i pass, c'è un ciclo — che causa un panic al momento della costruzione del motore invece di fallire silenziosamente al momento dell'analisi.

### Grafo delle dipendenze dei pass

```
  FormatPass (no dependencies)
       │
       ├──────────┬──────────┬──────────┐
       │          │          │          │
       v          v          v          v
  ImportPass  StringPass  EntropyPass  DisasmPass
       │          │          │
       │          │          │
       └──────────┴──────────┘
                  │
                  v
             ThreatPass (depends on all previous)
```

`FormatPass` viene eseguito per primo perché non ha dipendenze. I quattro pass intermedi dipendono tutti da `FormatPass` ma non l'uno dall'altro. `ThreatPass` dipende da import, stringhe ed entropia (legge i loro risultati per calcolare il punteggio).

### Esecuzione e gestione degli errori

Il PassManager esegue tutti i pass in ordine, misurando il tempo di ciascuno. Se un pass fallisce, l'errore viene registrato ma l'esecuzione continua — un fallimento nel disassembly non dovrebbe impedire al threat scoring di essere eseguito sui dati di import ed entropia già disponibili:

```rust
pub fn run_all(
    &self,
    ctx: &mut AnalysisContext,
) -> PassReport {
    let mut outcomes = Vec::with_capacity(self.passes.len());

    for &idx in &self.order {
        let pass = &self.passes[idx];
        let start = Instant::now();
        let result = pass.run(ctx);
        let duration_ms = start.elapsed().as_millis() as u64;

        let outcome = match result {
            Ok(()) => PassOutcome {
                name: pass.name(),
                success: true,
                duration_ms,
                error_message: None,
            },
            Err(e) => PassOutcome {
                name: pass.name(),
                success: false,
                duration_ms,
                error_message: Some(e.to_string()),
            },
        };

        outcomes.push(outcome);
    }

    PassReport { outcomes }
}
```

`PassReport` raccoglie ogni esito così che il chiamante possa ispezionare cosa è riuscito, cosa è fallito e quanto tempo ha impiegato ogni pass.

## Flusso dei dati

### Flusso di upload

```
Client                    Axum                     Engine              PostgreSQL
  │                         │                        │                     │
  │── PUT /api/upload ─────>│                        │                     │
  │   (multipart file)      │                        │                     │
  │                         │── sha256_hex(data) ──> │                     │
  │                         │                        │                     │
  │                         │── find_slug_by_sha256 ─┼────────────────────>│
  │                         │<── Some(slug) ─────────┼────────────────────┤│
  │<── { slug, cached: true }                        │       (if exists)  │
  │                         │                        │                     │
  │                         │  (if not cached)       │                     │
  │                         │── spawn_blocking ─────>│                     │
  │                         │                        │── FormatPass.run()  │
  │                         │                        │── ImportPass.run()  │
  │                         │                        │── StringPass.run()  │
  │                         │                        │── EntropyPass.run() │
  │                         │                        │── DisasmPass.run()  │
  │                         │                        │── ThreatPass.run()  │
  │                         │<── (ctx, report) ──────│                     │
  │                         │                        │                     │
  │                         │── BEGIN TRANSACTION ───┼────────────────────>│
  │                         │── insert_analysis ─────┼────────────────────>│
  │                         │── insert_pass_result ──┼──── (x6) ─────────>│
  │                         │── COMMIT ──────────────┼────────────────────>│
  │                         │                        │                     │
  │<── { slug, cached: false }                       │                     │
```

Il pattern check-and-return sullo SHA-256 significa che binari identici non vengono mai analizzati due volte. Il primo upload esegue l'intera pipeline (che può richiedere secondi per binari grandi), ma ogni successivo upload dello stesso file ritorna istantaneamente dalla cache.

La chiamata `spawn_blocking` è critica. L'analisi è CPU-bound (parsing, calcolo dell'entropia, disassembly). Eseguirla sull'executor asincrono bloccherebbe le altre richieste. `spawn_blocking` sposta il lavoro sul thread pool bloccante di Tokio.

### Flusso di recupero

```
Client                    Axum                    PostgreSQL
  │                         │                         │
  │── GET /analysis/{slug} ─>│                         │
  │                         │── fetch_by_slug ────────>│
  │                         │<── AnalysisRow + ────────│
  │                         │    Vec<PassResultRow>    │
  │                         │                         │
  │                         │── deserialize JSON       │
  │                         │   per pass_name          │
  │                         │                         │
  │<── AnalysisResponse ────│                         │
```

Il risultato di ogni pass è memorizzato come colonna JSONB. L'API ricostruisce la risposta tipizzata facendo corrispondere `pass_name` al deserializzatore appropriato.

## Design pattern

### Sealed trait pattern

**Dove:** `pass.rs`

**Cosa:** il trait `AnalysisPass` richiede `Sealed`, che è definito in un blocco `mod private`. Solo i tipi all'interno del crate `axumortem-engine` possono implementare `Sealed`, il che significa che solo questo crate può definire pass di analisi.

**Perché:** questo impedisce ai consumatori a valle di creare i propri pass che potrebbero violare gli invarianti (come dichiarare dipendenze circolari o scrivere in campi del contesto in cui non dovrebbero). Se in futuro servirà estensibilità, potrà essere aggiunta tramite un sistema di plugin con una validazione adeguata invece che tramite implementazioni del trait incontrollate.

**Compromesso:** meno flessibile di un trait aperto, ma la protezione degli invarianti ne vale la pena per uno strumento di sicurezza in cui la correttezza dell'ordinamento dei pass conta.

### Context Object pattern

**Dove:** `context.rs`

**Cosa:** `AnalysisContext` è una struct mutabile passata attraverso l'intera pipeline. Ogni pass legge ciò di cui ha bisogno e scrive i propri risultati.

**Perché:** i pass devono condividere dati senza conoscersi direttamente. Il contesto funge da lavagna tipizzata (blackboard). Usare `Option<T>` per ogni campo di risultato significa che i pass possono verificare a runtime se le loro dipendenze hanno prodotto un output.

**Alternativa considerata:** restituire i risultati da ogni pass e passarli esplicitamente. Scartata perché crea un accoppiamento stretto tra i pass e fa sì che l'aggiunta di nuovi pass richieda di cambiare la firma di tipo dell'orchestratore.

### Registrazione tramite factory

**Dove:** `lib.rs`

**Cosa:** tutti i pass sono registrati in un singolo `vec![]` in `AnalysisEngine::new()`:

```rust
let passes: Vec<Box<dyn AnalysisPass>> = vec![
    Box::new(FormatPass),
    Box::new(ImportPass),
    Box::new(StringPass),
    Box::new(EntropyPass),
    Box::new(DisasmPass),
    Box::new(ThreatPass),
];
```

**Perché:** un unico punto di registrazione. Aggiungere un nuovo pass significa aggiungere una riga qui più l'implementazione del pass. L'ordinamento topologico gestisce l'ordine automaticamente — non devi preoccuparti dell'ordine di inserimento.

### Serializzazione basata su macro

**Dove:** `upload.rs`

**Cosa:** la macro `add_pass!` genera la serializzazione dei risultati per tutti e sei i pass:

```rust
macro_rules! add_pass {
    ($field:ident, $name:expr) => {
        if let Some(ref r) = ctx.$field {
            results.push(NewPassResult {
                analysis_id,
                pass_name: api_name($name).to_string(),
                result: serde_json::to_value(r)?,
                duration_ms: durations.get($name).map(|&d| d as i32),
            });
        }
    };
}

add_pass!(format_result, "format");
add_pass!(import_result, "imports");
add_pass!(string_result, "strings");
add_pass!(entropy_result, "entropy");
add_pass!(disassembly_result, "disasm");
add_pass!(threat_result, "threat");
```

**Perché:** il risultato di ogni pass ha bisogno dello stesso trattamento: verificare se esiste, serializzarlo in JSON, associarlo alla sua durata. La macro elimina sei blocchi di codice quasi identici.

## Modelli di dati

### Schema del database

```sql
CREATE TABLE analyses (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sha256      TEXT NOT NULL UNIQUE,
    file_name   TEXT NOT NULL,
    file_size   BIGINT NOT NULL,
    format      TEXT NOT NULL DEFAULT '',
    architecture TEXT NOT NULL DEFAULT '',
    entry_point BIGINT,
    threat_score INTEGER,
    risk_level  TEXT,
    slug        TEXT NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pass_results (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id UUID NOT NULL REFERENCES analyses(id) ON DELETE CASCADE,
    pass_name   TEXT NOT NULL,
    result      JSONB NOT NULL,
    duration_ms INTEGER
);
```

Due tabelle. `analyses` contiene i metadati di riepilogo (ciò che mostreresti in una vista a lista). `pass_results` contiene l'output dettagliato di ogni pass come JSONB. Questa separazione significa che elencare le analisi recenti è una query indicizzata veloce, mentre i risultati dettagliati vengono caricati su richiesta.

La colonna `sha256` ha un vincolo `UNIQUE` — questa è la chiave di deduplicazione. Lo `slug` (i primi 12 caratteri dello SHA-256) funge da identificatore URL leggibile.

### Sistema di tipi del motore

Il motore usa ampiamente gli enum di Rust per la type safety:

```rust
pub enum BinaryFormat { Elf, Pe, MachO }
pub enum Architecture { X86, X86_64, Arm, Aarch64, Other(String) }
pub enum RiskLevel { Benign, Low, Medium, High, Critical }
pub enum EntropyClassification { Plaintext, NativeCode, Compressed, Packed, Encrypted }
pub enum FlowControlType { Next, Branch, ConditionalBranch, Call, Return, Interrupt }
pub enum CfgEdgeType { Fallthrough, ConditionalTrue, ConditionalFalse, Unconditional, Call }
```

Ogni enum deriva `Serialize` e `Deserialize`. Il motore non ha mai a che fare con stringhe grezze per questi concetti — il pattern matching impone una gestione esaustiva.

## Architettura di sicurezza

### Threat model

**Cosa proteggiamo:**
- Il server di analisi da upload di binari malevoli (il binario è input non fidato)
- I risultati dell'analisi dalla manomissione (verifica dell'integrità tramite SHA-256)
- Il database dall'injection (query SQLx verificate a compile-time)

**Da cosa non proteggiamo:**
- Evasione dall'analisi dinamica (AXUMORTEM è solo statico — non esegue mai il binario)
- Denial of service da binari estremamente grandi (mitigato dal limite di upload di 50MB)
- Attacchi side-channel basati sui tempi di analisi (le durate dei pass sono esposte nell'API)

### Livelli di difesa

1. **Limite di dimensione dell'upload** — `MAX_UPLOAD_SIZE` (default 50MB) previene l'esaurimento della memoria
2. **Isolamento su thread bloccanti** — l'analisi CPU-intensive viene eseguita su `spawn_blocking`, non sul runtime asincrono
3. **Limiti su istruzioni/funzioni** — i limiti del disassembly impediscono a binari patologici di causare loop infiniti
4. **Prevenzione della SQL injection** — tutte le query usano statement parametrizzati di SQLx verificati a compile-time
5. **Configurazione CORS** — restrizioni sulle origini configurabili
6. **Deduplicazione** — il controllo SHA-256 impedisce di ri-analizzare binari noti (e previene anche calcoli costosi ripetuti)

## Considerazioni sulle prestazioni

### Colli di bottiglia

1. **Disassembly** — la discesa ricorsiva attraverso le istruzioni x86 è il pass più lento. Il limite di 50k istruzioni mantiene limitato il caso peggiore.
2. **Scansione YARA** — compilare le regole è costoso, ma AXUMORTEM le compila una volta all'avvio del motore e riutilizza lo scanner compilato.
3. **Calcolo dell'entropia** — l'entropia di Shannon richiede di iterare su ogni byte. Per un binario da 50MB, sono 50 milioni di lookup di frequenza dei byte.

### Ottimizzazioni

- **Deduplicazione SHA-256** — l'ottimizzazione con il maggiore impatto. Nessun lavoro di analisi per i binari già visti.
- **YARA compilato una volta** — la compilazione delle regole avviene in `YaraScanner::new()`, non per ogni analisi.
- **Entropia a livello di sezione** — invece dell'entropia a finestra scorrevole byte per byte (che è O(n * window_size)), AXUMORTEM calcola l'entropia per sezione, che è O(total_bytes) con un fattore costante di 256 (un contatore per ogni possibile valore di byte).
- **`spawn_blocking`** — mantiene reattivo il runtime asincrono mentre l'analisi è in esecuzione.

### Scalabilità

**Verticale:** aumenta `TOKIO_WORKER_THREADS` e la dimensione del thread pool bloccante. Ogni analisi è indipendente, quindi più core significano più analisi concorrenti.

**Orizzontale:** il motore è stateless — l'unico stato condiviso è PostgreSQL. Più istanze di backend dietro un load balancer funzionerebbero senza modifiche al codice. La deduplicazione SHA-256 nel database previene il lavoro duplicato anche tra istanze diverse.

## Decisioni di design

### Perché una pipeline basata sui pass invece di un analizzatore monolitico?

**Decisione:** ogni capacità di analisi è un pass indipendente con dipendenze dichiarate.

**Alternative considerate:**
- Funzione monolitica che fa tutto in sequenza
- Architettura event-driven con message passing tra gli analizzatori

**Perché hanno vinto i pass:** modularità (aggiungere/rimuovere capacità senza toccare le altre), testabilità (ogni pass ha unit test isolati), risultati parziali (se il disassembly fallisce, ottieni comunque l'analisi degli import) e una gestione chiara delle dipendenze (l'ordinamento topologico gestisce l'ordine).

### Perché Rust per il motore?

**Decisione:** Rust per il motore di analisi, che è il componente critico per le prestazioni.

**Perché:** il parsing dei binari è un territorio intrinsecamente non sicuro — stai interpretando byte non fidati come dati strutturati. Il modello di ownership di Rust previene buffer overflow, use-after-free e data race che affliggono gli strumenti di analisi dei binari in C/C++. Il crate `goblin` gestisce il parsing dei formati in modo sicuro. `iced-x86` fornisce un disassembly ad alte prestazioni. E `yara-x` (la riscrittura ufficiale di YARA in Rust) elimina l'overhead di FFI.

### Perché memorizzare i risultati dei pass come JSONB?

**Decisione:** il risultato di ogni pass è serializzato in JSONB in una singola colonna.

**Alternative considerate:**
- Tabelle normalizzate per ogni pass (`entropy_results`, `import_results` separate, ecc.)
- Un database a documenti (MongoDB, SurrealDB)

**Perché JSONB:** gli schemi dei risultati dei pass cambiano man mano che i pass evolvono. Aggiungere un campo a `EntropyResult` richiede zero migrazioni del database — la colonna JSONB accetta qualunque cosa produca il serializzatore. Le tabelle normalizzate richiederebbero una migrazione per ogni cambio di schema. L'indicizzazione JSONB di PostgreSQL significa che le query nei risultati dei pass restano comunque veloci quando servono.

### Perché uno slug SHA-256 invece di un auto-increment?

**Decisione:** gli URL usano i primi 12 caratteri esadecimali dell'hash SHA-256 (`/analysis/a1b2c3d4e5f6`).

**Perché:** deterministico — lo stesso binario produce sempre lo stesso URL. Nessun ID sequenziale da enumerare. Lo slug esadecimale di 12 caratteri offre 48 bit di resistenza alle collisioni (2^48 = ~281 trilioni di valori possibili), che è più che sufficiente per uno strumento a istanza singola.

## Architettura di deploy

### Produzione

```
┌──────────────────────────────────────────────────┐
│ Docker Compose                                   │
│                                                  │
│  ┌──────────────────────┐                        │
│  │ Nginx (:22784)       │                        │
│  │ - serves static SPA  │                        │
│  │ - proxies /api → :3000                        │
│  └──────────┬───────────┘                        │
│             │                                    │
│  ┌──────────v───────────┐  ┌──────────────────┐  │
│  │ axumortem (:3000)    │  │ PostgreSQL       │  │
│  │ - Axum HTTP server   │──│ (:5432)          │  │
│  │ - analysis engine    │  │ - analyses table │  │
│  └──────────────────────┘  │ - pass_results   │  │
│                            └──────────────────┘  │
└──────────────────────────────────────────────────┘
```

### Sviluppo

```
┌──────────────────────────────────────────────────┐
│ Docker Compose (dev)                             │
│                                                  │
│  ┌──────────────────────┐                        │
│  │ Vite (:15723)        │                        │
│  │ - HMR dev server     │                        │
│  │ - proxies /api → :3000                        │
│  └──────────────────────┘                        │
│                                                  │
│  ┌──────────────────────┐  ┌──────────────────┐  │
│  │ axumortem (:3000)    │  │ PostgreSQL       │  │
│  │ - cargo watch        │──│ (:5432)          │  │
│  │ - auto-rebuild       │  └──────────────────┘  │
│  └──────────────────────┘                        │
└──────────────────────────────────────────────────┘
```

Il setup di sviluppo sostituisce Nginx con il dev server di Vite (con HMR) e aggiunge cargo watch per le rebuild automatiche del backend.

## Estensibilità

### Aggiungere un nuovo pass

1. Crea `passes/newpass.rs` con una struct che implementa `Sealed` e `AnalysisPass`
2. Aggiungi un campo di risultato ad `AnalysisContext` (ad esempio, `pub newpass_result: Option<NewPassResult>`)
3. Registra il pass in `AnalysisEngine::new()` aggiungendo `Box::new(NewPass)` al vec
4. L'ordinamento topologico gestisce l'ordine automaticamente in base al valore restituito da `dependencies()`

### Aggiungere un nuovo formato binario

1. Crea `formats/newformat.rs` con il parser
2. Aggiungi una variante all'enum `BinaryFormat`
3. Aggiungi il controllo dei magic byte e il dispatch in `formats::parse_format()`
4. Aggiungi una struct di informazioni specifiche del formato se necessario (come `PeInfo`, `ElfInfo`)

## Limitazioni

- **Solo analisi statica** — nessuna esecuzione in sandbox, nessun monitoraggio comportamentale. Un binario che decifra il proprio payload a runtime mostrerà un'entropia da dati cifrati ma nessun codice spacchettato.
- **Disassembly solo x86/x86_64** — i binari ARM, MIPS e RISC-V ricevono tutti gli altri pass ma nessun disassembly né CFG.
- **Nessuna analisi dei riferimenti incrociati** — il disassembler non traccia i riferimenti ai dati (quali funzioni leggono quali stringhe), solo il flusso di controllo.
- **Analisi di singoli file** — non può analizzare pacchetti multi-binario (installer MSI, APK con librerie native) come un'unità.
- **Nessuna analisi incrementale** — cambiare le regole YARA richiede di ri-analizzare tutti i binari. Non c'è modo di rieseguire un solo pass sui dati in cache.

## Prossimi passi

Prosegui con [03 - Implementazione](03-IMPLEMENTATION.it.md) per una panoramica a livello di codice di come è costruito ogni pass, con snippet reali dal codebase.
