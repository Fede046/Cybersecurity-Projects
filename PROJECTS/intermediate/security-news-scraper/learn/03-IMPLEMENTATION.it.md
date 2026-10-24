<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# Nadezhda: Implementazione

Questa è la walkthrough del codice. Segue lo stesso ordine da sinistra a destra della pipeline descritta in [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md), citando le funzioni reali così da poter aprire ciascuna mentre leggi. Il codice è referenziato per package e funzione, mai per numero di riga, perché i numeri di riga marciscono nell'istante in cui qualcuno modifica un file.

## Da dove iniziare a leggere

Apri `cmd/nadezhda/scrape.go` e trova `runScrape`. È l'intero tool in miniatura: carica la config, carica le sorgenti, apre lo store, costruisce un client di fetch, esegue la pipeline, stampa il riepilogo, arricchisce. L'unica riga che fa il lavoro vero è:

```go
summary, cstats, err := ingestAndCluster(ctx, fc, st, cfg, targets, now)
```

`ingestAndCluster` vive in `pipeline.go`, ed è il punto di giunzione condiviso che chiama anche `watch`. Tutto ciò che segue è quello che accade al suo interno e dopo di essa.

## Fetching educato

`internal/fetch` è un worker pool basato su `net/http`. Il comportamento interessante è la GET condizionale. Al primo fetch di una sorgente memorizza gli header di risposta `ETag` e `Last-Modified` nella tabella `fetch_state`. Al fetch successivo li rimanda indietro come `If-None-Match` e `If-Modified-Since`. Un feed invariato risponde `304 Not Modified` senza body, il che costa quasi nulla, e la riga della sorgente nel riepilogo dello scrape mostra `304` invece di un conteggio di parsing.

La logica di retry è volutamente conservativa. Rispetta l'header `Retry-After` quando il server lo invia, e *non* ritenta un timeout o un context annullato, perché ritentare un timeout di solito significa solo aspettare lo stesso timeout un'altra volta. Il rate limiting per host usa `golang.org/x/time/rate`, quindi due feed sullo stesso dominio non lo martellano in parallelo.

## Parsing e normalizzazione

`internal/parse` usa `gofeed`, che gestisce RSS e Atom e la maggior parte delle loro deviazioni nel mondo reale. Una lezione che vale la pena interiorizzare risiede nel parsing del tempo: i feed non concordano sui formati di data. Il feed di CISA usa RFC822 con una zona numerica, che una versione precedente della ricerca presumeva erroneamente fosse RFC1123Z. Il parser porta con sé una lista di layout di fallback e li prova in ordine, perché un feed le cui date non vengono parsate perde silenziosamente il suo segnale di recency.

`internal/normalize` fa tre cose piccole ma portanti. Canonicalizza gli URL, rimuovendo parametri di tracking come `utm_*` e `fbclid`, così lo stesso articolo condiviso con code di tracking diverse viene deduplicato correttamente. Rimuove l'HTML dai riepiloghi con `goquery`. E calcola un content hash SHA-256 e un hash del titolo, che sono le chiavi esatte di dedup.

## Ingestione, fail soft

`Run` di `internal/ingest` si dirama sulle sorgenti con un `errgroup`, ma con una particolarità: le goroutine per sorgente restituiscono sempre `nil`. L'errore di una sorgente viene registrato nella sua struct di risultato, non propagato verso l'alto, così un feed rotto non cancella mai il gruppo e non interrompe tutte le altre sorgenti. Il comando somma i risultati e riporta i fallimenti alla fine. Ecco perché puoi perdere The Register per un timeout e ottenere comunque uno scrape completo dalle altre sei.

Il dedup avviene nel database. `store.InsertArticle` è una semplice `INSERT`, e quando scatta il vincolo di unicità su `canonical_url` o `content_hash`, restituisce un sentinel:

```go
if errors.As(err, &se) && se.Code() == sqlite3.SQLITE_CONSTRAINT_UNIQUE {
    return 0, ErrDuplicate
}
```

Il chiamante tratta `ErrDuplicate` come un normale "questo ce l'ho già" e incrementa il contatore dei duplicati. Non esiste una race tra lettura e scrittura, perché il database è l'unica fonte di verità su ciò che esiste.

## Clustering con union find

`internal/cluster` implementa le componenti connesse. `Compute` prende gli elementi candidati, tokenizza ogni titolo ed esegue un `unionFind` su ogni coppia all'interno della finestra temporale. Due elementi vengono uniti quando condividono una CVE, oppure quando provengono da sorgenti diverse e i loro insiemi di token del titolo hanno una sovrapposizione di Jaccard pari o superiore alla soglia:

```go
sharedCVE := shareAny(prepared[i].cves, prepared[j].cves)
crossSource := prepared[i].item.SourceID != prepared[j].item.SourceID
titleMatch := crossSource &&
    jaccard(prepared[i].tokens, prepared[j].tokens) >= jaccardThreshold
if sharedCVE || titleMatch {
    uf.union(i, j)
}
```

`buildCluster` ricava poi `FirstSeen` e `LastSeen` del cluster dai tempi del membro più vecchio e di quello più recente, e il suo `SourceCount` dalle sorgenti distinte. `Rebuild` è il punto di ingresso che la pipeline chiama: preleva gli articoli candidati con `ClusterCandidates`, esegue `Compute` e scrive il risultato con `ReplaceClusters`, che svuota e riscrive le tabelle dei cluster in un'unica transazione così l'operazione è idempotente. Rieseguire uno scrape non produce mai cluster che derivano o duplicati.

Nota l'espressione sul timestamp in `ClusterCandidates`:

```sql
COALESCE(NULLIF(published_at, 0), fetched_at)
```

Un elemento senza tempo di pubblicazione ripiega sul suo tempo di fetch, così un feed che omette le date viene comunque clusterizzato all'interno della finestra invece di collassare all'epoch.

## I client CVE senza chiave

`internal/cve` contiene tre client dietro un'interfaccia comune. Il core CVE senza chiave, nel client della CVE list, è dove la ricerca ha dato i suoi frutti. Parsa il record cvelistV5, che ha un blocco `containers.cna` del vendor e zero o più blocchi `containers.adp` di authorized data publisher come CISA. Raccoglie ogni metrica CVSS da entrambi, poi applica la precedenza di versione: preferisce v4.0, poi v3.1, poi v3.0, poi v2.0. Questa ricerca in due posti è ciò che legge correttamente il punteggio 10.0 di Log4Shell dal container CISA-ADP quando il container del vendor contiene solo un placeholder.

Il client KEV scarica il catalogo una volta e costruisce una mappa di appartenenza. Mappa esplicitamente il campo ransomware, perché la sorgente fornisce una stringa:

```go
ransomware := entry.KnownRansomwareCampaignUse == "Known"
```

Il client EPSS parsa lo score e il percentile con `ParseFloat`, perché l'API li restituisce come stringhe tra virgolette. Se li leggi come numeri JSON vengono decodificati a zero senza alcun errore, che è il tipo silenzioso di bug che sopravvive fino alla produzione.

## Arricchimento con una cache

`Run` di `internal/enrich` chiede allo store quali CVE richiedono lavoro tramite `CVEsNeedingEnrichment`, che restituisce qualsiasi CVE mai arricchita, o una il cui risultato positivo è più vecchio del TTL della cache, o una il cui risultato negativo (non trovata) è più vecchio del TTL negativo, più breve. La cache negativa è ciò che impedisce al tool di martellare la sorgente CVE a ogni esecuzione per una CVE che ancora non esiste.

Il fallimento di KEV è fatale per l'esecuzione dell'arricchimento, by design. Se il fetch del catalogo KEV fallisce, il codice rifiuta di scrivere `is_kev = false` per tutto, perché un `false` sbagliato persisterebbe per un intero TTL e nasconderebbe silenziosamente le vulnerabilità sfruttate. I fallimenti di NVD ed EPSS sono soft e riprendibili.

## Ranking, in modo puro

`Score` di `internal/rank` è una funzione pura di una struct `Signals` e dei pesi della config. `recency` è il decadimento esponenziale a emivita, e `velocity` divide la dimensione del cluster per l'età con un floor, così un cluster di un solo elemento ottiene velocity zero:

```go
func recency(ageHours float64, halfLifeHours int) float64 {
    return math.Exp(-math.Ln2 * ageHours / float64(halfLifeHours))
}
```

`signalsFor` costruisce la struct dei segnali per un cluster, scorrendo le sue CVE per prendere il CVSS massimo, l'EPSS massimo e se una qualsiasi è elencata in KEV. `Rank` assegna un punteggio a ogni cluster e li ordina in modo stabile in senso decrescente, risolvendo i pareggi con il `LastSeen` più recente. Poiché è pura, i golden order test possono asserire un ordinamento esatto a partire da input fissi.

## Le query dello store che contano

