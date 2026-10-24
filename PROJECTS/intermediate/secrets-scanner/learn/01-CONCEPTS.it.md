# Concetti fondamentali di sicurezza

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci Portia. Non sono semplici definizioni. Approfondiremo perché sono importanti e come funzionano davvero nei sistemi in produzione.

## Secret Sprawl

### Cos'è

Il secret sprawl è la diffusione incontrollata di credenziali, API key, token e altri valori sensibili nel codice sorgente, nei file di configurazione, nelle pipeline CI/CD, nei log delle chat e nella documentazione. Una volta che un secret entra in un repository git, esiste in ogni clone per sempre, anche se il file viene poi eliminato.

### Perché è importante

GitHub ha riportato nel 2023 di aver rilevato oltre 12,8 milioni di nuovi secret trapelati nei repository pubblici in quell'anno. Quel numero copre solo i secret che cercano attivamente. Il numero reale è significativamente più alto.

**Uber, novembre 2016**
Due ingegneri hanno committato delle AWS access key in un repository GitHub privato. Sei mesi dopo, degli attaccanti che avevano accesso a quel repo hanno usato le chiavi per accedere a un bucket S3 contenente 57 milioni di record di passeggeri e autisti (nomi, indirizzi email, numeri di telefono, numeri di patente). Uber ha pagato agli attaccanti 100.000 $ per cancellare i dati e tacere, poi ha pagato 148 milioni di $ in un accordo con tutti i 50 procuratori generali degli stati USA. Le chiavi corrispondevano esattamente al pattern del prefisso `AKIA` che la regola `aws-access-key-id` di Portia rileva (`internal/rules/builtin.go`).

**Samsung, marzo 2022**
Il gruppo Lapsus$ ha esfiltrato 190GB di codice sorgente Samsung dai server Git interni. All'interno del codice: credenziali hardcoded per la piattaforma SmartThings di Samsung, chiavi di firma private e codice sorgente di bootloader e TrustZone. La superficie d'attacco esisteva perché i secret erano incorporati direttamente nel codice applicativo anziché iniettati a runtime.

**Twitter, gennaio 2023**
Un ricercatore ha scoperto che le app mobile di Twitter contenevano API key incorporate che potevano essere estratte tramite reverse engineering. Twitter ha riconosciuto il problema e ruotato le chiavi, ma l'incidente ha evidenziato un problema fondamentale: le app mobile distribuiscono codice compilato su milioni di dispositivi, e qualsiasi secret in quel codice è estraibile.

### Come funziona

I secret finiscono nel codice attraverso tre percorsi principali:

**Scorciatoie di sviluppo**: Uno sviluppatore ha bisogno di una API key per testare una funzionalità. La incolla in un file di configurazione, la committa e pianifica di spostarla nelle variabili d'ambiente più tardi. Non lo fa mai. La chiave resta per sempre nella history di git. Anche eliminare il file e committare l'eliminazione non aiuta, perché `git log --all --full-history` lo mostra ancora.

**Copia-incolla dalla documentazione**: I cloud provider spesso mostrano comandi di esempio con chiavi segnaposto come `AKIAIOSFODNN7EXAMPLE`. Gli sviluppatori sostituiscono il segnaposto con la loro chiave reale, dimenticano di ripristinarlo e committano il file.

**Configurazione CI/CD**: I secret sono impostati come variabili d'ambiente nelle pipeline CI (GitHub Actions, CircleCI, Jenkins). Quando uno sviluppatore copia un file di workflow o esporta la configurazione di una pipeline, quei valori finiscono in file sotto controllo di versione. La violazione di CircleCI di gennaio 2023 è avvenuta in parte perché i secret dei clienti erano memorizzati come variabili d'ambiente in chiaro accessibili ai job CI.

### Strategie di difesa

Portia implementa la scansione pre-commit, che intercetta i secret prima che entrino nella history di git. Ma la scansione è solo un livello:

1. **Hook pre-commit** - Esegui Portia sui file in staging prima di ogni commit
2. **Scansione nella pipeline CI** - Esegui Portia su ogni pull request come controllo obbligatorio
3. **Integrazione con Vault** - Usa HashiCorp Vault, AWS Secrets Manager o strumenti simili per iniettare i secret a runtime
4. **Variabili d'ambiente** - Tieni i secret completamente fuori dal codice. Fai riferimento a `${API_KEY}` invece di incollare la chiave.
5. **Rotazione** - Parti dal presupposto che i secret trapeleranno. Ruotali regolarmente così le chiavi trapelate scadono rapidamente.

