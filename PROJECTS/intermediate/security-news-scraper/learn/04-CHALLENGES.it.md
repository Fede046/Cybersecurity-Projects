<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.md -->

# Nadezhda: Sfide

Modi per estendere il progetto, ordinati grosso modo da un pomeriggio di lavoro a un'impresa seria. Ognuna indica il codice che andresti a toccare e offre un suggerimento, non una soluzione completa. Il modo migliore per capire una pipeline è aggiungerle uno stadio.

## Riscaldamento

**Aggiungi una fonte.** L'elenco delle fonti è `sources.yaml`, incorporato per impostazione predefinita. Aggiungi una voce con nome, URL del feed, tipo e peso, poi esegui `nadezhda sources` per renderla persistente e `nadezhda scrape` per scaricarla. Scegli un feed che si sovrapponga alle sette fonti esistenti (ad esempio un secondo outlet generalista di sicurezza) e osserva come inizia a unirsi ai cluster. Il `weight` della fonte alimenta il ranking, quindi una fonte di cui ti fidi di più fa salire un po' le sue storie.

**Ritara il ranking.** Ogni peso vive in `config`, sotto `rank.weights`. I valori predefiniti privilegiano le notizie (recency e velocity sono i pesi più alti). Inverti la logica: alza `cvss`, `kev` ed `epss` e abbassa `recency`, esegui lo scrape e confronta la parte alta di `nadezhda digest`. Vedrai la lista riordinarsi verso la gravità pura. Qui non c'è nessuna modifica al codice, ed è proprio il punto: il modello è dato, non logica.

**Aggiungi una watchlist di parole chiave.** Imposta `watchlist` nella config con i vendor e i prodotti che usi davvero. `matchesWatchlist` in `internal/rank` assegna un punteggio più alto a un cluster quando un termine compare nei suoi titoli o nei suoi CVE. Verifica che funzioni aggiungendo un vendor che sai essere nelle notizie di oggi e guardando le sue storie salire.

**Aggiungi un formato di export.** `internal/export` genera Markdown e JSON. Aggiungine un terzo, ad esempio un digest in testo semplice o un feed RSS dei cluster ordinati, così nadezhda può alimentare un altro tool. Mantieni il renderer una funzione pura dei cluster con punteggio, come lo è `export.Markdown`, in modo che resti testabile.

## Livello medio

**Aggiungi un provider AI.** `internal/ai` ha una sola interfaccia `Provider`. Tre provider condividono il client compatibile con OpenAI. Aggiungi un nuovo endpoint compatibile con OpenAI (un runtime locale diverso, o un modello ospitato) collegando il suo URL di base e il modello alla config e alla factory. La versione più difficile è un provider non compatibile con OpenAI, ed è per questo che Anthropic ha il proprio client: scoprirai dove le forme delle risposte differiscono davvero.

**Attiva il percorso di scrape degli articoli HTML.** Il client di fetch ha già `Allowed`, un gate per robots.txt, costruito e testato ma intenzionalmente mai chiamato, perché il percorso dei feed non lo usa. Collegalo a un extractor HTML per singola fonte, per una fonte il cui feed contiene solo il riassunto, così da poter scaricare il corpo completo dell'articolo ed estrarre più CVE. Questo è l'unico punto in cui robots.txt deve essere rispettato, quindi instrada il fetch prima attraverso `Allowed`. Stai costruendo il percorso del crawler che l'architettura ha deliberatamente tenuto separato.

**Rendi il webhook più intelligente.** Il daemon di watch invia un payload JSON `text`/`content`/`items` che funziona per Slack, Discord o un endpoint generico. Aggiungi un vero formatter Slack Block Kit o Discord embed dietro un interruttore nella config, in modo che un alert venga mostrato come una scheda ricca invece che come una riga di testo. L'insieme dei notevoli è già limitato e ordinato, quindi stai formattando, non filtrando.

**Riporta la salute per fonte.** `fetch_state` memorizza già lo stato dell'ultimo fetch e l'ora dell'ultimo fetch di ogni fonte. Aggiungi una vista `nadezhda sources --health` che segnali una fonte che ha restituito un errore o un 304 per molte esecuzioni di fila, così un feed morto in silenzio diventa visibile.

## Difficile

**Sostituisci il clustering a coppie con LSH.** `internal/cluster` confronta ogni coppia candidata all'interno della finestra, il che va bene per centinaia di articoli ed è quadratico per decine di migliaia. La ricerca ha rimandato SimHash come ottimizzazione di scala proprio per questo motivo. Implementa il locality sensitive hashing sugli insiemi di token dei titoli, in modo che i quasi duplicati finiscano nello stesso bucket e si confronti solo all'interno di un bucket. Mantieni l'attuale `Compute` come implementazione di riferimento e verifica che la versione LSH produca gli stessi cluster sulle fixture dei test prima di fidarti.

