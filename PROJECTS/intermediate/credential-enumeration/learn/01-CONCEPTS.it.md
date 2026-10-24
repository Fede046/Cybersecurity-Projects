# 01-CONCEPTS.md

# Concetti di sicurezza

Questo documento tratta i fondamenti di sicurezza alla base della credential enumeration: quali credenziali esistono su un tipico sistema Linux, dove si trovano, perché sono esposte e cosa ci fanno gli attaccanti reali.

## Esposizione delle credenziali ed enumerazione post-accesso

### Che cos'è

L'esposizione delle credenziali si verifica quando il materiale di autenticazione (password, token, chiavi private, segreti API) è memorizzato in un modo che consente l'accesso non autorizzato. L'enumerazione post-accesso è la fase di un attacco in cui, avendo ottenuto un certo livello di accesso a un sistema, l'attaccante cerca sistematicamente credenziali aggiuntive per ampliare la propria portata.

È una delle prime cose che gli attaccanti fanno dopo essere atterrati su una macchina. Non perché sia ingegnoso, ma perché funziona. Le workstation degli sviluppatori accumulano credenziali come pelucchi. Chiavi AWS in `~/.aws/credentials`. Chiavi private SSH senza passphrase in `~/.ssh/`. Token GitHub in `~/.gitconfig`. Password di database in `~/.pgpass`. Token di Vault in `~/.vault-token`. La maggior parte di questi file resta lì per mesi o anni, raramente sottoposta ad audit, spesso con permessi più larghi del necessario.

### Perché è importante

La violazione di Uber del settembre 2022 è l'esempio da manuale. Un attaccante diciottenne ha acquistato credenziali rubate sul dark web, ha aggirato la MFA tramite la fatigue da notifiche push e poi ha trovato credenziali hardcoded in script PowerShell su condivisioni di rete interne. Quelle credenziali davano accesso alla console AWS, all'amministrazione di Google Workspace, al pannello di amministrazione di Duo e alla dashboard del bug bounty di Uber su HackerOne. La compromissione iniziale è stata di social engineering; l'escalation è stata la raccolta di credenziali.

La violazione di LastPass (2022-2023) è ancora più diretta. Gli attaccanti hanno compromesso la macchina domestica di un senior DevOps engineer, hanno trovato chiavi SSH e chiavi di decifratura memorizzate localmente e le hanno usate per accedere allo storage cloud di produzione contenente i vault cifrati dei clienti. L'ingegnere aveva un accesso legittimo; le credenziali sulla sua macchina domestica hanno fornito il percorso.

Nell'attacco alla supply chain di Codecov (aprile 2021), gli attaccanti hanno modificato uno script bash uploader per esfiltrare le variabili d'ambiente dalle pipeline CI/CD. Le variabili trapelate includevano token, chiavi API e credenziali a cui i sistemi CI avevano accesso. Migliaia di clienti sono stati colpiti perché i loro ambienti di build avevano credenziali disponibili come variabili d'ambiente.

### Mappatura MITRE ATT&CK

Questo progetto implementa direttamente il rilevamento per queste tecniche:

| ID tecnica | Nome | Cosa rileviamo |
|-------------|------|----------------|
| T1552 | Unsecured Credentials | Tecnica padre per ogni esposizione di credenziali |
| T1552.001 | Credentials In Files | Credenziali AWS, .pgpass, .my.cnf, .netrc, .git-credentials, file .env |
| T1552.003 | Bash History | Segreti nella cronologia della shell (export TOKEN=, curl -u), pattern di comandi |
| T1552.004 | Private Keys | Chiavi SSH (cifrate/non cifrate), chiavi di service account GCP |
| T1555 | Credentials from Password Stores | GNOME Keyring, KDE Wallet, database KeePass, store di pass |
| T1555.001 | Keychain | Database dei keyring desktop |
| T1555.003 | Credentials from Web Browsers | logins.json/key4.db di Firefox, Login Data di Chromium |
| T1539 | Steal Web Session Cookie | cookies.sqlite di Firefox, database Cookies di Chromium |

