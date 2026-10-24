<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.md -->

# marshalsea: Sfide

Il modo migliore per capire una gadget chain è costruirne una. Il modo migliore per capire un detector è far passare un payload oltre di esso. Questo capitolo è un insieme graduato di progetti, ognuno dei quali nomina i file che toccheresti e il test che proverebbe che hai finito. Sono ordinati all'incirca per impegno.

Nessuno di questi è un accenno a un lavoro incompleto. Il laboratorio è completo: 268 test su sette suite, sei fasi del gate, 79 asserzioni, tutte verdi. Queste sono le porte che lascia deliberatamente aperte, e due di esse sono rinvii documentati con tradeoff reali che vengono nominati onestamente qui sotto anziché mascherati da esercizi.

Prima di iniziare:

```bash
just check       # seven suites plus the standalone controls
just gate        # everything, about ten minutes, runs real containers
just corpus      # every adversarial payload and what the detector decided
just scan        # the gadget scanner over whatever this process has loaded
just lint        # rubocop across 37 files
```

**Ogni sfida qui sotto dovrebbe finire in due modi: una suite verde, e un test che sarebbe fallito prima della tua modifica.** Quella seconda metà è la parte che conta. Questo progetto ha rilasciato due bypass del detector sotto 110 test passati, quindi una suite verde da sola non è prova di nulla. Se aggiungi una regola che afferma che il codice fa X, vai a rompere X e guarda un test diventare rosso prima di crederci.

## Riscaldamenti

**Aggiungi un tag di sink che il parser non conosce.** I tre tag di sink sono `u`, `U` e `d`, mappati a `_load`, `marshal_load` e `_load_data` in `Constants::SINK_METHODS`. Scegli qualsiasi altro tag dal formato, decidi cosa invocherebbe, e aggiungilo. Il punto dell'esercizio è scoprire quanto poco devi toccare: la tabella è una costante congelata, `Node#sink?` e `Node#sink_method` sono lookup in essa, e la scala del detector non nomina mai un tag direttamente. Dimostralo con una voce del corpus che deve essere rifiutata, e poi vai nella direzione opposta e conferma che il tuo nuovo tag *non* scatta su uno stream che meramente menziona la classe in posizione di valore.

**Rendi configurabile il budget delle stringhe di motivazione.** `REASON_MAX_NAME_BYTES` è 96 e `REASON_MAX_NAMES` è 8, entrambi hard-coded. Spostali su `Limits` così che un'applicazione che fa logging su un sistema con un budget di riga più piccolo possa ridurli. Il test che conta è quello avversariale: un nome di classe di 10.000 byte contenente newline deve comunque produrre una motivazione a riga singola, troncata, con escaping tramite `inspect` a qualunque tetto tu imposti. Una stringa di motivazione è output controllato dall'attaccante, e questa è una superficie di log injection prima di essere una preferenza di formattazione.

**Insegna allo scanner un altro metodo link.** `to_s` e `coerce` sono i due metodi attualmente classificati `GATE_LINK`: raggiungibili da un gadget a metà chain, mai invocati da un deserializzatore. Trovane un altro nella letteratura sulle chain pubblicate, aggiungilo a `ENTRY_POINTS` con `gate: GATE_LINK` e `formats: []`, e conferma che il conteggio dei link si muova mentre il conteggio dei raggiungibili no. Se i raggiungibili si muovono, l'hai classificato male, e questa è la lezione.

**Aggiungi un filtro `--namespace` a una scan che ti interessa.** `Scanner.new(namespace: "Gem")` esiste già e il justfile lo passa già (`just scan Gem`). Usalo per fare la scan solo del namespace della tua stessa applicazione, poi confronta con una scan completa. Il risultato interessante è di solito quante poche delle tue stesse classi siano candidate e quante di quelle delle tue dipendenze lo siano.

## Intermedio

**Scrivi una terza chain contro una CVE diversa.** Il registro è basato su directory: metti un file in `lib/marshalsea/chains/`, fai subclass di `Base`, e `inherited` lo registra. Ti servono `metadata` (name, vector, cve, gem, vincoli affected, kind), un `generate` che restituisce un oggetto o documento vivo, e un `serialize` se il `Marshal.dump` di default è sbagliato per il tuo entry point. CVE-2024-27281 di RDoc è un target ragionevole perché il meccanismo è documentato e i range affetti sono precisi; la trappola è che quattro delle versioni "corrette" contengono un fix incorretto, quindi i tuoi vincoli `affected` devono codificare `6.3.4.1` e non `6.3.4`. Dimostralo con un test di confine di versione in entrambe le direzioni, nel modo in cui `erb-def-module` asserisce `6.0.1` affetta e `6.0.1.1` no.

