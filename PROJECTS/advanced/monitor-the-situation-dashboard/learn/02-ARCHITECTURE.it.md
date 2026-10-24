<!--
©AngelaMos | 2026
02-ARCHITECTURE.md
-->

# Architettura

Questo documento copre la forma a runtime del sistema: come un singolo processo Go orchestra undici feed di dati eterogenei, come gli eventi passano da una goroutine collector a un browser, e come lo stato sopravvive a un riavvio.

## Il quadro generale

```
┌──────────────────────────────────────────────────────────────────────┐
│  External feeds (HTTP / WS, varying cadences)                        │
│  DShield · CF Radar · NVD · EPSS · KEV · ransomware.live · Coinbase  │
│  · USGS · NOAA SWPC · Wikipedia · GDELT · CelesTrak · wheretheiss    │
└───┬──────────────────────────────────────────────────────────────────┘
    │
    │   ▼ each collector is one errgroup goroutine
    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  collector   │  │  collector   │  │  collector   │  │  collector   │
│  (cve)       │  │  (kev)       │  │  (cfradar)   │  │  …           │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │                 │
       │  Emit(events.Event{Topic, Source, Timestamp, Payload})
       ▼                 ▼                 ▼                 ▼
       ╔═══════════════════════════════════════════════════════╗
       ║          bus.Bus  (buffered channel, 512)             ║
       ║                                                       ║
       ║   1. Persister.Save(ev)  → snapshot store (Redis)     ║
       ║   2. Broadcaster.Broadcast(topic, payload) → ws.Hub   ║
       ║   3. fanout(ev) → channel subscribers (alerts engine) ║
       ╚═══════════════════════════════════════════════════════╝
                         │              │              │
                         ▼              ▼              ▼
                    ┌─────────┐   ┌──────────┐   ┌─────────────┐
                    │ Redis   │   │ ws.Hub   │   │ alerts.     │
                    │ snapshot│   │ per-conn │   │ Dispatcher  │
                    │ store   │   │ buf 256  │   │ (CEL eval)  │
                    └─────┬───┘   └────┬─────┘   └──────┬──────┘
                          │            │                │
                          ▼            ▼                ▼
                    GET /v1/snapshot   /v1/ws        Telegram /
                    (REST hydrate)     (WS stream)   Discord / Slack

  Append-mostly Postgres tables (BRIN indexed) live alongside Redis
  for historical queries via /v1/intel/* REST endpoints (CVE search,
  KEV list, hijack timeline, ransomware victims, earthquakes).
```

Il bus è la spina dorsale. Tutto il resto è un producer (collector), un consumer (snapshot store, hub ws, dispatcher degli alert) o un percorso di query che bypassa del tutto il bus (le query storiche su Postgres servite dal package `intel`).

## La decisione del binario singolo

L'intero backend è `cmd/api/main.go` che esegue un solo binario. Non c'è uno scheduler, né un worker separato, né un message broker. Questo è intenzionale e portante.

L'orchestrazione è un singolo `errgroup.Group`:

```go
collectorGroup, collectorCtx := errgroup.WithContext(ctx)
collectorGroup.Go(func() error { return eventBus.Run(collectorCtx) })
collectorGroup.Go(func() error { return beat.Run(collectorCtx) })

if cfg.Collectors.DShield.Enabled {
    coll := dshield.NewCollector(dshield.CollectorConfig{...})
    collectorGroup.Go(func() error { return coll.Run(collectorCtx) })
}
// ... ten more collectors ...
```

`errgroup.WithContext` ti dà due garanzie:

1. **Cancellazione condivisa**. Se una qualsiasi goroutine restituisce un errore, il context di ogni altra goroutine viene cancellato. Un collector che esplode non lascia silenziosamente gli altri in esecuzione su un processo mezzo rotto.
2. **Teardown con attesa di tutti**. `collectorGroup.Wait()` si blocca finché ogni goroutine non ha effettivamente restituito. Combinato con un context padre `signal.NotifyContext(SIGINT, SIGTERM)` e un ritardo di drenaggio di 5 secondi prima che il server HTTP si fermi, ogni evento in volo ha la possibilità di essere salvato e trasmesso prima che il processo termini.

