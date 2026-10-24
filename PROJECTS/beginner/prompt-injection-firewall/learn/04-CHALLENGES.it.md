<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.it.md -->

# Sfide

Ogni sfida qui ha lo stesso criterio di completamento del progetto stesso: **il controllo deve diventare rosso quando lo rimuovi.** Scrivi il test, guardalo fallire, implementa, quindi elimina la tua implementazione e verifica che il tuo test, e nello specifico il tuo test, diventi rosso. Se rimane verde, hai costruito solo decorazione.

Due regole che si applicano a tutte le sfide:

- Una nuova regola a punteggio necessita di casi benigni nel corpus. Se non riesce a sopravvivere ad essi, non viene rilasciata.
- Un nuovo layer necessita di una propria classe nel benchmark, cosicché l'ablazione possa dimostrare che è portante anziché coperto da alibi.

## Facile

### 1. Aggiungere un marcatore di chat-template

**Cosa costruire:** i modelli attuali utilizzano control token che questo progetto non elenca. Aggiungi marcatori per una famiglia di modelli non coperta, ad esempio `<start_of_turn>` di Gemma o `<|START_OF_TURN_TOKEN|>` di Command-R.

**Perché:** la falsificazione dei delimitatori (delimiter forgery) è la regola di ingress con la precisione più elevata in assoluto, poiché i ticket di supporto reali non contengono sequenze di controllo del modello. Estenderla costa poco e il rischio di falsi positivi è prossimo allo zero.

**Cosa imparerai:** in che modo una regola basata sulla precisione differisce da una basata sulla recall, e perché questa ha una gravità `HIGH` mentre la regola imperativa è `MEDIUM`.

**Suggerimenti:** `config.CHAT_TEMPLATE_MARKERS`. Aggiungi casi a `bench/corpus/attacks/delimiter_forgery.yaml`. Verifica che il marcatore non possa comparire nella prosa ordinaria prima di aggiungerlo.

**Verifica che funzioni:** `just bench` mostra ancora il 100% su `delimiter_forgery` con un denominatore più ampio, e la classe hard-benign non ha subito variazioni.

### 2. Segnalare quale reading ha fatto match

**Cosa costruire:** quando ingress scatta su una lettura derivata (reading) anziché sul testo grezzo, il finding non lo specifica. Aggiungi questa informazione all'evidence.

**Perché:** "corrispondenza dopo decodifica base64" è un segnale materialmente diverso da "corrispondenza in testo in chiaro". Un operatore che esegue il triage degli alert vuole saperlo.

**Cosa imparerai:** perché l'`evidence` viene sottoposta a digest nell'audit log, e come aggiungere segnale senza inserire una stringa controllata dall'attaccante in un record.

**Suggerimenti:** `IngressLayer._first_imperative` itera sui reading. Restituisci quale ha fatto hit. Mantieni il campo evidence privo di testo grezzo dell'attaccante, oppure assicurati che il digest continui a coprirlo.

**Verifica che funzioni:** un payload base64 incorporato produce un'evidence che indica la decodifica, e l'audit log continua a non contenere alcun byte del payload.

### 3. Una modalità --explain per il benchmark

**Cosa costruire:** un flag che stampi, per ogni caso del corpus, quale layer e regola lo hanno intercettato e attraverso quale reading.

**Perché:** il benchmark indica che una classe è al 100%. Non specifica se ogni caso sia stato intercettato dalla regola prevista. Una classe può attestarsi al 100% mentre la metà dei casi viene intercettata accidentalmente da un layer diverso, il che significa che la tua regola non è testata.

**Cosa imparerai:** l'attribuzione distinta dal rilevamento, e perché esiste l'ablazione a due numeri.

**Suggerimenti:** `evaluate()` dispone già del verdetto. Fai propagare all'esterno i finding invece di un valore booleano.

**Verifica che funzioni:** eseguilo e individua almeno un caso intercettato da qualcosa di diverso dalla regola designata per la sua classe. Ce ne sono alcuni.

## Intermedio

### 4. Validazione dell'output strutturato

**Cosa costruire:** un layer che convalidi l'output del modello a fronte di uno schema prima che raggiunga l'applicazione, in modo che un modello a cui è stato detto di emettere JSON emetta JSON oppure il verdetto sia BLOCK.

**Perché:** una grossa fetta dei malfunzionamenti reali degli agent non è dovuta a esfiltrazione, ma al fatto che il modello viene convinto a emettere qualcosa che il parser a valle non si aspettava. Questa è la vulnerabilità `LLM05:2025 Improper Output Handling`.

