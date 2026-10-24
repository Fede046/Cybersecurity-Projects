# 03-IMPLEMENTATION.md

# Guida all'implementazione

Questo documento illustra come funziona il codice. Trattiamo il motore di rilevamento, l'estrazione dai file, l'analisi di rete e l'integrazione della CLI, con snippet di codice tratti dal progetto reale.

## Struttura dei file

```
src/dlp_scanner/
├── __init__.py
├── cli.py                  # Typer entry point
├── engine.py               # Scan orchestration
├── config.py               # Pydantic config models
├── constants.py            # Thresholds, types, defaults
├── models.py               # Finding, Location, ScanResult
├── compliance.py           # Rule-to-framework mapping
├── redaction.py            # Snippet masking
├── log.py                  # structlog configuration
├── scoring.py              # Shared match-to-finding conversion
├── commands/
│   ├── scan.py             # file, db, network commands
│   └── report.py           # convert, summary commands
├── detectors/
│   ├── base.py             # DetectionRule, DetectorMatch
│   ├── pattern.py          # Regex + checksum detection
│   ├── context.py          # Keyword proximity scoring
│   ├── entropy.py          # Shannon entropy detection
│   ├── registry.py         # Central detector registry
│   └── rules/
│       ├── pii.py          # SSN, email, phone, passport
│       ├── financial.py    # Credit cards, IBAN, NHS
│       ├── credentials.py  # AWS, GitHub, JWT, Stripe
│       └── health.py       # Medical records, DEA, NPI
├── extractors/
│   ├── base.py             # Extractor protocol
│   ├── plaintext.py        # .txt, .log, .cfg, source code
│   ├── pdf.py              # .pdf via PyMuPDF
│   ├── office.py           # .docx, .xlsx, .xls
│   ├── structured.py       # .csv, .json, .xml, .yaml, .parquet, .avro
│   ├── archive.py          # .zip, .tar.gz, .tar.bz2
│   └── email.py            # .eml, .msg
├── network/
│   ├── pcap.py             # PCAP/PCAPNG packet reader
│   ├── flow_tracker.py     # TCP flow reassembly
│   ├── protocols.py        # DPI protocol identification
│   └── exfiltration.py     # DNS exfil detection
├── reporters/
│   ├── base.py             # Reporter protocol
│   ├── console.py          # Rich terminal output
│   ├── json_report.py      # Structured JSON
│   ├── sarif.py            # SARIF 2.1.0
│   └── csv_report.py       # Flat CSV
└── scanners/
    ├── base.py             # Scanner protocol
    ├── file_scanner.py     # Directory walking + extraction
    ├── db_scanner.py       # DB schema introspection
    └── network_scanner.py  # PCAP payload scanning
```

## Costruire il motore di rilevamento

### Regole di rilevamento

Ogni regola di rilevamento è una struttura dati, non una gerarchia di classi. La dataclass `DetectionRule` contiene il pattern regex, il confidence score base, la funzione di validazione opzionale, le keyword di contesto e i tag dei framework di compliance:

```python
@dataclass(frozen=True, slots=True)
class DetectionRule:
    rule_id: str
    rule_name: str
    pattern: re.Pattern[str]
    base_score: float
    context_keywords: list[str] = field(default_factory=list)
    validator: Callable[[str], bool] | None = None
    compliance_frameworks: list[str] = field(
        default_factory=list
    )
```

I moduli di regole esportano semplici liste di queste struct. Ecco la regola SSN da `detectors/rules/pii.py`:

```python
SSN_PATTERN = re.compile(
    r"\b(?!000|666|9\d{2})\d{3}"
    r"[-\s]?"
    r"(?!00)\d{2}"
    r"[-\s]?"
    r"(?!0000)\d{4}\b"
)

PII_RULES: list[DetectionRule] = [
    DetectionRule(
        rule_id="PII_SSN",
        rule_name="US Social Security Number",
        pattern=SSN_PATTERN,
        base_score=0.45,
        context_keywords=SSN_CONTEXT,
        validator=_validate_ssn,
        compliance_frameworks=[
            "HIPAA", "CCPA", "GLBA", "GDPR",
        ],
    ),
    ...
]
```

La regex usa negative lookahead (`(?!000|666|9\d{2})`) per rifiutare i numeri di area SSN che la Social Security Administration non ha mai assegnato. È un primo filtro strutturale. La validazione vera avviene in `_validate_ssn`, che il `PatternDetector` chiama per ogni corrispondenza regex.

**Perché base_score è 0,45 e non di più:** Un numero di 9 cifre che corrisponde al formato SSN compare continuamente in numeri di serie, codici zip+4, frammenti di numeri di telefono e dati di test. La stringa `456-78-9012` corrisponde al pattern SSN e supera la validazione di area/gruppo/serie, ma senza contesto potrebbe essere qualsiasi cosa. Una base di 0,45 la mantiene nel livello di severità "medium" finché i boost di contesto non la spingono più in alto.

### Validazione dei checksum

I tre validatori di checksum mostrano approcci matematici diversi allo stesso problema: distinguere gli identificatori reali dalle sequenze casuali di cifre.

