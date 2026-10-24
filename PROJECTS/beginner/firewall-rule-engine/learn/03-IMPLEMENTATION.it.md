<!-- © AngelaMos | 2026 | 03-IMPLEMENTATION.md -->

# Analisi guidata del codice

Questo documento esamina il codice sorgente effettivo file per file. Ogni riferimento a funzioni include la sua posizione così da poterlo aprire e leggere in parallelo. L'obiettivo è rendere l'implementazione comprensibile, non ripetere il codice parola per parola. Apri i file nel tuo editor mentre leggi.

---

## Pattern del linguaggio V utilizzati

Un rapido riferimento ai pattern di V che incontrerai in tutta la codebase. Se conosci già Go o C, la maggior parte ti risulterà familiare. I concetti principali che potrebbero sorprenderti sono i tipi opzionali (option types) e gli enum flag.

**Sistema dei moduli.** Ogni file inizia con `module nome`. Le importazioni usano una sintassi selettiva: `import src.models { Rule, Ruleset, Finding }` porta i tipi specifici nello scope senza richiedere il prefisso `models.` a ogni chiamata. Il nome della directory è il nome del modulo. Tutti i file in una directory condividono lo stesso scope di modulo, motivo per cui i file di test possono accedere a funzioni private.

**Tipi opzionali.** `?Type` indica che il valore potrebbe essere `none`. Due modi per fare unwrap:

- `if val := optional { ... }` in cui il `:=` all'interno dell'`if` assegna il valore sottoposto a unwrap solo se non è `none`
- `val := opt or { default }` per fornire un fallback

Il parser fa ampio uso dei tipi opzionali per campi come `source ?NetworkAddr` e `in_iface ?string` nella struct `MatchCriteria` in `models.v`. Una regola di firewall può specificare o meno un indirizzo sorgente, e "non specificato" è semanticamente diverso da qualsiasi valore di indirizzo. Il tipo opzionale rende impossibile dimenticare questa distinzione.

**Tipi di risultato.** `!Type` significa che la funzione potrebbe restituire un errore. Stessi pattern di unwrap dei tipi opzionali. L'operatore di propagazione `!` consente ai chiamanti di far risalire gli errori verso l'alto senza scrivere codice boilerplate di gestione degli errori: `parse_network_addr(tokens[i])!` restituisce l'errore al chiamante se il parsing fallisce. La maggior parte delle funzioni di parsing restituisce `!` poiché l'input può sempre essere malformato.

**Enum flag.** L'attributo `@[flag]` su una dichiarazione enum la rende un bitfield invece di un enum a valore singolo. Ciascuna variante occupa una posizione di bit. L'enum `ConnState` in `models.v` usa questo approccio in modo che una singola variabile possa contenere qualsiasi combinazione di stati di connessione. Le operazioni: `.has()` verifica un flag, `.set()` ne attiva uno, `.all()` controlla se tutti i flag in un valore sono presenti in un altro, `.is_empty()` verifica se non ci sono flag impostati, `.zero()` crea un valore senza alcun flag impostato.

**Interpolazione di stringhe.** `'testo ${expression} altro testo'` con `${}` per qualsiasi espressione. V chiama automaticamente il metodo `.str()` quando interpoli un tipo che ne possiede uno, motivo per cui ogni enum in models.v definisce un metodo `str()`.

**Metodi per array.** `.map()`, `.filter()`, `.any()`, `.all()`, `.contains()` funzionano come ci si aspetta dai linguaggi funzionali. La variabile implicita `it` si riferisce all'elemento corrente. Ad esempio, nella funzione `find_mergeable_ports` in `optimizer.v`, `entries.map(it[0])` estrae il primo elemento da ciascun sotto-array per ricavare gli indici delle regole.

**Operatore `in`.** Verifica l'appartenenza agli array: `dp.start in high_traffic_ports` nella funzione `suggest_reordering` in `optimizer.v`. Funziona anche sulle mappe: `key !in tables_seen` nella funzione `export_as_iptables` in `generator.v`.

**`mut` per la mutabilità.** Variabili e parametri sono immutabili per impostazione predefinita. Dichiara `mut i := 0` per consentire la modifica. Anche i parametri di funzione che la funzione stessa modifica devono essere dichiarati `mut` nella signature, come la funzione `parse_nft_table` in `nftables.v` che accetta `mut ruleset Ruleset`.

---

## Punto di ingresso CLI (main.v)

`src/main.v`

Il punto di ingresso è un dispatcher di sottocomandi. La funzione `main` controlla `os.args.len`, estrae il sottocomando da `os.args[1]` ed esegue il dispatch tramite un'istruzione `match` verso uno di: `cmd_load`, `cmd_analyze`, `cmd_optimize`, `cmd_harden`, `cmd_export`, `cmd_diff`, `cmd_version`, `cmd_help`. I comandi sconosciuti stampano un errore ed escono con `config.exit_usage_error`.

Ciascuna funzione di comando segue lo stesso pattern: valida gli argomenti, chiama `load_ruleset` per analizzare il file di input, quindi chiama le funzioni del modulo appropriato e mostra i risultati.

**`load_ruleset`**: Il ponte tra la CLI e il resto del sistema. Legge il file, chiama `parser.detect_format` per rilevare automaticamente iptables vs nftables ed esegue il dispatch verso il parser corretto:

```v
fn load_ruleset(path string) !models.Ruleset {
	if !os.exists(path) {
		return error('file not found: ${path}')
	}
	content := os.read_file(path) or { return error('cannot read file: ${path}') }
	fmt := parser.detect_format(content)!
	return match fmt {
		.iptables { parser.parse_iptables(content)! }
		.nftables { parser.parse_nftables(content)! }
	}
}
```

Tutti e tre i passaggi propagano gli errori con `!`, quindi un percorso file non valido, un formato non riconosciuto o un errore di parsing producono ciascuno un messaggio di errore pulito.

**`cmd_harden`**: Il comando più complesso perché accetta flag. Usa il modulo `flag` di V per analizzare `--services` (nomi di servizi separati da virgola, predefinito su `config.default_services`), `--iface` (interfaccia di rete, predefinito su `config.default_iface`) e `--format` (iptables o nftables). Dopo aver analizzato i flag, divide la stringa dei servizi sulle virgole, rimuove gli spazi bianchi e chiama `generator.generate_hardened`.

**`cmd_analyze`**: Carica il ruleset, stampa un riepilogo, quindi esegue sia il rilevamento dei conflitti (`analyzer.analyze_conflicts`) sia i suggerimenti di ottimizzazione (`analyzer.suggest_optimizations`). I risultati vengono stampati come due sezioni separate con i rispettivi header.

**`cmd_diff`**: Carica due ruleset da due file diversi e li passa entrambi a `display.print_diff`. Poiché `load_ruleset` rileva automaticamente il formato, puoi eseguire il diff tra un file iptables e uno nftables. Il confronto usa la forma canonica `Rule.str()` che normalizza entrambi i formati alla stessa rappresentazione.

---

## Modulo Config (config.v)

`src/config/config.v`

Questo file contiene esclusivamente dichiarazioni `pub const`. Nessuna funzione, nessuna logica, nessuno stato. Tutto ciò di cui il resto della codebase ha bisogno come valore fisso risiede qui. Il punto è che nessun altro file contiene magic number o stringhe letterali che potrebbero disallinearsi nel tempo.

**Codici di uscita (Exit codes)**: `exit_success = 0`, `exit_parse_error = 1`, `exit_file_error = 2`, `exit_analysis_error = 3`, `exit_usage_error = 64`. Il codice di errore di utilizzo è 64 secondo la convenzione `sysexits.h` di BSD, che riserva i codici 64-78 per errori specifici del programma. Il punto di ingresso CLI in `main.v` li usa per ogni chiamata a `exit()`.

**Porte ben note (Well-known ports)**: Costanti con nome per SSH (22), DNS (53), HTTP (80), HTTPS (443), SMTP (25), NTP (123). La funzione `find_missing_rate_limits` in `optimizer.v` fa riferimento a `port_ssh` quando verifica i rate limit mancanti. Il generator li usa indirettamente tramite la mappa `service_ports`.

**CIDR e intervalli di rete**: `cidr_max_v4 = 32`, `private_ranges` che elenca lo spazio RFC 1918 (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) e gli indirizzi di loopback per v4 e v6. La funzione `generate_iptables_hardened` in `generator.v` itera su `private_ranges` per costruire regole di anti-spoofing che scartano i pacchetti che dichiarano di provenire da indirizzi privati quando arrivano sull'interfaccia pubblica.

**Rate limit**: `ssh_rate_limit = '3/minute'`, `ssh_rate_burst = 5`, `icmp_rate_limit = '1/second'`, `icmp_rate_burst = 5`. Questi alimentano direttamente la generazione dei template rafforzati. Il rate limiting di SSH rende impraticabili gli attacchi brute-force: 3 nuove connessioni al minuto con una tolleranza di burst pari a 5. Il rate limiting di ICMP previene gli attacchi di ping flood consentendo al contempo le richieste echo legittime.

