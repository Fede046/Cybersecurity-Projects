# Binary Analysis Tool: concetti

Questo modulo tratta i fondamenti di sicurezza alla base dell'analisi statica dei binari. Ogni concetto qui si mappa direttamente su un componente della pipeline di analisi di AXUMORTEM.

## Formati degli eseguibili binari

### Cosa sono

Ogni programma compilato è memorizzato come eseguibile binario in un formato che dice al sistema operativo come caricarlo in memoria e avviarne l'esecuzione. I tre formati dominanti sono:

- **ELF** (Executable and Linkable Format) — Linux, BSD, la maggior parte dei sistemi Unix
- **PE** (Portable Executable) — Windows
- **Mach-O** (Mach Object) — macOS, iOS

Non sono semplici "contenitori di codice macchina". Sono metadati strutturati che descrivono il layout della memoria, le dipendenze dalle librerie dinamiche, le tabelle dei simboli, le informazioni di rilocazione e le proprietà di sicurezza. Ogni campo di questi header è qualcosa che un analista può usare per determinare se un binario è legittimo.

### Perché sono importanti

Comprendere i formati binari è fondamentale perché ogni altra tecnica di analisi dipende dal parsing corretto del formato come primo passo. Se non riesci a trovare la sezione `.text`, non puoi fare il disassembly. Se non riesci a fare il parsing della tabella degli import, non puoi segnalare l'uso di API sospette. Se non riesci a identificare le sezioni, non puoi calcolare l'entropia per sezione.

**Il formato ELF** usa una struttura a livelli:

```
+------------------+
| ELF Header       |  Magic bytes (7f 45 4c 46), architecture, entry point
+------------------+
| Program Headers  |  Segments — what the kernel loads into memory
+------------------+
| .text            |  Executable code
| .rodata          |  Read-only data (strings, constants)
| .data            |  Initialized writable data
| .bss             |  Uninitialized data (zero-filled at load)
| .symtab          |  Symbol table (function/variable names)
| .strtab          |  String table (names referenced by symbols)
| .dynamic         |  Dynamic linking information
| .got / .plt      |  Global offset table / procedure linkage table
+------------------+
| Section Headers  |  Metadata about each section
+------------------+
```

**Il formato PE** è specifico di Windows e più complesso:

```
+------------------+
| DOS Header       |  Legacy MS-DOS stub (the "MZ" signature)
+------------------+
| PE Signature     |  "PE\0\0" (50 45 00 00)
+------------------+
| COFF Header      |  Machine type, number of sections, timestamp
+------------------+
| Optional Header  |  Entry point, image base, subsystem, data directories
+------------------+
| Section Table    |  .text, .rdata, .data, .rsrc, .reloc
+------------------+
| Sections         |  Actual code and data
+------------------+
```

I binari PE portano metadati extra che ELF non ha: un rich header (impronta del compilatore), la tabella delle TLS callback (codice che viene eseguito prima di `main`) e una data directory che punta a import, export, informazioni di debug e altro. Gli autori di malware abusano delle TLS callback per eseguire codice anti-debugging prima ancora che scatti il breakpoint del debugger sull'entry point.

### Come li analizza AXUMORTEM

AXUMORTEM usa il crate `goblin` per fare il parsing di tutti e tre i formati attraverso un singolo entry point. Il `FormatPass` fa il dispatch verso parser specifici per formato che estraggono un `FormatResult` unificato:

```rust
pub struct FormatResult {
    pub format: BinaryFormat,
    pub architecture: Architecture,
    pub bits: u8,
    pub endianness: Endianness,
    pub entry_point: u64,
    pub is_stripped: bool,
    pub is_pie: bool,
    pub has_debug_info: bool,
    pub sections: Vec<SectionInfo>,
    pub segments: Vec<SegmentInfo>,
    pub anomalies: Vec<FormatAnomaly>,
    pub pe_info: Option<PeInfo>,
    pub elf_info: Option<ElfInfo>,
    pub macho_info: Option<MachOInfo>,
    pub function_hints: Vec<u64>,
}
```

I dettagli specifici del formato vanno nelle proprie struct opzionali (`PeInfo`, `ElfInfo`, `MachOInfo`), mentre le proprietà comuni come sezioni e segmenti usano gli stessi tipi per tutti i formati.

