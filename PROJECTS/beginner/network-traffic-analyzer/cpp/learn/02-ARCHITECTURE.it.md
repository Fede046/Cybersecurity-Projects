# Architettura del sistema

## Panoramica ad alto livello

```
┌─────────────────────────────────────────────────────────────┐
│                     main.cpp                                │
│  • arg parsing        • thread coordination                 │
│  • offline vs live    • CSV/JSON export on exit             │
└───────────┬─────────────────────────────┬───────────────────┘
            │                             │
     ┌──────▼──────┐             ┌────────▼────────┐
     │ PcapCapture │             │   FTXUI screen  │
     │ (libpcap)   │             │   (main thread) │
     └──────┬──────┘             └────────▲────────┘
            │ got_packet()                │ PostEvent()
            │                    ┌────────┴────────┐
     ┌──────▼──────┐             │ application_    │
     │  IPv4/IPv6  │             │ thread          │
     │  (parser)   │             │ (UI update loop)│
     └──────┬──────┘             └────────▲────────┘
            │ Packet                      │ get_snapshot()
     ┌──────▼──────────────────────────┐  │
     │           Stats                 │──┘
     │  add_packet()  push()           │
     │  transport_map  application_map │
     │  ip_map  pairs  packets deque   │
     │  bandwidth_history              │
     │  StatsSnapshot (under mutex)    │
     └─────────────────────────────────┘
```

## Modello di threading

Ci sono tre contesti di esecuzione concorrenti:

**Thread di cattura** — avviato da `PcapCapture::start()` in `pcapCapture.cpp:80`. Esegue `pcap_loop()`, che chiama `callback()` → `got_packet()` per ogni pacchetto. Chiama `Stats::add_packet()` e `Stats::push()`. Non tocca mai la UI.

**Thread di aggiornamento della UI** (`application_thread`, `main.cpp:107`) — esegue un ciclo che:
1. Fa avanzare il timer del tempo trascorso
2. Controlla le condizioni di stop (limite di tempo, cattura terminata)
3. Chiama tutti i metodi `Stats::update_*()` per ricostruire le tabelle dello snapshot
4. Chiama `view.render(stats.get_snapshot(), ...)` per costruire un nuovo albero di elementi FTXUI
5. Memorizza l'elemento in `current_render` sotto `render_mtx`
6. Invia un evento `Custom` allo screen FTXUI per innescare un repaint

**Event loop di FTXUI** — gira sul thread principale tramite `screen.Loop(component)` in `main.cpp:139`. La lambda `Renderer` (`main.cpp:92`) legge `current_render` sotto `render_mtx` e lo restituisce. La lambda `CatchEvent` gestisce `q` ed `Escape` per impostare `ui_running = false` e chiamare `screen.Exit()`.

### Punti di sincronizzazione

| Risorsa condivisa | Protezione | Pattern di accesso |
|---|---|---|
| Mappe interne di `Stats` | `Stats::mtx` | Il thread di cattura scrive; il thread UI legge tramite i metodi update_* |
| `StatsSnapshot` dentro Stats | `Stats::mtx` | Entrambi i thread; lo snapshot viene aggiornato sul posto sotto lock |
| `current_render` | `render_mtx` | Il thread UI scrive; il renderer FTXUI legge |
| `capture_finished`, `ui_running` | `std::atomic<bool>` | Più thread leggono/scrivono |
| `timer` | `std::atomic<std::chrono::seconds>` | Il thread UI scrive; la lambda di rendering legge |

## Componenti

### PcapCapture (`include/capture/pcapCapture.hpp`, `src/capture/pcapCapture.cpp`)

Avvolge l'intero ciclo di vita di libpcap. Possiede l'handle pcap come `unique_ptr<pcap_t, decltype(&pcap_close)>`, così viene rilasciato alla distruzione indipendentemente da come l'oggetto termina.

