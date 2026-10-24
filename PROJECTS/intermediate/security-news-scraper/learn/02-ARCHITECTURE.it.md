<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# Nadezhda: Architettura

Questo documento è la mappa. Mostra la pipeline da un capo all'altro, il package che possiede ciascuna fase, il data model sottostante e le decisioni di design che hanno plasmato tutto. Leggi prima [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per il *perché* di ogni fase; questo file è il *come tutto si incastra*.

## La pipeline

Tutto scorre in una sola direzione, da un elenco di feed a un insieme classificato di superfici.

```
             sources.yaml (embedded default, or a user file)
                    │
                    ▼
   ┌───────────────────────────────────┐
   │ fetch   N workers, per-host rate   │   internal/fetch
   │         limit, conditional GET     │
   │         (ETag / Last-Modified)     │
   └───────────────┬───────────────────┘
                   │ raw bytes + fetch_state
                   ▼
   ┌───────────────────────────────────┐
   │ parse   RSS / Atom (gofeed)        │   internal/parse
   │         HTML fallback (goquery)    │
   └───────────────┬───────────────────┘
                   │ raw items
                   ▼
   ┌───────────────────────────────────┐
   │ normalize  canonical URL, strip    │   internal/normalize
   │            HTML, content hash      │
   └───────────────┬───────────────────┘
                   │
                   ▼
   ┌───────────────────────────────────┐
   │ ingest   fan-out, dedup, store     │   internal/ingest
   └───────┬───────────────────┬───────┘
           │                   │
   CVE extract (regex)   cluster (union-find)   internal/cluster
           │                   │
           ▼                   ▼
   ┌───────────────────────────────────┐
   │ enrich   CVE list / KEV / EPSS     │   internal/enrich + internal/cve
   │          cached, keyless           │
   └───────────────┬───────────────────┘
                   │
                   ▼
   ┌───────────────────────────────────┐
   │ rank   deterministic weighted      │   internal/rank
   │        score, news first           │
   └───────────────┬───────────────────┘
                   │ ordered clusters
      ┌────────────┼────────────┬────────────┐
      ▼            ▼            ▼            ▼
   digest        tui          ai          watch
  (export)    (browse)    (ideation)    (daemon)
```

Un singolo modulo Go possiede tutto. Non c'è alcun confine tra servizi, nessuna message queue e nessun database esterno. Il comando `scrape` esegue una volta l'intero flusso da sinistra a destra; il daemon `watch` lo esegue su un timer.

## Package e responsabilità

Ogni package `internal` ha un solo compito e una piccola superficie esportata.

| Package | Responsabilità |
|---|---|
| `config` | Carica e valida la configurazione e l'elenco delle fonti. Ogni parametro regolabile vive qui. |
| `source` | Il registro delle fonti e la selezione dell'extractor per singola fonte. |
| `fetch` | HTTP concorrente: worker pool, rate limit per host, GET condizionale, retry con backoff. |
| `parse` | RSS e Atom tramite gofeed, un ripiego HTML tramite goquery. |
| `normalize` | URL canonico, rimozione dell'HTML, hashing di contenuto e titolo, parsing del tempo. |
| `ingest` | L'orchestratore fan out. Fail soft per singola fonte, dedup all'inserimento. |
| `cluster` | Clustering union find per similarità del titolo e CVE condivisa. |
| `cve` | I client CVE senza chiave: CVE list, KEV, EPSS e l'opzionale NVD. |
| `enrich` | Orchestrazione dell'enrichment e cache con TTL. |
| `rank` | Il modello di scoring puro e deterministico. |
| `store` | SQLite: connessione, migrazioni e ogni query tipizzata. |
| `ai` | Il livello di ideation opt in: un client condiviso compatibile con OpenAI più un client Anthropic su misura. |
| `setup` | Il wizard per le credenziali integrato nel binario. |
| `watch` | Lo scheduler del daemon, disaccoppiato dal lavoro che esegue. |
| `tui` | Il browser da terminale basato su bubbletea. |
| `export` | I renderer del digest in Markdown e JSON. |

L'orchestrazione non vive dentro alcun package foglia. Vive in `cmd/nadezhda`, e il nucleo condiviso è una sola funzione, `ingestAndCluster` in `pipeline.go`, che sia `scrape` sia `watch` chiamano. È quella la giuntura che impedisce ai due comandi di divergere: quando si aggiunge un passaggio alla pipeline, entrambi i percorsi lo ottengono perché c'è un solo posto in cui aggiungerlo.

