<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# Nadezhda: Concetti

Questo documento spiega le idee su cui si basa il progetto. Se leggi un solo file per capire *perché* nadezhda funziona così, leggi questo. Ogni concetto qui si collega a un package che incontrerai in [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) e a una funzione che leggerai in [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md).

## I feed sono un contratto, lo scraping è un ripiego

I publisher offrono feed RSS e Atom come promessa leggibile da una macchina: ecco i miei ultimi elementi, strutturati, con un titolo, un link, un orario di pubblicazione e un riassunto. Nadezhda prende questa promessa alla lettera e tratta i feed come percorso primario. Ricorre al parsing dell'HTML grezzo solo quando una fonte non offre alcun feed, e anche in quel caso lo fa per singola fonte e dietro un interruttore.

Questa scelta ha una componente etica. Recuperare un feed configurato da un utente è un recupero guidato dall'utente, nella stessa categoria di un browser che carica una pagina o di un feed reader che controlla gli aggiornamenti. Fare scraping di HTML arbitrario su un intero sito è comportamento da crawler. I due casi meritano regole diverse, e nadezhda applica questa differenza: robots.txt viene rispettato nel percorso di scraping degli articoli HTML e deliberatamente non viene applicato al recupero dei feed sottoscritti. Diversi publisher, The Register tra questi, vietano in blocco i bot generici in robots.txt pur servendo un feed pubblico proprio per questo tipo di lettore. Rispettare robots sul feed significherebbe rifiutare un documento che il publisher ha costruito esplicitamente per te.

I sette feed iniziali (Krebs on Security, The Hacker News, BleepingComputer, SecurityWeek, Dark Reading, The Register e CISA) sono stati tutti verificati dal vivo. Una cosa utile che impari subito: solo due dei sette includono nel feed il corpo completo dell'articolo. Gli altri cinque contengono solo il riassunto. Questo limita il numero di CVE che puoi estrarre, perché puoi trovare un CVE ID solo nel testo che il feed ti fornisce davvero.

## Lo stack di intelligence sulle CVE

Un identificativo CVE come CVE-2021-44228 è solo un nome. Dice "questa specifica vulnerabilità esiste" e nulla su quanto sia grave, se venga sfruttata o quanto sia probabile che lo sia. Quattro diverse fonti di dati rispondono a queste domande, e ciascuna risponde a una domanda diversa. Capire a cosa serve ciascuna è il cuore di questo progetto.

### CVSS: quanto è grave, in teoria

Il Common Vulnerability Scoring System trasforma le caratteristiche di una vulnerabilità in un numero da 0.0 a 10.0 e in una fascia (LOW, MEDIUM, HIGH, CRITICAL). Esistono diverse versioni in circolazione. La versione 2 è vecchia. Le versioni 3.0 e 3.1 sono le più comuni. La versione 4.0 è arrivata nel 2023 e si sta diffondendo lentamente. Un singolo record CVE può riportare un punteggio in più versioni, in una sola versione o in nessuna.

Nadezhda risolve il problema con una precedenza fissa: preferisce v4.0, poi v3.1, poi v3.0, poi v2.0. Più recente è meglio, e l'assenza di un punteggio è una possibilità reale che il codice gestisce invece di darla per esclusa. Per questo il punteggio memorizzato è un valore nullable e non un semplice numero.

La parte sottile è *dove* si trova il punteggio. Nel record del CVE Program per **Log4Shell (CVE-2021-44228)**, la metrica CVSS non si trova nel container del vendor. Il container di Apache contiene solo una metrica segnaposto `other`. Il vero punteggio 10.0 si trova in un secondo container aggiunto dal programma di enrichment di CISA. Un parser che guarda solo il container del vendor riporta "nessun punteggio" per la vulnerabilità più famosa del decennio. Al contrario, in **Citrix Bleed 2 (CVE-2025-5777)** il punteggio CVSS 4.0 di 9.3 si trova nel container del vendor, dove ti aspetteresti di trovarlo. La lezione è guardare in entrambi i posti e applicare la precedenza a tutto ciò che trovi.

