# Caesar Cipher CLI Tool

## Che cos'è

Un tool da riga di comando che implementa il cifrario di Cesare, una delle più antiche tecniche di cifratura conosciute. Sposta ogni lettera del tuo testo di un numero fisso di posizioni nell'alfabeto. Il tool può cifrare messaggi, decifrarli se conosci la chiave, oppure violare testo cifrato provando tutti gli shift possibili e classificandoli tramite l'analisi delle frequenze.

## Perché è importante

Il cifrario di Cesare è debole per gli standard moderni, ma capire come violarlo insegna competenze fondamentali di crittoanalisi che si applicano anche a sistemi più robusti. Ogni professionista della sicurezza dovrebbe sapere perché i cifrari a sostituzione semplice falliscono.

**Scenari reali in cui questo si applica:**
- ROT13 è ancora usato per nascondere gli spoiler nei forum e nelle email (è il cifrario di Cesare con key=13)
- Capire l'analisi delle frequenze ti aiuta a violare altri cifrari a sostituzione che si trovano nelle sfide CTF
- Il concetto di brute force su un key space ridotto si applica a password deboli, PIN corti e crittografia progettata male

## Cosa imparerai

Questo progetto ti insegna come la crittografia classica crolla sotto l'analisi statistica. Costruendolo da solo, capirai:

**Concetti di sicurezza:**
- Cifrari a sostituzione: come funzionano e perché la frequenza dei caratteri li tradisce
- Attacchi brute force: quando il key space è abbastanza piccolo (qui 26 possibilità), provare tutto è banale
- Analisi delle frequenze: il vero testo inglese ha schemi di lettere prevedibili che sopravvivono alla cifratura

**Competenze tecniche:**
- Test statistico del chi-quadrato per valutare quanto un testo appaia "simile all'inglese"
- Costruire tool CLI con un corretto parsing degli argomenti (usando Typer)
- Implementare sia la cifratura sia la crittoanalisi nella stessa codebase

**Strumenti e tecniche:**
- Typer per interfacce a riga di comando con testo di aiuto automatico
- La libreria Rich per output colorato nel terminale e tabelle formattate
- Counter di Python per il conteggio delle frequenze e l'analisi statistica

## Prerequisiti

Prima di iniziare, dovresti conoscere:

**Conoscenze richieste:**
- Basi di Python: funzioni, classi, list comprehension
- Manipolazione delle stringhe: iterare sui caratteri, verificare se sono lettere
- Aritmetica modulare: perché `(25 + 3) % 26 = 2` fa ricominciare l'alfabeto dall'inizio

**Strumenti di cui avrai bisogno:**
- Python 3.12 o superiore
- pip per installare le dipendenze
- Un terminale in cui eseguire comandi

**Utili ma non obbligatori:**
- Statistica di base (cosa significa chi-quadrato)
- Un minimo di familiarità con i concetti di cifratura

## Avvio rapido

Fai girare il progetto in locale:
```bash
# Clone and navigate
cd PROJECTS/beginner/caesar-cipher

# Install with dependencies
pip install -e .

# Encrypt some text
caesar-cipher encrypt "HELLO WORLD" --key 3

# Decrypt it back
caesar-cipher decrypt "KHOOR ZRUOG" --key 3

# Crack it without knowing the key
caesar-cipher crack "KHOOR ZRUOG"
```

Output atteso: dovresti vedere `KHOOR ZRUOG` durante la cifratura, `HELLO WORLD` durante la decifratura e, durante il cracking, una tabella classificata di tutte le 26 possibili decifrature (con quella corretta in cima).

## Struttura del progetto
```
caesar-cipher/
├── src/caesar_cipher/
│   ├── cipher.py         # Core encryption/decryption logic
│   ├── analyzer.py       # Frequency analysis for cracking
│   ├── constants.py      # English letter frequencies, alphabet
│   ├── main.py          # CLI commands (encrypt/decrypt/crack)
│   └── utils.py         # File I/O and input validation
├── tests/               # Pytest test suite
└── pyproject.toml      # Project dependencies and config
```

## Prossimi passi

1. **Comprendi i concetti** - Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per capire perché i cifrari di Cesare sono violabili
2. **Studia l'architettura** - Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per vedere come si incastrano i vari pezzi
3. **Ripercorri il codice** - Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per una spiegazione riga per riga
4. **Estendi il progetto** - Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per costruire varianti come il cifrario di Vigenère

## Problemi comuni

**Errore "Key must be between -25 and 26"**
```
ValueError: Key must be between -25 and 26
```
Soluzione: il cifrario di Cesare ha senso solo con shift da -25 a 26. Usa una chiave in quell'intervallo. Il codice lo valida in `cipher.py:20-22`.

**Nessun output quando si fa il piping da stdin**
Soluzione: assicurati di inviare effettivamente del testo. Prova `echo "TEST" | caesar-cipher encrypt --key 5` invece di solo `caesar-cipher encrypt --key 5`.
