<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.it.md -->

# Architettura

## Panoramica ad alto livello

```
                        YOUR APPLICATION
                               |
              declares provenance explicitly
                               |
                               v
    +--------------------------------------------------+
    |                    Context                        |
    |  spans: (SYSTEM, "you are support")               |
    |         (USER,   "where is my order")             |
    |         (DATA,   "<ticket body>", origin=...)     |
    |  nonce: a3f9c1e0d47b2856   per request, CSPRNG    |
    |  tainted_by: derived from spans, never set        |
    +--------------------------------------------------+
                               |
                    firewall.inspect(ctx)
                               |
          +--------------------+--------------------+
          |                    |                    |
     normalize             ingress             provenance
     (reports)            (scored)            (invariant)
          |                    |                    |
          +--------------------+--------------------+
                               |
                     escalate() then decide()
                               |
                          Verdict  ---- BLOCK? stop here
                               |
                       firewall.render(ctx)
                               |
              +----------------+----------------+
              | <<<UNTRUSTED-a3f9c1e0d47b2856   |
              |   ...content verbatim...        |
              | <<<END-a3f9c1e0d47b2856>>>      |
              +----------------+----------------+
                               |
                        THE MODEL (untrusted)
                               |
                          AgentReply
                     text + tool_calls
                               |
                firewall.inspect_egress(reply, ctx)
                               |
                    +----------+----------+
                    |                     |
                toolauth               egress
               (invariant)           (invariant)
                    |                     |
                    +----------+----------+
                               |
                            decide()
                               |
                            Verdict
```

Due punti di ispezione, non uno solo. `inspect()` viene eseguito prima del modello. `inspect_egress()` viene eseguito dopo. Condividono l'algebra decisionale e nient'altro.

## Suddivisione dei componenti

| Modulo | Responsabilità | Legge il significato del payload? |
|---|---|---|
| `context.py` | Il prompt con tag di attendibilità (trust-tagged). Derivazione del taint | n/a |
| `verdict.py` | `Severity`, `Decision`, `Finding`, `Verdict` | n/a |
| `policy.py` | Quali layer vengono eseguiti, soglie, `escalate()`, `decide()` | n/a |
| `firewall.py` | Orchestrazione, delimitazione (fencing), wrapping fail-closed | n/a |
| `normalize/unicode.py` | Blocco tag, caratteri a larghezza zero, bidi, confusabili, NFKC | legge i byte, non decide nulla |
| `normalize/unwrap.py` | Decodifica del trasporto, sull'intero span e incorporata | legge i byte, non decide nulla |
| `normalize/views.py` | La chiusura delimitata (bounded closure) sulle letture reversibili | condiviso da tre layer |
| `layers/normalize.py` | Emette finding per ogni trasformazione richiesta | no |
| `layers/ingress.py` | Marcatori di template, testo con forma di istruzione | **sì, ed è esplicitamente etichettato** |
| `layers/provenance.py` | Presenza del nonce in DATA | no |
| `layers/toolauth.py` | Taint, guardie, effetti, schema degli argomenti | no |
| `layers/egress.py` | Chiusura dei canarini (canary closure), controllo dell'host URL | no |
| `tools.py` | `Effect`, `Guard`, `Tool`, `ToolCallRequest`, `AgentReply` | n/a |
| `audit.py` | Record JSONL che non contengono alcun contenuto dell'attaccante | n/a |
| `agent/mock.py` | Il modello volutamente ingenuo | n/a |
| `proxy/` | Superficie compatibile con OpenAI, provenienza dedotta | n/a |
| `arena/` | Sei livelli, sessioni, limiti, acquisizione dei bypass | n/a |

Una riga sta facendo qualcosa di diverso da tutte le altre, e questa è una scelta architetturale deliberata.

## Il modello dei dati

### L'attendibilità è un insieme chiuso

