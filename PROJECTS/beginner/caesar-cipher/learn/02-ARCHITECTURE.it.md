# Architettura del sistema

Questo documento analizza come è progettato il sistema e perché sono state prese determinate decisioni architetturali.

## Architettura di alto livello
```
┌─────────────────┐
│   CLI Layer     │  (main.py)
│  - encrypt cmd  │  Typer commands, Rich output
│  - decrypt cmd  │  
│  - crack cmd    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Cipher Layer   │  (cipher.py)
│  - CaesarCipher │  Core algorithm
│  - shift logic  │  Encryption/decryption
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Analysis Layer  │  (analyzer.py)
│ - chi-squared   │  Statistical scoring
│ - rank results  │  Frequency comparison
└─────────────────┘
         │
         ▼
┌─────────────────┐
│  Utils Layer    │  (utils.py, constants.py)
│  - file I/O     │  Support functions
│  - validation   │  Reference data
└─────────────────┘
```

### Scomposizione dei componenti

**CLI Layer (main.py)**
- Scopo: comandi rivolti all'utente che fanno il parsing degli argomenti e chiamano le funzioni principali
- Responsabilità: validazione dell'input, gestione dei file, output formattato con tabelle Rich
- Interfacce: espone tre comandi tramite Typer (encrypt, decrypt, crack)

**Cipher Layer (cipher.py)**
- Scopo: implementa l'algoritmo vero e proprio del cifrario di Cesare
- Responsabilità: shift dei caratteri, validazione della chiave, generazione brute force
- Interfacce: classe `CaesarCipher` con i metodi `encrypt()`, `decrypt()` e il metodo statico `crack()`

**Analysis Layer (analyzer.py)**
- Scopo: analisi statistica per identificare le decifrature corrette
- Responsabilità: calcolo del chi-quadrato, classificazione dei candidati in base alla frequenza dell'inglese
- Interfacce: classe `FrequencyAnalyzer` che assegna punteggi al testo e lo classifica

**Utils Layer (utils.py, constants.py)**
- Scopo: funzionalità condivise e dati di riferimento
- Responsabilità: lettura da file/stdin, scrittura dell'output, memorizzazione delle frequenze delle lettere inglesi
- Interfacce: funzioni standalone e costanti usate dagli altri layer

## Flusso dei dati

### Flusso di cifratura

Analisi passo per passo della cifratura di un testo:
```
1. User Input → CLI Parser (main.py:34-56)
   Validates key is 0-25, reads text from arg/file/stdin

2. CLI → CaesarCipher (cipher.py:13-28)
   Creates cipher instance with validated key

3. CaesarCipher → Encryption Loop (cipher.py:43-46)
   Shifts each character, preserves non-letters
   
4. Encrypted Text → Output (main.py:56-60)
   Writes to file or prints with Rich formatting
```

Esempio con riferimenti al codice:
```
1. User runs: caesar-cipher encrypt "HELLO" --key 3
   main.py:34 → validates key with utils.validate_key()
   main.py:47 → reads input via read_input()

2. main.py:48 → Creates CaesarCipher(key=3)
   cipher.py:16 → Stores key % 26 to handle wrapping

3. main.py:49 → Calls cipher.encrypt("HELLO")
   cipher.py:43-46 → Iterates each char with _shift_char()
   cipher.py:31-38 → Shifts H→K, E→H, L→O, L→O, O→R

4. main.py:52 → Outputs "KHOOR" via console.print()
```

### Flusso di cracking

Più complesso: prova tutte le chiavi e le classifica in base alla frequenza.
```
1. User Input → CLI (main.py:108-128)
   Reads ciphertext, gets options (--top N, --all)

2. CLI → Brute Force (cipher.py:53-60)
   Generates all 26 possible decryptions

3. Candidates → Frequency Analysis (analyzer.py:53-60)
   Scores each with chi-squared test

4. Ranked Results → Table Output (main.py:135-148)
   Displays top matches in Rich table
```

## Design pattern

### Strategy Pattern (implicito)

**Che cos'è:**
Separare l'algoritmo (operazioni del cipher) dalla sua applicazione (comandi CLI).

**Dove lo usiamo:**
La classe `CaesarCipher` in `cipher.py` è indipendente da come viene invocata. Potresti usarla da una web API, da una GUI o da una CLI senza modificare il codice del cipher.

**Perché l'abbiamo scelto:**
La separazione delle responsabilità rende i test più semplici. La logica del cipher non ha alcuna dipendenza da Typer o Rich. Puoi testare `cipher.py` senza doverti occupare del parsing degli argomenti da riga di comando.

