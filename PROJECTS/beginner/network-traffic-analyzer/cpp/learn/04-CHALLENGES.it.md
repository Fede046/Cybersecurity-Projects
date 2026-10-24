# Sfide ed estensioni

Queste sono cose concrete che puoi costruire sopra la codebase esistente. Ognuna ha un punto di partenza chiaro nel codice e un motivo per cui vale la pena farla.

## Principiante

### 1. Aggiungi la suddivisione per tipo ICMP

Al momento i pacchetti ICMP contano come un'unica voce di protocollo. ICMP trasporta molti tipi di messaggio: echo request/reply (ping), port unreachable, time exceeded (hop di traceroute), ecc.

**Cosa fare:** In `IPv4::handle_icmp()` (`src/packet/IP.cpp:73`), fai il cast del payload a `icmphdr` e leggi `type` e `code`. Estendi `ApplicationProtocol` in `packet.hpp` per includere `ICMP_ECHO`, `ICMP_UNREACHABLE`, ecc., oppure aggiungi un campo separato `icmp_type` a `Packet`.

**Perché è importante:** ICMP viene usato spesso per la ricognizione (ping sweep) e per i canali nascosti (tool di ICMP tunneling come `icmptunnel` codificano dati nel payload). Riuscire a distinguere gli echo request dai messaggi unreachable ti dice se sei sotto scansione o se ci sono route interrotte.

### 2. Codifica i protocolli a colori nella TUI

Aggiungi colore alle tabelle dei protocolli di trasporto e applicativi — TCP in blu, UDP in verde, ICMP in giallo, sconosciuti in rosso.

**Cosa fare:** In `view.cpp:render_transport()`, accedi alle singole celle con `table.SelectCell(row, col).Decorate(color(Color::Blue))`. `Table::SelectCell()` di FTXUI prende gli indici di riga e colonna.

**Perché è importante:** Gli operatori di sicurezza scorrono le dashboard sotto pressione. La codifica a colori permette all'occhio di saltare alle anomalie (traffico UDP inatteso, protocolli sconosciuti) senza leggere ogni riga.

### 3. Aggiungi un contatore del tasso di pacchetti

Mostra i pacchetti/sec accanto al grafico della banda. Il grafico della banda mostra byte/sec, ma il tasso di pacchetti è utile a sé — un'inondazione di pacchetti minuscoli a bassa banda è un pattern diverso rispetto a pochi trasferimenti grandi.

**Cosa fare:** Aggiungi `uint32_t last_p = 0` a `Stats` (accanto a `last_b`). In `update_bandwidth()` (`protocolStats.cpp:230`), calcola `delta_packets / elapsed` insieme al calcolo esistente sui byte. Aggiungilo a `StatsSnapshot` e mostralo in `render_header()` o `render_stats()`.

---

## Intermedio

### 4. Riassemblaggio degli stream TCP

Al momento ogni pacchetto TCP viene analizzato in modo indipendente. I protocolli di livello applicativo che si estendono su più pacchetti (risposte HTTP, trasferimenti FTP) non vengono ricostruiti. Il riassemblaggio combina i segmenti TCP in ordine di numero di sequenza in uno stream.

**Cosa fare:** Aggiungi una classe `StreamTable` che mappa `(src_ip, dst_ip, src_port, dst_port)` su un buffer ordinato di segmenti (indicizzati per numero di sequenza). In `got_packet()` (`pcapCapture.cpp:152`), dopo aver costruito un `Packet` TCP, inseriscilo nella tabella degli stream. Quando i segmenti arrivano in ordine, accodali al buffer dello stream. Quando un buco viene colmato, esegui l'identificazione del livello applicativo sul buffer completo.

**Perché è importante:** Il rilevamento HTTP tramite corrispondenza del verbo (`memcmp(payload_ptr, "GET ", 4)` in `packet.cpp:9`) funziona solo se la richiesta HTTP sta nel primo pacchetto. Per body PUT/POST grandi o connessioni lente, il verbo potrebbe trovarsi in un segmento precedente già elaborato. Il riassemblaggio è il modo in cui Snort, Suricata e gli engine DPI commerciali gestiscono questo caso.

### 5. Logging delle query DNS

Estrai i nomi delle query DNS dai pacchetti UDP sulla porta 53 e registrali con i timestamp.

**Cosa fare:** In `Packet::get_application_protocol()` (`packet.cpp:14`), quando viene identificato il DNS, analizza la sezione question del DNS a partire da `payload_ptr`. Il DNS è binario: i byte 0–11 sono l'header (ID, flag, contatori), i byte dal 12 in poi sono la sezione question come sequenza di etichette con prefisso di lunghezza (es. `\x03www\x07example\x03com\x00`). Percorri le etichette per estrarre l'FQDN. Aggiungi un campo stringa `dns_query` a `Packet` e mostralo nella tabella dei pacchetti.

**Perché è importante:** Il DNS è l'elenco telefonico che gli attaccanti usano sempre. I beacon C2 fanno check-in tramite DNS. L'esfiltrazione di dati codifica i dati nelle query DNS (DNS tunneling). Un semplice log delle query intercetta malware come `dnscat2`, che usa i record DNS TXT per una shell di comando — le query mostrano nomi di sottodominio assurdamente lunghi.

### 6. Engine di regole di alert

