# Concetti di sicurezza fondamentali

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci questo progetto. Non sono solo definizioni. Approfondiremo perché sono importanti e come funzionano davvero.

## Fondamenti del rate limiting

### Che cos'è

Il rate limiting controlla quante richieste un client può fare a un'API all'interno di una finestra temporale. Invece di accettare traffico illimitato, definisci regole come "100 richieste al minuto" o "5 tentativi di login all'ora". Quando un client supera il limite, rifiuti le sue richieste con HTTP 429 (Too Many Requests) o, nel caso di questo progetto, HTTP 420 (Enhance Your Calm).

Il meccanismo di base è un contatore. Per ogni client, tieni traccia di quante richieste ha fatto nella finestra temporale corrente. Incrementalo a ogni richiesta. Se il conteggio supera il limite, rifiuta la richiesta. Azzera il contatore quando la finestra scade.

### Perché è importante

Nel settembre 2022, Uber ha subito una violazione completa perché un attaccante ha fatto brute force dei codici MFA sull'account di un dipendente. L'attaccante ha fatto centinaia di tentativi finché non ha indovinato. L'assenza di rate limiting sull'endpoint MFA significava tentativi illimitati. Con un limite di 5 tentativi, l'attacco sarebbe fallito immediatamente.

Il rate limiting previene:
- **Esaurimento delle risorse** - Un client che consuma tutta la capacità del tuo server
- **Costi fuori controllo** - Se paghi per ogni chiamata API a terze parti, richieste illimitate = bollette illimitate
- **Degrado del servizio** - Client lenti che monopolizzano le connessioni, lasciando senza risorse i client veloci

Senza rate limiting, un singolo ciclo `while True: requests.get()` può mandare giù l'intera API.

### Come funziona

L'implementazione più semplice usa un contatore e un timestamp:
```python
# Naive approach (don't use this)
counters = {}

def check_rate_limit(client_id, limit, window_seconds):
    now = time.time()
    
    if client_id not in counters:
        counters[client_id] = {"count": 0, "window_start": now}
    
    client_data = counters[client_id]
    
    # Reset if window expired
    if now - client_data["window_start"] > window_seconds:
        client_data["count"] = 0
        client_data["window_start"] = now
    
    # Check limit
    if client_data["count"] >= limit:
        return False  # Rate limited
    
    client_data["count"] += 1
    return True  # Allowed
```

Questo ha delle race condition. Due richieste concorrenti possono entrambe leggere `count=99`, entrambe incrementare a 100 ed essere entrambe permesse quando il limite è 100. Ti servono operazioni atomiche, ed è per questo che questo progetto usa script Lua di Redis (`src/fastapi_420/storage/lua/sliding_window.lua:1-32`).

### Attacchi comuni

1. **Aggiramento distribuito del rate limit** - L'attaccante usa 1000 indirizzi IP per fare 99 richieste ciascuno (sotto il tuo limite di 100/minuto). Totale: 99.000 richieste/minuto. Difesa: aggiungi un limite globale per endpoint oltre ai limiti per utente. Vedi `src/fastapi_420/defense/layers.py:128-169`.

2. **Sfruttamento dei confini delle finestre** - Con le finestre fisse, se la finestra si azzera a :00, un attaccante fa 100 richieste a :59 e 100 a :01. Sono 200 richieste in 2 secondi, aggirando il tuo limite di "100/minuto". Difesa: usa invece le sliding window. Vedi il concetto più sotto.

3. **Spoofing degli header** - L'attaccante fa ruotare gli header `User-Agent` per apparire come client diversi. Difesa: fai fingerprinting di più attributi (IP + UA + Auth). Vedi `src/fastapi_420/fingerprinting/composite.py:46-95`.

### Strategie di difesa

Questo progetto implementa una difesa a livelli:

**Livello 1: Per utente per endpoint** - Limiti stretti per i singoli client. Ferma l'abuso da parte di un singolo utente.
**Livello 2: Globale per endpoint** - Impedisce che un endpoint venga sopraffatto da attacchi distribuiti.
**Livello 3: Circuit breaker globale** - Protegge l'intera API quando è sotto un DDoS su larga scala.

