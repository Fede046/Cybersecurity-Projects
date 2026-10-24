<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# marshalsea: Architettura

Questo capitolo è il design. Spiega i pezzi, le giunture tra di essi e la manciata di decisioni che sembrano strane finché non sai da cosa stanno proteggendo. La walkthrough del codice è in [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md); questa è la mappa che vuoi tenere aperta mentre la leggi.

## La forma

Due reader, un vocabolario. Niente nel percorso di ispezione fa mai rivivere un oggetto.

```
   Marshal bytes ──> Parser ──> Node graph ──┐
                                             ├──>  BoundaryDetector  ──>  Decision
   YAML document ──> Inspector ──> Document ─┘         (proceed / blocked / observed)

   loaded classes ──> Scanner ──> entry points + links      (offense: what is usable)
   chain registry ──> generate ──> serialize                (offense: build the payload)
   Marshal.load   ──> LoadGuard (TracePoint :call)          (defense: veto before the body)
```

Le due metà della libreria non si chiamano mai a vicenda. La metà offensiva costruisce payload e va a caccia di gadget; la metà difensiva legge byte e prende decisioni. Si incontrano solo nella test suite e nel gate, dove l'output di ciascuna è l'input dell'altra. Questo è deliberato: un detector il cui autore ha scritto anche i payload intercetterà solo i payload che il suo autore ha immaginato, quindi il corpus avversariale esiste come terzo artefatto rispetto al quale entrambe le metà vengono misurate.

## Reader uno: il parser Marshal

`Marshal` è un formato binario. Due byte di header per la versione, poi un albero di valori taggati. Il parser lo legge byte per byte e costruisce un grafo di nodi. Non chiama mai `Marshal.load`, non risolve mai un nome di classe in una classe reale, e non alloca mai nulla dallo stream eccetto stringhe e interi.

I tag che comprende, con quelli che contano marcati:

```
   0 nil        T true       F false      i fixnum      l bignum      f float
   " string     : symbol     ; symlink    @ object link  / regexp
   [ array      { hash       } hash+default          S struct
   I ivar       o object     e extended   C userclass
   c class      m module     M module (old)
   u userdef     <- SINK, dispatches Klass._load
   U usermarshal <- SINK, dispatches obj.marshal_load
   d data        <- SINK, dispatches Klass._load_data
```

Tre tag sono **sink**: `u`, `U` e `d`. Vederne uno in uno stream è un'affermazione vera che caricare lo stream invocherà un metodo specifico su un nome di classe specifico, prima che qualsiasi proc di allowlist possa essere eseguita. Questo è il segnale con la confidenza più alta che il parser produce, ed è anche, cosa importante, non sufficiente. La chain della CVE pubblicata produce **zero tag di sink**, perché ERB non definisce alcun `marshal_load`. Il solo rilevamento dei sink non la intercetta mai.

### Il parser è forense di proposito

Ecco la decisione di design che sembra sbagliata a prima vista.

CRuby rifiuta uno stream dove lo slot del nome della variabile d'istanza contiene qualcosa che non è un simbolo. Solleva un'eccezione e si ferma. Questo parser **continua**, registra il problema come un'anomalia nominata, e restituisce comunque un grafo completo.

Osserva la differenza sugli stessi 17 byte:

```
Marshal.load: ArgumentError: dump format error for symbol(0x69)
parser:       class_names=["Object"] anomalies=["instance variable name slot holds fixnum, not a symbol"]
detector:     blocked: stream is not canonical Marshal: instance variable name slot holds fixnum, not a
              symbol, so Marshal.load refuses it and there is nothing here to permit
```

Perché preoccuparsene, se `Marshal.load` lo rifiuterebbe comunque? Perché **l'obiettivo del parser è la descrizione, non il controllo di ammissione.** Se uno stream ostile nasconde un sink in uno slot dove dovrebbe stare un simbolo, un parser che solleva un'eccezione alla prima sorpresa strutturale riporta "malformato" e non ti dice nulla di cosa c'era dentro. Un parser che continua riporta il nome della classe, il sink, *e* l'anomalia. Ottieni un registro forense invece di un messaggio d'errore.

