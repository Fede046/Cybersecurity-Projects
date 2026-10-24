# Sfide Pratiche

Queste sfide si basano sul codebase effettivo. Ciascuna estende docksec con controlli di sicurezza reali che contano negli ambienti di produzione.

Inizia dal Livello 1 se il codebase è nuovo per te. Affrontale in sequenza perché le sfide successive si basano sui concetti precedenti.

## Livello 1: Aggiungere Controlli Semplici

Queste sfide insegnano il pattern degli analyzer senza richiedere approfondite conoscenze pregresse di Docker o di sicurezza.

### Sfida 1.1: Rilevare i Tag di Immagine :latest

**Cosa costruire:**
Aggiungere un controllo che segnali i container che eseguono immagini con il tag `:latest` o senza alcun tag.

**Perché è importante:**
`docker run nginx` scarica nginx:latest. L'immagine nginx:latest di domani potrebbe essere completamente diversa. Il blocco delle versioni (version pinning) previene rotture inattese e regressioni di sicurezza.

Incidente reale: nel 2019, una compromissione di Docker Hub ha fatto sì che i tag `:latest` puntassero a immagini contenenti backdoor per diverse ore. Le versioni con tag fissati non hanno subito impatti.

**Da dove iniziare:**
File: `internal/analyzer/container.go`

Osserva il metodo `checkPrivileged()` (righe 72-86). Il tuo controllo segue lo stesso pattern:
```go
func (a *ContainerAnalyzer) checkImageTag(
    target finding.Target,
    info types.ContainerJSON,
) finding.Collection {
    // Your code here
    // Hint: info.Config.Image contains the full image reference
    // Examples: "nginx:latest", "postgres:13.4", "redis" (no tag)
}
```

**Passaggi di implementazione:**

1. Verificare se la stringa dell'immagine termina con `:latest`
2. Verificare se l'immagine non contiene il carattere `:` (assume come impostazione predefinita :latest)
3. Creare il finding usando benchmark.Get("5.27") per il controllo CIS
4. Aggiungere la chiamata a questo metodo in `analyzeContainer()` attorno alla riga 60
5. Testare: `docker run -d --name test nginx:latest && docksec scan`

**Output previsto:**
```
[MEDIUM] Container uses :latest tag
  Container: test
  CIS Control: 5.27
  Description: Using :latest makes image version unpredictable...
```

**Suggerimenti:**
- Usa `strings.HasSuffix()` e `strings.Contains()`
- Un tag vuoto equivale per impostazione predefinita a `:latest`, quindi `nginx` e `nginx:latest` sono equivalenti
- Non segnalare `nginx@sha256:abc123...` (il pinning tramite digest è una buona pratica)

**Approfondimento:**
Dopo che l'implementazione di base funziona, gestisci i casi limite (edge case):
- Manifest multi-architettura: `nginx:latest@sha256:...` ha il tag latest ma il digest è bloccato
- Registry privati: `registry.example.com:5000/nginx:latest`
- Immagini ufficiali: `nginx` rispetto a `library/nginx` rispetto a `docker.io/library/nginx`

### Sfida 1.2: Verificare le Porte Sensibili Esposte

**Cosa costruire:**
Segnalare i container con porte pubblicate in intervalli sensibili (come 22, 3389, 5432).

**Perché è importante:**
Esporre accidentalmente SSH (22) o RDP (3389) su 0.0.0.0 è un errore comune. Gli scanner automatizzati individuano queste esposizioni nel giro di pochi minuti.

Esempio reale: nel 2020, oltre 45.000 host Docker hanno esposto la porta 2375 (Docker API) su Internet. Gli aggressori li hanno sfruttati per attività di cryptomining.

**Da dove iniziare:**
File: `internal/analyzer/container.go`

I binding delle porte del container si trovano in `info.NetworkSettings.Ports`. Si tratta di una mappa in cui le chiavi sono le porte del container e i valori sono i binding dell'host.
```go
func (a *ContainerAnalyzer) checkExposedPorts(
    target finding.Target,
    info types.ContainerJSON,
) finding.Collection {
    // Your code here
    // Hint: info.NetworkSettings.Ports is map[nat.Port][]nat.PortBinding
}
```

**Porte sensibili da verificare:**
- 22 (SSH)
- 23 (Telnet)
- 3306 (MySQL)
- 5432 (PostgreSQL)
- 6379 (Redis)
- 27017 (MongoDB)
- 3389 (RDP)
- 5900 (VNC)

**Passaggi di implementazione:**

1. Creare `internal/rules/ports.go` con la mappa delle porte sensibili:
```go
   var SensitivePorts = map[int]string{
       22: "SSH",
       23: "Telnet",
       // ... rest
   }
```

2. Iterare su `info.NetworkSettings.Ports`
3. Estrarre il numero di porta da nat.Port (formato: "8080/tcp")
4. Verificare se HostIP è "0.0.0.0" oppure vuoto (la stringa vuota indica tutte le interfacce)
5. Creare un finding se la porta è sensibile ed è esposta su tutte le interfacce

**Output previsto:**
```
[HIGH] Sensitive port exposed to all interfaces
  Container: database
  Port: 5432 (PostgreSQL) -> 0.0.0.0:5432
  Remediation: Bind to localhost only: docker run -p 127.0.0.1:5432:5432
```

**Suggerimenti:**
- Il formato di `nat.Port` è "3306/tcp" o "53/udp"
- Usa `strconv.Atoi()` per estrarre il numero di porta
- HostIP "0.0.0.0" indica tutte le interfacce; anche una stringa vuota indica tutte le interfacce
- Il binding su 127.0.0.1 limita l'accesso esclusivamente alla macchina locale

**Approfondimento:**
- Controlla gli intervalli di porte: `-p 8000-9000:8000-9000` espone 1000 porte
- Rileva le esposizioni su IPv6: `::` è l'equivalente IPv6 di `0.0.0.0`
- Considerazioni su Kubernetes: i servizi ClusterIP non compaiono tra le porte dei container

### Sfida 1.3: Individuare Container Senza Limiti di Risorse

**Cosa costruire:**
Verificare se i container hanno limiti di memoria e di CPU configurati.

**Perché è importante:**
Un container privo di limite di memoria può consumare tutta la RAM dell'host. Un singolo processo Node.js impazzito può causare il blocco di ogni container presente sull'host.

Incidente reale: un cluster Kubernetes presso Shopify ha mandato offline la produzione perché un pod privo di limiti di memoria ha avuto un leak di 64 GB, scatenando l'OOM killer che ha terminato pod di sistema critici.

**Da dove iniziare:**
I limiti di risorsa si trovano in `info.HostConfig.Memory` e `info.HostConfig.NanoCpus`.
```go
func (a *ContainerAnalyzer) checkResourceLimits(
    target finding.Target,
    info types.ContainerJSON,
) finding.Collection {
    var findings finding.Collection
    
    // Memory limit
    if info.HostConfig.Memory == 0 {
        // Create finding
    }
    
    // CPU limit
    if info.HostConfig.NanoCpus == 0 {
        // Create finding
    }
    
    return findings
}
```

