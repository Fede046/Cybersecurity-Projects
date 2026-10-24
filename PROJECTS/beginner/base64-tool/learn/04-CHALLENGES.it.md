# Sfide

## Facili

### 1. Aggiungi l'encoding ROT13

ROT13 sposta ogni lettera di 13 posizioni. È un cifrario di Cesare con una chiave fissa che è l'inverso di se stessa (encoding e decoding sono la stessa operazione).

**Cosa fare:**
- Aggiungi `ROT13 = "rot13"` a `EncodingFormat` in `constants.py`
- Scrivi `encode_rot13(data: bytes) -> str` e `decode_rot13(data: str) -> bytes` in `encoders.py`
- Aggiungi la voce a `ENCODER_REGISTRY`
- Scrivi uno scorer `_score_rot13` in `detector.py` (suggerimento: prova a decodificare e controlla se il risultato contiene più parole inglesi riconoscibili rispetto all'input)
- Aggiungi i test

**Perché è importante:** ROT13 è banale da implementare, ma lo scorer di rilevamento è complicato. Come distingui un testo ROT13 da un normale testo ASCII? Entrambi usano lo stesso set di caratteri. Ti servirà un'euristica a livello di lingua (analisi della frequenza delle lettere, corrispondenza con parole comuni).

### 2. Aggiungi una modalità di output JSON

Attualmente l'output è costituito da pannelli Rich oppure da testo grezzo. Aggiungi un flag `--json` che produca JSON strutturato.

**Cosa fare:**
- Aggiungi un flag `--json` / `-j` ai comandi `detect` e `peel` in `cli.py`
- Quando è attivo, serializza `DetectionResult` e `PeelResult` in JSON e scrivili su stdout
- Salta completamente la formattazione Rich in modalità JSON

**Output di esempio:**
```json
{
  "results": [
    {
      "format": "base64",
      "confidence": 0.95,
      "decoded_preview": "Hello World"
    }
  ]
}
```

**Perché è importante:** un output leggibile dalle macchine è essenziale per integrare tool CLI in pipeline automatizzate. I workflow di security automation (SOAR, correlazione SIEM) consumano JSON, non testo colorato da terminale.

### 3. Aggiungi il flag di file `--output`

Aggiungi un flag `-o` / `--output` per scrivere i risultati direttamente su un file invece che su stdout.

**Cosa fare:**
- Aggiungi l'opzione ai comandi `encode`, `decode` e `peel`
- Per encode/decode, scrivi il risultato grezzo nel file
- Per peel, scrivi l'output finale
- Stampa un messaggio di conferma su stderr

**Perché è importante:** quando decodifichi payload binari (campioni di malware, immagini), ti servono i byte esatti scritti su disco, non un'anteprima nel terminale.

---

## Medie

### 4. Aggiungi l'encoding ASCII85 (Base85)

ASCII85 codifica 4 byte in 5 caratteri usando 85 caratteri ASCII stampabili. È usato nei file PDF e nelle patch binarie di Git.

**Cosa fare:**
- Aggiungi `ASCII85 = "ascii85"` a `EncodingFormat`
- Usa `base64.a85encode()` e `base64.a85decode()` della libreria standard di Python
- Scrivi uno scorer per il rilevamento (i dati ASCII85 hanno una distribuzione dei caratteri caratteristica, e spesso includono caratteri come `!`, `@`, `#` che non compaiono in altri encoding)
- Gestisci il wrapper `<~...~>` usato dalla variante di Adobe

**Perché è importante:** incontrerai ASCII85 nell'analisi di malware PDF. I PDF malevoli incorporano contenuto eseguibile usando stream ASCII85.

### 5. Comando decode-chain (l'inverso di chain)

Il comando `chain` codifica attraverso più layer. Costruisci un comando `decode-chain` che prenda una specifica di catena e decodifichi in ordine inverso.

**Cosa fare:**
- Aggiungi un comando `decode-chain` a `cli.py`
- Accetta `--steps` come il comando chain
- Applica i decoder in ordine inverso: `--steps base64,hex` decodifica prima l'hex, poi il base64
- Mostra ogni passo intermedio

**Esempio:**
```bash
b64tool decode-chain "4147567362473873" --steps base64,hex
# Step 1: hex decode → "AGVsbG8s"
# Step 2: base64 decode → "Hello,"
```

**Perché è importante:** quando conosci l'ordine degli encoding grazie all'analisi (o all'output di `peel`), puoi verificarlo specificando esplicitamente la catena di decoding. È utile per validare i risultati di peel.

### 6. Visualizzazione hex dump

Aggiungi un comando `hexdump` che mostri i dati binari decodificati nel formato hex dump tradizionale (offset, byte in hex, rappresentazione ASCII).

**Cosa fare:**
- Aggiungi un comando `hexdump` che prenda input codificato e un flag di formato
- Decodifica l'input, poi visualizzalo in formato hex dump:
  ```
  00000000  48 65 6c 6c 6f 20 57 6f  72 6c 64 21 0a 00 ff fe  |Hello World!....|
  ```
- 16 byte per riga, offset a sinistra, ASCII a destra (i non stampabili come `.`)

**Perché è importante:** l'hex dump è la vista standard per l'analisi dei binari. I malware analyst vivono dentro le viste hex dump. Combinare decode + hexdump in un unico tool evita di dover passare in pipe attraverso `xxd` o `hexdump`.

---

## Difficili

### 7. Supporto per alfabeti Base64 personalizzati

Il malware DARKGATE usa alfabeti base64 personalizzati per eludere i tool di rilevamento che capiscono solo RFC 4648. Costruisci il supporto per encoding/decoding con alfabeti arbitrari.

**Cosa fare:**
- Aggiungi un'opzione `--alphabet` ai comandi encode e decode
- Accetta una stringa di 64 caratteri (o 65 con il carattere di padding)
- Implementa l'encode/decode con alfabeto personalizzato senza appoggiarti al modulo `base64` di Python (non supporta direttamente alfabeti personalizzati)
- La logica di encoding: prendi 3 byte, dividili in quattro valori da 6 bit, mappa ciascuno sull'alfabeto personalizzato
- Aggiungi un'opzione `--alphabet-file` per leggere l'alfabeto da un file

**Esempio:**
```bash
b64tool encode "secret" --format base64 --alphabet "ZYXWVUTSRQPONMLKJIHGFEDCBAzyxwvutsrqponmlkjihgfedcba9876543210+/"
```

**Perché è importante:** i malware reali lo usano. Costruirlo ti insegna la vera matematica dell'encoding (manipolazione di bit, non solo chiamate a funzioni di libreria). I team di threat intelligence hanno bisogno di tool che gestiscano alfabeti non standard.

### 8. Analisi della frequenza degli encoding

Aggiungi un comando `analyze` che prenda un file di campioni codificati e produca statistiche.

**Cosa fare:**
- Leggi un file in cui ogni riga è un campione codificato
- Esegui il rilevamento su ogni riga
- Riporta: distribuzione dei formati, confidenza media per formato, rilevamenti falliti
- Segnala i campioni che corrispondono a più formati (rilevamenti ambigui)
- Produci l'output come tabella riassuntiva e, opzionalmente, come JSON

**Perché è importante:** gli analisti SOC ricevono spesso indicatori in blocco (IOC) e devono classificarli rapidamente. Questa lista di stringhe è composta per lo più da token base64? Hash in hex? Un misto? L'analisi in blocco risponde a questa domanda.

### 9. Modalità streaming per file di grandi dimensioni

Attualmente il tool legge interi file in memoria. Aggiungi il supporto allo streaming per l'elaborazione di file di grandi dimensioni.

**Cosa fare:**
- Per l'encoding base64: leggi 3 byte alla volta (che producono 4 caratteri di output), oppure usa un buffer in multipli di 3
- Per il decoding base64: leggi 4 caratteri alla volta
- Per l'hex: leggi 1 byte / 2 caratteri alla volta
- Implementa con dei generatori per mantenere costante l'uso di memoria
- Aggiungi un flag `--stream` oppure rileva automaticamente in base alla dimensione del file

**Perché è importante:** mappare in memoria un file da 2 GB per codificarlo in base64 manderà in crash la maggior parte dei sistemi. Lo streaming elabora file di qualsiasi dimensione con memoria costante. È così che funzionano i tool di produzione come `base64` (coreutils).

---

## Per esperti

### 10. Rilevamento del formato binario nell'output decodificato

Dopo il decoding, rileva automaticamente cosa sono i dati binari decodificati (magic byte del tipo di file).

**Cosa fare:**
- Dopo decode o peel, confronta i primi byte con i magic number noti:
  - `\x89PNG\r\n\x1a\n` → immagine PNG
  - `PK\x03\x04` → archivio ZIP/DOCX/JAR
  - `\x7fELF` → eseguibile ELF (binario Linux)
  - `MZ` → eseguibile PE (binario Windows)
  - `%PDF` → documento PDF
  - `GIF87a` / `GIF89a` → immagine GIF
- Mostra il tipo di file rilevato accanto all'output decodificato
- Opzionalmente salva in un file con l'estensione corretta

**Perché è importante:** i payload dei malware sono spesso eseguibili codificati in base64. Sapere che un blob decodificato è un eseguibile PE piuttosto che un'immagine PNG cambia completamente il tuo approccio all'analisi. Il loader di DARKGATE codifica il proprio payload di secondo stadio in base64 personalizzato — l'output decodificato è una DLL.

### 11. Analisi dell'entropia

Aggiungi un comando `entropy` che calcoli l'entropia di Shannon dei dati prima e dopo il decoding.

**Cosa fare:**
- Calcola l'entropia di Shannon: `H = -Σ p(x) log2(p(x))` per ogni valore di byte
- Riporta l'entropia su una scala 0–8 (8 bit per byte = entropia massima)
- Confronta l'entropia prima e dopo il decoding
- Segnala output decodificato ad alta entropia (probabilmente cifrato o compresso, non solo codificato)

**Guida all'interpretazione:**
- 0–1: altamente strutturato/ripetitivo (come una sequenza di soli zeri)
- 3–5: testo in inglese, codice sorgente
- 6–7: dati compressi, alcuni formati binari
- 7.5–8: dati cifrati, byte casuali

**Perché è importante:** se rimuovi 3 layer di encoding e l'output ha entropia 7.9, è quasi certamente cifrato — un'ulteriore analisi dell'encoding non servirà. Se l'entropia è 4.5, probabilmente è testo leggibile. L'entropia è il primo controllo di triage nell'analisi dei malware.

### 12. Integrazione con le recipe di CyberChef

CyberChef (il tool open source di analisi dati di GCHQ) usa le "recipe" — descrizioni JSON di catene di operazioni. Costruisci la compatibilità di import/export.

**Cosa fare:**
- Esegui il parsing del formato JSON delle recipe di CyberChef
- Mappa le operazioni di CyberChef sui formati di b64tool (ad es. "To Base64" → encode base64)
- Esegui una recipe come comando chain
- Esporta i risultati di peel come recipe di CyberChef (operazioni inverse)

**Esempio:**
```bash
b64tool recipe --import recipe.json "input data"
b64tool peel "encoded_data" --export-recipe result.json
```

**Perché è importante:** CyberChef è onnipresente nelle competizioni CTF e nei security operations center. Poter scambiare recipe tra tool dimostra una reale interoperabilità. Significa anche che il tuo tool può sfruttare le migliaia di recipe di CyberChef già esistenti condivise dalla community.