### CWE: che tipo di debolezza è

Dove CVSS assegna un punteggio alla gravità, la Common Weakness Enumeration classifica il *tipo* di difetto. CVE-2025-5777 è CWE-125, una lettura fuori dai limiti (out of bounds read). Log4Shell è CWE-502, deserialization di dati non attendibili. CWE è ciò che ti permette di dire "continuiamo a rilasciare la stessa classe di bug" attraverso molte CVE, ed è un asse naturale su cui filtrare o raggruppare.

### CISA KEV: viene sfruttata in questo momento

Il segnale più azionabile dell'intero stack è il catalogo CISA Known Exploited Vulnerabilities. Una CVE nel catalogo KEV non è un rischio teorico. CISA la inserisce perché ha prove di sfruttamento attivo contro bersagli reali. Un CVSS 6.5 nel catalogo KEV merita spesso la tua attenzione prima di un CVSS 9.8 che nessuno ha toccato.

Il catalogo è un singolo download JSON, il che lo rende economico da recuperare una volta per esecuzione e da confrontare. Contiene un campo che fa inciampare quasi tutti: `knownRansomwareCampaignUse` non è un booleano. È la stringa `"Known"` o `"Unknown"`. Trattalo come un booleano e ogni voce sembra uguale. Mappa la stringa in modo esplicito e recuperi un segnale davvero utile, perché una voce KEV con uso noto in campagne ransomware è urgente quanto può esserlo una notizia di sicurezza.

### FIRST EPSS: quanto è probabile lo sfruttamento

L'Exploit Prediction Scoring System, pubblicato da FIRST, è una probabilità aggiornata quotidianamente, tra 0 e 1, che una data CVE venga sfruttata nei prossimi 30 giorni, più un percentile che la colloca rispetto a ogni altra CVE valutata. Dove KEV ti dice che lo sfruttamento è già in corso, EPSS ti dice che sta arrivando. Il punteggio EPSS di Log4Shell era vicinissimo al massimo della scala, come ti aspetteresti per un difetto che tutta internet stava cercando con scanning.

EPSS ha la sua trappola. Nella risposta dell'API, i valori `epss` e `percentile` sono *stringhe* JSON, non numeri. Fanne il parsing come float, altrimenti il punteggio viene letto silenziosamente come zero, il che disattiva di nascosto uno dei tuoi segnali di ranking senza alcun errore a segnalartelo.

## Senza chiave per impostazione predefinita

Puoi ottenere tutto questo senza una API key. Il nucleo delle CVE proviene dal repository cvelistV5 del CVE Program, che pubblica ogni record come JSON grezzo a un URL prevedibile e suddiviso in bucket. CISA KEV è un download pubblico. FIRST EPSS è una API pubblica. Nessuna di queste vincola i dati a una chiave.

La NVD API 2.0 è la fonte a cui ricorrono la maggior parte dei tutorial, e nadezhda la supporta, ma come potenziamento opzionale anziché come requisito. Senza chiave è soggetta a rate limiting di cinque richieste ogni trenta secondi, il che è penoso per un batch. Con una chiave è più veloce. La decisione di design è stata di rendere il trio senza chiave l'impostazione predefinita, in modo che lo strumento sia pienamente utile appena installato, e di lasciare che un utente con una chiave NVD la abiliti. Uno strumento il cui valore principale è bloccato dietro un passaggio di configurazione della chiave è uno strumento che la maggior parte delle persone non finisce mai di configurare.

## Il clustering trasforma la copertura in velocity

Se BleepingComputer, The Hacker News e Krebs pubblicano tutti sullo stesso difetto nello stesso pomeriggio, è un segnale più forte di ciascuno di loro da solo. Misurarlo richiede di riconoscere che tre titoli formulati in modo diverso descrivono una sola storia. Questo è il clustering.

