# Sfide di estensione

Idee per estendere questo progetto, ordinate per difficoltà. Ognuna insegna una competenza diversa. Non sentirti obbligato a farle in ordine.

## Sfide facili

### 1. Aggiungere il supporto a SHA3

SHA3 (Keccak) è una famiglia di hash completamente diversa da SHA2, basata su una costruzione a spugna (sponge construction) invece che su Merkle-Damgard. OpenSSL lo supporta attraverso la stessa EVP API.

**Cosa costruire:** aggiungi gli hasher SHA3-256 e SHA3-512.

**Cosa imparerai:** quanto poco codice richiede un nuovo algoritmo quando l'architettura è giusta. Il template EVPHasher fa sì che questa sia una modifica di due righe più la logica di rilevamento.

**Suggerimenti:**
- `EVP_sha3_256()` ed `EVP_sha3_512()` sono le funzioni di OpenSSL
- SHA3-256 produce un digest esadecimale di 64 caratteri (stessa lunghezza di SHA256), quindi l'auto-rilevamento basato sulla sola lunghezza non li distinguerà. Ti servirà un flag `--type sha3-256`
- Scrivi test rispetto a vettori noti dalla suite di test NIST SHA3

### 2. Cracking di hash in batch

Al momento lo strumento cracca un hash alla volta. I dump reali dei breach hanno milioni di hash.

**Cosa costruire:** accetta un file di hash (uno per riga) e craccali tutti in una singola esecuzione. Riporta quali sono stati craccati e quali no.

**Cosa imparerai:** ammortizzare le letture del dizionario su più target. Invece di rileggere la wordlist per ogni hash, calcoli l'hash di ogni candidato una volta e lo confronti con tutti i target simultaneamente.

**Suggerimenti:**
- Carica tutti gli hash target in un `std::unordered_set<std::string>`
- Per ogni candidato, calcolane l'hash e controlla `targets.count(hash_result)`
- È così che funzionano i veri strumenti di cracking. Craccare 1000 hash è a malapena più lento che craccarne 1

### 3. Visualizzazione colorata del tipo di hash

Durante l'auto-rilevamento, mostra all'utente quale tipo è stato rilevato prima che inizi il cracking.

**Cosa costruire:** aggiungi una riga colorata al banner che mostra l'algoritmo rilevato con un indicatore di confidenza.

**Cosa imparerai:** design della UI del terminale, sequenze di escape ANSI e il problema dell'ambiguità (SHA256 e SHA3-256 hanno la stessa lunghezza di digest).

**Suggerimenti:**
- Usa le costanti di colore esistenti in Config.hpp
- Considera di mostrare "SHA256 (auto-detected)" vs "SHA256 (specified)" così l'utente sa quale percorso è stato preso

## Sfide intermedie

### 4. Formato di file di regole personalizzato

L'attuale set di regole è hardcoded. I veri strumenti di cracking come hashcat e john supportano file di regole in cui gli utenti definiscono i propri pattern di mutazione.

**Cosa costruire:** un parser di file di regole che legge le regole da un file di testo:
```
:           # do nothing (try the word as-is)
c           # capitalize first letter
u           # uppercase all
l           # lowercase all
r           # reverse
$[0-9]      # append digit
^[0-9]      # prepend digit
sa@         # substitute a with @
se3         # substitute e with 3
```

**Cosa imparerai:** il parsing di linguaggi, il formato delle regole di hashcat (che è un vero standard di settore) e come la composizione delle regole crea conteggi esponenziali di candidati.

**Suggerimenti:**
- Inizia con codici di regola a singolo carattere, poi aggiungi regole parametriche come `$N` e `sXY`
- La documentazione del motore di regole di hashcat descrive la sintassi completa
- Un file di regole con 50 regole applicate a una wordlist da 10K produce 500K candidati. È comunque veloce

### 5. File di progresso per il cracking riprendibile

Se stai facendo brute force su una password di 8 caratteri e la tua macchina crasha al 60% del progresso, perdi tutto quel lavoro.

**Cosa costruire:** salva periodicamente il progresso (indice corrente, tempo trascorso, candidati testati) in un file. Al riavvio con `--resume`, riprendi da dove ti eri fermato.

**Cosa imparerai:** il checkpointing, le scritture atomiche su file (scrivi su un temp, poi rinomina) e l'importanza di un partizionamento del lavoro deterministico (il nostro brute force basato su indici rende tutto questo facile poiché ogni indice mappa esattamente su un candidato).

**Suggerimenti:**
- Per il brute force, salva l'indice piatto corrente. È tutto ciò che ti serve
- Per il dizionario, salva l'offset in byte nel file
- Scrivi il checkpoint ogni N secondi, non ogni N candidati (l'I/O è costoso rispetto all'hashing)

### 6. Mask Attack

Un mask attack è un brute force più intelligente. Invece di provare tutti i caratteri in ogni posizione, specifichi un pattern: `?u?l?l?l?d?d?d?d` significa maiuscola, tre minuscole, quattro cifre. Questo corrisponde a password come `Pass1234`.

**Cosa costruire:** un flag `--mask` che accetta la sintassi delle mask in stile hashcat:
```
?l = lowercase    ?u = uppercase
?d = digit        ?s = special
?a = all          A  = literal 'A'
```

**Cosa imparerai:** il massiccio guadagno di efficienza derivante da spazi di ricerca vincolati. `?u?l?l?l?d?d?d?d` è 26*26*26*26*10*10*10*10 = 4,5 miliardi di candidati. Il brute force completo di 8 caratteri dallo stesso set è 218 trilioni. È una riduzione di 48.000 volte.