Il rigore vive un layer più su. Il detector tratta qualsiasi anomalia registrata come un rifiuto immediato, e la sua stringa di motivazione dice precisamente perché: non "questo è pericoloso" ma "`Marshal.load` rifiuta questo e non c'è niente qui da permettere." Sono affermazioni diverse e solo la seconda è vera.

Quella separazione è la singola idea strutturale più importante nella libreria. **Il parser etichetta, il detector decide.** Significa anche che ai due componenti è permesso di essere in disaccordo sullo stesso stream, e quel disaccordo è una feature. C'è un test chiamato `test_sink_in_an_instance_variable_name_position_is_still_reported` che esiste per bloccarlo in posizione, e una proposta di rendere il parser rigoroso sugli slot di ruolo è stata rifiutata specificamente perché avrebbe cancellato quel test.

Lo stesso istinto appare nella decodifica dei float. Un float Marshal legacy porta un'estensione della mantissa dopo un byte NUL che il Ruby moderno legge ancora:

```
canonical 1.5:   "\x04\bf\b1.5"
legacy stream:   value=1.5  undecoded_tail="\x00abcde"  fully_decoded=false
Marshal.load:    1.5000000000055356
```

Il parser non indovina e non finge. Decodifica ciò che può, etichetta il resto come un `undecoded_tail`, e risponde onestamente a `fully_decoded?`. Un reader che restituisse silenziosamente `1.5` starebbe affermando un accordo con l'interprete che non si è guadagnato.

### Sigillatura

Una volta terminato il parsing, l'intero grafo viene congelato depth-first e anche il `Result` che lo avvolge viene congelato. Un risultato di parsing è una descrizione immutabile di byte che sono già avvenuti. Niente a valle, incluso un chiamante che lo riceve da `Decision#result`, può mutare il registro da cui è stata presa una decisione di policy.

## Reader due: l'inspector Psych

YAML non ha bisogno di un parser scritto a mano, perché Psych ne fornisce già uno che non fa rivivere nulla. `Psych.parse_stream` costruisce un AST di oggetti `Psych::Nodes::*` e si ferma. L'inspector percorre quell'AST.

Ciò che estrae è un `Reference` per ogni tag `!ruby/*`, che porta tre cose:

```
   !ruby/object:Gem::Version    ->  class_name: "Gem::Version"
                                    kind:       "object"
                                    revival:    "init_with"     <- what Psych WOULD call
                                    key_position: false
```

Il mapping dal tipo di tag al metodo di rivivificazione è l'intero valore dell'inspector, perché trasforma "questo documento menziona una classe" in "questo documento invocherebbe `init_with` su quella classe":

| Tipo di tag | Metodo che Psych invoca |
|---|---|
| `!ruby/object` | `init_with` |
| `!ruby/array`, `!ruby/string`, `!ruby/struct`, `!ruby/exception` | `init_with` |
| `!ruby/hash` | `[]=` |
| `!ruby/marshalable` | `marshal_load` |

**Gli alias vengono contati, mai espansi.** Una alias bomb non costa nulla da ispezionare, ed è questo il punto: un inspector che espandesse gli alias per poterli riportare avrebbe importato esattamente il denial-of-service contro cui esiste per mettere in guardia. Il conteggio è limitato e riportato invece di essere ignorato, perché costa comunque qualunque cosa spenda il loader finale.

La limitation notice dell'inspector stesso è schietta su dove si colloca, e vale la pena interiorizzarla:

> "Unlike Marshal, Psych's own allowlist is a real veto: Psych checks the tag before it revives the object, where Marshal runs its proc after the callback has already fired. Same intent, opposite outcome, decided entirely by where the check sits. That means `YAML.safe_load` with `permitted_classes` **is** a boundary and this inspector is only detection and reporting on top of it."

Un detector che si posizionasse come *sostituto* di `safe_load` starebbe vendendo un downgrade. Questo lo dice in una costante.

