# Concetti fondamentali di sicurezza

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci questo progetto. Non sono semplici definizioni. Approfondiremo perché sono importanti e come funzionano davvero.

## Capability di Linux

### Che cos'è

Le capability di Linux suddividono il privilegio monolitico di root in 41 permessi distinti. Invece di verificare "questo processo è root?", il kernel verifica "questo processo ha CAP_NET_ADMIN?" prima di consentire modifiche alla configurazione di rete.

Il modello delle capability è stato introdotto nel kernel 2.2 (1999) per abilitare la separazione dei privilegi. Un web server che si mette in ascolto sulla porta 80 ha bisogno solo di `CAP_NET_BIND_SERVICE`, non dell'accesso root completo. Un programma di backup ha bisogno solo di `CAP_DAC_READ_SEARCH` per leggere file protetti.

### Perché è importante

I container vengono eseguiti con un insieme predefinito di capability che abilitano le operazioni comuni. Docker rimuove per impostazione predefinita 13 capability pericolose, ma ne concede comunque altre 14. Riaggiungere le capability rimosse o concederne di nuove può consentire una container escape completa.

Quando esegui `docker run --privileged`, Docker concede tutte le 41 capability. Equivale a eseguire come root sull'host, aggirando ogni meccanismo di isolamento del container.

### Come funziona

Le capability sono memorizzate come bitmask a 64 bit nella struttura delle credenziali del processo. Il kernel verifica questi bit prima delle operazioni privilegiate. Ecco come questo progetto le decodifica da `/proc/PID/status`:

```go
// internal/proc/capabilities.go:87-112
func (c *CapabilitySet) HasCapability(name string) bool {
    bit, ok := capabilityNames[name]
    if !ok {
        return false
    }
    return (c.Effective & (1 << bit)) != 0
}
```

L'insieme `Effective` determina cosa può fare attualmente il processo. Gli altri insiemi (`Permitted`, `Inheritable`, `Bounding`, `Ambient`) controllano le transizioni delle capability attraverso exec() e i cambi di privilegio.

Il progetto mappa tutte le 41 capability su livelli di gravità in `internal/rules/capabilities.go:24-323`:

```go
"CAP_SYS_ADMIN": {
    Severity:    finding.SeverityCritical,
    Description: "Perform a range of system administration operations...",
}
```

Durante la scansione dei container, il codice in `internal/analyzer/container.go:76-102` itera sulle capability aggiunte e crea rilevamenti per quelle pericolose:

```go
for _, cap := range info.HostConfig.CapAdd {
    capName := strings.ToUpper(string(cap))
    capInfo, exists := rules.GetCapabilityInfo(capName)
    if !exists {
        continue
    }
    if capInfo.Severity >= finding.SeverityHigh {
        // Create finding with severity from capability database
    }
}
```

### Attacchi comuni

**Container escape tramite CAP_SYS_ADMIN**

Questa capability consente di montare filesystem. Un attaccante può montare il filesystem root dell'host all'interno del container:

```bash
# Inside container with CAP_SYS_ADMIN
mkdir /host
mount /dev/sda1 /host
chroot /host
# Now running on the host
```

L'exploit Shocker (CVE-2014-6407) usava `CAP_DAC_READ_SEARCH` in modo simile. Poteva aprire qualsiasi file descriptor tramite percorso, aggirando l'isolamento dei namespace.

**Process injection tramite CAP_SYS_PTRACE**

Con questa capability, un processo può usare ptrace() per agganciarsi a qualsiasi altro processo, leggerne la memoria e iniettare codice:

```c
ptrace(PTRACE_ATTACH, target_pid, NULL, NULL);
ptrace(PTRACE_POKETEXT, target_pid, address, shellcode);
ptrace(PTRACE_SETREGS, target_pid, NULL, &regs);
ptrace(PTRACE_CONT, target_pid, NULL, NULL);
```

Un attaccante in un container con `CAP_SYS_PTRACE` e `--pid=host` può iniettare codice nel PID 1 dell'host.

**Manipolazione della rete tramite CAP_NET_ADMIN**

Questa capability consente di modificare tabelle di routing, regole del firewall e namespace di rete. Un attaccante può:

- Reindirizzare il traffico destinato ad altri container
- Creare tunnel di rete per esfiltrare dati
- Modificare le regole iptables per aggirare le policy di rete
- Evadere verso il namespace di rete dell'host

