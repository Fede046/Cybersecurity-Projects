# Concetti di sicurezza fondamentali

Questo documento spiega i concetti di sicurezza che incontrerai mentre costruisci questo progetto. Non sono solo definizioni: approfondiremo perché sono importanti e come funzionano davvero.

## Cifrari a sostituzione

### Che cos'è

Un cifrario a sostituzione sostituisce ogni lettera del tuo messaggio con una lettera diversa secondo una regola fissa. Il cifrario di Cesare è la versione più semplice: sposta ogni lettera della stessa quantità. Se la tua chiave è 3, A diventa D, B diventa E e così via, finché Z ricomincia da capo diventando C.

### Perché è importante

I cifrari a sostituzione sono stati lo stato dell'arte della crittografia per secoli. Giulio Cesare usò davvero questo metodo per proteggere i messaggi militari intorno al 58 a.C. Capire perché falliscono ti insegna che la security through obscurity non funziona quando lo schema sottostante è troppo semplice.

### Come funziona

La trasformazione è semplicemente aritmetica modulare sulle posizioni nell'alfabeto:
```
Encryption: ciphertext_position = (plaintext_position + key) % 26
Decryption: plaintext_position = (ciphertext_position - key) % 26
```

Per esempio, cifrando "HELLO" con key=3:
```
H (position 7)  → (7+3) % 26 = 10 → K
E (position 4)  → (4+3) % 26 = 7  → H
L (position 11) → (11+3) % 26 = 14 → O
L (position 11) → (11+3) % 26 = 14 → O
O (position 14) → (14+3) % 26 = 17 → R

Result: KHOOR
```

### Attacchi comuni

1. **Brute force** - Prova tutte le 26 chiavi possibili. Con i computer moderni ci vogliono millisecondi. Il key space è troppo piccolo.
2. **Analisi delle frequenze** - Il testo inglese ha frequenze delle lettere note. La E compare il 12,7% delle volte, la T circa il 9%, la Z solo lo 0,07%. Questi schemi sopravvivono alla cifratura di Cesare.
3. **Known plaintext** - Se conosci anche solo una parte del messaggio, puoi calcolare la chiave immediatamente. Una sola coppia plaintext/ciphertext rivela tutto.

### Strategie di difesa

Non puoi difendere il cifrario di Cesare. È fondamentalmente rotto. Ma le lezioni si applicano a cifrari più robusti:
- Key space più grandi rendono il brute force impraticabile (ecco perché AES usa chiavi da 128+ bit)
- La randomizzazione spezza gli schemi di frequenza (i cipher moderni usano trasformazioni diverse per ogni blocco)
- La cifratura autenticata impedisce che gli attacchi known plaintext siano utili

## Analisi delle frequenze

### Che cos'è

Un attacco statistico che sfrutta la distribuzione non uniforme delle lettere nel linguaggio naturale. In inglese, la E è la lettera più comune. Nell'inglese cifrato con Cesare, sarà un'altra lettera la più comune, ma sotto c'è sempre la E. Confrontando la distribuzione delle frequenze del ciphertext con le frequenze note dell'inglese, puoi assegnare un punteggio a quanto è probabile che un dato shift sia corretto.

### Perché è importante

Al-Kindi descrisse questa tecnica nel IX secolo, più di mille anni fa. Violò tutti i cifrari a sostituzione semplice e rimase rilevante fino a quando non furono sviluppati i cifrari polialfabetici come quello di Vigenère. La crittoanalisi moderna usa ancora attacchi statistici, solo contro schemi più complessi.

### Come funziona

Il test del chi-quadrato misura quanto una distribuzione osservata differisce da una attesa:
```
χ² = Σ ((observed - expected)² / expected)
```