```python
class Trust(StrEnum):
    SYSTEM = "system"   # you wrote it
    USER   = "user"     # the human at the keyboard wrote it
    DATA   = "data"     # something on the internet wrote it
```

Tre livelli, non un valore numerico. Un punteggio numerico di attendibilità invita all'aritmetica, e non esiste alcun senso concreto in cui due span USER equivalgano a uno span SYSTEM.

### Uno span non può essere malformato

```python
@model_validator(mode = "after")
def _origin_matches_trust(self) -> Self:
    if self.trust is Trust.DATA and self.origin is None:
        raise ValueError("a DATA span must declare its origin")
    if self.trust is not Trust.DATA and self.origin is not None:
        raise ValueError(f"a {self.trust} span cannot have an origin")
    return self
```

Un contenuto non attendibile privo di una sorgente dichiarata non è rappresentabile. Non puoi dimenticare di indicare da dove proviene un documento, poiché il costruttore rifiuta l'operazione.

### Il taint è derivato, non memorizzato

Trattato in `01-CONCEPTS.it.md`, e vale la pena ribadirlo come proprietà architetturale: **non esiste alcun setter**. `tainted_by` è una `@property` calcolata a partire dagli span. La classe di bug in cui un flag e la realtà divergono non può verificarsi, poiché esiste una sola rappresentazione.

`Context` è immutabile (frozen) e ogni metodo builder restituisce una nuova istanza, cosicché un contesto non possa essere mutato a posteriori rispetto a un verdetto che lo ha già descritto.

### I finding si auto-identificano

```python
Finding(layer = "provenance", rule = "nonce-forgery",
        severity = Severity.CRITICAL, invariant = True,
        span_index = 2, evidence = "fence nonce present in DATA")
```

Ogni finding reca il proprio layer e la propria regola in modo che un consumer possa renderizzare `BLOCKED by provenance/nonce-forgery`. Questo non è un fattore puramente estetico. L'intero meccanismo didattico dell'arena si basa sul fatto che il giocatore possa vedere quale regola lo ha bloccato, quindi la leggibilità del verdetto è un requisito fondamentale anziché un dettaglio accessorio.

`evidence` è **derivata dal contenuto dell'attaccante** e viene trattata come ostile in ogni punto. Non raggiunge mai il log di audit senza digest.

## L'algebra decisionale

```
escalate(findings, policy):
    strict_data off  ->  unchanged
    strict_data on   ->  data-imperative and chat-template-marker
                         become invariant

decide(findings, policy):
    any invariant                      ->  BLOCK
    max scored severity >= threshold   ->  BLOCK
    otherwise                          ->  ALLOW
```

Tre proprietà che meritano di essere evidenziate.

**Le invarianti non sono soggette a soglia.** Nessuna impostazione di policy può disabilitarne una. `block_threshold` sposta l'asticella unicamente per i finding a punteggio.

**INFO non blocca mai.** `decide()` esclude `severity > Severity.INFO` dal confronto dei punteggi. Questo esiste affinché le segnalazioni `layer-disabled` possano essere incluse in ogni verdetto senza mai influenzarlo.

**L'escalation è una funzione pura sui finding, non un layer.** Il progetto originale prevedeva che il layer di provenance riemettesse i finding di ingress come invarianti. Ciò avrebbe richiesto che un layer ricevesse l'output di un altro layer, rompendo la struttura uniforme `inspect(ctx, policy)` che consente all'orchestratore di effettuare il wrapping di tutti i layer in modo identico. Renderla una funzione che il `Firewall` applica durante l'orchestrazione mantiene pulito il protocollo `Layer`.

## Fail-closed

Ogni chiamata ai layer è incapsulata (wrapped):

```python
def _run(self, name: str, ctx: Context) -> list[Finding]:
    if not self._enabled(name):
        return [self._disabled(name)]
    try:
        return self.layers[name].inspect(ctx, self.policy)
    except Exception as error:
        return [self._layer_error(name, error)]
```

