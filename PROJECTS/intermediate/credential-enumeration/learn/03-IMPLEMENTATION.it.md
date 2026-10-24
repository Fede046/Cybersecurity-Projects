# 03-IMPLEMENTATION.md

# Implementazione

Questo documento ripercorre il codice effettivo: come funziona ciascun componente, perché è strutturato in quel modo e quali pattern cercare quando leggi o estendi la codebase.

## Struttura dei file

```
src/
├── harvester.nim          # Entry point — CLI parsing, main orchestration
├── runner.nim             # Collector dispatch — maps categories to collectors
├── types.nim              # Type definitions — Finding, Report, Severity, etc
├── config.nim             # Constants — every path, pattern, threshold, color
├── collectors/
│   ├── base.nim           # Shared utils — file ops, permissions, factories
│   ├── browser.nim        # Firefox + Chromium credential stores
│   ├── ssh.nim            # SSH keys, config, authorized_keys
│   ├── git.nim            # .git-credentials, tokens, config helpers
│   ├── cloud.nim          # AWS, GCP, Azure, Kubernetes
│   ├── history.nim        # Shell history, command patterns, .env files
│   ├── keyring.nim        # GNOME, KDE, KeePass, pass, Bitwarden
│   └── apptoken.nim       # DB creds, dev tokens, infra tokens, Docker
└── output/
    ├── terminal.nim       # Color terminal with box drawing
    └── json.nim           # Structured JSON output
```

## La base a zero eccezioni

Ogni file sorgente si apre con:

```nim
{.push raises: [].}
```

Questo pragma di Nim dice al compilatore: nessuna procedura in questo file può propagare un'eccezione. Qualsiasi chiamata a una funzione che potrebbe sollevarne una (I/O su file, parsing di stringhe, ecc.) deve essere racchiusa in un try/except all'interno di questo file. Il compilatore lo impone a compile time e rifiuterà il codice che potrebbe propagare un'eccezione non gestita.

Questo è importante per un tool di sicurezza perché andare in crash a metà scansione significa risultati parziali, rilevamenti mancati e output inaffidabile. La garanzia di `{.push raises: [].}` significa che, se il tool compila, non andrà in crash a causa di eccezioni non gestite durante una scansione.

Dove un'operazione non può davvero evitare le eccezioni (come chiamare `readFile`), lo schema è:

```nim
proc readFileContent*(path: string): string =
  try:
    result = readFile(path)
  except CatchableError:
    result = ""
```

L'eccezione viene intercettata immediatamente e convertita in un valore predefinito sicuro. Questo schema si ripete in tutto `base.nim` per ogni operazione sul filesystem.

## Entry point: harvester.nim

### Parsing della CLI

La funzione `parseCli` costruisce un `HarvestConfig` a partire dagli argomenti da riga di comando usando `parseopt` della stdlib di Nim:

```nim
proc parseCli(): HarvestConfig =
  result = defaultConfig()

  var parser = initOptParser(
    commandLineParams(),
    shortNoVal = {'d', 'q', 'v', 'h'},
    longNoVal = @["dry-run", "quiet", "verbose", "help", "version"]
  )
```

I parametri `shortNoVal` e `longNoVal` dicono al parser quali flag non accettano valori. Senza di essi, `--quiet --format json` tenterebbe di interpretare `--format` come il valore di `--quiet`.

Il ciclo del parser usa `case parser.key.toLowerAscii()` per un matching insensibile alle maiuscole, quindi `--Target`, `--target` e `--TARGET` funzionano tutti. Ogni flag riconosciuto aggiorna la configurazione risultante. I flag non riconosciuti vengono scartati silenziosamente (`else: discard`).

### Parsing dei moduli

La funzione `parseModules` converte nomi di moduli separati da virgola in una sequenza di valori `Category`:

```nim
proc parseModules*(input: string): seq[Category] =
  result = @[]
  let parts = input.split(",")
  for part in parts:
    let name = part.strip().toLowerAscii()
    for cat in Category:
      if ModuleNames[cat] == name:
        result.add(cat)
        break
```

Questa funzione itera su ogni valore dell'enum `Category` e verifica se il nome del modulo corrisponde. I nomi di moduli sconosciuti vengono saltati silenziosamente. L'array `ModuleNames` in config.nim mappa ogni Category sul suo nome stringa, quindi la mappatura è definita in un unico posto.

