# Architettura

Questo file spiega come è organizzato il codice e perché. È il ponte tra "ho capito cosa sono gli header HTTP" (il file precedente) e "so leggere esattamente questo codice Python" (il file successivo). Leggilo dopo i concetti e prima dell'implementazione.

L'intero scanner è un solo file Python più un file di test. Meno di 700 righe in totale. È abbastanza piccolo da farti chiedere perché usiamo proprio la parola "architettura". La risposta: anche i programmi piccoli traggono beneficio dall'essere suddivisi in parti, e il MODO in cui vengono suddivisi conta. Uno scanner reale in produzione finirà per crescere fino a molti file. Imparare come appare la suddivisione a queste dimensioni rende meno spaventosi quelli più grandi.

## 1. Il quadro generale

Lo scanner è una pipeline. I dati scorrono da sinistra a destra attraverso quattro fasi:

```
   ┌────────┐  URL  ┌──────────┐  bytes   ┌────────────┐  findings  ┌──────────┐
   │  User  │ ────▶ │  scan()  │ ──────▶  │ evaluate_  │ ────────▶  │  render  │
   │  CLI   │       │  fetches │          │  header()  │            │  report  │
   └────────┘       └──────────┘  loop    │   ×6       │            └──────────┘
                                          └────────────┘
```

1. **Layer CLI.** L'utente digita `headers https://example.com`. `argparse` lo trasforma in un normale oggetto Python con `args.url` e `args.timeout`.
2. **Layer di rete (`scan()`).** Fa una richiesta HTTPS, segue i redirect, restituisce gli header grezzi come dict.
3. **Layer di valutazione (`evaluate_header()`).** Funzione pura. Prende una regola e i response header, restituisce un finding. Viene chiamata una volta per regola. Niente rete. Niente stampa.
4. **Layer di rendering (`_render_report()`).** Prende il report, stampa una tabella colorata più il pannello del voto più le raccomandazioni.

Il motivo per cui è strutturato così: ogni fase è testabile in modo indipendente. Possiamo passare a `evaluate_header()` un dict finto di header e controllare il finding, senza mai toccare internet. Possiamo costruire uno `ScanReport` finto e controllare punteggio e voto, senza eseguire affatto `scan()`.

Se una qualsiasi di queste fasi fosse mescolata con le altre (per esempio, se `scan()` stampasse direttamente il suo output e la matematica vivesse dentro la logica di stampa), testare significherebbe avviare ogni volta un web server reale o finto. Separarle è l'unico motivo per cui puoi scrivere 20 test che girano in meno di un secondo.

## 2. Le quattro forme di dati principali

Usiamo le **dataclass** per i nostri dati. Una dataclass è semplicemente una classe per cui Python scrive il boilerplate al posto tuo. Invece di questo:

```python
class HeaderRule:
    def __init__(self, header, severity, description, recommendation, must_match=None):
        self.header = header
        self.severity = severity
        self.description = description
        self.recommendation = recommendation
        self.must_match = must_match
```

Scrivi questo:

```python
@dataclass(frozen=True, slots=True)
class HeaderRule:
    header: str
    severity: Severity
    description: str
    recommendation: str
    must_match: str | None = None
```

Il decorator `@dataclass` in cima legge le dichiarazioni dei campi e genera `__init__` per te. Genera anche l'uguaglianza (`==`), una rappresentazione come stringa e alcune altre cose utili.

Due flag che vale la pena capire:

- **`frozen=True`** rende la dataclass **immutabile**. Una volta creato un `HeaderRule`, non puoi fare `rule.severity = "low"`. Tentare di modificare un campo solleva un errore. È utile per i tipi "value object" in cui vuoi la garanzia che nessun'altra parte del codice possa modificarli alle tue spalle.
- **`slots=True`** rende le istanze **più piccole in memoria**. Senza slots, ogni oggetto Python si porta dietro un dizionario nascosto per i suoi attributi, che è flessibile ma usa più memoria. Con slots, gli attributi vanno in slot fissi, nessun dict. Per un tipo record con un insieme noto di campi, è un vantaggio puro. Non ti costa nulla.

Abbiamo quattro forme in totale:

### 2.1 `HeaderRule`: "cosa stiamo cercando"

Una regola è una cosa da controllare. Nome dell'header, gravità, descrizione (per gli umani), una raccomandazione (cosa impostare se manca) e una regex `must_match` opzionale (per header come `X-Content-Type-Options` in cui il *valore* deve essere corretto, non solo presente: una semplice parola come `"nosniff"` funziona come controllo di sottostringa, mentre un pattern più ricco come `r"max-age\s*=\s*[1-9]"` rifiuta i valori HSTS che si disattivano da soli).

Questa è la regola che lo scanner percorre. L'intero elenco di regole vive in una costante a livello di modulo chiamata `RULES`. Aggiungere un nuovo controllo su un header significa aggiungere un elemento a quell'elenco. Il resto del codice è generico.

### 2.2 `HeaderFinding`: "cosa abbiamo trovato"

Un finding è il risultato dell'esecuzione di una regola sulla response del server. Contiene:

- La regola da cui proviene (così il renderer può mostrare gravità, raccomandazione, ecc. senza fare una seconda ricerca).
- Uno `status`: `ok`, `weak` o `missing`.
- L'`actual_value` inviato dal server (o `None` se l'header mancava).
- Una breve `note` leggibile che descrive cosa è successo ("Present", "Present and contains `nosniff`", "Header `X` is not set", ecc.).

Anche i finding sono frozen. Una volta valutata una regola, il risultato non cambia. Questo rende sicuro passare il report a più funzioni senza preoccuparsi che una di esse lo modifichi.

### 2.3 `ScanReport`: "il risultato completo"

Un report racchiude tutto ciò che riguarda una scansione. L'URL originale (quello digitato dall'utente), l'URL finale (dopo i redirect), lo status code HTTP e l'elenco dei finding (uno per regola).

La parte interessante: `score` e `grade` sono **proprietà calcolate**, non campi memorizzati. Sono funzioni decorate con `@property` che esaminano i finding al volo. Questo significa che:

- Non dobbiamo ricordarci di ricalcolarli quando i finding cambiano (non possono cambiare, il report è frozen, ma il principio resta valido).
- Un test può costruire un report con un insieme sintetico di finding e chiedere immediatamente `report.score`, senza alcun collegamento aggiuntivo.

### 2.4 I tipi `Severity` e `Status`

Questi sono **tipi Literal**:

```python
Severity = Literal["high", "medium", "low"]
Status = Literal["ok", "weak", "missing"]
```

Un tipo `Literal` dice al type checker "questa è una stringa, ma può essere solo uno di questi valori esatti." Perché preoccuparsene? Due motivi:

1. **Protezione dagli errori di battitura.** Se da qualche parte scrivi `severity = "hgih"`, mypy lo intercetta prima ancora che tu esegua il codice. Senza `Literal`, il tipo sarebbe semplicemente `str` e qualsiasi errore di battitura passerebbe inosservato.
2. **Documentazione.** Ti basta guardare la firma del tipo per conoscere i valori ammessi. Non devi fare grep nel codice per scoprirlo.

Avremmo potuto usare un `Enum`. La style guide di Carter preferisce `Literal` per piccoli insiemi fissi perché mantiene i valori come semplici stringhe (facili da stampare, facili da loggare, facili da usare come chiavi di un dict).

## 3. Separazione dei layer: il recinto dell'I/O

La decisione architetturale più importante in assoluto di questo progetto è il **recinto tra il codice che tocca la rete e il codice che non lo fa**. Seguiamolo.

```
┌─────────────────────────────────────────────────────────────────────┐
│  THE I/O LAYER                                                      │
│  - scan()                       (calls httpx.get, hits the network) │
│  - main()                       (reads sys.argv, calls scan)        │
│  - _render_report()             (writes to the terminal)            │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                │  passes a dict[str, str] or a ScanReport
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  THE PURE LAYER                                                     │
│  - evaluate_header(rule, headers) -> HeaderFinding                  │
│  - ScanReport.score property                                        │
│  - ScanReport.grade property                                        │
│  - RULES list, SEVERITY_POINTS dict                                 │
└─────────────────────────────────────────────────────────────────────┘
```

