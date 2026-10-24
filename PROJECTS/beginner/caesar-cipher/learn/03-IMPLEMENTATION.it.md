# Guida all'implementazione

Questo documento ripercorre il codice vero e proprio. Costruiremo le funzionalità principali passo dopo passo e spiegheremo le decisioni lungo il percorso.

## Panoramica della struttura dei file
```
caesar-cipher/
├── src/caesar_cipher/
│   ├── __init__.py     # Package exports for CaesarCipher and FrequencyAnalyzer
│   ├── cipher.py       # Core encryption/decryption logic (62 lines)
│   ├── analyzer.py     # Chi-squared frequency analysis (60 lines)
│   ├── constants.py    # English letter frequencies, alphabet (43 lines)
│   ├── main.py         # Typer CLI with 3 commands (171 lines)
│   └── utils.py        # File I/O and validation (40 lines)
├── tests/
│   ├── test_cipher.py  # Cipher tests with edge cases
│   ├── test_analyzer.py # Frequency analysis tests
│   └── test_cli.py     # End-to-end CLI tests
└── pyproject.toml      # Dependencies and tool config
```

## Costruire il cipher principale

### Passo 1: shift dei caratteri

Cosa stiamo costruendo: l'operazione fondamentale che prende una lettera e la sposta di N posizioni.

In `cipher.py:30-40`:
```python
def _shift_char(self, char: str, shift: int) -> str:
    """
    Shift a single character by the specified amount while preserving case
    """
    if char in UPPERCASE_LETTERS:
        idx = UPPERCASE_LETTERS.index(char)
        return UPPERCASE_LETTERS[(idx + shift) % ALPHABET_SIZE]
    if char in LOWERCASE_LETTERS:
        idx = LOWERCASE_LETTERS.index(char)
        return LOWERCASE_LETTERS[(idx + shift) % ALPHABET_SIZE]
    return char
```

**Perché questo codice funziona:**
- Righe 35-36: le lettere maiuscole vengono spostate all'interno delle maiuscole. 'A' + 3 = 'D'. Il modulo gestisce il ritorno all'inizio ('Z' + 1 = 'A').
- Righe 37-39: le lettere minuscole vengono spostate all'interno delle minuscole in modo indipendente. Preserva maiuscole e minuscole.
- Riga 40: i caratteri non alfabetici vengono restituiti invariati. Spazi, punteggiatura e numeri passano così come sono.

**Errori comuni qui:**
```python
# Wrong approach - converts everything to uppercase
def shift_char(char, shift):
    if char.isalpha():
        idx = ord(char.upper()) - ord('A')
        return chr((idx + shift) % 26 + ord('A'))
    return char

# Why this fails: "Hello" becomes "KHOOR" instead of "Khoor"
```

Ti serve una gestione separata per maiuscole e minuscole. Il codice usa due diverse stringhe di alfabeto (`UPPERCASE_LETTERS` e `LOWERCASE_LETTERS` da constants.py) per preservare in modo naturale maiuscole e minuscole.

### Passo 2: cifratura dell'intero messaggio

Ora dobbiamo applicare lo shift a ogni carattere del messaggio.

In `cipher.py:43-46`:
```python
def encrypt(self, plaintext: str) -> str:
    """
    Encrypt plaintext using the configured shift key
    """
    return "".join(self._shift_char(char, self.key) for char in plaintext)
```

**Cosa succede:**
1. La generator expression itera su ogni carattere: `for char in plaintext`
2. Chiama `_shift_char()` con la chiave dell'istanza
3. Unisce tutti i risultati in una stringa

**Perché lo facciamo in questo modo:**
È più efficiente in termini di memoria rispetto a costruire una lista e poi unirla. Per testi enormi, il generator produce i caratteri uno alla volta invece di tenere in memoria tutti i caratteri spostati.