Un'eccezione si traduce in `Finding(rule="layer-error", severity=CRITICAL, invariant=True)`, che forza un BLOCK. Un firewall che si comporta in fail-open a fronte di un'eccezione non gestita è peggiore di nessun firewall, perché segnala un esito positivo mentre non fa nulla.

Si noti il semplice `except Exception`. È intenzionale ed è l'unico punto in questa codebase in cui catturare qualsiasi eccezione è corretto: il principio è che *nessuna* modalità di errore debba mai raggiungere il modello.

`evidence` è `type(error).__name__` e mai il messaggio dell'eccezione, poiché i messaggi possono contenere l'input che li ha provocati.

**Un layer disabilitato viene registrato, non ignorato in silenzio.** `layer-disabled` con severità INFO entra nel verdetto, cosicché un consumer possa sempre distinguere tra "questo layer è stato eseguito e non ha trovato nulla" e "questo layer non è mai stato eseguito". L'arena sfrutta esattamente questo meccanismo per indicare a un giocatore che il livello 2 era superabile perché un layer era disattivato, non perché l'attacco fosse particolarmente ingegnoso.

## Modello di minaccia

**Difeso.** Un attaccante che controlla il contenuto di uno span non attendibile letto dall'agente: un ticket, un documento recuperato, una pagina web, il risultato di un tool, il corpo di un'email, il nome di un file. Questo è lo scenario realistico e di gran lunga più comune.

**Difeso, parzialmente e con onestà.** Un attaccante che controlla il turno dell'utente e cerca di persuadere l'agente a violare i propri vincoli, nella misura in cui i controlli di egress e l'autorizzazione dei tool tengono. Ingress intercetta i tentativi ingenui e manca quelli ben congegnati. Questo è dichiarato apertamente anziché nascosto.

**Non difeso.** Un attaccante che controlla il system prompt, il file di policy, i pesi del modello o il processo del firewall. Al di fuori del confine di attendibilità per definizione architetturale.

**Esplicitamente presunto ostile: il modello.** Il firewall non si fida mai dell'output del modello, dell'intento dichiarato dal modello né della conformità del modello. Il system prompt indica effettivamente al modello che gli span DATA sono inerti, poiché ciò aumenta il costo dell'attacco, ma tale istruzione è documentata come **non costituente un controllo di sicurezza**.

## Defense in depth, e cosa offre realmente ogni layer

```
attack class            normalize  ingress  provenance  toolauth  egress
----------------------  ---------  -------  ----------  --------  ------
role override                        X
delimiter forgery                    X
nonce forgery                                    X
unicode smuggling           X        X
encoded payload             X        X
embedded encoded            X        X
indirect injection                   X
tool coercion                                              X
tool arg smuggling                                         X
unconfirmed spend                                          X
encoded exfiltration                                                X
url exfiltration                                                    X
```

Il benchmark riporta questo dato come una **valutazione di ablazione a due numeri** (two-number ablation): cosa ciascun layer intercetta da solo, e cosa aggiunge allo stack completo. Sono due domande differenti.

`normalize` ottiene un punteggio di 10 da solo, e 0 marginale. Un singolo valore di leave-one-out lo riterrebbe inerte e lo eliminerebbe. Non è inerte, è **fornito di alibi** (alibied): `ingress` normalizza internamente per il proprio matching, quindi nello stack completo copre interamente la normalizzazione. Disattivando `ingress`, come accade nel livello 2 dell'arena, quei 10 casi sono l'unica cosa che si frappone tra il giocatore e un payload invisibile.

Un layer con **entrambi** i numeri pari a zero è inerte e viene rimosso. La suite di test fallisce se ciò dovesse mai accadere.

## Configurazione

Ogni costante risiede in `config.py`. Non la maggior parte. Tutte, inclusi i frammenti regex con cui sono assemblati i pattern imperativi e le tuple di codepoint con cui sono costruiti gli insiemi Unicode. Non vi sono magic number o stringhe magiche in nessun altro punto del package.