Il costo di questo design: non puoi scalare orizzontalmente la vista di un singolo utente su più processi (non c'è un bus condiviso). Il beneficio: zero modalità di guasto da sistema distribuito per ~99% dei deployment. **Se mai vorrai il multi-processo, sostituisci `bus.Bus` con NATS JetStream e mantieni tutto il resto.**

## L'event bus

`bus.Bus` è un channel Go con buffer con tre effetti a valle per ogni evento. La forma:

```go
type Bus struct {
    ch          chan events.Event   // size 512
    persister   Persister           // redis snapshot store
    broadcaster Broadcaster         // ws hub
    subscribers []chan events.Event // alerts dispatcher etc
}

func (b *Bus) Emit(ev events.Event) {
    select {
    case b.ch <- ev:
    default:
        b.dropped.Add(1)
        b.logger.Warn("event bus full, dropped", ...)
    }
}
```

Tre cose qui non sono ovvie:

- **`Emit` non è bloccante**. Un bus intasato scarta gli eventi invece di bloccare i collector. È la scelta giusta quando i collector sono sensibili al tempo (Coinbase, USGS) e preferiscono perdere un tick piuttosto che restare indietro rispetto al tempo reale.
- **Una sola goroutine drena il channel**. Il loop `Run` preleva gli eventi dal buffer, li salva, li trasmette e li distribuisce, in quell'ordine. Se il salvataggio è lento, la trasmissione resta indietro. Questo è accettabile perché lo snapshot è la fonte di verità per i client che si uniscono in ritardo.
- **I subscriber sono separati dal broadcast**. Il motore degli alert si iscrive tramite `Bus.Subscribe()`, che restituisce un channel con buffer da 256. Una valutazione lenta degli alert non intasa il bus; gli eventi vengono invece scartati al confine del subscriber.

Il bus è l'unico punto in cui il sistema tollera perdite. Sotto il bus (le scritture su Postgres dentro i repository dei collector) tutto è durevole. Sopra il bus (snapshot store) tutto è eventualmente consistente con semantica last-write-wins — un evento scartato significa semplicemente che l'evento successivo per quel topic sovrascrive quello perso qualche secondo dopo.

## Il pattern del collector

Ogni collector implementa approssimativamente:

```go
type CollectorConfig struct {
    Interval  time.Duration
    Fetcher   Fetcher       // calls the upstream API
    Repo      Repository    // persists historical rows
    Emitter   Emitter       // bus.Emit
    State     StateRecorder // health/freshness tracking
    Logger    *slog.Logger
}

func (c *Collector) Run(ctx context.Context) error {
    ticker := time.NewTicker(c.cfg.Interval)
    defer ticker.Stop()
    c.tick(ctx)             // immediate first poll
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            c.tick(ctx)
        }
    }
}
```

Il corpo di `tick` è specifico per ogni sorgente upstream ma la struttura è la stessa: recupera, salva nella propria tabella, emette un singolo evento. Fetcher, repository, emitter e state recorder sono interfacce — ogni collector viene fornito con un fake di ciascuna, così gli unit test non hanno bisogno di rete o database.

Alcuni collector sono più strani di altri:

- **DShield** unisce tre endpoint in un unico snapshot. Un `Enricher` opzionale arricchisce il blob dei top IP con il codice paese da AbuseIPDB e la classificazione da GreyNoise.
- **Coinbase** è un WebSocket persistente, non un poller. Il package `coinbase` si scompone in `client` (WS grezzo), `readloop` (lettura frame per frame con sequencer), `aggregator` (tick → barra al minuto), `reconnect` (backoff esponenziale) e `collector` (orchestrazione).
- **ISS** ha due cadenze: `PositionInterval` (10s) per la posizione live tramite `wheretheiss.at`, e `TLEInterval` (24h) per aggiornare il set di elementi orbitali da CelesTrak. Il TLE in cache viene propagato lato client tra un polling e l'altro, così il globo non resta bloccato sulla rete.
- **SWPC** ha sia una cadenza veloce (1m) sia una lenta (3h). L'indice Kp si aggiorna ogni 3h; il Bz del magnetometro si aggiorna ogni minuto. Un solo collector gestisce entrambe.

Ogni collector riporta tramite `state.RecordSuccess` o `state.RecordError` a ogni tick. La tabella `collector_state` è la fonte di verità per l'endpoint `/v1/admin/health`, che accende di rosso i pannelli quando un feed non si è aggiornato entro la finestra prevista.

## Lo snapshot store

Un nuovo browser deve mostrare uno stato utile immediatamente, prima che arrivi qualsiasi evento WebSocket. Lo snapshot store risolve questo problema.

`snapshot.Store` è un sottile wrapper su Redis. Il lato persister, collegato al bus, chiama `Save` per ogni evento:

```go
func (s StorePersister) Save(ctx context.Context, ev events.Event) error {
    raw, _ := json.Marshal(ev.Payload)
    if ev.Topic == events.TopicCoinbasePrice {
        return s.Store.MergeSymbolMap(ctx, ev.Topic, raw)
    }
    return s.Store.PutLatest(ctx, ev.Topic, raw)
}
```

Per la maggior parte dei topic, "latest" è una chiave Redis (`state:cve_new`, `state:kev_added`, ecc.) con il payload JSON grezzo. Per Coinbase, dove ci sono più simboli, il topic è un hash Redis con un campo per simbolo, così un tick di BTC non calpesta l'ultimo tick di ETH.

Il lato di servizio, dietro `GET /v1/snapshot`, scansiona `state:*` e restituisce un unico documento JSON unificato. Il frontend idrata tutto in un solo round trip, poi si allinea tramite WebSocket.

Lo store **non** salva dati storici. Lo fa Postgres. Lo store è una cache per l'avvio a freddo, non il sistema di record. Se Redis viene svuotato, la dashboard riparte a freddo per ~2 ore (finché il collector più lento non completa un ciclo) ma non perde alcun dato durevole.

## L'hub WebSocket

`ws.Hub` è un router di fan-out con buffer limitati per subscriber ed espulsione aggressiva dei consumer lenti.

```go
const (
    defaultSubscriberBuf = 256
    defaultPingInterval  = 30 * time.Second
    defaultMaxSubs       = 5000
)

func (h *Hub) Broadcast(topic events.Topic, payload []byte) {
    env, _ := EncodeEnvelope(string(topic), payload)
    h.mu.Lock()
    defer h.mu.Unlock()
    for sub := range h.subs {
        if _, ok := sub.topics[topic]; !ok {
            continue
        }
        select {
        case sub.msgs <- env:
        default:
            go sub.closeSlow()
        }
    }
}
```

Cose da notare:

- **Filtro per topic per subscriber**. Il client che si connette invia `?topics=cve_new,kev_added,...` e l'hub invia solo i topic corrispondenti. Un pannello che non ha bisogno dei tick di Coinbase non li paga.
- **Invio non bloccante**. Se il channel da 256 messaggi del subscriber è pieno, l'hub non aspetta. Pianifica una goroutine `closeSlow()` che chiude la connessione con `StatusPolicyViolation`.
- **Ping di heartbeat ogni 30s**. La logica di riconnessione del browser usa i pong mancanti come segnale di disconnessione; è più veloce e affidabile che aspettare un TCP RST.
- **Limite di capacità**. `MaxSubscribers: 5000` (configurabile) protegge contro i DoS da flood di connessioni. Oltre la capacità, le nuove connessioni ricevono `StatusTryAgainLater`.

Il formato dell'envelope è intenzionalmente minimale:

```json
{"ch":"cve_new","data":{...},"ts":"2026-05-08T18:30:12.123456789Z"}
```

`ch` è il topic, `data` è il payload grezzo, `ts` è il timestamp lato server al momento del fan-out. Nessun versioning del protocollo, nessuna autenticazione in banda — l'autenticazione avviene sull'handshake di upgrade.

## L'handshake snapshot+stream

La factory `createDashboardWS` del frontend implementa il pattern buffer-then-flush:

```ts
return {
  connect() { attach() },
  setReady() {
    ready = true
    sendInitIfReady()
    const flush = buffer
    buffer = []
    for (const ev of flush) onEvent(ev)
  },
  ...
}
```

Il ciclo di vita:

1. La app si avvia. `connect()` apre il WebSocket. `ready = false`. I messaggi in arrivo vanno in `buffer`.
2. La app recupera `GET /v1/snapshot` tramite TanStack Query. La risposta idrata gli store Zustand.
3. La app chiama `setReady()`. I messaggi in buffer vengono riprodotti attraverso `onEvent`. `ready = true`. I messaggi successivi fluiscono live.

L'unico messaggio client→server del protocollo è `{"op":"init"}`, inviato da `setReady()`. L'hub lato server legge e scarta i messaggi del client — il protocollo è solo server-push — ma il codice precedente chiudeva la connessione con `StatusPolicyViolation` alla ricezione di qualsiasi frame di dati, il che trasformava ogni `init` del client in una riconnessione forzata. Il loop `Serve` attuale legge e scarta in una goroutine separata per mantenere viva la connessione.

Questo handshake è la parte del sistema in assoluto più soggetta a errori. Gli unit test in `frontend/src/api/ws.test.ts` e `ws/hub_test.go` del backend esistono proprio perché l'abbiamo sbagliato due volte durante lo sviluppo.

## Storage: Redis + Postgres

La suddivisione è deliberata.

**Redis** contiene:

- Lo snapshot store (`state:*`) — una chiave per topic, last-write-wins.
- Il TLE della ISS — piccolo JSON, aggiornato ogni giorno, letto di continuo.
- I contatori del rate limiting — finestra scorrevole per IP/utente.
- I cooldown del motore degli alert — chiavi `SET NX EX` per (regola, canale).
- I token di sessione di autenticazione (rotazione del refresh token).

**Postgres** contiene tutto ciò che deve essere interrogabile storicamente: CVE, voci KEV, terremoti, vittime ransomware, tick e barre al minuto di BTC/ETH, eventi di interruzione, BGP hijack, snapshot di DShield, eventi mondiali, definizioni delle regole di alert, storico degli alert inviati, account utente, configurazioni dei canali di notifica.

Redis è il percorso caldo. Postgres è il percorso freddo. I due non dipendono mai l'uno dall'altro per la correttezza; se Redis è giù, la dashboard non ha stato live ma gli endpoint REST `intel` (`/v1/intel/cve/search`, `/v1/intel/kev/list`, ecc.) funzionano ancora direttamente su Postgres.

Lo schema Postgres usa BRIN su ogni colonna temporale e B-tree sulle chiavi primarie più alcuni indici selettivi (severity+lastmod per le CVE, mag+occurred per i terremoti, confidence+detected per gli hijack). Tutte le colonne di payload sono `jsonb` perché gli schemi evolvono più velocemente delle migration.

## Il motore degli alert

Il motore degli alert è un consumer in-process iscritto al bus. Per ogni evento, cerca le regole indicizzate per topic, valuta il predicato CEL di ogni regola sul payload e, se corrisponde, invia ai canali configurati dall'utente.

Indicizzato per topic, così il percorso caldo è `O(rules-for-this-topic)` e non `O(all-rules)`:

```go
e.rulesByTopic.Store(&indexed)  // atomic.Pointer[map[string][]compiledRule]
```

Lo scambio del puntatore fa sì che i ricaricamenti delle regole (ogni 30s) siano lock-free per il percorso di valutazione. Il refresh compila semplicemente il predicato CEL di ogni regola in un `cel.Program` e scambia atomicamente l'indice.

I cooldown sono per `(rule, channel)`:

```go
key := fmt.Sprintf("alert_cooldown:%s:%s:%s", rule.ID, ch.Type, ch.ID)
ok, err := e.cooldowns.TryAcquire(ctx, key, cooldown)
```

`TryAcquire` è un `SET NX EX` di Redis. Un utente con regole su Slack e Telegram per lo stesso predicato ha finestre di cooldown indipendenti su ciascuno — importante perché Slack e Telegram hanno rate limit diversi e un singolo cooldown globale farebbe sì che uno silenzi l'altro.

CEL è stato scelto al posto di un DSL personalizzato o di Lua. Motivi: è in sandbox per design (nessuna syscall, nessun I/O), ha un'impronta minuscola sul binario Go, e scrivere predicati è familiare a chiunque abbia usato condizioni IAM, regole Cloud Armor o filtri Envoy.

## La pipeline di autenticazione

Il flusso di autenticazione è un convenzionale JWT ES256 con chiavi a rotazione automatica:

```
1. POST /v1/auth/register
   → Argon2id(password) → users.password_hash
   → seed default alert rules

2. POST /v1/auth/login
   → verify Argon2id
   → mint access_token (15m, ES256, kid in header)
   → mint refresh_token (30d, opaque, stored in Redis)
   → set HttpOnly Secure cookies in production

3. Every authenticated request
   → middleware.Authenticator(verifier)
   → verifier checks ES256 signature + revocation list
   → ctx is populated with user identity

4. Public endpoints get a guest path
   → /v1/snapshot, /v1/ws, /v1/intel/* are read-only and unauthenticated
   → /v1/alerts/* and /v1/notifications/* require auth

5. Admin endpoints
   → middleware.RequireAdmin checks email vs cfg.App.AdminEmail
   → /v1/admin/db, /v1/admin/redis, /v1/admin/users
```

La chiave di firma si trova in `cfg.JWT.PrivateKeyPath`. Se manca all'avvio, `ensureJWTKeys` genera una nuova coppia di chiavi ES256 in `keys/`. Il lato pubblico è esposto su `/.well-known/jwks.json`, così qualsiasi servizio esterno può verificare i token della dashboard senza fidarsi del filesystem della dashboard.

I refresh token vengono ruotati a ogni utilizzo (swap-and-set su Redis), quindi un refresh token trafugato ha al massimo un utilizzo prima di essere invalidato.

## Architettura dello stato del frontend

Grosso modo l'inverso del backend:

- **TanStack Query v5** — REST. Gestisce il fetch dello snapshot, le query storiche `/v1/intel/*` e qualsiasi mutation dovuta ad azioni dell'utente (creare una regola, eliminare un canale).
- **Store Zustand** — stato live. Uno store per famiglia di topic: `prices`, `ticker`, `cve`, `kev`, `bgpHijack`, `outage`, `earthquake`, `ransomware`, `freshness`, `globeEvents`, `heartbeat`, `audio`, `ui`.
- **Selettori dei componenti** — i componenti si iscrivono a specifiche porzioni dello store tramite il pattern dei selettori di Zustand, così un aggiornamento di una CVE non fa ri-renderizzare il pannello di BTC.

L'handler dei messaggi WebSocket è un'unica tabella di dispatch che instrada gli envelope `{ch, data}` in arrivo verso lo store giusto. Aggiungere un nuovo feed al backend richiede di aggiungere una voce a questa tabella e un nuovo store Zustand.

Il globo (`pages/globe/Globe.tsx`) è il componente più pesante. Usa MapLibre per la proiezione e disegna cinque tipi distinti di layer: coropleta dei paesi (interruzioni), percorsi a cerchio massimo (BGP hijack), nuvole di punti (punti ASN di DShield, epicentri dei terremoti, marker ransomware) e una scia animata (traccia a terra della ISS). Gli aggiornamenti dei layer sono raggruppati per animation frame, perché modificare singolarmente i layer MapLibre per ogni messaggio WebSocket è il modo più rapido per far crollare il frame rate.

## Modalità di guasto a cui l'architettura sopravvive

| Guasto | Effetto | Ripristino |
|---|---|---|
| Un singolo feed va giù (es. USGS 502) | Quel collector logga l'errore, `RecordError`, la dashboard mostra quel pannello in giallo "stale". Gli altri pannelli non sono interessati. | Il tick successivo va a buon fine, la freschezza si ripristina. |
| Redis va giù | Lo snapshot restituisce vuoto, il WS funziona ancora, gli endpoint intel funzionano ancora. | Il persister del bus logga errori, non scarta nient'altro. |
| Postgres va giù | I collector che devono fare upsert loggano errori. L'emit del bus + WS funzionano ancora. La REST intel restituisce 503. | Postgres si riconnette, i collector riprendono al tick successivo. |
| Channel del bus pieno | `Emit` scarta l'evento, incrementa il contatore. Lo snapshot perde un aggiornamento. | L'evento successivo per quel topic sovrascrive nello snapshot. |
| Un subscriber WS si blocca | L'hub lo chiude con `StatusPolicyViolation`. | Il browser si riconnette, ottiene uno snapshot fresco, continua. |
| Processo terminato a metà scrittura | `errgroup.Wait()` blocca lo shutdown finché il lavoro in volo non termina (entro `ShutdownTimeout`). | Il riavvio riprende da dove si era interrotto, nessuna perdita di dati per le tabelle durevoli. |

Il sistema **non** è progettato per sopravvivere alla perdita dell'host. Non c'è replica, né elezione del leader, né quorum. È una dashboard a binario singolo su host singolo. Se ti serve l'alta disponibilità, metti una seconda istanza dietro un load balancer con la propria replica Postgres e accetti che le due istanze vedano stati live leggermente diversi (perché i tick WS di Coinbase arrivano in momenti diversi). Per il caso d'uso originale — un operatore, uno schermo, una macchina — quella complessità è il compromesso sbagliato.

## Dove andare dopo

Prosegui con [03-IMPLEMENTATION.md](03-IMPLEMENTATION.it.md) per il vero walkthrough del codice attraverso collector, bus, ws, alert, snapshot e collegamento del frontend.
