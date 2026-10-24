<!-- ©AngelaMos | 2026 -->
<!-- MECHANICS.md -->

# Meccanica

L'approfondimento a livello di byte e di matematica. Dove [03-IMPLEMENTATION.md](03-IMPLEMENTATION.it.md) dice cosa fa ogni modulo, questo documento mostra l'aritmetica che lo rende corretto e veloce. Tutto ciò che è qui corrisponde al sorgente in `cookie.zig`, `packet.zig`, `targets.zig`, `numtheory.zig`, `dedup.zig` e `ratelimit.zig`.

## Il SYN cookie, byte per byte

Il cookie è ciò che permette a zingela di non mantenere stato. `generate` costruisce un messaggio di 12 byte a partire dalla quadrupla, in network byte order, e ne calcola l'hash:

```
  data[12]:
   0        4      6            10   12
   +--------+------+------------+----+
   | ip_them|p_them|   ip_me    |p_me|
   +--------+------+------------+----+
     u32 BE  u16 BE   u32 BE    u16 BE

  cookie64 = SipHash64(2,4)(data, key)      key = 16 random bytes per scan
  seq32    = low 32 bits of cookie64
```

SipHash è una funzione pseudo-casuale con chiave. Senza la chiave non puoi prevederne l'output, quindi un attaccante non può falsificare un numero di sequenza che superi la convalida. Il `(2, 4)` indica il numero di round di compressione e di finalizzazione dello standard SipHash-2-4. Uno unit test in `cookie.zig` riproduce il test vector pubblicato di SipHash per il messaggio vuoto, il che dimostra che la primitiva è collegata correttamente.

In trasmissione, `seq32` va nel campo di sequenza TCP. Un target che accetta il SYN risponde con un SYN-ACK il cui campo acknowledgement è il tuo numero di sequenza più uno, perché TCP conferma il byte successivo che si aspetta. In ricezione, `validateSynAck` ricalcola `seq32` dagli indirizzi e dalle porte della risposta stessa e verifica:

```
  ack  ==  seq32  +%  1
```

Il `+%` è l'addizione con wrapping. Conta: `seq32` può valere `0xFFFFFFFF`, e in una build Zig safe un semplice `+ 1` su quel valore andrebbe in trap per overflow. Il wrapping fa sì che `0xFFFFFFFF + 1` diventi `0x00000000`, esattamente come lo calcola lo stack TCP del target.

Perché questo rifiuta gli spoof. Un pacchetto che non è stato generato in risposta a una delle nostre probe porterà, con probabilità schiacciante, un acknowledgement che non è uguale all'hash con chiave della sua quadrupla più uno. Un attaccante avrebbe bisogno della chiave della singola scansione per falsificarne uno. Quindi la convalida è un singolo ricalcolo dell'hash e un confronto, nessuna ricerca in tabella, e scarta a costo zero pacchetti vaganti, duplicati tardivi e spoof iniettati.

Per UDP, `udpSrcPort` deriva la porta sorgente stessa dall'hash, così la porta di destinazione della risposta porta l'informazione di convalida anche quando il payload non lo fa.

## Il checksum RFC 1071

L'internet checksum è la somma in complemento a uno di parole a 16 bit. `checksum` in `packet.zig` la implementa direttamente:

```
  sum = 0
  for each 16-bit big-endian word w in the buffer:
      sum += w                         (sum is 32-bit, so carries pile up high)
  if one odd byte remains:
      sum += byte << 8                 (pad on the right with zero)
  while sum >> 16 != 0:                (fold the carries back in)
      sum = (sum & 0xffff) + (sum >> 16)
  return ~sum                          (one's complement, low 16 bits)
```

Il loop di fold è l'end-around carry che definisce l'aritmetica in complemento a uno: qualsiasi overflow sopra il bit 15 viene risommato nella parola bassa. Per farsene un'idea concreta: se la somma parziale arrivasse a `0x1_2345`, un fold dà `0x2345 + 0x1 = 0x2346`, e il checksum restituito è `~0x2346 = 0xDCB9`. Il ricevente esegue la stessa identica somma includendo il campo checksum e si aspetta `0xFFFF`, ed è così che viene individuato un pacchetto corrotto.

