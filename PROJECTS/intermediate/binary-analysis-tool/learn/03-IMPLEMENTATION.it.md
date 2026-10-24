# Binary Analysis Tool: implementazione

Questo modulo illustra come viene costruito ogni componente principale, utilizzando codice reale estratto dalla codebase. Tracceremo la pipeline di analisi dall'upload del binario fino al threat score.

## Panoramica della struttura dei file

```
backend/crates/
├── axumortem-engine/src/
│   ├── lib.rs              Engine orchestrator — registers passes, computes SHA-256
│   ├── types.rs            Enums shared across passes (BinaryFormat, RiskLevel, etc.)
│   ├── context.rs          AnalysisContext — the mutable state bag for the pipeline
│   ├── pass.rs             AnalysisPass trait, PassManager, topological sort
│   ├── error.rs            EngineError enum (parse failures, missing dependencies)
│   ├── yara.rs             YaraScanner with 14 built-in detection rules
│   ├── formats/
│   │   ├── mod.rs          FormatResult structs, anomaly enums, parse dispatch
│   │   ├── elf.rs          ELF parser (sections, segments, RELRO, PIE detection)
│   │   ├── pe.rs           PE parser (imports, TLS callbacks, overlay detection)
│   │   └── macho.rs        Mach-O parser (universal binaries, code signatures)
│   └── passes/
│       ├── format.rs       Pass 1 — binary format identification
│       ├── imports.rs      Pass 2 — import/export extraction + suspicious API flagging
│       ├── strings.rs      Pass 3 — string extraction in ASCII and UTF-16LE
│       ├── entropy.rs      Pass 4 — Shannon entropy + packer detection
│       ├── disasm.rs       Pass 5 — x86/x86_64 disassembly + CFG construction
│       └── threat.rs       Pass 6 — 8-category scoring + MITRE mapping
│
└── axumortem/src/
    ├── main.rs             Server startup, signal handling, graceful shutdown
    ├── config.rs           CLI args + environment configuration
    ├── state.rs            AppState (database pool, engine Arc, config)
    ├── error.rs            API error types → HTTP status codes
    ├── routes/
    │   ├── upload.rs       PUT /api/upload — multipart handling + analysis dispatch
    │   ├── analysis.rs     GET /api/analysis/{slug} — result retrieval
    │   └── health.rs       GET /health — liveness probe
    ├── db/
    │   ├── models.rs       Database row structs (AnalysisRow, PassResultRow)
    │   └── queries.rs      SQL operations (insert, find by SHA-256, find by slug)
    └── middleware/
        └── cors.rs         CORS layer configuration
```

## Costruzione del motore

Il motore è il punto di ingresso che collega tutto insieme. Quando viene chiamato `AnalysisEngine::new()`, crea tutti e sei i pass e li passa al `PassManager`, che li ordina topologicamente:

```rust
pub struct AnalysisEngine {
    pass_manager: PassManager,
}

impl AnalysisEngine {
    pub fn new() -> Result<Self, EngineError> {
        let passes: Vec<Box<dyn pass::AnalysisPass>> =
            vec![
                Box::new(FormatPass),
                Box::new(ImportPass),
                Box::new(StringPass),
                Box::new(EntropyPass),
                Box::new(DisasmPass),
                Box::new(ThreatPass),
            ];

        let pass_manager = PassManager::new(passes);

        Ok(Self { pass_manager })
    }
}
```

L'ordine di registrazione non ha importanza — i pass possono essere elencati in qualsiasi ordine perché l'ordinamento topologico risolve l'ordine di esecuzione a partire dalle loro dipendenze dichiarate. `ThreatPass` potrebbe essere il primo nel vec e verrebbe comunque eseguito per ultimo, perché dipende da tutto il resto.

Il metodo `analyze` crea un nuovo `AnalysisContext`, esegue tutti i pass e restituisce sia il contesto (con i risultati) sia un report (con informazioni sui tempi e sugli errori):

```rust
pub fn analyze(
    &self,
    data: &[u8],
    file_name: &str,
) -> (AnalysisContext, PassReport) {
    let sha256 = compute_sha256(data);
    let file_size = data.len() as u64;
    let mut ctx = AnalysisContext::new(
        BinarySource::Buffered(Arc::from(data.to_vec())),
        sha256,
        file_name.to_string(),
        file_size,
    );
    let report = self.pass_manager.run_all(&mut ctx);
    (ctx, report)
}
```