**Approcci alternativi:**
- Ciclo con `result += shifted_char`: funziona ma crea stringhe intermedie (tempo O(n²) nel caso peggiore)
- List comprehension e poi join: `"".join([...])`: usa più memoria ma ha la stessa velocità

### Passo 3: decifratura (l'operazione inversa)

In `cipher.py:48-52`:
```python
def decrypt(self, ciphertext: str) -> str:
    """
    Decrypt ciphertext using the configured shift key
    """
    return "".join(self._shift_char(char, -self.key) for char in ciphertext)
```

**Parti chiave spiegate:**

La decifratura è semplicemente cifratura con una chiave negativa. Se abbiamo cifrato con +3, decifriamo con -3. Il metodo `_shift_char()` gestisce automaticamente gli shift negativi tramite l'aritmetica modulare:
```
'K' is position 10
Shift by -3: (10 + (-3)) % 26 = 7 % 26 = 7
Position 7 is 'H'
```

L'operatore modulo di Python gestisce correttamente i negativi: `(-1) % 26 = 25`, che riporta 'A' a 'Z'.

## Costruire il cracker brute force

### Il problema

Hai il ciphertext ma non conosci la chiave. Con solo 26 possibilità, provale tutte.

### La soluzione

Genera tutte le 26 decifrature e lascia che l'analisi delle frequenze scelga la migliore.

### Implementazione

In `cipher.py:53-60`:
```python
@staticmethod
def crack(ciphertext: str) -> list[tuple[int, str]]:
    """
    Attempt all possible shifts to decrypt ciphertext without knowing the key
    """
    results = []
    for shift in range(ALPHABET_SIZE):
        cipher = CaesarCipher(key=shift)
        decrypted = cipher.decrypt(ciphertext)
        results.append((shift, decrypted))
    return results
```

**Perché un metodo statico:**
Non opera su un'istanza specifica. Genera tutte le istanze possibili. Renderlo statico rende chiara l'intenzione: `CaesarCipher.crack(text)` è come chiedere "quali sono tutte le decifrature di Cesare di questo testo?"

**Ottimizzazione che non abbiamo fatto:**
Potremmo evitare di creare 26 oggetti cipher e chiamare direttamente `_shift_char()`. Ma qui la chiarezza batte la micro-ottimizzazione. Creare 26 oggetti è trascurabile.

## Costruire l'analisi delle frequenze

### Il problema

Quando violi un messaggio, ottieni 26 candidati. Quale di questi è vero inglese?

### La soluzione

Assegna un punteggio a ogni candidato in base a quanto le sue frequenze delle lettere corrispondono alle frequenze note dell'inglese. Punteggi chi-quadrato più bassi significano corrispondenze migliori.

### Implementazione

In `analyzer.py:27-42`:
```python
def calculate_chi_squared(self, text: str) -> float:
    """
    Calculate chi-squared statistic comparing text to expected English frequencies
    """
    text_upper = text.upper()
    letter_counts = Counter(char for char in text_upper if char.isalpha())
    
    if not letter_counts:
        return float("inf")
    
    total_letters = sum(letter_counts.values())
    chi_squared = 0.0
    
    for letter, expected_freq in self.reference_frequencies.items():
        observed_count = letter_counts.get(letter, 0)
        expected_count = (expected_freq / 100) * total_letters
        
        if expected_count > 0:
            chi_squared += ((observed_count - expected_count)**2) / expected_count
    
    return chi_squared
```

**Scomponiamolo:**

**Righe 29-30:** converte in maiuscolo e conta solo le lettere. `Counter` ci dà `{'H': 1, 'E': 1, 'L': 2, 'O': 1}` per "HELLO".

**Righe 32-33:** un testo vuoto ha un punteggio infinitamente cattivo. Non si può dare un punteggio al nulla.

**Riga 35:** il conteggio totale serve per convertire le percentuali in conteggi attesi.

