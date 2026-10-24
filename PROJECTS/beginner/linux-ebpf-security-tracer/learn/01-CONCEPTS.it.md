# Concetti - eBPF, Syscall e Security Observability

## eBPF: osservabilità programmabile del kernel

### Cos'è

eBPF (extended Berkeley Packet Filter) è una tecnologia che ti permette di eseguire piccoli programmi dentro il kernel Linux senza scrivere un modulo del kernel né modificare il codice sorgente del kernel. Pensalo come un linguaggio di scripting sicuro e in sandbox per il kernel.

Quando carichi un programma eBPF, il verifier del kernel ne controlla la sicurezza (nessun loop infinito, nessun accesso alla memoria fuori dai limiti, nessun crash del kernel), poi lo compila con il JIT in codice macchina nativo. Questo significa che i programmi eBPF girano a una velocità quasi nativa con forti garanzie di sicurezza.

### Perché è importante per la sicurezza

Prima di eBPF, avevi due opzioni per la visibilità a livello kernel:

1. **Moduli del kernel** - Accesso completo, ma un bug manda in crash il sistema. Caricare codice non attendibile nel kernel è intrinsecamente rischioso.
2. **Tracciamento delle system call (strace/ptrace)** - Sicuro ma lento. Il tracciamento basato su ptrace introduce un overhead da 10 a 100 volte sui processi tracciati.

eBPF ti dà visibilità a livello kernel con la sicurezza dello user-space. L'overhead prestazionale è tipicamente inferiore all'1%, e il verifier garantisce che il tuo programma non possa mandare in crash il kernel.

Per questo ogni importante tool di sicurezza cloud rilasciato dal 2020 (Falco, Tetragon, Tracee, la runtime security di Datadog) usa eBPF come fondamento.

### Come funziona

```
Your Python Script
      │
      ▼
  BCC Compiler
  (Clang/LLVM)
      │
      ▼
 eBPF Bytecode
      │
      ▼
 Kernel Verifier ──▶ Rejects unsafe programs
      │
      ▼
 JIT Compiler
      │
      ▼
 Native Machine Code
 (attached to tracepoint)
      │
      ▼
 Fires on every matching syscall
      │
      ▼
 Ring Buffer ──▶ Your Python callback
```

### BCC vs libbpf

Ci sono due framework principali per scrivere programmi eBPF:

**BCC (BPF Compiler Collection)** compila il tuo codice C eBPF a runtime usando Clang/LLVM. Il vantaggio è lo sviluppo rapido: scrivi codice C come stringa Python, lo carichi e parti. Lo svantaggio è che ogni host ha bisogno di LLVM e degli header del kernel installati, e ogni programma usa ~80MB di memoria.

**libbpf con CO-RE** (Compile Once, Run Everywhere) compila il tuo programma eBPF una sola volta in fase di build. Il binario funziona su diverse versioni del kernel grazie ai metadati BTF (BPF Type Format). Tool di produzione come Tetragon usano questo approccio perché è più leggero (~9MB) e non richiede toolchain di compilazione sugli host di produzione.

Questo progetto usa BCC perché stiamo costruendo un tool didattico, non un agent di produzione. L'API Python rende il codice leggibile, e la compilazione a runtime ti permette di sperimentare senza uno step di build.

## System call: la porta d'ingresso del kernel

### Cosa sono le system call

Ogni interazione tra un programma in user-space e il kernel passa attraverso le system call. Quando `cat` legge un file, chiama `openat()` per ottenere un file descriptor, `read()` per ottenere il contenuto e `write()` per stampare su stdout. Quando `curl` si connette a un server, chiama `socket()`, `connect()` e `read()`.

Non c'è modo di aggirarle. Anche se il malware è interamente in memoria, anche se è scritto in assembly, ha comunque bisogno di fare syscall per fare qualcosa di utile. Questo rende il tracciamento delle syscall un potente meccanismo di detection molto difficile da eludere.

### Syscall rilevanti per la sicurezza

Non tutte le oltre ~300 syscall di Linux contano per la sicurezza. Ecco quelle che questo tool traccia e perché:

**Esecuzione di processi** - `execve` scatta ogni volta che parte un nuovo programma. È la syscall più importante per il monitoraggio della sicurezza. Quasi ogni attacco implica l'esecuzione di qualcosa, che sia una shell, un payload o un tool legittimo usato in modo improprio.

**Accesso ai file** - `openat` mostra quali file i processi stanno leggendo o scrivendo. Un attaccante che legge `/etc/shadow` o scrive in `/etc/cron.d/` racconta una storia chiara.