## L'oggetto decisione

Tre stati, mutuamente esclusivi per costruzione:

```
   proceed    the bytes matched the configured policy
   blocked    the bytes violated it, and `reason` says which rule and why
   observed   the bytes violated it, the reporter was called, and the caller
              was handed the snapshot anyway  (monitoring mode)
```

Non c'è alcun predicato `accepted?`, e la sua assenza è deliberata. Sotto `observe_and_log`, "la policy ha permesso questo" e "questo stream è pulito" hanno **risposte diverse**, e un singolo predicato chiamato `accepted?` non può rispondere a entrambe. I chiamanti che scrivono `Marshal.load(d.snapshot) if d.accepted?` caricherebbero silenziosamente tutto ciò su cui la modalità di monitoraggio ha riportato. Costringere il chiamante a nominare lo stato che intende effettivamente vale i sei caratteri in più.

Lo stato viene validato nel costruttore rispetto a una lista congelata, quindi uno stato non valido è un `ArgumentError` alla costruzione anziché un predicato che restituisce silenziosamente false ovunque.

### La scala delle violazioni

Il detector controlla le regole in un ordine fisso e restituisce la prima violazione che trova. L'ordine non è arbitrario; va da "questo non è nemmeno caricabile" passando per "questo invoca durante il load" fino a "questo menziona una classe che non hai approvato":

```
   1. role anomaly        the parser recorded a malformed slot -> Marshal.load refuses it anyway
   2. sink tag            u / U / d  -> _load, marshal_load, or _load_data dispatches
   3. hash-dispatching key    #hash runs when the Hash is rebuilt
   4. eql?-dispatching key    #eql? runs as soon as two keys collide
   5. Range endpoint      #<=> runs when Range#marshal_load validates its ends
   ────────── deny_sinks_only stops here ──────────
   6. non-canonical version   declares a Marshal version no real Ruby emits
   7. unapproved class name   strict_allowlist only
```

Le regole da 1 a 5 sono affermazioni sul **dispatch**: sono vere indipendentemente da quali classi consideri fidate, ed è per questo che vengono eseguite sotto ogni policy incluso `deny_sinks_only`. Le regole 6 e 7 sono policy, e solo `strict_allowlist` le applica.

Le regole 3, 4 e 5 esistono a causa di tre bypass che sono stati rilasciati e trovati in seguito. Tutti e tre venivano accettati sotto `deny_sinks_only` **e** sotto allowlisting rigoroso con la classe in allowlist, perché nessuna delle due regole guardava il dispatch attraverso la posizione di chiave:

- Una chiave di hash sottoclasse di `String` che raggiunge `#eql?` su collisione di bucket.
- Un gadget annidato dentro un `Array` nudo usato come chiave. Diciannove byte: `"\x04\b{\x06[\x06o:\fUngated\x00i\x06"`.
- Un `Range` i cui estremi invocano `#<=>`.

Questo è l'argomento per cui la scala è dati anziché una catena di condizionali ad-hoc: ogni regola è un'affermazione separata e testabile individualmente su cosa farà l'interprete, e il corpus porta un payload per ciascuna.

### Le stringhe di motivazione sono output controllato dall'attaccante

Una stringa di motivazione cita un nome di classe, e un nome di classe viene dallo stream. Questo rende ogni stringa di motivazione una superficie di log injection, quindi le succedono tre cose prima che venga emessa:

- I nomi vengono troncati a 96 byte con un marcatore esplicito `[truncated, +N bytes]` anziché silenziosamente.
- Le liste mostrano al massimo 8 nomi con un esplicito `, and N more`.
- Tutto passa attraverso `String#inspect` su byte forzati a binario, quindi un nome di classe contenente un newline non può falsificare una riga di log.

Un detector che incollasse una stringa illimitata controllata dall'attaccante nei tuoi log avrebbe trasformato una difesa in un meccanismo di consegna.

## Lo scanner

