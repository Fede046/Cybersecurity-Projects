<!-- ©AngelaMos | 2026 -->
<!-- MECHANICS.md -->

# Come funzionano davvero i meccanismi

La [dichiarazione di conformità](./CONFORMANCE.it.md) ti dice *cosa* fa ogni meccanismo e quale codice di ritorno fornisce. Questo documento ti dice *come* funziona ciascuno sotto il cofano, byte per byte, e perché l'implementazione ha l'aspetto che ha. Se [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) è il giro dell'impianto idraulico, questo è il giro dell'acqua.

Tutto qui, tranne RSA, è `std.crypto` in puro Zig, testato contro vettori pubblicati. RSA è libcrypto. Ogni sezione nomina il file che puoi aprire per leggere il codice vero.

## La versione breve

| Famiglia | Meccanismi | Dove sta la matematica | Basato su |
|--------|-----------|-------------------|-----------|
| Digest | SHA-256 / 384 / 512 | `digest.zig` | `std.crypto.hash.sha2` |
| MAC | HMAC-SHA-256 / 384 / 512 | `mac.zig` | `std.crypto.auth.hmac` |
| AES | CBC, CBC-PAD, GCM | `cipher.zig` | `std.crypto.core.aes`, `aead.aes_gcm` |
| Key wrap | AES-KEY-WRAP (RFC 3394) | `cipher.zig` | il blocco AES sopra |
| ECDSA | P-256, P-384, raw e SHA-256 | `ecdsa.zig` | `std.crypto.sign.ecdsa` |
| ECDH | P-256, P-384, `CKD_NULL` | `ecdsa.zig` | `std.crypto.ecc` |
| RSA | PKCS#1 v1.5, PSS, OAEP | `rsa.zig` + `openssl.zig` | OpenSSL `libcrypto` |
| KDF | Argon2id (stiramento del PIN) | `pin.zig` | `std.crypto.pwhash.argon2` |
| Envelope | seal/wrap AES-256-GCM | `keystore.zig` | `std.crypto.aead.aes_gcm` |

Un tema attraversa tutti questi meccanismi: la parte pericolosa di ogni primitiva è gestita da una funzione testata della standard library, e il codice proprio del modulo è il cablaggio, la validazione e l'igiene di tempo costante e di memoria attorno ad essa.

## AES-CBC: concatenare i blocchi

CBC (Cipher Block Chaining) trasforma un block cipher, che cifra un solo blocco da 16 byte, in qualcosa che cifra un intero messaggio. Il trucco è che ogni blocco di plaintext viene messo in XOR con il blocco di ciphertext precedente prima della cifratura, così blocchi di plaintext identici non producono ciphertext identico.

```
   P0        P1        P2
   │         │         │
IV─XOR    ┌─►XOR    ┌─►XOR        (IV seeds the very first XOR)
   │      │  │      │  │
  AES     │ AES     │ AES
   │      │  │      │  │
   ▼      │  ▼      │  ▼
   C0─────┘  C1─────┘  C2
```

In `cipher.zig`, un passo di encrypt è esattamente quell'XOR seguito dalla cifratura, con il valore di catena aggiornato al nuovo ciphertext:

```zig
fn cbcEncStep(self: *Cipher, in16: *const [block]u8, out16: *[block]u8) void {
    var x: [block]u8 = undefined;
    defer std.crypto.secureZero(u8, &x);     // the XOR scratch held plaintext, scrub it
    for (0..block) |j| x[j] = in16[j] ^ self.chain[j];
    encBlockRaw(self.key(), &x, out16);
    self.chain = out16.*;                      // chain forward
}
```

La decifratura fa l'inverso: decifra il blocco, poi lo mette in XOR con il ciphertext precedente (tenuto in `chain`). La catena avanza al blocco di input, non a quello di output.