**Righe 38-41:** per ogni lettera A-Z, confronta osservato e atteso:
- Atteso: se il testo ha 100 lettere e la E dovrebbe essere il 12,7%, ci aspettiamo 12,7 lettere E
- Osservato: quante E ci sono effettivamente
- Il chi-quadrato somma le differenze al quadrato normalizzate per i valori attesi

Punteggi più bassi significano più vicino all'inglese. Il testo senza senso ha distribuzioni di frequenza sballate e ottiene punteggi alti.

### Classificare i candidati

In `analyzer.py:53-60`:
```python
def rank_candidates(self, candidates: list[tuple[int, str]]) -> list[tuple[int, str, float]]:
    """
    Rank decryption candidates by their English frequency score
    """
    scored = [
        (shift, text, self.score_text(text)) 
        for shift, text in candidates
    ]
    return sorted(scored, key=lambda x: x[2])
```

Prende una lista di tuple `(shift, decrypted_text)`, aggiunge i punteggi e ordina per punteggio (in ordine crescente, quindi il migliore è il primo).

## Implementazione della CLI

### Struttura dei comandi

Il tool ha tre comandi: encrypt, decrypt, crack. Tutti usano schemi simili.

**Comando encrypt** (`main.py:24-60`):
```python
@app.command()
def encrypt(
    text: Annotated[str | None, typer.Argument(...)] = None,
    key: Annotated[int, typer.Option("--key", "-k", ...)] = 3,
    input_file: Annotated[Path | None, typer.Option(...)] = None,
    output_file: Annotated[Path | None, typer.Option(...)] = None,
    quiet: Annotated[bool, typer.Option(...)] = False,
) -> None:
```

**La sintassi Annotated:**
Typer usa i type hint per generare la CLI. `Annotated[str | None, typer.Argument(...)]` significa "questo è un argomento stringa opzionale". Il testo di aiuto si trova nella chiamata `Argument()`.

**Flusso di validazione:**
```python
try:
    validate_key(key)  # utils.py:36
    plaintext = read_input(text, input_file)  # utils.py:11
    cipher = CaesarCipher(key=key)
    encrypted = cipher.encrypt(plaintext)
    # ... output ...
except (ValueError, OSError) as e:
    console.print(f"[red]Error:[/red] {e}")
    raise typer.Exit(code=1) from None
```

**Gestione degli errori:**
Cattura sia `ValueError` (dalla validazione) sia `OSError` (dalle operazioni sui file). Stampa l'errore in rosso usando Rich, poi termina con codice 1. Il `from None` sopprime la catena di eccezioni nell'output.

### Gestione dell'input

Tre modi per fornire l'input: argomento da riga di comando, file o stdin.

In `utils.py:11-24`:
```python
def read_input(text: str | None, input_file: Path | None) -> str:
    """
    Read input from text argument, file, or stdin
    """
    if text:
        return text
    
    if input_file:
        return input_file.read_text(encoding="utf-8")
    
    if not sys.stdin.isatty():
        return sys.stdin.read()
    
    raise ValueError("No input provided. Use TEXT argument, --input-file, or pipe to stdin")
```

**Ordine di priorità:**
1. Argomento di testo esplicito
2. File di input
3. Stdin (ma solo se è in pipe, non un terminale interattivo)

**Il controllo `sys.stdin.isatty()`:**
Restituisce `False` quando l'input arriva tramite pipe: `echo "test" | caesar-cipher encrypt`. Restituisce `True` quando l'esecuzione è interattiva. Leggiamo stdin solo se arriva tramite pipe, per evitare di restare bloccati in attesa di un input dell'utente che non arriverà.

### Output in tabella Rich

Il comando crack mostra i risultati in una tabella formattata.