Lo scanner risponde a una domanda diversa dal detector: non "questo stream è pericoloso" ma "**quali classi attualmente caricate in questo processo potrebbero essere usate come gadget.**"

Percorre `ObjectSpace.each_object(Module)`, e per ogni modulo con nome raccoglie i metodi auto-invocati che il modulo definisce *esso stesso* (non ereditati), poi valuta ciascuno.

### La tassonomia è una tabella, non una lista di nomi di metodi

È qui che l'asse gated-contro-ungated del capitolo sui concetti diventa codice. Ogni metodo candidato porta tre fatti:

| Metodo | Gate | Arity fornita dal deserializzatore | Formati |
|---|---|---|---|
| `marshal_load` | gated | 1 | Marshal, Psych |
| `_load_data` | gated | 1 | Marshal |
| `_load` | gated (singleton) | 1 | Marshal |
| `init_with` | soft | 1 | Psych |
| `hash` | ungated | 0 | Marshal, Psych |
| `eql?` | ungated | 1 | Marshal, Psych |
| `<=>` | ungated | 1 | Marshal |
| `==` | ungated | 1 | Psych |
| `[]=` | ungated | 2 | Psych |
| `method_missing` | ungated | variadic | Marshal, Psych |
| `respond_to_missing?` | ungated | 2 | Marshal, Psych |
| `respond_to?` | ungated | variadic | Psych |
| `to_s` | **link** | 0 | nessuno |
| `coerce` | **link** | 1 | nessuno |

Tre colonne e ognuna si guadagna il suo posto.

**La colonna del gate** è l'asse di [01-CONCEPTS.md](./01-CONCEPTS.it.md). Un metodo gated ha bisogno di un `respond_to?` veritiero; uno ungated viene invocato alla cieca.

**La colonna del formato** è il motivo per cui gli entry point vengono valutati per formato anziché globalmente. Un proxy con i metodi cancellati è un entry point Psych valido e uno Marshal non valido, quindi una singola risposta globale "questo è raggiungibile" sarebbe sbagliata per uno dei due. La scan di oggi trova 29 entry point raggiungibili attraverso Marshal e 33 attraverso Psych, e nessuno dei due insiemi contiene l'altro.

**La colonna dell'arity** è quella sottile. Un metodo chiamato `init_with` che prende tre argomenti non può essere chiamato da un deserializzatore che ne fornisce uno. Riportarlo è un falso positivo. Su un'image di default il controllo dell'arity rifiuta esattamente un candidato, ed è un buon esempio:

```
entry points whose arity cannot accept the deserializer's call: 1
  Psych::Visitors::ToRuby#init_with               arity=3    needs=1
```

Quello è il metodo visitor di Psych stesso, che si trova a condividere un nome con l'hook che invoca. Senza la colonna dell'arity starebbe in cima a ogni scan come un falso positivo permanente e confondente.

**Le righe dei link** sono quelle senza alcun formato. `to_s` è un vero passaggio nella universal chain pubblicata, ma `Marshal` non lo chiama mai. È un metodo che un *gadget* chiama una volta che una chain è già in movimento, non un metodo che un *deserializzatore* invoca per avviarne una. Confondere i due è la singola fonte più grande di falsi positivi nello scanning di gadget, quindi i link vengono raccolti, contati ed esclusi dalla raggiungibilità. La scan di oggi ne trova 53 accanto a 140 entry point.

### Raggiungibilità, e cosa si rifiuta deliberatamente di concludere

Mettendo insieme la tassonomia:

```
reachable?  =  is an entry point (not a link)
            AND its arity can accept the deserializer's call
            AND ( it is gated or soft-gated                     <- the tag alone reaches it
                  OR its body references object state
                  OR its source could not be read )             <- read that last one twice
```

L'ultima clausola è quella interessante. Per decidere se un metodo ungated come `#hash` è *interessante*, lo scanner fa il parsing del sorgente del metodo con Prism e chiede se il corpo fa riferimento a una variabile d'istanza o fa una chiamata senza receiver. Un `#hash` che restituisce un letterale è inerte; un `#hash` che legge `@name` è un potenziale pivot.

