# Hive Honeypot Network

## Che cos'è

Una rete di honeypot multi-protocollo che simula sei servizi reali (SSH, HTTP, FTP, SMB, MySQL, Redis), cattura ogni interazione dell'attaccante, mappa il comportamento sulle tecniche MITRE ATT&CK, estrae threat intelligence (IOC) e visualizza tutto tramite una dashboard web in tempo reale. Costruita interamente in Go con un frontend React.

## Perché è importante

Gli honeypot sono uno dei modi più efficaci per studiare il comportamento reale degli attaccanti. Ogni connessione a un honeypot è sospetta per definizione, dato che gli utenti legittimi non hanno alcun motivo di interagire con servizi finti. I team di sicurezza di aziende come Deutsche Telekom, Rapid7 e SANS usano reti di honeypot per rilevare host compromessi, raccogliere campioni di malware e tracciare campagne di attacco.

I dati del 2023 dell'honeypot SSH Cowrie di SANS hanno mostrato oltre 2,3 milioni di tentativi di brute force al mese contro un singolo sensore. Le organizzazioni che gestiscono honeypot scoprono regolarmente host interni compromessi che tentano movimenti laterali, intercettando ciò che sfugge ai log del firewall.

**Scenari reali in cui si applica:**
- Un team SOC distribuisce honeypot interni accanto ai server di produzione. Quando un attaccante si muove lateralmente dopo la compromissione iniziale, colpisce l'honeypot e fa scattare un alert immediato, intercettando ciò che l'EDR non ha rilevato
- Un team di threat intelligence gestisce honeypot esposti pubblicamente per tracciare le campagne delle botnet, raccogliendo IoC (indirizzi IP, hash di malware, liste di credenziali) che alimentano il loro SIEM e le blocklist
- Un incident responder usa i replay delle sessioni dell'honeypot per capire esattamente quali comandi ha eseguito un attaccante, quali tool ha scaricato e quali meccanismi di persistenza ha installato

## Cosa imparerai

Questo progetto ti insegna come funzionano gli honeypot a livello di protocollo. Implementando ogni servizio da zero, comprendi sia la prospettiva dell'attaccante sia gli strumenti del difensore.

**Concetti di sicurezza:**
- Emulazione dei protocolli a diversi livelli di profondità: quanto di SSH, HTTP, FTP, SMB, MySQL e Redis devi implementare per ingannare i tool automatici rispetto agli attaccanti umani
- Rilevamento delle tecniche MITRE ATT&CK: mappare gli eventi grezzi (tentativi di brute force, comandi di discovery, trasferimenti di tool) sugli ID delle tecniche del framework
- Estrazione degli Indicator of Compromise: ricavare indirizzi IP, URL, domini, fingerprint dei tool e coppie di credenziali dai dati degli eventi a livello di protocollo

**Competenze tecniche:**
- Costruire server TCP che parlano veri protocolli di rete (handshake SSH, pacchetti di greeting MySQL, comandi RESP, negotiate SMB)
- Architettura event-driven con pub/sub fan-out, worker pool e channel non bloccanti in Go
- Sviluppo di applicazioni full-stack con backend Go, persistenza PostgreSQL, streaming Redis e dashboard React

**Strumenti e tecniche:**
- golang.org/x/crypto/ssh per costruire un server SSH con shell finta, filesystem e registrazione delle sessioni
- Lo standard STIX 2.1 per la condivisione della threat intelligence, generando bundle compatibili con MISP e OpenCTI
- Il formato Asciicast v2 per la registrazione delle sessioni di terminale, riproducibili nel browser tramite xterm.js

## Prerequisiti

**Conoscenze richieste:**
- Fondamenti di Go: goroutine, channel, interfacce, gestione degli errori. Dovresti sentirti a tuo agio a scrivere programmi Go concorrenti
- Networking di base: TCP/IP, come i client si connettono ai server, com'è fatto un handshake di protocollo
- Basi di SQL: CREATE TABLE, INSERT, SELECT con JOIN. Il progetto usa PostgreSQL ma niente di esotico

