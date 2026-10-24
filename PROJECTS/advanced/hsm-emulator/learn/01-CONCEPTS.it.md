<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# Concetti di sicurezza fondamentali

Questo documento spiega le idee di sicurezza su cui è costruito il progetto. Non sono semplici definizioni. Ognuna è legata a un fallimento reale e al codice esatto di questo modulo che la difende.

## Custodia delle chiavi: l'applicazione non deve mai vedere la chiave

### Che cos'è

Un HSM esiste per far rispettare una regola: la chiave segreta entra una volta e non esce mai più in forma utilizzabile. L'applicazione che ha bisogno di una firma non riceve la chiave per calcolare la firma da sola. Consegna all'HSM una richiesta, e l'HSM calcola la firma dentro il proprio confine e restituisce solo il risultato. Il materiale della chiave non riattraversa mai la linea.

In PKCS#11 questa regola è espressa tramite due attributi di un oggetto chiave:

- `CKA_SENSITIVE = CK_TRUE` significa che il valore della chiave non può essere riletto tramite l'API. Chiedere `CKA_VALUE` restituisce `CKR_ATTRIBUTE_SENSITIVE`, non i byte.
- `CKA_EXTRACTABLE = CK_FALSE` significa che la chiave non può nemmeno essere esportata tramite wrapping (cifrata ed esportata).

### Perché è importante

Se la chiave è solo una variabile nel tuo processo, allora qualsiasi bug che legge la memoria del processo legge la chiave. Il bug **Heartbleed** del 2014 (CVE-2014-0160) era esattamente questo: un controllo dei limiti mancante in OpenSSL permetteva a un attaccante remoto di leggere fino a 64 KB di memoria del server per richiesta, e le chiavi private TLS erano tra le cose che la gente ne ha estratto. I server le cui chiavi vivevano in un HSM non erano esposti allo stesso modo. La chiave non si è mai trovata nello spazio di indirizzamento del web server da cui poter trapelare.

### Come funziona qui

Quando generi una chiave, il modulo marca di default il materiale segreto come sensitive e non estraibile, e calcola i due flag "always" che la specifica richiede. Da `keymgmt.zig`:

```zig
if (!obj.has(ck.CKA_SENSITIVE)) try obj.set(allocator, ck.CKA_SENSITIVE, &[_]u8{ck.CK_TRUE});
if (!obj.has(ck.CKA_EXTRACTABLE)) try obj.set(allocator, ck.CKA_EXTRACTABLE, &[_]u8{ck.CK_FALSE});
const always_sensitive: u8 = if (obj.getBool(ck.CKA_SENSITIVE)) ck.CK_TRUE else ck.CK_FALSE;
const never_extractable: u8 = if (!obj.getBool(ck.CKA_EXTRACTABLE)) ck.CK_TRUE else ck.CK_FALSE;
```

Quando poi un host prova a leggere quel valore, il controllo `sensitiveProtected` di `object.zig` scatta prima che venga copiato qualsiasi byte, e `C_GetAttributeValue` riporta la lunghezza del valore come `CK_UNAVAILABLE_INFORMATION` e restituisce `CKR_ATTRIBUTE_SENSITIVE`. La chiave è utilizzabile per firmare (passi il suo *handle* a `C_SignInit`) ma i suoi byte sono irraggiungibili.

Puoi osservarlo nello smoke harness: dopo aver generato una chiave AES, l'harness chiede `CKA_VALUE` e verifica che la risposta sia `CKR_ATTRIBUTE_SENSITIVE`, non la chiave.

### Attacchi comuni

1. **Divulgazione della memoria.** Un buffer over-read (Heartbleed), un leak di memoria non inizializzata, o un crash dump che viene spedito fuori dalla macchina. Se la chiave non è mai stata in chiaro nella tua memoria, il leak non produce nulla di utile.
2. **Esportazione della chiave tramite l'API.** Un attaccante che ottiene una sessione prova semplicemente a leggere la chiave. `CKA_SENSITIVE` rifiuta. Prova a esportarla con `C_WrapKey`. `CKA_EXTRACTABLE = CK_FALSE` rifiuta con `CKR_KEY_UNEXTRACTABLE`.
3. **Furto del disco a freddo.** Qualcuno copia il file del token. I valori sensitive al suo interno sono sigillati (vedi cifratura at rest, sotto), quindi il file è ciphertext.

### Strategie di difesa

