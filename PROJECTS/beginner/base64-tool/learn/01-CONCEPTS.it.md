# Concetti

## Encoding vs. cifratura

Questo è in assoluto il concetto più importante di questo progetto, e confondere le due cose causa vulnerabilità reali.

L'**encoding** trasforma i dati in una rappresentazione diversa. Chiunque può invertirlo. Non c'è nessuna chiave segreta. L'encoding Base64 è "sicuro" quanto scrivere qualcosa in alfabeto farfallino.

La **cifratura** trasforma i dati usando una chiave segreta. Senza la chiave, non puoi riottenere l'originale. AES, RSA, ChaCha20. Questi sono algoritmi di cifratura.

CWE-261 (Weak Encoding for Password) esiste proprio perché gli sviluppatori memorizzano le password con una "cifratura" base64 invece di un vero hashing. Nel 2018, una telecamera D-Link (CVE-2017-8417) memorizzava le credenziali di amministratore in base64 sul dispositivo. Chiunque avesse accesso alla rete poteva decodificarle all'istante.

La regola: se puoi invertirlo senza una password o una chiave, è encoding. L'encoding serve alla compatibilità, non alla sicurezza.

## Come funziona Base64

Base64 converte dati binari in 64 caratteri ASCII stampabili. L'alfabeto è `A-Z`, `a-z`, `0-9`, `+`, `/`, con `=` per il padding.

### Il processo di encoding

1. Prendi i byte in input
2. Leggili come un flusso di bit
3. Dividili in gruppi da 6 bit (non 8)
4. Mappa ogni valore a 6 bit (0-63) su un carattere dell'alfabeto

Perché 6 bit? Perché 2^6 = 64, il che dà esattamente 64 valori possibili per carattere.

```
Input:    "Hi"
Bytes:    0x48 0x69
Binary:   01001000 01101001

Split into 6-bit groups:
010010 | 000110 | 1001xx

Pad the last group with zeros:
010010 | 000110 | 100100

Map to alphabet:
18 → S    6 → G    36 → k

Add padding (input was 2 bytes, need 1 pad):
Result: "SGk="
```

Tre byte in input producono esattamente quattro caratteri in output. Quando l'input non è divisibile per 3, ottieni il padding `=`. Un byte avanzato produce `==`, due byte avanzati producono `=`.

### La variante Base64URL

Il base64 standard usa `+` e `/`, che sono caratteri speciali negli URL. Base64URL (RFC 4648 Sezione 5) li sostituisce:
- `+` diventa `-`
- `/` diventa `_`

I JWT usano base64url. Così come molti token delle API. Se vedi `-` o `_` in qualcosa che sembra base64, probabilmente è base64url.

### Overhead di dimensione

Base64 converte 3 byte in 4 caratteri. Si tratta di un aumento di dimensione del 33%. Un file da 1 MB diventa circa 1,33 MB quando è codificato in base64. Questo conta per cose come incorporare immagini nell'HTML o inviare allegati via email (che è esattamente il motivo per cui il base64 è stato inventato per MIME).

## Come funziona Base32

Stessa idea del base64, ma usa solo 32 caratteri: `A-Z` e `2-7`. Gruppi da 5 bit invece che da 6.

Perché useresti base32 invece di base64? In ambienti dove la distinzione tra maiuscole e minuscole è un problema. I nomi DNS non distinguono tra maiuscole e minuscole, quindi il base64 lì non funziona. I token TOTP (Google Authenticator) usano base32 per il segreto condiviso perché le persone potrebbero confondere maiuscole e minuscole quando digitano una chiave.

Base32 produce un output più lungo rispetto a base64 (60% di overhead contro 33%). Il compromesso è che meno caratteri possibili significano meno problemi di ambiguità.

## Encoding esadecimale

L'hex rappresenta ogni byte come due caratteri tra `0-9` e `a-f`. È una mappatura diretta da byte a testo.

```
Input:    "Hi"
Bytes:    0x48 0x69
Hex:      4869
```

Overhead di dimensione del 100% (ogni byte diventa due caratteri), ma è l'encoding più semplice e universalmente leggibile. Vedi l'hex in:
- Indirizzi MAC: `aa:bb:cc:dd:ee:ff`
- Codici colore: `#FF5733`
- Packet capture e hex dump
- Firme hash dei malware: `d41d8cd98f00b204e9800998ecf8427e`
- Analisi di file binari

## URL encoding (percent encoding)

Gli URL hanno caratteri riservati (`?`, `&`, `=`, `/`, `#`, ecc.) che hanno un significato speciale. L'URL encoding sostituisce i caratteri non sicuri con `%XX`, dove `XX` è il valore esadecimale.

```
Input:    "hello world&key=val"
Encoded:  "hello%20world%26key%3Dval"
```

