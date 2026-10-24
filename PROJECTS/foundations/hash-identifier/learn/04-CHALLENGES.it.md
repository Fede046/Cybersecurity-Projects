# Sfide

Hai letto il codice. Sai cosa fa ogni riga. Ora l'unico modo per far diventare davvero tua questa conoscenza è *modificare* il codice. Questa pagina è una scala di estensioni, dalla più facile alla più difficile. Non saltare i gradini — ognuno insegna una cosa che il successivo dà per scontata.

Per ogni sfida: **scrivi prima il test.** Poi fai passare il test. È il ritmo che gli sviluppatori professionisti usano davvero. Il file di test ti mostra già il pattern — copia una delle funzioni `test_*` esistenti, cambia l'input e l'output atteso, guardalo fallire, poi modifica il codice finché non passa.

## Livello 1 — prendere confidenza

### Sfida 1.1: aggiungere una nuova regola di prefisso

`PREFIX_RULES` oggi ha ~25 voci. Ne esistono decine di altre. Scegline una dalla lista qui sotto e aggiungila:

| Prefisso     | Algoritmo          | Da dove proviene                          |
| ------------ | ------------------ | ----------------------------------------- |
| `$pbkdf2$`   | PBKDF2-SHA1 (Atlassian) | Vecchi hash Atlassian / Jira         |
| `$ml$`       | macOS / iCloud Keychain | PBKDF2-SHA512 di Apple               |
| `{x-pbkdf2}` | PBKDF2 (alcuni Atlassian)  | Wrapper in stile LDAP             |
| `$sha1$`     | sha1crypt          | Una rara variante di crypt(3)             |
| `$md5,`      | Solaris MD5 crypt  | Nota la virgola invece di `$` — fai attenzione |

**Passi:**

1. Apri `hash_identifier.py`. Aggiungi una riga a `PREFIX_RULES`.
2. Apri `test_hash_identifier.py`. Copia un test di prefisso esistente (ad es. `test_argon2id_prefix_is_recognized`). Rinominalo. Sostituisci la stringa di input con un esempio del tuo nuovo prefisso. Aggiorna l'asserzione.
3. Esegui `just test`. Il nuovo test dovrebbe passare. Anche il meta-test `test_every_prefix_rule_is_recognized_with_high_confidence` dovrebbe continuare a passare.
4. Esegui il tool sul tuo nuovo input: `just run -- '$pbkdf2$...'`. Conferma che compaia il nuovo algoritmo.

**Cosa impari:** il design guidato dalle tabelle ripaga — hai scritto zero logica nuova, solo dati. Questo è l'obiettivo.

### Sfida 1.2: aggiungere una lunghezza a HEX_LENGTH_RULES

Al momento non c'è una regola per 24 caratteri hex (96 bit). Quella lunghezza è rara, ma `Tiger-128` e alcuni vecchi hash custom la producono.

1. Aggiungi `24: ["Tiger-128"]` a `HEX_LENGTH_RULES`.
2. Scrivi un test (`test_tiger128_length_returns_tiger128`).
3. Esegui `just test`.

**Variante:** cosa dovrebbe succedere se qualcuno passa una stringa di 24 caratteri che *non* è hex? Il controllo esistente `_is_hex` dovrebbe gestirlo. Leggi il passo 3 di `identify()` e verificalo.

### Sfida 1.3: aggiungere una modalità di output `--json`

Al momento la CLI stampa solo una tabella colorata. Aggiungi un flag `--json` che stampi invece i candidati in JSON. Il JSON è ciò che ogni altro tool vorrà consumare — il tuo output diventa leggibile dalle macchine.

**Suggerimenti:**

- `argparse` supporta i flag booleani tramite `action="store_true"`. Aggiungi `parser.add_argument("--json", action="store_true", help="...")`.
- Il modulo `json` della libreria standard ha `json.dumps(data)`.
- `HashCandidate` è una dataclass, quindi `dataclasses.asdict(candidate)` la converte in un semplice dict che `json.dumps` può serializzare.
- Testalo: `just run -- --json 5f4d...` dovrebbe produrre in output un array JSON.

**Variante:** fai in modo che il JSON includa un campo `input` di primo livello con la stringa originale, così un tool a valle sa cosa è stato identificato. E fai il pretty-print con `indent=2`.

## Livello 2 — comportamenti davvero nuovi

### Sfida 2.1: leggere gli hash da un file o da stdin

Al momento il tool prende un hash per invocazione. I workflow reali hanno file con milioni di hash. Estendi la CLI in modo che possa prendere l'input da un file (`--file hashes.txt`) o dallo standard input quando non viene fornito alcun argomento posizionale (così `cat hashes.txt | hashid` funziona).