**Costanti di visualizzazione**: Larghezze delle colonne per la tabella delle regole (`col_num = 5`, `col_chain = 12`, ecc.) e simboli Unicode per l'output da terminale (`sym_check`, `sym_cross`, `sym_warn`, `sym_arrow`, `sym_bullet`). Il modulo display li usa per costruire colonne a larghezza fissa senza dipendere dal rilevamento della larghezza del terminale.

**Mappa dei servizi**: `service_ports` mappa i nomi dei servizi sui numeri di porta. Questo è un letterale `map[string]int`. Quando esegui `fwrule harden -s ssh,http,https`, il generator cerca ciascun nome in questa mappa tramite `config.service_ports[svc] or { continue }`. I nomi di servizio sconosciuti vengono ignorati silenziosamente. La mappa copre ssh, dns, http, https, smtp, ntp, ftp, mysql, pg (PostgreSQL) e redis.

---

## Modulo Models (models.v)

`src/models/models.v`

Questo file definisce ogni tipo su cui opera il resto della codebase, oltre alle funzioni matematiche CIDR da cui dipende l'analyzer. È il vocabolario condiviso tra tutti i moduli.

### Enum

Sei enum: `Protocol`, `Action`, `Table`, `ChainType`, `RuleSource`, `Severity`. Ciascuno ha un metodo `.str()` che restituisce la forma canonica in formato stringa. Tutti usano il tipo di supporto `as u8` per mantenere ridotta l'occupazione di memoria.

`Protocol`: tcp, udp, icmp, icmpv6, all, sctp, gre. La variante `all` corrisponde a qualsiasi protocollo ed è l'impostazione predefinita quando una regola non ne specifica uno.

`Action`: Copre l'intero set di target di iptables. La variante `return_action` è denominata così perché `return` è una parola chiave di V. Il metodo `.str()` la mappa su `"RETURN"`.

`Table`: filter, nat, mangle, raw, security. La maggior parte dei ruleset usa solo filter. I file di testdata complessi usano filter e nat insieme.

`ChainType`: input, output, forward, prerouting, postrouting, custom. La variante `custom` è il raccoglitore per le catene definite dall'utente.

### ConnState come Flag Enum

```v
@[flag]
pub enum ConnState {
    new_conn
    established
    related
    invalid
    untracked
}
```

Cinque varianti. Trattandosi di un flag enum, ciascuna variante è un singolo bit. `new_conn` è il bit 0, `established` è il bit 1, `related` è il bit 2 e così via. Un singolo valore `ConnState` può contenere qualsiasi combinazione. Quando il parser legge `ESTABLISHED,RELATED`, chiama `.set(.established)` e `.set(.related)`, producendo un valore in cui i bit 1 e 2 sono impostati: `0b0110`.

Perché usare i bitfield anziché un array? Perché l'analyzer deve confrontare insiemi di stati in modo efficiente. "L'insieme degli stati della regola esterna contiene tutti gli stati della regola interna?" si risolve in una singola chiamata `outer.states.all(inner.states)`, che sotto il cofano è un AND a livello di bit. Un confronto tramite array richiederebbe cicli annidati.

### NetworkAddr

Tre campi: `address` (stringa), `cidr` (int, predefinito su 32), `negated` (bool). Un IP semplice come `192.168.1.1` ottiene `cidr = 32` (un singolo host). Un CIDR come `10.0.0.0/8` riceve `cidr = 8`. Gli indirizzi negati provenienti da `! -s 10.0.0.0/8` ottengono `negated = true`.

### ip_to_u32

La funzione `ip_to_u32` in `models.v` converte una stringa IP in notazione dotted-quad in un intero a 32 bit senza segno per calcoli matematici CIDR a livello di bit. Suddivide su `.`, valida che vi siano esattamente quattro ottetti, quindi elabora ciascuno di essi. Ogni ottetto viene validato carattere per carattere per garantire la presenza di sole cifre ASCII:

```v
for ch in trimmed.bytes() {
	if ch < `0` || ch > `9` {
		return error('invalid octet in address: ${ip}')
	}
}
val := trimmed.int()
if val < 0 || val > 255 {
	return error('invalid octet in address: ${ip}')
}
result = (result << 8) | u32(val)
```

La validazione per singolo carattere rifiuta qualsiasi cosa non sia una cifra prima di invocare `.int()` per la conversione, e un controllo di intervallo cattura i valori al di fuori di 0-255. Lo shift e l'OR accumulano i quattro ottetti in un unico valore a 32 bit.

Passo dopo passo per `192.168.1.1`:

| Iterazione | Ottetto | result prima dello shift | << 8     | \| ottetto   | result     |
|------------|---------|--------------------------|----------|--------------|------------|
| 1          | 192     | 0                        | 0        | 192          | 192        |
| 2          | 168     | 192                      | 49152    | 49320        | 49320      |
| 3          | 1       | 49320                    | 12625920 | 12625921     | 12625921   |
| 4          | 1       | 12625921                 | 3232235776| 3232235777  | 3232235777 |

Il valore finale `3232235777` equivale a `0xC0A80101`, ovvero `192*2^24 + 168*2^16 + 1*2^8 + 1`. Questo impacchetta quattro byte in un solo intero, che è la rappresentazione standard per gli indirizzi IPv4 nel codice di rete.

### cidr_contains

La funzione `cidr_contains` in `models.v` determina se la rete `inner` si trova all'interno della rete `outer`. Tre controlli:

Primo: se `outer.cidr > inner.cidr`, restituisce false immediatamente. Un /24 non può contenere un /8. La rete esterna deve avere ampiezza uguale o maggiore.

Secondo: se `outer.cidr == 0`, restituisce true immediatamente. Una rete /0 copre l'intero spazio di indirizzamento IPv4, quindi qualsiasi rete interna è contenuta:

```v
if outer.cidr == 0 {
	return true
}
```

Terzo: calcola `shift = 32 - outer.cidr` e confronta `(outer_ip >> shift) == (inner_ip >> shift)`. Questo esegue uno shift a destra su entrambi gli IP per scartare i bit di host, mantenendo solo il prefisso di rete. Se i prefissi corrispondono, inner si trova all'interno di outer.

Esempi concreti:

`10.0.0.0/8` contiene `10.1.2.3/32`? outer.cidr (8) <= inner.cidr (32), supera il primo controllo. Shift = 24. `ip_to_u32("10.0.0.0") >> 24 = 10`. `ip_to_u32("10.1.2.3") >> 24 = 10`. I prefissi di rete corrispondono. Sì.

`10.0.0.0/8` contiene `172.16.0.0/12`? outer.cidr (8) <= inner.cidr (12), supera. Shift = 24. `ip_to_u32("10.0.0.0") >> 24 = 10`. `ip_to_u32("172.16.0.0") >> 24 = 172`. I prefissi differiscono. No.

`192.168.1.0/24` contiene `192.168.0.0/16`? outer.cidr (24) > inner.cidr (16). Fallisce immediatamente il primo controllo. No. Un /24 è più stretto di un /16.

`192.168.0.0/16` contiene `192.168.1.0/24`? outer.cidr (16) <= inner.cidr (24), supera. Shift = 32 - 16 = 16. `ip_to_u32("192.168.0.0") >> 16 = 49320`. `ip_to_u32("192.168.1.0") >> 16 = 49320`. Uguali. Sì. L'intero blocco 192.168.1.0/24 si trova all'interno del blocco 192.168.0.0/16.

`0.0.0.0/0` contiene `192.168.1.0/24`? outer.cidr (0) <= inner.cidr (24), supera il primo controllo. Il controllo iniziale `outer.cidr == 0` restituisce true senza alcun calcolo sui bit. Sì.

Questa funzione viene chiamata dall'helper `addr_is_superset` in `conflict.v` ogni volta che deve determinare se un intervallo di indirizzi ne copre un altro. L'esempio delle regole oscurate dalla panoramica (10.0.0.0/8 oscura 10.0.0.0/24) funziona perché `cidr_contains` identifica correttamente che il /8 contiene il /24.

### PortSpec

Porta iniziale, porta finale facoltativa (predefinita su -1 indicante una porta singola) e flag di negazione. Il metodo `effective_end` normalizza le singole porte restituendo `start` quando `end < 0`. Ciò consente a tutto il codice di confronto delle porte di trattare ogni `PortSpec` come un intervallo senza gestire casi speciali per le singole porte. La funzione `port_range_contains` esegue quindi semplicemente il controllo `outer.start <= inner.start && outer.effective_end() >= inner.effective_end()`.

### MatchCriteria