**Attività di rete** - `connect` rivela le connessioni in uscita. Un web server che si connette all'improvviso a un IP nell'Europa orientale sulla porta 4444 è un campanello d'allarme. `bind` e `listen` mostrano processi che aprono porte per connessioni in ingresso (bind shell).

**Cambi di privilegi** - `setuid` e `setgid` mostrano le transizioni di privilegio. Un processo che chiama `setuid(0)` per diventare root è esattamente l'aspetto di una privilege escalation.

**Operazioni di sistema** - `ptrace` è usata per il debug ma anche per l'injection nei processi (MITRE ATT&CK T1055.008). `mount` può indicare tentativi di container escape. `init_module` carica moduli del kernel, ed è il modo in cui i rootkit si installano.

### La superficie di tracciamento delle syscall

```
User Space Process
        │
        │  execve("/bin/bash", ...)
        │  openat("/etc/shadow", O_RDONLY)
        │  connect(sockfd, {ip=10.0.0.1, port=4444})
        │  setuid(0)
        │
        ▼
  ┌─────────────────────────┐
  │   Syscall Entry Point   │◀── eBPF tracepoint here
  │   (kernel boundary)     │
  └─────────────────────────┘
        │
        ▼
  Kernel implementation
```

## Detection Engineering

### Dalle syscall agli alert di sicurezza

Una singola syscall presa in isolamento è raramente sospetta. `openat` scatta migliaia di volte al secondo su un sistema carico. L'arte della detection engineering sta nell'identificare quali pattern, sia singoli eventi con parametri insoliti sia sequenze di eventi, indicano un'attività malevola.

### Detection stateless

Alcuni eventi sono sospetti di per sé:

- `setuid(0)` chiamata da un processo in esecuzione con UID 1000 è quasi sempre un tentativo di escalation
- `openat("/etc/shadow")` da parte di uno script Python merita un'indagine
- `init_module()` che carica un modulo del kernel è sempre degno di nota
- `ptrace(PTRACE_ATTACH, target_pid)` è una primitiva di code injection

Queste sono detection "stateless" perché ogni evento viene valutato in modo indipendente.

### Detection stateful (correlazione di eventi)

Altre minacce diventano visibili solo quando correli più eventi:

**Pattern della reverse shell**: un attaccante su un server compromesso ha bisogno di riportare una shell interattiva sulla propria macchina. L'approccio classico:

```
1. socket(AF_INET, SOCK_STREAM)    # create TCP socket
2. connect(sockfd, attacker_ip)     # connect to attacker
3. dup2(sockfd, 0)                  # redirect stdin to socket
4. dup2(sockfd, 1)                  # redirect stdout to socket
5. dup2(sockfd, 2)                  # redirect stderr to socket
6. execve("/bin/bash")              # spawn shell
```

Nessuna singola syscall qui è sospetta. I programmi creano socket e si connettono a server in continuazione. Le shell vengono avviate di continuo. Ma una `connect` seguita da un `execve` di una shell dallo stesso PID nel giro di pochi secondi è un forte indicatore di reverse shell.

Questo tool lo implementa come regola stateful: mantiene una sliding window di eventi recenti per ogni PID. Quando arriva un execve di una shell, controlla se c'è stata di recente una `connect` dallo stesso PID o dal suo genitore.

### Esempi dal mondo reale

**2021 Log4Shell (CVE-2021-44228)**: l'exploit iniziale scatenava un lookup JNDI che scaricava ed eseguiva un payload. Dal punto di vista delle syscall: il processo Java (inatteso) chiamava `connect()` verso un server LDAP esterno, scaricava un file di classe, e poi `execve()` avviava una shell. I tool basati su eBPF lo rilevarono in tempo reale mentre i WAF venivano ancora aggiornati con le firme.

**2020 SolarWinds Supply Chain Attack**: il software Orion compromesso effettuava insolite connessioni in uscita verso `avsvmcloud.com`. Il tracciamento delle syscall avrebbe mostrato il processo Orion che chiamava `connect()` verso endpoint DNS/HTTP che non facevano parte del suo normale pattern di comunicazione.

**Container escape in Kubernetes**: CVE-2022-0185 sfruttava un heap overflow nella gestione del filesystem context del kernel. La sequenza di exploit prevedeva syscall `mount()` con parametri costruiti ad arte dall'interno di un container, qualcosa che tool basati su eBPF come Tetragon sono progettati specificamente per intercettare.

## Mappatura MITRE ATT&CK