## Permessi dei file di Linux

### Cosa sono

Ogni file su un sistema Linux ha tre insiemi di permessi: proprietario, gruppo e altri (world). Ogni insieme può avere i bit di lettura (r=4), scrittura (w=2) ed esecuzione (x=1). Quando vedi `0600` su una chiave SSH, significa che il proprietario può leggerla e scriverla, ma nessun altro può vederla. Quando vedi `0644`, il proprietario può leggere/scrivere ma il gruppo e tutti gli altri possono leggerla.

### Perché è critico per le credenziali

I permessi dei file sono la difesa principale per i file di credenziali su disco. Una chiave privata SSH con permessi `0644` significa che qualsiasi utente del sistema può leggerla. Su un server condiviso, qualsiasi servizio compromesso in esecuzione come qualsiasi utente può rubare quella chiave. Su una workstation a utente singolo, un malware in esecuzione come un utente diverso (o una container escape) ottiene accesso immediato.

OpenSSH stesso si rifiuta di usare una chiave privata con permessi eccessivamente permissivi. Stamperà `WARNING: UNPROTECTED PRIVATE KEY FILE!` e rifiuterà di autenticarsi. Ma gli altri file di credenziali non hanno una protezione del genere. Il tuo file di credenziali AWS in `~/.aws/credentials` non si cura dei propri permessi. Né lo fanno `~/.pgpass`, `~/.git-credentials` o `~/.vault-token`. Verranno letti dai rispettivi tool indipendentemente da quanto siano esposti.

### Come funziona il modello dei permessi

```
Permission bits:   Owner   Group   Others
                   rwx     rwx     rwx
0600 =             rw-     ---     ---    (owner read/write only)
0644 =             rw-     r--     r--    (everyone can read)
0700 =             rwx     ---     ---    (owner full access, directory)
0755 =             rwx     r-x     r-x    (everyone can read/enter directory)
```

La gerarchia di controllo dei permessi usata da questo tool:

| Condizione | Gravità | Motivazione |
|-----------|----------|-----------|
| Leggibile da chiunque (others ha il bit di lettura, `0o004`) | CRITICAL | Qualsiasi utente o processo del sistema può leggere il file |
| Leggibile dal gruppo (group ha il bit di lettura, `0o040`) | MEDIUM-HIGH | Gli utenti che condividono il gruppo possono leggerlo; comune nell'hosting condiviso |
| Più larghi del previsto (es. 0644 invece di 0600) | LOW | Più permissivi della best practice ma non immediatamente sfruttabili |
| Solo proprietario (file 0600, directory 0700) | INFO | Permessi corretti, annotati per completezza |

### Nel mondo reale: il collegamento con la violazione di Capital One

La violazione di Capital One del 2019 ha coinvolto un WAF configurato male che permetteva una SSRF, usata per interrogare il servizio di metadati dell'istanza EC2 e recuperare le credenziali del ruolo IAM. Sebbene sia un percorso di attacco specifico del cloud, il principio di fondo è lo stesso: credenziali accessibili a processi che non dovrebbero averle. Su una workstation Linux, permessi dei file eccessivamente permissivi creano la stessa classe di esposizione a livello di filesystem.

## Archiviazione delle credenziali nei browser

### Come i browser memorizzano le credenziali

Firefox e i browser basati su Chromium memorizzano entrambi le credenziali localmente nella home directory dell'utente.

**Firefox** usa un sistema basato sui profili con radice in `~/.mozilla/firefox/`. Ogni directory di profilo contiene:
- `logins.json` - Nomi utente e password memorizzati (cifrati con una chiave proveniente da key4.db)
- `key4.db` - Database delle chiavi NSS che contiene la chiave di cifratura per logins.json
- `cookies.sqlite` - Cookie di sessione che possono essere usati per il session hijacking

