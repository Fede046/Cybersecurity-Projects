# Panoramica

## Cosa fa questo progetto

b64tool è una CLI di encoding/decoding multi-formato che gestisce Base64, Base64URL, Base32, hex e URL encoding. La funzionalità di punta è il recursive layer peeling, che rileva e decodifica automaticamente più layer di encoding sovrapposti, la stessa tecnica che gli attaccanti usano per offuscare i payload di malware e aggirare i web application firewall.

Gli dai un blob codificato. Capisce di che formato si tratta, lo decodifica, controlla se il risultato è a sua volta codificato e continua finché non arriva ai dati originali. Pensalo come una cipolla. Ogni strato è un encoding diverso, e il tool li sbuccia uno alla volta.

## Perché esiste

Nel lavoro di sicurezza reale ti imbatti continuamente in dati codificati. I token JWT sono in base64. I certificati sono in base64. Gli hex dump compaiono nelle packet capture e nell'analisi di malware. L'URL encoding appare in ogni richiesta web. E quando gli attaccanti vogliono nascondere dei payload, non si limitano a codificarli una volta. Sovrappongono gli encoding: codificano il payload in base64, lo codificano in hex, poi applicano l'URL encoding all'hex. Questo tool gestisce tutto questo.

## Cosa imparerai

- Come funzionano davvero Base64, Base32, hex e URL encoding a livello di byte
- Perché l'encoding non è cifratura (e perché confonderli causa vulnerabilità reali)
- Il riconoscimento di pattern per identificare i formati di encoding
- Come gli attaccanti sovrappongono gli encoding per l'offuscamento (con esempi reali dal malware DARKGATE)
- Costruire una CLI Python pulita e tipizzata con typer e rich
- Algoritmi di rilevamento basati sulla confidenza

## Prerequisiti

- Python 3.14+
- Il package manager `uv`
- Dimestichezza di base con la riga di comando
- Comprensione della differenza tra byte e stringhe (utile, non obbligatoria)

## Avvio rapido

```bash
cd PROJECTS/beginner/base64-tool
uv sync
uv run b64tool --help
```

### Prova le basi

```bash
uv run b64tool encode "Hello World"
# Output: SGVsbG8gV29ybGQ=

uv run b64tool decode "SGVsbG8gV29ybGQ="
# Output: Hello World

uv run b64tool detect "SGVsbG8gV29ybGQ="
# Shows: base64, 95% confidence
```

### Prova l'encoding multi-layer

```bash
uv run b64tool chain "alert('xss')" --steps base64,hex
# Produces a hex-encoded base64 string

uv run b64tool peel "5957786c636e516f4a33687a63796370"
# Peels: hex → base64 → alert('xss')
```

### Supporto per le pipe

```bash
echo "SGVsbG8=" | uv run b64tool decode
cat encoded_payload.txt | uv run b64tool peel
```

## Struttura del progetto

```
base64-tool/
├── src/base64_tool/
│   ├── cli.py          # Typer commands (encode, decode, detect, peel, chain)
│   ├── constants.py    # Enums, thresholds, character sets
│   ├── encoders.py     # Pure encode/decode functions + registry
│   ├── detector.py     # Format detection with confidence scoring
│   ├── peeler.py       # Recursive multi-layer decoding
│   ├── formatter.py    # Rich terminal output
│   └── utils.py        # Input resolution, text helpers
├── tests/              # 78 tests across all modules
├── pyproject.toml      # Python 3.14, typer, rich
└── Justfile            # Task runner shortcuts
```

## Prossimi passi

- [01-CONCEPTS.md](./01-CONCEPTS.it.md) - Come funziona l'encoding, CVE reali, encoding vs cifratura
- [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) - Design del sistema, relazioni tra moduli, flusso dei dati
- [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) - Analisi guidata del codice con riferimenti ai file
- [04-CHALLENGES.md](./04-CHALLENGES.it.md) - Idee di estensione da facili a per esperti