La struct che memorizza tutto ciò su cui una regola può fare match. Undici campi: protocollo (predefinito su `.all`), sorgente e destinazione (entrambi `?NetworkAddr`), elenchi porte src/dst, interfaccia in/out (entrambe `?string`), stati di connessione, tipo ICMP, rate limit, limit burst e commento. I tipi opzionali per i campi di indirizzo e interfaccia non sono solo una comodità. Codificano la differenza semantica tra "questa regola non filtra sull'indirizzo sorgente" e "questa regola filtra su uno specifico indirizzo sorgente". Il rilevatore di conflitti si basa su questa distinzione: una sorgente `none` significa "corrisponde a tutte le sorgenti", che è un superset di qualsiasi sorgente specifica.

### Rule

Racchiude `MatchCriteria` con i metadati: tabella, nome della catena, tipo di catena, azione, argomenti di destinazione, numero di riga del file sorgente, testo grezzo e formato sorgente. Il campo `line_number` mantiene la posizione originale nel file per la segnalazione degli errori. Il campo `raw_text` memorizza la riga non analizzata per la visualizzazione. Il metodo `str` produce una forma canonica separata da tabulazioni (`chain\tprotocol\tsource\tdest\tports\taction`) che il modulo diff usa per confrontare le regole indipendentemente dal formato.

### Ruleset

Un elenco di regole, una mappa dal nome della catena alla policy predefinita e il formato sorgente. La visibilità `pub mut:` rende `rules` e `policies` mutabili dall'esterno del modulo, operazione necessaria ai parser durante la costruzione incrementale del ruleset. Il metodo `rules_by_chain` raggruppa gli indici delle regole per nome di catena in `map[string][]int`. Sia l'analyzer sia il generator lo invocano per iterare per catena invece di scansionare ripetutamente l'elenco piatto.

---

## Parsing: Funzioni comuni (common.v)

`src/parser/common.v`

Funzioni di parsing condivise usate sia dal parser iptables sia da quello nftables.

**`parse_network_addr`**: Gestisce tre formati di indirizzo: IP semplice (`192.168.1.1` assume come valore predefinito il CIDR /32), notazione CIDR (`10.0.0.0/8`) e formato negato (`!172.16.0.0/12`). Il prefisso di negazione viene rimosso per primo, poi l'indirizzo viene diviso su `/` se presente. Valida che la lunghezza del prefisso sia compresa tra 0 e 128 (il limite superiore accoglie indirizzi IPv6 anche se la matematica CIDR corrente gestisce solo v4).

**`parse_port_spec`**: Stesso pattern degli indirizzi: rimuove `!` per la negazione, divide su `:` per gli intervalli. Gli intervalli di porte usano il separatore a due punti (`1024:65535`), corrispondente alla convenzione di iptables. Le porte singole ottengono `end = -1`. Valida che tutte le porte siano comprese tra 0 e 65535.

**`parse_port_list`**: Suddivide una stringa separata da virgole e chiama `parse_port_spec` su ciascun elemento. Usata quando il parser iptables incontra `--dports 80,443,8080`.

**`parse_protocol`**: Mappa nomi stringa e numeri di protocollo IANA sulle varianti di `Protocol`. Accetta `'tcp'`, `'TCP'` e `'6'` come equivalenti. Le mappature numeriche (6 per TCP, 17 per UDP, 1 per ICMP, 58 per ICMPv6, 132 per SCTP, 47 per GRE) corrispondono alle assegnazioni IANA, che è ciò che gli strumenti a livello kernel talvolta emettono invece dei nomi.

**`parse_action`**: Mappa le stringhe delle azioni sulle varianti di `Action`. Non fa distinzione tra maiuscole e minuscole tramite `.to_upper()`.

**`parse_chain_type`**: Mappa le stringhe con i nomi delle catene sulle varianti di `ChainType`. Tutto ciò che non viene riconosciuto (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING) diventa `.custom`.

**`parse_conn_states`**: Suddivide una stringa di stati separati da virgola e imposta i flag su un bitfield `ConnState`. Inizia con `ConnState.zero()` (tutti i bit azzerati) e chiama `.set()` per ogni nome riconosciuto. I nomi di stato sconosciuti vengono ignorati silenziosamente tramite il ramo `else {}`.

**`detect_format`**: Esamina la prima riga non vuota e non di commento. Le righe che iniziano con `*` (header di tabella come `*filter`) segnalano iptables. Le righe che iniziano con `table` segnalano nftables. Le righe che iniziano con `:` (policy della catena) o `-A`/`-I` (regola) segnalano anch'esse iptables. Se nulla corrisponde, restituisce un errore. Questo rilevamento automatico viene invocato in `load_ruleset` in `main.v` in modo che gli utenti non debbano mai specificare il formato di input.

---

## Parsing: Formato iptables (iptables.v)

`src/parser/iptables.v`

**`tokenize_iptables`**: Un tokenizer byte per byte che gestisce stringhe racchiuse tra virgolette. Tre variabili di stato: `in_quote` traccia se lo scanner si trova all'interno di una regione tra virgolette, `quote_char` ricorda quale carattere di virgolette (`"` o `'`) l'ha aperta e `current` accumula i byte per il token in costruzione.

Lo scanner itera su ciascun byte della riga. In modalità normale, spazi e tabulazioni inviano il token corrente all'elenco di output. I caratteri di virgolette passano alla modalità quoted. In modalità quoted, solo la corrispondente virgoletta di chiusura termina il token; gli spazi vengono accumulati come parte del valore. I caratteri di virgolette stessi vengono rimossi dall'output, quindi `"DROPPED: "` diventa `DROPPED: `. Questo è importante per i prefissi di log e i commenti di iptables che contengono spazi.

Il test del parser in `parser_test.v` valida direttamente la gestione delle stringhe tra virgolette: la tokenizzazione di `-j LOG --log-prefix "DROPPED: "` produce 4 token, dove il quarto è `DROPPED: ` (spazio finale conservato, virgolette rimosse).

**`parse_iptables`**: Iterazione riga per riga con tre tipologie di riga:

- Le righe che iniziano con `*` impostano il contesto della tabella corrente tramite `parse_table`. La variabile `current_table` si propaga a tutte le regole successive fino alla comparsa del successivo header di tabella. È così che `iptables_complex.rules` (che contiene sia la sezione `*filter` sia quella `*nat`) assegna la tabella corretta a ciascuna regola.
- Le righe che iniziano con `:` definiscono le policy delle catene tramite `parse_chain_policy`. `:INPUT DROP [0:0]` diventa la catena `INPUT` con policy `DROP`. I contatori di pacchetti/byte `[0:0]` vengono ignorati.
- Le righe che iniziano con `-A` o `-I` vengono analizzate come regole da `parse_iptables_rule`, passando la tabella corrente e il numero di riga a base 1.
- Righe vuote, commenti (che iniziano con `#`) e righe `COMMIT` vengono saltate.

**`parse_chain_policy`**: Rimuove il `:` iniziale, divide sullo spazio, restituisce il nome della catena e l'azione della policy come tupla. Usa il ritorno multiplo di V: `!(string, Action)`.

**`parse_iptables_rule`**: Il parser principale delle regole. Dopo aver tokenizzato la riga, inizializza variabili mutabili per ogni possibile campo della regola, quindi scorre l'array dei token con un indice mutabile `i` e un'istruzione match che copre ogni flag riconosciuto:

| Flag                          | Cosa imposta                                      |
|-------------------------------|---------------------------------------------------|
| `-A`, `-I`                     | Nome della catena (token successivo)              |
| `-p`, `--protocol`             | Protocollo tramite `parse_protocol`               |
| `-s`, `--source`               | `NetworkAddr` sorgente, con controllo negazione   |
| `-d`, `--destination`          | `NetworkAddr` destinazione, con controllo negaz.  |
| `--sport`, `--source-port`     | Porta sorgente singola, con controllo negazione   |
| `--dport`, `--destination-port`| Porta destinazione singola, con controllo negaz.  |
| `--dports`                     | Elenco multiport tramite `parse_port_list`        |
| `--sports`                     | Elenco multiport sorgente                         |
| `-i`, `--in-interface`         | Nome dell'interfaccia di input                    |
| `-o`, `--out-interface`        | Nome dell'interfaccia di output                   |
| `--state`, `--ctstate`         | Stati di connessione tramite `parse_conn_states`  |
| `--icmp-type`                  | Stringa tipo ICMP                                 |
| `--limit`                      | Stringa rate limit                                |
| `--limit-burst`                | Conteggio del burst                               |
| `--comment`                    | Stringa di commento                               |
| `-j`, `--jump`                 | Azione; i token rimanenti diventano target args   |
| `-m`, `--match`                | Consumato e saltato (il nome estensione non salvato)|

