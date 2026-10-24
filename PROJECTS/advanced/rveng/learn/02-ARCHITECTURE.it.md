<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# Architettura

rveng è un unico motore di analisi con tre volti. Il motore è Python senza
framework che non sa nulla di HTTP o React. Un sottile layer FastAPI lo adatta al
web. Una app React consuma quella API. I progressi sono salvati dietro una piccola
interfaccia, così può essere sostituita. Questa forma esiste affinché il motore e
il contenuto delle lezioni possano essere estratti da questo progetto e montati
dentro un'applicazione più grande senza trascinarsi dietro un framework web.

## Un motore, tre volti

```
                         +--------------------------+
                         |   rveng/engine/  (pure)  |
                         |  elf disasm plt xref cfg  |
                         |  hex strings patch discover|
                         |  challenge (grading)      |
                         +------------+--------------+
                                      |
             +------------------------+------------------------+
             |                        |                        |
        +----+-----+           +------+------+          +------+------+
        | HTTP API |           |   library   |          |     CLI     |
        | FastAPI  |           |  (import it)|          | (potential) |
        +----+-----+           +-------------+          +-------------+
             |
        +----+-----+
        | React app|
        +----------+
```

Il motore è il prodotto. La API è un adapter. La app React è un volto. Lo stesso
motore che risponde a `/api/challenges/05-find-the-gate/disasm` potrebbe essere
importato direttamente da CertGames e pilotato senza alcun HTTP. Nulla in
`rveng/engine/` importa FastAPI, e nulla fa il rendering di HTML. Quella
separazione è il punto centrale.

## I moduli del motore

Ogni modulo è piccolo e fa una sola cosa sui byte grezzi. Nessuno di essi esegue
alcunché.

```
engine/elf.py       parse the ELF64 header, sections, and symbols from bytes
engine/hex.py       render a canonical offset/hex/ascii dump
engine/strings.py   scan bytes for printable runs
engine/disasm.py    decode x86-64 with capstone, annotate the interesting ops
engine/plt.py       resolve a PLT stub address to an imported symbol name
engine/xref.py      collect cross-references from decoded instructions
engine/cfg.py       split a function into basic blocks and connect them
engine/discover.py  find functions in a stripped binary by prologue scan
engine/patch.py     apply and diff byte edits, verify a patch statically
engine/challenge.py the challenge model and the solve-then-reveal grader
```

La direzione delle dipendenze è rigorosa: tutto dipende da `elf.py` e dal suo
buffer di byte, `disasm.py` alimenta `xref.py` e `cfg.py`, e `challenge.py`
dipende da `patch.py` per la valutazione delle patch. Nulla punta all'indietro
verso la API.

## Il layer API

`rveng/api/` è l'adapter. Carica le challenge selezionate una sola volta
all'avvio, espone un'analisi in sola lettura su di esse e valuta gli invii.

```
api/app.py        builds the FastAPI app and defines every route
api/store.py      loads challenges from disk; the progress store (Protocol + impls)
api/schemas.py    the Pydantic response and request shapes
api/limits.py     size and length caps for untrusted input
api/middleware.py a body-size cap enforced before the body is parsed
api/server.py     the ASGI entry point (create_app())
```

Le route sono deliberatamente noiose. Dato l'id di una challenge restituiscono
una vista esadecimale, un riepilogo ELF, un disassemblato, un grafo del flusso di
controllo, le cross-reference o l'elenco delle stringhe. Una route accetta un
invio e lo valuta. Nessuna route accetta un binario da analizzare; gli unici
binari nel sistema sono gli asset delle challenge selezionate.

## Come scorre una richiesta

Prendi chi sta imparando mentre disassembla la funzione `check` del campione:

```
browser
  GET /api/challenges/05-find-the-gate/disasm?symbol=check
    |
  nginx  (serves the React app, proxies /api to the API container)
    |
  FastAPI route in app.py
    |  look up the challenge by id  (store.py)
    |  parse its binary            (elf.ElfImage)
    |  find the symbol "check"     (image.symbol)
    |  disassemble the symbol      (disasm.disassemble_symbol)
    |  resolve any call names      (plt.plt_map)
    |  serialize to schemas.DisasmView
    v
  JSON back to the browser, rendered by the disasm pane
```

Il motore ha fatto l'analisi; la API ha solo tradotto id in chiamate e oggetti in
JSON. Aspetto cruciale, l'annotazione che evidenzia il gate nel disassemblato
viene trattenuta finché la challenge non è risolta, così chi impara vede prima un
disassemblato grezzo e onesto e il suggerimento "questo è il gate" solo dopo
averlo trovato da sé.

## I progressi dietro un'interfaccia

I progressi (quali challenge ha risolto una sessione) sono l'unico pezzo di
stato. Vivono dietro un protocollo `ProgressStore` con due metodi:

```
mark_solved(session, challenge_id) -> None
solved(session) -> set[str]
```

Due implementazioni lo soddisfano: `InMemoryProgress` (un dict, usato nei test)
e `SqliteProgress` (una tabella su file, il default con cui gira il server). Le
route vedono sempre e solo il protocollo, quindi sostituire lo storage
sottostante è una modifica di una riga in `create_app` e non tocca alcuna route.
Quando rveng viene estratto in un'applicazione host, quell'applicazione punta lo
stesso protocollo al proprio database, e nient'altro si sposta.

```
routes ---> ProgressStore (protocol)
                 |
        +--------+---------+
        |                  |
  InMemoryProgress    SqliteProgress    (host app plugs its own here)
```

## La topologia di deployment

rveng è self-hosted su localhost con Docker. Non c'è cloud, né TLS, né dominio,
né account. Due container in produzione:

```
PROD (compose.yml, project "rveng")        DEV (dev.compose.yml, project "rveng-dev")

  browser :8790                              browser :8791
      |                                          |
  [ nginx ]  serves built React dist         [ nginx ]  proxies everything
      |  /api                                    |  /        |  /api
  [ api ]  uvicorn (FastAPI engine)          [ vite HMR ] [ api ] uvicorn --reload
      |                                                        |
  rveng_data volume (sqlite progress)                    rveng_data_dev volume
```

nginx serve il frontend compilato e fa da proxy per `/api` verso il container
della API. La API è un puro adapter del motore che non impara mai a servire una
SPA, il che la mantiene estraibile. In sviluppo, nginx sta davanti al dev server
di Vite per l'hot reload invece di servire una build statica. I due progetti
usano nomi strutturali letterali (`rveng` e `rveng-dev`), quindi dev e prod sono
isolati nel namespace per costruzione e non possono entrare in collisione. Tutto
gira con default integrati, quindi un clone pulito si avvia senza alcuna
configurazione.

## Estrazione in un'applicazione host

Lo stato finale che questa architettura ti compra: montare rveng come
funzionalità altrove significa importare `rveng.engine`, riusare gli asset delle
challenge così come sono, puntare il protocollo `ProgressStore` allo storage
dell'host e montare i componenti React autosufficienti. Il motore e il contenuto
arrivano invariati. Questa è la differenza tra costruire un progetto usa e getta
e costruire una funzionalità che per caso sta anche in piedi da sola.
