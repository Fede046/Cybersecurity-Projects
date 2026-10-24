<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# Guida all'implementazione

Questo documento percorre il codice effettivo. Presuppone che tu abbia letto [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md). Il codice è referenziato per nome di file e di funzione, così puoi aprire il file e cercarlo. Gli snippet sono reali, leggermente ridotti per mantenere il focus.

## Panoramica della struttura dei file

```
src/
├── ck.zig             # the hand-written ABI: types, constants, structs, the function list
├── main.zig           # exports C_GetFunctionList, wires the 68-slot table, comptime asserts
├── config.zig         # every constant: sizes, KDF params, the mechanism list
├── util.zig           # padded(): space-pad a fixed C string field at comptime
├── core/
│   ├── state.zig       # the global Instance, acquire(), the generation counter, finalize()
│   ├── lock.zig        # the mutex
│   ├── env.zig         # storage-path resolution from std.c.environ
│   ├── token.zig       # the token record + atomic save/load
│   ├── session.zig     # the session Table, op-state unions, the GcmStream buffer
│   └── object_store.zig# the Object/Store types + the persistence codec with sealing
├── api/
│   ├── general.zig     # Initialize / Finalize / GetInfo / WaitForSlotEvent
│   ├── slot_token.zig  # slot/token/mechanism queries + InitToken / InitPIN / SetPIN
│   ├── session.zig     # OpenSession / Login / Logout / Get+SetOperationState
│   ├── object.zig      # CreateObject / Find / GetAttributeValue / SetAttributeValue
│   ├── crypto_ops.zig  # the digest/sign/verify/encrypt/decrypt/dual surface (the big one)
│   ├── keymgmt.zig     # GenerateKey(Pair) / WrapKey / UnwrapKey / DeriveKey
│   └── random.zig      # GenerateRandom / SeedRandom
└── crypto/
    ├── openssl.zig     # hand-written extern EVP/BN/OSSL_PARAM declarations
    ├── pin.zig         # Argon2id derive / verify
    ├── digest.zig      # the SHA-2 Hasher union + serializable state
    ├── mac.zig         # the HMAC-SHA-2 Mac union
    ├── cipher.zig      # AES-CBC/CBC-PAD/GCM + RFC 3394 key wrap
    ├── ecdsa.zig       # P-256/384 keygen, sign, verify, ECDH
    ├── rsa.zig         # the stateless libcrypto RSA bridge
    └── keystore.zig    # the envelope: master-key wrap/unwrap, value seal/unseal
```

## Costruire la ABI: il contratto con l'host

### I tipi scritti a mano

`ck.zig` è l'intero accordo con l'host, ed è pura forma. I tipi scalari corrispondono alla ABI Linux LP64:

```zig
pub const CK_BYTE = u8;
pub const CK_ULONG = c_ulong;   // 8 bytes on LP64 Linux/macOS; this is the #1 ABI hazard
pub const CK_RV = CK_ULONG;
pub const CK_SESSION_HANDLE = CK_ULONG;
pub const CK_OBJECT_HANDLE = CK_ULONG;
```

Usare `c_ulong` (non `u64`) è deliberato: è qualunque cosa sia l'`unsigned long` del C della piattaforma, che è esattamente ciò che usano gli header dell'host. Se sbagli questo, ogni campo di struct dopo il primo `CK_ULONG` si trova all'offset sbagliato, e l'host legge spazzatura senza accorgersene.

Le struct sono semplici `extern struct` con allineamento naturale. Niente `packed`, niente `align(1)`:

```zig
pub const CK_ATTRIBUTE = extern struct {
    type: CK_ATTRIBUTE_TYPE,
    pValue: ?*anyopaque,
    ulValueLen: CK_ULONG,
};
```

Il testo della specifica dice che le strutture sono packed a 1 byte, ma su Linux e macOS gli header reali non impostano alcun pragma di packing e usano l'allineamento naturale. Questa è la ABI de facto che ogni host Linux usa realmente, quindi `extern struct` è corretto e `packed` sarebbe sbagliato.