Il semplice `CKM_AES_CBC` non ha padding, quindi l'input deve essere un numero intero di blocchi. Un blocco finale parziale è un errore (`CKR_DATA_LEN_RANGE`), perché non c'è nulla da fare con 5 byte avanzati. L'IV arriva nel parametro del meccanismo e inizializza `chain`.

## AES-CBC-PAD: il padding e il blocco trattenuto

`CKM_AES_CBC_PAD` aggiunge il padding PKCS#7 così che qualsiasi lunghezza possa essere cifrata. PKCS#7 è semplice: se ti servono `n` byte di padding, accodi `n` copie del byte `n`. Se il messaggio è già allineato al blocco, aggiungi un intero blocco di `0x10` (16) così che ci sia sempre del padding da rimuovere.

```
   message ...... |  pad
   "ABCDE"  (5)   |  0B 0B 0B 0B 0B 0B 0B 0B 0B 0B 0B   (11 bytes of 0x0B)
                     └──────────── 11 = pad length ────────────┘
```

`encryptFinal` scrive il padding nell'ultimo blocco parziale e lo cifra:

```zig
const padlen: u8 = @intCast(block - self.partial_len);
for (self.partial_len..block) |j| self.partial[j] = padlen;
self.cbcEncStep(&self.partial, out[0..block]);
```

