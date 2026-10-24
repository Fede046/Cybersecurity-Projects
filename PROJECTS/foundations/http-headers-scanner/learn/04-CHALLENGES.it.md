# Sfide di estensione

Hai letto il codice e capisci cosa fa. È il momento di renderlo tuo.

Queste sfide sono ordinate approssimativamente per difficoltà. Le prime sono modifiche da 30 minuti. Le ultime sono progetti da weekend che cambiano in modo significativo il funzionamento dello scanner. Nessuna di esse ha "la risposta giusta" nascosta in questa cartella. Provale, sbagliale, falle giuste, impara qualcosa.

Se ti blocchi, la fonte di verità è sempre il codice. Rileggi `evaluate_header()`, `scan()` e `_render_report()`. L'intero scanner sta in uno schermo.

## Sfide di riscaldamento

### 1. Aggiungere un settimo header da controllare

**Obiettivo:** estendi `RULES` con un altro security header. Scegline uno tra:

- **`Cross-Origin-Opener-Policy`** (COOP): impedisce che altre origini interagiscano con la pagina tramite window.opener. Valore raccomandato: `same-origin`.
- **`Cross-Origin-Embedder-Policy`** (COEP): controlla quali risorse cross-origin possono essere caricate. Valore raccomandato: `require-corp`.
- **`Cross-Origin-Resource-Policy`** (CORP): controlla chi può incorporare questa risorsa. Valore raccomandato: `same-origin`.

**Perché è utile.** Questi tre header insieme (COOP, COEP, CORP) sono il modo moderno per isolare una pagina dagli attacchi cross-origin come Spectre. I siti reali ad alta sicurezza li impostano tutti e tre.

**Suggerimenti.**
- Aggiungi un `HeaderRule(...)` a `RULES`. Scegli una gravità. Decidi se ti serve un pattern `must_match` (una semplice parola come `"same-origin"` funziona come controllo di sottostringa; usa una regex più ricca se conta che "il valore debba essere esattamente una tra N opzioni").
- I totali del punteggio cambieranno. I test esistenti che verificano punteggi esatti potrebbero rompersi. Aggiorna quei test oppure scrivili in termini di percentuali.
- Provalo su un sito reale. `https://web.dev` è un buon candidato da controllare.

**Completato quando.** `just test` passa e `just run -- https://web.dev` mostra il tuo nuovo header nella tabella.

### 2. Aggiungere un flag `--json` per un output leggibile dalle macchine

**Obiettivo.** Fai in modo che lo scanner emetta un blob JSON quando l'utente passa `--json`, invece della tabella colorata.

**Perché è utile.** I sistemi di CI e le dashboard hanno bisogno di output strutturato. Una tabella adatta a grep va bene per gli umani, il JSON va bene per tutti gli altri.

**Suggerimenti.**
- Aggiungi un flag booleano `--json` a `_build_argument_parser()` usando `action="store_true"`.
- In `main()`, dopo che `scan()` restituisce il risultato, crea un ramo su `args.json`. Se è True, usa il modulo `json` della libreria standard per serializzare un dict con i campi del report. Se è False, chiama l'esistente `_render_report()`.
- `HeaderFinding` non è direttamente serializzabile in JSON perché contiene un `HeaderRule` annidato. Puoi appiattirlo manualmente, oppure usare `dataclasses.asdict()`, che converte un albero di dataclass frozen in dict annidati.

**Completato quando.** `just run -- https://example.com --json | jq '.score'` stampa solo il numero del punteggio.

### 3. Aggiungere un flag `--verbose` che stampa i response header grezzi

**Obiettivo.** Un flag opzionale che, quando è impostato, stampa sopra la tabella ogni header restituito dal server (inclusi quelli non legati alla sicurezza).

**Perché è utile.** Quando fai debug di "perché lo scanner dice che il mio HSTS è weak", vuoi vedere il valore grezzo inviato dal server.

**Suggerimenti.**
- Aggiungi il flag allo stesso modo di `--json`.
- La funzione `scan()` memorizza solo i finding, non la response grezza. Dovrai far passare gli header grezzi attraverso `ScanReport` (aggiungendo un nuovo campo) oppure fare in modo che `scan()` restituisca una tupla.
- Stampa gli header grezzi in `_render_report` quando viene passato un flag verbose. La firma del renderer dovrà essere aggiornata.

**Completato quando.** `just run -- https://example.com --verbose` mostra tutti gli header del server sopra la tabella dei finding.

## Sfide intermedie

### 4. Scansionare più URL in una sola esecuzione

**Obiettivo.** Permetti all'utente di passare diversi URL e di ottenere un report per URL. Bonus: una tabella riassuntiva alla fine.

