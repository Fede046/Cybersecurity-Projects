<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# marshalsea: Panoramica

## Cos'è

Un laboratorio di sicurezza sulla deserializzazione di oggetti Ruby, distribuito come gem più un container. Gli passi byte `Marshal` o un documento YAML e ti dice quali classi contengono e quali metodi quei byte attiverebbero, **senza mai far rivivere nulla**. Va a caccia, nel grafo delle classi caricate, delle classi che costituiscono gadget utilizzabili. Costruisce un payload funzionante per una vera CVE del 2026. E poi mette in piedi un'app Sinatra deliberatamente vulnerabile, così puoi vedere quel payload andare a segno via HTTP e vedere la difesa fermarlo.

L'intero progetto è organizzato attorno a una domanda che quasi ogni write-up salta: *perché la correzione ovvia non funziona?* Tutti dicono "non deserializzare input non fidato". Meno persone sanno spiegarti perché passare a `Marshal.load` una proc di allowlist fallisce, perché la stessa idea di allowlist in Psych invece funziona, e perché la differenza sta in quattro righe di distanza in `marshal.c`. Quella domanda è la spina dorsale di questo laboratorio, e ogni difesa che fornisce è accompagnata da una dichiarazione scritta di ciò che non è in grado di fare.

Niente qui è uno stub. 268 test distribuiti su sette suite, più un gate in sei fasi che pilota container reali con 79 asserzioni che devono passare tutte.

## Perché è importante

Serializzare un oggetto significa congelarlo in byte. Deserializzarlo significa scongelarlo. La trappola è che lo scongelamento non è una copia passiva: ricostruire un oggetto chiama metodi su di esso, e se è un attaccante a scegliere i byte, è l'attaccante a scegliere quali metodi vengono eseguiti. Concatena abbastanza metodi della libreria standard tra loro non correlati e dall'altra parte esce l'esecuzione di codice. In nessun punto di quei byte c'è un'istruzione che dica "esegui un comando". È questo che la rende una **gadget chain**, ed è per questo che il nome di questa classe di bug viene così spesso insegnato male.

Non è una stranezza di Ruby. Java, PHP, Python e .NET hanno tutti rilasciato un serializzatore comodo e hanno tutti ereditato lo stesso bug. I numeri onesti, tutti riproducibili:

- **69 delle 1.653 voci del CISA KEV sono CWE-502** (versione del catalogo 2026.07.24), il che la rende la **settima debolezza più comune** nel catalogo delle vulnerabilità confermate come sfruttate in the wild.
- **Il 34,8% di queste ha un uso noto da parte di ransomware, contro una baseline del 20,1%** per il catalogo nel suo complesso. I bug di deserializzazione hanno circa 1,7 volte più probabilità della media di finire nel toolkit di una crew ransomware.
- **Il tasso non sta calando.** Il 2025 è stato il singolo anno con il numero più alto mai registrato di aggiunte CWE-502 al KEV, e Microsoft SharePoint da solo ne ha raccolte cinque nell'arco di dodici mesi.

Quello che *non* troverai in questo repo è una cifra in dollari. Non esiste un numero credibile e citabile per il costo finanziario totale di questa classe di bug, e [01-CONCEPTS.md](./01-CONCEPTS.it.md) lo dice apertamente invece di riciclare una statistica di un vendor in una citazione dall'aria accademica.

**Scenari reali in cui si applica:**
- **Revisionare un layer di sessione o di cache.** I cookie di Rails erano serializzati con Marshal prima della 4.1, e i cache store di Active Support deserializzano ancora ciò che rileggono. La firma è l'unico controllo, e CVE-2019-5420 è il caso in cui la firma funzionava e la *chiave* era indovinabile.
- **Fare audit di un sink di secondo ordine.** CVE-2022-32224 è la forma moderna: i dati non fidati arrivano dal tuo stesso database, non via HTTP, e vanificano qualsiasi threat model che tracci il confine di fiducia al bordo della richiesta.
- **Costruire un detector.** Se stai scrivendo uno scanner per payload serializzati, la cosa più utile in questo repo è la prova che lo scanning a denylist di un formato di serializzazione perde per ragioni di architettura, non di impegno. `picklescan`, lo scanner su cui si affida l'ecosistema ML di Python, ha accumulato **oltre 26 CVE proprie**, e `fickling` di Trail of Bits ha lo stesso difetto. A entrambi è stata assegnata la CWE-184, "Incomplete List of Disallowed Inputs".

## Cosa imparerai