Il controllo avviene in `src/fastapi_420/defense/layers.py:47-84`. Tutti e tre i livelli devono essere superati affinché una richiesta vada a buon fine.

## Confronto tra algoritmi

### Sliding Window Counter

**Che cos'è:**
Approssima una vera sliding window usando due finestre fisse. Interpola tra la finestra corrente e quella precedente in base al tempo trascorso all'interno della finestra corrente. Raggiunge un'accuratezza del 99,997% con memoria O(1) per client.

La formula (da `src/fastapi_420/storage/lua/sliding_window.lua:18-19`):
```lua
weighted_count = previous_count * (1 - elapsed_ratio) + current_count
```

Se sei a 30 secondi in una finestra di 60 secondi, `elapsed_ratio = 0.5`. La finestra precedente riceve un peso del 50%, quella corrente del 100%.

**Perché è importante:**
È l'impostazione predefinita consigliata per la produzione perché elimina il problema dei burst ai confini delle finestre usando una memoria minima. Le vere sliding window richiedono di memorizzare il timestamp di ogni richiesta, consumando memoria O(n) dove n è il limite. Questo approccio usa memoria O(1) (solo due contatori) indipendentemente dalla dimensione del limite.

**Implementazione:**
Vedi `src/fastapi_420/algorithms/sliding_window.py:31-43`. L'algoritmo delega ai backend di storage, che gestiscono il calcolo pesato in modo atomico.

### Token Bucket

**Che cos'è:**
Immagina un secchio che contiene dei token. Ogni richiesta consuma un token. I token si ricaricano a un ritmo costante. Se il secchio è vuoto, le richieste vengono rifiutate. Il secchio ha una capacità massima, che permette burst fino a quella dimensione.

Implementazione in `src/fastapi_420/storage/lua/token_bucket.lua:17-26`:
```lua
local elapsed = now - last_refill
local tokens_to_add = elapsed * refill_rate
tokens = math.min(capacity, tokens + tokens_to_add)
```

**Quando usarlo:**
API che gestiscono upload di file, elaborazione di immagini o altri carichi di lavoro a burst. Un client potrebbe restare inattivo per minuti e poi caricare improvvisamente 10 file. Il token bucket permette il burst (se capacity=10) facendo comunque rispettare il ritmo medio nel tempo.

**Compromesso:**
È più complesso da ragionare rispetto alle finestre fisse/sliding. I client devono capire sia la capacità di burst sia il ritmo di ricarica. La documentazione diventa più difficile: "Puoi fare 100 richieste immediatamente, poi 1,67 richieste al secondo da lì in poi" contro un semplice "100 al minuto".

### Fixed Window

**Che cos'è:**
Dividi il tempo in finestre fisse (00:00-00:59, 01:00-01:59). Conta le richieste nella finestra corrente. Azzera il contatore quando la finestra scade.

Implementazione in `src/fastapi_420/storage/lua/fixed_window.lua:13-16`:
```lua
local current_window = math.floor(now / window_seconds)
local window_key = key .. ":" .. current_window
local count = redis.call('GET', window_key) or 0
```

**Problema dei burst ai confini:**
Alle 12:00:59, il client fa 100 richieste (limite raggiunto).
Alle 12:01:00, la finestra si azzera. Il client fa altre 100 richieste.
Risultato: 200 richieste in 2 secondi, violando del 100% il limite previsto di "100/minuto".

Questo è sfruttabile. Un attaccante che conosce i confini delle tue finestre può raddoppiare il suo ritmo effettivo.

**Quando usarlo:**
Solo se ti serve la massima semplicità e puoi tollerare i burst ai confini. Per esempio, tool di amministrazione interni in cui la precisione non conta.

## Client Fingerprinting

### Che cos'è

Identificare chi sta facendo una richiesta. Sembra semplice: basta usare l'indirizzo IP. Ma non è così semplice.

**Problema 1: NAT e proxy**
Un edificio di uffici potrebbe avere 500 dipendenti che condividono un unico IP pubblico tramite NAT. Se applichi il rate limit per IP, tutte le 500 persone condividono lo stesso limite. L'abuso dell'API da parte di una persona blocca tutti.

