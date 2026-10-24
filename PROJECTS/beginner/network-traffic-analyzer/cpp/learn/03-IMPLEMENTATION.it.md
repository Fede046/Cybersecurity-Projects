# Guida all'implementazione

Questo documento percorre ogni componente principale con riferimenti esatti a file e righe. Leggi il sorgente in parallelo a questo.

## Entry point — `main.cpp`

### Inizializzazione (righe 14–18)

```cpp
Stats stats;
PcapCapture capture;
capture.initialize();
argsParser parser(argc, argv);
```

`Stats` viene costruito di default con tutte le mappe vuote e `last_tick` impostato all'istante corrente. `PcapCapture` inizializza il suo `handle` con `nullptr`. `initialize()` chiama `pcap_findalldevs()` per popolare la lista collegata `interfaces`, così che `--interfaces` possa stamparle.

### Gestione degli argomenti (righe 24–47)

```cpp
if (parser.vm.contains("help")) { parser.print_help(); return 0; }
if (parser.vm.contains("interfaces")) { capture.print_interfaces(); return 0; }
```

`variables_map::contains()` di Boost controlla se il flag è stato passato. Uscita anticipata prima di qualsiasi configurazione di rete.

```cpp
std::vector<filter> filters;
if (parser.vm.contains("filter")) {
    auto &f = parser.vm["filter"].as<std::vector<std::string>>();
    for (auto &x : f) { filters.push_back(parse(x)); filterString += x + " "; }
}
std::string expression = get_bpf_filter(filters);
```

`--filter` è un'opzione componibile — può comparire più volte e Boost accumula tutti i valori in un `vector<string>`. Ogni stringa `key:value` viene analizzata in una struct `filter`, poi `get_bpf_filter()` le combina in un'unica espressione BPF.

### Percorso offline vs dal vivo (righe 51–74)

```cpp
bool isOffline = parser.vm.contains("offline");
capture.set_capabilities(interface, count, expression, limit, &stats);

if (isOffline) {
    capture.start_offline(parser.vm["offline"].as<std::string>());
    stats.update_packets();
    stats.update_application_stats();
    // ...all update methods
}
else {
    capture.start();
}
```

`set_capabilities()` memorizza la configurazione dentro `PcapCapture` e chiama `stats->set_packets_limit(packets_limit)`. In modalità offline, `start_offline()` gira in modo sincrono e blocca finché l'intero file non è stato elaborato — nessun threading. Tutte le statistiche vengono poi calcolate una sola volta prima dell'avvio della TUI. In modalità dal vivo, `start()` avvia il thread di cattura e ritorna immediatamente.

### Setup di FTXUI (righe 78–104)

```cpp
auto screen = ftxui::ScreenInteractive::Fullscreen();
View view;
std::mutex render_mtx;
ftxui::Element current_render = isOffline
    ? view.render(stats.get_snapshot(), ...)
    : ftxui::text("Starting capture...");

auto component = ftxui::Renderer([&] {
    std::lock_guard<std::mutex> lock(render_mtx);
    return current_render;
});

component |= ftxui::CatchEvent([&](ftxui::Event e) {
    if (e == ftxui::Event::Character('q') || e == ftxui::Event::Escape) {
        ui_running = false;
        screen.Exit();
        return true;
    }
    return true;
});
```

`current_render` contiene l'ultimo elemento costruito. La lambda `Renderer` viene chiamata dall'event loop di FTXUI a ogni repaint — restituisce semplicemente ciò che c'è in `current_render`, protetto da `render_mtx`. La lambda `CatchEvent` intercetta gli eventi da tastiera. Restituire `true` significa "evento consumato, non propagarlo".

Nota: `return true` per tutti gli eventi (non solo q/Esc) è intenzionale — impedisce al comportamento predefinito di FTXUI di elaborare altri tasti.

### Thread di aggiornamento della UI (righe 106–137)

```cpp
application_thread = std::thread([&] {
    while (!capture_finished && ui_running) {
        auto now = std::chrono::steady_clock::now();
        timer.store(std::chrono::duration_cast<std::chrono::seconds>(now - begin));

        if (timer.load() >= std::chrono::seconds(time) || !capture.isRunning())
            capture_finished = true;

        stats.update_packets();
        stats.update_application_stats();
        stats.update_transport_stats();
        stats.update_ip_stats(10);
        stats.update_pairs();
        stats.update_bandwidth();

        ftxui::Element new_frame = view.render(stats.get_snapshot(), ...);
        { std::lock_guard<std::mutex> lock(render_mtx); current_render = new_frame; }
        if (ui_running) screen.PostEvent(ftxui::Event::Custom);
    }
});
```