### La function list e l'unico simbolo esportato

Il cuore della ABI è `CK_FUNCTION_LIST`: una versione seguita da 68 puntatori a funzione in un ordine fisso che non puoi riordinare. `main.zig` ne costruisce un'unica istanza statica ed esporta il singolo simbolo che ne consegna l'indirizzo:

```zig
export fn C_GetFunctionList(ppFunctionList: *?*ck.CK_FUNCTION_LIST) callconv(.c) ck.CK_RV {
    ppFunctionList.* = &function_list;
    return ck.CKR_OK;
}

var function_list: ck.CK_FUNCTION_LIST = .{
    .version = ck.CK_VERSION{ .major = 2, .minor = 40 },
    .C_Initialize = general.C_Initialize,
    .C_Finalize = general.C_Finalize,
    // ... every one of the 68 slots, in canonical order ...
    .C_WaitForSlotEvent = general.C_WaitForSlotEvent,
};
```

Ogni slot punta a una vera funzione `callconv(.c)`. Nessuno è null, perché un host chiama attraverso questi puntatori senza controllare, e uno slot null provoca un segfault dentro l'host. Le operazioni deliberatamente non supportate puntano a una vera funzione che restituisce il codice `CKR_*` giusto, mai a null.

### Dimostrare il layout a compile time

`main.zig` non compila se la tabella ha la dimensione sbagliata:

```zig
comptime {
    std.debug.assert(@sizeOf(ck.CK_FUNCTION_LIST) == 69 * @sizeOf(usize));
    std.debug.assert(@sizeOf(ck.CK_ATTRIBUTE) == 24);
}
```

La tabella è la versione più 68 puntatori, quindi 69 larghezze di puntatore. `CK_ATTRIBUTE` è composta da tre campi da otto byte, quindi 24. Questi controlli costano poco, ma la vera prova è il cross-check a build time contro gli header OASIS, trattato più avanti.

### Lo script di versione

`pkcs11.map` è ciò che tiene nascosto tutto il resto:

```
PKCS11_2_40 {
    global:
        C_GetFunctionList;
    local:
        *;
};
```

`build.zig` lo applica con `lib.setVersionScript`. Tutto tranne `C_GetFunctionList` è `local`, quindi `objdump -T zig-out/lib/libhsm.so` mostra esattamente un simbolo esportato. Le altre funzioni `C_*` sono raggiungibili solo attraverso la tabella, e nessuno dei simboli di libcrypto trapela.

## Costruire un entry point: il template acquire-and-defer

Ogni entry point veloce ha lo stesso scheletro. `C_GetSessionInfo` da `api/session.zig` è un esempio pulito:

```zig
pub fn C_GetSessionInfo(hSession: ck.CK_SESSION_HANDLE, pInfo: *ck.CK_SESSION_INFO) callconv(.c) ck.CK_RV {
    const inst = state.acquire() orelse return ck.CKR_CRYPTOKI_NOT_INITIALIZED;
    defer state.mutex.unlock();
    const s = inst.sessions.get(hSession) orelse return ck.CKR_SESSION_HANDLE_INVALID;
    pInfo.* = .{
        .slotID = s.slot,
        .state = sessionState(s.flags, inst.logged_in),
        .flags = s.flags,
        .ulDeviceError = 0,
    };
    return ck.CKR_OK;
}
```

`state.acquire()` è l'unico modo di toccare l'istanza. La sua implementazione blocca il mutex, verifica che la libreria sia inizializzata *sotto il lock*, e se non lo è sblocca e restituisce null:

```zig
pub fn acquire() ?*Instance {
    mutex.lock();
    if (!@atomicLoad(bool, &present, .acquire)) {
        mutex.unlock();
        return null;
    }
    return &storage;
}
```