**Trade-off:**
- Pro: interfacce pulite, facile da testare, componenti riutilizzabili
- Contro: più file rispetto a un singolo script, leggermente più complesso per un progetto semplice

### Factory Pattern (metodi statici)

**Che cos'è:**
Usare `@staticmethod` per creare varianti senza istanziazione.

**Dove lo usiamo:**
```python
# cipher.py:52-60
@staticmethod
def crack(ciphertext: str) -> list[tuple[int, str]]:
    results = []
    for shift in range(ALPHABET_SIZE):
        cipher = CaesarCipher(key=shift)
        decrypted = cipher.decrypt(ciphertext)
        results.append((shift, decrypted))
    return results
```

**Perché l'abbiamo scelto:**
Il metodo `crack()` non appartiene a nessun valore di chiave in particolare. Genera tutte le istanze possibili. Renderlo statico rende l'API più chiara: `CaesarCipher.crack()` si legge come "prova tutte le chiavi di Cesare" senza bisogno di un'istanza.

## Separazione dei layer
```
┌────────────────────────────────────┐
│    CLI Layer (main.py)             │
│    - User interaction              │
│    - Does not do crypto            │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    Business Logic (cipher.py)      │
│    - Core algorithm                │
│    - Does not know about CLI       │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│    Support (utils.py)              │
│    - Generic helpers               │
│    - No domain logic               │
└────────────────────────────────────┘
```

### Perché i layer?

Impediscono che le dipendenze diventino circolari. La CLI può importare il cipher, ma il cipher non importa la CLI. Questo significa che:
- Puoi testare il cipher senza fare il mock di Typer
- Potresti usare il cipher in un'interfaccia diversa (web, GUI) senza modifiche
- Le modifiche alla formattazione dell'output non influiscono sulla correttezza crittografica

### Cosa sta dove

**CLI Layer (main.py):**
- File: `main.py`
- Import: può importare da cipher, analyzer, utils
- Vietato: nessuna logica crittografica, nessuna manipolazione diretta dell'alfabeto

**Business Layer (cipher.py, analyzer.py):**
- File: `cipher.py`, `analyzer.py`
- Import: può importare constants, utils. Non può importare main.
- Vietato: nessun parsing della riga di comando, nessuna formattazione Rich

**Support Layer (utils.py, constants.py):**
- File: `utils.py`, `constants.py`
- Import: solo la libreria standard
- Vietato: nessuna logica di dominio, resta generico

## Modelli di dati

### CaesarCipher
```python
# cipher.py:13-28
class CaesarCipher:
    def __init__(self, key: int, alphabet: str | None = None) -> None:
        if not -25 <= key <= 26:
            raise ValueError("Key must be between -25 and 26")
        
        self.key = key % ALPHABET_SIZE
        self.alphabet = alphabet or (UPPERCASE_LETTERS + LOWERCASE_LETTERS)
```

**Campi spiegati:**
- `key`: l'entità dello shift, normalizzata a 0-25 tramite modulo. Memorizzarla in questo modo significa che la cifratura non deve mai più gestire il ritorno all'inizio dell'alfabeto.
- `alphabet`: normalmente solo A-Z + a-z, ma configurabile per alfabeti personalizzati. Non usato nella CLI, ma estendibile.

**Relazioni:**
- Usa le costanti di `constants.py` (ALPHABET_SIZE, insiemi di lettere)
- Usato da tutti e tre i comandi CLI
- Non ha dipendenze dall'analyzer (relazione unidirezionale)

### FrequencyAnalyzer
```python
# analyzer.py:11-16
class FrequencyAnalyzer:
    def __init__(self) -> None:
        self.reference_frequencies = ENGLISH_LETTER_FREQUENCIES
```

**Campi spiegati:**
- `reference_frequencies`: dizionario che mappa 'A'-'Z' alle rispettive percentuali attese nel testo inglese. Caricato da `constants.py:17-43`.

**Relazioni:**
- Usato solo dal comando `crack`
- Opera sull'output di `CaesarCipher.crack()`
- Nessuna dipendenza dal cipher stesso

## Architettura di sicurezza

### Threat model

Da cosa ci stiamo proteggendo:
1. **Niente** - Questo è un tool didattico. Il cipher è intenzionalmente debole.
2. **Validazione dell'input** - Previene crash dovuti a chiavi errate o input mancante
3. **File injection** - Usa oggetti Path e una codifica esplicita per evitare problemi

Da cosa NON ci stiamo proteggendo (fuori ambito):
- Crittoanalisi - Il cipher è fatto per essere violato
- Attacchi side-channel - Questo è Python, i tempi non sono costanti
- Recupero della chiave - Il key space è banalmente piccolo

