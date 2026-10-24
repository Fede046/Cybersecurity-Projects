<!--
©AngelaMos | 2026
04-CHALLENGES.md
-->

# Challenge

Hai una dashboard funzionante. La domanda interessante è cosa aggiungerle. Questo file è un elenco graduato di estensioni, da "progetto da una serata" a "dovresti pubblicarci un post sul blog". Ogni voce include un suggerimento e un motivo reale per cui sarebbe importante.

## Facili

Queste richiedono qualche ora ciascuna. L'obiettivo è costruire la memoria muscolare per il collegamento collector + topic + store del frontend.

### Aggiungi un feed di advisory di un vendor aggiuntivo

Scegline uno tra: Microsoft Security Response Center (MSRC), Cisco PSIRT, Red Hat Security Advisories, Project Zero. Ognuno pubblica RSS o JSON.

**Suggerimento**: copia la struttura del collector `kev`. Ti servono un `Fetcher` che faccia HTTP+parsing, un `Repository` per l'upsert, una nuova costante di topic e uno store del frontend. Il feed CVRF di MSRC è una buona prima scelta perché fornisce JSON, non XML personalizzato.

**Perché è importante**: NVD è costantemente 1-7 giorni in ritardo rispetto alla divulgazione dei vendor. Un feed diretto di advisory dei vendor è il singolo miglioramento con la leva più alta che tu possa fare per una dashboard difensiva.

### Aggiungi un feed dei nodi di uscita Tor

`tornull/exit-list` (o `dan.me.uk/torlist`) pubblica l'elenco attuale dei nodi di uscita Tor. Collegalo come feed; sul frontend, colora gli exit Tor con un colore diverso quando compaiono nel pannello dei top IP di DShield.

**Suggerimento**: l'implementazione più semplice è un poller ogni 30 minuti verso un set Redis. L'enricher di DShield poi fa un `SISMEMBER` per ogni IP al momento dell'arricchimento e imprime `is_tor: true` sul payload.

**Perché è importante**: gli IP di uscita Tor che scansionano il tuo perimetro sono categoricamente diversi dai router domestici compromessi che scansionano il tuo perimetro. La differenziazione visiva è più veloce di un tooltip.

### Salva lo storico degli alert inviati con una pagina UI

Lo schema e `RecordFire` esistono già (vedi `internal/alerts/repository.go`). Ciò che manca è un endpoint `/v1/alerts/history` e una tabella nella pagina delle impostazioni.

**Suggerimento**: modella l'endpoint su `/v1/intel/cve/search` per la forma di paginazione, ordinamento e filtri. La route del frontend è già nell'albero della pagina delle impostazioni (`pages/settings/Alerts.tsx` o simile).

**Perché è importante**: quando un alert scatta e l'utente non l'ha visto, la prima domanda è "è davvero arrivato su Slack?" La pagina dello storico risponde senza un round trip sul database.

### Aggiungi un pulsante "alert di test"

Per ogni canale configurato, mostra un pulsante che fa una POST a `/v1/notifications/channels/:id/test`. L'endpoint invia un messaggio fisso `"This is a test alert from monitor-the-situation"` tramite il notifier del canale.

**Suggerimento**: si tratta di un endpoint con una sola route più una modifica al frontend con un solo pulsante. Il lavoro è il collegamento, non la logica. Riusa direttamente `notifSender.Send(...)`.

**Perché è importante**: la domanda di supporto più comune del tipo "gli alert non funzionano" in realtà è "ho configurato male il webhook". Un pulsante di test sposta il problema a monte.

## Medie

Questi sono progetti da weekend. Richiedono di toccare il bus, il layer di storage o entrambi.

### Aggiungi il tagging delle tecniche MITRE ATT&CK alle vittime ransomware