Inoltre, `config.py` è puro ASCII per progettazione. Gli insiemi Unicode sono costruiti a partire da codepoint interi anziché essere scritti come valori letterali:

```python
ZERO_WIDTH_CODEPOINTS: Final = (0x200B, 0x200C, 0x200D, 0x2060, 0xFEFF)
ZERO_WIDTH_CHARS: Final = frozenset(chr(p) for p in ZERO_WIDTH_CODEPOINTS)
```

Un file di configurazione riguardante caratteri invisibili che a sua volta contiene caratteri invisibili è un file che nessuno può revisionare.

`Policy` è la manopola di controllo a runtime: quali layer sono abilitati, la soglia di blocco, `strict_data`, la lista dei canarini, l'allowlist degli host e gli effetti proibiti. I livelli dell'arena sono file YAML caricati in istanze di `Policy`.

## Archiviazione

Quasi nulla, deliberatamente.

| Cosa | Dove | Perché |
|---|---|---|
| Sessioni | in memoria, LRU + TTL | L'arena è una demo, non un sistema di account |
| Record di audit | JSONL in `NS_AUDIT_PATH`, disabilitato per impostazione predefinita | Nessun percorso significa nessun record, che è il comportamento predefinito corretto |
| Payload di bypass | JSONL in `NS_BYPASS_PATH` | Affinché `just harvest` abbia qualcosa da leggere |
| Corpus | YAML nel repository | Revisionato a mano, mai scritto dall'applicazione |

La libreria di base non esegue **alcun tipo di I/O**. Nessuna rete, nessun filesystem, nessuna lettura dell'orologio al di fuori della misurazione del tempo trascorso. Questo è ciò che la rende testabile e ciò che rende sicuro esporre l'arena.

### Il record di audit non contiene contenuti dell'attaccante

```json
{"policy_id":"level-6","decision":"block","elapsed_ms":0.027,
 "spans":[{"trust":"data","origin":"ticket:ab15089d7d9114ef"}],
 "findings":[{"layer":"ingress","rule":"data-imperative",
              "severity":"MEDIUM","invariant":false,
              "span_index":2,"evidence_digest":"3081bc7f2caf4077"}]}
```

Sia `evidence` che il `ref` di origine sono sottoposti a digest crittografico. L'evidence deriva dal testo dello span, e in modalità proxy il ref di origine proviene direttamente dal corpo della richiesta, quindi entrambi sono influenzati dall'attaccante. Un log che l'operatore considera sicuro da conservare non deve diventare un luogo in cui vengono archiviati contenuti dell'attaccante.

## Distribuzione

```
                    Cloudflare tunnel (optional overlay)
                               |
                               v
               +----------------------------------+
               |  nginx  (serves the built UI,    |
               |          proxies /api)           |
               |  127.0.0.1:33572                 |
               +----------------+-----------------+
                               |  internal network only
               +----------------v-----------------+
               |  arena  (FastAPI, uvicorn)       |
               |  read_only: true                 |
               |  cap_drop: ALL                   |
               |  no-new-privileges               |
               |  tmpfs /tmp                      |
               |  volume /var/lib/not-sandboxed   |
               |  cpus 1.0, memory 512M           |
               +----------------------------------+
```

Il container delle API non viene mai pubblicato sull'host. Solo nginx effettua il bind su una porta, e si lega a `127.0.0.1`.

**Le porte sono numeri elevati configurabili, mai i valori predefiniti.** UI `33572`, dev UI `61878`, proxy `39441`. Eseguire il bind su una porta ben nota (well-known) è il modo più rapido per entrare in collisione con qualsiasi altro servizio che l'operatore stia già eseguendo.