### Livelli di difesa

Questo progetto non ha difese di sicurezza perché insegna la crittoanalisi, non costruisce crittografia sicura. Ma ha la validazione dell'input:
```
Layer 1: Key validation (utils.py:36-40, cipher.py:19-22)
    ↓
Layer 2: Input source validation (utils.py:11-24)
    ↓
Layer 3: File encoding (utils.py:17, 32)
```

La validazione garantisce che il programma non vada in crash, ma non c'è alcun confine di sicurezza. Non usare il cifrario di Cesare per segreti reali.

## Strategia di memorizzazione

### Nessuna memorizzazione persistente

Questo tool è stateless. Tutto avviene in memoria. L'input arriva da argomenti/file/stdin, l'output va su stdout/file e nulla viene salvato.

**Perché questa scelta:**
Semplicità. Non c'è bisogno di tenere traccia della cronologia o di salvare lo stato. Ogni comando è indipendente.

## Configurazione

### Variabili d'ambiente

Nessuna. Il progetto usa esclusivamente argomenti da riga di comando.

### Strategia di configurazione

**Sviluppo:**
```bash
pip install -e .  # Editable install
```

Tutta la configurazione è in `pyproject.toml`. Dipendenze, impostazioni del linter, configurazione dei test: tutto si trova lì.

**Produzione:**
```bash
pip install .  # Regular install
```

Stessa configurazione. Di questo tool non viene fatto il deploy in produzione perché è uno strumento didattico, ma se lo fosse, la configurazione non cambierebbe.

## Considerazioni sulle prestazioni

### Colli di bottiglia

Dove questo sistema rallenta sotto carico:
1. **Analisi delle frequenze su file enormi** - Il calcolo del chi-quadrato è O(n), dove n è la lunghezza del testo. Per file di più MB, questo si accumula quando viene eseguito 26 volte.
2. **Rendering delle tabelle Rich** - Stampare 26 righe di output è più lento del testo semplice.

Nessuno dei due conta nella pratica. Il cipher in sé è così veloce che l'I/O domina.

### Ottimizzazioni

Cosa abbiamo fatto per renderlo più veloce:
- **List comprehension in encrypt()**: usare `"".join(self._shift_char(char, self.key) for char in plaintext)` in `cipher.py:46` invece di costruire una lista e poi unirla è più efficiente in termini di memoria.
- **Return anticipato nel chi-quadrato**: se non ci sono lettere, restituisce subito infinito (analyzer.py:33) invece di tentare il calcolo su dati vuoti.

### Scalabilità

**Scalabilità verticale:**
Non si applica. Python single-threaded che elabora testo. Più CPU non aiuta.

**Scalabilità orizzontale:**
Potresti parallelizzare il comando crack per provare tutti i 26 shift in parallelo, ma è già istantaneo. Non vale la complessità.

## Decisioni di design

### Decisione 1: classi Cipher e Analyzer separate

**Cosa abbiamo scelto:**
Tenere la logica di cifratura in `CaesarCipher` e l'analisi statistica in `FrequencyAnalyzer`.

**Alternative considerate:**
- Mettere tutto in un'unica classe - Scartata perché mescolare crittografia e crittoanalisi nello stesso oggetto è concettualmente sbagliato
- Rendere l'analyzer un insieme di funzioni invece di una classe - Scartata perché le frequenze di riferimento sono stato condiviso

**Trade-off:**
Otteniamo una separazione più pulita e una migliore testabilità. Il costo è avere due import invece di uno quando vuoi violare i messaggi.

### Decisione 2: CLI con Typer invece di argparse

**Cosa abbiamo scelto:**
Usare Typer per la generazione automatica dell'help e i type hint.

**Alternative considerate:**
- argparse (stdlib) - Più verboso, niente type hint
- click - Simile a Typer ma senza la magia dei type hint

**Trade-off:**
Typer fornisce codice pulito al costo di una dipendenza in più. Per un progetto di apprendimento, la migliore leggibilità del codice ne vale la pena.

### Decisione 3: preservare maiuscole/minuscole e caratteri non alfabetici

**Cosa abbiamo scelto:**
Cifrare solo A-Z e a-z, lasciando invariati spazi/punteggiatura/numeri.

**Alternative considerate:**
- Convertire tutto in maiuscolo - Si perde informazione, rende l'output più brutto
- Cifrare anche gli spazi - Il Cesare storico non lo faceva, meno autentico

**Trade-off:**
Preservare maiuscole e minuscole rende l'output più leggibile ma complica leggermente la logica di shift (bisogna controllare separatamente maiuscole e minuscole).