Il ciclo aggiorna le statistiche, costruisce un nuovo albero di elementi FTXUI dallo snapshot, lo scambia in `current_render` (sotto lock) e dice all'event loop di FTXUI di fare il repaint. `PostEvent(Custom)` non è bloccante — accoda l'evento e ritorna.

Nel ciclo non c'è un `sleep_for` esplicito (è commentato alla riga 134). Il ciclo gira tanto velocemente quanto si completano i metodi di aggiornamento, il che è limitato dalla contesa sul mutex delle statistiche.

---

## PcapCapture — `src/capture/pcapCapture.cpp`

### `start()` (righe 50–88)

```cpp
handle.reset(pcap_open_live(interface.c_str(), SNAP_LEN, 1, 1000, errbuf));
```

- `SNAP_LEN = 1518` — numero massimo di byte da catturare per pacchetto (MTU Ethernet standard + header)
- `1` — modalità promiscua attiva
- `1000` — timeout di lettura in millisecondi (quanto a lungo pcap_loop resta bloccato in attesa di pacchetti)

```cpp
datalink_type(pcap_datalink(handle.get()));
```

Rileva il tipo di link e imposta `offset` + `get_ether_type` prima che arrivi qualsiasi pacchetto.

```cpp
if (pcap_compile(handle.get(), &fp, filter_exp.c_str(), 0, net) == -1)
    throw std::runtime_error(...);
if (pcap_setfilter(handle.get(), &fp) == -1)
    throw std::runtime_error(...);
```

La compilazione BPF avviene in `struct bpf_program fp` (allocata sullo stack). Dopo l'installazione, il kernel esegue questo programma BPF su ogni frame in arrivo.

```cpp
thread = std::thread([this]() {
    if (pcap_loop(handle.get(), num_packets, &PcapCapture::callback, reinterpret_cast<u_char *>(this)) < 0) {}
    running = false;
});
```

`pcap_loop` blocca il thread finché non sono stati catturati `num_packets` pacchetti (0 = illimitato) o finché non viene chiamato `pcap_breakloop()`. `this` viene passato come puntatore `user` — la funzione statica `callback` lo riconverte in `PcapCapture*` per chiamare `got_packet()`.

### `stop()` (righe 90–108)

```cpp
pcap_freecode(&fp);          // release BPF program memory
if (!handle) return;
running = false;
pcap_breakloop(handle.get()); // signal pcap_loop to exit on next packet
if (thread.joinable()) thread.join(); // wait for capture thread to finish
handle.reset();               // calls pcap_close via unique_ptr deleter
if (interfaces) { pcap_freealldevs(interfaces); interfaces = nullptr; }
```

Chiamato dal distruttore. L'ordine conta: prima interrompi il loop, poi fai il join, poi chiudi l'handle.

### `got_packet()` (righe 152–179)

```cpp
uint16_t ether_type = get_ether_type(packet);

if (ether_type == ETHERTYPE_IP) {
    IPv4 ip(packet + offset);
    TransportProtocol prot = ip.get_protocol();
    Packet packetView(v4, prot, ip.get_source(), ip.get_dest(),
                      ip.get_src_port(), ip.get_dest_port(),
                      header->len, ip.get_payload_len(), ip.get_payload_ptr());
    stats->add_packet(packetView);
    stats->push(packetView);
}
```

`packet + offset` salta oltre l'header di link layer (14, 16 o 20 byte a seconda del tipo DLT). `IPv4 ip(packet + offset)` analizza tutti gli header nel costruttore. La costruzione di `Packet` chiama `get_application_protocol()` e annulla `payload_ptr`. Poi vengono chiamati sia `add_packet()` (aggiorna tutte le mappe) sia `push()` (inserisce nella deque dei pacchetti recenti) — entrambi acquisiscono internamente il mutex delle statistiche.

---

## Parsing IP — `src/packet/IP.cpp`

### Costruttore IPv4 (righe 18–50)

```cpp
ip_hdr = reinterpret_cast<const ip *>(data);
src = inet_ntoa(ip_hdr->ip_src);    // converts in_addr to "a.b.c.d" string
dst = inet_ntoa(ip_hdr->ip_dst);
ip_hdr_len = ip_hdr->ip_hl * 4;    // ip_hl is 4-bit field: header length in 32-bit words
if (ip_hdr_len < 20) throw std::runtime_error("Failed to initial IPv4 ");
```

