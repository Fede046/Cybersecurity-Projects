<!-- ©AngelaMos | 2026 -->
<!-- CONFORMANCE.md -->

# Dichiarazione di conformità PKCS#11 v2.40

L'AngelaMos HSM Emulator implementa l'intera C ABI Cryptoki (PKCS#11) v2.40: tutte
le 68 funzioni della `CK_FUNCTION_LIST` canonica, verificate dalla macchina contro gli
header OASIS in vendor a build time. Questo documento registra ogni punto in cui il
modulo restringe il comportamento: il codice di ritorno esatto che fornisce, la clausola
della specifica che lo consente, e il perché. Ogni restrizione qui è una decisione
documentata con un valore di ritorno definito.

Una funzione che non è applicabile a un token software fisso restituisce il codice
specifico che la specifica definisce per quella situazione. I risultati
`CKR_FUNCTION_NOT_SUPPORTED` che restano segnano il confine tra le superfici delle
operazioni single-shot e multi-part, trattate nella sezione 2.

**Specifiche**

- PKCS#11 Base Specification v2.40 (OASIS, errata 01) — semantica delle funzioni e codici di ritorno.
- PKCS#11 Current Mechanisms v2.40 (OASIS, errata 01) — parametri per meccanismo.

I numeri di sezione qui sotto si riferiscono alla specifica Base, a meno che una riga
non nomini il documento Mechanisms.

---

## 1. Conformità a livello di funzione

### 1.1 `C_WaitForSlotEvent` — §5.5 (gestione di slot e token)

Il modulo espone un unico slot fisso (ID 0) il cui token è sempre presente. Nessun
evento di inserimento o rimozione può mai verificarsi, quindi la funzione riporta
quel fatto con precisione invece di fingere di supportare gli eventi degli slot hardware.

| Chiamata | Ritorno | Base |
|------|--------|-------|
| `flags` ha `CKF_DONT_BLOCK`, nessun evento in attesa | `CKR_NO_EVENT` | §5.5: un poll non bloccante senza eventi in attesa restituisce `CKR_NO_EVENT`. Per uno slot fisso nessun evento è mai in attesa, quindi questa è sempre la risposta. |
| `flags` azzera `CKF_DONT_BLOCK` (bloccante) | `CKR_FUNCTION_NOT_SUPPORTED` | Un'attesa bloccante non deve ritornare finché non si verifica un evento. Per uno slot software fisso nessun evento può verificarsi, quindi bloccare appenderebbe il chiamante per sempre. Il modulo rifiuta invece la modalità bloccante. |
| `pReserved != NULL_PTR` | `CKR_ARGUMENTS_BAD` | §5.5: `pReserved` è riservato e deve essere `NULL_PTR` nella v2.40. |
| chiamata prima di `C_Initialize` | `CKR_CRYPTOKI_NOT_INITIALIZED` | §5.4 semantica generale. |

Nota di interoperabilità: `pkcs11-tool --wait` chiama questa funzione in modalità
bloccante e quindi riceve immediatamente `CKR_FUNCTION_NOT_SUPPORTED` invece di bloccarsi.
Gli host che fanno polling con `CKF_DONT_BLOCK` (il caso comune per l'enumerazione degli
slot) ottengono il corretto `CKR_NO_EVENT`.

### 1.2 `C_GetFunctionStatus`, `C_CancelFunction` — §5.15 (gestione delle funzioni parallele)

Entrambe sono funzioni legacy dell'epoca delle chiamate Cryptoki parallele (asincrone).
La v2.40 non ha un modello di esecuzione parallela, e la specifica definisce la risposta
canonica per un'implementazione seriale:

| Chiamata | Ritorno | Base |
|------|--------|-------|
| `C_GetFunctionStatus` | `CKR_FUNCTION_NOT_PARALLEL` | §5.15: l'unico ritorno sensato per una libreria che non esegue funzioni in parallelo. |
| `C_CancelFunction` | `CKR_FUNCTION_NOT_PARALLEL` | §5.15: idem. |

### 1.3 `C_SeedRandom` — §5.14 (generazione di numeri casuali)

| Chiamata | Ritorno | Base |
|------|--------|-------|
| `C_SeedRandom` | `CKR_RANDOM_SEED_NOT_SUPPORTED` | L'RNG è il CSPRNG del sistema operativo, ottenuto tramite `std.Io.randomSecure` (`getrandom(2)`, `arc4random_buf` o `/dev/urandom` a seconda della piattaforma e della libc). Il materiale di seed fornito dal chiamante non può reinizializzarlo in modo significativo, quindi il modulo rifiuta invece di scartare silenziosamente il seed (il che trarrebbe in inganno il chiamante). |
| `C_GenerateRandom` | pienamente supportata | — |

---

## 2. Confini della superficie delle operazioni

Questi sono i limiti deliberati della superficie delle operazioni multi-part. Ognuno
restituisce un codice specifico così che un chiamante possa distinguere "chiamata
sbagliata per questo meccanismo" da un fallimento a runtime.

### 2.1 AES-GCM è bufferizzato in modo sicuro rispetto al RUP — §5.8, §5.9

La cifratura e la decifratura multi-part `CKM_AES_GCM` bufferizzano l'intero messaggio
ed eseguono l'operazione autenticata **una sola volta a `*Final`**:

- `C_EncryptUpdate` / `C_DecryptUpdate` accodano la parte a un buffer interno ed
  emettono **0 byte**. §5.8/§5.9 consentono a un Update di produrre meno byte di output
  di quanti ne consumi (block buffering); produrre l'intero risultato a `*Final` è conforme.
- `C_EncryptFinal` emette ciphertext + tag da 128 bit; `C_DecryptFinal` verifica il tag
  e solo allora rilascia il plaintext.

Questa è una decisione di sicurezza. Un *decrypt* GCM in streaming costruito su un
rilascio incrementale restituirebbe **plaintext non verificato** prima che il tag sia
controllato (release of unverified plaintext, "RUP") — un anti-pattern per un HSM.
Bufferizzare finché il tag non viene verificato rende il RUP impossibile per costruzione.

Poiché il buffer contiene l'intero messaggio, un singolo messaggio GCM è limitato:

| Condizione | Ritorno |
|-----------|--------|
| la lunghezza bufferizzata supererebbe 16 MiB (`max_gcm_stream_len`), encrypt | `CKR_DATA_LEN_RANGE` |
| la lunghezza bufferizzata supererebbe 16 MiB, decrypt | `CKR_ENCRYPTED_DATA_LEN_RANGE` |

Validazione rigorosa dei parametri (`CK_GCM_PARAMS`):

| Parametro | Accettato | Altrimenti |
|-----------|----------|------|
| `ulIvLen` | esattamente 12 byte | `CKR_MECHANISM_PARAM_INVALID` |
| `ulIvBits` | `0` o `96` | `CKR_MECHANISM_PARAM_INVALID` |
| `ulTagBits` | esattamente `128` | `CKR_MECHANISM_PARAM_INVALID` |
| `pIv` | non NULL | `CKR_MECHANISM_PARAM_INVALID` |
| `ulAADLen` | ≤ 256 byte | `CKR_ARGUMENTS_BAD` |

Nota di interoperabilità: un host DEVE richiedere un tag da 128 bit e fornire un IV da 12 byte.
Con `pkcs11-tool` questo significa `--iv <24 hex chars> --tag-bits-len 128`; omettere uno
dei due fa scattare `CKR_MECHANISM_PARAM_INVALID`.

### 2.2 Operazioni a doppia funzione — §5.12

| Funzione | Modalità supportate | Altrimenti |
|----------|-----------------|------|
| `C_DigestEncryptUpdate`, `C_SignEncryptUpdate` (lato encrypt) | AES-CBC, AES-CBC-PAD, AES-GCM | — |
| `C_DecryptDigestUpdate`, `C_DecryptVerifyUpdate` (lato decrypt) | solo AES-CBC | non CBC → `CKR_FUNCTION_NOT_SUPPORTED` |
| qualsiasi ramo a doppia funzione che usi un'operazione RSA di sign/verify | — | `CKR_FUNCTION_NOT_SUPPORTED` |

Il lato decrypt accoppia il **plaintext recuperato** da ogni `C_Decrypt*Update`
all'operazione di digest/verify. Quell'accoppiamento è esatto solo quando il cipher
rilascia esattamente i byte decifrati a ogni chiamata:

- **AES-CBC** rilascia immediatamente ogni blocco decifrato, quindi i byte in uscita sono
  uguali ai byte da sottoporre a digest per ogni chiamata. Supportato.
- **AES-CBC-PAD** trattiene il blocco finale fino a `C_DecryptFinal` (non può conoscere
  il padding fino alla fine), quindi il digest perderebbe l'ultimo blocco.
- **AES-GCM** bufferizza tutto e rilascia a `C_DecryptFinal` (§2.1), quindi il
  digest non riceverebbe nulla in modo incrementale.

Accoppiare uno degli ultimi due desincronizzerebbe la coppia, quindi il modulo restituisce
`CKR_FUNCTION_NOT_SUPPORTED` invece di produrre silenziosamente un digest sbagliato. Il
lato encrypt non ha un vincolo del genere — esegue digest/firma del plaintext di **input**,
che è completamente disponibile a ogni chiamata, quindi tutte e tre le modalità funzionano.

### 2.3 RSA è single-shot — §5.8, §5.9, §5.11

Ogni operazione RSA è una singola esponenziazione modulare sull'intero input, quindi RSA
non ha una forma multi-part:

| Chiamata su un'operazione RSA | Ritorno |
|--------------------------|--------|
| `C_EncryptUpdate` / `C_EncryptFinal` | `CKR_FUNCTION_NOT_SUPPORTED` |
| `C_DecryptUpdate` / `C_DecryptFinal` | `CKR_FUNCTION_NOT_SUPPORTED` |
| `C_SignUpdate` / `C_SignFinal` | `CKR_FUNCTION_NOT_SUPPORTED` |
| `C_VerifyUpdate` / `C_VerifyFinal` | `CKR_FUNCTION_NOT_SUPPORTED` |

Usa i one-shot `C_Encrypt` / `C_Decrypt` / `C_Sign` / `C_Verify`. I meccanismi
hash-then-sign (`CKM_SHA256_RSA_PKCS`, le varianti PSS) hanno comunque bisogno dell'intero
messaggio prima della singola operazione RSA, quindi anche loro sono one-shot.

### 2.4 Sign / Verify-Recover — §5.11

| Meccanismo | Supportato | Altrimenti |
|-----------|-----------|------|
| `CKM_RSA_PKCS` (EMSA-PKCS1-v1.5 tipo 1, messaggio recuperabile dalla firma) | sì | altro meccanismo → `CKR_MECHANISM_INVALID` |

`C_SignRecover` rifiuta con `CKR_DATA_LEN_RANGE` un input che non può entrare nel modulo
con l'overhead di PKCS#1 v1.5 (lunghezza dei dati + 11 byte > modulo). `C_VerifyRecover`
restituisce il messaggio recuperato; una firma la cui lunghezza non è la lunghezza del modulo
restituisce `CKR_SIGNATURE_LEN_RANGE`, e una codifica malformata restituisce
`CKR_SIGNATURE_INVALID`. Il recover raw (`CKM_RSA_X_509`) non è offerto.

### 2.5 Get / SetOperationState — §5.6

Lo stato delle operazioni è **solo per il digest**:

| Chiamata | Ritorno | Base |
|------|--------|-------|
| `C_GetOperationState` con un'operazione attiva di sign/verify/encrypt/decrypt/sign-recover/verify-recover | `CKR_STATE_UNSAVEABLE` | §5.6: una libreria può rifiutarsi di salvare uno stato che non può serializzare. Qui solo lo stato del digest è salvabile. |
| `C_GetOperationState` senza alcuna operazione attiva | `CKR_OPERATION_NOT_INITIALIZED` | §5.6. |
| `C_SetOperationState` con `hEncryptionKey` o `hAuthenticationKey` diverso da zero | `CKR_KEY_NOT_NEEDED` | §5.6: un digest non ha bisogno di una chiave, quindi passarne una è un errore. |
| `C_SetOperationState` con un blob malformato (byte di versione sbagliato, tag di hasher sconosciuto, lunghezza sbagliata) | `CKR_SAVED_STATE_INVALID` | §5.6. |

Il blob salvato è `[version][hasher tag][raw hasher state]`, validato al ripristino.
È **opaco e valido solo per la stessa build**: porta con sé lo stato grezzo dell'hasher della
standard library, il cui layout non è stabile tra build diverse. PKCS#11 non promette la
portabilità dello stato delle operazioni tra implementazioni o build (§5.6). Il ripristino è
esatto al byte all'interno dello stesso binario; qualsiasi altro input fallisce in modo chiuso
tramite i controlli di versione, tag e lunghezza esatta sopra indicati.

---

## 3. Vincoli dei meccanismi

### 3.1 Derivazione di chiavi ECDH — §5.13, Mechanisms (CKM_ECDH1_DERIVE)

| Aspetto | Valore | Altrimenti |
|--------|-------|------|
| KDF | solo `CKD_NULL` | altra KDF → `CKR_MECHANISM_PARAM_INVALID` |
| punto pubblico del peer | SEC1 raw non compresso **oppure** avvolto in un `OCTET STRING` DER | malformato → `CKR_MECHANISM_PARAM_INVALID` |
| curve | P-256, P-384 | — |
| shared data | nessuno (`CKD_NULL` non porta shared data) | — |

### 3.2 RSA — Mechanisms (CKM_RSA_PKCS, _PSS, _OAEP)

| Aspetto | Valore | Altrimenti |
|--------|-------|------|
| dimensione della chiave | 2048–4096 bit | fuori intervallo → `CKR_KEY_SIZE_RANGE` |
| esponente pubblico | fisso a 65537 (F4) | non selezionabile alla keygen |
| hash PSS / OAEP | SHA-256, SHA-384, SHA-512 | altro → `CKR_MECHANISM_PARAM_INVALID` |
| hash MGF | deve essere uguale all'hash del contenuto | discrepanza → `CKR_MECHANISM_PARAM_INVALID` |
| label OAEP (source) | non supportata | `ulSourceDataLen != 0` → `CKR_MECHANISM_PARAM_INVALID` |

I meccanismi dedicati `CKM_SHA384_RSA_PKCS` / `CKM_SHA512_RSA_PKCS` non sono
pubblicizzati; SHA-384 e SHA-512 sono raggiungibili tramite il parametro `hashAlg` di
`CKM_RSA_PKCS_PSS` e `CKM_RSA_PKCS_OAEP`.

### 3.3 AES — Mechanisms (CKM_AES_*)

Solo AES-128 e AES-256. AES-192 non è implementato (la standard library di Zig
non espone AES a 192 bit). Una lunghezza della chiave al di fuori di {16, 32} byte → `CKR_KEY_SIZE_RANGE`.

### 3.4 ECDSA — Mechanisms (CKM_ECDSA*)

Curve P-256 e P-384. Meccanismi `CKM_ECDSA` (input già sottoposto a hash) e
`CKM_ECDSA_SHA256`. `CKM_ECDSA_SHA384` e `CKM_ECDSA_SHA512` sono fuori scope.

### 3.5 Key wrap — §5.13

| Aspetto | Valore | Altrimenti |
|--------|-------|------|
| target che si può avvolgere | solo chiavi segrete (`CKO_SECRET_KEY`) | target asimmetrico → `CKR_KEY_NOT_WRAPPABLE` |
| meccanismi di wrapping | `CKM_AES_KEY_WRAP` (RFC 3394), `CKM_RSA_PKCS_OAEP` | altro → `CKR_MECHANISM_INVALID` |
| target non estraibile | rifiutato | `CKR_KEY_UNEXTRACTABLE` |
| blob avvolto manomesso all'unwrap | rifiutato | `CKR_WRAPPED_KEY_INVALID` |

---

## 4. Modello di oggetti e token (informativo)

- **Un unico slot fisso** (ID 0), sempre presente, che ospita **un token**. Il login è
  necessario per vedere o usare gli oggetti privati.
- **Cifrato at rest.** Gli oggetti token persistono su un file sotto un envelope selettivo:
  solo i *valori* degli attributi sensitive sono sigillati con AES-256-GCM sotto una
  master key per token. La master key è avvolta sotto un unico **keyslot del PIN User**
  (KEK derivata con Argon2id). Per design non c'è alcun keyslot SO per i segreti dello user — il
  Security Officer non deve poter leggere il materiale delle chiavi dello user.
- **Gli oggetti e gli attributi pubblici restano in chiaro** e sono visibili prima del login,
  il che è corretto secondo la specifica: solo il materiale privato/sensitive è protetto dal login.
- In memoria, gli attributi sensitive sono in chiaro solo mentre lo User ha fatto login;
  il logout e il teardown della sessione li risigillano e azzerano la master key.

---

## 5. Elenco dei meccanismi pubblicizzati

`C_GetMechanismList` restituisce questi 21 meccanismi. Le unità della dimensione della chiave
dipendono dal meccanismo secondo la specifica (bit per RSA ed EC, byte per AES e HMAC).

| Meccanismo | Min | Max | Flag |
|-----------|-----|-----|-------|
| `CKM_SHA256` | 0 | 0 | DIGEST |
| `CKM_SHA384` | 0 | 0 | DIGEST |
| `CKM_SHA512` | 0 | 0 | DIGEST |
| `CKM_SHA256_HMAC` | 32 | 64 | SIGN, VERIFY |
| `CKM_SHA384_HMAC` | 32 | 64 | SIGN, VERIFY |
| `CKM_SHA512_HMAC` | 32 | 64 | SIGN, VERIFY |
| `CKM_AES_KEY_GEN` | 16 | 32 | GENERATE |
| `CKM_AES_CBC` | 16 | 32 | ENCRYPT, DECRYPT |
| `CKM_AES_CBC_PAD` | 16 | 32 | ENCRYPT, DECRYPT |
| `CKM_AES_GCM` | 16 | 32 | ENCRYPT, DECRYPT |
| `CKM_EC_KEY_PAIR_GEN` | 256 | 384 | GENERATE_KEY_PAIR, EC_NAMEDCURVE |
| `CKM_ECDSA` | 256 | 384 | SIGN, VERIFY, EC_NAMEDCURVE |
| `CKM_ECDSA_SHA256` | 256 | 384 | SIGN, VERIFY, EC_NAMEDCURVE |
| `CKM_ECDH1_DERIVE` | 256 | 384 | DERIVE, EC_NAMEDCURVE |
| `CKM_RSA_PKCS_KEY_PAIR_GEN` | 2048 | 4096 | GENERATE_KEY_PAIR |
| `CKM_RSA_PKCS` | 2048 | 4096 | SIGN, VERIFY, ENCRYPT, DECRYPT, SIGN_RECOVER, VERIFY_RECOVER |
| `CKM_SHA256_RSA_PKCS` | 2048 | 4096 | SIGN, VERIFY |
| `CKM_RSA_PKCS_PSS` | 2048 | 4096 | SIGN, VERIFY |
| `CKM_SHA256_RSA_PKCS_PSS` | 2048 | 4096 | SIGN, VERIFY |
| `CKM_RSA_PKCS_OAEP` | 2048 | 4096 | ENCRYPT, DECRYPT, WRAP, UNWRAP |
| `CKM_AES_KEY_WRAP` | 16 | 32 | WRAP, UNWRAP |

---

## 6. Riepilogo: codici di ritorno deliberati

| Confine | Codice di ritorno |
|----------|-------------|
| `C_WaitForSlotEvent`, poll non bloccante | `CKR_NO_EVENT` |
| `C_WaitForSlotEvent`, modalità bloccante | `CKR_FUNCTION_NOT_SUPPORTED` |
| `C_WaitForSlotEvent`, `pReserved != NULL` | `CKR_ARGUMENTS_BAD` |
| `C_GetFunctionStatus`, `C_CancelFunction` | `CKR_FUNCTION_NOT_PARALLEL` |
| `C_SeedRandom` | `CKR_RANDOM_SEED_NOT_SUPPORTED` |
| messaggio GCM oltre 16 MiB | `CKR_DATA_LEN_RANGE` / `CKR_ENCRYPTED_DATA_LEN_RANGE` |
| parametri GCM errati | `CKR_MECHANISM_PARAM_INVALID` / `CKR_ARGUMENTS_BAD` |
| lato decrypt a doppia funzione, non CBC | `CKR_FUNCTION_NOT_SUPPORTED` |
| RSA multi-part (`*Update` / `*Final`) | `CKR_FUNCTION_NOT_SUPPORTED` |
| Sign/Verify-Recover, non `CKM_RSA_PKCS` | `CKR_MECHANISM_INVALID` |
| Get/SetOperationState, operazione non di digest | `CKR_STATE_UNSAVEABLE` |
| SetOperationState con un handle di chiave | `CKR_KEY_NOT_NEEDED` |
| SetOperationState, blob malformato | `CKR_SAVED_STATE_INVALID` |
| ECDH con KDF diversa da `CKD_NULL` | `CKR_MECHANISM_PARAM_INVALID` |
| OAEP con una label | `CKR_MECHANISM_PARAM_INVALID` |
| lunghezza della chiave AES diversa da 16/32 byte | `CKR_KEY_SIZE_RANGE` |
| wrap di un target asimmetrico | `CKR_KEY_NOT_WRAPPABLE` |
| wrap di una chiave non estraibile | `CKR_KEY_UNEXTRACTABLE` |

Ogni voce qui sopra è messa alla prova dagli unit test, dallo smoke harness in-process o da
un'esecuzione cross-process di `pkcs11-tool`; i codici corretti per slot/parallelismo/RNG sono
verificati in `examples/smoke.zig` contro lo shared object compilato.
