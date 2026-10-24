# Sfide di estensione

Hai costruito il progetto base. Ora rendilo tuo estendendolo con nuove funzionalità.

Queste sfide sono ordinate per difficoltà. Inizia da quelle più facili per acquisire sicurezza, poi affronta quelle più difficili quando vuoi approfondire.

## Sfide facili

### Sfida 1: aggiungere header di rate limit personalizzati

**Cosa costruire:**
Aggiungi header personalizzati alle risposte di rate limit oltre agli header standard RateLimit-*. Includi header come X-RateLimit-Policy (quale regola è stata raggiunta) e X-RateLimit-Scope (user/endpoint/global).

**Perché è utile:**
Debug e logica lato client. I client possono vedere esattamente quale limite hanno raggiunto e adattare il loro comportamento di conseguenza. Se raggiungono il limite per endpoint invece di quello per utente, sanno se rallentare su tutti gli endpoint o solo su quello corrente.

**Cosa imparerai:**
- Modificare la struttura dati RateLimitResult
- Aggiungere informazioni alle risposte HTTP tramite middleware
- Fare debug del comportamento del rate limiting

**Suggerimenti:**
- Guarda `src/fastapi_420/types.py:66-92` dove è definito RateLimitResult. Aggiungi nuovi campi per il nome della policy e lo scope.
- Modifica la proprietà `headers` alle righe 84-92 per includere i tuoi nuovi header.
- Testalo facendo richieste e controllando gli header della risposta: `curl -i localhost:8000/api/test`

**Verifica che funzioni:**
```bash
curl -i http://localhost:8000/api/test
# Should see:
# RateLimit-Limit: 100
# RateLimit-Remaining: 99
# X-RateLimit-Policy: default
# X-RateLimit-Scope: user
```

### Sfida 2: aggiungere limiti specifici per endpoint tramite decorator

**Cosa costruire:**
Crea un decorator che applichi limiti diversi a endpoint diversi senza usare la configurazione del middleware. Permetti agli sviluppatori di scrivere `@rate_limit("10/minute", "100/hour")` direttamente sulle funzioni delle route.

**Perché è utile:**
Più esplicito della configurazione del middleware. Gli sviluppatori vedono i limiti direttamente nel codice accanto all'endpoint. Più facile da mantenere perché i limiti stanno insieme al codice che proteggono.

**Cosa imparerai:**
- Decorator Python con parametri
- Preservare i metadati delle funzioni con functools.wraps
- Integrarsi con il sistema di dependency injection di FastAPI

**Suggerimenti:**
- Studia il decorator esistente in `src/fastapi_420/limiter.py:143-177`
- La sfida è farlo funzionare senza passare esplicitamente la request. Usa Depends() di FastAPI per iniettare la request.
- Non dimenticare di chiamare `functools.wraps(func)` per preservare il nome e la docstring della funzione originale.

**Verifica che funzioni:**
```python
@app.get("/custom")
@rate_limit("5/minute")
async def custom_endpoint():
    return {"limited": True}

# Make 5 requests, should work
# 6th request should return HTTP 420
```

### Sfida 3: aggiungere un prefisso alle chiavi Redis per ambiente

**Cosa costruire:**
Aggiungi automaticamente alle chiavi Redis un prefisso con il nome dell'ambiente (dev/staging/prod), così che ambienti diversi possano condividere un'istanza Redis senza collisioni tra chiavi.

**Perché è utile:**
Risparmio sui costi. Invece di eseguire 3 istanze Redis separate, ne esegui una con chiavi in namespace distinti. Le chiavi di sviluppo non interferiscono con quelle di produzione.

**Cosa imparerai:**
- Pattern di design delle chiavi Redis
- Configurazione basata sull'ambiente
- Strategie di namespacing delle chiavi

**Suggerimenti:**
- Modifica `RateLimitKey.build()` in `src/fastapi_420/types.py:354-362` per includere il prefisso dell'ambiente
- Ottieni l'ambiente dalle impostazioni: `self._settings.ENVIRONMENT`
- Il risultato dovrebbe essere: `dev:ratelimit:v1:user:...` invece di `ratelimit:v1:user:...`

**Verifica che funzioni:**
```bash
# Set environment
export RATELIMIT_ENVIRONMENT=staging

# Start app, make requests
# Check Redis keys
redis-cli keys "*"
# Should see: staging:ratelimit:v1:user:...
```

## Sfide intermedie

### Sfida 4: implementare la ponderazione del costo delle richieste

**Cosa costruire:**
Permetti a richieste diverse di consumare quantità diverse di "budget" del rate limit. Una semplice GET potrebbe costare 1 punto, mentre la generazione di un report costoso ne costa 10. I client con 100 punti/minuto possono fare 100 richieste semplici o 10 costose.