`ip_hl * 4`: il campo Internet Header Length è di 4 bit, misurato in parole da 32 bit. Il valore minimo è 5 (= 20 byte, senza opzioni). Si converte in byte moltiplicando per 4.

```cpp
switch (ip_hdr->ip_p) {
case IPPROTO_TCP:  IPv4::handle_tcp();  break;
case IPPROTO_UDP:  IPv4::handle_udp();  break;
case IPPROTO_ICMP: IPv4::handle_icmp(); break;
// ...
}
```

`ip_p` è il campo Protocol (byte 9 dell'header IP). La IANA li assegna: 6 = TCP, 17 = UDP, 1 = ICMP.

### Gestore TCP di IPv4 (righe 52–62)

```cpp
const auto *tcp = reinterpret_cast<const tcphdr *>(
    reinterpret_cast<const u_char *>(ip_hdr) + ip_hdr_len
);
src_port = ntohs(tcp->source);
dest_port = ntohs(tcp->dest);
payload_ptr = reinterpret_cast<const u_char *>(tcp) + tcp->doff * 4;
payload_len = ntohs(ip_hdr->ip_len) - (ip_hdr_len + tcp->doff * 4);
```

`ip_hdr` punta all'header IP. Aggiungere `ip_hdr_len` byte (dopo aver prima fatto il cast a `u_char*` per l'aritmetica sui byte) porta all'header TCP. `tcp->doff` è il Data Offset TCP: il numero di parole da 32 bit nell'header TCP. `tcp->doff * 4` dà i byte. Il payload inizia dopo l'header TCP.

`ntohs()` converte dal network byte order (big-endian) all'host byte order. Tutti i campi multi-byte nei protocolli di rete sono big-endian.

### Percorrere gli header di estensione IPv6 (righe 82–136)

```cpp
ptr = reinterpret_cast<const uint8_t *>(ip_hdr + 1); // past the 40-byte fixed header
while (true) {
    switch (hdr) {
    case IPPROTO_TCP: IPv6::handle_tcp(); return;
    // ...
    case IPPROTO_HOPOPTS:
    case IPPROTO_ROUTING:
    case IPPROTO_DSTOPTS: {
        const auto *ext = reinterpret_cast<const ip6_ext *>(ptr);
        hdr = ext->ip6e_nxt;
        ptr += (ext->ip6e_len + 1) * 8;
        break;
    }
    case IPPROTO_FRAGMENT: {
        const auto *frag = reinterpret_cast<const ip6_frag *>(ptr);
        hdr = frag->ip6f_nxt;
        ptr += sizeof(ip6_frag);
        break;
    }
    default: protocol = TransportProtocol::UNKNOWN; return;
    }
}
```

`ip_hdr + 1` — l'aritmetica dei puntatori su `ip6_hdr*` avanza di `sizeof(ip6_hdr) = 40` byte, arrivando esattamente al primo header di estensione o all'header di trasporto.

`(ext->ip6e_len + 1) * 8` — formula della RFC 2460. `ip6e_len` è la lunghezza in unità da 8 byte senza contare i primi 8 byte. Quindi i byte totali = `(len + 1) * 8`.

---

## Rilevamento del protocollo applicativo — `src/packet/packet.cpp`

### Identificazione in due fasi (righe 4–57)

```cpp
ApplicationProtocol Packet::get_application_protocol() {
    if (!payload_ptr || payload_len < 4) goto check_port;

    if (transport_protocol == TransportProtocol::TCP) {
        if (!memcmp(payload_ptr, "GET ", 4) || !memcmp(payload_ptr, "POST", 4) || ...)
            return ApplicationProtocol::HTTP;
    }
    if ((src_port == 53 || dst_port == 53) && payload_len >= 12)
        return ApplicationProtocol::DNS;
    if (transport_protocol == TransportProtocol::TCP && payload_len >= 3) {
        if (payload_ptr[0] == 0x16 && payload_ptr[1] == 0x03)
            return ApplicationProtocol::HTTPS;
    }

check_port:
    uint16_t port = (src_port < dst_port) ? src_port : dst_port;
    // switch on port ...
}
```

Fase 1 — ispezione del payload. `memcmp(payload_ptr, "GET ", 4)` confronta i primi 4 byte del payload con la stringa letterale. Le richieste HTTP/1.x iniziano sempre con un verbo. I record TLS iniziano con `0x16 0x03` (Content-Type=Handshake, Version=3.x).

Fase 2 — fallback sulla porta tramite `goto check_port`. Si usa `goto` per saltare oltre la Fase 1 quando il payload è nullo o troppo corto. `port = min(src_port, dst_port)` — per le connessioni client→server, la porta del server è tipicamente quella well-known e sarà numericamente più piccola.