## Entropia di Shannon

### Cos'è

L'entropia di Shannon (dal nome di Claude Shannon, il padre della teoria dell'informazione) misura la casualità di una stringa. Risponde alla domanda: "Quanto mi sorprenderebbe ogni carattere?" Il testo inglese ha bassa entropia (puoi prevedere la lettera successiva). Una API key come `xK9mP2vL5nQ8jR3t` ha alta entropia (ogni carattere è imprevedibile).

La formula: `H = -Σ p(x) * log₂(p(x))`

Dove `p(x)` è la frequenza di ciascun carattere divisa per la lunghezza della stringa.

### Perché è importante

La sola regex non basta per rilevare tutti i secret. Alcune regole corrispondono a pattern strutturali (prefisso `AKIA` per le chiavi AWS, prefisso `ghp_` per i token GitHub). Ma molti secret non hanno alcun prefisso strutturale. Una password come `xK9mP2vL5nQ8jR3t` sembra una sequenza di caratteri casuali, e quella casualità è essa stessa il segnale.

Il testo leggibile da un essere umano (nomi di variabili, commenti, URL) ha un'entropia di circa 2,5-3,5 bit per carattere. I secret generati casualmente hanno tipicamente un'entropia superiore a 4,0. Calcolando l'entropia e confrontandola con una soglia, Portia può segnalare stringhe ad alta casualità che non corrispondono ad alcun pattern regex specifico.

### Come funziona

L'implementazione in `internal/rules/entropy.go` funziona in tre passaggi:

**Passaggio 1: Rilevamento del charset** - Determina se la stringa sembra esadecimale (`0-9a-f`), base64 (`A-Za-z0-9+/=`) o alfanumerica generica. Questo conta perché la soglia di entropia è diversa: una stringa hex ha bisogno di un'entropia più bassa per essere sospetta (dato che l'hex ha solo 16 caratteri possibili), mentre le stringhe alfanumeriche richiedono un'entropia più alta.

**Passaggio 2: Conteggio della frequenza dei caratteri** - Conta quante volte compare ciascun carattere. Per `aabbcc`, le frequenze sono: `a=2, b=2, c=2`. Ogni carattere ha probabilità 2/6 = 0,333.

**Passaggio 3: Calcolo dell'entropia** - Per ogni carattere unico, calcola `-p * log₂(p)` e somma i risultati. Per `aabbcc`: `-0.333 * log₂(0.333) * 3 unique chars = 1.585 bits`. Per una stringa casuale di 20 caratteri tutti unici: `-0.05 * log₂(0.05) * 20 = 4.322 bits`.

**Soglie usate in Portia:**
- Stringhe hex (charset `0-9a-f`): minimo 3,0 bit
- Stringhe base64 (charset `A-Za-z0-9+/=`): minimo 4,0 bit
- Stringhe alfanumeriche: minimo 3,5 bit

Queste soglie sono configurate per regola in `internal/rules/builtin.go`. Regole come `generic-password` e `generic-secret` usano soglie di entropia per filtrare falsi positivi a bassa casualità come `password = "admin"`.

### Insidie comuni

**Bassa entropia non significa sicuro.** La stringa `aaaaaaaaAAAAAAAA` ha bassa entropia ma potrebbe comunque essere una API key valida per un sistema progettato male. L'entropia è un segnale, non l'unico segnale.

**Il charset conta.** Una stringa hex `deadbeef` ha bassa entropia assoluta ma potrebbe essere sospetta in un contesto hex. La funzione `DetectCharset` di Portia (`internal/rules/entropy.go`) adegua le soglie in base al set di caratteri.

## Rilevamento basato su espressioni regolari

### Cos'è

La maggior parte dei cloud provider e dei servizi usa formati strutturati per le proprie credenziali. Le access key AWS iniziano sempre con `AKIA`, `ABIA`, `ACCA` o `ASIA` seguiti da 16 caratteri alfanumerici maiuscoli. I PAT di GitHub iniziano con `ghp_` seguito da 36 caratteri alfanumerici. Le chiavi live di Stripe iniziano con `sk_live_`. Questi pattern sono abbastanza specifici da essere individuati con espressioni regolari mantenendo tassi di falsi positivi molto bassi.