### Rilevamento delle anomalie

Il parser del formato non si limita a estrarre dati, segnala anche i problemi strutturali. Ogni anomalia è una specifica variante di enum:

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

Un entry point nell'ultima sezione è sospetto perché i packer (UPX, Themida, ASPack) spacchettano il codice a runtime e saltano all'entry point originale, che memorizzano nella sezione finale. I compilatori legittimi mettono l'entry point in `.text`, che non è quasi mai l'ultima sezione.

### Esempio reale: Stuxnet

Stuxnet (2010) era un binario PE che prendeva di mira le centrifughe nucleari iraniane. L'analisi del formato ha rivelato più anomalie: il suo header PE aveva un timestamp insolito (una data di compilazione del 2001, chiaramente falsificata), conteneva un certificato Authenticode firmato rubato a Realtek Semiconductor e le sue sezioni includevano payload cifrati con valori di entropia superiori a 7,9. La tabella degli import faceva riferimento a `DeviceIoControl` per comunicare con l'hardware PLC Siemens S7-300 — un'API di cui nessun software da ufficio legittimo avrebbe mai bisogno.

### Insidie comuni

**Fidarsi del nome della sezione.** I nomi delle sezioni sono indicativi, non vincolanti. I malware chiamano abitualmente le sezioni di codice packed `.text` o `.code` per sembrare normali. AXUMORTEM controlla entropia e permessi invece di basarsi solo sui nomi.

**Ignorare i binari stripped.** Quando un binario è stripped (senza tabella dei simboli), perdi i nomi delle funzioni. Questo non è sempre malevolo — le build di release sono comunemente stripped. Ma in combinazione con altri indicatori (alta entropia, import sospetti), lo stripping si aggiunge al threat score.

## Analisi dell'entropia

### Cos'è

L'entropia di Shannon misura la casualità dei dati. Per l'analisi dei binari, l'entropia ti dice che tipo di dati c'è in ogni sezione: il testo semplice si aggira intorno a 3-4 bit/byte, il codice nativo compilato si colloca a 5-6, i dati compressi raggiungono 6,5-7 e i dati cifrati o packed si avvicinano al massimo teorico di 8 bit/byte.

La formula calcola quanto ogni byte sia "sorprendente" in media:

```
H = -Σ p(x) * log₂(p(x))    for each possible byte value (0-255)
```

Se ogni valore di byte compare con la stessa frequenza (perfettamente casuale), l'entropia è 8,0. Se compare un solo valore di byte (tutti zeri), l'entropia è 0,0.

### Perché è importante

Packing e cifratura sono i due modi più comuni con cui gli autori di malware nascondono i propri payload. Un binario packed comprime il codice reale e include un piccolo stub di unpacking che decomprime tutto a runtime. Un binario cifrato fa la stessa cosa ma con la cifratura. Entrambe le tecniche producono sezioni con entropia superiore a 7,0 — e questo è un indizio inequivocabile durante il triage.

AXUMORTEM classifica l'entropia in cinque livelli:

```
Plaintext    : < 3.5    (config files, string tables)
Native Code  : < 6.0    (compiled x86/x64/ARM instructions)
Compressed   : < 7.0    (zlib, gzip, resource data)
Packed       : < 7.2    (UPX, Themida, ASPack)
Encrypted    : >= 7.2   (AES-encrypted payloads, XOR-obfuscated code)
```

### Rilevamento dei packer

L'entropia da sola non basta. AXUMORTEM combina più segnali per rilevare il packing:

**Nomi di sezione noti.** I packer lasciano impronte nei nomi delle sezioni:

```rust
const PACKER_SECTION_NAMES: &[(&str, &str)] = &[
    ("UPX0", "UPX"),
    ("UPX1", "UPX"),
    (".themida", "Themida"),
    (".vmp0", "VMProtect"),
    (".vmp1", "VMProtect"),
    (".aspack", "ASPack"),
    (".MPRESS1", "MPRESS"),
    (".MPRESS2", "MPRESS"),
    (".enigma1", "Enigma"),
    (".enigma2", "Enigma"),
];
```

