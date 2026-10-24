<!--
©AngelaMos | 2026
03-IMPLEMENTATION.md
-->

# Implementazione

Questo è il walkthrough a livello di codice. È organizzato per area di responsabilità, non per file. Seguiamo il percorso di un evento dall'API esterna fino a un pixel sul globo, poi torniamo indietro per coprire il codice trasversale.

Tutti gli snippet sono estratti dal codice reale. Gli snippet intenzionalmente non sono annotati con i numeri di riga — la formattazione e i piccoli refactoring invalidano i numeri di riga nel giro di pochi giorni, e il package + il nome della funzione bastano per ritrovare qualsiasi blocco con `grep`.

## Il tipo Event

Tutto nel sistema passa attraverso `events.Event`. È deliberatamente minuscolo.

```go
// internal/events/event.go
package events

import "time"

type Event struct {
    Topic     Topic     `json:"topic"`
    Timestamp time.Time `json:"timestamp"`
    Source    string    `json:"source"`
    Payload   any       `json:"payload"`
}
```

`Payload any` è la scelta interessante. Avremmo potuto renderlo un'unione sigillata di payload tipizzati, uno per topic. Abbiamo scelto di non farlo. I motivi:

- **Forme di payload eterogenee.** Un'aggiunta alla KEV è un singolo ID CVE con una data. Uno snapshot di DShield è un blob fuso proveniente da tre endpoint. Un tick di Coinbase è un prezzo più un volume. Forzarli in un unico albero di tipi Go non aggiunge alcuna sicurezza a runtime, perché i consumer (snapshot store, hub ws, motore degli alert) fanno tutti immediatamente un nuovo marshal in JSON.
- **Il bus è l'unico confine tipizzato lato producer.** Ogni collector impacchetta il proprio payload come `json.RawMessage` prima di emetterlo, quindi la forma del JSON viene fissata nel collector. I subscriber pagano un `json.Unmarshal` se hanno bisogno dei campi.

Il compromesso è reale: un errore di battitura nel nome di un campo del payload in un collector non verrà intercettato a tempo di compilazione. Lo mitighiamo con test golden-file nella directory `testdata/` di ogni collector.

## L'enum dei topic

```go
// internal/events/topic.go
type Topic string

const (
    TopicHeartbeat        Topic = "heartbeat"
    TopicScanFirehose     Topic = "scan_firehose"
    TopicInternetOutage   Topic = "internet_outage"
    TopicBGPHijack        Topic = "bgp_hijack"
    TopicCVENew           Topic = "cve_new"
    TopicCVEUpdated       Topic = "cve_updated"
    TopicEPSS             Topic = "epss"
    TopicKEVAdded         Topic = "kev_added"
    TopicRansomwareVictim Topic = "ransomware_victim"
    TopicCoinbasePrice    Topic = "coinbase_price"
    TopicEarthquake       Topic = "earthquake"
    TopicSpaceWeather     Topic = "space_weather"
    TopicWikipediaITN     Topic = "wiki_itn"
    TopicGDELTSpike       Topic = "gdelt_spike"
    TopicISSPosition      Topic = "iss_position"
    TopicCollectorState   Topic = "collector_state"
)

func (t Topic) IsValid() bool { ... }
func AllTopics() []Topic       { ... }
```

`IsValid` è usato dall'handler WebSocket per mettere in whitelist i parametri di query `?topics=`. `AllTopics` è il fallback predefinito quando un client si connette senza specificare i topic. Aggiungere un feed significa aggiungere una costante qui, un case in `IsValid` e una voce in `AllTopics`. Poi un collector emette con il nuovo topic e il layer WS lo raccoglie automaticamente.

## Il bus

`internal/bus/bus.go` è la spina dorsale.

```go
type Bus struct {
    ch          chan events.Event
    persister   Persister
    broadcaster Broadcaster
    logger      *slog.Logger
    dropped     atomic.Uint64

    subsMu      sync.RWMutex
    subscribers []chan events.Event
    subDropped  atomic.Uint64
}

func (b *Bus) Emit(ev events.Event) {
    select {
    case b.ch <- ev:
    default:
        b.dropped.Add(1)
        b.logger.Warn("event bus full, dropped", "topic", ev.Topic, "source", ev.Source)
    }
}
```

`Emit` è l'API lato producer. La `select` con un `default` la rende non bloccante — se il buffer è pieno, scartiamo e incrementiamo il contatore invece di bloccare la goroutine del collector chiamante.