I post di ransomware.live includono il nome del threat actor. Mappa i nomi degli attori su set noti di tecniche ATT&CK (la CISA pubblica delle mappature; così come l'ATT&CK Navigator di MITRE). Memorizza gli ID delle tecniche sulla riga del ransomware e mostrali come piccoli badge sul pannello.

**Suggerimento**: la mappatura in sé è un dato, non codice. Costruisci un `actor_tactics.yaml` e caricalo all'avvio. Il collector arricchisce il payload prima del salvataggio e dell'emissione. Bonus: quando compare un attore non ancora mappato, logga un warning così ti ricordi di estendere la mappatura.

**Perché è importante**: sapere che REvil usa T1486 (Data Encrypted for Impact) e T1078 (Valid Accounts) dice a un difensore cosa controllare quando REvil compare in the wild. I nomi non sono azionabili; le tecniche sì.

### Aggiungi un tracker ransomware Bitcoin "segui i soldi"

Quando il post di una vittima ransomware include un indirizzo di wallet (alcuni post di leak lo fanno), cercalo su fonti pubbliche in stile Chainalysis o usa l'API di blockchair per lo storico delle transazioni. Mostra i trasferimenti in entrata recenti del wallet come una sparkline.

**Suggerimento**: questo è l'unico feed del progetto che ha un fan-out illimitato (un wallet → molte chiamate allo storico delle tx), quindi mettilo dietro una cache per wallet con un TTL lungo. Non rifare il fetch a ogni rendering della pagina.

**Perché è importante**: i flussi dei pagamenti ransomware sono informazioni pubbliche sulla blockchain. Mostrarli accanto al post della vittima è unico rispetto alla tesi "fondere i segnali" della dashboard.

### Sostituisci il bus in-process con NATS JetStream

Mantieni la stessa interfaccia `Bus` (`Emit`, `Subscribe`, `Run`). Sostituisci l'implementazione da un channel Go a NATS JetStream. Ora la dashboard può eseguire più istanze dietro un load balancer.

**Suggerimento**: i subject per stream di JetStream si mappano in modo pulito sui topic. Usa un consumer durevole per ogni istanza del backend per il dispatcher degli alert, e una sottoscrizione pull-based per il broadcast dell'hub WebSocket. Mantieni il salvataggio dello snapshot in Redis, non in JetStream — Redis resta la cache lato lettura.

**Perché è importante**: è il passo successivo naturale quando un operatore diventa un piccolo team. Il capitolo sull'architettura prometteva che sarebbe stata la sostituzione di un solo componente; la challenge è verificarlo davvero.

### Aggiungi trace OpenTelemetry attraverso il bus

`core.NewTelemetry` inizializza già il tracer OTel. Fai passare il trace context attraverso `Bus.Emit`, così un singolo evento CVE può essere tracciato da fetch NVD → upsert → bus → snapshot → broadcast ws → ack del frontend.

**Suggerimento**: aggiungi un parametro `context.Context` a `Emit` (oggi non ne riceve uno). Inietta lo span context nel payload dell'evento come header `traceparent`. Il persister e il broadcaster lo raccolgono. Il frontend logga un `console.debug` con il trace ID così puoi correlare in Tempo o Honeycomb.

**Perché è importante**: quando un pannello è "stale" e un altro è fresco, il distributed tracing ti dice quale collector è lento. Senza trace, leggi i log e tiri a indovinare.

### Migra dal polling allo stream di eventi CVE 2.0 di NVD quando arriverà

NVD promette un feed event-driven da anni. Quando verrà rilasciato (o se lo sarà già quando leggerai questo), sostituire il polling delle CVE ogni 2 ore con una sottoscrizione in streaming riduce la latenza della dashboard sulle nuove CVE da "fino a due ore" a "secondi". La struttura del collector è già pronta per questo — `coinbase` è l'implementazione di riferimento in streaming.

**Suggerimento**: prendi come modello `internal/collectors/coinbase/`. La parte più difficile è il recupero dei buchi alla riconnessione, che `sequencer.go` mostra.

**Perché è importante**: se costruisci dashboard per un SOC, la latenza è una feature. Il team che usa il tuo strumento se ne accorgerà.

## Difficili

Questi sono progetti da post sul blog. Ognuno è un investimento di più settimane.

### Aggiungi un feed malware basato su YARA tramite Malshare o VirusTotal Hunting

Malshare pubblica un feed giornaliero di nuovi hash di malware. VirusTotal Hunting ha match YARA live. Collega uno dei due, salva i match con la regola YARA che ha fatto match e mostra "questa regola è scattata N volte nelle ultime 24h" su un nuovo pannello.

**Suggerimento**: VT Hunting richiede una chiave API a pagamento. Inizia con Malshare. Il pannello del frontend può essere una sparkline per regola più un click-through verso l'ultimo hash che ha fatto match.

**Perché è importante**: collegare un hit YARA a un diff delle CVE, a una voce della KEV e a un leak ransomware è tutta la proposta della dashboard. Questo aggiunge il segnale lato file a quelli lato rete e lato patch già presenti.

### Costruisci un replay dei diff "cosa è appena cambiato?"

Al momento ogni pannello mostra lo stato attuale. Aggiungi uno slider "macchina del tempo": trascinalo indietro, e la dashboard ricostruisce ogni pannello com'era in quel momento.

**Suggerimento**: è ciò per cui è stato costruito BRIN. Ogni store di pannello ha bisogno di una query `at(t time.Time)` sulle tabelle storiche di Postgres. Il frontend deve disconnettere il WebSocket, fare il rendering dalla query storica e riconnettersi quando l'utente rilascia lo slider. Fai attenzione al pannello di Coinbase — le barre al minuto sono rade rispetto ai tick; devi scegliere quali mostrare.

**Perché è importante**: nella revisione post-incidente, "cosa mostrava la dashboard 30 minuti prima della pagina dell'incidente" è la domanda più spaventosa da lasciare senza risposta.

### Aggiungi il rilevamento degli hijack MOAS da un collector BGP pubblico (non Cloudflare)

Il rilevamento degli hijack di CF Radar è closed-source. RIS Live di RIPE (`ris-live.ripe.net`) è un feed BGP pubblico e gratuito con volumi da firehose. Implementa il tuo detector MOAS a partire dallo stream grezzo: mantieni una finestra scorrevole di osservazioni `(prefix, origin_as)`, genera un alert quando un prefisso ha contemporaneamente due ASN di origine distinti.

**Suggerimento**: è almeno una settimana di lavoro. Inizia scrivendo un `client.go` che si iscriva al WebSocket di RIS Live. Tieni le osservazioni in una LRU limitata. Regola la confidenza in base alle relazioni tra AS (gli ASN fratelli non sono hijack, i peer potrebbero esserlo).

**Perché è importante**: costruire un detector di hijack dai dati BGP grezzi è il progetto canonico del tipo "ora capisci il routing di internet". CF Radar è comodo; RIS Live è educativo.

### Trasforma la dashboard in un SaaS multi-tenant

Al momento c'è un solo utente (l'operatore). Trasformala in un SaaS ospitato in cui ogni cliente ottiene regole di alert, configurazioni dei canali e budget di rate limiting isolati. Stessi feed di dati (il mondo è il mondo), stato degli alert per tenant.