### Perché è importante

Il rilevamento basato sul prefisso funziona perché i cloud provider progettano intenzionalmente i formati delle loro chiavi in modo che siano identificabili. AWS usa il prefisso `AKIA` proprio perché i tool possano rilevare le chiavi trapelate. GitHub ha cambiato il formato dei token da stringhe esadecimali casuali a token con prefisso (`ghp_`, `gho_`, `ghs_`, `ghr_`) nel 2021 per abilitare esattamente questo tipo di scansione.

### Come funziona

Ogni regola in `internal/rules/builtin.go` ha quattro componenti che lavorano insieme:

**Keywords** - Matching rapido di stringhe per eliminare circa il 95% dei chunk prima della regex. Se un chunk non contiene la keyword `AKIA`, non ha senso eseguire su di esso la regex delle chiavi AWS. È un'ottimizzazione delle prestazioni. Vedi la funzione MatchKeywords in `internal/rules/registry.go`.

**Pattern** - La regex vera e propria. Per le chiavi AWS: `\b((?:AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16})\b`. I word boundary `\b` impediscono di corrispondere a sottostringhe di token più lunghi. Il capture group `(...)` estrae solo la porzione di secret.

**SecretGroup** - Quale capture group della regex contiene il valore del secret. Il gruppo 0 è l'intero match; il gruppo 1 è il primo gruppo tra parentesi. La maggior parte delle regole usa il gruppo 1 per estrarre la chiave senza il contesto circostante.

**Allowlist** - Pattern per regola che sopprimono falsi positivi noti. Le chiavi AWS hanno un'allowlist per `AKIAIOSFODNN7EXAMPLE` (la chiave di esempio della documentazione AWS).

### 150 regole organizzate per provider

Le regole builtin di Portia coprono:
- **Cloud provider**: AWS (3), GCP (3), Azure (3), Alibaba Cloud (1), IBM Cloud (1), Cloudflare (2)
- **Source control**: GitHub (6), GitLab (3), Bitbucket (1)
- **Pagamenti**: Stripe (4), Square (2), Razorpay (1), Braintree (1), Coinbase (1)
- **Comunicazione**: Slack (4), Twilio (3), SendGrid (1), Discord (3), Telegram (1), Microsoft Teams (1), Intercom (1)
- **Email**: Mailchimp (1), Mailgun (1), Resend (1), Brevo (1), Postmark (1)
- **Infrastruttura**: Heroku (1), DigitalOcean (3), Supabase (2), Confluent (1), Fly.io (1), Render (1), Vercel (1), Netlify (1), PlanetScale (3), Neon (1), Upstash (1), Turso (1)
- **Hosting/VPS**: Hetzner (1), Linode (1), Vultr (1)
- **AI/ML**: OpenAI (2), Anthropic (1), HuggingFace (1), Replicate (1), Groq (1), Perplexity (1)
- **CI/CD**: CircleCI (1), Buildkite (1), GitHub Actions (1)
- **Gestione dei secret**: Vault (1), Doppler (2), 1Password (1)
- **Crittografiche**: chiavi SSH/PGP/PKCS8 (6), JWT (1), Age (1)
- **Package registry**: NPM (1), PyPI (1), RubyGems (1), Docker Hub (1)
- **Generiche**: password (1), secret (1), api-key (1), token (1)
- **Connection string**: PostgreSQL (1), MySQL (1), MongoDB (1), Redis (1), Firebase URL (1), Cloudinary (1)
- **Monitoring/Observability**: Datadog (1), New Relic (1), Grafana (2), Sentry (1), PostHog (1), Axiom (1), Dynatrace (1), Honeycomb (1), Elastic (1), Segment (1), Rollbar (1), Mixpanel (1), Amplitude (1)
- **Strumenti per sviluppatori**: Figma (1), Linear (1), Postman (1), Algolia (1), Contentful (1), Snyk (1), SonarQube (1), Freshdesk (1), Zendesk (1)
- **Altro**: Terraform (1), Shopify (4), Okta (1), LaunchDarkly (2), Infracost (1), Prefect (1), Pulumi (1), Databricks (1), HubSpot (1), PagerDuty (1), Atlassian (1), Facebook (1), Twitter (1), Firebase FCM (1), Mapbox (2), Doppler CLI (1)

## Riduzione dei falsi positivi

### Cos'è