I dati binari sono avvolti in `Arc<[u8]>` in modo da poter essere condivisi senza essere copiati. La variante `BinarySource::Buffered` viene usata per gli upload HTTP; `BinarySource::Mapped` verrebbe usata per i file mappati in memoria da disco.

## Costruzione del parser del formato

Il pass del formato è il più semplice — delega a `formats::parse_format()`:

```rust
pub struct FormatPass;

impl AnalysisPass for FormatPass {
    fn name(&self) -> &'static str {
        "format"
    }

    fn dependencies(&self) -> &[&'static str] {
        &[]
    }

    fn run(
        &self,
        ctx: &mut AnalysisContext,
    ) -> Result<(), EngineError> {
        let result = formats::parse_format(ctx.data())?;
        ctx.format_result = Some(result);
        Ok(())
    }
}
```

Nessuna dipendenza — questo pass viene sempre eseguito per primo. La funzione `parse_format` usa il rilevamento automatico di `goblin` per identificare il formato dai magic byte ed effettuare il dispatch verso il parser appropriato (ELF, PE o Mach-O).

Il risultato include il rilevamento delle anomalie strutturali. L'enum delle anomalie copre specifici campanelli d'allarme:

```rust
pub enum FormatAnomaly {
    EntryPointOutsideText { ep: u64, text_range: (u64, u64) },
    EntryPointInLastSection { ep: u64, section: String },
    EntryPointOutsideSections { ep: u64 },
    RwxSection { name: String },
    EmptySectionName { index: usize },
    StrippedBinary,
    SuspiciousSectionName { name: String, reason: String },
    HighVirtualToRawRatio { section: String, ratio: f64 },
    TlsCallbackPresent { count: usize },
    OverlayPresent { offset: u64, size: u64 },
}
```

Ogni variante di anomalia trasporta le proprie evidenze — l'indirizzo dell'entry point, il nome della sezione, il valore del rapporto. Non si tratta solo di "qualcosa non va", ma di "ecco esattamente cosa non va e i numeri che lo dimostrano".

### Hashing delle sezioni

Ogni sezione riceve il proprio hash SHA-256. Ciò consente il diffing a livello di sezione tra binari — se due binari condividono lo stesso hash di `.text` ma hanno hash di `.rsrc` diversi, la modifica è avvenuta nella sezione delle risorse (comune nel software trojanizzato in cui il codice è identico ma il payload è incorporato nelle risorse).

## Costruzione del pass di analisi degli import

Il pass degli import estrae gli import delle funzioni, segnala le API sospette e rileva le combinazioni pericolose:

```rust
pub struct SuspiciousApiDef {
    pub name: &'static str,
    pub tag: &'static str,
    pub mitre_id: &'static str,
}

pub const SUSPICIOUS_APIS: &[SuspiciousApiDef] = &[
    SuspiciousApiDef {
        name: "VirtualAllocEx",
        tag: "injection",
        mitre_id: "T1055",
    },
    SuspiciousApiDef {
        name: "WriteProcessMemory",
        tag: "injection",
        mitre_id: "T1055",
    },
    SuspiciousApiDef {
        name: "CreateRemoteThread",
        tag: "injection",
        mitre_id: "T1055",
    },
    SuspiciousApiDef {
        name: "IsDebuggerPresent",
        tag: "anti-debug",
        mitre_id: "T1622",
    },
    // ... 22 suspicious APIs total
];
```

Ogni definizione di API include un `tag` per il raggruppamento (injection, hollowing, anti-debug, persistence) e un ID di tecnica MITRE ATT&CK. Il pass confronta i nomi degli import con questo elenco e controlla anche la presenza di combinazioni — gruppi di API che insieme indicano una specifica tecnica di attacco.

Il rilevamento delle combinazioni funziona raccogliendo tutte le API importate in un `HashSet` e verificando poi se specifici gruppi sono tutti presenti. La catena di process injection richiede che tre API specifiche siano tutte presenti nello stesso binario. Trovarne solo una è degno di nota; trovarle tutte e tre è un forte indicatore.

### Gestione degli import su Linux

I binari ELF di Linux non hanno una tabella degli import in stile Windows. Al contrario, il pass estrae i simboli dinamici e li confronta con le API sospette specifiche di Linux: `ptrace` (debugging/injection dei processi), `mmap` + `mprotect` (generazione di codice a runtime), `dlopen` + `dlsym` (caricamento dinamico) e API di rete combinate con `execve` (pattern di reverse shell).