**Suggerimento**: la tabella `users` e il package `auth` sono già quasi a posto. La parte difficile è il partizionamento dei dati: regole di alert e canali sono per tenant; i dati di CVE/KEV/ransomware sono condivisi. Il frontend deve limitare le pagine di amministrazione in base all'appartenenza all'organizzazione. La fatturazione con Stripe è un discorso a parte. Prevedi almeno un mese.

**Perché è importante**: questo è il percorso da "progetto personale" a "prodotto vero". Farlo bene insegna pattern di multi-tenancy che si applicano a ogni SaaS B2B.

### Aggiungi una visualizzazione di flusso WebGL sopra il globo

Oggi il globo mostra eventi discreti: punti, archi di hijack, pin della ISS. Aggiungi un sistema di particelle WebGL che scorra continuamente lungo gli archi BGP, con intensità proporzionale alla confidenza dell'hijack rilevato. Poi, quando un'interruzione è attiva, mostra il paese in rosso pulsante con frequenza proporzionale al numero di ASN coinvolti.

**Suggerimento**: l'API dei custom layer di MapLibre supporta passaggi WebGL grezzi. Guarda TripsLayer e ArcLayer di deck.gl come riferimento. Il numero di particelle diventa costoso in fretta — limita a ~5000 particelle attive per frame.

**Perché è importante**: è ciò che fa diventare virale uno screenshot della dashboard. L'estetica conta quando il pubblico di uno strumento sono operatori che lo fissano per 8 ore al giorno.

## Hardening del deployment

I deployment di produzione reali ne hanno bisogno. L'attuale Justfile ti porta all'80%.