Il lato di drenaggio, in `Run`, è il consumer:

```go
func (b *Bus) Run(ctx context.Context) error {
    defer b.closeSubscribers()
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case ev := <-b.ch:
            if b.persister != nil {
                if err := b.persister.Save(ctx, ev); err != nil {
                    b.logger.Error("persist event failed", "err", err, "topic", ev.Topic)
                }
            }
            if b.broadcaster != nil {
                payload, err := json.Marshal(ev.Payload)
                if err != nil {
                    b.logger.Error("marshal payload failed", "err", err, "topic", ev.Topic)
                    continue
                }
                b.broadcaster.Broadcast(string(ev.Topic), payload)
            }
            b.fanout(ev)
        }
    }
}
```

L'ordine conta. Prima si salva, così lo snapshot è corretto per qualsiasi browser che si connetta tra questo evento e il successivo. Poi si trasmette, così i consumer live vedono l'evento subito dopo il salvataggio. Infine si distribuisce ai subscriber dei channel (motore degli alert), perché potrebbero valutare predicati CEL lenti.

`fanout` è il secondo punto in cui si scartano eventi:

```go
func (b *Bus) fanout(ev events.Event) {
    b.subsMu.RLock()
    subs := b.subscribers
    b.subsMu.RUnlock()
    for _, ch := range subs {
        select {
        case ch <- ev:
        default:
            b.subDropped.Add(1)
        }
    }
}
```

Un motore degli alert lento non può intasare il bus. Perde semplicemente delle valutazioni di alert, cosa che viene loggata e resa visibile tramite `SubscriberDroppedCount()` per il monitoraggio operativo.

## Un tour guidato di un collector

Ogni collector ha lo stesso scheletro: polling, fetch, salvataggio delle righe, emissione degli eventi, registrazione dello stato. Useremo quello delle CVE perché è il più rappresentativo — HTTP interrogabile, dipendente da una seconda API (EPSS), con sia upsert sia emissione di eventi per riga.

### Collector delle CVE — Il tick

```go
// internal/collectors/cve/collector.go
func (c *Collector) tick(ctx context.Context) {
    end := time.Now().UTC()
    start := end.Add(-c.cfg.Window)

    resp, err := c.cfg.NVD.Fetch(ctx, start, end)
    if err != nil {
        c.logger.Warn("nvd fetch", "err", err)
        c.cfg.State.RecordError(ctx, Name, err.Error())
        return
    }
```

Nota la struttura: una finestra `[end - Window, end]`, non "tutto". I feed delle CVE sono prevalentemente append ma possono ripubblicare modifiche, quindi una finestra di 2 ore con un intervallo di polling di 2 ore dà una sovrapposizione di una finestra che cattura le modifiche tardive senza rielaborare l'intero storico.

```go
    ids := make([]string, 0, len(resp.Vulnerabilities))
    rows := make([]Row, 0, len(resp.Vulnerabilities))
    for _, v := range resp.Vulnerabilities {
        score, severity := v.PrimarySeverity()
        raw, _ := json.Marshal(v)
        rows = append(rows, Row{
            CveID:        v.CVE.ID,
            Published:    v.CVE.Published.Time,
            LastModified: v.CVE.LastModified.Time,
            Severity:     severity,
            CVSS:         score,
            Payload:      raw,
        })
        ids = append(ids, v.CVE.ID)
    }

    scores, err := c.cfg.EPSS.LookupBatch(ctx, ids)
```

Due chiamate API per tick: NVD per il corpo delle CVE, EPSS per i punteggi. Le facciamo in sequenza perché EPSS è indicizzato sugli ID CVE appena restituiti dalla chiamata a NVD. L'errore di EPSS viene loggato ma non interrompe il tick — una CVE senza punteggio EPSS è comunque utile, solo non altrettanto ben prioritizzata.

```go
    for _, row := range rows {
        if err := c.cfg.Repo.Upsert(ctx, row); err != nil { ... }
        if s, ok := scores[row.CveID]; ok {
            _ = c.cfg.Repo.UpdateEPSS(ctx, row.CveID, s.Score, s.Percentile)
            row.EPSSScore = &s.Score
            row.EPSSPercentile = &s.Percentile
        }
        body, _ := json.Marshal(row)
        c.cfg.Emitter.Emit(events.Event{
            Topic:     events.TopicCVENew,
            Timestamp: end,
            Source:    Name,
            Payload:   json.RawMessage(body),
        })
    }
    c.cfg.State.RecordSuccess(ctx, Name, emitted)
}
```