Responsabilità principali:
- `initialize()` — scopre tutte le interfacce di rete tramite `pcap_findalldevs()`
- `datalink_type()` — rileva il tipo di header di link layer e imposta l'offset in byte + l'estrattore dell'EtherType
- `start()` — apre il dispositivo in modalità promiscua, compila e installa il filtro BPF, avvia il thread di cattura
- `start_offline()` — apre il file pcap, lo elabora in modo sincrono (nessun thread aggiuntivo)
- `got_packet()` — analizza ogni frame grezzo: estrae l'EtherType, costruisce `IPv4` o `IPv6`, costruisce il `Packet`, lo inoltra a `Stats`
- `~PcapCapture()` — chiama `stop()`: interrompe il loop pcap, fa il join del thread, libera il programma di filtro e l'elenco delle interfacce

La callback C-style di `pcap_loop` richiede una funzione statica. `callback()` (riga 132) usa il puntatore `user` (che contiene `this` convertito in `u_char*`) per inoltrare al metodo d'istanza `got_packet()`.

### IP_class / IPv4 / IPv6 (`include/packet/IP.hpp`, `src/packet/IP.cpp`)

Parser polimorfico degli header IP. `IP_class` è una base astratta che dichiara gestori di trasporto puramente virtuali (`handle_tcp()`, `handle_udp()`, ecc.). Sia `IPv4` sia `IPv6` ereditano da essa.

Il parsing è basato sul costruttore: sia `IPv4(const u_char *data)` sia `IPv6(const u_char *data)` accettano un puntatore all'header IP (già spostato oltre l'header di link layer) e completano tutto il parsing nel costruttore. Dopo la costruzione, l'oggetto espone solo accessor puri: `get_source()`, `get_dest()`, `get_src_port()`, `get_dest_port()`, `get_protocol()`, `get_payload_len()`, `get_payload_ptr()`.

Dispatch del trasporto in IPv4: `switch(ip_hdr->ip_p)` alla riga 28, con dispatch verso `handle_tcp/udp/icmp/icmpv6/igmp`.

Dispatch del trasporto in IPv6: un ciclo `while(true)` alla riga 94 che o gestisce un protocollo di trasporto (e ritorna) o avanza oltre un header di estensione noto e continua.

### Packet (`include/packet/packet.hpp`, `src/packet/packet.cpp`)

Un tipo valore che contiene tutto ciò che è stato estratto da un singolo frame:
- `ip_version` — `v4` o `v6` (enum `IPVersion`)
- `transport_protocol` — TCP/UDP/ICMP/ICMP6/IGMP/UNKNOWN (enum class `TransportProtocol`)
- `application_protocol` — HTTP/HTTPS/DNS/SSH/ecc. (enum class `ApplicationProtocol`)
- `src`, `dst` — indirizzi IP come stringhe
- `src_port`, `dst_port` — numeri di porta
- `total_len` — lunghezza completa del frame dall'header pcap
- `payload_len` — lunghezza del payload di trasporto
- `payload_ptr` — puntatore nel buffer di pcap, annullato dopo l'esecuzione di `get_application_protocol()`

Il costruttore calcola `application_protocol` tramite `get_application_protocol()` (packet.cpp:4) e poi annulla immediatamente `payload_ptr`. Questo impedisce ai chiamanti di dereferenziare un puntatore valido solo durante la callback.

`get_application_protocol()` usa prima l'ispezione del payload (verbi HTTP, byte dell'header del record TLS), poi ripiega sull'identificazione basata sulla porta.

### Stats (`include/stats/protocolStats.hpp`, `src/stats/protocolStats.cpp`)

Engine di statistiche thread-safe. Stato interno:
- `transport_map` — `unordered_map<TransportProtocol, protocolStats>`
- `application_map` — `unordered_map<ApplicationProtocol, protocolStats>`
- `ip_map` — `unordered_map<string, IPStats>` (contatori bidirezionali per IP)
- `pairs` — `map<pair<string,string>, protocolStats>` (per coppia src→dst)
- `packets` — `deque<Packet>` (anello limitato dei pacchetti recenti)
- `snapshot` — `StatsSnapshot` (righe di visualizzazione precostruite, aggiornate dai metodi `update_*`)
- `bandwidth_history` — `vector<BandwidthPoint>` (serie temporale)