Tutto ciò che sta nel **layer puro**:
- Prende input.
- Restituisce output.
- Non tocca la rete. Non tocca il filesystem. Non stampa nulla.

Tutto ciò che sta nel **layer di I/O**:
- Parla con il mondo esterno.
- Alla fine passa i suoi risultati al layer puro.

Perché è importante: il layer puro è **banalmente testabile**. Costruisci gli input a mano, controlli gli output. Nessun mocking necessario. Il layer di I/O è più difficile da testare (devi fare il mock della rete o del terminale), ma è anche la parte piccola. La maggior parte dei bug in uno scanner sta nella matematica, non nella chiamata a httpx. Mettere la matematica in una funzione pura significa che testi la matematica con sicurezza e il codice di rete con solo pochi smoke test.

Questo a volte viene chiamato **functional core, imperative shell**. Espressione coniata da Gary Bernhardt. Stessa idea: mantieni pura la parte che decide cosa fare e spingi ai bordi la parte che lo fa davvero.

## 4. Il flusso dei dati, dall'inizio alla fine

Seguiamo una scansione dal terminale all'output. L'utente digita:

```
$ just run -- https://github.com
```

```
1. SHELL → main()
   ─────────────────────────────────────────────────────
   `just run` calls `uv run headers https://github.com`.
   `uv run` activates the venv, runs `headers` which is
   declared in pyproject.toml as the entry point that
   maps to http_headers_scanner:main.
   sys.argv is now ["headers", "https://github.com"].

2. main() → _build_argument_parser()
   ─────────────────────────────────────────────────────
   We construct an argparse parser, add the `url` arg
   and the `--timeout` option, then call parse_args().
   Result: args.url = "https://github.com", args.timeout = 10.0

3. main() → scan(url, timeout)
   ─────────────────────────────────────────────────────
   scan() calls httpx.get() with follow_redirects=True
   and a custom User-Agent. httpx does DNS, opens a TCP
   connection, negotiates TLS, sends the GET request,
   reads the response, follows any redirects. Returns a
   Response object.

4. scan() → response_headers (a dict[str, str])
   ─────────────────────────────────────────────────────
   We convert httpx's Headers object to a plain dict.
   This is the "leave the I/O world, enter the pure
   world" handoff. Past this point, nothing knows or
   cares about httpx.

5. scan() → [evaluate_header(rule, headers) for rule in RULES]
   ─────────────────────────────────────────────────────
   A list comprehension. Run evaluate_header() once for
   each of the six rules. Each call is pure: it looks
   up the header in the dict (case-insensitive), checks
   must_match (via re.search, case-insensitive) if set,
   returns a HeaderFinding.

6. scan() → ScanReport
   ─────────────────────────────────────────────────────
   Bundle the URL, final URL, status code, and findings
   into a frozen ScanReport. scan() is done.

7. main() → _render_report(report, console)
   ─────────────────────────────────────────────────────
   The renderer builds a rich Table, adds one row per
   finding, prints it. Then builds the grade Panel,
   prints it. Then iterates over non-ok findings and
   prints their recommendations.

8. main() → exit code
   ─────────────────────────────────────────────────────
   Look at report.grade. A or B → return 0 (success).
   C or D → return 1 (warning). F or network error → 2.
   sys.exit(main()) sends the code to the shell.
