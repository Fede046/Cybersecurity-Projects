# Sfide

Hai letto l'intero progetto. Capisci perché ogni riga è com'è. E adesso?

La risposta onesta è: **costruisci qualcosa sopra di esso.** Il modo più rapido per imparare davvero ciò che hai appena letto è estenderlo. Le sfide qui sotto sono ordinate grosso modo dalla più facile alla più difficile. Scegline una che ti interessa, abbozza cosa cambieresti e prova.

Se rimani bloccato, i pezzi rilevanti del codice esistente sono linkati. Se ne completi una e vuoi condividerla, fai il fork del repo e apri una PR: il tier foundations è pensato come un trampolino di lancio, non come qualcosa di finito.

## Una nota sull'ambito

Non provare a farle tutte. Non provare nemmeno a farne cinque. Scegline *una*, falla bene e fermati. Lo scopo non è aggiungere funzionalità: è interiorizzare il codice esistente interagendoci.

Per ogni sfida qui sotto troverai:

- **Cosa** — una o due frasi che descrivono la funzionalità.
- **Perché è interessante** — cosa imparerai costruendola.
- **Da dove iniziare** — quali file toccherai.
- **Attenzione a** — le trappole di sicurezza o di correttezza in cui cadono i principianti.

---

## Tier 1 — piccole funzionalità (~30 minuti ciascuna)

### 1. Aggiungere un comando `search`

**Cosa:** `pv search <substring>` elenca ogni nome di entry che contiene la sottostringa (senza distinzione tra maiuscole e minuscole). Come `pv list`, ma filtrato.

**Perché è interessante:** Facile da fare, ma ti costringe a leggere con attenzione `main.py` e `vault.py` e a trovare il punto giusto in cui aggiungere un nuovo comando. Un buon riscaldamento per una "prima PR".

**Da dove iniziare:** Copia il comando `list_entries` in `main.py`. Filtra `unlocked.names()` prima di renderizzare la tabella.

**Attenzione a:**
- Confronto senza distinzione tra maiuscole e minuscole: usa `name.lower()` e `query.lower()`.
- Una query di ricerca vuota dovrebbe essere rifiutata, non restituire "ogni entry".

### 2. Aggiungere un comando `count`

**Cosa:** `pv count` stampa solo il numero di entry. Utile per gli script shell.

**Perché è interessante:** Il più piccolo comando nuovo possibile. Ti costringe a pensare al formato di output (numero + newline, nessuna decorazione) in modo che possa essere messo in pipe: `if [ "$(pv count)" -eq 0 ]; then ...`.

**Da dove iniziare:** Copia `gen`, il comando esistente più semplice. Leggi le entry, stampa `len(unlocked.entries)`.

**Attenzione a:**
- Usa `print()`, non `console.print()`, in modo che l'output sia adatto alle pipe.
- Un vault vuoto stampa comunque `0`, non "vault is empty."

### 3. Mostrare un timestamp di "ultimo utilizzo"

**Cosa:** Aggiungi un campo `last_used_at` a `Entry`. Il comando `get` lo aggiorna (e salva il vault).

**Perché è interessante:** Toccherai ogni livello: la dataclass `Entry`, i suoi `from_dict`/`to_dict`, il percorso di salvataggio e `get` in `main.py`. Un buon modo per vedere l'architettura in movimento.

**Da dove iniziare:** Aggiungi il campo con `field(default_factory=lambda: "")` in modo che i vecchi vault si aprano anche senza di esso. Aumenta il conteggio implicito dei campi in `from_dict`.

**Attenzione a:**
- I vecchi vault non avranno il campo: gestisci il caso della chiave mancante nello stesso modo in cui lo fanno `created_at` e `updated_at`.
- `get` ora modifica il vault → deve chiamare `save()` prima che termini il blocco `with`.
- Questo cambia il threat model: un attaccante che ruba il vault ora scopre *quale entry hai usato più di recente*. Documenta il compromesso.