È nella decifratura che le cose si fanno sottili. Il decifratore non può rimuovere il padding finché non sa di avere l'*ultimo* blocco, ma in modalità streaming non sa quale blocco sia l'ultimo finché lo stream non termina. Quindi il decrypt CBC-PAD **trattiene un blocco**: ogni volta che arriva un blocco completo, emette il blocco trattenuto *precedente* e trattiene quello nuovo. `decryptFinal` decifra il blocco finale trattenuto e rimuove il padding. Questo comportamento di trattenimento è esattamente il motivo per cui CBC-PAD non può partecipare al lato decrypt di una funzione duale (il digest perderebbe l'ultimo blocco); vedi la sezione 2.2 di [CONFORMANCE.md](./CONFORMANCE.it.md).

Il controllo del padding è a tempo costante. Un controllo ingenuo ritorna non appena un byte di pad è sbagliato, il che fa trapelare dove è fallito (l'attacco padding-oracle di Vaudenay, e Lucky Thirteen contro TLS). Questo modulo fa prima l'OR di tutte le differenze, poi decide una sola volta:

```zig
const padlen = pt[block - 1];
if (padlen == 0 or padlen > block) return Error.EncryptedDataInvalid;
var bad: u8 = 0;
for (0..block) |j| {
    const is_pad = j >= block - padlen;
    if (is_pad) bad |= pt[j] ^ padlen;     // accumulate, do not branch out early
}
if (bad != 0) return Error.EncryptedDataInvalid;
```

## AES-GCM: autenticato, e perché qui è bufferizzato

GCM (Galois/Counter Mode) fa due cose contemporaneamente: cifra con AES in modalità counter, e calcola un tag di autenticazione sul ciphertext e sugli associated data usando la moltiplicazione in un campo di Galois. Il tag è l'intero valore di GCM: la decifratura lo verifica, e un tag sbagliato significa che il ciphertext è stato manomesso (o che hai usato la chiave o il nonce sbagliati).

```
   nonce(12) + counter ──► AES ──► keystream ──XOR──► ciphertext
                                                          │
   AAD + ciphertext ──► GHASH (GF(2^128) mult) ──► tag(16)
```

Questo modulo usa il GCM one-shot della standard library, con un nonce da 12 byte (96 bit) e un tag da 128 bit, e fino a 256 byte di associated data:

```zig
aesgcm.Aes256Gcm.encrypt(out[0..input.len], tag, input, ad, self.iv, self.key_buf[0..32].*);
```

Il nonce a 96 bit è la scelta standard: viene usato direttamente come contatore iniziale senza l'hashing aggiuntivo che una lunghezza diversa richiederebbe. Il tag fisso a 128 bit è il tag a piena robustezza; i tag troncati indeboliscono l'autenticazione.

**Perché bufferizzato.** Un decrypt GCM in streaming che rilascia il plaintext blocco per blocco restituirebbe byte non verificati prima di aver visto il tag. Questo è il release of unverified plaintext, ed è un anti-pattern per un modulo di sicurezza (vedi [01-CONCEPTS.md](./01-CONCEPTS.it.md)). Questo modulo accumula l'intero messaggio in `GcmStream` ed esegue l'operazione autenticata una sola volta a `*Final`, dove il decrypt verifica il tag prima di restituire un singolo byte. Il buffer è limitato a 16 MiB così che un host non possa esaurire la memoria. Il costo è che GCM è limitato a un singolo messaggio; il beneficio è che il RUP è impossibile per costruzione.

Con un tag sbagliato, la decifratura restituisce `error.EncryptedDataInvalid` e nessun plaintext. La standard library azzera internamente il buffer di output prima di restituire l'errore, quindi una decifratura fallita non lascia dietro di sé nulla di utilizzabile.

## AES Key Wrap (RFC 3394): cifrare una chiave con una chiave

Non puoi semplicemente cifrare una chiave con AES e chiamarla wrapped; vuoi che il risultato renda evidente la manomissione, così che una chiave avvolta corrotta venga *rifiutata* invece di essere svolta in spazzatura. AES Key Wrap della RFC 3394 lo fa con un valore iniziale fisso (l'integrity check value, ICV) e sei passate sui dati.

L'ICV è la costante `A6 A6 A6 A6 A6 A6 A6 A6`. Il wrapping la antepone, poi esegue 6 round in cui ogni blocco a 64 bit della chiave viene mescolato con un registro `A` corrente attraverso il block cipher AES e un contatore:

```zig
var a: [8]u8 = key_wrap_iv;                 // A6 A6 ... A6
// for j in 0..6, for each 64-bit block i:
@memcpy(blk[0..8], &a);
@memcpy(blk[8..16], r[i * 8 ..][0..8]);
encBlockRaw(kek, &blk, &enc);                // B = AES(A | R[i])
@memcpy(&a, enc[0..8]);                       // A = MSB64(B)
xorCounter(&a, n * j + i + 1);                // A ^= t (the round counter)
@memcpy(r[i * 8 ..][0..8], enc[8..16]);       // R[i] = LSB64(B)
```

L'unwrapping esegue i round all'indietro e poi controlla che l'`A` recuperato sia uguale all'ICV. Se anche un solo bit della chiave avvolta è stato invertito, `A` non corrisponderà a `A6...A6`, e l'unwrap fallisce. Il controllo viene fatto facendo l'OR delle differenze (non con un confronto che esce in anticipo), e in caso di fallimento l'output parziale viene pulito:

```zig
var diff: u8 = 0;
for (a, key_wrap_iv) |x, y| diff |= x ^ y;
if (diff != 0) {
    std.crypto.secureZero(u8, out[0..plain_len]);
    return WrapError.Integrity;             // -> CKR_WRAPPED_KEY_INVALID at the API
}
```

L'implementazione è verificata contro il known-answer test della sezione 4.1 della RFC 3394: avvolgere la chiave di test sotto la KEK di test produce esattamente il ciphertext pubblicato, e lo svolgimento fa il round trip. L'ICV da 8 byte è il motivo per cui una chiave avvolta è 8 byte più lunga della chiave che avvolge.

## SHA-2 e HMAC: hashing e hashing con chiave

SHA-256, SHA-384 e SHA-512 sono i meccanismi di digest. L'`Hasher` in `digest.zig` è una tagged union sui tre, e l'hashing multi-part (`C_DigestUpdate` ripetuto) alimenta semplicemente lo stato corrente:

```zig
pub const Hasher = union(enum) {
    sha256: sha2.Sha256,
    sha384: sha2.Sha384,
    sha512: sha2.Sha512,
};
```

Un digest è verificato contro il classico vettore `SHA-256("abc")` negli unit test e di nuovo nello smoke harness attraverso il vero `.so`.

La parte interessante è la **serializzazione dello stato delle operazioni**. `C_GetOperationState` permette a un host di fare uno snapshot di un digest a metà stream e di riprenderlo più tardi con `C_SetOperationState`. Il modulo serializza lo stato grezzo dell'hasher con un byte di versione e un tag di tipo:

```
   [version=1][tag: 1=sha256 2=sha384 3=sha512][raw hasher state bytes]
```

```zig
out[0] = config.op_state_version;
out[1] = d.stateTag();
d.writeState(out[config.op_state_header_len..]);
```

Questo blob è **valido solo per la stessa build**: porta con sé il layout in memoria dell'hasher della standard library, che non è garantito stabile tra build o implementazioni. La specifica consente esattamente questo (lo stato delle operazioni non è portabile), e il ripristino valida la versione, il tag e la lunghezza esatta, fallendo in modo chiuso su qualsiasi altra cosa. Solo lo stato del digest è salvabile; provare a salvare un'operazione di sign o encrypt restituisce `CKR_STATE_UNSAVEABLE`.

HMAC (`mac.zig`) è la costruzione di hash con chiave `H(key ⊕ opad || H(key ⊕ ipad || message))`, ma non la implementi a mano; lo fanno `HmacSha256` e simili della standard library. Il compito del modulo è verificare il tag a tempo costante, cosa che `crypto_ops.zig` fa con `ctEql`. HMAC-SHA-256 è verificato contro il test case 2 della RFC 4231.

## ECDSA: firmare su una curva

ECDSA firma su una curva ellittica. Questo modulo supporta NIST P-256 e P-384, con due meccanismi: `CKM_ECDSA` (fornisci un valore già sottoposto a hash) e `CKM_ECDSA_SHA256` (il modulo prima calcola l'hash del messaggio con SHA-256).

### La curva e la codifica della chiave

La curva è identificata da un OID in DER, memorizzato in `CKA_EC_PARAMS`. P-256 è `06 08 2A 86 48 CE 3D 03 01 07`, P-384 è `06 05 2B 81 04 00 22`. `curveFromParams` confronta i byte:

```zig
pub fn curveFromParams(ec_params: []const u8) ?Curve {
    if (std.mem.eql(u8, ec_params, &oid_p256)) return .p256;
    if (std.mem.eql(u8, ec_params, &oid_p384)) return .p384;
    return null;
}
```

La chiave pubblica è un punto in forma SEC1 non compressa: un byte `0x04` seguito dalle coordinate X e Y. Per P-256 sono `1 + 32 + 32 = 65` byte; l'attributo `CKA_EC_POINT` lo avvolge in un `OCTET STRING` DER, quindi il valore memorizzato è di 67 byte (l'header DER `0x04 0x41` più il punto da 65 byte). `wrapEcPoint` e `unwrapEcPoint` gestiscono quell'incapsulamento, inclusa la codifica della lunghezza DER in forma lunga per il punto più grande di P-384.

### Prehash, riduzione, firma

Il messaggio (o il suo hash) viene ridotto a uno scalare prendendo i byte più a sinistra pari alla dimensione dell'ordine della curva, che è ciò che ECDSA fa con un hash che può avere una larghezza diversa da quella della curva:

```zig
fn reduce(curve: Curve, dgst: []const u8, out: *[max_scalar]u8) []const u8 {
    const n = curve.scalarLen();
    if (dgst.len >= n) @memcpy(out[0..n], dgst[0..n])     // take the leftmost n bytes
    else // left-pad with zeros
}
```

La firma vera e propria usa `signPrehashed`. Una sottigliezza che vale la pena conoscere: l'ECDSA deterministico di Zig non è il semplice RFC 6979. Mescola un campo di rumore, quindi la firma non è una funzione fissa del messaggio e della chiave. Questo significa che non puoi fissare un known-answer test sull'output della firma (due firme dello stesso messaggio differiscono), quindi la test suite usa invece un vettore RFC 6979 come known-answer test di *verifica*, e fa il round trip delle proprie firme. Il rumore viene da `randomSecure`, con un fallback al deterministico se l'entropia non è disponibile:

```zig
const nz: ?[slen]u8 = if (io.randomSecure(&noise)) |_| noise else |_| null;
const sig = kp.signPrehashed(ph, nz) catch return Error.Crypto;
```

Una firma P-256 è di 64 byte (r e s, 32 ciascuno); P-384 è di 96. La verifica ricostruisce la chiave pubblica dal punto SEC1, valida che il punto sia sulla curva e controlla la firma. Lo smoke harness firma tramite il modulo, verifica, poi inverte un byte e conferma `CKR_SIGNATURE_INVALID`.

## ECDH: accordarsi su un segreto condiviso

ECDH (`CKM_ECDH1_DERIVE`) permette a due parti con coppie di chiavi EC di calcolare lo stesso segreto condiviso senza mai trasmetterlo. Ciascuna moltiplica il proprio scalare privato per il punto pubblico dell'altra; la matematica garantisce che entrambe arrivino allo stesso punto, e il segreto è la coordinata X di quel punto.

```
   Alice:  secret = X-coordinate of (alice_private * bob_public)
   Bob:    secret = X-coordinate of (bob_private * alice_public)
   alice_private * bob_public == bob_private * alice_public   (same point)
```

L'implementazione moltiplica il punto del peer per lo scalare e prende la coordinata X affine:

```zig
const peer = Pt.fromSec1(peer_point_sec1) catch return Error.Crypto;  // validates on-curve
const shared = peer.mul(s, .big) catch return Error.Crypto;
const xb = shared.affineCoordinates().x.toBytes(.big);
@memcpy(out[0..n], xb[0..n]);
```

`fromSec1` rifiuta un punto che non è sulla curva, che è la difesa contro gli invalid-curve attack (fornire un punto fuori curva scelto con cura per far trapelare lo scalare privato). Il modulo supporta solo la funzione di derivazione delle chiavi `CKD_NULL`, il che significa che la coordinata X condivisa grezza è la chiave derivata senza ulteriore KDF, e accetta il punto del peer sia come SEC1 raw sia avvolto in DER. Il valore derivato è corretto al bit: lo smoke harness deriva da entrambi i lati e verifica che corrispondano, e un test cross-process conferma che è uguale a `openssl pkeyutl -derive` byte per byte. Lo smoke harness arriva persino a derivare un lato con un punto del peer raw e l'altro con uno avvolto in DER, e conferma che entrambi concordano.

## RSA: tre schemi, una esponenziazione modulare

RSA è l'unica famiglia che questo modulo non implementa da sé. La standard library di Zig non ha un RSA pubblico, quindi `rsa.zig` chiama la `libcrypto` di OpenSSL attraverso le dichiarazioni `extern` scritte a mano in `openssl.zig`. Ogni operazione RSA è una singola esponenziazione modulare sull'intero input, quindi RSA non ha una forma multi-part (niente `C_SignUpdate`); usi le chiamate one-shot.

Le chiavi sono memorizzate in forma nativa PKCS#11: il modulo e l'esponente pubblico sono pubblici, e il lato privato conserva l'esponente privato più i quattro valori CRT (Chinese Remainder Theorem) che rendono le operazioni private circa quattro volte più veloci. L'esponente pubblico è fisso a 65537 (il primo F4), il default universale. `rsa.zig` è stateless: ricostruisce un `EVP_PKEY` da questi componenti a ogni chiamata invece di conservarne uno.

I tre schemi differiscono nel modo in cui applicano il padding prima dell'esponenziazione:

**PKCS#1 v1.5** (`CKM_RSA_PKCS`, `CKM_SHA256_RSA_PKCS`). Il padding classico. Per la firma, il messaggio (o il suo hash, avvolto in un DigestInfo) viene riempito con `00 01 FF FF ... FF 00` fino alla dimensione del modulo. Semplice e onnipresente, ma la sua forma di decifratura è quella vulnerabile all'oracolo di Bleichenbacher, ed è per questo che qui i fallimenti di decifratura collassano in un unico codice di ritorno uniforme.

**PSS** (`CKM_RSA_PKCS_PSS`, `CKM_SHA256_RSA_PKCS_PSS`). Probabilistic Signature Scheme. Mescola un salt casuale nel padding, quindi firmare due volte lo stesso messaggio dà firme diverse, e ha una prova di sicurezza che a PKCS#1 v1.5 manca. Il modulo richiede che l'hash MGF sia uguale all'hash del contenuto e rifiuta le discrepanze:

```zig
if (mgfHash(pp.mgf) != h) return .{ .err = ck.CKR_MECHANISM_PARAM_INVALID };
```

**OAEP** (`CKM_RSA_PKCS_OAEP`). Optimal Asymmetric Encryption Padding, per la cifratura e il key wrapping. È la controparte di PSS per la cifratura: randomizzato, con una prova, e il sostituto raccomandato della cifratura PKCS#1 v1.5 proprio perché resiste a Bleichenbacher.

La distinzione tra hash-then-sign e raw è visibile in `sign`. Quando è specificato un digest, usa `EVP_DigestSign` (che calcola l'hash e poi firma); quando l'input è già un hash, usa direttamente `EVP_PKEY_sign`:

```zig
if (p.digest != .none) {
    // EVP_DigestSignInit + EVP_DigestSign: hash the message, then sign
} else {
    // EVP_PKEY_sign: sign the pre-hashed value as-is
}
```

Sign/VerifyRecover (`C_SignRecover` / `C_VerifyRecover`) usano la proprietà di PKCS#1 v1.5 per cui il messaggio è recuperabile dalla firma. Il modulo offre il recover solo per `CKM_RSA_PKCS`, e prima di firmare controlla che i dati entrino nel modulo con l'overhead di 11 byte della v1.5.

## Argon2id: trasformare un PIN debole in una chiave forte

Un PIN ha bassa entropia, quindi la forma memorizzata deve essere costosa da attaccare e la KEK derivata deve essere forte. Argon2id, il vincitore della Password Hashing Competition, è memory-hard: ogni tentativo deve allocare e attraversare un grande blocco di memoria, il che neutralizza il parallelismo massiccio che una GPU o un ASIC porta a un hash veloce.

I parametri in `config.zig` sono `t = 3` iterazioni, `m = 64 MiB`, `p = 1` lane. Ogni tentativo di PIN costa 64 MiB di memoria e tre passate su di essa. `pin.zig` deriva l'output da 32 byte dal PIN e da un salt casuale da 16 byte:

```zig
pub fn derive(io: std.Io, allocator: std.mem.Allocator, pin: []const u8, salt: *const Salt, out: *Hash) !void {
    try argon2.kdf(allocator, out, pin, salt, params, .argon2id, io);
}
```

La stessa derivazione svolge un doppio compito. Per l'autenticazione, l'output viene confrontato (a tempo costante) con l'hash memorizzato. Per l'envelope, l'output è la key-encryption key che avvolge la master key. La variante `id` fonde le modalità indipendente dai dati e dipendente dai dati, offrendo resistenza sia agli attacchi side channel sia a quelli time-memory-tradeoff.

## L'envelope: sigillare i valori e avvolgere la master key

Lo schema at rest ha due operazioni GCM, entrambe in `keystore.zig`.

**Avvolgere la master key.** La master key (32 byte casuali) è cifrata sotto la KEK derivata dal PIN. La forma avvolta è salt, nonce, ciphertext e tag, memorizzati nel record del token. Un PIN sbagliato deriva una KEK sbagliata, il tag GCM fallisce, e `unwrap` restituisce false invece di una chiave fasulla:

```zig
gcm.decrypt(out, &w.ct, w.tag, "", w.nonce, kek) catch {
    std.crypto.secureZero(u8, out);
    return false;
};
```

**Sigillare un valore.** Ogni valore di attributo sensitive è cifrato con GCM sotto la master key, con il *tipo* dell'attributo come associated data, e un nonce fresco per ogni sigillatura:

```zig
pub fn seal(io: std.Io, mk: *const MasterKey, ad: []const u8, plain: []const u8, out: []u8) !usize {
    var nonce: [nonce_len]u8 = undefined;
    try io.randomSecure(&nonce);            // fresh nonce every time
    @memcpy(out[0..nonce_len], &nonce);
    const ct = out[nonce_len..][0..plain.len];
    const tag = out[nonce_len + plain.len ..][0..tag_len];
    gcm.encrypt(ct, tag, plain, ad, nonce, mk.*);
    return sealedLen(plain.len);
}
```

Gli associated data legano il ciphertext al suo tipo di attributo. L'unseal di un valore il cui `ad` non corrisponde al tipo in cui viene caricato fa fallire il tag, quindi un `CKA_PRIVATE_EXPONENT` sigillato non può essere scambiato in uno slot `CKA_VALUE`. Il nonce fresco per ogni sigillatura è ciò che mantiene sicuro GCM; riusare un nonce sotto la stessa chiave è l'unico foot-gun catastrofico di GCM, e il design non lo fa mai.

Un valore sigillato è di `nonce_len + plaintext_len + tag_len` byte, cioè `12 + n + 16`. Il record su disco memorizza quella lunghezza così che l'unseal sappia dove finisce il ciphertext e dove inizia il tag.

## Tempo costante e azzeramento: i meccanismi trasversali

Non sono meccanismi che un host chiama, ma sono il macchinario che rende sicuri i meccanismi.

**Confronto a tempo costante.** Qualsiasi confronto di un segreto deve guardare ogni byte indipendentemente da dove si trovi la prima discrepanza, altrimenti il tempo che impiega fa trapelare la posizione. I segreti a dimensione fissa (l'hash del PIN) usano `std.crypto.timing_safe.eql`, che mette in XOR tutti gli elementi in un unico accumulatore e lo verifica senza branch. I MAC a lunghezza variabile usano `ctEql`, la stessa idea scritta a mano. I controlli CBC-PAD e RFC 3394 accumulano tutte le loro differenze prima di decidere, per la stessa ragione.

**Azzeramento.** Un segreto lasciato nella memoria liberata o inattiva può essere recuperato da un crash dump, da una pagina in swap o da una lettura cold boot. Il modulo azzera i segreti con `std.crypto.secureZero`, che prende una slice `[]volatile`:

```zig
pub fn secureZero(comptime T: type, s: []volatile T) void {
    @memset(s, 0);
}
```

Il `volatile` è tutto il punto. Un semplice `@memset(&key, 0)` senza letture successive è un dead store che l'ottimizzatore è libero di eliminare. Marcare la slice come volatile dice al compilatore che la scrittura è un effetto collaterale osservabile che deve avvenire. È la stessa tecnica che usa ogni seria libreria crittografica C, e Zig la incorpora in un'unica funzione verificabile. I segreti sullo stack la abbinano a `defer`; i segreti sull'heap passano per `secureFree`; lo stato delle operazioni di sessione si azzera da solo al teardown; il logout risigilla e cancella la master key. L'effetto complessivo è che un token inattivo e disconnesso non contiene materiale di chiavi in chiaro da nessuna parte.

Una peculiarità specifica di Zig: nelle build Debug e ReleaseSafe, impostare un optional a null avvelena il suo payload a `0xAA`, e `Allocator.free` esegue memset a `0xAA` sulla memoria liberata. Quindi dopo un `secureZero` seguito da null vedi `0xAA`, non zeri. Il segreto è distrutto (è `0xAA`, non la chiave), semplicemente non è zero. I test quindi verificano che "il pattern del segreto sia sparito", non che "tutti i byte siano zero". In una build release senza quei memset di sicurezza, è lo `secureZero` esplicito a fare il lavoro.

## Scegliere un meccanismo

Il più delle volte la scelta è dettata da ciò con cui devi interoperare, ma ecco le indicazioni che il design incorpora:

- **Cifratura simmetrica:** preferisci GCM (`CKM_AES_GCM`). Autentica, quindi la manomissione viene rilevata. Usa CBC-PAD solo quando un peer richiede CBC non autenticato, e in quel caso abbinalo a un MAC separato.
- **Cifratura RSA / trasporto di chiavi:** preferisci OAEP (`CKM_RSA_PKCS_OAEP`). La cifratura PKCS#1 v1.5 è il bersaglio di Bleichenbacher; OAEP è il sostituto moderno.
- **Firme RSA:** preferisci PSS (`CKM_RSA_PKCS_PSS`). Ha una prova di sicurezza che alla v1.5 manca. Usa la v1.5 (`CKM_SHA256_RSA_PKCS`) per compatibilità con i sistemi che se la aspettano.
- **Firme in generale:** ECDSA ti dà la stessa sicurezza di RSA con chiavi e firme molto più piccole (una firma P-256 è di 64 byte contro i 256 di RSA-2048). Preferiscilo quando entrambe le parti lo supportano.
- **Key wrapping:** AES-KEY-WRAP (RFC 3394) per avvolgere una chiave simmetrica sotto un'altra chiave simmetrica; RSA-OAEP per avvolgere sotto una chiave pubblica.
- **Key agreement:** ECDH (`CKM_ECDH1_DERIVE`) per derivare un segreto condiviso da due coppie di chiavi EC.

## Il dettaglio dietro il determinismo di ECDSA

Se sei curioso di sapere perché la test suite verifica un vettore esterno ma non può verificare il proprio output di firma:

Il semplice ECDSA RFC 6979 è completamente deterministico: il nonce è una funzione del messaggio e della chiave privata, quindi firmare due volte lo stesso messaggio produce la stessa firma, e puoi fissare un known-answer test sull'output. Il `signPrehashed` di Zig, invece, mescola un campo di rumore opzionale nella derivazione del nonce (un design di hedged signature che aggiunge una difesa contro i fault attack). Questo rende l'output non deterministico, quindi due firme dello stesso messaggio differiscono.

La conseguenza per i test è che verifichi contro RFC 6979 (una firma nota deve risultare valida sotto la chiave nota) ma non puoi verificare che la tua firma riproduca la firma RFC 6979 byte per byte. La suite fa entrambe le cose che *puoi* fare: usa il vettore RFC 6979 come known-answer test di verifica, e fa il round trip delle proprie firme (firma, poi verifica, poi manomette e conferma il rifiuto). Questo è il modo corretto di testare uno schema di hedged signature, e vale la pena capirlo così che il design dei test non sembri una lacuna.

## Prossimi passi

1. Leggi [CONFORMANCE.md](./CONFORMANCE.it.md) per le regole esatte sui parametri e i codici di ritorno a ogni confine di questi meccanismi.
2. Apri `src/crypto/cipher.zig` e leggi il test RFC 3394, poi `src/crypto/ecdsa.zig` e leggi il test ECDH su entrambi i lati. I test sono brevi e mostrano ogni meccanismo dall'inizio alla fine.
3. Prova le estensioni in [04-CHALLENGES.md](./04-CHALLENGES.it.md), molte delle quali aggiungono un meccanismo e ti guidano attraverso lo stesso cablaggio che hai appena visto per quelli esistenti.