**L'immagine del container incorpora il virtualenv e l'entrypoint chiama direttamente `uvicorn`.** Una versione precedente eseguiva `uv run uvicorn` all'avvio del container, il che ri-risolveva l'ambiente a ogni boot e, aspetto ancora più critico, richiedeva una cache scrivibile. Con `read_only: true` entrava in un loop di riavvio continuo a causa dell'errore `Could not create temporary file`. Nessun test avrebbe potuto intercettarlo, poiché la suite di test non avvia mai un container.

**Gli ambienti di sviluppo e produzione sono isolati tramite namespace per progettazione.** I nomi dei progetti e dei container sono valori letterali strutturali, un set per ciascun file compose, mai variabili. Ogni recipe di compose passa esplicitamente il proprio file env. Non vi è alcuna opzione di commutazione manuale tra ambienti e quindi nulla che si possa dimenticare.

## Decisioni architetturali

**Prima la libreria, poi il proxy.** Il proxy è un wrapper di comodo in cui la provenienza è dedotta e pertanto più debole. Progettare prima il proxy avrebbe reso la modalità debole la modalità predefinita.

**Un agente mock deterministico come backend predefinito.** Poiché il firewall non ispeziona mai il modello, il comportamento del firewall è identico sia che il backend sia il mock, un modello locale o un'API ospitata. Il mock rende i fallimenti inequivocabili: se il segreto fuoriesce, è il firewall ad aver fallito, e la giustificazione "il modello ha avuto una giornata no" non è ammessa come spiegazione. Inoltre viene eseguito nella CI, offline, gratuitamente e senza alcuna variabilità (flakiness).

**Il chiamante dichiara la provenienza.** Questo è l'unico compito che l'API delega all'utente e non si tratta di una svista. È l'unico modo onesto per sapere cosa sia non attendibile, e l'API è strutturata in modo tale che tale dichiarazione sia inevitabile: `Context.data()` richiede un `Origin`.

**I livelli dell'arena sono configurazioni, non livelli di difficoltà.** Il livello 3 non è "più difficile", bensì "la provenance è ora attiva". Un giocatore che supera il livello 2 impara che normalizzazione più ingress non sono sufficienti, il che rappresenta un insegnamento autentico e utile. Regolare la difficoltà non avrebbe insegnato nulla.

**La normalizzazione è un `Layer`, non un pre-passaggio svincolato.** Il contratto di fail-closed deve coprirlo, e la struttura uniforme è ciò che consente all'orchestratore di incapsulare in modo identico tutti e tre i layer lato richiesta. Il costo è che `ingress` normalizza nuovamente al suo interno, per cui tale lavoro viene svolto due volte. Questo è intenzionale: l'indipendenza dei layer è una proprietà che i test verificano direttamente, e condividere una shadow in cache li avrebbe accoppiati.

## Prestazioni

```
p50   0.53 ms
p99   3.09 ms
```

Il costo principale è rappresentato dalla chiusura delimitata (bounded closure) in `normalize/views.py`, utilizzata da `ingress`, `provenance` ed `egress`. È contingentata due volte: `MAX_VARIANT_ROUNDS` limita la profondità, mentre un limite di varianti per chiamante limita l'ampiezza. `egress` riceve 96 letture, `ingress` e `provenance` ne ricevono 24, poiché ingress viene eseguito su ogni richiesta ed egress viene eseguito sull'output del modello.

Quella chiusura è la ragione per cui il p99 è di tre millisecondi anziché di un quarto di millisecondo. Prima che venisse applicata a ingress, il p50 era di 0,05 ms e un payload codificato incorporato nella prosa passava indisturbato. Questo costo computazionale compra una reale classe di rilevamento ed è dichiarato esplicitamente anziché nascosto.

Entrambe le direzioni sono limitate. `MAX_NORMALIZE_BYTES` limita l'input del decoder a 256 KiB. `MAX_EGRESS_BYTES` limita la scansione in uscita a 64 KiB, per stringa e in aggregato sull'intera risposta, e il suo superamento costituisce un **BLOCK invariante**. Rifiutarsi di scansionare un messaggio in uscita non è un motivo valido per rilasciarlo.