**Suggerimenti:**
- Fai il parsing della mask in un vector di set di caratteri, uno per posizione
- Il calcolo del key space diventa `prodotto della dimensione del charset di ogni posizione`
- Il partizionamento funziona allo stesso modo del brute force (indice piatto, decomposizione a radix misto)

## Sfide avanzate

### 7. Generatore e lookup di Rainbow Table

Le rainbow table sono catene di hash precalcolate che scambiano spazio su disco con tempo di cracking. Invece di calcolare l'hash di ogni candidato a runtime, costruisci una tabella offline e fai un lookup.

**Cosa costruire:** due modalità: `--generate-table` crea un file di rainbow table per un dato charset e lunghezza, `--rainbow` cracca usando una tabella precalcolata.

**Cosa imparerai:** il compromesso tempo-memoria nella crittoanalisi, le reduction function, la costruzione delle catene e perché i salt rendono inutili le rainbow table. Questo è uno degli attacchi più eleganti in tutta la sicurezza informatica.

**Suggerimenti:**
- Una rainbow table non memorizza ogni hash. Memorizza catene: punti di inizio e punti di fine. Ogni catena copre migliaia di hash
- La reduction function converte un hash di nuovo in un candidato (non un'inversione dell'hash, solo una mappatura deterministica)
- La lunghezza della catena controlla il compromesso: catene più lunghe = tabella più piccola ma lookup più lento
- Inizia con un esempio piccolo (4 caratteri minuscoli) per verificare la correttezza prima di scalare
- Il paper originale del 1980 di Martin Hellman descrive il concetto. Il paper del 2003 di Philippe Oechslin introduce il miglioramento "rainbow"

### 8. Cracking accelerato su GPU con CUDA

L'opzione nucleare. Sposta il loop di hash-and-compare sulla GPU.

**Cosa costruire:** un kernel CUDA che calcola gli hash dei candidati in parallelo sulla GPU. La CPU genera i candidati e carica i batch; la GPU calcola migliaia di hash simultaneamente.

**Cosa imparerai:** la programmazione GPU, il design dei kernel CUDA, la gestione della memoria host-device e perché le GPU sono molto più veloci nel calcolo parallelo (migliaia di core semplici contro pochi core complessi).

**Suggerimenti:**
- Non puoi usare OpenSSL sulla GPU. Implementa SHA256 in puro CUDA (l'algoritmo è pubblico, circa 100 righe di codice del kernel)
- Carica i candidati in batch (ad esempio, 1 milione alla volta) per ammortizzare il costo del trasferimento host-to-device
- Usa `cudaMemcpyAsync` con gli stream per sovrapporre calcolo e trasferimento
- Inizia solo con SHA256. Far funzionare un algoritmo su GPU è un risultato significativo
- Questo potrebbe essere un proprio progetto avanzato autonomo nel repository

## Sfida esperta

### 9. Cracking distribuito sulla rete

Suddividi il key space su più macchine. Un coordinatore assegna i range di lavoro; i worker calcolano gli hash e riferiscono.

**Cosa costruire:** un coordinatore che accetta connessioni dai nodi worker, assegna range di key space, raccoglie i risultati e gestisce i fallimenti dei worker (riassegnando il loro range a un altro worker).

**Cosa imparerai:** i fondamenti dei sistemi distribuiti: distribuzione del lavoro, heartbeating, fault tolerance e il pattern del coordinatore. Questa è la stessa architettura che usano le operazioni di password cracking su larga scala.

**Suggerimenti:**
- Usa socket TCP o gRPC per la comunicazione
- Il coordinatore divide il key space totale in chunk e li assegna su richiesta
- I worker inviano heartbeat periodici con il loro progresso. Se un worker diventa silenzioso, il coordinatore riassegna il suo chunk
- Pensa a cosa succede se due worker affermano di aver craccato lo stesso hash (il coordinatore dovrebbe gestire i risultati duplicati in modo controllato)
- Questo si abbina bene con la Sfida 8 (ogni worker potrebbe essere accelerato su GPU)

## Sfide di prestazioni

### 10. Suite di benchmark

Quanto è veloce lo strumento in realtà? Confronta diversi algoritmi di hash, configurazioni di threading e modalità di attacco.

**Cosa costruire:** una modalità benchmark (`--benchmark`) che esegue test standardizzati e riporta i risultati:
```
SHA256 dictionary (10K words):  2.4M h/s
SHA256 brute force (6 chars):   2.1M h/s
MD5 dictionary (10K words):     3.8M h/s
SHA512 dictionary (10K words):  1.9M h/s
Threads: 1=600K  2=1.2M  4=2.3M  8=2.4M
```

**Cosa imparerai:** la metodologia del microbenchmarking, perché i risultati variano tra le esecuzioni e dove sono i veri colli di bottiglia (indizio: è OpenSSL, non il tuo codice).

**Suggerimenti:**
- Usa `std::chrono::steady_clock` per la misurazione dei tempi
- Esegui ogni benchmark più volte e riporta la mediana, non la media
- Vincola i thread a core specifici con `pthread_setaffinity_np` per risultati consistenti
- Confronta i tuoi numeri con i benchmark di hashcat per vedere la differenza tra CPU e GPU

## Come ottenere aiuto

Se ti blocchi su una qualsiasi sfida:

1. Leggi il codice sorgente rilevante. L'architettura è progettata in modo che ogni componente sia comprensibile in isolamento
2. Scrivi prima un test che fallisce. Se riesci a descrivere il comportamento atteso in un test, l'implementazione diventa più chiara
3. Inizia in piccolo. Fai funzionare la versione più semplice possibile, poi aggiungi complessità
4. Controlla la documentazione di hashcat per un precedente del mondo reale. La maggior parte di queste sfide sono versioni semplificate di feature che gli strumenti di cracking di produzione già implementano
