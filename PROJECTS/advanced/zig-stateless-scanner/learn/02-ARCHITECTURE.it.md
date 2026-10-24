<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# Architettura

zingela è due macchine imbullonate insieme: un core stateless di trasmissione e ricezione a line rate che possiede l'hot path dei pacchetti, e un control plane memory-safe che interpreta le tue intenzioni, cadenza l'invio e presenta i risultati. Questo documento mostra come si incastrano.

## Il quadro generale

```
        you                       control plane                     data plane
   ------------      ------------------------------------      --------------------
   --target      ->  cli.zig  -> scancmd.zig                    tx.zig  (transmit)
   --ports           parse       validate, build Config    ->    pull target
   --rate            help        select backend                  stamp frame
   --backend         banner      launch engines                  rate-gate
                                   |          \                   push to ring
                                   |           \                       |
                                   v            v                      v
                              output.zig     rx.zig  (receive)   packet_io.zig
                              dashboard        read frame          AF_XDP or
                              table            validate cookie     AF_PACKET or
                              NDJSON           classify            connect
                                   ^            dedup
                                   |            push found host
                                   +---------------+
```

Il control plane non tocca mai un raw socket nel loop caldo. Il data plane non alloca mai dopo l'avvio. Questa separazione è ciò che mantiene lo scanner sia sicuro sia veloce.

## Il flusso di dati stateless

Segui una probe e la sua risposta lungo tutto il percorso. Questo è il cuore del design.

```
  TRANSMIT                                            RECEIVE
  --------                                            -------
  targets.Engine.next()                               packet_io rxPoll()
     gives (ip, port)                                    reads a frame
        |                                                   |
        v                                                   v
  cookie.seq(ip,port,src,srcport)                     classify.classifyTcp()
     = low 32 bits of                                    is it SYN-ACK / RST / ICMP?
       SipHash(4-tuple)                                    |
        |                                                  v
        v                                             cookie.validateSynAck(ack,...)
  template.stamp(frame, ip, port)                        ack == cookie + 1 ?
     writes Eth+IP+TCP,                                    |
     puts cookie in TCP seq                          yes --+-- no
        |                                             |          |
        v                                             v          v
  ratelimit token gate                          dedup.insert   drop (stray/spoof)
        |                                          new host?
        v                                             |
  packet_io txSubmit + txKick  --- wire --->      output emit
```

Nulla collega la colonna di sinistra a quella di destra tranne l'aritmetica. Il cookie scritto nel numero di sequenza in trasmissione è lo stesso cookie ricalcolato e verificato in ricezione. Non c'è una tabella condivisa, nessuna ricerca per correlation ID, nessuna memoria per probe. Una risposta che non porta un cookie valido viene scartata prima che possa inquinare i risultati.

## I due motori

Il motore di trasmissione e il motore di ricezione girano come task concorrenti lanciati con `io.concurrent` su `std.Io.Threaded` di Zig. Non si tratta della vecchia keyword `async`, che Zig ha rimosso, e non è un future per ogni probe. Sono due worker di lunga durata.

Il motore di trasmissione gira caldo. Preleva i target dalla permutazione degli indirizzi, timbra ciascuno in un frame buffer preallocato, lo fa passare attraverso il gate di rate del token bucket e lo consegna al backend dei pacchetti. Dopo l'avvio non alloca: il ring di trasmissione viene mappato in memoria una sola volta, e lo spazio di lavoro per pacchetto proviene da un buffer fisso.

Il motore di ricezione legge i frame dal backend, classifica ciascuno come SYN-ACK, RST o un errore ICMP rilevante, convalida il cookie, deduplica tramite una hash table lockless a indirizzamento aperto così che un host che risponde due volte venga riportato una sola volta, e spinge gli host appena trovati verso il lato di output. Le risposte sono rade rispetto alle probe, quindi il motore di ricezione è molto meno caldo di quello di trasmissione, ed è per questo che il design dei backend descritto più avanti può permettersi di essere asimmetrico.

## La scala dei backend

Il percorso di input e output dei pacchetti viene scelto a runtime attraverso un'unica interfaccia (`packet_io.zig`) con diverse implementazioni. Il core dello scanner non sa né si cura di quale sia attiva.

