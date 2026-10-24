# Analisi dell'implementazione

Questo file ripercorre riga per riga il codice vero e proprio di `http_headers_scanner.py` (e un po' di `test_http_headers_scanner.py`). Alla fine dovresti capire ogni parte del file: cosa fa, perché è lì e cosa si romperebbe se la rimuovessi.

Questo è il file più lungo della cartella learn. Affrontalo a pezzi. L'ordine qui sotto corrisponde all'ordine in cui le cose compaiono nel sorgente.

## 0. Convenzioni di lettura

Ogni sezione nomina una funzione, una classe o una costante di `http_headers_scanner.py`. Apri il file nel tuo editor di fianco e cerca il nome. Gli estratti di codice in questa guida sono reali, copiati direttamente dal file, ma il file è anche abbastanza corto da poterlo scorrere tutto in un paio di pagine.

## 1. La docstring del file

Il file inizia con una lunga stringa tra tripli apici. In Python, una stringa in cima a un file si chiama **module docstring**. È il posto ufficiale in cui spiegare di cosa tratta il file.

```python
"""
©AngelaMos | 2026
http_headers_scanner.py

Scan a URL and grade its HTTP security headers A–F

When a browser asks a website for a page, the server sends back the
page itself PLUS a bunch of metadata called "HTTP response headers."
...
"""
```

Alcune cose da notare:

- **Le prime tre righe sono l'intestazione standard dei file del progetto.** Ogni file del progetto inizia così: una riga di copyright, una riga quasi vuota, il nome del file. La parte `©AngelaMos | 2026` è il branding del progetto, non qualcosa che vedresti normalmente in un tutorial Python generico.
- **Il corpo è insolitamente lungo per una docstring.** La maggior parte dei file ha un riassunto di una riga. Questa è dettagliata perché è un progetto didattico. La docstring è la prima cosa che vede qualsiasi lettore (`help(http_headers_scanner)` la stampa, gli IDE la mostrano al passaggio del mouse), quindi la usiamo per insegnare.
- **Termina con un elenco di "cosa espone questo file".** È una convenzione reale. Dice ai lettori cosa possono importare dal modulo senza dover scorrere 600 righe.

## 2. Import

```python
import argparse
import re
import sys
from dataclasses import dataclass
from typing import Literal

import httpx
from rich.console import Console
from rich.panel import Panel
from rich.table import Table
```

PEP 8 (la style guide di Python) vuole che gli import siano raggruppati in tre sezioni separate da righe vuote:

1. **Libreria standard**: cose che vengono fornite con Python. Qui: `argparse`, `re`, `sys`, `dataclasses`, `typing`.
2. **Terze parti**: cose che hai installato con `uv` / `pip`. Qui: `httpx`, `rich`.
3. **Locali**: cose dello stesso progetto. Non ne abbiamo.

`re` è il modulo della libreria standard per le espressioni regolari. Usiamo `re.search(pattern, value, re.IGNORECASE)` dentro `evaluate_header()` per verificare se il valore di un header corrisponde al pattern richiesto da una regola. Le regex ci danno un modo per esprimere "max-age deve essere un intero positivo" in una riga, invece di dover fare noi il parsing della struttura della direttiva HSTS.

Ogni modulo è importato con un breve commento inline che spiega a cosa ci serve. I principianti chiedono spesso "ma cosa fa `import`?" Risposta breve: dice a Python "vai a trovare questo modulo e rendi disponibili i suoi nomi in questo file." `import argparse` rende disponibile `argparse.ArgumentParser`. `from dataclasses import dataclass` rende disponibile il nome semplice `dataclass`, così possiamo usare `@dataclass` direttamente senza scrivere `@dataclasses.dataclass`.

## 3. I tipi Severity e Status

```python
Severity = Literal["high", "medium", "low"]
Status = Literal["ok", "weak", "missing"]
```

Questi sono **type alias**. Danno un nome amichevole a un tipo più complesso. D'ora in poi, ovunque scrivi `Severity`, il type checker legge `Literal["high", "medium", "low"]`.

Il motivo per cui esiste `Literal`: un normale `str` significa "qualsiasi stringa." Se annotassimo `severity: str`, allora `severity = "hgih"` verrebbe accettato senza problemi ed esploderebbe solo a runtime, quando qualcosa provasse a cercarlo. Con `severity: Severity`, mypy si rifiuta di far avvicinare `"hgih"` al campo. L'errore di battitura viene intercettato mentre scrivi.

È più disciplina di quella che vedrai nella maggior parte del Python per principianti online. È una scelta deliberata per un progetto didattico: vogliamo che tu assimili l'abitudine presto.

## 4. La dataclass `HeaderRule`

```python
@dataclass(frozen=True, slots=True)
class HeaderRule:
    header: str
    severity: Severity
    description: str
    recommendation: str
    must_match: str | None = None
```

Una dataclass è una normale classe in cui le parti noiose (costruttore, uguaglianza, repr come stringa) vengono scritte per te dal decorator `@dataclass`. Abbiamo trattato i flag `frozen` e `slots` in `02-ARCHITECTURE.md`. In breve: `frozen` impedisce a chiunque di modificare i campi dopo la costruzione, `slots` rende le istanze più piccole in memoria.

I campi:

- **`header`**: il nome dell'header HTTP che stiamo cercando. Memorizzato con le maiuscole canoniche (ad es. `"Strict-Transport-Security"`) ma confrontato senza distinzione tra maiuscole e minuscole al momento della ricerca.
- **`severity`**: determina il punteggio. `"high"` = 30 punti, `"medium"` = 15, `"low"` = 5.
- **`description`**: una frase che spiega l'header. Attualmente usata per la documentazione; potremmo anche mostrarla nella tabella.
- **`recommendation`**: cosa aggiungere per correggere un header mancante o debole. Mostrata nella sezione "Recommendations" in fondo all'output.
- **`must_match`**: opzionale. Un pattern regex a cui il valore deve corrispondere (senza distinzione tra maiuscole e minuscole) per essere considerato `ok`. Per HSTS il pattern è `r"max-age\s*=\s*[1-9]"` (rifiuta `max-age=0`); per `X-Content-Type-Options` è `"nosniff"` (una semplice parola funziona come corrispondenza di sottostringa con `re.search`). Se è `None`, basta la sola presenza.

Il `= None` finale su `must_match` è il suo **valore predefinito**. Significa che puoi costruire un `HeaderRule` senza specificarlo. Solo i campi con un valore predefinito possono essere omessi al momento della costruzione.

## 5. La tabella `RULES`

```python
RULES: list[HeaderRule] = [
    HeaderRule(
        header="Strict-Transport-Security",
        severity="high",
        ...
        must_match=r"max-age\s*=\s*[1-9]",
    ),
    HeaderRule(
        header="Content-Security-Policy",
        severity="high",
        ...
    ),
    ...
]
```

Questa è l'**unica fonte di verità** per quali header controlliamo. Il pattern lista-di-dataclass è uno dei più utili in Python: ogni voce è strutturata, immutabile e facile da estendere.

Un dettaglio del pattern: usiamo **argomenti keyword** per ogni campo, non posizionali. Scriviamo `HeaderRule(header="...", severity="...", ...)`, non `HeaderRule("...", "...", ...)`. Perché? Due motivi:

1. **Leggibilità.** Quando qualcuno legge il codice, vede `severity="high"` e sa esattamente cosa significa il secondo valore. La forma posizionale `("Strict-Transport-Security", "high", ...)` lo costringe a contare i campi.
2. **Sicurezza nei refactoring.** Se in seguito aggiungi un nuovo campo (diciamo `references: list[str]`), le chiamate posizionali potrebbero far finire il nuovo valore nel posto sbagliato. Le chiamate keyword sono inequivocabili.

Perché questa lista sta a livello di modulo e non dentro una funzione? Perché non cambia mai. Costruirla una volta al momento dell'import costa meno che ricostruirla a ogni scansione. È anche accessibile alla test suite (`from http_headers_scanner import RULES`).

## 6. La mappatura `SEVERITY_POINTS`

```python
SEVERITY_POINTS: dict[Severity, int] = {
    "high": 30,
    "medium": 15,
    "low": 5,
}
```

Un dizionario che associa ogni gravità al suo valore in punti. Nota l'annotazione di tipo: `dict[Severity, int]`. Dice al type checker "le chiavi devono essere `"high"` / `"medium"` / `"low"`, i valori devono essere int." Se provassi ad aggiungere `"critical": 50` a questo dict, mypy si rifiuterebbe: `"critical"` non fa parte del tipo Literal `Severity`.

Perché un dict e non una funzione con tre if? Perché sono dati, non logica. Il codice data driven è più facile da estendere (aggiungi un'altra gravità, modifichi una riga) e più facile da testare (puoi verificare con assert i valori esatti in punti).

## 7. La dataclass `HeaderFinding`

```python
@dataclass(frozen=True, slots=True)
class HeaderFinding:
    rule: HeaderRule
    status: Status
    actual_value: str | None
    note: str
```

Un finding è il risultato della valutazione di una regola su una response. Contiene:

- **`rule`**: la regola che è stata valutata. Memorizzare l'intera regola dentro il finding (invece del solo nome) significa che il renderer non deve mai fare una seconda ricerca per conoscere la gravità o la raccomandazione.
- **`status`**: uno tra `"ok"`, `"weak"`, `"missing"`. Il tipo Literal intercetta gli errori di battitura.
- **`actual_value`**: qualunque cosa il server abbia effettivamente inviato. `None` se l'header mancava.
- **`note`**: una breve stringa comprensibile. Mostrata nella tabella.

Perché `actual_value` è tipizzato `str | None`? Perché il campo a volte è davvero una stringa e a volte niente. `None` è il modo di Python per dire "nessun valore." Il tipo `str | None` lo rende esplicito. Ovunque usi `finding.actual_value`, il type checker ti costringe a gestire il caso None oppure ad asserire che non possa essere None.

La sintassi `|` (ad es. `str | None`) è il modo moderno (Python 3.10+). Il modo più vecchio era `Optional[str]` dal modulo `typing`. Funzionano entrambi; la nuova sintassi è più breve.

## 8. La dataclass `ScanReport` con proprietà calcolate

```python
@dataclass(frozen=True, slots=True)
class ScanReport:
    url: str
    final_url: str
    status_code: int
    findings: list[HeaderFinding]

    @property
    def score(self) -> int:
        ...

    @property
    def grade(self) -> str:
        ...
```

Un report ha quattro campi memorizzati più due proprietà calcolate.

### 8.1 Perché `final_url` è separato da `url`

`url` è ciò che l'utente ha digitato. `final_url` è dove è arrivato dopo i redirect. Spesso sono uguali. Sono diversi quando, ad esempio, `http://example.com/` reindirizza a `https://example.com/`. Teniamo traccia di entrambi perché:

- L'utente vuole vedere riconosciuto nell'output l'URL che ha digitato.
- Il voto appartiene davvero all'URL finale (la destinazione del redirect è ciò che il suo browser mostra effettivamente).

### 8.2 La proprietà `score`

```python
@property
def score(self) -> int:
    total = sum(SEVERITY_POINTS[r.severity] for r in RULES)
    if total == 0:
        return 0

    earned = 0.0
    for finding in self.findings:
        full = SEVERITY_POINTS[finding.rule.severity]
        if finding.status == "ok":
            earned += full
        elif finding.status == "weak":
            earned += full / 2

    return int((earned / total) * 100 + 0.5)
```

Passo per passo:

1. **`@property`** sulla riga sopra trasforma il metodo in qualcosa a cui accedi senza parentesi. `report.score`, non `report.score()`. Sembra un campo, ma viene calcolato su richiesta.
2. **`total = sum(SEVERITY_POINTS[r.severity] for r in RULES)`** calcola il totale dei punti ottenibili percorrendo le regole. L'espressione dentro `sum(...)` è una **generator expression**: produce un numero per regola (il valore in punti per la gravità di quella regola), poi sum li somma. Con le attuali regole 2 high, 2 medium, 2 low, total = 100.
3. **`if total == 0: return 0`** è una guardia. Se qualcuno eliminasse la tabella delle regole a runtime, altrimenti divideremmo per zero. Restituire zero è una risposta sicura.
4. **Il ciclo principale** percorre ogni finding. Per ciascuno, cerca il valore in punti pieno per la gravità della sua regola. Se lo status è `ok`, aggiunge i punti pieni. Se `weak`, ne aggiunge la metà. Se `missing`, non aggiunge nulla (nessun ramo esplicito; la variabile resta invariata).
5. **`int((earned / total) * 100 + 0.5)`** è il punteggio finale. Il `+ 0.5` seguito da `int(...)` è un arrotondamento manuale half-up. Lo usiamo perché il `round()` integrato di Python usa l'arrotondamento bancario (round half to even), che porterebbe `round(0.5)` a `0` e `round(2.5)` a `2`. Matematicamente difendibile (annulla la distorsione su un campione ampio) ma sorprendente al confine del `.5`, dove un punteggio dovrebbe sempre arrotondare per eccesso. `int(x + 0.5)` è la forma che tutti si aspettano.

### 8.3 La proprietà `grade`

```python
@property
def grade(self) -> str:
    score = self.score
    if score >= 90:
        return "A"
    if score >= 80:
        return "B"
    if score >= 70:
        return "C"
    if score >= 60:
        return "D"
    return "F"
```

Nota che ogni ramo esegue direttamente `return`, quindi non ci sono `elif` né un `else` finale. È un idioma comune chiamato **early return**. La funzione si legge dall'alto verso il basso: appena una condizione corrisponde, hai finito. Evita anche l'"arrow code", in cui ogni ramo è più indentato del precedente.

Nota anche che **chiamiamo `self.score` una sola volta**, memorizziamo il risultato e poi lo confrontiamo cinque volte. Se scrivessimo `if self.score >= 90:` ecc., la proprietà verrebbe rieseguita ogni volta. Per una minuscola funzione di punteggio non conterebbe, ma vale la pena prendere presto l'abitudine di mettere in cache le ricerche costose ripetute.

## 9. `evaluate_header()`: il cuore dello scanner

Questa è la **funzione pura** al centro di tutto. Niente rete. Niente print. Solo regola più header in ingresso, finding in uscita.

```python
def evaluate_header(
    rule: HeaderRule,
    response_headers: dict[str, str],
) -> HeaderFinding:
    target = rule.header.lower()

    actual_value: str | None = None
    for name, value in response_headers.items():
        if name.lower() == target:
            actual_value = value
            break

    if actual_value is None:
        return HeaderFinding(
            rule=rule,
            status="missing",
            actual_value=None,
            note=f"Header `{rule.header}` is not set",
        )

    if rule.must_match is None:
        return HeaderFinding(
            rule=rule,
            status="ok",
            actual_value=actual_value,
            note="Present",
        )

    if re.search(rule.must_match, actual_value, re.IGNORECASE):
        return HeaderFinding(
            rule=rule,
            status="ok",
            actual_value=actual_value,
            note=f"Present and matches `{rule.must_match}`",
        )

    return HeaderFinding(
        rule=rule,
        status="weak",
        actual_value=actual_value,
        note=(
            f"Present but does not match `{rule.must_match}` "
            f"(got `{actual_value}`)"
        ),
    )
```

Tre rami, in ordine:

1. **Mancante.** Cicliamo sui response header, convertiamo in minuscolo ogni nome e lo confrontiamo con il target in minuscolo. Se non troviamo mai una corrispondenza, `actual_value` resta `None` e restituiamo un finding `missing`.
2. **Presente, nessun controllo must_match.** Se la regola non richiede un pattern specifico, basta la sola presenza. Restituisce `ok`.
3. **Presente, con controllo must_match.** Se la regola ha un `must_match`, esegue `re.search(pattern, value, re.IGNORECASE)`. Una semplice parola come `"nosniff"` funziona come controllo di sottostringa; un pattern più ricco come `r"max-age\s*=\s*[1-9]"` impone una condizione reale (HSTS deve essere impostato a un intero positivo, non al dannoso `max-age=0`). Se il pattern corrisponde, `ok`. Altrimenti, `weak`.

Alcune cose che vale la pena sottolineare:

**La ricerca case-insensitive.** I nomi degli header HTTP non distinguono tra maiuscole e minuscole secondo l'RFC 7230. Server diversi li restituiscono con maiuscole diverse. Alcuni restituiscono `Strict-Transport-Security`, altri `strict-transport-security`, alcuni persino `STRICT-TRANSPORT-SECURITY` (raro ma legale). Convertire in minuscolo entrambi i lati è il modo portabile più semplice per gestirlo.

Avremmo potuto usare un dict case-insensitive (httpx ne restituisce uno), ma la funzione dovrebbe accettare un dict semplice per agevolare i test. In pratica `scan()` converte già i response header in un semplice `dict[str, str]` prima di chiamare `evaluate_header`, quindi questa funzione non vede mai direttamente un oggetto `httpx.Headers`, ma il contratto è "qualsiasi `dict[str, str]` funziona", ed è questo che rende banali gli input di test costruiti a mano.

**Perché usciamo dal ciclo in anticipo con `break`.** Una volta trovato l'header, abbiamo ciò che ci serve. Continuare il ciclo sprecherebbe CPU.

**Le f-string in `note`.** Una f-string è una stringa con segnaposto `{expression}` che vengono riempiti a runtime. `f"Header `{rule.header}` is not set"` diventa `Header `Strict-Transport-Security` is not set` se l'header della regola è HSTS. I backtick intorno al nome dell'header lo fanno apparire a spaziatura fissa se il renderer gestisce il markdown, e in generale aiutano l'occhio.

**Nessun ramo else.** Ogni ramo restituisce un valore. Una volta eseguito il return, la funzione è terminata. Non serve scrivere `elif` o `else`. È lo stesso pattern di early return della proprietà `grade`.

## 10. `scan()`: la chiamata di rete

```python
DEFAULT_USER_AGENT: str = (
    "http-headers-scanner/1.0 "
    "(+https://github.com/CarterPerez-dev/Cybersecurity-Projects)"
)


def scan(
    url: str,
    *,
    timeout: float = 10.0,
    user_agent: str = DEFAULT_USER_AGENT,
) -> ScanReport:
    response = httpx.get(
        url,
        timeout=timeout,
        follow_redirects=True,
        headers={"User-Agent": user_agent},
    )

    response_headers = dict(response.headers)
    findings = [evaluate_header(rule, response_headers) for rule in RULES]

    return ScanReport(
        url=url,
        final_url=str(response.url),
        status_code=response.status_code,
        findings=findings,
    )
```

### 10.1 Lo User-Agent

La stringa User-Agent identifica chi sta facendo la richiesta. I browser inviano cose come `Mozilla/5.0 (X11; Linux x86_64) ...`. Il nostro scanner invia `http-headers-scanner/1.0 (+https://...)`. È una forma di cortesia per due motivi:

- Gli operatori dei server che leggono i loro access log possono capire chi li sta interrogando e consultare la pagina del nostro progetto se si chiedono il perché.
- Alcuni siti bloccano lo UA predefinito di httpx. Uno UA personalizzato ha più probabilità di ottenere una risposta reale.

### 10.2 Il `*,` nella firma

```python
def scan(
    url: str,
    *,
    timeout: float = 10.0,
    user_agent: str = DEFAULT_USER_AGENT,
) -> ScanReport:
```

Il `*,` tra `url` e `timeout` costringe i chiamanti a passare `timeout` e `user_agent` come keyword. Non puoi chiamare `scan("https://example.com", 5.0)`. Devi chiamare `scan("https://example.com", timeout=5.0)`.

Perché imporlo? Perché `5.0` non significa ovviamente "cinque secondi di timeout" quando leggi il punto della chiamata. `timeout=5.0` sì. Gli argomenti keyword-only rendono i punti di chiamata più leggibili e sicuri nei refactoring. Il costo è esattamente un pezzo di testo in più (`timeout=`) quando chiami la funzione.

### 10.3 `follow_redirects=True`

Quando il server dice "questo URL è stato spostato, prova quest'altro", seguiamo automaticamente il redirect. Molti siti reindirizzano `http://` a `https://` oppure `www.` al dominio nudo. L'utente ha digitato un URL ma il suo browser finirebbe su uno diverso. Vogliamo valutare quello che il suo browser vedrebbe effettivamente.

### 10.4 Il passaggio al layer puro

```python
response_headers = dict(response.headers)
findings = [evaluate_header(rule, response_headers) for rule in RULES]
```

Queste due righe sono il passaggio "lascia il mondo dell'I/O, entra nel mondo puro" di cui abbiamo parlato nel file dell'architettura. `dict(response.headers)` converte l'oggetto Headers di httpx in un semplice dict. La list comprehension nella riga successiva esegue `evaluate_header()` per ogni regola.

Una **list comprehension** è una scorciatoia per "crea una lista eseguendo un'espressione su ogni elemento di una sorgente." Il ciclo for equivalente sarebbe:

```python
findings = []
for rule in RULES:
    findings.append(evaluate_header(rule, response_headers))
```

Stesso risultato, più righe. La comprehension è preferibile quando il corpo è una singola espressione.

## 11. Rendering

```python
STATUS_COLORS: dict[Status, str] = {
    "ok": "green",
    "weak": "yellow",
    "missing": "red",
}

GRADE_COLORS: dict[str, str] = {
    "A": "bright_green",
    "B": "green",
    "C": "yellow",
    "D": "red",
    "F": "bright_red",
}


def _render_report(report: ScanReport, console: Console) -> None:
    table = Table(...)
    table.add_column(...)
    ...
    for finding in report.findings:
        status_color = STATUS_COLORS[finding.status]
        table.add_row(...)
    console.print(table)

    if report.final_url.startswith("http://"):
        console.print(
            "[yellow]Note:[/yellow] this response was served over plain "
            "HTTP. Browsers IGNORE HSTS over HTTP, ..."
        )

    grade_color = GRADE_COLORS[report.grade]
    panel = Panel(...)
    console.print(panel)

    actionable = [f for f in report.findings if f.status != "ok"]
    if actionable:
        console.print("\n[bold]Recommendations:[/bold]")
        for finding in actionable:
            console.print(...)
```

Il renderer usa **rich**, una libreria di terze parti per un output del terminale gradevole. I pattern:

- **Un oggetto `Table`** con delle colonne. Aggiungi le righe una alla volta. `console.print(table)` la disegna come una tabella con bordi Unicode.
- **`[green]something[/green]`** è la sintassi di markup di rich. È all'incirca come l'HTML per i colori del terminale. `[bold cyan]Result[/bold cyan]` mostrerebbe "Result" in grassetto ciano.
- **`Panel(...)`** racchiude il contenuto in un riquadro con bordo.

Il renderer è volutamente separato da `scan()` e `evaluate_header()`. Il codice puro non sa e non si preoccupa dei colori. Se mai volessimo una modalità di output JSON per la CI, aggiungeremmo un secondo renderer (`_render_json(report)`) e lasceremmo invariato tutto il resto del codice.

La riga `actionable = [f for f in report.findings if f.status != "ok"]` è un'altra comprehension: costruisce una lista di ogni finding il cui status non è `ok`. Sono quelli per cui abbiamo raccomandazioni. Se la lista è vuota (punteggio perfetto), saltiamo del tutto la sezione.

**L'avviso HTTP.** Subito dopo la tabella, controlliamo `report.final_url.startswith("http://")`. Secondo l'RFC 6797 §8.1, i browser DEVONO IGNORARE l'header `Strict-Transport-Security` quando arriva tramite HTTP in chiaro: conta solo l'HSTS ricevuto tramite HTTPS. Quindi se un utente punta lo scanner su `http://example.com` e il server restituisce HSTS, quell'HSTS ottiene il punteggio pieno nella nostra valutazione anche se nessun browser reale lo rispetterebbe. La nota gialla rende visibile l'avvertenza nell'unico punto che conta: il report rivolto all'utente. Non cambiamo la logica di valutazione (una regola, un esito), ma l'utente vede proprio accanto al punteggio una riga onesta che dice "questo voto è fuorviante finché il sito non impone HTTPS".

## 12. Il collegamento con argparse

```python
def _build_argument_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="headers",
        description="Scan a URL for HTTP security headers and grade the result A–F.",
    )
    parser.add_argument(
        "url",
        help="Full URL to scan (must include http:// or https://).",
    )
    parser.add_argument(
        "--timeout",
        type=float,
        default=10.0,
        help="Seconds to wait before giving up on the request (default: 10).",
    )
    return parser
```

**argparse** è il parser degli argomenti da riga di comando della libreria standard. Dichiari cosa accetta il tuo programma e argparse si occupa del resto: parsing, conversione dei tipi, generazione dell'output di `--help`, rifiuto degli input non validi.

Due argomenti dichiarati:

- **`url`**: posizionale (senza prefisso `--`). Obbligatorio. Se l'utente non lo fornisce, argparse termina con un errore e stampa automaticamente l'utilizzo.
- **`--timeout`**: opzionale. Il valore predefinito è `10.0`. `type=float` dice ad argparse di convertire la stringa `"5"` nel float `5.0`.

La funzione è volutamente separata da `main()` così i test possono costruire il parser e chiamare `parse_args([...])` su una lista sintetica, senza dover pasticciare con `sys.argv`.

## 13. `main()`: l'orchestrazione

```python
def main() -> int:
    parser = _build_argument_parser()
    args = parser.parse_args()
    console = Console()

    try:
        report = scan(args.url, timeout=args.timeout)
    except httpx.RequestError as exc:
        console.print(f"[red]Request failed:[/red] {type(exc).__name__}: {exc}")
        return 2

    _render_report(report, console)

    if report.grade in ("A", "B"):
        return 0
    if report.grade in ("C", "D"):
        return 1
    return 2
```

Questa funzione è piccola di proposito. Il suo compito è fare da collante. I passi:

1. Costruisce il parser argparse. `parse_args()` senza argomenti legge implicitamente `sys.argv`.
2. Crea una `Console` (l'oggetto principale di rich per la stampa).
3. Prova a fare la scansione. Se viene sollevato `httpx.RequestError` (il genitore di ogni errore legato alla rete), stampa un messaggio pulito e restituisce l'exit code 2.
4. Esegue il rendering del report.
5. Sceglie un exit code in base al voto.

Il `try / except` qui è l'**unico** punto in cui catturiamo gli errori di rete. Li lasciamo propagare da `httpx.get()` attraverso `scan()` fino a `main()`. Il motivo: le funzioni di livello più basso non possono sapere cosa fare con gli errori. La CLI sa cosa fare (mostrarli all'utente, uscire). La CLI è il layer giusto in cui catturarli.

`type(exc).__name__` è "il nome della classe dell'eccezione come stringa." Per un timeout di connessione sarebbe `ConnectTimeout`. Per un fallimento DNS, `ConnectError`. Includerlo nell'output dà all'utente un indizio su cosa è andato storto senza riversare un traceback completo.

## 14. L'entrypoint dello script

```python
if __name__ == "__main__":
    sys.exit(main())
```

Questo pattern compare in ogni script Python. Significa "se questo file è stato invocato direttamente (non importato come modulo), esegui main."

Quando esegui `python http_headers_scanner.py`, Python imposta una variabile speciale `__name__` a `"__main__"`. Quando qualche altro codice esegue `import http_headers_scanner`, `__name__` viene invece impostata a `"http_headers_scanner"`.

Quindi `if __name__ == "__main__":` significa "solo quando viene eseguito come script, non quando viene importato." I test importano il file, quindi hanno bisogno che `main()` NON venga eseguita automaticamente.

`sys.exit(main())` chiama main, poi passa il suo valore di ritorno (0, 1 o 2) al sistema operativo come exit code.

## 15. Analisi del file di test

I test si trovano in `test_http_headers_scanner.py`. Non esamineremo ogni riga, ma ecco i pattern principali.

### 15.1 Fixture

```python
@pytest.fixture
def hsts_rule() -> HeaderRule:
    return HeaderRule(
        header="Strict-Transport-Security",
        severity="high",
        ...
        must_match=r"max-age\s*=\s*[1-9]",
    )
```

Una **fixture** è il modo di pytest per dire "prima che questo test venga eseguito, prepara questa cosa per lui." Qualsiasi funzione di test che ha un parametro chiamato `hsts_rule` riceverà ciò che questa fixture restituisce. Pytest fa la corrispondenza per nome.

Usiamo le fixture così la regola viene costruita in un solo punto. Se la forma di `HeaderRule` cambia (viene aggiunto un nuovo campo), aggiorniamo la fixture, non cinque test diversi.

### 15.2 I test della funzione pura

```python
def test_evaluate_header_present_with_required_substring(
    hsts_rule: HeaderRule,
) -> None:
    headers = {"Strict-Transport-Security": "max-age=31536000"}
    finding = evaluate_header(hsts_rule, headers)
    assert finding.status == "ok"
    assert finding.actual_value == "max-age=31536000"
```

Ogni test segue il pattern **arrange-act-assert**:

1. **Arrange.** Costruisci l'input. Qui: un piccolo dict di header.
2. **Act.** Chiama la funzione sotto test.
3. **Assert.** Verifica che il risultato sia quello atteso.

`assert` è il modo di Python per dire "questo deve essere vero, altrimenti fai fallire il test." Se `finding.status != "ok"`, pytest solleva un AssertionError e stampa quale fosse il valore effettivo.

Poiché `evaluate_header` è pura, questi test sono semplicissimi. Nessun mocking, nessuna preparazione oltre alla fixture, nessun teardown.

### 15.3 I test di punteggio e voto

L'helper `_make_report` costruisce uno `ScanReport` sintetico associando a ogni regola uno status. Poi il test chiede `report.score` e `report.grade` e verifica con assert che siano quelli attesi.

Questo è il vantaggio di aver reso `score` e `grade` proprietà di `ScanReport`: possiamo testarle senza eseguire `scan()`. Ci limitiamo a costruire gli input a mano.

### 15.4 I test di scansione con mock di respx

```python
@respx.mock
def test_scan_mocks_a_clean_response_and_grades_it_correctly() -> None:
    respx.get("https://safe.example.com/").mock(
        return_value=httpx.Response(
            status_code=200,
            headers={
                "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
                ...
            },
        )
    )

    report = scan("https://safe.example.com/")

    assert report.status_code == 200
    assert report.score == 100
```

Il decorator `@respx.mock` sopra la funzione dice a respx: "durante questo test, intercetta ogni chiamata httpx e usa le route che configuro qui sotto."

`respx.get("https://safe.example.com/").mock(return_value=httpx.Response(...))` dice "quando qualcosa fa una HTTP GET a quell'URL, restituisci questa risposta preconfezionata, non andare davvero su internet."

Poi `scan("https://safe.example.com/")` fa il suo lavoro. httpx prova a recuperare l'URL; respx intercetta; torna la response finta; il resto del codice non si accorge mai della differenza. Facciamo assert sul punteggio.

Il test sul redirect (`test_scan_records_final_url_after_redirect`) configura due route simulate: la prima restituisce un 301 verso la seconda, la seconda restituisce 200. Lo scanner segue il redirect e verifichiamo con assert che `report.final_url` rifletta dove siamo arrivati.

## 16. Tooling: lint, type-check, formattazione

Il progetto include quattro tool di qualità collegati tramite `just`:

```
just lint    # runs ruff, then pylint, then mypy
just format  # runs yapf in place
just test    # runs pytest
just fix     # runs ruff with --fix (auto-fixes what it can)
```

Cosa fa ogni tool:

- **ruff** è un linter Python veloce. Intercetta un lungo elenco di problemi di stile e di correttezza. Sostituto moderno di flake8.
- **pylint** è un linter più lento e più rigido nelle sue opinioni. Intercetta problemi diversi da ruff. Li eseguiamo entrambi perché i loro controlli si completano a vicenda.
- **mypy** è il type checker statico. Legge le annotazioni di tipo e verifica ogni chiamata rispetto a esse. Intercetta errori di battitura come `severity = "hgih"` e molti altri bug mentre scrivi.
- **yapf** è il formattatore del codice. Riscrive il file per adeguarlo a uno stile configurato (limite di colonne, indentazione, ecc.). Significa che il progetto ha un aspetto unico e coerente indipendentemente da chi ha scritto ogni riga.
- **pytest** è il test runner. Trova i file che iniziano con `test_`, esegue ogni funzione al loro interno il cui nome inizia con `test_` e riporta successi e fallimenti.

In un workflow reale configureresti un **pre-commit hook** che esegue `just lint` e `just test` prima di ogni commit, così il codice rotto non viene mai committato. Non l'abbiamo fatto in questo progetto per mantenere minimale il livello foundations, ma estenderlo in questo senso è una delle sfide.

## 17. Il pyproject.toml

`pyproject.toml` è il moderno file di metadati dei progetti Python. Sostituisce la vecchia combinazione `setup.py` + `setup.cfg`. Vale la pena dargli un'occhiata, anche se di solito non lo modifichi quotidianamente.

Sezioni principali:

- **`[project]`**: nome, versione, descrizione, requisito di versione di Python, dipendenze.
- **`[project.optional-dependencies]`**: dipendenze di sviluppo (pytest, mypy, ecc.) di cui gli utenti finali non hanno bisogno.
- **`[project.scripts]`**: dichiara lo script da riga di comando `headers`. È per questo che `uv run headers` funziona: sa che deve invocare `http_headers_scanner:main`.
- **`[tool.ruff]`, `[tool.mypy]`, `[tool.pylint.*]`, `[tool.pytest.ini_options]`**: configurazione per ciascun tool. Centralizzare la configurazione in un unico file è comodo.

## 18. Insidie comuni quando si estende il progetto

Alcune cose che hanno messo in difficoltà le persone quando hanno aggiunto nuove regole o funzionalità:

**Dimenticare di aggiornare il totale del punteggio nei test.** Attualmente `RULES` contiene sei regole per un totale di 100 punti. Se ne aggiungi una settima, il calcolo del punteggio funziona ancora (somma qualunque cosa ci sia nella lista), ma i test che hanno scritto nel codice il punteggio atteso (ad es. "il punteggio dovrebbe essere 50 quando metà mancano") potrebbero rompersi. Soluzione: scrivi i test in termini di percentuali, non di conteggi assoluti di punti.

**Aggiungere una regola il cui parsing del valore non è banale.** Il nostro campo `must_match` è una singola regex. È più che sufficiente per "inizia con `nosniff`" o "max-age è un intero positivo", ma alcuni header reali richiedono un parsing molto più complesso (la CSP, per esempio, ha una propria grammatica di direttive, espressioni di sorgente e nonce). Se la tua nuova regola richiede un parsing strutturato, esegui il parsing in `evaluate_header()` in base al nome dell'header della regola, oppure estendi `HeaderRule` con un nuovo campo come `value_validator: Callable[[str], bool] | None`.

**Dimenticare il confronto case-insensitive.** Il nuovo codice che esegue `if "X-Frame-Options" in response.headers` non rileverà i server che restituiscono `x-frame-options`. Converti sempre in minuscolo entrambi i lati quando confronti i nomi degli header.

**Provare a scansionare più URL senza async.** L'API sincrona blocca su un URL alla volta. Scansionare 100 URL in sequenza è lento. Se vuoi la concorrenza, passa a `httpx.AsyncClient` e usa `asyncio.gather`. Il file delle sfide ne contiene un abbozzo.

## 19. Suggerimenti per il debug

Quando qualcosa va storto:

**Esegui con `-v` per l'output verbose di pytest.**
```
uv run pytest -v
```
Ti mostra il nome di ogni test mentre viene eseguito. È più facile individuare quale è fallito.

**Usa il flag `--pdb` per un debugger interattivo.**
```
uv run pytest --pdb
```
Ti porta nel debugger di Python al primo test che fallisce. Digita `l` per il sorgente intorno al fallimento, `p variable_name` per ispezionare, `c` per continuare.

**Stampa gli header effettivi quando lo scanner dà risultati sbagliati.**
Modifica `scan()` per stampare `response_headers` prima del ciclo. Esegui lo scanner su un sito noto. Confronta ciò che vedi con ciò che dicono i dev tools del tuo browser. User-Agent diversi a volte ricevono risposte diverse.

**Usa `curl -I` come controllo di sanità.**
```
curl -I https://example.com
```
Il flag `-I` recupera solo gli header. Se gli header che vedi lì non corrispondono a quelli riportati dallo scanner, c'è qualcosa che non va nella richiesta che lo scanner sta facendo.

## 20. Prossimo passo

Leggi **[04-CHALLENGES.md](./04-CHALLENGES.it.md)** per idee su come estendere lo scanner. Scegline una che ti interessa, provala e guarda come regge l'architettura quando la metti sotto pressione.