L'intero modulo è la difesa, ma le mosse centrali sono: marcare di default i segreti come sensitive e non estraibili, far passare ogni lettura attraverso un unico punto di controllo (`sensitiveProtected`), e non costruire mai un percorso di codice che restituisca byte segreti grezzi al chiamante. Le operazioni prendono *handle*, non chiavi.

## Il modello di login: oggetti pubblici, oggetti privati e ruoli

### Che cos'è

Un token PKCS#11 ha due ruoli: il **Security Officer** (SO), che amministra il token e configura lo User, e lo **User**, che usa effettivamente le chiavi. Gli oggetti sono *pubblici* (visibili a chiunque abbia una sessione) oppure *privati* (`CKA_PRIVATE = CK_TRUE`, visibili solo dopo che lo User ha fatto login).

### Perché è importante

La separazione dei ruoli è ciò che ti permette di consegnare a qualcuno un token senza consegnargli le chiavi che contiene. Lo SO può inizializzare e resettare, ma in un design corretto lo SO non può leggere i segreti dello User. Se il ruolo di amministratore potesse leggere il materiale delle chiavi dello user, "amministratore" sarebbe solo un secondo nome per "attaccante che ha ottenuto l'admin".

### Come funziona qui

La visibilità è una sola funzione, `object_store.visible`:

```zig
pub fn visible(obj: *const Object, logged_in: ?ck.CK_USER_TYPE) bool {
    if (!obj.isPrivate()) return true;
    return logged_in == ck.CKU_USER;
}
```

Ogni entry point che riguarda gli oggetti (`C_FindObjects`, `C_GetAttributeValue`, `C_DestroyObject`, i recuperi delle chiavi dentro `crypto_ops.zig`) chiama prima questa. Un oggetto privato semplicemente non esiste per un chiamante che non ha fatto login come User. Lo smoke harness lo dimostra: crea un oggetto privato, fa logout, esegue `C_FindObjects` e verifica che l'oggetto privato non compaia e che recuperarlo restituisca `CKR_OBJECT_HANDLE_INVALID`. L'oggetto non è solo nascosto dagli elenchi; è irraggiungibile tramite handle.

Punto cruciale: questo modulo non dà allo SO **nessun keyslot per i segreti dello user**. La master key che protegge i valori sensitive è avvolta solo sotto il PIN dello User (vedi sotto). Uno SO che resetta il token può cancellarlo, ma non può leggere ciò che conteneva. È una scelta di design deliberata, documentata nella sezione 4 di [CONFORMANCE.md](./CONFORMANCE.it.md).

### Attacchi comuni

1. **Confusione dei privilegi.** Un attaccante con accesso SO prova a leggere le chiavi dello User. L'assenza di un keyslot SO significa che lo SO non ha nulla con cui decifrare.
2. **Enumerazione pre-login.** Un attaccante senza credenziali elenca gli oggetti sperando che le chiavi private trapelino nell'elenco. `visible` le tiene fuori.

## Autenticazione: stirare il PIN e chiudere la porta a chiave

### Che cos'è

User e SO si autenticano con un PIN. Un PIN è per natura corto e a bassa entropia (spesso da quattro a otto cifre), quindi devono essere vere due cose: la forma memorizzata deve essere costosa da attaccare offline, e i tentativi online devono essere limitati fino a un blocco netto.

### Perché è importante

Se memorizzi il PIN stesso, o un hash veloce del PIN, un attaccante che ruba il file del token esegue un attacco a dizionario in pochi secondi. La **violazione di LinkedIn** del 2012 ha fatto trapelare 6,5 milioni di hash di password SHA-1 senza salt; poiché SHA-1 è veloce e gli hash non avevano salt, la maggior parte è stata craccata quasi immediatamente. Un PIN protetto da un hash veloce non è messo meglio.

### Come funziona qui

Il PIN viene stirato con **Argon2id**, la KDF memory-hard che ha vinto la Password Hashing Competition. Da `config.zig` e `pin.zig`:

```zig
const params: argon2.Params = .{
    .t = config.pin_kdf_t,      // 3 iterations
    .m = config.pin_kdf_m_kib,  // 64 MiB of memory
    .p = config.pin_kdf_p,      // 1 lane
};
```

Ogni tentativo costa 64 MiB di memoria e tre passate su di essa, il che fa crollare il throughput di un rig di brute force. Su disco finiscono solo un salt casuale da 16 byte e l'hash derivato da 32 byte. Il PIN non viene mai memorizzato in nessuna forma. La verifica è a tempo costante:

```zig
pub fn verify(io: std.Io, allocator: std.mem.Allocator, pin: []const u8, salt: *const Salt, expected: *const Hash) !bool {
    var got: Hash = undefined;
    defer std.crypto.secureZero(u8, &got);
    try derive(io, allocator, pin, salt, &got);
    return std.crypto.timing_safe.eql(Hash, got, expected.*);
}
```

Per i tentativi online, il token conta i fallimenti e si blocca dopo tre. Da `slot_token.zig`, le informazioni del token riflettono i flag count low / final try / locked, e `C_Login` rifiuta una volta che il contatore raggiunge il limite:

```zig
if (inst.token.user_fail >= config.login_max_attempts) {
    state.mutex.unlock();
    return ck.CKR_PIN_LOCKED;
}
```

Lo smoke harness inserisce tre PIN sbagliati e verifica che il quarto tentativo restituisca `CKR_PIN_LOCKED` e che `CKF_USER_PIN_LOCKED` sia impostato nei flag del token.

### Errori comuni

**Errore: confrontare l'hash con un normale controllo di uguaglianza**
```zig
// Bad: std.mem.eql returns as soon as it finds a mismatched byte.
// The time it takes leaks how many leading bytes matched.
return std.mem.eql(u8, &got, expected);

// Good: timing_safe.eql looks at every byte before deciding.
return std.crypto.timing_safe.eql(Hash, got, expected.*);
```

**Errore: un hash veloce per il PIN**
```zig
// Bad: a thief with the file runs billions of SHA-256 guesses per second.
var h: [32]u8 = undefined;
std.crypto.hash.sha2.Sha256.hash(pin, &h, .{});

// Good: Argon2id makes each guess cost 64 MiB and three passes.
try argon2.kdf(allocator, &h, pin, &salt, params, .argon2id, io);
```

## Cifratura at rest e il pattern envelope

### Che cos'è

Il token persiste su un file così che le chiavi sopravvivano a un riavvio. I valori sensitive in quel file sono cifrati. La parte ingegnosa è *come* viene gestita la chiave di cifratura: una **master key** (MK) casuale per token cifra i segreti, e la MK è a sua volta cifrata ("wrapped") sotto una chiave derivata dal PIN dello User. Questo schema a due livelli è il pattern **envelope**.

### Perché è importante

Cifrare ogni segreto direttamente sotto una chiave derivata dal PIN sembra più semplice, ma significa che cambiare il PIN richiede di ricifrare ogni segreto. Con l'envelope, cambiare il PIN ri-avvolge una master key da 32 byte e nient'altro. Significa anche che la costosa derivazione Argon2id avviene una volta per login, non una volta per oggetto. AWS KMS, Google Tink e ogni sistema serio di gestione dei segreti usano la envelope encryption per queste ragioni.

La minaccia a cui risponde è la memorizzazione in chiaro dei segreti, CWE-312 (cleartext storage of sensitive information). Un file di token che contiene solo byte di chiavi su disco è a un singolo `cat` dalla compromissione totale.

### Come funziona qui

```
   User PIN ──Argon2id──▶ KEK ──AES-256-GCM wrap──▶ wrapped MK (on disk, in the token record)
                                                          │
                                                  C_Login unwraps with the KEK
                                                          ▼
   random master key (MK) ──AES-256-GCM seal──▶ sealed secret values (on disk, in the object file)
```

`keystore.zig` contiene le due metà. `wrap` deriva la KEK dal PIN e cifra con GCM la MK; `unwrap` fa l'inverso, e un PIN sbagliato fa fallire il tag GCM e restituisce false invece di spazzatura:

```zig
pub fn unwrap(io: std.Io, allocator: std.mem.Allocator, pin_bytes: []const u8, w: *const Wrapped, out: *MasterKey) !bool {
    var kek: MasterKey = undefined;
    defer std.crypto.secureZero(u8, &kek);
    try deriveKek(io, allocator, pin_bytes, &w.salt, &kek);
    gcm.decrypt(out, &w.ct, w.tag, "", w.nonce, kek) catch {
        std.crypto.secureZero(u8, out);
        return false;
    };
    return true;
}
```

`seal` e `unseal` proteggono i singoli valori degli attributi, legando ciascuno al proprio tipo di attributo come associated data così che un `CKA_PRIVATE_EXPONENT` sigillato non possa essere scambiato dove ci si aspettava un `CKA_VALUE`. L'object store sigilla in modo *selettivo*: solo i valori sensitive (i byte della chiave AES, l'esponente privato RSA e i fattori CRT, lo scalare EC) sono cifrati. Il materiale pubblico (il modulo, il punto EC, le label) resta in chiaro così da essere visibile prima del login, il che è corretto secondo la specifica.