```

La cosa fondamentale da notare: i passi 5 e 6 sono puri. Se vuoi scrivere un test che eserciti la logica di punteggio e dei finding, salti completamente il passo 3 e chiami direttamente `evaluate_header()` con input costruiti a mano. È esattamente ciò che fa `test_http_headers_scanner.py`.

## 5. Perché ogni funzione ha la dimensione che ha

Una domanda comune tra i principianti: come fai a sapere quando dividere un pezzo di codice in una nuova funzione?

Una regola pratica utile: **un compito per funzione**. Se riesci a descrivere cosa fa una funzione in una frase senza "e", probabilmente ha la dimensione giusta. Se devi dire "questa recupera l'URL E fa il parsing degli header E li valuta E stampa la tabella", è troppo grande.

Guarda le nostre funzioni attraverso questa lente:

- **`evaluate_header()`**: "Applica una regola a un insieme di header e restituisce un finding." Un compito.
- **`scan()`**: "Recupera un URL e restituisce un report." Un compito. Chiama anche `evaluate_header()` internamente, ma quella è delega, non un secondo compito.
- **`_render_report()`**: "Stampa in modo leggibile un report nel terminale." Un compito.
- **`_build_argument_parser()`**: "Costruisce il parser argparse." Un compito. Vale la pena separarla così i test possono chiamarla senza avviare main().
- **`main()`**: "Unisce le altre e sceglie un exit code." Un compito: l'orchestrazione.

Il prefisso underscore su `_render_report` e `_build_argument_parser` è una convenzione di Python che significa "questo è privato, non fa parte dell'API pubblica." L'altro codice nello stesso file può comunque chiamarle, ma chiunque importi il modulo dovrebbe considerarle interne.

## 6. L'unica fonte di verità: l'elenco `RULES`

Nota che `RULES` è definito una sola volta, in cima al file, come elenco di oggetti `HeaderRule`. Le funzioni di punteggio, la funzione di valutazione e il renderer percorrono tutte questo elenco a runtime. Nessuna di esse ha una conoscenza scritta nel codice di quali header esistano.

Cosa ci fa guadagnare: **aggiungere un settimo header da controllare è una modifica di una riga**. Aggiungi un `HeaderRule` all'elenco. Lo scanner lo rileva automaticamente. La test suite lo rileva automaticamente (l'helper per i report sintetici usa direttamente `RULES`). Nessun codice in `scan()`, nel rendering o nel punteggio deve cambiare.

Questo è ciò che si intende quando si parla di codice "data driven". Il comportamento è determinato dai dati (la tabella delle regole), non da logica scritta nel codice caso per caso. È anche uno dei pattern più facili da riconoscere una volta che inizi a cercarlo.

Lo stesso pattern, con gli stessi vantaggi, compare negli scanner reali in produzione:
- Nuclei (un vuln scanner) legge template YAML che assomigliano molto al nostro HeaderRule, solo più grandi.
- I plugin di ESLint sono per lo più tabelle di regole.
- Gli script NSE di Nmap sono singoli file di regole in una directory.

## 7. Perché usiamo httpx e non requests

`requests` è la famosa libreria HTTP di Python. `httpx` è quella più recente. Hanno API molto simili. Abbiamo scelto httpx perché:

- **Type hint di prima classe.** mypy capisce httpx senza configurazioni aggiuntive. requests richiede ancora gli stub `types-requests`.
- **Supporto HTTP/2.** I siti più recenti parlano HTTP/2 per impostazione predefinita. requests supporta solo HTTP/1.1.
- **Pronto per l'async.** httpx ha un'API sincrona (quella che usiamo) e un'API async per quando crescerai.
- **Manutenzione attiva.** requests è in modalità manutenzione. httpx è dove avviene il nuovo sviluppo.

Per questo progetto usiamo solo l'API sincrona. L'API async ci permetterebbe di scansionare molti URL in parallelo, che è un'ottima sfida di estensione in `04-CHALLENGES.md`.

## 8. Perché usiamo respx per i test

Quando scrivi un test che chiama `scan("https://example.com")`, hai un problema: il test ora dipende dal fatto che example.com sia raggiungibile, veloce e restituisca header prevedibili. Nessuna di queste cose è garantita. Il test sarebbe **flaky** (a volte passa, a volte fallisce, per motivi che non c'entrano con il tuo codice).

`respx` risolve questo problema intercettando ogni chiamata httpx all'interno di un test e restituendo ciò che hai configurato. Schematicamente:

```
   Test code             respx (interceptor)         The real internet
   ─────────             ───────────────────         ─────────────────
   respx.get(URL).mock(   intercepts httpx.get,
       return_value=...   never sends a packet,
   )                      hands back the fake
   scan(URL)         ───▶ Response object        ──╳  never reached