## Decisioni di design

### Un unico binario statico

Lo store è `modernc.org/sqlite`, un'implementazione di SQLite scritta in puro Go. Non c'è CGO, quindi la cross compilation è banale e i binari di release per quattro piattaforme escono da un'unica build. Non c'è alcun database server da eseguire, nessuna connection string da configurare e nessun container richiesto per lo strumento principale. Il database è un file. È lo stesso compromesso che fa il progetto ja3-ja4 per il suo intelligence store, ed è quello giusto per uno strumento che una persona esegue sulla propria macchina.

### Enrichment senza chiave, integrato in scrape

Le prime versioni usavano la NVD API come fonte CVE e separavano l'enrichment in un comando a sé, perché NVD senza chiave è soggetta a rate limiting di cinque richieste ogni trenta secondi e non vuoi quello strozzamento sul percorso delle notizie. Il pivot senza chiave ha eliminato il motivo di entrambe le scelte. I record cvelistV5 del CVE Program sono senza chiave e senza throttling, quindi l'enrichment è veloce, e proprio perché è veloce si integra direttamente in `scrape`. Il flusso predefinito è ora un solo comando, `scrape`, poi `tui`.

L'enrichment dentro `scrape` viene eseguito in modalità best effort con un timeout di cinque minuti, e i suoi errori non sono fatali. Se l'endpoint della CVE list è giù, le notizie vengono comunque acquisite, raggruppate in cluster e classificate; semplicemente ottieni meno dettagli CVE in quell'esecuzione. Il percorso delle notizie non si blocca né fallisce mai a causa dell'enrichment, perché le notizie sono il prodotto.

### Il daemon non sa nulla del lavoro

Il package `watch` è uno scheduler puro. Definisce due interfacce, un `Ticker` e un `Notifier`, e riceve la pipeline come funzione iniettata:

```go
type Options struct {
    Interval   time.Duration
    RunAtStart bool
    Cycle      func(context.Context) (Report, error)
    Notifier   Notifier
    NewTicker  func(time.Duration) Ticker
    Out        io.Writer
}
```

`internal/watch` non importa nulla da `store`, `ingest` o `enrich`. Il comando in `cmd/nadezhda/watch.go` costruisce la closure `Cycle` concreta che esegue la vera pipeline e la passa allo scheduler. Il vantaggio è la testabilità: lo scheduler è sottoposto a unit test con un ticker finto il cui channel viene pilotato a mano da un test e un ciclo finto che si limita a contare, così il loop, lo shutdown graduale e il comportamento fail soft sono tutti verificati senza clock, senza rete e senza database. È la stessa disciplina che consiste nel separare un engine dal suo I/O in modo che l'engine possa essere testato in isolamento.

### Fetch time, non publish time, per gli alert

Il daemon watch invia un webhook quando compaiono storie davvero nuove e ad alto segnale. Definire "nuovo" è il punto in cui si annida un bug sottile. I timestamp di un cluster derivano dall'orario di *pubblicazione* degli articoli. Ma un advisory pubblicato la settimana scorsa e recuperato per la prima volta oggi è nuovo *per te*, anche se il suo orario di pubblicazione è vecchio. Filtrare gli alert per orario di pubblicazione lo scarterebbe silenziosamente.

Quindi il daemon ha una query dedicata nello store, `NewlyFetchedClusters`, che filtra su `articles.fetched_at`, il quale è monotono a ogni ciclo di scrape. Un articolo duplicato non aggiorna mai `fetched_at` (gli inserimenti sono solo di tipo insert, e un feed invariato restituisce un 304 che non tocca nulla), quindi una storia genera un alert esattamente una volta, quando viene acquisita per la prima volta. È il genere di dettaglio di correttezza che emerge solo quando si pensa con attenzione a cosa misura davvero il proprio watermark.

## Il data model