I profili di Firefox sono elencati in `profiles.ini`. Un utente potrebbe avere più profili (personale, lavoro), ciascuno con i propri archivi di credenziali.

**I browser basati su Chromium** (Chrome, Brave, Vivaldi, Chromium) memorizzano i dati sotto `~/.config/<browser>/Default/` (e profili numerati come `Profile 1`, `Profile 2`):
- `Login Data` - Database SQLite delle password memorizzate
- `Cookies` - Cookie di sessione
- `Web Data` - Dati di compilazione automatica, metodi di pagamento salvati

Su Linux, Chromium cifra le password memorizzate usando il keyring di sistema (GNOME Keyring o KWallet). Senza il keyring sbloccato, le password cifrate non sono direttamente utilizzabili, ma i file rivelano comunque quali siti hanno credenziali memorizzate, e i cookie di sessione potrebbero essere utilizzabili senza decifratura.

### Perché è importante

La violazione di CircleCI del gennaio 2023 ha coinvolto il laptop compromesso di un ingegnere da cui sono stati rubati token di sessione dall'archiviazione del browser. Quei token fornivano accesso ai sistemi interni di CircleCI, che a loro volta contenevano segreti dei clienti (variabili d'ambiente, chiavi API, token). L'attaccante non ha avuto bisogno di craccare password. I cookie di sessione dall'archiviazione del browser sono bastati.

Database di credenziali del browser leggibili da chiunque (0644) sono un rilevamento CRITICAL perché significa che qualsiasi processo del sistema può leggere le credenziali cifrate e i cookie di sessione. Anche con la cifratura, i cookie sono spesso immediatamente utilizzabili per il session hijacking.

## Sicurezza delle chiavi SSH

### Come funzionano le chiavi SSH

Le coppie di chiavi SSH sono costituite da una chiave privata (memorizzata localmente) e una chiave pubblica (collocata sui server remoti in `~/.ssh/authorized_keys`). La chiave privata dimostra la tua identità. Se qualcuno ha la tua chiave privata, può autenticarsi come te su qualsiasi server che si fidi della chiave pubblica corrispondente.

Le chiavi private esistono in diversi formati:
- Formato OpenSSH (`-----BEGIN OPENSSH PRIVATE KEY-----`) - default moderno
- RSA PEM (`-----BEGIN RSA PRIVATE KEY-----`) - formato legacy
- ECDSA PEM (`-----BEGIN EC PRIVATE KEY-----`) - curva ellittica
- DSA PEM (`-----BEGIN DSA PRIVATE KEY-----`) - deprecato ma ancora presente
- PKCS#8 (`-----BEGIN PRIVATE KEY-----`) - wrapper generico

### Protezione con passphrase

Le chiavi private possono opzionalmente essere cifrate con una passphrase. Una chiave cifrata contiene marcatori come `ENCRYPTED`, `Proc-Type: 4,ENCRYPTED`, `bcrypt` o `aes256-ctr` nella sua intestazione. Senza la passphrase, il file della chiave è inutile per un attaccante. Ma una chiave non cifrata è immediatamente utilizzabile.

La ripartizione della gravità:

| Stato della chiave | Permessi | Gravità | Perché |
|-----------|------------|----------|-----|
| Non cifrata | Leggibile da chiunque | CRITICAL | Chiunque può rubare e usare immediatamente la chiave |
| Cifrata | Leggibile da chiunque | CRITICAL | La passphrase può essere forzata con brute force offline |
| Non cifrata | Solo proprietario | HIGH | Permessi corretti ma nessuna defense-in-depth |
| Cifrata | Solo proprietario | INFO | Entrambe le protezioni sono attive |

### Debolezze della configurazione SSH

Anche il file di configurazione SSH (`~/.ssh/config`) può rivelare problemi di sicurezza:
- `PasswordAuthentication yes` - consente l'autenticazione basata su password, più debole di quella basata su chiave
- `StrictHostKeyChecking no` - disabilita la verifica della host key, rendendo possibili attacchi machine-in-the-middle
- Le voci Host rivelano a quali server si connette l'utente, fornendo agli attaccanti un elenco di bersagli

