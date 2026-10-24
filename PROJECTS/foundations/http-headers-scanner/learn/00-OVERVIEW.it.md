# http-headers-scanner

Un piccolo tool da riga di comando che visita un sito web, chiede quali sono le "regole" che il sito dice al tuo browser di seguire e assegna al sito un voto da A a F in base a quanto sono buone quelle regole.

Questo è il progetto intermedio del livello **foundations**. Il primo progetto foundations (`hash-identifier`) era pura logica senza rete. Il terzo (`password-manager`) è una piccola applicazione completa. Questo progetto si colloca tra i due: un solo viaggio attraverso internet, poi un po' di calcoli per il voto.

## Perché qualcuno l'ha costruito

Ogni volta che visiti un sito web, il tuo browser e il server del sito hanno una breve conversazione. Il browser chiede "dammi la pagina" e il server restituisce due cose:

1. La pagina stessa (l'HTML, le immagini, gli script).
2. Un breve elenco di **regole** su come trattare quella pagina.

Quel secondo elenco è ciò che ci interessa qui. Si chiama **response header**. Alcuni di questi header riguardano la sicurezza. Dicono al tuo browser cose come:

- "D'ora in poi, parlami solo tramite HTTPS. Mai HTTP in chiaro."
- "Non permettere a nessun altro sito web di inserirmi dentro un `<iframe>`."
- "Se ricevi un file da me e non sei sicuro di che tipo sia, non tirare a indovinare. Trattalo in modo rigoroso."
- "Blocca gli script a meno che non provengano da questo elenco preciso di posizioni fidate."

Se un sito web si dimentica di inviare questi header, gli attacchi reali diventano più facili. Alcuni esempi che sono davvero successi ad aziende reali:

- **Clickjacking.** Senza `X-Frame-Options`, un attaccante può caricare il sito web di una vittima dentro un iframe nascosto su una pagina malevola, ingannarti facendoti cliccare quello che sembra un pulsante sulla pagina malevola, ma il tuo clic in realtà finisce sul sito della vittima. Usato contro Twitter, Facebook e le impostazioni di Adobe Flash nel periodo tra il 2008 e il 2012.
- **SSL stripping.** Senza `Strict-Transport-Security`, un attaccante sullo stesso wifi della caffetteria può declassare la tua prima visita a HTTP in chiaro, mettersi in mezzo e leggere o modificare tutto ciò che invii. Moxie Marlinspike lo ha dimostrato a Black Hat 2009 ed è ancora efficace su qualsiasi sito che si dimentica di HSTS.
- **MIME sniffing.** Senza `X-Content-Type-Options: nosniff`, un browser potrebbe trattare un file caricato come qualcosa che non è (un'"immagine" che il browser decide assomigliare a HTML e che esegue come script). Un attacco reale contro le vecchie versioni di IE che funzionava perché il browser cercava di rendersi utile.

Questo scanner NON corregge nessuno di questi problemi. Ti dice solo se a un sito web mancano gli header che li avrebbero prevenuti. Questo è il primo lavoro utile nella sicurezza: sapere cosa non va prima di poter provare a correggerlo.

## Cosa imparerai costruendolo

Questo non è un tutorial che ti insegna Python dalla primissima riga. Presuppone che tu sappia installare qualcosa sul tuo computer ed eseguire un comando nel terminale. Tutto il resto lo vedremo insieme.

Dopo aver completato questo progetto dovresti capire:

**Concetti di sicurezza**
- Cosa sono gli header HTTP e perché alcuni di essi sono importanti per la sicurezza
- Gli attacchi specifici che ciascuno dei principali security header previene (clickjacking, MIME sniffing, mixed content, XSS, fuga del referer)
- Come una "griglia di valutazione" ti permette di trasformare molti piccoli controlli in un unico punteggio finale, che è ciò che fanno scanner reali come Mozilla Observatory e securityheaders.com

**Concetti di Python**
- Come fare una richiesta HTTP dal codice con `httpx`
- Come le `dataclasses` ti danno piccole "forme" di dati senza scrivere i costruttori a mano
- Come i type hint `Literal` vincolano un valore a un piccolo insieme fisso di stringhe, così gli errori di battitura vengono intercettati presto
- Come separare il "codice che fa I/O" (parla con la rete) dal "codice che è pura matematica", così puoi testare la matematica senza toccare la rete
- Come `pytest` esegue i test, cos'è una fixture e come `respx` ti permette di simulare risposte HTTP

**Tooling da riga di comando**
- Come `argparse` fa il parsing di flag come `--timeout 5` in un oggetto ordinato
- Come gli exit code (il numero che un programma restituisce quando termina) possono comunicare successo o fallimento ad altri programmi e ai sistemi di CI
- Come `rich` disegna tabelle e pannelli colorati nel terminale

## Come appare quando lo esegui

```bash
$ just run -- https://github.com
```

Vedrai qualcosa di simile (nel terminale i colori sono reali):

```
                  Headers for https://github.com/ (HTTP 200)
┌─────────────────────────────┬─────────┬──────────┬─────────────────────────┐
│ header                      │ status  │ severity │ note                    │
├─────────────────────────────┼─────────┼──────────┼─────────────────────────┤
│ Strict-Transport-Security   │ ok      │ high     │ Present and contains... │
│ Content-Security-Policy     │ ok      │ high     │ Present                 │
│ X-Content-Type-Options      │ ok      │ medium   │ Present and contains... │
│ X-Frame-Options             │ ok      │ medium   │ Present                 │
│ Referrer-Policy             │ ok      │ low      │ Present                 │
│ Permissions-Policy          │ missing │ low      │ Header ... is not set   │
└─────────────────────────────┴─────────┴──────────┴─────────────────────────┘
╭─ Result ───────────────────╮
│ Grade: A                   │
│ Score: 95 / 100            │
╰────────────────────────────╯

Recommendations:
  • Permissions-Policy — Add: Permissions-Policy: camera=(), microphone=(), geolocation=()
```

I punti principali:
- Una riga verde della tabella significa "questo header è impostato ed è utile".
- Una riga gialla significa "questo header è impostato ma il valore è sbagliato" (per esempio, `Strict-Transport-Security` è stato inviato ma con `max-age=0`, che di fatto lo disattiva).
- Una riga rossa significa "questo header manca del tutto".
- Il pannello del voto riassume tutto in una sola lettera, così puoi capire a colpo d'occhio se un sito ha le basi in ordine.

## A chi è rivolto

Puoi essere assolutamente alle prime armi con Python e con la sicurezza. Dovresti già sapere:

- Come aprire un terminale sul tuo computer.
- Come clonare un repository git (o scaricare una cartella di file).
- Come leggere del testo senza andare nel panico quando qualcosa non funziona al primo tentativo.

NON devi sapere in anticipo:

- Cos'è HTTP. Lo spiegheremo.
- Cos'è una dataclass. Lo spiegheremo.
- Cos'è pytest, cos'è il mocking, cos'è argparse. Tutto spiegato.

## Prerequisiti in concreto

**Software che devi avere installato sul tuo computer.** Sono tutti e tre gratuiti.

- **Python 3.13 o più recente.** Python è il linguaggio di programmazione in cui è scritto. La versione 3.13 ha aggiunto parte della sintassi che usiamo. Se hai un Python più vecchio (3.10, 3.11, 3.12) otterrai degli errori. Verifica con `python3 --version`.
- **Un terminale.** Mac e Linux ne hanno uno integrato (Terminal.app o uno qualsiasi delle decine disponibili su Linux). Gli utenti Windows dovrebbero installare Windows Terminal dal Microsoft Store, oppure usare quello integrato in VS Code.
- **Una connessione internet funzionante.** Lo scanner fa una vera richiesta HTTPS verso qualunque URL tu gli indichi.

Lo script di installazione (`install.sh`) configurerà tutto il resto per te: `uv` (un package manager per Python), `just` (un command runner), un virtual environment e tutte le librerie che questo progetto usa.

## Avvio rapido

Dalla cartella del progetto:

```bash
# One-shot install. Sets up Python tooling, installs dependencies.
./install.sh

# Scan a real site.
just run -- https://example.com

# Add a custom timeout if a site is slow to respond.
just run -- https://github.com --timeout 5

# Run the tests to confirm the code works on your machine.
just test
```

Se `./install.sh` fallisce con "permission denied", esegui prima `chmod +x install.sh` per marcare il file come eseguibile, poi riprova.

## Struttura del progetto

L'intero progetto è volutamente piccolo. Due file Python più il tooling.

```
http-headers-scanner/
├── http_headers_scanner.py        the actual scanner: rules, scoring, CLI
├── test_http_headers_scanner.py   tests for the scanner
├── pyproject.toml                 project metadata + dependency list
├── uv.lock                        exact versions of every dependency
├── justfile                       shortcut commands (just test, just run, etc.)
├── install.sh                     one-shot installer
├── README.md                      a brief readme for the GitHub page
└── learn/                         this folder you are reading
    ├── 00-OVERVIEW.md             you are here
    ├── 01-CONCEPTS.md             what HTTP is, what each header does, real attacks
    ├── 02-ARCHITECTURE.md         how the code is organised and why
    ├── 03-IMPLEMENTATION.md       line by line walkthrough of the code
    └── 04-CHALLENGES.md           extension ideas to make it your own
```

Tutto ciò che è importante si trova in **un solo file Python** (`http_headers_scanner.py`). È voluto, per un progetto foundations. Un solo file che puoi tenere a mente. I progetti più grandi (il password manager, qualsiasi cosa in `intermediate/`) sono suddivisi in molti file perché devono esserlo. Questo non ne ha bisogno.

## Problemi comuni la prima volta

**`python3: command not found`**
Probabilmente hai Python installato ma con un nome diverso. Prova `python --version`. Se risponde 3.13+, modifica `install.sh` e sostituisci `python3` con `python`. Se nessuno dei due funziona, installa Python da python.org (oppure `brew install python@3.13` su Mac, `sudo apt install python3.13` su Debian/Ubuntu).

**`./install.sh: Permission denied`**
Il file non è marcato come eseguibile. Esegui `chmod +x install.sh` e riprova.

**`just: command not found` dopo l'installazione**
Lo script di installazione mette `just` in `~/.local/bin`. Quella cartella potrebbe non essere nel tuo PATH in un terminale appena aperto. Riavvia il terminale oppure esegui `export PATH="$HOME/.local/bin:$PATH"`. Per renderlo permanente, aggiungi quella riga a `~/.bashrc` o `~/.zshrc`.

**Errori di rete durante la scansione di un URL reale**
Se sei dietro un firewall aziendale o una VPN, alcuni siti potrebbero rifiutarsi di rispondere, oppure potrebbero bloccare lo User-Agent predefinito dello scanner. Non è un bug nel codice, è il mondo che è fastidioso. Prova prima un URL diverso come `https://example.com` per confermare che i meccanismi di base funzionino.

## Dove andare dopo

1. **[01-CONCEPTS.md](./01-CONCEPTS.it.md)** spiega HTTP da zero, cosa fa effettivamente ciascun header e gli attacchi reali che prevengono. Leggilo prima del codice. Il codice ha molto più senso una volta che capisci da cosa proteggono gli header.
2. **[02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md)** spiega come il codice è suddiviso in parti e perché. Utile quando vuoi estendere lo scanner senza fare pasticci.
3. **[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md)** ripercorre il codice riga per riga. È il più lungo dei cinque file.
4. **[04-CHALLENGES.md](./04-CHALLENGES.it.md)** è per quando hai letto tutto e vuoi idee per estendere il progetto.

## Progetti correlati in questo repo

- `PROJECTS/foundations/hash-identifier/`: ancora più piccolo. Pura logica, niente rete. Un buon riscaldamento se questo ti sembra troppo grande.
- `PROJECTS/foundations/password-manager/`: il passo successivo rispetto a questo. Più file, crittografia vera.
- `PROJECTS/advanced/bug-bounty-platform/`: come appare una versione seria di questa idea quando la fai crescere fino a diventare un prodotto completo.