Uno scanner basato su regex ingenuo produce centinaia o migliaia di falsi positivi: valori segnaposto, variabili di template, fixture di test, chiavi di esempio dalla documentazione. Se l'output è rumoroso, gli sviluppatori lo ignorano. La riduzione dei falsi positivi è l'insieme di tecniche che rendono l'output dello scanner abbastanza affidabile da agire su di esso.

### Perché è importante

Il report State of Secrets Sprawl 2023 di GitGuardian ha rilevato che i team che usavano scanner di secret con alti tassi di falsi positivi avevano un tasso di remediation inferiore del 40% rispetto ai team con scanner a basso tasso di falsi positivi. Il rumore uccide l'azione. Se un finding su due è un falso positivo, gli sviluppatori smettono di leggere i finding.

### Come funziona

Portia usa una catena di filtri a 5 livelli in `internal/engine/filter.go` e `internal/engine/detector.go`:

**Livello 1: Pre-filtraggio per keyword** (`internal/rules/registry.go` MatchKeywords)
Prima di eseguire qualsiasi regex, controlla se il chunk contiene almeno le keyword rilevanti per una qualche regola. Un file pieno di HTML senza alcuna occorrenza di `password`, `secret`, `key`, `token` o di un qualsiasi prefisso specifico di un provider non corrisponderà ad alcuna regola. Questo elimina subito circa il 95% dei chunk ed è una pura ottimizzazione delle prestazioni.

**Livello 2: Validazione strutturale** (`internal/engine/detector.go` extractSecret)
Dopo i match della regex, estrai il secret usando il capture group. Se il capture group è vuoto o il match è solo whitespace, scartalo. Questo gestisce i casi limite in cui una regex corrisponde alla sintassi circostante ma non cattura nulla di significativo.

**Livello 3: Rilevamento dei placeholder** (`internal/engine/filter.go` IsPlaceholder)
Controlla il valore del secret rispetto a pattern segnaposto noti: `example`, `test`, `dummy`, `fake`, `placeholder`, `YOUR_API_KEY`, `xxxx...`, `****...`, `${VARIABLE}`, `{{TEMPLATE}}`, `TODO`, `CHANGEME`, `REPLACE_ME`. Questi sono definiti in `internal/rules/registry.go` GlobalValueAllowlist.

**Livello 4: Rilevamento dei template** (`internal/engine/filter.go` IsTemplated)
Controlla se il secret è una variabile di template: `${ENV_VAR}`, `{{handlebars}}`, `os.getenv(...)`, `process.env.X`, `System.getenv(...)`, `ENV[...]`. Questi non sono secret reali; sono riferimenti a secret memorizzati altrove.

**Livello 5: Filtraggio per stopword** (`internal/engine/filter.go` IsStopword)
Controlla se il secret, una volta diviso sui delimitatori (`_`, `-`, `.`, `/`), contiene parole di programmazione comuni come `function`, `controller`, `database`, `config`, ecc. L'elenco di stopword in `internal/engine/filter.go` contiene oltre 700 parole. Questo intercetta falsi positivi come `module_controller_factory` che la regex potrebbe far corrispondere a un token generico.

**Livello 6: Path allowlisting** (`internal/rules/registry.go` GlobalPathAllowlist)
Salta i file noti per contenere dati non sensibili: `go.mod`, `go.sum`, `package-lock.json`, `yarn.lock`, file `.min.js`, `node_modules/`, `vendor/`, formati binari.

### Insidia comune

Il più grande errore nella riduzione dei falsi positivi: essere troppo aggressivi. Se filtri troppi risultati, ti sfuggiranno secret reali. L'approccio di Portia è conservativo per impostazione predefinita. Ogni livello di filtro prende di mira una classe specifica di falsi positivi con alta precisione. L'elenco di stopword corrisponde solo alle parti separate dai delimitatori (non alle sottostringhe) per evitare di filtrare secret reali che contengono per caso parole comuni come sottostringhe.

## HIBP k-Anonymity

### Cos'è

Have I Been Pwned (HIBP) è il database di Troy Hunt con miliardi di password e credenziali provenienti da data breach noti. Portia può verificare se un secret rilevato compare in qualche breach noto. Ma inviare il secret a una API esterna vanificherebbe lo scopo. La k-Anonymity risolve il problema: invii solo 5 caratteri dell'hash SHA-1 del secret, e il server restituisce tutti gli hash che condividono quel prefisso. Controlli localmente se il tuo hash completo compare nei risultati.