**Rapporto tra dimensione virtuale e raw.** I packer allocano grandi regioni di memoria virtuale ma memorizzano dati minimi su disco (il payload compresso). Un rapporto superiore a 10:1 è sospetto.

**PUSHAD all'entry point.** UPX e molti altri packer iniziano con `PUSHAD` (opcode `0x60`) per salvare tutti i registri prima della routine di unpacking. AXUMORTEM controlla il primo byte all'offset dell'entry point.

**Sezioni RWX.** I binari legittimi non hanno quasi mai sezioni leggibili, scrivibili E eseguibili. I packer hanno bisogno di RWX perché scrivono il codice decompresso nella stessa sezione da cui lo eseguono.

### Esempio reale: WannaCry

Il ransomware WannaCry (maggio 2017) ha infettato oltre 200.000 sistemi in 150 paesi. Il dropper iniziale era un binario PE con un'enorme sezione delle risorse (`.rsrc`) che aveva un'entropia di 7,89 — chiaramente cifrata. All'interno di quella sezione c'era il vero payload del ransomware, cifrato con AES con una chiave derivata dall'hash della tabella degli import del binario stesso. L'analisi dell'entropia ha segnalato il payload istantaneamente durante il triage, anche se il resto del binario sembrava un normale servizio Windows.

### Esempio reale: NotPetya

NotPetya (giugno 2017) ha causato danni stimati in 10 miliardi di dollari a livello globale. Il wiper travestito da ransomware è stato distribuito attraverso un aggiornamento trojanizzato di M.E.Doc, un software fiscale ucraino. Il binario modificato aveva due sezioni ad alta entropia che non corrispondevano all'originale: una conteneva l'exploit EternalBlue, l'altra conteneva il credential dumper Mimikatz. Il delta di entropia tra il binario legittimo di M.E.Doc e la versione trojanizzata è stato il primo indicatore che ha attirato l'attenzione dei ricercatori.

## Analisi degli import e delle API

### Cos'è

Quando un binario chiama funzioni del sistema operativo (aprire file, allocare memoria, creare connessioni di rete), elenca quelle funzioni nella propria tabella degli import. La tabella degli import è una mappa dai nomi delle librerie ai nomi delle funzioni che il dynamic linker risolve al momento del caricamento.

Certe combinazioni di API sono essenzialmente una firma di specifiche tecniche di attacco. `VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread` è process injection. `NtUnmapViewOfSection` + `SetThreadContext` è process hollowing. Queste combinazioni compaiono in quasi ogni malware Windows che usa quelle tecniche.

### Definizioni delle API sospette

AXUMORTEM mantiene un database di API sospette, ciascuna etichettata con una categoria di tecnica e un ID MITRE ATT&CK:

```rust
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
        name: "NtUnmapViewOfSection",
        tag: "hollowing",
        mitre_id: "T1055.012",
    },
    SuspiciousApiDef {
        name: "IsDebuggerPresent",
        tag: "anti-debug",
        mitre_id: "T1622",
    },
    // ... 22 suspicious APIs total
];
```

Il sistema rileva anche 14 *combinazioni* di API sospette — gruppi di import che insieme indicano una specifica tecnica. Un binario che importa solo `VirtualAllocEx` è sospetto. Un binario che importa insieme `VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread` sta quasi certamente facendo process injection.

### Analisi specifica per Linux

L'analisi degli import non è solo per Windows. AXUMORTEM segnala indicatori specifici di Linux:

- `ptrace` con `PTRACE_ATTACH` — process injection su Linux (T1055.008)
- `mmap` + `mprotect` con `PROT_EXEC` — generazione di codice o injection a runtime
- `dlopen` + `dlsym` — caricamento dinamico di librerie, comune nei rootkit
- `connect` + `execve` — pattern di reverse shell
- `bind` + `listen` + `accept` — listener di rete (potenziale backdoor)

### Esempio reale: la botnet Mirai

La botnet Mirai (2016) era un binario ELF che prendeva di mira i dispositivi IoT. La sua tabella degli import era insolita per il binario di un dispositivo embedded: `socket`, `connect`, `send`, `recv` (attesi per uno scanner di rete), ma anche `execve`, `fork` e `kill`. La combinazione di API di rete con API di gestione dei processi lo ha segnalato come probabile agent di una botnet. Il binario stripped di Mirai aveva solo 14 import unici — molti meno di un'applicazione legittima, il che di per sé era sospetto perché suggeriva un malware ottimizzato a mano.