Punteggi più bassi significano corrispondenze migliori. In `analyzer.py:27-42`, il codice lo calcola così:
```python
def calculate_chi_squared(self, text: str) -> float:
    text_upper = text.upper()
    letter_counts = Counter(char for char in text_upper if char.isalpha())
    
    total_letters = sum(letter_counts.values())
    chi_squared = 0.0
    
    for letter, expected_freq in self.reference_frequencies.items():
        observed_count = letter_counts.get(letter, 0)
        expected_count = (expected_freq / 100) * total_letters
        
        if expected_count > 0:
            chi_squared += ((observed_count - expected_count)**2) / expected_count
    
    return chi_squared
```

### Errori comuni

**Errore 1: non gestire correttamente maiuscole e minuscole**
```python
# Bad - misses lowercase letters
def count_letters(text):
    return Counter(c for c in text if c.isupper())

# Good - normalize to uppercase first
def count_letters(text):
    return Counter(c for c in text.upper() if c.isalpha())
```

L'analisi delle frequenze ha bisogno di tutte le lettere. Il codice in `analyzer.py:29` converte in maiuscolo prima di contare.

**Errore 2: includere caratteri non alfabetici nel conteggio delle frequenze**

Spazi, punteggiatura e numeri falseranno le tue statistiche. Conta solo le lettere vere e proprie. Il codice usa `if char.isalpha()` per filtrare correttamente.

**Errore 3: un testo corto dà risultati inaffidabili**

Ti servono almeno 50-100 lettere perché l'analisi delle frequenze funzioni. Con "HI" cifrato, non ci sono abbastanza dati. Il test del chi-quadrato restituisce `float("inf")` per le stringhe vuote in `analyzer.py:33`.

## Attacchi brute force

### Che cos'è

Semplicemente provare ogni chiave possibile finché non ne trovi una che funziona. Per il cifrario di Cesare, sono solo 26 tentativi. Il tuo computer può fare milioni di tentativi al secondo, quindi è istantaneo.

### Perché è importante

Il brute force stabilisce la sicurezza massima assoluta di qualsiasi cipher. Anche con un'implementazione perfetta, se il key space è troppo piccolo, il cipher è rotto. Ecco perché la complessità delle password conta: ogni carattere aggiuntivo moltiplica lo spazio di ricerca in modo esponenziale.

### Come funziona

Il metodo `crack()` in `cipher.py:53-60` lo implementa:
```python
@staticmethod
def crack(ciphertext: str) -> list[tuple[int, str]]:
    results = []
    for shift in range(ALPHABET_SIZE):
        cipher = CaesarCipher(key=shift)
        decrypted = cipher.decrypt(ciphertext)
        results.append((shift, decrypted))
    return results
```

Prova lo shift 0, lo shift 1, lo shift 2, fino allo shift 25. Restituisci tutti i risultati e lascia che l'analisi delle frequenze scelga il migliore.

### Analisi del key space
```
Caesar cipher:     26 possible keys (2^4.7 bits)
4-digit PIN:       10,000 possibilities (2^13 bits)
8-char password:   ~200 trillion (2^47 bits if using a-z, A-Z, 0-9)
AES-128:           2^128 ≈ 10^38 possibilities
```

Qualsiasi cosa sotto 2^40 è considerata oggi attaccabile con brute force. Cesare è ridicolmente debole.

## Come si collegano questi concetti
```
Substitution Cipher (weak pattern)
    ↓
preserves letter frequencies
    ↓
Frequency Analysis (detects pattern)
    ↓
scores all possible keys
    ↓
Brute Force (tries all keys)
    ↓
Cipher is broken
```

La catena di vulnerabilità: la sostituzione semplice crea uno schema rilevabile, l'analisi delle frequenze sfrutta quello schema, il brute force rende pratico provare tutte le chiavi.

## Standard e framework di settore

### OWASP Top 10