```

Un test che usa respx è veloce (niente rete), deterministico (la response finta è esattamente quella che hai configurato) e funziona offline. Il costo è che devi fare attenzione: `respx` intercetta solo httpx, quindi una scansione che usasse `requests` dietro le quinte aggirerebbe silenziosamente il mock.

## 9. Filosofia di gestione degli errori

Due livelli di gestione degli errori.

In `scan()`, **lasciamo propagare gli errori**. Se il DNS fallisce, se l'host rifiuta la connessione, se la richiesta va in timeout, httpx solleva un'eccezione. Non la catturiamo. Il compito della funzione è recuperare e valutare, non decidere cosa fare quando il recupero fallisce.

In `main()`, **catturiamo `httpx.RequestError` una sola volta** e lo trasformiamo in un messaggio pulito più l'exit code 2. L'utente non ha bisogno di vedere un traceback Python di 30 righe per "impossibile connettersi". Ha bisogno di vedere "request failed".

Questo è un pattern che vale la pena interiorizzare: **il codice della libreria lascia risalire gli errori, il codice della CLI li traduce in output comprensibile per l'utente.** L'autore della libreria non sa da quale contesto viene chiamata la libreria, quindi non dovrebbe fingere di sapere come gestire gli errori. L'autore della CLI sa esattamente in quale contesto avviene la chiamata (un utente ha digitato un comando), quindi può presentare gli errori in modo appropriato.

## 10. Exit code che significano qualcosa

La maggior parte delle CLI restituisce l'exit code 0 per il successo e 1 per qualsiasi tipo di fallimento. Il nostro scanner ne usa tre:

```
0 → grade A or B   (CI: green, no action needed)
1 → grade C or D   (CI: yellow, worth investigating)
2 → grade F or network error  (CI: red, must fix)
```

È utile quando integri lo scanner nella CI. Una pipeline può eseguire:

```
just run -- https://my-deployed-site.com
if [ $? -gt 1 ]; then exit 1; fi   # fail the build only on F or error
```

Puoi decidere da solo quale soglia conta come "far fallire la build". Il punto è che lo scanner ti dà le informazioni per decidere. Un exit code binario successo/fallimento butta via troppi dettagli.

## 11. Cosa NON appartiene a questa architettura

Per un progetto foundations, teniamo deliberatamente fuori alcune cose:

- **Niente async.** Un singolo URL non ne ha bisogno. L'API sincrona è più facile da leggere.
- **Niente sottocomandi.** Niente `headers scan`, `headers explain`, `headers config`. Un solo compito, eseguilo.
- **Niente file di configurazione.** Tutte le impostazioni provengono dai flag della CLI. Niente `~/.config/headers.toml`. Aggiungine uno se lo estendi.
- **Niente database.** Ogni esecuzione è indipendente. Nessuna cronologia. Aggiungine uno se vuoi vedere le tendenze nel tempo (il file delle sfide ha i dettagli).
- **Nessun sistema di plugin.** La tabella delle regole è solo un elenco. Per "estendere" lo scanner, modifichi l'elenco.

Sono tutte cose che uno scanner più maturo avrebbe, e ognuna di esse è un'ottima idea di estensione. Nessuna di esse appartiene a un progetto il cui obiettivo è "essere la cosa più piccola possibile che insegna l'idea di fondo."

## 12. Riferimento ai file principali

Una mappa rapida del progetto:

| Percorso | Cosa contiene |
|------|---------------|
| `http_headers_scanner.py` | L'intero scanner. Regole, valutazione, scansione, CLI. |
| `test_http_headers_scanner.py` | Tutti i test. Usa pytest e respx. |
| `pyproject.toml` | Metadati del progetto, dipendenze, configurazioni dei tool (ruff, mypy, pylint, yapf, pytest). |
| `uv.lock` | Versioni esatte di ogni dipendenza transitiva. Build riproducibili. |
| `justfile` | Comandi scorciatoia: `just test`, `just lint`, `just run`. |
| `install.sh` | Installer in un solo passaggio per i nuovi clone. |
| `learn/` | La documentazione che stai leggendo. |

## Prossimo passo

Passa a **[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md)** per la spiegazione riga per riga, oppure salta a **[04-CHALLENGES.md](./04-CHALLENGES.it.md)** per idee di estensione.
