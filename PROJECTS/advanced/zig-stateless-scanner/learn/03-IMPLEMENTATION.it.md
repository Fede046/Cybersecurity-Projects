<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# Guida all'implementazione

Un tour guidato del codice. Cita funzioni e file, mai numeri di riga, così resta corretto mentre il sorgente si sposta. Leggilo con il sorgente aperto accanto. Per i dettagli a livello di byte e di matematica, segui i rimandi a [MECHANICS.md](MECHANICS.it.md).

## Formati wire e checksum: `packet.zig`

Tutto ciò che lo scanner mette sul cavo parte da qui. Gli header Ethernet, IPv4, TCP, UDP e IPv6 sono dichiarati come `extern struct` così che l'ordine dei campi e l'impacchettamento corrispondano esattamente al wire, e ciascuno porta un'asserzione di dimensione a compile time così che un campo sbagliato non possa mai cambiare silenziosamente il layout. I campi multi-byte vengono convertiti esplicitamente in network byte order nel punto di scrittura, mai lasciati al caso.

Il checksum vive in due funzioni. `checksum` è la somma scalare in complemento a uno di RFC 1071. `checksumSimd` è la stessa matematica su un accumulatore vettoriale per buffer grandi. Entrambe sono convalidate rispetto all'esempio svolto pubblicato in RFC 1071. `tcpChecksum` e `tcpChecksum6` costruiscono lo pseudo-header ed eseguono la somma sul segmento TCP per IPv4 e IPv6. C'è anche `incrementalUpdate`, il trucco di RFC 1624 che corregge un checksum quando cambia un singolo campo senza risommare l'intero pacchetto.

`ScanType` e `OsProfile` codificano le varianti stealth. `ScanType.probeFlags` restituisce il byte dei flag TCP per uno scan syn, fin, null, xmas, maimon, ack o window, e `OsProfile.options` restituisce i byte delle opzioni TCP di un SYN realistico Linux, Windows o macOS, così che un fingerprint sembri quello di un client reale anziché di uno scanner.

L'union `Addr` è il modo in cui un unico percorso dei risultati serve entrambe le famiglie di indirizzi: contiene o un `u32` per IPv4 o un `[16]u8` per IPv6, così il codice di classificazione, output e dedup è scritto una sola volta.

## Il cookie: `cookie.zig`

Questa è la statelessness in circa sessanta righe. Un `Cookie` è una chiave da 16 byte, creata da entropia casuale per singola scansione. `generate` impacchetta la quadrupla in un piccolo buffer e restituisce `std.hash.SipHash64(2, 4).toInt` su di esso, un hash completo a 64 bit. `seq` lo tronca ai 32 bit che stanno in un numero di sequenza TCP. `validateSynAck` ricalcola `seq` dagli indirizzi e dalle porte della risposta e restituisce se l'acknowledgement è uguale a `seq +% 1`, usando l'addizione con wrapping perché il cookie può valere `0xFFFFFFFF` e una somma normale andrebbe in overflow-trap in una build safe.

Le varianti IPv6 `generate6`, `seq6` e `validateSynAck6` fanno lo stesso su indirizzi da 16 byte. `udpSrcPort` riusa l'hash per derivare una porta sorgente per target per UDP, così che la porta di destinazione della risposta porti essa stessa informazioni di convalida. Uno unit test riproduce il test vector canonico di SipHash per il messaggio vuoto, così la primitiva è dimostrata, non presunta.

## Il motore degli indirizzi: `targets.zig` e `numtheory.zig`

`parseCidr` trasforma `192.0.2.0/24` in un `Range` di indirizzi iniziale e finale. `IpPicker.build` prende i tuoi intervalli, sottrae ogni blocco riservato tramite `subtractReserved`, ordina ciò che resta e costruisce un indice a prefissi cumulativi così che un intero piatto possa essere mappato su un indirizzo concreto con una ricerca binaria in `IpPicker.at`. È qui che `10.0.0.0/8` si riduce a nulla: la sottrazione dei riservati rimuove l'intero blocco privato.

`Engine` è la permutazione. `Engine.init` calcola la dimensione totale dello spazio indirizzi-e-porte, trova il più piccolo primo superiore con `numtheory.smallestPrimeAbove` e sceglie una nuova radice primitiva con `numtheory.findPrimitiveRoot`. Nota qui la differenza con zmap: zmap ha in hardcode una tabella di undici coppie primo e radice primitiva, mentre zingela calcola a runtime il primo per la dimensione esatta della tua scansione, così una scansione piccola ottiene un gruppo piccolo e non spreca iterazioni a ritirare valori fuori intervallo. `Engine.next` fa avanzare il gruppo con `numtheory.mulMod`, decodifica l'elemento risultante in un indirizzo e una porta tramite il picker e restituisce un `Target`. `initShard` affetta il gruppo in archi contigui così che più macchine possano condividere una scansione tramite seed.

`numtheory.findPrimitiveRoot` è quella interessante. Anziché fare test a forza bruta, fattorizza `prime - 1` con `distinctPrimeFactors` e, per ogni candidato casuale, verifica con `isPrimitiveRoot` che nessun `modExp(candidate, (prime-1)/q, prime)` sia uguale a uno. Un candidato che supera il test genera l'intero gruppo. La matematica è in [MECHANICS.md](MECHANICS.it.md).