Il `defer state.mutex.unlock()` nel chiamante garantisce che il lock venga rilasciato su ogni percorso, incluso il ritorno anticipato `CKR_SESSION_HANDLE_INVALID`. È per questo che non c'è alcun leak del lock: non puoi dimenticare lo sblocco perché `defer` lo esegue per te.

### Il pattern di lunghezza a due chiamate

Gli output a lunghezza variabile usano la danza a due chiamate della specifica: passi un buffer null per conoscere la dimensione, poi passi un buffer vero. Ecco il percorso di encrypt da `crypto_ops.zig`, il ramo AES:

```zig
const need: ck.CK_ULONG = @intCast(cipher.encryptOutLen(c.mode, in.len));
if (pEncryptedData == null) {
    pulEncryptedDataLen.* = need;          // first call: report the size
    return ck.CKR_OK;
}
if (pulEncryptedDataLen.* < need) {
    pulEncryptedDataLen.* = need;          // buffer too small: report size again
    return ck.CKR_BUFFER_TOO_SMALL;        // and do NOT consume the operation
}
```

La sottigliezza importante è che un `CKR_BUFFER_TOO_SMALL` non smantella l'operazione. Ci si aspetta che l'host riprovi con un buffer più grande, e lo stato dell'operazione deve essere ancora lì quando lo fa. Una chiamata single-shot riuscita, invece, *termina* l'operazione (vedrai `sess.endEncrypt(...)` dopo che i byte sono stati scritti).

## Fare login: il pattern snapshot-unlock-recheck

`C_Login` in `api/session.zig` è la funzione più accurata della codebase, perché esegue una costosa derivazione Argon2id che non deve girare mentre si tiene il lock globale, e non deve salvare un risultato obsoleto se il token è stato reinizializzato durante quella derivazione.

Passo uno: acquisire, validare e leggere ciò di cui la parte lenta ha bisogno. Poi registrare la generazione, prendere io e allocator, e rilasciare il lock:

```zig
const gen = state.cryptoBegin();   // returns the current generation, marks an op in flight
const io = inst.io();
const allocator = inst.allocator();
state.mutex.unlock();              // release the lock for the slow part
defer std.crypto.secureZero(u8, &hash);
```

Passo due: il lavoro lento, senza alcun lock tenuto. Verificare il PIN e, se si tratta di un login User, svolgere la master key:

```zig
const ok = pin.verify(io, allocator, pinSlice(pPin, ulPinLen), &salt, &hash) catch {
    state.cryptoAbort();
    return ck.CKR_FUNCTION_FAILED;
};
// ... if ok and userType == CKU_USER, keystore.unwrap(...) the master key ...
```

Passo tre: ribloccare, e rifiutare se il mondo è cambiato nel frattempo:

```zig
state.mutex.lock();
defer state.mutex.unlock();
state.cryptoEnd();
if (state.currentGeneration() != gen) return ck.CKR_FUNCTION_FAILED;  // token was reinitialized
if (inst.logged_in != null) return ck.CKR_USER_ALREADY_LOGGED_IN;
if (ok) {
    inst.logged_in = userType;
    if (have_mk) {
        inst.mk = mk;
        object_store.unlock(allocator, &inst.objects, mk) catch {
            inst.relock();
            inst.logged_in = null;
            return ck.CKR_FUNCTION_FAILED;
        };
    }
    token.save(inst.io(), inst.token) catch {};
    return ck.CKR_OK;
}
```

Il controllo di `generation` è la difesa time-of-check-to-time-of-use resa esplicita. `C_InitToken` e `C_InitPIN` chiamano `state.bumpGeneration()`. Se una delle due è stata eseguita mentre questo login stava derivando Argon2id, il thread che salva vede `currentGeneration() != gen` e butta via il suo risultato obsoleto invece di fare login in un token che non esiste più nella forma che aveva controllato.

In caso di successo, `object_store.unlock` percorre ogni valore sensitive sigillato e lo decifra sul posto con la master key ora svolta. Se questo fallisce (un valore sigillato corrotto), la funzione riblocca e rifiuta il login, così uno store danneggiato non può lasciarti a metà dissigillato.