**Perché è utile.** Quando fai l'audit dei siti di un'azienda, non vuoi eseguire lo scanner una volta per ogni URL. Eseguilo una volta sull'intero elenco.

**Suggerimenti.**
- Cambia l'argomento argparse `url` in modo che accetti `nargs="+"` (uno o più). `args.url` diventa una lista di stringhe.
- Cicla sugli URL in `main()`. Chiama `scan()` per ciascuno, esegui il rendering di ogni report, raccoglili.
- Per il riepilogo, aggiungi una piccola tabella finale con una riga per URL: URL, punteggio, voto.
- L'exit code diventa interessante. Probabilmente il voto peggiore tra tutti gli URL.

**Casi limite.**
- Un URL fallisce (errore di rete) ma gli altri hanno successo. Esci comunque con codice 2?
- Due URL restituiscono lo stesso URL finale (dopo i redirect). Li deduplichi o li mostri entrambi?

### 5. Aggiungere un flag di soglia in stile `--allow-warnings`

**Obiettivo.** Permetti all'utente di dire "per me il voto C va bene, esci con un codice diverso da zero solo se il voto è D o inferiore."

**Perché è utile.** Integrazioni con la CI. Team diversi hanno soglie di accettazione diverse.

**Suggerimenti.**
- Aggiungi `--min-grade` che accetta un valore da `{"A", "B", "C", "D", "F"}`. Imposta come predefinito `"C"` (all'incirca il comportamento attuale).
- In `main()`, confronta `report.grade` con la soglia e scegli l'exit code di conseguenza.
- I voti hanno un ordine naturale. Puoi confrontarli con un piccolo helper che associa ciascuno a un intero.

**Completato quando.** `just run -- https://example.com --min-grade C` esce con 0 se il sito ha ottenuto C o meglio, altrimenti esce con 1.

### 6. Mettere in cache i risultati su disco

**Obiettivo.** Quando l'utente riscansiona lo stesso URL entro l'ultima ora, restituisci il risultato in cache invece di interrogare la rete.

**Perché è utile.** Quando stai iterando sul renderer o sul punteggio, non vuoi bombardare un sito reale a ogni esecuzione dei test.

**Suggerimenti.**
- Memorizza i report in cache in `~/.cache/http-headers-scanner/`.
- Nome del file ricavato da un hash dell'URL (`hashlib.sha256(url.encode()).hexdigest()`).
- Usa `dataclasses.asdict(report)` per convertirlo in JSON e `json.dump()` per scriverlo.
- Includi un timestamp nel file in cache. Ignora la cache quando è più vecchia di un'ora.
- Aggiungi un flag `--no-cache` per aggirarla.

**Parte delicata.** `HeaderFinding` contiene un `HeaderRule`. Quando carichi dalla cache, devi ricostruire quelle regole dal JSON. Oppure, più semplicemente, metti in cache solo le parti che ti interessano e ricostruisci da zero la forma del report.

### 7. Rilevare e segnalare il mix HTTP/HTTPS

**Obiettivo.** Se l'utente passa un URL `http://` e il server reindirizza a `https://`, segnalalo in modo evidente nell'output.

**Perché è utile.** Molti siti impongono HTTPS tramite redirect, ma la catena di redirect stessa non è cifrata nel primo passaggio e viene manomessa dagli attaccanti (è proprio ciò da cui protegge HSTS). Sapere se è avvenuto un redirect è un segnale utile.

**Suggerimenti.**
- Dopo `scan()`, confronta `report.url` (quello che l'utente ha digitato) con `report.final_url` (dove è arrivato).
- Se l'utente ha digitato `http://...` e l'URL finale è `https://...`, stampa una nota di una riga: "Note: this URL was upgraded from HTTP to HTTPS via redirect. Without HSTS, the first request is vulnerable to interception."
- Meglio ancora, togli punti se HSTS manca E l'utente è arrivato tramite http.

## Sfide avanzate

### 8. Renderlo async per scansionare molti siti in parallelo

**Obiettivo.** Usa `httpx.AsyncClient` più `asyncio.gather` in modo che scansionare 50 URL richieda circa il tempo di una sola scansione, non di 50.

**Perché è utile.** Gli audit reali coprono molti host. La scansione sequenziale è il collo di bottiglia.

**Suggerimenti.**
- Aggiungi una `async def scan_async(url, ...)` accanto a `scan()`. Usa `async with httpx.AsyncClient() as client:` e `client.get(...)`.
- In `main()`, costruisci un `asyncio.gather()` su una lista di chiamate a `scan_async`.
- Limite di concorrenza: non sparare 5000 URL tutti insieme. Usa `asyncio.Semaphore(20)` per limitare il parallelismo.
- Le funzioni pure (`evaluate_header`, `score`, `grade`) non cambiano. È questo il vantaggio di separare la logica pura dall'I/O.

**Attenzione a.** Alcuni siti applicano rate limiting alle scansioni aggressive. Lo User-Agent predefinito ci identifica; rispetta eventuali header `Retry-After` se vengono restituiti.

### 9. Aggiungere un analizzatore statico per il valore della CSP

**Obiettivo.** Attualmente controlliamo solo che `Content-Security-Policy` sia presente. Costruisci un sotto-analizzatore che guardi dentro il valore della CSP e segnali problemi specifici:

- Contiene `'unsafe-inline'` in `script-src`? Debolezza grave, punti in meno.
- Contiene un'origine wildcard (`*`) in `script-src`? Idem.
- Manca `default-src`? Vale la pena segnalarlo.

**Perché è utile.** Una CSP che consente `'unsafe-inline'` è a malapena una CSP. Gli scanner reali (Mozilla Observatory) fanno questa analisi. Anche il tuo può farla.

**Suggerimenti.**
- Aggiungi una nuova dataclass `CSPAnalysis` con campi come `has_unsafe_inline: bool`, `wildcard_origins: list[str]`, ecc.
- Aggiungi una funzione pura `parse_csp(value: str) -> CSPAnalysis`. Grammatica della CSP: le direttive sono separate da punto e virgola; ogni direttiva è `name source1 source2 ...`.
- Fai in modo che lo status `weak` della regola CSP tenga conto del risultato dell'analisi. L'`evaluate_header` di base avrà bisogno di una via d'uscita per le regole che hanno un validatore personalizzato.

**Parte delicata.** La CSP è davvero complessa. La specifica ufficiale si trova in `w3.org/TR/CSP3`. Inizia facendo il parsing solo di `script-src`; ignora il resto.

### 10. Costruire un monitor continuo

**Obiettivo.** Esegui lo scanner su un elenco di URL ogni 24 ore e invia un avviso quando il voto di un sito peggiora.

**Perché è utile.** La configurazione deriva nel tempo. Un sito che sei mesi fa aveva voto A può scendere a B perché qualcuno ha disattivato HSTS per fare debug di qualcosa e si è dimenticato di riattivarlo. Vuoi saperlo.

**Suggerimenti.**
- Ti serve una memorizzazione persistente. L'inizio più semplice: un database SQLite con le colonne `(url, run_at, score, grade)`. La libreria standard ha `sqlite3`, quindi nessuna nuova dipendenza.
- Uno script separato legge un elenco di URL (un URL per riga, in un file), esegue lo scanner, scrive i risultati.
- Confronta il voto di oggi con il voto precedente più recente per ogni URL. Se oggi è peggiore, emetti un avviso (stampa su stdout, invia un'email, pubblica un messaggio su Slack, a tua scelta).
- Una voce di `cron` o un timer di systemd avvia lo script una volta al giorno.

**Attenzione a.** Le reti sono inaffidabili. Un timeout in un giorno non significa necessariamente un peggioramento del voto. Probabilmente vorrai una regola del tipo "due fallimenti consecutivi" prima di inviare un avviso.

### 11. Confronto basato sui browser: scansionare lo stesso URL con tre User-Agent diversi

**Obiettivo.** I siti a volte servono header diversi ai bot rispetto ai browser reali. Aggiungi una modalità `--compare-uas` che scansiona lo stesso URL con lo UA predefinito dello scanner, con uno UA di Chrome e con uno UA di Googlebot, poi mostra un confronto affiancato degli header.

**Perché è utile.** Un sito potrebbe servire una CSP restrittiva agli utenti reali ma una CSP permissiva ai bot, o viceversa. Saperlo è una prova concreta di una configurazione errata.

**Suggerimenti.**
- La funzione `scan()` accetta già un argomento `user_agent`. Eseguila tre volte con tre valori diversi.
- Uno UA reale di Chrome ha questo aspetto: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36`.
- Lo UA di Googlebot è `Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)`.
- Il layer di rendering ha bisogno di una modalità di "confronto a tre colonne". Una Table di rich con le colonne "Default", "Chrome", "Googlebot" e una riga per header.

## Sfida per esperti

### 12. Trasformarlo in un servizio con una REST API

**Obiettivo.** Avvolgi lo scanner in un piccolo server HTTP. Fai una POST con un URL, ricevi indietro un report JSON.

**Perché è utile.** Altri servizi in una pipeline di sicurezza possono chiamare il tuo. Inoltre, è il modo naturale per rendere lo scanner accessibile da un frontend.

**Tempo stimato.** Un weekend se non hai mai usato FastAPI, qualche ora se l'hai già usato.

**Prerequisiti.** Un po' di familiarità con i framework web sarebbe d'aiuto. Il progetto intermedio `siem-dashboard` in questo repo è un esempio più approfondito di come costruirne uno.

**Bozza dell'architettura.**

```
┌─────────────────────────────────────────────────────┐
│ FastAPI app                                          │
│                                                      │
│  POST /scan { "url": "..." }                         │
│       └──▶ async scan_async(url) ────▶ ScanReport    │
│       ◀── { "url": ..., "grade": ..., findings: ... }│
│                                                      │
│  GET /healthz                                        │
│       └──▶ {"ok": true}                              │
└─────────────────────────────────────────────────────┘
```

**Passi.**

1. **Aggiungi fastapi e uvicorn alle dipendenze.** `uv add fastapi uvicorn[standard]`.
2. **Crea un nuovo file `server.py`** che importi `scan_async` dalla versione async del tuo scanner.
3. **Definisci un modello Pydantic** per il body della richiesta (`class ScanRequest(BaseModel): url: HttpUrl`).
4. **Definisci un modello di risposta** che rispecchi `ScanReport` per la serializzazione. `HttpUrl` valida gli URL al confine dell'API.
5. **Aggiungi la route.** `@app.post("/scan", response_model=ScanResponse)` poi `async def scan_endpoint(req: ScanRequest)`.
6. **Aggiungi un endpoint healthz** così i monitor esterni possono confermare che il servizio è attivo.
7. **Aggiungi il rate limiting.** Altrimenti la gente userà il tuo servizio per scansionare sconosciuti. Il progetto avanzato `api-rate-limiter` in questo repo è un buon riferimento.

**Checklist per la produzione.**
- [ ] Validazione: rifiuta gli IP interni (`127.0.0.1`, `10.0.0.0/8`, ecc.) per prevenire SSRF.
- [ ] Timeout: ogni scansione ha un limite rigido, così un target lento non può tenere occupati i worker.
- [ ] Logging: ogni richiesta viene registrata con `url`, `grade`, `duration_ms`.
- [ ] CORS: decidi quali origini possono chiamare la tua API.
- [ ] Deploy: un Dockerfile che esegue uvicorn, una variabile d'ambiente per la porta.

**Obiettivo extra.** Costruisci un minuscolo frontend single-page (HTML + JS vanilla, nessun framework necessario) che permetta a un utente di incollare un URL e mostri il report. Ora hai un prodotto vero.

## Altre direzioni

Alcune idee più piccole se nessuna delle precedenti ti attira:

- **Modalità per daltonici.** Sostituisci i colori verde/giallo/rosso con verde/giallo/rosso più forme distinte (✓, ⚠, ✗) così la tabella è leggibile per gli utenti daltonici.
- **Importazione di file HAR.** Invece di scansionare un URL live, leggi gli header da un file `.har` (HTTP Archive, quello che i browser esportano dai dev tools). Ti permette di scansionare response catturate in precedenza.
- **Mostrare le differenze rispetto a una baseline nota e corretta.** Salva un "report di riferimento" per un URL. Alla scansione successiva, mostra solo le modifiche ("HSTS è diventato più debole", "X-Frame-Options è scomparso").
- **Aggiungere una modalità `--strict`** che alza l'asticella: toglie punti anche per problemi minori (ad es. `max-age` inferiore a sei mesi, `includeSubDomains` mancante).

## Cosa fare quando sei bloccato

La strategia che funziona ogni singola volta:

1. **Riduci.** Riduci il problema all'osso. Commenta tutto tranne il pezzo più piccolo che continua a comportarsi male.
2. **Stampa.** Quando `evaluate_header` non restituisce ciò che ti aspettavi, stampa gli input e gli output. Non tirare a indovinare come sono fatti. Guardali.
3. **Leggi il test.** Se un test fallisce, il test ti sta dicendo esattamente cosa si aspettava. Leggi l'assertion. Leggi gli input. Confronta con l'output effettivo.
4. **Fai il diff rispetto a una versione funzionante.** Usa `git diff` per vedere cosa è cambiato dall'ultima volta che i test sono passati. Il bug è nelle tue modifiche il 99% delle volte.

Se dopo tutto questo hai ancora bisogno di aiuto, la scheda GitHub Discussions del repository è il posto giusto. Porta:
- Cosa hai provato.
- Cosa ti aspettavi.
- Cosa è successo davvero (l'errore completo, se c'è).
- Il riproduttore più piccolo che mostra il problema.

"Non funziona" non è un'informazione sufficiente per poterti aiutare. I quattro punti qui sopra sì.

## Quando completi una sfida

Rendila tua. Se costruisci qualcosa di interessante, scrivine. Fai il push sul tuo fork. Apri una pull request verso il repo principale se il tuo miglioramento è utile in generale. Il modo migliore per consolidare ciò che hai imparato è insegnarlo a qualcun altro.
