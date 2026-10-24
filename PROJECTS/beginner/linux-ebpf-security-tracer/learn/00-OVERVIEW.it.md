# eBPF Security Tracer - Panoramica

## Cos'è

Un tracer di syscall Linux in tempo reale basato su eBPF che monitora l'esecuzione dei processi, l'accesso ai file, le connessioni di rete, i cambi di privilegi e le operazioni di sistema. Valuta gli eventi rispetto a regole di detection mappate sulle tecniche MITRE ATT&CK e produce alert colorati in base alla severità.

## Perché è importante

Il monitoraggio di sicurezza tradizionale si basa sull'aggregazione dei log a posteriori. Nel momento in cui controlli syslog, un attaccante potrebbe già averlo cancellato. eBPF ti permette di osservare le syscall mentre accadono, a livello di kernel, con un overhead quasi nullo. È così che funzionano sotto il cofano strumenti di sicurezza moderni come Falco, Tetragon e Tracee.

### Scenari reali

1. **Incident Response**: durante una violazione in corso, devi vedere quali processi sono in esecuzione, quali file stanno toccando e dove si stanno connettendo. Questo tool offre quella visibilità in tempo reale senza dover distribuire un intero stack SIEM.

2. **Validazione dell'hardening del server**: dopo aver messo in sicurezza un server di produzione, esegui il tracer per verificare che solo i processi attesi accedano a file sensibili come `/etc/shadow` o alle chiavi SSH. Qualsiasi accesso inatteso genera un alert.

3. **Sicurezza dei container**: negli ambienti Kubernetes, i container non dovrebbero mai caricare moduli del kernel né montare i filesystem dell'host. Il tracing basato su eBPF intercetta questi tentativi di escape a livello di syscall prima che abbiano successo.

## Cosa imparerai

### Concetti di sicurezza
- L'osservabilità a livello di syscall e perché è importante per la difesa
- Identificazione delle tecniche MITRE ATT&CK a partire dai dati grezzi delle syscall
- Detection engineering: trasformare i pattern delle syscall in regole di sicurezza
- Analisi comportamentale vs detection basata su firme

### Competenze tecniche
- Scrivere programmi eBPF in C che si agganciano ai tracepoint del kernel
- Usare i binding Python di BCC (BPF Compiler Collection)
- Architettura a ring buffer per la comunicazione tra kernel e userspace
- Correlazione degli eventi con algoritmi a finestra scorrevole
- Output strutturato di eventi di sicurezza (JSON, classificazione della severità)

### Strumenti
- Framework BCC e pipeline di compilazione eBPF
- Tooling CLI in Python con Typer e Rich
- ruff, mypy, yapf per la qualità del codice
- uv per la gestione dei pacchetti Python

## Prerequisiti

### Conoscenze richieste
- Amministrazione Linux di base (processi, file, permessi, networking)
- Fondamenti di Python (funzioni, classi, strutture dati)
- Una certa familiarità con la sintassi C (i programmi eBPF sono piccoli, ma devi saperli leggere)
- Comprensione di cosa sono le chiamate di sistema (anche se non le hai mai tracciate)

### Strumenti richiesti
- Linux con kernel 5.8+ (verifica con `uname -r`)
- Accesso root (eBPF richiede CAP_SYS_ADMIN)
- Python 3.10+
- Package manager uv
- Strumenti BCC (installati tramite `install.sh`)

### Utile ma non necessario
- Familiarità con strace o ltrace
- Concetti base di networking (TCP/IP, socket)
- Esperienza con strumenti di monitoraggio della sicurezza o SIEM

## Avvio rapido

```bash
git clone https://github.com/CarterPerez-dev/Cybersecurity-Projects.git
cd Cybersecurity-Projects/PROJECTS/beginner/linux-ebpf-security-tracer

# Install everything
./install.sh

# Start tracing
sudo uv run ebpf-tracer

# In another terminal, trigger a detection:
cat /etc/shadow    # triggers "Sensitive File Read"
```

Output atteso:

```
eBPF Security Tracer v1.0.0
Format: live | Min severity: LOW | Type: all
Press Ctrl+C to stop

[14:30:01] LOW      execve         pid=1234 comm=bash /usr/bin/cat
[14:30:01] MEDIUM   openat         pid=1234 comm=cat /etc/shadow [Sensitive File Read]
```

## Struttura del progetto

```
src/
├── main.py          # CLI entrypoint
├── config.py        # All constants and detection rule metadata
├── loader.py        # BCC loader, ring buffer setup, signal handling
├── processor.py     # Raw event parsing, enrichment, filtering
├── detector.py      # Detection engine (stateless + stateful rules)
├── renderer.py      # Output formatters (JSON, live, table)
└── ebpf/            # eBPF C programs compiled by BCC at runtime
    ├── process_tracer.c
    ├── file_tracer.c
    ├── network_tracer.c
    ├── privilege_tracer.c
    └── system_tracer.c
```

## Prossimi passi

- [01-CONCEPTS.it.md](01-CONCEPTS.it.md) - Fondamenti di eBPF, tracing delle syscall, osservabilità per la sicurezza
- [02-ARCHITECTURE.it.md](02-ARCHITECTURE.it.md) - Progettazione del sistema, ring buffer, pipeline di detection
- [03-IMPLEMENTATION.it.md](03-IMPLEMENTATION.it.md) - Walkthrough del codice di ogni modulo
- [04-CHALLENGES.it.md](04-CHALLENGES.it.md) - Idee di estensione e sfide

## Problemi comuni

**"Error: eBPF tracing requires root privileges"**
Esegui con sudo: `sudo uv run ebpf-tracer`. I programmi eBPF richiedono CAP_SYS_ADMIN per essere caricati.

**"Kernel X.Y detected. Requires 5.8+"**
Il tuo kernel è troppo vecchio per il supporto al ring buffer. Aggiorna il kernel oppure usa una VM/un container con un kernel più recente.

**"BCC Python bindings not found"**
Installa il pacchetto di sistema: `sudo apt install python3-bpfcc` (Debian/Ubuntu) oppure `sudo dnf install python3-bcc` (Fedora). BCC non è installabile con pip, deve arrivare dal package manager della tua distro.

## Progetti correlati

- [Simple Port Scanner](../../simple-port-scanner/) - Basi della ricognizione di rete
- [Simple Vulnerability Scanner](../../simple-vulnerability-scanner/) - Identificazione delle vulnerabilità
- [Linux CIS Hardening Auditor](../../linux-cis-hardening-auditor/) - Hardening del sistema