## L'object store e il codec di sigillatura

### Gli oggetti sono sacchetti di attributi

`object_store.zig` modella un oggetto come una lista di attributi, ciascuno con un flag `sealed`. `set` sostituisce o accoda, liberando in modo sicuro il vecchio valore e cancellando qualsiasi flag sealed obsoleto:

```zig
pub fn set(self: *Object, allocator: std.mem.Allocator, t: ck.CK_ATTRIBUTE_TYPE, bytes: []const u8) !void {
    if (bytes.len > config.max_attr_value_len) return error.AttrTooLarge;
    if (self.findPtr(t)) |a| {
        const dup = try allocator.dupe(u8, bytes);
        secureFree(allocator, a.value);   // scrub the old value before freeing
        a.value = dup;
        a.sealed = false;                  // a fresh plaintext value is not sealed
        return;
    }
    // ... append a new attribute ...
}
```

### Cosa viene sigillato

`shouldSeal` decide se un dato attributo su un dato oggetto è materiale segreto che deve essere cifrato at rest. È segreto se il tipo è uno degli attributi di materiale della chiave (il valore AES/HMAC, o l'esponente privato RSA e i fattori CRT) e l'oggetto è marcato sensitive o non estraibile:

```zig
pub fn shouldSeal(self: *const Object, t: ck.CK_ATTRIBUTE_TYPE) bool {
    if (!isSecretMaterial(t)) return false;
    if (self.getBool(ck.CKA_SENSITIVE)) return true;
    return self.has(ck.CKA_EXTRACTABLE) and !self.getBool(ck.CKA_EXTRACTABLE);
}
```

Un valore pubblico (un modulo, un punto EC, una label) restituisce false e viene memorizzato in chiaro, così resta leggibile prima del login.

### Il codec: sigilla in uscita, marca come sigillato in entrata

`serialize` scrive solo gli oggetti token, e sigilla ogni valore sensitive mentre lo scrive:

```zig
if (!a.sealed and e.obj.shouldSeal(a.type)) {
    const key = mk orelse return error.NoMasterKey;
    const scratch = try allocator.alloc(u8, keystore.sealedLen(a.value.len));
    defer allocator.free(scratch);
    const wrote = try keystore.seal(io, &key, std.mem.asBytes(&a.type), a.value, scratch);
    // append the sealed bytes
} else {
    // append the plaintext value
}
```

Nota `std.mem.asBytes(&a.type)` come associated data: il tipo dell'attributo è legato al tag GCM, quindi un `CKA_PRIVATE_EXPONENT` sigillato non può essere spostato in uno slot che si aspetta un `CKA_VALUE` senza rompere l'autenticazione.

`parse` rilegge i valori e, dopo il caricamento, marca quelli sensitive come sigillati così che non vengano trattati come plaintext utilizzabile finché `C_Login` non li dissigilla. L'intero caricamento è fail-safe: un magic sbagliato, una versione sbagliata, un record troncato o troppi attributi fanno tutti sì che lo store venga svuotato invece di andare in crash.

## Il buffer GCM sicuro rispetto al RUP

`GcmStream` in `session.zig` è l'accumulatore che rende sicuro il GCM in streaming. `append` fa crescere un buffer sull'heap, azzerando in modo sicuro la vecchia memoria sottostante a ogni realloc, e applica il limite di 16 MiB:

```zig
pub fn append(self: *GcmStream, allocator: std.mem.Allocator, bytes: []const u8) error{ OutOfMemory, TooLarge }!void {
    if (bytes.len == 0) return;
    const needed = self.len + bytes.len;
    if (needed > config.max_gcm_stream_len) return error.TooLarge;
    if (self.buf == null or self.buf.?.len < needed) {
        var new_cap: usize = if (self.buf) |b| b.len else 256;
        while (new_cap < needed) new_cap *|= 2;
        if (new_cap > config.max_gcm_stream_len) new_cap = config.max_gcm_stream_len;
        const fresh = try allocator.alloc(u8, new_cap);
        if (self.buf) |old| {
            @memcpy(fresh[0..self.len], old[0..self.len]);
            std.crypto.secureZero(u8, old);   // scrub the old buffer, it held plaintext
            allocator.free(old);
        }
        self.buf = fresh;
    }
    @memcpy(self.buf.?[self.len..][0..bytes.len], bytes);
    self.len += bytes.len;
}
```

In `crypto_ops.zig`, `C_EncryptUpdate` e `C_DecryptUpdate` per il ramo GCM accodano e riportano zero byte scritti. La vera chiamata GCM avviene in `C_*Final`, dove per il decrypt il tag viene verificato prima che venga restituito qualsiasi plaintext. Lo smoke harness fa passare un messaggio multi-blocco attraverso `C_DecryptUpdate` in pezzi da 19 byte e verifica che ogni update emetta zero, poi ottiene il plaintext solo a `C_DecryptFinal`.

## Il ponte RSA verso libcrypto

`rsa.zig` è stateless. Non tiene mai un `EVP_PKEY` tra una chiamata e l'altra; ne ricostruisce uno ogni volta dai componenti nativi PKCS#11. `generate` estrae i componenti da una chiave appena generata:

```zig
pub fn generate(bits: u32) Error!Generated {
    const ctx = ossl.EVP_PKEY_CTX_new_id(ossl.pkey_rsa, null) orelse return Error.Crypto;
    defer ossl.EVP_PKEY_CTX_free(ctx);
    if (ossl.EVP_PKEY_keygen_init(ctx) <= 0) return Error.Crypto;
    if (ossl.EVP_PKEY_CTX_set_rsa_keygen_bits(ctx, @intCast(bits)) <= 0) return Error.Crypto;
    var pkey: ?*ossl.EVP_PKEY = null;
    if (ossl.EVP_PKEY_generate(ctx, &pkey) <= 0) return Error.Crypto;
    defer ossl.EVP_PKEY_free(pkey);
    // extract n, e, d, p, q, dmp1, dmq1, iqmp into a Generated struct
}
```

`sign` ricostruisce la chiave dai componenti memorizzati, poi esegue hash-then-sign (per `CKM_SHA256_RSA_PKCS` e le varianti PSS) oppure firma direttamente un valore già sottoposto a hash (`CKM_RSA_PKCS` raw). I componenti vengono passati freschi a ogni chiamata; lo stato dell'operazione nella sessione conteneva solo l'handle.

I binding in `openssl.zig` sono dichiarazioni `extern fn` scritte a mano, non `@cImport`. È questo che mantiene pulito il `.so` di produzione: non c'è alcun header tradotto che tiri dentro simboli, e lo script di versione non esporta nient'altro che `C_GetFunctionList`. Un dettaglio che vale la pena vedere è l'uso di `BN_clear_free` invece di `BN_free` per i BIGNUM privati:

```zig
pub extern fn BN_clear_free(a: ?*BIGNUM) void;  // zeroes the BIGNUM before freeing
```

`BN_free` non azzera la memoria; `BN_clear_free` sì. I componenti privati (d, p, q, i valori CRT) vengono liberati con `BN_clear_free` così non restano nell'heap liberato.

## Pattern di gestione dei segreti

La stessa manciata di pattern compare ovunque viva un segreto.

**Segreti sullo stack: defer secureZero.** Un hash derivato, una master key sullo stack, un buffer decifrato:

```zig
var hash: pin.Hash = undefined;
defer std.crypto.secureZero(u8, &hash);
```

`secureZero` prende una slice `[]volatile`, il che vieta al compilatore di eliminare la scrittura come dead store. Un semplice `@memset(&hash, 0)` senza letture successive può essere eliminato dall'ottimizzazione; `secureZero` no.

**Segreti sull'heap: secureFree.** Ogni valore di attributo viene liberato tramite un'unica funzione che prima pulisce:

```zig
fn secureFree(allocator: std.mem.Allocator, value: []u8) void {
    std.crypto.secureZero(u8, value);
    allocator.free(value);
}
```

**Stato delle operazioni: azzeramento al teardown.** Le union delle operazioni di sessione si azzerano da sole quando un'operazione termina. `endSign`, `endEncrypt` e simili chiamano lo `zeroize`/`deinit` della union (che esegue `secureZero` sull'intera union e libera qualsiasi memoria heap), poi impostano il campo a null. La `Table` delle sessioni pulisce gli slot all'apertura, alla chiusura e con un `wipeAll` completo a `C_Finalize`.