**Fai seguire allo scanner i link in chain effettive.** Al momento lo scanner riporta entry point e link come due liste separate e non li connette mai. Questo è onesto ma si ferma un passo prima della domanda interessante: *dato questo entry point, cosa può raggiungere?* Costruisci una seconda passata che, per ogni entry point raggiungibile, percorra il corpo del metodo con Prism cercando chiamate a metodi su receiver tenuti in ivar, ed emetta chain candidate a due passi. Il deliverable onesto non è un trova-chain, è una lista *classificata* più una dichiarazione scritta del suo tasso di falsi positivi, perché un walk statico non può sapere cosa conterranno quelle ivar al momento del load. Misura quel tasso rispetto all'unica chain in questo repo di cui si sa che è reale.

**Chiudi il buco `#hash` della guardia senza il costo della strict mode.** `LoadGuard` deliberatamente non osserva `#hash` e `#eql?` di default, perché sono tra i metodi più caldi in Ruby. `strict: true` li osserva e lo paga. Trova una terza opzione: osserva `#hash` ma filtra il `TracePoint` ai receiver la cui classe è fuori da un set permesso prima di fare qualsiasi altro lavoro, oppure usa `TracePoint#enable(target:)` per dare scope alla traccia anziché filtrare dentro l'handler. Poi **misuralo**, sulle dimensioni di payload che contano, e pubblica il numero con la dimensione del payload allegata. Se la tua versione non è significativamente più economica di `strict: true`, quello è un risultato reale e dovrebbe essere scritto come tale.

**Costruisci la busta di sessione autenticata.** Il target deserializza un cookie base64 senza alcuna firma, il che è realistico per un target didattico e *non* è ciò che un'applicazione reale dovrebbe fare. Aggiungi una busta HMAC o AEAD davanti a `Marshal.load`, verificala prima che i byte raggiungano qualsiasi deserializzatore, e poi scrivi il test che fa arrivare la lezione: **una firma valida non rende il payload sicuro.** Firma un vero payload gadget con la chiave corretta e conferma che venga comunque eseguito. Questa è CVE-2019-5420 e CVE-2018-15133 riprodotte nel tuo stesso codice, ed è la dimostrazione più pulita possibile che la firma affronta la manomissione, non l'origine non fidata. Questa è una decisione di scope deliberata nel laboratorio attuale, non una svista; il motivo per cui è stata lasciata fuori è che la lezione delle due allowlist è più chiara senza un layer crypto di mezzo.

**Estendi l'inspector Psych a Oj.** La gem `oj` di Ruby supporta l'istanziazione di oggetti da JSON nella sua modalità `:object`, che è la sua **default**. Un parser JSON capace di object-injection out of the box è il fatto più sorprendente in tutta quest'area e non ha rappresentazione in questo laboratorio. Scrivi un terzo reader che legge il JSON in modalità Oj senza caricarlo, riporta quali classi istanzierebbe, e produce un `Decision` nello stesso vocabolario degli altri due. Il vincolo di design è la parte interessante: i due reader esistenti condividono deliberatamente una classe `Decision`, quindi un terzo che avesse bisogno di una forma diversa ti starebbe dicendo qualcosa sull'astrazione.

## Avanzato

**Costruisci la fase di eager-load che il contratto di design richiede.** Lo scanner può vedere solo le classi che sono state richieste, quindi una scan di un processo nudo vede 691 moduli e una scan dentro un'app Rails avviata ne vede diverse migliaia. La correzione ovvia è una fase che richiede ogni file in ogni gem installata prima di fare la scan. È anche **esecuzione di codice arbitrario by design**: richiedere una gem esegue il suo codice top-level, quindi una fase di eager-load eseguita contro gem non fidate è un footgun della supply-chain puntato contro l'operatore. Costruiscila, e costruisci l'isolamento che la rende difendibile: un container separato senza rete, un mount in sola lettura, un timeout, e un flag di opt-in esplicito il cui testo d'aiuto dice cosa fa. Il deliverable che conta è il threat model scritto, non il loop. Questo è il singolo elemento aperto più grande nel progetto ed è stato rinviato esattamente per questo motivo.

**Modella una seconda versione dell'interprete.** Il parser modella Ruby 3.4 e successive, e il floor della gem è `>= 3.4` perché `Marshal.load` non validava il byte di segno del bignum fino alla 3.4. Questo significa che il parser e l'interprete *sono in disaccordo* sulla 3.3, e `just package` prova quel confine in entrambe le direzioni a ogni esecuzione. Rendi il parser consapevole della versione: accetta una versione target, rilassa il controllo del segno del bignum sotto la 3.4, e abbassa il floor. Poi scrivi il test differenziale che ti tiene onesto, facendo passare lo stesso stream attraverso il vero `Marshal.load` sia su un container 3.3 sia 3.4 e asserendo che il parser concordi con **ciascuno** di essi. Scoprirai rapidamente che "per quale Ruby è questo stream" non è una domanda a cui lo stream può rispondere, che è la vera lezione.