**Algoritmo di Luhn** per le carte di credito (in `detectors/rules/financial.py`):

```python
def luhn_check(number: str) -> bool:
    digits = [int(d) for d in number if d.isdigit()]
    if len(digits) < 13:
        return False

    odd_digits = digits[-1::-2]
    even_digits = digits[-2::-2]
    total = sum(odd_digits)
    for d in even_digits:
        total += sum(divmod(d * 2, 10))
    return total % 10 == 0
```

L'algoritmo lavora da destra a sinistra: prendi una cifra sì e una no partendo da quella più a destra, e sommale. Per le cifre rimanenti, raddoppia ciascuna e, se il risultato supera 9, sottrai 9 (che è ciò che fa `sum(divmod(d * 2, 10))`). Se il totale complessivo è divisibile per 10, il numero è valido. Un numero casuale di 16 cifre ha circa il 10% di probabilità di superare Luhn, quindi riduce i falsi positivi di circa il 90%.

**Mod-97** per gli IBAN (ISO 7064):

```python
def iban_check(value: str) -> bool:
    cleaned = value.replace(" ", "").upper()
    if len(cleaned) < 15 or len(cleaned) > 34:
        return False

    rearranged = cleaned[4:] + cleaned[:4]
    numeric = ""
    for char in rearranged:
        if char.isalpha():
            numeric += str(ord(char) - ord("A") + 10)
        else:
            numeric += char

    return int(numeric) % 97 == 1
```

Sposta il codice paese e le cifre di controllo (i primi 4 caratteri) alla fine, converti le lettere in numeri a due cifre (A=10, B=11, ecc.), poi controlla che l'intero numero mod 97 sia uguale a 1. Il tasso di falsi positivi è di circa 1 su 97.

**Mod-11** per i numeri NHS:

```python
def nhs_check(value: str) -> bool:
    digits = value.replace("-", "").replace(" ", "")
    if len(digits) != 10 or not digits.isdigit():
        return False

    weights = range(10, 1, -1)
    total = sum(
        int(d) * w
        for d, w in zip(digits[:9], weights, strict=False)
    )
    remainder = 11 - (total % 11)
    if remainder == 11:
        remainder = 0
    if remainder == 10:
        return False
    return remainder == int(digits[9])
```

