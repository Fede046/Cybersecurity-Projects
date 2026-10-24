<!-- © AngelaMos | 2026 | 00-OVERVIEW.md -->

# Firewall Rule Engine

## Cos'è

Uno strumento CLI chiamato `fwrule` che fa il parsing di ruleset iptables e nftables, rileva conflitti tra regole (shadowing, contraddizioni, duplicati), suggerisce ottimizzazioni di performance e di sicurezza, e genera ruleset hardened da zero. Legge file di ruleset grezzi, esegue un'analisi a coppie su ogni regola in ciascuna chain, e produce in output i riscontri con valutazioni di severità e suggerimenti di correzione. Scritto in V.

## Perché è importante

Le misconfigurazioni dei firewall sono dietro a una percentuale significativa delle violazioni cloud, e il problema è quasi sempre lo stesso: qualcuno ha scritto regole che sembrano corrette sulla carta ma interagiscono in modi che non si aspettava.

La violazione di Capital One nel 2019 (CVE-2019-5418) è avvenuta perché un WAF era mal configurato, permettendo a un attaccante di eseguire SSRF contro il servizio di metadata EC2 ed esfiltrare 100 milioni di record di clienti da S3. Le regole del firewall erano troppo permissive sul ruolo del WAF, e nessuno se ne accorse. L'azienda pagò 80 milioni di dollari in multe e 190 milioni di dollari in accordi.

AWS pubblica che le misconfigurazioni dei security group sono una delle principali cause di esposizione dei bucket S3. La violazione di Imperva nel 2019 risaliva a una API key di AWS esposta attraverso un'istanza interna mal configurata che avrebbe dovuto essere isolata da un firewall. La National Security Agency ha pubblicato un advisory di cybersecurity (U/OO/179891-20) specificamente sulle regole di firewall IPsec VPN mal configurate che permettono il movimento laterale dell'avversario.

Tre scenari concreti in cui questo strumento si applica:

1. **Regole shadowed**: Aggiungi `-A INPUT -p tcp --dport 80 -j ACCEPT` presto nella tua chain, poi più avanti aggiungi una regola più specifica per bloccare una subnet malevola nota sulla porta 80. La regola specifica non scatta mai perché l'ampio ACCEPT cattura tutto per primo. Questa è la singola misconfigurazione di firewall più comune.

2. **Connection tracking mancante**: Un ruleset permette SSH sulla porta 22 ma non ha alcuna regola `ESTABLISHED,RELATED` vicino alla cima. Ogni pacchetto in ogni sessione TCP esistente deve attraversare l'intera chain invece di matchare immediatamente. Su un server affollato, questo significa migliaia di valutazioni di regole non necessarie al secondo.

3. **Nessun rate limiting su SSH**: La porta 22 è aperta con un semplice ACCEPT. Un attaccante lancia hydra o medusa contro di essa con migliaia di tentativi di password al minuto. Un rate limit di 3/minuto con burst 5 avrebbe reso il brute force impraticabile.

## Cosa imparerai

**Concetti di sicurezza:**
- Come netfilter elabora i pacchetti attraverso tabelle e chain (filter, nat, mangle, raw)
- La differenza tra iptables (lo strumento userspace legacy) e nftables (il suo rimpiazzo dal Linux 3.13)
- L'ordine di valutazione delle regole e perché la posizione nella chain determina il comportamento
- Il connection tracking (conntrack) e il firewalling stateful con gli stati NEW, ESTABLISHED, RELATED, INVALID
- Le policy di default-deny contro default-accept, e perché la distinzione conta

**Competenze tecniche:**
- Il linguaggio di programmazione V (sintassi, moduli, option type, tagged union, flag enum)
- Il parsing di linguaggi domain-specific (tokenizzazione dei flag iptables, parsing della struttura a blocchi nftables)
- Rilevamento di conflitti a coppie: controllare ogni coppia di regole per relazioni di superset/subset su IP, porte e protocolli
- Matematica CIDR: convertire indirizzi IP in interi a 32 bit e confrontare i prefissi di rete con bit shift

**Strumenti:**
- `v fmt` per la formattazione del codice (il formatter integrato di V, simile a gofmt)
- `v test` per eseguire la test suite
- Il command runner `just` per le recipe di build, test, formattazione e smoke test

## Prerequisiti

### Richiesti

- Conoscenze di base di networking: TCP/IP, cosa sono le porte, cosa fa un firewall
- Familiarità con la riga di comando: navigare tra le directory, eseguire comandi, leggere l'output del terminale
- Esperienza con qualsiasi linguaggio di programmazione: se sai leggere C, Go o Python, V avrà senso immediatamente

### Strumenti