`checksumSimd` calcola lo stesso valore più velocemente su buffer grandi. Legge il buffer in blocchi larghi quanto l'unità vettoriale della macchina, reinterpreta ogni blocco come un vettore di parole a 16 bit, ne inverte i byte su una CPU little-endian così che l'aritmetica sia big-endian, e accumula in un vettore di lane a 32 bit. Dopo il corpo vettorizzato riduce le lane a un'unica somma, gestisce eventuali byte di coda con il loop scalare, e fa fold e complemento esattamente come sopra. Il risultato è identico a quello della funzione scalare, come asseriscono i test.

C'è una terza funzione, `incrementalUpdate`, da RFC 1624. Quando cambia esattamente un campo a 16 bit di un pacchetto, non è necessario risommare tutto. Dati il vecchio checksum, la vecchia parola e la nuova parola:

```
  sum = ~old_check + ~old_word + new_word
  fold carries
  return ~sum
```

Ecco perché timbrare un nuovo indirizzo di destinazione in un template in cache non costa un passaggio completo di checksum su ogni pacchetto.

## La permutazione a gruppo ciclico

Il motore degli indirizzi visita ogni indirizzo e porta esattamente una volta, in ordine rimescolato, con tre numeri di stato. La matematica è quella del gruppo moltiplicativo degli interi modulo un primo.

Scegli un primo `p` appena più grande del numero di target `N`. L'insieme `{1, 2, ..., p-1}` con la moltiplicazione modulo `p` è un gruppo ciclico di ordine `p-1`. Una **radice primitiva** `g` è un generatore di quel gruppo: la sequenza

```
  g^1, g^2, g^3, ...  (mod p)
```

tocca ciascuno dei `p-1` residui non nulli esattamente una volta prima di ripetersi. Quindi il motore mantiene un valore corrente e lo fa avanzare:

```
  current = (current * g) mod p        one multiply, one modulo, per target
```

Ogni `current` è un intero distinto in `[1, p-1]`. Se capita che superi `N` (ci sono solo una manciata di valori del genere, dato che `p` è appena sopra `N`), il motore semplicemente avanza di nuovo senza emettere nulla, e questo è il re-roll. Ogni valore in `[1, N]` viene prodotto esattamente una volta, in un ordine che sembra casuale perché la moltiplicazione per una radice primitiva rimescola la sequenza.

Codificare sia l'indirizzo sia la porta in un unico elemento è una divisione. Per l'indice di target `idx` in `[1, N]`, sia `idx0 = idx - 1`, allora:

```
  ip_position   = idx0 / number_of_ports
  port_position = idx0 % number_of_ports
```

così un singolo attraversamento del gruppo randomizza l'intero spazio indirizzi-e-porte in una volta sola, non porta per porta.

Trovare una radice primitiva senza forza bruta è la parte ingegnosa, in `numtheory.findPrimitiveRoot`. Un candidato `g` genera l'intero gruppo se e solo se, per ogni fattore primo distinto `q` di `p-1`, `g^((p-1)/q) mod p` non è uno. Se una qualsiasi di quelle potenze è uno, `g` genera solo un sottogruppo proprio e viene scartato. Quindi il codice fattorizza `p-1` una volta con `distinctPrimeFactors`, poi testa candidati casuali con `modExp` e `isPrimitiveRoot` finché uno non supera il test. Ogni scansione ottiene un generatore nuovo, quindi ogni scansione è una permutazione diversa.

zingela differisce qui da zmap in un modo che vale la pena dichiarare. zmap ha in hardcode undici coppie primo e radice primitiva, una per ogni dimensione di spazio supportata, la più grande delle quali è `2^48 + 23`. zingela chiama `smallestPrimeAbove(N)` per la dimensione esatta della tua scansione e trova un generatore a runtime. Una scansione di un `/24` su una porta usa un primo vicino a 256, non un primo vicino a `2^48`, quindi non spreca iterazioni a ritirare valori che cadono fuori da un gigantesco gruppo fisso.

## La tabella di dedup

Un host che risponde su due porte, o che risponde due volte a una probe ritrasmessa, deve essere contato una volta. `dedup.zig` è un hash set a indirizzamento aperto di chiavi a 64 bit con linear probing.

