<!-- ©AngelaMos | 2026 -->
<!-- 02-ARCHITECTURE.md -->

# Architettura del sistema

Questo documento spiega come è costruito il modulo e perché è costruito in quel modo. Presuppone che tu abbia letto [01-CONCEPTS.md](./01-CONCEPTS.it.md).

## Architettura di alto livello

Il modulo è composto da tre livelli. Una sottile façade C-ABI riceve le chiamate dall'host e le valida. Un livello di stato centrale (core state) possiede il token attivo, le sessioni e gli oggetti dietro un unico lock. Un livello crittografico esegue la matematica vera e propria e il codec di storage. Le chiamate scendono, i risultati risalgono, e l'unica cosa che l'host vede mai è la tabella delle funzioni in cima.

```
   ┌──────────────────────────────────────────────────────────────┐
   │  HOST: pkcs11-tool, OpenSSL, p11-kit, Java SunPKCS11           │
   └───────────────────────────────┬──────────────────────────────┘
                                    │  C ABI, through a function-pointer table
                                    ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  LAYER 1  ABI façade                                           │
   │  main.zig          one exported symbol, the 68-slot table      │
   │  ck.zig            every CK_ type, constant, struct            │
   │  api/general       Initialize / Finalize / GetInfo             │
   │  api/slot_token    slot + token + mechanism queries, PIN admin │
   │  api/session       OpenSession / Login / Logout / op-state     │
   │  api/object        CreateObject / Find / GetAttributeValue     │
   │  api/crypto_ops    digest / sign / verify / encrypt / decrypt  │
   │  api/keymgmt       GenerateKey(Pair) / Wrap / Unwrap / Derive  │
   │  api/random        GenerateRandom / SeedRandom                 │
   └───────────────────────────────┬──────────────────────────────┘
                                    │  acquire() the instance under the lock
                                    ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  LAYER 2  core state  (one global Instance, one mutex)         │
   │  state             the Instance, init-args, generation counter │
   │  lock              the mutex (std.atomic.Mutex + spin/yield)   │
   │  session           session Table, op-state unions, GCM buffer  │
   │  object_store      attribute-bag objects, the persist codec    │
   │  token             the token record: PINs, fail counts, MK     │
   │  env               reads storage paths from std.c.environ      │
   └───────────────────────────────┬──────────────────────────────┘
                                    │  call the primitives
                                    ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  LAYER 3  crypto                                               │
   │  digest / mac      SHA-2, HMAC (pure Zig std.crypto)           │
   │  cipher            AES-CBC/CBC-PAD/GCM, RFC 3394 wrap (pure)    │
   │  ecdsa             P-256/384 sign/verify/keygen, ECDH (pure)   │
   │  rsa + openssl     RSA via libcrypto (hand-written extern)     │
   │  pin               Argon2id derive/verify (pure)               │
   │  keystore          master-key wrap, value seal/unseal (pure)   │
   └──────────────────────────────────────────────────────────────┘
```

### Scomposizione dei componenti