**Concetti di sicurezza:**
- **Dispatch gated contro dispatch ungated.** L'asse su cui è costruito l'intero laboratorio, e uno che non è messo per iscritto nella letteratura Ruby pubblicata. `marshal_load` e `_load` sono gated: `Marshal` chiama prima `respond_to?` e solleva `TypeError` in caso di risposta negativa. `hash`, `eql?`, `<=>`, `[]=` e l'`init_with` di Psych vengono invocati alla cieca. Una classe proxy con i metodi cancellati è quindi un entry point *valido* per YAML e uno *non valido* per Marshal. Stessa classe, esito opposto.
- **Perché una allowlist è un buttafuori e l'altra è un'autopsia.** La proc di `Marshal.load` viene eseguita in `r_post_proc`, dopo che `load_funcall` ha già attivato il tuo gadget. Psych controlla il tag *prima* della rivivificazione. Intento identico, esito opposto, deciso interamente da dove si trova il controllo.
- **Entry point contro link.** `to_s` è un vero link nella universal chain pubblicata, ma `Marshal` non lo chiama mai. Confondere "un metodo che un gadget chiama" con "un metodo che il deserializzatore invoca" è una fabbrica di falsi positivi, e lo scanner li modella come due tipi di nodo diversi.
- **Quanto ti costa una guardia parziale.** CVE-2026-41316 è l'esempio svolto: Ruby 2.7.0 ha aggiunto una guardia per impedire l'esecuzione di codice via `Marshal.load` sugli oggetti ERB, e copriva due dei cinque entry point. Sei anni di una difesa corretta con tre porte lasciate aperte. NVD la cataloga come CWE-502 **e CWE-693, Protection Mechanism Failure**, e il doppio mapping è il nocciolo della storia.
- **Come distinguere questa classe di bug da quelle con cui viene confusa.** Il capitolo si apre sfatando l'esempio più citato di essa.

**Competenze tecniche:**
- **Leggere un formato binario senza eseguirlo.** Il formato wire di Marshal tag per tag: version byte, packing dei fixnum, tabelle dei simboli, back-reference tra oggetti, variabili d'istanza, i tre tag di sink.
- **Reflection su un grafo di classi vivo.** `ObjectSpace.each_object(Module)`, `instance_method`, `source_location` e Prism per decidere se il corpo di un metodo tocca davvero lo stato dell'oggetto, con ogni errore ingoiato contato e nominato.
- **Costruire un payload senza farlo detonare.** Costruire `{proxy => 1}` in Ruby chiama `#hash` sulla chiave e attiva la chain *nel tuo stesso processo*. Il builder innesta uno stream in posizione di chiave ricavandolo da un dump autonomo.
- **Porre il veto in un punto che il linguaggio non ti offre.** Un `TracePoint` su `:call` scatta prima che il corpo di un metodo venga eseguito, che è esattamente il punto di veto che la proc di allowlist ti nega.

**Strumenti e tecniche:**
- **`just`** come command runner, con ogni fase eseguita in un container Docker pinnato e `--network none` ovunque tranne che sul target.
- **Minitest** per sette suite, e una disciplina **differenziale**: eseguire il vero `Marshal.load` e il vero `Psych`, osservare cosa è stato effettivamente invocato, asserire che il modello concordi, con guardie di liveness in entrambe le direzioni affinché un oracolo morto non possa passare in silenzio.
- **Prism** per l'analisi del sorgente, e **Sinatra su Rack 3** per il target vulnerabile.

## Prerequisiti

Non ti serve esperienza pregressa di deserializzazione o di sicurezza Ruby. Questo è un progetto di livello beginner nel senso che parte dai primi principi, non nel senso che il materiale sia superficiale.

**Conoscenze richieste:**
- **Basi di Ruby.** Classi, moduli, variabili d'istanza, blocchi. Se sai leggere `def foo(x)` e sai cosa significa `@bar`, sai leggere questo codice.
- **Byte.** Cos'è un byte, che `"\x04\b"` è composto da due byte e non da sei caratteri, e più o meno che aspetto ha un hex dump.
- **Cosa significa "la libreria standard è caricata nel tuo processo".** L'intera premessa dello scanner è che una classe che nessuno ha ancora richiesto non può essere un gadget.

**Strumenti di cui avrai bisogno:**
- **Docker**, e sinceramente è tutto. Ogni recipe nel justfile viene eseguita in un container contro una Ruby pinnata. Non hai mai bisogno di una Ruby sul tuo host.
- **`just`.** Installalo con `curl -sSf https://just.systems/install.sh | bash -s -- --to ~/.local/bin`.
- **La gem, se vuoi solo il reader.** `gem install marshalsea` richiede Ruby 3.4 o più recente e nessun container.