**Applicazione nel mondo reale:**
L'API di OpenAI usa questo pattern. Modelli diversi costano quantità diverse di "token". GPT-4 è più costoso di GPT-3.5. Lo stesso concetto si applica alle API in cui alcuni endpoint sono computazionalmente costosi.

**Cosa imparerai:**
- Estendere il rate limiting oltre il semplice conteggio delle richieste
- Progettare API flessibili che tengano conto del consumo di risorse
- Bilanciare semplicità e potenza

**Approccio implementativo:**

1. **Aggiungi un parametro cost al metodo check**
   - File da creare: nessuno (modifica quelli esistenti)
   - File da modificare: `src/fastapi_420/limiter.py:181`, `src/fastapi_420/algorithms/base.py:26`

2. **Moltiplica il contatore per il costo**
   - Agganciati alle operazioni di incremento dello storage
   - Passa il costo agli script Lua
   - Aggiorna l'algoritmo sliding window per gestire conteggi pesati

3. **Testa i casi limite:**
   - E se cost=0? (Richieste gratuite)
   - E se cost=1000 ma il limite è 100? (Una singola richiesta esaurisce il limite)
   - E se il costo è negativo? (Dovrebbe essere rifiutato come non valido)

**Suggerimenti:**
- Inizia aggiungendo un parametro opzionale `cost: int = 1` a `limiter.check()`
- Nello script Lua, cambia `redis.call('INCR', current_key)` in `redis.call('INCRBY', current_key, cost)`
- Per i test, crea endpoint con costi diversi: `/cheap` (cost=1), `/expensive` (cost=10)

**Punti extra:**
Rendi il costo configurabile per endpoint tramite decorator: `@limiter.limit("100/minute", cost=5)`

### Sfida 5: aggiungere una challenge CAPTCHA per i client sospetti

**Cosa costruire:**
Quando un client raggiunge ripetutamente i rate limit, invece di bloccarlo completamente, restituisci una challenge CAPTCHA. Se la risolve, lascia passare la richiesta. Tieni traccia del tasso di successo dei CAPTCHA per client.

**Applicazione nel mondo reale:**
La modalità "I'm Under Attack" di Cloudflare funziona così. Il traffico sospetto riceve challenge CAPTCHA invece di blocchi netti. Riduce i falsi positivi (bloccare utenti legittimi) continuando a fermare i bot.

**Cosa imparerai:**
- Integrare servizi di terze parti (hCaptcha, reCAPTCHA)
- Strategie di enforcement progressivo
- Bilanciare la sicurezza con la user experience

**Approccio implementativo:**

1. **Tieni traccia del numero di violazioni per client**
   - Aggiungi un contatore: "violations:{identifier}" in Redis
   - Incrementalo quando il rate limit viene superato
   - Azzeralo dopo un CAPTCHA risolto con successo

2. **Restituisci una challenge CAPTCHA invece di 420**
   - Quando le violazioni > soglia (es. 3), restituisci HTTP 429 con una challenge CAPTCHA
   - Il corpo della risposta include la site key del CAPTCHA
   - Il client risolve il CAPTCHA e invia il token della soluzione

3. **Valida la soluzione del CAPTCHA**
   - Nuovo endpoint: POST /verify-captcha
   - Valida il token con l'API del CAPTCHA
   - Se valido, concedi un bypass temporaneo (memorizza il token in Redis con scadenza)

4. **Aggira il rate limit con un CAPTCHA valido**
   - Verifica la presenza del token di bypass CAPTCHA durante l'estrazione del fingerprint
   - Se presente e valido, salta il controllo del rate limit

**Suggerimenti:**
- Usa hCaptcha (API più semplice di reCAPTCHA)
- Memorizza i token di bypass con scadenza di 5 minuti
- Tieni traccia del tasso di risoluzione dei CAPTCHA: `captcha_solved / captcha_presented`

**Test:**
Non è facile testare i CAPTCHA nei test automatici. Invece:
1. Crea un verificatore CAPTCHA mock che restituisce sempre true
2. Inietta il mock nell'ambiente di test
3. In produzione, usa la vera API di hCaptcha

### Sfida 6: implementare rate limit basati sulla geolocalizzazione

**Cosa costruire:**
Applica rate limit diversi in base alla posizione geografica del client. Limiti più stretti per le regioni con alta attività di bot (es. limita i client di certi paesi a 10/minuto contro 100/minuto per il traffico nazionale).

**Applicazione nel mondo reale:**
I siti di e-commerce vedono spesso traffico di bot proveniente principalmente da certe regioni. Applicare limiti regionali riduce le frodi senza impattare i clienti internazionali legittimi.

**Cosa imparerai:**
- Database di geolocalizzazione IP (MaxMind GeoIP2)
- Rate limiting basato su policy
- Considerazioni sulla discriminazione geografica (fai attenzione con questo)