**Cosa imparerai:** dove si colloca il confine tra una competenza del firewall e una competenza dell'applicazione, che è una linea di demarcazione su cui si può legittimamente discutere.

**Suggerimenti:** implementa la struttura del protocollo `Layer` per il lato egress. Decidi deliberatamente se una violazione dello schema debba essere un invariante o a punteggio, e metti per iscritto la motivazione.

**Verifica che funzioni:** un documento valido passa, un documento iniettato che vìola lo schema viene bloccato, e svuotare il controllo rende rosso esattamente un test.

### 5. Policy di taint per singola origine

**Cosa costruire:** un taint che trasferisca il canale di origine nella decisione sul tool, in modo che `NO_UNTRUSTED_INFLUENCE` possa essere allentato per un'origine di cui l'operatore si fida maggiormente, ad esempio un wiki interno, pur rimanendo rigoroso per una coda pubblica di ticket.

**Perché:** il modello attuale è binario. I deployment reali prevedono delle gradazioni, e forzare tutto sull'impostazione più rigida è il modo in cui un controllo finisce per essere disattivato del tutto.

**Cosa imparerai:** come aggiungere espressività a una policy senza creare una via per disabilitare accidentalmente un invariante. Questa è la parte difficile, ed è il fulcro dell'intero esercizio.

**Suggerimenti:** `Context.tainted_by` restituisce già le origini, non un booleano. `Tool.guards` è la sede a cui appartiene l'allentamento del vincolo. Considera cosa succede quando sono presenti contemporaneamente due span DATA provenienti da origini diverse, poiché la risposta non è scontata.

**Verifica che funzioni:** un'origine fidata autorizza il tool, una non fidata lo rifiuta, **e la combinazione di entrambe lo rifiuta**. Quest'ultimo caso è quello fondamentale.

### 6. Rendere il proxy onesto sui contenuti di retrieval incollati

**Cosa costruire:** un header opt-in o un campo di richiesta che consenta a un client di contrassegnare quale messaggio contiene contenuti recuperati, in modo che il proxy possa trattarlo come DATA anziché inferire USER.

**Perché:** questo è il punto debole documentato del proxy. I contenuti recuperati e incollati all'interno di un messaggio utente rappresentano ciò che fa la maggior parte delle applicazioni RAG, ed è esattamente il caso in cui l'inferenza sbaglia.

**Cosa imparerai:** perché l'API della libreria impone che la provenance sia dichiarata, costruendo la versione più ridotta possibile di questo meccanismo per un protocollo che non dispone di alcuno slot dedicato.

**Suggerimenti:** `infer_context` in `proxy/infer.py`. Il modello `ChatMessage` consente già campi aggiuntivi. Mantieni la modalità di fallimento conservativa: una marcatura non riconosciuta deve significare non attendibile, non attendibile.

**Verifica che funzioni:** la stessa injection viene bloccata se contrassegnata e non viene bloccata se non contrassegnata, e il warning di avvio continua a scattare, poiché la modalità rimane comunque più debole rispetto alla libreria.

## Avanzato

### 7. Un secondo victim agent

**Cosa costruire:** un'implementazione di `Agent` supportata da un modello locale tramite Ollama, controllata da un flag e disattivata per impostazione predefinita.

**Perché:** il mock è deterministico e ingenuo per costruzione. Un modello reale non è nessuna delle due cose. Osservare il firewall comportarsi in modo identico con entrambi è la dimostrazione che non ispeziona mai il modello.

**Cosa imparerai:** cosa cambia effettivamente quando il backend è reale, ovvero nulla sul versante del firewall e parecchio sul versante "l'attacco ha funzionato?".

**Suggerimenti:** `Agent` è composto da un solo metodo. Tienilo fuori da ogni test, da ogni gate e da ogni percorso predefinito. L'architettura offline e priva di chiavi API è una feature e romperla costituirebbe una regressione.

**Verifica che funzioni:** la stessa policy produce gli stessi verdetti con entrambi i backend a fronte del medesimo corpus. Nei punti in cui differiscono, la differenza risiede nel fatto che il modello abbia acconsentito o meno, mai in ciò che il firewall ha deciso.

### 8. Generazione e rotazione dei canary

**Cosa costruire:** derivare i canary per ogni sessione da una funzione con chiave invece di registrare valori letterali, cosicché un canary fuoriuscito identifichi quale sessione lo ha perso.

**Perché:** in un deployment reale si desidera sapere quale tenant, quale richiesta, quale documento. Un canary statico segnala che si è verificato un leak. Uno derivato indica dove è avvenuto.