### Orchestrazione principale

La funzione `main` è il fulcro del flusso di controllo:

```nim
proc main() =
  let conf = parseCli()

  if conf.dryRun:
    renderDryRun(conf)
    quit(0)

  var report = runCollectors(conf)
  report.metadata.timestamp = now().utc.format("yyyy-MM-dd'T'HH:mm:ss'Z'")

  case conf.outputFormat
  of fmtTerminal: renderTerminal(report, conf.quiet, conf.verbose)
  of fmtJson: renderJson(report, conf.outputPath)
  of fmtBoth:
    renderTerminal(report, conf.quiet, conf.verbose)
    renderJson(report, conf.outputPath)

  var hasHighSeverity = false
  for sev in [svCritical, svHigh]:
    if report.summary[sev] > 0:
      hasHighSeverity = true
      break

  if hasHighSeverity: quit(1) else: quit(0)
```

Il timestamp viene impostato dopo il completamento della scansione (non prima), così riflette il momento in cui i risultati sono stati prodotti. Il pragma `{.cast(raises: []).}` viene usato attorno alla formattazione dell'orario perché `now()` e `format()` potrebbero tecnicamente sollevare eccezioni, ma in pratica non lo fanno mai con stringhe di formato valide. Il cast dice al compilatore "so quello che sto facendo".

Il controllo dell'exit code considera solo CRITICAL e HIGH. I rilevamenti MEDIUM, LOW e INFO non provocano un exit code diverso da zero. Questo mantiene il tool utile nelle pipeline CI, dove vuoi fallire sulle esposizioni reali ma non sulle note informative.

## Runner: runner.nim

### Smistamento dei collector

La funzione `getCollector` è la tabella di routing:

```nim
proc getCollector(cat: Category): CollectorProc =
  case cat
  of catBrowser: browser.collect
  of catSsh: ssh.collect
  of catCloud: cloud.collect
  of catHistory: history.collect
  of catKeyring: keyring.collect
  of catGit: git.collect
  of catApptoken: apptoken.collect
```

L'istruzione case di Nim su un enum è esaustiva: se aggiungi un nuovo valore di `Category` senza aggiungere un ramo case, il compilatore rifiuterà il codice. Questa garanzia a compile time previene la classe di bug "ho dimenticato di collegare il nuovo modulo".

Il tipo di ritorno `CollectorProc` è un tipo di puntatore a funzione definito in types.nim:

```nim
CollectorProc = proc(config: HarvestConfig): CollectorResult {.nimcall, raises: [].}
```

La convenzione di chiamata `{.nimcall, raises: [].}` significa che la funzione usa la convenzione di chiamata nativa di Nim e non può sollevare eccezioni. Questo contratto è imposto a livello di tipo.

### Aggregazione dei risultati

La funzione `runCollectors` itera sui moduli abilitati, chiama ciascun collector e costruisce il report:

```nim
proc runCollectors*(config: HarvestConfig): Report =
  let start = getMonoTime()
  var results: seq[CollectorResult] = @[]
  var moduleNames: seq[string] = @[]

  for cat in config.enabledModules:
    moduleNames.add(ModuleNames[cat])
    let collector = getCollector(cat)
    let res = collector(config)
    results.add(res)

  let elapsed = getMonoTime() - start

  var summary: array[Severity, int]
  for res in results:
    for finding in res.findings:
      inc summary[finding.severity]
```

L'uso di `getMonoTime()` (clock monotono) invece di `now()` per la misurazione dei tempi è importante. Il tempo monotono è immune alle regolazioni dell'orologio e alle correzioni NTP. Se l'orologio di sistema cambia durante una scansione, `getMonoTime()` fornisce comunque una durata accurata.

L'array di riepilogo usa `Severity` come tipo di indice, quindi `summary[svCritical]` fornisce direttamente il numero di rilevamenti critici. La proc `inc` incrementa sul posto senza allocazioni.

## Utility di base: collectors/base.nim

### Ispezione dei permessi

Le funzioni di verifica dei permessi usano direttamente POSIX `stat`:

```nim
proc getPermsString*(path: string): string =
  var statBuf: Stat
  try:
    if stat(path.cstring, statBuf) == 0:
      let mode = statBuf.st_mode and 0o7777
      result = "0" & toOct(mode.int, 3)
    else:
      result = "unknown"
  except CatchableError:
    result = "unknown"
```