- **V 0.5+** (lo script di installazione lo gestisce automaticamente se non ce l'hai)
- Il command runner **just** (opzionale ma rende tutto più facile)

### Utili ma non richiesti

- Esperienza di amministrazione di sistemi Linux
- Lavoro pratico con iptables o nftables
- Familiarità con la notazione CIDR e le subnet mask

## Avvio rapido

```bash
git clone https://github.com/CarterPerez-dev/Cybersecurity-Projects.git
cd PROJECTS/beginner/firewall-rule-engine

./install.sh

fwrule analyze testdata/iptables_conflicts.rules
```

Il comando `analyze` fa il parsing del ruleset, rileva che la regola 8 (ACCEPT tcp/22 da 10.0.0.0/8) è shadowed dalla regola 7 (ACCEPT tcp/22 da ovunque), trova che le regole 9 e 10 sono duplicati (entrambe ACCEPT tcp/80), e segnala la contraddizione tra la regola 11 (ACCEPT tcp/443 da 192.168.1.0/24) e la regola 12 (DROP tcp/443 da 192.168.0.0/16 che contiene 192.168.1.0/24).

Prova qualche altro comando:

```bash
fwrule load testdata/nftables_basic.rules

fwrule harden -s ssh,http,https -f nftables

fwrule export testdata/iptables_basic.rules -f nftables

fwrule diff testdata/iptables_basic.rules testdata/nftables_basic.rules
```

## Struttura del progetto

```
firewall-rule-engine/
├── src/
│   ├── main.v                  CLI entry point, subcommand dispatch
│   ├── config/
│   │   └── config.v            Constants: ports, CIDR ranges, rate limits, service map
│   ├── models/
│   │   └── models.v            Core types: Rule, Ruleset, Finding, NetworkAddr, PortSpec
│   ├── parser/
│   │   ├── common.v            Shared parsing: protocols, actions, CIDR, port specs
│   │   ├── iptables.v          iptables-save format tokenizer and rule parser
│   │   ├── nftables.v          nftables block-structured format parser
│   │   └── parser_test.v       Parser test suite
│   ├── analyzer/
│   │   ├── conflict.v          Pairwise analysis: shadows, contradictions, duplicates
│   │   ├── optimizer.v         Optimization: port merging, reordering, missing conntrack
│   │   └── analyzer_test.v     Analyzer test suite
│   ├── generator/
│   │   ├── generator.v         Hardened ruleset generation, format conversion
│   │   └── generator_test.v    Generator test suite
│   └── display/
│       └── display.v           Terminal output: tables, colored findings, diffs
├── testdata/
│   ├── iptables_basic.rules    Clean iptables ruleset
│   ├── iptables_complex.rules  Larger iptables ruleset with NAT and multiple tables
│   ├── iptables_conflicts.rules Intentionally broken rules for testing conflict detection
│   ├── nftables_basic.rules    Clean nftables ruleset
│   ├── nftables_complex.rules  Larger nftables ruleset
│   └── nftables_conflicts.rules Intentionally broken nftables rules
├── learn/                      You are here
├── install.sh                  One-command setup (installs V if needed, builds, installs)
├── Justfile                    Build/test/format/smoke-test recipes
├── v.mod                       V module metadata
└── LICENSE
```

## Prossimi passi

1. [01-CONCEPTS.md](./01-CONCEPTS.it.md) - Architettura di netfilter, iptables contro nftables, valutazione delle chain, connection tracking
2. [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) - Come i moduli parser, analyzer e generator interagiscono, flusso dei dati dal testo grezzo ai riscontri
3. [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) - Walkthrough del codice: tokenizzazione, matematica CIDR, rilevamento di conflitti a coppie, generazione di ruleset hardened
4. [04-CHALLENGES.md](./04-CHALLENGES.it.md) - Estensioni: supporto IPv6, import da sistema live, parsing di firewalld/ufw, visualizzazione delle regole

## Problemi comuni

**`v: command not found` dopo il clone**

Esegui `./install.sh`. Clona il compilatore V dal sorgente, lo compila, e lo aggiunge al tuo PATH. Se l'hai già eseguito e ottieni ancora l'errore, riavvia la shell o esegui `source ~/.zshrc` (o `~/.bashrc`).

**I test falliscono con errori di import**

Esegui i test dalla root del progetto, non da dentro `src/`:
```bash
v test src/
```
V risolve gli import dei moduli relativamente alla root del progetto. Eseguire `v test` da dentro una sottodirectory rompe i path di import.

**`v fmt` segnala errori di formattazione**

Esegui il formatter in modalità scrittura per correggerli automaticamente:
```bash
v fmt -w src/
```
Oppure usa `just fmt` che fa la stessa cosa. Per controllare senza modificare i file, usa `just fmt-check`.

**Binario non trovato dopo l'installazione**

Lo script di installazione mette il binario in `~/.local/bin/fwrule`. Se quella directory non è nel tuo PATH, aggiungila oppure esegui il binario direttamente:
```bash
~/.local/bin/fwrule analyze testdata/iptables_conflicts.rules
```