### Insidie comuni

**Guardare gli import in isolamento.** `VirtualAlloc` è una delle API di Windows più comunemente usate — ogni grande applicazione la chiama. Solo la combinazione con `WriteProcessMemory` e `CreateRemoteThread` è sospetta. Il rilevamento delle combinazioni di AXUMORTEM evita questa trappola dei falsi positivi.

**Dimenticarsi della risoluzione dinamica.** I malware evitano frequentemente gli import statici usando `GetProcAddress` o `dlsym` per risolvere le API a runtime. La tabella degli import sembra pulita, ma l'analisi delle stringhe rivela i nomi delle API incorporati nel binario. Ecco perché il pass delle stringhe di AXUMORTEM lavora insieme all'analisi degli import.

## Analisi delle stringhe

### Cos'è

Le stringhe sono l'intelligence più rapida che puoi estrarre da un binario. I percorsi dei file rivelano i target. Gli URL rivelano i server C2. Le chiavi di registro rivelano i meccanismi di persistenza. I comandi di shell rivelano il comportamento di post-exploitation. Persino gli artefatti di debug possono rivelare l'ambiente di build dello sviluppatore.

AXUMORTEM estrae le stringhe in più codifiche (ASCII, UTF-16LE) e le classifica in 14 categorie:

```rust
pub enum StringCategory {
    Url,              // http://, https://, ftp://
    IpAddress,        // IPv4 and IPv6 addresses
    FilePath,         // /etc/passwd, C:\Windows\System32
    RegistryKey,      // HKEY_LOCAL_MACHINE\...
    ShellCommand,     // cmd.exe /c, powershell, /bin/bash
    CryptoWallet,     // Bitcoin, Ethereum addresses
    Email,            // user@domain.com
    SuspiciousApi,    // API names that didn't appear in imports
    PackerSignature,  // UPX!, .themida, MPRESS
    DebugArtifact,    // /rustc/, .pdb, DWARF
    AntiAnalysis,     // VMware, VirtualBox, Sandboxie, x64dbg
    PersistencePath,  // CurrentVersion\Run, crontab, systemd
    EncodedData,      // Base64-encoded blobs (min 20 chars)
    Generic,          // Everything else
}
```

Sette di queste categorie sono segnalate come sospette e alimentano direttamente il threat scoring: `SuspiciousApi`, `PackerSignature`, `AntiAnalysis`, `PersistencePath`, `EncodedData`, `ShellCommand` e `CryptoWallet`.

### Estrazione UTF-16LE

I binari Windows memorizzano frequentemente le stringhe in codifica UTF-16LE (due byte per carattere, little-endian). Un URL come `http://evil.com` memorizzato in UTF-16LE appare come `h\x00t\x00t\x00p\x00:\x00/\x00/\x00e\x00v\x00i\x00l\x00.\x00c\x00o\x00m\x00` in un hex dump. Se scansioni solo le stringhe ASCII, perdi metà dell'intelligence in un binario PE. AXUMORTEM scansiona entrambe le codifiche.

### Rilevamento del Base64

I malware codificano payload, comandi e URL in Base64 per eludere il semplice string matching. AXUMORTEM segnala le stringhe codificate in Base64 che sono lunghe almeno 20 caratteri e hanno un padding corretto. Cerca specificamente i prefissi `TVqQ`, `TVpQ`, `TVoA` e `TVpB` — queste sono le codifiche Base64 della firma "MZ" dell'header PE, che indicano un eseguibile incorporato nascosto all'interno di un blob Base64.

### Esempio reale: APT29 (Cozy Bear)

La backdoor SUNBURST di APT29 incorporata nell'aggiornamento di SolarWinds Orion (dicembre 2020) era sofisticata, ma l'analisi delle stringhe l'ha comunque intercettata. Il malware conteneva seed hardcoded dell'algoritmo di generazione dei domini, blocchi di configurazione codificati in Base64 e stringhe che facevano riferimento a strumenti di anti-analisi (`apimonitor`, `dnspy`, `ilspy`, `fiddler`, `wireshark`). Le stringhe di anti-analisi venivano usate per rilevare gli ambienti degli analisti e disattivare la backdoor — ma servivano anche come firme per la detection.