Aggiungi un engine di valutazione delle regole che lancia alert quando il traffico soddisfa condizioni configurabili: "alert se un singolo IP invia > 1000 pacchetti in 10 secondi" (port scan), "alert se le query DNS/sec sono > 100" (tunneling), "alert se compare un nuovo IP che non era nella baseline" (movimento laterale).

**Cosa fare:** Crea una struct `AlertRule` con una funzione di condizione e una soglia. Crea un `AlertEngine` che il thread di aggiornamento della UI chiama dopo `update_ip_stats()`. Le regole ispezionano lo snapshot di `Stats` alla ricerca di violazioni delle soglie. Memorizza gli alert scattati in una `deque<Alert>` e aggiungi un pannello in `view.cpp` per mostrarli.

**Perché è importante:** Questo è il cuore di ciò che fa un SIEM. I SIEM (Splunk, Elastic SIEM, IBM QRadar) correlano eventi da più sorgenti, ma l'idea di fondo — valutare condizioni di regole rispetto a metriche osservate, lanciare un alert quando vengono superate — è ciò che stai costruendo qui.

---

## Avanzato

### 7. Hex dump del payload dei pacchetti

Aggiungi una vista di dettaglio che mostri i byte esadecimali grezzi e la rappresentazione ASCII del payload di un pacchetto selezionato — come il pannello inferiore di Wireshark.

**Cosa fare:** Il `payload_ptr` viene attualmente annullato dopo l'esecuzione di `get_application_protocol()` (`packet.hpp:53`). Per supportare l'hex dump, copia i byte del payload in un `vector<uint8_t>` prima di annullarlo. Aggiungi allo stato di `View` un indice del pacchetto selezionato, un modo per navigarlo (tasti freccia tramite `CatchEvent`) e un metodo `render_hex_dump()` che formatta i byte in righe `XX XX XX ... | .text..`.

FTXUI non ha un widget di hex dump integrato. Lo costruiresti con una serie di elementi `hbox({text(hex_col) | fixed(50), text(ascii_col)})`.

**Perché è importante:** L'ispezione del payload è il modo in cui verifichi che un alert sia reale. Il "traffico DNS" è facile da rilevare. Sapere se quel traffico DNS contiene query normali o un'esfiltrazione codificata in base64 richiede di guardare i byte.

### 8. Baseline delle anomalie e rilevamento delle deviazioni

Registra una baseline del traffico su una finestra configurabile (es. i primi 60 secondi), poi segnala le deviazioni. Un nuovo protocollo che compare, un IP normalmente silenzioso che diventa uno dei top talker, o traffico TCP su una porta solo UDP meritano tutti un'indagine.

**Cosa fare:** Aggiungi una classe `Baseline` che fa uno snapshot di `transport_map` e `ip_map` dopo la finestra di baseline. Aggiungi una funzione `compare(current_snapshot, baseline)` che calcola gli Z-score o le variazioni percentuali per ogni metrica. Memorizza le deviazioni in un `vector<Deviation>` e mostrale in un nuovo pannello della TUI.

**Perché è importante:** L'attacco SolarWinds del 2020 è rimasto non rilevato per 9 mesi in parte perché il traffico malevolo imitava la normale telemetria del prodotto Orion — sembrava traffico atteso a chiunque lo controllasse manualmente. Un confronto automatico con la baseline avrebbe segnalato il nuovo pattern di beacon rispetto alla baseline precedente alla compromissione.

### 9. Esportazione PCAP durante la cattura dal vivo

Permetti all'utente di scrivere in tempo reale un file `.pcap` del traffico catturato, non solo di esportare le statistiche alla fine. Questo abilita l'analisi offline in Wireshark a posteriori.

**Cosa fare:** Usa `pcap_dump_open()` per creare un handle di file di dump, e `pcap_dump()` dentro `got_packet()` (`pcapCapture.cpp:152`) per scrivere ogni pacchetto grezzo con il suo `pcap_pkthdr`. Aggiungi un flag `--write` ad `argsParser` (`argsParse.cpp`). L'handle di dump è un'altra risorsa C che beneficia di un wrapping RAII.

**Perché è importante:** I tool di cattura dal vivo nell'incident response scrivono sempre file pcap. Vuoi catturare prima e analizzare dopo — soprattutto se l'attacco è ancora in corso. Tool come `tcpdump -w` e `dumpcap` fanno esattamente questo.

### 10. Rilevamento di port scanning consapevole del protocollo

Rileva gli stream solo-SYN (in cui le connessioni TCP vengono avviate ma mai completate) e segnalali come potenziali port scan.

**Cosa fare:** Nella tabella degli stream TCP (dalla sfida 4), traccia i flag TCP di ogni pacchetto. Un SYN senza risposta SYN-ACK entro un timeout è una scansione half-open. Conta il numero di porte di destinazione distinte da un singolo IP sorgente in una finestra temporale — più di N porte uniche in M secondi = probabile scansione. Questo è l'algoritmo usato da `portsentry`, `snort` e dai firewall moderni.

La botnet Mirai del 2016 ha scansionato l'intera Internet IPv4 alla ricerca di porte Telnet/SSH aperte in meno di un'ora eseguendo scansioni SYN da oltre 100.000 dispositivi infetti contemporaneamente. Rilevare i pattern di scansione a livello di pacchetto — non solo contare i tentativi di connessione — è il modo in cui i sistemi IDS di rete intercettano questo fenomeno.