**Utili ma non richiesti:**
- Una scorsa a [CWE-502](https://cwe.mitre.org/data/definitions/502.html), così il vocabolario nel capitolo sui concetti ti arriva più velocemente.
- Familiarità con la versione di questo bug in *qualsiasi altro* linguaggio (`readObject` di Java, `unserialize` di PHP, `pickle` di Python). Il capitolo sui concetti mappa deliberatamente tutti e quattro l'uno sull'altro.

## Avvio rapido

Installa il reader:

```bash
gem install marshalsea
```

Osserva un payload senza eseguirlo:

```ruby
require "marshalsea"

payload = Marshal.dump(Gem::Requirement.new(">= 0"))
result  = Marshalsea::Marshal::Parser.new(payload).parse

result.class_names
# => ["Gem::Requirement", "Gem::Version"]

result.sinks.map { |s| "#{s.class_name}##{s.sink_method}" }
# => ["Gem::Requirement#marshal_load", "Gem::Version#marshal_load"]
```

Prendi una decisione invece di fare un'osservazione:

```ruby
detector = Marshalsea::Marshal::BoundaryDetector.new(allowed_class_names: %w[Hash String])
decision = detector.inspect_stream(untrusted_bytes)

decision.blocked?   # => true
decision.reason
# => "stream reaches \"Gem::Requirement\"#marshal_load during load, before any allowlist can run"

Marshal.load(decision.snapshot) if decision.proceed?
```

Nulla di quanto sopra istanzia una classe, chiama un costruttore o invoca `Marshal.load`. Leggi `Marshalsea::Marshal::BoundaryDetector::LIMITATION_NOTICE` prima di fare affidamento su `proceed?`; è una costante nella libreria proprio perché non possa essere saltata.

Poi esegui il laboratorio stesso da un checkout:

```bash
just scan        # hunt the loaded class graph for usable gadget entry points
just corpus      # every adversarial payload, and what the detector decides about each
just target      # stand up the vulnerable app and attack it over HTTP
just gate        # everything: suites, version matrix, exploit, detector, target, packaging
```

`just scan` su una `ruby:4.0-slim` di default stampa un header e poi ogni entry point che ha giudicato raggiungibile:

```
modules=691 candidates=193 gated=11 reachable=43 suppressed=3 candidates_lost=false
analysed=43 unanalysable=142 unreadable=8

  gated      Date._load
  ungated    Gem::Requirement#hash                /usr/local/lib/ruby/4.0.0/rubygems/requirement.rb:195
  ungated    Gem::Specification#method_missing    /usr/local/lib/ruby/4.0.0/rubygems/specification.rb:2055
  ...
  gated      Time._load

suppressed errors (this scan under-reports):
  source_parse     3

142 candidates have no Ruby source and were never analysed; the reachability filter does not cover them
```

Leggi prima gli ultimi due blocchi. Lo scanner riporta ciò che *non è riuscito* a vedere con lo stesso rilievo di ciò che ha trovato, e questo è deliberato: uno strumento di scoperta di gadget che silenziosamente sotto-riporta è peggio che nessuno strumento.

> [!TIP]
> Quei numeri dipendono dall'image e dal carico, non sono fatti su Ruby. `ObjectSpace` non può riportare una classe che nessuno ha ancora richiesto, quindi richiedere più codice produce più candidati. Riesegui `just scan` nel tuo ambiente invece di fidarti delle cifre stampate qui.

## Struttura del progetto

```
deserialization-gadget-lab/
├── lib/marshalsea/
│   ├── marshal/
│   │   ├── parser.rb            # the Marshal format reader that never calls Marshal.load
│   │   ├── node.rb              # the parse graph, sealed and frozen before it is returned
│   │   ├── boundary_detector.rb # policy, three decision states, limitation notice
│   │   ├── load_guard.rb        # TracePoint veto that fires before the hook body
│   │   ├── limits.rb            # fourteen resource ceilings, all on by default
│   │   ├── float_body.rb        # float decoding that labels what it cannot decode
│   │   └── constants.rb errors.rb
│   ├── psych/inspector.rb       # YAML AST reader, revives nothing
│   ├── chains/                  # the directory is the chain identity, no registry to rot
│   │   ├── base.rb erb_def_method.rb erb_def_module.rb psych_init_with.rb
│   ├── scanner.rb               # reflection over the loaded class graph
│   └── chains.rb version.rb
├── target/                      # the deliberately vulnerable Sinatra app + containers
├── scripts/                     # the six gate stages and the gem auditor
├── test/                        # seven suites, the adversarial corpus, standalone controls
├── learn/                       # this teaching track
└── justfile
```

La singola giuntura più importante da capire per prima è la separazione tra `parser.rb` e `boundary_detector.rb`. Il parser è deliberatamente **forense**: continua a fare il parsing di uno stream che CRuby stesso rifiuterebbe, così un sink nascosto in uno slot dove dovrebbe stare un simbolo resta visibile nel report invece di svanire dietro un errore di parsing. Il detector è la metà **rigorosa**, e rifiuta sull'anomalia che il parser ha registrato. Quella separazione è il motivo per cui uno stream ostile può essere al tempo stesso pienamente descritto e fermamente rifiutato, ed è la decisione di design a cui si aggancia il resto della libreria.

## Prossimi passi

1. **Comprendi le idee.** Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md). Si apre sfatando l'esempio più citato di questa classe di bug, poi costruisce l'asse gated-contro-ungated, l'argomento delle due allowlist e il registro verificato degli incidenti dietro a tutto questo.
2. **Osserva il design.** Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per i due reader, i tre stati di decisione, la tabella di tassonomia dello scanner e perché il parser e il detector sono deliberatamente in disaccordo.
3. **Percorri il codice.** Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per tracciare i byte in un grafo di nodi, un grafo di nodi in una decisione e un oggetto vivo in un payload, con la proxy chain di ActiveSupport come pezzo forte.
4. **Estendilo.** Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per progetti che vanno dalla scrittura di una nuova chain alla chiusura del bypass a esecuzione differita della guardia.