### Nel mondo reale: la catena della violazione di Codecov

Quando il bash uploader di Codecov è stato compromesso nel 2021, uno dei tipi di credenziali esfiltrati dagli ambienti CI/CD erano le chiavi SSH. Gli attaccanti hanno usato chiavi SSH rubate dalla pipeline CI di Twitch per accedere a repository Git interni, il che ha contribuito al massiccio leak del codice sorgente di Twitch nell'ottobre 2021. Le chiavi SSH trovate in un ambiente sono diventate il punto di ingresso in un altro.

## Credenziali dei cloud provider

### AWS

Le credenziali AWS si trovano in `~/.aws/credentials` in formato INI con profili. Ogni profilo può contenere:
- `aws_access_key_id` - inizia con `AKIA` per le chiavi statiche o `ASIA` per le chiavi di sessione temporanee
- `aws_secret_access_key` - la componente segreta
- `aws_session_token` - presente per le credenziali temporanee

Le chiavi statiche (`AKIA`) hanno una lunga durata e rappresentano il rischio più alto. Funzionano finché non vengono esplicitamente ruotate o eliminate. Le chiavi di sessione (`ASIA`) sono temporanee ma comunque pericolose durante la loro finestra di validità. Il file associato `~/.aws/config` contiene profili, impostazioni di regione e, opzionalmente, configurazioni SSO o MFA.

Un file di credenziali AWS leggibile da chiunque è un rilevamento CRITICAL. Qualsiasi chiave statica trovata lì può essere usata per effettuare chiamate API ai servizi AWS con i permessi di cui dispone l'utente o il ruolo IAM associato.

### GCP

Le credenziali di Google Cloud sono memorizzate in `~/.config/gcloud/`. Il file più sensibile è `application_default_credentials.json`, che può contenere credenziali utente (da `gcloud auth application-default login`) oppure una chiave di service account (un file JSON con una chiave privata). Le chiavi di service account hanno gravità HIGH perché non scadono e spesso hanno permessi ampi. Le credenziali utente sono MEDIUM perché sono legate a una sessione interattiva e possono avere token a breve durata.

### Azure

La Azure CLI memorizza le cache dei token in `~/.azure/accessTokens.json` e `~/.azure/msal_token_cache.json`. Contengono token OAuth che possono essere usati per effettuare chiamate API ad Azure. Su un sistema multiutente, una cache dei token leggibile significa che altri utenti possono impersonare l'utente Azure autenticato.

### Kubernetes