### Attacchi comuni

1. **Furto del disco.** Il file è sigillato sotto una chiave che il ladro non ha. Senza il PIN, la MK resta avvolta.
2. **Manomissione.** Invertire un byte in un valore sigillato rompe il tag GCM all'unseal, e il modulo fallisce in modo chiuso (fail closed). I test a livello di store invertono un byte e verificano `error.AuthFailed`.
3. **Downgrade.** Un attaccante sostituisce un vecchio file di token risalente a prima di un cambio di PIN. La MK in quel file è avvolta sotto la vecchia KEK, quindi il nuovo PIN non può svolgerla. Anche la versione del record viene incrementata, quindi un vecchio file dell'era in chiaro viene rifiutato senza appello.

## Release of Unverified Plaintext (RUP)

### Che cos'è

AES-GCM è un cipher autenticato: la decifratura sia decifra sia verifica un tag da 128 bit, ed è il tag a dirti che il ciphertext non è stato manomesso. Un decrypt *in streaming* che restituisce il plaintext pezzo per pezzo ha un problema: ti consegna byte prima di aver visto il tag. Se agisci su quei byte e poi il tag risulta sbagliato, hai agito su dati controllati dall'attaccante. Questo è il release of unverified plaintext.

### Perché è importante

Per un HSM questo è inaccettabile. Il punto è proprio essere affidabile riguardo a ciò che restituisce. Restituire plaintext che non è stato autenticato, anche solo brevemente, è un punto d'appoggio per attacchi chosen-ciphertext. La comunità crittografica tratta la resistenza al RUP come una proprietà che un uso serio di AEAD deve avere.

### Come funziona qui

Il modulo rende il RUP impossibile per costruzione. GCM è implementato come operazione **bufferizzata**: `C_EncryptUpdate` e `C_DecryptUpdate` accodano a un buffer interno ed emettono zero byte, e il lavoro vero avviene una sola volta a `C_*Final`, dove il decrypt verifica il tag prima che venga rilasciato qualsiasi plaintext. L'accumulatore vive in `session.zig` come `GcmStream`, ed è limitato a 16 MiB così che un host non possa esaurire la memoria facendo streaming all'infinito:

```zig
pub fn append(self: *GcmStream, allocator: std.mem.Allocator, bytes: []const u8) error{ OutOfMemory, TooLarge }!void {
    if (bytes.len == 0) return;
    const needed = self.len + bytes.len;
    if (needed > config.max_gcm_stream_len) return error.TooLarge;
    ...
}
```

Lo smoke harness invia in streaming un messaggio multi-blocco attraverso `C_DecryptUpdate` in pezzi da 19 byte, verifica che ogni update emetta zero byte, e ottiene il plaintext solo a `C_DecryptFinal`. Poi inverte un byte e conferma che la decifratura restituisce `CKR_ENCRYPTED_DATA_INVALID` senza rilasciare alcun plaintext.

## Side channel: timing e azzeramento della memoria

### Che cos'è

Un side channel è un'informazione che trapela attraverso *come* viene eseguito un calcolo invece che dal suo output: quanto tempo ha impiegato, quale memoria ha toccato, qual è lo stato della cache dopo. Due side channel contano più di tutti per un modulo che gestisce chiavi: il timing (un confronto che esce in anticipo rivela dove si è fermato) e il residuo (un segreto lasciato nella memoria liberata dopo l'uso).

### Perché è importante

**Timing.** L'**attacco di Bleichenbacher** del 1998 e il suo ritorno del 2017, **ROBOT**, hanno recuperato segreti cifrati con RSA misurando come un server rispondeva a ciphertext PKCS#1 v1.5 malformati. **Lucky Thirteen** (2013) ha fatto lo stesso contro il padding CBC di TLS usando solo il timing. La lezione: qualsiasi branch il cui timing dipende da un segreto è un leak.

**Residuo.** Il **cold boot attack** del 2008 (Halderman et al.) ha mostrato che la DRAM conserva il suo contenuto per secondi o minuti dopo la perdita di alimentazione, abbastanza a lungo da farne il dump e recuperare le chiavi. Se una chiave resta nella memoria liberata, un crash dump, una pagina finita in swap o una lettura cold boot può recuperarla.

### Come funziona qui