**`main.zig` (l'entry point).** Esporta esattamente un simbolo, `C_GetFunctionList`, che consegna all'host un puntatore a un `CK_FUNCTION_LIST` statico. I 68 puntatori a funzione di quella tabella sono collegati alle funzioni `api/*` nell'ordine canonico. Un blocco `comptime` verifica che la tabella sia di `69 * @sizeOf(usize)` byte (versione più 68 puntatori) e che `CK_ATTRIBUTE` sia di 24 byte, così una regressione dell'ABI non può compilare.

**`ck.zig` (il contratto).** La ABI Cryptoki v2.40 scritta a mano: typedef scalari, oltre 200 costanti, ogni struct disposta per la ABI Linux LP64 con allineamento naturale, i typedef dei puntatori a funzione e la struct `CK_FUNCTION_LIST`. Niente qui dentro viene eseguito; è pura forma, e quella forma è l'accordo con l'host.

**`api/*` (gli entry point).** Un file per ogni gruppo di funzioni `C_*`. Ogni entry point segue lo stesso scheletro: acquisisce l'istanza sotto il lock, valida l'handle di sessione e gli argomenti, esegue il lavoro o lo smista nel livello crittografico, imposta la lunghezza in/out, restituisce il codice `CKR_*` preciso. Queste funzioni sono il luogo in cui vivono le regole della specifica (query di lunghezza a due chiamate, la macchina a stati delle operazioni, il controllo del login).

**`core/state.zig` (la spina dorsale).** Possiede l'unica `Instance` globale: l'allocator, il backend `std.Io`, la modalità di locking, il token, la tabella delle sessioni, l'object store, il ruolo di login corrente e la master key svolta (unwrapped). Espone `acquire()` (l'unico modo di toccare l'istanza), il parser degli init-args e il contatore di generazione usato per rendere sicure le operazioni lente.

**`core/session.zig`, `core/object_store.zig`, `core/token.zig`.** I tre archivi di dati. Le sessioni contengono lo stato delle operazioni per sessione. L'object store contiene gli oggetti e il codec di persistenza. Il token contiene lo stato di autenticazione e la master key avvolta.

**`crypto/*` (la matematica).** Primitive stateless o autosufficienti. Tutto tranne RSA è `std.crypto` in puro Zig. RSA è `libcrypto` raggiunta tramite dichiarazioni `extern` scritte a mano in `openssl.zig`.

## Flusso dei dati

### Firmare con una chiave RSA, dall'inizio alla fine

Ecco cosa succede quando un host chiama `C_Sign` dopo un `C_SignInit` con una chiave RSA. Mostra ogni livello.

```
1. host: C_SignInit(session, {CKM_SHA256_RSA_PKCS}, hRsaPriv)
   api/crypto_ops C_SignInit
     state.acquire()                      -> lock, get the *Instance
     sessions.get(hSession)               -> the Session, or CKR_SESSION_HANDLE_INVALID
     signInitOp(...)                       -> classify the mechanism as RSA-sign,
                                             parse the scheme/digest params,
                                             fetch+validate the private key object
     sess.sign_op = .{ .rsa = { key, params, sig_len } }   (just the handle + params)
     return CKR_OK                         (unlock via defer)

2. host: C_Sign(session, data, dataLen, NULL, &sigLen)     (length query)
   api/crypto_ops C_Sign
     signLen(&sess.sign_op.?)             -> modulus length
     *pulSignatureLen = that; return CKR_OK

3. host: C_Sign(session, data, dataLen, sigBuf, &sigLen)   (real call)
   api/crypto_ops C_Sign, the .rsa arm
     rsaPrivateComponents(inst, op.key, CKA_SIGN)   -> re-fetch the components,
                                                       refuse if sealed or usage-denied
     rsa.sign(components, params, data, out)        -> crypto layer
       openssl.zig: rebuild EVP_PKEY from the components,
                    EVP_DigestSign (hash-then-sign) for SHA256-RSA-PKCS
     *pulSignatureLen = n
     sess.endSign()                       -> zeroize the op union, clear it
     return CKR_OK                         (unlock via defer)
```

Spiccano due punti di design. Primo, lo stato dell'operazione memorizza solo l'*handle* della chiave e i parametri già analizzati, mai il materiale della chiave; i componenti vengono recuperati di nuovo sotto il lock per la firma vera e propria. Secondo, il pattern a due chiamate (buffer NULL per conoscere la lunghezza, poi il buffer vero) è gestito in cima, e una risposta di buffer troppo piccolo non consuma l'operazione, esattamente come richiede la specifica.

### Fare login, dall'inizio alla fine

`C_Login` è il flusso più interessante perché esegue una costosa derivazione Argon2id che non deve essere fatta tenendo il lock, e deve difendersi dal fatto che il token venga reinizializzato nel frattempo.

```
1. acquire(), validate the session, read the salt/hash and (for User) the wrapped MK
2. gen = cryptoBegin(); grab io + allocator; UNLOCK   <- release the lock for the slow part
3. pin.verify(...)                 -> Argon2id, the expensive part, no lock held
   keystore.unwrap(...)            -> if User, derive the KEK and unwrap the MK
4. LOCK again; cryptoEnd()
   if generation changed since step 2  -> someone reinitialized the token: abort
   commit: set logged_in, install the MK, object_store.unlock() to unseal secrets
   token.save(...); return CKR_OK
```

Il contatore `generation` è la rete di sicurezza. `C_InitToken` e `C_InitPIN` lo incrementano. Se un secondo thread ha reinizializzato il token durante la lunga chiamata Argon2id, il thread che sta per salvare vede che la generazione è cambiata e si rifiuta di applicare il suo risultato ormai obsoleto. Questa è la classica difesa time-of-check-to-time-of-use, resa esplicita.

## Design pattern

### Il template acquire-and-defer

Ogni entry point veloce usa la stessa forma:

```zig
pub fn C_SomeOp(hSession: ck.CK_SESSION_HANDLE, ...) callconv(.c) ck.CK_RV {
    const inst = state.acquire() orelse return ck.CKR_CRYPTOKI_NOT_INITIALIZED;
    defer state.mutex.unlock();
    const sess = inst.sessions.get(hSession) orelse return ck.CKR_SESSION_HANDLE_INVALID;
    // ... do the work, return a precise CKR_* ...
}
```

`acquire()` blocca il mutex e restituisce l'istanza solo se la libreria è inizializzata; altrimenti sblocca e restituisce null così che il chiamante possa rispondere `CKR_CRYPTOKI_NOT_INITIALIZED`. Il `defer` garantisce lo sblocco su ogni percorso di ritorno. Questo pattern è usato in circa sessanta entry point ed è la ragione per cui il locking è uniforme e difficile da sbagliare.

**Perché questo e non un accessor all'istanza lock-free?** Una versione precedente esponeva un `current()` lock-free che leggeva il puntatore all'istanza senza il lock. Aveva una finestra: `C_Finalize` poteva smantellare l'allocator e la tabella delle sessioni tra il controllo di presenza e il lock vero e proprio. La correzione è stata rendere `acquire()` l'unico accessor e verificare `present` *sotto* il lock. Ora non c'è modo di toccare l'istanza senza tenere il mutex.

### Il template snapshot-unlock-recheck (per le operazioni lente)

Argon2id richiede tempo reale (deve farlo). Tenere il lock globale durante la sua esecuzione serializzerebbe l'intero modulo dietro un singolo login. Quindi gli entry point lenti (`C_Login`, `C_InitToken`, `C_InitPIN`, `C_SetPIN`) fanno uno snapshot di ciò che serve loro, registrano la generazione, rilasciano il lock, eseguono il lavoro lento, poi ribloccano e verificano la generazione prima di salvare. È l'unico modo sicuro per eseguire un calcolo lungo su uno stato condiviso senza congelare il mondo o rischiare una scrittura obsoleta.

### Tagged union per lo stato delle operazioni

Una sessione può avere un'operazione attiva per ogni tipo. Ciascuna è una tagged union Zig che contiene esattamente lo stato di cui quell'operazione ha bisogno e niente di più:

```zig
pub const SignOp = union(enum) {
    mac: mac.Mac,         // HMAC: the running hash state
    ec:  ecdsa.SignState, // ECDSA: the curve, scalar, and accumulator
    rsa: RsaSig,          // RSA: just the key handle + parsed params
};
```

Questo mantiene lo stato dell'operazione inline nella `Session` (nessuna allocazione sull'heap per sign/verify/digest), fa di "meccanismo sbagliato per questa chiamata" un semplice ramo di `switch`, e permette al teardown di azzerare l'intera union con un solo `secureZero`. L'unica eccezione è GCM, che ha bisogno di un buffer espandibile e quindi porta con sé una slice sull'heap (`GcmStream`).