Salva-poi-emetti, per ogni riga. Emettiamo `TopicCVENew` per ogni riga nella finestra, anche per quelle già presenti nella tabella. Il motivo: il punteggio EPSS di una CVE può cambiare senza che la CVE stessa cambi, e il frontend dovrebbe rifletterlo. Potremmo separare in `cve_new` e `cve_updated` in base a insert o update, ma i consumer non hanno ancora bisogno di questa distinzione.

`RecordSuccess` scrive una riga in `collector_state`, così sia `/v1/admin/health` sia la fascia di freschezza sul frontend possono vedere "il collector delle CVE è stato eseguito 12 minuti fa, ha elaborato 47 righe".

### DShield — Fusione di più endpoint

DShield è l'esempio più pulito del pattern "fondere più endpoint upstream in un unico evento".

```go
// internal/collectors/dshield/collector.go
snaps, err := c.cfg.Fetcher.FetchAll(ctx) // hits 3 endpoints concurrently

merged := map[string]json.RawMessage{"ts": tsRaw}
for _, s := range snaps {
    payload := s.Payload
    if s.Kind == KindTopIPs {
        payload = c.enrichSources(ctx, payload)
    }
    _ = c.cfg.Persister.PutSnapshot(ctx, now, s.Kind, payload)
    merged[s.Kind] = payload
}

body, _ := json.Marshal(merged)
c.cfg.Emitter.Emit(events.Event{
    Topic:     events.TopicScanFirehose,
    Timestamp: now,
    Source:    Name,
    Payload:   json.RawMessage(body),
})
```

Il JSON unificato ha la forma `{ts, top_ips, top_ports, country_dist}`. Il `DShieldPanel` del frontend consuma il blob unificato in un solo passaggio di rendering. Il salvataggio su Postgres è per tipo, così possiamo interrogare "mostrami i top IP del 2026-04-01" senza fare il parsing di un blob fuso.

Il passaggio `enrichSources` è opzionale e viene eseguito solo se è configurato un `Enricher` (AbuseIPDB + GreyNoise). Itera sulla lista dei top IP, cerca ogni IP e restituisce righe arricchite con codice paese, classificazione e nome del threat actor. Se il lookup fallisce o l'enricher è nil, ricade sul payload originale.

### Coinbase — WebSocket persistente

Coinbase è l'unico collector che non fa polling. La scomposizione in più file riflette quanto diventano disordinati i veri client WebSocket:

- `client.go` — connessione grezza, `ReadFrame`, `WriteFrame`.
- `readloop.go` — loop dei frame con reset del sequencer sui frame di snapshot.
- `aggregator.go` — OHLC da tick a barra al minuto.
- `reconnect.go` — backoff esponenziale alla disconnessione.
- `sequencer.go` — rilevamento dei buchi su `sequence_num`.
- `collector.go` — orchestrazione.

L'aggregator è il più interessante:

```go
// internal/collectors/coinbase/aggregator.go
func (a *Aggregator) Push(t Tick) (*MinuteBar, MinuteBar) {
    minute := t.TS.UTC().Truncate(time.Minute)
    cur, exists := a.open[t.Symbol]

    if !exists {
        cur = MinuteBar{Symbol: t.Symbol, Minute: minute, Open: t.Price, High: t.Price, Low: t.Price, Close: t.Price, Volume24hAtClose: t.Volume24h}
        a.open[t.Symbol] = cur
        return nil, cur
    }

    if minute.After(cur.Minute) {
        closed := cur
        cur = MinuteBar{...}  // start new minute
        a.open[t.Symbol] = cur
        return &closed, cur
    }

    // same minute, update OHLC
    if t.Price.GreaterThan(cur.High) { cur.High = t.Price }
    if t.Price.LessThan(cur.Low)     { cur.Low  = t.Price }
    cur.Close = t.Price
    cur.Volume24hAtClose = t.Volume24h
    a.open[t.Symbol] = cur
    return nil, cur
}
```

Restituisce `(closed, current)`. Quando un tick attraversa il confine di un minuto, il minuto precedente viene restituito come `closed` e il collector lo salva in `btc_eth_minute`. Il minuto corrente viene sempre restituito per la visualizzazione live.