## Regole YARA

### Cosa sono

YARA è un motore di pattern matching progettato per l'identificazione dei malware. Una regola YARA definisce un insieme di pattern di stringhe e una condizione booleana. Se la condizione corrisponde, la regola scatta. Le regole YARA sono lo standard di settore per la condivisione delle firme di malware — feed di threat intelligence, motori antivirus e prodotti EDR usano tutti YARA.

Una regola ha tre parti: metadati (descrizione, severità), stringhe (pattern da cercare) e una condizione (logica booleana che combina quei pattern):

```yara
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

Questa regola scatta se il binario contiene il classico trio di process injection (`VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread`) OPPURE la coppia di process hollowing (`NtUnmapViewOfSection` + `WriteProcessMemory`).

### Le regole integrate di AXUMORTEM

AXUMORTEM include 14 regole YARA che coprono le principali categorie di malware:

| Regola | Categoria | Severità | Cosa rileva |
|------|----------|----------|----------------|
| `suspicious_upx_packed` | packer | medium | Nomi di sezione UPX + magic bytes |
| `suspicious_anti_debug` | evasion | high | `IsDebuggerPresent`, trap `INT 2Dh` |
| `suspicious_process_injection` | injection | critical | Classiche catene di API di injection |
| `suspicious_keylogger` | spyware | high | `GetAsyncKeyState`, `SetWindowsHookEx` |
| `suspicious_crypto_mining` | miner | medium | URL di pool Stratum, algoritmi di mining |
| `suspicious_persistence` | persistence | high | Chiavi Run, creazione di servizi, schtasks |
| `suspicious_network_backdoor` | backdoor | high | Bind/listen/accept + pattern cmd/shell |
| `suspicious_ransomware` | ransomware | critical | API di cifratura + stringhe di note di riscatto |
| `suspicious_shellcode` | shellcode | critical | NOP sled, egg hunter, stub comuni |
| `suspicious_obfuscation` | obfuscation | medium | Loop XOR, pattern di codice auto-modificante |
| `suspicious_linux_anti_debug` | evasion | high | Controlli di TracerPid in `/proc/self/status` |
| `suspicious_linux_persistence` | persistence | high | Modifica di crontab, systemd, `.bashrc` |
| `suspicious_linux_c2` | c2 | critical | Pattern di reverse shell, comandi codificati |
| `suspicious_mpress` | packer | medium | Firme di sezione del packer MPRESS |

### In cosa YARA-X differisce dal YARA classico

AXUMORTEM usa `yara-x` (la riscrittura in Rust di YARA) invece dell'implementazione originale in C. `yara-x` è pienamente compatibile con le regole YARA esistenti ma aggiunge validazione a compile-time, messaggi di errore migliori e integrazione nativa con Rust senza overhead di FFI. Lo scanner compila le regole una volta all'inizializzazione del motore e riutilizza il ruleset compilato per ogni binario.

### Esempio reale: Flame (sKyWIper)

Flame (scoperto nel maggio 2012) è stato uno dei malware più complessi mai trovati — un toolkit modulare di cyberspionaggio da 20MB che prendeva di mira paesi del Medio Oriente. È stato identificato in parte tramite regole YARA che corrispondevano al suo insolito motore di scripting Lua incorporato in una DLL di Windows, più firme YARA per il suo database SQLite personalizzato usato per memorizzare i dati rubati. Flame ha dimostrato perché avere un ampio insieme di regole di detection è importante: nessuna singola regola lo ha intercettato, ma la combinazione di più corrispondenze a bassa confidenza ha prodotto una detection ad alta confidenza.

## Disassembly e control flow graph

### Cos'è

Il disassembly converte i byte di codice macchina grezzo di nuovo in istruzioni assembly leggibili da un umano. Un control flow graph (CFG) mappa come l'esecuzione fluisce tra i basic block — sequenze di istruzioni con un singolo punto di ingresso e un singolo punto di uscita.

AXUMORTEM usa il crate `iced-x86` per fare il disassembly dei binari x86 e x86_64. Partendo dall'entry point e da eventuali hint della tabella dei simboli, usa il disassembly a discesa ricorsiva (recursive descent): segui ogni istruzione e, quando incontri un branch, aggiungi entrambi i target alla worklist.

```
Basic Block: 0x401000 - 0x40100F
┌────────────────────────────────────────┐
│ push rbp                               │
│ mov rbp, rsp                           │
│ sub rsp, 0x20                          │
│ cmp dword [rbp-4], 0                   │
│ jne 0x401020                           │
└──────────┬────────────────┬────────────┘
           │ (false)        │ (true)
           v                v