La syscall `stat` recupera i metadati del file dal kernel. Il campo mode contiene i bit dei permessi nei 12 bit meno significativi (maschera `0o7777`). La funzione `toOct` converte nella rappresentazione ottale. Il prefisso stringa "0" produce il formato familiare: "0600", "0644", ecc.

Le verifiche di leggibilità da parte di chiunque e del gruppo estraggono bit specifici:

```nim
proc isWorldReadable*(path: string): bool =
  var statBuf: Stat
  try:
    if stat(path.cstring, statBuf) == 0:
      result = (statBuf.st_mode.int and WorldReadBit) != 0
  except CatchableError:
    discard
```

`WorldReadBit` è la costante `0o004` di config.nim. L'AND bit a bit isola solo il bit "lettura per gli altri". Se è diverso da zero, il file è leggibile da chiunque.

### Gravità dai permessi

La funzione `permissionSeverity` incapsula la logica che passa dai permessi alla gravità:

```nim
proc permissionSeverity*(path: string, isDir: bool = false): Severity =
  let perms = getNumericPerms(path)
  if perms < 0:
    return svInfo
  if (perms and WorldReadBit) != 0:
    return svCritical
  if (perms and GroupReadBit) != 0:
    return svMedium
  let expected = if isDir: OwnerOnlyDirPerms else: OwnerOnlyFilePerms
  if perms > expected:
    return svLow
  result = svInfo
```

Permessi negativi significano che la chiamata stat è fallita (il file non esiste o non possiamo leggerne i metadati). Leggibile da chiunque è sempre CRITICAL. Leggibile dal gruppo è MEDIUM. Qualsiasi cosa più larga dei permessi attesi (0600 per i file, 0700 per le directory) è LOW. I permessi corretti sono INFO.

### Factory dei rilevamenti

Le due funzioni factory costruiscono oggetti `Finding` con metadati coerenti:

```nim
proc makeFinding*(
  path: string,
  description: string,
  category: Category,
  severity: Severity
): Finding =
  Finding(
    path: path,
    category: category,
    severity: severity,
    description: description,
    credential: none(Credential),
    permissions: getPermsString(path),
    modified: getModifiedTime(path),
    size: getFileSizeBytes(path)
  )
```

Ogni rilevamento ottiene automaticamente i permessi correnti, la data di modifica e la dimensione del file del percorso di destinazione. I collector non devono ricordarsi di recuperarli. La variante `makeFindingWithCred` accetta un parametro `Credential` aggiuntivo racchiuso in `some()`.

### Oscuramento dei valori

La funzione `redactValue` mostra i primi N caratteri e sostituisce il resto:

```nim
proc redactValue*(value: string, showChars: int = 4): string =
  if value.len <= showChars:
    result = "*".repeat(value.len)
  else:
    result = value[0 ..< showChars] & "*".repeat(value.len - showChars)
```

Per i valori di lunghezza minore o uguale a `showChars`, l'intero valore viene mascherato. Per i valori più lunghi, vengono mostrati i primi 4 caratteri. Questo fornisce abbastanza contesto per identificare il tipo di credenziale (ad es. "ghp_" per i token GitHub, "AKIA" per le chiavi AWS) senza esporre il segreto completo.

## Implementazioni dei collector

### Collector SSH: ssh.nim

Il collector SSH ha quattro sotto-scanner: `scanKeys`, `scanConfig`, `scanAuthorizedKeys` e `scanKnownHosts`.

**La scansione delle chiavi** è il sotto-scanner più complesso. Percorre la directory `~/.ssh/`, legge ogni file e verifica se inizia con un'intestazione PEM:

```nim
proc isPrivateKey*(content: string): bool =
  for header in SshKeyHeaders:
    if content.startsWith(header):
      return true
```

`SshKeyHeaders` in config.nim contiene tutti e cinque i formati di intestazione PEM. La verifica usa `startsWith` invece di `contains` perché le intestazioni PEM devono trovarsi all'inizio del file.

Una volta trovata una chiave privata, il rilevamento della cifratura cerca marcatori noti:

```nim
proc isEncrypted*(content: string): bool =
  for marker in SshEncryptedMarkers:
    if marker in content:
      return true
```

Il calcolo della gravità combina lo stato di cifratura e i permessi:

```nim
if not encrypted:
  sev = svHigh
else:
  sev = svInfo

if perms >= 0 and (perms and WorldReadBit) != 0:
  sev = svCritical
elif perms >= 0 and (perms and GroupReadBit) != 0:
  if sev < svHigh:
    sev = svHigh
```

Una chiave non cifrata parte da HIGH. Una chiave cifrata parte da INFO. Poi i permessi la innalzano: leggibile da chiunque impone CRITICAL indipendentemente dalla cifratura. Leggibile dal gruppo la porta almeno a HIGH.

**La scansione della configurazione** cerca impostazioni deboli:

```nim
if stripped.toLowerAscii().startsWith("passwordauthentication yes"):
  weakSettings.add("PasswordAuthentication enabled")

if stripped.toLowerAscii().startsWith("stricthostkeychecking no"):
  weakSettings.add("StrictHostKeyChecking disabled")
```

Questi sono rilevamenti MEDIUM perché indeboliscono la sicurezza della connessione SSH ma non espongono direttamente credenziali.

### Collector dei browser: browser.nim

**La scansione di Firefox** inizia leggendo `profiles.ini` per trovare le directory dei profili:

```nim
let lines = readFileLines(profilesIniPath)
var profiles: seq[string] = @[]
var currentPath = ""

for line in lines:
  let stripped = line.strip()
  if stripped.startsWith("[Profile"):
    if currentPath.len > 0:
      profiles.add(currentPath)
    currentPath = ""

  if stripped.toLowerAscii().startsWith("path="):
    currentPath = stripped.split("=", maxsplit = 1)[1]
```

Si tratta di un parser INI minimale che estrae il valore `Path=` da ogni sezione `[Profile*]`. Il `maxsplit = 1` è importante perché i percorsi dei profili potrebbero a loro volta contenere caratteri `=`.

Per ogni profilo, lo scanner verifica tre file di credenziali:

```nim
let credFiles = [
  (FirefoxLoginsFile, "Firefox stored logins database"),
  (FirefoxCookiesDb, "Firefox cookies database"),
  (FirefoxKeyDb, "Firefox key database")
]

for (fileName, desc) in credFiles:
  let filePath = profileDir / fileName
  if safeFileExists(filePath):
    let sev = if isWorldReadable(filePath): svCritical
              elif isGroupReadable(filePath): svHigh
              else: svMedium
```

Nota che i file di credenziali dei browser hanno sempre una gravità almeno MEDIUM, anche con permessi corretti. Questo perché i file stessi contengono dati sensibili (password cifrate, cookie di sessione) che potrebbero essere esfiltrati e attaccati offline.

**La scansione di Chromium** segue uno schema simile ma gestisce più varianti di browser (Chrome, Brave, Vivaldi, Chromium) e profili numerati (`Default`, `Profile 1`, `Profile 2`, ecc.).

### Collector cloud: cloud.nim

**La scansione AWS** mostra l'analisi delle credenziali più dettagliata. Legge il file delle credenziali riga per riga, contando i profili e classificando i tipi di chiave:

```nim
if stripped.toLowerAscii().startsWith("aws_access_key_id"):
  let parts = stripped.split("=", maxsplit = 1)
  if parts.len == 2:
    let keyVal = parts[1].strip()
    if keyVal.startsWith(AwsStaticKeyPrefix):
      inc staticKeys
    elif keyVal.startsWith(AwsSessionKeyPrefix):
      inc sessionKeys
```

Le chiavi statiche (prefisso `AKIA`) hanno una lunga durata e innalzano la gravità a HIGH. Le chiavi di sessione (prefisso `ASIA`) sono temporanee e meno pericolose. La distinzione conta per stabilire le priorità di remediation.

**La scansione di Kubernetes** esegue il parsing dello YAML del kubeconfig per contare contesti e utenti e per rilevare i metodi di autenticazione:

```nim
if "token:" in stripped:
  hasTokenAuth = true
if "client-certificate-data:" in stripped:
  hasCertAuth = true
```

L'autenticazione con token ha gravità HIGH perché i bearer token forniscono accesso diretto all'API. L'autenticazione con certificato viene annotata ma non innalzata, perché i certificati sono più difficili da usare da soli.

### Collector della cronologia: history.nim

**Il pattern matching dei segreti** cerca pattern noti di variabili d'ambiente:

```nim
proc matchesSecretPattern*(line: string): bool =
  let upper = line.toUpperAscii()
  for pattern in SecretPatterns:
    if pattern in upper:
      if "export " in line.toLowerAscii() or
         line.strip().startsWith(pattern.split("=")[0]):
        return true
```

