<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# Concetti

Questo documento spiega le idee su cui è costruito zingela e ancora ciascuna a un evento reale, così che la posta in gioco sia concreta.

## Il three-way handshake, e cosa può apprendere una probe

Una connessione TCP si apre con tre pacchetti. Il client invia un SYN. Se la porta è aperta, il server risponde con un SYN-ACK. Il client completa la connessione con un ACK. Se la porta è chiusa, il server risponde al SYN con un RST. Se un firewall sta scartando il traffico, non torna indietro nulla.

Un SYN scan stateless invia solo il primo pacchetto e legge la risposta:

```
  zingela                target
    | ----- SYN -------->  |   open   ->  SYN-ACK comes back
    | ----- SYN -------->  |   closed ->  RST comes back
    | ----- SYN -------->  |   filtered -> silence
```

zingela non invia mai l'ACK finale, quindi la connessione non viene mai stabilita e l'applicazione target non vede mai una sessione completata. Ecco perché un SYN scan è veloce e leggero. Ed ecco anche perché la classificazione ha esattamente tre esiti: open (SYN-ACK), closed (RST) e filtered (nessuna risposta entro la finestra di attesa). UDP è più torbido, e riceve più avanti una trattazione onesta tutta sua.

## Statelessness: porta lo stato nel pacchetto

L'idea centrale. Uno scanner stateful ricorda ogni probe inviata per poter abbinare una risposta. Uno scanner stateless non ricorda nulla e invece fa sì che ogni risposta dimostri la propria legittimità.

zingela lo fa con un hash con chiave chiamato SYN cookie. Prima di inviare una probe a un target, calcola un valore a partire dalla quadrupla (indirizzo sorgente, porta sorgente, indirizzo di destinazione, porta di destinazione) usando SipHash con chiave derivata da entropia casuale per singola scansione, e scrive i 32 bit bassi nel campo del numero di sequenza TCP. Un target reale che risponde a un SYN restituisce quel numero di sequenza più uno nel campo acknowledgement del suo SYN-ACK. Quando arriva la risposta, zingela ricalcola lo stesso hash dagli indirizzi e dalle porte del pacchetto e verifica che l'acknowledgement sia uguale al cookie più uno. Se corrisponde, la risposta è genuina. Se non corrisponde, il pacchetto è vagante, spoofato o un'eco obsoleta, e viene scartato a costo zero.

Il guadagno è enorme. Non c'è una tabella delle connessioni, quindi l'uso di memoria è costante indipendentemente da quanti miliardi di target scansioni. Il motore di trasmissione e il motore di ricezione non condividono alcuno stato per probe, quindi girano come thread indipendenti alla massima velocità. Le risposte spoofate vengono rifiutate dall'aritmetica anziché da una regola di firewall.

Il costo è sottile e vale la pena capirlo. Poiché lo scanner non conserva alcuna traccia di ciò che ha inviato, non può sapere che una probe è andata persa, quindi la perdita appare identica a una porta filtered. Gli scanner stateless seri affrontano il problema con la ritrasmissione incondizionata (inviare ogni probe alcune volte) anziché con il tracciamento delle risposte, perché il tracciamento reintrodurrebbe lo stato che hai faticato tanto a eliminare.

## I SYN cookie sono nati da un attacco reale

L'idea del cookie non è un'invenzione degli scanner. Era la difesa ideata da Daniel Bernstein nel 1996 contro gli attacchi denial-of-service di tipo SYN flood, in cui un attaccante invia valanghe di SYN e non completa mai l'handshake, esaurendo la tabella delle connessioni semi-aperte del server. La risposta di Bernstein fu smettere di memorizzare le connessioni semi-aperte e codificare invece lo stato necessario nel numero di sequenza stesso, convalidandolo al ritorno dell'ACK. Uno scanner stateless esegue lo stesso trucco dall'altro lato del cavo: è lo scanner, non il server, la parte che si rifiuta di mantenere lo stato.

## Percorrere lo spazio degli indirizzi senza una lista

Per scansionare un intervallo devi visitare ogni indirizzo al suo interno, idealmente in ordine rimescolato così da non martellare una subnet alla volta facendo scattare il suo sistema di intrusion detection. Il modo ingenuo è costruire una lista, mescolarla e iterare. Alla scala di internet, la sola lista occupa decine di gigabyte.

zingela prende in prestito la risposta di zmap: un gruppo ciclico moltiplicativo su un primo appena superiore alla dimensione dello spazio dei target. Mantiene tre numeri (un valore corrente, un generatore e il primo) e avanza con una moltiplicazione e un modulo per target. Poiché il generatore è una radice primitiva del primo, la sequenza visita ogni elemento dello spazio esattamente una volta prima di tornare all'inizio. È un mescolamento perfetto con memoria O(1): nessuna lista, nessuna bitmap del lavoro rimanente, nessuna chiamata a un generatore di numeri casuali per indirizzo.