**Strumenti che ti serviranno:**
- Go 1.25+ (il progetto usa funzionalità moderne di Go)
- Docker e Docker Compose (per PostgreSQL, Redis e l'intero stack)
- Node.js 20+ e pnpm (per il frontend React)

**Utile ma non richiesto:**
- Familiarità con uno qualsiasi dei sei protocolli (SSH, HTTP, FTP, SMB, MySQL, Redis) a livello di rete
- Esperienza con il framework MITRE ATT&CK
- Conoscenza di React e TypeScript (per la dashboard)

## Avvio rapido

```bash
cd PROJECTS/advanced/honeypot-network

# Check dependencies and build
./install.sh

# Start everything with Docker
just dev-up

# Or run just the backend locally (requires PostgreSQL and Redis)
just dev-serve
```

Output atteso: sei servizi honeypot iniziano ad ascoltare, l'API risponde sulla porta 8000 e la dashboard si carica su http://localhost:5173. Connettiti con `ssh root@localhost -p 2222` usando una password qualsiasi per vedere la tua prima sessione catturata.

## Struttura del progetto

```
honeypot-network/
├── cmd/hive/              # CLI entry point
├── pkg/types/             # Shared domain types (Event, Session, IOC)
├── internal/
│   ├── sshd/              # SSH honeypot (shell, filesystem, commands)
│   ├── httpd/             # HTTP honeypot (WordPress, phpMyAdmin fakes)
│   ├── ftpd/              # FTP honeypot (auth capture, upload logging)
│   ├── smbd/              # SMB honeypot (negotiate-only)
│   ├── mysqld/            # MySQL honeypot (wire protocol, query logging)
│   ├── redisd/            # Redis honeypot (RESP commands)
│   ├── event/             # Event bus + processor pipeline
│   ├── store/             # PostgreSQL + Redis persistence
│   ├── mitre/             # ATT&CK technique detection engine
│   ├── intel/             # IOC extraction, STIX export, blocklists
│   ├── api/               # REST + WebSocket dashboard API
│   └── ...                # config, geo, ratelimit, session, ui
├── frontend/              # React 19 + TypeScript dashboard
├── migrations/            # PostgreSQL schema (goose format)
├── infra/                 # Docker, nginx, Redis configs
├── learn/                 # You are here
└── compose.yml            # Production Docker Compose
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per imparare la teoria degli honeypot, l'emulazione dei protocolli e come funziona la mappatura su MITRE ATT&CK
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere il design event-driven e il flusso dei dati
3. **Ripercorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per scoprire come viene emulato ciascun protocollo
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee come aggiungere Telnet, SMTP o il rilevamento di anomalie con ML

## Problemi comuni

**Errore della host key SSH ad avvii ripetuti**
```
ssh: handshake failed: ssh: no common algorithm for host key
```
Soluzione: elimina `data/hostkey_ed25519` e riavvia. Verrà generata automaticamente una nuova chiave.

**Connessione a PostgreSQL rifiutata**
Soluzione: assicurati che il database sia in esecuzione. Con Docker: `just dev-up postgres`. In locale: verifica che PostgreSQL sia in ascolto sulla porta 5432.

**Il WebSocket del frontend non si connette**
Soluzione: il dev server di Vite fa da proxy per `/ws/*` verso il backend. Assicurati che il backend sia in esecuzione sulla porta 8000 prima di avviare il frontend.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **SIEM Dashboard** (intermedio) - Costruisce la parte di monitoraggio che consuma i dati di tool come questo
- **DLP Scanner** (principiante) - Un altro tool di rilevamento, focalizzato sugli schemi di esfiltrazione dei dati
- **AI Threat Detection** (avanzato) - Aggiunge il machine learning alla pipeline di analisi