┌──────────────────┐  ┌──────────────────┐
│ Block: 0x401010  │  │ Block: 0x401020  │
│ call 0x402000    │  │ xor eax, eax     │
│ jmp 0x401030     │  │ jmp 0x401030     │
└──────────┬───────┘  └──────────┬───────┘
           │                     │
           v                     v
         ┌───────────────────────────┐
         │ Block: 0x401030           │
         │ leave                     │
         │ ret                       │
         └───────────────────────────┘
```

Ogni nodo è un basic block. Ogni arco rappresenta un possibile percorso di esecuzione: i branch condizionali creano due archi (true/false), i salti incondizionati ne creano uno e le call creano archi verso la funzione chiamata.

### Perché i CFG sono importanti

I CFG rivelano la struttura del programma che i listati di disassembly grezzi oscurano. Un analista che guarda una lista piatta di 10.000 istruzioni non riesce a vedere facilmente che c'è un loop che controlla la presenza di un debugger a ogni iterazione, o che una funzione ha 15 branch di gestione degli errori ma un solo percorso di successo. Il CFG rende questi pattern visivamente evidenti.

I CFG sono anche la base per analisi più avanzate: rilevamento del dead code, identificazione dei loop, scoring di similarità delle funzioni e decompilazione partono tutti dalla costruzione del CFG.

### Limiti

AXUMORTEM limita il disassembly a 1.000 funzioni e 50.000 istruzioni per binario. Questo evita che l'analisi si blocchi su binari grandi (una build completa di Chromium ha milioni di istruzioni). La visualizzazione del CFG è limitata a 500 istruzioni per funzione per mantenere il frontend reattivo.

### Esempio reale: Equation Group

L'Equation Group (collegato alla NSA, esposto dagli Shadow Brokers nel 2016) ha creato alcuni dei malware più sofisticati mai scoperti, tra cui EQUATIONDRUG e GRAYFISH. L'analisi di Kaspersky Lab si è basata pesantemente sul confronto dei control flow graph per collegare diversi campioni di malware allo stesso team di sviluppo. Nonostante il pesante offuscamento, la struttura del CFG delle funzioni chiave — in particolare le routine di cifratura personalizzate — rimaneva riconoscibile tra le varianti. Questa è la potenza dell'analisi strutturale: anche quando i pattern a livello di byte cambiano, il grafo della logica è più difficile da camuffare.

## Threat scoring

### Cos'è

Il threat scoring prende tutti i singoli finding di ogni pass di analisi e produce un unico punteggio da 0 a 100 che quantifica la pericolosità del binario. AXUMORTEM assegna punteggi in otto categorie indipendenti, ciascuna con un tetto massimo di punti:

```
┌──────────────────────────────────────────────┐
│ Category                 │ Max Points │       │
├──────────────────────────┼────────────┤       │
│ Import/API Analysis      │    20      │       │
│ Entropy Analysis         │    15      │       │
│ Packing Detection        │    15      │       │
│ String Analysis          │    10      │       │
│ Section Anomalies        │    10      │       │
│ Entry Point Anomalies    │    10      │       │
│ Anti-Analysis Detection  │    10      │       │
│ YARA Matches             │    10      │       │
├──────────────────────────┼────────────┤       │
│ Maximum Total            │   100      │       │
└──────────────────────────────────────────────┘
```

Ogni categoria riceve un punteggio indipendente ed è limitata al proprio massimo. Un binario che innesca ogni regola sugli import ottiene comunque al massimo 20 punti dagli import. Questo impedisce a una singola categoria rumorosa di sovrastare il punteggio finale.

### Livelli di rischio

Il punteggio totale si mappa su cinque livelli di rischio:

| Intervallo di punteggio | Livello di rischio | Binario tipico |
|-------------|-----------|----------------|
| 0-15 | BENIGN | Applicazioni standard, utility di sistema |
| 16-35 | LOW | Strumenti legittimi con alcune feature sospette |
| 36-55 | MEDIUM | Strumenti dual-use (Sysinternals, moduli Metasploit) |
| 56-75 | HIGH | Probabilmente malevolo, forti indicatori comportamentali |
| 76-100 | CRITICAL | Malware attivo con più tecniche confermate |

I confini sono definiti come costanti:

```rust
const BENIGN_MAX: u32 = 15;
const LOW_MAX: u32 = 35;
const MEDIUM_MAX: u32 = 55;
const HIGH_MAX: u32 = 75;
```

### Scoring basato sulle evidenze

Ogni punto nel threat score è accompagnato da un nome di regola e da una motivazione. Non è una black box del tipo "ha totalizzato 72". L'analista può vedere esattamente quali regole sono scattate:

```
Import Analysis: 15/20
  - Process injection chain (VirtualAllocEx + WriteProcessMemory + CreateRemoteThread): +15

