<!--
©AngelaMos | 2026
00-OVERVIEW.md
-->

# Monitor the Situation Dashboard

## Che cos'è

Una dashboard di situational awareness in tempo reale che fonde undici feed di intelligence live (firehose di mass-scan, BGP hijack, interruzioni di Internet, CVE fresche con punteggio EPSS, diff della CISA KEV, post di leak delle vittime ransomware, terremoti M2.5+, meteo spaziale NOAA, traccia orbitale della ISS, Wikipedia ITN, picchi GDELT, più i tick di BTC/ETH) in un'unica vista SOC. Un singolo binario Go esegue ogni collector come goroutine di un `errgroup`, fa passare gli eventi attraverso un bus in-process e li distribuisce (fan-out) ai browser via WebSocket.

L'espressione "monitoring the situation" è nata come meme su Twitter/X a metà 2025. Questa è la build che monitora davvero la situazione.

## Perché è importante

La maggior parte delle "dashboard" che vedi sono grafici Grafana statici puntati su un singolo job Prometheus. La vera situational awareness, quella che un SOC usa davvero, richiede di fondere feed estremamente diversi con cadenze estremamente diverse e di presentarli in modo coerente a un essere umano sotto stress. È difficile.

Alcuni incidenti reali che questo tipo di dashboard avrebbe colto minuti prima rispetto a leggere Twitter:

- **BGP hijack di Cloudflare di giugno 2024** da parte di Eletel (AS267613) per il traffico di Orange España. CF Radar ha pubblicato il punteggio di confidenza dell'hijack nel giro di pochi minuti. Una dashboard che interroga quell'endpoint lo fa emergere prima che il ciclo delle notizie lo raggiunga.
- **Log4Shell, dicembre 2021 (CVE-2021-44228)**. L'EPSS ha raggiunto 0.97 a poche ore dalla divulgazione, e l'inserimento nella KEV è seguito poco dopo. Un pannello di velocità delle CVE pesato per EPSS e diff della KEV è esattamente il modo in cui individui "questa conta" in un'alluvione di CVE.
- **Interruzione di CrowdStrike del 2024**. Il feed delle interruzioni di Cloudflare Radar ha segnalato cali anomali di connettività per ASN prima che qualcuno identificasse la causa principale. Un operatore con il pannello delle interruzioni aperto vede che qualcosa non va prima di sapere cosa.

Lo scopo di questo progetto è insegnare come si costruisce davvero un'infrastruttura di dati live per le operazioni di sicurezza: il pattern del collector, il pattern del bus, il fan-out WebSocket, e come mantenere tutto ciò osservabile e corretto attraverso tre ordini di grandezza di variazione delle cadenze (da meno di un secondo a giornaliera).

## Cosa imparerai

**Teoria di sicurezza e operazioni**

- **Economia dei feed di threat intel**: quali feed costano davvero (throttling della chiave API di NVD, lookup su AbuseIPDB), quali sono gratuiti ma fragili (scraping di DShield), quali sono commerciali ma ne valgono la pena (Cloudflare Radar). Quale budget di latenza ha ciascuno.
- **EPSS vs CVSS**: perché "CVSS critico 9.8" non è la stessa cosa di "questa verrà sfruttata domani" e perché il percentile EPSS è il segnale di prioritizzazione corretto per i primi 30 minuti di attenzione di un operatore.
- **CISA KEV**: cosa significa davvero Known Exploited Vulnerabilities, perché un alert sul diff della KEV è il singolo segnale più forte a cui un professionista possa iscriversi, e come si correla (o non si correla) con l'EPSS.
- **BGP hijack vs route leak**: come funziona il rilevamento MOAS (multiple origin AS), perché esiste il punteggio di confidenza e cosa rappresenta davvero un ASN nella tabella di routing globale.
- **Perché le trasformazioni di coordinate contano**: la posizione della ISS richiede la propagazione SGP4 a partire da un TLE, non un endpoint da interrogare, perché l'endpoint da interrogare è `wheretheiss.at` e la fonte dei TLE è CelesTrak, e si aggiornano con cadenze diverse. Mescolarli ingenuamente ti dà satelliti che si teletrasportano.

**Pattern dei sistemi distribuiti**

- **Event bus in-process** come alternativa pragmatica a Kafka o NATS quando la tua scala è "una macchina, undici producer, un fan-out". L'architettura a binario singolo è una feature, non un vincolo.
- **Backpressure e fan-out lossy**: quando scartare eventi, quando bloccare il producer e perché i subscriber WebSocket lenti non devono mai bloccare le goroutine dei collector.
- **Protocollo snapshot-then-stream**: come un browser che si connette ottiene lo stato attuale del mondo via REST e poi passa in modo pulito a WebSocket senza perdere né contare due volte gli eventi.
- **Indici BRIN per serie temporali prevalentemente append in Postgres**, e perché li scegli prima del partitioning.
- **Orchestrazione `errgroup` a binario singolo**: come avviare undici worker eterogenei, propagare la cancellazione e fare pulizia su `SIGTERM` senza scrivere un supervisore di processi.

**Autenticazione e crittografia**

- **JWT con Ed25519 (ES256) a rotazione automatica**: perché la firma asimmetrica deve stare davanti a qualsiasi dashboard pubblica, come funziona l'esposizione `JWKS` e perché ti permette di cambiare le chiavi senza riemettere ogni token attivo.
- **AES-256-GCM at rest** per i segreti dei webhook, così un leak del database non si trasforma in una compromissione Discord/Slack/Telegram di ogni utente che riceve alert.
- **Argon2id** per l'hashing delle password al posto di bcrypt, con la motivazione della scelta dei parametri.