Il doppio controllo (pattern presente in upper E `export` oppure inizio con il nome della chiave) previene i falsi positivi. `PATH=/usr/bin` contiene `=` ma non corrisponde a `KEY=`, `TOKEN=` o `PASSWORD=`. La funzione richiede sia la corrispondenza del pattern sia la prova che si tratti di un'effettiva assegnazione di variabile.

**Il pattern matching dei comandi** usa un matcher personalizzato simile ai glob:

```nim
proc matchesCommandPattern*(line: string): bool =
  let lower = line.toLowerAscii()
  for pattern in HistoryCommandPatterns:
    let parts = pattern.split(".*")
    if parts.len >= 2:
      var allFound = true
      var searchFrom = 0
      for part in parts:
        let idx = lower.find(part, start = searchFrom)
        if idx < 0:
          allFound = false
          break
        searchFrom = idx + part.len
      if allFound:
        return true
```

Pattern come `"curl.*-h.*authoriz"` vengono divisi su `".*"` e ogni segmento viene cercato in sequenza. L'indice `searchFrom` garantisce che i segmenti corrispondano nell'ordine. Questo implementa un matching di base simile alle regex senza includere una libreria di regex.

**I limiti di righe della cronologia** prevengono l'esaurimento delle risorse:

```nim
const MaxHistoryLines = 50000
```

Uno sviluppatore che usa la stessa shell da anni potrebbe avere centinaia di migliaia di voci di cronologia. Scansionarle tutte sarebbe lento e richiederebbe molta memoria. Il limite di 50.000 righe copre la cronologia più recente (dove è più probabile che i segreti siano ancora validi) mantenendo limitato l'uso delle risorse.

**La scansione ricorsiva dei .env** usa un attraversamento delle directory a profondità limitata:

```nim
proc walkForEnv(
  dir: string,
  depth: int,
  excludePatterns: seq[string],
  result: var CollectorResult
) =
  if depth > MaxEnvDepth:
    return
```

Il limite di profondità di 5 e le esclusioni di directory (`node_modules`, `vendor`, `.git`, `__pycache__`, `.venv`, `.cache`) mantengono veloce l'attraversamento ricorsivo. Senza questi limiti, scansionare una directory con `node_modules` profondamente annidati richiederebbe minuti.

### Collector dei keyring: keyring.nim

Il collector dei keyring scansiona cinque diversi archivi di credenziali. Lo scanner di KeePass si distingue per la sua ricerca ricorsiva dei file:

```nim
proc walkForKdbx(
  dir: string,
  depth: int,
  excludePatterns: seq[string],
  result: var CollectorResult
) =
  if depth > 5:
    return
```

I file di database KeePass (`.kdbx`) possono essere memorizzati ovunque nella home directory, non solo in una posizione standard. L'attraversamento ricorsivo li trova ovunque si trovino, mentre il limite di profondità e le esclusioni di directory impediscono una scansione fuori controllo.

Lo scanner di pass (password-store) conta le voci cifrate con GPG:

```nim
for kind, path in walkDir(passDir, relative = false):
  if kind == pcFile and path.endsWith(".gpg"):
    inc entryCount
```

Il numero di voci dice all'utente (o all'attaccante) quante credenziali sono memorizzate, anche se le voci stesse sono cifrate con GPG e non direttamente leggibili.

### Collector dei token applicativi: apptoken.nim

Il collector dei token applicativi usa un tipo generico `AppTarget` per gestire le directory dei dati delle applicazioni:

```nim
type
  AppTarget = object
    path: string
    name: string
    description: string
    isDir: bool
```

Questo permette al collector di definire i target di scansione come dati invece che come codice:

```nim
let appTargets = [
  AppTarget(path: SlackDir, name: "Slack",
            description: "Slack desktop session data", isDir: true),
  AppTarget(path: DiscordDir, name: "Discord",
            description: "Discord desktop session data", isDir: true),
  ...
]
```

Ogni target viene verificato con la stessa logica: esiste, e quali sono i suoi permessi? Il flag `isDir` determina se usare la verifica dei permessi per file o per directory.

La scansione delle credenziali dei database è più dettagliata. Per il `.pgpass` di PostgreSQL:

```nim
let lines = readFileLines(pgpassPath)
var entryCount = 0
for line in lines:
  if line.strip().len > 0 and not line.strip().startsWith("#"):
    inc entryCount
```

Le righe non vuote e non di commento vengono contate come voci di connessione. Il conteggio finisce nei metadati della credenziale, così l'output può mostrare "PostgreSQL password file with 3 entries" invece di un semplice "PostgreSQL password file found."

## Rendering dell'output

### Renderer per il terminale: terminal.nim

Il renderer per il terminale gestisce la complessità dell'allineamento del testo nelle tabelle con box drawing quando le stringhe contengono codici colore ANSI invisibili e caratteri UTF-8 multi-byte.

**Il calcolo della lunghezza visiva** rimuove le sequenze di escape ANSI e conta solo i caratteri visibili:

```nim
proc visualLen(s: string): int =
  var i = 0
  while i < s.len:
    if s[i] == '\e':
      while i < s.len and s[i] != 'm':
        inc i
      if i < s.len:
        inc i
    elif (s[i].ord and 0xC0) == 0x80:
      inc i
    else:
      inc result
      inc i
```

Le sequenze di escape ANSI iniziano con `\e` e terminano con `m`. I byte di continuazione UTF-8 hanno lo schema `10xxxxxx` (la maschera `0xC0` verifica i due bit più significativi). Solo i byte che non sono né di escape né di continuazione contano come caratteri visibili. Questo è necessario perché una stringa come `"\e[31mERROR\e[0m"` ha 5 caratteri visibili ("ERROR") ma è lunga 15 byte.

**La scrittura delle righe del box** usa questa lunghezza visiva per riempire ogni riga esattamente fino a `BoxWidth` caratteri:

```nim
proc writeBoxLine(content: string) =
  try:
    stdout.write content
    let vLen = visualLen(content)
    let pad = BoxWidth - vLen - 1
    if pad > 0:
      stdout.write " ".repeat(pad)
    stdout.writeLine BoxVertical
  except CatchableError:
    discard
```

Il `-1` tiene conto del carattere `BoxVertical` di chiusura. Questo produce bordi del box perfettamente allineati indipendentemente da quanti codici colore o caratteri Unicode siano presenti nel contenuto.

**I badge di gravità** combinano colore ed etichetta:

```nim
proc sevBadge(sev: Severity): string =
  SeverityColors[sev] & ColorBold & " " & SeverityLabels[sev] & " " & ColorReset
```

Gli array `SeverityColors` e `SeverityLabels` sono indicizzati dall'enum `Severity`, quindi recuperare il colore di una gravità è un accesso diretto all'array.

### Renderer JSON: json.nim

Il renderer JSON converte ogni tipo in un `JsonNode` con funzioni ricorsive:

```nim
proc findingToJson(f: Finding): JsonNode =
  result = newJObject()
  {.cast(raises: []).}:
    result["path"] = newJString(f.path)
    result["category"] = newJString($f.category)
    result["severity"] = newJString($f.severity)
    result["description"] = newJString(f.description)
    result["permissions"] = newJString(f.permissions)
    result["modified"] = newJString(f.modified)
    result["size"] = newJInt(f.size)
    if f.credential.isSome:
      result["credential"] = credentialToJson(f.credential.get())
```

Il blocco `{.cast(raises: []).}` è necessario perché le operazioni JSON di Nim tecnicamente possono sollevare eccezioni, ma in pratica non lo faranno quando si costruiscono oggetti a partire da dati noti e validi. Il cast è limitato al solo blocco di costruzione del JSON.

L'operatore `$` applicato ai valori enum produce la rappresentazione stringa ("browser", "critical", ecc.) definita dalle assegnazioni dei valori dell'enum in types.nim.

## Strategia di test

### Unit test: test_all.nim

Gli unit test coprono funzioni pure che non richiedono uno stato del filesystem:

**I test di oscuramento** verificano le condizioni limite:

```nim
suite "redactValue":
  test "short value fully redacted":
    check redactValue("abc", 4) == "***"

  test "value longer than showChars":
    check redactValue("mysecret", 4) == "myse****"
```

**I test di rilevamento delle chiavi** convalidano tutti e cinque i formati di chiave più i casi negativi:

```nim
suite "isPrivateKey":
  test "OpenSSH key":
    check isPrivateKey("-----BEGIN OPENSSH PRIVATE KEY-----\ndata")
  test "public key rejected":
    check isPrivateKey("-----BEGIN PUBLIC KEY-----\ndata") == false
```