### 4. Nascondere le password in `get` a meno che non venga passato `--show`

**Cosa:** `pv get github` mostra tutto tranne la password (`••••••••`). `pv get github --show` mostra la password vera.

**Perché è interessante:** Una piccola modifica di UX, ma è il tipo di funzionalità che i veri password manager offrono per i momenti "sono in riunione e sto condividendo lo schermo".

**Da dove iniziare:** Aggiungi un flag `--show / -s` al comando `get`. In `_render_entry`, ramifica in base al flag.

**Attenzione a:**
- Imposta come default nascosto, con visibilità opt-in: sicuro per impostazione predefinita.
- Il carattere punto elenco `•` potrebbe non essere visualizzato in ogni terminale: fornisci un'alternativa.

---

## Tier 2 — funzionalità medie (qualche ora ciascuna)

### 5. Implementare `pv export` e `pv import`

**Cosa:** `pv export <path>` scrive ogni entry in un file JSON in chiaro (dopo aver chiesto la master password e mostrato un forte avviso "SEI SICURO"). `pv import <path>` fa l'inverso.

**Perché è interessante:** Questa funzionalità è reale, utile e pericolosa. Reale perché ogni password manager ha bisogno di migrazione in ingresso e in uscita. Utile perché gli utenti muoiono, perdono i telefoni, cambiano strumenti. Pericolosa perché le credenziali in chiaro su disco sono esattamente ciò che abbiamo costruito questo tool per evitare.

**Da dove iniziare:** Aggiungi due comandi a `main.py`. L'export serializza `unlocked.entries` e scrive un file JSON con modalità 0600. L'import legge il JSON, convalida la struttura, chiama `add_entry` per ciascuna.

**Attenzione a:**
- Il file di export è in chiaro. Stampa un gigantesco avviso rosso prima di scrivere.
- Imposta come percorso di export predefinito `./pv-export.json`, non la home directory dell'utente: fagli pensare a dove risiederà.
- L'import dovrebbe gestire con garbo il caso "entry già esistente": o chiede, o accetta un flag `--force`, o salta.
- Convalida il JSON importato nello stesso modo in cui lo convalida `Entry.from_dict`: non fidarti della struttura del file.
- Un file esportato con permessi di filesystem deboli è una vera trappola. Imposta esplicitamente la modalità 0600 con `os.open` (lo stesso trucco usato da `_atomic_write`).

### 6. Aggiungere il punteggio di robustezza della password