I confronti tra segreti sono a tempo costante. La verifica del PIN usa `std.crypto.timing_safe.eql`; la verifica del MAC in `crypto_ops.zig` usa un `ctEql` scritto a mano che fa l'OR di ogni differenza tra byte prima di decidere:

```zig
fn ctEql(a: []const u8, b: []const u8) bool {
    if (a.len != b.len) return false;
    var diff: u8 = 0;
    for (a, b) |x, y| diff |= x ^ y;
    return diff == 0;
}
```

I segreti vengono azzerati quando escono dallo scope. I segreti sullo stack usano `defer std.crypto.secureZero(...)`; le union delle operazioni di sessione si azzerano da sole durante il teardown; l'object store libera ogni valore segreto tramite `secureFree`, che pulisce prima di rilasciare:

```zig
fn secureFree(allocator: std.mem.Allocator, value: []u8) void {
    std.crypto.secureZero(u8, value);
    allocator.free(value);
}
```

`secureZero` prende una slice `[]volatile` proprio perché al compilatore sia vietato eliminare la scrittura in ottimizzazione considerandola morta (non ha letture successive). Al logout l'intero store viene risigillato e la master key viene cancellata, quindi un token inattivo e disconnesso non contiene segreti in chiaro in RAM. [MECHANICS.md](./MECHANICS.it.md) tratta in modo più approfondito i meccanismi di tempo costante e azzeramento.

### Errori comuni

**Errore: azzerare tramite una slice non volatile**
```zig
// Bad: with no later read, the optimizer may delete this @memset entirely.
@memset(&key, 0);

// Good: secureZero forces the store to happen.
std.crypto.secureZero(u8, &key);
```

**Errore: un padding oracle tramite codici di errore distinti**
```zig
// Bad: returning a different error for "bad padding" vs "bad MAC" tells an
// attacker which step failed, which is what Bleichenbacher/Lucky13 exploit.

// Good: this module maps RSA decrypt failures to one uniform CKR code and
// lets libcrypto handle the padding check in constant time. CBC-PAD padding
// verification ORs all the pad bytes before deciding, no early exit.
```

## Come si relazionano questi concetti

I concetti sono livelli di un unico sistema. Ognuno dipende da quello sottostante.

```
   Key custody (sensitive, unextractable)
            │ requires
            ▼
   The login model (who may see what)
            │ requires
            ▼
   Authentication (Argon2id PIN + lockout)
            │ unlocks
            ▼
   Encryption at rest (envelope: PIN -> KEK -> MK -> sealed secrets)
            │ relies on
            ▼
   Authenticated encryption done safely (GCM, no RUP)
            │ relies on
            ▼
   Side-channel hygiene (constant-time compares, zeroized memory)
```

Se il confronto a tempo costante fa trapelare informazioni, il PIN cade. Se il PIN cade, l'envelope si apre. Se l'envelope si apre, il modello di login non conta più e la custodia delle chiavi è violata. La robustezza della catena è la robustezza del suo anello più debole, ed è per questo che il modulo è uniforme su tutti questi aspetti.

## Standard e framework del settore

### OWASP Top 10 (2021)

- **A02:2021 Cryptographic Failures.** La categoria principale. Questo progetto è uno studio su come non commetterli: KDF robusta per il PIN, AES-256-GCM per lo storage, nonce freschi, cifratura autenticata, nessuna primitiva fatta in casa tranne dove sono da manuale (assemblaggio della modalità CBC, wrap RFC 3394) e testate contro vettori pubblicati.
- **A04:2021 Insecure Design.** Il pattern envelope, la decisione che lo SO non possa leggere i segreti dello User e il GCM sicuro rispetto al RUP sono scelte a livello di design, non aggiunte posticce.
- **A07:2021 Identification and Authentication Failures.** Il lockout del PIN e il modello dei ruoli affrontano brute force e confusione dei privilegi.

### MITRE ATT&CK

- **T1552 Unsecured Credentials** e **T1555 Credentials from Password Stores.** Un HSM è la contromisura: le credenziali che autenticano tutto il resto non stanno in un archivio leggibile.
- **T1003 OS Credential Dumping.** L'azzeramento e una custodia che non tiene mai le chiavi in chiaro in memoria riducono ciò che un dump della memoria può restituire.
- **T1588.004 / T1649 Obtain or Forge Certificates.** Mantenere le chiavi di firma non estraibili è ciò che impedisce a una violazione di produrre certificati falsificati, la modalità di fallimento di DigiNotar.