**Gestione della negazione**: Quando `!` compare come token a sé stante, un booleano `next_negated` viene impostato a true. Il successivo indirizzo o porta analizzato controlla questo flag, crea la struct con `negated: true` e reimposta il flag a false. Questo approccio a due fasi evita il lookahead e mantiene il tokenizer completamente all'oscuro della semantica di iptables.

**Multiport**: Il token `--dports` innesca `parse_port_list` da `common.v`, che divide sulle virgole. `-m multiport --dports 80,443` produce due elementi `PortSpec` in `dst_ports`. La parte `-m multiport` viene consumata dal gestore `-m` che avanza semplicemente l'indice oltre il nome dell'estensione. Nota che `--dports` sostituisce l'intero array `dst_ports` anziché fare append, poiché multiport definisce l'elenco completo delle porte.

**Argomenti del target**: Dopo aver analizzato l'azione da `-j`, il parser verifica se i token rimanenti iniziano con `--`. In caso affermativo, raccoglie tutti i token rimanenti in `target_args`. Questo cattura parametri come `--log-prefix "DROPPED: "` e `--to-destination 10.0.0.1:8080`.

---

## Parsing: Formato nftables (nftables.v)

`src/parser/nftables.v`

nftables usa una struttura a blocchi (`table { chain { rule } }`) anziché flag lineari. Il parser traccia l'annidamento attraverso una gerarchia di funzioni.

**`parse_nftables`**: Ciclo di livello superiore. Quando incontra una riga che inizia con `table`, chiama `parse_nft_table` che restituisce sia un valore `Table` sia l'indice della riga successiva da elaborare. Questa restituzione a tupla (`!(Table, int)`) è l'idioma di V per consumare numeri variabili di righe senza la mutazione di un contatore condiviso.

**`parse_nft_table`**: Estrae il nome della tabella dall'header filtrando le parole chiave note delle famiglie (`table`, `inet`, `ip`, `ip6`, `arp`, `bridge`, `netdev`). Il primo token che non sia una di queste parole chiave è il nome della tabella. Questo gestisce tutti i prefissi di famiglia di nftables senza mantenere un elenco esplicito di famiglie valide. Esegue quindi la scansione delle righe fino alla `}` di chiusura, delegando le righe che iniziano con `chain` a `parse_nft_chain`.