`add_packet()` (riga 19) acquisisce un lock e aggiorna tutte le mappe grezze in un'unica sezione critica. I metodi `update_*()` acquisiscono il lock, ordinano/formattano i dati e ricostruiscono i vettori `snapshot.*_rows` corrispondenti. `get_snapshot()` restituisce una copia dello snapshot sotto il lock.

### View (`include/TUI/view.hpp`, `src/TUI/view.cpp`)

Compositore di layout FTXUI senza stato. `render()` (view.cpp:5) costruisce l'intero layout del terminale a partire da uno `StatsSnapshot`:

```
┌─────── header ──────────────────────────────────────┐
│ title | interface | filter  │  traffic summary       │
├─────────────────────────────────────────────────────┤
│ transport table │ app table │ pairs table            │  ← hbox, bordered
├──────────────────────────────────────────────────────┤
│ IP table (scrollable)  │  bandwidth graph            │  ← hbox
├─────────────────────────────────────────────────────┤
│ packets table (right panel, scrollable, width=100)   │
├─────────────────────────────────────────────────────┤
│ footer: timer + exit hint                            │
└─────────────────────────────────────────────────────┘
```

`render_bandwidth()` (riga 138) definisce una `GraphFunction` — una lambda che riceve le dimensioni in pixel del widget grafico e restituisce un `vector<int>` che mappa ogni pixel x a un'altezza y. Interpola tra gli ultimi 50 campioni di banda e scala in base a `max_bandwidth`.

Tutte le sezioni a tabella usano `ftxui::Table` con lo stile della riga di intestazione (bordo `DOUBLE` sulla riga 0, `LIGHT` sulle restanti).

### Filter (`include/cli/filter.hpp`, `src/cli/filter.cpp`)

Modulo a due funzioni:

`parse(str)` (filter.cpp:5) — divide una stringa `key:value` al primo `:`. Mappa i nomi di chiave noti (`protocol`, `port`, `src`, `dst`, `ip`) sull'enum `filter_type`. Lancia `std::invalid_argument` se non è presente alcun `:`.

`get_bpf_filter(filters)` (filter.cpp:27) — raggruppa più filtri per tipo in una `map<filter_type, vector<string>>`. Mappa i valori rivolti all'utente sulla sintassi BPF (es. `protocol:dns` → `port 53`, `ip:v4` → `ip`). Combina i filtri dello stesso tipo con `or`, quelli di tipo diverso con `and`. Restituisce la stringa dell'espressione BPF risultante.

### argsParser (`include/cli/argsParse.hpp`, `src/cli/argsParse.cpp`)

Wrapper sottile attorno a `Boost.Program_options`. Definisce tutte le opzioni CLI nel costruttore e memorizza i risultati del parsing in un `po::variables_map vm` pubblico. Opzioni:

| Flag | Default | Descrizione |
|------|---------|-------------|
| `-i`, `--interface` | `wlan0` | Interfaccia di rete |
| `-c`, `--count` | `0` (illimitato) | Limite sul numero di pacchetti |
| `--time`, `-t` | `INT_MAX` | Durata della cattura (secondi) |
| `-r`, `--offline` | — | Leggi da file pcap |
| `-f`, `--filter` | — | Espressioni di filtro (componibili, multiple) |
| `-n`, `--limit` | `43` | Numero massimo di voci visualizzate |
| `--csv` / `--json` | — | Percorsi di esportazione |

## Flusso dei dati

### Cattura dal vivo