**Passaggi di implementazione:**

1. Verificare il campo `Memory` (in byte, 0 significa illimitato)
2. Verificare il campo `NanoCpus` (1 CPU = 1e9 nanocpu, 0 significa illimitato)
3. Creare finding separati per ciascun limite mancante
4. Utilizzare il controllo CIS 5.10 per la memoria, 5.11 per la CPU

**Output previsto:**
```
[MEDIUM] Container has no memory limit
  Container: web-app
  CIS Control: 5.10
  Current: unlimited
  Remediation: docker run --memory=512m ...

[MEDIUM] Container has no CPU limit
  Container: web-app
  CIS Control: 5.11
  Current: unlimited
  Remediation: docker run --cpus=1.5 ...
```

**Suggerimenti:**
- `Memory` è espressa in byte: 512MB = 536870912
- `NanoCpus` utilizza la notazione scientifica: 2 CPU = 2000000000
- Alcune piattaforme impostano automaticamente limiti minimi (controlla anche MemoryReservation)
- CPU shares rispetto a CPU quota: controlli differenti, entrambi importanti

**Approfondimento:**
- Controlla MemorySwap (dovrebbe essere uguale a Memory, altrimenti i container possono fare swap su disco)
- Controlla CPUShares (pesi relativi tra container)
- Valida che i limiti siano ragionevoli (un limite di memoria di 1MB andrà in crash immediatamente)

## Livello 2: Costruire Nuovi Analyzer

Queste sfide richiedono la comprensione delle API di Docker e del parsing dei file.

### Sfida 2.1: Aggiungere un Network Security Analyzer

**Cosa costruire:**
Un nuovo analyzer che controlli le configurazioni di rete di Docker per individuare problemi di sicurezza.

**Problemi di sicurezza da rilevare:**
- Utilizzo della rete bridge predefinita (i container possono comunicare tra loro)
- Reti prive di cifratura
- Reti con IPv6 abilitato (spesso dimenticate e non monitorate)
- Reti personalizzate senza un'adeguata configurazione IPAM

**Perché è importante:**
La rete bridge predefinita di Docker non offre alcun isolamento tra i container. Un container compromesso può effettuare pivoting verso gli altri presenti sulla stessa rete.

Attacco reale: nel 2018 l'attacco di cryptojacking subito da Tesla su Kubernetes ha sfruttato il networking dei container per propagarsi tra i pod, arrivando a compromettere credenziali di amministrazione.

**Da dove iniziare:**
Crea `internal/analyzer/network.go`.

Il pattern:
```go
type NetworkAnalyzer struct {
    client *docker.Client
}

func NewNetworkAnalyzer(client *docker.Client) *NetworkAnalyzer {
    return &NetworkAnalyzer{client: client}
}

func (a *NetworkAnalyzer) Name() string {
    return "network"
}

func (a *NetworkAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    networks, err := a.client.ListNetworks(ctx)
    if err != nil {
        return nil, err
    }
    
    var findings finding.Collection
    for _, network := range networks {
        findings = append(findings, a.checkNetwork(network)...)
    }
    
    return findings, nil
}
```

**Chiamate all'SDK Docker necessarie:**
```go
// List all networks
client.NetworkList(ctx, types.NetworkListOptions{})

// Inspect specific network
client.NetworkInspect(ctx, networkID, types.NetworkInspectOptions{})
```

**Controlli da implementare:**

1. **Rilevamento del bridge predefinito:**
   - Il nome della rete è "bridge" e Driver è "bridge"
   - Finding: Raccomanda reti definite dall'utente con il flag `--network`

2. **Verifica della cifratura:**
   - Reti overlay definite dall'utente (Driver == "overlay")
   - Controllare che Options["encrypted"] != "true"
   - Finding: Abilitare la cifratura con `docker network create --opt encrypted`

3. **IPv6 abilitato:**
   - Controllare EnableIPv6 == true
   - Finding: Spesso IPv6 scavalca le regole di firewall, assicurarsi che vi siano regole appropriate

**Passaggi di implementazione:**

1. Creare il file dell'analyzer seguendo il pattern illustrato sopra
2. Aggiungerlo a `buildAnalyzers()` in scanner.go quando il target è "networks"
3. Aggiungere il target network ai flag della CLI
4. Aggiungere i controlli CIS per la sicurezza di rete (sezione 2.1)
5. Testare con: `docker network ls && docksec scan --target networks`

**Output previsto:**
```
[MEDIUM] Containers using default bridge network
  Network: bridge
  Containers: web-1, web-2, db-1
  CIS Control: 2.1
  Remediation: Create user-defined network: docker network create --driver bridge app-network
```

**Suggerimenti:**
- L'inspect della rete mostra quali container sono collegati
- Alcune reti sono gestite dal sistema (ingress, docker_gwbridge) - filtrale
- Il driver di rete determina le funzionalità (bridge vs overlay vs macvlan)

**Approfondimento:**
- Verifica la presenza di sovrapposizioni di rete con le subnet dell'host (conflitti di routing)
- Valida che la configurazione IPAM prevenga l'esaurimento degli IP
- Rileva reti esposte tramite porte pubblicate prive di regole di firewall

### Sfida 2.2: Aggiungere un Volume Security Analyzer

**Cosa costruire:**
Un analyzer che verifichi i volumi Docker alla ricerca di esposizione di dati sensibili e permessi inappropriati.

**Controlli di sicurezza:**
- Volumi anonimi (non gestiti, persistono dopo la rimozione del container)
- Volumi contenenti segreti o credenziali
- Volumi con permessi accessibili in lettura a tutti (world-readable)
- Volumi montati in molteplici container (condivisione non intenzionale)

**Perché è importante:**
I volumi Docker persistono i dati all'esterno del filesystem del container. Un container cancellato lascia dietro di sé il volume con dati potenzialmente sensibili.

Problema reale: AWS ha riscontrato oltre 2000 volumi Docker pubblicamente accessibili contenenti credenziali di database e chiavi API provenienti da container abbandonati.

**Da dove iniziare:**
Crea `internal/analyzer/volume.go`.
```go
func (a *VolumeAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    volumes, err := a.client.VolumeList(ctx, filters.Args{})
    if err != nil {
        return nil, err
    }
    
    var findings finding.Collection
    for _, vol := range volumes.Volumes {
        // Inspect volume
        // Check mountpoint permissions
        // Scan for secrets in volume data
        // Check usage (anonymous vs named)
    }
    
    return findings, nil
}
```