**Attacca la tolleranza forense del parser.** Il parser deliberatamente continua dove CRuby si ferma, e il detector rifiuta sulle anomalie che registra. Quella separazione è il design, ma è anche una superficie d'attacco: qualsiasi punto in cui il modello del formato del parser diverge da quello di CRuby è un potenziale differenziale. Vanne a trovare uno. Costruisci un fuzzer che genera stream, dà in pasto ciascuno sia al parser sia a un vero `Marshal.load` in sandbox, e segnala ogni caso in cui il parser riporta un insieme di classi o di sink che il comportamento effettivo dell'interprete contraddice. Questo è il lavoro di sicurezza di valore più alto disponibile nel repo, e un singolo differenziale confermato sarebbe un riscontro reale.

**Scrivi il detector che batte una denylist.** [01-CONCEPTS.md](./01-CONCEPTS.it.md) sostiene che lo scanning a denylist di uno stream di serializzazione perde sull'architettura, con oltre 26 CVE di picklescan e CWE-184 come prova. Prendilo sul serio e progetta l'alternativa. Che aspetto avrebbe una policy *strutturale*, una che decide sulla forma del grafo anziché su una lista di nomi? Uno stream contenente solo primitive, array, hash con chiavi primitive e stringhe è sicuro per costruzione, indipendentemente da quali classi esistano. Implementalo come quarta policy, misura quanti payload del mondo reale rifiuta (quel numero sarà alto, e quello è il costo onesto), e scrivi dove si trova effettivamente il confine tra "sicuro per costruzione" e "utile".

**Rendi il target un vero corpus di app vulnerabili.** Il target ha quattro endpoint su due deserializzatori. Aggiungi il sink di secondo ordine da CVE-2022-32224: una route che scrive dati influenzati dall'attaccante in uno store e una route *diversa* che li rilegge e li deserializza, così che il payload non appaia mai nella richiesta che innesca l'esecuzione. Questa è la forma che vanifica un confine di fiducia tracciato al bordo HTTP, ed è molto più difficile da ragionare rispetto al caso diretto. La fase del gate che lo prova deve estendersi su due richieste, che è di per sé una cosa utile da aver costruito.

## Un capstone: far passare un payload oltre il detector

Se vuoi un progetto che leghi insieme l'intero laboratorio, fai questo.

Il `LIMITATION_NOTICE` del detector dice ad alta voce che una decisione di accept significa solo "questi byte hanno corrisposto a questa policy." Il tuo compito è renderlo concreto: **trova uno stream che il detector strict-allowlist accetta e che fa comunque qualcosa che un operatore non sancirebbe.**

Hai tre angolazioni e sono tutte legittime:

1. **Trova una classe degna di essere messa in allowlist che è pericolosa comunque.** La notice lo concede direttamente: "Class allowlisting compares serialized names. It does not prove that the corresponding Ruby code is harmless." Un'applicazione che mette `ERB` in allowlist accetta la chain della CVE pubblicata, perché quel payload porta zero tag di sink. Trova una seconda classe con la stessa proprietà.
2. **Trova un dispatch che il parser non modella.** La scala copre i tag di sink, `#hash` e `#eql?` in posizione di chiave, e `#<=>` negli estremi di `Range`. Tre di quelle cinque regole sono state aggiunte perché un bypass è arrivato prima. Non c'è motivo di credere che la lista sia completa. Vai a leggere `marshal.c` e trova la sesta.
3. **Trova un differenziale parser-contro-interprete.** Se il parser e CRuby sono in disaccordo su cosa contiene uno stream, il detector sta decidendo su un grafo che non è quello che verrà caricato.

Le regole dell'esercizio, che sono le stesse regole a cui il progetto si attiene:

- Il tuo bypass deve essere **riproducibile da una stringa di byte**, non da un grafo di oggetti costruito a mano. Se non riesci a scriverlo come byte, non è un payload.
- Deve essere accettato sotto `strict_allowlist`, non solo sotto `deny_sinks_only`. Battere la policy più debole non prova nulla.
- Fornisci il fix **e il mutante**: aggiungi la regola che lo intercetta, poi sventra la regola e guarda la voce del corpus diventare rossa. Una regola che non riesci a uccidere è una regola che non hai testato.
- Aggiungi un **controllo negativo** accanto ad esso: un payload della stessa forma che deve comunque essere accettato. Altrimenti non riesci a distinguere la tua nuova regola da un detector che rifiuta tutto.

Quando l'avrai fatto una volta, capirai questa classe di bug meglio di quanto insegni qualsiasi write-up, perché sarai stato su entrambi i lati dello stesso file.

## Dove andare dopo

Rileggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) con il codice di [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) fresco in testa. L'argomento delle due allowlist si legge diversamente una volta che hai visto `in_hash_key_position` innestare un payload byte per byte, e lo smontaggio di Equifax si legge diversamente una volta che sai quanto lavoro ci voglia per essere sicuri di una sola affermazione.

Poi esegui `just corpus`, scegli una riga qualsiasi in quella tabella, e vai a trovare il test che ce l'ha messa.