```
User: just run -i eth0 -f protocol:tcp
  ↓
main.cpp:33-47   Parse args, build filter vector
main.cpp:49      get_bpf_filter() → "tcp" BPF string
main.cpp:54      capture.set_capabilities(interface, count, "tcp", limit, &stats)
main.cpp:73      capture.start()
  ↓
pcapCapture.cpp:52-64  pcap_lookupnet → pcap_open_live (promiscuous, SNAP_LEN=1518)
pcapCapture.cpp:64     datalink_type() → set offset + get_ether_type lambda
pcapCapture.cpp:68-75  pcap_compile + pcap_setfilter (BPF "tcp" installed in kernel)
pcapCapture.cpp:80-87  spawn thread → pcap_loop(callback)
  ↓
[Capture thread: per-packet]
pcapCapture.cpp:132-136  callback() → got_packet()
pcapCapture.cpp:158      get_ether_type(packet) → ETHERTYPE_IP or ETHERTYPE_IPV6
pcapCapture.cpp:162      IPv4 ip(packet + offset)  — constructor parses headers
  IP.cpp:18-50              extract src/dst, walk to TCP/UDP handler
  IP.cpp:52-61              handle_tcp: ports, payload_ptr, payload_len
pcapCapture.cpp:165-168  Packet packetView(...) — constructor runs get_application_protocol()
  packet.cpp:4-57            memcmp payload bytes, port-based fallback
                             payload_ptr = nullptr
pcapCapture.cpp:167      stats->add_packet(packetView) — lock, update all maps
pcapCapture.cpp:168      stats->push(packetView)       — lock, push to deque
  ↓
[UI update thread: every loop iteration]
main.cpp:117-121  update_transport_stats(), update_application_stats(),
                  update_ip_stats(10), update_pairs(), update_bandwidth()
main.cpp:124-126  view.render(stats.get_snapshot(), ...) → Element
main.cpp:127-130  store in current_render under render_mtx, PostEvent to FTXUI
  ↓
[FTXUI main thread: on Custom event]
main.cpp:92-95  Renderer lambda reads current_render under render_mtx → display
```

### Analisi offline

```
main.cpp:61-70   capture.start_offline(pcap_file)   — runs synchronously
  pcapCapture.cpp:199-211  pcap_open_offline → pcap_loop (no thread)
  [same per-packet flow as above]
main.cpp:64-69   stats.update_packets() + update_application_stats() + ...
  ↓
main.cpp:86-88   view.render(stats.get_snapshot(), ...) → initial current_render
screen.Loop()    FTXUI displays static result (no UI update thread spawned)
```

## Decisioni di design

### Decisione: parsing nel costruttore vs getter lazy

Il design originale di `IP_class` aveva `get_protocol()` come punto di ingresso per tutto il parsing — un getter con effetti collaterali. Questo crea dipendenze d'ordine nascoste: chiamare `get_src_port()` prima di `get_protocol()` restituisce 0 su IPv4 o spazzatura su IPv6.

Il codice unificato (visibile negli attuali costruttori di `IPv4`/`IPv6`) analizza tutto al momento della costruzione. I getter restituiscono valori già calcolati. Nessun requisito di ordinamento per i chiamanti.

### Decisione: StatsSnapshot come tipo valore

`StatsSnapshot` contiene i dati di visualizzazione preformattati come righe `vector<vector<string>>`. Il thread UI chiama `get_snapshot()`, che copia fuori questa struct sotto il mutex. Il renderer FTXUI poi lavora sulla propria copia senza bisogno di mantenere alcun lock.

L'alternativa — far sì che il renderer faccia direttamente il lock di Stats — significherebbe far interagire il mutex di rendering e il mutex delle statistiche, con il rischio di deadlock o di bloccare il thread di cattura sul lavoro della UI.

### Decisione: handle RAII per pcap_t

`pcap_t*` è una risorsa C con `pcap_close()` come distruttore. Memorizzarlo come `unique_ptr<pcap_t, decltype(&pcap_close)>` significa che viene rilasciato automaticamente quando `PcapCapture` viene distrutto, anche se vengono lanciate eccezioni. `handle.reset()` in `stop()` lo rilascia esplicitamente in anticipo quando la cattura termina.

### Decisione: offset + lambda per i tipi di link

Invece di catene di if sparse per `got_packet()`, il metodo `datalink_type()` imposta sia l'intero `offset` sia l'oggetto funzione `get_ether_type` una sola volta, all'apertura del dispositivo. `got_packet()` resta pulito: `uint16_t ether_type = get_ether_type(packet)`.

Questo rende l'aggiunta di un nuovo tipo di link un singolo `case` in più in `datalink_type()` anziché una modifica al percorso critico (hot path).