## Estensibilità

**Una nuova regola a punteggio** viene inserita in `ingress.py` con il rispettivo pattern in `config.py`, assieme ai casi di corpus benigno. Se non riesce a superare il corpus benigno, non viene rilasciata.

**Un nuovo layer invariante** implementa il protocollo `Layer`, si registra in `LAYER_ORDER` e necessita di una propria classe di benchmark affinché l'ablazione possa dimostrare che sia strutturalmente portante.

**Una nuova guardia per i tool** aggiunge un membro `Guard`, una costante di regola in `TOOLAUTH_INVARIANT_RULES` e un ramo in `_guard_findings`. Esiste un test strutturale che fallisce se esiste un membro `Guard` privo di una regola sottostante; questo test esiste perché `USER_CONFIRMED` era stato dichiarato e non veniva letto da nessuna parte per undici milestone.

**Un nuovo backend** implementa `Agent`, che consiste in un singolo metodo: `respond(prompt) -> AgentReply`.

## Limitazioni

- **Ingress è incompleto per progettazione.** Non un bug, ma una caratteristica intrinseca della categoria.
- **La provenienza del proxy è dedotta ed è errata per la maggior parte delle applicazioni RAG.** Il retrieval incollato viene letto come USER.
- **Un turno precedente dell'assistant viene mappato come USER, non DATA.** Mapparlo come DATA contaminerebbe (taint) ogni conversazione dopo il primo turno e renderebbe inutile l'autorizzazione dei tool in modalità proxy. Un payload sopravvissuto a un turno precedente viene riletto come semi-attendibile, e questa è una reale debolezza qui documentata anziché lasciata a una scoperta successiva.
- **Non esiste una modalità di sanitizzazione.** Il progetto prevedeva originariamente una terza decisione, `SANITIZE`, che avrebbe inviato al modello il testo normalizzato anziché l'originale. Era stata dichiarata e mai implementata, e la stesura di questa documentazione è ciò che lo ha portato alla luce: `Decision.SANITIZE` e `Verdict.sanitized` sono esistiti per undici milestone con zero punti di produzione nell'intera codebase. Entrambi sono stati eliminati anziché lasciati al loro posto, poiché un membro pubblico di un enum viene interpretato come una funzionalità rilasciata. La logica stessa del progetto sconsiglia di costruirla: la normalizzazione non deve mai riscrivere silenziosamente il contenuto legittimo di un utente, il che è l'intera ragione per cui il layer segnala invece di decidere. **Il testo originale è ciò che raggiunge il modello, sempre.**
- **Le sessioni sono in-process.** Due repliche dell'arena non condividono lo stato.

## Confronto

| | Questo progetto | Librerie di guardrail | LLM-as-judge | Classificatore fine-tuned |
|---|---|---|---|---|
| Resiste alla parafrasi | layer invarianti sì, ingress no | no | no | parzialmente |
| Richiede un modello | no | di solito no | **sì** | sì |
| Esso stesso vulnerabile a injection | no | no | **sì** | no |
| Funziona offline | sì | sì | no | con i pesi |
| Blocca l'uso improprio dei tool | sì | raramente | no | no |
| Blocca l'esfiltrazione | sì | a volte | no | no |
| Onesto sui propri limiti | è il punto focale | variabile | raramente | raramente |

La riga fondamentale è l'ultima. La maggior parte dei tool in questo ambito riporta un tasso di rilevamento e nessun tasso di falsi positivi, oppure riporta entrambi a fronte di un corpus creato da loro stessi. Questo progetto riporta il tasso di falsi positivi sulla classe complessa, su una riga dedicata, e ti avvisa nel suo stesso output di non citare il dato sul rilevamento fuori contesto.