## Costruzione del pass di estrazione delle stringhe

L'estrazione delle stringhe è ingannevolmente complessa. Il pass gestisce due codifiche, 14 categorie e molteplici strategie di rilevamento:

```rust
pub struct ExtractedString {
    pub value: String,
    pub offset: u64,
    pub encoding: StringEncoding,
    pub length: usize,
    pub category: StringCategory,
    pub is_suspicious: bool,
    pub section: Option<String>,
}
```

Ogni stringa riceve un offset (posizione nel binario), codifica, categoria, flag di sospetto e la sezione in cui è stata trovata (se ricade all'interno del confine di una sezione nota).

### Estrazione ASCII

L'estrattore ASCII esamina ogni byte del binario, costruendo stringhe a partire da sequenze di caratteri stampabili (0x20-0x7E più tab, newline, ritorno a capo). Quando la sequenza raggiunge `MIN_STRING_LENGTH` (4 caratteri) e viene incontrato un byte non stampabile, la stringa viene finalizzata.

### Estrazione UTF-16LE

I binari Windows memorizzano molte stringhe in UTF-16LE. L'estrattore cerca coppie di byte in cui il secondo byte è 0x00 (il caso comune per i caratteri ASCII codificati in UTF-16LE). In questo modo vengono intercettate la maggior parte delle stringhe in lingua inglese nei binari PE che l'estrazione ASCII non rileverebbe.

### Rilevamento delle categorie

Ogni stringa estratta attraversa una pipeline di categorizzazione. La logica di rilevamento è basata su pattern senza ricorrere a regex per ragioni di prestazioni:

```rust
const URL_PREFIXES: &[&str] = &["http://", "https://", "ftp://"];

const SHELL_INDICATORS: &[&str] = &[
    "cmd.exe", "cmd /c", "cmd /k",
    "powershell", "pwsh",
    "/bin/sh", "/bin/bash", "/bin/zsh",
    "bash -c", "sh -c",
    "| bash", "|bash",
    "| /bin/sh", "|/bin/sh",
];

const PACKER_SIGNATURES: &[&str] = &[
    "UPX!", "MPRESS", ".themida", ".vmp", ".enigma",
    "PEC2", "ASPack", "MEW ",
];

const ANTI_ANALYSIS_INDICATORS: &[&str] = &[
    "VMware", "VirtualBox", "VBox", "QEMU",
    "sandbox", "Sandboxie", "wireshark", "procmon",
    "x64dbg", "x32dbg", "ollydbg", "IDA Pro", "Ghidra",
];
```

Il rilevamento degli URL controlla i prefissi. Il rilevamento degli indirizzi IP valida quattro ottetti separati da punti nell'intervallo 0-255. Il rilevamento dei comandi di shell effettua il matching con interpreti di shell noti e pattern di pipe. Il rilevamento dell'anti-analisi cerca i nomi di debugger, VM e strumenti di sandbox.

### Rilevamento di Base64

Il pass segnala i potenziali dati codificati in Base64 usando una validazione strutturale piuttosto che tentando di decodificare:

- Minimo 20 caratteri
- Solo caratteri Base64 validi (A-Z, a-z, 0-9, +, /)
- Padding corretto (= o == alla fine, se presente)
- Segnala specificamente i prefissi `TVqQ`, `TVpQ`, `TVoA`, `TVpB` — queste sono codifiche Base64 dell'header PE MZ, che indicano un eseguibile Windows incorporato

### Statistiche

Il pass calcola statistiche aggregate insieme all'elenco grezzo delle stringhe:

```rust
pub struct StringStatistics {
    pub total: usize,
    pub by_encoding: HashMap<StringEncoding, usize>,
    pub by_category: HashMap<StringCategory, usize>,
    pub suspicious_count: usize,
}
```

Questo fornisce al threat scorer e al frontend dati di riepilogo rapidi senza dover rielaborare l'intero elenco delle stringhe.

## Costruzione del pass di analisi dell'entropia

Il pass dell'entropia combina il calcolo dell'entropia di Shannon con l'analisi strutturale per rilevare il packing:

```rust
impl AnalysisPass for EntropyPass {
    fn name(&self) -> &'static str {
        "entropy"
    }

    fn dependencies(&self) -> &[&'static str] {
        &["format"]
    }

    fn run(
        &self,
        ctx: &mut AnalysisContext,
    ) -> Result<(), EngineError> {
        let format_result = ctx
            .format_result
            .as_ref()
            .ok_or_else(|| EngineError::MissingDependency {
                pass: "entropy".into(),
                dependency: "format".into(),
            })?;

        let data = ctx.data();
        let result = analyze_entropy(
            data,
            &format_result.sections,
            format_result.entry_point,
        );
        ctx.entropy_result = Some(result);
        Ok(())
    }
}
```

Il pattern di controllo delle dipendenze compare in ogni pass che richiede risultati precedenti. Se `format_result` è `None` (perché `FormatPass` è fallito), il pass dell'entropia restituisce un errore `MissingDependency` invece di andare in panic.

### Implementazione dell'entropia di Shannon

Il calcolo principale dell'entropia è compatto:

```rust
fn shannon_entropy(data: &[u8]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    let mut freq = [0u64; BYTE_RANGE];
    for &byte in data {
        freq[byte as usize] += 1;
    }
    let len = data.len() as f64;
    freq.iter()
        .filter(|&&c| c > 0)
        .map(|&c| {
            let p = c as f64 / len;
            -p * p.log2()
        })
        .sum()
}
```

Un array a dimensione fissa di 256 contatori (uno per ogni possibile valore di byte). Un singolo passaggio attraverso i dati per contare le frequenze. Poi la formula di Shannon: per ogni valore di byte che compare almeno una volta, calcola `-(p * log2(p))` e somma. Il risultato varia da 0,0 (tutti byte identici) a 8,0 (perfettamente casuale).

Il `filter(|&&c| c > 0)` è importante — `log2(0)` è meno infinito. I valori di byte che non compaiono mai vengono saltati del tutto.

### Analisi per sezione

Il pass calcola l'entropia per ogni sezione individualmente, non solo per il binario nel suo insieme:

```rust
for section in sections {
    let section_data = read_section_data(
        data,
        section.raw_offset,
        section.raw_size,
    );
    let entropy = if section_data.is_empty() {
        0.0
    } else {
        shannon_entropy(section_data)
    };
    let classification = classify_entropy(entropy);
    let vr_ratio = if section.raw_size > 0 {
        section.virtual_size as f64 / section.raw_size as f64
    } else {
        0.0
    };

    let mut flags = Vec::new();

    if entropy > HIGH_ENTROPY_THRESHOLD {
        flags.push(EntropyFlag::HighEntropy);
    }
    if vr_ratio > VIRTUAL_RAW_RATIO_THRESHOLD {
        flags.push(EntropyFlag::HighVirtualToRawRatio);
    }
    if section.raw_size == 0 && section.virtual_size > 0 {
        flags.push(EntropyFlag::EmptyRawData);
    }
    if section.permissions.is_rwx() {
        flags.push(EntropyFlag::Rwx);
    }
    // ...
}
```

L'analisi per sezione è importante perché un binario può avere una normale sezione `.text` (entropia ~5,5) ma una sezione `.rsrc` packed (entropia ~7,8). L'entropia complessiva potrebbe avere una media di circa 6,5, che sembra quella di dati compressi. L'analisi per sezione individua la sezione packed che l'analisi complessiva non coglierebbe.

### Rilevamento di PUSHAD

Un'euristica semplice ma efficace: controllare se il primo byte all'entry point è `0x60` (l'opcode PUSHAD di x86). UPX e molti altri packer salvano tutti i registri con PUSHAD prima di eseguire lo stub di unpacking:

```rust
if first_byte == PUSHAD_OPCODE {
    packing_indicators.push(PackingIndicator {
        indicator_type: "entry_point".into(),
        description: "PUSHAD at entry point".into(),
        evidence: format!(
            "byte 0x{PUSHAD_OPCODE:02x} at EP offset 0x{ep_file_offset:x}"
        ),
        packer_name: None,
    });
}
```

### Decisione sul packing

Il rilevamento del packing combina tutti i segnali: i nomi di sezione noti impostano direttamente `packer_name`. Gli indicatori strutturali (dati raw vuoti con allocazione virtuale, elevato rapporto V/R) accumulano un conteggio. Se viene identificato il nome di un packer OPPURE scatta un numero sufficiente di indicatori strutturali, viene segnalato il packing:

```rust
let packing_detected = packer_name.is_some()
    || structural_count >= STRUCTURAL_INDICATORS_FOR_PACKING;
```

`STRUCTURAL_INDICATORS_FOR_PACKING` è 2 — sono necessari almeno due indicatori strutturali per segnalare il packing senza un nome di packer noto. Questo previene i falsi positivi dovuti a una singola sezione anomala.

## Costruzione del pass di disassembly

Il pass di disassembly converte il codice macchina grezzo in istruzioni strutturate, basic block e control flow graph:

```rust
impl AnalysisPass for DisasmPass {
    fn name(&self) -> &'static str {
        "disasm"
    }

    fn dependencies(&self) -> &[&'static str] {
        &["format"]
    }

    fn run(
        &self,
        ctx: &mut AnalysisContext,
    ) -> Result<(), EngineError> {
        let format_result = ctx.format_result.as_ref()
            .ok_or_else(|| EngineError::MissingDependency {
                pass: "disasm".into(),
                dependency: "format".into(),
            })?;

        let arch = &format_result.architecture;
        let bits = match arch {
            Architecture::X86 => 32u32,
            Architecture::X86_64 => 64,
            _ => {
                ctx.disassembly_result = Some(empty_result(
                    format_result.bits,
                    format_result.entry_point,
                ));
                return Ok(());
            }
        };

        let data = ctx.data();
        let sections = &format_result.sections;
        let entry_point = format_result.entry_point;

        let mut seeds = vec![entry_point];
        seeds.extend_from_slice(&format_result.function_hints);

        let result = disassemble(data, sections, bits, entry_point, &seeds);
        ctx.disassembly_result = Some(result);
        Ok(())
    }
}
```

Le architetture non-x86 ricevono un risultato vuoto anziché un errore. Questo è intenzionale — un binario Mach-O AArch64 deve comunque ricevere il parsing del formato, l'analisi degli import, le stringhe, l'entropia e il threat scoring. Viene saltato solo il disassembly.

### Discesa ricorsiva

Il disassembler usa un approccio a discesa ricorsiva (recursive descent) basato su worklist. Partendo dagli indirizzi seed (entry point + hint della tabella dei simboli), scopre le funzioni seguendo le istruzioni call:

```rust
fn disassemble(
    data: &[u8],
    sections: &[SectionInfo],
    bits: u32,
    entry_point: u64,
    seeds: &[u64],
) -> DisassemblyResult {
    let exec_sections: Vec<&SectionInfo> = sections
        .iter()
        .filter(|s| s.permissions.execute && s.raw_size > 0)
        .collect();

    let mut functions = Vec::new();
    let mut visited_functions = HashSet::new();
    let mut total_instructions = 0;
    let mut function_queue: VecDeque<u64> =
        seeds.iter().copied().collect();

    while let Some(func_addr) = function_queue.pop_front() {
        if functions.len() >= MAX_FUNCTIONS
            || total_instructions >= MAX_INSTRUCTIONS
        {
            break;
        }
        if !visited_functions.insert(func_addr) {
            continue;
        }
        if vaddr_to_offset(sections, func_addr).is_none() {
            continue;
        }

        let (func_info, discovered_calls) =
            disassemble_function(
                data, sections, &exec_sections,
                bits, func_addr,
                func_addr == entry_point,
                MAX_INSTRUCTIONS - total_instructions,
            );

        total_instructions += func_info.instruction_count;
        functions.push(func_info);

        for call_target in discovered_calls {
            if !visited_functions.contains(&call_target) {
                function_queue.push_back(call_target);
            }
        }
    }
    // ...
}
```

L'insieme `visited_functions` previene i loop infiniti (chiamate ricorsive). I limiti `MAX_FUNCTIONS` (1.000) e `MAX_INSTRUCTIONS` (50.000) impediscono a binari patologici di causare il blocco dell'analizzatore. La chiamata `disassemble_function` di ciascuna funzione restituisce sia le informazioni sulla funzione SIA un elenco dei target di call scoperti — tali target vengono inseriti nella worklist per l'elaborazione successiva.

### Costruzione del CFG

Ogni funzione produce un control flow graph con nodi (basic block) e archi (transizioni del flusso di controllo):

```rust
pub struct FunctionCfg {
    pub nodes: Vec<CfgNode>,
    pub edges: Vec<CfgEdge>,
}

pub struct CfgNode {
    pub id: u64,
    pub label: String,
    pub instruction_count: usize,
    pub instructions_preview: String,
}

pub struct CfgEdge {
    pub from: u64,
    pub to: u64,
    pub edge_type: CfgEdgeType,
}
```

I confini dei basic block sono determinati dai target dei branch (l'inizio di un blocco) e dalle istruzioni di branch (la fine di un blocco). Il tipo di arco distingue tra branch condizionali (true/false), salti incondizionati, chiamate di funzione e fallthrough (esecuzione sequenziale verso il blocco successivo).

## Costruzione del pass di threat scoring

Il pass delle minacce è il più complesso — aggrega i finding di ogni altro pass in un punteggio unificato:

```rust
impl AnalysisPass for ThreatPass {
    fn name(&self) -> &'static str {
        "threat"
    }

    fn dependencies(&self) -> &[&'static str] {
        &["format", "imports", "strings", "entropy", "disasm"]
    }

    fn run(
        &self,
        ctx: &mut AnalysisContext,
    ) -> Result<(), EngineError> {
        let yara_scanner = YaraScanner::new()?;
        let yara_matches = yara_scanner.scan(ctx.data())?;

        let format_result = ctx.format_result.as_ref();
        let import_result = ctx.import_result.as_ref();
        let string_result = ctx.string_result.as_ref();
        let entropy_result = ctx.entropy_result.as_ref();

        let result = compute_threat_score(
            format_result,
            import_result,
            string_result,
            entropy_result,
            &yara_matches,
        );
        ctx.threat_result = Some(result);
        Ok(())
    }
}
```

Si noti che ThreatPass dipende da tutti gli altri cinque pass. Inoltre esegue la scansione YARA qui invece che in un pass separato, perché i match di YARA confluiscono direttamente nel threat score e in nessun altro punto della pipeline.

### Architettura di scoring

Ogni categoria di scoring è una funzione indipendente che restituisce una `ScoringCategory`:

```rust
pub fn compute_threat_score(
    format_result: Option<&FormatResult>,
    import_result: Option<&ImportResult>,
    string_result: Option<&StringResult>,
    entropy_result: Option<&EntropyResult>,
    yara_matches: &[YaraMatch],
) -> ThreatResult {
    let cat_import = score_imports(import_result, string_result);
    let cat_entropy = score_entropy(entropy_result);
    let cat_packing = score_packing(entropy_result, format_result, string_result);
    let cat_strings = score_strings(string_result);
    let cat_sections = score_sections(format_result);
    let cat_ep = score_entry_point(format_result);
    let cat_anti = score_anti_analysis(import_result, string_result);
    let cat_yara = score_yara(yara_matches);
    // ...
}
```

Ciascuna funzione `score_*` valuta indipendentemente la propria categoria e applica il tetto massimo previsto per quella categoria. Il risultato include ogni singola regola di scoring che è scattata, con il relativo valore in punti e la stringa delle evidenze. Questo rende lo scoring completamente trasparente — l'analista può vedere esattamente il motivo per cui il punteggio è quello che è.

### Tetti massimi delle categorie

Delle costanti definiscono i punti massimi per categoria:

```rust
const IMPORT_MAX: u32 = 20;
const ENTROPY_MAX: u32 = 15;
const PACKING_MAX: u32 = 15;
const STRING_MAX: u32 = 10;
const SECTION_MAX: u32 = 10;
const ENTRY_POINT_MAX: u32 = 10;
const ANTI_ANALYSIS_MAX: u32 = 10;
const YARA_MAX: u32 = 10;
```

I tetti massimi garantiscono che nessuna singola categoria domini. Un binario che attiva ogni possibile regola sugli import ottiene comunque al massimo 20 punti dagli import, lasciando che gli altri 80 punti siano determinati dalle restanti sette categorie.

### Classificazione del livello di rischio

Il punteggio totale si mappa su un livello di rischio mediante un semplice confronto con delle soglie:

```rust
const BENIGN_MAX: u32 = 15;
const LOW_MAX: u32 = 35;
const MEDIUM_MAX: u32 = 55;
const HIGH_MAX: u32 = 75;
```

Un punteggio di 0-15 è BENIGN. 16-35 è LOW. 36-55 è MEDIUM. 56-75 è HIGH. 76+ è CRITICAL.

## Costruzione dello scanner YARA

Lo scanner YARA compila 14 regole integrate a partire da una costante stringa e fornisce un metodo `scan`:

```rust
rule suspicious_upx_packed {
    meta:
        description = "Detects UPX packed binaries"
        category = "packer"
        severity = "medium"
    strings:
        $upx0 = "UPX0"
        $upx1 = "UPX1"
        $upx_magic = { 55 50 58 21 }
    condition:
        ($upx0 and $upx1) or $upx_magic
}
```

Questa regola scatta se il binario contiene sia le stringhe "UPX0" che "UPX1" (i nomi delle sezioni UPX) OPPURE i magic byte di UPX `0x55 0x50 0x58 0x21` (ASCII "UPX!"). Il pattern esadecimale `{ 55 50 58 21 }` trova corrispondenze con i byte grezzi a qualsiasi offset.

Regole più complesse combinano molteplici indicatori:

```rust
rule suspicious_process_injection {
    meta:
        description = "Detects potential process injection capabilities"
        category = "injection"
        severity = "critical"
    strings:
        $api1 = "VirtualAllocEx"
        $api2 = "WriteProcessMemory"
        $api3 = "CreateRemoteThread"
        $api4 = "NtUnmapViewOfSection"
    condition:
        ($api1 and $api2 and $api3) or ($api4 and $api2)
}
```

La condizione usa la logica booleana: o la classica catena di injection (tutte e tre le API) OPPURE la coppia di process hollowing. La forza di YARA sta nell'esprimere queste condizioni multi-indicatore in modo conciso.

## La route di upload

Il layer HTTP fa da ponte tra il frontend e il motore. L'handler di upload riceve un file multipart, controlla la cache SHA-256 e avvia l'analisi:

```rust
pub async fn handle(
    State(state): State<AppState>,
    mut multipart: Multipart,
) -> Result<Json<UploadResponse>, ApiError> {
    let (file_name, data) = extract_file(&mut multipart).await?;

    let sha256 = axumortem_engine::sha256_hex(&data);

    if let Some(slug) =
        queries::find_slug_by_sha256(&state.db, &sha256).await?
    {
        return Ok(Json(UploadResponse { slug, cached: true }));
    }

    let engine = Arc::clone(&state.engine);
    let name_clone = file_name.clone();

    let (ctx, report) =
        tokio::task::spawn_blocking(move || {
            engine.analyze(&data, &name_clone)
        })
        .await?;

    // ... persist to database ...

    Ok(Json(UploadResponse { slug, cached: false }))
}
```

Il controllo SHA-256 avviene prima di qualsiasi operazione di analisi. Se il binario è già stato visto in precedenza, la risposta è immediata. La chiamata `spawn_blocking` è fondamentale — l'analisi è CPU-bound e bloccherebbe il runtime di Tokio se eseguita su un task asincrono.

### Persistenza dei risultati basata su macro

L'handler di upload usa una macro per serializzare il risultato di ciascun pass in JSONB:

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

Ogni risultato di un pass presente nel contesto viene serializzato in un valore JSON e associato alla durata della sua esecuzione. La funzione `api_name` traduce i nomi interni dei pass in nomi adatti alle API (ad esempio, "disasm" diventa "disassembly").

## Strategia di testing

### Unit test

Ciascun pass dispone della propria suite di test. I test di `FormatPass` usano binari di fixture (eseguibili ELF reali compilati per i test):

```rust
fn load_fixture(name: &str) -> Vec<u8> {
    let path = format!(
        "{}/tests/fixtures/{name}",
        env!("CARGO_MANIFEST_DIR"),
    );
    std::fs::read(&path).unwrap_or_else(|e| {
        panic!("fixture {path}: {e}")
    })
}

#[test]
fn parse_elf_basic_metadata() {
    let data = load_fixture("hello_elf");
    let result = formats::parse_format(&data).unwrap();

    assert_eq!(result.format, BinaryFormat::Elf);
    assert_eq!(result.architecture, Architecture::X86_64);
    assert_eq!(result.bits, 64);
    assert_eq!(result.endianness, Endianness::Little);
    assert!(result.entry_point > 0);
    assert!(result.is_pie);
    assert!(!result.is_stripped);
}
```

### Test del PassManager

Lo stesso sistema di pass dispone di test per l'ordinamento delle dipendenze, la gestione dei fallimenti e il rilevamento dei cicli usando mock pass:

```rust
#[test]
fn topological_sort_respects_dependencies() {
    let log = Arc::new(Mutex::new(Vec::new()));

    let passes: Vec<Box<dyn AnalysisPass>> = vec![
        Box::new(MockPass { name: "c", deps: vec!["b"], /* ... */ }),
        Box::new(MockPass { name: "a", deps: vec![], /* ... */ }),
        Box::new(MockPass { name: "b", deps: vec!["a"], /* ... */ }),
    ];

    let manager = PassManager::new(passes);
    let mut ctx = make_ctx();
    let report = manager.run_all(&mut ctx);

    let execution_order = log.lock().unwrap().clone();
    assert_eq!(execution_order, vec!["a", "b", "c"]);
    assert!(report.all_succeeded());
}

#[test]
#[should_panic(expected = "cycle detected")]
fn detects_cycle() {
    let passes: Vec<Box<dyn AnalysisPass>> = vec![
        Box::new(MockPass { name: "a", deps: vec!["b"], /* ... */ }),
        Box::new(MockPass { name: "b", deps: vec!["a"], /* ... */ }),
    ];
    let _manager = PassManager::new(passes);
}
```

Il test di rilevamento dei cicli verifica che l'ordinamento topologico vada in panic al momento della costruzione (non al momento dell'analisi) quando i pass dichiarano dipendenze circolari.

### Esecuzione dei test

```bash
cd backend
cargo test
```

Per uno specifico pass:

```bash
cargo test -p axumortem-engine --lib passes::entropy
```

## Errori comuni di implementazione

**Dimenticare `spawn_blocking` per il lavoro della CPU.** Se il motore di analisi viene eseguito su un task asincrono invece che su un thread bloccante, affama il runtime di Tokio. Altre richieste (health check, recuperi in cache) rimarrebbero bloccate fino al termine dell'analisi. Usa sempre `spawn_blocking` per le operazioni che richiedono più di pochi millisecondi di tempo CPU.

**Non porre tetti ai limiti del disassembly.** Senza `MAX_FUNCTIONS` e `MAX_INSTRUCTIONS`, un binario con molte funzioni piccole (o, peggio, codice offuscato che sembra contenere funzioni infinite) farebbe girare l'analizzatore all'infinito. I tetti rendono limitato il tempo di analisi nel caso peggiore.

**Trattare i risultati `None` come errori.** Se `ImportPass` fallisce, `ThreatPass` dovrebbe comunque calcolare un threat score parziale usando qualunque dato sia disponibile. Il pattern `Option<T>` nel contesto rende tutto questo naturale — le funzioni di scoring accettano `Option<&ImportResult>` e saltano lo scoring basato sugli import se questo è `None`.

**Fidarsi dei confini delle sezioni.** Un binario malevolo può dichiarare dimensioni di sezione che si estendono oltre la fine del file. La funzione `read_section_data` convalida che `raw_offset + raw_size` non superi la lunghezza del file prima di estrarre lo slice.

## Consigli per il debug

**L'analisi non restituisce alcun risultato:** Controlla gli esiti in `PassReport`. Se un pass è fallito, il messaggio di errore indicherà cosa è andato storto. Cause comuni: il binario è troppo piccolo (meno di 4 byte non possono essere analizzati), oppure il formato non è riconosciuto (non è ELF, PE o Mach-O).

**Il threat score sembra errato:** Ispeziona l'array `categories` nel `ThreatResult`. Ogni categoria mostra il proprio punteggio individuale, il massimo e le specifiche regole scattate con le relative stringhe di evidenza. Confronta le evidenze con il binario per verificarne la correttezza.

**Il disassembly è vuoto per un binario x86:** Verifica che l'indirizzo dell'entry point mappi su una sezione valida con permessi di esecuzione. Se l'entry point si trova in una sezione non eseguibile (un'anomalia di per sé), il disassembler non può trovare il codice.

**Le regole YARA non trovano corrispondenze:** Verifica che le stringhe esistano effettivamente nel binario. Usa `strings` (strumento Unix) o un hex editor per confermarlo. Il matching delle stringhe in YARA è case-sensitive per impostazione predefinita — `"VirtualAllocEx"` non corrisponderà a `"virtualallocex"`.

## Prossimi passi

Prosegui con [04 - Sfide](04-CHALLENGES.it.md) per idee di estensione che spaziano dall'aggiunta di nuovi pass di analisi alla creazione di un cluster di analisi distribuito.
