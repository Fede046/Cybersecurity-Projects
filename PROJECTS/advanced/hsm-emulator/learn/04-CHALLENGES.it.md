<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.md -->

# Sfide di estensione

Hai un HSM software funzionante. Ora rendilo tuo. Queste sfide sono ordinate per difficoltà, e ognuna nomina i file e le funzioni reali che toccherai così non devi andare a cercarli. Inizia dalle facili per imparare la forma della codebase, poi vai più a fondo.

La regola d'oro di questo progetto si applica a ogni sfida: niente è finito finché uno strumento esterno non lo mette alla prova. Aggiungi uno unit test contro un vettore pubblicato, poi dimostralo tramite `pkcs11-tool` o lo smoke harness.

## Sfide facili

### Sfida 1: aggiungere SHA-224 come meccanismo di digest

**Cosa costruire:** Supportare `CKM_SHA224` accanto ai digest SHA-256/384/512 esistenti.

**Perché è utile:** SHA-224 è ancora richiesto da alcuni profili di compliance, e aggiungerlo ti insegna il percorso esatto che un nuovo meccanismo compie attraverso i livelli.

**Cosa imparerai:**
- Il percorso in quattro passi "aggiungi un meccanismo" descritto in [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md)
- Come il test ABI verifica gratuitamente una nuova costante

**Suggerimenti:**
- Aggiungi `CKM_SHA224` a `ck.zig`. Il test "every hand-coded constant equals its OASIS value" in `tests/abi_test.zig` verificherà automaticamente il valore contro l'header.
- Aggiungi un ramo `sha224` alla union `Hasher` in `crypto/digest.zig` (la std di Zig ha `sha2.Sha224`), e assegnagli un nuovo tag di stato per la serializzazione dello stato delle operazioni.
- Aggiungilo a `config.supported_mechanisms` e al ramo digest di `C_GetMechanismInfo` in `slot_token.zig`.

**Verifica che funzioni:**
```bash
pkcs11-tool --module zig-out/lib/libhsm.so --hash --mechanism SHA224 --input-file msg.bin
```
Confronta con `sha224sum msg.bin`. Aggiungi uno unit test in `digest.zig` contro il vettore `SHA-224("abc")`.

### Sfida 2: esporre un dettaglio delle informazioni del token che controlli tu

**Cosa costruire:** Rendere la stringa del modello o il numero di serie del token configurabile tramite una variabile d'ambiente invece che tramite una costante fissa.

**Perché è utile:** Nei deployment reali i token hanno delle label. È un'introduzione delicata ai livelli config e slot-token.

**Cosa imparerai:**
- Come `config.zig` centralizza ogni costante
- Come `env.zig` legge l'ambiente sul confine C (qui non c'è un'arena `std.process`)

**Suggerimenti:**
- `C_GetTokenInfo` in `slot_token.zig` riempie i campi `model` e `serialNumber` a partire da `config.token_model` e `config.token_serial` tramite `util.padded`.
- Aggiungi una lettura dell'ambiente in `env.zig` e ripiega sulla costante quando la variabile non è impostata, rispecchiando come funzionano i percorsi di storage.
- Ricorda che i campi sono array fissi riempiti con spazi, non stringhe terminate da NUL.

**Verifica che funzioni:** `pkcs11-tool -T` dovrebbe mostrare il tuo valore, e una variabile non impostata dovrebbe mostrare il default.

### Sfida 3: aggiungere una recipe `just` che firma e poi verifica con OpenSSL

**Cosa costruire:** Una recipe che genera una chiave EC nel modulo, firma un file, esporta la chiave pubblica e verifica la firma con la riga di comando `openssl`.

**Perché è utile:** La verifica incrociata contro un'implementazione indipendente è esattamente il modo in cui questo progetto dimostra la correttezza. La stai trasformando in un one-liner.