La chiave impacchetta indirizzo e porta: `(ip << 16) | port`. Il suo valore massimo è `0x0000FFFFFFFFFFFF`, che non può mai essere uguale alla sentinella di slot vuoto `0xFFFFFFFFFFFFFFFF`, quindi la sentinella è sicura.

L'hash è un finalizer fmix64:

```
  x ^= x >> 33
  x *= 0xff51afd7ed558ccd
  x ^= x >> 33
```

Questo distribuisce la chiave a bassa entropia su tutti i 64 bit così che indirizzi sequenziali non si raggruppino in un'unica catena di probing. `insert` calcola lo slot dall'hash mascherato sulla dimensione della tabella, poi procede linearmente in avanti finché non trova la chiave (un duplicato, restituisce false) o uno slot vuoto (inserisce, restituisce true). La tabella cresce e fa rehash quando supera i sette decimi di riempimento, il che mantiene corte le catene di probing. Poiché la tabella è di proprietà dell'unico motore di ricezione, non ha bisogno di lock.

## Il token bucket

`ratelimit.zig` cadenza la trasmissione con una banca di nanosecondi anziché con una sleep. `init` calcola un costo per token `step_ns = 1_000_000_000 / rate` e una capacità per brevi burst. `takeBatch` legge il clock monotono, aggiunge alla banca i nanosecondi trascorsi dall'ultima chiamata fino al tetto, e restituisce quanti token interi (`bank / step_ns`) il mittente può spendere ora, detraendoli. Il mittente non fa mai busy-waiting: chiede un batch, ne invia altrettanti e chiede di nuovo. `refund` restituisce i token per gli invii che non è stato possibile completare, e `withJitter` perturba il passo per una temporizzazione di Poisson. Questo è un vero token bucket, che è un meccanismo diverso dal controller a feedback proporzionale di masscan su un ring di timestamp a 256 slot.

## Prestazioni misurate

Queste sono le funzioni dell'hot path cronometrate da `zig build bench` su un Intel Core i7-14700KF, single core, ReleaseFast. Ogni misurazione varia il proprio input a ogni iterazione e incorpora il risultato in un valore stampato così che l'ottimizzatore non possa eliminare il lavoro. I numeri scalano con la CPU; esegui il bench sulla tua macchina.

| Operazione | ns/op | Rate |
|---|---|---|
| Checksum RFC 1071, scalare, 40 B | 6.0 | 6.7 GB/s |
| Checksum RFC 1071, scalare, 1500 B | 185 | 8.1 GB/s |
| Checksum RFC 1071, SIMD, 1500 B | 30.4 | 49 GB/s |
| Generazione cookie SipHash | 14.0 | 71 M/s |
| Convalida cookie SipHash | 14.8 | 68 M/s |
| Stamp del frame SYN, frame completo | 22.9 | 44 M/s |
| Passo della permutazione a gruppo ciclico | 4.6 | 216 M/s |
| Insert di dedup | 15.4 | 65 M/s |

Ci sono due cose che vale la pena leggere in questa tabella.

Primo, il rate di stamp dei frame rispetto al tetto. Un singolo core timbra circa 44 milioni di frame SYN completi al secondo. Il percorso di trasmissione del kernel `AF_PACKET` smaltisce da 1,5 a 2,5 milioni di pacchetti al secondo. Quindi la CPU costruisce frame circa venti volte più velocemente di quanto il kernel riesca a inviarli. Lo scanner non è mai CPU-bound su hardware standard. Questa è l'intera base del posizionamento onesto: poiché ogni scanner incontra lo stesso tetto del kernel, non c'è alcuna vittoria di throughput grezzo da rivendicare lì, e l'unica vera strada verso più pacchetti al secondo è scavalcare il kernel con `AF_XDP`.

Secondo, il checksum SIMD è più veloce di quello scalare solo su buffer grandi. Su un header da 40 byte la preparazione vettoriale non si ripaga, quindi vince lo scalare. Su un frame da 1500 byte la versione vettoriale è circa sei volte più veloce. Lo scanner timbra header piccoli, quindi `template.stamp` usa il `checksum` scalare, e il benchmark rende visibile questa scelta anziché lasciarla presunta.
