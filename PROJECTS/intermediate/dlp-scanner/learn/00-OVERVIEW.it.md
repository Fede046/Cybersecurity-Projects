# 00-OVERVIEW.md

# DLP Scanner

## Che cos'è

Uno scanner di Data Loss Prevention da riga di comando che rileva dati sensibili su tre superfici: file (PDF, DOCX, XLSX, CSV, JSON, XML, YAML, Parquet, Avro, archivi, email), database (PostgreSQL, MySQL, MongoDB, SQLite) e catture di rete (PCAP/PCAPNG con parsing dei protocolli e riassemblaggio TCP). Usa una pipeline di confidence scoring che combina il matching tramite regex, la validazione dei checksum (Luhn per le carte di credito, Mod-97 per gli IBAN, Mod-11 per i numeri NHS), l'analisi di prossimità delle keyword e il rilevamento dell'entropia di Shannon. I finding sono classificati per severità e mappati sui framework di compliance (HIPAA, PCI-DSS, GDPR, CCPA, SOX, GLBA, FERPA). L'output supporta tabelle Rich in console, JSON, SARIF 2.1.0 per la CI/CD e CSV per i team di compliance.

## Perché è importante

Le violazioni di dati che comportano l'esposizione di PII continuano a verificarsi perché le organizzazioni non riescono a trovare dati sensibili di cui non sanno l'esistenza. La violazione di Equifax del 2017 ha esposto 147 milioni di SSN a partire da un'applicazione Apache Struts non aggiornata, ma il problema di fondo era che gli SSN erano memorizzati in chiaro in più tabelle di database senza che nessuno tenesse traccia di dove vivessero quei dati. Nel 2019 Capital One ha perso 100 milioni di richieste di credito da un bucket S3 perché un WAF mal configurato permetteva una server-side request forgery, e nessuno aveva scansionato quei file per accorgersi che SSN e numeri di carte di credito non cifrati stavano in semplici export CSV. La violazione di Marriott (2018) ha esposto 500 milioni di record, inclusi 5,25 milioni di numeri di passaporto non cifrati, in parte perché il sistema di prenotazione di Starwood era stato integrato senza un inventario dei dati che avrebbe segnalato quei campi come sensibili.

Non sono problemi di fallimento del firewall. Sono problemi di mancanza di visibilità. Gli strumenti DLP esistono per rispondere a "dove sono i nostri dati sensibili?" prima che siano gli attaccanti a rispondere per te. Le soluzioni commerciali (Symantec DLP, Microsoft Purview, Netskope) costano cifre a sei zeri e richiedono un deployment enterprise, ma la logica di rilevamento di base è semplice: pattern matching con validazione, analisi del contesto per ridurre i falsi positivi e mappatura sui framework di compliance per dare priorità alla remediation.

Questo progetto costruisce un motore DLP da zero, insegnandoti le stesse tecniche di rilevamento che alimentano i sistemi in produzione.

**Scenari reali in cui si applica:**
- Security engineer che scansionano file share prima di una migrazione al cloud per trovare PII da cifrare
- Team di compliance che verificano tabelle di database alla ricerca di PHI regolamentate da HIPAA che non dovrebbero essere in chiaro
- Analisti SOC che ispezionano catture PCAP alla ricerca di credenziali o PII trasmesse in chiaro
- Team DevOps che eseguono controlli DLP nelle pipeline CI/CD per intercettare i segreti prima che arrivino in produzione
- Incident responder che determinano quali dati sensibili erano accessibili da un segmento di rete compromesso

## Cosa imparerai

**Concetti di sicurezza:**
- I livelli di classificazione dei dati e come i dati PII, PHI, PCI e le credenziali si mappano sui requisiti normativi
- Confidence scoring: perché la sola regex produce falsi positivi e come la validazione dei checksum, le keyword di contesto e la co-occorrenza di entità li riducono
- Mappatura sui framework di compliance: i 18 identificatori di HIPAA, i dati del titolare della carta di PCI-DSS, le categorie di dati personali del GDPR, le informazioni sui consumatori del CCPA
- Network DLP: rilevare dati sensibili in transito, l'esfiltrazione DNS tramite label di sottodominio ad alta entropia, payload codificati in base64 nei body HTTP
- Strategie di redaction: perché non memorizzi mai il contenuto grezzo trovato nei finding

**Competenze tecniche:**
- Costruire una pipeline di estrazione del testo multi-formato che gestisce oltre 14 formati di file attraverso un'interfaccia Protocol unificata
- Introspezione dello schema di database su 4 motori di database con campionamento statistico (TABLESAMPLE BERNOULLI, aggregazione $sample)
- Riassemblaggio di stream TCP da pacchetti grezzi usando l'ordinamento per numero di sequenza e la normalizzazione bidirezionale della chiave di flusso
- Pipeline di confidence scoring: punteggi base, boost da checksum, finestre di prossimità delle keyword di contesto, co-occorrenza di entità
- Output SARIF 2.1.0 per l'integrazione con il code scanning di GitHub

**Strumenti e tecniche:**
- CLI Typer con parametri in stile Annotated e propagazione delle opzioni globali tramite il contesto di Click
- Pydantic 2.x per la validazione della configurazione con caricamento da YAML
- structlog con integrazione della stdlib per il logging JSON strutturato
- orjson per la serializzazione JSON ad alte prestazioni
- asyncpg, aiomysql, pymongo async, aiosqlite per l'accesso asincrono ai database
- dpkt per il parsing veloce di PCAP (100 volte più veloce di Scapy)
- pytest con hypothesis per il property-based testing delle regole di rilevamento

## Prerequisiti

**Conoscenze richieste:**
- Fondamenti di Python: dataclass, type hint, list comprehension, context manager
- Networking di base: TCP/IP, porte, pacchetti, cosa contengono i file PCAP
- SQL di base: SELECT, WHERE, schemi di tabelle, tipi di colonna
- Basi di sicurezza: cosa sono le PII, perché SSN e numeri di carte di credito vanno protetti, quali framework di compliance esistono

**Strumenti di cui avrai bisogno:**
- Python 3.12+ (usa la sintassi generica moderna e `from __future__ import annotations`)
- Il package manager uv (installazione: `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Un terminale con supporto UTF-8 (per l'output in console di Rich)

**Utili ma non obbligatori:**
- Esperienza con regex e pattern matching
- Familiarità con dpkt o Scapy per l'analisi dei pacchetti
- Conoscenza degli URI di database e delle connection string
- Comprensione del formato SARIF per il tooling di sicurezza in CI/CD

## Avvio rapido

```bash
bash install.sh
dlp-scan file ./data
dlp-scan file ./data -f json -o results.json
dlp-scan db sqlite:///path/to/database.db
dlp-scan report summary results.json
```