**Cosa imparerai:**
- Come `pkcs11-tool` e `openssl` interoperano attraverso il modulo
- Perché un oracolo indipendente è una prova più forte di un self-test

**Suggerimenti:**
- Guarda le recipe esistenti nel `justfile` e il flusso dello smoke per la sequenza di comandi.
- `pkcs11-tool --read-object --type pubkey` esporta la chiave pubblica; `openssl dgst -verify` controlla la firma.

**Verifica che funzioni:** La recipe esce con 0 e stampa `Verified OK`.

## Sfide intermedie

### Sfida 4: AES-192

**Cosa costruire:** Supportare chiavi AES a 192 bit per `CKM_AES_KEY_GEN`, `CKM_AES_CBC`, `CKM_AES_CBC_PAD` e `CKM_AES_GCM`.

**Perché è utile:** Attualmente non è supportato perché la standard library di Zig non espone AES a 192 bit (vedi la sezione 3.3 di [CONFORMANCE.md](./CONFORMANCE.it.md)). Risolverlo ti insegna lo stesso pattern di ponte verso libcrypto che usa RSA.

**Applicazione nel mondo reale:** Alcuni profili FIPS e sistemi legacy richiedono specificamente AES-192.

**Approccio di implementazione:**
1. Decidi il backend. La std di Zig non ha Aes192, quindi instrada le chiavi a 192 bit attraverso l'AES EVP di libcrypto, nel modo in cui `rsa.zig` instrada RSA.
2. Aggiungi a `openssl.zig` le dichiarazioni extern di cui hai bisogno (le funzioni cipher EVP).
3. Fai un branch sulla lunghezza della chiave nelle funzioni di blocco e GCM di `cipher.zig`, chiamando libcrypto per le chiavi da 24 byte e mantenendo la std per quelle da 16 e 32.
4. Aggiorna `cipher.validKeyLen` e i controlli `CKR_KEY_SIZE_RANGE` per accettare 24.

**Suggerimenti:**
- Le funzioni `encBlockRaw` / `decBlockRaw` attualmente fanno `switch (key.len)` su 16 e 32 con `unreachable` altrimenti. Quello switch è dove va il ramo per 24 byte.
- Mantieni i pattern di tempo costante e di azzeramento identici a quelli dei rami esistenti.

**Verifica che funzioni:** Un vettore NIST AES-192-CBC in `cipher.zig`, più un round trip con `pkcs11-tool` usando una chiave da 24 byte.

### Sfida 5: un backend con un file per oggetto

**Cosa costruire:** Sostituire il singolo file degli oggetti con una directory in cui ogni oggetto token è un file a sé, come fa SoftHSM2.

**Perché è utile:** Un singolo file significa riscrivere tutto a ogni modifica. I file per oggetto ti permettono di scrivere solo ciò che è cambiato e rendono il backup una copia per oggetto.

**Cosa imparerai:**
- I compromessi di design dello storage (un file contro molti)
- Come il codec di persistenza in `object_store.zig` separa la serializzazione dall'I/O