**Approccio implementativo:**

1. **Aggiungi il lookup di geolocalizzazione**
   - Libreria: `geoip2` con il database MaxMind
   - Durante l'estrazione del fingerprint, cerca l'indirizzo IP
   - Aggiungi il codice paese a FingerprintData

2. **Definisci policy basate sulla posizione geografica**
   - Configurazione: `GEO_LIMITS = {"US": "100/minute", "CN": "10/minute", "default": "50/minute"}`
   - Caricale dall'ambiente o da un file di configurazione

3. **Applica la policy al controllo del rate limit**
   - In LayeredDefense, controlla il codice paese del client
   - Seleziona il limite appropriato da GEO_LIMITS
   - Ripiega sul valore predefinito se il paese non è configurato

4. **Monitora e adatta**
   - Registra nei log le violazioni del rate limit per paese
   - Dashboard che mostra richieste/violazioni per regione

**Suggerimenti:**
- Il database MaxMind GeoLite2 è gratuito ma richiede la registrazione
- Metti in cache i lookup geografici (la mappatura da IP a paese non cambia di frequente)
- Considera le implicazioni sulla privacy. Registrare indirizzi IP con codici paese potrebbe costituire PII in alcune giurisdizioni.

**Considerazioni etiche:**
Il rate limiting basato sulla posizione geografica può essere discriminatorio. Usalo solo per scopi di sicurezza legittimi. Documenta perché certe regioni hanno limiti più stretti. Valuta di offrire una challenge CAPTCHA invece di un blocco netto.

## Sfide avanzate

### Sfida 7: costruire una dashboard dei rate limit

**Cosa costruire:**
Una dashboard web che mostra statistiche di rate limiting in tempo reale. Le metriche includono richieste/sec, principali trasgressori, stato del circuit breaker, prestazioni degli algoritmi. Costruita con FastAPI + htmx o React.

**Perché è difficile:**
Richiede di raccogliere metriche senza impattare le prestazioni del rate limiting. Serve un'aggregazione efficiente di eventi ad alta frequenza. Deve gestire le query della dashboard senza rallentare i controlli dei rate limit.

**Cosa imparerai:**
- Raccolta di metriche ad alte prestazioni
- Aggregazione di dati time-series
- Costruire dashboard di amministrazione
- WebSocket o Server-Sent Events per aggiornamenti in tempo reale

**Modifiche architetturali necessarie:**
```
┌─────────────────┐
│  Rate Limiter   │
│  (existing)     │
└────────┬────────┘
         │
         ├─ Check limit (fast path)
         │
         └─ Emit metrics (async, non-blocking)
                  ↓
         ┌────────────────┐
         │ Metrics Buffer │
         │ (in-memory)    │
         └────────┬───────┘
                  │
         Batch write every 5 seconds
                  ↓
         ┌────────────────┐
         │ Redis TimeSeries│
         │ or InfluxDB     │
         └────────┬───────┘
                  │
                  ↓
         ┌────────────────┐
         │   Dashboard    │
         │   (FastAPI)    │
         └────────────────┘
```

**Passi di implementazione:**

**Fase 1: raccolta delle metriche** (3-5 ore)
- Aggiungi un emettitore di metriche al rate limiter
- Raggruppa le metriche in un buffer in memoria
- Scrivile su Redis ogni 5 secondi
- Metriche: requests_total, violations_total, latency_histogram

**Fase 2: aggregazione** (3-5 ore)
- Crea un task in background per aggregare le metriche a 5 secondi in bucket da 1 minuto/1 ora/1 giorno
- Memorizzale in sorted set di Redis: `ZADD metrics:requests:minute:{timestamp} {count} {endpoint}`
- Retention: conserva i dati a 5 secondi per 10 minuti, i dati a 1 minuto per 24 ore, i dati a 1 ora per 30 giorni

