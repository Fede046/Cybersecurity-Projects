<!-- © AngelaMos | 2026 | 02-ARCHITECTURE.md -->

# Architettura

## Panoramica del sistema

Lo strumento legge un file di ruleset grezzo (l'output di `iptables-save` o `nft list ruleset`), lo converte in una rappresentazione interna e poi esegue una delle varie operazioni disponibili in base al sottocomando eseguito. Ecco il quadro completo:

```
                         ┌──────────────┐
                         │  Raw Ruleset │
                         │    (file)    │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │ detect_format│
                         └──────┬───────┘
                                │
                 ┌──────────────┼──────────────┐
                 │                             │
          ┌──────▼───────┐              ┌──────▼───────┐
          │parse_iptables│              │parse_nftables│
          └──────┬───────┘              └──────┬───────┘
                 │                             │
                 └──────────────┬──────────────┘
                                │
                         ┌──────▼───────┐
                         │   Ruleset    │
                         │ (internal)   │
                         └──────┬───────┘
                                │
          ┌─────────┬───────────┼───────────┬──────────┐
          │         │           │           │          │
   ┌──────▼──┐ ┌───▼────┐ ┌───▼────┐ ┌────▼───┐ ┌───▼────┐
   │ display │ │analyze │ │optimize│ │ export │ │  diff  │
   │  table  │ │conflict│ │suggest │ │ruleset │ │compare │
   └─────────┘ └───┬────┘ └───┬────┘ └────┬───┘ └───┬────┘
                   │           │           │         │
            ┌──────▼───────────▼──┐   ┌────▼───┐ ┌──▼─────┐
            │   print_findings    │   │ string │ │print_  │
            └─────────────────────┘   │ output │ │ diff   │
                                      └────────┘ └────────┘
```

Ciascun sottocomando della CLI mappa su una funzione in `main.v` che chiama uno o più moduli. Nessun sottocomando tocca più di due o tre moduli. Il comando `harden` fa eccezione: salta del tutto il parser e va dritto al generator.

| Comando    | Moduli utilizzati                  |
|------------|------------------------------------|
| `load`     | parser, display                    |
| `analyze`  | parser, analyzer, display          |
| `optimize` | parser, analyzer, display          |
| `harden`   | generator, display (solo banner)   |
| `export`   | parser, generator                  |
| `diff`     | parser, display                    |


## Disposizione dei moduli

In V, ogni directory sotto `src/` è un modulo. Il nome del modulo corrisponde al nome della directory. I file all'interno di un modulo condividono automaticamente lo stesso namespace, quindi `common.v`, `iptables.v` e `nftables.v` appartengono tutti al `module parser` e possono chiamare direttamente le funzioni l'uno dell'altro.

```
src/
├── main.v          (module main)
├── config/
│   └── config.v    (module config)
├── models/
│   └── models.v    (module models)
├── parser/
│   ├── common.v    (module parser)
│   ├── iptables.v  (module parser)
│   └── nftables.v  (module parser)
├── analyzer/
│   ├── conflict.v  (module analyzer)
│   └── optimizer.v (module analyzer)
├── generator/
│   └── generator.v (module generator)
└── display/
    └── display.v   (module display)
```

Il grafo delle dipendenze:

```
                        ┌──────┐
                        │ main │
                        └──┬───┘
                           │
          ┌────────┬───────┼────────┬──────────┐
          │        │       │        │          │
      ┌───▼──┐ ┌──▼───┐ ┌─▼──┐ ┌──▼────┐ ┌───▼───┐
      │parser│ │analyz.│ │gen.│ │display│ │config │
      └──┬───┘ └──┬───┘ └─┬──┘ └──┬────┘ └───────┘
         │        │       │       │
         │     ┌──▼───┐   │    ┌──▼───┐
         ├────►│models │◄──┘   │models│
         │     └──┬───┘        └──┬───┘
         │        │               │
         ▼        ▼               ▼
      ┌──────┐ ┌──────┐       ┌──────┐
      │config│ │config│       │config│
      └──────┘ └──────┘       └──────┘
```

Tre cose da notare:

- `config` e `models` sono moduli foglia. Non importano nulla dal progetto.
- `parser`, `analyzer`, `generator` e `display` non si importano mai reciprocamente. Zero dipendenze incrociate.
- `main` è l'unico modulo che importa tutto. È la root di composizione (composition root).

Ciò significa che puoi riscrivere l'intero parser nftables senza toccare l'analyzer, oppure rinnovare il layer di visualizzazione senza che il generator ne sia a conoscenza. V impedisce le importazioni circolari in fase di compilazione, quindi questa struttura non può degradarsi accidentalmente.


## Flusso dei dati

### Comando `load`

```
fwrule load testdata/iptables_basic.rules

  os.read_file(path)
       │
       ▼
  detect_format(content)  ──►  RuleSource.iptables
       │
       ▼
  parse_iptables(content)
       │
       ├── iterate lines
       ├── "*filter"         → set current_table = .filter
       ├── ":INPUT DROP"     → policies["INPUT"] = .drop
       ├── "-A INPUT ..."    → tokenize → parse flags → Rule
       │
       ▼
  Ruleset { rules: [...], policies: {...}, source: .iptables }
       │
       ├── display.print_banner()
       ├── display.print_summary(rs)
       └── display.print_rule_table(rs)
```

`load_ruleset` in `main.v` legge il file, rileva automaticamente il formato ed esegue il dispatch verso il parser corretto. Il `Ruleset` risultante passa al modulo display per il rendering.

### Comando `analyze`

```
fwrule analyze testdata/iptables_conflicts.rules

  load_ruleset(path)  ──►  Ruleset
       │
       ├── analyzer.analyze_conflicts(rs)
       │       │
       │       ├── rs.rules_by_chain()  →  map[string][]int
       │       │
       │       └── for each chain:
       │              find_duplicates(rules, indices)
       │              find_shadowed_rules(rules, indices)
       │              find_contradictions(rules, indices)
       │              find_redundant_rules(rules, indices)
       │       │
       │       └──►  []Finding
       │
       ├── analyzer.suggest_optimizations(rs)
       │       │
       │       └── for each chain:
       │              find_mergeable_ports(rules, indices)
       │              suggest_reordering(rules, indices)
       │              find_missing_rate_limits(rules, indices)
       │              find_missing_conntrack(rules, indices)
       │       │
       │       └── find_missing_logging(rs)  (whole-ruleset check)
       │       │
       │       └──►  []Finding
       │
       └── display.print_findings(conflicts)
           display.print_findings(optimizations)
```

Due passaggi: rilevamento dei conflitti (cose non funzionanti) e poi analisi di ottimizzazione (cose che potrebbero essere migliorate). Entrambi restituiscono `[]Finding` che il modulo display visualizza colorati in base alla gravità.

### Comando `harden`

```
fwrule harden -s ssh,http,https -f nftables

  flag.new_flag_parser(args)
       │
       ├── services = ["ssh", "http", "https"]
       ├── iface = "eth0", format = nftables
       │
       ▼
  generator.generate_hardened(services, iface, .nftables)
       │
       ├── default-deny policy → loopback accept → conntrack
       ├── anti-spoofing (RFC 1918 on public iface)
       ├── ICMP rate-limited
       ├── per-service rules from config.service_ports
       ├── drop logging → final drop
       │
       └──►  string (printed to stdout)
```

Questo è l'unico comando che non analizza un file. Costruisce un ruleset da zero usando template e la mappatura servizio-porta da `config.v`.

### Comando `export`

```
fwrule export testdata/iptables_basic.rules -f nftables

  load_ruleset(path)  ──►  Ruleset (source: .iptables)
       │
       ▼
  generator.export_ruleset(rs, .nftables)
       │
       ├── group rules by table
       ├── for each chain: header + policy, then rule_to_nftables per rule
       │
       └──►  string (printed to stdout)
```

Ogni struct `Rule` contiene informazioni sufficienti per essere serializzata in entrambi i formati. Le funzioni `rule_to_iptables` e `rule_to_nftables` leggono i campi dalla struct e ricostruiscono la sintassi di destinazione.

### Comando `diff`

```
fwrule diff old.rules new.rules

  load_ruleset(path1)  ──►  Ruleset (left)
  load_ruleset(path2)  ──►  Ruleset (right)
       │
       ▼
  display.print_diff(left, right)
       │
       ├── build_rule_set(rules)  →  map[string]bool  (both sides)
       ├── keys in left but not right → "only in left"
       ├── keys in right but not left → "only in right"
       └── no differences → "Rulesets are equivalent"
```

Il diff normalizza ogni regola in una stringa canonica tramite `Rule.str()` e confronta gli insiemi. Confronta il contenuto semantico, non il testo grezzo, quindi una regola iptables e una regola nftables che esprimono la stessa policy vengono mostrate come equivalenti.


## Tipi fondamentali

Tutti i tipi risiedono in `src/models/models.v`. Il parser li produce, l'analyzer li ispeziona, i moduli generator e display li consumano.

### Ruleset

```
Ruleset {
    rules    []Rule              ordered list of all parsed rules
    policies map[string]Action   chain name → default action ("INPUT" → .drop)
    source   RuleSource          iptables or nftables
}
```

Il contenitore di livello superiore. `rules` è ordinato in base alla posizione nel file originale. `policies` mappa i nomi delle catene sulle loro azioni predefinite. Il metodo `rules_by_chain()` raggruppa gli indici delle regole per nome di catena in modo che l'analyzer possa limitare i confronti all'interno di una singola catena.

### Rule

```
Rule {
    table       Table            filter, nat, mangle, raw, security
    chain       string           "INPUT", "FORWARD", or custom name
    chain_type  ChainType        parsed enum for known chains
    action      Action           accept, drop, reject, log, masquerade, ...
    criteria    MatchCriteria    all match conditions (see below)
    target_args string           extra args after -j (e.g., --log-prefix "...")
    line_number int              original line number in source file
    raw_text    string           unparsed original line
    source      RuleSource       which format this rule came from
}
```

Che l'input fosse iptables o nftables, ogni regola analizzata diventa questa stessa struct. `chain_type` assume come valore predefinito `.custom` per le catene definite dall'utente. `line_number` e `raw_text` sopravvivono al parsing in modo che i finding possano fare riferimento al file originale.

### MatchCriteria

```
MatchCriteria {
    protocol    Protocol         default: .all (matches everything)
    source      ?NetworkAddr     optional source CIDR
    destination ?NetworkAddr     optional destination CIDR
    src_ports   []PortSpec       source port ranges
    dst_ports   []PortSpec       destination port ranges
    in_iface    ?string          input interface
    out_iface   ?string          output interface
    states      ConnState        bitmask: new|established|related|invalid
    icmp_type   ?string          ICMP type string
    limit_rate  ?string          rate limit (e.g., "3/minute")
    limit_burst ?int             burst count
    comment     ?string          rule comment
}
```

È qui che i tipi opzionali di V (`?Type`) mostrano il loro valore. `source ?NetworkAddr` significa "questa regola può vincolare o meno l'indirizzo sorgente". Quando è `none`, la regola corrisponde a qualsiasi sorgente. Quando è impostata, corrisponde solo a quella rete. Questa distinzione è fondamentale per la logica di superset/subset: `source = none` è un superset di `source = 10.0.0.0/8`, poiché "corrisponde a qualsiasi cosa" include "corrisponde a questa rete". Senza i tipi opzionali non saresti in grado di distinguere "nessun vincolo" da "corrisponde esplicitamente a 0.0.0.0/0".

### Finding

```
Finding {
    severity     Severity        info, warning, critical
    title        string          short label ("Shadowed rule detected")
    description  string          full explanation with rule numbers
    rule_indices []int           zero-based indices into Ruleset.rules
    suggestion   string          actionable fix
}
```

L'output sia del rilevamento dei conflitti sia dell'analisi delle ottimizzazioni. `rule_indices` contiene indici in base zero all'interno di `Ruleset.rules`, consentendo al layer display di mostrare "Regole 7, 8" senza dover mantenere gli oggetti regola completi.

### NetworkAddr e PortSpec

```
NetworkAddr {                PortSpec {
    address string               start   int
    cidr    int = 32             end     int = -1
    negated bool                 negated bool
}                            }
```

`NetworkAddr` memorizza un IP e la lunghezza del prefisso. Il campo `cidr` è impostato di default su 32 (un singolo host). Il flag `negated` gestisce i prefissi `!` sia in iptables (`! -s 10.0.0.0/8`) sia in nftables (`ip saddr != 10.0.0.0/8`).

`PortSpec` memorizza una porta o un intervallo di porte. Una singola porta come 22 ha `end = -1`, e `effective_end()` restituisce `start` in quel caso in modo che i calcoli sugli intervalli funzionino uniformemente. Un intervallo come `1024:65535` ha `start = 1024, end = 65535`.

`cidr_contains` e `port_range_contains` sono le due primitive di contenimento su cui è costruita l'intera logica superset/subset dell'analyzer.

### ConnState come enum @[flag]

```
@[flag]
pub enum ConnState {
    new_conn       bit 0 → value 1
    established    bit 1 → value 2
    related        bit 2 → value 4
    invalid        bit 3 → value 8
    untracked      bit 4 → value 16
}
```

L'attributo `@[flag]` rende questo un bitfield. Ciascuna variante è una potenza di due, e un singolo valore `ConnState` può rappresentare più stati contemporaneamente. `ESTABLISHED,RELATED` corrisponde a due bit impostati in un unico intero. I metodi `set()`, `has()`, `all()` e `is_empty()` vengono generati automaticamente da V.

Questo rispecchia il reale funzionamento del sistema di conntrack del kernel: gli stati di connessione sono flag bitmask, non valori mutuamente esclusivi. Un pacchetto in stato `ESTABLISHED` non è contemporaneamente `NEW`, ma una regola può corrispondere sia a `ESTABLISHED` sia a `RELATED` simultaneamente. Il bitfield rende banali i controlli di subset nell'analyzer: `outer.states.all(inner.states)` è una singola operazione di AND a livello di bit.


## Progettazione dei parser

Il parser risolve un problema di doppio formato. L'output di `iptables-save` e quello di `nft list ruleset` esprimono gli stessi concetti di firewalling ma con una sintassi completamente differente.

### Parser di iptables (iptables.v)

L'output di `iptables-save` è orientato alle righe. Ogni regola è una singola riga con coppie flag-valore:

```
-A INPUT -p tcp -s 10.0.0.0/8 --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

Il parser lavora in due fasi. `tokenize_iptables` divide sugli spazi bianchi rispettando le stringhe racchiuse tra virgolette, dopodiché l'iteratore di token consuma le coppie flag-valore:

```
["-A", "INPUT", "-p", "tcp", "-s", "10.0.0.0/8", "--dport", "22", "-j", "ACCEPT"]
  │       │       │      │     │         │            │         │     │       │
  └─chain─┘       └proto─┘    └──source──┘            └──port───┘    └action─┘
```

L'operatore di negazione `!` è gestito da un flag `next_negated` che si propaga al successivo indirizzo o porta analizzato. A livello di file, `parse_iptables` itera su tutte le righe: `*filter` imposta la tabella corrente, `:INPUT DROP [0:0]` registra le policy delle catene, `COMMIT` viene ignorato e le righe che iniziano con `-A`/`-I` vengono inviate al parser delle regole.

### Parser di nftables (nftables.v)

L'output di nftables è strutturato a blocchi delimitati da parentesi graffe:

```
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        tcp dport 22 accept
    }
}
```

Il parser usa un'iterazione riga per riga con tre livelli di annidamento:

```
parse_nftables            scans for "table" lines
    │
    └── parse_nft_table   extracts table name, scans for "chain" lines
            │
            └── parse_nft_chain   extracts chain name + policy, scans for rule lines
                    │
                    └── parse_nft_rule   tokenizes a single rule line