**Telemetria frontend**

- Un globo 3D MapLibre con ombreggiatura delle interruzioni per paese, densità di punti per ASN per le sorgenti di mass-scan, marker delle vittime ransomware, epicentri dei terremoti e una traccia live della ISS propagata sul client.
- TanStack Query v5 + Zustand per la separazione snapshot/stream: REST per l'avvio a freddo, i push WebSocket nello stesso store, i componenti si iscrivono ai selettori.

## Prerequisiti

**Dovresti trovarti a tuo agio con:**

- **Go** — goroutine, channel, context, `errgroup`, il modello di cancellazione di `context.Context`. Se non hai mai scritto `select { case <-ctx.Done(): ... }` ti perderai.
- **Basi di PostgreSQL** — `CREATE INDEX`, `EXPLAIN`, colonne JSONB. Non pianificazione avanzata delle query, solo abbastanza per leggere una migration.
- **Fondamenti di HTTP e WebSocket** — come funziona l'handshake di upgrade WS, a cosa servono i frame ping/pong, cosa significa backpressure.
- **TypeScript e React** — hook, gestione dello stato. Non ti serve esperienza con MapLibre o D3; il progetto li usa ma i pattern sono spiegati.

**Utile ma non obbligatorio:**

- Intuizione su BGP e ASN (CIDR, lunghezza del prefisso, "cos'è AS15169"). Leggi la [documentazione BGP di Cloudflare Radar](https://developers.cloudflare.com/radar/) se vuoi un'introduzione.
- Astrodinamica SGP4/TLE per il pannello della ISS (il progetto usa `joshuaferrara/go-satellite` così non devi implementarla tu).
- CEL (Common Expression Language) per i predicati delle regole di alert. Il README di Google `cel-go` è il punto di partenza giusto.

**Strumenti:**

- Go 1.25+
- Node 20+ e `pnpm`
- Docker e Docker Compose
- `just` ([installazione](https://github.com/casey/just)) — ogni recipe è nel `Justfile`
- Opzionale: una chiave API di NVD (alza il budget di polling delle CVE da 5 req/30s a 50 req/30s) e un token API di Cloudflare (`CF_RADAR_TOKEN`) per i feed di Radar.

## Avvio rapido

```bash
cd PROJECTS/advanced/monitor-the-situation-dashboard

# Optional: fill in API keys (nothing is required to boot)
cp .env.example .env

# Bring up postgres, redis, backend, frontend in dev mode
just dev-up

# In another shell, run migrations
just migrate
```

Visita `http://localhost:8432`. Dovresti vedere il globo inizializzarsi, poi i pannelli popolarsi uno alla volta man mano che il loro primo ciclo di polling restituisce dati. Il pannello heartbeat diventa verde entro cinque secondi.

Se vedi una fascia `NO SIGNAL`, il WebSocket è fallito. Controlla `just logs backend` — la causa più comune è `notifications: encryption key not set`, che disabilita il motore degli alert ma non la dashboard in sé.

## Output atteso

Entro circa un minuto da `just dev-up`:

- **Pannello del globo** in rotazione, con la traccia della ISS disegnata in un'orbita polare. I punti dei terremoti compaiono man mano che USGS li pubblica (tipicamente 5-30 all'ora in tutto il mondo sopra M2.5).
- **Pannello di velocità delle CVE** popolato con le ultime due ore di voci NVD, ordinate per EPSS. Diverse righe con il tag `KEV` se qualcuna è entrata di recente nel catalogo.
- **Pannello DShield** che mostra gli ASN che fanno più scansioni. Se hai aspettato l'ora intera per il primo polling, il pannello mostra le prime 10 sorgenti di mass scanning per volume di segnalazioni.
- **Pannelli BTC/ETH** che si aggiornano live (Coinbase ha la cadenza più bassa del sistema, sotto il secondo).
- **Pannello delle interruzioni** tranquillo la maggior parte dei giorni. Cloudflare Radar fa emergere qualcosa di interessante più o meno una volta a settimana.

## Dove andare dopo

Questo file serve per orientarsi. Il vero materiale didattico è suddiviso tra:

| File | Focus | Tempo di lettura |
|---|---|---|
| [01-CONCEPTS.md](01-CONCEPTS.it.md) | Teoria della threat intel: EPSS, KEV, BGP hijack, firehose di mass-scan, la situational awareness come disciplina | 20 min |
| [02-ARCHITECTURE.md](02-ARCHITECTURE.it.md) | Pipeline di collector a binario singolo, event bus, snapshot store, fan-out WebSocket, storage di serie temporali con BRIN | 30 min |
| [03-IMPLEMENTATION.md](03-IMPLEMENTATION.it.md) | Walkthrough del codice attraverso collector, bus, ws, motore degli alert, REST degli snapshot, collegamento dello store frontend | 45 min |
| [04-CHALLENGES.md](04-CHALLENGES.it.md) | Idee di estensione, feed aggiuntivi da collegare, miglioramenti al motore degli alert, hardening del deployment | 15 min |

Leggili in ordine se è il tuo primo incontro con il progetto. Salta a `03-IMPLEMENTATION` se hai già fatto il deploy di sistemi simili e vuoi confrontare i pattern.