Ma se il sorgente non può essere letto, lo scanner valuta il metodo **raggiungibile**, non inerte. Questa è la direzione corretta in cui uno strumento di sicurezza può sbagliare. Su un'image di default 8 candidati sono in quello stato, e la scan lo dice.

### La sotto-segnalazione viene segnalata, ad alta voce

Uno scanner di gadget che ingoia silenziosamente gli errori è peggio che nessuno scanner, perché produce una lista breve e pulita che si legge come "niente da vedere qui." Quindi ogni errore ingoiato viene contato e attribuito a un sito nominato:

```
suppressed errors (this scan under-reports):
  source_parse     3

142 candidates have no Ruby source and were never analysed; the reachability filter does not cover them
```

Ci sono cinque siti di soppressione e tre di essi sono marcati **lossy**, il che significa che un fallimento lì significa che un candidato non è mai stato nemmeno creato. `Report#candidates_lost?` è il predicato che distingue "questa scan è leggermente meno precisa" da "a questa scan mancano interamente delle voci," e `complete?` e `fully_analysed?` rispondono a due domande separate anziché a una sfocata.

Il numero di riferimento da una `ruby:4.0-slim` di default (Ruby 4.0.6), rimisurato il 2026-07-31:

```
modules=691  candidates=193  entry_points=140  links=53
gated=11  soft=5  ungated=124
reachable=43  (28 of them ungated)   marshal=29  psych=33
unanalysable=142  unreadable=8  suppressed=3
```

**Quei numeri non sono fatti su Ruby.** `ObjectSpace` non può riportare una classe che nessuno ha ancora richiesto, quindi sono un'affermazione su ciò che questo specifico processo aveva caricato. Richiedere `active_support` li sposta tutti. Rieseguilo tu stesso invece di citare questi.

E l'inquadramento onesto, dichiarato nel progetto anziché implicito: uno strumento di scoperta di gadget ti dice quali chain esistono **oggi**, in **questo** processo. Non è un controllo.

## Le chain

Tre payload, e la directory è l'identità. Ogni file sotto `lib/marshalsea/chains/` fa subclass di `Base`, e `Base.inherited` lo registra. Non c'è alcun file di registro centrale che elenca i nomi delle chain, perché un file di registro è una cosa che marcisce quando qualcuno aggiunge una chain e si dimentica di aggiornarlo.

```
  erb-def-method   primitive  CVE-2026-41316   def_method
  erb-def-module   chain      CVE-2026-41316   hash
  psych-init-with  chain      none             init_with
```

**Primitiva e chain sono etichette diverse e la differenza è portante.** Una *chain* scatta dentro il deserializzatore senza alcuna cooperazione dall'applicazione. Una *primitiva* forgia un oggetto oltre una guardia e resta inerte finché l'applicazione non ci fa qualcosa. Etichettare `erb-def-method` come chain lo sopravvaluterebbe; cancellarlo cancellerebbe la lezione che il call site pericoloso può vivere nel tuo stesso codice.

Ogni chain dichiara le versioni che affligge come vincoli `Gem::Requirement` reali, così il confine è interrogabile anziché prosa:

```
  erb 4.0.3   affected? true
  erb 4.0.3.1 affected? false
  erb 6.0.1   affected? true
  erb 6.0.1.1 affected? false
  erb 6.0.4   affected? false
```

### Il builder non deve mai eseguire il proprio payload

Questo è il vincolo che dà forma alla metà offensiva, e non è ovvio finché non ti morde.

La chain `erb-def-module` entra attraverso la posizione di chiave di hash. Quindi il modo naturale di costruirla è `Marshal.dump({ proxy => 1 })`. Questo chiama `#hash` sul proxy **mentre il tuo builder sta costruendo il letterale**, il che attiva la chain localmente, nel processo che doveva generare un payload destinato a qualche altro posto.