**Cosa imparerai:** la tensione tra un canary su cui è possibile effettuare il matching a basso costo computazionale e uno che trasporta informazioni, e l'origine di `MIN_CANARY_LENGTH`.

**Suggerimenti:** il matcher rimuove i caratteri non alfanumerici ed esegue il casefold prima del confronto. Qualsiasi cosa venga generata deve sopravvivere a questo processo conservando almeno 8 caratteri, e `Firewall.__init__` solleverà un `PolicyError` in caso contrario.

**Verifica che funzioni:** due sessioni ottengono canary diversi, ciascuna fa match solo con il proprio, e un canary troppo corto dopo la rimozione dei caratteri spuri viene rifiutato esplicitamente in fase di costruzione anziché scartato silenziosamente.

### 9. Generazione di corpus avversari

**Cosa costruire:** un generatore che componga primitive di attacco — encoding, unicode smuggling, delimiter forgery, indirezione — in stack che il corpus scritto a mano non contiene, e segnali quali composizioni riescono a passare.

**Perché:** l'output stesso del benchmark dichiara che il corpus misura ciò a cui l'autore ha pensato. Questo è l'attacco diretto a tale limitazione.

**Cosa imparerai:** perché esiste il test esaustivo su 584 stack per egress e perché il lato ingress non ha un equivalente. Aspettati di trovare delle mancate rilevazioni reali. È proprio questo lo scopo.

**Suggerimenti:** modellalo sul property test di egress. Non promuovere automaticamente nel corpus con punteggio ciò che trovi; l'input non attendibile non deve mai registrarsi da solo nella ground truth. Scrivi i risultati in `candidates/` per la revisione, che è ciò che `just harvest` fa già.

**Verifica che funzioni:** individua almeno una composizione che riesce a passare, e sei in grado di indicare con precisione il motivo per cui le regole esistenti la mancano.

## Esperto

### 10. Un'argomentazione formale per l'invariante di fencing

**Cosa costruire:** una dichiarazione verificata automaticamente o rigorosamente argomentata della proprietà "uno span DATA non può terminare il proprio fence", comprese le assunzioni su cui poggia.

**Perché:** il progetto definisce questo aspetto come un invariante. Questo termine dovrebbe significare qualcosa di più forte di "lo abbiamo testato approfonditamente".

**Cosa imparerai:** quanta parte di una proprietà di sicurezza sia effettivamente garantita dal CSPRNG, quanta dal renderer e cosa accade al confine tra i due.

**Piano d'azione:**
1. Formula la proprietà con precisione. Su cosa si quantifica e cosa è consentito sapere all'avversario?
2. Enumera le assunzioni: entropia del nonce, freschezza per richiesta, il fatto che `render` sia l'unico percorso verso una stringa di prompt, il fatto che il modello non lasci fuoriuscire il fence nel proprio output.
3. Verifica ciascuna assunzione a fronte del codice. **Almeno una è più debole di quanto appaia**, e scoprire quale sia è l'obiettivo dell'esercizio.
4. Redigi l'argomentazione e scrivi il controesempio per l'assunzione che non regge.

**Verifica che funzioni:** sei in grado di indicare una configurazione di deployment in cui l'invariante non regge e spiegare esattamente il motivo.

### 11. Taint multi-turn attraverso una conversazione reale

**Cosa costruire:** un modello di taint che persista lungo un'intera conversazione, compreso il caso esplicitamente rimandato in fase di progettazione: un turno precedente dell'assistente che ha già superato egress ma che conteneva contenuti influenzati dall'attaccante.

**Perché:** questa è una **debolezza nota e documentata**. Mappare i turni dell'assistente come DATA applicherebbe il taint a ogni conversazione dopo il primo turno e renderebbe inutile l'autorizzazione dei tool in modalità proxy. Mapparli come USER significa che un payload sopravvissuto a un turno precedente viene riletto come semi-attendibile. Nessuna delle due scelte è corretta.

**Cosa imparerai:** perché la soluzione attuale è un compromesso documentato anziché una soluzione definitiva, e quanto costa l'approccio corretto.

**Piano d'azione:**
1. Riproducilo. Fai passare un payload attraverso egress al turno uno e fai in modo che influenzi il turno tre.
2. Determina quale sia l'effettivo livello di trust corretto per un turno precedente dell'assistente. Argomenta entrambi i punti di vista prima di costruire.
3. Prendi in considerazione un quarto livello di `Trust`, e valuta attentamente se ne valga la pena, poiché un insieme chiuso di tre valori è parte del motivo per cui il modello risulta comprensibile.
4. Misura l'impatto della tua modifica sull'autorizzazione dei tool in una normale sessione multi-turn. Se al quarto turno qualsiasi operazione viene rifiutata, hai reso l'agent inutilizzabile, il che costituisce un diverso tipo di fallimento.