**`parse_nft_chain`**: Estrae il nome della catena da righe come `chain input {`, convertendolo in maiuscolo per normalizzarlo alla convenzione di iptables (`input` diventa `INPUT`). Le righe che iniziano con `type` sono metadati della catena (dichiarazione dell'hook con policy). Tutte le altre righe non vuote e non di commento vengono analizzate come regole. Il fallimento del parsing di una regola usa `or { i++; continue }` per saltare le righe non analizzabili senza interrompere l'intero file.

**`extract_nft_policy`**: Analizza `type filter hook input priority 0; policy drop;`. Divide su `;`, trova il segmento che inizia con `policy`, rimuove la parola chiave e analizza il testo rimanente come azione. Restituisce `?Action` in modo che le catene prive di policy esplicite restituiscano `none`.

**`parse_nft_rule`**: Basato su token come il parser iptables, ma confronta parole chiave nftables anziché flag con trattino. I token sono prodotti dividendo la riga sugli spazi, per poi filtrare le stringhe vuote.

| Espressione                   | Cosa imposta                                    |
|-------------------------------|-------------------------------------------------|
| `tcp`, `udp`                  | Protocollo, poi `parse_nft_port_match` per porte|
| `ip saddr X`                  | `NetworkAddr` sorgente                          |
| `ip daddr X`                  | `NetworkAddr` destinazione                     |
| `ip protocol X`               | Protocollo per nome                             |
| `ct state X`                  | Stati di connessione tramite `parse_conn_states` |
| `iifname X`, `iif X`          | Interfaccia di input (virgolette rimosse)       |
| `oifname X`, `oif X`          | Interfaccia di output (virgolette rimosse)      |
| `limit rate X`                | Rate limit (multi-token, raccoglie fino all'azione)|
| `log`                          | Azione LOG con argomento `prefix` facoltativo   |
| `counter`                      | Saltato (il contatore è metadato, non logica di match)|
| `comment X`                    | Stringa commento (virgolette rimosse)           |
| `accept`/`drop`/`reject`/ecc. | Azione terminale                                |

**Parsing del rate limit**: L'espressione `limit rate` in nftables si estende su un numero variabile di token, rendendola una delle sequenze di parsing più delicate. Dopo aver rilevato `limit` e poi `rate`, il parser entra in un ciclo di raccolta che accumula i token in `rate_parts` finché non incontra una parola chiave di azione (`accept`, `drop`, `reject`, `log`) o `counter`. I token raccolti vengono uniti con spazi e memorizzati come stringa di rate. Per una regola come `limit rate 3/minute burst 5 packets accept`, questo produce `limit_rate = "3/minute burst 5 packets"`. L'istruzione `continue` successiva alla memorizzazione garantisce che il ciclo riesamini il token dell'azione nell'iterazione seguente anziché consumarlo come parte della stringa di rate.

**L'azione come parola chiave nuda**: In nftables, le azioni non sono contrassegnate con `-j`. Compaiono come token autonomi alla fine della regola: `accept`, `drop`, `reject`, `masquerade`, `return`, `queue`. Il parser verifica queste presenze e imposta l'azione direttamente. L'azione `log` riceve un trattamento speciale perché può contenere un argomento `prefix` che deve essere catturato in `target_args`. Se alla fine dei token non viene trovata alcuna parola chiave di azione, il parser restituisce un errore:

```v
final_action := action or { return error('no action found in rule: ${line}') }
```

**Sintassi dei set** (la funzione `parse_nft_port_match`): nftables usa `{ 80, 443 }` per i set di porte. Quando il token dopo `dport` è `{`, il parser raccoglie i token fino a `}`, rimuove le virgole con `.replace(',', '')` e analizza ciascuno come specifica di porta. La rimozione delle virgole è necessaria poiché la tokenizzazione basata su spazi lascia le virgole attaccate ai numeri di porta (`"80,"` invece di `"80"`). Le porte singole senza parentesi graffe seguono il percorso più semplice, analizzando direttamente un singolo token.

La funzione `parse_nft_port_match` restituisce l'indice aggiornato in modo che il chiamante possa riprendere l'iterazione nella posizione corretta. Sia `is_dport` che `is_sport` vengono determinati verificando il token nella posizione iniziale. Le porte analizzate vengono aggiunte all'array mutabile appropriato (`dst_ports` o `src_ports`) passato per riferimento.

**Accoppiamento di protocollo e porta**: In nftables, protocollo e porta fanno parte della stessa espressione: `tcp dport 22`. Il parser gestisce questo aspetto impostando il protocollo quando incontra `tcp` o `udp`, chiamando poi immediatamente `parse_nft_port_match` per verificare se il token successivo è `dport` o `sport`. In caso affermativo, la funzione gestisce il parsing delle porte e restituisce il nuovo indice. Se il token successivo non è né `dport` né `sport`, la funzione restituisce l'indice invariato e nessuna porta viene aggiunta. L'istruzione `continue` salta il default `i++` alla fine del ciclo poiché l'indice è già stato fatto avanzare dal port matcher.

---

## Rilevamento dei conflitti (conflict.v)

`src/analyzer/conflict.v`

**`analyze_conflicts`**: Ottiene gli indici raggruppati per catena da `rs.rules_by_chain()`, quindi per ciascuna catena esegue quattro passaggi di rilevamento: `find_duplicates`, `find_shadowed_rules`, `find_contradictions`, `find_redundant_rules`. Raggruppare per catena per prima cosa è fondamentale perché netfilter valuta le regole all'interno di una singola catena in modo sequenziale. Una regola in INPUT non può oscurare una regola in FORWARD poiché elaborano flussi di traffico completamente diversi.

### match_is_superset

La funzione centrale per il rilevamento dei conflitti. Determina se i criteri `outer` corrispondono a un superset stretto del traffico coperto dai criteri `inner`. Ogni dimensione deve superare il test di superset. La funzione restituisce false alla prima dimensione che fallisce, fornendo un'uscita anticipata (early exit).

1. **Protocollo**: Se outer è `.all`, corrisponde a qualsiasi protocollo, quindi è sempre un superset. Se outer specifica un protocollo che differisce da quello di inner, non può essere un superset.

2. **Indirizzo sorgente**: Delega a `addr_is_superset`. La logica:
   - outer è `none` (nessun filtro sorgente) -> superset di qualsiasi cosa, restituisce true
   - outer ha un valore, inner è `none` (inner corrisponde a tutto) -> outer non può coprire "tutto", restituisce false
   - entrambi hanno un valore -> delega a `cidr_contains` da models.v

3. **Indirizzo di destinazione**: Stesso pattern della sorgente.

4. **Porte di destinazione e sorgente**: Delega a `ports_is_superset`. Un elenco outer vuoto significa "corrisponde a tutte le porte", che è un superset di qualsiasi cosa. Un outer non vuoto deve coprire ogni porta presente nell'elenco di inner. Per ciascun intervallo di porte interno, almeno un intervallo di porte esterno deve contenerlo completamente (verificato tramite `port_range_contains`).

5. **Interfacce**: Delega a `iface_is_superset`. Outer `none` è un superset. Se entrambi presenti, devono essere stringhe identiche.

6. **Stati di connessione**: Se outer possiede vincoli di stato (non è vuoto), inner deve possederli tutti. `outer.states.all(inner.states)` esegue un controllo bitwise. Se outer non ha vincoli di stato, il controllo passa comunque.

### matches_overlap

Simile a `match_is_superset` ma bidirezionale. Due criteri si sovrappongono se possono corrispondere allo stesso pacchetto. Per i protocolli, la sovrapposizione richiede che uno dei due sia `.all` oppure che entrambi corrispondano. Per gli indirizzi, `addrs_overlap` controlla se una delle due direzioni di `cidr_contains` è valida (A contiene B o B contiene A). Per le porte, `ports_overlap` verifica se una coppia di intervalli di porte dei due elenchi si interseca tramite `pa.start <= pb.effective_end() && pb.start <= pa.effective_end()`.

### Passaggi di rilevamento

**`find_shadowed_rules`**: Ciclo annidato con `i < j`, cosicché la regola `i` precede sempre la regola `j` nella catena. Un oscuramento (shadow) viene rilevato quando `match_is_superset(rules[i].criteria, rules[j].criteria)` è true E le due regole presentano azioni differenti (`rules[i].action != rules[j].action`):

```v
if match_is_superset(rules[i].criteria, rules[j].criteria)
	&& rules[i].action != rules[j].action {
	findings << Finding{
		severity:     .critical
		title:        'Shadowed rule detected'
		description:  'Rule ${indices[j] + 1} (${rules[j].action.str()}) can never match because rule ${
			indices[i] + 1} (${rules[i].action.str()}) catches all its traffic first'
		rule_indices: [indices[i], indices[j]]
		suggestion:   'Remove rule ${indices[j] + 1} or reorder it before rule ${
			indices[i] + 1}'
	}
}
```

Il confronto delle azioni è cruciale: se entrambe le regole hanno la stessa azione, la regola successiva è ridondante (inutile ma innocua) anziché oscurata (attivamente errata). Gli oscuramenti scattano solo quando le azioni differiscono, poiché ciò significa che il comportamento previsto della regola successiva non potrà mai avere effetto. Dato che netfilter elabora le regole dall'alto verso il basso e si ferma alla prima corrispondenza, la regola successiva è codice morto. Questa è una gravità CRITICAL poiché la regola ha effetto nullo indipendentemente dall'intento.

Scenario concreto: la regola 7 è `DROP tcp/22` (blocca SSH da ovunque) e la regola 8 è `ACCEPT tcp/22 from 10.0.0.0/8` (consenti SSH dall'intervallo 10.x). La regola 7 non ha vincoli di sorgente, quindi `match_is_superset(rule7, rule8)` è true: il protocollo corrisponde (entrambi TCP), `addr_is_superset(none, 10.0.0.0/8)` è true (nessun vincolo è superset di qualsiasi vincolo) e la porta 22 corrisponde alla porta 22. Le azioni differiscono (DROP vs ACCEPT), pertanto questo è uno shadow. La regola 8 non scatterà mai.

**`find_contradictions`**: Sempre con `i < j`. Due regole si contraddicono quando le rispettive corrispondenze si sovrappongono e le loro azioni sono in conflitto (una consente, l'altra nega). La sottigliezza essenziale è che se una regola è un puro superset dell'altra, la funzione la salta. Quel caso viene già segnalato come shadow. Le contraddizioni si applicano unicamente a sovrapposizioni parziali in cui alcuni pacchetti corrispondono a entrambe le regole ma nessuna delle due copre completamente l'altra.

Scenario concreto: la regola 11 è `ACCEPT tcp/443 from 192.168.1.0/24` e la regola 12 è `DROP tcp/443 from 192.168.0.0/16`. I loro indirizzi sorgente si sovrappongono (il /24 è all'interno del /16), le porte corrispondono e le azioni sono in conflitto (ACCEPT vs DROP). Tuttavia la regola 12 non è un puro superset della regola 11 poiché la regola 12 cattura anche sorgenti come 192.168.2.0/24 che la regola 11 non copre. Si tratta dunque di una contraddizione, non di uno shadow.

La funzione `actions_conflict` definisce il conflitto come uno di tipo "allow" (ACCEPT) e uno di tipo "deny" (DROP o REJECT). DROP vs REJECT non è un conflitto (entrambi negano). ACCEPT vs ACCEPT non è un conflitto (entrambi consentono).

**`find_duplicates`**: Usa `criteria_equal` combinato con l'uguaglianza delle azioni. Due regole sono duplicate solo quando ogni campo corrisponde esattamente: stesso protocollo, stessa sorgente, stessa destinazione, stesse porte, stesse interfacce, stessi stati e stessa azione. Questo è il passaggio di rilevamento più semplice.

**`find_redundant_rules`**: Relazione di superset con la medesima azione. Un ampio `ACCEPT tcp` e uno specifico `ACCEPT tcp dport 80` per la stessa catena significano che la regola specifica non produce alcun effetto pratico perché la regola ampia accetta già tutto il TCP, inclusa la porta 80. Questo differisce dagli shadow per due aspetti: le azioni devono coincidere e il finding ha gravità INFO anziché CRITICAL (la regola specifica è innocua, solo ridondante). Il controllo esclude esplicitamente i duplicati esatti (`!criteria_equal(...)`), poiché segnalati da `find_duplicates`.

### Funzioni helper

Il rilevatore di conflitti possiede un layer di funzioni helper che gestiscono l'unwrap dei tipi opzionali e la logica di confronto per ogni tipo di campo. Comprenderle è fondamentale per capire perché i controlli di superset e overlap funzionano correttamente.

**`addr_is_superset`**: Questa funzione dimostra il pattern per i tipi opzionali usato in tutto il modulo conflict. La logica si legge come una tabella di verità:

| outer   | inner   | risultato | motivazione                                        |
|---------|---------|-----------|----------------------------------------------------|
| `none`  | `none`  | true      | nessun filtro è superset di nessun filtro          |
| `none`  | some    | true      | nessun filtro corrisponde a tutto, superset di ogni|
| some    | `none`  | false     | un filtro specifico non può coprire "tutto"        |
| some    | some    | cidr      | delega a `cidr_contains`                           |

Questo pattern si ripete per `iface_is_superset`: outer `none` è superset di qualsiasi cosa, entrambi presenti devono essere stringhe uguali. E per `ports_is_superset`: un elenco outer vuoto è superset di qualsiasi cosa, outer non vuoto deve coprire tutte le porte di inner.

La regola uniforme è: "nessun vincolo" è sempre un superset (corrisponde a tutto), e "un vincolo presente" può essere un superset solo se copre dimostrabilmente l'altro.

**`criteria_equal`**: Controllo di uguaglianza campo per campo. Usa `addrs_equal` per il confronto degli indirizzi: entrambi `none` è uguale, entrambi presenti devono corrispondere in indirizzo, CIDR e flag di negazione, uno `none` e uno presente non è uguale. Usa `ports_equal` per gli elenchi di porte: devono avere la stessa lunghezza e ogni coppia deve corrispondere in start, effective_end e negazione. Usa `opt_str_equal` per stringhe opzionali come i nomi di interfaccia: entrambi `none` è uguale, entrambi presenti devono essere identici, misti non sono uguali.

**`addrs_overlap`**: Se uno dei due indirizzi è `none`, l'overlap è true (una dimensione non filtrata corrisponde a tutto, quindi qualsiasi valore specifico si sovrappone a essa). Se entrambi sono presenti con flag di negazione diversi, l'overlap è cautelativamente assunto true poiché valutare l'intersezione tra un intervallo negato e uno non negato è complesso. Altrimenti, controlla `cidr_contains` in entrambe le direzioni: se A contiene B o B contiene A, si sovrappongono.

**`ports_overlap`**: Se uno dei due elenchi di porte è vuoto, l'overlap è true (nessun filtro di porta indica tutte le porte). Altrimenti, controlla ogni coppia dei due elenchi. Due intervalli di porte si sovrappongono quando `a.start <= b.effective_end() && b.start <= a.effective_end()`. Questo è il test standard di sovrapposizione tra intervalli: due intervalli [a,b] e [c,d] si sovrappongono quando a <= d e c <= b.

---

## Suggerimenti di ottimizzazione (optimizer.v)

`src/analyzer/optimizer.v`

**`suggest_optimizations`**: Raggruppa le regole per catena ed esegue sette controlli per catena (`find_mergeable_ports`, `suggest_reordering`, `find_missing_rate_limits`, `find_missing_conntrack`, `find_unreachable_after_drop`, `find_overly_permissive`, `find_redundant_terminal_drop`), più un controllo globale (`find_missing_logging`).

**`find_mergeable_ports`**: Raggruppa le regole a porta singola tramite una chiave composita formata da `protocol|source|destination|action`. Le regole con chiavi identiche differiscono solo per la porta di destinazione e potrebbero essere accorpate in una sola regola multiport. Ad esempio, tre regole distinte che accettano TCP sulle porte 80, 443 e 8080 dalla stessa sorgente condividerebbero la chiave `tcp|*|*|accept` e potrebbero diventare un'unica regola con `--dports 80,443,8080`.

I gruppi con 2-15 regole vengono segnalati. Il limite superiore di 15 deriva da `config.multiport_max`, che corrisponde al limite dell'estensione multiport di iptables. Il modulo `xt_multiport` del kernel supporta al massimo 15 porte per regola. Gruppi più grandi di 15 non possono usare multiport e vengono saltati.

La funzione prende in considerazione unicamente le regole con esattamente una porta di destinazione (`rule.criteria.dst_ports.len != 1`). Le regole con intervalli di porte o con porte multiple stanno già usando una forma di multi-matching e non sono candidate per un ulteriore accorpamento.

**`find_missing_rate_limits`**: Due passaggi. Primo passaggio: costruisce una mappa di tutte le porte che possiedono già un rate limiting in qualche punto della catena. Ciò previene i falsi positivi quando una regola dedicata di rate-limit coesiste con una normale regola accept per la stessa porta. Secondo passaggio: controlla le regole SSH (porta 22) che eseguono ACCEPT senza rate limiting. SSH è l'unica porta verificata perché rappresenta il bersaglio primario di brute-force sui server esposti a Internet.

**`find_missing_conntrack`**: Cerca una regola con `.has(.established)` tra i suoi stati. Tre possibili esiti: nessuna regola conntrack presente e la catena ha più di 2 regole (WARNING), conntrack presente ma in posizione 3 o successiva nella catena (INFO, perché funziona ma costa cicli non necessari), oppure conntrack si trova nelle prime posizioni (nessun finding). La regola ESTABLISHED/RELATED è l'ottimizzazione di performance a più alto impatto in qualsiasi firewall. Su un server trafficato, la stragrande maggioranza dei pacchetti appartiene a connessioni esistenti. Senza un conntrack anticipato, ognuno di quei pacchetti attraversa l'intera catena.

**`suggest_reordering`**: Salta le prime tre regole di ciascuna catena poiché sono tipicamente regole di loopback, conntrack e invalid-drop che dovrebbero rimanere in cima. Per le regole rimanenti, se una regola corrisponde a una porta ad alto traffico (HTTP 80, HTTPS 443, DNS 53 da `config`) ed è un ACCEPT, suggerisce di spostarla prima. La motivazione: netfilter valuta le regole in modo sequenziale, quindi posizionare le regole sollecitate di frequente vicino alla cima riduce il numero medio di confronti per pacchetto.

**`find_missing_logging`**: Controllo globale, non per singola catena. Per ciascuna catena con una policy predefinita DROP o REJECT, controlla se esiste una regola LOG in quella catena. Senza logging, i pacchetti scartati scompaiono silenziosamente e la diagnosi dei problemi di connettività diventa un tirare a indovinare.

---

## Generazione di configurazioni rafforzate (generator.v)

`src/generator/generator.v`

**`generate_hardened`**: Esegue il dispatch verso `generate_iptables_hardened` o `generate_nftables_hardened` in base al formato richiesto. Entrambi i generator seguono lo stesso ordine di template.

### generate_iptables_hardened

Costruisce un ruleset completo in formato `iptables-save` come array di stringhe unite con ritorni a capo. L'ordine del template non è casuale: riflette la priorità di elaborazione dei pacchetti di un firewall di produzione, dove le prime regole gestiscono il traffico a volume più elevato e le regole successive gestiscono casi progressivamente più rari:

1. **Default deny**: `:INPUT DROP [0:0]`, `:FORWARD DROP [0:0]`, `:OUTPUT ACCEPT [0:0]`. Approccio basato su whitelist: nega tutto, poi consenti solo ciò che è necessario.
2. **Loopback**: Consenti tutto il traffico sull'interfaccia `lo`. Bloccare il loopback compromette la maggior parte delle applicazioni che comunicano internamente.
3. **Conntrack**: Accetta ESTABLISHED/RELATED (pacchetti appartenenti a connessioni esistenti) e scarta INVALID. Posizionato all'inizio per prestazioni.
4. **Anti-spoofing**: Itera su `config.private_ranges` e scarta i pacchetti provenienti da indirizzi RFC 1918 in arrivo sull'interfaccia pubblica. Un pacchetto che dichiara di provenire da `10.0.0.0/8` sulla tua interfaccia `eth0` esposta a Internet è falsificato (spoofed).
5. **ICMP**: Echo-request con rate limit impostato a `config.icmp_rate_limit` (1/second). Echo-reply, destination-unreachable e time-exceeded sono sempre consentiti poiché essenziali per la path MTU discovery e per traceroute.
6. **Servizi**: Regole per servizio regolate da `config.service_ports`. SSH ottiene lo stato conntrack NEW e rate limit impostato a `config.ssh_rate_limit`. Il DNS ottiene sia TCP che UDP poiché i trasferimenti di zona usano TCP. NTP riceve solo UDP. Tutto il resto adotta TCP per impostazione predefinita.
7. **Logging**: LOG con rate limit e prefisso `config.log_prefix_dropped` prima dello scarto finale. Il rate limiting previene l'inondazione dei log durante un DDoS.
8. **DROP finale**: Scarto esplicito come misura di sicurezza aggiuntiva anche se la policy della catena è già DROP. Difesa in profondità (defense in depth).

### generate_nftables_hardened

Stesso template nella sintassi nftables con annidamento dei blocchi `table inet filter { chain input { ... } }`. Differenze sintattiche: `ct state established,related accept` invece di `-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`. `ip saddr 10.0.0.0/8 drop` invece di `-s 10.0.0.0/8 -j DROP`. `limit rate 3/minute burst 5 packets` invece di `-m limit --limit 3/minute --limit-burst 5`. Include inoltre le dichiarazioni `chain forward` e `chain output`.

### Esportazione del formato

**`export_ruleset`**: Esegue il dispatch verso `export_as_iptables` o `export_as_nftables`.

**`export_as_iptables`**: Itera su tutte le regole, tracciando le tabelle incontrate nella mappa `tables_seen`. Quando compare una nuova tabella, la funzione emette un `COMMIT` per la tabella precedente (se presente), quindi `*nometabella`, poi tutte le policy delle catene. Ogni regola viene convertita da `rule_to_iptables` e aggiunta in coda. Un `COMMIT` finale viene emesso al termine di tutte le regole. Questo gestisce ruleset multi-tabella: un ruleset con regole filter e nat produce due blocchi distinti `*filter ... COMMIT` e `*nat ... COMMIT`.

**`export_as_nftables`**: Raggruppa le catene per tabella usando una mappa `table_chains` di tipo `map[string]map[string][]int`. Il primo passaggio itera su tutte le regole e popola questa mappa, raggruppando gli indici delle regole per nome di tabella e nome di catena:

```v
mut table_chains := map[string]map[string][]int{}
for i, rule in rs.rules {
	tbl := rule.table.str()
	if tbl !in table_chains {
		table_chains[tbl] = map[string][]int{}
	}
	table_chains[tbl][rule.chain] << i
}
```

Il secondo passaggio itera sulla mappa `table_chains`. Per ogni tabella emette `table inet X {`. Per ogni catena all'interno di quella tabella, consulta la policy ed emette il blocco della catena. Le catene standard (INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING) ottengono le dichiarazioni degli hook con `type filter hook X priority 0; policy Y;`. Il nome dell'hook deriva dal nome della catena tramite un'istruzione match che mappa `INPUT` su `input`, `OUTPUT` su `output`, ecc. Le catene personalizzate non ricevono dichiarazioni di hook poiché il match restituisce una stringa vuota, e il controllo sulla stringa vuota salta la riga dell'hook. Ogni indice di regola viene quindi convertito tramite `rule_to_nftables` ed emesso all'interno del blocco della catena.

Questo approccio raggruppato per tabella produce un output nftables corretto anche quando il ruleset di partenza ha regole provenienti da tabelle diverse intercalate, poiché la mappa di raggruppamento raccoglie tutte le catene per tabella prima che venga generato qualsiasi output.

### Funzioni di conversione delle regole

**`rule_to_iptables`**: Costruisce una stringa di regola iptables aggiungendo condizionalmente parti a un array `[]string` unito con spazi. L'ordine di emissione segue la convenzione di iptables:

1. Catena: `-A ${r.chain}` (sempre presente)
2. Protocollo: `-p ${r.criteria.protocol.str()}` (saltato quando `.all`)
3. Sorgente: `-s` o `! -s` con `address/cidr` (solo se la sorgente non è `none`)
4. Destinazione: `-d` o `! -d` (solo se la destinazione non è `none`)
5. Interfacce: `-i ${iface}` e/o `-o ${oface}` (solo se presenti)
6. Conntrack: `-m conntrack --ctstate NEW,ESTABLISHED,RELATED,INVALID` costruito controllando ogni bit di flag singolarmente
7. Porte: `--dport X` per porta singola, `-m multiport --dports X,Y,Z` per porte multiple
8. Rate limit: `-m limit --limit X --limit-burst Y` (solo se limit_rate è impostato)
9. Azione: `-j ${r.action.str()}` seguito da eventuali `target_args`

La ricostruzione dello stato del conntrack è l'inverso di `parse_conn_states`: verifica ogni flag con `.has()` e costruisce un array di stringhe con i nomi degli stati, unendoli poi con virgole. Questo garantisce un round-trip perfetto attraverso parse -> model -> emit.

**`rule_to_nftables`**: Stesso pattern di emissione condizionale, con sintassi differente. L'ordine di emissione segue la convenzione di nftables:

1. Interfacce: `iifname "X"` e/o `oifname "X"` con valori tra virgolette
2. Sorgente: `ip saddr [!= ]address/cidr` usando `!=` per la negazione al posto di `!`
3. Destinazione: `ip daddr [!= ]address/cidr`
4. Conntrack: `ct state new,established,related,invalid` (minuscolo, separato da virgole)
5. Protocollo e porte combinati: `tcp dport 443` per porta singola, `tcp dport { 80, 443 }` per la sintassi dei set, `ip protocol tcp` quando il protocollo è specificato ma senza porte
6. Rate limit: `limit rate X` inline
7. Azione: parola chiave minuscola (`accept`, `drop`, `reject`); `log` con argomento facoltativo `prefix` gestito in modo speciale

La differenza strutturale principale rispetto a `rule_to_iptables` risiede nel punto 5: in nftables, protocollo e porta costituiscono una singola espressione. Quando c'è un protocollo ma non ci sono porte, la regola usa `ip protocol tcp`. Quando sono presenti porte, il nome del protocollo fa parte dell'espressione della porta: `tcp dport 80`. Questo viene gestito da un blocco a tre rami `if/else if/else` nella logica di emissione del protocollo.

---

## Modulo Display (display.v)

`src/display/display.v`

Il modulo display è l'unico modulo che dipende dal package `term` di V per l'output a colori. Non tocca mai i dettagli interni di parser, analyzer o generator. Legge unicamente i tipi di `models` e le costanti di `config`.

**`print_banner`**: Disegna un riquadro usando caratteri di disegno box Unicode con `term.bold()` e `term.cyan()`. Mostra il nome dell'applicazione e `config.version`. Il banner usa la funzione `term.dim()` per il testo a contrasto ridotto come il numero di versione e il sottotitolo.

**`print_rule_table`**: Costruisce una tabella a larghezza fissa usando le costanti di colonna di config (`col_num = 5`, `col_chain = 12`, `col_proto = 8`, `col_source = 22`, `col_dest = 22`, `col_ports = 16`, `col_action = 12`). La riga di intestazione concatena chiamate a `pad_right` per ogni etichetta di colonna. Una riga separatrice di trattini si ripete per l'intera larghezza. Ogni regola ottiene una riga con il suo indice a base 1, nome della catena, protocollo, sorgente, destinazione, porte e azione colorata.

La funzione `format_addr` effettua l'unwrap del tipo opzionale: se l'indirizzo è `none` (nessun vincolo di sorgente o destinazione), restituisce `'*'`. Altrimenti chiama `truncate` per adattare l'indirizzo alla larghezza della colonna. La funzione `format_ports` fa lo stesso per gli elenchi di porte, unendo porte multiple con virgole prima della troncatura. La funzione `colorize_action` mappa ACCEPT su `term.green()`, DROP e REJECT su `term.red()` e LOG su `term.yellow()`. Le altre azioni non vengono colorate.

**`print_finding`**: Esegue il rendering di un singolo finding. Il badge di gravità è racchiuso tra parentesi quadre e colorato tramite `colorize_severity`: `[CRITICAL]` in rosso grassetto, `[WARNING]` in giallo, `[INFO]` in ciano. Il titolo è in grassetto. I numeri delle regole interessate vengono convertiti da base 0 (rappresentazione interna) a base 1 (visibile all'utente) tramite `.map('${it + 1}')`. La riga di suggerimento usa `config.sym_arrow` (una freccia destra Unicode) come prefisso e `term.green()` per il testo, rendendo i consigli operativi visivamente distinti dal testo descrittivo.

**`print_findings`**: Conta prima i finding per categoria di gravità (critical, warning, info), poi stampa un header di riepilogo con conteggi codificati a colori, quindi itera su ciascun finding e lo stampa. Se non esistono finding, stampa `config.sym_check` (un segno di spunta Unicode) in verde con "No issues found".

**`print_summary`**: Mostra il formato del ruleset, il conteggio totale delle regole, il conteggio delle catene e il dettaglio per ciascuna catena. Ogni voce di catena mostra il conteggio delle sue regole e la policy predefinita colorata. La ricerca della policy usa `if p := rs.policies[chain_name]` per gestire le catene prive di policy esplicite (che ricevono un trattino attenuato).

**`print_diff`**: Confronta due ruleset convertendo ciascuno in un insieme di stringhe canoniche. La funzione `build_rule_set` itera su tutte le regole e crea una `map[string]bool` usando `Rule.str()` come chiave. La forma canonica è separata da tabulazioni `chain\tprotocol\tsource\tdest\tports\taction`, che normalizza sia le regole iptables sia quelle nftables alla stessa rappresentazione. Le differenze insiemistiche vengono calcolate iterando su ciascuna mappa e controllando l'appartenenza nell'altra. Le regole presenti solo a sinistra ricevono un prefisso `-` rosso. Le regole presenti solo a destra ricevono un prefisso `+` verde. I ruleset identici stampano "Rulesets are equivalent" con un segno di spunta verde.

**Funzioni di utilità**: `pad_right` aggiunge spazi a destra di una stringa fino a una larghezza target. Se la stringa è già pari o superiore alla larghezza target, viene restituita invariata. `truncate` accorcia le stringhe che superano una lunghezza massima e aggiunge `...` come indicatore di omissione. Per lunghezze massime pari o inferiori a 3, salta i puntini di sospensione e tronca semplicemente, poiché non vi sarebbe spazio sia per il contenuto sia per l'indicatore di omissione.

---

## Test

Convenzioni di test in V: i file terminano con `_test.v`, le funzioni hanno il prefisso `test_` e le asserzioni usano la parola chiave `assert`. Non è necessario alcun framework di test aggiuntivo.

**Accesso interno al modulo.** I file di test si dichiarano nello stesso modulo del codice che testano. `src/parser/parser_test.v` usa `module parser`, ottenendo accesso a funzioni private come `tokenize_iptables`. `src/analyzer/analyzer_test.v` usa `module analyzer` per chiamare direttamente `find_shadowed_rules`, `match_is_superset`, `ports_overlap` e altre funzioni non esportate. Questo è lo stesso pattern dei file `_test.go` nello stesso package di Go.

**File di testdata.** I test usano `@VMODROOT`, una costante a tempo di compilazione che risolve nella directory contenente `v.mod` (la root del progetto). Ad esempio, `os.read_file(@VMODROOT + '/testdata/iptables_basic.rules')` carica i file fixture con un percorso che funziona indipendentemente dalla directory di lavoro corrente.

**Esecuzione dei test.** `v test src/` dalla root del progetto rileva ed esegue tutti i file `_test.v` in parallelo. Il Justfile racchiude questo comando come `just test`. L'output dei singoli test con cronometraggio usa `just test-verbose`, che passa il flag `-stats`. I test sono progettati per essere veloci: nessuna dipendenza esterna, nessuna chiamata di rete, nessuna creazione di file. Tutto legge dalla directory testdata o costruisce struct inline.

**Isolamento dei test.** Ciascuna funzione di test costruisce i propri dati ed esegue le asserzioni. Non vi è alcuno stato condiviso tra i test, nessun setup/teardown e nessuna dipendenza dall'ordine di esecuzione. Un test come `test_find_shadowed_rule` in `analyzer_test.v` crea due struct `Rule` con criteri specifici, chiama direttamente la funzione privata `find_shadowed_rules` e asserisce che il risultato abbia la gravità e il titolo attesi.

**Test del parser** (`src/parser/parser_test.v`): Organizzati in tre livelli.

Il primo livello è costituito dai test unitari per ciascuna funzione di parsing condivisa. `parse_network_addr` viene testata con IP semplice (asserisce `cidr == 32`), CIDR /8, CIDR /24 e negato (asserisce `negated == true`). `parse_port_spec` copre porta singola, intervallo (asserisce `start == 1024` e `end == 65535`) e negato. `parse_port_list` testa il parsing con separazione da virgola con e senza spazi. `parse_protocol` verifica il parsing dei nomi (tcp, udp, icmp), il parsing dei numeri IANA (`'6'` -> tcp, `'17'` -> udp) e la tolleranza al maiuscolo/minuscolo (`'TCP'` -> tcp). `parse_action` copre ACCEPT, DROP, REJECT, LOG, MASQUERADE. `parse_table` copre filter, nat, mangle. `parse_chain_type` verifica i tipi standard e il fallback personalizzato. `parse_conn_states` verifica stato singolo, stati multipli, tutti e quattro gli stati e input non sensibile a maiuscole/minuscole.

Il secondo livello riguarda il rilevamento del formato e i test del tokenizer. `detect_format` viene testata con header di tabelle iptables, policy di catene, righe di regole, blocchi nftables e input che iniziano con commenti (verificando che il commento venga saltato). Il tokenizer riceve tre test: suddivisione di base (8 token da una regola semplice), stringhe con doppie virgolette contenenti spazi (verifica che `"DROPPED: "` diventi `DROPPED: ` con spazio conservato e virgolette rimosse) e stringhe con virgolette singole (verifica che `'my rule'` diventi `my rule`).

Il terzo livello è costituito dai test di integrazione che caricano file di testdata reali. `test_parse_iptables_basic_rule_count` carica `iptables_basic.rules` e asserisce 9 regole. `test_parse_iptables_basic_policies` controlla che INPUT sia DROP, FORWARD sia DROP, OUTPUT sia ACCEPT. `test_parse_iptables_basic_ssh_rule` verifica che la regola SSH abbia protocollo TCP, una porta di destinazione 22 e azione ACCEPT. `test_parse_iptables_basic_conntrack` controlla che la regola conntrack abbia entrambi gli stati `.established` e `.related` impostati. `test_parse_nftables_basic_rule_count` carica `nftables_basic.rules` e asserisce 8 regole. I test sui file complessi verificano le regole multiport da `iptables_complex.rules`, l'estrazione dei rate limit e le regole MASQUERADE della tabella NAT.

**Test dell'analyzer** (`src/analyzer/analyzer_test.v`): Il file di test più completo, articolato in test di rilevamento, test delle funzioni helper e test dell'optimizer.

I test di rilevamento usano struct `Rule` costruite manualmente per verificare ciascun passaggio in isolamento. `test_find_shadowed_rule` crea una regola TCP ampia (nessun filtro porta) e una regola TCP/porta-80 stretta, chiama direttamente `find_shadowed_rules` e asserisce un finding CRITICAL con "Shadowed" nel titolo. `test_find_contradiction` crea due regole con criteri sovrapposti (TCP porta 80 da 192.168.1.0/24 con ACCEPT vs TCP porta 80 verso 10.0.0.0/8 con DROP), asserisce un finding WARNING. `test_find_duplicate` passa la stessa regola due volte, asserisce un WARNING. `test_find_redundant` usa un ampio TCP/ACCEPT e uno stretto TCP-porta-80/ACCEPT, asserisce un finding INFO. `test_no_false_positives_disjoint_rules` crea regole TCP/22 e UDP/53, esegue `analyze_conflicts` su un `Ruleset` completo e asserisce nessun finding CRITICAL. Questo test è importante perché valida che regole su protocolli differenti non vengano contrassegnate erroneamente.

I test delle funzioni helper verificano i mattoni fondamentali. `matches_overlap` viene testata con: stesso protocollo e porta (si sovrappongono), protocolli diversi (non si sovrappongono), protocollo `.all` (si sovrappone a qualsiasi cosa), porte non sovrapposte (tcp/80 vs tcp/443), elenco porte vuoto che corrisponde a tutto. `match_is_superset` viene testata con: criteri più ampi che coprono criteri più stretti, criteri più stretti che non coprono quelli più ampi, protocollo `.all` come superset di uno specifico, contenimento CIDR tramite 10.0.0.0/8 che contiene 10.1.2.0/24, non contenimento CIDR tramite 10.0.0.0/24 che non contiene 172.16.0.0/24.

I test di `criteria_equal`: criteri identici, porte diverse, protocollo diverso, indirizzi corrispondenti, indirizzo `none` vs `some`. I test di `actions_conflict`: accept/drop (true), accept/reject (true), drop/accept (true), accept/accept (false), drop/drop (false), drop/reject (false). Helper per le porte: `ports_overlap` per stessa porta, porte diverse, intervallo che contiene una porta singola, elenchi vuoti. `ports_is_superset` per outer vuoto (superset), inner vuoto (non superset), intervallo che copre la porta singola, porta singola che non copre l'altra. Helper per gli indirizzi: `addr_is_superset` con CIDR più ampio, outer `none` (sempre superset), inner `none` (non superset), entrambi `none` (true).

Test dell'optimizer: `test_find_mergeable_ports` crea tre regole TCP sulle porte 80, 443, 8080 con stesso protocollo/sorgente/destinazione/azione, asserisce un finding che suggerisce l'accorpamento. `test_find_missing_rate_limits_ssh` crea una regola TCP/22 ACCEPT senza rate limiting, asserisce un finding WARNING. `test_find_missing_conntrack_empty` passa input vuoti, asserisce nessun finding (guardia per casi limite). I test `test_opt_str_equal_*` verificano l'helper di uguaglianza per stringhe opzionali con i casi entrambi-none, stesso-valore, valore-diverso e uno-none.

**Test del generator** (`src/generator/generator_test.v`): Testa il template rafforzato in entrambi i formati. La strategia di test per i generator è il contenimento di stringhe: genera l'output e asserisce che compaiano specifiche sottostringhe. Questo verifica che siano presenti le regole corrette senza essere fragile rispetto a spazi bianchi esatti o all'ordine interno alle sezioni.

I test di iptables hardened verificano: policy default deny (cerca `:INPUT DROP` e `:FORWARD DROP`), regole di loopback (cerca `-A INPUT -i lo -j ACCEPT`), conntrack (cerca `--ctstate ESTABLISHED,RELATED -j ACCEPT` e `--ctstate INVALID -j DROP`), SSH con rate limit (cerca la compresenza di `--dport 22`, `--limit 3/minute` e `--limit-burst 5`), porte HTTP/HTTPS, anti-spoofing per tutti e tre gli intervalli RFC 1918 (controlla individualmente `-s 10.0.0.0/8`, `-s 172.16.0.0/12`, `-s 192.168.0.0/16`), rate limiting ICMP, prefisso di log, DROP finale come ultima regola INPUT (scansiona l'ultima riga `-A INPUT` e asserisce che sia uguale a `-A INPUT -j DROP`), marcatori COMMIT, doppio protocollo per DNS (controlla sia `-p tcp --dport 53` sia `-p udp --dport 53`), solo UDP per NTP (controlla `-p udp --dport 123`) e interfaccia personalizzata (passa `'ens192'` e cerca `-i ens192`).

I test di nftables hardened verificano: struttura table/chain, sintassi conntrack (cerca `ct state established,related accept`), rate limit SSH, anti-spoofing (cerca `ip saddr 10.0.0.0/8 drop`), loopback (cerca `iifname "lo" accept`), doppio protocollo per DNS.

Test unitari di `rule_to_iptables`: Ciascun test costruisce una struct `Rule` con criteri specifici e asserisce che la stringa di output contenga i flag attesi. Il test porta TCP controlla `-A INPUT`, `-p tcp`, `--dport 80`, `-j ACCEPT`. Il test sorgente negata controlla `! -s 10.0.0.0/8`. Il test multiport controlla `-m multiport --dports`.

Test unitari di `rule_to_nftables`: Stesso pattern con la sintassi nftables. La sorgente negata verifica `!= 10.0.0.0/8` (sintassi di negazione nftables). Il set multiport controlla `tcp dport {` con entrambi i numeri di porta presenti. Log con prefisso verifica che `log prefix "DROPPED: "` compaia come espressione singola.

Test di integrazione dell'esportazione: `test_export_ruleset_iptables` crea un `Ruleset` minimale con una regola e una policy, lo esporta e verifica che l'output contenga `*filter`, `:INPUT DROP`, `-A INPUT` e `COMMIT`. `test_export_ruleset_nftables` fa lo stesso e controlla `table inet filter`, `chain input`, `tcp dport 80`. `test_export_empty_ruleset` verifica che l'esportazione di un ruleset vuoto non vada in crash.