### Esegui il backend anche dietro un proxy non Cloudflare

`cloudflared.compose.yml` copre Cloudflare Tunnel. Aggiungi un `compose.traefik.yml` e un `compose.nginx-direct.yml` alternativi per un edge self-hosted. Le configurazioni differiscono per terminazione TLS, inoltro degli header e regole di upgrade WebSocket.

**Suggerimento**: la configurazione `TrustedProxyHops` è la manopola chiave. Cloudflare aggiunge gli header noti CF-Connecting-IP; Traefik aggiunge `X-Forwarded-For`. Imposta correttamente il numero di hop fidati per ogni setup, altrimenti i tuoi rate limit e i limiti WS per IP si applicano all'IP del proxy, non all'IP del client.

### Aggiungi una replica di lettura Postgres per `/v1/intel/*`

Gli endpoint di query storiche sono pesanti in lettura e tollerano secondi di ritardo. Punta un pool di lettura `pgx` a una replica in streaming; instrada le letture di `intel.Handler` verso di essa.

**Suggerimento**: è un nuovo pool in `core` e una manopola di configurazione. I collector continuano a scrivere sul primario. Solo le letture di `intel/handler.go` vanno ripuntate.

### Aggiungi metriche Prometheus per il bus e l'hub

`Bus.DroppedCount`, `Bus.SubscriberDroppedCount`, `Hub.SubscriberCount` sono già esposti nel codice ma solo loggati. Collegali a un `prometheus.GaugeFunc`. Aggiungi istogrammi di latenza sui tick dei collector tramite `core.Telemetry`.

**Suggerimento**: Otel esporta già verso Prometheus se imposti l'exporter su `prometheus`. Il lavoro sta nel dare nomi coerenti alle metriche e scrivere le regole di alerting (ad es. page quando il rate di `bus_dropped_total` > 0 per 5m).

### Dependabot, Renovate e pinning della supply chain

Il file del modulo Go fa già il pinning tramite checksum. Aggiungi `.github/dependabot.yml` per Go e pnpm. Blocca le immagini base docker per digest, non per tag. Fai fallire la CI quando `go mod download -x` segnala un qualsiasi modulo non verificato.

**Suggerimento**: il progetto `bug-bounty-platform` in questo repo ha un setup di hardening della CI che vale la pena copiare.

### Genera SBOM e attesta la provenienza

`syft` produce SBOM. `slsa-github-generator` produce la provenienza SLSA. Aggiungi un workflow di release che emetta entrambi, firmi il binario con `cosign` e faccia il push delle immagini firmate su GHCR con attestazione.

**Suggerimento**: lo SBOM è una singola invocazione di `syft packages dir:.`. La provenienza è un workflow riutilizzabile di GitHub Actions. La parte difficile è inserire `cosign verify` nella pipeline di deploy, così le immagini non firmate non possono essere deployate.

## Collegamenti con altri progetti di questo repo

Alcune idee di estensione che attingono codice dai progetti fratelli:

- **Honeypot-network** pubblica gli IP degli attaccanti. Inseriscili come 13° feed e colora le righe del pannello DShield che si sovrappongono.
- **API rate limiter** — il suo store a finestra scorrevole sarebbe un miglioramento plug-in per il limite di connessioni WebSocket per IP.
- **Bug bounty platform** — quando una CVE sulla dashboard corrisponde a un programma in scope, mostra il link al programma sulla riga della CVE.
- **AI threat detection** — collega il suo punteggio di anomalia sugli IP degli attaccanti all'enricher di DShield, così gli IP con anomalia alta vengono mostrati con un glifo diverso.

## Nota finale

La dashboard non è "finita". È una base con opinioni precise. Ogni estensione di questo elenco è stata abbozzata, e alcune prototipate, nel corso della vita del progetto. Nessuna di esse richiede di smontare l'architettura, perché l'architettura è stata scelta proprio per mantenere pulita la separazione producer-bus-consumer.

Se costruisci qualcosa di interessante sopra di essa, invia una PR o apri una issue. Soprattutto per la categoria difficile — l'obiettivo è che questa codebase sia l'implementazione di riferimento più pulita di "dashboard di security operations in tempo reale" che qualcuno abbia mai pubblicato.