## Timbrare i frame: `template.zig`

`SynTemplate.init` costruisce un frame di base una sola volta a partire da una `Config` (MAC sorgente e di destinazione, indirizzo e porta sorgente, cookie, profilo del sistema operativo, tipo di scansione), stendendo gli header Ethernet, IPv4 e TCP con tutto ciò che non cambia tra un target e l'altro. `stamp` è la funzione calda per target: copia la base, scrive l'indirizzo di destinazione, varia il campo di identificazione IP, scrive la porta di destinazione, calcola il cookie con `cookie.seq` e lo colloca nel campo di sequenza o di acknowledgement a seconda del tipo di scansione, e conclude con i checksum IP e TCP. `stampVariant` sovrappone indirizzi sorgente decoy per la suite stealth. `SynTemplate6` è la controparte IPv6.

## Cadenza: `ratelimit.zig`

`TokenBucket` è una banca di nanosecondi. `init` imposta un passo per token in nanosecondi a partire dai pacchetti al secondo richiesti e una capacità per brevi burst. `takeBatch` legge il clock monotono, accredita nella banca il tempo trascorso e restituisce quanti token il motore di trasmissione può spendere in questo momento, ed è così che il mittente resta al rate senza busy-waiting. `refund` restituisce i token quando un invio non è stato completato, e `withJitter` perturba la temporizzazione per la modalità di jitter di Poisson. Questo è un vero token bucket, distinto dal controller a feedback proporzionale di masscan su un ring di timestamp.

## I motori: `tx.zig` e `rx.zig`

`tx.zig` è il loop di trasmissione: innesca il bucket, preleva i target dall'`Engine`, li timbra tramite il template, li sottopone al gate del bucket e li raggruppa in batch nel ring di trasmissione del backend prima di dare il colpo al kernel perché li invii. Non alloca nulla dopo l'avvio. `rx.zig` apre il socket di ricezione per l'ethertype giusto e legge i frame in un loop.

La classificazione vive in `classify.zig`. `classifyTcp` ispeziona un segmento TCP ricevuto e decide open, closed o filtered, subordinato a un cookie valido così che contino solo le risposte genuine, e mappa gli errori ICMP rilevanti su filtered. `classifyTcp6` fa lo stesso per IPv6, inclusi gli unreachable ICMPv6. La deduplicazione vive in `dedup.zig`: `Dedup.insert` mescola la chiave con un finalizer fmix64 e la colloca in una tabella a indirizzamento aperto con linear probing, restituendo se l'host è stato visto per la prima volta così che ogni host venga riportato una sola volta.

## I backend: `packet_io.zig`, `afpacket.zig`, `afxdp.zig`, `connect.zig`

`packet_io.zig` definisce l'interfaccia chiamata dai motori: apertura, riserva di uno slot di trasmissione, submit, kick, polling dei frame ricevuti, chiusura. `afpacket.zig` la implementa su un socket `AF_PACKET` con un `PACKET_TX_RING` mappato in memoria e `PACKET_QDISC_BYPASS`, il percorso universale che non richiede moduli kernel. `afxdp.zig` implementa il percorso accelerato su `AF_XDP` con la propria regione di memoria utente e i propri ring, compilato solo quando è impostato `-Dxdp`. `connect.zig` è il fallback non privilegiato: esegue chiamate `connect` raw non bloccanti tramite `std.os.linux`, le attende con `poll` e legge l'errore del socket per classificare open, closed o filtered. Ha un proprio thread pool dimensionato perché una connect blocca un thread, quindi la concorrenza è pari alla dimensione del pool.

Questo approccio con socket raw è deliberato. Il connect scanner pilota direttamente socket non bloccanti e `poll` anziché passare per la connect-with-timeout del livello di networking standard, che `connect.zig` non usa per il percorso di scansione.

## UDP, rilevamento dei servizi, stealth e output

`udp.zig` contiene la tabella dei payload a compile time e la classificazione UDP, dove un unreachable ICMP tipo 3 codice 3 significa closed e il silenzio viene riportato onestamente come open-or-filtered, perché un servizio UDP può semplicemente non rispondere a una probe che non riconosce. `service.zig` è la seconda fase opt-in: su una porta aperta invia una NULL probe e una richiesta HTTP, cattura il banner e rileva TLS senza decifrarlo. `stealth.zig` raccoglie le funzionalità di evasion protette. `output.zig` disegna la dashboard live truecolor e la tabella dei risultati sul terminale mentre scrive NDJSON sullo standard output, così la vista visuale e quella per le macchine non entrano mai in collisione.

## Dove vivono le prove

Ognuno di questi moduli include unit test nello stesso file, eseguiti con `zig build test`. Il checksum ha il suo vettore RFC, il cookie ha il suo vettore SipHash e un round trip cookie-più-uno, il motore degli indirizzi ha un test di proprietà di biiezione su uno spazio piccolo, e la tabella di dedup ha test di crescita e di collisione. L'intera pipeline viene esercitata end to end all'interno di un network namespace prima della release. Questa è la regola del repository: nessun codice senza prova.
