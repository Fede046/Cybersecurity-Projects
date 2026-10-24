<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# Nadezhda: Panoramica

## Cos'è

Un motore di intelligence su notizie di sicurezza e CVE scritto in Go. Lo punti su un insieme di feed RSS, e lui li scarica con educazione, raggruppa in cluster la stessa notizia così come appare su testate diverse, estrae ogni CVE menzionata, arricchisce ciascuna con intelligence autorevole sugli exploit, ordina l'intero insieme in base alla rilevanza nel mondo reale e ti consegna un dossier navigabile nel tuo terminale. Viene distribuito come un singolo binario statico con uno store SQLite locale, e per fare tutto questo non richiede alcuna API key.

Lo scopo del progetto è capire, costruendolo, come si trasforma un getto continuo di titoli di sicurezza in un segnale ordinato per priorità. Ottieni un aggregatore funzionante che puoi eseguire su feed reali, una pipeline di enrichment senza chiavi alimentata dalle stesse fonti autorevoli che userebbe un SOC, e una codebase abbastanza piccola da essere letta in un pomeriggio.

## Perché è importante

Ogni giorno viene pubblicata più news di sicurezza di quanta qualsiasi persona possa leggerne, e quasi tutta è rumore rispetto all'unica notizia che conta per te in questo momento. Il problema difficile non è raccogliere i titoli. È decidere quali tre dei quattrocento devi affrontare prima di pranzo.

Il segnale è di solito una combinazione degli stessi fattori. Una vulnerabilità ripresa da molte testate nel giro di poche ore è in trend per un motivo. Una CVE che la CISA ha appena aggiunto al suo catalogo Known Exploited Vulnerabilities viene usata oggi contro obiettivi reali. Una falla con un punteggio EPSS alto è una che il resto del mondo sta per iniziare a sfruttare. Nadezhda esiste per calcolare questa combinazione al posto tuo e ordinare in base ad essa.

Il costo di non cogliere il segnale non è ipotetico.

- **Log4Shell, dicembre 2021.** CVE-2021-44228, una falla di remote code execution in Apache Log4j 2, è passata da una issue su GitHub a ogni testata di sicurezza del pianeta nel giro di poche ore. Il suo punteggio base CVSS è un perfetto 10.0, è finita quasi subito nel catalogo CISA KEV e il suo punteggio EPSS stava vicino al massimo della scala. Un tool che ordina per velocità cross-testata più KEV più EPSS l'avrebbe portata in cima alla lista già al primo scrape, che è esattamente il comportamento che questo progetto è costruito per produrre.
- **La violazione di Equifax, 2017.** Gli attaccanti sono entrati attraverso una vulnerabilità nota di Apache Struts, CVE-2017-5638, per la quale esisteva una patch da mesi. Le informazioni necessarie per dare priorità a quella patch erano nei feed pubblici per tutto il tempo. Il fallimento fu di triage, non di intelligence.
- **MOVEit, 2023.** Il gruppo Cl0p ha sfruttato in massa CVE-2023-34362 nel prodotto MOVEit Transfer, colpendo centinaia di organizzazioni. Gli advisory, l'inserimento in KEV e la copertura delle testate sono arrivati tutti in una finestra ristretta. La velocità era l'indizio.

**Scenari reali in cui si applica:**
- **Triage di threat intelligence.** Un analista esegue uno scrape ogni mattina e legge la parte alta di un dossier ordinato invece di quaranta schede del browser.
- **Contenuti e ricerca.** Uno scrittore o un formatore trova il cluster che ogni testata sta coprendo, vede le CVE e i segnali di exploit che ci stanno dietro, e lo trasforma in un post o in una lezione.
- **Consapevolezza situazionale personale.** Un ingegnere tiene una watchlist dei vendor e dei prodotti che usa, e viene avvisato quando una falla elencata in KEV in uno di essi inizia a diventare di tendenza.

## Cosa imparerai

Questo progetto insegna come si costruisce una vera pipeline di aggregazione e prioritizzazione. Costruendola tu stesso, capirai:

**Concetti di sicurezza:**
- **Lo stack di intelligence sulle CVE.** Come CVSS, CWE, il record del CVE Program, il catalogo CISA KEV e FIRST EPSS rispondono ciascuno a una domanda diversa su una vulnerabilità, e come combinarli senza una API key.
- **Perché KEV ed EPSS battono il solo CVSS.** Un CVSS 9.8 che nessuno sfrutta è meno urgente di un CVSS 6.5 presente nel catalogo KEV. Il progetto codifica questo giudizio nel suo ranking.
- **Il clustering cross-fonte come segnale.** Perché la stessa notizia che compare rapidamente su molte testate è di per sé la cosa più utile che puoi misurare, e come misurarla.
- **Recupero automatizzato educato.** Richieste condizionali, rate limiting per host, e dove robots.txt si applica e dove no.

**Competenze tecniche:**
- **Ingestione concorrente e fail soft.** Scaricare molti feed in parallelo in modo che un feed rotto non interrompa mai l'esecuzione.
- **Ranking deterministico.** Costruire un modello di scoring il cui output è una funzione fissa dei suoi input, così che lo stesso corpus venga sempre ordinato allo stesso modo e possa essere testato contro un ordine golden.
- **Un singolo binario statico con un database embedded.** Usare SQLite in puro Go in modo da non avere CGO, né database esterno, né dipendenze a runtime.
- **Un daemon disaccoppiato.** Progettare uno scheduler che non sa nulla del lavoro che esegue, così da poterlo testare con unit test usando un clock finto.

**Strumenti e tecniche:**
- **`gofeed`** per il parsing di RSS e Atom, e **`goquery`** per il percorso di fallback HTML.
- **`modernc.org/sqlite`**, un SQLite senza cgo, in modalità WAL così che un lettore e uno scrittore coesistano.
- **`bubbletea`** e **`lipgloss`** per la UI da terminale.
- **Ollama** per un modello linguistico locale e senza chiavi, con OpenAI, Gemini e Anthropic disponibili come alternative opt-in.

## Prerequisiti

Non ti serve esperienza pregressa di threat intelligence. Ti serve però una certa dimestichezza con quanto segue.

**Conoscenze richieste:**
- **Basi di Go.** Struct, interfacce, goroutine ed errori. Se riesci a leggere un ciclo `for` su un canale, riesci a leggere questo codice.
- **SQL a livello base.** Lo store è semplice SQLite con una manciata di tabelle e query scritte a mano.
- **Cos'è una CVE.** Che CVE-2021-44228 indica una singola vulnerabilità, che CVSS ne valuta la gravità e che di solito esiste una patch. Il resto è spiegato in [01-CONCEPTS.it.md](./01-CONCEPTS.it.md).

**Strumenti necessari:**
- **Un toolchain Go**, versione 1.25 o successiva. Lo script `install.sh` ne installa uno al posto tuo se manca.
- **Nient'altro per eseguire il core.** Nessuna API key, nessun database server, nessun Docker. Le fonti di enrichment predefinite sono tutte senza chiavi.