**Fase 3: backend della dashboard** (4-6 ore)
- Endpoint FastAPI:
  - `GET /api/metrics/summary` - Stato corrente (statistiche dell'ultimo minuto)
  - `GET /api/metrics/timeseries?metric=requests&window=1h` - Dati storici
  - `GET /api/metrics/top-violators?limit=10` - Client con più violazioni
- Endpoint WebSocket: `/ws/metrics` - Aggiornamenti in tempo reale

**Fase 4: frontend della dashboard** (6-8 ore)
- Chart.js o Recharts per i grafici time-series
- Aggiornamento automatico ogni 5 secondi
- Card che mostrano: richieste totali, violazioni dei rate limit, stato del circuit breaker, endpoint principali
- Tabella delle violazioni recenti con fingerprint del client (troncato), endpoint, timestamp

**Insidie:**
- **Non bloccare i controlli dei rate limit**: la raccolta delle metriche deve essere asincrona. Usa `asyncio.create_task()` in modalità fire-and-forget.
- **Overflow del buffer**: se il buffer delle metriche cresce troppo (milioni di eventi), scarta i più vecchi o campiona.
- **Memoria di Redis**: i dati time-series crescono in fretta. Imposta policy di scadenza.

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Raccogliere metriche senza aggiungere più di 5ms di latenza ai controlli dei rate limit
- [ ] Mostrare le richieste/sec aggiornate ogni 5 secondi
- [ ] Mostrare i 10 principali trasgressori dei rate limit con la suddivisione per endpoint
- [ ] Visualizzare le transizioni di stato del circuit breaker
- [ ] Gestire 10k richieste/sec senza sovraccaricare il backend della dashboard

### Sfida 8: implementare un circuit breaker distribuito

**Cosa costruire:**
Un circuit breaker che condivide lo stato tra più server API usando Redis. Quando un server fa scattare il circuito, tutti i server entrano immediatamente nello stesso stato. L'implementazione attuale è solo per processo.

**Tempo stimato:**
2-3 giorni per l'implementazione completa con i test

**Prerequisiti:**
Dovresti aver completato prima la Sfida 4 (ponderazione del costo delle richieste) e la Sfida 7 (dashboard), perché capire le metriche è fondamentale per il coordinamento distribuito.

**Cosa imparerai:**
- Coordinamento nei sistemi distribuiti
- Pub/sub di Redis per la propagazione degli eventi
- Gestione di uno stato eventualmente consistente
- Gestire le partizioni di rete e le race condition

**Pianificare questa funzionalità:**

Prima di scrivere codice, rifletti su:
- **Consenso**: come si mettono d'accordo i server sullo stato del circuito? (Redis come unica fonte di verità)
- **Ritardo di propagazione**: il server A fa scattare il circuito, quanto tempo passa prima che il server B lo sappia? (Il pub/sub dà ~100ms)
- **Modalità di guasto**: e se la connessione a Redis fallisce? (Ripiega sul circuit breaker locale)
- **Race condition**: più server che provano a far scattare il circuito simultaneamente (Usa SET NX di Redis)

**Architettura di alto livello:**
```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Server 1    │       │  Server 2    │       │  Server 3    │
│  Circuit     │       │  Circuit     │       │  Circuit     │
│  Breaker     │       │  Breaker     │       │  Breaker     │
└──────┬───────┘       └──────┬───────┘       └──────┬───────┘
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              │
                   ┌──────────▼───────────┐
                   │      Redis           │
                   │  Circuit State:      │
                   │  - is_open           │
                   │  - trip_time         │
                   │  - request_count     │
                   │                      │
                   │  Pub/Sub Channel:    │
                   │  circuit:events      │
                   └──────────────────────┘
```

**Fasi di implementazione:**

**Fase 1: stato condiviso** (6-8 ore)
- Memorizza lo stato del circuito in un hash Redis: `circuit:state`
- Campi: is_open (0/1), last_trip_time, failure_count, request_count
- Ogni server legge lo stato prima di controllare le soglie
- Usa il TTL di Redis per azzerare automaticamente dopo recovery_time

**Fase 2: scatto atomico** (4-6 ore)
- Usa uno script Lua per l'operazione atomica di scatto:
```lua
  local current_count = redis.call('GET', 'circuit:requests')
  if current_count >= threshold then
      redis.call('HSET', 'circuit:state', 'is_open', '1')
      redis.call('HSET', 'circuit:state', 'trip_time', now)
      redis.call('PUBLISH', 'circuit:events', 'TRIPPED')
      return 1
  end
  return 0
```
- Solo un server riesce a far scattare il circuito, gli altri vedono subito is_open=1

**Fase 3: propagazione degli eventi** (5-7 ore)
- Iscriviti al canale pub/sub di Redis: `circuit:events`
- Eventi: TRIPPED, HALF_OPEN, CLOSED
- Un task in background ascolta gli eventi e aggiorna la cache locale
- La cache locale riduce le letture su Redis (controlla prima in locale, poi Redis se obsoleta)

**Fase 4: degrado graceful** (4-6 ore)
- Se la connessione a Redis fallisce, ripiega sul circuit breaker locale
- Registra un warning: "Distributed circuit breaker unavailable, using local"
- Quando Redis si riconnette, sincronizza lo stato locale con quello distribuito
- Gestisci il caso in cui il circuito locale è aperto ma quello distribuito è chiuso (fidati di quello distribuito)

**Sfide note:**

1. **Clock skew**
   - Problema: i server hanno orologi leggermente diversi. Il server A pensa che il circuito debba chiudersi alle 12:00:30, il server B alle 12:00:31.
   - Suggerimento: usa il comando TIME di Redis per ottenere l'ora del server, non quella locale. Oppure usa una scadenza basata su TTL invece del confronto tra timestamp.

2. **Partizione di rete**
   - Problema: il server C non riesce a raggiungere Redis e ripiega sul circuito locale. Il circuito locale scatta a una soglia più bassa e blocca il traffico legittimo.
   - Suggerimento: aggiungi una metrica di "confidenza". Se la connessione a Redis è instabile, aumenta le soglie o disabilita del tutto il circuito (fail-open).

3. **Thundering herd**
   - Problema: il circuito si apre e 100 server provano tutti a entrare nello stato half-open simultaneamente dopo recovery_time.
   - Suggerimento: randomizza il tempo di recupero: `recovery_time ± random(0, 5)` così che i server scaglionino i loro tentativi.

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Far scattare il circuito su tutti i server entro 500ms dal superamento della soglia
- [ ] Sincronizzare lo stato anche se un server si riavvia (legge da Redis all'avvio)
- [ ] Gestire in modo graceful il fallimento della connessione a Redis (ripiegare sul locale)
- [ ] Supportare modifiche manuali dello stato del circuito tramite Redis CLI: `HSET circuit:state is_open 1`
- [ ] Superare il load test: 10 server, 100k richieste/sec, il circuito scatta correttamente su tutti

### Sfida 9: costruire un algoritmo personalizzato - Generic Cell Rate Algorithm (GCRA)

**Cosa costruire:**
Implementa l'algoritmo GCRA usato dalle reti ATM e da alcuni rate limiter ad alte prestazioni. Più preciso del token bucket, permette i burst entro una "cell delay variation tolerance" definita.

**Tempo stimato:**
3-4 giorni inclusi ricerca, implementazione e test approfonditi

**Prerequisiti:**
Una solida comprensione degli algoritmi token bucket e sliding window. Leggi la specifica GCRA prima di iniziare.

**Cosa imparerai:**
- Algoritmi delle reti di telecomunicazione
- Rate limiting con precisione inferiore al secondo
- Modellazione del tempo continuo contro discreto
- Dimostrazioni di correttezza degli algoritmi

**Come funziona GCRA:**

GCRA tiene traccia del "Theoretical Arrival Time" (TAT). Ogni richiesta dovrebbe arrivare non prima del TAT. Se la richiesta arriva al tempo T:
```
if T >= TAT:
    TAT = max(TAT, T) + 1/rate
    ALLOW
else:
    DENY
```

Con la tolleranza ai burst, permetti le richieste che arrivano fino a `limit` unità di tempo in anticipo:
```
if T >= TAT - limit:
    TAT = max(TAT, T) + 1/rate
    ALLOW
else:
    DENY
```

**Implementazione:**

1. **Crea il file dell'algoritmo**: `src/fastapi_420/algorithms/gcra.py`
2. **Definisci lo stato**: memorizza il TAT in Redis/memoria
3. **Implementa il metodo check**:
```python
   async def check(self, storage, key, rule, timestamp=None):
       now = timestamp or time.time()
       rate = rule.requests / rule.window_seconds  # requests per second
       limit = rule.requests  # burst tolerance
       
       tat = await storage.get_gcra_state(key)
       if tat is None:
           tat = now
       
       if now >= tat - limit / rate:
           # Allow
           new_tat = max(tat, now) + 1/rate
           await storage.set_gcra_state(key, new_tat, ttl=rule.window_seconds)
           remaining = calculate_remaining(new_tat, now, rate, limit)
           return RateLimitResult(allowed=True, remaining=remaining, ...)
       else:
           # Deny
           return RateLimitResult(allowed=False, retry_after=tat - now, ...)
```

4. **Aggiungi i metodi di storage**: `get_gcra_state()` e `set_gcra_state()` ai backend di storage
5. **Scrivi test completi**: casi limite come aggiustamenti dell'orologio, ritmi molto alti, ritmi frazionari

**Strategia di test:**
```python
# Test burst tolerance
rule = RateLimitRule(requests=10, window_seconds=10)  # 1 req/sec, burst of 10
algo = GCRAAlgorithm()

# Should allow 10 immediate requests
for i in range(10):
    assert algo.check(storage, "test", rule, timestamp=0).allowed

# 11th should deny
assert not algo.check(storage, "test", rule, timestamp=0).allowed

# After 1 second, should allow 1 more
assert algo.check(storage, "test", rule, timestamp=1.0).allowed
```

**Confronto con il token bucket:**
Entrambi gli algoritmi permettono i burst, ma GCRA è più preciso. Il token bucket si aggiorna con incrementi discreti (aggiungi X token ogni Y secondi). GCRA usa il tempo continuo (il TAT può essere qualsiasi valore float).

## Combina e abbina

Combina le funzionalità per progetti più grandi:

**Idea di progetto 1: dashboard completa del rate limiting con limiti geografici**
- Combina la Sfida 7 (dashboard) + la Sfida 6 (geolocalizzazione)
- Aggiungi una visualizzazione su mappa che mostra le richieste per paese
- Pannello di amministrazione per regolare i limiti geografici in tempo reale
- Risultato: rate limiting pronto per la produzione con policy geografiche

**Idea di progetto 2: limiti pesati sul costo con bypass tramite CAPTCHA**
- Combina la Sfida 4 (ponderazione del costo) + la Sfida 5 (CAPTCHA)
- Le operazioni costose costano più punti
- Quando i punti sono esauriti, offri un CAPTCHA per ottenere un aumento temporaneo
- Risultato: un sistema flessibile che permette i burst con verifica

## Sfide di integrazione nel mondo reale

### Integrazione con Prometheus per il monitoraggio

**L'obiettivo:**
Esportare le metriche del rate limiting verso Prometheus per visualizzarle in Grafana. Tenere traccia di richieste, violazioni, latenza degli algoritmi, salute dello storage.

**Cosa ti servirà:**
- La libreria prometheus-client
- Il JSON di una dashboard Grafana
- Comprensione dei tipi di metrica (counter, gauge, histogram)

**Piano di implementazione:**
1. Aggiungi prometheus_client alle dipendenze
2. Crea le metriche in `src/fastapi_420/metrics.py`:
```python
   from prometheus_client import Counter, Histogram
   
   requests_total = Counter(
       'ratelimit_requests_total',
       'Total requests checked',
       ['algorithm', 'allowed']
   )
   
   check_latency = Histogram(
       'ratelimit_check_duration_seconds',
       'Time to check rate limit'
   )
```
3. Instrumenta il codice: `requests_total.labels(algorithm='sliding_window', allowed='true').inc()`
4. Esponi un endpoint delle metriche: `@app.get("/metrics")` restituisce `prometheus_client.generate_latest()`
5. Configura lo scrape config di Prometheus, puntandolo a `/metrics`

**Attenzione a:**
- Label ad alta cardinalità (non includere ID utente nelle label, usa <10 valori univoci per label)
- Convenzioni sui nomi delle metriche (usa underscore, aggiungi l'unità come suffisso)
- Impatto sulle prestazioni (prometheus_client è veloce ma non gratuito, aggiunge ~50μs per aggiornamento di metrica)

### Deploy su Kubernetes

**L'obiettivo:**
Eseguire l'API con rate limiting in Kubernetes con Redis, monitoraggio e auto-scaling.

**Cosa imparerai:**
- Deployment e Service di Kubernetes
- ConfigMap e Secret per la configurazione
- Horizontal Pod Autoscaler
- Deployment di Redis con persistenza

**Passi:**

1. **Crea un Dockerfile** per l'API:
```dockerfile
   FROM python:3.12-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY src/ ./src/
   CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0"]
```

2. **Manifest di Kubernetes**:
```yaml
   # redis-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: redis
   spec:
     replicas: 1
     template:
       spec:
         containers:
         - name: redis
           image: redis:7-alpine
           ports:
           - containerPort: 6379
   
   # api-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api
   spec:
     replicas: 3
     template:
       spec:
         containers:
         - name: api
           image: your-api:latest
           env:
           - name: RATELIMIT_REDIS_URL
             value: "redis://redis:6379"
           ports:
           - containerPort: 8000
   
   # api-service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: api
   spec:
     selector:
       app: api
     ports:
     - port: 80
       targetPort: 8000
     type: LoadBalancer
```

3. **ConfigMap** per le impostazioni:
```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: ratelimit-config
   data:
     RATELIMIT_ALGORITHM: "sliding_window"
     RATELIMIT_DEFAULT_LIMIT: "100/minute"
     RATELIMIT_CIRCUIT_THRESHOLD: "10000"
```

4. **Horizontal Pod Autoscaler**:
```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: api-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: api
     minReplicas: 3
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 70
```

**Checklist per la produzione:**
- [ ] Redis ha un volume persistente per i dati
- [ ] Secret usati per la configurazione sensibile (non ConfigMap)
- [ ] Limiti delle risorse impostati (CPU/memoria)
- [ ] Probe di liveness e readiness configurate
- [ ] Il logging va su stdout (catturato da Kubernetes)
- [ ] Endpoint delle metriche esposto per lo scraping di Prometheus

## Sfide sulle prestazioni

### Sfida: gestire 100k richieste/secondo

**L'obiettivo:**
Ottimizzare il rate limiter per gestire 100k richieste/sec su una singola macchina senza crollare.

**Collo di bottiglia attuale:**
A 10k req/sec, la latenza di rete di Redis diventa il fattore limitante. Ogni richiesta = 1 chiamata Redis = 1ms di round trip = max 1000 req/sec per connessione.

**Approcci di ottimizzazione:**

**Approccio 1: connection pooling**
- Come: aumenta `REDIS_MAX_CONNECTIONS` da 100 a 1000
- Guadagno: parallelismo 100 volte maggiore, può gestire 100k req/sec
- Compromesso: overhead delle connessioni Redis, più memoria

**Approccio 2: caching locale con consistenza eventuale**
- Come: metti in cache lo stato del rate limit in memoria per 100ms. Sincronizza con Redis solo ogni 100ms.
- Guadagno: riduce le chiamate a Redis del 90%, miglioramento massimo di 10 volte
- Compromesso: l'accuratezza scende a ~95%. Due server potrebbero permettere 110/100 richieste in una finestra.

**Approccio 3: batching**
- Come: accumula le richieste per 10ms, inviale a Redis in un'unica pipeline
- Guadagno: riduce l'overhead di rete, miglioramento di ~5 volte
- Compromesso: 10ms di latenza aggiuntiva, complessità

**Fanne il benchmark:**
```bash
# Load test with wrk
wrk -t12 -c400 -d30s http://localhost:8000/api/test

# Should see:
# Requests/sec: 100000
# Latency avg: 4ms
# Latency 99th: 15ms
```

Metriche obiettivo:
- **Throughput**: >100k richieste/sec
- **Latenza p50**: <5ms
- **Latenza p99**: <20ms

### Sfida: ridurre l'uso di memoria del 90%

**L'obiettivo:**
Ridurre il consumo di memoria per 1 milione di chiavi di rate limit attive da ~100MB a ~10MB.

**Uso attuale:**
Ogni chiave memorizza:
- Voce del dict Python: 56 byte
- WindowEntry: 24 byte
- Chiave stringa (~50 caratteri): 50 byte
Totale: ~130 byte/chiave * 1M chiavi = 130MB

**Aree di ottimizzazione:**

**Area 1: usare Redis invece della memoria**
- Cosa è inefficiente: gli oggetti Python hanno un overhead
- Come risolvere: memorizza in Redis (più efficiente in termini di memoria)
- Risparmio: riduzione del 70%

**Area 2: comprimere le chiavi**
- Cosa è inefficiente: chiavi come "ratelimit:v1:user:POST:/api/endpoint:192.168.1.1:Mozilla...:60" sono lunghe 100+ caratteri
- Come risolvere: fai l'hash delle chiavi in stringhe esadecimali di 16 caratteri
- Risparmio: riduzione dell'80% della dimensione delle chiavi

**Area 3: usare gli slots**
- Cosa è inefficiente: l'attributo `__dict__` di Python sulle dataclass
- Come risolvere: aggiungi `__slots__` a WindowEntry, RateLimitResult
- Risparmio: riduzione del 30% della dimensione degli oggetti

## Sfide di sicurezza

### Sfida: aggiungere livelli di rate limit basati su JWT

**Cosa implementare:**
Rate limit diversi in base ai claim del JWT. Livello gratuito: 10/minuto. Livello a pagamento: 100/minuto. Enterprise: 1000/minuto.

**Threat model:**
Questo protegge da:
- Abuso del livello gratuito (creare molti account per ottenere più quota)
- Allocazione equa delle risorse (i clienti paganti ottengono ciò per cui hanno pagato)

**Implementazione:**

1. **Estrai il livello dal JWT**:
```python
   # In auth extractor
   payload = jwt.decode(token, secret, algorithms=['HS256'])
   tier = payload.get('tier', 'free')  # free, paid, enterprise
```

2. **Aggiungi il livello al fingerprint**:
```python
   # In FingerprintData
   tier: str | None = None
```

3. **Seleziona il limite in base al livello**:
```python
   # In RateLimiter.check
   tier_limits = {
       'free': ['10/minute'],
       'paid': ['100/minute'],
       'enterprise': ['1000/minute'],
   }
   rules = [RateLimitRule.parse(l) for l in tier_limits.get(tier, tier_limits['free'])]
```

**Testare la sicurezza:**
- Prova a falsificare un JWT con tier=enterprise
- Dovrebbe fallire la validazione della firma
- Prova a creare un account gratuito e ad abusare dell'endpoint
- Dovresti raggiungere il limite di 10/minuto
- Verifica che gli utenti paganti possano superare i limiti del livello gratuito

### Sfida: superare l'OWASP API Security Top 10

**L'obiettivo:**
Verificare il rate limiter rispetto all'OWASP API Security Top 10 e correggere eventuali lacune.

**Lacune attuali:**
- **API4:2023 - Unrestricted Resource Consumption**: parzialmente coperta dal rate limiting. Lacuna: nessun limite per IP sugli endpoint non autenticati.
- **API5:2023 - Broken Function Level Authorization**: lacuna: il bypass del circuit breaker non verifica se l'utente "autenticato" abbia effettivamente il permesso.
- **API8:2023 - Security Misconfiguration**: lacuna: nessuna validazione che in produzione Redis sia configurato.

**Rimedio:**

1. **Aggiungi limiti per IP per i non autenticati**:
```python
   if not fingerprint.auth_identifier:
       # Stricter limits for anonymous
       rules = [RateLimitRule.parse('10/minute')]
```

2. **Aggiungi un controllo dei permessi al bypass del circuito**:
```python
   def _should_bypass_circuit(self, context):
       if not context.is_authenticated:
           return False
       # Check actual permissions, not just authentication
       return context.has_permission('circuit.bypass')
```

3. **Aggiungi la validazione per la produzione**:
```python
   @model_validator(mode='after')
   def validate_production(self):
       if self.ENVIRONMENT == 'production':
           if not self.storage.REDIS_URL:
               raise ValueError("Production requires Redis")
       return self
```

## Idee per contribuire

Hai completato una sfida? Condividila:

1. **Fai il fork del repo**
2. **Implementa la tua estensione** in un feature branch: `git checkout -b feature/captcha-challenge`
3. **Documentala** - Aggiungi alla cartella learn/ una spiegazione di come funziona
4. **Invia una PR** con:
   - La tua implementazione
   - Test che dimostrano che funziona
   - Documentazione nella cartella learn/
   - Esempio d'uso in examples/

Le buone estensioni potrebbero essere integrate nel progetto principale. Anche se non vengono integrate, avrai un pezzo per il tuo portfolio che dimostra security engineering nel mondo reale.

## Mettiti ancora alla prova

### Costruisci qualcosa di nuovo

Usa i concetti che hai imparato qui per costruire:
- **API Gateway con rate limiting** - Reverse proxy che aggiunge il rate limiting a qualsiasi API di backend
- **Rate limiter per le query al database** - Limita le query al secondo per prevenire il sovraccarico del database
- **Throttler per code di task distribuite** - Applica il rate limit all'esecuzione dei job in background

### Studia implementazioni reali

Confronta la tua implementazione con tool di produzione:
- **Kong API Gateway** - Supporta più strategie di rate limiting, leggi il codice del loro plugin Lua
- **Cloudflare Workers** - Rate limiting all'edge, studia i post del loro blog sull'architettura
- **GitHub API** - Leggi la loro documentazione sul rate limiting, comprendi il loro sistema di livelli

Leggi il loro codice, comprendi i loro compromessi, ruba le loro buone idee.

### Scrivine

Documenta le tue estensioni:
- Post sul blog: "Adding Geolocation-Based Rate Limiting to FastAPI"
- Tutorial: "How I Built a Rate Limiting Dashboard in a Weekend"
- Confronto: "Sliding Window vs GCRA: Performance Benchmarks"

Insegnare agli altri è il modo migliore per verificare di aver capito.

## Ottenere aiuto

Bloccato su una sfida?

1. **Fai debug in modo sistematico**
   - Cosa ti aspettavi? (Comportamento specifico)
   - Cosa è successo davvero? (Log, errori, output sbagliato)
   - Qual è il caso di test più piccolo che lo riproduce?

2. **Leggi il codice esistente**
   - Funzionalità simili già implementate? (Guarda algorithms/ per i pattern)
   - Test che mostrano come funzionano i componenti? (tests/test_*.py contiene esempi)

3. **Cerca problemi simili**
   - Issue di GitHub su progetti simili
   - Stack Overflow per errori specifici
   - Documentazione di Redis/FastAPI per domande sulle API

4. **Chiedi aiuto**
   - Apri una discussione su GitHub con:
     - La sfida su cui stai lavorando
     - Cosa hai provato
     - L'errore specifico o il comportamento inatteso
     - La tua ipotesi su cosa non va
   - Non limitarti a incollare gli errori. Spiega cosa hai capito.

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

- [ ] Sfida facile 1: header personalizzati
- [ ] Sfida facile 2: sintassi con decorator
- [ ] Sfida facile 3: prefissi per ambiente
- [ ] Sfida intermedia 4: ponderazione del costo delle richieste
- [ ] Sfida intermedia 5: integrazione CAPTCHA
- [ ] Sfida intermedia 6: limiti per geolocalizzazione
- [ ] Sfida avanzata 7: dashboard dei rate limit
- [ ] Sfida avanzata 8: circuit breaker distribuito
- [ ] Sfida avanzata 9: algoritmo GCRA

Le hai completate tutte? Hai padroneggiato il rate limiting. È il momento di costruire qualcosa di nuovo o di restituire qualcosa alla community.