**Problema 2: blocchi IPv6 /64**
Gli utenti controllano interi prefissi IPv6 /64 (18 quintilioni di indirizzi). Un attaccante può ruotare tra milioni di IP senza limitazioni. La soluzione in `src/fastapi_420/fingerprinting/ip.py:89-107` normalizza gli indirizzi IPv6 al loro prefisso /64, trattando tutti gli IP di un blocco come un'unica identità.

**Problema 3: reti mobili**
Gli operatori mobili usano il carrier-grade NAT (CGNAT). Il tuo IP cambia ogni pochi minuti mentre ti sposti tra le celle. Il rate limiting basato sugli IP penalizza gli utenti mobili legittimi.

### Perché è importante

Nel 2020, l'app SNKRS di Nike ha subito attacchi di bot durante i lanci di sneaker in edizione limitata. I bot usavano reti di proxy residenziali (indirizzi IP reali di utenti domestici) per aggirare i limiti basati sugli IP. Il rate limiting di Nike era inefficace perché ogni richiesta dei bot proveniva da un IP diverso e dall'aspetto legittimo.

La soluzione è il fingerprinting composito. Combina più attributi:
- Indirizzo IP (normalizzato per IPv6)
- Stringa User-Agent
- Header Accept-Language e Accept-Encoding
- Fingerprint TLS (JA3)
- Token di autenticazione
- ASN geografico

Vedi `src/fastapi_420/fingerprinting/composite.py:96-163` per come questi si combinano in un fingerprint univoco.

### Come funziona

Il `CompositeFingerprinter` ha tre livelli preimpostati:

**Relaxed**: solo IP + Auth
Ideale per: API interne in cui gli utenti si autenticano e gli IP sono stabili.

**Normal** (predefinito): IP + User-Agent + Auth
Ideale per: la maggior parte delle API pubbliche. Bilancia l'accuratezza con il tasso di falsi positivi.

**Strict**: tutti gli attributi, inclusi hash degli header, fingerprint TLS, geo
Ideale per: API ad alta sicurezza in cui puoi tollerare che alcuni utenti legittimi abbiano fingerprint diversi tra una sessione e l'altra.

Esempio di fingerprint da `src/fastapi_420/fingerprinting/composite.py:96-125`:
```
Normal level: "192.168.1.1:Mozilla/5.0...:user_abc123"
Strict level: "192.168.1.1:Mozilla/5.0...:en-US:gzip:8a3bc9:user_abc123:ja3_hash:AS15169"
```

### Errori comuni

**Errore 1: fidarsi ciecamente di X-Forwarded-For**
```python
# Bad
client_ip = request.headers.get("X-Forwarded-For").split(",")[0]

# Good - from src/fastapi_420/fingerprinting/ip.py:61-81
def _parse_x_forwarded_for(self, header, request):
    ips = [ip.strip() for ip in header.split(",")]
    
    # Walk backwards, stop at first untrusted IP
    for ip in reversed(ips):
        if ip not in self.trusted_proxies:
            return ip
    
    return ips[0]  # All IPs are trusted proxies
```

Chiunque può inviare un header `X-Forwarded-For`. Se non lo validi rispetto agli IP dei proxy fidati, gli attaccanti possono falsificare la propria identità.

**Errore 2: fare l'hash dell'intero User-Agent**
Lo User-Agent cambia con gli aggiornamenti del browser. `Mozilla/5.0 (Version 120.0)` diventa `Mozilla/5.0 (Version 121.0)` il mese successivo. Se fai l'hash della stringa completa, gli utenti legittimi appaiono come nuovi client dopo ogni aggiornamento del browser, aggirando i limiti.

Meglio: estrai solo il browser e la piattaforma, ignorando la versione: `Mozilla/5.0 (Windows)` invece della stringa completa.

## Circuit Breaker

### Che cos'è

Un circuit breaker monitora il carico del sistema e passa automaticamente da uno stato all'altro:

**Closed** (normale): tutte le richieste sono permesse, il circuito monitora il tasso di errore
**Open** (emergenza): rifiuta subito la maggior parte delle richieste, saltando i controlli del rate limit
**Half-open** (recupero): permette un traffico limitato per verificare se il sistema si è ripreso