```

Ciascuna funzione accetta l'intero array `lines []string` e un indice iniziale, restituendo il nuovo indice dopo aver consumato il proprio blocco. La chiusura `}` restituisce il controllo al chiamante.

All'interno di ciascuna riga di regola, i token di parole chiave guidano il parsing: `tcp`/`udp` impostano il protocollo e avviano il parsing delle porte, `ip saddr`/`daddr` estraggono gli indirizzi, `ct state` estrae il connection tracking e le parole chiave terminali (`accept`, `drop`, `reject`) impostano l'azione. `parse_nft_port_match` gestisce sia porte singole (`dport 22`) sia insiemi racchiusi tra graffe (`dport { 80, 443 }`).

### Layer di parsing condiviso (common.v)

Entrambi i parser condividono le funzioni definite in `common.v`: `parse_network_addr` (CIDR + negazione), `parse_port_spec` (porte singole, intervalli, negazione), `parse_port_list` (separate da virgola), `parse_protocol` (nomi e numeri convertiti in enum), `parse_action`, `parse_table`, `parse_chain_type` e `parse_conn_states` (stati separati da virgola convertiti in bitfield).

`detect_format` esamina la prima riga non vuota e non di commento. `*`, `:` o `-A` indicano iptables. `table` indica nftables.


## Progettazione dell'analyzer

### Confronto a coppie

L'analyzer raggruppa le regole per catena tramite `rs.rules_by_chain()`, poi confronta ogni coppia all'interno di ciascuna catena (N*(N-1)/2 confronti per catena). Le regole in catene diverse non vengono mai confrontate perché il kernel valuta ciascuna catena in modo indipendente.

### I quattro tipi di conflitto

```
┌────────────────┬─────────────────────────────────────────────────────┐
│ Tipo           │ Come viene rilevato                                 │
├────────────────┼─────────────────────────────────────────────────────┤
│ Shadowed       │ La regola A appare prima della regola B nella       │
│ (Oscurata)     │ catena. I criteri di A sono un superset di quelli   │
│                │ di B. B non potrà mai scattare perché A cattura     │
│                │ tutto il suo traffico prima.                        │
├────────────────┼─────────────────────────────────────────────────────┤
│ Contradiction  │ Le regole A e B si sovrappongono nei criteri di     │
│ (Contraddiz.)  │ corrispondenza ma hanno azioni opposte (una accetta,│
│                │ una scarta o rifiuta). Non è un superset completo,  │
│                │ altrimenti verrebbe classificato come shadowing.    │
├────────────────┼─────────────────────────────────────────────────────┤
│ Duplicate      │ Due regole hanno criteri identici E la stessa       │
│ (Duplicata)    │ azione. La seconda è solo un peso morto.            │
├────────────────┼─────────────────────────────────────────────────────┤
│ Redundant      │ La regola A è un superset della regola B con la     │
│ (Ridondante)   │ stessa azione, ma non sono duplicati esatti. B è    │
│                │ non necessaria ma non dannosa.                      │
└────────────────┴─────────────────────────────────────────────────────┘
```

### Matematica superset/subset

"La regola A cattura ogni pacchetto catturato dalla regola B?" viene scomposto campo per campo.

Il **contenimento CIDR** converte gli IP in interi a 32 bit e confronta i prefissi tramite shift di bit:

```
outer = 10.0.0.0/8       inner = 10.1.2.0/24