```
sources(id, name, title, url, type, weight, tags, enabled)
fetch_state(source_id, etag, last_modified, last_fetched, last_status)
articles(id, source_id,
         canonical_url  UNIQUE,      -- exact dedup
         content_hash   UNIQUE,      -- exact dedup
         title, summary, body, author, published_at, fetched_at)
cves(id TEXT PRIMARY KEY,
     description,
     cvss_score, cvss_version, cvss_severity, cvss_vector,   -- score nullable
     cwe, is_kev, kev_date_added, kev_ransomware,
     epss, epss_percentile, nvd_published, nvd_modified,
     enriched_at, enrich_status)
article_cves(article_id, cve_id)                             -- many to many
clusters(id, cluster_key, first_seen, last_seen, size)
cluster_members(cluster_id, article_id)
ai_notes(id, cluster_id, provider, ...  UNIQUE(cluster_id, provider))
schema_migrations(version, applied_at)
```

Qui contano due punti di design. Primo, la deduplicazione è imposta dal database, non dalla logica applicativa. Un articolo è univoco sia sul suo URL canonico sia sul suo content hash, quindi reingerire lo stesso elemento è una violazione di vincolo che il codice intercetta e tratta come un normale esito "ce l'ho già", non come un errore. Secondo, le migrazioni sono versionate e solo in avanti, applicate automaticamente all'apertura dello store. Una mancata corrispondenza della versione dello schema su un database esistente è un errore rumoroso, mai una cancellazione silenziosa. Lo store si apre in modalità WAL con un busy timeout, così la TUI o una dashboard possono leggere mentre uno scrape scrive.

## Una storia dal feed al dossier

Segui un elemento da un capo all'altro.

1. `fetch` richiede il feed di BleepingComputer con l'ETag salvato l'ultima volta. Il server restituisce 200 con nuovo contenuto (un 304 chiuderebbe qui la storia, a basso costo).
2. `parse` trasforma l'RSS in elementi. `normalize` rende canonico ogni URL, rimuove i parametri di tracking e calcola un content hash.
3. `ingest` inserisce il nuovo articolo. Una regex CVE trova `CVE-2025-5777` nel titolo e lo collega. L'inserimento dell'articolo di una seconda testata sullo stesso difetto riesce come riga a sé.
4. `cluster` ricostruisce. I due articoli condividono `CVE-2025-5777`, quindi entrano in un unico cluster. La sua dimensione è 2 e il suo `last_seen` avanza.
5. `enrich` vede che `CVE-2025-5777` ha bisogno di dati, recupera il suo record cvelistV5, legge il punteggio CVSS 4.0 di 9.3 dal container del vendor, controlla il catalogo KEV (presente, ransomware "Known") e recupera il suo punteggio EPSS. Tutto viene messo in cache con un TTL.
6. `rank` assegna un punteggio al cluster. Due testate più un orario di pubblicazione recente più una voce KEV più un CVSS alto lo portano vicino alla cima.
7. `tui` lo visualizza: una riga colorata con un rank, due pallini per le testate, un indicatore di severità CRITICAL e un chip KEV. Premendo invio si apre il dossier con entrambi i link alle fonti e la scheda CVE completa.

## Considerazioni su performance e sicurezza

- **La cortesia è imposta, non facoltativa.** Ogni fetch è soggetto a rate limit per host, usa GET condizionale in modo che un feed invariato costi un solo economico 304, e porta uno User-Agent onesto. Un retry rispetta `Retry-After` e non ritenta mai un timeout o una cancellazione.
- **Le credenziali sono gestite con cura.** Il wizard AI scrive le chiavi in `~/.config/nadezhda/credentials` con permessi `0600`, usando un file temporaneo e un rename atomico, in modo che i permessi di un file esistente vengano resi più stretti anziché lasciati com'erano. Una chiave esportata dalla shell vince sempre sul file, e il loader del file accetta solo nomi di chiave presenti in una allowlist, il che impedisce a un file di credenziali manomesso di iniettare qualcosa come `LD_PRELOAD` nell'ambiente del processo.
- **I client AI non seguono i redirect.** Un redirect potrebbe far trapelare un header `x-api-key` verso un host non previsto, quindi i client rifiutano di seguirli.
- **Gli ambienti dei sottoprocessi sono sanificati.** L'azione "open in browser" nella TUI avvia un processo figlio con un ambiente ripulito, non con quello completo, così un segreto nel processo padre non viene consegnato al launcher del browser.
- **Il fallimento è soft dove dovrebbe e rumoroso dove deve.** Un feed rotto non interrompe uno scrape. Un enrichment che fallisce non blocca le notizie. Ma un database corrotto o una migrazione fallita fermano il programma, perché proseguire oltre quei punti corromperebbe silenziosamente i tuoi risultati.