Implementazione in `src/fastapi_420/defense/circuit_breaker.py:35-57`:
```python
async def check(self, storage):
    now = time.time()
    
    if self._state.is_open:
        if now - self._state.last_failure_time >= self.recovery_time:
            await self._enter_half_open()
            return True  # Allow request to test recovery
        return False  # Circuit still open
    
    request_count = await self._get_request_count(storage)
    
    if request_count >= self.threshold:
        await self._trip(now)  # Open the circuit
        return False
```

### Perché è importante

Durante un attacco DDoS, il tuo stesso rate limiter diventa un collo di bottiglia. Controllare i limiti richiede query al database (lookup su Redis). Se ricevi 1 milione di richieste al secondo, sono 1 milione di query Redis al secondo. Redis crollerà.

Il circuit breaker dice: "Siamo sotto attacco. Smettiamo di controllare i limiti individuali. Rifiutiamo tutto tranne gli utenti autenticati". Questo riduce il carico su Redis da 1 milione di query al secondo a quasi zero, permettendo al sistema di sopravvivere.

L'interruzione di AWS di febbraio 2020 in US-East-1 è stata causata in parte da guasti a cascata, quando un servizio non riusciva a gestire il carico e questo induceva altri servizi a ritentare ripetutamente, peggiorando il problema. I circuit breaker prevengono le tempeste di retry.

### Modalità di difesa

Configurato in `src/fastapi_420/config.py:85-94`, il circuit breaker supporta diverse strategie:

**Adaptive**: permetti gli utenti autenticati, blocca gli anonimi
Usala quando: la maggior parte degli attacchi proviene da bot non autenticati

**Lockdown**: blocca quasi tutto, permetti solo i client noti come affidabili
Usala quando: sei sotto un attacco grave ed è accettabile bloccare temporaneamente parte del traffico legittimo

**Challenge**: restituisci challenge CAPTCHA invece di blocchi netti
Non implementata nel progetto base ma mostrata nella sezione delle sfide

**Disabled**: nessun circuit breaker
Usala quando: hai una protezione DDoS dedicata davanti (Cloudflare, Akamai)

## HTTP 420 "Enhance Your Calm"

Questo progetto usa HTTP 420 invece dello standard 429 (Too Many Requests). Perché?

HTTP 420 era originariamente usato dall'API di Twitter nel 2010-2015 con il testo "Enhance Your Calm" quando i client raggiungevano i rate limit. Non è uno status code ufficiale (i codici registrati presso IANA si fermano a 418), ma è diventato uno semi-standard per il rate limiting nella comunità delle API.

Vantaggi:
- Immediatamente riconoscibile dagli sviluppatori che conoscono l'API di Twitter
- Il messaggio "Enhance Your Calm" è più amichevole di "Too Many Requests"
- Facile da filtrare nei log: `grep "420" access.log`

L'implementazione in `src/fastapi_420/exceptions.py:40-68` include gli header corretti:
```python
HTTP/1.1 420 Enhance Your Calm
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 45
Retry-After: 45
```

Questi header seguono la bozza di standard IETF per gli header di rate limit, rendendoli compatibili con le librerie client che comprendono le risposte 429.

## Standard e framework del settore

### OWASP Top 10

Questo progetto affronta:
- **A01:2021 - Broken Access Control** - Il rate limiting previene gli attacchi brute force all'autenticazione. Senza limiti su `/login`, gli attaccanti possono provare milioni di combinazioni di password.
- **A04:2021 - Insecure Design** - Implementare un rate limiting adeguato fin dall'inizio è design sicuro. Aggiungerlo dopo è più difficile e spesso incompleto.
- **A05:2021 - Security Misconfiguration** - La validazione della configurazione in `src/fastapi_420/config.py:165-179` assicura che i rate limit siano effettivamente abilitati e configurati correttamente prima dell'avvio.

### MITRE ATT&CK

Tecniche rilevanti che questo progetto difende o previene:
- **T1110.001** - Password Guessing: il rate limiting degli endpoint di login ferma gli attacchi brute force
- **T1110.003** - Password Spraying: i limiti per utente impediscono di provare password comuni su molti account
- **T1498** - Network Denial of Service: il circuit breaker e i limiti globali mitigano i DDoS a livello applicativo
- **T1496** - Resource Hijacking: previene l'abuso di endpoint ad alto consumo di calcolo (inferenza ML, rendering)

