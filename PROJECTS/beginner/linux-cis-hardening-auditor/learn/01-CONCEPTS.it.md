<!-- © AngelaMos | 2026 | 01-CONCEPTS.md -->

# Concetti fondamentali di sicurezza

Questo documento tratta i fondamenti di sicurezza alla base dei benchmark CIS, dell'hardening dei sistemi e dell'audit di conformità. Alla fine dovresti capire perché esiste ciascuna delle sei sezioni dell'audit, quali attacchi reali prevengono e come i controlli si mappano sui framework di compliance che incontrerai negli ambienti di produzione. Il tempo di lettura è di circa 20-25 minuti.

---

## Benchmark CIS

### Cosa sono

Il Center for Internet Security (CIS) pubblica guide di hardening chiamate benchmark per sistemi operativi, piattaforme cloud, database e applicazioni. Ogni benchmark è un documento PDF che contiene centinaia di controlli prescrittivi, organizzati in sezioni, con un titolo, una motivazione, una procedura di audit e un comando di remediation per ognuno di essi.

Questo progetto implementa il **CIS Debian Linux 12 Benchmark v1.1.0**, che contiene 104 controlli distribuiti su sei sezioni. Ogni controllo è di Level 1 (da applicare a ogni sistema) oppure di Level 2 (per ambienti ad alta sicurezza in cui l'hardening potrebbe compromettere alcune funzionalità). Ogni controllo è inoltre scored (concorre alla percentuale di conformità) oppure unscored (solo a titolo consultivo).

Il benchmark non inventa nuove idee di sicurezza. Codifica le lezioni che i team di sicurezza hanno imparato da decenni di violazioni, incident response e penetration testing. Quando il benchmark dice "disabilita l'IP forwarding", è perché l'IP forwarding su un host che non è un router è stato usato in innumerevoli attacchi per far transitare il traffico tra segmenti di rete.

### Come funzionano i livelli

I controlli di **Level 1** sono la baseline. Sono pensati per essere applicati a ogni sistema senza compromettere le funzionalità standard. Se fai il deploy di un server Debian e applichi tutti i controlli di Level 1, il sistema dovrebbe comunque funzionare normalmente come web server, database server o host applicativo. Esempi: disabilitare i filesystem inutilizzati, impostare noexec su /tmp, disabilitare il login SSH come root.

I controlli di **Level 2** forniscono un hardening più profondo ma possono limitare le funzionalità. Disabilitare il filesystem squashfs (controllo 1.1.6) è di Level 2 perché rompe i pacchetti snap su Ubuntu. Disabilitare vfat (controllo 1.1.8) è di Level 2 perché i sistemi UEFI hanno bisogno di vfat per la EFI System Partition. Un'organizzazione che esegue container potrebbe saltare le restrizioni su squashfs. Un server bare-metal con BIOS legacy può disabilitare vfat in sicurezza.

La distinzione tra livelli conta nella pratica. Un audit di conformità che richiede "CIS Level 1" è raggiungibile per quasi ogni deployment Linux. Un audit che richiede "CIS Level 2" richiede un'attenta valutazione di quali controlli sia sicuro applicare nel tuo ambiente specifico.

### Scored vs. Unscored

I controlli scored concorrono alla percentuale di conformità. Se hai 80 controlli scored e ne superi 64, il tuo punteggio è 80%. I controlli unscored sono raccomandazioni che non influiscono sul punteggio. Sono tipicamente controlli in cui la configurazione corretta dipende interamente dal contesto dell'organizzazione (come la destinazione syslog a cui inviare i log).

Questo progetto tratta tutti i 104 controlli come scored per impostazione predefinita. Il motore di punteggio in `engine.sh` conta solo i risultati PASS e FAIL (non WARN o SKIP) quando calcola le percentuali, il che corrisponde a come funzionano le linee guida di punteggio CIS.

---

## Hardening del sistema

### Che cos'è

L'hardening del sistema è il processo di riduzione della superficie di attacco di un sistema disabilitando le funzionalità non necessarie, restringendo i permessi e configurando i controlli di sicurezza. Un sistema operativo appena installato è progettato per un'ampia compatibilità, non per la sicurezza. Viene distribuito con decine di moduli del kernel caricati, servizi abilitati e configurazioni predefinite permissive, perché il vendor non conosce il tuo caso d'uso.

L'hardening è l'opposto. Parti dalle impostazioni predefinite e blocchi metodicamente tutto ciò che non è richiesto per il ruolo specifico del sistema.

### Perché è importante

La violazione di Equifax del 2017 ha esposto 147 milioni di numeri di previdenza sociale. Il vettore di attacco iniziale era CVE-2017-5638, una vulnerabilità di Apache Struts. Ma la violazione è stata catastrofica a causa di carenze sistemiche di hardening. Il server compromesso aveva credenziali non cifrate memorizzate nei file di configurazione, nessuna segmentazione di rete che isolasse il livello web dal livello database, e certificati SSL scaduti sugli strumenti di monitoraggio interni, il che significava che il rilevamento delle intrusioni era stato di fatto cieco per 19 mesi.

L'hardening non avrebbe impedito la vulnerabilità di Struts. Ma avrebbe contenuto il raggio d'impatto. Se /tmp avesse avuto noexec impostato, la web shell caricata dall'attaccante non sarebbe stata eseguita. Se il sottosistema di audit fosse stato abilitato, la compromissione iniziale avrebbe generato degli alert. Se i permessi sui file di configurazione sensibili fossero stati ristretti, l'attaccante non avrebbe trovato le credenziali del database in chiaro.

Lo scopo dell'hardening non è rendere un sistema invulnerabile. È rendere il divario tra la compromissione iniziale e la violazione completa il più ampio possibile.

### Defense in Depth

I benchmark CIS implementano la defense in depth attraverso sei livelli, ognuno corrispondente a una sezione del benchmark. Nessun singolo controllo ferma ogni attacco. Ma insieme creano un sistema in cui un attaccante deve superare più difese indipendenti.

```
Layer 6: System Maintenance (file permissions, user accounts)
    ↑ prevents privilege escalation via misconfigurations
Layer 5: Access Controls (cron, SSH, passwords, PAM)
    ↑ restricts who can authenticate and how
Layer 4: Logging and Auditing (auditd, rsyslog)
    ↑ ensures attacks leave evidence
Layer 3: Network Configuration (sysctl, firewall, protocols)
    ↑ limits lateral movement and network attacks
Layer 2: Services (remove unnecessary daemons)
    ↑ reduces attack surface
Layer 1: Initial Setup (filesystems, bootloader, kernel)
    ↑ hardens the foundation
```

Un attaccante che compromette un'applicazione web su un sistema sottoposto ad hardening si trova davanti tutti e sei i livelli. Non può depositare eseguibili in /tmp (Layer 1). Non può fare pivot attraverso un servizio DHCP o NFS non necessario (Layer 2). Non può reindirizzare il traffico tramite IP forwarding (Layer 3). Le sue azioni vengono registrate da auditd (Layer 4). Non può accedere via SSH come root (Layer 5). Non può trovare un secondo account con UID 0 (Layer 6).

---

## Sezione 1: Configurazione iniziale

### Hardening del filesystem

I controlli da 1.1.1 a 1.1.8 disabilitano i moduli del kernel per i filesystem che i server non dovrebbero mai avere bisogno di montare. cramfs, freevxfs, jffs2, hfs, hfsplus, squashfs, udf e vfat sono tutti filesystem progettati per sistemi embedded, hardware Apple, supporti ottici o dispositivi rimovibili. Un server Linux di produzione non ha alcun motivo di montare volumi HFS di un Mac.

Il rischio non è teorico. CVE-2013-1773 era un buffer overflow nel driver del filesystem VFAT del kernel Linux che permetteva l'esecuzione di codice arbitrario durante il mount di un'immagine di filesystem FAT appositamente costruita. CVE-2019-19813 era un use-after-free nel driver del filesystem HFS+. Se il modulo del kernel è caricato, qualsiasi utente che possa innescare un mount (o indurre il kernel a eseguire un auto-mount) può sfruttare queste vulnerabilità. Disabilitare il modulo elimina completamente la superficie di attacco.

La funzione di controllo `_check_module_disabled` in `01_initial_setup.sh` verifica che ogni modulo abbia una direttiva `install [module] /bin/true` in `/etc/modprobe.d/`, che fa sì che il kernel ignori silenziosamente le richieste di caricamento.

I controlli da 1.2.1 a 1.2.4 assicurano che /tmp sia una partizione separata con le opzioni di mount `noexec`, `nosuid` e `nodev`. La directory /tmp è scrivibile da chiunque, il che la rende il primo posto in cui un attaccante deposita i payload dopo aver ottenuto l'accesso iniziale. Con noexec, quei payload non verranno eseguiti. Con nosuid, i binari setuid collocati in /tmp non possono elevare i privilegi. Con nodev, non è possibile creare device node.

### Protezione del bootloader

I controlli da 1.4.1 a 1.4.3 proteggono il processo di avvio. Senza una password del bootloader (1.4.1), chiunque abbia accesso fisico o alla console può modificare i parametri di avvio di GRUB per avviare in modalità single-user e reimpostare la password di root. Il controllo 1.4.3 verifica che la modalità single-user richieda l'autenticazione `sulogin`, il che impedisce al classico attacco "init=/bin/bash" di concedere una shell root non autenticata.

### Hardening del kernel

ASLR (Address Space Layout Randomization, controllo 1.5.1) rende casuali gli indirizzi di memoria usati dalle librerie di sistema, dallo stack e dall'heap. Senza ASLR (`kernel.randomize_va_space = 0`), un attaccante che trova una vulnerabilità di buffer overflow può prevedere esattamente dove finirà in memoria il suo shellcode. Con ASLR completo (`kernel.randomize_va_space = 2`), deve tirare a indovinare tra miliardi di indirizzi possibili.

La restrizione dei core dump (controllo 1.5.2) impedisce ai programmi di scrivere su disco il contenuto della loro memoria quando vanno in crash. I core dump possono contenere password, chiavi di cifratura e altri dati sensibili che si trovavano in memoria al momento del crash.

---

## Sezione 2: Servizi

### Riduzione della superficie di attacco

Ogni servizio in esecuzione è un potenziale punto di ingresso. La Sezione 2 del CIS verifica 16 servizi che non dovrebbero essere installati su un server sottoposto ad hardening a meno che non ci sia un requisito di business esplicito. La filosofia è semplice: se non ti serve, rimuovilo.

I servizi verificati includono:

- **xinetd e openbsd-inetd** (2.1.1, 2.1.2): super-daemon legacy che restano in ascolto di connessioni e avviano altri servizi. Sono reliquie degli anni '90, quando la memoria era costosa ed eseguire un daemon separato per ogni servizio era uno spreco. I sistemi moderni usano la socket activation di systemd. inetd è stato un bersaglio prediletto negli attacchi per decenni.

- **X Window System** (2.2.1): un display server grafico non ha posto su un server headless. Il protocollo X è stato progettato senza pensare alla sicurezza. Eseguire X aumenta drasticamente la superficie di attacco e fornisce capacità (cattura dello schermo, registrazione dell'input) che gli attaccanti sfruttano per la raccolta di dati.

- **NFS, Samba, LDAP, DNS, FTP** (da 2.2.5 a 2.2.8): ciascuno è un servizio di rete complesso con una lunga storia di vulnerabilità. Una configurazione errata di un server NFS nel 2020 (nessuna CVE specifica, si trattava di un errore di configurazione) ha permesso a un attaccante di montare una condivisione di file interna da internet ed esfiltrare 4TB di dati dei clienti da un'istituzione finanziaria non identificata.

- **Configurazione MTA solo locale** (2.2.15): i mail transfer agent come Postfix per impostazione predefinita restano in ascolto su tutte le interfacce. Il controllo 2.2.15 verifica che l'MTA resti in ascolto solo su loopback (`inet_interfaces = loopback-only`). Un MTA in ascolto su un'interfaccia pubblica è un open relay pronto a verificarsi.

---

## Sezione 3: Configurazione di rete

### Parametri di rete del kernel

Il kernel Linux espone parametri di rete regolabili tramite `/proc/sys/net/` e `sysctl`. I controlli da 3.1.1 a 3.2.6 verificano che questi parametri siano impostati su valori sicuri.

**IP forwarding** (3.1.1): `net.ipv4.ip_forward` controlla se il kernel instrada i pacchetti tra le interfacce di rete. Su un server con due NIC, il forwarding abilitato trasforma quel server in un router. Un attaccante che compromette un server del genere può far transitare il traffico tra segmenti di rete, aggirando i controlli di segmentazione della rete. A meno che il server non sia esplicitamente un router o un gateway VPN, deve essere disabilitato.

**ICMP redirect** (3.1.2, 3.1.4): i messaggi ICMP redirect dicono a un host di usare un gateway diverso per una destinazione specifica. Un attaccante sulla rete locale può inviare ICMP redirect contraffatti per reinstradare il traffico di una vittima attraverso una macchina controllata dall'attaccante, abilitando attacchi man-in-the-middle. Disabilitare send_redirects e accept_redirects elimina questo vettore.

**Source routing** (3.1.3): i pacchetti con source routing permettono al mittente di specificare il percorso esatto attraverso la rete. Questo aggira le policy di routing e le regole del firewall perché è l'attaccante a stabilire quali router attraversa il pacchetto. Le reti moderne non dovrebbero mai accettare pacchetti con source routing.

**Reverse path filtering** (3.2.4): `rp_filter` fa sì che il kernel verifichi che i pacchetti in ingresso su un'interfaccia possano legittimamente arrivare su quell'interfaccia in base alla tabella di routing. Questo blocca gli attacchi di IP spoofing in cui un attaccante invia pacchetti con un indirizzo sorgente contraffatto.

**TCP SYN cookies** (3.2.5): `tcp_syncookies` difende dagli attacchi denial-of-service di tipo SYN flood. Senza i SYN cookie, un attaccante che invia milioni di pacchetti SYN può esaurire la tabella delle connessioni del server, impedendo ai client legittimi di connettersi. Con i SYN cookie abilitati, il kernel usa una tecnica crittografica per convalidare i tentativi di connessione senza allocare risorse finché l'handshake non è completato.

### Configurazione del firewall

I controlli da 3.3.1 a 3.3.5 verificano che iptables sia installato, che le policy predefinite per le chain INPUT, FORWARD e OUTPUT siano DROP o REJECT e che ogni porta aperta abbia una regola firewall corrispondente.

Una policy firewall default-deny significa che qualsiasi traffico non esplicitamente consentito viene scartato. È l'opposto della policy default-accept con cui viene distribuita la maggior parte delle distribuzioni. Il default-deny intercetta i casi in cui uno sviluppatore avvia un servizio su una porta inattesa, una dipendenza apre un listener di debug o una configurazione errata espone un servizio interno alla rete.

### Moduli di protocolli non comuni

I controlli da 3.4.2 a 3.4.5 disabilitano i moduli del kernel DCCP, SCTP, RDS e TIPC. Sono protocolli di trasporto specializzati che la maggior parte dei server non usa mai. Ognuno ha avuto vulnerabilità del kernel (CVE-2017-6074 era un double-free in DCCP che permetteva una privilege escalation locale fino a root). Disabilitare il modulo è la mitigazione più semplice.

---

## Sezione 4: Logging e auditing

### Perché il logging è importante

Senza logging, una violazione è invisibile. Il dwell time mediano degli attaccanti (il divario tra la compromissione iniziale e il rilevamento) era di 16 giorni nel 2023 secondo il report M-Trends di Mandiant. Per le organizzazioni con un logging scarso era significativamente più lungo. La violazione di Target nel 2013 non è stata rilevata per settimane nonostante FireEye generasse alert, perché il team di sicurezza non aveva un'infrastruttura di logging adeguata per correlare gli eventi.

### auditd

I controlli da 4.1.1 a 4.1.14 verificano che il framework di audit di Linux (`auditd`) sia installato, abilitato, configurato per avviarsi al boot e che abbia regole che coprono le system call e i file critici per la sicurezza.

Il framework di audit opera a livello di kernel. Quando è configurato con delle regole, genera voci di log ogni volta che un processo chiama specifiche system call o modifica specifici file. Le regole di audit in questo progetto verificano:

- **Modifiche dell'orario** (4.1.5): `adjtimex`, `settimeofday`, `clock_settime`. Gli attaccanti modificano l'orario di sistema per far sembrare che la loro attività sia avvenuta durante una finestra di manutenzione o per invalidare token di autenticazione basati sul tempo.
- **Modifiche di utenti/gruppi** (4.1.6): modifiche a `/etc/passwd`, `/etc/shadow`, `/etc/group`. Qualsiasi modifica non autorizzata a questi file indica creazione di account, privilege escalation o persistenza.
- **Modifiche all'ambiente di rete** (4.1.7): `sethostname`, `setdomainname`, modifiche a `/etc/hosts`. Cambiare l'hostname o la configurazione DNS può far parte di un attacco man-in-the-middle o di un reindirizzamento C2.
- **Modifiche ai permessi DAC** (4.1.11): `chmod`, `chown`, `fchmod`, `setxattr`. Queste system call modificano i controlli di accesso ai file. Gli attaccanti le usano per rendere leggibili file sensibili o per impostare il bit setuid ai fini della privilege escalation.
- **Tentativi di accesso non autorizzati** (4.1.12): tentativi falliti di accesso ai file che restituiscono EACCES o EPERM. Uno schema di tentativi di accesso negati è un forte indicatore di un attaccante che sta sondando il filesystem alla ricerca di file sensibili.

### rsyslog

I controlli da 4.2.1 a 4.2.4 verificano che rsyslog sia installato, abilitato, configurato con una modalità di creazione dei file restrittiva (0640 o più stretta) e che abbia regole di logging definite. rsyslog cattura i messaggi delle applicazioni e del sistema che auditd non copre: eventi di autenticazione, avvio/arresto dei servizi, messaggi del kernel e log applicativi.

---

## Sezione 5: Accesso, autenticazione e autorizzazione

### Sicurezza di cron

I controlli da 5.1.1 a 5.1.4 verificano che cron sia abilitato e che i file e le directory relativi a cron abbiano permessi restrittivi. Un attaccante che può scrivere in `/etc/cron.daily/` può installare un meccanismo di persistenza che viene eseguito ogni giorno come root. Restringere le directory di cron alla modalità 700 con proprietario root:root impedisce agli utenti non root di piazzare cron job.

### Hardening di SSH

I controlli da 5.2.1 a 5.2.14 riguardano la configurazione di SSH. SSH è il principale vettore di accesso remoto per i server Linux, e la sua configurazione determina direttamente se il server è accessibile agli attaccanti.

**Login come root** (5.2.8): `PermitRootLogin no` obbliga gli amministratori a effettuare il login con un account nominativo e poi a elevarsi a root con `sudo`. Questo crea un audit trail di chi ha eseguito quali azioni. Quando il login come root è consentito, più amministratori condividono la password di root e non c'è modo di determinare quale persona abbia eseguito quali comandi.

**Crittografia debole** (da 5.2.11 a 5.2.13): questi controlli verificano che SSH non consenta cipher in modalità CBC (vulnerabili ad attacchi di recupero del plaintext come CVE-2008-5161), MAC basati su MD5 (computazionalmente compromessi) o algoritmi di scambio di chiavi deboli come diffie-hellman-group1-sha1 (vulnerabile all'attacco Logjam, CVE-2015-4000, che permetteva ad attaccanti man-in-the-middle di effettuare il downgrade delle connessioni a chiavi da 512 bit).

**MaxAuthTries** (5.2.6): limitare i tentativi di autenticazione a 4 rende impraticabili gli attacchi brute-force via SSH. Senza questo limite, un attaccante può provare migliaia di password in una singola connessione.

**LoginGraceTime** (5.2.14): la finestra temporale per completare l'autenticazione. Impostarla a 60 secondi impedisce agli attaccanti di aprire centinaia di connessioni e tenerle aperte indefinitamente, il che può esaurire il pool di connessioni del daemon SSH.

### Policy delle password

I controlli 5.3.1 e da 5.4.1 a 5.4.3 verificano i moduli PAM per la qualità delle password e le impostazioni di login.defs. `PASS_MAX_DAYS` impone la rotazione delle password. `PASS_MIN_DAYS` impedisce agli utenti di tornare immediatamente alla loro vecchia password. `PASS_WARN_AGE` avvisa in anticipo gli utenti della scadenza.

Il controllo 5.5.1 verifica il blocco degli account. Senza `pam_faillock` o `pam_tally2`, un attaccante può tentare un numero illimitato di password contro un account locale tramite PAM senza alcun meccanismo di blocco.

---

## Sezione 6: Manutenzione del sistema

### Permessi dei file

I controlli da 6.1.1 a 6.1.5 verificano che i file di sistema critici abbiano proprietà e permessi corretti:

- `/etc/passwd` dovrebbe essere 644 root:root (leggibile da tutti, scrivibile solo da root)
- `/etc/shadow` dovrebbe essere 640 root:shadow (solo root e il gruppo shadow possono leggere gli hash delle password)
- `/etc/group` dovrebbe essere 644 root:root
- `/etc/gshadow` dovrebbe essere 640 root:shadow

Se `/etc/shadow` fosse leggibile da chiunque, qualsiasi utente del sistema potrebbe estrarre gli hash delle password e craccarli offline con strumenti come hashcat. Una GPU moderna può testare miliardi di combinazioni di hash al secondo contro hash SHA-512.

### Integrità degli account

I controlli da 6.2.1 a 6.2.5 verificano la presenza di UID duplicati, GID duplicati, nomi utente duplicati, account con UID 0 diversi da root e voci legacy "+" nei file passwd/shadow/group.

**Account con UID 0** (6.2.4): qualsiasi account con UID 0 ha pieni privilegi di root indipendentemente dal suo nome. Una tecnica di persistenza comune è creare un account dall'aspetto innocuo come "sysadmin" o "backup" con UID 0. Il controllo 6.2.4 lo intercetta verificando che solo "root" abbia UID 0.

**Voci legacy +** (6.2.5): il prefisso "+" nei file passwd/shadow/group è una reliquia di NIS (Network Information Service). Una riga che inizia con "+" in /etc/passwd dice al sistema di includere voci da un server NIS remoto. Se NIS non è in uso, una voce "+" può introdurre account inattesi o, in alcune configurazioni, creare un account equivalente a root senza password.

---

## Come si collegano questi concetti

```
CIS Benchmark (104 controls)
    ↓ organized into
6 Sections (defense in depth layers)
    ↓ each control has
Level (1 = baseline, 2 = advanced)
    ↓ and produces
Scored Results (PASS / FAIL / WARN / SKIP)
    ↓ which feed into
Compliance Score (percentage)
    ↓ which maps to
Regulatory Frameworks (NIST 800-53, PCI DSS, SOC 2)
```

---

## Standard e framework del settore

### NIST 800-53

I benchmark CIS si mappano direttamente sulle famiglie di controlli NIST 800-53:
- **AC (Access Control)**: hardening di SSH, permessi di cron, policy delle password
- **AU (Audit and Accountability)**: configurazione di auditd, regole di audit, rsyslog
- **CM (Configuration Management)**: parametri del kernel, servizi disabilitati, opzioni di mount
- **SC (System and Communications Protection)**: policy del firewall, protocolli disabilitati, IP forwarding

### PCI DSS

Il Requisito 2 di PCI DSS ("Do not use vendor-supplied defaults") si mappa direttamente sull'hardening CIS. Il Requisito 2.2 richiede specificamente la rimozione di servizi, protocolli e daemon non necessari, che è esattamente ciò che verifica la Sezione 2 del CIS. I QSA (Qualified Security Assessor) PCI accettano regolarmente i report di audit dei benchmark CIS come evidenza per la conformità al Requisito 2.

### MITRE ATT&CK

I controlli di questo progetto si mappano sulle mitigazioni ATT&CK:
- **M1042 (Disable or Remove Feature)**: disabilitazione di filesystem e servizi inutilizzati (Sezioni 1, 2)
- **M1037 (Filter Network Traffic)**: configurazione del firewall (Sezione 3)
- **M1029 (Remote Data Storage)**: inoltro con rsyslog (Sezione 4)
- **M1032 (Multi-factor Authentication)**: imposizione dell'autenticazione SSH con chiave (Sezione 5)
- **M1018 (User Account Management)**: controlli sugli UID duplicati, restrizione dell'UID 0 (Sezione 6)

---

## Esempi reali

### Caso di studio 1: la violazione di Equifax (2017)

La compromissione iniziale è stata CVE-2017-5638, una vulnerabilità di remote code execution in Apache Struts. Ma l'indagine sulla violazione ha rivelato che i server compromessi mancavano dei controlli di hardening di base definiti dai benchmark CIS. La certificate authority interna aveva certificati scaduti, il che significava che l'ispezione del traffico cifrato era disabilitata. I permessi del filesystem erano permissivi. La segmentazione di rete era assente. Gli attaccanti si sono mossi lateralmente dal livello web al livello database senza far scattare alcun alert, perché l'audit logging era insufficiente.

Un audit CIS Level 1 su quei server avrebbe segnalato le regole di audit mancanti, i permessi dei file permissivi e l'assenza di controlli di rete. L'exploit iniziale avrebbe comunque funzionato, ma il movimento laterale dell'attaccante e l'esfiltrazione dei dati sarebbero stati significativamente più difficili e avrebbero generato eventi di audit.

### Caso di studio 2: l'attacco SolarWinds SUNBURST (2020)

La backdoor SUNBURST inserita negli aggiornamenti di SolarWinds Orion usava comunicazioni C2 sofisticate, ma i sistemi di build compromessi mostravano segni di un hardening inadeguato. I build server avevano l'IP forwarding abilitato tra segmenti di rete, l'audit logging non era configurato per catturare le modifiche ai file nella pipeline di build e l'accesso SSH non era limitato ai gruppi autorizzati.

La lezione: anche l'attacco alla supply chain più sofisticato sfrutta il divario tra l'avere strumenti di sicurezza e l'eseguire effettivamente l'hardening dei sistemi su cui quegli strumenti girano.

---

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Perché il benchmark CIS distingue tra controlli di Level 1 e di Level 2? Fai un esempio in cui l'applicazione di un controllo di Level 2 comprometterebbe delle funzionalità.
2. Un attaccante ha compromesso un'applicazione web e può scrivere file in /tmp. Spiega quali tre opzioni di mount limiterebbero la sua capacità di usare /tmp per la privilege escalation e perché ciascuna funziona.
3. Perché `kernel.randomize_va_space = 1` (ASLR parziale) è insufficiente mentre `2` (ASLR completo) è richiesto dal benchmark CIS?
4. Un analista SOC scopre che `net.ipv4.conf.all.accept_redirects = 1` su un server di produzione. Descrivi l'attacco specifico che questo rende possibile.

---

## Letture di approfondimento

**Essenziali:**
- [CIS Debian Linux 12 Benchmark v1.1.0](https://www.cisecurity.org/benchmark/debian_linux) - Il documento di benchmark effettivo implementato da questo progetto
- [NIST 800-123: Guide to General Server Security](https://csrc.nist.gov/publications/detail/sp/800-123/final) - Il documento fondamentale sull'hardening dei server

**Approfondimenti:**
- [Linux Audit Framework Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/auditing-the-system_security-hardening) - Guida completa alla configurazione di auditd
- [OpenSSH Hardening Guide](https://www.ssh-audit.com/hardening_guides.html) - Analisi dettagliata della sicurezza dei cipher e dello scambio di chiavi SSH

**Contesto storico:**
- [Mandiant M-Trends Report](https://www.mandiant.com/m-trends) - Report annuale sui dwell time degli attaccanti, sulle tecniche e sul ruolo del logging nel rilevamento