In `main.py:135-148`:
```python
table = Table(title="Caesar Cipher Brute Force Results")
table.add_column("Rank", style="cyan", justify="right")
table.add_column("Shift", style="magenta", justify="right")
table.add_column("Score", style="yellow", justify="right")
table.add_column("Decrypted Text", style="green")

display_count = len(ranked) if show_all else min(top, len(ranked))

for rank, (shift, text_result, score) in enumerate(ranked[:display_count], 1):
    table.add_row(
        str(rank),
        str(shift),
        f"{score:.2f}",
        text_result[:80]  # Truncate long text
    )

console.print(table)
```

**Perché Rich:**
L'output colorato rende evidente quale colonna è quale. La tabella si formatta e allinea le colonne automaticamente. Molto meglio che stampare valori separati da tabulazioni.

**Troncamento:**
`text_result[:80]` limita il testo mostrato a 80 caratteri. Altrimenti decifrature molto lunghe renderebbero la tabella illeggibile.

## Strategia di test

### Unit test per il cipher

Esempio di test per la cifratura (`test_cipher.py:14-16`):
```python
def test_encrypt_basic(self) -> None:
    cipher = CaesarCipher(key=3)
    assert cipher.encrypt("HELLO") == "KHOOR"
```

**Cosa testa:**
La funzionalità di shift di base. Se questo fallisce, è tutto rotto.

**Casi limite testati:**
- Ritorno all'inizio: `test_alphabet_wraparound_uppercase()` verifica 'XYZ' → 'ABC'
- Preservazione di maiuscole/minuscole: `test_encrypt_mixed_case()` verifica "Hello World" → "Khoor Zruog"
- Caratteri non alfabetici: `test_encrypt_preserves_punctuation()` verifica "Hello!" → "Khoor!"
- Stringa vuota: `test_empty_string()` verifica "" → ""

**Test di roundtrip** (`test_cipher.py:42-46`):
```python
def test_encrypt_decrypt_roundtrip(self) -> None:
    cipher = CaesarCipher(key=13)
    original = "The Quick Brown Fox Jumps Over The Lazy Dog!"
    encrypted = cipher.encrypt(original)
    decrypted = cipher.decrypt(encrypted)
    assert decrypted == original
```

Questo intercetta i bug di asimmetria. Se encrypt e decrypt non sono veri inversi, fallisce.

### Test di integrazione per l'analyzer

Verificare che l'analisi delle frequenze scelga effettivamente la risposta giusta (`test_analyzer.py:44-54`):
```python
def test_rank_candidates_with_actual_cipher(self) -> None:
    cipher = CaesarCipher(key=3)
    plaintext = "THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG"
    ciphertext = cipher.encrypt(plaintext)
    
    candidates = CaesarCipher.crack(ciphertext)
    analyzer = FrequencyAnalyzer()
    ranked = analyzer.rank_candidates(candidates)
    
    best_shift, best_text, _best_score = ranked[0]
    assert best_shift == 3
    assert best_text == plaintext
```

Questo è un test di integrazione perché usa insieme sia il cipher sia l'analyzer. Verifica che l'intero workflow di crack funzioni end to end.

### Test della CLI

Testare i comandi veri e propri (`test_cli.py:14-18`):
```python
def test_encrypt_basic(self) -> None:
    result = runner.invoke(app, ["encrypt", "HELLO", "--key", "3"])
    assert result.exit_code == 0
    assert "KHOOR" in result.stdout
```

Usa il test runner di Typer per simulare l'invocazione da riga di comando. Controlla l'exit code e l'output.

**Test di I/O su file** (`test_cli.py:63-73`):
```python
def test_encrypt_from_file(self, tmp_path: Path) -> None:
    input_file = tmp_path / "input.txt"
    input_file.write_text("HELLO WORLD")
    
    result = runner.invoke(
        app,
        ["encrypt", "--input-file", str(input_file), "--key", "3"]
    )
    assert result.exit_code == 0
    assert "KHOOR ZRUOG" in result.stdout
```

Crea un file temporaneo, lo legge e verifica l'output. Questo testa il percorso di gestione dei file senza lasciare residui.

## Insidie comuni nell'implementazione