**Chiamate all'SDK Docker:**
```go
// List volumes
client.VolumeList(ctx, filters.Args{})

// Inspect volume (get mountpoint path)
client.VolumeInspect(ctx, volumeID)

// Find containers using volume
client.ContainerList(ctx, types.ContainerListOptions{
    Filters: filters.NewArgs(filters.Arg("volume", volumeName)),
})
```

**Controlli da implementare:**

1. **Volumi anonimi:**
   - Il nome è una stringa esadecimale (64 caratteri)
   - Nessuna label
   - Non referenziato in alcun file compose

2. **Verifiche dei permessi:**
   - Leggere il mountpoint del volume: `/var/lib/docker/volumes/<name>/_data`
   - Verificare la modalità della directory con `os.Stat()`
   - Segnalare se leggibile da tutti (mode & 0004)

3. **Scansione dei segreti:**
   - Eseguire una scansione ricorsiva dei file nel volume
   - Utilizzare l'esistente `rules.DetectSecrets()` sul contenuto dei file
   - Limitare la scansione a file di testo di dimensione inferiore a 1MB

4. **Volumi condivisi:**
   - Verificare se molteplici container montano lo stesso volume
   - Segnalare come potenziale condivisione involontaria di dati

**Passaggi di implementazione:**

1. Creare l'analyzer con l'elenco dei volumi
2. Implementare il controllo dei permessi con os.Stat
3. Aggiungere uno scanner ricorsivo di file per i segreti (attenzione: i volumi possono essere enormi)
4. Aggiungere il riferimento incrociato con i container per trovare volumi condivisi
5. Gestire gli errori di permesso in modo controllato (i volumi potrebbero non essere leggibili)

**Output previsto:**
```
[HIGH] Volume contains potential secrets
  Volume: postgres_data
  File: /var/lib/docker/volumes/postgres_data/_data/pg_hba.conf
  Secret Type: Password
  Remediation: Use Docker secrets instead of files in volumes

[MEDIUM] Anonymous volume detected
  Volume: a4f32bc8d3e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6
  Containers: none (orphaned)
  Remediation: Remove with docker volume rm or use named volumes
```