---

## Engine delle statistiche — `src/stats/protocolStats.cpp`

### `add_packet()` (righe 19–42)

```cpp
void Stats::add_packet(const Packet &packet) {
    std::lock_guard<std::mutex> lock(mtx);

    ++snapshot.total_p;
    snapshot.total_b += packet.total_len;

    auto &t = transport_map[packet.transport_protocol];
    t.packets++;
    t.bytes += packet.total_len;

    auto &a = application_map[packet.application_protocol];
    a.packets++;
    a.bytes += packet.payload_len;

    ip_map[packet.src].packets_sent++;
    ip_map[packet.src].bytes_sent += packet.total_len;
    ip_map[packet.dst].packets_received++;
    ip_map[packet.dst].bytes_received += packet.total_len;

    auto key = std::make_pair(packet.src, packet.dst);
    pairs[key].packets++;
    pairs[key].bytes += packet.total_len;
}
```

`unordered_map::operator[]` costruisce di default il valore se la chiave non esiste. `protocolStats` ha tutti i membri inizializzati a zero di default, quindi il primo pacchetto per qualsiasi protocollo inserisce una struct a zero e poi la incrementa. Stesso pattern per `IPStats` e per la mappa `pairs`.

Nota: `total_p` e `total_b` vengono aggiornati direttamente in `snapshot` (non in una struct separata) così sono immediatamente visibili in `get_snapshot()` senza una chiamata `update_*` aggiuntiva.

### `update_transport_stats()` (righe 93–107)

```cpp
std::vector<std::pair<TransportProtocol, protocolStats>> tps(transport_map.begin(), transport_map.end());
std::sort(tps.begin(), tps.end(), [](auto &a, auto &b) { return a.second.packets > b.second.packets; });
```

Non si può ordinare un `unordered_map` sul posto — prima si copia in un vettore, poi si ordina. La lambda confronta per numero di pacchetti in ordine decrescente. Ogni riga viene poi formattata con `std::format("{:.2f}", ...)` per i valori in MB e le percentuali.

### `update_bandwidth()` (righe 230–252)

```cpp
auto now = steady_clock::now();
double elapsed = duration_cast<duration<double>>(now - last_tick).count();

if (elapsed >= 1.0) {
    uint32_t delta_bytes = snapshot.total_b - last_b;
    snapshot.bandwidth = delta_bytes / elapsed;
    last_b = snapshot.total_b;
    last_tick = now;

    const double alpha = 0.2;
    smooth_bandwidth = alpha * snapshot.bandwidth + (1.0 - alpha) * smooth_bandwidth;
    snapshot.bandwidth_history.push_back({ts, smooth_bandwidth});
}
snapshot.max_bandwidth = std::max(snapshot.max_bandwidth, snapshot.bandwidth);
```

Campiona solo una volta al secondo (quando `elapsed >= 1.0`). `delta_bytes = current_total - last_snapshot_total`. Diviso per i secondi trascorsi = byte/sec. L'EMA con alpha=0.2 smussa i picchi. `max_bandwidth` viene aggiornato a ogni chiamata (non solo quando è trascorso un secondo) così traccia il picco effettivo.

---

## Costruttore di filtri — `src/cli/filter.cpp`

### `parse()` (righe 5–25)

```cpp
filter parse(const std::string &str) {
    auto pos = str.find(':');
    if (pos == std::string::npos)
        throw std::invalid_argument("Invalid filter format: '" + str + "' (expected key:value)");
    std::string type = str.substr(0, pos);
    std::string value = str.substr(pos + 1);
    if (type == "protocol") return {PROTOCOL, value};
    if (type == "port")     return {PORT, value};
    if (type == "dest")     return {IP_DEST, value};
    if (type == "src")      return {IP_SRC, value};
    if (type == "ip")       return {IP_TYPE, value};
    return {NONE, value};
}
```

`find(':')` restituisce `string::npos` (il valore massimo di `size_t`) se non trova nulla. Il throw previene il bug di overflow unsigned `npos + 1` che altrimenti farebbe restituire a `substr(npos + 1)` l'intera stringa come valore.

### `get_bpf_filter()` (righe 27–97)