Il framework MITRE ATT&CK fornisce un linguaggio comune per categorizzare il comportamento degli avversari. Questo tool mappa ogni regola di detection a specifiche tecniche ATT&CK:

| Detection | Technique | Tactic |
|-----------|-----------|--------|
| Privilege Escalation | T1548 - Abuse Elevation Control | Privilege Escalation |
| Sensitive File Read | T1003.008 - /etc/passwd and /etc/shadow | Credential Access |
| SSH Key Access | T1552.004 - Private Keys | Credential Access |
| Process Injection | T1055.008 - Ptrace System Calls | Defense Evasion |
| Kernel Module Load | T1547.006 - Kernel Modules | Persistence |
| Reverse Shell | T1059.004 - Unix Shell | Execution |
| Persistence via Cron | T1053.003 - Cron | Persistence |
| Log Tampering | T1070.002 - Clear Linux Logs | Defense Evasion |

## Errori comuni

### Errore: dare per scontato che i nomi delle syscall siano stabili

Il nome delle system call varia tra architetture e versioni del kernel. Su x86_64, `open()` è stata sostituita da `openat()` come syscall principale per aprire i file. Usa sempre l'interfaccia tracepoint (`syscalls:sys_enter_openat`) invece dei kprobe sulle funzioni grezze delle syscall, perché i tracepoint hanno un'ABI stabile.

### Errore: ignorare il volume degli eventi

Su un server carico, `execve` e `openat` scattano centinaia di volte al secondo. Un detection engine che fa elaborazioni costose per ogni evento resterà indietro. Questo tool usa un ring buffer (non un perf buffer) e mantiene semplice la logica di detection proprio per questo motivo.

### Errore: troppi alert

Se ogni `openat` di `/etc/passwd` genera un alert, gli operatori disattiveranno il tool nel giro di un giorno. Una buona detection engineering significa capire cosa è normale. Root che legge `/etc/shadow` è previsto (PAM lo fa a ogni login). Uno script Python che lo legge è insolito. Il contesto conta.

## Come si collegano i concetti

```
eBPF Programs ──────────────────┐
(C code in kernel)              │
  │                             │
  │ capture syscall args        │ compile + load
  │                             │
  ▼                             │
Ring Buffer ◀───────────────────┘
  │                        via BCC Python
  │ events flow to
  │ user space
  ▼
Detection Engine
  │
  │ evaluate against rules
  │ correlate sequences
  │
  ▼
MITRE ATT&CK Mapping
  │
  │ classify severity
  │
  ▼
Alert Output
```

## Standard di settore

- **MITRE ATT&CK for Linux** - Framework per categorizzare il comportamento degli avversari sui sistemi Linux
- **NIST SP 800-137** - Information Security Continuous Monitoring, che i tool basati su eBPF supportano direttamente
- **CIS Controls v8, Control 8** - Audit Log Management. Il tracciamento eBPF fornisce i dati di audit grezzi

## Metti alla prova la tua comprensione

1. Perché il malware non può evitare la detection basata sulle syscall usando l'accesso diretto alla memoria del kernel dallo user space?

2. Vedi questa sequenza dal PID 4521: `socket(AF_INET, SOCK_STREAM)`, poi `connect(10.0.0.5:443)`, poi `execve("/usr/bin/curl")`. È una reverse shell? Perché sì o perché no?

3. Una regola di detection scatta a ogni `openat("/etc/passwd")`. Su un server con 100 utenti che effettuano il login ogni ora, quanti falsi positivi all'ora ti aspetteresti? Come li ridurresti?

4. Qual è la differenza tra agganciare un programma eBPF a un kprobe e a un tracepoint? Quale è più affidabile per l'uso in produzione?

## Letture di approfondimento

### Essenziali
- [ebpf.io](https://ebpf.io) - Documentazione ufficiale di eBPF e risorse di apprendimento
- [BCC Reference Guide](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md) - Riferimento API per tutte le funzionalità di BCC
- [MITRE ATT&CK for Linux](https://attack.mitre.org/matrices/enterprise/linux/) - Matrice completa delle tecniche

### Approfondimento
- [Learning eBPF by Liz Rice](https://www.oreilly.com/library/view/learning-ebpf/9781098135119/) - Libro completo sulla programmazione eBPF
- [BPF Performance Tools by Brendan Gregg](https://www.brendangregg.com/bpf-performance-tools-book.html) - Riferimento per l'analisi di sistema basata su eBPF
- [Falco Rules Repository](https://github.com/falcosecurity/rules) - Guarda come un tool di produzione definisce le regole di detection