## Problemi comuni

**`just scan` riporta molti meno candidati di quanti te ne aspettassi**
```
modules=691 candidates=193 gated=11 reachable=43
```
Questo è corretto, non è un bug. `ObjectSpace.each_object(Module)` vede solo ciò che il processo corrente ha caricato, e un processo `ruby -Ilib` nudo e crudo ha caricato pochissimo. Richiedi `active_support`, oppure fai lo scan dentro il boot della tua applicazione, e i numeri salgono. Uno scanner di gadget ti dice quali chain esistono in *questo* processo, che è un'affermazione sul tuo grafo di dipendenze, non su Ruby.

**Il detector ha accettato uno stream e qualcosa è comunque andato storto**
```
decision.proceed?  # => true
```
Leggi `Marshalsea::Marshal::BoundaryDetector::LIMITATION_NOTICE`. `proceed?` significa *questi byte corrispondono alla policy che hai configurato*. Non significa che il payload sia sicuro. La chain pubblicata di CVE-2026-41316 produce **zero tag di sink**, quindi una policy basata solo sui sink non la intercetta mai; solo l'allowlisting delle classi lo fa, e un'applicazione che mette `ERB` in allowlist la accetterà comunque.

**La guardia a runtime ha reso le cose più lente di quanto la documentazione lasciasse intendere**
```
LoadGuard, 45-byte session cookie: 40x
```
Anche questo è corretto. Il costo della guardia non è un moltiplicatore, è un **~46 microsecondi per load** quasi costante speso per abilitare il `TracePoint`. Che è 1.0x su un documento da 488 KB e 40x su un session cookie, e un session cookie è esattamente ciò che questo laboratorio deserializza. Il numero è pubblicato con la dimensione del payload allegata perché pubblicarlo senza si leggerebbe come un'approvazione che non si è guadagnato.

**La gem si rifiuta di installarsi su Ruby 3.3**
```
marshalsea requires Ruby version >= 3.4
```
Misurato, non scelto. `Marshal.load` non validava il byte di segno del bignum fino alla 3.4. Il parser accetta solo `+` e `-`, quindi modella la 3.4 e successive; eseguilo sulla 3.3 ed entra in disaccordo con l'interprete che esiste per modellare. `just package` riprova quel confine in entrambe le direzioni a ogni esecuzione.

## Progetti correlati

Se hai trovato questo interessante, dai un'occhiata a:
- **binary-analysis-tool**: lo stesso muscolo del "leggi un formato di file con precisione e non eseguirlo mai", applicato agli eseguibili invece che ai grafi di oggetti.
- **api-security-scanner**: l'altra metà della questione del sink, trovare dove i byte non fidati entrano in un'applicazione in primo luogo.
- **lisdex** (zero-day-vulnerability-scanner): cosa succede quando il parser di cui stai facendo audit è scritto in C e la modalità di fallimento è la corruzione di memoria invece dell'object injection.
