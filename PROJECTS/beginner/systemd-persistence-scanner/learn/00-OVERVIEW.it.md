# Sentinel - Systemd Persistence Scanner

## Cos'è

Uno strumento CLI a singolo binario che scansiona un sistema Linux alla ricerca di meccanismi di persistenza attraverso 17 moduli scanner: unit systemd, cron job, profili di shell, configurazione SSH, hijacking di LD_PRELOAD, moduli del kernel, regole udev, script init.d, autostart XDG, job at, script MOTD, configurazione PAM, script di login sshrc, hook di logrotate, generatori systemd, script di completamento bash e hook delle interfacce di rete. Ogni finding è etichettato con un ID di tecnica MITRE ATT&CK e una severità da info a critical.

## Perché è importante

La persistenza è il modo in cui gli attaccanti sopravvivono ai reboot. Dopo l'accesso iniziale, la prima cosa che fa un avversario competente è installare un meccanismo che lo riporti dentro. Linux ha decine di posizioni in cui il codice può essere innescato automaticamente: servizi di boot, eventi di login, cambi hardware, task pianificati, inizializzazione della shell, caricamento di moduli, eventi dei dispositivi.

Nel compromesso SolarWinds del 2020, gli attaccanti hanno modificato i servizi systemd sui build server Linux per mantenere l'accesso attraverso gli aggiornamenti. Nell'incidente Codecov del 2021, gli attaccanti hanno iniettato comandi curl negli script dei profili di shell sui runner CI/CD. Il malware Orbit per Linux del 2022 ha usato LD_PRELOAD per agganciare (hook) le funzioni di libc e nascondersi da ogni strumento di detection sul sistema.

**Scenari del mondo reale in cui questo si applica:**
- Triage di incident response: rilascia il binario su un host compromesso e ottieni un inventario completo della persistenza in pochi secondi
- Audit di hardening: stabilisci una baseline di un server pulito, poi fai il diff dopo i deploy per intercettare persistenze non intenzionali
- Threat hunting: scansiona una fleet alla ricerca di pattern di persistenza noti come reverse shell nel cron o manipolazione SUID negli script di profilo

## Cosa imparerai

Questo progetto ti insegna come funziona la persistenza su Linux a ogni livello della sequenza di boot e login. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**
- Tassonomia della persistenza su Linux: le 17 categorie di posizioni in cui il codice viene eseguito automaticamente
- La tattica Persistence di MITRE ATT&CK (TA0003): mappare tecniche reali alla logica di detection
- Detection euristica: pattern matching per reverse shell, catene di download-and-execute, payload codificati, hijacking di alias e primitive di privilege escalation
- Baseline diffing: stabilire uno stato known-good e rilevare il drift

**Competenze tecniche:**
- Layout di modulo Go con package interni, tipi condivisi e un layer CLI
- Pattern del registry degli scanner con registrazione in fase di init ed esecuzione parallela
- Scansione concorrente con errgroup e raccolta dei risultati protetta da mutex
- Motore di pattern a regex compilate con matching ordinato per severità
- Framework CLI Cobra con sottocomandi, flag persistenti e output strutturato

**Strumenti e tecniche:**
- golangci-lint v2 con i formatter gci, gofumpt, golines per uno stile di codice consistente
- `go install` per la distribuzione del binario con un singolo comando
- Il command runner just per i workflow di sviluppo
- Output JSON per l'ingestion nel SIEM e l'integrazione nella pipeline

## Prerequisiti

**Conoscenze richieste:**
- Basi di Go: struct, interfacce, goroutine, channel, gestione degli errori
- Layout del filesystem Linux: /etc, /home, /var/spool, /lib, /run
- Comprensione di base di cosa fanno i servizi, il cron e i profili di shell

**Strumenti di cui avrai bisogno:**
- Go 1.25+ (il go.mod specifica questo minimo)
- golangci-lint v2 (per il linting)
- just (opzionale, per eseguire i comandi di sviluppo)

**Utile ma non richiesto:**
- Familiarità con la sintassi dei file unit di systemd
- Esperienza con i pattern regex
- Comprensione del framework MITRE ATT&CK

## Quick Start

```bash
cd PROJECTS/beginner/systemd-persistence-scanner

./install.sh

./bin/sentinel scan
```

Oppure installalo globalmente:

```bash
go install github.com/CarterPerez-dev/sentinel/cmd/sentinel@latest
sentinel scan
```

Output atteso: lo strumento stampa il banner SENTINEL in ciano/rosso alternati, scansiona tutte le 17 categorie di persistenza in parallelo e visualizza eventuali finding raggruppati per severità con etichette a colori, percorsi dei file, snippet di evidenza e ID di tecnica MITRE. Su un sistema pulito vedrai per lo più finding di livello info per servizi e cron job legittimi.

## Struttura del progetto

```
systemd-persistence-scanner/
├── cmd/sentinel/          # Entry point (main.go)
├── pkg/types/             # Shared domain types (Finding, Severity, Scanner interface)
├── internal/
│   ├── cli/               # Cobra commands (scan, baseline save/diff)
│   ├── scanner/           # 17 scanner modules + registry + pattern engine + helpers
│   ├── baseline/          # JSON snapshot save/load/diff
│   ├── config/            # Ignore-list filtering
│   ├── report/            # Terminal and JSON output formatters
│   └── ui/                # Color, spinner, banner, symbols
├── testdata/              # Fixture files for each scanner's tests
├── Justfile               # Development command runner
└── install.sh             # Zero-friction setup script
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per imparare le tecniche di persistenza su Linux e la mappatura a MITRE ATT&CK
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) per vedere il registry degli scanner, l'esecuzione parallela e il flusso dei dati
3. **Esamina il codice** - Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per il motore di pattern, i moduli scanner e il baseline diffing
4. **Estendi il progetto** - Leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per idee come aggiungere nuovi scanner e l'integrazione con YARA

## Problemi comuni

**"go: go.mod requires go >= 1.25"**
Ti serve Go 1.25 o successivo. Scaricalo da https://go.dev/dl/

**Nessun finding su un container o VM minimale**
È previsto. I sistemi puliti hanno pochi meccanismi di persistenza. Prova `sentinel scan --root testdata` per scansionare le fixture di test incluse e vedere il motore di detection in azione.

**Permesso negato su /var/spool/cron o /etc/shadow**
Alcune directory richiedono l'accesso root. Esegui `sudo sentinel scan` per una scansione completa del sistema, oppure usa `--root` per scansionare un'immagine di filesystem montata senza privilegi elevati.

## Progetti correlati

Se hai trovato questo interessante, dai un'occhiata a:
- [Linux CIS Hardening Auditor](../../linux-cis-hardening-auditor) - Verifica di conformità rispetto ai benchmark CIS
- [Simple Vulnerability Scanner](../../simple-vulnerability-scanner) - Scansione delle dipendenze basata sui CVE
- [Linux eBPF Security Tracer](../../linux-ebpf-security-tracer) - Monitoraggio delle syscall in tempo reale con eBPF