## Separazione dei livelli

```
┌────────────────────────────────────────────────────────┐
│  Layer 1: ABI façade  (ck.zig, api/*)                    │
│  - Validates handles, arguments, and the operation FSM   │
│  - Translates between the C ABI and Zig types            │
│  - Does NOT do cryptography or own state                 │
└────────────────────────────────────────────────────────┘
              ↓ may call into
┌────────────────────────────────────────────────────────┐
│  Layer 2: core state  (state, session, object_store,     │
│           token, lock, env)                              │
│  - Owns the single Instance behind the mutex             │
│  - Knows objects, sessions, login, persistence           │
│  - Does NOT speak the C ABI or pick return codes         │
└────────────────────────────────────────────────────────┘
              ↓ may call into
┌────────────────────────────────────────────────────────┐
│  Layer 3: crypto  (digest, mac, cipher, ecdsa, rsa,      │
│           pin, keystore, openssl)                        │
│  - Pure primitives: takes bytes, returns bytes/errors    │
│  - Does NOT know about sessions, handles, or the ABI     │
└────────────────────────────────────────────────────────┘
```

### Perché i livelli

- **Il livello crittografico è testabile in isolamento.** Ogni file in `crypto/` ha unit test che girano senza una sessione o un token, contro vettori pubblicati (NIST SP800-38A per AES-CBC, RFC 4231 per HMAC, RFC 3394 per il key wrap, RFC 6979 per ECDSA). Un bug nella matematica viene intercettato lì, non in un test di integrazione.
- **Il livello ABI può essere verificato meccanicamente.** Poiché `ck.zig` è pura forma senza logica, `tests/abi_test.zig` può confrontarlo campo per campo con gli header OASIS tradotti.
- **I codici di ritorno vivono in esattamente un livello.** Il livello crittografico restituisce errori Zig; il livello ABI li mappa su `CKR_*`. C'è un solo `mapCipherErr`, un solo `mapSetErr`. Chi legge e cerca "da dove viene `CKR_DATA_LEN_RANGE`" ha un solo posto in cui guardare.