**Logout: ribloccare e cancellare.** `inst.relock()` risigilla ogni valore sensitive e cancella la master key:

```zig
pub fn relock(self: *Instance) void {
    if (self.mk) |*mk| object_store.lock(self.io(), self.allocator(), &self.objects, mk.*) catch {
        object_store.scrubUnsealed(&self.objects);  // if re-seal fails, scrub in place: fail closed
    };
    self.wipeMasterKey();
}
```

Se la risigillatura fallisce (per esempio, per out of memory), `scrubUnsealed` azzera sul posto i valori in chiaro e li marca come sigillati, così una pulizia fallita non lascia mai un segreto in chiaro. Questo è il principio fail-closed tradotto in codice.

## Il cross-check della ABI

`tests/abi_test.zig` è ciò che trasforma "la ABI corrisponde alla specifica" da una speranza in un invariante a build time. `build.zig` esegue `addTranslateC` su uno shim che include gli header OASIS in vendor, producendo un modulo `p11c`. Il test poi confronta il `ck.zig` scritto a mano con esso.

Uguaglianza del layout, campo per campo:

```zig
test "hand-coded structs match OASIS-translated layout byte-for-byte" {
    try expectSameLayout(ck.CK_ATTRIBUTE, p11c.CK_ATTRIBUTE);
    try expectSameLayout(ck.CK_FUNCTION_LIST, p11c.CK_FUNCTION_LIST);
    // ... every struct ...
}
```