`internal/store` nasconde tutto l'SQL. Due query meritano di essere conosciute per nome. `DigestClusters(since)` filtra `WHERE last_seen >= ?`, che si basa sul tempo di pubblicazione, e alimenta la TUI e il digest. `NewlyFetchedClusters(sinceFetched)` filtra `WHERE a.fetched_at >= ?`, che si basa sul tempo di fetch, e alimenta solo gli alert del daemon watch. Esistono come due query separate di proposito, per la ragione tempo di fetch contro tempo di pubblicazione spiegata nel documento di architettura, e un test dello store asserisce direttamente la distinzione così che nessuno le fonda insieme in futuro.

## Il daemon watch

`Run` di `internal/watch` è il loop:

```go
for {
    select {
    case <-ctx.Done():
        return shutdown(out)
    case <-ticker.C():
        if stop, err := cycleAndNotify(ctx, opts, out); stop {
            return shutdown(out)
        } else if err != nil {
            fmt.Fprintf(out, "watch: cycle error: %v\n", err)
        }
    }
}
```

Un errore di ciclo viene loggato e il loop continua, che è il contratto fail soft per un daemon a lunga esecuzione. `ctx.Done()` restituisce `nil`, quindi Ctrl-C o SIGTERM sono un'uscita pulita, non un errore. `Once` esegue lo stesso `cycleAndNotify` una sola volta per l'uso con cron.

Il ciclo concreto vive in `cmd/nadezhda/watch.go`. Dopo aver eseguito `ingestAndCluster` e l'arricchimento, chiama `buildNotable`, che preleva `NewlyFetchedClusters`, li ordina per ranking e conserva quelli notevoli fino a un limite configurato. `isNotable` è la policy: un cluster è notevole se il suo punteggio supera la soglia oppure, quando `notify_on_kev` è impostato, se contiene una CVE elencata in KEV. `representativeArticle` sceglie il titolo della testata con la fiducia più alta per rappresentare il cluster nell'alert.

## Il livello AI

`internal/ai` definisce una sola interfaccia `Provider` con un metodo `Generate`. Tre provider (qwen, openai, gemini) condividono un unico client compatibile con OpenAI, e Anthropic ha il proprio client, scritto su `net/http` grezzo, perché differisce per header di autenticazione, posizionamento del system prompt e forma della risposta. Le chiavi vengono lette solo dall'ambiente. Un modello come un Qwen locale tende ad avvolgere il suo JSON in prosa, quindi il parser dei risultati estrae il JSON scansionando le parentesi graffe bilanciate e provando ogni oggetto candidato invece di assumere che l'intera risposta sia JSON.

## Il wizard di setup

`internal/setup` è la storia di installazione senza attriti. `nadezhda ai` esegue un wizard interattivo e rieseguibile. Scrive le chiavi in un file `0600` con un file temporaneo e un rename atomico, così restringe i permessi di un file esistente invece di fidarsene. `Load` viene eseguito a ogni comando e imposta ogni chiave nell'ambiente solo se non è già impostata, quindi una variabile di shell vince sempre, e accetta solo nomi di chiave in allowlist così che un file manomesso non possa iniettare variabili d'ambiente arbitrarie nel processo.

## Errori comuni

Questi sono i bug reali che sono stati individuati e corretti, e quelli che più probabilmente reintrodurrai.

- **Parsare i campi EPSS o KEV con il tipo sbagliato.** Gli score EPSS sono stringhe; il flag ransomware è una stringa. Entrambi decodificano in modo errato in silenzio.
- **Leggere il CVSS solo dal container del vendor.** Riporterai "nessun punteggio" proprio per le CVE più importanti.
- **Usare il tempo di pubblicazione per gli alert.** Un advisory retrodatato recuperato oggi ha un tempo di pubblicazione vecchio. Genera l'alert sul tempo di fetch.
- **Scrivere `is_kev = false` quando il fetch di KEV è fallito.** Un negativo sbagliato persiste per un intero TTL. Fai fallire l'esecuzione.
- **Lasciare che un feed rotto interrompa lo scrape.** Fai il fan out in modo fail soft, registra l'errore per sorgente.

## Suggerimenti per il debug

- Esegui `nadezhda scrape` due volte. La seconda esecuzione dovrebbe mostrare ogni sorgente come `304` o tutti duplicati, dimostrando l'idempotenza. Se mostra nuovi articoli entrambe le volte, il dedup è rotto.
- Punta `nadezhda cve CVE-2021-44228` su una CVE nota dopo uno scrape che la menziona. Dovresti vedere CVSS 10.0, un elenco in KEV e un EPSS alto. Se il punteggio manca, il sospetto è la tua ricerca nei container.
- Per il daemon, esegui `nadezhda watch --once` su un database di prova e leggi il report di ciclo di una riga. Stampa i conteggi di new, duplicate, cluster, enriched, KEV e notable, che ti dice quale stage si sta comportando male.