**Verifica che funzioni:** il payload cross-turn viene intercettato, **e** una conversazione benigna di dieci turni autorizza ancora i propri tool al decimo turno.

## Combinazioni

- **9 più 3.** Genera stack avversari, poi usa la modalità explain per verificare quale layer ha intercettato i sopravvissuti. Troverai casi intercettati dalla regola sbagliata.
- **5 più 11.** Il taint per singola origine e il taint multi-turn rappresentano lo stesso problema su scale temporali differenti.
- **4 più 6.** La validazione dell'output unita a un'onesta provenance nel proxy ti porta vicino a una soluzione drop-in che non mente sulle proprie garanzie.

## Ottimizzazione delle prestazioni

La chiusura in `normalize/views.py` domina il p99, attestandosi a circa 3 ms. Due direzioni di intervento:

**Memoizzare tra i layer.** Sia `ingress` che `provenance` chiamano `readings()` sul testo del medesimo span all'interno della stessa richiesta. Il caching tra di essi è semplice. Il motivo per cui non è stato fatto è che l'indipendenza dei layer è una proprietà verificata direttamente dai test, e una cache condivisa li accoppierebbe. **Non farlo senza un test che dimostri che i layer rimangono indipendenti**, il che è esattamente ciò che rende interessante la sfida.

**Limitare la frontiera in modo più intelligente.** La chiusura esplora breadth-first fino a un budget prefissato. Alcune viste sono molto più produttive di altre su input reali. Misura prima di ottimizzare, perché a un certo punto la chiusura saturava a 12 varianti e portare il budget da 96 a 5000 non ha cambiato nulla.

## Lavoro sulla sicurezza

- **Sanitizzazione.** Non esiste una modalità di sanitizzazione e il design si oppone alla sua introduzione, poiché la normalizzazione non deve mai riscrivere silenziosamente contenuti legittimi. Se non sei d'accordo, implementala e sostieni la tua tesi. Definisci cosa vede il modello e cosa vede l'utente, e come un revisore possa distinguerli.
- **Rilevamento del firewall stesso.** Un modello a cui viene chiesto di descrivere il proprio prompt descriverà il fence. Ha importanza? Argomenta in entrambi i sensi, poi misura se un attaccante che conosce il formato del fence tragga alcun vantaggio.
- **Rate limiting basato sul costo, non sul conteggio.** L'arena limita i tentativi. Non limita la quantità di lavoro generata da un tentativo, e un ticket di 4.000 caratteri di codifiche concatenate costa molto di più rispetto a un ticket normale.

## Contribuire al progetto

Se individui un vero bypass, la segnalazione utile include: il payload esatto, la policy superata, quale layer avrebbe dovuto intercettarlo, e se si tratta di un ingress miss (previsto, ingress è incompleto per progettazione) o della rottura di un invariante (grave).

La distinzione è fondamentale. I miss di ingress sono a basso impatto e previsti. La violazione di un invariante è un problema serio, poiché quei layer dovrebbero reggere indipendentemente dal payload.

## Checklist di completamento

- [ ] Aggiunta una regola e un caso benigno che ne garantisce l'onestà
- [ ] Svuotato il proprio controllo e osservato il **proprio** test diventare rosso
- [ ] Eseguito `just bench` e letta la riga hard-benign, non quella aggregata
- [ ] Trovato un caso non coperto dal corpus
- [ ] Spiegato a qualcun altro perché ingress è a punteggio e provenance non lo è
- [ ] Superato il livello 6 dell'arena senza leggere il codice sorgente
- [ ] Letto un verdetto e previsto quale layer fosse scattato prima di verificare

## Come ottenere aiuto

Leggi prima di tutto il verdetto. Ogni finding indica il rispettivo layer e la relativa regola, che è esattamente lo scopo di quel campo.

`just bench` con una policy modificata è il modo più rapido per isolare quale layer sia responsabile di un comportamento imprevisto. Disattiva i layer uno alla volta.

Quando qualcosa passa mentre non dovrebbe, controlla se il tuo test passa attraverso `Firewall.inspect` oppure direttamente tramite `decide()`. La seconda opzione salta `escalate()`, e questa differenza ha nascosto un falso positivo in questo progetto per undici milestone.
