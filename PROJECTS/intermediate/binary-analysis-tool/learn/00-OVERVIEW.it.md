# Binary Analysis Tool: panoramica

## Cos'è

AXUMORTEM è un motore di analisi statica dei binari che fa il parsing di eseguibili ELF, PE e Mach-O, li fa passare attraverso una pipeline di analisi a sei passaggi (pass) e produce un threat score con la mappatura delle tecniche MITRE ATT&CK. Carichi un binario e ottieni un'analisi completa: header, import, stringhe, entropia, disassembly e un threat score da 0 a 100.

## Perché è importante

L'analisi dei binari è la prima cosa che succede quando un file sospetto arriva sulla scrivania di un analista. Ogni workflow di triage del malware inizia da qui: in che formato è, cosa importa, è packed, quali stringhe tradiscono l'intento.

Quando la backdoor SUNBURST di SolarWinds è stata scoperta nel dicembre 2020, è stata l'analisi dei binari a rivelare il `SolarWinds.Orion.Core.BusinessLayer.dll` trojanizzato. Gli analisti hanno scoperto che la tabella degli import della DLL includeva API di rete inattese, le sue stringhe contenevano logica hardcoded di generazione dei domini C2 e il suo profilo di entropia mostrava sezioni che non corrispondevano a un assembly .NET legittimo. Quella singola analisi ha dato il via a uno dei più grandi sforzi di incident response della storia.

Nel 2017, l'attacco alla supply chain di CCleaner (il build server di Avast stessa) ha distribuito un binario PE trojanizzato a 2,27 milioni di utenti. La backdoor è stata scoperta perché i ricercatori hanno notato che le sezioni del binario avevano un'entropia anomala e la tabella degli import includeva `VirtualAlloc` + `CreateRemoteThread`, la classica catena di process injection. Uno strumento come AXUMORTEM segnala esattamente quella combinazione.

Più di recente, la compromissione della supply chain di 3CX (marzo 2023) ha incorporato una DLL trojanizzata all'interno del client desktop legittimo. L'analisi statica del binario ha rivelato payload di shellcode cifrati nascosti in sezioni ad alta entropia, stringhe anti-analisi che facevano riferimento al rilevamento dei debugger e catene di import sospette che puntavano al process hollowing. L'attacco ha colpito oltre 600.000 organizzazioni in tutto il mondo.

## Cosa imparerai

**Concetti di sicurezza:**
- Come funzionano i formati binari ELF, PE e Mach-O a livello di byte
- Cosa ti dice l'entropia su packing, cifratura e offuscamento
- Come le regole YARA rilevano famiglie di malware, packer e tecniche di evasione
- Come i sistemi di threat scoring quantificano il rischio a partire da indicatori statici
- Identificazione delle tecniche MITRE ATT&CK a partire dagli import delle API

**Competenze tecniche:**
- Costruire una pipeline di analisi modulare con pass ordinati per dipendenza
- Fare il parsing dei formati binari con il crate `goblin`
- Disassembly x86/x86_64 e costruzione del control flow graph
- Compilazione e scansione delle regole YARA con `yara-x`
- Calcolo e classificazione dell'entropia di Shannon
- Architettura full-stack: backend Rust/Axum con frontend React/TypeScript

**Strumenti e tecniche:**
- Organizzazione di un workspace Rust con più crate
- Framework HTTP Axum con upload di file multipart
- Query verificate a compile-time con SQLx su PostgreSQL
- React 19 con TanStack Query per il data fetching asincrono
- Validazione degli schemi con Zod sul frontend
- Docker Compose per gli ambienti di produzione e di sviluppo

## Prerequisiti

**Conoscenze richieste:**
- Saper leggere Rust con disinvoltura (ownership, trait, enum, pattern matching)
- Comprensione di base di come funzionano gli eseguibili (sezioni, header, linking)
- Familiarità con le API web (REST, JSON, codici di stato HTTP)
- Fondamenti di React/TypeScript (componenti, hook, routing)

**Utile ma non richiesto:**
- Assembly x86 (il modulo di disassembly ti insegnerà le basi)
- Sintassi delle regole YARA (trattata nel modulo sui concetti)
- Esperienza di analisi malware (questo progetto è pensato per costruire quella competenza)

**Strumenti necessari:**
- Docker e Docker Compose
- Toolchain Rust (per lo sviluppo locale)
- Node.js 22+ e pnpm (per lo sviluppo frontend)
- Il command runner `just` (opzionale ma raccomandato)

## Quick Start

```bash
git clone https://github.com/CarterPerez-dev/Cybersecurity-Projects.git
cd Cybersecurity-Projects/PROJECTS/intermediate/binary-analysis-tool

docker compose up -d
```