È anche il motivo per cui la colonna del volume al minuto è stata rinominata nella migration `0005`: la colonna originale si chiamava `volume` e avevamo documentato "volume per minuto", ma il channel ticker di Coinbase fornisce solo il volume mobile sulle 24 ore, non la dimensione per singolo trade. La rinomina in `volume_24h_at_close` rende visibile nello schema la semantica effettiva.

### ISS — Due cadenze, propagazione lato client

```go
// cmd/api/main.go
coll := iss.NewCollector(iss.CollectorConfig{
    PositionInterval: cfg.Collectors.ISS.PositionInterval,  // 10s
    TLEInterval:      cfg.Collectors.ISS.TLEInterval,       // 24h
    Fetcher:          iss.NewClient(iss.ClientConfig{}),
    TLEStore:         iss.NewTLEStore(redis.Client),
    ...
})
```

Il collector della ISS interroga `wheretheiss.at` ogni 10s per la posizione live (questa è la fonte usata dal "pin corrente della ISS" della dashboard). Interroga anche CelesTrak ogni 24 ore per il TLE (Two-Line Element set), lo mette in cache in Redis e lo emette.

Il frontend ha `frontend/src/lib/sgp4.ts` (o equivalente) che propaga il TLE in cache a ogni animation frame per disegnare la traccia orbitale. Quindi l'orbita visibile è calcolata lato client a partire da un TLE vecchio di 24 ore; solo il "pin corrente" viene inviato dal server ogni 10s.

Questo è un buon esempio di **spostare il calcolo sul client** quando l'algoritmo è ben definito e l'input di dati è piccolo. SGP4 è ~200 righe di matematica; inviare la posizione propagata via WebSocket a ogni animation frame saturerebbe la rete senza alcun beneficio.

## L'hub WebSocket

`internal/ws/hub.go`:

```go
const (
    defaultSubscriberBuf = 256
    defaultPingInterval  = 30 * time.Second
    defaultPingTimeout   = 10 * time.Second
    defaultWriteTimeout  = 5 * time.Second
    defaultMaxSubs       = 5000
)

func (h *Hub) Broadcast(topic events.Topic, payload []byte) {
    env, err := EncodeEnvelope(string(topic), payload)
    if err != nil {
        h.logger.Error("encode envelope", "err", err, "topic", topic)
        return
    }
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

Il lock copre l'intera iterazione. Va bene così perché l'iterazione è veloce (solo un invio non bloccante sul channel per ogni sub) e il lock serializza rispetto ad `add`/`remove`, che sono gli unici altri scrittori di `h.subs`. La chiusura della connessione avviene in una goroutine separata per evitare di tenere il lock durante qualsiasi I/O.

Il commento sui consumer lenti nel blocco delle costanti merita di essere citato per intero, perché è il genere di cosa che impari solo sbagliando:

> Buffer in uscita per connessione. Deve assorbire il tasso di eventi di picco × la finestra di stallo del browser: heartbeat (5s) + coinbase (~8/s di picco) + ISS (10s) + occasionali raffiche di CVE/KEV/ransomware. 16 è troppo piccolo — una singola pausa del GC del browser lo riempie e innesca la chiusura per consumer lento, che l'utente vede come "NO SIGNAL".

### Il loop Serve

```go
// internal/ws/hub.go
func (h *Hub) Serve(ctx context.Context, c *cdrws.Conn, topics []events.Topic) error {
    sub := newSubscriber(topics, h.bufSize, func() {
        _ = c.Close(cdrws.StatusPolicyViolation, "slow consumer")
    })
    if !h.add(sub) {
        _ = c.Close(cdrws.StatusTryAgainLater, "server at capacity")
        return ErrAtCapacity
    }
    defer h.remove(sub)

    connCtx, cancelRead := context.WithCancel(ctx)
    defer cancelRead()
    go func() {
        for {
            if _, _, err := c.Read(connCtx); err != nil {
                cancelRead()
                return
            }
        }
    }()
    pingT := time.NewTicker(h.pingInterval)
    defer pingT.Stop()

    for {
        select {
        case msg := <-sub.msgs:
            wctx, cancel := context.WithTimeout(connCtx, h.writeTimeout)
            err := c.Write(wctx, cdrws.MessageText, msg)
            cancel()
            if err != nil { return err }
        case <-pingT.C:
            pctx, cancel := context.WithTimeout(connCtx, h.pingTimeout)
            err := c.Ping(pctx)
            cancel()
            if err != nil { return err }
        case <-connCtx.Done():
            _ = c.Close(cdrws.StatusNormalClosure, "")
            return connCtx.Err()
        }
    }
}
```

La goroutine di lettura dedicata è il post-mortem di un bug reale. La libreria `coder/websocket` ha un helper `c.CloseRead(ctx)` che chiude automaticamente la connessione alla ricezione di qualsiasi frame. Lo usavamo. Il `setReady()` del frontend invia `{"op":"init"}` dopo l'idratazione dello snapshot. Quel messaggio faceva scattare il percorso "received unexpected data" di `CloseRead`, uccidendo la connessione con `StatusPolicyViolation`. L'utente vedeva un ciclo di riconnessioni.

La correzione: drenare esplicitamente i messaggi client→server con `c.Read(connCtx)` e scartarli. La connessione resta viva; i frame ping/pong vengono comunque gestiti dalla libreria.

### Envelope

```go
// internal/ws/envelope.go
func EncodeEnvelope(channel string, payload []byte) ([]byte, error) {
    var buf bytes.Buffer
    buf.WriteString(`{"ch":`)
    chRaw, _ := json.Marshal(channel)
    buf.Write(chRaw)
    buf.WriteString(`,"data":`)
    if len(payload) == 0 {
        buf.WriteString("null")
    } else {
        buf.Write(payload)
    }
    buf.WriteString(`,"ts":`)
    tsRaw, _ := json.Marshal(time.Now().UTC().Format(time.RFC3339Nano))
    buf.Write(tsRaw)
    buf.WriteByte('}')
    return buf.Bytes(), nil
}
```

Costruito a mano, non tramite `json.Marshal` di una struct. Motivo: il campo `data` è JSON già codificato. Fare il marshal di una struct con `Data json.RawMessage` funziona ma passa per un percorso di encoder generico. La concatenazione è più veloce ed evita di riallocare per il campo più grande. A ~50k eventi/min di picco, questo conta.

## Lo snapshot store

```go
// internal/snapshot/store.go
const (
    keyPrefix       = "state:"
    keyHashPrice    = "state:coinbase_price"
    scanCount       = 100
    wrongTypeMarker = "WRONGTYPE"
)