La correzione è non mettere mai l'oggetto in un hash. `Base#in_hash_key_position` dumpa l'oggetto da solo, taglia via i suoi due byte di header, e innesta il body in un frame di hash a una voce scritto a mano:

```
   Marshal.dump(nil)      "\x04\b" "0"
                           ^^^^^^ take the header

   Marshal.dump(proxy)    "\x04\b" <body>
                                   ^^^^^^ take the body

   result                 "\x04\b" "{\x06" <body> "0"
                                    ^^^^^^         ^^^ nil value
                                    one-entry hash frame
```

C'è un modo in cui quell'innesto può andare storto, e il codice rifiuta anziché rischiarlo. Gli object link di Marshal sono **posizionali**: `@6` significa "il sesto oggetto registrato." Innestare un body dietro un nodo hash sposta ogni indice di uno, quindi un grafo di payload contenente una back-reference si decodificherebbe silenziosamente in un grafo diverso da quello che hai costruito. Quindi dopo l'innesto, il builder fa il parsing del proprio output e solleva `ObjectLinkRefusedError` se qualche object link è sopravvissuto. Un generatore di payload che può produrre un grafo che non intendeva è peggio di uno che rifiuta.

## La guardia a runtime

`LoadGuard` fa ciò che la proc di allowlist non può fare: pone il veto **prima** che il corpo del metodo venga eseguito. Un `TracePoint` su `:call` scatta all'ingresso del metodo, quindi sollevare un'eccezione dall'handler significa che il corpo non viene mai eseguito.

```
   Marshal.load
        │
        ├─ allocate Klass
        ├─ dispatch marshal_load  ──> TracePoint :call fires HERE
        │                             owner not permitted -> raise
        │                             (body never runs)
        └─ r_post_proc ──> your allowlist proc would have run HERE, too late
```

La lista degli hook è derivata per enumerazione, non indovinata. Tracciare ogni `:call` e `:c_call` durante il load di un payload contenente un oggetto semplice, una classe con `marshal_load`, una classe con `_load`, una `Struct`, un oggetto extended, `Hash`/`Array`/`String` sottoclassati, una chiave con `#hash` custom, un `Range`, `Time`, `Rational`, `Regexp` e un'`Exception` produce questa superficie di dispatch completa:

```
ALL distinct method_ids seen: [:_load, :hash, :initialize, :load, :marshal_load]
```

Piccola, ed è per questo che la guardia è trattabile.

Una regola strutturale regge l'intera cosa: **la guardia non invoca mai un metodo sul receiver che sta ispezionando.** Risolve il nome del proprietario attraverso `Object.instance_method(:class)`, `Object.instance_method(:is_a?)` e `Module.instance_method(:name)`, legati al receiver anziché chiamati su di esso. Il receiver è un gadget, un proxy con i metodi cancellati risponde a `.class` e `.is_a?` attraverso `method_missing`, e `method_missing` è esattamente ciò attraverso cui entra la chain distribuita. Una guardia che chiede all'oggetto cosa sia attiva la chain su cui stava per porre il veto, dentro un handler di `TracePoint` che non traccia le proprie chiamate annidate. [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) ha la trascrizione di quel fallimento e il test che lo fissa.

### Fornisce i propri bypass, incluso uno lasciato aperto di default

Il set di hook di default osserva `marshal_load`, `_load`, `_load_data`, `method_missing` e `respond_to_missing?`. **Non** osserva `#hash` e `#eql?`. Questo è un buco deliberato e documentato, ed ecco il confronto a tre vie sullo stesso payload da 19 byte:

```
default guard  -> LOADED, #hash fired, watches?(:hash)=false
strict guard   -> blocked: deserialization hook OKey#hash is not permitted, #hash never fired
detector       -> blocked, nothing loaded at all
```

La guardia di default lascia passare la forma con chiave ungated. Perché lasciarla aperta? Perché `#hash` e `#eql?` sono tra i metodi più caldi in Ruby, e osservarli cambia completamente sia il costo sia il profilo dei falsi positivi. `strict: true` lo chiude e accetta quel costo. Il **detector** intercetta la stessa forma prima che qualsiasi byte venga caricato, che è il posto più economico dove intercettarla, e la limitation notice della guardia lo indica esplicitamente.