### CWE

- **CWE-312 Cleartext Storage of Sensitive Information.** Difesa tramite l'envelope cifrato at rest.
- **CWE-316 Cleartext Storage in Memory.** Difesa tramite l'azzeramento e il nuovo blocco al logout.
- **CWE-208 Observable Timing Discrepancy.** Difesa tramite il confronto a tempo costante.
- **CWE-326 Inadequate Encryption Strength** e **CWE-327 Use of a Broken or Risky Cryptographic Algorithm.** Difesa tramite AES-256-GCM, Argon2id e schemi di firma standard.
- **CWE-522 Insufficiently Protected Credentials.** Difesa tramite lo stiramento con Argon2id e il lockout.

## Esempi reali

### Caso di studio: Heartbleed (CVE-2014-0160, 2014)

Un controllo dei limiti mancante nell'heartbeat TLS di OpenSSL permetteva a un attaccante remoto di leggere la memoria del server in blocchi da 64 KB. Chiavi private, cookie di sessione e password sono tutti trapelati. La difesa che ha funzionato è stata la custodia: le organizzazioni le cui chiavi private vivevano in un HSM non hanno fatto trapelare quelle chiavi, perché le chiavi non si sono mai trovate nella memoria del processo vulnerabile. L'applicazione di `CKA_SENSITIVE` e l'azzeramento in questo progetto sono lo stesso principio in miniatura.

### Caso di studio: la famiglia Bleichenbacher (1998, ROBOT 2017)

Daniel Bleichenbacher ha mostrato che un oracolo di decifratura RSA che rivela se il padding PKCS#1 v1.5 era valido permette a un attaccante di decifrare un ciphertext con circa un milione di query. Diciannove anni dopo ROBOT ha trovato la stessa falla ancora attiva nei principali stack TLS perché le "correzioni" non erano uniformi. La lezione incorporata in questo modulo: non lasciare mai che l'errore che restituisci dipenda da quale controllo interno è fallito, e preferisci OAEP, il cui design resiste all'attacco. Qui i fallimenti del decrypt RSA collassano in un unico codice di ritorno.

## Verifica la tua comprensione

Prima di andare avanti, assicurati di saper rispondere a queste domande.

1. Un host genera una chiave AES, poi chiama immediatamente `C_GetAttributeValue` per `CKA_VALUE`. Cosa ottiene in risposta, e quale funzione lo ha deciso? Quale sarebbe la risposta per `CKA_MODULUS_BITS` su una chiave pubblica RSA, e perché è diversa?
2. Percorri l'envelope da un PIN dello User fino a un valore segreto decifrato. Nomina ogni chiave nella catena e cosa cifra cosa. Perché cambiare il PIN non richiede di ricifrare ogni oggetto?
3. Perché `C_DecryptUpdate` per AES-GCM restituisce ogni volta zero byte, e dove appare effettivamente il plaintext? Quale attacco renderebbe possibile un rilascio pezzo per pezzo?
4. Lo SO può resettare il token. Perché lo SO non può leggere le chiavi dello User? Indica la decisione di design che rende vero questo.

Se qualcuna di queste non ti è chiara, rileggi la sezione corrispondente. L'implementazione avrà molto più senso una volta che questi concetti saranno chiari.

## Letture di approfondimento

**Essenziali:**
- [PKCS#11 v2.40 Base Specification (OASIS)](https://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html). Il contratto che questo modulo implementa. Leggi le sezioni sulla semantica delle funzioni e sul modello a oggetti.
- [SoftHSMv2](https://github.com/softhsm/SoftHSMv2). L'HSM software open source di riferimento. Questo progetto ne prende in prestito la suddivisione in tre livelli e l'idea dell'object store.

**Approfondimenti:**
- Bleichenbacher, "Chosen Ciphertext Attacks Against Protocols Based on the RSA Encryption Standard PKCS #1" (CRYPTO 1998), e il resoconto di ROBOT su robotattack.org per la ricomparsa moderna.
- Halderman et al., "Lest We Remember: Cold Boot Attacks on Encryption Keys" (USENIX Security 2008), la motivazione dietro `secureZero`.
- Il paper di Argon2 e la RFC 9106 per capire perché una KDF memory-hard batte un hash veloce per i segreti a bassa entropia.

**Contesto storico:**
- I documenti PKCS#11 originali di RSA Security (pre-OASIS) per capire come lo standard sia arrivato ad avere l'aspetto che ha, e perché "la function list è l'API".