La configurazione di Kubernetes in `~/.kube/config` contiene contesti dei cluster, definizioni degli utenti e dati di autenticazione. Questo può includere:
- Bearer token (accesso diretto all'API)
- Dati dei certificati client (certificati incorporati)
- Configurazioni degli auth provider

Una configurazione Kubernetes con bearer token ha gravità HIGH perché quei token spesso forniscono accesso cluster-admin o un accesso ampio ai namespace. La campagna TeamTNT del 2022 ha preso di mira specificamente le credenziali Kubernetes sugli host compromessi per diffondersi tra i cluster di container.

## La cronologia della shell come superficie di attacco

### Cosa c'è nella cronologia della shell

I file di cronologia della shell (`~/.bash_history`, `~/.zsh_history`, `~/.fish_history`) registrano ogni comando digitato in una sessione del terminale. Gli sviluppatori digitano abitualmente segreti direttamente nelle loro shell:

**Export di segreti:**
```
export API_KEY=sk-proj-abc123...
export DATABASE_URL=postgresql://admin:password@prod.db:5432/app
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Comandi contenenti credenziali:**
```
curl -H "Authorization: Bearer ghp_xxxx" https://api.github.com/repos
curl -u admin:s3cret https://internal-api.corp.net/deploy
mysql -u root -pMyPassword production_db
sshpass -p 'server_pass' ssh deploy@prod.server.com
```

Queste voci persistono nei file di cronologia indefinitamente a meno che non vengano cancellate esplicitamente. Su una workstation compromessa, la cronologia della shell è uno dei primi posti che un attaccante controlla.

### File d'ambiente

Oltre alla cronologia della shell, i file `.env` sparsi nelle directory dei progetti contengono segreti applicativi. Questi file seguono lo schema `KEY=value` e sono usati da framework e tool per caricare la configurazione. Una scansione ricorsiva della home directory trova comunemente file `.env`, `.env.local`, `.env.production` e `.env.staging` contenenti password di database, chiavi API e token di servizio.

### Nel mondo reale: la violazione di Dropbox

Nel novembre 2022, Dropbox ha reso noto che un attaccante aveva avuto accesso a 130 repository GitHub interni dopo un attacco di phishing. L'indagine ha rilevato che l'attaccante aveva ottenuto credenziali memorizzate nella configurazione d'ambiente usata dai sistemi CI/CD. Lo schema è coerente: credenziali nei file di configurazione, accessibili dopo l'accesso iniziale.

## Keyring e password manager

### Keyring desktop

Gli ambienti desktop Linux forniscono un'archiviazione delle credenziali a livello di sistema:

**GNOME Keyring** (`~/.local/share/keyrings/`) memorizza password, passphrase delle chiavi SSH e segreti applicativi in file `.keyring`. Il keyring predefinito viene sbloccato quando l'utente effettua il login e resta sbloccato per la durata della sessione. Se i file del keyring sono leggibili da altri utenti, il contenuto cifrato può essere esfiltrato per un attacco offline.

**KDE Wallet** (`~/.local/share/kwalletd/`) svolge la stessa funzione per i desktop KDE.

### Database dei password manager

I database di **KeePass** (file `.kdbx`) possono trovarsi ovunque nella home directory. Sono cifrati con una master password (e opzionalmente un key file), ma trovare un database KeePass dice a un attaccante che esiste un vault di password e che vale la pena prenderlo di mira. Il file del database più un keylogger per la master password danno accesso a ogni credenziale memorizzata.

**pass** (password-store) in `~/.password-store/` usa file cifrati con GPG organizzati in directory. Ogni file `.gpg` è una credenziale. Il numero di voci rivela la portata delle credenziali memorizzate.

**Bitwarden** memorizza i dati locali del vault in `~/.config/Bitwarden/` e `~/.config/Bitwarden CLI/`. Come KeePass, il vault è cifrato, ma la sua presenza e accessibilità meritano di essere documentate.

## Archiviazione delle credenziali Git

### Credenziali Git in chiaro

Il file `~/.git-credentials` memorizza le credenziali in chiaro in formato URL: `https://user:token@github.com`. Questo file viene creato quando si usa il credential helper `store` (`git config credential.helper store`). Ogni riga è un URL completo con autenticazione incorporata. Ha gravità HIGH per impostazione predefinita e CRITICAL se leggibile da chiunque.

### Credential helper

I file di configurazione di Git (`~/.gitconfig`, `~/.config/git/config`) specificano i credential helper. L'helper `store` salva in `.git-credentials` in chiaro. Altri helper come `cache` (temporaneo in memoria), `osxkeychain` o `gnome-keyring` sono più sicuri, ma la loro configurazione rivela comunque come l'utente gestisce l'autenticazione Git.

### Pattern dei token

I personal access token di GitHub seguono prefissi noti: `ghp_` (PAT classico), `gho_` (OAuth), `ghu_` (user-to-server), `ghs_` (server-to-server), `ghr_` (refresh). I token GitLab iniziano con `glpat-`. Trovare questi pattern nei file di configurazione di Git significa che i token sono stati inseriti in modo hardcoded, probabilmente per errore.

### Nel mondo reale: il leak di Mercedes-Benz

Nel gennaio 2024, dei ricercatori di sicurezza hanno trovato un token GitHub in un repository pubblico di Mercedes-Benz che forniva accesso illimitato al GitHub Enterprise Server interno dell'azienda. Il token, probabilmente sottoposto a commit per errore, esponeva codice sorgente, chiavi di accesso al cloud, progetti e documenti di design interni. È esattamente la classe di esposizione che la scansione delle credenziali Git rileva.

## Token applicativi e credenziali dei database

### File di credenziali dei database

Diversi client di database supportano file di credenziali nella home directory:

- `~/.pgpass` - File delle password di PostgreSQL. Formato: `hostname:port:database:username:password`, una voce per riga. PostgreSQL impone i permessi 0600 su questo file, ma non impedisce che il file esista con permessi peggiori
- `~/.my.cnf` - Configurazione del client MySQL. Può contenere sezioni `[client]` con voci `password=`
- `~/.rediscli_auth` - Credenziali di autenticazione della CLI di Redis
- `~/.mongorc.js` - File di avvio della shell di MongoDB, può contenere comandi di autenticazione

### Token di sviluppo

- `~/.npmrc` - Autenticazione al registry npm. Contiene `_authToken=` per la pubblicazione dei pacchetti
- `~/.pypirc` - Credenziali PyPI per la pubblicazione di pacchetti Python
- `~/.config/gh/hosts.yml` - Token OAuth della GitHub CLI

### Token di infrastruttura

- `~/.terraform.d/credentials.tfrc.json` - Token API di Terraform Cloud
- `~/.vault-token` - Token di autenticazione di HashiCorp Vault
- `~/.config/helm/repositories.yaml` - Credenziali dei repository di chart Helm
- `~/.config/rclone/rclone.conf` - Credenziali di cloud storage di Rclone (possono contenere token OAuth o chiavi API)
- `~/.docker/config.json` - Token di autenticazione ai registry Docker

### Dati applicativi

Le applicazioni desktop memorizzano i dati di sessione localmente:
- Slack (`~/.config/Slack/`) - Token di sessione dei workspace
- Discord (`~/.config/discord/`) - Token di autenticazione
- VS Code (`~/.config/Code/`) - Credenziali delle estensioni, che potenzialmente includono chiavi SSH e token API nelle impostazioni

## Errori comuni

**Presumere che la cifratura significhi sicurezza.** Una chiave SSH cifrata con permessi 0644 è comunque un rilevamento CRITICAL. La chiave cifrata può essere esfiltrata e la passphrase forzata con brute force offline con strumenti come John the Ripper. La cifratura è defense-in-depth, non un sostituto di permessi corretti.

**Ignorare i file "solo di configurazione".** La configurazione AWS (`~/.aws/config`) non contiene segreti direttamente, ma rivela la struttura dell'account, le regioni e se la MFA è richiesta. Una configurazione Kubernetes senza token mostra comunque gli endpoint dei cluster e i namespace. Questi dati di ricognizione aiutano gli attaccanti a pianificare ulteriori sfruttamenti.

**Dimenticarsi dei file temporanei.** La cronologia della shell si accumula nel tempo. Un segreto esportato sei mesi fa è ancora in `.bash_history` a meno che non venga ripulito manualmente. I file d'ambiente nelle directory dei progetti potrebbero essere stati creati durante lo sviluppo e mai rimossi dopo il deploy.

**Fidarsi dei sistemi a utente singolo.** "Sono l'unico utente su questa macchina" non significa che le credenziali siano al sicuro. Malware, container escape, exploit dei browser e applicazioni compromesse girano tutti come processi con un certo livello di accesso al file system. Le credenziali leggibili da chiunque sono accessibili a tutti loro.

## Come si collegano questi concetti

```
                            ┌──────────────────────┐
                            │   Initial Access      │
                            │  (phishing, exploit,  │
                            │   stolen creds, etc)  │
                            └──────────┬───────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │  Post-Access          │
                            │  Enumeration          │
                            │  (this tool)          │
                            └──────────┬───────────┘
                                       │
              ┌──────────────┬─────────┼─────────┬────────────────┐
              ▼              ▼         ▼         ▼                ▼
      ┌──────────────┐ ┌─────────┐ ┌───────┐ ┌─────────┐ ┌───────────┐
      │ SSH Keys     │ │ Cloud   │ │ Git   │ │ Browser │ │ App       │
      │ T1552.004    │ │ T1552.001│ │Tokens │ │ T1555.003│ │ Tokens    │
      └──────┬───────┘ └────┬────┘ └───┬───┘ └────┬────┘ └─────┬─────┘
             │              │          │           │            │
             ▼              ▼          ▼           ▼            ▼
      ┌──────────────┐ ┌─────────┐ ┌───────┐ ┌─────────┐ ┌───────────┐
      │ Lateral      │ │ Cloud   │ │ Source │ │ Session │ │ Database  │
      │ Movement     │ │ Pivot   │ │ Code  │ │ Hijack  │ │ Access    │
      │              │ │         │ │ Access │ │         │ │           │
      └──────────────┘ └─────────┘ └───────┘ └─────────┘ └───────────┘
```

Ogni tipo di credenziale abilita un diverso percorso di escalation. Le chiavi SSH abilitano il movimento laterale verso altri server. Le credenziali cloud consentono il pivot nell'infrastruttura cloud. I token Git espongono i repository di codice sorgente. I cookie del browser abilitano il session hijacking. I token applicativi danno accesso diretto a database e servizi.

Il filo conduttore sono i permessi dei file. Ogni rilevamento di questo tool si riduce a: il file di credenziali è accessibile a più entità di quanto dovrebbe, e la credenziale stessa è protetta (cifrata, protetta da passphrase) o in chiaro?

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Perché una chiave SSH non cifrata con permessi 0600 è classificata HIGH anziché CRITICAL? Cosa la porterebbe a CRITICAL?
2. Un attaccante trova `~/.aws/credentials` con due profili: uno che usa chiavi `AKIA` e uno che usa chiavi `ASIA`. Quale è più preoccupante e perché?
3. Perché questo tool scansiona i file `.env` in modo ricorsivo ma limita la profondità a 5 directory? Cosa succederebbe senza un limite di profondità?
4. Un file logins.json di Firefox è cifrato. Perché è comunque un rilevamento?
5. In che modo la scansione della cronologia della shell differisce dalla scansione dei file d'ambiente in termini di ciò che viene rilevato e del perché la gravità è diversa?

## Letture di approfondimento

**Essenziali:**
- [MITRE ATT&CK: Unsecured Credentials](https://attack.mitre.org/techniques/T1552/) - La mappatura del framework per tutto ciò che questo tool rileva
- [MITRE ATT&CK: Credentials from Password Stores](https://attack.mitre.org/techniques/T1555/) - Furto di credenziali da browser e keyring
- [CIS Benchmarks for Linux](https://www.cisecurity.org/benchmark/distribution_independent_linux) - Raccomandazioni di hardening per i permessi dei file

**Approfondimenti:**
- [Uber Security Incident Report (2022)](https://www.uber.com/newsroom/security-update/) - La raccolta di credenziali post-accesso nella pratica
- [CircleCI Security Incident (2023)](https://circleci.com/blog/jan-4-2023-incident-report/) - Furto di token dal browser che porta alla compromissione della piattaforma
- [LastPass Security Incident (2022-2023)](https://blog.lastpass.com/2023/03/security-incident-update-recommended-actions/) - Furto di credenziali da una macchina domestica che porta a una violazione della produzione

**Contesto storico:**
- [Codecov Supply Chain Attack (2021)](https://about.codecov.io/security-update/) - Esfiltrazione di variabili d'ambiente su larga scala
- [Twitch Source Code Leak (2021)](https://blog.twitch.tv/en/2021/10/06/updates-on-the-twitch-security-incident/) - Credenziali rubate che consentono l'accesso al codice sorgente