Moltiplica le prime 9 cifre per pesi decrescenti (10, 9, 8, ..., 2), sommale, calcola `11 - (sum mod 11)` e confronta il risultato con la cifra di controllo. Se il risultato è 10, il numero non è valido (l'NHS non li emette mai). Se il risultato è 11, la cifra di controllo è 0.

**Luhn-80840** per gli NPI (in `detectors/rules/health.py`):

```python
def _validate_npi(value: str) -> bool:
    digits = value.replace("-", "").replace(" ", "")
    if len(digits) != 10 or not digits.isdigit():
        return False

    prefixed = "80840" + digits
    total = 0
    for i, d in enumerate(reversed(prefixed)):
        n = int(d)
        if i % 2 == 1:
            n *= 2
            if n > 9:
                n -= 9
        total += n
    return total % 10 == 0
```

La validazione degli NPI (National Provider Identifier) è una variante di Luhn. Il trucco è anteporre `80840` (il prefisso del settore sanitario assegnato dall'ANSI) prima di eseguire l'algoritmo di Luhn standard. Questo prefisso non fa parte dell'NPI stesso, ma lo standard ISO lo richiede per il calcolo della cifra di controllo. Un numero casuale di 10 cifre ha circa il 10% di probabilità di superarla, il che rende questo controllo utile ma non definitivo. Il punteggio base di 0,10 riflette il fatto che i pattern NPI corrispondono a molti numeri di 10 cifre non correlati, e servono keyword di contesto come "provider" o "npi" per spingere il punteggio in un territorio su cui si possa agire.

### Rilevamento dei pattern

Il `PatternDetector` in `detectors/pattern.py` itera su tutte le regole attive, esegue ogni regex sul testo in input, filtra tramite l'allowlist e applica la validazione dei checksum:

```python
class PatternDetector:
    def detect(self, text: str) -> list[DetectorMatch]:
        matches: list[DetectorMatch] = []

        for rule in self._rules:
            for m in rule.pattern.finditer(text):
                matched_text = m.group()

                if self._is_allowlisted(matched_text):
                    continue

                score = rule.base_score

                if rule.validator is not None:
                    if rule.validator(matched_text):
                        score = min(1.0, score + CHECKSUM_BOOST)
                    else:
                        continue

                matches.append(
                    DetectorMatch(
                        rule_id=rule.rule_id,
                        ...
                        score=score,
                    )
                )

        return matches
```

Quando una regola ha un validatore e la corrispondenza non supera la validazione, la corrispondenza viene scartata del tutto (`continue`). Un pattern Visa che corrisponde a `4532015112830366` ma non supera Luhn non è una carta di credito. Quando la validazione passa, il punteggio riceve un boost di +0,30 (`CHECKSUM_BOOST`). È un boost aggressivo perché le corrispondenze che superano il checksum sono nella stragrande maggioranza reali: la combinazione Luhn + prefisso Visa ha un tasso di falsi positivi inferiore all'1%.

L'allowlist usa una ricerca in un frozen set, con default `KNOWN_TEST_VALUES` (numeri di carta di test comuni, SSN di esempio come `123-45-6789`). Questo impedisce agli strumenti DLP di segnalare i propri dati di test, una lamentela comune nei deployment in produzione.

### Scoring delle keyword di contesto

Dopo il rilevamento dei pattern, `apply_context_boost` in `detectors/context.py` scansiona il testo circostante alla ricerca di keyword che indicano che il valore trovato è effettivamente un dato sensibile:

```python
def apply_context_boost(
    text: str,
    matches: list[DetectorMatch],
    window_tokens: int = DEFAULT_CONTEXT_WINDOW_TOKENS,
) -> list[DetectorMatch]:
    tokens = text.lower().split()
    boosted: list[DetectorMatch] = []

    for match in matches:
        if not match.context_keywords:
            boosted.append(match)
            continue

        char_to_token = _char_offset_to_token_index(
            text, match.start
        )
        window_start = max(
            0, char_to_token - window_tokens
        )
        window_end = min(
            len(tokens), char_to_token + window_tokens
        )
        window_text = " ".join(
            tokens[window_start:window_end]
        )

        boost = _compute_keyword_boost(
            window_text,
            match.context_keywords,
            window_tokens,
        )

        new_score = min(1.0, match.score + boost)
        ...
```

La finestra è bidirezionale: 10 token in ciascuna direzione a partire dalla corrispondenza. Il boost è pesato in base alla distanza: una keyword proprio accanto alla corrispondenza contribuisce fino a `CONTEXT_BOOST_MAX` (0,35), mentre una al bordo della finestra non contribuisce quasi nulla. Questo riflette un'osservazione reale: "SSN: 456-78-9012" è quasi certamente un SSN, mentre "SSN" che compare 50 parole lontano da "456-78-9012" è un segnale più debole.

La funzione `_compute_keyword_boost` trova la migliore keyword corrispondente nella finestra e calcola `CONTEXT_BOOST_MAX * proximity_factor`, dove la prossimità è `1.0 - (distance / max_distance)`. Conta solo la keyword con il punteggio più alto, non la somma di tutte le keyword. Questo impedisce che il keyword stuffing gonfi i punteggi.

### Boost di co-occorrenza

Dopo il boost di contesto, `_apply_cooccurrence_boost` controlla se più tipi diversi di PII compaiono vicini tra loro:

```python
def _apply_cooccurrence_boost(
    matches: list[DetectorMatch],
) -> list[DetectorMatch]:
    if len(matches) < 2:
        return matches

    proximity_threshold = 500

    for i, match in enumerate(matches):
        has_neighbor = False
        for j, other in enumerate(matches):
            if i == j:
                continue
            if other.rule_id == match.rule_id:
                continue
            distance = abs(match.start - other.start)
            if distance < proximity_threshold:
                has_neighbor = True
                break

        if has_neighbor:
            new_score = min(
                1.0, match.score + COOCCURRENCE_BOOST
            )
            ...
```

Un SSN vicino a un numero di carta di credito è una prova più forte di ciascuno dei due da solo. Il boost è +0,15 (`COOCCURRENCE_BOOST`), e richiede valori di `rule_id` diversi (due SSN uno accanto all'altro non lo attivano). La soglia di 500 caratteri corrisponde grosso modo a un breve paragrafo o a qualche colonna di database.

Questa euristica conta nella pratica. I dati della violazione di Capital One contenevano export CSV in cui SSN, numeri di carte di credito e indirizzi comparivano in colonne adiacenti. Il rilevamento della co-occorrenza avrebbe segnalato questi file come priorità critica.

### Rilevamento dell'entropia di Shannon

L'`EntropyDetector` in `detectors/entropy.py` trova le regioni ad alta entropia che potrebbero contenere segreti, dati cifrati o credenziali codificate in base64:

```python
def shannon_entropy(data: bytes) -> float:
    if not data:
        return 0.0

    counts = Counter(data)
    total = len(data)
    return -sum(
        (c / total) * math.log2(c / total)
        for c in counts.values()
    )
```

L'entropia di Shannon misura il contenuto informativo medio per byte. Il testo inglese si colloca intorno a 3,5-4,5 bit. I dati codificati in base64 sono a 5,5-6,0. I byte veramente casuali si avvicinano a 8,0 (log2(256)). Il detector usa una finestra scorrevole di 256 byte con un passo di 128 byte:

```python
def detect_high_entropy_regions(
    data: bytes,
    threshold: float = DEFAULT_ENTROPY_THRESHOLD,
    window_size: int = WINDOW_SIZE,
    step: int = WINDOW_STEP,
) -> list[tuple[int, int, float]]:
    ...
    while i + window_size <= len(data):
        window = data[i:i + window_size]
        h = shannon_entropy(window)

        if h >= threshold:
            end = i + window_size
            while end + step <= len(data):
                next_window = data[
                    end - window_size + step:end + step
                ]
                next_h = shannon_entropy(next_window)
                if next_h < threshold:
                    break
                h = max(h, next_h)
                end += step

            regions.append((i, end, h))
            i = end
        else:
            i += step
```

Quando l'entropia supera la soglia (default 7,2), il detector estende la regione in avanti finché l'entropia non scende sotto la soglia. Questo unisce finestre adiacenti ad alta entropia in un'unica regione invece di riportare decine di rilevamenti sovrapposti.

La soglia di default di 7,2 è volutamente alta. I payload di rete che contengono dati di protocolli binari o contenuti compressi spesso arrivano a 6,0-7,0, il che genererebbe un volume enorme di falsi positivi. A 7,2, il detector cattura principalmente blob cifrati, segreti codificati in base64 e materiale di chiavi casuale.

## Pipeline di estrazione dai file

### Il protocollo Extractor

Tutti gli estrattori implementano un protocollo a due metodi:

```python
class Extractor(Protocol):
    def extract(self, path: str) -> list[TextChunk]: ...

    @property
    def supported_extensions(self) -> frozenset[str]: ...
```

Il `FileScanner` costruisce una mappa da estensione a estrattore all'inizializzazione, iterando su tutte le istanze degli estrattori e indicizzandole per le estensioni supportate. Quando scansiona un file, cerca l'estrattore in base all'estensione del file e chiama `extract`.

### Estrazione del testo semplice

Il `PlaintextExtractor` legge i file in chunk da 500 righe per mantenere limitata la memoria:

```python
class PlaintextExtractor:
    def extract(self, path: str) -> list[TextChunk]:
        chunks: list[TextChunk] = []

        with open(
            path, encoding="utf-8", errors="replace",
        ) as f:
            lines: list[str] = []
            line_number = 1
            chunk_start = 1

            for line in f:
                lines.append(line)
                if len(lines) >= CHUNK_MAX_LINES:
                    chunks.append(
                        TextChunk(
                            text="".join(lines),
                            location=Location(
                                source_type="file",
                                uri=path,
                                line=chunk_start,
                            ),
                        )
                    )
                    chunk_start = line_number + 1
                    lines = []
                line_number += 1

            if lines:
                chunks.append(...)

        return chunks
```

Ogni `TextChunk` porta nella sua `Location` il numero di riga iniziale, così i finding possono riportare in quale punto del file si è verificata la corrispondenza. Il parametro `errors="replace"` fa sì che i file di testo contaminati da dati binari (comuni nei file di log con dati binari incorporati) non mandino in crash l'estrattore.

### Costruzione della mappa delle estensioni

La funzione `_build_extension_map` in `file_scanner.py` costruisce la mappatura dalle estensioni agli estrattori:

```python
def _build_extension_map() -> dict[str, Extractor]:
    extractors: list[Extractor] = [
        PlaintextExtractor(),
        PDFExtractor(),
        DocxExtractor(),
        XlsxExtractor(),
        XlsExtractor(),
        CsvExtractor(),
        JsonExtractor(),
        XmlExtractor(),
        YamlExtractor(),
        ParquetExtractor(),
        AvroExtractor(),
        ArchiveExtractor(),
        EmlExtractor(),
        MsgExtractor(),
    ]

    ext_map: dict[str, Extractor] = {}
    for extractor in extractors:
        for ext in extractor.supported_extensions:
            ext_map[ext] = extractor

    return ext_map
```

Aggiungere un nuovo formato significa creare una classe estrattore con `extract` e `supported_extensions`, poi aggiungerla a questa lista. Lo scanner non ha bisogno di sapere nulla del formato.

### Logica di attraversamento del file scanner

Il metodo `FileScanner._scan_directory` applica una catena di filtri prima di smistare a un estrattore:

```python
def _scan_directory(self, directory, result):
    iterator = (
        directory.rglob("*")
        if self._file_config.recursive
        else directory.glob("*")
    )

    for path in iterator:
        if not path.is_file():
            continue
        if self._is_excluded(path, directory):
            continue

        suffix = _get_full_suffix(path)
        if suffix not in self._allowed_extensions:
            continue

        file_size = path.stat().st_size
        if file_size > max_bytes:
            continue
        if file_size == 0:
            continue

        self._scan_file(path, result)
        result.targets_scanned += 1
```

La funzione `_get_full_suffix` gestisce le estensioni composte come `.tar.gz` e `.tar.bz2` controllando il suffisso del nome del file prima di ripiegare su `path.suffix.lower()`. Il controllo di esclusione confronta il percorso relativo, il nome del file e i singoli componenti del percorso, quindi un pattern come `__pycache__` corrisponde a qualunque profondità.

## Analisi di rete

### Orchestrazione dello scanner

Il `NetworkScanner` unisce i moduli di rete in una pipeline a più passaggi. La vecchia implementazione decodificava i pacchetti grezzi come UTF-8 ed eseguiva direttamente il rilevamento. La riscrittura tiene conto del protocollo:

```python
def _scan_pcap(self, path, result):
    tracker = FlowTracker()
    dns_detector = DnsExfilDetector(
        entropy_threshold=(
            self._net_config.dns_label_entropy_threshold
        ),
    )
    packet_count = 0

    for packet in read_pcap(
        path,
        max_packets=self._net_config.max_packets,
    ):
        packet_count += 1
        tracker.add_packet(packet)

        if (
            packet.protocol == "udp"
            and (
                packet.src_port == DNS_PORT
                or packet.dst_port == DNS_PORT
            )
        ):
            self._process_dns_packet(
                packet.payload, packet.src_ip,
                packet.dst_ip, path, packet_count,
                dns_detector, result,
            )

        if packet.payload:
            exfil_indicators = detect_base64_payload(
                packet.payload,
                src_ip=packet.src_ip,
                dst_ip=packet.dst_ip,
            )
            for indicator in exfil_indicators:
                finding = _indicator_to_finding(
                    indicator, str(path), packet_count,
                )
                result.findings.append(finding)

    txt_indicators = dns_detector.check_txt_volume()
    for indicator in txt_indicators:
        ...

    self._scan_reassembled_flows(tracker, path, result)
```

Durante il loop sui pacchetti accadono tre cose: ogni pacchetto entra nel `FlowTracker` per il successivo riassemblaggio TCP, i pacchetti UDP sulla porta 53 vengono analizzati come DNS e passati al `DnsExfilDetector`, e ogni payload viene controllato da `detect_base64_payload` alla ricerca di dati codificati in base64/esadecimale. Dopo il loop, vengono controllati i rapporti di volume delle query TXT e i flussi TCP vengono riassemblati per la scansione del contenuto.

La scansione dei flussi riassemblati usa un'estrazione del testo che tiene conto del protocollo:

```python
def _extract_scannable_text(self, stream, protocol):
    if protocol == "http":
        return self._extract_http_text(stream)
    if protocol in ("tls", "ssh"):
        return ""
    try:
        return stream.decode("utf-8", errors="replace")
    except Exception:
        return ""
```

I flussi HTTP vengono analizzati da `parse_http`, che estrae gli URI, gli header sensibili (`cookie`, `authorization`, `set-cookie`) e i body. I flussi TLS e SSH vengono saltati del tutto poiché il contenuto è cifrato e non può essere scansionato. Tutto il resto finisce in un tentativo di decodifica UTF-8.

Gli indicatori di esfiltrazione DNS e i rilevamenti di payload codificati vengono convertiti in oggetti `Finding` tramite `_indicator_to_finding`, che mappa i tipi di indicatore sugli ID delle regole tramite la tabella di lookup `EXFIL_RULE_MAP`. I rilevamenti basati su regex dai flussi riassemblati passano per `match_to_finding` come per gli altri scanner.

### Parsing PCAP

La funzione `read_pcap` in `network/pcap.py` legge i pacchetti usando dpkt e produce struct `PacketInfo`:

```python
def read_pcap(path, max_packets=0):
    with open(path, "rb") as f:
        try:
            pcap = dpkt.pcap.Reader(f)
        except ValueError:
            f.seek(0)
            pcap = dpkt.pcapng.Reader(f)

        count = 0
        for timestamp, buf in pcap:
            if max_packets > 0 and count >= max_packets:
                break

            packet = _parse_ethernet(timestamp, buf)
            if packet is not None:
                yield packet
                count += 1
```

Il fallback try/except gestisce sia PCAP (libpcap) sia PCAPNG (il formato più recente di Wireshark). Si usa dpkt invece di Scapy perché è circa 100 volte più veloce per il parsing di pacchetti in blocco. Scapy costruisce oggetti di protocollo ricchi con livelli di dissezione; dpkt fa un parsing minimo e ti dà i byte grezzi.

### Riassemblaggio dei flussi TCP

Il `FlowTracker` in `network/flow_tracker.py` raggruppa i pacchetti in flussi e riassembla gli stream TCP:

```python
def make_flow_key(packet):
    forward = (
        packet.src_ip, packet.dst_ip,
        packet.src_port, packet.dst_port,
    )
    reverse = (
        packet.dst_ip, packet.src_ip,
        packet.dst_port, packet.src_port,
    )
    return min(forward, reverse)
```

La chiave bidirezionale è la più piccola, in ordine lessicografico, tra la quadrupla di andata e quella di ritorno. Questo significa che i pacchetti `(A->B)` e `(B->A)` finiscono nello stesso flusso. Il metodo `reassemble_stream` ordina i segmenti per numero di sequenza TCP e deduplica le ritrasmissioni:

```python
def reassemble_stream(self, key):
    flow = self._flows.get(key)
    if flow is None:
        return b""

    sorted_segments = sorted(
        flow.segments, key=lambda s: s[0]
    )

    seen_offsets: set[int] = set()
    parts: list[bytes] = []
    for seq, data in sorted_segments:
        if seq not in seen_offsets:
            seen_offsets.add(seq)
            parts.append(data)

    return b"".join(parts)
```

Le ritrasmissioni TCP riusano lo stesso numero di sequenza, quindi la deduplicazione per numero di sequenza evita dati duplicati nello stream riassemblato. È un riassemblaggio semplificato che non gestisce i segmenti sovrapposti (in cui le ritrasmissioni contengono dati diversi), ma copre il caso comune.

### Identificazione del protocollo

La funzione `identify_protocol` in `network/protocols.py` esegue la Deep Packet Inspection tramite il confronto dei prefissi in byte:

```python
def identify_protocol(payload: bytes) -> str:
    if not payload:
        return "unknown"

    if _is_http_request(payload):
        return "http"
    if payload.startswith(HTTP_RESPONSE_PREFIX):
        return "http"
    if (len(payload) > 2
            and payload[:2] == TLS_RECORD_PREFIX):
        return "tls"
    if payload.startswith(SSH_PREFIX):
        return "ssh"
    if payload.startswith(SMTP_BANNER_PREFIX):
        return "smtp"

    return "unknown"
```

Le richieste HTTP vengono identificate controllando se la prima parola prima di uno spazio è un metodo HTTP noto (`GET`, `POST`, `PUT`, ecc.). I record TLS iniziano con `\x16\x03` (ContentType=Handshake + versione principale 3). I banner SSH iniziano con `SSH-`. I saluti dei server SMTP iniziano con `220`.

Questo conta per la DLP perché gli stessi dati sensibili richiedono una gestione diversa a seconda del protocollo di trasporto. Un SSN in un body HTTP può essere letto e segnalato con alta confidenza. Lo stesso SSN in uno stream cifrato TLS non può essere letto, ma puoi segnalare il flusso come "traffico cifrato contenente dati sconosciuti" e correlarlo con altri segnali.

### Rilevamento dell'esfiltrazione DNS

Il `DnsExfilDetector` in `network/exfiltration.py` analizza le query DNS alla ricerca di pattern che suggeriscono un tunneling di dati:

```python
def _check_subdomain_entropy(self, name, src_ip, dst_ip):
    parts = name.split(".")
    if len(parts) < 3:
        return None

    subdomain = ".".join(parts[:-2])
    if not subdomain:
        return None

    entropy = shannon_entropy_str(subdomain)
    if entropy > self._entropy_threshold:
        return ExfilIndicator(
            indicator_type="dns_high_entropy",
            description=(
                f"High subdomain entropy ({entropy:.2f}) "
                f"suggesting DNS tunneling"
            ),
            confidence=min(
                0.95,
                0.50 + (entropy - 3.0) * 0.15,
            ),
            source_ip=src_ip,
            dst_ip=dst_ip,
            evidence=name,
        )
```

I sottodomini legittimi (`www`, `mail`, `api`, `cdn`) hanno un'entropia molto bassa. Una query come `aGVsbG8gd29ybGQ.evil.com` ha un'entropia del sottodominio superiore a 4,0 perché i dati codificati in base64 usano la maggior parte dello spazio dei caratteri alfanumerici. Il detector estrae tutto ciò che precede le ultime due label del dominio (il dominio registrabile), calcola l'entropia di Shannon e segnala le query sopra la soglia.

Il confidence score scala linearmente da 0,50 (a entropia 3,0) a 0,95 (a entropia 6,0). Questo rispecchia l'osservazione che un'entropia più alta significa un rilevamento più sicuro: un'entropia di 4,1 potrebbe essere un hash di CDN, ma un'entropia di 5,5 è quasi certamente un dato codificato.

## Compliance e classificazione della severità

### Mappatura della severità

La funzione `score_to_severity` in `compliance.py` mappa i confidence score sui livelli di severità usando una tabella di soglie:

```python
SEVERITY_SCORE_THRESHOLDS = [
    (0.85, "critical"),
    (0.65, "high"),
    (0.40, "medium"),
    (0.20, "low"),
]

def score_to_severity(score: float) -> Severity:
    for threshold, severity in SEVERITY_SCORE_THRESHOLDS:
        if score >= threshold:
            return severity
    return "low"
```

Le soglie sono calibrate in modo che:
- **Critical** (0,85+): corrispondenze validate dal checksum con keyword di contesto (ad es. SSN vicino a "social security")
- **High** (0,65+): corrispondenze validate dal checksum o contesto forte senza validazione
- **Medium** (0,40+): corrispondenze di pattern senza validazione o contesto forti
- **Low** (0,20+): corrispondenze deboli che potrebbero essere falsi positivi

### Mappatura sui framework

La `RULE_FRAMEWORK_MAP` in `compliance.py` è una tabella di lookup statica:

```python
RULE_FRAMEWORK_MAP = {
    "PII_SSN": ["HIPAA", "CCPA", "GLBA", "GDPR"],
    "PII_DRIVERS_LICENSE_FL": ["CCPA", "HIPAA"],
    "FIN_CREDIT_CARD_VISA": ["PCI_DSS", "GLBA"],
    "FIN_CREDIT_CARD_MC": ["PCI_DSS", "GLBA"],
    "FIN_IBAN": ["GDPR", "GLBA"],
    "HEALTH_NPI": ["HIPAA"],
    "NET_DNS_EXFIL_HIGH_ENTROPY": [],
    ...
}
```

Gli ID delle regole corrispondono alle regole di rilevamento effettive invece di usare categorie generiche. Le regole delle carte di credito sono divise per circuito (`FIN_CREDIT_CARD_VISA`, `FIN_CREDIT_CARD_MC`, `FIN_CREDIT_CARD_AMEX`, `FIN_CREDIT_CARD_DISC`), e ciascuna attiva PCI-DSS e GLBA. Le regole sulle patenti di guida specifiche per stato (`PII_DRIVERS_LICENSE_FL`, `PII_DRIVERS_LICENSE_IL`) si mappano su CCPA e HIPAA insieme al pattern generico CA. Gli indicatori di esfiltrazione di rete (`NET_DNS_EXFIL_*`, `NET_ENCODED_*`) hanno liste di framework vuote, poiché il DNS tunneling è una questione di rilevamento, non un tipo di dato regolamentato.

Gli SSN attivano quattro framework perché sono considerati informazioni sanitarie protette (HIPAA), informazioni personali (CCPA), identificatori finanziari (GLBA) e dati personali (GDPR). Ogni regola ha anche una voce corrispondente in `RULE_REMEDIATION_MAP` con un testo di indicazioni specifico. Le regole sconosciute ripiegano su un default generico.

La mappatura è volutamente conservativa. Un SSN potrebbe attivare SOX se compare in dati di reporting finanziario, ma senza il contesto di business lo scanner non può determinarlo. I framework elencati sono quelli in cui la sola presenza del tipo di dato crea un obbligo di compliance.

## Modulo di scoring condiviso

La funzione `match_to_finding` in `scoring.py` centralizza la conversione da `DetectorMatch` a `Finding`. Tutti e tre gli scanner importano da questo unico punto invece di duplicare la logica di severità/compliance/redaction:

```python
def match_to_finding(
    match: DetectorMatch,
    text: str,
    location: Location,
    redaction_style: RedactionStyle,
) -> Finding:
    severity = score_to_severity(match.score)
    frameworks = get_frameworks_for_rule(match.rule_id)
    if match.compliance_frameworks:
        combined = (
            set(frameworks) | set(match.compliance_frameworks)
        )
        frameworks = sorted(combined)
    remediation = get_remediation_for_rule(match.rule_id)

    snippet = redact(
        text, match.start, match.end,
        style=redaction_style,
    )

    return Finding(
        rule_id=match.rule_id,
        rule_name=match.rule_name,
        severity=severity,
        confidence=match.score,
        location=location,
        redacted_snippet=snippet,
        compliance_frameworks=frameworks,
        remediation=remediation,
    )
```

La funzione concatena in una sola chiamata la classificazione della severità, la ricerca dei framework di compliance, le indicazioni di remediation e la redaction. La logica di unione dei framework gestisce il caso in cui una regola di rilevamento porti con sé la propria lista `compliance_frameworks`: questi vengono uniti ai framework del modulo di compliance, deduplicati e ordinati per un output deterministico.

Ogni scanner la chiama nel proprio loop sulle corrispondenze:

```python
for match in matches:
    if match.score < min_confidence:
        continue

    finding = match_to_finding(
        match, chunk.text, chunk.location,
        self._redaction_style,
    )
    result.findings.append(finding)
```

Aggiungere un nuovo framework di compliance o cambiare le soglie di severità ha effetto su tutti e tre gli scanner in modo uniforme senza toccare il codice degli scanner.

## Redaction

La funzione `redact` in `redaction.py` costruisce uno snippet con il contenuto mascherato:

```python
def redact(text, start, end, style="partial"):
    matched = text[start:end]

    if style == "none":
        return _build_snippet(text, start, end, matched)
    if style == "full":
        return _build_snippet(
            text, start, end, REDACTED_LABEL
        )

    redacted = _partial_redact(matched)
    return _build_snippet(text, start, end, redacted)
```

La funzione `_partial_redact` applica un mascheramento che tiene conto del formato:

```python
def _partial_redact(value):
    stripped = value.replace("-", "").replace(" ", "")

    if len(stripped) >= 9 and stripped.isdigit():
        return MASK_CHAR * (len(value) - 4) + value[-4:]

    if "@" in value:
        local, domain = value.rsplit("@", maxsplit=1)
        masked_local = (
            local[0] + MASK_CHAR * (len(local) - 1)
        )
        return f"{masked_local}@{domain}"

    if len(value) > 8:
        visible = max(4, len(value) // 4)
        return (
            MASK_CHAR * (len(value) - visible)
            + value[-visible:]
        )

    return MASK_CHAR * len(value)
```

Per le sequenze di cifre (SSN, carte di credito), preserva le ultime 4 cifre: `***-**-6789`. Per le email, mantiene il primo carattere e il dominio: `j****@example.com`. Per le altre stringhe (API key, token), mostra l'ultimo 25%. I valori brevi sotto gli 8 caratteri vengono mascherati completamente.

La funzione `_build_snippet` aggiunge 20 caratteri di contesto su ciascun lato e antepone/accoda `...` quando il contesto è troncato. Questo dà agli analisti abbastanza testo circostante per capire vicino a cosa si trovasse il dato senza esporre l'intero contenuto del documento.

## Integrazione della CLI

### Propagazione delle opzioni globali

Il callback di Typer memorizza le opzioni globali nel dict di contesto di Click:

```python
@app.callback()
def main(ctx: typer.Context, config: ..., verbose: ..., version: ...):
    ctx.ensure_object(dict)
    ctx.obj["config_path"] = config
    ctx.obj["verbose"] = verbose
```

I sottocomandi le recuperano tramite `ctx.ensure_object(dict)`:

```python
def _run_scan(ctx, scan_type, target, output_format, output_file):
    obj: dict[str, Any] = ctx.ensure_object(dict)
    config_path = obj.get("config_path", "")
    verbose = obj.get("verbose", False)
```

Questo pattern permette a `dlp-scan -v -c custom.yml file ./data` di propagare il flag verbose e il percorso della configurazione al comando di scansione dei file senza duplicare quelle opzioni su ogni sottocomando.

### Strategia di logging

Il livello di logging si adatta al formato di output:

```python
if verbose:
    configure_logging(level="DEBUG")
elif output_format == "console":
    configure_logging(level="INFO")
else:
    configure_logging(level="WARNING")
```

Quando l'output è leggibile dalle macchine (JSON, SARIF, CSV), il logging viene impostato su WARNING così che i messaggi di structlog scritti su stderr non contaminino stdout. Questo impedisce che `dlp-scan file ./data -f json | jq` si rompa a causa di righe di log mescolate all'output JSON. Per l'output in console, il logging a livello INFO fornisce un riscontro sull'avanzamento. La modalità verbose abilita DEBUG per il troubleshooting.

### Conversione dei report

Il comando `report convert` legge un risultato di scansione JSON e lo rigenera in un altro formato:

```python
@report_app.command("convert")
def convert(input_file, output_format="sarif", output_file=""):
    raw = path.read_bytes()
    data = orjson.loads(raw)
    result = _rebuild_result(data)

    config = ScanConfig()
    engine = ScanEngine(config)

    output = engine.generate_report(result, fmt)
    ...
```

La funzione `_rebuild_result` deserializza la struttura JSON di nuovo in oggetti `ScanResult`, `Finding` e `Location`. Legge dalla sezione `scan_metadata` i campi a livello di scansione e itera su `findings` per ricostruire ogni `Finding` con la sua `Location`. Questo è necessario perché `orjson.loads` produce semplici dict, ma i reporter si aspettano istanze tipizzate di dataclass.

## Strategia di test

### Property-based testing

Il progetto usa Hypothesis per il property-based testing delle regole di rilevamento. Invece di testare pochi input noti, Hypothesis genera stringhe casuali vincolate dai formati delle regole e verifica che la pipeline di rilevamento le gestisca correttamente.

Per i validatori: Hypothesis genera sequenze casuali di cifre e verifica che `luhn_check`, `iban_check` e `nhs_check` restituiscano True solo per gli input che soddisfano le proprietà matematiche (divisibilità per 10, mod 97 = 1, corrispondenza della cifra di controllo mod 11).

Per il boost di contesto: Hypothesis genera testo casuale con keyword incorporate a distanze variabili e verifica che il boost sia sempre compreso tra 0 e `CONTEXT_BOOST_MAX`, e che le keyword più vicine producano boost più alti.

### Eseguire i test

```bash
uv run pytest -m unit              # fast unit tests
uv run pytest -m integration       # tests with file I/O
uv run pytest --cov=src            # coverage report
```

La test suite usa dei marker (`unit`, `integration`, `slow`) per separare i test veloci da quelli che richiedono un accesso reale al filesystem. Il `conftest.py` fornisce fixture condivise per directory temporanee, configurazioni di esempio e file di dati di test.

## Dipendenze

- **typer**: framework CLI con dichiarazione degli argomenti tramite type hint. Lo stile `Annotated` evita l'accumulo di decorator.
- **rich**: tabelle nel terminale con colori. Usato da `ConsoleReporter` per l'output colorato per severità.
- **structlog**: logging strutturato con integrazione della stdlib. Rendering JSON o console in base alla configurazione.
- **pydantic**: validazione della configurazione. Intercetta i valori YAML non validi prima che la scansione inizi.
- **orjson**: serializzazione JSON veloce. Da 3 a 10 volte più veloce del json della stdlib per liste di finding grandi.
- **ruamel.yaml**: parser YAML che gestisce la specifica 1.2 e preserva i commenti.
- **dpkt**: parsing PCAP. Circa 100 volte più veloce di Scapy per l'iterazione di pacchetti in blocco.
- **pymupdf**: estrazione del testo dai PDF con preservazione del layout.
- **python-docx/openpyxl/xlrd**: estrazione dai formati Office.
- **defusedxml/lxml**: parsing XML sicuro (defusedxml blocca gli attacchi XXE).
- **pyarrow/fastavro**: estrazione da formati colonnari (Parquet, Avro).
- **asyncpg/aiomysql/pymongo/aiosqlite**: driver di database asincroni.

## Prossimi passi

Hai visto come funziona il codice. Ora:
1. Prova le sfide in [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee di estensione
2. Modifica una regola di rilevamento ed esegui i test per vedere come cambia lo scoring
3. Scansiona i tuoi file con `dlp-scan file ./your-directory` e ispeziona l'output