### Insidia 1: dimenticare di gestire le chiavi negative

**Sintomo:**
Crash o output errato quando si usa `--key -3`.

**Causa:**
```python
# Problematic code - doesn't normalize negative keys
def __init__(self, key: int):
    self.key = key  # If key=-3, modulo in _shift_char breaks
```

**Correzione:**
```python
# Correct approach (cipher.py:23)
self.key = key % ALPHABET_SIZE
```

Normalizzare durante la costruzione significa che tutto il resto del codice può dare per scontato che la chiave sia 0-25.

**Perché è importante:**
Il cifrario di Cesare con key=-3 è uguale a key=23. Entrambi spostano a sinistra di 3. La normalizzazione li rende equivalenti.

### Insidia 2: non gestire l'input vuoto

**Sintomo:**
Divisione per zero nel calcolo del chi-quadrato.

**Causa:**
```python
# Bad - crashes if text is empty
def calculate_chi_squared(text):
    total = sum(counts.values())
    # ... expected_count / total will divide by zero
```

**Correzione:**
```python
# Good - early return (analyzer.py:32-33)
if not letter_counts:
    return float("inf")
```

Un testo vuoto ha una corrispondenza di frequenza infinitamente cattiva. Non può essere inglese se non ci sono lettere.

### Insidia 3: confondere cifratura e decifratura

**Sintomo:**
`decrypt("KHOOR")` con key=3 restituisce testo senza senso invece di "HELLO".

**Causa:**
```python
# Wrong - decrypt shifts forward instead of backward
def decrypt(self, ciphertext):
    return self.encrypt(ciphertext)  # This is just double encryption!
```

**Correzione:**
```python
# Right - decrypt shifts backward (cipher.py:51)
return "".join(self._shift_char(char, -self.key) for char in ciphertext)
```

La decifratura deve invertire lo shift. La chiave negativa fa proprio questo.

## Suggerimenti per il debug

### Tipo di problema 1: l'analisi delle frequenze dà la risposta sbagliata

**Problema:** il candidato classificato al primo posto non è la decifratura corretta.

**Come fare debug:**
1. Controlla `constants.py:17-43` - Le frequenze dell'inglese sono corrette?
2. Stampa i punteggi di tutti i 26 candidati - Quello corretto è vicino alla cima?
3. Prova con un ciphertext più lungo - Un testo breve non ha abbastanza lettere per un'analisi delle frequenze affidabile

**Cause comuni:**
- Il testo è troppo corto (meno di 50 lettere)
- Il testo non è in inglese (l'analisi delle frequenze presuppone l'inglese)
- Il testo ha una distribuzione delle parole insolita (gergo tecnico, nomi)

### Tipo di problema 2: la CLI si blocca quando esegui un comando

**Problema:** `caesar-cipher encrypt --key 3` resta bloccato per sempre.

**Come fare debug:**
1. Controlla se hai dimenticato di fornire il testo di input
2. Guarda `utils.py:20-21` - Se stdin non è un TTY, resta in attesa di input tramite pipe
3. Fornisci il testo: `caesar-cipher encrypt "HELLO" --key 3` oppure passalo tramite pipe: `echo "HELLO" | caesar-cipher encrypt --key 3`

**Causa comune:**
Eseguire il comando senza testo e senza passare nulla a stdin tramite pipe. Il codice attende un input che non arriva mai.

## Principi di organizzazione del codice

### Perché cipher.py è indipendente
```
cipher.py
├── Only imports from constants
├── No CLI dependencies
└── No analyzer dependencies
```

Separiamo il cipher dalle sue applicazioni perché:
- Puoi fare unit test della cifratura senza fare il mock della CLI
- Il cipher potrebbe essere usato in una web API senza cambiare nulla
- I test si concentrano sulla correttezza dell'algoritmo, non sull'I/O