Nadezhda fa clustering con un approccio a componenti connesse (union-find). Due elementi entrano nello stesso cluster quando rientrano in una finestra temporale (72 ore per impostazione predefinita) e condividono un CVE ID oppure, se provengono da testate diverse, hanno titoli sufficientemente simili secondo la sovrapposizione di Jaccard sull'insieme di token. La condizione di testate diverse sulla corrispondenza dei titoli è deliberata. Raggruppare due elementi con lo stesso titolo dalla *stessa* fonte fonde i post di follow-up dello stesso publisher, il che è rumore. Richiedere che la corrispondenza dei titoli attraversi testate diverse è ciò che ha eliminato un vero falso positivo durante lo sviluppo, in cui due distinti advisory CISA avevano per caso un titolo generico in comune.

La dimensione e il tasso di crescita di un cluster diventano il segnale di *velocity* nel ranking. Una storia ripresa da otto testate in sei ore si classifica sopra una storia pubblicata da una testata un giorno fa, anche se la seconda ha una CVE più spaventosa.

## Il ranking è deterministico e mette le notizie al primo posto

Il punteggio finale di un cluster è una somma pesata di segnali normalizzati:

```
score =  w_recency  * recency_decay(age)
       + w_velocity * normalized(cluster_size / age)
       + w_kev      * is_kev
       + w_cvss     * normalized(max_cvss)
       + w_epss     * max_epss
       + w_source   * source_weight
       + w_keyword  * keyword_match(watchlist)
```

`recency_decay` è una funzione esponenziale a emivita, quindi una storia perde metà del suo peso di recency ogni numero configurato di ore. Ogni peso vive nella configurazione, non nel codice, quindi non ci sono numeri magici da cercare e il modello è completamente regolabile.

I pesi predefiniti mettono le notizie al primo posto: recency, velocity, source e keyword insieme valgono il 70 percento, e i segnali CVE (KEV, CVSS, EPSS) il restante 30. Questo riflette una precisa decisione di prodotto. Nadezhda è uno strumento di *news* di sicurezza. Una breach o una campagna senza alcuna CVE non dovrebbe essere sepolta sotto una nota di patch di routine solo perché la nota di patch ha un numero associato. I dati CVE sono intelligence che arricchisce una storia, non il motivo per cui la storia conta.

Poiché il ranking è una funzione pura degli input memorizzati, lo stesso corpus produce sempre lo stesso ordine. Questa proprietà è ciò che permette al progetto di verificare il proprio ranking con golden order test: fornisci input fissi, aspettati un ordinamento esatto.

## Riferimenti

- **MITRE CVE**: il sistema di identificazione. https://www.cve.org
- **CVE Program cvelistV5**: la fonte dei record senza chiave. https://github.com/CVEProject/cvelistV5
- **CVSS**, pubblicato da FIRST: il sistema di scoring e le sue versioni. https://www.first.org/cvss/
- **CWE**, MITRE: la tassonomia delle debolezze. https://cwe.mitre.org
- **CISA KEV**: il catalogo di quanto viene sfruttato in the wild. https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- **FIRST EPSS**: probabilità di sfruttamento. https://www.first.org/epss/
- **NVD API 2.0**: il potenziamento opzionale. https://nvd.nist.gov/developers/vulnerabilities

## Metti alla prova la tua comprensione

- Perché un CVSS 6.5 nel catalogo KEV spesso supera in classifica un CVSS 9.8 che non vi è presente? Cosa cambieresti nei pesi per far sì che accada o che non accada?
- Il progetto estrae meno CVE da cinque dei sette feed che dagli altri due. Perché, e in quale punto della pipeline viene imposto questo limite?
- Se i punteggi EPSS vengono letti come zero a causa del bug di tipizzazione delle stringhe, quale segnale di ranking scompare, e qualche test lo individuerebbe?
- Perché la condizione di similarità dei titoli è limitata alle coppie tra testate diverse? Costruisci un esempio in cui eliminare questa restrizione produce un cluster sbagliato.
- Il punteggio CVSS di Log4Shell non si trova nel container del vendor del suo record CVE. Dov'è, e cosa riporta invece un parser ingenuo?
