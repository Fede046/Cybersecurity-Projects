<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.md -->

# Sfide ed estensioni

Modi per portare zingela più lontano, ordinati grosso modo dal più accessibile al più profondo. Ciascuno indica il codice che tocca e l'idea che insegna. Alcuni sono vere lacune della build attuale, segnalate come tali. Mentre testi uno qualsiasi di questi, scansiona solo ciò che sei autorizzato a scansionare.

## Riscaldamento

### Aggiungi un payload di probe

`udp.zig` contiene una tabella a compile time di payload UDP per protocollo (i byte che fanno rispondere un servizio DNS, NTP o SNMP). Aggiungine uno per un protocollo che non copre ancora, per esempio una richiesta stats di memcached o un handshake IKE. Imparerai perché lo scanning UDP ha bisogno di un payload vero: a differenza di TCP, una probe UDP nuda verso una porta aperta di solito non riceve risposta, quindi il silenzio viene riportato onestamente come open-or-filtered. Un buon payload trasforma alcuni di questi in open certi. È lo stesso motivo per cui il worm SQL Slammer del 2003 fece stare tutto il suo exploit in un unico pacchetto UDP da 376 byte verso la porta 1434: i servizi UDP rispondono al contenuto, non ai tentativi di connessione.

### Estendi il rilevamento dei servizi

`service.zig` esegue una NULL probe e una richiesta HTTP sulle porte aperte e cattura il banner. Aggiungi una probe per un altro servizio comune, per esempio la stringa di versione SSH o un TLS ClientHello che registri il certificato. Nota il confine che il progetto mantiene deliberatamente: rileva TLS ma non calcola un fingerprint JA4, perché un tool Rust dedicato lo fa già. Rispetta quel confine o finirai per duplicare il lavoro.

## Costruisci qualcosa di reale

### Un vero benchmark AF_XDP

Questa è la domanda aperta onesta dell'intero progetto. Il backend di trasmissione `AF_XDP` è disponibile dietro `-Dxdp`, ma il confronto diretto con masscan su hardware 10GbE non è stato misurato. Prepara due macchine con NIC adeguate, esegui zingela con `-Dxdp` verso un sink, misura i pacchetti al secondo sostenuti con `perf` e confronta con masscan sullo stesso hardware con e senza PF_RING. Pubblica il numero reale. Il paper Zippier ZMap ha misurato zmap a 14,23 Mpps contro masscan a 6,4 Mpps su 10G, quindi c'è un obiettivo concreto da riprodurre e battere. Non accettare una cifra su veth o loopback come line rate; quelle misurano il loopback del kernel, non la NIC.

### Avvisi sul rate e un gate rigido

Il design prevede un avviso ben visibile sopra un milione di pacchetti al secondo e un gate di conferma esplicito sopra i dieci milioni, così che lo scanning veloce sia sempre una scelta deliberata. Collega tutto questo a `ratelimit.zig` e al parser degli argomenti. Il motivo è scritto nelle interruzioni di servizio: i default aggressivi di masscan gli sono costati il null-routing da parte degli ISP, e Slammer ha mostrato cosa fa a una rete un mittente stateless senza cadenza. Questo ti insegna dove risiede la responsabilità in un tool che può inondare un collegamento per sbaglio.

### Un checkpoint di ripresa atomico

Una scansione lunga che muore a metà dovrebbe poter essere ripresa. Poiché il motore degli indirizzi è un gruppo ciclico con stato O(1) (un valore corrente, un generatore, un primo e un contatore di passi), l'intero avanzamento di una scansione è una manciata di interi. Scrivili periodicamente su disco in modo atomico e aggiungi un flag per riprendere da quel file. Questo insegna perché la permutazione stateless non è solo veloce ma anche operativamente comoda: puoi fare il checkpoint di una scansione a livello di internet in pochi byte, cosa che una lista di indirizzi mescolata non potrebbe mai fare.

## Vai a fondo

### UDP IPv6 e stealth IPv6

L'attuale supporto IPv6 è solo TCP SYN. Lo scanning UDP e la suite stealth sono solo IPv4. Estenderli a IPv6 significa rielaborare il percorso dei payload UDP e le funzionalità di evasion per portare `packet.Addr` fino in fondo, come già fa il percorso TCP. La chiave di dedup per IPv6 è un hash con perdita dei 144 bit di indirizzo-e-porta ridotti a 64 bit, quindi valuta se la tua estensione ha bisogno di una chiave esatta. È un vero esercizio di architettura su quanto lontano ti porta un tipo union.

### Letterali IPv4-in-IPv6 incorporati

Indirizzi come `::ffff:1.2.3.4` incorporano un indirizzo IPv4 nella notazione IPv6. Il parser in `netutil.zig` non li accetta ancora. Aggiungerli è un compito di parsing piccolo e ben delimitato con casi limite taglienti, un buon modo per imparare il formato testuale IPv6 e per esercitarsi nel property-testing di un parser rispetto alla sua inversa.

### Sharding su più macchine

`Engine.initShard` affetta già il gruppo ciclico in archi contigui in base all'identificatore e al numero di shard, così che più macchine che condividono un seed possano ciascuna scansionare una fetta disgiunta dello stesso spazio senza coordinamento. Costruisci l'orchestrazione attorno a questo: un launcher che avvia N shard, raccoglie il loro NDJSON e unisce i risultati. È così che vengono effettivamente eseguite le scansioni su scala internet, e insegna perché una permutazione deterministica con seed è la primitiva giusta per la distribuzione.

### Un archivio dei risultati e una vista di rilevamento

zingela emette NDJSON sullo standard output. Immettilo in un archivio (SQLite, o un file colonnare) e costruisci delle query sopra: nuovi host dall'ultima scansione, servizi cambiati, porte che si sono aperte. Questo è il caso d'uso di misurazione per cui esiste lo scanning di internet. Quando Heartbleed esplose nel 2014, i ricercatori scansionarono ripetutamente l'intera internet con zmap proprio per tracciare quali server restassero vulnerabili nel tempo. Un archivio dei risultati è ciò che trasforma uno scanner in uno strumento di misura.

## Studia, non trasformare in arma

### Esperimenti di rilevabilità

La suite stealth (jitter, decoy, fingerprint realistici del sistema operativo, tipi di scansione alternativi), protetta da `--authorized-scan`, esiste affinché i tester autorizzati possano studiare come appare una scansione a un sistema di intrusion detection. Allestisci un laboratorio con il tuo IDS, scansiona i tuoi target e misura quali tecniche cambiano il profilo degli alert e quali no. La botnet Mirai del 2016 fu individuata in parte perché il suo scanning era rumoroso e uniforme. Comprendere la rilevabilità è conoscenza difensiva. Usarla per scansionare sistemi che non possiedi non lo è, e la protezione esiste per mantenere netta questa distinzione.