Uguaglianza delle costanti, per ogni costante che esiste in entrambi:

```zig
test "every hand-coded constant equals its OASIS value" {
    inline for (@typeInfo(ck).@"struct".decls) |d| {
        if (@hasDecl(p11c, d.name)) {
            // assert ck.<name> == p11c.<name>, error.ConstantMismatch if not
        }
    }
    try std.testing.expect(checked >= 100);  // and we checked at least 100 of them
}
```

E le signature C-ABI di ogni funzione: ogni voce di `CK_FUNCTION_LIST` viene confrontata con quella tradotta per numero di parametri e dimensioni/allineamenti dei parametri. Aggiungi a `ck.zig` una costante che esiste anche negli header OASIS e questo loop la rileva automaticamente; sbagli un offset e la build fallisce indicando il campo esatto che diverge.

## Lo smoke harness

`examples/smoke.zig` non è uno unit test. Fa `dlopen` del `.so` *compilato* e chiama attraverso la function list esattamente come un host esterno, che è l'unico modo per intercettare bug di export e bug di forma della ABI che i test in-process non vedono:

```zig
var lib = try std.DynLib.open(default_module);
const getFunctionList = lib.lookup(GetFunctionList, "C_GetFunctionList") orelse return error.SymbolNotFound;
var list_ptr: ?*ck.CK_FUNCTION_LIST = null;
try check("C_GetFunctionList", getFunctionList(&list_ptr));
const f = list_ptr orelse return error.NullFunctionList;
```

Da lì percorre un ciclo di vita completo: inizializza il token, fa login come SO, inizializza il PIN dello User, imposta il PIN, crea e trova oggetti, dimostra che un oggetto privato è nascosto dopo il logout, genera chiavi AES/EC/RSA, firma e verifica (e manomette), cifra e decifra, deriva un segreto ECDH da entrambi i lati, esegue wrap e unwrap, esegue GCM in streaming e verifica che sia uguale al one-shot, mette alla prova le funzioni duali, esegue il recover con RSA, fa il round trip dello stato delle operazioni, e infine verifica ogni caso limite di conformità (`C_WaitForSlotEvent`, `C_GetFunctionStatus`, `C_SeedRandom`). Alla fine stampa un blocco di riepilogo così puoi vedere a colpo d'occhio cosa è passato.