**Approccio di implementazione:**
1. Mantieni gli esistenti `serialize` / `parse` per gli attributi di un singolo oggetto; stai cambiando solo il modo in cui sono raggruppati su disco.
2. Dai a ogni oggetto un id stabile su disco (l'handle è monotono e non viene mai riusato, il che aiuta).
3. Scrivi la logica di sigillatura senza modifiche; ogni file oggetto sigilla comunque i suoi valori sensitive sotto la master key.

**Suggerimenti:**
- La logica seal-on-write e mark-sealed-on-read in `object_store.zig` non si preoccupa di quanti file ci siano. Mantienila.
- `env.zig` oggi risolve un percorso; tu risolverai una directory.

**Verifica che funzioni:** Crea diversi oggetti, riavvia il processo e conferma che vengano tutti ricaricati. Corrompi un file oggetto e conferma che va perso solo quell'oggetto, non l'intero store.

## Sfide avanzate

### Sfida 6: implementare l'interface discovery di PKCS#11 v3.0

**Cosa costruire:** Esportare `C_GetInterface` accanto a `C_GetFunctionList`, restituendo un `CK_FUNCTION_LIST_3_0` agli host che chiedono la v3.0.

**Perché è difficile:** Stai estendendo la ABI stessa, la parte che deve corrispondere all'host byte per byte. Un errore qui significa che gli host vanno in crash, non che si comportano semplicemente male.

**Cosa imparerai:**
- Come funziona l'interface discovery della v3.0 e perché gli host ripiegano su `C_GetFunctionList` quando `C_GetInterface` è assente (il percorso di fallback di OpenJDK)
- Come `CK_FUNCTION_LIST_3_0` condivide un prefisso con `CK_FUNCTION_LIST` e accoda i nuovi puntatori

**Modifiche architetturali necessarie:**
```
   C_GetInterface(name, version, &iface, flags)
        │
        ▼
   CK_INTERFACE { pInterfaceName, pFunctionList -> CK_FUNCTION_LIST_3_0, flags }
        │
   (C_GetFunctionList still returns the v2.40 CK_FUNCTION_LIST prefix)
```

**Passi di implementazione:**
1. Aggiungi `CK_FUNCTION_LIST_3_0` e `CK_INTERFACE` a `ck.zig`, con i 24 nuovi typedef di puntatori a funzione della v3.0.
2. Costruisci un'unica tabella più grande e consegna un puntatore tipizzato nell'uno o nell'altro modo, impostando correttamente il campo versione per ogni chiamante.
3. Esporta `C_GetInterface` in `main.zig` e aggiungilo allo script di versione in `pkcs11.map`.
4. Estendi `tests/abi_test.zig` per verificare le nuove struct contro gli header OASIS v3.0 (mettili in vendor accanto a quelli v2.40).

**Insidie:**
- I primi N membri di `CK_FUNCTION_LIST_3_0` devono essere identici byte per byte a `CK_FUNCTION_LIST`. Il test di cross-check è la tua rete di sicurezza.
- Le nuove funzioni v3.0 che non implementi hanno comunque bisogno di funzioni stub reali e correttamente tipizzate nella tabella. Mai null.

**Risorse:** La specifica PKCS#11 v3.1, e la nota di ricerca `08-pkcs11.md` in `docs/zig/reference/` che tratta la superficie v3.0.

### Sfida 7: l'API AEAD basata su messaggi della v3.0

**Cosa costruire:** Implementare `C_EncryptMessage` / `C_DecryptMessage` (e le varianti begin/next/final) per AES-GCM.

**Perché è difficile:** L'API a messaggi è il modo in cui il provider di OpenSSL 3.x preferisce sempre più spesso fare GCM, e ha un contratto di gestione dei nonce diverso dal classico `C_Encrypt`.

**Cosa imparerai:**
- La forma dell'API a messaggi della v3.0 e come differisce dai flussi single-shot e update/final della v2.40
- Come mantenere la tua posizione sicura rispetto al RUP con la nuova API

**Approccio di implementazione:**
1. Richiede prima la Sfida 6 (la tabella v3.0).
2. L'API a messaggi passa il nonce e l'AAD tramite parametri della famiglia `CK_MESSAGE` invece che tramite il parametro del meccanismo.
3. Mantieni per il decrypt il comportamento bufferizzato, verify-before-release, che il percorso classico ha già in `GcmStream`.

**Suggerimenti:** Il tuo accumulatore GCM esistente e la validazione rigorosa dei parametri in `buildCipher` sono la maggior parte del lavoro; la parte nuova è il parsing dei parametri dell'API a messaggi.

### Sfida 8: fork safety

**Cosa costruire:** Rilevare un `fork()` e rifiutarsi di operare nel processo figlio con lo stato obsoleto ereditato dal padre, oppure reinizializzare in modo pulito.

**Perché è difficile:** Un processo figlio eredita lo stato inizializzato del padre, inclusa una master key svolta, il che è un rischio reale. La risposta di PKCS#11 è il flag `CKF_INTERFACE_FORK_SAFE`, ma solo se lo gestisci davvero.

**Cosa imparerai:**
- Perché lo stato crittografico ereditato attraverso un fork è pericoloso
- Come tracciare il process id e invalidare lo stato quando cambia

**Approccio di implementazione:**
1. Registra il pid nella `Instance` a `C_Initialize`.
2. In `state.acquire()`, confronta il pid corrente; in caso di discrepanza, tratta la libreria come non inizializzata nel figlio (forzando una reinizializzazione) invece di operare sui segreti del padre.
3. Solo allora pubblicizza `CKF_INTERFACE_FORK_SAFE` (un flag di interfaccia v3.0, quindi questa sfida si abbina alla Sfida 6).

**Insidie:** La master key ereditata da un figlio deve essere cancellata, non usata. Fai attenzione che il rilevamento avvenga prima che qualsiasi operazione possa toccare `inst.mk`.

## Sfide per esperti

### Sfida 9: un backend di storage SQLite con supporto multi-token

**Cosa costruire:** Sostituire il backend su file con SQLite, ed eliminare il limite di un solo token così che il modulo possa ospitare più token su più slot.

**Tempo stimato:** Una settimana o più.

**Prerequisiti:** Dovresti aver fatto prima la Sfida 5 (storage per oggetto), perché questa si basa sulla separazione del codec dall'I/O.

**Cosa imparerai:**
- Come un vero HSM software (SoftHSM2 ha esattamente questo) struttura lo storage multi-token
- Come fare sharding dello stato globale per slot senza perdere la semplicità del lock unico, o come passare a lock per slot

**Pianificare questa funzionalità:**

Prima di scrivere codice, rifletti su:
- Come cambia la `Instance` globale con il multi-slot? Oggi è un token, una tabella delle sessioni, un object store. Ti serve una collezione indicizzata per slot.
- Il singolo mutex globale è ancora adeguato, o vuoi lock per slot? I lock per slot aggiungono concorrenza reale ma anche un rischio reale di deadlock.
- Come migri un token esistente su file singolo nel database alla prima esecuzione?

**Architettura di alto livello:**
```
   slots: [N] of Token
     each Token: its own PIN state, master key, object store
   storage: SQLite (one table for tokens, one for objects, sealed values as BLOBs)
   locking: either the one global mutex (simple) or per-slot (concurrent)
```

**Fasi di implementazione:**

**Fase 1: multi-token in memoria.** Trasforma il singolo token/store/sessioni in `Instance` in collezioni indicizzate per slot. Mantieni il backend su file. Fai in modo che `C_GetSlotList` restituisca più slot e che le operazioni per slot vengano instradate correttamente.

**Fase 2: il backend SQLite.** Linka SQLite (il ponte verso libcrypto in `openssl.zig` è il tuo modello per fare il binding di una libreria C in modo pulito). Sposta serialize/parse in modo che leggano e scrivano righe. Lascia intatta la logica di sigillatura; i valori sono ancora BLOB sigillati con GCM.

**Fase 3: migrazione e concorrenza.** Migra un token su file esistente alla prima apertura. Decidi e implementa il modello di locking.

**Fase 4: rifinitura.** Gestione degli errori per un database bloccato o corrotto, e il passaggio completo dei test tramite `pkcs11-tool` contro più token.

**Criteri di successo:**
- [ ] `C_GetSlotList` restituisce più di uno slot
- [ ] Due token contengono chiavi e PIN indipendenti
- [ ] Le chiavi sopravvivono a un riavvio, caricate da SQLite
- [ ] Un database corrotto o bloccato fallisce in modo chiuso, non restituisce mai una chiave sbagliata
- [ ] Il test ABI e lo smoke harness passano ancora senza modifiche

## Combina le sfide

Combina le sfide in progetti più grandi:

**Un modulo nativo v3.0.** Sfida 6 (interfaccia) più Sfida 7 (API a messaggi) più Sfida 8 (fork safety) ti danno un modulo con cui un moderno provider di OpenSSL 3.x parla nativamente.

**Uno store di livello produzione.** Sfida 5 (per oggetto) più Sfida 9 (SQLite, multi-token) ti danno qualcosa con le capacità di storage di SoftHSM2.

## Sfide di integrazione nel mondo reale

### Pilotare il modulo dal provider pkcs11 di OpenSSL

**L'obiettivo:** Usare questo modulo come key store dietro OpenSSL stesso, così che i comandi `openssl` firmino e decifrino attraverso di esso.

**Cosa ti servirà:**
- Il `pkcs11-provider` (il moderno provider di OpenSSL 3.x) configurato per puntare a `libhsm.so`
- Una configurazione di OpenSSL che carichi il provider

**Attenzione a:** Il provider mette alla prova angoli che `pkcs11-tool` non tocca, in particolare riguardo ai parametri RSA-PSS e a GCM. È un ottimo modo per trovare lacune di conformità. Qualsiasi cosa faccia inciampare è un bug reale o una restrizione documentata in [CONFORMANCE.md](./CONFORMANCE.it.md).

### Usare il modulo da Java tramite SunPKCS11

**L'obiettivo:** Far sì che un programma Java generi una chiave e firmi attraverso il modulo usando `java.security` e il provider SunPKCS11.

**Cosa imparerai:** SunPKCS11 è il classico consumatore con fallback rispetto a `C_GetInterface`, quindi questo valida anche che il tuo percorso `C_GetFunctionList` sia solido (e motiva la Sfida 6 se vuoi il percorso v3.0).

## Sfide di prestazioni

### Ridurre la contesa sul lock con sessioni concorrenti

**L'obiettivo:** Permettere a molte sessioni di fare crittografia in parallelo invece di serializzarsi sull'unico mutex globale.

**Collo di bottiglia attuale:** Ogni entry point prende `state.mutex`. Due sessioni che firmano contemporaneamente si serializzano, anche se le loro operazioni sono indipendenti.

**Approcci:**
- **Stato delle operazioni per sessione, lock solo per lo stato condiviso.** Lo stato delle operazioni vive già nella `Session`; potresti fare la crittografia fuori dal lock e bloccare solo per recuperare e salvare, nel modo in cui `C_Login` fa già per Argon2id.
- **Lock per slot.** Si abbina al lavoro multi-slot della Sfida 9.

**Fai un benchmark:** Avvia N thread che firmano ciascuno in un loop e misura il throughput prima e dopo. Fai attenzione alle race; il pattern del contatore di generazione in `state.zig` è il tuo modello per un lock-release-relock sicuro.

## Sfide di sicurezza

### Dimostrare il comportamento a tempo costante con Valgrind

**L'obiettivo:** Usare memcheck di Valgrind e `classify` / `declassify` di Zig per rilevare qualsiasi branch o accesso in memoria che dipenda da un segreto.

**Cosa imparerai:** La cosa più vicina a un verificatore di tempo costante che Zig abbia. `classify` marca la memoria segreta come non inizializzata per Valgrind, che poi segnala qualsiasi condizionale che dipenda da essa.

**Implementazione:** Aggiungi una build di debug che classifica l'hash del PIN e il materiale delle chiavi prima della crittografia, eseguila sotto `valgrind --tool=memcheck --track-origins=yes`, e insegui qualsiasi segnalazione "conditional jump depends on uninitialised value". La nota di ricerca `16-constant-time-security.md` in `docs/zig/reference/` illustra l'API.

**Testare la sicurezza:** L'obiettivo è un'esecuzione pulita, senza avvisi di condizionali su segreti, attraverso una firma e un decrypt completi.

### Fare fuzzing del codec di persistenza

**L'obiettivo:** Lanciare file di token e di oggetti malformati contro `parse` e confermare che fallisca sempre in modo chiuso (svuotandosi) e non vada mai in crash né legga fuori dai limiti.

**Threat model:** Questo protegge da un file di store malevolo o corrotto. Il codec ha già controlli di lunghezza e di limiti; il fuzzing dimostra che sono completi.

**Implementazione:** Genera byte di record casuali e mutati, passali a `object_store.parse` e `token.loadFrom`, e verifica che il risultato sia un parsing valido o uno store vuoto pulito, mai un panic. I test esistenti "rejects a bad magic" e "fails safe on a truncated record" sono i tuoi punti di partenza.

## Idee per contribuire

Hai finito qualcosa? Condividilo:

1. Fai il fork del repo.
2. Implementa la tua estensione in un branch, con unit test contro un vettore pubblicato e una prova tramite smoke harness o `pkcs11-tool`.
3. Documentala: aggiorna [CONFORMANCE.md](./CONFORMANCE.it.md) se hai cambiato un confine, e aggiungi una nota al documento learn pertinente.
4. Apri una PR con l'implementazione, i test e la prova con lo strumento esterno.

Le buone estensioni (un nuovo meccanismo con vettori, un vero backend di storage, la superficie v3.0) sono esattamente il tipo di cosa che rende il progetto più utile alla persona successiva.

## Mettiti ancora più alla prova

### Costruisci qualcosa di nuovo

Usa ciò che hai imparato qui per costruire:
- Un **provider PKCS#11 per un tipo di chiave diverso**, come Ed25519 o ML-DSA (entrambi sono nella std di Zig), seguendo il percorso di ECDSA.
- Un **host minimale** che carica qualsiasi modulo PKCS#11 e lo mette alla prova, il lato consumatore dell'interfaccia che hai appena implementato.

### Studia implementazioni reali

Leggi queste e ruba le loro buone idee:
- **SoftHSMv2** per lo store a directory per token e l'astrazione crittografica OpenSSL/Botan.
- **OpenSC `pkcs11-spy`** per come uno shim trasparente registra ogni chiamata, che è un pezzo pulito di ingegneria della ABI di per sé.

## Ottenere aiuto

Bloccato su una sfida?

1. **Fai debug in modo sistematico.** Cosa ti aspettavi, cosa è successo, qual è l'input più piccolo che lo riproduce? Esegui il caso che fallisce tramite `just spy` e leggi la chiamata e il codice di ritorno esatti.
2. **Leggi il codice esistente.** Il meccanismo che stai aggiungendo ha quasi certamente un fratello già implementato. Il percorso è lo stesso.
3. **Appoggiati al cross-check.** Se hai toccato `ck.zig`, esegui `zig build test`; il test ABI ti dirà con precisione quale campo o costante è diverso dagli header OASIS.

## Completamento delle sfide

Tieni traccia dei tuoi progressi:

- [ ] Facile 1: digest SHA-224
- [ ] Facile 2: informazioni del token configurabili
- [ ] Facile 3: recipe di verifica incrociata con OpenSSL
- [ ] Intermedia 4: AES-192 tramite libcrypto
- [ ] Intermedia 5: backend con un file per oggetto
- [ ] Avanzata 6: `C_GetInterface` v3.0
- [ ] Avanzata 7: API a messaggi v3.0
- [ ] Avanzata 8: fork safety
- [ ] Esperto 9: SQLite, multi-token

Le hai finite tutte? Hai costruito un HSM software multi-token, compatibile con la v3.0, con un vero backend database e un comportamento a tempo costante dimostrato. A quel punto capisci la custodia delle chiavi a un livello che la maggior parte degli ingegneri in attività non raggiunge mai. Vai a leggere la documentazione PKCS#11 di un vero vendor di HSM; riconoscerai ogni decisione di design che hanno preso e il perché.