**Cosa:** Quando un utente esegue `pv add`, mostragli un punteggio di robustezza (weak/fair/strong/excellent) prima di salvare. Usa una libreria come [`zxcvbn-python`](https://github.com/dwolfhub/zxcvbn-python).

**Perché è interessante:** Una funzionalità di UX pratica. Ti costringe ad aggiungere una nuova dipendenza (toccando `pyproject.toml` e `uv.lock`) e a capire la differenza tra "sembra casuale a un essere umano" e "sopravvivrebbe a un attacco di indovinello offline".

**Da dove iniziare:** Aggiungi `zxcvbn` a `pyproject.toml`. Chiamalo sulla password inserita. Mappa il punteggio 0-4 su un colore e un'etichetta. Permetti all'utente di procedere comunque.

**Attenzione a:**
- Non *bloccare* l'utente dal salvare una password debole: potrebbe avere una buona ragione. Avvisa e chiedi.
- Il punteggio dovrebbe essere mostrato prima che la password diventi permanente: se aspetti fino a dopo `save()`, l'utente deve eliminare e riaggiungere per riprovare.

### 7. Aggiungere un comando `pv copy <name>`

**Cosa:** Copia una password negli appunti di sistema senza stamparla. Usa [`pyperclip`](https://github.com/asweigart/pyperclip).

**Perché è interessante:** I veri password manager lo fanno. Ti costringe a pensare alle differenze tra piattaforme (le API degli appunti differiscono su Linux/Mac/Windows) e al compromesso di sicurezza di tenere le credenziali negli appunti.

**Da dove iniziare:** Aggiungi `pyperclip` alle dipendenze. Nuovo comando che sblocca il vault, recupera l'entry, ne copia la password e stampa una conferma (NON la password).

**Attenzione a:**
- Su Linux serve `xclip` o `wl-clipboard` installato a livello di sistema: documentalo.
- Il contenuto degli appunti persiste finché qualcos'altro non lo sovrascrive. Sfida bonus: avvia un thread in background che svuota gli appunti dopo 30 secondi.
- Pyperclip nelle sessioni SSH remote non funziona: gestisci il fallimento al momento dell'import con un errore chiaro.

### 8. Aggiungere un flag `--verify` a `init`

**Cosa:** Dopo aver creato un vault, prova subito a sbloccarlo con la stessa password. Se lo sblocco fallisce, vai nel panico: qualcosa è corrotto.

**Perché è interessante:** Difesa in profondità. Il ciclo salvataggio → nuovo sblocco avrebbe intercettato alcune classi di bug durante lo sviluppo. Costruisce l'abitudine di "testare il percorso di lettura a ogni scrittura" da cui traggono beneficio i progetti basati su formati di file.

**Da dove iniziare:** Aggiungi un flag `--verify / -V` a `init`. Dopo che `UnlockedVault.create()` ritorna, chiama `UnlockedVault.unlock()` con la stessa password. Stampa un segno di spunta verde o un panico rosso.

**Attenzione a:**
- La chiamata di verifica costa un'altra derivazione completa di Argon2 (~0,5s). Rendila opt-in, non predefinita.
- Se la verifica *fallisce*, qualcosa non va molto: rifiuta di lasciare al suo posto il nuovo vault. Eliminalo.

---

## Tier 3 — funzionalità più grandi (un weekend ciascuna)

### 9. Aggiungere TOTP (password monouso basate sul tempo)

**Cosa:** Alcuni siti usano TOTP per la 2FA. Attualmente conservi il secret altrove (Google Authenticator, Authy). Aggiungi un campo per il secret TOTP a `Entry` e un comando `pv totp <name>` che stampa il codice corrente a 6 cifre.

**Perché è interessante:** Imparerai cos'è davvero TOTP (è l'[RFC 6238](https://datatracker.ietf.org/doc/html/rfc6238) ed è sorprendentemente semplice: HMAC-SHA1 della finestra corrente di 30 secondi). La libreria Python `pyotp` lo fa in due righe, ma scrivere il nucleo da soli è un esercizio da 30 righe.

**Da dove iniziare:** Aggiungi `pyotp` alle dipendenze. Aggiungi un campo opzionale `totp_secret` a `Entry`. Aggiungi il comando in `main.py`.

**Attenzione a:**
- Il secret TOTP è sensibile almeno quanto la password. Appartiene *dentro* il vault cifrato, non in un file a parte.
- Lo sfasamento dell'orologio conta. Stampa "questo codice è valido ancora per X secondi" in modo che l'utente non provi a usarne uno che sta per scadere.
- Importare i secret TOTP da codici QR è un progetto a sé: non provare a farlo qui.

### 10. Rendere trasparente l'aggiornamento del costo della KDF

**Cosa:** Quando un utente sblocca un vault i cui parametri Argon2 sono inferiori ai default attuali del codice, ricalcola *automaticamente* la chiave con i nuovi default e salva: stessa password, derivazione più forte. Stampa un messaggio: "Vault parameters upgraded to current defaults."

**Perché è interessante:** Questo è ciò che fanno i veri password manager. È il motivo per cui memorizziamo i parametri della KDF nel file. Ti costringe a capire a fondo il flusso di `change_master_password` e a pensare alla UX per un'"operazione lunga che gli utenti non hanno richiesto".

**Da dove iniziare:** All'interno di `UnlockedVault.unlock`, dopo la decifratura riuscita, confronta `kdf_parameters` con `KdfParameters.defaults()`. Se differiscono, esegui una modifica in loco (chiama qualcosa come `change_master_password(same_password, new_kdf_parameters=...)`) e salva.

**Attenzione a:**
- Paga il nuovo costo di Argon2 una sola volta, non due. Rifattorizza `change_master_password` in modo che possa anche "aggiornare in loco" senza ruotare la password.
- L'utente vedrà due pause consecutive di ~0,5 secondi. Spiega loro il motivo con un messaggio prima della seconda.
- Aggiungi un flag `--no-auto-upgrade` per gli utenti che non lo vogliono.

### 11. Implementare un comando `pv backup` con snapshot versionati

**Cosa:** `pv backup` scrive una copia del vault corrente in `~/.password-vault/backups/vault-YYYY-MM-DD-HHMMSS.json` e conserva gli ultimi N backup. Aggiungi un comando `pv restore <timestamp>` che sovrascrive il vault attivo a partire da un backup.

**Perché è interessante:** I sistemi reali hanno bisogno di backup, ma i backup sono *anche* una superficie di sicurezza: sono altri file che un attaccante può rubare. Ti costringe a pensare a: quanti conservarne, dove memorizzarli, se cifrare anche l'indice dei backup, cosa succede al restore (vuoi semantica atomica).

**Da dove iniziare:** Aggiungi i due comandi. Usa il pattern di scrittura atomica esistente (non scrivere il backup con `Path.write_bytes`; usa `_atomic_write`). Usa il pattern di file lock esistente.

**Attenzione a:**
- I backup sono copie complete del file cifrato: sono cifrati, quindi sono "sicuri da perdere davanti alla disk forensics" *nella stessa misura* del vault attivo, ma non di più.
- Potare i vecchi backup richiede cautela: non eliminare il file che sta attualmente venendo letto da un altro processo `pv`. Usa lo stesso advisory lock.
- "Ripristina dal backup N" deve convalidare che il backup N sia un vault *vero* (analizza l'envelope, controlla la versione) prima di sovrascrivere il file attivo.

### 12. Aggiungere una web UI in un comando separato `pv web`

**Cosa:** Un web server solo locale (`localhost:8080`) che serve una semplice UI per navigare nel vault. Si spegne automaticamente dopo 10 minuti di inattività.

**Perché è interessante:** Ti costringe a pensare a *ogni* compromesso di sicurezza a cui non dovevi pensare con una CLI. Gestione della master password in un browser, CSRF, XSS, timeout di sessione, HTTPS o meno su localhost, cosa fare quando si apre una seconda scheda.

**Da dove iniziare:** Usa [`starlette`](https://www.starlette.io) o [`fastapi`](https://fastapi.tiangolo.com) per il server. Renderizza le entry con i template Jinja2. Non usare i cookie: usa un'unica sessione in memoria che scade automaticamente.

**Attenzione a:**
- È davvero più difficile di quanto sembri. Le web UI dei veri password manager sono lavori di ingegneria a tempo pieno. Lo scopo di *questa* versione è imparare quali sono i compromessi, non rilasciare un tool di produzione.
- Il browser ora fa parte del tuo threat model. Le estensioni del browser possono leggere il DOM. Altre schede possono navigare verso il tuo `localhost:8080`. La same-origin policy è il tuo unico amico.
- Logging: Starlette/FastAPI registrerà gentilmente ogni richiesta. Assicurati che l'access log non includa la master password (non lo farà, se fai correttamente il POST del form, ma controlla).

---

## Tier 4 — progetti di taglio ricerca

Questi non hanno una forma pulita del tipo "costruisci questa esatta funzionalità". Sono direzioni in cui spingere il progetto se hai assorbito tutto e vuoi continuare.

### 13. Fare l'audit del threat model di un vero password manager

Scegli un vero password manager open source: [Bitwarden](https://github.com/bitwarden), [KeePassXC](https://github.com/keepassxreboot/keepassxc), [pass](https://www.passwordstore.org). Leggi la sua documentazione e il suo codice sorgente per i pezzi equivalenti di *questo* progetto: come deriva le chiavi, come memorizza il file, come gestisce la rotazione della master password? Scrivi un confronto.

Imparerai che i veri password manager fanno compromessi diversi, a volte per buone ragioni, a volte per ragioni legacy. L'esercizio di identificare *quale è quale* è il tipo di analisi che gli ingegneri della sicurezza fanno di mestiere.

### 14. Scrivere un lettore del formato vault in un altro linguaggio

Il formato del vault è documentato in [02-ARCHITECTURE.it.md §3](./02-ARCHITECTURE.it.md#3-il-formato-del-file-del-vault-su-disco) e le chiavi JSON sono costanti in `constants.py`. Scrivi un client di sola lettura in Rust, Go o in qualsiasi cosa tu stia imparando dopo. Troverai le librerie da usare (`argon2`, `aes-gcm`), farai corrispondere versioni/parametri e convaliderai il round trip tra linguaggi.

È un esercizio davvero valido. Dimostra *perché* abbiamo messo per iscritto il formato, e probabilmente fa emergere punti in cui il formato sottospecifica qualcosa (il tipo di cose che i progetti di interoperabilità del mondo reale scoprono di continuo).

### 15. Fare il threat modeling di una debolezza deliberata

Scegli un'assunzione dalla sezione sul threat model di [01-CONCEPTS.it.md §12](./01-CONCEPTS.it.md#12-mettere-tutto-insieme-il-threat-model). Prova a sconfiggerla.

Esempi:
- "Non ci difendiamo da un keylogger." Prova a scrivere un keylogger Python che osserva lo stdin del processo `pv` (scoprirai che è sorprendentemente difficile perché `getpass` legge dal dispositivo terminale, non da stdin). Poi prova a scriverne uno che si aggancia a livello di OS su Linux: di quali permessi ha bisogno?
- "Non cancelliamo davvero la chiave dalla memoria." Usa uno strumento di debug della memoria (`gcore` + `strings`) su un processo `pv` in esecuzione per trovare la chiave AES. Poi progetta cosa proteggerebbe *davvero* da questo e spiega perché non l'abbiamo implementato.

L'obiettivo non è trasformare qualcosa in un'arma. È sentire visceralmente la differenza tra "non dichiariamo di difenderci da X" e "non potremmo farlo nemmeno volendo".

---

## Cosa leggere dopo

Se sei arrivato fino a 03-IMPLEMENTATION e hai finito con questo progetto:

- **Il [tier intermediate](../../../intermediate/)** — progetti che coinvolgono web server, database e più file. Il salto da questo progetto a quelli è molto più piccolo ora di quanto non fosse prima che iniziassi.
- **Il [tier advanced](../../../advanced/)** — progetti che coinvolgono veri sistemi distribuiti e primitive di sicurezza serie.
- **[Crypto 101](https://www.crypto101.io)** di Laurens Van Houtven — un libro gratuito che approfondisce ogni idea crittografica in questo progetto. Particolarmente consigliato se hai trovato interessanti i §8-9 in [01-CONCEPTS.it.md](./01-CONCEPTS.it.md) e vuoi il contesto completo.
- **[Cryptography Engineering](https://www.schneier.com/books/cryptography-engineering/)** di Ferguson, Schneier e Kohno — il libro di testo. Più impegnativo di Crypto 101, ma il riferimento definitivo per "come i sistemi crittografici falliscono davvero nella pratica".

Hai finito il progetto più difficile del tier foundations. Ora sai più sulla conservazione delle password nel mondo reale degli ingegneri responsabili di [la maggior parte delle violazioni che abbiamo citato](./01-CONCEPTS.it.md#13-violazioni-reali-che-hanno-reso-giuste-queste-scelte). Vale la pena fermarsi un attimo ad apprezzarlo.