### Strategie di difesa

Rimuovi tutte le capability, poi riaggiungi solo ciò che serve:

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

Questo progetto rileva quando i container aggiungono capability pericolose. La gravità proviene dal database delle capability, non da logica hardcoded. Aggiungere una nuova capability rischiosa richiede solo di aggiornare `internal/rules/capabilities.go`.

## Isolamento dei namespace

### Che cos'è

I namespace di Linux forniscono viste separate delle risorse di sistema. I processi in namespace diversi vedono insiemi diversi di PID, interfacce di rete, mount point, oggetti IPC, hostname, utenti e cgroup.

Docker crea per impostazione predefinita 6 namespace per ogni container:
- **PID namespace** - isola gli ID dei processi
- **Network namespace** - isola le interfacce di rete
- **Mount namespace** - isola i mount point del filesystem
- **IPC namespace** - isola le IPC System V e le code di messaggi POSIX
- **UTS namespace** - isola hostname e nome di dominio
- **User namespace** - isola gli ID di utenti e gruppi (opzionale, non predefinito)

### Perché è importante

Condividere un namespace con l'host infrange quel confine di isolamento. Eseguire con `--pid=host` permette al container di vedere tutti i processi dell'host. Con `CAP_SYS_PTRACE`, può iniettare codice al loro interno.

Il più pericoloso è `--net=host`, che colloca il container sullo stack di rete dell'host. A quel punto il container può mettersi in ascolto su qualsiasi porta dell'host, intercettare tutto il traffico e riconfigurare la rete dell'host.

### Come funziona

Il kernel mantiene un inode di namespace per ogni tipo di namespace. I processi appartengono ai namespace tramite questi inode. La directory `/proc/PID/ns/` li espone come symlink:

```bash
$ ls -l /proc/self/ns/
lrwxrwxrwx 1 user user 0 Jan 31 12:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 user user 0 Jan 31 12:00 net -> 'net:[4026531840]'
```

I container in namespace diversi hanno numeri di inode diversi. Il codice in `internal/analyzer/container.go:125-161` verifica se i container condividono i namespace dell'host:

```go
if info.HostConfig.NetworkMode == "host" {
    control, _ := benchmark.Get("5.9")
    f := finding.New("CIS-5.9", control.Title, finding.SeverityHigh, target)
    findings = append(findings, f)
}
```

I manifest Kubernetes commettono spesso questo errore:

```yaml
spec:
  hostNetwork: true  # Shares host network namespace
  hostPID: true      # Shares host PID namespace
  containers:
  - name: monitor
    image: debug-tools
```

### Attacchi comuni

**Scoperta dei processi dell'host tramite --pid=host**

Con il PID namespace dell'host, un attaccante può enumerare tutti i processi dell'host, trovare servizi sensibili e identificare bersagli di attacco:

```bash
# Inside container with --pid=host
ps aux | grep -i password
ls -la /proc/*/environ | xargs grep -a SECRET
```

Questo espone le righe di comando dei processi, le variabili d'ambiente e i file descriptor aperti.

**Sniffing di rete tramite --net=host**

Sul namespace di rete dell'host, il container può eseguire strumenti di cattura dei pacchetti per intercettare il traffico destinato ad altri container o all'host:

```bash
tcpdump -i eth0 -w capture.pcap
# Exfiltrate capture.pcap later
```

Questo aggira completamente l'isolamento di rete di Docker.

### Strategie di difesa

Non condividere mai i namespace dell'host in produzione. Il codice in `internal/analyzer/container.go:145-161` segnala tutti i namespace condivisi:

```go
if info.HostConfig.PidMode == "host" {
    // Creates HIGH severity finding for CIS 5.15
}
if info.HostConfig.IpcMode == "host" {
    // Creates HIGH severity finding for CIS 5.16
}
```

Per il debug, usa `docker exec` per eseguire gli strumenti nei namespace del container invece di condividere quelli dell'host.

## Profili di sicurezza (Seccomp, AppArmor, SELinux)

### Che cos'è

I profili di sicurezza impongono un mandatory access control che va oltre i permessi Unix tradizionali:

**Seccomp** (Secure Computing Mode) filtra le system call usando programmi BPF (Berkeley Packet Filter). Può bloccare syscall pericolose come `ptrace()`, `mount()` e `reboot()`.