masscan risolve lo stesso problema con un block cipher (una rete di Feistel che chiama BlackRock2). La retrospettiva del 2024 "Ten Years of ZMap" ne ha misurato la conseguenza: masscan trova sensibilmente meno host di zmap, probabilmente a causa di distorsioni in quella randomizzazione. zingela sceglie la strada del gruppo ciclico proprio per evitare quella distorsione. La [sezione sul gruppo ciclico di MECHANICS](MECHANICS.it.md) mostra la matematica.

## Rate limiting, e perché il default è lento

zingela usa di default 10.000 pacchetti al secondo, e avvisa in modo vistoso prima che tu spinga verso i milioni. Quella prudenza è deliberata, ed è una lezione scritta a suon di interruzioni di servizio.

Il 25 gennaio 2003 il worm SQL Slammer (che sfruttava la CVE-2002-0649 in Microsoft SQL Server sulla porta UDP 1434) dimostrò lo scanning stateless nella sua forma più distruttiva. Era un singolo pacchetto UDP da 376 byte che, una volta infettato un host, iniziava immediatamente a sparare copie verso indirizzi casuali alla massima velocità consentita dalla scheda di rete. Infettò circa 75.000 host in dieci minuti, raddoppiando all'incirca ogni 8,5 secondi, e il puro volume del traffico di scansione saturò i collegamenti e mise offline parti di internet. Slammer non manteneva alcuno stato e non limitava nulla. È il gemello ammonitore di ogni scanner stateless.

La lezione è che un tool in grado di emettere pacchetti a line rate è un tool che può inondare una rete per sbaglio. I default aggressivi dello stesso masscan gli sono costati il null-routing da parte degli ISP. zingela usa un token bucket per cadenzare la trasmissione e fa dello scanning veloce una scelta esplicita e accompagnata da avvisi, anziché il default.

## Gli intervalli riservati sono esclusi per costruzione

Alcuni blocchi di indirizzi non devono mai essere scansionati: loopback, lo spazio privato RFC 1918, link-local, multicast e gli intervalli di documentazione. zingela controlla una tabella degli intervalli riservati prima di costruire qualsiasi pacchetto, e quella soglia di esclusione non può essere scavalcata da alcun flag. Se lo punti su un intervallo che si sovrappone a uno spazio riservato, la porzione riservata viene sottratta e semplicemente non viene mai sondata. Ecco perché puntare lo scanner su `10.0.0.0/8` produce zero target: l'intero blocco è spazio privato RFC 1918 e viene rimosso prima del primo pacchetto.

## Lo spoofing della sorgente è protetto, non di default

Puoi falsificare l'IP sorgente di una scansione. Farlo di default sarebbe sconsiderato, e funziona a malapena: solo circa un quinto dei sistemi autonomi su internet permette egress spoofato, perché la maggior parte implementa il filtraggio anti-spoofing descritto in BCP 38. L'uso legittimo è testare la conformità a BCP 38 della tua rete. zingela tratta quindi lo spoofing e ogni altra funzionalità di evasion come protetti da un riconoscimento esplicito `--authorized-scan`, mai come un default silenzioso.

## Il confine legale

Lo scanning invia pacchetti non richiesti a macchine che potresti non possedere. Negli Stati Uniti questo può ricadere sotto il Computer Fraud and Abuse Act, la stessa legge del 1986 in base alla quale fu condannato Robert Tappan Morris dopo che il suo worm del 1988 scansionò e infettò circa 6.000 macchine, all'incirca un decimo dell'internet di allora. I tribunali hanno trattato lo scanning non autorizzato in modo incoerente, ma la regola sicura è semplice e assoluta: scansiona solo sistemi che possiedi o che hai il permesso scritto esplicito di testare.

Le stesse tecniche servono alla difesa. Quando Heartbleed (CVE-2014-0160) esplose nell'aprile 2014, i ricercatori usarono zmap per scansionare ripetutamente l'intera internet, misurando quanti server fossero vulnerabili e quanto velocemente gli operatori applicassero le patch. La misurazione a livello di internet, il tracciamento delle vulnerabilità e l'inventario degli asset sono esattamente ciò a cui servono questi tool, quando hai l'autorità per eseguirli. I default di zingela, le sue protezioni e la sua soglia di esclusione esistono per mantenere il tool dal lato giusto di quel confine.

## Perché uno scanning aggressivo ti fa notare

La botnet Mirai del 2016 scansionava le porte TCP 23 e 2323 sull'intero spazio IPv4, provando circa 60 credenziali telnet di default, e mise insieme una botnet abbastanza grande da abbattere Krebs on Security, OVH e il provider DNS Dyn nell'ottobre 2016. Era rumorosa, veloce e indiscriminata, ed era banalmente rilevabile proprio per questo. La suite stealth di zingela esiste affinché i tester autorizzati possano studiare la rilevabilità (jitter, decoy, fingerprint realistici del sistema operativo, tipi di scansione alternativi), non perché chiunque possa essere più silenzioso mentre fa danni. Ecco perché è protetta. Vedi [04-CHALLENGES.md](04-CHALLENGES.it.md) per gli argomenti di evasion proposti come esercizi di studio.