Lo spazio è `%20` nell'URL encoding standard ma `+` nel form encoding (`application/x-www-form-urlencoded`). La distinzione conta nel security testing delle applicazioni web.

## RFC 4648

RFC 4648 è lo standard di riferimento per l'encoding Base16, Base32 e Base64. Punti chiave:

- Definisce gli alfabeti esatti e le regole di padding
- Specifica quando i decoder DEVONO (MUST) rifiutare e quando POSSONO (MAY) accettare input malformato
- Il padding del Base64 è `=`, e DEVE (MUST) comparire solo alla fine
- I decoder che accettano caratteri esterni all'alfabeto creano problemi di sicurezza (diventano possibili attacchi padding oracle)

Il modulo `base64` di Python segue RFC 4648. Il parametro `validate=True` in `b64decode` impone una conformità rigorosa, rifiutando qualsiasi carattere esterno all'alfabeto. Questo progetto usa un decoding rigoroso.

## L'encoding negli attacchi

### Double encoding (bypass dei WAF)

I Web Application Firewall (WAF) controllano i parametri delle richieste alla ricerca di contenuto malevolo. Se applichi l'URL encoding a un payload, il WAF lo decodifica una volta e lo controlla. Ma cosa succede se lo codifichi due volte?

```
Payload:    <script>alert(1)</script>
Single:     %3Cscript%3Ealert(1)%3C/script%3E    ← WAF catches this
Double:     %253Cscript%253Ealert(1)%253C/script%253E  ← WAF decodes once,
            sees %3C (looks harmless), passes it through
```

Il web server decodifica di nuovo, e il browser riceve il payload XSS originale. Questa è la categoria A05:2025 Injection di OWASP. Il double encoding funziona ancora contro WAF configurati male nel 2026.

### Offuscamento multi-layer (malware)

Il malware DARKGATE (venduto attivamente sui forum del dark web) codifica la propria configurazione e l'output del keylogger usando un alfabeto base64 personalizzato e non standard. Le versioni più vecchie usavano un alfabeto personalizzato hardcoded. La versione 5.2.3+ rende casuale l'alfabeto partendo da un seed basato su un hardware ID.

I ricercatori di Kroll hanno trovato una debolezza: l'hardware ID è un hash MD5 ASCII da 32 byte, e DARKGATE somma questi byte per ottenere un seed. La somma ha un'entropia limitata, il che rende fattibile il recupero dell'alfabeto personalizzato tramite brute force.

Ecco perché capire l'encoding è importante per la sicurezza: i malware reali usano trucchi di encoding reali, e gli analisti devono decodificarli.

### Esfiltrazione di dati

Gli attaccanti esfiltrano dati codificandoli e incorporandoli in traffico apparentemente normale:
- Dati base64 in query di record DNS TXT
- Payload codificati in hex negli header HTTP
- Dati in URL encoding nei parametri di query che sembrano tracciamento analytics

Tool come questo aiutano i difensori a individuare e decodificare questi pattern.

## Rilevamento dell'encoding

Come fai a capire che encoding è qualcosa? Pattern matching ed euristiche.

**Segnali di Base64:**
- Caratteri di `A-Za-z0-9+/=`
- Lunghezza divisibile per 4
- Termina con 0, 1 o 2 caratteri `=`
- Maiuscole e minuscole mescolate (lettere maiuscole E minuscole)

**Segnali di Base32:**
- Solo maiuscole `A-Z` e cifre `2-7`
- Lunghezza divisibile per 8
- Padding con 0, 1, 3, 4 o 6 caratteri `=`

**Segnali di hex:**
- Solo `0-9` e `a-f` (oppure `A-F`)
- Lunghezza pari
- Maiuscole/minuscole coerenti (tutto minuscolo o tutto maiuscolo)
- A volte ha separatori (`:`, `-`, spazi)

**Segnali di URL encoding:**
- Contiene sequenze `%XX` in cui XX è un hex valido

La parte complicata: alcune stringhe corrispondono a più formati. La stringa hex `CAFE` è anche base32 valido (se completata con padding fino a 8 caratteri) e base64 (se la lunghezza torna). Un buon rilevamento usa un punteggio di confidenza, non un sì/no binario. Questo progetto assegna a ogni formato una confidenza da 0.0 a 1.0 e li ordina.

## Verifica la tua comprensione

1. Perché l'output base64 ha sempre una lunghezza divisibile per 4?
2. Uno sviluppatore memorizza le API key in base64 in un file di configurazione e la chiama "configurazione cifrata". Cosa c'è che non va?
3. Vedi la stringa `JBSWY3DPEBLW64TMMQ======`. Che encoding è e come fai a capirlo?
4. Un alert dell'IDS mostra `%253Cscript%253E` in un parametro dell'URL. Cos'è successo?
5. Perché un malware dovrebbe usare un alfabeto base64 non standard invece di quello di RFC 4648?