Gli altri limiti, dichiarati nella stessa notice anziché in una nota a piè di pagina:

- **Copre solo la finestra del load.** Una classe che non porta alcun hook viene istanziata liberamente e scatta ogni volta che l'applicazione la tocca successivamente. Questo è al di fuori di qualsiasi finestra che questa guardia possa vedere, ed è esattamente come funziona la primitiva `erb-def-method`.
- **Ha scope per thread.** Un load su un altro thread non è coperto.
- **Il suo costo non è un moltiplicatore.** Abilitare un `TracePoint` costa all'incirca 46 microsecondi per load, abbastanza vicino a una costante, quindi il rapporto è deciso da quanto lavoro fa il load stesso: 1.0x su un documento da 488 KB, 1.1x su 46 KB, 20.7x su una sessione da 142 byte, **40.4x su un session cookie da 45 byte**. Un session cookie è esattamente ciò che questo laboratorio deserializza, quindi il numero è pubblicato con la dimensione del payload allegata. Una versione precedente della ricerca di questo progetto registrava "1.4x" da una singola misura con payload grande, e quella cifra è ora marcata con una correzione datata, perché un rapporto citato senza la sua dimensione di payload si legge come un'approvazione che non si è guadagnato.

## Il target

Un'app Sinatra su Rack 3, in un container senza alcuna route fuori dall'host, root filesystem in sola lettura, capability eliminate, `no-new-privileges`, un limite di pids, e un unico `tmpfs` scrivibile per il file canary.

Quattro endpoint, disposti come due coppie abbinate così che la differenza sia a un `curl` di distanza:

```
   POST /session       issue a benign session cookie

   GET  /render        Marshal.load, then compile the template      VULNERABLE
   GET  /render/safe   inspect the stream first, then load          DEFENDED

   GET  /yaml/unsafe   YAML.unsafe_load the same session            VULNERABLE
   GET  /yaml/safe     inspect, then YAML.safe_load                 DEFENDED

   GET  /canary        report whether the canary file exists
```

L'abbinamento è la lezione di [01-CONCEPTS.md](./01-CONCEPTS.it.md) resa eseguibile. `/render` e `/yaml/unsafe` raggiungono entrambi l'esecuzione di codice con lo stesso oggetto ERB. `/yaml/safe` lo rifiuta **per tag**, prima della rivivificazione, perché l'allowlist di Psych è un vero veto. `/render/safe` può solo ispezionare i byte e sperare, perché quella di Marshal non lo è.

Nota cosa fa `/render/safe` dopo che il detector accetta: chiama comunque `Marshal.load` dentro un `rescue`, controlla comunque che il risultato sia effettivamente un hash di sessione, e solo allora compila. Tre layer, perché il primo esplicitamente non è un confine.

Il gate del target pilota tutto questo da un **secondo container sulla stessa rete interna** anziché da una porta dell'host. Le reti Docker `--internal` bloccano le porte pubblicate, quindi un gate che provasse a fare `curl` dall'host fallirebbe o costringerebbe la rete a non essere isolata. Attaccare il target dall'interno della rete mantiene reale l'isolamento dell'egress.

## Limiti: quattordici assi, tutti attivi di default

Il parser limita quattordici risorse separate, e ognuna di esse è applicata a meno che tu non faccia opt-out:

```
   stream bytes        nodes               registered objects    symbol definitions
   collection entries  scalar bytes        total scalar bytes    object links
   symbol references   symbol name bytes   class name bytes      instance variables
   struct members      nesting depth
```

Due note di design. I limiti sono **attivi di default e con opt-out**, non disattivi di default e con opt-in, perché il chiamante che ne ha più bisogno è quello che non ha mai letto questa pagina. E `Limits.permissive` fissa comunque `max_depth`, perché la ricorsione illimitata in un parser a discesa ricorsiva è uno stack overflow anziché un parsing lento, e "permissive" non dovrebbe significare "manda in crash il processo."