**Suggerimenti:**

- Rendi opzionale l'argomento posizionale `hash` con `nargs="?"`.
- Aggiungi `--file` con `type=argparse.FileType("r")`.
- Usa `sys.stdin.read()` (o itera `sys.stdin` riga per riga) quando mancano entrambi.
- L'output per l'input batch dovrebbe essere diverso — probabilmente una riga per input, non una tabella colorata per ciascuno. Decidi quale formato ha senso e documentalo.

**Variante:** cosa dovrebbe succedere se lo stesso input compare 1000 volte nel file? Dovresti rieseguire `identify()` ogni volta, o mettere in cache i risultati? Prova entrambe le soluzioni e misura con `just run` su un file di 1M di hash ripetuti. (Lezione più ampia: la cache è vantaggiosa solo quando la funzione è pura. La nostra lo è.)

### Sfida 2.2: aggiungere suggerimenti sulla modalità di hashcat

Hashcat assegna una modalità numerica a ogni algoritmo: 0 per MD5, 100 per SHA-1, 3200 per bcrypt, ecc. L'elenco completo è documentato su [hashcat.net/wiki/doku.php?id=example_hashes](https://hashcat.net/wiki/doku.php?id=example_hashes).

Estendi `HashCandidate` con un campo opzionale `hashcat_mode: int | None`. Quando costruisci un candidato, cerca la sua modalità (ti servirà un `dict[str, int]` che mappi nome dell'algoritmo → modalità) e compilala.

Poi stampa la modalità nella tabella e aggiorna il suggerimento "next step" alla fine di `main()` per proporre l'esatto comando hashcat:

```
Next step: hashcat -m 3200 -a 0 '$2b$12$EixZ...' wordlist.txt
```

**Suggerimenti:**

- I campi `Optional` di una dataclass hanno bisogno di valori di default: `hashcat_mode: int | None = None`.
- Il lookup della modalità è un'altra tabella di dati — mantieni il design data-driven.
- John the Ripper usa nomi diversi (ad es. `bcrypt`, `raw-md5`). Aggiungi anche quelli se ti senti generoso.

### Sfida 2.3: riconoscere più input "non hash"

Il passo 5 intercetta solo JWT e blob base64. Molte altre cose vengono incollate per sbaglio negli identificatori di hash. Aggiungi rilevatori per:

- **URL** — iniziano con `http://` o `https://`. Di' all'utente che è un URL.
- **Hex con prefisso `0x`** — indirizzi Ethereum, indirizzi di memoria. Dillo all'utente.
- **Base58** — usato dagli indirizzi Bitcoin e dagli hash IPFS. L'alfabeto è `123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz` (niente `0`, `O`, `I`, `l`).
- **Base32** — lettere maiuscole + cifre 2-7. Usato da alcuni indirizzi onion di Tor e dai secret TOTP.

Ognuno è un nuovo ramo nel passo 5, che restituisce un candidato "non è un hash" con confidenza LOW. Sii onesto con la confidenza — questi sono indizi di *forma*, non certezze.

## Livello 3 — estendere il modello

### Sfida 3.1: rilevamento di più hash in una stringa

Alcuni dump di breach contengono record *combinati* come `user:hash:salt`. I workflow reali spesso devono suddividerli nelle parti costituenti prima dell'identificazione. Aggiungi una modalità `--split` che:

1. Prende una stringa con campi separati da due punti.
2. Identifica ogni campo in modo indipendente.
3. Stampa una tabella che mostra quale campo è cosa (username, hash, salt, spazzatura).

**Suggerimenti:**

- Si tratta principalmente di euristiche. Un campo composto solo da lettere è probabilmente uno username; un campo che corrisponde a una regola di hash è probabilmente l'hash; un campo corto dall'aspetto casuale potrebbe essere un salt.
- Estrai la logica di classificazione dei campi da `identify()` — è un nuovo livello sopra di essa.
- Testa su righe dall'aspetto realistico come `alice:$2b$12$EixZ...`, `bob:5f4dcc3b5aa765d61d8327deb882cf99:salt123`.

### Sfida 3.2: ribilanciare la confidenza con pesi delle prove

Al momento `confidence` è un singolo valore: high, medium o low. Ma le prove sono più sfumate. Una stringa di 32 caratteri hex che *inoltre* non contiene caratteri oltre la 'f' è "molto probabilmente hex." Una stringa di 13 caratteri dal charset DES è "compatibile con DES a 13 caratteri", ma gli stessi caratteri potrebbero comparire in molte stringhe corte.

Sostituisci `confidence: Literal["high", "medium", "low"]` con `confidence_score: float` nell'intervallo 0.0–1.0. Calcola il punteggio da più pesi delle prove:

- corrispondenza sul prefisso: 0.95
- corrispondenza su forma speciale: 0.85
- corrispondenza sulla lunghezza (1° candidato): 0.55
- corrispondenza sulla lunghezza (N-esimo candidato): 0.55 / N
- corrispondenza sul charset: piccolo bonus additivo
- indizio "non è un hash": 0.30

Poi nella CLI, rimappa il punteggio a un colore (>0.8 verde, 0.5–0.8 giallo, <0.5 ciano) per la visualizzazione. L'utente vede ancora tre fasce; il modello interno diventa più ricco.

**Cosa impari:** come funzionano i sistemi di scoring sotto il cofano. È la stessa idea dei classificatori antispam bayesiani, del ranking di rilevanza nella ricerca e delle euristiche antivirus — combinare più segnali deboli in un unico punteggio numerico, poi suddividere in fasce per la visualizzazione.

### Sfida 3.3: suggerire la *difficoltà di crack* accanto all'algoritmo

Una volta che conosci l'algoritmo, sai implicitamente anche quanto è difficile craccare l'hash. MD5 si cracca a miliardi di tentativi al secondo su una GPU moderna; bcrypt a migliaia. Argon2id con parametri robusti potrebbe essere a centinaia.

Aggiungi un campo `crack_difficulty: Literal["trivial", "moderate", "hard", "very_hard"]` a `HashCandidate`, compilato da una tabella per algoritmo. Stampalo nella tabella di output.

**Variante:** per gli hash parametrizzati (cost factor di bcrypt, memoria/tempo di Argon2), estrai i parametri dalla stringa PHC. Un hash bcrypt con cost factor 4 è molto più debole di uno con cost factor 14. Fai in modo che il tuo output lo rifletta:

```
algorithm  difficulty  reason
─────────  ──────────  ──────────────────────────────────────────
bcrypt     moderate    cost=4 — much weaker than the default 12
bcrypt     hard        cost=12 — the modern default
bcrypt     very_hard   cost=14 — paranoid setting
```

## Livello 4 — renderlo reale

### Sfida 4.1: eseguire l'identificazione su un vero dump di breach

Il file di password di [HaveIBeenPwned](https://haveibeenpwned.com/Passwords) è un elenco distribuito pubblicamente di ~1 miliardo di hash SHA-1 di password trapelate, disponibile come torrent. (Usa la versione **SHA-1**, non la versione NTLM — e usala solo per analisi didattiche; non provare a "craccarlo" per alcuno scopo malevolo. Gli hash stessi sono pubblici.)

Esegui il tuo tool sulle prime 1000 righe. Conferma che li identifichi tutti come SHA-1 (40 caratteri hex). Misura il throughput: quanti hash al secondo riesce a elaborare il tuo tool? Dov'è il collo di bottiglia?

**Lezione più ampia:** il cervello impiega microsecondi; il collo di bottiglia è la stampa della CLI. Per elaborare milioni di hash salteresti `rich` e manderesti JSON in streaming su stdout. Quello è un programma diverso per un caso d'uso diverso.

### Sfida 4.2: confrontarlo con `hashid` e `name-that-hash`

Due tool esistenti fanno più o meno ciò che fa il nostro:

- [hashid](https://github.com/psypanda/hashID) — il classico, vecchio di ~10 anni, scritto in Python puro.
- [name-that-hash](https://github.com/HashPals/Name-That-Hash) — più recente, più completo, più aggressivo nel tirare a indovinare.

Eseguili tutti e tre sugli stessi input. Confronta:

- Quale rileva più formati?
- Quale ha il miglior tasso di falsi positivi (dice "sicuramente SHA-256" su spazzatura che non lo è)?
- Quale è il più veloce su un input batch?

Scrivi un resoconto delle tue scoperte. È esattamente il tipo di lavoro che i ricercatori di sicurezza fanno quando scelgono un tool per l'uso in produzione.

### Sfida 4.3: trasformarlo in un hook di `pre-commit`

`pre-commit` è un tool che esegue controlli prima che tu faccia `git commit`. A volte le persone fanno accidentalmente commit di password hash nei repo (pessima cosa). Costruisci un hook di `pre-commit` che esegua il tuo identificatore su ogni file modificato e rifiuti il commit se trova qualcosa che sembra un vero hash.

**Suggerimenti:**

- Leggi il [tutorial sugli hook di pre-commit](https://pre-commit.com/#new-hooks).
- Per ogni riga di ogni file modificato, esegui `identify(line)`. Se il candidato principale ha confidenza HIGH e non è un PHC generico o un "non è un hash", rifiuta.
- Exit code 1 = blocca il commit; exit code 0 = consentilo.
- Aggiungi commenti di allowlist come `# pragma: allow-hash` in modo che le fixture di test legittime non vengano bloccate.

Questo è il tipo di progetto "piccolo tool, impatto reale" che finisce nelle toolchain dei veri team di sicurezza.

## Livello 5 — rompere il modello

### Sfida 5.1: perché questo è un problema difficile?

Finora abbiamo fatto matching sulla *struttura*: prefisso, lunghezza, charset. Ma il modello ha dei limiti. Considera:

- Due algoritmi diversi che producono output della stessa lunghezza (MD5 vs NTLM a 32 caratteri hex). *Non possiamo* distinguerli dalla sola struttura.
- Un algoritmo il cui output è a volte maiuscolo e a volte minuscolo (librerie diverse producono maiuscole/minuscole diverse per lo stesso algoritmo).
- Uno SHA-256 troncato — qualcuno ha preso i primi 32 caratteri hex di un output SHA-256 e l'ha chiamato "hash." Lo identificheremmo come MD5.

Scrivi un breve documento (`docs/limitations.md`) che elenchi ogni caso in cui il tool *non può* distinguere due formati e spieghi perché. È il tipo di onestà che i buoni tool di sicurezza documentano fin da subito. Non fingere che il tool sia più capace di quanto sia.

### Sfida 5.2: identificazione probabilistica con un classificatore ML

L'approccio strutturale è interpretabile ma limitato. L'estremo opposto è addestrare un classificatore su hash etichettati e lasciare che impari la struttura al posto tuo.

Costruisci un piccolo esperimento:

1. Genera 100k campioni di addestramento etichettati: password note hashate con ciascun algoritmo. (Usa `hashlib`, `bcrypt`, `argon2-cffi` di Python, ecc.)
2. Addestra un classificatore semplice — `sklearn.linear_model.LogisticRegression` con feature basate su n-gram di caratteri funziona bene per questo.
3. Esegui il classificatore su campioni di test tenuti da parte. Misura l'accuratezza.
4. Confrontalo con il tuo identificatore basato su regole sullo stesso test set.

**Cosa imparerai:** probabilmente il sistema basato su regole vince sui formati comuni (ha già incorporati tutti i priori strutturali) e il modello ML probabilmente vince sui casi insoliti (coglie pattern che non avevi pensato di codificare). I tool del mondo reale combinano entrambi — prima le regole, ML per gli spareggi. È così che funzionano davvero i classificatori antispam, i web application firewall e i motori antivirus.

> **Non eseguire nessuna di queste sfide su dati reali di utenti senza permesso.** Le sfide presuppongono archivi di breach rilasciati pubblicamente, i tuoi dati di test o input di CTF. Identificare hash da dati su cui non hai diritti è un discorso diverso e al di fuori dello scopo di questo progetto.

## Dove andare dopo le sfide

Se hai completato il livello 4 o 5, ora dovresti avere una solida percezione dell'*intero ciclo*: leggere il codice → capire → modificare → testare → misurare → documentare. Quel ciclo è l'intero lavoro di un security engineer. L'argomento specifico conta poco — faresti lo stesso ciclo per un port scanner, un fuzzer, una regola SIEM o un vulnerability scanner.

Quando sei pronto per altro:

- **`PROJECTS/beginner/hash-cracker`** — il naturale fratello di questo tool. Hai imparato a identificare un hash; quel progetto ti insegna come craccarne uno.
- **`PROJECTS/foundations/http-headers-scanner`** — un altro progetto del tier foundations, Python in un unico file. I/O leggermente più impegnativo (fa una richiesta di rete) ma la stessa forma architetturale.
- **`PROJECTS/foundations/password-manager`** — il progetto foundations più difficile. Cifratura, key derivation e un vero vault su disco. Leggi prima [01-CONCEPTS.md](./01-CONCEPTS.it.md) per capire perché Argon2id è la scelta giusta per il passo di key derivation.
- **`PROJECTS/beginner/simple-port-scanner`** — una volta assorbiti i pattern qui, il port scanning è il naturale passo successivo. Socket, concorrenza e tempistiche entrano tutti in gioco.

Buona fortuna. Leggi tutto. Scrivi codice. Rompilo. Aggiustalo. È così che si impara.