**Suggerimenti:**
- Il mountpoint del volume richiede accesso root sull'host, gestisci gli errori di permission denied
- Non scansionare file binari (controlla l'estensione del file e i magic byte)
- I nomi dei volumi anonimi sono hash SHA256 (64 caratteri esadecimali)
- Alcuni volumi sono gestiti da plugin (controlla il campo Driver)

**Approfondimento:**
- Integra con le API dei provider cloud per verificare la cifratura a riposo (at rest) dei volumi
- Verifica i driver di volume per la sicurezza (plugin locali rispetto a plugin di storage cloud)
- Rileva volumi montati da sorgenti non attendibili (NFS, CIFS)

### Sfida 2.3: Runtime Behavior Analyzer

**Cosa costruire:**
Un analyzer che monitori il comportamento a runtime dei container e rilevi anomalie.

**Cosa rilevare:**
- Processi in esecuzione come root all'interno del container
- Processi con capability impreviste (da /proc/<pid>/status)
- Connessioni di rete verso IP/porte sospetti
- Scritture di file verso percorsi sensibili

**Perché è importante:**
L'analisi statica intercetta problemi in fase di build. L'analisi a runtime rileva lo sfruttamento attivo di vulnerabilità (active exploitation).

Caso d'uso reale: strumenti di sicurezza a runtime per Kubernetes come Falco rilevano i crypto miner monitorando processi con nomi come "xmrig" o picchi improvvisi nell'uso di CPU.

**Da dove iniziare:**
Questo esercizio è avanzato. Dovrai:
1. Elencare i processi del container tramite il filesystem `/proc`
2. Leggere le capability dei processi da `/proc/<pid>/status`
3. Monitorare le connessioni di rete da `/proc/net/tcp`
4. Tracciare le operazioni sui file (richiede inotify o BPF)

**Implementazione:**
```go
func (a *RuntimeAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    containers, _ := a.client.ListContainers(ctx, true)
    
    var findings finding.Collection
    for _, c := range containers {
        // Get container PID namespace
        inspect, _ := a.client.InspectContainer(ctx, c.ID)
        pid := inspect.State.Pid
        
        // Read /proc/<pid>/status for capabilities
        // Read /proc/<pid>/net/tcp for network connections
        // Check process user (UID 0 = root)
    }
    
    return findings, nil
}
```

**Sotto-sfide all'interno della sfida:**

1. **Enumerazione dei processi:**
   - I processi del container si trovano nel PID namespace dell'host
   - Trova il root PID del container dall'inspect
   - Esplora `/proc/<pid>/task/` per i thread

2. **Parsing delle capability:**
   - `/proc/<pid>/status` contiene la riga CapEff (maschera di bit esadecimale)
   - Converti l'esadecimale nei nomi delle capability usando l'esistente mappa rules.Capabilities
   - Segnala capability impreviste (SYS_ADMIN in un web server?)

3. **Monitoraggio di rete:**
   - `/proc/<pid>/net/tcp` mostra le connessioni aperte
   - Esegui il parsing di indirizzi IP e porte in formato esadecimale
   - Segnala connessioni verso IP noti per essere malevoli o porte inusuali

**Suggerimenti:**
- Dovrai eseguire exec all'interno del container o leggere il `/proc` dell'host
- Il PID del container in inspect è il PID dell'host, non il PID 1 del container
- La maschera delle capability è un valore esadecimale a 64 bit, esegui il parsing con strconv.ParseUint
- Il monitoraggio di rete genera race condition (le connessioni sono transitorie)

**Approfondimento:**
- Usa BPF/eBPF per monitorare le syscall a basso overhead
- Integra con feed di threat intelligence per la reputazione degli indirizzi IP
- Rileva tentativi di container escape (syscall sospette)
- Monitora leak di file descriptor ed esaurimento delle risorse

## Livello 3: Funzionalità Avanzate

Queste sfide aggiungono funzionalità significative e richiedono modifiche architetturali.

### Sfida 3.1: Aggiungere Script di Remediation

**Cosa costruire:**
Generare automaticamente script di shell per correggere i problemi rilevati.

**Esempio:**
Finding: "Container running with --privileged"
Script generato:
```bash
#!/bin/bash
# Fix for container 'web-1': Remove privileged flag

# Stop container
docker stop web-1

# Get current run command
docker inspect web-1 --format='{{.Config.Cmd}}'

# Recreate without --privileged
docker run -d \
  --name web-1 \
  --network bridge \
  --volume /app:/app \
  nginx:1.21.3

docker rm web-1  # Remove old container
```

**Perché è importante:**
Mostrare il problema è utile. Mostrare come risolverlo è ancora meglio. Gli script generati automaticamente riducono il tempo che intercorre tra la rilevazione e la remediation.

**Da dove iniziare:**
Aggiungi il metodo `GenerateRemediation()` a ciascun tipo di finding.

File: `internal/finding/finding.go`
```go
type Finding struct {
    // ... existing fields
    RemediationScript string  // Add this field
}

func (f *Finding) GenerateScript() string {
    switch f.RuleID {
    case "CIS-5.4":  // Privileged container
        return generatePrivilegedFix(f.Target)
    case "CIS-5.3":  // Dangerous capability
        return generateCapabilityFix(f.Target, extractCapability(f.Title))
    // ... other cases
    }
    return ""
}
```

**Implementazione:**

1. Estrarre la configurazione del container da Docker inspect
2. Costruire un nuovo comando docker run privo della vulnerabilità di sicurezza
3. Includere i passaggi per eseguire backup/ripristino dei dati se necessario
4. Aggiungere verifiche di validazione prima dell'esecuzione

**Per container privilegiati:**
```go
func generatePrivilegedFix(target finding.Target) string {
    // Get current container config
    // Remove --privileged flag
    // Regenerate docker run command
    // Include capability adds if needed
}
```

**Per problemi di capability:**
```go
func generateCapabilityFix(target finding.Target, cap string) string {
    // Get current caps
    // Remove dangerous cap
    // Suggest alternatives (SYS_ADMIN -> specific caps)
}
```

**Formato di output:**
```bash
# Generated by docksec
# WARNING: Review before executing

# Backup container volumes
docker run --rm --volumes-from web-1 -v $(pwd):/backup alpine tar czf /backup/web-1-volumes.tar.gz /data

# Stop and remove old container
docker stop web-1
docker rm web-1

# Recreate with security fixes
docker run -d \
  --name web-1 \
  --memory=512m \
  --cpu-shares=1024 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --network=app-network \
  -v /app/data:/data \
  nginx:1.21.3

echo "Container recreated. Verify functionality before deleting backup."
```

**Suggerimenti:**
- Recupera il comando originale tramite inspect del container (Config.Cmd, HostConfig)
- Preserva variabili d'ambiente e label
- Gestisci i mount dei volumi con cautela (rischio di perdita dati)
- Testa gli script generati prima in ambienti non di produzione

**Approfondimento:**
- Supporta la rigenerazione di docker-compose per i container gestiti da compose
- Aggiungi script di rollback (per ripristinare la configurazione iniziale)
- Valida gli script con shellcheck prima di produrli in output
- Supporta il patching di file YAML di Kubernetes per i deployment su K8s

### Sfida 3.2: Motore di Policy-as-Code

**Cosa costruire:**
Consentire agli utenti di definire policy di sicurezza personalizzate in YAML e applicarle.

**Esempio di policy:**
```yaml
# security-policy.yaml
policies:
  - id: company-baseline
    name: "Company Security Baseline"
    rules:
      - check: no-privileged
        severity: critical
        
      - check: require-user
        severity: high
        exclude:
          - images: ["mysql:*", "postgres:*"]  # DB containers need root
        
      - check: memory-limit
        severity: medium
        parameters:
          minimum: 128Mi
          maximum: 2Gi
          
      - check: allowed-registries
        severity: high
        parameters:
          registries:
            - "registry.company.com"
            - "gcr.io/company-*"
```

**Perché è importante:**
Ambienti diversi hanno requisiti diversi. L'ambiente di sviluppo tollera tag :latest, la produzione no. Le policy codificano queste regole sotto forma di codice.

Utilizzo reale: Netflix Titus impone policy mediante admission controller. Ha respinto il 40% delle richieste iniziali di container a causa di violazioni di policy.

**Da dove iniziare:**
Crea il package `internal/policy/`.
```go
// internal/policy/policy.go
type Policy struct {
    ID    string
    Name  string
    Rules []Rule
}

type Rule struct {
    Check      string
    Severity   finding.Severity
    Parameters map[string]interface{}
    Exclude    *ExcludeConfig
}

type ExcludeConfig struct {
    Images     []string
    Containers []string
    Namespaces []string
}

func LoadPolicy(path string) (*Policy, error) {
    // Parse YAML file
}

func (p *Policy) Evaluate(findings finding.Collection) (finding.Collection, error) {
    // Filter findings based on policy rules
    // Apply exclusions
    // Override severities
}
```

**Passaggi di implementazione:**

1. Definire lo schema delle policy in struct Go
2. Utilizzare gopkg.in/yaml.v3 per il parsing
3. Aggiungere un livello di valutazione delle policy tra scanner e reporter
4. Supportare il pattern matching per i nomi delle immagini (pattern glob)
5. Aggiungere la validazione delle policy (intercettare nomi di controlli non validi)

**Logica di valutazione delle policy:**
```go
func (r *Rule) Matches(f *finding.Finding) bool {
    // Check if finding matches rule check type
    if !r.matchesCheck(f.RuleID) {
        return false
    }
    
    // Apply exclusions
    if r.Exclude != nil && r.Exclude.Matches(f.Target) {
        return false
    }
    
    // Check parameters (memory limits, etc.)
    return r.matchesParameters(f)
}
```

**Matching delle esclusioni:**
```go
func (e *ExcludeConfig) Matches(target finding.Target) bool {
    for _, pattern := range e.Images {
        if matched, _ := filepath.Match(pattern, target.Name); matched {
            return true
        }
    }
    return false
}
```

**Integrazione con lo scanner:**
```go
// internal/scanner/scanner.go
func (s *Scanner) Scan(ctx context.Context) error {
    findings, _ := s.runAnalyzers(ctx, analyzers)
    
    // Apply policy if configured
    if s.cfg.PolicyFile != "" {
        policy, _ := policy.LoadPolicy(s.cfg.PolicyFile)
        findings, _ = policy.Evaluate(findings)
    }
    
    filtered := s.filterFindings(findings)
    s.reporter.Report(filtered)
}
```

**Output previsto:**
```
Loading policy: security-policy.yaml
Policy: Company Security Baseline (company-baseline)
Evaluating 147 findings against 5 rules...

Excluded 12 findings:
  - mysql:8.0 (database images allowed to run as root)
  - postgres:13 (database images allowed to run as root)

Severity overrides applied: 3
  - CIS-5.27: LOW → MEDIUM (company policy)

Final results: 132 findings
  CRITICAL: 5
  HIGH: 23
  MEDIUM: 67
  LOW: 37
```

**Suggerimenti:**
- Usa yaml.v3 per il parsing (messaggi di errore migliori rispetto a v2)
- Valida la policy al momento del caricamento (i controlli sconosciuti devono dare errore)
- Supporta policy multiple (a livelli: baseline + specifica per l'ambiente)
- Esegui il caching dei risultati di valutazione delle policy (lo stesso finding viene controllato più volte)

**Approfondimento:**
- Supporta l'ereditarietà delle policy (estensione di una policy base)
- Aggiungi un framework di testing delle policy (test-policy.yaml)
- Implementa il versioning e la migrazione delle policy
- Crea una libreria di policy con standard comuni (PCI-DSS, SOC2, ecc.)

### Sfida 3.3: Modalità di Monitoraggio Continuo

**Cosa costruire:**
Un demone a lunga esecuzione che monitori gli eventi Docker ed esegua scansioni in tempo reale.

**Funzionalità:**
- Monitorare il flusso di eventi Docker per l'avvio/arresto dei container
- Scansionare automaticamente i nuovi container entro pochi secondi dalla creazione
- Inviare avvisi in caso di violazioni di sicurezza (webhook, Slack, PagerDuty)
- Mantenere lo stato dei container noti come conformi rispetto a quelli segnalati

**Perché è importante:**
Le scansioni pianificate vengono eseguite a cadenza oraria. Il monitoraggio in tempo reale rileva i problemi in pochi secondi.

Scenario reale: uno sviluppatore lancia `docker run --privileged` per fare debug. Senza monitoraggio, te ne accorgi alla scansione successiva (1 ora). Con il monitoraggio, l'alert scatta in 5 secondi.

**Architettura:**
```
Docker Event Stream
        ↓
Event Processor → Analyzer Queue → Scanner
        ↓
    Finding Store → Alert Router
        ↓
    Webhooks/Slack/etc
```

**Da dove iniziare:**
Crea `cmd/docksec/daemon.go` per il comando daemon.
```go
func newDaemonCmd(cfg *config.Config) *cobra.Command {
    cmd := &cobra.Command{
        Use:   "daemon",
        Short: "Run continuous monitoring daemon",
        RunE: func(cmd *cobra.Command, args []string) error {
            return runDaemon(cmd.Context(), cfg)
        },
    }
    
    flags := cmd.Flags()
    flags.StringVar(&cfg.WebhookURL, "webhook", "", "Webhook URL for alerts")
    flags.DurationVar(&cfg.ScanInterval, "interval", 10*time.Second, "Scan interval")
    
    return cmd
}
```

**Elaborazione degli eventi:**
```go
// internal/daemon/daemon.go
func (d *Daemon) watchEvents(ctx context.Context) error {
    events, errs := d.client.Events(ctx, types.EventsOptions{
        Filters: filters.NewArgs(
            filters.Arg("type", "container"),
            filters.Arg("event", "start"),
            filters.Arg("event", "stop"),
        ),
    })
    
    for {
        select {
        case event := <-events:
            d.handleEvent(event)
        case err := <-errs:
            return err
        case <-ctx.Done():
            return ctx.Err()
        }
    }
}

func (d *Daemon) handleEvent(event events.Message) {
    switch event.Action {
    case "start":
        // Queue container for scanning
        d.scanQueue <- event.Actor.ID
    case "stop":
        // Remove from active tracking
        d.tracker.Remove(event.Actor.ID)
    }
}
```

**Coda di scansione:**
```go
func (d *Daemon) processScanQueue(ctx context.Context) {
    for {
        select {
        case containerID := <-d.scanQueue:
            // Scan single container
            findings := d.scanContainer(ctx, containerID)
            
            // Compare with baseline
            if d.hasNewFindings(containerID, findings) {
                d.sendAlert(containerID, findings)
            }
            
            // Update baseline
            d.baseline[containerID] = findings
            
        case <-ctx.Done():
            return
        }
    }
}
```

**Invio degli alert:**
```go
func (d *Daemon) sendAlert(containerID string, findings finding.Collection) {
    critical := findings.FilterBySeverity(finding.SeverityCritical)
    if len(critical) == 0 {
        return
    }
    
    payload := map[string]interface{}{
        "container": containerID,
        "findings":  critical,
        "timestamp": time.Now(),
    }
    
    // Send to webhook
    d.webhook.Send(payload)
    
    // Send to Slack
    d.slack.Send(formatSlackMessage(payload))
}
```

**Gestione dello stato:**
```go
// Track baseline findings per container
type FindingBaseline struct {
    mu        sync.RWMutex
    baselines map[string]finding.Collection  // containerID -> findings
}

func (fb *FindingBaseline) Update(containerID string, findings finding.Collection) {
    fb.mu.Lock()
    defer fb.mu.Unlock()
    fb.baselines[containerID] = findings
}

func (fb *FindingBaseline) HasChanged(containerID string, findings finding.Collection) bool {
    fb.mu.RLock()
    defer fb.mu.RUnlock()
    
    baseline, exists := fb.baselines[containerID]
    if !exists {
        return true  // New container
    }
    
    return !finding.Equal(baseline, findings)
}
```

**Passaggi di implementazione:**

1. Creare il comando daemon in cmd/docksec/daemon.go
2. Implementare l'ascolto del flusso di eventi
3. Aggiungere la coda di scansione con rate limiting
4. Costruire la logica di confronto dei finding
5. Implementare gli alert via webhook
6. Aggiungere la gestione del graceful shutdown

**Output previsto:**
```bash
$ docksec daemon --webhook=https://hooks.slack.com/... --interval=10s

Starting docksec daemon...
Watching Docker events...
Baseline: 127 containers scanned

[2025-01-31 10:23:45] Container started: web-prod-3
[2025-01-31 10:23:50] Scan complete: 3 findings (1 CRITICAL)
[2025-01-31 10:23:50] ⚠️  Alert sent: Privileged container detected

[2025-01-31 10:25:12] Container stopped: worker-7
[2025-01-31 10:25:12] Removed from tracking
```

**Formato dell'alert su Slack:**
```
🚨 Critical Security Finding

Container: web-prod-3
Image: nginx:latest
Finding: Container running with --privileged flag
Severity: CRITICAL
CIS Control: 5.4

Started: 2025-01-31 10:23:45
Scanned: 2025-01-31 10:23:50

Remediation: Recreate container without --privileged flag
```

**Suggerimenti:**
- L'Events API di Docker può perdere eventi se l'elaborazione è lenta (usa un canale con buffer)
- Applica il rate limit alle scansioni per non sovraccaricare il demone
- Rendi persistente la baseline su disco (per resistere al riavvio del demone)
- Gestisci gli eventi di rinomina dei container (l'ID rimane identico, il nome cambia)

**Approfondimento:**
- Aggiungi un exporter di metriche Prometheus (endpoint /metrics)
- Implementa la deduplicazione degli alert (non inviare notifiche ripetute dello stesso alert)
- Supporta regole di instradamento degli alert (critico → PagerDuty, basso → email)
- Aggiungi una UI web per mostrare lo stato dei container in tempo reale
- Integra con sistemi SIEM (Splunk, Elasticsearch)

## Livello 4: Idoneità alla Produzione (Production Readiness)

Queste sfide rendono docksec di livello enterprise per ambienti di produzione.

### Sfida 4.1: Ottimizzazione delle Prestazioni

**Cosa costruire:**
Fare in modo che lo scanner gestisca oltre 10.000 container senza andare in timeout né consumare memoria eccessiva.

**Colli di bottiglia attuali:**

1. **Ricerche sequenziali dei controlli CIS:**
   `benchmark.Get("5.4")` chiamato per ogni singolo finding
   
2. **Operazioni su stringhe nel percorso critico (hot path):**
   `strings.ToUpper()` su ciascun controllo di capability
   
3. **Crescita incontrollata della memoria:**
   Tutti i finding vengono mantenuti in memoria fino alla fine

**Ottimizzazione 1: Precalcolo delle ricerche dei controlli**

Prima:
```go
func (a *ContainerAnalyzer) checkPrivileged(...) finding.Collection {
    control, _ := benchmark.Get("5.4")  // Map lookup every call
    f := finding.New("CIS-5.4", control.Title, ...)
}
```

Dopo:
```go
var (
    controlPrivileged = benchmark.MustGet("5.4")  // Lookup once at init
    controlCapabilities = benchmark.MustGet("5.3")
    controlMounts = benchmark.MustGet("5.5")
)

func (a *ContainerAnalyzer) checkPrivileged(...) finding.Collection {
    f := finding.New("CIS-5.4", controlPrivileged.Title, ...)
}
```

**Ottimizzazione 2: Generazione chiavi per la mappa delle capability**

Prima:
```go
cap = strings.ToUpper(string(cap))  // Allocates new string
capInfo, exists := rules.GetCapabilityInfo(cap)
```

Dopo:
```go
// Build lookup with both cases at init
var capabilityLookup = buildCapabilityLookup()

func buildCapabilityLookup() map[string]CapabilityInfo {
    m := make(map[string]CapabilityInfo, len(Capabilities)*2)
    for cap, info := range Capabilities {
        m[cap] = info
        m[strings.ToLower(cap)] = info  // Support lowercase
        m[strings.TrimPrefix(cap, "CAP_")] = info  // Without prefix
    }
    return m
}

// Now direct lookup without string operations
capInfo, exists := capabilityLookup[string(cap)]
```

**Ottimizzazione 3: Output in streaming**

Prima:
```go
// Collect all findings
findings = append(findings, ...)

// Write at end
reporter.Report(findings)
```

Dopo:
```go
// Stream findings as discovered
findingsChan := make(chan finding.Finding, 100)

go func() {
    for f := range findingsChan {
        reporter.ReportOne(f)  // Write immediately
    }
}()

// Analyzer sends to channel
findingsChan <- f
```

**Risultati del benchmark:**
```
Before optimizations:
  1000 containers: 12.3s, 847MB
  10000 containers: 183s, 8.4GB

After optimizations:
  1000 containers: 3.1s, 124MB
  10000 containers: 35s, 980MB

Improvement: 5.2x faster, 8.6x less memory
```

**Passaggi di implementazione:**

1. Eseguire il profiling con `go test -cpuprofile=cpu.prof -memprofile=mem.prof`
2. Analizzare con `go tool pprof cpu.prof`
3. Identificare i percorsi critici (verifica delle capability, lookup dei controlli)
4. Precalcolare i lookup all'init del package
5. Aggiungere l'opzione di output in streaming ai reporter
6. Ripetere il benchmark e confrontare

**Suggerimenti:**
- Usa `benchmark_test.go` per misurare i miglioramenti
- pprof mostra esattamente le funzioni che consumano CPU/memoria
- Ottimizzare prematuramente è una cattiva pratica, ma questi sono colli di bottiglia misurati
- Baratta la memoria a favore della velocità (mappe precalcolate) quando la mappa ha dimensioni ridotte

### Sfida 4.2: Test Suite Completa

**Cosa costruire:**
Raggiungere oltre l'80% di code coverage con test significativi, non solo per coprire linee di codice.

**Categorie di test:**

1. **Test unitari:** Singole funzioni in isolamento
2. **Test di integrazione:** Componenti che lavorano congiuntamente
3. **Test end-to-end:** Flusso di scansione completo
4. **Fuzzing:** Gestione di input casuali

**Esempio di test unitario:**

File: `internal/rules/capabilities_test.go`
```go
func TestCapabilityRiskLevels(t *testing.T) {
    tests := []struct {
        capability string
        minSeverity finding.Severity
    }{
        {"CAP_SYS_ADMIN", finding.SeverityCritical},
        {"CAP_SYS_PTRACE", finding.SeverityCritical},
        {"CAP_NET_ADMIN", finding.SeverityHigh},
        {"CAP_NET_BIND_SERVICE", finding.SeverityLow},
    }
    
    for _, tt := range tests {
        t.Run(tt.capability, func(t *testing.T) {
            info, exists := GetCapabilityInfo(tt.capability)
            if !exists {
                t.Fatalf("capability %s not found", tt.capability)
            }
            if info.Severity < tt.minSeverity {
                t.Errorf("severity = %v, want >= %v", 
                    info.Severity, tt.minSeverity)
            }
        })
    }
}
```

**Esempio di test di integrazione:**

File: `internal/analyzer/container_test.go`
```go
func TestContainerAnalyzer_PrivilegedDetection(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    // Start test Docker container
    ctx := context.Background()
    client, _ := docker.NewClient()
    
    containerID, cleanup := createPrivilegedContainer(t, ctx, client)
    defer cleanup()
    
    // Run analyzer
    analyzer := NewContainerAnalyzer(client)
    findings, err := analyzer.Analyze(ctx)
    
    if err != nil {
        t.Fatalf("Analyze() error = %v", err)
    }
    
    // Verify finding
    var found bool
    for _, f := range findings {
        if f.RuleID == "CIS-5.4" && f.Target.ID == containerID {
            found = true
            if f.Severity != finding.SeverityCritical {
                t.Errorf("severity = %v, want CRITICAL", f.Severity)
            }
        }
    }
    
    if !found {
        t.Error("privileged container not detected")
    }
}

func createPrivilegedContainer(t *testing.T, ctx context.Context, client *docker.Client) (string, func()) {
    resp, err := client.CreateContainer(ctx, &container.Config{
        Image: "alpine:latest",
        Cmd:   []string{"sleep", "3600"},
    }, &container.HostConfig{
        Privileged: true,
    }, nil, nil, "")
    
    if err != nil {
        t.Fatalf("create container: %v", err)
    }
    
    client.StartContainer(ctx, resp.ID, types.ContainerStartOptions{})
    
    cleanup := func() {
        client.StopContainer(ctx, resp.ID, 1)
        client.RemoveContainer(ctx, resp.ID, types.ContainerRemoveOptions{Force: true})
    }
    
    return resp.ID, cleanup
}
```

**Esempio di test end-to-end:**

File: `cmd/docksec/main_test.go`
```go
func TestFullScanWorkflow(t *testing.T) {
    // Setup test environment
    client, _ := docker.NewClient()
    ctx := context.Background()
    
    // Create containers with known issues
    privilegedID, _ := createPrivilegedContainer(t, ctx, client)
    noUserID, _ := createRootUserContainer(t, ctx, client)
    defer cleanupContainers(t, ctx, client, privilegedID, noUserID)
    
    // Run scan
    output := runDocksec(t, []string{"scan", "--target", "containers", "--output", "json"})
    
    // Parse JSON output
    var result struct {
        Findings []finding.Finding `json:"findings"`
    }
    json.Unmarshal([]byte(output), &result)
    
    // Verify expected findings
    if len(result.Findings) < 2 {
        t.Errorf("got %d findings, want at least 2", len(result.Findings))
    }
    
    hasPrivileged := false
    hasRootUser := false
    for _, f := range result.Findings {
        if f.RuleID == "CIS-5.4" {
            hasPrivileged = true
        }
        if f.RuleID == "CIS-4.1" {
            hasRootUser = true
        }
    }
    
    if !hasPrivileged {
        t.Error("privileged container not detected")
    }
    if !hasRootUser {
        t.Error("root user not detected")
    }
}

func runDocksec(t *testing.T, args []string) string {
    cmd := exec.Command("./docksec", args...)
    output, err := cmd.CombinedOutput()
    if err != nil {
        t.Fatalf("docksec failed: %v\n%s", err, output)
    }
    return string(output)
}
```

**Esempio di test con fuzzing:**

File: `internal/parser/dockerfile_test.go`
```go
func FuzzDockerfileParsing(f *testing.F) {
    // Seed corpus with valid Dockerfiles
    f.Add("FROM alpine\nRUN echo hello")
    f.Add("FROM scratch\nCOPY binary /")
    f.Add("FROM ubuntu:20.04\nUSER nobody")
    
    f.Fuzz(func(t *testing.T, input string) {
        // Should never panic
        result, err := parser.Parse(strings.NewReader(input))
        
        if err != nil {
            return  // Invalid syntax is fine
        }
        
        // If parsed successfully, AST should be valid
        if result.AST == nil {
            t.Error("nil AST with no error")
        }
    })
}
```

Esecuzione del fuzzing: `go test -fuzz=FuzzDockerfileParsing -fuzztime=1m`

**Misurazione della copertura:**
```bash
# Generate coverage
go test -coverprofile=coverage.out ./...

# View in browser
go tool cover -html=coverage.out

# Check percentage
go tool cover -func=coverage.out | grep total
```

**Copertura target:**
- Package rules: 90% (logica pura, facile da testare)
- Analyzer: 75% (integrazione Docker, alcuni percorsi difficili da testare)
- Scanner: 80% (logica di orchestrazione)
- Reporter: 85% (formattazione dell'output)
- Complessiva: 80%

**Passaggi di implementazione:**

1. Aggiungere test unitari per tutti i package delle regole
2. Aggiungere test di integrazione che richiedono Docker (usare build tag)
3. Aggiungere test e2e nel package cmd/
4. Aggiungere fuzzing per i parser
5. Configurare la CI affinché fallisca se la copertura scende sotto il 75%

**Suggerimenti:**
- Usa test table-driven (più facili per aggiungere casi di test)
- Esegui il mock del client Docker per i test unitari (testify/mock)
- Usa i build tag per separare i test di integrazione: `// +build integration`
- File golden per gli output attesi (terminal reporter)

### Sfida 4.3: Supporto a Kubernetes

**Cosa costruire:**
Estendere lo scanner per operare sui pod Kubernetes, analizzando i container tramite le API di K8s anziché tramite l'API di Docker.

**Perché è importante:**
I deployment di produzione utilizzano Kubernetes. È necessario scansionare i pod, verificare i Pod Security Standards, validare i contesti di sicurezza (security context).

**Controlli specifici per Kubernetes:**

1. Pod in esecuzione come privilegiato
2. hostNetwork, hostPID, hostIPC abilitati
3. Capability aggiunte oltre i valori predefiniti
4. securityContext assente o permissivo
5. Service account con permessi eccessivi
6. Violazioni dei Pod Security Standards

**Da dove iniziare:**
Crea il package `internal/k8s/` con un wrapper per il client Kubernetes.
```go
// internal/k8s/client.go
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

type Client struct {
    clientset *kubernetes.Clientset
}

func NewClient() (*Client, error) {
    // In-cluster config (when running as pod)
    config, err := rest.InClusterConfig()
    if err != nil {
        // Fallback to kubeconfig
        config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
    }
    
    clientset, err := kubernetes.NewForConfig(config)
    return &Client{clientset: clientset}, nil
}
```

**Creare l'analyzer per Kubernetes:**

File: `internal/analyzer/kubernetes.go`
```go
type KubernetesAnalyzer struct {
    client *k8s.Client
}

func (a *KubernetesAnalyzer) Analyze(ctx context.Context) (finding.Collection, error) {
    pods, err := a.client.ListPods(ctx, metav1.ListOptions{})
    if err != nil {
        return nil, err
    }
    
    var findings finding.Collection
    for _, pod := range pods.Items {
        findings = append(findings, a.analyzePod(pod)...)
    }
    
    return findings, nil
}

func (a *KubernetesAnalyzer) analyzePod(pod v1.Pod) finding.Collection {
    var findings finding.Collection
    
    target := finding.Target{
        Type: finding.TargetPod,
        Name: pod.Namespace + "/" + pod.Name,
        ID:   string(pod.UID),
    }
    
    // Check pod-level security context
    if pod.Spec.HostNetwork {
        f := finding.New("K8S-PSS-BASELINE", "Pod uses host network", 
            finding.SeverityHigh, target).
            WithDescription("hostNetwork: true gives pod access to host's network stack.")
        findings = append(findings, f)
    }
    
    // Check each container
    for _, container := range pod.Spec.Containers {
        findings = append(findings, a.analyzeContainer(target, container)...)
    }
    
    return findings
}

func (a *KubernetesAnalyzer) analyzeContainer(
    podTarget finding.Target, 
    container v1.Container,
) finding.Collection {
    var findings finding.Collection
    
    // Check security context
    if container.SecurityContext != nil {
        if container.SecurityContext.Privileged != nil && 
            *container.SecurityContext.Privileged {
            f := finding.New("K8S-PRIVILEGED", "Privileged container in pod", 
                finding.SeverityCritical, podTarget).
                WithDescription("Container " + container.Name + " runs as privileged.")
            findings = append(findings, f)
        }
        
        // Check capabilities
        if container.SecurityContext.Capabilities != nil {
            for _, cap := range container.SecurityContext.Capabilities.Add {
                capInfo, exists := rules.GetCapabilityInfo(string(cap))
                if exists && capInfo.Severity >= finding.SeverityHigh {
                    f := finding.New("K8S-CAP-ADD", "Dangerous capability added",
                        capInfo.Severity, podTarget).
                        WithDescription("Container " + container.Name + 
                            " adds capability " + string(cap))
                    findings = append(findings, f)
                }
            }
        }
    }
    
    // Check if running as root
    if container.SecurityContext == nil || 
        container.SecurityContext.RunAsNonRoot == nil ||
        !*container.SecurityContext.RunAsNonRoot {
        f := finding.New("K8S-ROOT-USER", "Container may run as root",
            finding.SeverityMedium, podTarget).
            WithDescription("Container " + container.Name + 
                " does not enforce non-root user.")
        findings = append(findings, f)
    }
    
    return findings
}
```

**Mappatura dei Pod Security Standards:**
```go
// internal/k8s/pss.go
type PodSecurityStandard string

const (
    PSSPrivileged PodSecurityStandard = "privileged"
    PSSBaseline   PodSecurityStandard = "baseline"
    PSSRestricted PodSecurityStandard = "restricted"
)

func EvaluatePSS(pod v1.Pod) (PodSecurityStandard, []string) {
    violations := []string{}
    
    // Privileged: no restrictions
    // Baseline: minimal restrictions
    // Restricted: hardened configuration
    
    // Check baseline violations
    if pod.Spec.HostNetwork {
        violations = append(violations, "hostNetwork must be false")
    }
    if pod.Spec.HostPID {
        violations = append(violations, "hostPID must be false")
    }
    if pod.Spec.HostIPC {
        violations = append(violations, "hostIPC must be false")
    }
    
    // If baseline violated, return baseline
    if len(violations) > 0 {
        return PSSBaseline, violations
    }
    
    // Check restricted violations
    for _, container := range pod.Spec.Containers {
        if container.SecurityContext == nil {
            violations = append(violations, container.Name + ": missing securityContext")
            continue
        }
        
        if container.SecurityContext.AllowPrivilegeEscalation == nil ||
            *container.SecurityContext.AllowPrivilegeEscalation {
            violations = append(violations, container.Name + 
                ": allowPrivilegeEscalation must be false")
        }
        
        if container.SecurityContext.RunAsNonRoot == nil ||
            !*container.SecurityContext.RunAsNonRoot {
            violations = append(violations, container.Name + 
                ": runAsNonRoot must be true")
        }
        
        if container.SecurityContext.SeccompProfile == nil ||
            container.SecurityContext.SeccompProfile.Type != v1.SeccompProfileTypeRuntimeDefault {
            violations = append(violations, container.Name + 
                ": seccompProfile must be RuntimeDefault")
        }
    }
    
    if len(violations) > 0 {
        return PSSRestricted, violations
    }
    
    return PSSRestricted, nil
}
```

**Integrazione nella CLI:**
```go
// cmd/docksec/main.go
func newScanCmd(cfg *config.Config) *cobra.Command {
    cmd := &cobra.Command{
        Use:   "scan",
        Short: "Scan containers for security issues",
        RunE:  runScan,
    }
    
    flags := cmd.Flags()
    flags.StringSliceVarP(&cfg.Targets, "target", "t", []string{"all"},
        "Scan targets: docker, kubernetes, all")
    flags.BoolVar(&cfg.K8sEnabled, "k8s", false, "Enable Kubernetes scanning")
    flags.StringVar(&cfg.Kubeconfig, "kubeconfig", "", "Path to kubeconfig")
    
    return cmd
}
```

**Esempio di output:**
```
Scanning Kubernetes cluster...
Namespace: default

[CRITICAL] Privileged container in pod
  Pod: default/nginx-deployment-7d6c4f8b9-x7k2m
  Container: nginx
  CIS Control: K8S-PRIVILEGED
  Remediation: Remove privileged: true from securityContext

[HIGH] Pod uses host network
  Pod: default/monitoring-agent-dw8qz
  PSS Violation: Baseline
  Remediation: Set hostNetwork: false

[MEDIUM] Container may run as root
  Pod: default/web-app-5b8c9d6f4-mz3lp
  Container: app
  PSS Violation: Restricted
  Remediation: Set securityContext.runAsNonRoot: true

Summary:
  Pods scanned: 47
  PSS Restricted: 12
  PSS Baseline: 23
  PSS Privileged: 12
  Total findings: 89
```

**Passaggi di implementazione:**

1. Aggiungere la dipendenza k8s.io/client-go
2. Creare il wrapper del client k8s
3. Implementare l'analyzer per Kubernetes
4. Aggiungere la logica di valutazione dei PSS
5. Estendere il tipo Target per i pod
6. Aggiungere controlli CIS specifici per K8s

**Suggerimenti:**
- Usa k8s.io/client-go v0.28.0 o successiva
- Gestisci l'autenticazione sia in-cluster che tramite kubeconfig
- Il contesto di sicurezza del pod è distinto da quello del container
- Alcuni controlli si applicano al pod, altri ai singoli container
- Attenzione ai dereferenziamenti di puntatori nil (molti campi di K8s sono puntatori)

**Approfondimento:**
- Aggiungi l'analisi delle NetworkPolicy (verifica di regole default-deny)
- Scansiona le configurazioni di PodSecurityPolicy/PodSecurityAdmission
- Controlla i permessi RBAC (service account con privilegi eccessivi)
- Valida le configurazioni degli admission controller
- Scansiona gli Helm chart prima del deployment

## Sfide Bonus

### Bonus 1: Integrazione della Scansione CVE

Integrare con Trivy o Grype per aggiungere la scansione delle vulnerabilità.

### Bonus 2: Generatore di Report di Conformità

Generare report di conformità per standard come SOC2, PCI-DSS e HIPAA in base ai finding rilevati.

### Bonus 3: Rilevamento delle Anomalie con Machine Learning

Utilizzare tecniche di ML per rilevare container che manifestano comportamenti anomali rispetto ai pattern storici.

### Bonus 4: Analyzer del Grafo di Docker Compose

Costruire un grafo delle dipendenze a partire dai file compose, rilevando dipendenze circolari e configurazioni di rete eccessivamente permissive.

### Bonus 5: Estensione per il Browser

Creare un'estensione per il browser che esegua docksec durante la visualizzazione di Dockerfile su GitHub.

## Ottenere Aiuto

**Bloccato su una sfida?**

1. Controlla il codice esistente per individuare pattern simili
2. Leggi la documentazione dell'SDK Docker: https://pkg.go.dev/github.com/docker/docker
3. Consulta il CIS Docker Benchmark: https://www.cisecurity.org/benchmark/docker
4. Esamina strumenti analoghi: il codice sorgente di Docker Bench, Trivy, Falco

**Hai trovato un bug durante l'implementazione?**

Fa parte del processo di apprendimento. Esegui il debug:
- Aggiungi istruzioni di log nell'analyzer
- Esegui con il flag --verbose
- Usa `docker inspect` per verificare i valori attesi
- Controlla i log del demone Docker: `journalctl -u docker`

**Vuoi contribuire con la tua soluzione?**

Scrivi codice pulito, aggiungi test, aggiorna la documentazione. Queste sfide rappresentano ottimi progetti per il proprio portfolio.
