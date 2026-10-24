# Sfide di estensione

Hai costruito il progetto base. Ora rendilo tuo estendendolo con nuove funzionalità.

Queste sfide sono ordinate per difficoltà. Inizia con quelle più facili per acquisire sicurezza, poi affronta quelle più difficili quando vuoi approfondire.

## Sfide facili

### Sfida 1: aggiungere il supporto allo shift per posizione

**Cosa costruire:**
Invece di spostare ogni lettera della stessa quantità, sposta la prima lettera di 1, la seconda di 2, la terza di 3 e così via. Questa si chiama variante autokey.

**Perché è utile:**
Questo rompe l'analisi delle frequenze. La stessa lettera del plaintext viene cifrata in lettere diverse del ciphertext a seconda della posizione. Molto più robusto del Cesare base.

**Cosa imparerai:**
- Come funziona la cifratura dipendente dalla posizione
- Perché i cifrari polialfabetici sono più difficili da violare
- Modificare il ciclo principale del cipher

**Suggerimenti:**
- Guarda `cipher.py:43-46` - Dovrai passare a `_shift_char()` sia il carattere sia la sua posizione
- Usa `enumerate()` per ottenere le posizioni dei caratteri
- La chiave diventa `(self.key + position) % ALPHABET_SIZE`

**Verifica che funzioni:**
```bash
caesar-cipher encrypt "AAA" --key 1
# Should give "ABC" not "BBB"
```

### Sfida 2: supportare alfabeti personalizzati

**Cosa costruire:**
Permetti agli utenti di specificare il proprio alfabeto, come "QWERTYUIOPASDFGHJKLZXCVBNM" (ordine della tastiera) invece di "ABC...Z".

**Perché è utile:**
Gli alfabeti personalizzati sono usati in veri cifrari storici (come i dispositivi a disco cifrante). Questo ti insegna la flessibilità dei cipher.

**Cosa imparerai:**
- Come validare un input personalizzato
- Perché l'ordine dell'alfabeto non influisce sulla debolezza di Cesare
- Design delle interfacce per parametri opzionali

**Suggerimenti:**
- `CaesarCipher.__init__()` accetta già un parametro `alphabet` in `cipher.py:16`
- Aggiungi un'opzione `--alphabet` in `main.py`
- Valida che l'alfabeto abbia 26 caratteri univoci

**Verifica che funzioni:**
```bash
caesar-cipher encrypt "HELLO" --key 1 --alphabet "ZYXWVUTSRQPONMLKJIHGFEDCBA"
# Should shift using reverse alphabet
```

### Sfida 3: aggiungere la visualizzazione delle statistiche

**Cosa costruire:**
Un comando `stats` che mostri la distribuzione delle frequenze delle lettere per qualsiasi testo, con un grafico a barre visivo nel terminale.

**Perché è utile:**
Visualizzare le frequenze rende evidente perché l'analisi delle frequenze funziona. Vedrai subito i picchi di E e T.

**Cosa imparerai:**
- Visualizzazione dei dati nel terminale
- Usare le progress bar di Rich o una formattazione personalizzata
- Presentare informazioni statistiche in modo chiaro

**Suggerimenti:**
- Usa `collections.Counter` come in `analyzer.py:30`
- Rich può creare grafici a barre con caratteri `█` ripetuti N volte
- Ordina le lettere per frequenza per rendere evidenti gli schemi

**Verifica che funzioni:**
```bash
caesar-cipher stats "THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG"
# Should show E and O as most frequent
```

## Sfide intermedie

### Sfida 4: implementare la modalità di codifica ROT13

