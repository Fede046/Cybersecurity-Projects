# Come funzionano davvero gli algoritmi di rate limiting

Questo pacchetto include tre algoritmi. La maggior parte delle persone ne sceglie semplicemente uno senza capire cosa fanno di diverso. Ecco cosa succede dietro le quinte.

## La versione breve

| Algoritmo | Bravo in | Scarso in |
|-----------|---------|--------|
| Sliding Window | Accuratezza, efficienza della memoria | Niente, in realtà |
| Token Bucket | Gestire i burst con eleganza | Può sembrare imprevedibile |
| Fixed Window | Semplicità, velocità | Burst ai confini (vedi sotto) |

Lo sliding window è il predefinito perché Cloudflare lo ha testato su 400 milioni di richieste e ha misurato un'accuratezza del 99,997%. Non è marketing, è semplicemente il risultato della matematica.

## Fixed Window (e perché ha un problema)

Il fixed window è l'approccio più semplice. Dividi il tempo in blocchi (diciamo, finestre di 1 minuto), conta le richieste in ogni blocco, blocca quando il conteggio raggiunge il limite.

```
Window 1 (0:00-0:59)    Window 2 (1:00-1:59)
[--------------------]  [--------------------]
      87 requests              12 requests
```

Il problema a cui nessuno pensa: cosa succede al confine?

Supponiamo che tu abbia un limite di 100 richieste/minuto. Un client invia 100 richieste a 0:59 (permesse, il contatore è a 100). Poi a 1:00 la finestra si azzera e ne invia altre 100 (permesse, nuova finestra). Sono 200 richieste in 2 secondi. Il limite dovrebbe essere 100/minuto, ma ne ha ottenute 200.

Questo si chiama problema dei burst ai confini. Non è teorico. Gli attaccanti lo conoscono.

```
0:00           0:59  1:00           1:59
[---------------||--][--||---------------]
                100     100
                   ^
                   200 requests in 2 seconds
```

Il fixed window resta utile quando ti serve la massima velocità e puoi tollerare questo caso limite. Ma per la maggior parte delle API, vuoi qualcosa di meglio.

## Sliding Window Counter (il predefinito)

Questo risolve il problema dei confini guardando due finestre ed effettuando un'interpolazione pesata.

L'idea: invece di confini netti tra le finestre, calcola una media pesata in base a quanto sei avanti nella finestra corrente.

```python
elapsed_ratio = (current_time % window_size) / window_size
weighted_count = (previous_window_count * (1 - elapsed_ratio)) + current_window_count
```

Supponiamo che tu sia al 40% del minuto corrente. Prendi il 60% del conteggio della finestra precedente più il 100% del conteggio della finestra corrente. Questo crea un effetto "scorrevole" in cui la finestra avanza in modo fluido invece di saltare.

```
Previous window    Current window
[-------50-------] [---20---.........]
                        ^
                        40% elapsed

weighted = (50 * 0.6) + 20 = 30 + 20 = 50
```

Il costo in memoria è di soli due interi per client (conteggio precedente, conteggio corrente). L'accuratezza del 99,997% deriva dal fatto che stai approssimando una vera sliding window con uno storage minimo.

Il compromesso è un errore medio di circa il 6% rispetto a un perfetto sliding window log. In pratica nessuno se ne accorge.

## Token Bucket (per il traffico a burst)

Il token bucket ragiona sul rate limiting in modo diverso. Invece di contare le richieste, hai un secchio che si riempie di token nel tempo. Ogni richiesta prende un token. Niente token, niente richiesta.

```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

[OOOOOOOOOO]  <- full bucket (10 tokens)

5 requests come in at once:
[OOOOO-----]  <- 5 tokens left

Wait 3 seconds:
[OOOOOOOO--]  <- refilled to 8 tokens
```

La differenza chiave rispetto allo sliding window: la tolleranza ai burst.

Con lo sliding window a 60 richieste/minuto, un client può fare al massimo 60 richieste in qualsiasi periodo di 60 secondi. Costante, prevedibile.

Con il token bucket con capacità di 60 token/minuto e ricarica di 1 token/secondo, un client può consumare istantaneamente tutti i 60 token se li ha accumulati, e poi attendere le ricariche. A burst, ma rispettando comunque il ritmo medio.

La matematica:
```python
tokens = min(capacity, tokens + (elapsed_time * refill_rate))
if tokens >= 1:
    tokens -= 1
    return allowed
return denied
```

Quando usarlo: API in cui i client legittimi hanno schemi a burst. App mobili che si sincronizzano all'apertura. Endpoint di elaborazione batch. Qualsiasi cosa in cui "60 al minuto in media" abbia più senso di "mai più di 60 in un qualsiasi minuto".

## Come memorizziamo tutto questo

Tutti e tre gli algoritmi devono tenere traccia dello stato per client. Il livello di storage se ne occupa.

Per lo sliding window, memorizziamo:
```
key: "rl:v1:user:GET:/api/data:abc123:60"
value: {count: 47, window_start: 1703894400}
```

Per il token bucket:
```
key: "rl:v1:user:GET:/api/data:abc123:bucket"
value: {tokens: 8.5, last_update: 1703894567.123}
```

Lo storage in memoria usa dict Python con eviction LRU quando raggiungi il numero massimo di chiavi. Lo storage Redis usa script Lua per rendere atomica l'operazione di lettura-modifica-scrittura (maggiori dettagli sul perché nel documento sull'architettura).

## Scegliere un algoritmo

Nella maggior parte dei casi: sliding window. È accurato, efficiente in memoria e non ha strani casi limite.

Usa il token bucket quando:
- I tuoi client hanno legittimamente bisogno di capacità di burst
- Il "ritmo medio" conta più del "ritmo istantaneo"
- Stai applicando il rate limiting a cose come upload di file od operazioni batch

Usa il fixed window quando:
- Ti servono le massime prestazioni e la minima complessità
- Il burst ai confini è accettabile per il tuo caso d'uso
- Stai facendo qualcosa come quote API giornaliere in cui la precisione al minuto non conta

## La matematica dietro l'accuratezza dello sliding window

Se sei curioso di sapere perché l'affermazione sul 6% di errore regge:

Il caso peggiore è quando tutte le richieste della finestra precedente sono avvenute proprio alla fine e tutte le richieste della finestra corrente sono avvenute proprio all'inizio. La formula pesata conta in modo leggermente errato in questo scenario.

Ma in pratica le richieste sono distribuite in modo abbastanza uniforme (soprattutto durante un attacco, che è quando l'accuratezza conta di più). I test nel mondo reale mostrano che l'errore si attesta in media intorno allo 0,003% con carico normale. Il 6% è il caso peggiore teorico con temporizzazione avversaria.

Per il rate limiting, è più che sufficiente. Non stai facendo contabilità finanziaria. Sbagliare di una o due richieste su cento va bene.