Questo rende la codebase più facile da comprendere. Se i test in `test_cipher.py` passano, l'algoritmo è corretto indipendentemente da come viene invocato.

### Convenzioni di naming

- `_shift_char` (underscore iniziale) = metodo helper interno, non fa parte dell'API pubblica
- `UPPERCASE_LETTERS` (tutto maiuscolo) = costante che non dovrebbe cambiare
- `crack` (verbo) = i metodi sono azioni

Seguire questi schemi rende più facile distinguere tra API pubblica, implementazione interna e configurazione.

## Estendere il codice

### Aggiungere un nuovo alfabeto

Vuoi supportare i numeri nella cifratura? Ecco il procedimento:

1. **Modifica le costanti** in `constants.py:10-11`
```python
   DIGITS = string.digits
   ALL_CHARS = UPPERCASE_LETTERS + LOWERCASE_LETTERS + DIGITS
```

2. **Aggiorna _shift_char** in `cipher.py:30-40`
```python
   def _shift_char(self, char: str, shift: int) -> str:
       if char in UPPERCASE_LETTERS:
           idx = UPPERCASE_LETTERS.index(char)
           return UPPERCASE_LETTERS[(idx + shift) % 26]
       if char in LOWERCASE_LETTERS:
           idx = LOWERCASE_LETTERS.index(char)
           return LOWERCASE_LETTERS[(idx + shift) % 26]
       if char in DIGITS:
           idx = DIGITS.index(char)
           return DIGITS[(idx + shift) % 10]  # 10 digits
       return char
```

3. **Aggiungi i test** in `test_cipher.py`
```python
   def test_encrypt_digits(self) -> None:
       cipher = CaesarCipher(key=3)
       assert cipher.encrypt("Test123") == "Whvw456"
```

L'architettura rende tutto questo semplice perché la logica del cipher è isolata da tutto il resto.

## Dipendenze

### Perché ogni dipendenza

- **typer** (0.20.0): framework CLI con help automatico e type hint. Migliore di argparse per il Python moderno.
- **rich** (14.2.0): output colorato nel terminale e formattazione delle tabelle. Rende leggibile l'output del comando crack.
- **pytest** (9.0.2): framework di test. Lo standard per i test in Python.
- **mypy** (1.19.0): type checker statico. Intercetta gli errori di tipo prima del runtime.
- **ruff** (0.14.8): linter veloce che combina molti tool. Sostituisce flake8, isort, black.

### Sicurezza delle dipendenze

Controlla la presenza di vulnerabilità:
```bash
pip install pip-audit
pip-audit
```

Se vedi una vulnerabilità in typer o rich, controlla se esiste una versione più recente che la corregge. Aggiorna i vincoli di versione in `pyproject.toml:7-9`.

## Build e deploy

### Build
```bash
# Install in development mode
pip install -e .

# Or build a wheel
pip install build
python -m build
```

Questo produce una wheel in `dist/` che può essere installata ovunque.

### Sviluppo locale
```bash
# Install with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Run linter
ruff check src/ tests/

# Run type checker
mypy src/
```

L'extra `[dev]` include i tool di test e linting da `pyproject.toml:10-17`.

### Deploy in produzione

Per un tool CLI, "produzione" significa pubblicare su PyPI:
```bash
# Build
python -m build

# Upload to PyPI
python -m twine upload dist/*
```

Poi gli utenti installano con `pip install caesar-salad-cipher` e ottengono il comando `caesar-cipher` nel loro PATH.

## Prossimi passi

Hai visto come funziona il codice. Ora:

1. **Prova le sfide** - [04-CHALLENGES.md](./04-CHALLENGES.it.md) contiene idee di estensione come il cifrario di Vigenère
2. **Modifica il codice** - Cambia il punteggio delle frequenze per usare i bigrammi (coppie di due lettere) invece delle singole lettere
3. **Leggi progetti correlati** - Guarda tool reali di analisi delle frequenze per trarne ispirazione