### Perché è importante

Se Portia trova `password = "P@ssw0rd123"` nel tuo codice, sapere che compare in 47.000 breach noti aggiunge urgenza. Una password unica, generata casualmente, che non è mai stata compromessa è meno urgente di una password di uso comune presente in ogni wordlist di credential stuffing.

Il dump del breach Collection #1 del 2019 conteneva 773 milioni di indirizzi email unici e 21 milioni di password uniche. Il leak RockYou2021 del 2021 conteneva 8,4 miliardi di password. Se il tuo secret rilevato compare in questi dataset, gli attaccanti lo hanno già.

### Come funziona

Implementazione in `internal/hibp/client.go`:

1. **Hash SHA-1**: Calcola `SHA-1(secret)`. Per "P@ssw0rd123": `a94a8fe5ccb19ba61c4c0873d391e987982fbbd3`
2. **Estrazione del prefisso**: Prendi i primi 5 caratteri esadecimali: `a94a8`
3. **Query all'API**: `GET https://api.pwnedpasswords.com/range/a94a8`
4. **Risposta**: L'API restituisce circa 500-800 suffissi di hash con i conteggi delle occorrenze
5. **Matching locale**: Controlla se i restanti 35 caratteri del tuo hash compaiono nella risposta

L'API non vede mai l'hash completo, quindi non può determinare quale password stai controllando. Questa è la garanzia della k-anonymity.

**Cache LRU** (`internal/hibp/client.go`): I risultati sono memorizzati in una cache LRU da 10.000 voci. Se lo stesso prefisso è già stato interrogato in precedenza, viene restituito il risultato in cache. Questo evita chiamate API ridondanti quando si scansionano grandi codebase con secret simili.

**Circuit breaker** (`internal/hibp/client.go`): Usando la libreria gobreaker di Sony, il client tiene traccia dei fallimenti dell'API. Dopo 5 errori consecutivi, il circuito si apre e le richieste vengono rifiutate immediatamente per 60 secondi. Questo previene i fallimenti a cascata se la API di HIBP è giù o sottoposta a rate limiting.

### Insidia comune

HIBP è progettato per le password, non per le API key. Una AWS access key trapelata non comparirà nei database dei breach di password. Il controllo HIBP è più utile per i finding di tipo password (le regole `generic-password` e `generic-secret`) e meno utile per i token specifici di un provider. Portia invia a HIBP solo i finding di password e generic-secret, saltando del tutto i token specifici dei provider.

## Formato di output SARIF

### Cos'è

SARIF (Static Analysis Results Interchange Format) è uno standard OASIS (versione 2.1.0) per esprimere l'output dei tool di analisi statica. È un formato basato su JSON che GitHub, Azure DevOps, GitLab e altre piattaforme CI possono consumare per mostrare i finding direttamente nelle pull request.

### Perché è importante

Senza SARIF, integrare uno scanner nella CI richiede una logica di parsing personalizzata per il formato di output di ogni tool. Con SARIF, produci un unico file JSON e ogni piattaforma CI importante sa come mostrarlo. GitHub Code Scanning, per esempio, accetta upload SARIF e mostra i finding come annotazioni sul diff della PR.

### Come funziona

Il reporter SARIF di Portia in `internal/reporter/sarif.go` produce un documento conforme alla v2.1.0:

```json
{
  "$schema": "https://json.schemastore.org/sarif-2.1.0.json",
  "version": "2.1.0",
  "runs": [{
    "tool": {
      "driver": {
        "name": "portia",
        "rules": [...]
      }
    },
    "results": [...]
  }]
}
```

Ogni finding corrisponde a un `result` SARIF con:
- `ruleId` - L'ID della regola Portia (ad es. `aws-access-key-id`)
- `level` - Mappato dalla severità di Portia (CRITICAL/HIGH = `error`, MEDIUM = `warning`, LOW = `note`)
- `message` - La descrizione del finding
- `locations` - Percorso del file e numero di riga
- `properties` - Metadati specifici di Portia (entropia, stato HIBP, secret mascherato)

Per fare l'upload su GitHub Code Scanning:
```bash
portia scan --format sarif . > results.sarif
gh api repos/{owner}/{repo}/code-scanning/sarifs \
  -f sarif=@results.sarif \
  -f commit_sha=$(git rev-parse HEAD)
```
