<!--
©AngelaMos | 2026
01-CONCEPTS.md
-->

# Concetti

Prima di esaminare il codice, devi capire la teoria di sicurezza e di operazioni attorno a cui è costruita la dashboard. Salta questa parte se sai già cosa significa il percentile EPSS, cos'è davvero un BGP hijack e perché un analista SOC con un solo schermo ha bisogno di un mix di feed diverso da uno che ne ha otto.

## La situational awareness come disciplina

L'origine militare di "situational awareness" (Endsley, 1995) suddivide il termine in tre livelli:

1. **Percezione** — cosa sta succedendo in questo momento nell'ambiente.
2. **Comprensione** — cosa significano insieme quelle osservazioni grezze.
3. **Proiezione** — cosa è probabile che accada dopo.

La maggior parte delle dashboard di sicurezza si ferma al livello 1. Mostrano alert, grafici del traffico e conteggi delle CVE. Una dashboard utile deve rendere economico il livello 2 (fusione visiva tra i feed) e almeno spingere verso il livello 3 (diff della KEV, curve di crescita dell'EPSS, post sui leak site dei ransomware che storicamente precedono di 3-7 giorni la divulgazione pubblica di una violazione).

Gli undici feed di questo progetto sono scelti in modo che nessuna vista a schermo singolo richieda all'operatore di fare riferimenti incrociati a mente. Il globo spazializza i dati legati agli ASN (interruzioni, hijack, sorgenti di scansione) sulla stessa superficie degli eventi fisici (terremoti, ISS, vittime ransomware per paese). Il pannello delle CVE mostra le barre EPSS accanto ai badge KEV, così "dovrebbe importarmi?" è una decisione visiva, non una query al database.

## Gli undici feed

| Feed | Cosa misura | Cadenza di aggiornamento | Budget di latenza |
|---|---|---|---|
| DShield (SANS Internet Storm Center) | ASN sorgente che fanno più scansioni e porte bersaglio | 1h | 5 min |
| Cloudflare Radar — interruzioni | Cali di connettività a livello di paese con attribuzione agli ASN | 5 min | 30 sec |
| Cloudflare Radar — BGP hijack | Anomalie MOAS sui prefissi con punteggio di confidenza | 5 min | 30 sec |
| NVD CVE 2.0 | CVE appena pubblicate o modificate | 2h | 30 min |
| FIRST EPSS | Percentile di previsione di exploit per CVE | 2h | ore |
| CISA KEV | Catalogo ufficiale "sfruttate in the wild" | 1h | ore |
| ransomware.live | Post delle vittime sui leak site dei ransomware | 15m | minuti |
| Coinbase WS | Tick spot di BTC/ETH | persistente | sotto il secondo |
| USGS GeoJSON | Terremoti globali M2.5+ | 1m | 30 sec |
| NOAA SWPC | Indice Kp, Bz GSM, flusso di raggi X, velocità/densità del vento solare | 1m / 3h | minuti |
| Wikipedia ITN + GDELT | Titoli delle notizie mondiali e picchi di tono degli eventi | 5m / 15m | minuti |
| Posizione ISS + passaggi | Punto subsatellitare e TLE | 10s / 24h | secondi |

La colonna della cadenza è ciò che vede l'operatore. Il budget di latenza è quanto il sistema può restare indietro rispetto alla fonte prima che i dati siano abbastanza sbagliati da informare male una decisione. Coinbase ha un budget sotto il secondo perché BTC si muove; la KEV ha ore perché l'aggiunta alla KEV è essa stessa un processo lento. **Costruire una dashboard significa far corrispondere l'indicatore di freschezza di ogni pannello al budget giusto**, non semplicemente interrogare tutto il più velocemente possibile.

## EPSS vs CVSS, e perché è importante

CVSS (Common Vulnerability Scoring System) fornisce un numero di gravità da 0 a 10. CVSS è un *modello di rischio* nel senso peggiore: una griglia statica valutata una sola volta alla divulgazione che chiede "se sfruttata, quanto è grave?" Non risponde a "verrà sfruttata?"

EPSS (Exploit Prediction Scoring System, gestito da FIRST.org) risponde alla seconda domanda. Produce, per ogni CVE in NVD, una probabilità tra 0 e 1 che la CVE venga sfruttata nei prossimi 30 giorni, più un percentile rispetto al resto della popolazione. Il modello viene aggiornato ogni giorno sulla base della telemetria di sfruttamento osservata.

I numeri pratici:

- Circa il 6% di tutte le CVE pubblicate viene mai sfruttato in the wild.
- Un percentile EPSS sopra ~95 è una soglia di triage utile per "guardala oggi".
- L'appartenenza al catalogo KEV è binaria, curata a mano, e arriva in ritardo rispetto all'EPSS. La KEV è la prova più costosa (una CVE è finita nella KEV perché sono stati osservati attacchi reali) e l'EPSS è il segnale predittivo più economico.

Il pannello delle CVE in questa dashboard usa il percentile EPSS come chiave di ordinamento primaria, con un badge KEV come marcatore visivo secondario. Il CVSS viene mostrato ma con minore enfasi.

**Esempio reale**: CVE-2017-5638 (Apache Struts, violazione di Equifax). CVSS 10.0. L'EPSS alla divulgazione era 0.97. La KEV l'ha inserita in seguito. Nel momento in cui una dashboard basata solo sul CVSS l'avrebbe portata all'attenzione "critica", l'EPSS diceva già "questa verrà sfruttata entro 30 giorni con probabilità del 97%".

## La CISA KEV e perché un diff è importante

Il catalogo CISA Known Exploited Vulnerabilities è un elenco mantenuto a mano di CVE che la CISA ha osservato essere sfruttate contro sistemi federali statunitensi o in campagne diffuse. Ogni voce include un `dateAdded` e un `dueDate` (entro cui le agenzie civili federali devono applicare la patch).

Il catalogo è piccolo (~1000 voci) e cresce di 1-10 a settimana. Il segnale interessante non è lo snapshot, è il **diff**. Quando una CVE viene *aggiunta* alla KEV oggi, significa che la CISA ha appena confermato uno sfruttamento in the wild che ieri non aveva visibilità pubblica. Quel diff è tra i singoli segnali più forti a cui un difensore aziendale possa iscriversi.

Il collector della KEV interroga il catalogo, calcola l'insieme delle novità dall'ultimo polling ed emette un evento `kev_added` per ogni nuova CVE. Il frontend mostra un banner e il motore degli alert, opzionalmente, invia notifiche Telegram/Discord/Slack.

## BGP hijack e route leak

Internet instrada il traffico fidandosi del fatto che gli ASN (Autonomous System, le entità che gestiscono le reti) annuncino onestamente quali prefissi IP possiedono. Non tutti lo fanno onestamente.

Un **BGP hijack** si verifica quando un ASN annuncia un prefisso che non possiede. Due varianti:

- **Origin hijack (MOAS, Multiple Origin AS)**: due ASN annunciano entrambi lo stesso prefisso. Il punteggio di confidenza di Cloudflare Radar fonde questo dato con lo storico di validazione dei prefissi.
- **Manipolazione del percorso**: un AS annuncia un prefisso più specifico o un percorso più breve in modo che il traffico lo preferisca. È così che l'hijack di Amazon Route 53 del 2018 ha rubato le credenziali di MyEtherWallet.

Un **route leak** è simile ma accidentale: un router mal configurato annuncia le rotte dei clienti a peer verso cui non avrebbero dovuto essere propagate.

L'endpoint BGP di Radar restituisce i candidati hijack con: `started_at`, `duration_sec`, `confidence` (0-100), `hijacker_asn`, `victim_asns`, `prefixes`. Il collector filtra in base al `MinConfidence` configurato (default 60) prima di salvare ed emettere. Il pannello del globo disegna le regioni di hijack sul percorso a cerchio massimo tra le geolocalizzazioni dell'ASN hijacker e dell'ASN vittima.

**Incidenti reali su cui documentarsi**:

- Aprile 2018, AS10297 (eNet) ha dirottato i prefissi di Route 53 per due ore, reindirizzato MyEtherWallet e rubato ~$150k di ETH.
- Giugno 2024, AS267613 (Eletel) ha dirottato i prefissi di Orange España, mettendo l'operatore parzialmente offline.
- Aprile 2010, China Telecom (AS23724) ha annunciato per errore 50.000 prefissi in tutto il mondo per ~18 minuti ("la grande deviazione cinese di internet").

## Firehose di mass-scan

DShield, gestito dal SANS Internet Storm Center, aggrega i log dei firewall di migliaia di contributori volontari. Pubblica:

- Le principali sorgenti segnalate (quali IP hanno scansionato il maggior numero di contributori nelle ultime 24h).
- Le porte più prese di mira (quali porte TCP/UDP sono state colpite più duramente).
- La distribuzione delle scansioni a livello di paese.

Questo è l'unico feed del sistema che fornisce una vista globale, lato attaccante, di "cosa viene scansionato in questo momento". *Non* è un feed di attacchi nel senso di un IDS; è il rumore di fondo di internet. Il segnale sta nel *delta*: quando la porta 7547 (TR-069 / CWMP) balza in cima alla tabella, probabilmente stai assistendo all'avvio della prossima botnet IoT di classe Mirai. Quando balza la porta 8728 (MikroTik Winbox), stai assistendo di nuovo allo sfruttamento dei router MikroTik.

Il collector di DShield unisce tre endpoint (sorgenti principali, porte principali, distribuzione per paese) in un singolo blob di snapshot e lo emette come `scan_firehose`.

## Snapshot vs Stream — Il protocollo in due fasi

La dashboard parla due protocolli che devono coesistere:

1. **Snapshot REST** — un browser appena avviato chiede `GET /v1/snapshot` e ottiene lo stato più recente di ogni topic in un unico blob JSON. Proviene da Redis, popolato continuamente dal bus.
2. **Stream WebSocket** — lo stesso browser apre `/v1/ws?topics=cve_new,kev_added,...` e riceve gli eventi man mano che accadono.

I due devono essere messi in sequenza correttamente, altrimenti il browser vede uno stato impossibile. Se lo snapshot è in ritardo rispetto allo stream anche di un solo secondo, puoi ricevere un evento `cve_updated` per una CVE che lo snapshot non conosce. Il frontend gestisce questo con un buffer:

- Il WS si apre. Il frontend mette in buffer tutti i messaggi.
- Lo snapshot REST ritorna. Il frontend idrata lo stato.
- Il frontend chiama `setReady()` sull'handle del WS. I messaggi in buffer vengono scaricati nello stesso stato.
- I messaggi successivi fluiscono live.

Questo è un classico pattern snapshot+stream (usato da Kafka Connect, dal kernel Linux, dalla replica logica di Postgres). Sbagliarlo è la causa più comune dei bug "mancano dei dati" nelle UI in tempo reale.

## Perché un bus in-process invece di Kafka o NATS

Un riflesso comune è "fan-out in tempo reale → broker". Per questa scala (undici producer, decine di subscriber WS, una sola macchina) un broker è eccessivo. Il calcolo dei costi:

| Opzione | Pro | Contro a questa scala |
|---|---|---|
| Kafka | Durevole, scalabile orizzontalmente, riproducibile | Costo operativo multi-nodo, volumi persistenti, sistema di ACL, Zookeeper/KRaft, un'intera settimana di setup operativo |
| NATS | Più leggero di Kafka, operatività più semplice | Comunque un processo separato, comunque una superficie di autenticazione separata, aggiunge comunque un hop di rete |
| Redis Pub/Sub | Redis è già in esecuzione | Lossy, nessun modello di backpressure, nessuna persistenza, si blocca su un subscriber lento |
| Channel in-process | Zero operatività, lossy per design (drop-on-full), deploy di un solo binario | Non può scalare oltre un processo |

La dashboard non ha bisogno di scalare oltre un processo. Undici collector alle cadenze elencate sopra producono ben meno di 100 eventi/sec di picco. Un channel Go con buffer di dimensione 512 è più che sufficiente. **Il "vincolo" del binario singolo è in realtà la feature** — l'operatività si riduce a `docker compose up` e un Justfile.

Se mai dovessi scalare (ad esempio, per offrire questo come prodotto SOC multi-tenant), sostituisci il bus con NATS JetStream e mantieni il resto. I collector e il frontend non cambiano.

## Gestione dei consumer lenti

Un subscriber WebSocket che smette di leggere è un rischio di denial-of-service per ogni altro subscriber. Se l'hub si blocca sulla `Write` del client lento, la latenza sale per tutti. Se l'hub mette in buffer senza limiti, la memoria cresce fino all'OOM.

Il pattern di questo progetto: **buffer limitato per subscriber, drop+disconnessione in caso di overflow**. L'hub dà a ogni subscriber un channel da 256 messaggi. In fase di invio, l'hub fa una select non bloccante. Se il channel è pieno, l'hub chiude in modo asincrono la connessione con `StatusPolicyViolation: slow consumer`. Il client si riconnette, ottiene uno snapshot fresco da REST e continua.

256 non è arbitrario. L'heartbeat scatta ogni 5s, Coinbase ha picchi di ~8 tick/sec, la ISS ogni 10s. Un buffer che contiene solo 16 messaggi si riempie durante una singola pausa del GC del browser e innesca un ciclo di riconnessione. 256 assorbe il jitter tipico senza mascherare i client davvero bloccati.

## Storage di serie temporali con BRIN

Le tabelle della dashboard sono prevalentemente append: gli eventi arrivano in modo monotono per `occurred_at`, e le query filtrano quasi sempre per un intervallo di tempo recente. Questo è il carico di lavoro per cui è stato costruito BRIN (Block Range Index).

Un indice BRIN memorizza riepiloghi per intervallo di blocchi (min/max della colonna indicizzata) invece di voci per riga. Per una colonna naturalmente correlata con l'ordine fisico delle righe (come è `occurred_at`, perché le righe arrivano in ordine temporale), BRIN è **da decine a centinaia di volte più piccolo di un B-tree** con prestazioni comparabili sulle scansioni per intervallo.

Compromesso: BRIN è ottimo per `WHERE ts > now() - interval '24h'` e inutile per `WHERE id = ?`. Le migration di questo progetto usano B-tree sulle chiavi primarie e BRIN sulle colonne temporali:

```sql
CREATE INDEX idx_quakes_time_brin ON earthquakes USING BRIN (occurred_at);
CREATE INDEX idx_quakes_mag       ON earthquakes (mag DESC, occurred_at DESC);
```

Il primo indice serve "mostrami i terremoti dell'ultima ora". Il secondo serve "mostrami i terremoti più forti dell'ultima settimana".

Dovresti ricorrere a BRIN prima di considerare il partitioning nativo. Il partitioning risolve problemi che BRIN non risolve (retention per partizione tramite `DROP PARTITION`, pruning parallelo), ma aggiunge complessità operativa. Per questa dimensione del dataset — misurata in decine di milioni di righe su mesi, non miliardi — BRIN da solo regge.

## Autenticazione: ES256 con chiavi a rotazione automatica

La firma dei JWT ha due forme praticabili per una dashboard esposta al pubblico:

- **HS256** (HMAC SHA-256, simmetrico). Semplice. Lo stesso segreto firma e verifica. Chiunque abbia il segreto può coniare token.
- **ES256** (ECDSA P-256, asimmetrico). La chiave privata firma. La chiave pubblica verifica. Puoi pubblicare liberamente la chiave pubblica (endpoint JWKS) così altri servizi possono verificare senza detenere il segreto di firma.

Questo progetto usa ES256 (Ed25519 in alcune note legacy — l'implementazione effettiva è ECDSA P-256 secondo la RFC 7518). I motivi:

- La dashboard espone `/.well-known/jwks.json`, così un futuro microservizio potrebbe verificare i token della dashboard senza fidarsi del filesystem della dashboard.
- La rotazione della chiave privata richiede solo di aggiornare il firmatario; i verificatori recuperano automaticamente il nuovo JWKS.
- Hash delle password Argon2id più token ES256 significa che un leak del database da solo non permette a un attaccante di coniare sessioni admin.

Il codice di bootstrap genera la coppia di chiavi se manca (vedi `ensureJWTKeys` in `cmd/api/main.go`). Per i deployment reali monti le chiavi da un secret store, ma la generazione automatica rende lo sviluppo privo di attriti.

## Cifratura dei segreti dei webhook

Gli alert configurabili (Slack, Discord, Telegram) richiedono di memorizzare URL di webhook e token di bot. Queste sono credenziali. Un database compromesso senza cifratura consegna all'attaccante il webhook di ogni utente che riceve alert, che può usare immediatamente per spam, phishing o per spingere link malevoli nei canali di comunicazione dell'utente.

Il package `notifications` avvolge le configurazioni dei webhook fornite dagli utenti con AES-256-GCM. La chiave di cifratura è un valore di 32 byte caricato da `NOTIFICATION_ENCRYPTION_KEY`. La cifratura produce `(ciphertext, nonce)`; entrambi vengono memorizzati, nessuno dei due viene loggato. La decifratura avviene su richiesta al momento dell'invio dell'alert e il testo in chiaro non viene mai salvato.

Se la variabile d'ambiente non è impostata, l'intero motore degli alert viene disabilitato con un warning nei log. **Fallire in modo chiuso** (fail closed) è la postura di sicurezza corretta qui: è meglio non avere alert che avere credenziali di webhook in chiaro nel database.

## Verifica la tua comprensione

Prima di passare all'architettura, dovresti essere in grado di rispondere a:

1. Perché il percentile EPSS è un segnale di triage migliore del CVSS per i primi 30 minuti di attenzione di un operatore?
2. Qual è la differenza tra un origin BGP hijack (MOAS) e un hijack con manipolazione del percorso? Quale dei due verrebbe colto in modo più affidabile da un feed Radar con punteggio di confidenza?
3. Perché il sistema mette in buffer i messaggi WebSocket sul client finché lo snapshot REST non ritorna, invece di connettersi e disegnare semplicemente gli eventi man mano che arrivano?
4. Qual è il reale limite di scala di un bus basato su channel Go in-process, e a che punto passeresti a NATS o Kafka?
5. Perché BRIN è la scelta giusta per le colonne `occurred_at` e la scelta sbagliata per le colonne di chiave primaria?
6. Cosa va storto se `NOTIFICATION_ENCRYPTION_KEY` è impostata a un valore di 16 byte invece di 32?
7. Perché la dashboard chiude i consumer WebSocket lenti invece di bufferizzare in modo più aggressivo?

Se qualcuno di questi punti non è chiaro, rileggi la sezione pertinente. Se hanno tutti senso, prosegui con [02-ARCHITECTURE.md](02-ARCHITECTURE.it.md).
