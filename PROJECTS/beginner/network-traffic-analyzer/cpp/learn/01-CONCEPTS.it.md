# Concetti

## Cattura dei pacchetti a livello kernel

Quando apri Wireshark e vedi i pacchetti, sono successe molte cose prima che il primo byte arrivi sullo schermo. Il kernel del sistema operativo riceve ogni frame dalla scheda di rete, e normalmente consegna ai processi utente solo i frame indirizzati alla tua macchina (o all'indirizzo di broadcast della tua subnet). Uno sniffer di pacchetti ha bisogno di tutto — inclusi i frame indirizzati ad altri host.

Questa è la modalità promiscua. Quando libpcap apre un dispositivo con `pcap_open_live(..., 1, ...)` (l'`1` è il flag di modalità promiscua, `pcapCapture.cpp:59`), chiede al kernel di passare tutti i frame indipendentemente dall'indirizzo MAC di destinazione. Lo stack di rete del kernel vede i frame prima che il livello di routing scarti quelli irrilevanti.

Il kernel copia i frame corrispondenti dallo spazio kernel a un ring buffer in user space. Il tuo programma legge da quel buffer tramite `pcap_loop()`. La copia è il collo di bottiglia — per questo i tool di cattura ad alte prestazioni (come quelli usati nei data center) usano meccanismi di kernel bypass come DPDK o XDP per saltare completamente la copia.

## BPF — Berkeley Packet Filter

BPF è una piccola macchina virtuale che gira dentro il kernel. Quando passi un'espressione di filtro come `tcp and port 443`, libpcap la compila in bytecode BPF e installa quel programma nel kernel. Il kernel esegue il programma BPF su ogni frame prima di decidere se copiarlo nello userspace.

Il vantaggio: su una rete trafficata, il 99% dei frame viene scartato nel kernel senza mai toccare lo userspace. Spostare il filtraggio dallo user space a BPF ha ridotto l'uso della CPU da ~80% a ~5% in scenari di monitoraggio di rete in produzione con filtri specifici per porta.

In questo progetto, `filter.cpp` costruisce le stringhe delle espressioni BPF (`get_bpf_filter()`, riga 27), e `pcapCapture.cpp` le compila e le installa tramite `pcap_compile()` + `pcap_setfilter()` (righe 68–75).

### Scrivere espressioni BPF

Sintassi BPF accettata da pcap:
```
tcp                        — only TCP traffic
port 443                   — source or destination port 443
host 192.168.1.1           — to or from specific IP
src host 10.0.0.1          — from specific source
dst host 10.0.0.1 and port 80  — combined with AND
tcp or udp                 — combined with OR
```

Il costruttore di filtri del progetto mappa la propria sintassi chiave:valore su BPF:
- `protocol:https` → `port 443`
- `ip:v4` → `ip`
- `src:192.168.1.1` → `src host 192.168.1.1`
- Più filtri dello stesso tipo vengono combinati in OR, tipi diversi vengono combinati in AND

## Frame Ethernet e tipi di link layer

Ogni pacchetto su una rete fisica inizia con un header di link layer. Su Ethernet (il caso comune), è un header Ethernet di 14 byte: 6 byte di MAC di destinazione, 6 byte di MAC sorgente, 2 byte di EtherType.

Ma non tutte le interfacce usano header Ethernet. La pseudo-interfaccia `any` di Linux usa `DLT_LINUX_SLL` (un header sintetico di 16 byte). Alcuni ambienti usano `DLT_LINUX_SLL2` (header di 20 byte). L'offset prima del livello IP varia a seconda del tipo di link.

`pcapCapture.cpp:datalink_type()` (righe 13–39) gestisce questo con uno switch sul tipo di link restituito da `pcap_datalink()`. Imposta sia l'`offset` (quanti byte saltare prima del livello IP) sia una lambda `get_ether_type` che estrae il campo EtherType dalla posizione corretta.

```
DLT_EN10MB  → offset = 14, EtherType at bytes 12-13
DLT_LINUX_SLL  → offset = 16, protocol at bytes 14-15
DLT_LINUX_SLL2 → offset = 20, protocol at bytes 18-19
```

EtherType `0x0800` = IPv4, `0x86DD` = IPv6. La funzione `got_packet()` (riga 152) legge l'EtherType dal pacchetto usando `get_ether_type(packet)` e fa dispatch verso `IPv4` o `IPv6` di conseguenza.

## Parsing degli header di protocollo

Dopo aver saltato l'header di link layer, l'header IP inizia a `packet + offset`. Il parsing è un semplice cast di puntatori grezzi:

```cpp
// IP.cpp:19 — cast raw bytes to ip header struct
ip_hdr = reinterpret_cast<const ip *>(data);
```

La struct `ip` di `<netinet/ip.h>` mappa i campi su offset di byte noti — `ip_hl` ai bit 0–3 del byte 0 (la lunghezza dell'header IP in parole da 4 byte), `ip_src` e `ip_dst` ai byte 12–15 e 16–19.

La lunghezza dell'header IP è `ip_hl * 4`. Il minimo è 20 byte (senza opzioni). IPv4.cpp lo valida alla riga 25:
```cpp
if (ip_hdr_len < 20) throw std::runtime_error("Failed to initial IPv4 ");
```

L'header di trasporto segue immediatamente l'header IP:
```cpp
// IP.cpp:53 — walk past the IP header to reach TCP
const auto *tcp = reinterpret_cast<const tcphdr *>(
    reinterpret_cast<const u_char *>(ip_hdr) + ip_hdr_len
);
```

L'header TCP ha una propria lunghezza variabile: `tcp->doff * 4` byte (campo Data Offset, minimo 20 byte). Il payload inizia immediatamente dopo:
```cpp
// IP.cpp:58 — TCP payload pointer
payload_ptr = reinterpret_cast<const u_char *>(tcp) + tcp->doff * 4;
payload_len = ntohs(ip_hdr->ip_len) - (ip_hdr_len + tcp->doff * 4);
```

Nota il `reinterpret_cast<const u_char *>(tcp)` prima dell'addizione. L'aritmetica dei puntatori su un puntatore tipizzato avanza di multipli di `sizeof(T)` — senza il cast a puntatore a byte, `tcp + doff * 4` avanzerebbe di `doff * 4 * sizeof(tcphdr)` byte, cioè 20 volte troppo.

## Identificazione del protocollo applicativo

Il rilevamento del protocollo applicativo usa due strategie, provate in ordine (`packet.cpp:4–57`):

**Ispezione del payload (deep packet inspection):** Confronta i primi byte del payload con valori magic noti:
- HTTP: i primi 4 byte sono `GET `, `POST`, `HEAD`, `PUT ` o `HTTP` (riga 9)
- TLS/HTTPS: il primo byte è `0x16` (tipo di record TLS = handshake), il secondo è `0x03` (versione major) (riga 18)

**Fallback basato sulla porta:** Quando il payload è assente o non riconosciuto, controlla le porte well-known:
- TCP 22 → SSH, 25 → SMTP, 80 → HTTP, 443 → HTTPS
- UDP 53 → DNS, 443 → QUIC, 123 → NTP

L'identificazione del protocollo avviene nel costruttore di `Packet` (packet.hpp:52):
```cpp
application_protocol = get_application_protocol();
this->payload_ptr = nullptr;  // null after identification — payload no longer needed
```

Impostare `payload_ptr = nullptr` dopo l'uso è intenzionale. Il puntatore punta dentro il ring buffer interno di libpcap, che è valido solo durante la callback di `pcap_loop`. Una volta che il `Packet` è memorizzato nella deque `Stats.packets`, il puntatore sarebbe pendente (dangling). Annullarlo rende la cosa esplicita.

## Thread safety e il pattern dello snapshot

Due thread accedono all'oggetto `Stats` in modo concorrente: il thread di cattura (chiama `add_packet()`, `push()`) e il thread di aggiornamento della UI (chiama tutti i metodi `update_*()` e `get_snapshot()`).

Tutti i metodi di `Stats` acquisiscono il lock su `mtx` all'ingresso (`protocolStats.cpp:20`, `94`, `116`, `141`, ecc.). Le operazioni di scrittura si completano sotto il lock. Le letture tramite `get_snapshot()` (protocolStats.hpp:89) restituiscono una copia della struct snapshot sotto lo stesso lock:

```cpp
StatsSnapshot get_snapshot() {
    std::lock_guard<std::mutex> lock(mtx);
    return snapshot;  // copy-on-exit
}
```

La lambda di rendering FTXUI in `main.cpp` (riga 92) legge da `current_render` sotto `render_mtx`, non direttamente da `Stats`. Il thread di aggiornamento della UI in `application_thread` (riga 107) chiama `get_snapshot()`, costruisce un nuovo albero di elementi FTXUI, lo memorizza in `current_render` sotto `render_mtx`, poi invia un evento personalizzato per innescare un repaint:

```cpp
ftxui::Element new_frame = view.render(stats.get_snapshot(), ...);
{
    std::lock_guard<std::mutex> lock(render_mtx);
    current_render = new_frame;
}
screen.PostEvent(ftxui::Event::Custom);
```

Questo design fa sì che il thread dell'event loop di FTXUI non tocchi mai direttamente `Stats` — legge solo l'elemento `current_render` già costruito.

## Header di estensione IPv6

IPv6 ha eliminato il campo opzioni dall'header fisso e lo ha sostituito con gli header di estensione — una catena di header opzionali tra l'header base di 40 byte e il livello di trasporto. Ogni header di estensione ha un campo `Next Header` che punta al successivo nella catena.

Il costruttore di `IPv6` (`IP.cpp:82–136`) percorre questa catena in un ciclo `while(true)`. A ogni iterazione legge il tipo dell'header corrente e o fa dispatch verso il gestore di trasporto (TCP/UDP/ICMP/ICMPV6/IGMP) e ritorna, oppure avanza oltre un header di estensione noto (Hop-by-Hop Options, Routing, Destination Options, Fragment) e continua:

```cpp
case IPPROTO_HOPOPTS:
case IPPROTO_ROUTING:
case IPPROTO_DSTOPTS: {
    const auto *ext = reinterpret_cast<const ip6_ext *>(ptr);
    hdr = ext->ip6e_nxt;
    ptr += (ext->ip6e_len + 1) * 8;  // len in 8-byte units, not counting first 8
    break;
}
```

L'aritmetica `(ext->ip6e_len + 1) * 8` è la formula standard della RFC 2460: il campo lunghezza conta unità da 8 byte escludendo i primi 8 byte.

## Calcolo della banda

La banda viene calcolata una volta al secondo in `update_bandwidth()` (`protocolStats.cpp:230–252`):

```
delta_bytes = total_bytes_now - total_bytes_last_tick
bandwidth   = delta_bytes / elapsed_seconds  (bytes/sec)
```

La banda grezza è rumorosa (traffico a raffiche, tempistica dei tick variabile), quindi una media mobile esponenziale la smussa:
```cpp
const double alpha = 0.2;
smooth_bandwidth = alpha * snapshot.bandwidth + (1.0 - alpha) * smooth_bandwidth;
```

Un alpha di 0.2 dà ai campioni recenti un peso del 20% e alla media storica un peso dell'80%. Alpha più basso = più liscio ma con più ritardo. Il valore smussato viene memorizzato in `bandwidth_history` per il grafico della TUI.

Il grafico della TUI in `view.cpp:138–174` mappa gli ultimi 50 campioni di banda sulla larghezza del widget grafico usando un'interpolazione lineare tra i campioni — scalando ciascun campione a un valore di altezza basato sulla banda massima corrente.