ip_to_u32("10.0.0.0")  = 0x0A000000
ip_to_u32("10.1.2.0")  = 0x0A010200

shift = 32 - 8 = 24

0x0A000000 >> 24 = 0x0A
0x0A010200 >> 24 = 0x0A

Stesso prefisso dopo lo shift → 10.0.0.0/8 contiene 10.1.2.0/24
```

Il **contenimento dell'intervallo di porte** è un semplice controllo dei limiti:

```
outer = 1024:65535     inner = 8080:8443

outer.start (1024) <= inner.start (8080)  ✓
outer.end (65535)  >= inner.end (8443)    ✓

→ outer contiene inner
```

**Gerarchia dei protocolli**: il protocollo `.all` è un superset di qualsiasi protocollo specifico. Se la regola esterna corrisponde a `.all` e quella interna a `.tcp`, la regola esterna copre tutto ciò che copre quella interna.

**Gestione dei tipi opzionali**: `none` (nessun vincolo) è un superset di qualsiasi valore specifico. `source = none` copre `source = 10.0.0.0/8` perché "corrisponde a qualsiasi cosa" include "corrisponde a questa rete". Se la regola esterna ha un indirizzo specifico, anche quella interna deve averne uno, e il contenimento CIDR deve essere valido.

### Perché i finding contengono indici di regole

Ogni `Finding` include `rule_indices` che puntano a posizioni specifiche in `Ruleset.rules`. Il layer display li usa per stampare "Regole: 7, 12" accanto a ogni finding senza dover mantenere gli oggetti `Rule` stessi.


## Progettazione del generator

### Ruleset rafforzati basati su template

`generate_hardened` esegue il dispatch verso `generate_iptables_hardened` o `generate_nftables_hardened`. Entrambi costruiscono un array di stringhe riga per riga seguendo lo stesso template logico: policy default-deny, accept su loopback, conntrack, anti-spoofing (RFC 1918 sull'interfaccia pubblica), ICMP con rate limit, regole per servizio da `config.service_ports`, logging dei drop e un DROP finale esplicito. SSH riceve un rate limit (`3/minute` burst 5). Il DNS ottiene sia TCP che UDP. NTP ottiene solo UDP. Tutto il resto ottiene TCP.

### Esportazione del formato

`export_ruleset` itera su ogni regola e chiama `rule_to_iptables` o `rule_to_nftables` per ricostruire la sintassi:

```
Rule { protocol: .tcp, dst_ports: [PortSpec{22}], action: .accept }
    │
    ├── rule_to_iptables → "-A INPUT -p tcp --dport 22 -j ACCEPT"
    └── rule_to_nftables → "tcp dport 22 accept"