Entropy Analysis: 9/15
  - Section .upx0 entropy 7.45 exceeds threshold: +6
  - Overall entropy 6.92 exceeds 6.8: +3

Packing Detection: 8/15
  - Known packer section name "UPX0": +5
  - PUSHAD opcode at entry point: +3

YARA Matches: 5/10
  - suspicious_upx_packed (packer): +3
  - suspicious_process_injection (injection): +5 (capped at 10)

Total: 37/100 — MEDIUM
```

### Mappatura MITRE ATT&CK

Ogni finding si mappa su un ID di tecnica MITRE ATT&CK. Questo collega i risultati dell'analisi statica al più ampio ecosistema della threat intelligence:

- **T1055** — Process Injection (VirtualAllocEx, WriteProcessMemory)
- **T1055.012** — Process Hollowing (NtUnmapViewOfSection)
- **T1055.004** — Asynchronous Procedure Call Injection (QueueUserAPC)
- **T1622** — Debugger Evasion (IsDebuggerPresent, NtQueryInformationProcess)
- **T1547.001** — Boot/Logon Autostart: Registry Run Keys
- **T1543.003** — Create/Modify System Service
- **T1105** — Ingress Tool Transfer (URLDownloadToFile)
- **T1134** — Access Token Manipulation (OpenProcessToken, AdjustTokenPrivileges)
- **T1140** — Deobfuscate/Decode Data (CryptDecrypt)

## Come si relazionano questi concetti

```
                    Binary Upload
                         │
                         v
              ┌─────────────────────┐
              │    Format Parsing   │ ◄── Binary format knowledge
              │  (ELF / PE / Mach-O)│
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          v              v              v
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │  Imports   │ │  Strings   │ │  Entropy   │
   │  Analysis  │ │  Analysis  │ │  Analysis  │
   └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
          │              │              │
          │    ┌─────────┘              │
          │    │                        │
          v    v                        v
   ┌────────────────┐          ┌────────────────┐
   │   Disassembly  │          │    Packing     │
   │   + CFG        │          │   Detection    │
   └────────┬───────┘          └────────┬───────┘
            │                           │
            └───────────┬───────────────┘
                        │
                        v
              ┌─────────────────────┐
              │   Threat Scoring    │ ◄── YARA rules
              │   + MITRE Mapping   │
              └─────────────────────┘