**AppArmor** limita l'accesso ai file usando regole basate sui percorsi. Un profilo può consentire la lettura di `/etc/nginx/` ma negare la scrittura ovunque tranne che in `/var/log/nginx/`.

**SELinux** (Security-Enhanced Linux) impone un mandatory access control basato sui tipi. Usa contesti di sicurezza come `system_u:object_r:httpd_sys_content_t:s0` per controllare gli accessi.

### Perché è importante

Senza questi profili, un container compromesso ha accesso completo alle syscall e può tentare tecniche di container escape. Il profilo seccomp predefinito di Docker blocca circa 44 syscall pericolose, ma i container possono essere eseguiti con `--security-opt seccomp=unconfined`.

### Come funziona

Docker applica un profilo seccomp predefinito a meno che tu non lo sovrascriva. Il profilo è un file JSON che definisce le syscall consentite:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Il codice in `internal/analyzer/container.go:163-207` verifica la presenza di profili mancanti o disabilitati:

```go
for _, opt := range info.HostConfig.SecurityOpt {
    if opt == "seccomp=unconfined" {
        // Creates HIGH severity finding
    }
}
```

I profili AppArmor sono file di testo in `/etc/apparmor.d/`:

```
profile docker-default flags=(attach_disconnected,mediate_deleted) {
  deny @{PROC}/* w,
  deny /sys/[^f]*/** wklx,
  capability setuid,
}
```

### Errori comuni

**Disabilitare seccomp per comodità**

Gli sviluppatori disabilitano seccomp durante il troubleshooting senza comprenderne il rischio:

```bash
# Bad: Complete syscall access
docker run --security-opt seccomp=unconfined app

# Good: Custom profile with needed syscalls
docker run --security-opt seccomp=./custom-profile.json app
```

Il progetto lo rileva in `internal/analyzer/container.go:186-193`.

**Profili completamente assenti**

Le versioni più vecchie di Docker o alcune configurazioni non applicano profili per impostazione predefinita. Il codice verifica sia la disabilitazione esplicita sia l'assenza:

```go
if !hasSeccomp && !info.HostConfig.Privileged {
    // Creates MEDIUM severity finding
}
```

### Strategie di difesa

Usa sempre i profili di sicurezza in produzione. Parti dai valori predefiniti di Docker e personalizza solo quando necessario. Il progetto segnala sia i profili disabilitati sia quelli mancanti con gravità diverse (HIGH per la disabilitazione esplicita, MEDIUM per l'assenza).

## Mount di percorsi sensibili

### Che cos'è

I bind mount espongono directory e file dell'host all'interno dei container. Alcuni percorsi, se montati, danno accesso completo all'host, come `/var/run/docker.sock` (controllo di Docker), `/proc` (informazioni sui processi) e `/` (l'intero filesystem).

### Perché è importante

Il socket Docker è un socket UNIX che accetta comandi della Docker API. Un container con questo socket montato può creare nuovi container privilegiati, evadendo di fatto verso l'host:

```bash
# Inside container with /var/run/docker.sock mounted
docker run --privileged -v /:/host -it ubuntu chroot /host /bin/bash
# Now on the host
```

È così che è avvenuta la violazione del cluster Kubernetes di Tesla.

### Come funziona

I bind mount di Docker si specificano con `-v` o `--mount`. La Docker API include le informazioni sui mount nell'ispezione del container:

```json
{
  "Mounts": [
    {
      "Type": "bind",
      "Source": "/var/run/docker.sock",
      "Destination": "/var/run/docker.sock"
    }
  ]
}
```

Il codice in `internal/analyzer/container.go:104-145` confronta ogni mount con un database di oltre 200 percorsi pericolosi definiti in `internal/rules/paths.go`:

```go
for _, mount := range info.Mounts {
    source := mount.Source
    if rules.IsDockerSocket(source) {
        // Creates CRITICAL severity finding
    }
    if rules.IsSensitivePath(source) {
        severity := rules.GetPathSeverity(source)
        // Creates finding with path-specific severity
    }
}
```

Il database delle regole include:

