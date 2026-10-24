# DNS Lookup Tool - Panoramica del progetto

## Cosa fa questo progetto

Questo è un tool professionale di ricognizione DNS realizzato come applicazione a riga di comando. Esegue query DNS, reverse lookup, tracciamento della risoluzione e raccolta di informazioni WHOIS. Il tool è pensato per l'analisi di rete, la ricerca di sicurezza e per imparare come funziona l'infrastruttura DNS a livello tecnico.

A differenza dei semplici wrapper di `dig` o `nslookup`, questo progetto implementa query DNS async concorrenti, il tracciamento del percorso di risoluzione a partire dai root server e una formattazione dell'output strutturata, adatta sia all'analisi umana sia all'elaborazione automatizzata.

## Funzionalità principali

**Query sui record DNS** (`dnslookup/cli.py:112-167`)
- Interroga più tipi di record contemporaneamente (A, AAAA, MX, NS, TXT, CNAME, SOA)
- Selezione di un server DNS personalizzato
- Timeout configurabili
- Output JSON per il parsing

**Reverse DNS lookup** (`dnslookup/cli.py:170-216`)
- Risoluzione dei record PTR IPv4 e IPv6
- Utile per identificare la proprietà dei server e rilevare pattern di hosting

**DNS Trace** (`dnslookup/cli.py:219-263`)
- Traccia il percorso completo di risoluzione dai root server, attraverso il TLD, fino ai nameserver autoritativi
- Visualizza la gerarchia di delega DNS
- Mostra quali server vengono interrogati a ogni passo

**Operazioni batch** (`dnslookup/cli.py:266-350`)
- Elabora centinaia di domini da un file
- Query async concorrenti per la velocità
- Esportazione dei risultati in JSON

**WHOIS lookup** (`dnslookup/cli.py:353-393`)
- Dettagli di registrazione del dominio
- Informazioni sul registrar, date di creazione, date di scadenza
- Informazioni sui name server

## Perché è importante per la sicurezza

Il DNS è una superficie d'attacco fondamentale. Questo tool insegna:

1. **Tecniche di ricognizione**: come gli attaccanti enumerano l'infrastruttura
2. **Architettura DNS**: capire la delega rende più chiari spoofing e hijacking
3. **Information leakage**: quali dati il DNS espone sulla tua infrastruttura
4. **Rilevamento degli attacchi**: riconoscere pattern DNS insoliti

Incidenti reali a cui si applicano queste conoscenze:
- **Attacco DDoS a Dyn (2016)**: un'enorme interruzione dell'infrastruttura DNS ha colpito Twitter, Netflix, Reddit
- **Campagna Sea Turtle (2019)**: DNS hijacking di matrice statale contro agenzie governative (MITRE ATT&CK: T1584.002)
- **DNSpionage (2018)**: DNS hijacking per la raccolta di credenziali

## Architettura tecnica
```
User Command
    ↓
cli.py (Typer interface)
    ↓
resolver.py (dnspython async wrapper)
    ↓
DNS Protocol Operations
    ↓
output.py (Rich formatting)
    ↓
Terminal Display
```

L'architettura separa le responsabilità in modo pulito:
- **Livello CLI**: interazione con l'utente, parsing degli argomenti (`cli.py`)
- **Livello di risoluzione**: operazioni del protocollo DNS (`resolver.py`)
- **Livello di presentazione**: formattazione dell'output (`output.py`)

## Percorso di apprendimento

Questo progetto insegna:
1. **Meccanica del protocollo DNS**: come funzionano realmente le query a livello di pacchetto
2. **Python async**: usare `asyncio` per operazioni di rete concorrenti
3. **Design di CLI**: costruire tool professionali a riga di comando con Typer
4. **Gestione degli errori**: timeout di rete, risposte NXDOMAIN e SERVFAIL
5. **Strutture dati**: modellare i record DNS in modo pulito
6. **Mentalità di sicurezza**: pensare sia da difensore sia da attaccante

## Esempi di avvio rapido
```bash
# Basic query - all record types
dnslookup query example.com

# Specific records with custom DNS server
dnslookup query example.com --type A,MX --server 8.8.8.8

# Trace resolution path (shows DNS hierarchy)
dnslookup trace example.com

# Reverse lookup to find hostname
dnslookup reverse 8.8.8.8

# Batch reconnaissance
echo "example.com" > domains.txt
echo "example.org" >> domains.txt
dnslookup batch domains.txt --output results.json
```

## File chiave da comprendere

| File | Scopo | Righe di codice |
|------|---------|---------------|
| `resolver.py` | Logica DNS principale | ~400 |
| `cli.py` | Interfaccia dei comandi | ~400 |
| `output.py` | Formattazione per il terminale | ~400 |
| `whois_lookup.py` | Operazioni WHOIS | ~200 |

## Funzionalità di sicurezza

- **Nessuna cache**: ogni query è nuova (evita dati obsoleti)
- **Supporto per nameserver personalizzati**: testa su server DNS specifici
- **Controllo dei timeout**: evita blocchi su server che non rispondono
- **Trasparenza degli errori**: mostra esattamente cosa è fallito e perché

## Su cosa costruirai dopo

Dopo aver padroneggiato questo progetto, sarai pronto per:
- Sistemi di rilevamento del DNS tunneling
- Server DNS personalizzati con monitoraggio di sicurezza
- Tool automatizzati di enumerazione dei sottodomini
- Raccolta di threat intelligence basata sul DNS

## Applicazioni nel mondo reale

Esattamente questa funzionalità viene usata in:
- **Penetration testing**: fase iniziale di ricognizione
- **Incident response**: indagine su domini sospetti
- **Threat hunting**: tracciamento dell'infrastruttura C2
- **Monitoraggio dell'infrastruttura**: validazione delle configurazioni DNS
- **Ricerca di sicurezza**: analisi dei pattern DNS

## Vettori d'attacco che questo tool aiuta a comprendere

1. **Ricognizione DNS** (MITRE T1590.002): la raccolta di informazioni che gli attaccanti eseguono
2. **DNS tunneling**: esfiltrazione di dati attraverso query DNS
3. **DNS cache poisoning**: come risposte contraffatte potrebbero reindirizzare il traffico
4. **Enumerazione dei sottodomini**: trovare infrastruttura nascosta
5. **DNS amplification**: come il DNS può essere trasformato in un'arma per attacchi DDoS

Ora approfondisci `01-CONCEPTS.md` per capire i fondamenti del protocollo DNS su cui si basa questo tool.