## Architettura di deployment

Questo è un tool CLI locale, non un servizio deployato. L'installazione avviene tramite pip:
```bash
pip install caesar-salad-cipher
```

L'entry point in `pyproject.toml:31-32` rende disponibile il comando:
```toml
[project.scripts]
caesar-cipher = "caesar_cipher.main:app"
```

Dopo l'installazione, `caesar-cipher` è nel tuo PATH e chiama `main.py:app()`.

## Strategia di gestione degli errori

### Tipi di errore

1. **Chiave non valida** - Chiave fuori dall'intervallo da -25 a 26
   - Sollevato da `cipher.py:20-22` e `utils.py:36-40`
   - Catturato in `main.py:59, 97, 152` e stampato come errore

2. **Input mancante** - Nessun testo fornito
   - Sollevato da `utils.py:23` se tutte le sorgenti (argomento, file, stdin) sono vuote
   - Catturato nei comandi principali

3. **File non trovato** - Il file di input non esiste
   - Sollevato da `Path.read_text()` in `utils.py:17`
   - Catturato genericamente come OSError

### Meccanismi di ripristino

Non c'è alcun ripristino automatico. Il tool termina con codice 1 in caso di errore:
```python
# main.py:59-60
except (ValueError, OSError) as e:
    console.print(f"[red]Error:[/red] {e}")
    raise typer.Exit(code=1) from None
```

**Perché uscire invece di riprovare:**
È un tool CLI. Se l'utente ha fornito una chiave errata, deve correggerla ed eseguire di nuovo. Non ha senso restare in esecuzione.

## Estendibilità

### Dove aggiungere funzionalità

Vuoi aggiungere il supporto per i numeri? Ecco dove va fatto:

1. Modifica `constants.py` per includere '0'-'9' nell'alfabeto
2. Aggiorna _shift_char() in `cipher.py:31-40` per gestire le cifre
3. Adatta i test in `test_cipher.py` per verificare lo shift delle cifre

Vuoi aggiungere il cifrario di Vigenère?

1. Crea `vigenere.py` con una struttura di classe simile
2. Aggiungi il comando `vigenere` in `main.py`
3. Riutilizza `utils.py` per l'I/O, ma l'analisi delle frequenze non funzionerà (Vigenère è polialfabetico)

## Limitazioni

Limitazioni architetturali attuali:
1. **Funziona solo con l'alfabeto latino** - Nessun supporto per scritture cirilliche, arabe o ideografiche. Risolvere questo richiederebbe costanti multi-alfabeto e tabelle di frequenza diverse.
2. **Nessuna derivazione della chiave** - La chiave è letteralmente solo un numero. Non si possono usare password. Servirebbe una KDF (ma è eccessivo per Cesare).
3. **Single-threaded** - Non può sfruttare più core. Non vale la pena sistemarlo, visto che crack() gira comunque in meno di un millisecondo.

Questi non sono bug, sono trade-off consapevoli. Il progetto serve per imparare la crittografia classica, non per costruire tool di produzione.

## Confronto con sistemi simili

### Tool ROT13 online

In cosa siamo diversi:
- I tool ROT13 fanno solo shift=13. Noi supportiamo qualsiasi chiave da 0 a 25.
- Abbiamo l'analisi delle frequenze integrata per il cracking.

Perché abbiamo fatto scelte diverse:
ROT13 è un caso particolare. Noi insegniamo l'algoritmo generale e come violarlo.

### CyberChef

CyberChef è un coltellino svizzero con decine di codifiche, incluso Cesare. Il nostro tool è costruito appositamente per imparare la crittoanalisi, quindi includiamo il punteggio statistico e la classificazione che CyberChef non mette in risalto.

## Riferimento ai file principali

Mappa rapida di dove trovare le cose:

- `src/caesar_cipher/cipher.py` - Algoritmo principale: _shift_char(), encrypt(), decrypt(), crack()
- `src/caesar_cipher/analyzer.py` - Calcolo del chi-quadrato e classificazione dei candidati
- `src/caesar_cipher/main.py` - Comandi CLI e formattazione delle tabelle Rich
- `src/caesar_cipher/constants.py` - Frequenze delle lettere inglesi (la chiave per violare Cesare)
- `tests/test_cipher.py` - Test di roundtrip cifratura/decifratura
- `tests/test_analyzer.py` - Test di correttezza dell'analisi delle frequenze

## Prossimi passi

Ora che hai capito l'architettura:
1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per la spiegazione del codice
2. Prova a modificare l'algoritmo di shift per ruotare al contrario (le chiavi negative funzionano già, ma rendi diverso il comportamento predefinito)