- Socket dei container runtime (Docker, containerd, CRI-O)
- Directory di sistema (`/proc`, `/sys`, `/dev`)
- Directory di configurazione (`/etc`, `/root`)
- Segreti di Kubernetes (`/var/lib/kubelet`, `/etc/kubernetes`)
- Credenziali dei cloud provider (`/root/.aws`, `/root/.kube`)
- Dati degli agent CI/CD (`/var/lib/jenkins`, `/home/runner`)

Ogni percorso ha un livello di gravità e una descrizione in `internal/rules/paths.go:32-1100`.

### Attacchi comuni

**Escape tramite il socket Docker**

È la tecnica di container escape più comune:

```python
# Python script inside container
import docker
client = docker.from_env()  # Uses /var/run/docker.sock

# Create privileged container with host filesystem mounted
container = client.containers.run(
    'ubuntu',
    'chroot /host /bin/bash',
    privileged=True,
    volumes={'/': {'bind': '/host'}},
    detach=True,
    stdin_open=True,
    tty=True
)
```

**Accesso alla memoria dei processi tramite /proc**

Montare `/proc` espone la memoria dei processi dell'host. Leggere `/proc/PID/mem` può estrarre credenziali, chiavi SSH e altri segreti dai processi in esecuzione.

**Manipolazione del kernel tramite /sys**

Il filesystem `/sys` espone i parametri del kernel. Scrivere in `/sys/kernel/` può disabilitare funzionalità di sicurezza, modificare le impostazioni della CPU e innescare vulnerabilità del kernel.

### Strategie di difesa

Non montare mai il socket Docker a meno che il container non fornisca specificamente Docker-as-a-Service (come i runner CI/CD). Anche in quel caso, usa alternative come la modalità rootless di Docker o Kaniko per le build.

Il database dei percorsi del progetto è completo. Aggiungere un nuovo percorso pericoloso richiede solo una voce in `internal/rules/paths.go`.

## Rilevamento dei segreti

### Che cos'è

I segreti sono credenziali incorporate in Dockerfile, variabili d'ambiente o file docker-compose. Includono chiavi API, password, chiavi private, URL di database e token.

### Perché è importante

I segreti nelle image dei container sono facilmente individuabili. Chiunque possa fare il pull dell'image può estrarre i segreti:

```bash
docker history image:tag
docker inspect image:tag
docker run image:tag env
```

Nel 2019, alcuni ricercatori hanno trovato oltre 100.000 image su Docker Hub con segreti trapelati. Molti erano chiavi API attive per AWS, GitHub e altri servizi.

### Come funziona

Questo progetto usa due tecniche per il rilevamento dei segreti:

**Pattern matching** con oltre 80 espressioni regolari in `internal/rules/secrets.go:29-821`:

```go
{
    Type: SecretTypeAWSKey,
    Pattern: regexp.MustCompile(`(AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16}`),
    Description: "AWS Access Key ID",
}
```

**Analisi dell'entropia** usando l'entropia di Shannon:

```go
// internal/rules/secrets.go:910-925
func CalculateEntropy(s string) float64 {
    freq := make(map[rune]float64)
    for _, c := range s {
        freq[c]++
    }
    length := float64(len(s))
    var entropy float64
    for _, count := range freq {
        p := count / length
        entropy -= p * math.Log2(p)
    }
    return entropy
}
```

I segreti hanno tipicamente un'entropia elevata (>4,5 bit per carattere) perché sono casuali. Il codice in `internal/analyzer/dockerfile.go:154-213` scansiona i Dockerfile:

```go
if rules.IsSensitiveEnvName(varName) {
    // Check if variable name looks sensitive
}
if rules.IsHighEntropyString(varValue, 16, 4.5) {
    // Check if value has high entropy
}
secrets := rules.DetectSecrets(line)
for _, secret := range secrets {
    // Create finding for each detected secret type
}
```

### Pattern comuni

Il progetto rileva:
- Chiavi AWS: `AKIA...`, `aws_secret_access_key=...`
- Token GitHub: `ghp_...`, `gho_...`
- Chiavi API di Google: `AIza...`
- Chiavi private: `-----BEGIN PRIVATE KEY-----`
- JWT: `eyJ...` (JSON codificato in base64)
- URL di database: `postgres://user:pass@host/db`
- Pattern generici: `API_KEY=...`, `PASSWORD=...`

Consulta `internal/rules/secrets.go:29-821` per l'elenco completo.