Visita `http://localhost:22784` e carica un binario. Dovresti vedere:

```
Analysis complete
Format: ELF | Architecture: x86_64 | Size: 15.2 KB
Threat Score: 12/100 (BENIGN)
```

Per lo sviluppo con hot reload:

```bash
docker compose -f dev.compose.yml up -d
```

Il dev server del frontend gira su `http://localhost:15723`, l'API del backend sulla porta `3000`.

## Struttura del progetto

```
binary-analysis-tool/
├── backend/
│   └── crates/
│       ├── axumortem-engine/          # Core analysis library
│       │   └── src/
│       │       ├── lib.rs             # Engine entry — orchestrates all passes
│       │       ├── types.rs           # BinaryFormat, Architecture, RiskLevel enums
│       │       ├── context.rs         # AnalysisContext — carries results between passes
│       │       ├── pass.rs            # AnalysisPass trait + PassManager (topo sort)
│       │       ├── yara.rs            # YARA scanner with 14 built-in rules
│       │       ├── formats/           # ELF, PE, Mach-O parsers
│       │       └── passes/            # The six analysis passes
│       │           ├── format.rs      # 1. Binary format detection
│       │           ├── imports.rs     # 2. Import/export extraction
│       │           ├── strings.rs     # 3. String extraction + categorization
│       │           ├── entropy.rs     # 4. Shannon entropy + packing detection
│       │           ├── disasm.rs      # 5. x86/x86_64 disassembly + CFG
│       │           └── threat.rs      # 6. Threat scoring (8 categories, 100pt max)
│       │
│       └── axumortem/                 # HTTP server
│           └── src/
│               ├── main.rs            # Axum server with graceful shutdown
│               ├── routes/            # upload, analysis retrieval, health check
│               ├── db/                # PostgreSQL models, queries, migrations
│               └── middleware/        # CORS configuration
│
├── frontend/
│   └── src/
│       ├── api/                       # Axios client, Zod schemas, React Query hooks
│       ├── pages/
│       │   ├── landing/               # Drag-drop upload interface
│       │   └── analysis/              # Tabbed results (overview, headers, imports, ...)
│       └── config.ts                  # Risk level colors, route definitions
│
├── infra/docker/                      # Dockerfiles for production builds
├── compose.yml                        # Production: nginx + backend + postgres
├── dev.compose.yml                    # Development: vite + backend + postgres
└── justfile                           # Command runner recipes
```

## Prossimi passi

| Prossimo | Argomento |
|------|-------|
| [01 - Concetti](01-CONCEPTS.it.md) | Internals dei formati binari, teoria dell'entropia, regole YARA, threat modeling |
| [02 - Architettura](02-ARCHITECTURE.it.md) | Design della pipeline di pass, flusso dei dati, interazioni tra componenti |
| [03 - Implementazione](03-IMPLEMENTATION.it.md) | Panoramica del codice con snippet reali di ogni pass |
| [04 - Sfide](04-CHALLENGES.it.md) | Costruisci estensioni: nuovi pass, nuovi formati, dashboard |

## Problemi comuni

**La build Docker fallisce su `yara-x`:**
Il crate `yara-x` richiede `protobuf-compiler` in fase di build. Il Dockerfile di produzione lo installa, ma se fai la build in locale ti serve `apt install protobuf-compiler` (Debian/Ubuntu) o `brew install protobuf` (macOS).

**Errori di SQLx a compile-time:**
SQLx verifica le query rispetto al database a compile-time. Se il database non è in esecuzione, imposta `SQLX_OFFLINE=true` per usare i metadati delle query in cache. Il file `sqlx-data.json` nel repo fornisce questa cache.

**Il frontend non riesce a raggiungere l'API:**
In modalità di sviluppo, il dev server di Vite fa da proxy alle richieste `/api` verso il backend. Assicurati che il container del backend sia in esecuzione sulla porta 3000. Controlla `dev.compose.yml` per le corrette mappature delle porte.

**L'analisi ritorna un disassembly vuoto:**
Il disassembly viene eseguito solo sui binari x86 e x86_64. I binari ARM e AArch64 avranno tutti gli altri pass completati normalmente, ma la scheda del disassembly sarà vuota.

## Progetti correlati

| Progetto | Collegamento |
|---------|------------|
| [Network Traffic Analyzer](../../beginner/network-traffic-analyzer/) | Cattura traffico che l'analisi dei binari può correlare con la comunicazione C2 |
| [SIEM Dashboard](../siem-dashboard/) | Acquisisce i risultati delle analisi come eventi di sicurezza per il monitoraggio centralizzato |
| [Encrypted P2P Chat](../../advanced/encrypted-p2p-chat/) | Usa primitive crittografiche che l'analisi dell'entropia dei binari segnalerebbe |