func (s *Store) PutLatest(ctx context.Context, topic events.Topic, payload json.RawMessage) error {
    return s.rdb.Set(ctx, keyPrefix+string(topic), []byte(payload), 0).Err()
}

func (s *Store) MergeSymbolMap(ctx context.Context, topic events.Topic, payload json.RawMessage) error {
    var tick struct{ Symbol string `json:"symbol"` }
    if err := json.Unmarshal(payload, &tick); err != nil || tick.Symbol == "" {
        return fmt.Errorf("coinbase merge: no symbol in payload")
    }
    key := keyPrefix + string(topic)
    err := s.rdb.HSet(ctx, key, tick.Symbol, []byte(payload)).Err()
    if err == nil { return nil }
    if !isWrongType(err) { return fmt.Errorf("redis hset %s/%s: %w", topic, tick.Symbol, err) }
    if delErr := s.rdb.Del(ctx, key).Err(); delErr != nil { ... }
    // retry as hash
    return s.rdb.HSet(ctx, key, tick.Symbol, []byte(payload)).Err()
}
```

`MergeSymbolMap` ha un percorso di migrazione auto-riparante: se la chiave esiste come stringa (formato legacy) e `HSet` fallisce con `WRONGTYPE`, la eliminiamo e riproviamo come hash. Questo significa che uno sviluppatore che ha aggiornato a metà di un tick non deve fare manualmente `redis-cli del`.

`GetAll` usa `SCAN` invece di `KEYS`. `KEYS *` è O(n) e bloccante sul thread principale di Redis; `SCAN` è incrementale. Per uno snapshot store con ~16 chiavi è eccessivo, ma l'abitudine è quella giusta — `KEYS` in qualsiasi percorso di codice di produzione è un modo per spararsi sui piedi.

```go
func (s *Store) readKey(ctx context.Context, key string) (json.RawMessage, error) {
    t, _ := s.rdb.Type(ctx, key).Result()
    switch t {
    case "string":
        v, err := s.rdb.Get(ctx, key).Bytes()
        return json.RawMessage(v), err
    case "hash":
        fields, _ := s.rdb.HGetAll(ctx, key).Result()
        m := make(map[string]json.RawMessage, len(fields))
        for k, v := range fields {
            m[k] = json.RawMessage(v)
        }
        return json.Marshal(m)
    }
    return nil, fmt.Errorf("unsupported redis type %q for key %s", t, key)
}
```

Deserializzazione consapevole del tipo. L'handler HTTP dello snapshot è stupido — `core.OK(w, all)` e un header no-cache — perché tutto il lavoro avviene nello store.

## Il motore degli alert

`internal/alerts/engine.go` è la parte più complessa del sistema, perché deve compilare i predicati, aggiornarli e inviare con cooldown e consegna parallela.

### Compilazione e refresh

```go
func (e *Engine) reload(ctx context.Context) error {
    all, _ := e.repo.ListAll(ctx)
    indexed := make(map[string][]compiledRule, len(all))
    for _, r := range all {
        prog, err := e.compile(r.Predicate)
        if err != nil {
            e.logger.Warn("alerts: skipping rule with bad predicate",
                "rule_id", r.ID, "topic", r.Topic, "err", err)
            continue
        }
        indexed[r.Topic] = append(indexed[r.Topic], compiledRule{rule: r, program: prog})
    }
    e.rulesByTopic.Store(&indexed)
    return nil
}
```

`atomic.Pointer[map[string][]compiledRule]` fornisce un percorso di lettura lock-free. La map stessa non viene mai modificata dopo `Store`; il reload costruisce sempre una map nuova e scambia il puntatore. I lettori prendono il puntatore una volta e operano su una map stabile.

Gli errori di compilazione non fanno fallire l'intero reload — un singolo predicato di regola errato viene loggato e saltato. Questo conta: la UI delle regole permette agli utenti di scrivere predicati, e un utente che digita `event.severity = "high"` (assegnazione invece di confronto) non dovrebbe rompere silenziosamente gli alert per tutti.

### Valutazione

```go
func (e *Engine) Evaluate(ctx context.Context, ev events.Event) {
    idx := e.rulesByTopic.Load()
    if idx == nil { return }
    rules, ok := (*idx)[string(ev.Topic)]
    if !ok || len(rules) == 0 { return }

    payload, err := normalizePayload(ev.Payload)
    if err != nil {
        e.logger.Warn("alerts: payload normalize failed", ...)
        return
    }

    for _, cr := range rules {
        if !cr.rule.Enabled { continue }
        match, err := evalPredicate(cr.program, payload)
        if err != nil { ... continue }
        if !match { continue }
        e.fire(ctx, cr.rule, ev, payload)
    }
}
```

`normalizePayload` gestisce la disciplina del tipo `any`. Il bus trasporta `Payload any`; CEL ha bisogno di una `map[string]any` per accedere ai campi. La funzione prova il percorso economico (è già una map), poi `json.RawMessage` (un solo unmarshal), poi un round-trip completo tramite `json.Marshal`+`Unmarshal`. La maggior parte degli eventi arriva come `json.RawMessage`, perché è ciò che producono i collector.

### Invio

```go
func (e *Engine) fire(ctx context.Context, rule Rule, ev events.Event, payload map[string]any) {
    channels, _ := e.loader.LoadChannels(ctx, rule.UserID)
    cooldown := time.Duration(rule.CooldownSec) * time.Second
    message := formatMessage(rule, ev, payload)

    var wg sync.WaitGroup
    var mu sync.Mutex
    delivered := []string{}
    deliveryErrs := map[string]string{}

    for _, ch := range channels {
        ch := ch
        key := fmt.Sprintf("alert_cooldown:%s:%s:%s", rule.ID, ch.Type, ch.ID)
        ok, _ := e.cooldowns.TryAcquire(ctx, key, cooldown)
        if !ok { continue }
        wg.Add(1)
        go func() {
            defer wg.Done()
            err := e.notifier.SendAlert(ctx, ch, message)
            mu.Lock(); defer mu.Unlock()
            if err != nil {
                deliveryErrs[ch.ID] = err.Error()
                return
            }
            delivered = append(delivered, ch.ID)
        }()
    }
    wg.Wait()

    body, _ := json.Marshal(payload)
    errBody, _ := json.Marshal(deliveryErrs)
    _ = e.repo.RecordFire(ctx, HistoryRow{
        RuleID: rule.ID, UserID: rule.UserID,
        FiredAt: time.Now().UTC(),
        Payload: body,
        DeliveredTo: delivered,
        DeliveryErrors: errBody,
    })
}
```

Tre cose che vale la pena evidenziare:

- **Cooldown per canale tramite `SET NX EX` di Redis**. `RedisCooldown.TryAcquire` è un singolo round-trip e il TTL garantisce che la chiave si ripulisca da sola.
- **Consegna parallela**. Telegram è lento (2-5s per un invio senza bot soggetto a rate limiting), Slack è veloce (~200ms). Una consegna sequenziale allineerebbe tutto al più lento. Inviamo a tutti i canali in modo concorrente, poi `wg.Wait` attende che tutti ritornino.
- **Lo storico degli invii viene registrato una volta per ogni attivazione della regola, non per canale**. `delivered` elenca quali canali hanno avuto successo; `deliveryErrs` mappa l'ID del canale al testo dell'errore. Questo rende la pagina "storico degli alert" della UI utile per il debug dei guasti delle integrazioni.

## Cifratura dei webhook

```go
// internal/notifications/crypto.go
func (e *Encryptor) Encrypt(plaintext []byte) (ciphertext, nonce []byte, err error) {
    block, _ := aes.NewCipher(e.key)
    gcm, _ := cipher.NewGCM(block)
    nonce = make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil { ... }
    ciphertext = gcm.Seal(nil, nonce, plaintext, nil)
    return ciphertext, nonce, nil
}
```

AES-256-GCM della libreria standard. La chiave viene decodificata in base64 da `NOTIFICATION_ENCRYPTION_KEY` e validata all'avvio per essere esattamente di 32 byte:

```go
func NewEncryptor(b64Key string) (*Encryptor, error) {
    key, _ := base64.StdEncoding.DecodeString(b64Key)
    if len(key) != 32 {
        return nil, fmt.Errorf("encryption key must be 32 bytes, got %d", len(key))
    }
    return &Encryptor{key: key}, nil
}
```

Se `NewEncryptor` restituisce un errore, l'intero motore degli alert viene disabilitato per tutta la vita del processo — `bridge` e `notifBridge` restano nil, quindi i controlli `notifBridge != nil` in `cmd/api/main.go` saltano la configurazione del motore. Questo è un comportamento **fail-closed**: una configurazione errata disabilita la funzionalità invece di eseguirla con una cifratura rotta o assente.

## Collegamento del frontend

### Il driver WebSocket

`frontend/src/api/ws.ts`:

```ts
export function createDashboardWS(opts: CreateDashboardWSOpts): DashboardWS {
  const backoff = opts.backoff ?? DEFAULT_BACKOFF
  const onEvent = opts.onEvent ?? (() => undefined)

  let driver: WSDriver | null = null
  let ready = false
  let opened = false
  let closed = false
  let buffer: WSEvent[] = []
  let nextDelay = backoff.initialMs
  let reconnectTimer: ReturnType<typeof setTimeout> | null = null

  function attach() {
    if (closed) return
    const d = opts.driver()
    driver = d
    opened = false
    d.onOpen = () => {
      nextDelay = backoff.initialMs
      opened = true
      sendInitIfReady()
    }
    d.onMessage = (data) => {
      let parsed: WSEvent
      try { parsed = JSON.parse(data) as WSEvent } catch { return }
      if (ready) onEvent(parsed)
      else buffer.push(parsed)
    }
    d.onClose = () => {
      driver = null
      opened = false
      if (closed) return
      const delay = nextDelay
      nextDelay = Math.min(nextDelay * 2, backoff.maxMs)
      reconnectTimer = setTimeout(attach, delay)
    }
  }
  return {
    connect() { attach() },
    setReady() {
      ready = true
      sendInitIfReady()
      const flush = buffer
      buffer = []
      for (const ev of flush) onEvent(ev)
    },
    disconnect() { ... },
  }
}
```

La factory accetta un thunk `driver`, così i test possono iniettare un fake. `browserDriver` è l'implementazione di produzione che avvolge il `WebSocket` della piattaforma. Lo svuotamento del buffer in `setReady` è l'handshake snapshot+stream lato client.

La riconnessione usa un backoff esponenziale con `nextDelay * 2` limitato a `maxMs` (default 30s). A un'apertura pulita, `nextDelay` si azzera, così una riconnessione dopo un lungo uptime riparte da 1s.

### Instradamento verso gli store

La sequenza di avvio di `App` (parafrasata):

1. Monta i provider (client di TanStack Query, store Zustand già globali).
2. Apri il WebSocket tramite `createDashboardWS`. Passagli un `onEvent` che smista in base a `ev.ch` verso lo store giusto.
3. Recupera `/v1/snapshot`. In caso di successo, idrata ogni store dalla risposta.
4. Chiama `ws.setReady()`. Gli eventi in buffer vengono scaricati tramite `onEvent` negli store ora idratati.

Gli store sono piatti per topic. Ad esempio, `stores/cve.ts` contiene l'elenco delle CVE recenti ed espone un setter (`upsertCVE`) che il dispatcher WS chiama. I componenti si iscrivono tramite `useCVEStore(state => state.list)`, così un tick di Coinbase non fa ri-renderizzare il pannello delle CVE.

### Il globo

`frontend/src/pages/globe/Globe.tsx` è il grande consumer. Si iscrive allo store `globeEvents`, che è un fan-in degli store specifici per topic: interruzioni, BGP hijack, terremoti, vittime ransomware, punti ASN del mass-scan, posizione della ISS. Il componente del globo ascolta, calcola le modifiche ai layer e le applica in un batch `requestAnimationFrame`.

La traccia orbitale della ISS guidata dal TLE è propagata lato client. A ogni animation frame, il propagatore SGP4 fa avanzare l'anomalia media del satellite ed emette una coppia (lat, lon) che il layer del globo disegna come polilinea. Il TLE stesso arriva dallo snapshot o da un evento `iss_position` con un campo `tle`; il pin della posizione (separato dalla traccia orbitale) arriva dal polling ogni 10 secondi di `wheretheiss.at`.

## Errori comuni

Un breve elenco di errori che abbiamo commesso e che potresti commettere anche tu.

**Dimenticare di registrare un topic in `IsValid`/`AllTopics`.** Aggiungere una costante di topic non basta. L'handler WebSocket valida `?topics=` rispetto a `IsValid`, e il fallback predefinito usa `AllTopics`. Se ne dimentichi uno, il tuo nuovo topic silenziosamente non raggiunge mai il browser.

**Mettere lavoro costoso in `Bus.Run`.** Il loop di drenaggio è a goroutine singola. Se aggiungi una scrittura sincrona sul DB prima del broadcast, limiti il tasso di broadcast dell'intero sistema al tasso di scrittura sul DB. Sposta il lavoro costoso in un channel di un subscriber o nell'implementazione del persister.

**Usare `c.CloseRead` su `coder/websocket`.** Sembra utile. Chiude la connessione alla ricezione di qualsiasi frame, inclusa l'operazione `init` del client che la dashboard usa. O dreni esplicitamente le letture o progetti un protocollo rigorosamente server-push con semantica di chiusura alla ricezione — scegline uno.

**Non validare la lunghezza della chiave di cifratura all'avvio.** AES-256 accetta silenziosamente una chiave da 16 byte come AES-128 se non controlli. Il costruttore `NewEncryptor` deve imporre `len(key) == 32`.

**Restituire `*MinuteBar` dall'aggregator e dimenticare il controllo su nil.** La firma di ritorno di `Push` `(*MinuteBar, MinuteBar)` mette la barra chiusa dietro un puntatore. Salvare `*closed` senza un controllo su nil è il bug più comune che abbiamo intercettato in code review nel package di Coinbase.

**B-tree sulle colonne temporali su larga scala.** Allettante perché è il default. Per decine di milioni di righe prevalentemente append, BRIN è decine di volte più piccolo e altrettanto veloce per le scansioni per intervallo. Se usi un B-tree su `occurred_at`, il tuo indice diventa più grande della tabella nel giro di pochi mesi.

**Indicizzare i cooldown degli alert globalmente invece che per canale.** Un utente con regole Telegram e Slack sullo stesso predicato vuole un cooldown indipendente su ciascuno. Usare come chiave solo `rule_id` fa sì che il canale che scatta per primo silenzi l'altro.

## Dove andare dopo

Prosegui con [04-CHALLENGES.md](04-CHALLENGES.it.md) per idee di estensione, lavoro su funzionalità più difficili e hardening del deployment.