### Cosa vive dove

- **Il livello 1** può importare il livello 2 e il livello 3. Non possiede alcuno stato mutabile proprio.
- **Il livello 2** può importare il livello 3. Non importa mai un file `api/*` (nessuna dipendenza verso l'alto).
- **Il livello 3** importa solo `std`, `ck` (per costanti e tipi), `config` e, per RSA, i binding `openssl`. Non risale mai.

## Modelli di dati

### Object

Un oggetto è un sacchetto di attributi. Ogni attributo è un tag di tipo, un valore in byte e un flag che dice se il valore è attualmente sigillato.

```zig
pub const Attribute = struct {
    type: ck.CK_ATTRIBUTE_TYPE,  // CKA_*
    value: []u8,                  // owned bytes
    sealed: bool = false,         // true when the value is GCM ciphertext at rest
};

pub const Object = struct {
    attrs: std.ArrayList(Attribute),
    // get / set / has / getBool / isToken / isPrivate / shouldSeal / clone / deinit
};
```

Il flag `sealed` è l'analogo in memoria dell'envelope at rest. Mentre lo User ha fatto login, i valori sensitive sono in chiaro e `sealed = false`. Al logout vengono ricifrati sul posto e `sealed = true`. Un'operazione crittografica che trova sigillato un valore di cui ha bisogno restituisce `CKR_USER_NOT_LOGGED_IN`, il che copre il caso in cui una chiave sensitive del token sia stata caricata sigillata e lo user non abbia fatto login per dissigillarla.

### Object store

Un array fisso di 256 slot, ciascuno dei quali contiene opzionalmente una entry, con un contatore di handle monotono che non riusa mai un handle.

```zig
pub const Store = struct {
    slots: [config.max_objects]?Entry = @splat(null),
    next_handle: ck.CK_OBJECT_HANDLE = 1,  // never reused, so stale handles stay invalid
};
```

L'handle `0` è riservato come `CK_INVALID_HANDLE`. Un host che conserva un handle dopo `C_DestroyObject` e lo ripassa ottiene `CKR_OBJECT_HANDLE_INVALID`, non un oggetto riciclato, perché il contatore va sempre e solo in avanti.

### Session

```zig
pub const Session = struct {
    slot: ck.CK_SLOT_ID,
    flags: ck.CK_FLAGS,               // CKF_RW_SESSION, CKF_SERIAL_SESSION
    find: Find,                        // the active C_FindObjects cursor
    digest_op:  ?digest.Hasher,        // one active op of each kind, at most
    sign_op:    ?SignOp,
    verify_op:  ?VerifyOp,
    encrypt_op: ?EncryptOp,
    decrypt_op: ?DecryptOp,
    sign_recover_op:   ?RsaRecover,
    verify_recover_op: ?RsaRecover,
};
```

La tabella delle sessioni è un array fisso di 64 slot. Aprire una sessione pulisce lo slot prima del riuso; chiuderla libera qualsiasi memoria heap (il buffer GCM) e azzera lo slot. L'intera tabella viene cancellata a `C_Finalize`.

### Token record

Lo stato di autenticazione del token è una piccola struct fissa, serializzata su disco in un record `extern struct` di dimensione fissa:

```zig
pub const Token = struct {
    initialized: bool,
    label: [32]u8,
    so:   PinSlot,           // SO salt + Argon2id hash
    user: ?PinSlot,          // User salt + hash, null until C_InitPIN
    so_fail: u32,            // failure counters for lockout
    user_fail: u32,
    user_mk: ?keystore.Wrapped,  // the master key, wrapped under the User KEK
};
```

`PinSlot` è un salt e un hash; il PIN stesso non è mai presente. `user_mk` è la master key avvolta: un salt, un nonce, il ciphertext e un tag GCM. Questo è l'intero stato di autenticazione e della chiave at rest, e sta in poche centinaia di byte.

## Architettura di sicurezza

### Threat model

Ciò a cui il modulo è costruito per resistere:

1. **Furto del disco.** Un attaccante copia i file del token e degli oggetti. I valori sensitive sono sigillati sotto una master key che è avvolta sotto il PIN dello User; senza il PIN i file sono ciphertext.
2. **Attacco offline al PIN.** L'attaccante ha i file ed esegue un dizionario contro il PIN. Argon2id (64 MiB, t=3) rende costoso ogni tentativo.
3. **Attacco online al PIN.** L'attaccante tenta contro il modulo attivo. Tre fallimenti bloccano il token.
4. **Esportazione della chiave tramite l'API.** Un attaccante che ha fatto login prova a leggere o a esportare tramite wrapping la chiave. Sensitive e unextractable rifiutano.
5. **Manomissione at rest.** L'attaccante inverte byte in un valore sigillato. Il tag GCM fallisce all'unseal e il modulo fallisce in modo chiuso.
6. **Confusione dei privilegi.** Uno SO prova a leggere i segreti dello User. Non c'è alcun keyslot SO per essi.
7. **Race sul risultato obsoleto.** Un login lento entra in race con una reinizializzazione del token. Il contatore di generazione rifiuta il salvataggio obsoleto.
8. **Residuo in memoria.** Un crash dump o una lettura cold boot prova a recuperare le chiavi dalla memoria liberata o inattiva. L'azzeramento e il nuovo blocco al logout restringono quella finestra.

Ciò che è esplicitamente fuori scope:

- **Side channel fisici.** Analisi della potenza, emanazioni elettromagnetiche e attacchi microarchitetturali come cachebleed sono problemi hardware. Il lavoro software sul tempo costante affronta il timing, non la potenza. Un vero HSM aggiunge contromisure fisiche che questo software non può avere.
- **Un host malevolo mentre lo User ha fatto login.** Il modello di PKCS#11 si fida del processo chiamante durante una sessione autenticata. Se l'host è compromesso mentre il login è attivo, può chiedere al modulo di firmare qualsiasi cosa. Il modulo protegge la *chiave*, non le intenzioni dell'host.
- **La ABI Windows.** Il layout delle struct è la ABI Linux/macOS LP64 con allineamento naturale. Windows usa un packing a 1 byte e un `CK_ULONG` da 4 byte, una build separata che qui non è un obiettivo.

### Difesa in profondità

```
   On disk:     sealed values (AES-256-GCM) + wrapped MK (AES-256-GCM under Argon2id KEK)
        ↓ C_Login unseals into RAM
   In RAM:      plaintext only while User-logged-in; relocked + MK wiped on logout/close/finalize
        ↓ a sensitive value needed by an op while sealed
   At the API:  CKR_USER_NOT_LOGGED_IN; sensitive reads -> CKR_ATTRIBUTE_SENSITIVE
        ↓ every comparison of a secret
   In compute:  constant-time PIN and MAC checks; uniform error codes on RSA decrypt
```

## Strategia di storage

Ci sono due file, entrambi con percorsi risolti dall'ambiente.

- **Il file del token** (`ANGELAMOS_HSM_TOKEN`, default `$HOME/.angelamos-hsm-token`). Un singolo record di dimensione fissa che contiene lo stato di autenticazione e la master key avvolta. Viene scritto con un file temporaneo seguito da un rename, così che un crash a metà scrittura non possa lasciare un record a metà.
- **Il file degli oggetti** (`ANGELAMOS_HSM_OBJECTS`, default `$HOME/.angelamos-hsm-objects`). Un record a lunghezza variabile che contiene solo gli oggetti *token* (`CKA_TOKEN = CK_TRUE`); gli oggetti di sessione non vengono mai resi persistenti. I valori degli attributi sensitive sono sigillati; i valori pubblici sono memorizzati in chiaro.

Entrambi i record portano un magic number e una versione. La versione è stata incrementata quando è stata introdotta la sigillatura, quindi un vecchio file dell'era in chiaro viene rifiutato invece di essere interpretato male. Un file degli oggetti corrotto viene analizzato come vuoto invece di causare un crash, così uno store danneggiato degrada a "nessun oggetto" invece che a comportamento indefinito.

Questo è l'unico punto in cui il design diverge da SoftHSM2, che usa una directory per token e un file per oggetto (o un database SQLite). Un singolo file è più semplice e sufficiente per un emulatore; il layout a directory per oggetto è indicato come estensione in [04-CHALLENGES.md](./04-CHALLENGES.it.md).

## Configurazione

Ogni parametro regolabile vive in `config.zig`. Non ci sono numeri magici sparsi nel codice; un valore come il limite del buffer GCM o il costo in memoria di Argon2id viene nominato una volta e referenziato ovunque.

```bash
ANGELAMOS_HSM_TOKEN     # path to the token record (default: $HOME/.angelamos-hsm-token)
ANGELAMOS_HSM_OBJECTS   # path to the object store (default: $HOME/.angelamos-hsm-objects)
```

Leggere l'ambiente su un confine C è un piccolo problema a sé: non c'è un'arena `std.process` su cui appoggiarsi, quindi `env.zig` percorre direttamente `std.c.environ`. Costanti degne di nota in `config.zig`: lunghezza del PIN da 4 a 255, Argon2id t=3 / m=64 MiB / p=1, tre tentativi di login, chiavi AES da 16 o 32 byte, RSA da 2048 a 4096 bit, il limite di 16 MiB dello stream GCM, 256 oggetti, 64 sessioni.

## Considerazioni sulle prestazioni

Questo è un emulatore, quindi correttezza e chiarezza pesano più del throughput. Detto questo, il design evita le trappole ovvie:

- **Il lock globale viene rilasciato durante Argon2id.** Un login non congela tutte le altre sessioni per la durata della KDF. Il lock viene tenuto solo per le letture rapide dello stato e per il salvataggio finale.
- **AES-NI quando presente.** `std.crypto.core.aes` seleziona a compile time il percorso AES hardware quando il target lo ha, così CBC e GCM girano sull'unità AES della CPU, che è anche a tempo costante per design.
- **Lo stato delle operazioni è inline.** Lo stato di sign, verify e digest vive nella struct `Session` senza allocazioni sull'heap per operazione. Solo GCM, che deve bufferizzare l'intero messaggio, tocca l'heap.

Il collo di bottiglia sotto carico è il singolo mutex globale: è un design seriale, che corrisponde al modello di esecuzione seriale di PKCS#11 (`C_GetFunctionStatus` e `C_CancelFunction` esistono solo per il modello parallelo ormai defunto e restituiscono `CKR_FUNCTION_NOT_PARALLEL`). Lo sharding per slot è possibile ma inutile qui, visto che c'è un solo slot.

## Decisioni di design

### libcrypto per RSA, puro Zig per tutto il resto

**Cosa abbiamo scelto:** AES, SHA-2, HMAC, ECDSA, ECDH e Argon2id vengono da `std.crypto`. RSA linka la `libcrypto` di OpenSSL.

**Alternative considerate:** Un RSA in puro Zig. Scartato perché `std.crypto` non ha un RSA pubblico (esiste privatamente dentro il codice TLS), e le librerie RSA Zig di terze parti disponibili sono o focalizzate sulle blind signature o non abbastanza affidabili. Scrivere a mano RSA, con i suoi schemi di padding e i suoi requisiti di tempo costante, è esattamente il tipo di cosa che non dovresti scrivere a mano.

**Compromessi:** Una dipendenza C e una build leggermente più grande. Mitigati facendo il binding di `libcrypto` tramite dichiarazioni `extern` scritte a mano invece di `@cImport`, e tramite uno script di versione che esporta solo `C_GetFunctionList`, così nessuno dei simboli di OpenSSL trapela fuori dal modulo.

### Sigillatura selettiva, non cifratura dell'intero file

**Cosa abbiamo scelto:** Solo i *valori* degli attributi sensitive sono cifrati at rest. Il materiale pubblico resta in chiaro.

**Alternative considerate:** Cifrare l'intero file degli oggetti sotto la master key. Scartato perché gli oggetti pubblici e il materiale delle chiavi pubbliche devono essere visibili prima del login (puoi leggere il modulo di una chiave pubblica senza PIN). Cifrare tutto romperebbe questo comportamento e costringerebbe a fare login solo per elencare il token.

**Compromessi:** Il codec è più complesso (deve decidere attributo per attributo), ma il comportamento è corretto secondo la specifica e la superficie pubblica resta utilizzabile prima del login.

### Un singolo keyslot User, nessun keyslot SO per i segreti

**Cosa abbiamo scelto:** La master key è avvolta solo sotto il PIN dello User. Lo SO può amministrare e resettare, ma non ha alcun percorso verso i valori segreti dello User.

**Alternative considerate:** Un secondo keyslot che avvolge la MK sotto il PIN dello SO, così che lo SO possa recuperarla. Scartato per principio: uno SO che può leggere le chiavi dello User è una backdoor. Il costo è che non c'è recupero della chiave se il PIN dello User viene perso, che è il compromesso corretto per un modulo di sicurezza.

### GCM bufferizzato, non in streaming tramite libcrypto

**Cosa abbiamo scelto:** AES-GCM accumula l'intero messaggio ed esegue l'operazione autenticata una sola volta a `*Final`. Trattato in [01-CONCEPTS.md](./01-CONCEPTS.it.md) e [MECHANICS.md](./MECHANICS.it.md).

**Alternative considerate:** Usare lo streaming-decrypt EVP di libcrypto. Scartato perché rilascia plaintext non verificato prima del controllo del tag. Il buffering elimina del tutto la classe di bug RUP, al costo di un limite di 16 MiB per messaggio.

## Strategia di gestione degli errori

Gli errori sono di due tipi e vengono gestiti in due posti.

1. **Errori Zig nel livello crittografico.** Un fallimento del cipher, un out-of-memory, un record malformato. Questi sono valori `error` di Zig. Il livello crittografico non sceglie mai un codice `CKR_*`; restituisce l'errore.
2. **Codici `CKR_*` nel livello ABI.** Ogni entry point mappa l'errore Zig sul codice di ritorno preciso che la specifica richiede, tramite uno di una manciata di mapper (`mapCipherErr`, `mapSetErr`, gli switch delle singole chiamate). I fallimenti della decifratura RSA collassano in un unico codice così che non possano essere usati come padding oracle.

La regola guida è **fail closed**. Se una risigillatura al logout fallisce (per esempio per out of memory), il plaintext viene comunque pulito sul posto (`scrubUnsealed`) così che una pulizia fallita non lasci mai un segreto in chiaro. Se un'operazione lenta non riesce a dimostrare che il suo risultato è ancora attuale, restituisce `CKR_FUNCTION_FAILED` invece di salvare uno stato obsoleto.

## Estendibilità

### Aggiungere un nuovo meccanismo

La lista dei meccanismi è dati, non codice. Per aggiungerne uno:

1. Aggiungi la costante `CKM_*` a `ck.zig` (il test ABI la verificherà automaticamente contro l'header OASIS).
2. Aggiungila a `config.supported_mechanisms` e dalle un ramo `CK_MECHANISM_INFO` in `C_GetMechanismInfo` di `slot_token.zig`.
3. Implementa la primitiva in un file `crypto/*` con unit test contro un vettore pubblicato.
4. Collegala allo smistamento pertinente in `crypto_ops.zig` o `keymgmt.zig` (gli helper `modeOf` / `hashModeOf` / `isRsa*Mech`).

Il design a livelli fa sì che ogni passo sia locale: la matematica finisce nel livello 3 con i suoi test, lo smistamento nel livello 1.

### Dove andrebbe la v3.0

PKCS#11 v3.0 aggiunge un secondo entry point, `C_GetInterface`, e 24 funzioni tra cui la famiglia AEAD basata su messaggi. La ABI è disposta in modo che un `CK_FUNCTION_LIST_3_0` (lo stesso prefisso più i nuovi puntatori) possa essere aggiunto accanto alla tabella esistente, e `C_GetInterface` esportato insieme a `C_GetFunctionList`. È la più grande delle estensioni di [04-CHALLENGES.md](./04-CHALLENGES.it.md).

## Limitazioni

Sono compromessi consapevoli, non bug.

1. **Uno slot, un token.** Un vero HSM espone molti slot. Qui ce n'è esattamente uno, sempre presente. Il multi-slot significherebbe fare sharding dello stato globale per slot.
2. **Storage su file singolo.** Niente file per oggetto, niente database. Va bene per un emulatore; un backend a directory è l'approccio di SoftHSM2 ed è un'estensione elencata.
3. **Solo AES-128 e AES-256.** La standard library di Zig non ha AES a 192 bit. Supportarlo significherebbe instradare AES-192 attraverso libcrypto come RSA.
4. **Nessuna superficie v3.0.** Solo v2.40, per scelta di scope. La struttura lascia spazio per essa.
5. **ABI Linux/macOS.** Il layout packed di Windows è una build separata.

## Confronto con sistemi simili

### SoftHSM2

L'HSM software open source di riferimento, e il modello che questo progetto segue.

- **Uguale:** la suddivisione in tre parti façade / store / crypto, la gestione degli attributi a due chiamate, il PIN per token con materiale delle chiavi private cifrato, il backend crittografico OpenSSL per RSA.
- **Diverso:** SoftHSM2 memorizza ogni oggetto come file sotto una directory per token (o in SQLite); questo usa un unico file degli oggetti. SoftHSM2 supporta più token e slot; questo ne ha uno. SoftHSM2 è in C++; questo è in Zig con una ABI verificata dalla macchina, che SoftHSM2 non ha.

### Un vero HSM hardware (Thales Luna, AWS CloudHSM, YubiHSM)

- **Stessa interfaccia.** Un host non può notare la differenza a livello di API; è questo il senso di PKCS#11.
- **Confine diverso.** Un HSM hardware fa rispettare la custodia con un chip fisico e una risposta alla manomissione. Questo la fa rispettare con i confini dei processi e la disciplina del software. La versione software resiste alle minacce del modello sopra; non resiste a un attaccante con accesso fisico e un oscilloscopio.

## Riferimento ai file chiave

Mappa rapida di dove trovare le cose:

- `src/ck.zig`: la ABI. Inizia da qui.
- `src/main.zig`: il simbolo esportato e la tabella collegata.
- `src/core/state.zig`: l'istanza, `acquire()`, il contatore di generazione.
- `src/api/session.zig`: il login, il pattern snapshot-unlock-recheck in `C_Login`.
- `src/api/crypto_ops.zig`: l'intera superficie sign/verify/encrypt/decrypt.
- `src/core/object_store.zig`: gli oggetti e il codec di sigillatura.
- `src/crypto/keystore.zig`: l'envelope (wrap/unwrap, seal/unseal).
- `src/crypto/cipher.zig`: le modalità AES e il key wrap RFC 3394.

## Prossimi passi

Ora che hai capito il design:
1. Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per percorrere il codice effettivo della ABI, un'operazione dall'inizio alla fine e i pattern di gestione dei segreti.
2. Leggi [MECHANICS.md](./MECHANICS.it.md) per i dettagli crittografici di ogni meccanismo.
3. Prova a modificare `config.supported_mechanisms` e osserva come rispondono il test ABI e `C_GetMechanismInfo`.