## Errori di implementazione comuni

### Errore: un semplice `zig build` non corrisponde al modulo distribuito

**Sintomo:** ispezioni un segreto azzerato in un debugger e vedi byte `0xAA`, non zeri, e concludi che l'azzeramento è rotto.

**Causa:** `zig build` senza flag è una build Debug. In Debug e ReleaseSafe, Zig avvelena il payload di un optional a `0xAA` quando lo imposti a null, e `Allocator.free` esegue memset a `0xAA` sulla memoria liberata. Quindi `secureZero` seguito da null lascia `0xAA` (il segreto è sparito, solo che non è zero).

**Correzione:** testa che "il pattern del segreto sia sparito", non che "tutti i byte siano zero". I test di sessione verificano che il byte segreto sia assente, non che il buffer sia zero. L'artifact distribuito è `zig build --release=safe`, dove è lo `secureZero` esplicito a proteggerti (nelle build release senza il safety check non c'è poisoning, quindi lo zero esplicito conta).

### Errore: confrontare segreti con `std.mem.eql`

**Sintomo:** un controllo di PIN o MAC che funziona a livello funzionale ma fa trapelare informazioni sul timing.

**Causa:** `std.mem.eql` ritorna al primo byte non corrispondente.

**Correzione:** `std.crypto.timing_safe.eql` per gli array a dimensione fissa (l'hash del PIN), o l'helper `ctEql` in `crypto_ops.zig` per i MAC a lunghezza variabile. Entrambi guardano ogni byte prima di decidere.

### Errore: lasciare null uno slot della function list

**Sintomo:** l'host va in segfault nel momento in cui chiama una funzione non supportata.

**Causa:** un puntatore null nella tabella.

**Correzione:** ogni slot punta a una vera funzione `callconv(.c)`. Le operazioni non supportate restituiscono un codice `CKR_*`; non sono mai null. L'assert di dimensione `comptime` in `main.zig` e il test ABI proteggono la forma.

## Suggerimenti per il debug

### Osserva ogni chiamata con pkcs11-spy

Quando una chiamata dell'host si comporta male, il modo più rapido per vedere cosa ha effettivamente inviato è `pkcs11-spy`, che si mette davanti al tuo modulo e registra nel log ogni chiamata:

```bash
export PKCS11SPY=$PWD/zig-out/lib/libhsm.so
export PKCS11SPY_OUTPUT=/tmp/spy.log
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/pkcs11/pkcs11-spy.so -T
```

La recipe `just spy` incapsula questo. Il log mostra ogni chiamata `C_*`, i suoi argomenti e il suo codice di ritorno, così puoi trovare la chiamata esatta che ha restituito la cosa sbagliata e con quali input.

### Leggi il codice di ritorno, poi leggi la specifica

I codici di ritorno PKCS#11 sono specifici. `CKR_OPERATION_NOT_INITIALIZED` significa che è arrivato un `C_Sign` senza un `C_SignInit`. `CKR_OPERATION_ACTIVE` significa un secondo `C_SignInit` mentre uno era attivo. `CKR_ATTRIBUTE_SENSITIVE` significa che hai chiesto un valore sigillato. Quando un host fallisce, il codice che ha ricevuto di solito dà il nome al bug.

### Conferma la superficie di export

Se un host non riesce proprio a caricare il modulo, controlla che l'unico simbolo sia presente e che nient'altro sia trapelato:

```bash
objdump -T zig-out/lib/libhsm.so | grep ' g '   # should show only C_GetFunctionList
```

## Principi di organizzazione del codice

### Perché un file per gruppo di funzioni

La suddivisione `api/*` rispecchia il raggruppamento della specifica stessa (slot/token, sessione, oggetti, crittografia, gestione delle chiavi). Chi legge e cerca "come funziona il login" apre `api/session.zig`; "come funziona la firma" apre `api/crypto_ops.zig`. Ogni file importa i livelli core e crypto di cui ha bisogno e niente di cui non ha bisogno.

### Perché il livello crittografico non sceglie mai un codice di ritorno

Le funzioni crittografiche restituiscono errori Zig (`error.Crypto`, `cipher.Error.DataLenRange`). Il livello ABI li mappa su `CKR_*`. Questo mantiene il livello crittografico testabile in modo indipendente (un test verifica `error.AuthFailed`, non `CKR_*`) e mantiene la policy dei codici di ritorno in un solo livello. C'è un solo `mapCipherErr`, un solo `mapSetErr`.

## Estendere il codice

### Aggiungere un meccanismo, in concreto

Diciamo che vuoi aggiungere HMAC-SHA-512 (c'è già, ma i passi sono gli stessi per qualsiasi meccanismo nuovo):

1. **Costante.** Aggiungi `CKM_FOO` a `ck.zig`. Se gli header OASIS la definiscono, il test ABI ne verifica il valore gratuitamente.
2. **Pubblicizzalo.** Aggiungi `CKM_FOO` a `config.supported_mechanisms` e dagli un ramo in `C_GetMechanismInfo` di `slot_token.zig` con i limiti di dimensione della chiave e i flag corretti.
3. **Implementa.** Aggiungi la primitiva nel file `crypto/*` pertinente. Scrivi uno unit test contro un vettore pubblicato e aggiungi il file a `test_all.zig`.
4. **Smistamento.** Collegalo all'helper classificatore a cui appartiene (`mac.macLenOf`, `cipher.modeOf`, `ecdsa.hashModeOf`, o i controlli RSA `isRsa*Mech`) così che le funzioni `*Init` lo instradino verso di esso.

### Aggiungere un entry point

Copia lo scheletro acquire-and-defer da una qualsiasi funzione `api/*`. Valida prima l'handle di sessione, poi gli argomenti, poi esegui il lavoro, poi imposta le lunghezze in/out e restituisci il codice preciso. Se l'operazione è lenta (una KDF, una keygen che non dovrebbe tenere il lock), usa il pattern snapshot-unlock-recheck di `C_Login`.

## Stile del codice

Il progetto segue le convenzioni del repository: un commento di intestazione del file e nessun commento inline altrove, ogni costante in `config.zig` (niente numeri magici nella logica), e codice pulito secondo `zig fmt`. Esegui insieme il controllo del formatter e la suite completa:

```bash
just ci    # zig fmt --check + zig build test + zig build smoke
```

## Build e test

```bash
zig build --release=safe   # the shipped artifact
zig build test             # ABI cross-check + the crypto/core unit suite
zig build smoke            # dlopen the built .so and run the full lifecycle
```

`zig build test` esegue due binari di test: `tests/abi_test.zig` (il cross-check OASIS) e `src/test_all.zig` (che include ogni file di test `crypto/*` e `core/*`). I test crittografici girano contro vettori pubblicati, quindi una regressione nella matematica viene intercettata a livello di unità, e il test ABI intercetta qualsiasi deriva del layout nello stesso passo.

## Prossimi passi

1. Leggi [MECHANICS.md](./MECHANICS.it.md) per il dettaglio crittografico dietro lo smistamento che hai appena visto: come si concatena AES-CBC, come viene verificato il padding PKCS#7, come funziona il wrapping RFC 3394, come calcolano ECDSA ed ECDH, e in cosa differiscono gli schemi di RSA.
2. Leggi [CONFORMANCE.md](./CONFORMANCE.it.md) per il codice di ritorno esatto a ogni confine deliberato.
3. Apri `examples/smoke.zig` e segui un'operazione dalla chiamata `f.C_...` giù attraverso i livelli che ora conosci.