```

Le funzioni di esportazione gestiscono anche gli elementi strutturali: header di tabella, dichiarazioni di catena con policy e marcatori specifici del formato come `COMMIT` per iptables.


## Decisioni architetturali

### Perché V

V compila in un binario nativo senza dipendenze a runtime. Esegui `v .` e ottieni un singolo eseguibile. Nessun interprete, nessuna VM, nessuna libreria condivisa oltre a libc. Per uno strumento di sicurezza destinato a girare su server blindati, questo fa la differenza. Il file `v.mod` indica `dependencies: []`.

La sintassi è volutamente semplice. Se sai leggere C, Go o Python, sai leggere V all'istante. I tipi opzionali (`?Type`) offrono la null safety senza la complessità cerimoniale di Rust. L'attributo enum `@[flag]` fornisce gratuitamente operazioni bitfield, mappandosi perfettamente sul funzionamento degli stati di conntrack nel kernel.

### Perché il confronto a coppie invece di un albero di decisione o BDD

È un algoritmo O(n^2), ed esistono approcci asintoticamente più veloci (alberi di decisione, BDD, alberi di intervalli). Ma con 100 regole per catena, il confronto a coppie esegue 4.950 controlli tra interi, terminando in meno di un millisecondo. Persino con 1.000 regole (caso estremo) si producono circa 500.000 confronti, comunque completati in pochi millisecondi.

Ancora più importante: il confronto a coppie genera finding che fanno riferimento esattamente a due regole. "La regola 7 oscura la regola 12" è immediatamente utilizzabile e comprensibile. Un approccio basato su BDD richiederebbe ulteriore lavoro per risalire alle specifiche regole coinvolte.

### Perché nessuna dipendenza esterna

La libreria standard di V fornisce `os` (I/O su file), `flag` (parsing degli argomenti), `term` (colori ANSI) e `strings` (manipolazione di stringhe). Ciò copre tutto il necessario. Le dipendenze esterne nei tool di sicurezza creano rischi di supply chain. Un singolo binario statico può essere copiato su qualsiasi sistema Linux ed eseguito immediatamente senza l'intervento di alcun package manager.

### Perché moduli separati invece di un singolo file

Si sarebbe potuto inserire tutto in un unico file: per V non farebbe differenza. Ma avere moduli separati garantisce confini imposti dal compilatore (il parser non può chiamare funzioni di display), file di test indipendenti (`v test src/parser/` esegue i test del parser in isolamento) e una chiara navigabilità (un bug nel rilevamento dei conflitti implica controllare `src/analyzer/conflict.v`).

Aggiungere un nuovo parser (ad esempio per le regole `ufw`) richiederebbe un nuovo file in `src/parser/`, una nuova variante in `RuleSource`, un nuovo ramo `case` in `detect_format` e un nuovo ramo in `load_ruleset`. Nessuna modifica sarebbe necessaria in analyzer, generator o display, in quanto operano già sull'astrazione comune `Ruleset`.