**Cosa costruire:**
Un comando speciale `rot13` ottimizzato per il caso comune shift=13. Rendilo bidirezionale (il ROT13 del ROT13 è l'originale).

**Applicazione nel mondo reale:**
ROT13 è effettivamente usato su Reddit, nei forum e nelle email per gli spoiler. Non è sicurezza, solo offuscamento.

**Cosa imparerai:**
- Perché shift=13 è speciale (è l'inverso di sé stesso in un alfabeto di 26 lettere)
- Alias dei comandi nei tool CLI
- Implementazioni specializzate rispetto a quelle generiche

**Approccio all'implementazione:**

1. **Aggiungi il comando** a `main.py`
   - File da creare: nessuno (aggiungi al file esistente)
   - File da modificare: `main.py` (aggiungi `@app.command()` per rot13)

2. **Ottimizza per il caso speciale**
   - Dato che la chiave è sempre 13, puoi scriverla in modo fisso nel codice
   - Non servono encrypt/decrypt separati (stessa operazione)

3. **Testa i casi limite:**
   - Cosa succede se il testo contiene numeri o punteggiatura?
   - Gestisce correttamente Unicode?

**Suggerimenti:**
- ROT13 è semplicemente `CaesarCipher(key=13)`
- Il comando può essere più semplice di encrypt/decrypt, dato che non ha l'argomento key
- Fallo funzionare su stdin per il piping: `echo "SECRET" | caesar-cipher rot13`

**Punti extra:**
Supporta ROT47, che sposta tutti i caratteri ASCII stampabili, non solo le lettere.

### Sfida 5: aggiungere una classificazione basata su dizionario

**Cosa costruire:**
Oltre alla sola analisi delle frequenze, verifica se il testo decifrato contiene parole inglesi valide prese da una word list. Classifica i candidati in base al numero di parole riconosciute.

**Perché è utile:**
L'analisi delle frequenze può essere ingannata da testi brevi o gergo tecnico. Il controllo con dizionario è più robusto per i messaggi brevi.

**Cosa imparerai:**
- Lavorare con le word list (usa `/usr/share/dict/words` su Unix o scaricane una)
- Combinare più metodi di punteggio
- Ottimizzazione delle prestazioni (caricamento e ricerca nelle word list)

**Approccio all'implementazione:**

1. **Carica la word list** in un set per lookup O(1)
   - Crea `dictionary.py` con una funzione `load_words()`
   - Memorizzala come set per controlli `word in dictionary` veloci

2. **Assegna il punteggio in base alle parole trovate**
   - Dividi il testo decifrato in parole
   - Conta quante sono nel dizionario
   - Normalizza per il totale delle parole per ottenere una percentuale

3. **Combina con il punteggio delle frequenze**
   - Pondera il punteggio del dizionario e quello delle frequenze
   - Regola i pesi: magari 70% dizionario, 30% frequenze

**Suggerimenti:**
- Scarica una word list: `curl https://github.com/dwyl/english-words/raw/master/words_alpha.txt > words.txt`
- Normalizza maiuscole/minuscole: `word.lower() in dictionary`
- Gestisci la punteggiatura: `text.replace('!', '').replace('?', '').split()`

### Sfida 6: violare cifrari in più lingue

**Cosa costruire:**
Supporta il cracking di cifrari in lingue diverse dall'inglese caricando tabelle di frequenza differenti. Aggiungi il supporto per spagnolo, francese e tedesco.

**Applicazione nel mondo reale:**
L'analisi delle frequenze è universale. Ogni lingua ha schemi di lettere caratteristici. Lo spagnolo usa la Ñ, il tedesco usa Ä/Ö/Ü, il francese ha gli accenti.

**Cosa imparerai:**
- Come le distribuzioni di frequenza differiscono tra le lingue
- Gestione di Unicode in Python
- Progettare il supporto multilingua

**Implementazione:**

1. **Aggiungi le tabelle di frequenza** a `constants.py`
   - Cerca online le frequenze delle lettere in spagnolo
   - Memorizzale in dict come `SPANISH_LETTER_FREQUENCIES`

2. **Aggiorna l'analyzer** perché accetti un parametro di lingua
```python
   analyzer = FrequencyAnalyzer(language='spanish')
```

3. **Supporto CLI**
```bash
   caesar-cipher crack "SADDW FW UYMJYJ" --language spanish
```

**Insidie:**
- Alfabeti estesi (lo spagnolo ha 27 lettere con la Ñ)
- Distinzione tra maiuscole e minuscole per i caratteri accentati
- Normalizzi 'É' in 'E' o le tratti separatamente?

## Sfide avanzate

### Sfida 7: costruire un'implementazione del cifrario di Vigenère

**Cosa costruire:**
Un cifrario polialfabetico che usa una parola chiave per determinare shift diversi per posizioni diverse. Molto più difficile da violare di Cesare.

**Perché è difficile:**
Il cifrario di Vigenère fu chiamato per secoli "le chiffre indéchiffrable" (il cifrario indecifrabile). Violarlo richiede prima di trovare la lunghezza della chiave, poi l'analisi delle frequenze su ogni posizione.

**Cosa imparerai:**
- Sostituzione polialfabetica
- Esame di Kasiski per trovare la lunghezza della chiave
- Test statistico dell'indice di coincidenza
- Crittoanalisi in più fasi

**Modifiche architetturali necessarie:**
```
Add to cipher layer:
┌─────────────────────┐
│  VigenereCipher     │
│  - repeating key    │
│  - position logic   │
└─────────────────────┘

Add to analysis layer:
┌─────────────────────┐
│  VigenereAnalyzer   │
│  - Kasiski method   │
│  - IC calculation   │
└─────────────────────┘
```

**Passi di implementazione:**

1. **Fase di ricerca**
   - Documentati sull'algoritmo del cifrario di Vigenère
   - Comprendi l'esame di Kasiski
   - Studia la formula dell'indice di coincidenza

2. **Fase di design**
   - `VigenereCipher` dovrebbe ereditare da `CaesarCipher`? Probabilmente no, è abbastanza diverso.
   - Come gestire la ripetizione della chiave quando è più corta del testo?
   - Memorizzare la chiave come stringa o come lista di shift?

3. **Fase di implementazione**
   - Inizia dalla cifratura: ripeti la chiave fino alla lunghezza del testo
   - Aggiungi la decifratura: uguale, ma sottraendo gli shift
   - Aggiungi l'esame di Kasiski: trova le sequenze ripetute, calcola l'MCD delle distanze

4. **Fase di test**
   - Unit test della cifratura con varie lunghezze di chiave
   - Testa con ciphertext noti (sfide su Vigenère online)
   - Benchmark: quanto ci vuole per violare un messaggio di 100 caratteri?

**Insidie:**
- La chiave "CAT" significa shift di [2, 0, 19] (C=2, A=0, T=19)
- I caratteri non alfabetici non dovrebbero far avanzare la posizione nella chiave
- Il cracking richiede almeno 100-200 caratteri di ciphertext

**Risorse:**
- Wikipedia: Vigenère cipher - Buona panoramica dell'algoritmo
- Paper "Breaking the Vigenère Cipher" - Metodi di crittoanalisi dettagliati

### Sfida 8: implementare la visualizzazione dell'analisi delle frequenze

**Cosa costruire:**
Un tool web che mostri grafici delle frequenze in tempo reale mentre digiti il ciphertext. Usa Flask per il backend e Chart.js per la visualizzazione.

**Tempo stimato:**
1-2 giorni, incluso imparare le cose di frontend.

**Prerequisiti:**
Dovresti aver completato prima la sfida sulla visualizzazione delle statistiche. Questa si basa su quel concetto.

**Cosa imparerai:**
- Costruire interfacce web per tool crittografici
- Visualizzazione dei dati in tempo reale
- Collegare l'analisi in Python a grafici JavaScript

**Pianificare questa funzionalità:**

Prima di scrivere codice, rifletti su:
- Come invia il testo il frontend al backend? (richiesta POST con JSON)
- Quanto velocemente riesci a calcolare le frequenze per 10kb di testo? (Dovrebbe essere istantaneo)
- Ti servono i websocket o basta HTTP? (Per questo HTTP va bene)

**Architettura di alto livello:**
```
┌──────────────┐         ┌──────────────┐
│   Browser    │◄────────┤   Flask      │
│  Chart.js    │  JSON   │  analyzer.py │
└──────────────┘         └──────────────┘
      ▲                          │
      │                          │
      └──────────┬───────────────┘
            Frequencies
```

**Fasi di implementazione:**

**Fase 1: backend Flask** (2-3 ore)
- Crea `web.py` con un'app Flask
- Aggiungi un endpoint `/analyze` che riceve il testo e restituisce le frequenze
- Riutilizza `FrequencyAnalyzer` dal codice esistente

**Fase 2: frontend** (3-4 ore)
- Pagina HTML con una textarea per il ciphertext
- JavaScript per inviare il testo al keyup
- Grafico a barre Chart.js per mostrare le frequenze

**Fase 3: aggiungere la decifratura di Cesare** (2-3 ore)
- Pulsante per violare il ciphertext
- Mostrare tutti i 26 candidati con i punteggi
- Evidenziare la corrispondenza migliore

**Fase 4: rifinitura** (2-3 ore)
- Aggiungere indicatori di caricamento
- Mostrare il conteggio dei caratteri
- Interruttore per la dark mode

**Strategia di test:**
- Test manuali: digita vari ciphertext, verifica che i grafici si aggiornino
- Prova con testo molto lungo (10kb+) per assicurarti che non ci siano rallentamenti
- Testa su browser diversi

**Sfide note:**
1. **Prestazioni del ridisegno del grafico**
   - Problema: ricreare il grafico a ogni tasto premuto è lento
   - Suggerimento: aggiorna i dati del grafico invece di distruggerlo e ricrearlo

2. **Gestione dell'input vuoto**
   - Problema: il testo vuoto causa una divisione per zero
   - Suggerimento: restituisci un array vuoto quando il testo è vuoto

**Criteri di successo:**
La tua implementazione dovrebbe:
- [ ] Aggiornare il grafico delle frequenze in tempo reale mentre digiti
- [ ] Mostrare tutti i 26 candidati del crack con i punteggi
- [ ] Gestire senza problemi input da 10.000 caratteri
- [ ] Funzionare sui browser mobile
- [ ] Visualizzare i risultati entro 100ms dall'inserimento del testo

## Mescola e combina

Combina le funzionalità per progetti più grandi:

**Idea di progetto 1: tool multi-cipher**
- Combina la Sfida 7 (Vigenère) + la Sfida 6 (multilingua)
- Aggiungi il cifrario Playfair
- Risultato: un coltellino svizzero per la crittografia classica

**Idea di progetto 2: suite di crittoanalisi**
- Combina la Sfida 5 (dizionario) + la Sfida 4 (ROT13) + la Sfida 8 (visualizzazione)
- Aggiungi la decifratura automatica (prova tutti i metodi)
- Risultato: un tool che prende qualsiasi ciphertext e capisce quale cipher è stato usato

## Sfide di integrazione con il mondo reale

### Integrazione con le sfide di cifratura online

**L'obiettivo:**
Rendere il tool in grado di recuperare ciphertext da CryptoPals o da altre sfide CTF online e violarli automaticamente.

**Cosa ti servirà:**
- Un client HTTP (libreria requests)
- Parsing di risposte HTML o JSON
- Gestione dei rate limit

**Piano di implementazione:**
1. Aggiungi l'opzione `--url` al comando crack
2. Recupera il ciphertext dall'URL
3. Esegui il crack e invia indietro la risposta

**Attenzione a:**
- CAPTCHA sui siti delle sfide
- Codifiche diverse (base64, hex)
- Throttling dopo troppe richieste

### Deploy come bot Telegram

**L'obiettivo:**
Permettere agli utenti di cifrare/decifrare messaggi tramite chat Telegram.

**Cosa imparerai:**
- Costruire chatbot
- Gestione stateless delle conversazioni
- Integrazione con API

**Passi:**
1. Registra il bot con BotFather su Telegram
2. Usa la libreria python-telegram-bot
3. Fai il parsing dei comandi: `/encrypt <text> <key>`
4. Restituisci risultati formattati

**Checklist per la produzione:**
- [ ] Gestire gli errori con eleganza (mostrare messaggi comprensibili all'utente)
- [ ] Rate limit per utente (prevenire lo spam)
- [ ] Fare logging dell'utilizzo per il debug
- [ ] Fare il deploy su un server (Heroku, AWS Lambda)

## Sfide sulle prestazioni

### Sfida: gestire file da 1MB istantaneamente

**L'obiettivo:**
Far funzionare il comando crack su file enormi senza ritardi percepibili.

**Collo di bottiglia attuale:**
L'analisi delle frequenze viene eseguita sul testo completo 26 volte. Per 1MB, sono 26MB di elaborazione.

**Approcci di ottimizzazione:**

**Approccio 1: campionare invece di analizzare tutto**
- Come: analizza solo i primi 10.000 caratteri
- Guadagno: velocità 100 volte superiore sui file grandi
- Trade-off: meno accurato se l'inizio non è rappresentativo

**Approccio 2: elaborazione parallela**
- Come: usa il multiprocessing per testare tutti i 26 shift simultaneamente
- Guadagno: accelerazione quasi lineare con i core della CPU
- Trade-off: l'overhead lo rende più lento sui file piccoli

**Approccio 3: ottimizzare il calcolo del chi-quadrato**
- Come: metti in cache le frequenze attese, usa numpy per i calcoli
- Guadagno: velocità 2-3 volte superiore
- Trade-off: aggiunge la dipendenza da numpy

**Fai un benchmark:**
```bash
# Generate large file
python -c "import random, string; print(''.join(random.choices(string.ascii_uppercase, k=1000000)))" > large.txt

# Time it
time caesar-cipher crack --input-file large.txt
```

Metriche obiettivo:
- File da 1MB: meno di 1 secondo
- File da 10MB: meno di 10 secondi

### Sfida: ridurre l'uso di memoria

**L'obiettivo:**
Violare file enormi senza caricarli interamente in memoria.

**Prima fai il profiling:**
```bash
python -m memory_profiler main.py crack --input-file huge.txt
```

**Aree comuni di ottimizzazione:**
- Lettura del file in streaming invece di `read_text()`
- Generator expression invece di liste
- Non memorizzare tutte le 26 decifrature complete, solo le frequenze delle lettere

## Sfide di sicurezza

### Sfida: aggiungere HMAC per l'autenticazione dei messaggi

**Cosa implementare:**
Genera un HMAC (hash con chiave) insieme al ciphertext, così che i destinatari possano verificare che i messaggi non siano stati manomessi.

**Threat model:**
Questo protegge da:
- Modifica del messaggio (un attaccante che altera il ciphertext)
- Contraffazione (un attaccante che crea messaggi falsi)

**Implementazione:**
```python
import hmac
import hashlib

def encrypt_and_mac(plaintext: str, key: int, mac_key: bytes) -> tuple[str, str]:
    cipher = CaesarCipher(key=key)
    ciphertext = cipher.encrypt(plaintext)
    mac = hmac.new(mac_key, ciphertext.encode(), hashlib.sha256).hexdigest()
    return ciphertext, mac
```

**Testare la sicurezza:**
- Prova a modificare il ciphertext senza aggiornare il MAC
- Tenta di contraffare il MAC con la chiave sbagliata
- Verifica che i messaggi legittimi superino la validazione

### Sfida: implementare la protezione dai timing attack

**L'obiettivo:**
Fare in modo che la decifratura impieghi un tempo costante indipendentemente dal fatto che la chiave sia corretta. Impedisce agli attaccanti di usare i tempi per indovinare le chiavi.

**Threat model:**
Attualmente, le chiavi errate potrebbero fallire più velocemente di quelle corrette (uscita anticipata nella validazione). Un attaccante che misura i tempi di risposta potrebbe sfruttarlo.

**Rimedio:**
- Decifra sempre completamente anche se sai che è sbagliato
- Aggiungi un ritardo casuale per normalizzare i tempi
- Usa `hmac.compare_digest()` per il confronto dei MAC (tempo costante)

## Idee per contribuire

Hai completato una sfida? Condividila:

1. **Fai il fork del repo**
2. **Implementa la tua estensione** in un branch come `feature/vigenere-cipher`
3. **Documentala** - Aggiungi alla cartella learn una spiegazione di come funziona la tua estensione
4. **Invia una PR** con:
   - La tua implementazione
   - Test che coprono i casi limite
   - Documentazione in learn/
   - Esempio d'uso nel README

Le estensioni valide potrebbero essere integrate nel progetto principale.

## Mettiti ancora più alla prova

### Costruisci qualcosa di nuovo

Usa i concetti che hai imparato qui per costruire:
- Cifrario Playfair - Sostituzione di digrammi usando un quadrato chiave 5x5
- Cifrario Bifid - Combina sostituzione e trasposizione
- Cifrario Four-square - Usa quattro matrici 5x5
- Cifrario di Hill - Cifrario polialfabetico basato sulla moltiplicazione di matrici

### Studia implementazioni reali

Confronta la tua implementazione con tool di produzione:
- CyberChef - Come strutturano il supporto multi-cipher?
- John the Ripper - Come ottimizzano il brute force?
- Cryptool - Come presentano i contenuti didattici?

Leggi il loro codice, comprendi i loro trade-off, ruba le loro idee migliori.

### Scrivine

Documenta la tua estensione:
- Post sul blog: "How I Built a Vigenère Cracker"
- Tutorial: "Breaking Classical Ciphers with Python"
- Confronto: "Caesar vs Vigenère vs Enigma: Complexity Analysis"

Insegnare agli altri è il modo migliore per verificare di aver capito.

## Chiedere aiuto

Bloccato su una sfida?

1. **Fai debug in modo sistematico**
   - Cosa ti aspettavi? "L'analisi delle frequenze dovrebbe classificare 'HELLO' al primo posto"
   - Cosa è successo davvero? "Ha classificato più in alto un testo senza senso"
   - Il caso di test più piccolo? "Input: 'KHOOR', shift atteso: 3, shift ottenuto: 7"

2. **Leggi il codice esistente**
   - L'analyzer usa il chi-quadrato: come funziona?
   - Guarda `test_analyzer.py`: quali casi vengono testati?

3. **Cerca problemi simili**
   - Google: "python frequency analysis not working short text"
   - StackOverflow: tag [cryptography] [python]

4. **Chiedi aiuto**
   - Pubblica nelle GitHub discussions
   - Includi: cosa hai provato, cosa è successo, cosa ti aspettavi
   - Mostra il codice: "Ecco la mia modifica a _shift_char(), non fa correttamente il ritorno all'inizio dell'alfabeto"

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

- [ ] Sfida facile 1: variante autokey
- [ ] Sfida facile 2: alfabeti personalizzati
- [ ] Sfida facile 3: visualizzazione delle statistiche
- [ ] Sfida intermedia 4: modalità ROT13
- [ ] Sfida intermedia 5: classificazione con dizionario
- [ ] Sfida intermedia 6: multilingua
- [ ] Sfida avanzata 7: cifrario di Vigenère
- [ ] Sfida avanzata 8: visualizzazione web

Le hai completate tutte? Hai padroneggiato la crittografia classica. È il momento di imparare la crittografia moderna (AES, RSA, curve ellittiche) o di contribuire a questo progetto con le tue estensioni.