**I test di pattern matching** coprono sia i veri positivi sia i veri negativi:

```nim
suite "matchesSecretPattern":
  test "export with KEY=":
    check matchesSecretPattern("export API_KEY=some_value")
  test "non-secret assignment":
    check matchesSecretPattern("export PATH=/usr/bin") == false
```

**I test sulla gravità dei permessi** usano percorsi inesistenti per testare il percorso di gestione degli errori:

```nim
suite "permissionSeverity":
  test "returns svInfo for unreadable path":
    check permissionSeverity("/nonexistent/path/abc123") == svInfo
```

**I test di parsing dei moduli** verificano la conversione da CLI a Category, inclusi i casi limite:

```nim
suite "parseModules":
  test "single module":
    check parseModules("ssh") == @[catSsh]
  test "unknown module ignored":
    check parseModules("fake,nonexistent").len == 0
```

Esecuzione dei test: `just test` compila ed esegue la test suite.

### Test di integrazione: Docker

Il test di integrazione basato su Docker crea un ambiente controllato con file di credenziali noti e convalida che lo scanner li rilevi tutti.

**Il Dockerfile** usa una build multi-stage:

Stage 1 (builder): compila credenum dal sorgente usando l'immagine ufficiale Nim Alpine, producendo un binario statico.

Stage 2 (runtime): Ubuntu 24.04 con un account `testuser`. La directory `planted/` viene copiata nella home directory dell'utente di test, creando file di credenziali realistici in tutte e 7 le categorie. I permessi vengono impostati esplicitamente per creare scenari di gravità specifici (ad es. `chmod 0644` sulla chiave SSH non protetta per renderla leggibile da chiunque).

**Lo script di validazione** esegue credenum sulla home directory dell'utente di test e verifica i rilevamenti attesi:

```bash
OUTPUT=$(credenum --target /home/testuser --format json 2>&1) || true

check "SSH unprotected private key"    "no passphrase"
check "AWS credentials with static keys" "static keys"
check "Firefox stored logins"          "Firefox stored logins"
check "History secret pattern"         "Secret in shell history"
```

Ogni funzione `check` cerca una stringa attesa nell'output JSON. La validazione copre tutte e 7 le categorie di collector con oltre 30 verifiche individuali.

Esecuzione dei test di integrazione: `just docker-test` costruisce l'immagine Docker ed esegue la validazione.

## Sistema di build

### Configurazione del compilatore Nim: config.nims

Il file `config.nims` configura il compilatore Nim senza flag da riga di comando:

```nim
switch("mm", "orc")
```

ORC (Overflowing Reference Counting) è la gestione della memoria moderna di Nim. Combina il reference counting con un cycle collector, fornendo una pulizia deterministica senza le pause di un garbage collector tradizionale.

**Linking statico con musl:**

```nim
when defined(musl):
  var muslGcc = findExe("musl-gcc")
  if muslGcc.len > 0:
    switch("gcc.exe", muslGcc)
    switch("gcc.linkerexe", muslGcc)
  switch("passL", "-static")
```

Quando viene passato `-d:musl`, il compilatore usa musl-gcc invece del gcc di sistema, producendo un binario completamente statico senza dipendenza da glibc. Questo binario gira su qualsiasi sistema Linux indipendentemente dalla versione di glibc.

**Cross-compilation:**

```nim
when defined(crossX86):
  switch("passC", "-target x86_64-linux-musl")
  switch("passL", "-target x86_64-linux-musl")
  switch("os", "linux")
  switch("cpu", "amd64")
```

L'integrazione con zigcc usa il backend del compilatore C di Zig come toolchain di cross-compilation. Passare i target triple tramite `-passC` e `-passL` produce binari per architetture diverse senza bisogno di un sysroot completo per la cross-compilation.

### Target di build del Justfile

Il Justfile organizza i comandi per gruppo:

Gruppo **dev**: `build` (debug), `run` (build + esecuzione), `scan` (build + scansione dell'utente corrente), `check` (controllo dei tipi senza compilazione)

Gruppo **prod**: `release` (ottimizzata), `release-static` (musl), `release-small` (musl + UPX), `build-x86` (cross-compilation x86_64), `build-arm64` (cross-compilation ARM64)

Gruppo **test**: `test` (unit test), `docker-build` (build dell'immagine di test), `docker-test` (test di integrazione completo)

Gruppo **lint**: `fmt` (formattazione con nph), `fmt-check` (verifica della formattazione)

Il Justfile usa variabili shell per una configurazione DRY:

```just
bin     := "bin/credenum"
src     := "src/harvester.nim"
```

Tutti i target di build fanno riferimento a queste variabili, quindi cambiare il nome del binario o l'entry point richiede di modificare una sola riga.

## Principi di organizzazione del codice

**Un file per ogni responsabilità.** Ogni collector è un file a sé. I tipi sono separati dalla configurazione. I renderer dell'output sono separati l'uno dall'altro. Questo significa che puoi capire il collector dei browser leggendo solo `browser.nim`.

**Costanti separate dalla logica.** Tutti i percorsi, i pattern, le soglie, i colori e le etichette si trovano in `config.nim`. I collector importano `config` per ottenere i propri target di scansione. Questa separazione significa che puoi sottoporre ad audit ogni percorso di credenziali controllato dal tool leggendo un solo file.

**Utility condivise in base, non duplicate.** La verifica dei permessi, la lettura dei file, la costruzione dei rilevamenti e l'oscuramento si trovano in `base.nim`. Nessun collector reimplementa le chiamate stat sui file o la costruzione dei rilevamenti.

**I tipi definiscono il contratto.** Il tipo `CollectorProc` garantisce che ogni collector abbia la stessa firma. Il tipo `Report` definisce cosa ricevono i renderer dell'output. L'ordinamento dell'enum `Severity` determina il comportamento dei confronti. Questi tipi sono l'architettura, imposta dal compilatore.

## Estendere il codice

### Aggiungere un nuovo collector

1. Aggiungi un nuovo valore di `Category` in `types.nim` (ad es. `catContainer = "container"`)
2. Aggiungi il nome del modulo e la descrizione agli array in `config.nim`
3. Crea `collectors/container.nim`:
   - Inizia con `{.push raises: [].}`
   - Importa `../types`, `../config`, `base`
   - Implementa `proc collect*(config: HarvestConfig): CollectorResult`
   - Usa `newCollectorResult`, `makeFinding`, `makeFindingWithCred` da base.nim
4. Importa e instrada in `runner.nim`:
   - `import collectors/container`
   - Aggiungi `of catContainer: container.collect` all'istruzione case
5. Aggiungi test fixture in `tests/docker/planted/` e verifiche in `validate.sh`

### Aggiungere un nuovo target di scansione a un collector esistente

1. Aggiungi la costante del percorso a `config.nim` (ad es. `PodmanConfig* = ".config/containers/auth.json"`)
2. Aggiungi la logica di scansione al collector pertinente (ad es. `apptoken.nim`)
3. Aggiungi una test fixture e una verifica di validazione

### Aggiungere un nuovo formato di output

1. Aggiungi un valore a `OutputFormat` in `types.nim`
2. Crea `output/sarif.nim` con una proc di rendering
3. Aggiungi il ramo del case e la gestione del flag CLI in `harvester.nim`

## Dipendenze

Il progetto usa solo la libreria standard di Nim. Non è richiesto alcun pacchetto esterno.

| Import | Scopo |
|--------|---------|
| `std/parseopt` | Parsing degli argomenti CLI |
| `std/os` | Operazioni sui file, manipolazione dei percorsi, home directory |
| `std/posix` | POSIX stat per l'ispezione dei permessi |
| `std/strutils` | Operazioni sulle stringhe (split, strip, find, contains) |
| `std/times` | Formattazione dei timestamp |
| `std/monotimes` | Misurazione delle prestazioni |
| `std/options` | Optional[Credential] per i rilevamenti |
| `std/tables` | Coppie chiave-valore dei metadati delle credenziali |
| `std/json` | Costruzione dell'output JSON |
| `std/unittest` | Framework di test |

Zero dipendenze esterne significa nessun rischio di supply chain, nessun conflitto di versioni e nessun accesso di rete necessario per la build. L'intero progetto si compila da un'installazione di Nim appena fatta.

## Prossimi passi

- Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per idee di estensione che vanno da nuovi collector all'automazione della remediation
- Prova ad aggiungere un nuovo target di scansione a un collector esistente. Inizia con qualcosa di semplice, come aggiungere il rilevamento delle credenziali di Docker Compose al collector apptoken