```
  --backend auto  picks the first that works:

   1. AF_XDP zero-copy      fastest, driver-dependent          -Dxdp build
   2. AF_XDP with XDP_SKB   copy mode, works on more drivers    -Dxdp build
   3. AF_PACKET + TX_RING   universal, kernel-bypass TX ring    always built
      + QDISC_BYPASS        the v1 default, matches masscan-stock
   4. AF_PACKET sendto      plain, used by the ground-truth smoke
   5. connect scan          TCP via the OS stack, no raw needed  --connect
```

Il percorso AF_XDP è asimmetrico: accelera la trasmissione e si abbina a un percorso di ricezione `AF_PACKET`. Le risposte sono rade, quindi questo scambia qualche punto percentuale di throughput teorico in ricezione con un design molto più semplice, senza steering lato ricezione né cablaggio di mappe. Quando gira `--backend auto`, una self-probe a due socket verifica prima se gli invii raw lasciano effettivamente la macchina. Su un'istanza cloud o una VM in cui l'hypervisor scarta silenziosamente i frame raw, quella probe fallisce e zingela ripiega sul connect scanner con un avviso chiaro, anziché riportare una scansione che non ha inviato nulla.

## La mappa dei moduli

| Modulo | Responsabilità |
|---|---|
| `main.zig` | entry point: interpreta gli argomenti, costruisce la config, smista verso un sottocomando |
| `cli.zig` | parsing degli argomenti, help, il banner e la versione |
| `scancmd.zig` | il comando scan: convalida, seleziona il backend, lancia trasmissione e ricezione, pilota l'output |
| `txcmd.zig` | il comando di blast in sola trasmissione |
| `targets.zig` | il motore di permutazione a gruppo ciclico, il cercatore di radici primitive, il selettore di indirizzi e porte, la soglia di esclusione |
| `numtheory.zig` | esponenziazione modulare, primalità, test delle radici primitive per la permutazione |
| `packet.zig` | header del formato wire, il checksum RFC 1071 (scalare e SIMD), preset SYN realistici per sistema operativo |
| `cookie.zig` | generazione e convalida dei SYN cookie SipHash, IPv4 e IPv6 |
| `template.zig` | il template del frame SYN che viene timbrato per ogni target |
| `ratelimit.zig` | il pacer a token bucket |
| `tx.zig` | il motore di trasmissione |
| `rx.zig` | il motore di ricezione |
| `classify.zig` | classificazione delle risposte in open, closed, filtered |
| `dedup.zig` | la tabella di dedup lockless a indirizzamento aperto |
| `packet_io.zig` | l'interfaccia dei backend, con le implementazioni `afpacket.zig`, `afxdp.zig` e `connect.zig` |
| `udp.zig` | payload UDP e classificazione specifica per UDP |
| `service.zig` | la fase opt-in di rilevamento di banner e servizi |
| `stealth.zig` | le funzionalità di evasion protette |
| `output.zig` | la dashboard live, la tabella dei risultati e NDJSON |
| `netutil.zig`, `ndp.zig`, `rawprobe.zig` | risoluzione di interfaccia e gateway, neighbor discovery IPv6, la self-probe per il cloud |

## Perché la separazione conta per la sicurezza

Il worm SQL Slammer del 2003 e ogni worm di scanning basato su buffer overflow venuto dopo condividono una causa radice: codice di gestione dei pacchetti scritto a mano in C che corrompe la memoria sotto carico. zingela viene compilato e testato prima di tutto in modalità ReleaseSafe di Zig, dove uno slice fuori dai limiti, un integer overflow o una dereferenziazione di null sono una trap definita anziché una corruzione silenziosa della memoria. Le struct del formato wire portano asserzioni di dimensione a compile time, ogni checksum e cookie viene verificato rispetto a un test vector pubblicato, e l'intera scansione viene convalidata end to end all'interno di un network namespace prima di ogni release. La [spiegazione dell'implementazione](03-IMPLEMENTATION.it.md) mostra dove vive ciascuna di queste protezioni.