### CWE

Common weakness enumeration coperte:
- **CWE-770** - Allocation of Resources Without Limits: lo scopo stesso di questo progetto
- **CWE-307** - Improper Restriction of Excessive Authentication Attempts: risolta con il rate limiting degli endpoint di autenticazione
- **CWE-799** - Improper Control of Interaction Frequency: categoria ampia, il rate limiting è la mitigazione

## Esempi dal mondo reale

### Caso di studio 1: l'approccio di GitHub

L'API di GitHub usa un sofisticato sistema di rate limiting descritto nella loro documentazione. Per gli utenti autenticati: 5000 richieste/ora. Per le app OAuth: varia in base allo scope. Per le richieste non autenticate: 60/ora per IP.

Lezione chiave: limiti diversi per diversi livelli di autenticazione. L'implementazione in `examples/app.py:42-70` mostra questo pattern:
```python
auth_limiter = ScopedRateLimiter(
    endpoint_rules={
        "POST:/auth/login": ["3/minute", "10/hour"],      # Strict
        "POST:/auth/register": ["2/minute", "5/hour"],    # Very strict
    }
)

user_limiter = ScopedRateLimiter(
    default_rules=["60/minute", "500/hour"],              # Relaxed for authenticated
)
```

### Caso di studio 2: l'attacco DDoS a Dyn del 2016

Il 21 ottobre 2016, un massiccio attacco DDoS ha mandato giù siti importanti tra cui Twitter, Netflix e Reddit. L'attacco ha usato la botnet Mirai (milioni di dispositivi IoT compromessi) per inondare i server DNS di Dyn.

Il tradizionale rate limiting per IP ha fallito perché:
1. Il traffico proveniva da milioni di IP univoci (attacco distribuito)
2. Ogni singolo IP restava sotto i limiti
3. L'aggregato superava la capacità di 100 volte

Difesa necessaria: limiti globali per endpoint + circuit breaker. Quando il traffico totale verso `/dns-query` superava la soglia, il circuit breaker avrebbe dovuto rifiutare immediatamente la maggior parte delle richieste invece di provare a elaborarle tutte.

La difesa a tre livelli di questo progetto (`src/fastapi_420/defense/layers.py:47-84`) affronta esattamente questo scenario. Il livello 1 (per utente) non aiuterà. Il livello 2 (globale per endpoint) inizia a bloccare quando il traffico dell'endpoint supera `ENDPOINT_LIMIT_MULTIPLIER` (predefinito 10 volte il limite per utente). Il livello 3 (circuit breaker) fornisce la protezione di ultima istanza.

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere:

1. Hai un limite fixed window di 100/minuto. Alle 12:00:30, un client ha fatto 50 richieste. A che ora si azzera il suo contatore? Può fare altre 100 richieste alle 12:00:59?

2. La tua API serve sia utenti mobili che desktop. Gli utenti mobili condividono gli IP a causa del NAT dell'operatore. Come implementi un rate limiting che non penalizzi ingiustamente gli utenti mobili? Di quali attributi fai fingerprinting?

3. Un attaccante controlla un blocco IPv6 /64 (18 quintilioni di indirizzi). Stai applicando il rate limit per IP. Quante richieste può fare prima di raggiungere il tuo limite di 100/minuto? Come lo risolvi?

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. L'implementazione avrà più senso una volta che questi fondamenti ti saranno chiari.

## Letture di approfondimento

**Essenziali:**
- [IETF Rate Limit Headers Draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) - Lo standard per gli header RateLimit-* che questo progetto implementa
- [Redis Lua Scripting](https://redis.io/docs/interact/programmability/eval-intro/) - Come funzionano le operazioni atomiche

**Approfondimenti:**
- [Generic Cell Rate Algorithm](https://en.wikipedia.org/wiki/Generic_cell_rate_algorithm) - L'algoritmo delle telecomunicazioni che ha ispirato il token bucket
- [Stripe's Rate Limiting](https://stripe.com/blog/rate-limiters) - Come una grande azienda di API ragiona sul rate limiting

**Contesto storico:**
- [Twitter's API History](https://developer.twitter.com/en/docs/rate-limits) - Origine dello status code HTTP 420