**Utile ma non obbligatorio:**
- **Docker**, solo se vuoi il modello linguistico locale opzionale, che gira come container Ollama.
- Una scorsa al [catalogo CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) e alla documentazione di [FIRST EPSS](https://www.first.org/epss/).

## Avvio rapido

```bash
# Install (grabs a prebuilt binary, no Go needed):
curl -fsSL https://angelamos.com/nadezhda/install.sh | bash
# or, with a Go toolchain:
go install github.com/CarterPerez-dev/nadezhda/cmd/nadezhda@latest

# Pull the latest news, cluster it, and enrich every CVE:
nadezhda scrape

# Browse the ranked dossier:
nadezhda tui

# Render a Markdown digest of the top stories:
nadezhda digest --top 20
```

Output atteso: `scrape` stampa una tabella per fonte (conteggi di parsed, new, duplicate, CVE), poi una riga come `132 clusters (14 multi-source, largest 5)`, poi un riepilogo dell'enrichment come `enriched 80/80 CVEs (14 KEV, 0 not found)`. `tui` apre un elenco colorato e scorrevole in cui ogni notizia mostra il suo rank, le sue testate e la sua CVE peggiore, e premendo un tasto si apre il dossier completo del cluster selezionato.

## Struttura del progetto

```
security-news-scraper/
├── cmd/nadezhda/          # the CLI: one file per command, plus the shared pipeline seam
├── internal/
│   ├── fetch/             # concurrent, rate-limited, conditional HTTP
│   ├── parse/             # RSS/Atom via gofeed, HTML fallback via goquery
│   ├── normalize/         # canonical URLs, content hashing, time parsing
│   ├── ingest/            # the fan-out orchestrator (fail soft per source)
│   ├── cluster/           # union-find clustering by title and shared CVE
│   ├── cve/               # keyless CVE clients: CVE list, KEV, EPSS (+ optional NVD)
│   ├── enrich/            # enrichment orchestration and the TTL cache
│   ├── rank/              # the deterministic weighted scoring model
│   ├── store/             # SQLite: connection, migrations, typed queries
│   ├── ai/                # the opt-in ideation layer (four providers)
│   ├── setup/             # the in-binary credential wizard
│   ├── watch/             # the daemon scheduler (pure, injected pipeline)
│   ├── tui/               # the bubbletea browser
│   └── export/            # Markdown and JSON digest renderers
├── testdata/              # captured feed and API fixtures (drive offline tests)
├── install.sh             # the one-shot installer
└── justfile
```

La cosa più importante da capire per prima è la pipeline in `cmd/nadezhda/pipeline.go` e `scrape.go`. Tutto ciò che sta in `internal` esiste per alimentare una sola funzione, `ingestAndCluster`, e tutto ciò che sta a valle esiste per ordinare e presentare ciò che essa ha prodotto.

## Prossimi passi

1. **Capisci le idee.** Leggi [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) per lo stack di intelligence sulle CVE, il clustering come segnale e il modello di ranking, ciascuno ancorato a un incidente reale.
2. **Guarda il design.** Leggi [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md) per il layout dei package, la decisione di un enrichment senza chiavi e il flusso dei dati dal feed al dossier.
3. **Percorri il codice.** Leggi [03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md) per seguire una CVE da un elemento del feed fino a un cluster ordinato e arricchito.
4. **Estendilo.** Leggi [04-CHALLENGES.it.md](./04-CHALLENGES.it.md) per progetti che vanno da "aggiungi una fonte" a "sostituisci il clustering con SimHash".

## Problemi comuni

**`scrape` mostra una fonte con un errore**
```
  theregister: fetch theregister: context deadline exceeded
```
Soluzione: un feed lento o rotto non interrompe mai un'esecuzione. Le altre fonti vengono comunque ingerite, quella fallita viene riportata in fondo e il prossimo scrape la riprova. Un fallimento persistente di solito significa che l'URL del feed è stato spostato. Controlla `nadezhda sources`.

**`tui` dice che non c'è nulla da mostrare**
```
no clusters yet. run: nadezhda scrape
```
Soluzione: la TUI legge ciò che è già nello store. Esegui prima `nadezhda scrape` una volta. Lo scrape esegue l'enrichment automaticamente, quindi non c'è un passaggio separato.

**`ideate` dice che l'AI non è configurata**
Soluzione: il layer AI è opt-in e spento per impostazione predefinita. Esegui `nadezhda ai` per configurare un provider (incolla una chiave, oppure puntalo a un Ollama locale). L'aggregatore è pienamente utile anche con l'AI disabilitata.

## Progetti correlati

Se ti è sembrato interessante, guarda:
- **sbom-generator-vulnerability-matcher**: l'altra metà della storia delle CVE, che confronta le vulnerabilità in questo feed con le dipendenze che distribuisci davvero.
- **ja3-ja4-tls-fingerprinting**: la stessa forma di tool, un motore di intelligence senza chiavi su uno store SQLite embedded, applicato a fingerprint di rete invece che a notizie.
