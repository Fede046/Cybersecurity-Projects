<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# marshalsea: Concetti

Questo capitolo è la teoria su cui è costruito il laboratorio. Ogni affermazione al suo interno è o ricondotta a una fonte primaria con un URL, o verificata eseguendo Ruby in un container pinnato, e dice quale delle due. Dove un'affermazione ampiamente ripetuta si è rivelata sbagliata, la correzione è qui invece di un'omissione silenziosa.

## Inizia con quella che tutti sbagliano

**L'esempio più citato di insecure deserialization non era insecure deserialization.**

Cerca "insecure deserialization breach" ed Equifax torna in cima. La violazione del 2017, 147 milioni di persone, il più grande accordo per data breach mai registrato. È nelle slide, nel materiale dei corsi, nelle risposte ai colloqui e nel paragrafo introduttivo di un enorme numero di write-up su questa esatta classe di bug.

Era **OGNL expression injection**, e NVD la classifica come **CWE-755, Improper Handling of Exceptional Conditions**. Non CWE-502.

Ecco [il record NVD per CVE-2017-5638](https://nvd.nist.gov/vuln/detail/CVE-2017-5638), testualmente:

> "The Jakarta Multipart parser in Apache Struts 2 2.3.x before 2.3.32 and 2.5.x before 2.5.10.1 has incorrect exception handling and error-message generation during file-upload attempts, which allows remote attackers to execute arbitrary commands via a crafted Content-Type, Content-Disposition, or Content-Length HTTP header, as exploited in the wild in March 2017 with a Content-Type header containing a #cmd= string."

Il meccanismo: un header `Content-Type` malformato solleva un'eccezione, il messaggio dell'eccezione viene costruito da una routine che valuta OGNL, e l'espressione OGNL dell'attaccante viene eseguita. **Nessun grafo di oggetti viene ricostruito in alcun momento. Non c'è alcun payload serializzato da nessuna parte.** È più vicino alla template injection che alla deserializzazione.

CISA concorda, e vale la pena vedere con quanta precisione. Nel catalogo KEV, CVE-2017-5638 è intitolata "Apache Struts *Remote Code Execution* Vulnerability" e mappata a CWE-20, mentre la sua gemella CVE-2017-9805 è intitolata "Apache Struts *Deserialization of Untrusted Data* Vulnerability" e mappata a CWE-502. Stesso prodotto, stesso anno, classe di bug diversa. Qualcuno le ha nominate diversamente di proposito.

### Come distinguerle

Struts ha avuto due famose RCE nel 2017, a sei mesi di distanza, e solo la seconda è deserializzazione:

| | CVE-2017-5638 | CVE-2017-9805 |
|---|---|---|
| NVD published | 2017-03-10 | 2017-09-15 |
| Componente | Jakarta Multipart parser | REST plugin |
| Meccanismo | OGNL injection durante la generazione del messaggio d'errore | Deserializzazione XStream senza type filtering |
| NVD CWE | **CWE-755** | **CWE-502** |
| CVSS v3.1 | 9.8 | 8.1 |
| Usata contro Equifax | **Sì** | **No** |

### Perché il mito è così resistente

La timeline lo spiega completamente, e la Apache Software Foundation ha documentato la correzione essa stessa. CVE-2017-9805, quella che *è* deserializzazione, fu divulgata il 2017-09-04. Equifax annunciò la violazione il 2017-09-07. Tre giorni di distanza. Le prime notizie tirarono a indovinare ragionevolmente la CVE fresca. Equifax corresse il resoconto il 2017-09-13. L'ASF [pubblicò un media alert](https://news.apache.org/foundation/entry/media-alert-the-apache-software) il 2017-09-14 che esiste specificamente per correggere questo:

> "Following this announcement, additional claims stated that the breach was caused by **CVE-2017-9805**, an exploit in Apache Struts that was disclosed on 4 September 2017."
>
> "On 13 September 2017, Equifax issued a statement confirming that 'The vulnerability was Apache Struts **CVE-2017-5638**'."

La correzione perse. L'articolo originario di Quartz porta ancora la propria nota di rettifica e il suo URL slug dice ancora "nine-year-old security flaw" mentre il titolo è stato riscritto.

Quattro fonti primarie sono state aperte e sottoposte a ricerca testuale per questo capitolo, e nessuna di esse usa la parola:

1. **GAO-18-559** (2018-08-30, 40 pagine): "deserialization," "serialization," e "OGNL" appaiono **zero volte**, così come qualsiasi numero di CVE.
2. **US House Committee on Oversight majority staff report** (dicembre 2018, 96 pagine): nomina esplicitamente CVE-2017-5638 e cita direttamente NVD. "Deserialization" appare **zero volte**.
3. **Comunicato stampa di Equifax stessa** (2017-09-15): "The attack vector used in this incident occurred through a vulnerability in Apache Struts (CVE-2017-5638)."
4. **Comunicato stampa dell'atto d'accusa del DOJ** (2020-02-10): "the defendants exploited a vulnerability in the Apache Struts Web Framework." "Deserialization": zero occorrenze.

Questa è una storia migliore di quanto fosse il mito: **un'inferenza plausibile, fatta sotto pressione temporale, che ha corso più veloce della propria rettifica per nove anni.** E insegna qualcosa che il mito non può, cioè come guardare una CVE e distinguere l'expression injection dalla deserializzazione.

Se vuoi la versione corretta in una frase: *il più grande accordo per data breach mai registrato è venuto da una OGNL injection in Struts non patchata, una classe di bug diversa che viene frequentemente etichettata erroneamente come deserializzazione.*

## Quindi cos'è la deserializzazione, davvero

Serializzare un oggetto scrive il suo stato in byte. Deserializzare rilegge quei byte in un oggetto vivo. La trappola è che il secondo passaggio non è una copia. Per ricostruire un oggetto, il runtime deve eseguire codice:

```
   Marshal.dump           bytes on the wire            Marshal.load
   ────────────           ─────────────────            ────────────
   object state    ──>    "\x04\bU:\x15Gem::..."  ──>  allocate the class
                                                       CALL its marshal_load
                                                       CALL #hash on hash keys
                                                       CALL #<=> on Range ends
                                                       return the object
```

Ognuno di quei `CALL` è un metodo che l'attaccante ha scelto scegliendo i byte. Questa è l'intera vulnerabilità. `Marshal.load` non sta "facendo parsing"; è un piccolo interprete, pilotabile dall'attaccante, sul tuo grafo di classi caricate.

La documentazione di Ruby stessa per `Marshal` lo dice chiaramente, e lo fa da anni:

> "By design, `Marshal.load` can deserialize almost any class loaded into the Ruby process. In many cases this can lead to remote code execution if the `Marshal` data is loaded from an untrusted source. As a result, `Marshal.load` is not suitable as a general purpose serialization format and you should never unmarshal user supplied input or other untrusted data."

Python dice la stessa cosa riguardo a `pickle`:

> "The `pickle` module **is not secure**. Only unpickle data you trust. It is possible to construct malicious pickle data which will **execute arbitrary code during unpickling**. Never unpickle data that could have come from an untrusted source, **or that could have been tampered with**."

PHP lo dice riguardo a `unserialize()`:

> "**Do not pass untrusted user input to unserialize() regardless of the `options` value of `allowed_classes`.** Unserialization can result in code being loaded and executed due to object instantiation and autoloading."

Tre linguaggi, tre documentazioni ufficiali, che dicono tutte la stessa cosa da un decennio. Tutti e tre ancora a generare CVE. **Il fallimento non è la documentazione mancante.** Quell'inquadramento è più utile di qualsiasi punteggio di severità, ed è il motivo per cui questo laboratorio spende il suo impegno sul *perché le correzioni ovvie falliscono* piuttosto che sul ripetere l'avvertimento.

Nota la clausola che la maggior parte dei write-up elimina dalla citazione di Python: "or that could have been tampered with." È la metà che motiva la frase su `hmac` che la segue nella documentazione reale. Il punto didattico è preciso: **hmac affronta la manomissione, non l'origine non fidata.** Firmare un payload prodotto da un attaccante che possiede la chiave non ti compra esattamente nulla.

## La gadget chain

Ecco la parte che fa sembrare questa classe di bug magia, e la parte che smette di sembrare magia una volta che ne vedi la forma.

Un payload non contiene codice. Contiene una descrizione di un grafo di oggetti. Quello che fa l'attaccante è scegliere un insieme di classi che sono *già caricate nel tuo processo*, disporle in modo che far rivivere una chiami un metodo sulla successiva, e continuare finché l'ultima fa qualcosa di utile. Ogni classe in quella sequenza è un **gadget**. La sequenza è una **chain**.

```
   attacker controls only the SHAPE of the graph
   ┌───────────────────────────────────────────────────────────┐
   │  Hash                                                     │
   │   └─ key: DeprecatedInstanceVariableProxy                 │
   │        @instance = ERB (with attacker-controlled @src)    │
   │        @method   = :def_module                            │
   └───────────────────────────────────────────────────────────┘
                             │
        Marshal.load rebuilds the Hash, which rehashes its keys
                             │
                             v
        proxy#hash  ->  method_missing  ->  @instance.def_module
                             │
                             v
        ERB#def_module -> ERB#def_method -> module_eval(@src)
                             │
                             v
                     attacker's Ruby runs
```

In nessun punto di quel payload c'è un'istruzione che dice "esegui un comando". Ogni passaggio è un normale metodo che fa esattamente ciò per cui è stato scritto. `#hash` dovrebbe essere chiamato quando ricostruisci un hash. `method_missing` dovrebbe inoltrare. `def_module` dovrebbe compilare un template. L'attaccante ha fornito solo la disposizione.

Questo è il motivo per cui la Apache Software Foundation si rifiutò di trattare Commons Collections come vulnerabile nel 2015, e la loro dichiarazione è l'articolazione più chiara dell'idea che chiunque abbia pubblicato:

> "this is not the only known and especially not unknown useable gadget. So replacing your installations with a hardened version of Apache Commons Collections will not make your application resist this vulnerability."

Tutte e tre le famose CVE Java del 2015 (CVE-2015-4852 per Oracle WebLogic, CVE-2015-7501 per Red Hat JBoss, CVE-2015-6420 per Cisco) hanno come scope i *vendor a valle*. **Apache Commons Collections di per sé non ricevette mai una CVE**, perché `InvokerTransformer` stava facendo esattamente ciò per cui era documentato. La vulnerabilità era `readObject()` su byte non fidati. La libreria era munizione.

Prima che arrivasse il fix a monte, la rimediazione di ultima istanza era cancellare fisicamente i file di classe `InvokerTransformer`, `InstantiateFactory` e `InstantiateTransformer` dai jar distribuiti. Questo ti dice quanto bene stesse andando la strategia del "basta metterlo in allowlist".

## L'asse che conta: gated contro ungated

Questa è l'idea organizzatrice dell'intero laboratorio, non è messa per iscritto nella letteratura Ruby pubblicata, ed è stata stabilita qui per esecuzione su Ruby 3.0.7, 3.1.7, 3.3.8, 3.3.12, 3.4.10 e 4.0.6, con controlli negativi.

I deserializzatori di Ruby raggiungono gli oggetti controllati dall'attaccante in due modi diversi, e la differenza decide se una classe sia utilizzabile del tutto:

```
GATED      the deserializer calls respond_to?(m, true) FIRST
           false -> TypeError, chain dead
           reachable through method_missing ONLY if respond_to_missing? also answers true

UNGATED    the deserializer calls the method directly
           no gate, no check
           method_missing catches it for free
```

La tabella verificata per `Marshal.load`, identica su Ruby da 3.0.7 a 4.0.6:

| Metodo | Gated? | Quando `Marshal.load` lo invoca |
|---|---|---|
| `marshal_load(data)` | **GATED** su `respond_to?(:marshal_load, true)` | L'oggetto è stato dumpato via `marshal_dump` (il tag `U`) |
| `self._load(str)` | **GATED**, sulla classe | L'oggetto è stato dumpato via `_dump` (il tag `u`). Il gate è sulla singleton class |
| `respond_to_missing?(m, true)` | *è* il gate | Chiamato prima dei due sopra ogni volta che il metodo non è concretamente definito. Esso stesso un sink raggiungibile |
| `method_missing(m)` | eredita il gate | Scatta per i due sopra solo se `respond_to_missing?` ha restituito true |
| `hash` | **UNGATED** | L'oggetto è una chiave di Hash, o annidato in un Array o Set usato come chiave |
| `eql?(other)` | **UNGATED** | Solo su collisione di hash-bucket tra due chiavi |
| `<=>(other)` | **UNGATED** | `Range#marshal_load` valida i suoi estremi, quindi scatta su entrambi gli estremi di un Range limitato |

E i negativi verificati, che contano altrettanto. `Marshal.load` **non** invoca mai direttamente nessuno di questi: `to_s`, `to_str`, `to_ary`, `to_hash`, `to_proc`, `to_int`, `inspect`, `==`, `coerce`, `each`, `call`, `<<`, `+`, `length`, `size`, `freeze`.

Quella lista negativa è facile da sbagliare e costosa da sbagliare. `Gem::RequestSet::Lockfile#to_s` è un vero passaggio nella universal chain pubblicata, quindi è tentante catalogare `to_s` come un sink. Ma è chiamato *da un altro gadget*, non da `Marshal`. **`to_s` è un link, mai un entry point.** Uno scanner che confonde i due produce un diluvio di falsi positivi, che è il motivo per cui questo progetto li modella come due tipi di nodo diversi e riporta 53 link separatamente dai 140 entry point.

### Il guadagno: la stessa classe, esiti opposti

Prendi una classe proxy che annulla la definizione di ogni metodo pubblico, la forma che Rails usa per i suoi deprecation proxy. Raggiunta attraverso un sink **gated**:

```
1) marshal_load via method_missing (respond_to_missing? => true)  -> fires
2) NEGATIVE: respond_to_missing? => false                         -> TypeError, chain dead
3) NEGATIVE: no marshal_load, no method_missing                   -> TypeError, chain dead
4) fully-wiped proxy                                              -> TypeError, chain dead
```

La riga 4 è il guadagno. Una classe che annulla la definizione di tutto **non può** essere un entry point per `marshal_load`, perché ha annullato `respond_to?` senza fornire `respond_to_missing?`.

Ora la stessa classe azzerata attraverso un sink **ungated**:

```
A) wiped proxy as a Hash key      [UNGATED #hash]  -> method_missing(hash) | ok
B) wiped proxy inside an Array key [UNGATED #hash]  -> method_missing(hash) | ok
C) wiped proxy inside a Set        [UNGATED #hash]  -> method_missing(hash) | ok
D) two colliding wiped keys                         -> MM(hash) | MM(hash) | MM(eql?) | ok
```

Stessa classe. Il percorso gated muore, il percorso ungated scatta. Tutto ciò che fa lo scanner è costruito su quella distinzione.

Vale la pena fermarsi sulla riga D, perché è quella che ha morso questo progetto. `#eql?` scatta solo su una **collisione di bucket**, il che significa che una fixture di test con una singola chiave non può mai osservarlo. Una versione precedente dell'oracolo differenziale di questo laboratorio dumpava `{ key => nil }`, una chiave, nessuna collisione, e `#eql?` era inosservabile per costruzione. Tre bypass del detector sono stati rilasciati sotto una suite verde a causa di ciò. L'oracolo ora usa hash a due chiavi in collisione.

### Psych non è Marshal, e la differenza è più netta di quanto sembri

Lo stesso esercizio per `YAML.unsafe_load`, verificato identico su Psych 3.3.2, 4.0.4, 5.1.2, 5.2.2 e 5.3.1:

| Metodo | Gated? | Trigger |
|---|---|---|
| `init_with(coder)` | **soft gate**: `o.respond_to?(:init_with)` come normale chiamata Ruby, quindi `method_missing` lo intercetta | qualsiasi mapping `!ruby/object:X` |
| `marshal_load(data)` | gated su `respond_to?(:marshal_load)` | il tag `!ruby/marshalable:X` |
| `hash` | **UNGATED** | oggetto usato come chiave di mapping YAML |
| `==` | **UNGATED** | inserimento della chiave di mapping |
| `[]=(k, v)` | **UNGATED** | `!ruby/hash:Subclass`, dove Psych chiama `[]=` sulla sottoclasse allocata |

La differenza più netta, e una che questa sessione di ricerca non ha trovato da nessuna parte nella letteratura: Psych chiama `instance.respond_to?(:init_with)` come **normale chiamata di metodo Ruby**, non attraverso il `rb_obj_respond_to` a livello C che usa Marshal. Su un proxy con i metodi cancellati, `respond_to?` stesso cade in `method_missing`, che restituisce qualcosa di truthy, quindi Psych chiama poi `init_with`, che cade anch'esso in `method_missing`.

**Una classe proxy con i metodi completamente cancellati è un entry point YAML valido e un entry point Marshal non valido.** Classe identica, esito opposto. Il laboratorio ha un test che asserisce esattamente questo in entrambe le direzioni, ed è il motivo per cui lo scanner valuta gli entry point **per formato** anziché globalmente: la scan di oggi trova 29 entry point raggiungibili attraverso Marshal e 33 attraverso Psych, e quei due insiemi non sono annidati.

### Un punto cieco che vale la pena conoscere

Una sottoclasse di `String` che fa override di `#hash` ed è usata come chiave di hash Marshal **non ha mai `#hash` chiamato.** Il `rb_any_hash` interno di Ruby tratta come caso speciale `T_STRING` e fa l'hash dei byte direttamente. Verificato per esecuzione su Ruby 4.0.6:

```
loading a String-subclass key dispatched: []
loading an Object-subclass key dispatched: [:"Object subclass"]
```

Conseguenza per chiunque scriva uno scanner: **le sottoclassi di String sono morte come entry point `#hash`.** Riportale e produci falsi positivi. Le sottoclassi di Array, Hash, Object e Struct fanno tutte dispatch normalmente. (Lo stesso fast path C plausibilmente copre `Symbol`, `Integer`, `Float`, `nil`, `true` e `false`, ma questi non possono essere sottoclassati, quindi non è stato possibile testarlo e non viene affermato qui.)

## Le due allowlist

Questa è la spina dorsale del progetto. Tutti insegnano "non deserializzare input non fidato". Quasi nessuno spiega perché la correzione ovvia fallisce, e la risposta è un fatto specifico e verificabile su dove si trova una chiamata di funzione.

`Marshal.load` accetta una proc. È tentante usarla come allowlist:

```ruby
Marshal.load(bytes, ->(obj) { raise SecurityError unless ALLOWED.include?(obj.class); obj })
```

**Questo non funziona.** Ecco il caso `TYPE_USRMARSHAL` da `marshal.c` sul master di ruby/ruby, con l'ordinamento annotato:

```c
case TYPE_USRMARSHAL:
    VALUE name = r_unique(arg);
    VALUE klass = path2class(name);
    ...
    v = obj_alloc_by_klass(klass, arg, &oldclass);   /* 1. allocate */
    ...
    v = r_entry(v, arg);
    data = r_object(arg);
    load_funcall(arg, v, s_mload, 1, &data);         /* 2. YOUR GADGET RUNS */
    ...
    v = r_post_proc(v, arg);                         /* 3. proc finally sees it */
    break;
```

`r_post_proc` è dove viene invocata la tua proc. È due istruzioni dopo `load_funcall(... s_mload ...)`, che è la chiamata che esegue `marshal_load`. Nel momento in cui alla tua proc viene passato l'oggetto e solleva, il gadget è già scattato. Conferma eseguita su Ruby 4.0.6:

```
EXP-B: allowlist proc that permits everything EXCEPT Inner
  proc raised: blocked Inner
  side effects fired BEFORE the proc could veto: ["Inner#marshal_load RAN"]
  => allowlist proc FAILED TO PREVENT the callback
```

Non c'è nemmeno una keyword di allowlist a cui ripiegare. `Marshal.load` accetta esattamente `proc` e `freeze:`:

```
Marshal.load(data, permitted_classes: [String])
  ArgumentError: unknown keyword: :permitted_classes
```

L'allowlist di Psych è genuinamente un veto, per esattamente un motivo: controlla il tag **prima** della rivivificazione. Stesso intento, esito opposto, deciso interamente da dove si trova il controllo.

```
Marshal   bytes ──> build the object ──> RUN its hook ──> your allowlist runs
                                         ^^^^^^^^^^^^     too late, an autopsy

Psych     bytes ──> CHECK the tag ──> refuse
                    ^^^^^^^^^^^^^^   in time, a bouncer
```

Il laboratorio lo rende eseguibile anziché asserirlo. Un test carica lo stesso payload concettuale attraverso entrambi i deserializzatori e asserisce su quali callback sono scattati:

- `Psych.safe_load(document, permitted_classes: [])` solleva `Psych::DisallowedClass` e `init_with` **non è mai stato eseguito**.
- `Marshal.load(blob, ->(o) { o })` ritorna, e `marshal_load` **è già stato eseguito**.

L'applicazione target espone entrambi così puoi fare `curl` della differenza. `/render` e `/yaml/unsafe` raggiungono entrambi l'esecuzione di codice con lo stesso oggetto ERB. `/yaml/safe` lo rifiuta per tag. `/render/safe` può solo ispezionare prima i byte e sperare.

Quell'asimmetria è la posizione di Ruby nell'ecosistema più ampio, e non è lusinghiera: **`Marshal` non ha alcun JEP 290, nessun `weights_only`, nessun `allowed_classes` e nessun momento `.NET 9`.** Psych ha ottenuto un default sicuro in Ruby 3.1. `Marshal` ha ottenuto un avvertimento nella documentazione. Questo è il motivo per cui questo laboratorio esiste e il motivo per cui prende di mira specificamente Marshal.

## L'esempio svolto: CVE-2026-41316

Vecchio di quattro mesi al momento della scrittura, e il caso didattico più pulito disponibile perché la patch è di tre righe e puoi leggerle tutte.

- **Advisory**: [ruby-lang.org, 2026-04-21](https://www.ruby-lang.org/en/news/2026/04/21/erb-cve-2026-41316/). GHSA-q339-8rmv-2mhv. NVD pubblicata 2026-04-23.
- **CVSS v3.1 8.1.** **CWE-502 e CWE-693 (Protection Mechanism Failure).** Il doppio mapping è il nocciolo della storia.
- **Affette**: erb `< 4.0.3.1`, `= 4.0.4`, `>= 5.0.0 < 6.0.1.1`, `>= 6.0.2 < 6.0.4`. **Patchate**: 4.0.3.1, 4.0.4.1, 6.0.1.1, 6.0.4.
- **Credit**: TristanInSec.
- **Precondizione, citata dall'advisory:**
  > "Any Ruby application that calls `Marshal.load` on untrusted data AND has both `erb` and `activesupport` loaded is vulnerable to arbitrary code execution."

Tre database danno tre date diverse per essa (rubysec 2026-04-13, ruby-lang 2026-04-21, NVD 2026-04-23), e NVD assegna CWE-502 più CWE-693 mentre la pagina GHSA elenca solo CWE-693. Cita quella che hai effettivamente estratto.

**Il meccanismo.** Ruby 2.7.0 ha aggiunto una guardia sulla variabile d'istanza `@_init` così che un oggetto ERB ricostruito attraverso `Marshal.load` si rifiutasse di valutare il suo template. `ERB#result` ed `ERB#run` la controllano. `ERB#def_method`, `ERB#def_module` ed `ERB#def_class` valutavano il sorgente del template **senza** controllarla.

Leggere il sorgente distribuito in un container pinnato lo rende concreto. In erb 4.0.4.1, una versione patchata, l'assegnazione e i controlli appaiono così:

```
in def initialize(...)    | @_init = self.class.singleton_class
in def result(b=...)      | unless @_init.equal?(self.class.singleton_class)
in def def_method(mod,..) | unless @_init.equal?(self.class.singleton_class)
```

E il motivo per cui un controllo aggiunto corregge tutti e tre i metodi, letto dal sorgente patchato:

```ruby
def def_module(methodname='erb')
  mod = Module.new
  def_method(mod, methodname, @filename || '(ERB)')
  mod
end

def def_class(superklass=Object, methodname='result')
  cls = Class.new(superklass)
  def_method(cls, methodname, @filename || '(ERB)')
  cls
end
```

`def_module` e `def_class` delegano entrambi a `def_method`, quindi proteggere `def_method` chiude l'intera famiglia. Ecco come appare "correggi la guardia, non il sintomo" come diff.

**La primitiva dell'exploit**, che spiega perché questi tre metodi erano *sfruttabili* anziché meramente non protetti. `def_method` avvolge il sorgente del template in un `def <methodname> ... end` generato. Un attaccante che controlla `@src` lo prefissa con `end\n`, chiudendo in anticipo il wrapper generato, così che il codice iniettato venga eseguito al momento di `module_eval`, durante la definizione, anziché aspettare che qualcuno chiami il metodo. Questo laboratorio costruisce esattamente questo, e puoi stamparlo:

```ruby
Marshalsea::Chains::ErbDefMethod.canary("/tmp/canary", "pwned").src
# => "#\nend\nFile.write(\"/tmp/canary\", \"pwned\")\ndef _marshalsea_unused\n"
```

`ERB#def_method` non si limita ad anteporre il wrapper. La sua riga effettiva, letta dal sorgente distribuito, è:

```ruby
src = self.src.sub(/^(?!#|$)/) { "def #{methodname}\n" } << "\nend\n"
```

Inserisce `def <name>` prima della **prima riga che non è né un commento né vuota**. Quella regex esiste perché un template ERB genuinamente compilato inizia con un commento magico di encoding, che deve restare sulla riga uno:

```ruby
ERB.new("hello <%= 1 %>").src
# => "#coding:UTF-8\n_erbout = +''; _erbout.<< \"hello \".freeze; ..."
```

Quindi il `#` iniziale del payload sta impersonando quel commento magico, il che spinge il punto di inserimento giù sull'`end`. Eseguire la stessa sostituzione sul payload produce esattamente questo:

```ruby
#                                     # the fake magic comment, so the `sub` skips line 1
def render_it                         # the wrapper lands HERE, on the `end` line
end                                   # and closes immediately: an empty method
File.write("/tmp/canary", "pwned")    # now at module_eval top level, runs during eval
def _marshalsea_unused                # a second empty method, which eats the
                                      # wrapper's own appended "\nend\n"
end
```

Verificato eseguendolo: il payload scatta durante `eval`, ed entrambi `render_it` e `_marshalsea_unused` sono definiti come metodi vuoti in seguito.

```
fired during eval: [:RAN_AT_EVAL_TIME]
methods defined:   [:_marshalsea_unused, :render_it]
```

Questo è l'intero trucco. Niente aspetta che qualcuno chiami `render_it`.

**`def_module` non prende argomenti**, che è ciò che lo rende raggiungibile da una gadget chain anziché solo da codice applicativo cooperante, ed è il motivo per cui questo laboratorio fornisce due payload con etichette diverse:

| Payload | Tipo | Entra attraverso | Scatta quando | Richiede |
|---|---|---|---|---|
| `erb-def-module` | **chain** | `#hash` ungated su una chiave di hash | dentro `Marshal.load`, nessuna chiamata applicativa | activesupport caricato nel target |
| `erb-def-method` | **primitiva** | il bypass della guardia `@_init` | solo quando l'applicazione chiama `def_method` | niente |

Chiamare la seconda una "chain" sarebbe una menzogna, e la distinzione insegna qualcosa di reale: il call site pericoloso può vivere nel *tuo* codice. L'applicazione target del laboratorio chiama `template.def_method(...)` se l'oggetto deserializzato vi risponde, che è una cosa plausibile per un layer di caching dei template e che è esattamente la cooperazione di cui la primitiva ha bisogno.

**Perché CWE-693 conta pedagogicamente.** Una mitigazione esisteva, era deliberata, era vecchia di sei anni, e copriva due dei cinque entry point. Le guardie parziali si leggono come sicurezza e si auditano come sicurezza. Questo è l'argomento più forte disponibile a sostegno della posizione che mettere in allowlist i singoli sink è un gioco perdente.

Il gate dell'exploit del laboratorio dimostra entrambe le metà tirando due image reali a una versione di distanza: la chain scatta su quella vulnerabile ed è bloccata su quella patchata, con il confine asserito in entrambe le direzioni.

## Cosa è effettivamente successo in Ruby

Versione breve, tutta ricondotta a fonti primarie.

**CVE-2013-0156, quella che ha fatto preoccupare il mondo Ruby.** Il parser di parametri XML di Rails permetteva a una richiesta di dichiarare il *tipo* di un parametro, e la lista supportata includeva `yaml` e `symbol`. Quindi un body di richiesta poteva istruire il framework a passare byte controllati dall'attaccante al parser YAML non sicuro, prima che venisse eseguito qualsiasi codice applicativo, su **ogni** controller:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bang type="yaml">--- !ruby/object:Time {}
</bang>
```

Cinque righe, e contiene l'intera classe di vulnerabilità. Il workaround dell'advisory a monte cancella `"symbol"` e `"yaml"` da `ActiveSupport::XmlMini::PARSING`, ed è schietto sul resto:

> "there is no fix for YAML object injection"

**rubygems.org fu compromesso il 2013-01-30** usando questa classe di falla, ed è documentato dai manutentori stessi. RubyGems.org chiamava `YAML.load` sul `metadata.gz` delle gem caricate, quindi un attaccante caricò una gem i cui metadata istanziavano oggetti ed esfiltravano file di configurazione. Secondo [il resoconto dei manutentori stessi](https://blog.rubygems.org/2013/01/31/data-verification.html), **nessuna API key fu effettivamente esposta**, perché il servizio girava su Heroku e teneva i segreti in `ENV` anziché nei file di configurazione. Un incidente dello stile di deployment, non un controllo. La risposta richiese circa 53 ore e includeva la ri-verifica dei checksum SHA512 per ogni gem rispetto ai mirror della community.

Nota il vettore: non una richiesta HTTP in un'app Rails, ma l'**elaborazione dei metadata delle gem**. Stesso sink, porta diversa. Non c'è una cifra in dollari e non è necessario che ce ne sia una. Il costo fu la fiducia a livello di ecosistema.

**La coppia della key-management, a tredici anni di distanza, e vanno insieme.**

- **CVE-2019-5420** (Rails, CVSS 9.8): in modalità development Rails derivava `secret_key_base` dal nome stesso dell'applicazione, che un attaccante poteva recuperare richiedendo una route non valida. Con la chiave, conia un payload correttamente firmato e ottieni RCE. **NVD assegna CWE-330 e CWE-77, non CWE-502.** La causa principale era un segreto prevedibile; la deserializzazione era meramente il payload. Il fix fu la generazione della chiave, non un cambiamento del serializzatore.
- **CVE-2026-39324** (rack-session, CVSS 9.3): la chiave andava bene, e il *percorso di fallimento* cadeva aperto. Citando NVD: "If cookie decryption fails, the implementation falls back to a default decoder instead of rejecting the cookie." Il fail-open ha battuto la crittografia.

Insieme sostengono la tesi che "firma e cifra il payload" è necessario e dimostrabilmente non sufficiente: ciò che fa il tuo codice quando la verifica **fallisce** è parte del controllo. (Le applicazioni Rails esplicitamente *non* sono affette da quella di rack-session. Usa un percorso di codice diverso. Sbagliare questo disinformerebbe ogni lettore Rails.)

**CVE-2022-32224, la forma moderna.** Il `serialize :options` di Active Record defaultava a YAML e deserializzava con `YAML.unsafe_load`. Un attaccante che può scrivere nel database, tipicamente via SQL injection, scala a RCE quando la riga viene riletta. Due cose la rendono il miglior caso didattico moderno. È un **sink di secondo ordine**: i dati non fidati arrivano dal tuo stesso database, quindi qualsiasi threat model che traccia il confine di fiducia al bordo HTTP la manca interamente. E **il fix ha rilasciato un opt-out**, `use_yaml_unsafe_load`, perché il safe-by-default rompeva applicazioni reali. Quella tensione tra un default sicuro e una scappatoia di compatibilità che silenziosamente ripristina la vulnerabilità è l'idea più trasferibile qui, e ricorre in Psych 4, PyTorch 2.6 e nel serializzatore dei cookie di Rails.

**Due negativi che valgono più della maggior parte dei positivi.**

I due pezzi più significativi di ricerca sulla deserializzazione Ruby degli anni recenti **non hanno ricevuto alcuna CVE.** L'[escape di Gem::SafeMarshal](https://nastystereo.com/security/ruby-safe-marshal-escape.html) di Luke Jahnke (2024-12-03) e la sua universal gadget chain per Ruby 3.4 (2024-11-24) sono stati entrambi corretti come normali point release di RubyGems. Una ricerca per keyword su NVD per `SafeMarshal` restituisce zero risultati. **Uno scanner, un corpus o un curriculum costruito solo su feed di CVE mancherà lo stato dell'arte effettivo.** Traccia invece le intestazioni del changelog `### Security:` di RubyGems e i resoconti dei ricercatori primari.

E: **nessuna CVE di deserializzazione Ruby o Rails appare nel catalogo CISA KEV**, verificato programmaticamente contro tutte le 1.653 voci. Le uniche voci Rails in KEV sono path traversal. La deserializzazione Ruby è una ricca area di ricerca **senza alcun evento di sfruttamento di massa confermato da CISA**. C'è uno sfruttamento documentato da botnet commodity di CVE-2013-0156 e una compromissione di ecosistema documentata. Questa è un'affermazione reale ma molto più piccola di quella che si fa di solito, e dirla chiaramente è il modo più veloce per mantenere un lettore competente.

## Anche tutti gli altri l'hanno preso

**Java, 2015.** La tecnica atterrò all'OWASP AppSec California il 2015-01-28, nel "Marshalling Pickles: How Deserializing Objects Will Ruin Your Day" di Chris Frohoff e Gabriel Lawrence, che copriva Python `pickle`, Ruby `Marshal`, serializzazione PHP e Java **insieme** e introdusse ysoserial. Ruby era nel talk originale; questo progetto non è una nota a piè di pagina della storia Java, è parte della stessa divulgazione.

L'era esplose dieci mesi dopo, e non a causa del talk. [Il post di Foxglove Security](https://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/) il 2015-11-06 lo weaponizzò contro prodotti enterprise nominati e svergognò pubblicamente i vendor:

> "Even though proof of concept code was released OVER 9 MONTHS AGO, none of the products mentioned in the title of this post have been patched, along with many more."

Il security alert di Oracle atterrò quattro giorni dopo. Questa è una cosa utile da capire su come la divulgazione effettivamente muove i vendor.

La risposta strutturale di Java fu **JEP 290**, "Filter Incoming Serialization Data," creata il 2016-04-22 e consegnata in Java 9. Il caveat critico: **un serialization filter non è abilitato o configurato di default.** Nove anni dopo ysoserial, il default di Java è ancora non filtrato.

**.NET** si è spinto più lontano. La guida di Microsoft è la dichiarazione di vendor più inequivocabile su questa classe ovunque: "**`BinaryFormatter` is insecure and can't be made secure.**" La loro analogia vale la pena rubarla: "assume that calling `BinaryFormatter.Deserialize` over a payload is the equivalent of interpreting that payload as a standalone executable and launching it." A partire da .NET 9 l'implementazione in-box solleva un'eccezione all'uso.

**PHP** ha lo spazio di gadget più denso di tutti loro, perché `__wakeup`, `__destruct` e `__toString` danno molti più metodi magici raggiungibili del singolo `readObject` di Java. All'incirca decine di chain per Java, e all'incirca **170 chain attraverso 44 framework** per phpggc. (Le 44 directory di framework sono state contate due volte e concordano. Le 170 provengono da un singolo conteggio, quindi trattalo come approssimativo; il confronto regge in ogni caso.) Il miglior caso in-the-wild è CVE-2015-8562 (Joomla), dove il payload arriva in un **header User-Agent**, che rende il punto "l'input non fidato è ovunque, non solo nel body della richiesta" meglio di qualsiasi diagramma. Sucuri ha documentato la curva: primo exploit 2015-12-12, e entro il 12-14 "basically every site and honeypot we have being attacked."

La singola migliore idea da rubare a PHP è il lavoro `phar://` di Sam Thomas (Black Hat USA 2018). I metadata di un archivio Phar sono deserializzati da **normali operazioni sui file**: `fopen`, `file_exists`, `file_get_contents`, `filesize`. Un'applicazione può subire deserializzazione **senza alcuna chiamata `unserialize()` da nessuna parte nel suo codice**. Questo è l'argomento più forte disponibile contro il modello mentale "fai grep della funzione pericolosa e hai trovato la superficie d'attacco". La lezione trasferibile per i lettori Ruby: **la domanda dell'audit non è "dove chiamiamo `Marshal.load`" ma "cosa raggiunge un deserializzatore".**

**Python** fornisce il miglior argomento che lo scanning a denylist perde. `picklescan`, lo scanner su cui si affida l'ecosistema ML, ha accumulato **oltre 26 CVE proprie** tra 2025-02 e 2026-06:

| CVE | CVSS | Il bypass |
|---|---|---|
| CVE-2025-1716 | 9.8 | `pip` non era nella lista degli unsafe-globals; `pip.main()` tira un pacchetto malevolo |
| CVE-2025-1889 | 9.8 | Estensioni di file non standard cadono fuori dallo scope dello scan |
| CVE-2025-1945 | 9.8 | Invertire bit negli header ZIP nasconde il pickle allo scanner mentre `torch.load()` lo carica comunque |
| CVE-2025-10156 | 9.8 | Un CRC deliberatamente errato ferma lo scanner |
| CVE-2025-71350 | 8.1 | `torch.utils.collect_env.run` non era bloccato |

Ventisei CVE, ognuna "ci siamo dimenticati di *questo* callable." E non è un singolo strumento difettoso: `fickling` di Trail of Bits ha il fallimento identico in CVE-2026-22608, concatenabile a RCE **mentre lo strumento riporta il file come sicuro**.

Nota la CWE sulla prima sia di `fickling` sia di picklescan: **CWE-184, "Incomplete List of Disallowed Inputs."** MITRE ha una classe di debolezza dedicata per "alla tua denylist manca qualcosa," ed entrambi i principali pickle scanner dell'ecosistema l'hanno ricevuta.

**Lo scanning a denylist di uno stream di serializzazione non perde sull'esecuzione. Perde sull'architettura.** Questo è il contrappeso onesto allo scanner di questo progetto stesso, ed è dichiarato esattamente in quei termini: uno strumento di scoperta di gadget ti dice quali chain esistono *oggi*. Non è un controllo.

L'unica cosa che ha funzionato, in tre ecosistemi, è stata **cambiare il default.** PyTorch ha ribaltato `weights_only` di `torch.load` a `True` in 2.6.0 (2025-01-29). NumPy ha ribaltato `allow_pickle` a `False` in 1.16.3, con le sue release notes che dicono "in response to CVE-2019-6446." Ruby ha reso `YAML.load` sicuro in 3.1. Tutti e tre hanno rotto i workflow di utenti reali, e quel costo è precisamente ciò che ha tenuto in piedi il default non sicuro per anni. Un decennio di avvertimenti non ha fatto nulla; cambiare il default sì.

## Dove sta questa classe, onestamente

**La citazione OWASP attuale è `A08:2025 – Software or Data Integrity Failures`.** Nota la dicitura esatta, perché la denominazione è andata alla deriva attraverso tre edizioni e sbagliarla è l'errore più probabile che chiunque faccia qui:

| Edizione | Numero | Nome esatto |
|---|---|---|
| 2017 | A08:2017 | **Insecure Deserialization** (una sua categoria dedicata) |
| 2021 | A08:2021 | Software **and** Data Integrity Failures |
| **2025** | **A08:2025** | Software **or** Data Integrity Failures |

L'insecure deserialization non ha più avuto una sua categoria Top 10 dal 2017. È una delle 14 CWE mappate dentro A08, e CWE-502 è esplicitamente tra esse.

**CWE-502** è denominata "Deserialization of Untrusted Data," è stata nella CWE Top 25 ogni anno dal 2019 al 2025, e **si classifica #15 nella lista 2025** con un punteggio di 5.23, su di un posto rispetto a #16 nel 2024. I suoi vicini sono stack e heap buffer overflow.

**I numeri KEV**, scaricati e analizzati localmente anziché chiesti a un motore di ricerca (versione del catalogo 2026.07.24):

| Misura | Valore |
|---|---|
| Totale voci KEV | 1.653 |
| Voci mappate a **CWE-502** | **69 (4,17%)** |
| Rank di CWE-502 tra tutte le CWE in KEV | **7º** |
| Voci CWE-502 con uso noto in campagne ransomware | **24 di 69 (34,8%)** |
| Baseline: tutte le voci KEV con uso noto da ransomware | 332 di 1.653 (**20,1%**) |

Due risultati da portarsi via da questo. **I bug di deserializzazione hanno circa 1,7 volte più probabilità di essere usati da crew ransomware rispetto alla vulnerabilità media confermata-sfruttata.** È l'affermazione di impatto onesta più forte disponibile per questa classe, viene da un catalogo governativo anziché da un vendor, e chiunque abbia `curl` può riprodurla. E **il tasso non sta calando**: il 2025 è stato il singolo anno con il numero più alto mai registrato, e Microsoft SharePoint da solo ha raccolto cinque voci KEV CWE-502 nell'arco di dodici mesi, una di esse ("ToolShell," CVE-2025-53770) con uso da ransomware confermato.

Se vedi la pagina CWE di MITRE riportare 11 voci KEV per CWE-502 anziché 69, entrambe sono giuste e misurano cose diverse. La metodologia della CWE Top 25 analizza i 39.080 record CVE pubblicati tra 2024-06-01 e 2025-06-01, quindi il suo conteggio ha come scope una finestra di pubblicazione di un anno. Le 69 sono il conteggio di tutti i tempi nel catalogo vivo. Citarle affiancate senza quella nota si legge come un errore.

**Non c'è una cifra in dollari aggregata credibile per questa classe di bug.** Circolano numeri di quella forma. Vengono dal marketing dei vendor e da estrapolazioni da sondaggi sul "costo medio di una violazione," e non sono riconducibili a dati di incidenti. Un repo che ricicla un numero di marketing in una citazione dall'aria accademica è peggio che non avere alcun numero.

## Correzioni: cose che leggerai che sono sbagliate

Ogni voce qui sotto è qualcosa che un write-up sicuro di sé asserisce plausibilmente, controllato contro una fonte primaria, e trovato sbagliato. Questa lista è la cosa più utile in questo capitolo, perché è una mappa di dove la riproposizione popolare di questa classe di vulnerabilità fallisce.

| L'affermazione comune | Cosa dice il record |
|---|---|
| **Equifax fu una violazione da deserializzazione.** | **Sbagliato.** CVE-2017-5638 è OGNL injection, NVD **CWE-755**, CISA KEV **CWE-20**. La CVE di deserializzazione di Struts è CVE-2017-9805, un bug diverso in un componente diverso, e non era il vettore di Equifax. |
| **CVE-2013-0156 è un record CWE-502.** | **Sbagliato.** NVD assegna **CWE-20**. Così fa CVE-2013-3567 (Puppet). Filtrare NVD per CWE-502 per enumerare "le CVE di deserializzazione" elimina silenziosamente la più famosa Ruby. |
| **CVE-2019-5420 è una CVE di deserializzazione.** | **Fuorviante.** NVD assegna **CWE-330 e CWE-77**. La causa principale è un `secret_key_base` prevedibile in modalità dev. Il fix fu la generazione della chiave. |
| **La categoria OWASP attuale è "A08:2021 Software and Data Integrity Failures."** | **Datata.** È **A08:2025 Software *or* Data Integrity Failures.** "Or", non "and". |
| **`YAML.load` è non sicuro in Ruby.** | **Dipendente dalla versione, e l'affermazione non qualificata ora è sbagliata.** Verificato per esecuzione: Psych 3.3.2 (Ruby 3.0.7) deserializza oggetti arbitrari; Psych 4.0.4 (Ruby 3.1.7) e successivi sollevano `Psych::DisallowedClass`. **`Marshal.load` ricostruisce oggetti arbitrari su ogni versione testata.** Non ha mai ottenuto un default sicuro. |
| **Apache Commons Collections ebbe una CVE.** | **Sbagliato, e nasconde la lezione.** Tutti e tre gli identificatori hanno come scope il vendor: CVE-2015-4852 (Oracle), CVE-2015-7501 (Red Hat), CVE-2015-6420 (Cisco). La libreria non ne ricevette mai una, e l'ASF argomentò pubblicamente che non avrebbe dovuto. |
| **L'attacco ransomware SFMTA / San Francisco Muni del 2016 fu Oracle WebLogic CVE-2015-4852.** | **Non supportato, e contraddetto da SFMTA.** La loro stessa dichiarazione: "The SFMTA network was not breached from the outside, nor did hackers gain entry through our firewalls." L'associazione a WebLogic descrive il *toolkit generale dell'attaccante attraverso molte vittime*, non un riscontro su SFMTA. CISA KEV marca CVE-2015-4852 `knownRansomwareCampaignUse: "Unknown"`. |
| **RDoc CVE-2024-27281 fu corretta in 6.3.4 / 6.4.1 / 6.5.1 / 6.6.3.** | **Sbagliato.** L'advisory Ruby dichiara che quelle contengono un **fix incorretto**. Le versioni corrette sono 6.3.4.1, 6.4.1.1, 6.5.1.1, 6.6.3.1. |
| **CVE-2026-39324 (rack-session) affligge Rails.** | **Sbagliato.** L'advisory a monte dice che Rails tipicamente non è affetto; usa un percorso di codice diverso. |
| **Le CVE 2024-2025 di ruby-saml sono bug di deserializzazione.** | **Sbagliato.** CVE-2024-45409 è CWE-347. CVE-2025-25291 / CVE-2025-25292 sono signature wrapping via un differenziale di parser. Serie, critiche, ampiamente sfruttate, e non CWE-502. |
| **L'escape di Gem::SafeMarshal di Luke Jahnke ha una CVE.** | **Nessuna CVE esiste.** Corretto come normale point release di RubyGems sotto un'intestazione di changelog `### Security:`. |
| **La violazione di Equifax costò $1.4 miliardi "per filing SEC."** | **Non verificata come cifra depositata.** Si riconduce a commenti di earnings-call riportati dalla stampa. Difendibile: **$113.3M** (10-K FY2017, testuale) e **$575M a $700M** (accordo FTC). |

Un avvertimento metodologico cade da quella tabella e vale la pena dichiararlo da solo: **filtrare un corpus di vulnerabilità per CWE-502 per "trovare le CVE di deserializzazione" è un metodo rotto, e elimina silenziosamente gran parte del canone.** CVE-2013-0156 è CWE-20. CVE-2015-8562 (Joomla, sfruttata di massa) è CWE-20. CVE-2016-4010 (Magento) è CWE-74. CVE-2025-27407 (graphql-ruby) è CWE-94. I record più vecchi precedono il mapping coerente a CWE-502, e le CNA sono in disaccordo con NVD di routine. I numeri KEV in questo capitolo usano il campo `cwes` del catalogo stesso e quindi ereditano la stessa limitazione: sono un pavimento, non un censimento.

Infine, un pericolo permanente. Questo argomento è pesantemente inquinato da output di content farm. Un articolo che afferma una RCE da object-injection di `Oj.load` in Ruby usa un **ID CVE segnaposto**, non ha alcun GHSA, nessun range di versioni, e nomina un prodotto che non esiste. Se un'affermazione non ha una fonte primaria, non appartiene a un documento didattico. Quella regola ha fatto più lavoro su questo argomento che su qualsiasi altro in questo repo.

## Dove andare dopo

[02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) trasforma queste idee in struttura: i due reader che condividono un vocabolario, i tre stati di decisione e perché non c'è un predicato `accepted?`, la tassonomia gate-più-formato-più-arity dello scanner, e perché al parser e al detector è deliberatamente permesso di essere in disaccordo sullo stesso stream.