Questo progetto affronta:
- **A02:2021 - Cryptographic Failures** - Dimostra perché gli algoritmi crittografici deboli falliscono. Mostra una corretta validazione della chiave (anche se l'algoritmo stesso è didattico, non pronto per la produzione).

### MITRE ATT&CK

Tecniche rilevanti:
- **T1552.001** - Credentials from Password Stores - Una cifratura debole delle credenziali memorizzate può essere violata in questo modo
- **T1140** - Deobfuscate/Decode Files or Information - Gli attaccanti usano l'analisi delle frequenze su ROT13 e "offuscamenti" simili

### CWE

Common weakness enumeration trattate:
- **CWE-327** - Use of a Broken or Risky Cryptographic Algorithm - Il cifrario di Cesare è l'esempio da manuale
- **CWE-326** - Inadequate Encryption Strength - 4,7 bit di robustezza della chiave sono inadeguati per qualsiasi cosa

## Esempi reali

### Caso di studio 1: il cifrario del Killer dello Zodiaco (1969)

Il Killer dello Zodiaco inviava messaggi cifrati ai giornali. Il suo cifrario Z408 era una sostituzione omofonica (più simboli per lettera) ma cadde comunque sotto l'analisi delle frequenze. Fu risolto nel 1969 da un insegnante e da sua moglie con carta e matita.

Cosa è successo: il killer usò 54 simboli diversi, ma questi corrispondevano comunque a 26 lettere. L'analisi delle frequenze rivelò gli schemi. Il cifrario Z340 richiese 51 anni per essere violato (finalmente risolto nel 2020) perché usava tecniche polialfabetiche.

Come si sarebbe potuto evitare: con una cifratura moderna con una lunghezza di chiave adeguata e randomizzazione. Usare ROT13 o Cesare su dati sensibili è security theater.

### Caso di studio 2: CVE-2015-2187 (ROT13 per le password)

Nel 2015, alcuni ricercatori scoprirono che il firmware di alcuni router memorizzava le password di amministrazione usando la codifica ROT13. Questo è il cifrario di Cesare con shift=13. Chiunque avesse accesso ai file poteva decifrare istantaneamente tutte le password.

Cosa è successo: gli sviluppatori hanno confuso la codifica con la cifratura. ROT13 è pensato per nascondere gli spoiler, non per la sicurezza.

Quali difese hanno fallito: nessuna security review ha intercettato l'uso di un cipher rotto per la memorizzazione delle credenziali.

Lezione: non usare mai il cifrario di Cesare (o ROT13) per nulla che riguardi la sicurezza. Usa bcrypt, scrypt o argon2 per l'hashing delle password.

## Verifica la tua comprensione

Prima di passare all'architettura, assicurati di saper rispondere a:

1. Se cifri "ATTACK AT DAWN" con key=7, la lettera A verrà sempre cifrata nella stessa lettera? Perché questo è importante per la sicurezza?

2. Intercetti "WKLV LV D WHVW" e sai che è cifrato con Cesare. Descrivi due modi diversi per decifrarlo senza provare con brute force tutte le 26 chiavi.

3. Perché l'analisi delle frequenze funziona sul cifrario di Cesare ma non sui cipher moderni come AES?

Se queste domande ti sembrano poco chiare, rileggi le sezioni pertinenti. L'implementazione avrà più senso una volta che questi fondamenti saranno chiari.

## Letture di approfondimento

**Essenziali:**
- "The Code Book" di Simon Singh - Il capitolo sui cifrari storici spiega Cesare e l'analisi delle frequenze con ottimi esempi
- Wikipedia: Chi-squared test - Fondamento matematico per il punteggio statistico

**Approfondimenti:**
- "Applied Cryptography" di Bruce Schneier - Il capitolo 1 spiega perché i cifrari classici falliscono
- Il manoscritto originale di Al-Kindi sull'analisi delle frequenze (sono disponibili versioni tradotte)

**Contesto storico:**
- "The Codebreakers" di David Kahn - Storia della crittoanalisi dall'antichità alla Seconda guerra mondiale