Falliscono prima di allocare:

```
DepthLimitError:      exceeded depth 4
LimitExceededError:   stream bytes 110 exceeds 8
```

## Il gate

Sei fasi, eseguite da `just gate`, 79 asserzioni, che devono passare tutte:

| Fase | Cosa dimostra |
|---|---|
| `check` | le sette test suite più gli script di controllo autonomi |
| `matrix` | su quali versioni di Ruby scatta la chain, eseguendola in ciascuna |
| `exploit` | il confine della CVE in entrambe le direzioni: scatta sull'image vulnerabile, bloccata su quella patchata, a un `docker pull` di distanza |
| `detector` | il corpus avversariale, ogni payload e cosa ha deciso il detector |
| `target` | sfruttamento end-to-end su HTTP reale, più isolamento, error-leak e il controllo dei tag di sink |
| `package` | l'artefatto costruito, l'audit del manifest, il percorso di installazione, l'identità della release, i controlli negativi |

**Ogni fase porta un input che deve rifiutare.** Un gate con solo casi positivi non può rilevare un checker che dice sempre sì. Il corpus del detector è l'esempio più chiaro, perché gli accept e i reject stanno nella stessa tabella:

```
  object_in_value_position_control       accept Foo
  hash_key_object                        reject Foo        stream puts "Foo" in a hash key, so its #hash runs d
  hash_key_object_link                   reject Foo        stream puts "Foo" in a hash key, so its #hash runs d
  hash_key_struct                        reject Foo        stream puts "Foo" in a hash key, so its #hash runs d
  hash_key_extended                      reject Foo,Comparable stream puts "Comparable" in a hash key, so its #hash
  hash_key_user_class_over_array         reject Foo        stream puts "Foo" in a hash key, so its #hash runs d
  hash_key_bare_array                    reject Foo        stream puts "Foo" in a hash key, so its #hash runs d
  hash_key_nested_bare_array             reject Foo        stream puts "Foo" in a hash key, so its #hash runs d
  hash_key_user_class_over_string        reject Foo        stream puts "Foo" in a hash key, so its #eql? runs d
  hash_key_extended_over_string          reject Comparable stream puts "Comparable" in a hash key, so its #eql?
  hash_key_regexp                        accept
  hash_key_array_of_primitives           accept
  hash_key_empty_array                   accept
  range_with_object_endpoints            reject Foo,Range  stream puts "Foo" in a Range endpoint, so its #<=> r
  range_with_primitive_endpoints         accept Range
```

Leggi gli accept con la stessa attenzione dei reject. `hash_key_regexp`, `hash_key_array_of_primitives` e `hash_key_empty_array` sono tutti payload in posizione di chiave che **non** devono essere rifiutati, e sono ciò che impedisce alle regole sulla posizione di chiave di degenerare in "rifiuta qualsiasi cosa in una chiave."

`object_in_value_position_control` è il più affilato di tutti. La stessa classe, nello stesso stream, in posizione di **valore** anziché di chiave, deve essere accettata. Senza di esso, un detector che rifiutasse semplicemente ogni stream che menziona `Foo` passerebbe ogni caso di rifiuto nella tabella.

C'è un ulteriore raffinamento che vale la pena nominare, perché c'è voluto un bug reale per impararlo: **due controlli che rifiutano entrambi lo stesso input si fanno da alibi a vicenda.** Se un controllo viene rifiutato dal controllo sui tag *e* dal controllo sulle classi, sventrare uno dei due lascia il gate verde. Quindi il gate del target porta un documento YAML che l'inspector approva e che Psych comunque rifiuta, specificamente per isolare i due layer l'uno dall'altro.

## Dove andare dopo

[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) percorre il codice: byte in un grafo di nodi, un grafo di nodi in una decisione, un oggetto vivo in un payload, e la proxy chain di ActiveSupport end to end come pezzo forte.