### Strategie di difesa

Usa i Docker secrets o le variabili d'ambiente a runtime:

```bash
# Bad: Secret in Dockerfile
ENV API_KEY=sk-abc123

# Good: Secret from environment
docker run -e API_KEY=sk-abc123 app

# Better: Docker secrets (Swarm)
echo "sk-abc123" | docker secret create api_key -
docker service create --secret api_key app
```

Il progetto segnala sia i segreti hardcoded sia le stringhe ad alta entropia nelle istruzioni `ENV`, `ARG` e `RUN`.

## CIS Docker Benchmark

### Che cos'è

Il CIS Docker Benchmark è una guida alla configurazione di sicurezza basata sul consenso, pubblicata dal Center for Internet Security. Fornisce oltre 100 controlli distribuiti su 7 sezioni:

1. Host Configuration
2. Docker Daemon Configuration  
3. Docker Daemon Files and Directories
4. Container Images and Build Files
5. Container Runtime
6. Docker Security Operations
7. Docker Swarm Configuration

Ogni controllo ha un ID (come "5.4"), un titolo, una descrizione e i passi di remediation. I controlli sono contrassegnati come "scored" (richiesti per la conformità) o "not scored" (raccomandati).

### Perché è importante

Il benchmark rappresenta le best practice del settore sviluppate da professionisti della sicurezza. Seguirlo previene le configurazioni errate comuni che portano a violazioni.

Molti framework di compliance (PCI DSS, HIPAA, SOC 2) richiedono di seguire i benchmark CIS o standard equivalenti. Questo progetto automatizza la verifica della conformità.

### Come funziona

Il progetto registra tutti i controlli CIS in `internal/benchmark/controls.go:61-end`. Ogni controllo include dei metadati:

```go
Register(Control{
    ID:          "5.4",
    Section:     "Container Runtime",
    Title:       "Ensure that privileged containers are not used",
    Description: "Using --privileged gives all capabilities to the container...",
    Remediation: "Do not run containers with --privileged flag...",
    Severity:    finding.SeverityCritical,
    Scored:      true,
    Level:       1,
    References:  []string{"https://docs.docker.com/engine/reference/run/"},
})
```

Gli analyzer creano rilevamenti che fanno riferimento a questi controlli:

```go
control, _ := benchmark.Get("5.4")
f := finding.New("CIS-5.4", control.Title, finding.SeverityCritical, target).
    WithDescription(control.Description).
    WithRemediation(control.Remediation).
    WithCISControl(control.ToCISControl())
```

Il flag `--cis` filtra i rilevamenti per ID di controllo. L'output SARIF include gli ID dei controlli CIS come tag per l'integrazione con gli strumenti di compliance.

### Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Perché `CAP_SYS_ADMIN` è peggiore di `CAP_NET_RAW`? Quale attacco specifico rende possibile ciascuna?
2. Se un container condivide il PID namespace dell'host ma non ha capability speciali, può comunque essere pericoloso? In che modo?
3. Perché ci servono sia il pattern matching sia l'analisi dell'entropia per il rilevamento dei segreti? Fai un esempio di segreto che ciascuna tecnica intercetterebbe.

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. L'implementazione avrà più senso una volta assimilati questi fondamenti.

## Letture di approfondimento

**Essenziali:**
- CIS Docker Benchmark v1.6.0 (PDF) - la specifica completa implementata da questo progetto
- Linux Programmer's Manual capabilities(7) - la documentazione di riferimento sulle capability
- Docker Security Documentation - le best practice ufficiali di sicurezza di Docker

**Approfondimenti:**
- "Understanding and Hardening Linux Containers" (whitepaper di NCC Group) - funzionamento interno dei container e tecniche di escape
- "A Measurement Study of Docker Container Configurations on GitHub" (articolo di ricerca) - dati sulle configurazioni errate nel mondo reale
- Sorgente del kernel: security/capability.c - come funzionano effettivamente le verifiche delle capability nel kernel

**Contesto storico:**
- L'audit di sicurezza originale di Docker di ThoughtWorks (2014) - ha identificato molti dei problemi affrontati da questo benchmark
- "Dirty COW" (CVE-2016-5195) - vulnerabilità del kernel sfruttabile dai container
- La vulnerabilità di runc CVE-2019-5736 - container escape tramite un'image malevola
