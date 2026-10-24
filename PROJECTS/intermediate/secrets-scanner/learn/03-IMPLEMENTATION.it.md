# Walkthrough dell'Implementazione

Questo documento percorre i principali file di codice di Portia. Per ogni sezione vedremo cosa fa il codice, perché è progettato in quel modo e a cosa fare attenzione.

## Shannon Entropy (`internal/rules/entropy.go`)

Il modulo di entropy risponde a una sola domanda: "Questa stringa è abbastanza casuale da essere un secret?"

**Funzione ShannonEntropy:**
La funzione prende una stringa e un charset (l'insieme di tutti i caratteri possibili). Conta quante volte compare ogni carattere, calcola la probabilità di ciascun carattere (conteggio / totale), poi somma `-p * log₂(p)` su tutti i caratteri.

Il parametro charset è importante per il calcolo. Se calcoli l'entropy usando solo i caratteri presenti nella stringa, ogni stringa con tutti i caratteri distinti ottiene la stessa entropy. Calcolandola rispetto al charset completo (ad esempio, tutti i 62 caratteri alfanumerici), il risultato riflette quanta parte dello spazio di casualità disponibile la stringa utilizza effettivamente.

**Funzione DetectCharset:**
Prima di calcolare l'entropy, Portia indovina il charset:
- Se tutti i caratteri sono `0-9a-f`, è esadecimale. Usa il charset esadecimale da 16 caratteri.
- Se tutti i caratteri sono `A-Za-z0-9+/=`, è base64. Usa il charset base64 da 64 caratteri.
- Altrimenti, usa come default il charset alfanumerico da 62 caratteri.

Questo conta perché una stringa esadecimale `deadbeefcafe` ha un'entropy diversa a seconda che la si valuti rispetto a 16 caratteri possibili (hex) o a 62 (alfanumerico). La valutazione hex è più generosa e la identifica correttamente come moderatamente casuale all'interno del suo charset.

**Soglie in pratica:**
- `password = "admin"` → entropy ~2,3 (sotto la maggior parte delle soglie, filtrata)
- `password = "xK9mP2vL5nQ8jR3t"` → entropy ~4,0 (sopra la soglia, segnalata)
- `AKIAIOSFODNN7EXAMPLE` → nessun controllo di entropy necessario (regola strutturale, il prefisso AKIA è sufficiente)

## Rule Registry (`internal/rules/registry.go`)

Il registry è una semplice mappa da ID di regola a struct di regola, con alcuni metodi chiave:

**MatchKeywords** è la funzione critica per le performance. Per ogni regola nel registry, controlla se una qualsiasi keyword della regola compare (case-insensitive) nella stringa di contenuto. Nel caso peggiore questo è O(regole * keyword * lunghezza_contenuto), ma in pratica `strings.Contains` con keyword brevi su chunk di lunghezza media è veloce.

Il valore restituito è uno slice di regole corrispondenti. Se un chunk contiene `password`, il registry restituisce tutte le regole che hanno `password` come keyword. Se il chunk contiene `AKIA`, restituisce le regole per le access key AWS.

**Le allowlist globali** sono definite in fondo a `registry.go`:
- `GlobalPathAllowlist` - pattern regex per i path da saltare (go.mod, package-lock.json, node_modules/, vendor/, estensioni binarie, JS minificato)
- `GlobalValueAllowlist` - pattern regex per i valori da ignorare (example, test, dummy, fake, placeholder, YOUR_API_KEY, xxxx..., TODO, CHANGEME)

Queste sono separate dalle allowlist per singola regola. Un'allowlist per regola (come `AKIAIOSFODNN7EXAMPLE` di AWS) si applica solo a quella specifica regola. Le allowlist globali si applicano a tutte le regole.

## Regole di Detection (`internal/rules/builtin.go`)

Ogni regola è una struct `types.Rule` con questi campi:

```go
type Rule struct {
    ID          string         // unique identifier like "aws-access-key-id"
    Description string         // human-readable "AWS Access Key ID"
    Severity    Severity       // SeverityCritical, SeverityHigh, etc.
    Keywords    []string       // fast pre-filter: ["AKIA", "ABIA", "ACCA", "ASIA"]
    Pattern     *regexp.Regexp // the actual detection regex
    SecretGroup int            // which capture group contains the secret (0=whole match)
    Entropy     *float64       // minimum entropy threshold (nil = no check)
    Allowlist   Allowlist      // per-rule path/value/stopword overrides
    SecretType  SecretType     // classification: APIKey, Token, Password, etc.
}
```

**Analisi della regola per le access key AWS:**

```go
{
    ID:          "aws-access-key-id",
    Description: "AWS Access Key ID",
    Severity:    types.SeverityCritical,
    Keywords:    []string{"AKIA", "ABIA", "ACCA", "ASIA"},
    Pattern:     regexp.MustCompile(`\b((?:AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16})\b`),
    SecretGroup: 1,
    SecretType:  types.SecretTypeAPIKey,
}
```

- **Keywords**: Quattro possibili prefissi. Se il chunk non contiene nessuna di queste quattro stringhe, salta completamente questa regola.
- **Pattern**: Word boundary `\b`, poi uno dei quattro prefissi, poi esattamente 16 caratteri alfanumerici maiuscoli, poi word boundary. L'intero match viene catturato nel gruppo 1.
- **SecretGroup**: 1 significa estrarre dal primo gruppo tra parentesi (l'intera chiave).
- **Nessuna soglia di entropy**: Le chiavi AWS hanno una struttura fissa, quindi la validazione dell'entropy non serve. Prefisso + lunghezza sono sufficienti.
- **Nessuna allowlist**: L'allowlist globale dei valori intercetta già `AKIAIOSFODNN7EXAMPLE`.

**Analisi della regola per la password generica:**

```go
{
    ID:          "generic-password",
    Description: "Password in Assignment",
    Severity:    types.SeverityHigh,
    Keywords:    []string{"password", "passwd", "pwd"},
    Pattern:     regexp.MustCompile(`(?i)(?:password|passwd|pwd)\s*[:=]\s*['"]([^'"]{8,})['"]`),
    SecretGroup: 1,
    Entropy:     ptr(3.5),
    SecretType:  types.SecretTypePassword,
}
```

- **Keywords**: Tre varianti di "password"
- **Pattern**: Match case-insensitive per password/passwd/pwd, seguito da `:` o `=`, spazi opzionali, poi una stringa tra apici di almeno 8 caratteri. Il gruppo 1 cattura solo il valore della password.
- **Soglia di entropy**: 3,5 bit. Questo filtra `password = "admin123"` (bassa entropy) mentre cattura `password = "xK9mP2vL5nQ8jR3t"` (alta entropy).
- La funzione helper `ptr()` crea un `*float64` a partire da un letterale, dato che Go non permette di prendere direttamente l'indirizzo di una costante.

## Directory Source (`internal/source/directory.go`)

La directory source percorre un filesystem e produce chunk:

**Callback di WalkDir**: Per ogni voce del file system:
1. Controlla la cancellazione del context (permette uno shutdown pulito)
2. Salta le directory note come non interessanti (`.git`, `node_modules`, `vendor`, `__pycache__`, `.venv`)
3. Controlla se il path relativo corrisponde a qualche pattern di esclusione
4. Salta le estensioni di file binari (`.png`, `.jpg`, `.exe`, `.zip`, ecc.)
5. Controlla la dimensione del file rispetto al limite massimo (default 1MB)
6. Se tutti i controlli passano, chiama `emitChunks`

**emitChunks**: Apre il file e lo legge riga per riga usando `bufio.Scanner`:
- Accumula le righe in uno `strings.Builder`
- Ogni 50 righe, invia il testo accumulato come `types.Chunk` con il path relativo del file e il numero della riga iniziale
- Dopo il ciclo, invia le righe rimanenti come chunk finale

**Perché chunk da 50 righe?** È un compromesso tra uso di memoria e accuratezza della detection. Chunk più grandi usano più memoria per worker. Chunk più piccoli potrebbero spezzare un secret multi-riga su due chunk. 50 righe sono un giusto compromesso pratico: la maggior parte dei secret sta su una sola riga, e 50 righe sono abbastanza poche da essere elaborate rapidamente.

**Funzione isExcluded**: Controlla due pattern:
- `filepath.Match` sul nome del file (solo il base name). Gestisce pattern come `*.env`
- `strings.Contains` sul path relativo completo. Gestisce pattern come `test/fixtures`

## Git Source (`internal/source/git.go`)

La git source usa go-git v5 per scansionare la cronologia del repository senza invocare il binario `git`.

**scanHistory**: Apre il repository con `git.PlainOpen`, poi:
1. Ottiene un iteratore dei commit filtrato per branch (se specificato)
2. Per ogni commit, controlla che la data sia successiva a `--since` ed entro `--depth`
3. Ottiene il tree del commit e percorre tutte le voci
4. Per ogni blob (file), legge il contenuto, controlla dimensione/esclusioni/estensioni binarie
5. Divide in chunk da 50 righe con i metadati del commit (SHA, autore, data)

**scanStaged**: Per la scansione pre-commit:
1. Apre il repository e legge l'index di git (l'area di staging)
2. Per ogni voce dell'index, legge il contenuto del blob dall'object store
3. Produce chunk solo per i file attualmente in staging

**readBlob**: Legge un oggetto blob di git in una stringa:
```go
func readBlob(obj *object.Blob) (string, error) {
    reader, err := obj.Reader()
    if err != nil {
        return "", err
    }
    defer reader.Close()
    data, err := io.ReadAll(reader)
    if err != nil {
        return "", err
    }
    return string(data), nil
}
```

Usa `io.ReadAll` anziché `strings.Builder.ReadFrom` perché `strings.Builder` non ha un metodo `ReadFrom`. Un errore comune quando si lavora con i blob di go-git.

## Detector (`internal/engine/detector.go`)

Il detector è il punto in cui le regole incontrano il contenuto:

**Flusso della funzione Detect:**
1. Chiama `registry.MatchKeywords(chunk.Content)` per ottenere solo le regole rilevanti
2. Se nessuna regola corrisponde per keyword, restituisce subito nil (fast path)
3. Divide il contenuto del chunk in righe
4. Per ogni regola corrispondente, per ogni riga:
   - Esegue `rule.Pattern.FindAllStringSubmatchIndex(line, -1)` per trovare tutti i match
   - Per ogni match, chiama `extractSecret` per ottenere il secret dal gruppo di cattura
   - Se la regola ha una soglia di entropy, calcola l'entropy e salta se è sotto la soglia
   - Chiama `FilterFinding` per i controlli sui falsi positivi
   - Se tutti i controlli passano, aggiunge ai finding

**Funzione extractSecret:**
```go
func extractSecret(line string, loc []int, group int) string {
    if group > 0 && len(loc) > group*2+1 {
        start := loc[group*2]
        end := loc[group*2+1]
        if start >= 0 && end >= 0 {
            return line[start:end]
        }
    }
    if len(loc) >= 2 {
        return line[loc[0]:loc[1]]
    }
    return ""
}
```

L'array `loc` restituito da `FindAllStringSubmatchIndex` contiene coppie di indici di inizio/fine per ogni gruppo di cattura. Il gruppo 0 è l'intero match (indici 0,1), il gruppo 1 è il primo gruppo tra parentesi (indici 2,3), e così via. Se il gruppo richiesto non esiste o ha indici negativi (cioè il gruppo non ha partecipato al match), si ricade sull'intero match.

## Filter (`internal/engine/filter.go`)

La filter chain è l'ultima difesa contro i falsi positivi:

**IsStopword**: La correzione fondamentale qui è stata passare dal matching per substring al matching esatto dopo lo split per delimitatori. L'implementazione originale controllava se una qualsiasi stopword fosse una substring del secret. Questo faceva sì che `AKIAIOSFODNN7EXAMPLE` corrispondesse perché "example" compariva come substring. La correzione:

```go
parts := strings.FieldsFunc(lower, func(r rune) bool {
    return r == '_' || r == '-' || r == '.' || r == '/'
})
for _, part := range parts {
    if _, ok := stopwords[part]; ok {
        return true
    }
}
```

Questo divide sui caratteri delimitatori comuni e controlla ogni parte in modo indipendente. `AKIAIOSFODNN7EXAMPLE` non viene diviso in "example" perché non c'è un delimitatore prima. Ma `module_controller_config` viene diviso in ["module", "controller", "config"], che sono tutte stopword.

**Orchestrazione di FilterFinding:**
```
IsPlaceholder(secret) → true = skip
IsTemplated(secret)   → true = skip
IsStopword(secret)    → true = skip
rule.Allowlist.Values  → match = skip
GlobalPathAllowlist    → match = skip
rule.Allowlist.Paths   → match = skip
All checks pass        → finding is real
```

Ogni livello è indipendente. Un finding deve essere intercettato da un solo livello per essere filtrato.

## Pipeline (`internal/engine/pipeline.go`)

La pipeline collega tutto con le primitive di concorrenza di Go:

**Setup:**
```go
chunks := make(chan types.Chunk, p.workers*4)
findingsCh := make(chan types.Finding, p.workers*4)
g, gctx := errgroup.WithContext(ctx)
```

Due channel con buffer: uno per i chunk (source → worker), uno per i finding (worker → collector). La dimensione del buffer è `workers * 4` per bilanciare il backpressure.

**Goroutine della source:** Esegue `src.Chunks(gctx, chunks)` e fa il defer di `close(chunks)`. Quando la source termina (o il context viene cancellato), il channel si chiude e i worker svuotano gli elementi rimanenti.

**Goroutine dei worker:** Ogni worker cicla sul channel dei chunk:
```go
for chunk := range chunks {
    if gctx.Err() != nil {
        return gctx.Err()
    }
    results := p.detector.Detect(chunk)
    for _, f := range results {
        findingsCh <- f
    }
}
```

I worker condividono un `sync.WaitGroup` separato dall'errgroup. Quando tutti i worker hanno finito, una goroutine chiude il channel dei finding.

**Goroutine del collector:** Semplice ciclo che raccoglie tutti i finding in uno slice:
```go
for f := range findingsCh {
    mu.Lock()
    allFindings = append(allFindings, f)
    mu.Unlock()
}
```

Il mutex non è strettamente necessario dato che c'è una sola goroutine collector, ma protegge da modifiche future e rende felice il data race detector.

**Deduplicazione:** Dopo che tutte le goroutine sono terminate, `dedup` rimuove i finding duplicati creando una chiave composta da `ruleID + "|" + filePath + "|" + secret + "|" + commitSHA`. Questo gestisce i casi in cui lo stesso secret compare in chunk sovrapposti o in più commit git.

## Client HIBP (`internal/hibp/client.go`)

Il client HIBP controlla i secret rispetto al database di breach di Troy Hunt:

**Hashing SHA-1:** Il secret viene sottoposto a hash con SHA-1 (sì, SHA-1 è crittograficamente rotto, ma HIBP lo usa come chiave di lookup, non per la sicurezza). L'hash viene convertito in maiuscolo e diviso: i primi 5 caratteri sono il prefisso, i restanti 35 il suffisso.

**Query con k-anonymity:** Invia il prefisso a `https://api.pwnedpasswords.com/range/{prefix}`. L'API restituisce tutti i suffissi di hash che condividono quel prefisso, insieme ai conteggi delle occorrenze:
```
0018A45C4D1DEF81644B54AB7F969B88D65:21
00D4F6E8FA6EECAD2A3AA415EEC418D38EC:2
```

Il client esegue il parsing di ogni riga, la divide su `:` e controlla se un qualsiasi suffisso corrisponde al nostro. In tal caso, il secret è stato trovato in un breach.

**Cache LRU:** Prima di effettuare una chiamata all'API, controlla la cache usando il prefisso di 5 caratteri come chiave. La cache LRU contiene 10.000 voci. Dato che ogni prefisso copre tutti i secret con quel prefisso, il caching è molto efficace quando si scansionano codebase grandi con secret simili.

**Circuit breaker:** Avvolge la chiamata HTTP in un `gobreaker.CircuitBreaker`. Impostazioni:
- Massimo di fallimenti consecutivi: 5
- Timeout (periodo di recupero): 60 secondi
- Dopo 5 fallimenti consecutivi, il circuito si apre e restituisce immediatamente un errore per tutte le chiamate successive. Dopo 60 secondi, entra nello stato half-open e lascia passare una richiesta per verificare se l'API è tornata disponibile.

**Campo baseURL:** Il client ha un campo `baseURL` che di default punta all'URL reale dell'API HIBP. Nei test viene sovrascritto per puntare a un `httptest.Server`. Questo evita la necessità di un mocking basato su interfacce e mantiene il codice semplice.

## Reporter (`internal/reporter/`)

**Terminal reporter** (`terminal.go`):
- Ordina i finding per severity (prima i CRITICAL)
- Colori: rosso per CRITICAL, rosso (non grassetto) per HIGH, giallo per MEDIUM, ciano per LOW
- Maschera i secret: mostra i primi 4-6 e gli ultimi 4-6 caratteri con degli asterischi in mezzo
- Tronca gli SHA dei commit a 8 caratteri per leggibilità
- Mostra i valori di entropy quando presenti
- Mostra lo stato di breach HIBP e il conteggio quando controllato

**JSON reporter** (`json.go`):
- Produce un oggetto JSON con un array `findings` e un oggetto `summary`
- Ogni finding ha: rule_id, description, severity, secret (mascherato), entropy, file, line, commit, author, hibp_status, breach_count
- Il summary ha: total_findings, total_rules, duration, hibp_checked, hibp_breached

**SARIF reporter** (`sarif.go`):
- Produce JSON conforme a SARIF v2.1.0
- Mappa la severity di Portia ai livelli SARIF: CRITICAL/HIGH = "error", MEDIUM = "warning", LOW = "note"
- Ogni finding diventa un `result` SARIF con `ruleId`, `message`, `level` e `locations`
- Le definizioni delle regole sono incluse nell'array `tool.driver.rules`
- Le proprietà personalizzate (entropy, stato HIBP, secret mascherato) vanno in `result.properties`