**Rendi persistente il watermark degli alert.** Il daemon ricava "nuovo dall'ultimo ciclo" dall'orologio di sistema del ciclo corrente, il che è corretto tranne in caso di un salto indietro dell'orologio, dove un articolo potrebbe generare un alert due volte. Aggiungi una piccola tabella `watch_state` che registra l'ultimo punto notificato e fai in modo che `buildNotable` la legga al posto dell'inizio del ciclo in memoria. Questo fa anche sopravvivere gli alert a un riavvio in modo pulito, così un daemon che va in crash e riparte non rinotifica l'arretrato.

**Incorpora la versione reale nelle build di `go install`.** I binari di release portano con sé la loro versione perché goreleaser la inietta con `ldflags`. Una semplice build con `go install` mostra il valore predefinito di sviluppo, perché `go install` non inietta nulla. Leggi la versione del modulo da `runtime/debug.ReadBuildInfo` come fallback in `internal/version`, in modo che una build con tag tramite `go install` riporti il proprio tag. È una piccola modifica che ti insegna come Go incorpora i metadati di build.

**Aggiungi la ricerca full text.** SQLite include FTS5. Aggiungi una tabella virtuale sui titoli e sui corpi degli articoli e un comando `nadezhda search "term"`, così un utente può cercare in tutto l'archivio per prodotto, indipendentemente dal fatto che sia in tendenza o meno. Fai attenzione alla migrazione: è una nuova tabella, applicata solo in avanti, nello stesso modo in cui `internal/store` applica ogni altra modifica di schema.

## Esperto

**Costruisci una dashboard in sola lettura.** Lo store si apre in modalità WAL proprio perché un lettore possa girare mentre uno scrape scrive. Costruisci una piccola UI web che legga `DigestClusters` e serva lo stesso dossier ordinato che mostra la TUI, senza un percorso di scrittura proprio. Il vincolo interessante è che non deve mai migrare né scrivere, quindi apre il database in sola lettura e fallisce rumorosamente se lo schema è più recente di quanto riesca a capire.

**Rendi il clustering incrementale.** `Rebuild` ricalcola ogni cluster da zero a ogni scrape, il che è corretto e semplice ed eventualmente lento. Progetta una versione incrementale che riclusterizzi solo gli articoli all'interno della finestra attiva e li fonda nei cluster esistenti, e dimostra che converge allo stesso risultato di una ricostruzione completa. È davvero difficile, perché l'identità del cluster deve restare stabile tra un'esecuzione e l'altra affinché il segnale di velocity abbia un significato.

**Impara un modello di rilevanza.** Il ranking è una somma pesata fissa. Raccogli quali storie un utente apre davvero nella TUI e addestra un modello che preveda la rilevanza a partire dagli stessi segnali più la cronologia dell'utente. Mantieni il modello deterministico come cold start e come fallback, e tratta il punteggio appreso come un ulteriore input pesato, in modo che il sistema degradi in modo graduale quando il modello è assente o sbagliato.

**Distribuisci un diff.** Aggiungi un `nadezhda digest --since-last` che mostri solo ciò che è cambiato dal digest precedente: nuovi cluster, cluster cresciuti e CVE che sono entrati di recente in KEV. Questo è il report che una persona vuole davvero ogni mattina, e ti costringe a pensare a cosa significhi "cambiato" per un cluster, che è lo stesso problema di watermark che risolve il daemon di watch.

## Applicazioni nel mondo reale

- Un feed di triage mattutino per un piccolo team di sicurezza, che sostituisce una cartella di segnalibri del browser con un'unica lista ordinata e un alert Slack per i picchi di CVE presenti in KEV.
- Un input di ricerca per uno scrittore o un formatore che deve sapere di cosa parla oggi il mondo della sicurezza, con le evidenze sui CVE allegate.
- Un monitor personale basato su una watchlist dei prodotti esatti che usi, così senti parlare di una falla in tendenza in uno di essi prima che arrivi l'email del tuo vendor.

## Collegamenti con altri progetti

- **sbom-generator-vulnerability-matcher** risponde alla domanda che questo progetto solleva. Nadezhda ti dice che un CVE è in tendenza ed è sfruttato. Il matcher SBOM ti dice se quel CVE è in qualcosa che rilasci davvero. Insieme formano un ciclo di prioritizzazione.
- **ja3-ja4-tls-fingerprinting** è la stessa architettura applicata a un segnale diverso: un motore di intelligence senza chiavi su uno store SQLite incorporato, con la stessa disciplina di ancorare gli output a vettori di risposta noti e di mantenere il motore testabile in isolamento. Leggi le sue idee di clustering insieme a questa.