```

Il parsing del formato alimenta tutto ciò che sta a valle. L'analisi degli import e delle stringhe lavorano in parallelo, ciascuna fornendo evidenze che il threat scorer combina. L'analisi dell'entropia alimenta il rilevamento del packing, che è una propria categoria di scoring. Il disassembly dipende dal parsing del formato per i confini delle sezioni e gli entry point. Il threat scorer aggrega tutto.

## Standard e framework di settore

### OWASP

Sebbene OWASP si concentri sulla sicurezza delle applicazioni web, l'analisi dei binari interseca diverse preoccupazioni di OWASP:
- **A08:2021 Software and Data Integrity Failures** — l'analisi dei binari verifica che il software distribuito non sia stato manomesso (attacchi alla supply chain come SolarWinds, CCleaner, 3CX)
- **A06:2021 Vulnerable and Outdated Components** — l'analisi degli import rivela a quali librerie si collega un binario

### MITRE ATT&CK

AXUMORTEM mappa i finding su queste tattiche e tecniche ATT&CK:

| Tattica | Tecniche rilevate |
|--------|-------------------|
| Execution | T1055 (Process Injection), T1055.012 (Hollowing), T1055.004 (APC) |
| Persistence | T1547.001 (Run Keys), T1543.003 (System Service) |
| Defense Evasion | T1622 (Debugger Evasion), T1140 (Deobfuscation), T1027 (Obfuscation) |
| Credential Access | T1134 (Token Manipulation) |
| Command and Control | T1071 (Application Layer Protocol), T1105 (Ingress Tool Transfer) |

### CWE

L'analisi dei binari rileva indicatori relativi a:
- **CWE-693** Protection Mechanism Failure (bypass dell'anti-debug)
- **CWE-506** Embedded Malicious Code (corrispondenze delle regole YARA)
- **CWE-829** Inclusion of Functionality from Untrusted Control Sphere (import sospetti)

## Metti alla prova la tua comprensione

1. Un binario ha un'entropia complessiva di 7,4 e solo 3 import (`GetProcAddress`, `LoadLibraryA`, `VirtualAlloc`). L'entry point è nell'ultima sezione, che si chiama `UPX1`. Cosa puoi concludere su questo binario, e quali categorie di scoring di AXUMORTEM scatterebbero?

2. Analizzi due binari con hash SHA-256 di sezione identici per `.text` ma tabelle degli import diverse. Uno importa `ReadFile` + `WriteFile`, l'altro importa `ReadFile` + `WriteFile` + `CreateRemoteThread` + `VirtualAllocEx`. In che modo differirebbero i threat score di AXUMORTEM, e perché il secondo binario potrebbe essere stato modificato?

3. Un binario PE non ha import sospetti, ha bassa entropia (5,2 complessiva) e supera tutte le regole YARA. Ma l'analisi delle stringhe trova `cmd.exe /c`, `powershell -enc` e tre blob codificati in Base64 che iniziano con `TVqQ`. Cosa ti dice questo sul comportamento del binario, e perché l'analisi degli import non l'ha intercettato?

## Approfondimenti

**Essenziali:**
- [PE Format Specification (Microsoft)](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format) — riferimento autorevole per le strutture dell'header PE
- [ELF Specification (Oracle)](https://docs.oracle.com/cd/E19683-01/816-1386/chapter6-46512/index.html) — la specifica ELF originale della System V ABI
- [YARA Documentation](https://yara.readthedocs.io/) — sintassi delle regole, moduli, best practice
- [MITRE ATT&CK](https://attack.mitre.org/) — la matrice completa delle tecniche citata in tutto il documento

**Approfondimento tecnico:**
- *Practical Malware Analysis* (Sikorski & Honig) — il libro di testo definitivo sull'analisi dei binari
- *Practical Binary Analysis* (Andriesse) — focalizzato su ELF con esercizi in stile CTF
- [The Art of Unpacking (Peter Ferrie)](https://www.virusbulletin.com/uploads/pdf/conference/vb2007/VB2007-Ferrie.pdf) — approfondimento sugli internals dei packer

**Storici:**
- [SolarWinds SUNBURST Analysis (Microsoft)](https://www.microsoft.com/en-us/security/blog/2020/12/18/analyzing-solorigate-the-compromised-dll-file-that-started-a-sophisticated-cyberattack-and-how-microsoft-defender-helps-protect/) — analisi dettagliata del binario della DLL trojanizzata
- [Stuxnet Dossier (Symantec)](https://docs.broadcom.com/doc/security-response-w32-stuxnet-dossier-11-en) — report completo di reverse engineering
- [WannaCry Analysis (Kaspersky)](https://securelist.com/wannacry-ransomware-used-in-widespread-attacks-all-over-the-world/78351/) — smontaggio del ransomware con osservazioni sull'entropia