```cpp
std::map<filter_type, std::vector<std::string>> groups;
for (const auto &x : f) {
    switch (x.type) {
    case PROTOCOL:
        if (x.val == "dns") groups[PROTOCOL].emplace_back("port 53");
        else if (x.val == "http") groups[PROTOCOL].emplace_back("port 80");
        // ...
        break;
    case IP_TYPE:
        if (x.val == "v4" || x.val == "4" || x.val == "ipv4") groups[IP_TYPE].emplace_back("ip");
        else if (x.val == "v6" || ...) groups[IP_TYPE].emplace_back("ip6");
        else throw std::invalid_argument("Unknown IP type: '" + x.val + "'");
        break;
    }
}
// combine: same-type = OR, different types = AND
for (auto &[type, parts] : groups) {
    if (!first_group) result += " and ";
    if (parts.size() > 1) result += "(";
    for (size_t i = 0; i < parts.size(); ++i) {
        result += parts[i];
        if (i + 1 < parts.size()) result += " or ";
    }
    if (parts.size() > 1) result += ")";
}
```

`std::map` (ordinata) garantisce un ordine di output deterministico indipendentemente dall'ordine di inserimento. La combinazione AND/OR di BPF segue la semantica standard dei filtri di rete: lo stesso tipo di filtro con più valori significa "corrisponde a uno qualsiasi di questi" (OR), mentre tipi di filtro diversi devono corrispondere tutti (AND).

Esempio: `-f protocol:http -f protocol:https -f port:8080` → `(port 80 or port 443 or port 8080)`

---

## Rendering della TUI — `src/TUI/view.cpp`

### Composizione del layout (righe 5–47)

```cpp
auto transport_section = hbox({
    render_transport(data) | flex,
    separator(),
    render_application(data) | flex,
    separator(),
    render_pairs(data) | flex,
}) | border;

auto ip_section = hbox({
    render_ip(data) | border | size(HEIGHT, LESS_THAN, 10) | frame | vscroll_indicator,
    render_bandwidth(data) | border | flex
});

auto right_panel = render_packets(data) | border | size(WIDTH, EQUAL, 100) | frame | vscroll_indicator;
```

FTXUI usa un modello di layout dichiarativo. `hbox` affianca gli elementi. `| flex` fa espandere un elemento per riempire lo spazio disponibile. `| size(HEIGHT, LESS_THAN, 10)` limita l'altezza. `| frame | vscroll_indicator` aggiunge il supporto allo scorrimento.

`separator()` disegna una linea verticale tra gli elementi.

### Grafico della banda (righe 138–175)

```cpp
GraphFunction fn = [this, data](int width, int height) {
    std::vector<int> output(width, 0);
    size_t n = data.bandwidth_history.size();
    size_t start = n > 50 ? n - 50 : 0;  // last 50 samples

    double max_bw = 1.0;
    for (size_t i = start; i < n; ++i)
        max_bw = std::max(max_bw, data.bandwidth_history[i].bytes_per_sec);

    for (int x = 0; x < width; ++x) {
        double t = (double)x / (width - 1);
        double idx_f = start + t * (n - start - 1);
        size_t i0 = (size_t)idx_f;
        size_t i1 = std::min(i0 + 1, n - 1);
        double frac = idx_f - i0;
        double bw = data.bandwidth_history[i0].bytes_per_sec * (1.0 - frac)
                  + data.bandwidth_history[i1].bytes_per_sec * frac;
        output[x] = static_cast<int>(bw / max_bw * (height - 1));
    }
    return output;
};
```

La `GraphFunction` mappa `width` colonne dello schermo su altezze intere limitate da `height`. Per ogni colonna di pixel `x`, calcola un indice frazionario nell'array dei campioni (mappando la larghezza dello schermo sull'intervallo dei campioni), interpola linearmente tra campioni adiacenti, poi normalizza rispetto all'altezza del grafico. `max_bw = 1.0` come valore minimo previene la divisione per zero quando non è ancora stato visto traffico.

### Rendering delle tabelle (righe 83–93, pattern ripetuto per tutte le tabelle)

```cpp
Table table(data.transport_rows);
table.SelectAll().Border(LIGHT);
table.SelectRow(0).Decorate(bold);
table.SelectRow(0).SeparatorVertical(LIGHT);
table.SelectRow(0).Border(DOUBLE);
return vbox({text("=== Transport protocols === ") | bold, table.Render()}) | flex;
```

`data.transport_rows` è un `vector<vector<string>>` in cui la riga 0 è l'intestazione. `SelectAll().Border(LIGHT)` disegna bordi leggeri attorno a tutte le celle. `SelectRow(0).Border(DOUBLE)` sovrascrive la riga di intestazione con un bordo doppio per distinguerla visivamente. `SelectRow(0).Decorate(bold)` rende in grassetto il testo dell'intestazione.
