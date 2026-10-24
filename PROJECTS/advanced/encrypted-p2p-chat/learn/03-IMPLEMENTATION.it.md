# Guida all'implementazione

Questo documento percorre ogni parte significativa della chat P2P cifrata, file per file, spiegando non solo cosa fa il codice ma perché è stata presa ogni decisione. Se vuoi costruire qualcosa come Signal da zero, questa è la mappa.

---

## Panoramica della struttura dei file

```
backend/
 app/
 api/
 auth.py # WebAuthn registration/login endpoints
 encryption.py # Key upload, prekey bundle retrieval
 rooms.py # Chat room creation and listing
 websocket.py # WebSocket endpoint (receives raw frames)
 core/
 passkey/
 passkey_manager.py # WebAuthn: registration, auth, clone detection
 dependencies.py # current_user, issue_session, revoke_session
 websocket_manager.py # Connection pool, heartbeats, live query sub
 surreal_manager.py # SurrealDB client (messages, live queries)
 redis_manager.py # Redis client (sessions, challenges)
 exceptions.py # Custom exception hierarchy
 exception_handlers.py # FastAPI exception-to-HTTP mappings
 enums.py # Presence status enum
 models/
 Base.py # SQLModel engine, async session maker
 User.py # User table (id, username, webauthn_user_handle, ...)
 Credential.py # WebAuthn credential storage (BE/BS, sign_count)
 IdentityKey.py # X25519 + Ed25519 identity public keys (no privates)
 SignedPrekey.py # Signed prekey publics + signature
 OneTimePrekey.py # Single-use prekey publics (is_used flag)
 schemas/
 auth.py # Pydantic schemas for registration/auth flows
 websocket.py # Pydantic schemas for WS message types
 surreal.py # Pydantic schemas for SurrealDB documents
 rooms.py # Pydantic schemas for room operations
 common.py # Shared response schemas
 services/
 auth_service.py # User creation, credential management, search
 prekey_service.py # Public-only prekey storage + bundle lookup
 message_service.py # Pass-through ciphertext writer
 websocket_service.py # WS routing + per-user rate cap + membership
 presence_service.py # Online/offline/away status via SurrealDB
 config.py # All constants + Settings (env vars)
 factory.py # FastAPI app factory with middleware
 main.py # Uvicorn entry point

frontend/
 src/
 crypto/
 primitives.ts # Low-level WebCrypto: X25519, Ed25519, AES-GCM, HKDF, HMAC
 x3dh.ts # Client-side X3DH: generate keys, initiate, receive
 double-ratchet.ts # Client-side Double Ratchet: init, encrypt, decrypt, serialize
 crypto-service.ts # High-level orchestrator: session management, key upload
 key-store.ts # IndexedDB persistence for all key material
 message-store.ts # Message history persistence
 websocket/
 websocket-manager.ts # WS connection, reconnect, heartbeat, message queue
 message-handlers.ts # Dispatch incoming WS messages to stores
 stores/
 auth.store.ts # Authentication state (nanostores)
 session.store.ts # Session token management
 messages.store.ts # Message list per room
 rooms.store.ts # Room list and active room
 presence.store.ts # Online status per user
 typing.store.ts # Typing indicator state
 ui.store.ts # UI state (sidebar open, modals)
 settings.store.ts # User preferences
 services/
 auth.service.ts # WebAuthn browser API calls
 room.service.ts # Room CRUD via API
 components/
 Auth/ # Login, Register, PasskeyButton
 Chat/ # MessageList, ChatInput, ConversationList, etc.
 Layout/ # AppShell, Sidebar, Header, ProtectedRoute
 UI/ # Button, Modal, Input, Tooltip, etc.
 pages/
 Chat.tsx # Main chat page (conversation + message pane)
 Login.tsx # Login page
 Register.tsx # Registration page
 Home.tsx # Landing page
 NotFound.tsx # 404
 types/
 encryption.ts # Type definitions for crypto structures
 chat.ts # Message and room types
 websocket.ts # WS message types
 auth.ts # Auth types
 guards.ts # Runtime type guards for WS messages
 api.ts # API response types
 components.ts # Component prop types
 index.ts # Re-exports + constants
 lib/
 api-client.ts # HTTP client for REST endpoints
 base64.ts # Base64 helpers
 validators.ts # Input validation
 date.ts # Date formatting
 config.ts # Frontend constants (URLs, timeouts)
```

---

## Costruire lo scambio di chiavi X3DH

X3DH (Extended Triple Diffie-Hellman) risolve un problema specifico: Alice vuole inviare a Bob un messaggio cifrato, ma Bob è offline. Non hanno mai comunicato prima. Non c'è modo di fare un handshake live. X3DH permette ad Alice di calcolare un segreto condiviso usando le chiavi pre-pubblicate di Bob, così che quando Bob torna online possa derivare lo stesso segreto e decifrare tutto ciò che Alice ha inviato mentre era via.

### Passo 1: generazione delle chiavi

Il fondamento di X3DH è la generazione di due tipi di coppie di identity key per utente. Ecco la generazione della identity key X25519 da `frontend/src/crypto/x3dh.ts`:

```python
def generate_identity_keypair_x25519(self) -> tuple[str, str]:
 private_key = X25519PrivateKey.generate
 public_key = private_key.public_key

 private_bytes = private_key.private_bytes(
 encoding = serialization.Encoding.Raw,
 format = serialization.PrivateFormat.Raw,
 encryption_algorithm = serialization.NoEncryption
 )
 public_bytes = public_key.public_bytes(
 encoding = serialization.Encoding.Raw,
 format = serialization.PublicFormat.Raw
 )

 return (
 bytes_to_base64url(private_bytes),
 bytes_to_base64url(public_bytes)
 )
```

C'è anche `generate_identity_keypair_ed25519` in `frontend/src/crypto/x3dh.ts`, che segue esattamente lo stesso pattern ma usa invece `Ed25519PrivateKey.generate`.

**Perché la codifica Raw?** I formati PEM e DER includono header di metadati, identificatori di algoritmo e struttura ASN.1. Sono byte sprecati quando conosci già l'algoritmo su entrambi i lati. La codifica Raw ti dà esattamente 32 byte per X25519 e 32 byte per Ed25519. È l'ingombro minimo, e corrisponde direttamente a ciò che la WebCrypto API si aspetta sul frontend quando importa con `importKey("raw", ...)`.

**Perché base64url?** Le chiavi devono viaggiare attraverso JSON (su HTTP e WebSocket) ed essere memorizzate in colonne di testo PostgreSQL. La codifica esadecimale raddoppia la dimensione (32 byte diventano 64 caratteri). Il base64 standard usa i caratteri `+` e `/` che richiedono l'URL encoding. Base64url usa invece `-` e `_`, che sono sicuri in URL, JSON e parametri di query senza escaping. La libreria `webauthn.helpers` fornisce `bytes_to_base64url` e `base64url_to_bytes`, quindi ci appoggiamo a quelle anziché scriverne di nostre.

**Perché due tipi di chiave (X25519 ed Ed25519)?** X25519 è una funzione Diffie-Hellman. Prende due chiavi e produce un segreto condiviso. È ciò che serve per l'accordo sulla chiave vero e proprio. Ma X25519 non può produrre firme. Non puoi dimostrare "sono il proprietario di questa chiave pubblica" con il solo X25519. Ed25519 è un algoritmo di firma che opera sulla stessa famiglia di curve (Curve25519), quindi la generazione delle chiavi è altrettanto veloce. La identity key Ed25519 firma la Signed Prekey, dimostrando ad Alice che è stato davvero Bob a pubblicare quella prekey e non qualche attaccante nel mezzo. Separare i due tipi di chiave segue il principio crittografico di usare chiavi distinte per scopi distinti. Se usassi una singola chiave sia per il DH sia per la firma, una vulnerabilità in un'operazione potrebbe compromettere l'altra.

### Passo 2: creazione della signed prekey

La Signed Prekey (SPK) è una chiave X25519 semi-statica che viene ruotata ogni 48 ore. Ecco `generate_signed_prekey` da `frontend/src/crypto/x3dh.ts`:

```python
def generate_signed_prekey(self,
 identity_private_key_ed25519: str) -> tuple[str,
 str,
 str]:
 spk_private = X25519PrivateKey.generate
 spk_public = spk_private.public_key

 spk_private_bytes = spk_private.private_bytes(
 encoding = serialization.Encoding.Raw,
 format = serialization.PrivateFormat.Raw,
 encryption_algorithm = serialization.NoEncryption
 )
 spk_public_bytes = spk_public.public_bytes(
 encoding = serialization.Encoding.Raw,
 format = serialization.PublicFormat.Raw
 )

 identity_private_bytes = base64url_to_bytes(identity_private_key_ed25519)
 identity_private = Ed25519PrivateKey.from_private_bytes(
 identity_private_bytes
 )

 signature = identity_private.sign(spk_public_bytes)

 return (
 bytes_to_base64url(spk_private_bytes),
 bytes_to_base64url(spk_public_bytes),
 bytes_to_base64url(signature)
 )
```

**Perché firmare con Ed25519?** La firma dimostra che la identity key Ed25519 a lungo termine di Bob avalla questa specifica SPK. Quando Alice recupera il prekey bundle di Bob, verifica questa firma prima di eseguire le operazioni DH. Senza la firma, un man-in-the-middle potrebbe sostituire la chiave pubblica SPK nel bundle di Bob con la propria. Alice calcolerebbe un segreto condiviso con l'attaccante anziché con Bob, e l'attaccante potrebbe inoltrare i messaggi tra loro leggendo tutto.

**Cosa dimostra la firma?** Dimostra esattamente una cosa: "L'entità che controlla la chiave privata Ed25519 corrispondente alla chiave pubblica Ed25519 pubblicata da Bob ha scelto di firmare questa specifica chiave pubblica SPK." Questo lega la SPK all'identità di Bob.

**Perché non autofirmare con X25519?** X25519 è una funzione DH, non uno schema di firma. Non esiste un metodo `sign` su una chiave X25519. In teoria potresti convertire tra i formati di chiave Curve25519 ed Ed25519 (condividono la stessa curva sottostante), ma fare quella conversione è un modo per spararsi sui piedi. È soggetta a errori, non tutte le librerie la supportano e viola il principio di separazione delle chiavi. Usare una chiave di firma Ed25519 dedicata è più semplice e più sicuro.

### Passo 3: assemblaggio del prekey bundle

Quando Alice vuole scrivere a Bob, il server assembla il prekey bundle di Bob. Questo avviene in `backend/app/services/prekey_service.py`:

```python
async def get_prekey_bundle(
 self,
 session: AsyncSession,
 user_id: UUID
) -> PreKeyBundle:
 # Fetch identity key
 ik_statement = select(IdentityKey).where(IdentityKey.user_id == user_id)
 ...
 # Fetch active signed prekey
 spk_statement = select(SignedPrekey).where(
 SignedPrekey.user_id == user_id,
 SignedPrekey.is_active
 ).order_by(SignedPrekey.created_at.desc)
 ...
 if not signed_prekey:
 signed_prekey = await self.rotate_signed_prekey(session, user_id)

 # Fetch one unused OPK
 opk_statement = select(OneTimePrekey).where(
 OneTimePrekey.user_id == user_id,
 not OneTimePrekey.is_used
 ).limit(1)
 ...
 if one_time_prekey:
 one_time_prekey.is_used = True
 ...
```

**Perché la rotazione automatica?** Se la signed prekey di Bob fosse scaduta e non ce ne fossero di attive, il recupero del bundle fallirebbe. La rotazione automatica in `backend/app/services/prekey_service.py` garantisce che ci sia sempre una SPK attiva. La rotazione genera una nuova coppia di chiavi X25519, la firma con la identity key Ed25519 di Bob, marca le vecchie SPK come inattive e imposta una scadenza di 48 ore (`SIGNED_PREKEY_ROTATION_HOURS = 48` in `config.py`).

**Perché marcare la OPK come usata?** Le One-Time Prekey forniscono forward secrecy per l'handshake iniziale. Ogni OPK viene consumata esattamente una volta. Quando Alice recupera il bundle di Bob, il server marca quella OPK come `is_used = True` ed esegue il commit della modifica. Se una seconda persona prova anch'essa ad avviare una conversazione con Bob più o meno nello stesso momento, ottiene una OPK diversa (o nessuna). Questa garanzia di uso singolo significa che anche se le chiavi a lungo termine di Bob venissero compromesse in seguito, un attaccante non potrebbe calcolare retroattivamente il segreto condiviso per le conversazioni che hanno usato una OPK, perché la chiave privata della OPK è stata consumata e l'output DH derivato da essa non può essere ricostruito a partire dalle sole altre chiavi.

**Cosa succede se non è disponibile alcuna OPK?** Il protocollo funziona comunque. La dataclass `PreKeyBundle` ha `one_time_prekey: str | None = None`. Quando il bundle viene assemblato senza una OPK, il lato mittente di X3DH calcola semplicemente `dh1 + dh2 + dh3` invece di `dh1 + dh2 + dh3 + dh4`. Questo dà una forward secrecy leggermente più debole per l'handshake iniziale (si basa sul fatto che la signed prekey non sia compromessa), ma il Double Ratchet fornisce comunque forward secrecy per tutti i messaggi successivi. Il sistema compensa chiamando periodicamente `replenish_one_time_prekeys` per mantenere un pool di 100 OPK (`DEFAULT_ONE_TIME_PREKEY_COUNT = 100` in `config.py`).

### Passo 4: X3DH lato mittente

Questo è il cuore dell'accordo iniziale sulla chiave. Ecco le operazioni DH da `frontend/src/crypto/x3dh.ts`:

```python
dh1 = alice_ik_private.exchange(bob_spk_public)
dh2 = alice_ek_private.exchange(bob_ik_public)
dh3 = alice_ek_private.exchange(bob_spk_public)

used_one_time_prekey = False
if bob_bundle.one_time_prekey:
 bob_opk_public_bytes = base64url_to_bytes(bob_bundle.one_time_prekey)
 bob_opk_public = X25519PublicKey.from_public_bytes(bob_opk_public_bytes)
 dh4 = alice_ek_private.exchange(bob_opk_public)
 key_material = dh1 + dh2 + dh3 + dh4
 used_one_time_prekey = True
else:
 key_material = dh1 + dh2 + dh3

f = b'\xff' * X25519_KEY_SIZE
hkdf = HKDF(
 algorithm = hashes.SHA256,
 length = X25519_KEY_SIZE,
 salt = b'\x00' * X25519_KEY_SIZE,
 info = b'X3DH',
)
shared_key = hkdf.derive(f + key_material)
```

Prima che questo codice venga eseguito, il mittente verifica la firma della signed prekey in `frontend/src/crypto/x3dh.ts`. Se la verifica fallisce, viene sollevato un `ValueError` e lo scambio si interrompe. Questa è la protezione contro il MITM.

**Ogni operazione DH spiegata:**

- **dh1 = IK_A.exchange(SPK_B)** -- La identity key a lungo termine di Alice con la signed prekey di Bob. Questo fornisce autenticazione reciproca: solo la vera Alice e il vero Bob possono calcolare questo valore.
- **dh2 = EK_A.exchange(IK_B)** -- La ephemeral key di Alice con la identity key di Bob. Questo fornisce forward secrecy dal lato di Alice: la ephemeral key viene usata una volta e scartata.
- **dh3 = EK_A.exchange(SPK_B)** -- La ephemeral key di Alice con la signed prekey di Bob. Questo fornisce un mescolamento aggiuntivo. Anche se IK_A o IK_B fosse compromessa, questo output DH resta sconosciuto all'attaccante.
- **dh4 = EK_A.exchange(OPK_B)** (opzionale) -- La ephemeral key di Alice con la one-time prekey di Bob. Questo fornisce protezione dal replay e forward secrecy aggiuntiva, perché la OPK viene usata esattamente una volta.

**Perché il padding 0xFF?** La riga `f = b'\xff' * X25519_KEY_SIZE` antepone 32 byte di `0xFF` al materiale di chiave passato a HKDF. Questo deriva direttamente dalla specifica di Signal. Lo scopo è garantire che l'input di HKDF non sia mai composto da soli zeri. Se tutti e quattro gli output DH fossero per caso zero (il che indicherebbe un fallimento catastrofico, come qualcuno che sostituisce il punto identità), il padding `0xFF` garantisce che l'input di HKDF abbia comunque un'entropia elevata. È una difesa "cintura e bretelle".

**Perché un salt di zeri?** La specifica di Signal definisce il salt come 32 byte di `0x00`. HKDF richiede un salt, e usare tutti zeri equivale a non usare alcun salt (HKDF tratta comunque un salt di lunghezza zero come una stringa di zeri). Il salt è fisso anziché casuale perché entrambi i lati devono derivare la stessa chiave senza comunicarsi il salt. Se usassi un salt casuale, dovresti trasmetterlo, il che aggiunge complessità e dimensione del messaggio senza alcun beneficio di sicurezza in questo caso (gli output DH forniscono già la casualità).

**Perché info="X3DH"?** Il parametro `info` di HKDF è una stringa di separazione di dominio. Garantisce che la chiave derivata sia legata a questo specifico protocollo. Se lo stesso materiale di chiave venisse accidentalmente riutilizzato in un contesto diverso (per esempio, un handshake TLS), la stringa info produrrebbe una chiave derivata diversa. Questo previene gli attacchi cross-protocol.

**Associated data: IK_A || IK_B.** Dopo aver derivato la chiave condivisa, il codice calcola `associated_data = alice_ik_public_bytes + bob_ik_public_bytes` in `frontend/src/crypto/x3dh.ts`. Questi associated data vengono poi passati ad AES-GCM come AAD (Additional Authenticated Data). Legano i messaggi cifrati alla specifica coppia di identity key che ha eseguito l'handshake. Se un attaccante prova a reindirizzare i messaggi tra utenti diversi, il controllo dell'AAD fallirà durante la decifratura.

### Passo 5: X3DH lato ricevente

La logica lato ricevente vive in `frontend/src/crypto/x3dh.ts`. Le operazioni sono speculari:

```python
dh1 = bob_spk_private.exchange(alice_ik_public) # mirrors alice_ik.exchange(bob_spk)
dh2 = bob_ik_private.exchange(alice_ek_public) # mirrors alice_ek.exchange(bob_ik)
dh3 = bob_spk_private.exchange(alice_ek_public) # mirrors alice_ek.exchange(bob_spk)
dh4 = bob_opk_private.exchange(alice_ek_public) # mirrors alice_ek.exchange(bob_opk)
```

Questo funziona per come è definito Diffie-Hellman: `A_private.exchange(B_public) == B_private.exchange(A_public)`. La matematica è commutativa. Entrambi i lati calcolano gli stessi quattro output DH da 32 byte, li concatenano nello stesso ordine e derivano la stessa chiave condivisa tramite gli stessi parametri HKDF (salt di zeri, info `X3DH`, padding `0xFF`).

Il ricevente costruisce anche `associated_data = alice_ik_public_bytes + bob_ik_public_bytes` nello stesso ordine (prima la IK di Alice, poi la IK di Bob) in `frontend/src/crypto/x3dh.ts`. L'ordine conta. Se Alice mettesse la sua per prima sul lato mittente ma Bob invertisse l'ordine sul lato ricevente, gli associated data non corrisponderebbero e la decifratura fallirebbe.

### Errori comuni

**SBAGLIATO: usare un salt casuale in HKDF**

```python
# WRONG - breaks compatibility between sender and receiver
salt = os.urandom(32)
hkdf = HKDF(algorithm=hashes.SHA256, length=32, salt=salt, info=b'X3DH')
```

Se usi un salt casuale, mittente e ricevente deriveranno chiavi condivise diverse. Il mittente dovrebbe trasmettere il salt insieme alla ephemeral key, e a quel punto avresti aggiunto complessità senza alcun guadagno di sicurezza, dato che gli output DH forniscono già la casualità.

**SBAGLIATO: dimenticare di verificare la firma della signed prekey**

```python
# WRONG - allows MITM to substitute their own SPK
def perform_x3dh_sender(self, alice_identity_private_x25519, bob_bundle, ...):
 # Skipped: self.verify_signed_prekey(...)
 alice_ik_private_bytes = base64url_to_bytes(alice_identity_private_x25519)
 ...
```

Senza la verifica della firma, un attaccante che controlla la rete può sostituire la chiave pubblica SPK di Bob con la propria. Alice calcolerebbe un segreto condiviso con l'attaccante. L'attaccante esegue poi X3DH con Bob usando la vera SPK di Bob, e inoltra i messaggi tra loro. Né Alice né Bob si accorgono dell'intercettazione. La correzione è in `frontend/src/crypto/x3dh.ts`: verificare prima di eseguire lo scambio.

**SBAGLIATO: riutilizzare le one-time prekey**

```python
# WRONG - breaks forward secrecy for initial handshake
opk = get_opk_for_user(user_id)
# forgot to mark as used
# another conversation uses the same OPK
```

Se due conversazioni usano la stessa OPK, un attaccante che in seguito compromette la chiave privata della OPK di Bob può calcolare l'output DH4 per entrambe le conversazioni. Questo vanifica lo scopo delle chiavi monouso. La correzione è in `backend/app/services/prekey_service.py`: `one_time_prekey.is_used = True` immediatamente al momento del recupero, seguito da `session.commit`.

### Implementazione X3DH del frontend

Il frontend rispecchia l'X3DH del backend in TypeScript usando la WebCrypto API. La logica del mittente lato client vive in `x3dh.ts`:

```typescript
export async function initiateX3DH(
 identityKeyPair: IdentityKeyPair,
 recipientBundle: PreKeyBundle
): Promise<X3DHResult> {
 const signatureValid = await verifySignedPreKey(
 recipientBundle.identity_key_ed25519,
 recipientBundle.signed_prekey,
 recipientBundle.signed_prekey_signature
 )
 if (!signatureValid) {
 throw new Error("Invalid signed prekey signature")
 }
 ...
 const dh1 = await x25519DeriveSharedSecret(senderIdentityPrivate, recipientSignedPreKeyPublic)
 const dh2 = await x25519DeriveSharedSecret(ephemeralKeyPair.privateKey, recipientIdentityPublic)
 const dh3 = await x25519DeriveSharedSecret(ephemeralKeyPair.privateKey, recipientSignedPreKeyPublic)
 ...
 const sharedKey = await hkdfDerive(concatenated, EMPTY_SALT, X3DH_INFO, 32)
```

La derivazione HKDF usata qui proviene da `frontend/src/crypto/primitives.ts`:

```typescript
export async function hkdfDerive(
 inputKeyMaterial: Uint8Array,
 salt: Uint8Array,
 info: Uint8Array,
 outputLength: number = HKDF_OUTPUT_SIZE
): Promise<Uint8Array> {
 const baseKey = await subtle.importKey(
 "raw", inputKeyMaterial.buffer as ArrayBuffer,
 { name: "HKDF" }, false, ["deriveBits"]
 )
 const derivedBits = await subtle.deriveBits(
 { name: "HKDF", hash: "SHA-256",
 salt: salt.buffer as ArrayBuffer,
 info: info.buffer as ArrayBuffer },
 baseKey, outputLength * 8
 )
 return new Uint8Array(derivedBits)
}
```

C'è una differenza importante tra le chiamate HKDF del backend e del frontend. Il backend antepone `0xFF * 32` al materiale di chiave prima di chiamare HKDF (seguendo alla lettera la specifica di Signal). Il frontend passa direttamente gli output DH concatenati. Entrambi producono una chiave condivisa da 32 byte, ma la differenza di padding significa che le implementazioni X3DH del backend e del frontend non sono intercambiabili per la stessa conversazione. Questo è voluto: nel modello E2E, il frontend esegue X3DH interamente lato client, e l'X3DH del backend è usato solo per il percorso di inizializzazione delle chiavi assistito dal server (che è un fallback deprecato). Quando la cifratura lato client è attiva, gira solo il codice `x3dh.ts` del frontend.

`subtle.deriveBits` della WebCrypto API con l'algoritmo `HKDF` gestisce internamente i passi di extract-then-expand. Fornisci l'input key material, un salt, una stringa info e la lunghezza di output desiderata in bit (non in byte, ed è per questo che compare `outputLength * 8`). La chiamata `importKey` con `{ name: "HKDF" }` crea un oggetto chiave non estraibile adatto solo alla derivazione, il che impedisce che il materiale di chiave grezzo venga accidentalmente esportato o usato in modo improprio.

L'X3DH lato ricevente in `x3dh.ts` rispecchia il mittente con gli stessi scambi di chiavi della versione Python. La proprietà commutativa di ECDH garantisce che entrambi i lati derivino la stessa chiave condivisa da 32 byte.

---

## Costruire il Double Ratchet

Il Double Ratchet fornisce forward secrecy continua e recupero dopo un'intrusione dopo l'handshake X3DH iniziale. "Forward secrecy" significa che compromettere la chiave attuale non rivela i messaggi passati. "Recupero dopo un'intrusione" (break-in recovery) significa che anche se un attaccante ruba le chiavi attuali, i messaggi futuri diventano illeggibili non appena avvengono nuovi passi del ratchet DH.

### Inizializzazione

Il mittente si inizializza tramite `initializeRatchetSender` in `frontend/src/crypto/double-ratchet.ts`:

```ts
export async function initializeRatchetSender(
 peerId: string,
 sharedKey: Uint8Array,
 peerPublicKey: Uint8Array,
): Promise<DoubleRatchetState> {
 const dhKeyPair = await generateX25519KeyPair
 const dhPublicKey = await exportPublicKey(dhKeyPair.publicKey)
 const peerKey = await importX25519PublicKey(peerPublicKey)
 const dhOutput = await x25519DeriveSharedSecret(dhKeyPair.privateKey, peerKey)

 // KDF_RK per spec: HKDF(salt = previous_root_key, IKM = dh_output, info = "DoubleRatchet")
 const derivedKeys = await hkdfDerive(dhOutput, sharedKey, RATCHET_INFO, 64)
 const rootKey = derivedKeys.slice(0, 32)
 const sendingChainKey = derivedKeys.slice(32, 64)

 return { peer_id: peerId, root_key: rootKey, sending_chain_key: sendingChainKey, /* ... */ }
}
```

Il ricevente si inizializza tramite `initializeRatchetReceiver`:

```ts
export async function initializeRatchetReceiver(
 peerId: string,
 sharedKey: Uint8Array,
 dhKeyPair: CryptoKeyPair,
): Promise<DoubleRatchetState> {
 const dhPublicKey = await exportPublicKey(dhKeyPair.publicKey)
 return {
 peer_id: peerId,
 root_key: sharedKey,
 sending_chain_key: new Uint8Array(0),
 receiving_chain_key: null,
 dh_private_key: dhKeyPair,
 dh_public_key: dhPublicKey,
 dh_peer_public_key: null,
 /* ... */
 }
}
```

**Perché il mittente esegue un passo DH in più?** Il mittente genera una nuova coppia di chiavi DH ed esegue immediatamente uno scambio DH con la chiave pubblica SPK di Bob. Questo produce un `dh_output` che viene passato a `_kdf_rk` insieme alla `shared_key` di X3DH per derivare la prima `root_key` e la prima `sending_chain_key`. Questo passo in più significa che il primo messaggio ha già un contributo del ratchet DH oltre all'output di X3DH. Il ricevente, al contrario, non esegue questo passo durante l'inizializzazione. Memorizza semplicemente la chiave condivisa di X3DH come root key e attende il primo messaggio, che porterà la nuova chiave pubblica DH del mittente e innescherà il primo passo del ratchet del ricevente.

### Operazioni delle catene KDF

La derivazione della root key chain in `frontend/src/crypto/double-ratchet.ts`:

```python
def _kdf_rk(self, root_key: bytes, dh_output: bytes) -> tuple[bytes, bytes]:
 hkdf = HKDF(
 algorithm = hashes.SHA256,
 length = HKDF_OUTPUT_SIZE * 2,
 salt = root_key,
 info = b'',
 )
 output = hkdf.derive(dh_output)
 new_root_key = output[: HKDF_OUTPUT_SIZE]
 new_chain_key = output[HKDF_OUTPUT_SIZE :]
 return new_root_key, new_chain_key
```

`_kdf_rk` è la KDF della root chain. Viene eseguita ogni volta che avviene un passo del ratchet DH (quando il mittente o il ricevente genera una nuova coppia di chiavi DH). La `root_key` è usata come salt HKDF, e il nuovo output DH è l'input key material. HKDF produce 64 byte (`HKDF_OUTPUT_SIZE * 2 = 64`), che vengono divisi a metà: i primi 32 byte diventano la nuova root key, e i secondi 32 byte diventano la nuova chain key. La root key non cifra mai direttamente nulla. Agisce come una "master key" che fa da seme a ogni nuova catena. Poiché la root key viene aggiornata con un nuovo output DH a ogni passo del ratchet, anche se un attaccante compromette la root key attuale, il passo successivo del ratchet incorporerà un nuovo output DH che l'attaccante non conosce (assumendo che la chiave privata DH sia sicura), e la root key sarà "guarita". Questa è la proprietà di recupero dopo un'intrusione (break-in recovery).

Il parametro `info` vuoto è intenzionale. La specifica di Signal non usa una stringa info per la KDF della root chain. La separazione di dominio avviene implicitamente perché l'HKDF della root chain usa sempre la root key come salt, mentre l'HKDF di X3DH usa un salt di zeri. Salt diversi producono output diversi anche con lo stesso input, quindi non c'è contaminazione incrociata.

La derivazione della chain key in `frontend/src/crypto/double-ratchet.ts`:

```python
def _kdf_ck(self, chain_key: bytes) -> tuple[bytes, bytes]:
 h_chain = hmac.HMAC(chain_key, hashes.SHA256)
 h_chain.update(b'\x01')
 next_chain_key = h_chain.finalize

 h_message = hmac.HMAC(chain_key, hashes.SHA256)
 h_message.update(b'\x02')
 message_key = h_message.finalize

 return next_chain_key, message_key
```

**Perché HMAC con 0x01 e 0x02?** La chain key deve produrre due output: la chain key successiva e la message key. Usare HMAC con costanti diverse a singolo byte è il modo più semplice per derivare due valori indipendenti da un unico input. `0x01` produce la chain key successiva, `0x02` produce la message key. Le costanti sono arbitrarie ma fissate per convenzione (la specifica di Signal usa esattamente questi valori). La cosa importante è che siano diverse, così che i due output HMAC siano crittograficamente indipendenti.

**Perché due HMAC separati?** Nella libreria Python `cryptography` non puoi riutilizzare un oggetto HMAC dopo aver chiamato `finalize`. Ogni calcolo HMAC è una nuova istanza. Anche se potessi riutilizzarlo, produrre due output diversi dalla stessa chiave richiede due input diversi, il che significa due operazioni.

**Cos'è la proprietà unidirezionale?** Dato `next_chain_key = HMAC(chain_key, 0x01)`, non puoi calcolare `chain_key` a partire da `next_chain_key`. HMAC è una funzione unidirezionale. Questo significa che se un attaccante ottiene una message key per il messaggio N, non può derivare la chain key che l'ha prodotta, e quindi non può derivare le message key per i messaggi da 0 a N-1. Questa è la forward secrecy all'interno di una singola catena.

### Cifratura dei messaggi

Ecco il percorso di cifratura da `frontend/src/crypto/double-ratchet.ts`:

```python
def encrypt_message(self, state, plaintext, associated_data):
 state.sending_chain_key, message_key = self._kdf_ck(state.sending_chain_key)
 nonce, ciphertext = self._encrypt_with_message_key(message_key, plaintext, associated_data)

 if state.dh_private_key:
 dh_public = state.dh_private_key.public_key
 dh_public_bytes = dh_public.public_bytes(
 encoding = serialization.Encoding.Raw,
 format = serialization.PublicFormat.Raw
 )
 else:
 dh_public_bytes = b'\x00' * X25519_KEY_SIZE

 encrypted_msg = EncryptedMessage(
 ciphertext = ciphertext,
 nonce = nonce,
 dh_public_key = dh_public_bytes,
 message_number = state.sending_message_number,
 previous_chain_length = state.previous_sending_chain_length
 )
 state.sending_message_number += 1
 return encrypted_msg
```

E la cifratura AES-GCM sottostante in `frontend/src/crypto/double-ratchet.ts`:

```python
def _encrypt_with_message_key(self, message_key, plaintext, associated_data):
 aesgcm = AESGCM(message_key)
 nonce = os.urandom(AES_GCM_NONCE_SIZE)
 ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data)
 return nonce, ciphertext
```

**Perché includere dh_public_key nell'header?** Il destinatario ha bisogno della chiave pubblica DH attuale del mittente per determinare se eseguire un passo del ratchet DH. Se la chiave pubblica DH nell'header del messaggio è diversa dall'ultima chiave DH del peer conosciuta, il destinatario sa che il mittente ha fatto avanzare il ratchet, e deve farlo anch'esso per derivare la corretta chain key di ricezione. Senza questo campo dell'header, il destinatario non avrebbe modo di sapere quale catena usare.

**Perché tenere traccia di message_number?** I messaggi possono arrivare fuori ordine via WebSocket. Se il destinatario riceve il messaggio 5 prima del messaggio 3, deve sapere che i messaggi 3 e 4 sono stati saltati. Il `message_number` dice al destinatario esattamente quale posizione nella catena ha prodotto la chiave di questo messaggio. Il `previous_chain_length` dice al destinatario quanti messaggi sono stati inviati sulla catena precedente prima che il mittente facesse avanzare il ratchet, il che serve per calcolare le chiavi saltate della vecchia catena.

### Decifratura dei messaggi

Il percorso di decifratura in `frontend/src/crypto/double-ratchet.ts` gestisce tre casi distinti:

```python
def decrypt_message(self, state, encrypted_msg, associated_data):
 # Case 1: Check if we already stored this message's key (out-of-order)
 skipped_key = self._try_skipped_message_key(
 state, encrypted_msg.dh_public_key, encrypted_msg.message_number
 )
 if skipped_key:
 return self._decrypt_with_message_key(
 skipped_key, encrypted_msg.nonce, encrypted_msg.ciphertext, associated_data
 )

 # Case 2: New DH ratchet key means sender ratcheted
 if encrypted_msg.dh_public_key != state.dh_peer_public_key:
 if state.dh_peer_public_key:
 self._store_skipped_message_keys(
 state, encrypted_msg.previous_chain_length, state.dh_peer_public_key
 )
 self._dh_ratchet_receive(state, encrypted_msg.dh_public_key)

 # Case 3: Message number ahead of current position means skipped messages
 if encrypted_msg.message_number > state.receiving_message_number:
 self._store_skipped_message_keys(
 state, encrypted_msg.message_number, encrypted_msg.dh_public_key
 )

 # Advance chain and decrypt
 state.receiving_chain_key, message_key = self._kdf_ck(state.receiving_chain_key)
 state.receiving_message_number += 1

 plaintext = self._decrypt_with_message_key(
 message_key, encrypted_msg.nonce, encrypted_msg.ciphertext, associated_data
 )
 return plaintext
```

**Caso 1: chiave saltata trovata.** La message key era già stata calcolata e memorizzata durante una precedente operazione di salto. Estraila dal dict, decifra, fatto. Questo gestisce i messaggi saltati in precedenza che arrivano in ritardo.

**Caso 2: nuovo passo del ratchet.** La chiave pubblica DH nell'header del messaggio non corrisponde all'ultima chiave pubblica del peer conosciuta. Questo significa che il mittente ha eseguito un passo del ratchet DH. Il ricevente deve prima memorizzare le chiavi saltate per gli eventuali messaggi rimanenti sulla vecchia catena (fino a `previous_chain_length`), poi eseguire il proprio passo del ratchet DH per derivare la nuova chain key di ricezione.

**Caso 3: stessa catena, messaggi saltati.** Il numero del messaggio è più avanti della posizione attuale sulla stessa catena. Il ricevente deve far avanzare la chain key, memorizzando ogni message key intermedia, finché non raggiunge la posizione corretta.

### Gestione dei messaggi fuori ordine

Il meccanismo di salto in `frontend/src/crypto/double-ratchet.ts`:

```python
def _store_skipped_message_keys(self, state, until_message_number, dh_public_key):
 num_to_skip = until_message_number - state.receiving_message_number

 if num_to_skip > self.max_skip:
 raise ValueError(
 f"Cannot skip {num_to_skip} messages "
 f"(MAX_SKIP={self.max_skip})"
 )

 if len(state.skipped_message_keys) + num_to_skip > self.max_cache:
 self._evict_oldest_skipped_keys(state, num_to_skip)

 chain_key = state.receiving_chain_key
 for msg_num in range(state.receiving_message_number, until_message_number):
 chain_key, message_key = self._kdf_ck(chain_key)
 state.skipped_message_keys[(dh_public_key, msg_num)] = message_key

 state.receiving_chain_key = chain_key
```

**MAX_SKIP = 1000** (`config.py`). Se un attaccante invia un messaggio che dichiara `message_number = 999999999`, il ricevente dovrebbe calcolare 999999999 operazioni HMAC per derivare tutte le chiavi intermedie. Questo è un attacco denial-of-service. Il limite di 1000 significa che il ricevente rifiuterà qualsiasi messaggio che richieda di saltare più di 1000 posizioni. In pratica, anche una consegna fuori ordine aggressiva raramente supera qualche decina di messaggi saltati.

**MAX_CACHE = 2000** (`config.py`). Anche con il limite di salto, un attaccante determinato potrebbe accumulare lentamente chiavi saltate inviando molti piccoli buchi. Il limite della cache impedisce una crescita illimitata della memoria. Quando la cache è piena, `_evict_oldest_skipped_keys` rimuove le voci più vecchie per fare spazio. Le voci più vecchie sono quelle con più probabilità di essere davvero perse (se un messaggio non è arrivato dopo che sono state memorizzate altre 2000 chiavi saltate, probabilmente non arriverà mai).

### Errori comuni

**Non controllare il max skip (DoS tramite enormi buchi nei numeri dei messaggi).** Senza il controllo `num_to_skip > self.max_skip`, un attaccante può costringere il ricevente a calcolare milioni di operazioni HMAC inviando un singolo messaggio con un numero di messaggio elevato. Il costo in CPU è lineare rispetto alla dimensione del buco.

**Non rimuovere le vecchie chiavi saltate (esaurimento della memoria).** Senza rimozione, il dict `skipped_message_keys` cresce senza limiti. Un attaccante può causare un OOM inviando molti messaggi con numeri di messaggio crescenti ma non sequenziali su più passi del ratchet.

**Riutilizzare i nonce (catastrofico per AES-GCM).** AES-GCM usa un nonce da 12 byte. Se lo stesso nonce viene usato due volte con la stessa chiave, un attaccante può fare lo XOR dei due ciphertext per eliminare il keystream e recuperare entrambi i testi in chiaro. Il codice usa `os.urandom(AES_GCM_NONCE_SIZE)` in `frontend/src/crypto/double-ratchet.ts` per avere ogni volta un nuovo nonce casuale. Poiché ogni message key viene usata esattamente una volta (ogni chain key avanza dopo la derivazione), e ogni message key riceve un nonce casuale, la probabilità di una collisione è trascurabile. Ma se qualcuno modificasse il codice per riutilizzare le message key, il nonce casuale sarebbe l'unica difesa, e con 12 byte le collisioni per il paradosso del compleanno diventano probabili dopo circa 2^48 messaggi con la stessa chiave.

---

## Costruire l'autenticazione WebAuthn

### Flusso di registrazione

La registrazione è gestita in `backend/app/core/passkey/passkey_manager.py`. Il server genera le opzioni di registrazione:

```python
def generate_registration_options(
 self, user_id, username, display_name, exclude_credentials
):
 challenge = secrets.token_bytes(WEBAUTHN_CHALLENGE_BYTES)
 ...
 options = generate_registration_options(
 rp_id = self.rp_id,
 rp_name = self.rp_name,
 user_id = user_id,
 user_name = username,
 challenge = challenge,
 attestation = AttestationConveyancePreference.NONE,
 authenticator_selection = AuthenticatorSelectionCriteria(
 resident_key = ResidentKeyRequirement.REQUIRED,
 user_verification = UserVerificationRequirement.PREFERRED,
 ),
 exclude_credentials = exclude_creds,
 )
```

**ResidentKeyRequirement.REQUIRED** significa che l'authenticator deve memorizzare la credenziale sul dispositivo stesso (una "credenziale discoverable" o "passkey"). Questo permette il login senza username: l'utente tocca semplicemente la propria chiave di sicurezza o usa la biometria, e l'authenticator presenta la credenziale senza che l'utente digiti uno username. Senza REQUIRED, il server dovrebbe fornire un suggerimento sull'ID della credenziale durante l'autenticazione, il che significa che l'utente deve prima fornire il proprio username.

**La generazione della challenge** usa `secrets.token_bytes(32)` in `backend/app/core/passkey/passkey_manager.py`. Il modulo `secrets` è l'interfaccia CSPRNG di Python, equivalente a `/dev/urandom`. La challenge deve essere imprevedibile per prevenire i replay attack. La challenge viene memorizzata in Redis con un TTL di 600 secondi (`WEBAUTHN_CHALLENGE_TTL_SECONDS = 600` in `config.py`), così scade prima che un attaccante possa realisticamente catturarla e riprodurla.

**La verifica dell'attestation** in `backend/app/core/passkey/passkey_manager.py` chiama `verify_registration_response` di py_webauthn. Questa verifica che la risposta dell'authenticator sia stata firmata correttamente, che l'RP ID corrisponda, che l'origine corrisponda e che la challenge corrisponda. Il controllo `expected_origin = self.rp_origin` in `backend/app/core/passkey/passkey_manager.py` è critico: garantisce che la registrazione sia avvenuta dal frontend legittimo, non da una pagina di phishing su un dominio diverso.

### Flusso di autenticazione

L'autenticazione segue lo stesso pattern challenge-response. Il server genera le opzioni in `backend/app/core/passkey/passkey_manager.py`, invia la challenge al client, l'authenticator del client firma la challenge con la propria chiave privata e il server verifica in `backend/app/core/passkey/passkey_manager.py`.

**Perché challenge-response?** Senza una challenge nuova, un attaccante che intercetta una risposta di autenticazione potrebbe riprodurla per sempre. La challenge garantisce che ogni autenticazione sia unica. L'authenticator firma la challenge (insieme ad altri dati come l'RP ID e l'origine) con la propria chiave privata. Il server verifica la firma usando la chiave pubblica memorizzata. Anche se la risposta firmata viene intercettata, non può essere riutilizzata perché l'autenticazione successiva avrà una challenge diversa.

**Perché controllare l'origine?** Il controllo dell'origine (`expected_origin = self.rp_origin`) in `backend/app/core/passkey/passkey_manager.py` previene il phishing delle credenziali. Se un attaccante allestisce un sito falso su `evil-chat.com` e induce l'utente ad autenticarsi, l'authenticator includerà `evil-chat.com` come origine nei dati firmati. Quando l'attaccante prova a inoltrare quella risposta firmata al server reale, il controllo dell'origine fallisce perché il server si aspetta `http://localhost:3000` (o qualunque sia l'origine di produzione). L'authenticator lega la firma all'origine, rendendo il phishing strutturalmente impossibile.

### Rilevamento dei cloni

Da `backend/app/core/passkey/passkey_manager.py`:

```python
if (credential_current_sign_count != 0 and new_sign_count != 0
 and new_sign_count <= credential_current_sign_count):
 logger.error(
 "Signature counter did not increase: current=%s, new=%s. Possible cloned authenticator detected!",
 credential_current_sign_count,
 new_sign_count
 )
 raise ValueError(
 "Signature counter anomaly detected - potential cloned authenticator"
 )
```

**Cosa significa clonazione.** Se qualcuno copia fisicamente una chiave di sicurezza hardware (o estrae la chiave privata da un authenticator software), sia l'originale sia il clone hanno la stessa credenziale. Ciascuno ha il proprio contatore di firma che parte dallo stesso valore. Quando l'originale si autentica, il suo contatore si incrementa a N+1. Quando il clone si autentica, anche il suo contatore si incrementa, ma a partire da N, producendo N+1 (o un valore diverso a seconda di quando è stato usato ciascuno). Se il server vede un contatore minore o uguale all'ultimo contatore registrato, sa che due dispositivi stanno usando la stessa credenziale.

**Perché il contatore deve aumentare.** Ogni autenticazione incrementa il contatore di almeno 1. Se il server aveva memorizzato counter=5 e l'autenticazione successiva riporta counter=4, qualcosa non va. O il contatore è tornato indietro (impossibile in un'implementazione corretta) o viene usato un dispositivo diverso con un valore del contatore più vecchio.

**Caso limite con i contatori a 0.** Alcuni authenticator (in particolare gli authenticator di piattaforma come Touch ID o Windows Hello) riportano sempre counter=0 e non lo incrementano mai. La condizione `credential_current_sign_count != 0 and new_sign_count != 0` gestisce questo caso: se uno dei due contatori è 0, il controllo viene saltato completamente. Questo significa che il rilevamento dei cloni non è disponibile per quegli authenticator, ma rifiutarli escluderebbe una larga parte degli utenti. Il compromesso è accettabile perché gli authenticator di piattaforma sono intrinsecamente più difficili da clonare (sono legati al secure enclave del dispositivo).

---

## Costruire il livello WebSocket

### Gestione delle connessioni

Da `websocket_manager.py`:

```python
async def connect(self, websocket, user_id):
 await websocket.accept
 if user_id not in self.active_connections:
 self.active_connections[user_id] = []
 if len(self.active_connections[user_id]) >= WS_MAX_CONNECTIONS_PER_USER:
 await self._send_error(websocket, "max_connections", ...)
 await websocket.close
 return False
 self.active_connections[user_id].append(websocket)
 await presence_service.set_user_online(user_id)
 self.heartbeat_tasks[user_id] = asyncio.create_task(self._heartbeat_loop(websocket, user_id))
 await self._subscribe_to_messages(user_id)
 return True
```

**Perché liste di connessioni per utente?** Un utente potrebbe avere la chat aperta contemporaneamente sul telefono e sul laptop. Ogni dispositivo crea la propria connessione WebSocket. Il dict `active_connections` mappa `UUID -> list[WebSocket]`, così quando arriva un messaggio per un utente, viene inviato a tutte le sue connessioni attive. Senza questo, il supporto multi-dispositivo non funzionerebbe.

**Perché massimo 5?** `WS_MAX_CONNECTIONS_PER_USER = 5` (da `config.py`). Senza un limite, un client malevolo potrebbe aprire migliaia di connessioni WebSocket ed esaurire la memoria del server. Cinque è abbastanza generoso per un uso multi-dispositivo legittimo (telefono, tablet, laptop, desktop, magari uno in più) pur limitando il consumo di risorse. Il numero specifico è configurabile tramite variabile d'ambiente.

**Perché l'heartbeat?** Le connessioni WebSocket possono morire silenziosamente. Un utente chiude lo schermo del laptop, il WiFi cade, o un gateway NAT manda in timeout la connessione. Lo stack TCP potrebbe non rilevare la connessione morta per minuti o ore. Il loop di heartbeat in `websocket_manager.py` invia un ping ogni `WS_HEARTBEAT_INTERVAL` (30 secondi per default). Se l'invio fallisce, significa che la connessione è morta, e `disconnect` la ripulisce. Questo mantiene accurato il dict `active_connections` e garantisce che lo stato di presenza rifletta la realtà.

### Sottoscrizione alle live query

Da `websocket_manager.py`:

```python
async def _subscribe_to_messages(self, user_id):
 def message_callback(update: LiveMessageUpdate):
 asyncio.create_task(self._handle_live_message(user_id, update))

 live_id = await surreal_db.live_messages_for_user(
 user_id = str(user_id),
 callback = message_callback
 )
 self.live_query_ids[user_id] = live_id
```

**SurrealDB spinge i nuovi messaggi al server, il server li inoltra al client. Nessun polling necessario.** SurrealDB ha una funzionalità nativa di live query. Quando registri una live query come `LIVE SELECT * FROM messages WHERE recipient_id = $user_id`, SurrealDB spinge ogni nuovo record corrispondente al callback in tempo reale. Il server non ha bisogno di interrogare periodicamente il database con un timer. Quando viene creato un nuovo messaggio in SurrealDB (da `message_service.store_encrypted_message`), la live query scatta, `_handle_live_message` in `websocket_manager.py` lo raccoglie, lo incapsula in uno schema di messaggio WebSocket e lo invia a tutte le connessioni attive di quell'utente.

Questo è più efficiente del polling perché la latenza è nulla (il callback scatta nell'istante in cui il record viene creato) e non ci sono query sprecate (nessun ciclo di polling a vuoto quando non ci sono messaggi in attesa).

### Pulizia delle connessioni morte

Il metodo `disconnect` in `websocket_manager.py` gestisce la pulizia:

```python
async def disconnect(self, websocket, user_id):
 if user_id in self.active_connections:
 if websocket in self.active_connections[user_id]:
 self.active_connections[user_id].remove(websocket)

 if not self.active_connections[user_id]:
 del self.active_connections[user_id]
 await presence_service.set_user_offline(user_id)
 # Kill live query
 # Cancel heartbeat task
```

Le connessioni morte vengono rilevate anche durante `send_message` in `websocket_manager.py`. Se l'invio a un WebSocket solleva un'eccezione, quel WebSocket viene aggiunto a una lista `dead_connections`, e ogni connessione morta viene ripulita tramite `disconnect` al termine del loop di invio. Questa pulizia lazy significa che il sistema si autoripara: anche se un heartbeat non rileva una connessione morta, il successivo tentativo di invio di un messaggio la intercetterà.

La pulizia imposta l'utente come offline e termina la live query solo quando viene rimossa l'ultima connessione di quell'utente. Se ha ancora un'altra scheda aperta, l'altra connessione continua a funzionare e l'utente resta online.

---

## Costruire la pipeline dei messaggi

### Memorizzazione lato server (passthrough)

Da `backend/app/services/message_service.py`:

```python
async def store_encrypted_message(
 self, session, sender_id, recipient_id, ciphertext, nonce, header, room_id
):
 ...
 surreal_message = {
 "sender_id": str(sender_id),
 "recipient_id": str(recipient_id),
 "room_id": room_id,
 "ciphertext": ciphertext,
 "nonce": nonce,
 "header": header,
 "sender_username": sender_user.username,
 ...
 }
 result = await surreal_db.create_message(surreal_message)
```

Il server memorizza i dati cifrati senza alcuna capacità di decifratura. I campi `ciphertext`, `nonce` e `header` sono stringhe codificate in base64 che arrivano dal client e vengono memorizzate così come sono. Il server non vede mai il testo in chiaro. Il server non vede mai le chiavi di cifratura. Il server potrebbe essere completamente compromesso e l'attaccante comunque non potrebbe leggere i messaggi, perché le message key esistono solo sui dispositivi dei client, in IndexedDB.

Questo è l'approccio E2E corretto. Molti sistemi di chat "cifrati" cifrano lato server, il che significa che il server ha le chiavi e potrebbe decifrare tutto. Qui, la cifratura avviene in `crypto-service.ts` nel browser, e il server è un semplice relay.

### Inizializzazione della conversazione

Da `backend/app/services/message_service.py`:

```python
async def initialize_conversation(self, session, sender_id, recipient_id):
 # Check for existing ratchet state
 # Fetch sender's identity key
 # Fetch recipient's prekey bundle
 # Perform X3DH sender-side
 x3dh_result = initiateX3DH (frontend/src/crypto/x3dh.ts)(
 alice_identity_private_x25519 = sender_ik.private_key,
 bob_bundle = recipient_bundle,
 bob_identity_public_ed25519 = recipient_ik.public_key_ed25519
 )
 # Initialize Double Ratchet with shared key
 dr_state = double_ratchet.initialize_sender(
 shared_key = x3dh_result.shared_key,
 peer_public_key = recipient_spk_public_bytes
 )
 # Serialize and persist ratchet state to PostgreSQL
```

Questo è il ponte tra lo scambio di chiavi e la cifratura continua. Il metodo orchestra l'intero flusso: verifica se esiste già una conversazione (idempotenza), recupera da PostgreSQL le chiavi di entrambi gli utenti, esegue lo scambio X3DH, inizializza il Double Ratchet con la chiave condivisa risultante, serializza lo stato del ratchet e lo rende persistente. Dopo il completamento di questo metodo, `encrypt_message` e `decrypt_message` possono operare usando lo stato del ratchet memorizzato.

Il controllo in `backend/app/services/message_service.py` è importante: se esiste già uno stato del ratchet per questa coppia mittente-destinatario, il metodo lo restituisce senza fare nulla. Questo impedisce una reinizializzazione accidentale, che genererebbe un nuovo segreto condiviso e romperebbe la conversazione esistente.

---

## Dettagli di implementazione della sicurezza

### Confronto a tempo costante

Da `frontend/src/crypto/primitives.ts`:

```typescript
export function constantTimeEqual(a: Uint8Array, b: Uint8Array): boolean {
 if (a.length !== b.length) return false
 let result = 0
 for (let i = 0; i < a.length; i++) {
 result |= a[i] ^ b[i]
 }
 return result === 0
}
```

**Perché XOR + OR invece di un confronto byte per byte?** Un confronto ingenuo come `a[0] === b[0] && a[1] === b[1] && ...` va in short-circuit alla prima discrepanza. Se i byte da 0 a 5 corrispondono ma il byte 6 no, il confronto restituisce `false` dopo aver controllato 7 byte. Un attaccante che misura il tempo di risposta può determinare che i primi 6 byte sono corretti e solo il byte 6 è sbagliato. Provando tutti i 256 valori per il byte 6 e misurando quale richiede un po' più di tempo (perché il confronto procede fino al byte 7), può indovinare il valore corretto. Ripetendo per ogni byte, l'intero segreto viene recuperato.

La versione a tempo costante fa lo XOR di ogni coppia di byte e accumula i risultati con OR. Se un qualsiasi byte differisce, almeno un bit in `result` sarà impostato. La funzione elabora sempre ogni byte indipendentemente da dove si trovi una discrepanza. Il tempo di esecuzione dipende solo dalla lunghezza dell'array, non dal contenuto.

Il controllo della lunghezza all'inizio fa trapelare se le lunghezze sono uguali. Questo è generalmente accettabile perché le lunghezze delle chiavi non sono segrete. Ma se la lunghezza fosse segreta, dovresti rendere a tempo costante anche quel confronto.

### Generazione di numeri casuali

Sul backend Python, `os.urandom(AES_GCM_NONCE_SIZE)` in `frontend/src/crypto/double-ratchet.ts` legge dal generatore di numeri casuali crittograficamente sicuro (CSPRNG) del sistema operativo. Su Linux, è supportato da `/dev/urandom`, che attinge entropia da eventi hardware (temporizzazione del disco, interrupt di rete, ecc.) e mantiene uno stato CSPRNG computazionalmente indistinguibile dalla vera casualità.

Sul frontend nel browser, la funzione `generateRandomBytes` in `frontend/src/crypto/primitives.ts`:

```typescript
export function generateRandomBytes(length: number): Uint8Array {
 const bytes = new Uint8Array(length)
 crypto.getRandomValues(bytes)
 return bytes
}
```

Questa chiama il CSPRNG della WebCrypto API. Ogni browser principale lo implementa usando sotto il cofano il CSPRNG del sistema operativo. Chrome usa il CSPRNG di BoringSSL, Firefox usa NSS, Safari usa CommonCrypto. Tutti in definitiva attingono dal pool di entropia del kernel.

La stessa funzione è usata per la generazione dei nonce in `aesGcmEncrypt` in `frontend/src/crypto/primitives.ts`: `const nonce = generateRandomBytes(AES_GCM_NONCE_SIZE)`. È usata anche per la generazione delle challenge WebAuthn lato server tramite `secrets.token_bytes(WEBAUTHN_CHALLENGE_BYTES)` in `backend/app/core/passkey/passkey_manager.py`, che è il wrapper di Python attorno a `os.urandom`.

**Perché un CSPRNG?** Nonce, challenge e chiavi effimere devono essere imprevedibili. Se un attaccante può prevedere il nonce successivo, può precalcolare il keystream per AES-GCM e decifrare i messaggi in tempo reale. Se può prevedere la chiave effimera successiva, può calcolare il segreto condiviso DH. Se può prevedere la challenge WebAuthn successiva, può prefirmare una risposta e riprodurla.

**Perché non Math.random?** `Math.random` usa un PRNG (generatore di numeri pseudocasuali) inizializzato da una sorgente a bassa entropia (spesso solo l'ora corrente o un semplice contatore). La maggior parte dei motori JavaScript usa xorshift128+ o un PRNG veloce simile. Lo stato interno è tipicamente di 128 bit, e può essere ricostruito osservando appena 4-5 output consecutivi. Un attaccante che conosce lo stato interno di `Math.random` può prevedere ogni output futuro, il che comprometterebbe ogni nonce e ogni chiave effimera generati dall'applicazione. Nel 2015, dei ricercatori hanno dimostrato il recupero dell'intero stato xorshift128+ dal `Math.random` di V8 osservando solo pochi output. `Math.random` va bene per mescolare un mazzo di carte in un'animazione della UI. È catastrofico per la crittografia.

### Serializzazione delle chiavi

Tutte le chiavi sono serializzate come byte raw codificati in base64url. Per esempio, in `frontend/src/crypto/x3dh.ts`:

```python
private_bytes = private_key.private_bytes(
 encoding = serialization.Encoding.Raw,
 format = serialization.PrivateFormat.Raw,
 encryption_algorithm = serialization.NoEncryption
)
...
return (
 bytes_to_base64url(private_bytes),
 bytes_to_base64url(public_bytes)
)
```

**Perché base64url invece dell'esadecimale?** La codifica esadecimale rappresenta ogni byte con due caratteri esadecimali, quindi 32 byte diventano 64 caratteri. Base64url rappresenta ogni 3 byte con 4 caratteri, quindi 32 byte diventano circa 43 caratteri. È un risparmio di spazio del 33%. Quando memorizzi migliaia di chiavi e le trasmetti via WebSocket, il risparmio si accumula.

**Perché il formato Raw invece di PEM?** Il formato PEM avvolge la chiave in header `-----BEGIN PRIVATE KEY-----`, codifica in base64 una struttura ASN.1 codificata in DER e aggiunge un a capo ogni 64 caratteri. Per una chiave privata X25519 da 32 byte, PEM produce circa 120 byte. Il formato Raw produce esattamente 32 byte. PEM è utile quando devi identificare l'algoritmo della chiave dalla codifica (l'OID ASN.1 ti dice "questa è X25519"). Qui, entrambi i lati conoscono già l'algoritmo, quindi i metadati sono uno spreco.

---

## Flusso dei dati: traccia completa di un messaggio

Ecco cosa succede quando Alice digita "Hello Bob" e il messaggio raggiunge lo schermo di Bob. Ogni passo, ogni file, ogni trasformazione.

**1. Alice digita "Hello Bob" in Chat.tsx.** Il componente `ChatInput` cattura il testo in un handler di submit del form. L'handler crea un ID di messaggio temporaneo (una stringa casuale per gli aggiornamenti ottimistici della UI) e richiama la logica di invio del messaggio. A questo punto il testo in chiaro esiste solo nell'heap JavaScript del browser.

**2. crypto-service.ts `encrypt("bob-uuid", "Hello Bob")`.** Viene invocato il metodo `CryptoService.encrypt` in `frontend/src/crypto/crypto-service.ts`. Per prima cosa verifica se esiste una sessione di ratchet con Bob.

**3. Caricamento dello stato del ratchet da IndexedDB.** `getRatchetState(peerId)` in `frontend/src/crypto/crypto-service.ts` controlla prima la `Map<string, DoubleRatchetState>` in memoria (indicizzata per ID del peer). Se lo stato non è in memoria, interroga IndexedDB tramite `key-store.ts`, deserializza il JSON memorizzato riportandolo a un oggetto `DoubleRatchetState` (ricostruendo gli oggetti CryptoKey dai byte codificati in base64) e lo mette in cache. Se non esiste alcuno stato, `establishSession(peerId)` avvia il flusso X3DH completo: recupera dal server il prekey bundle di Bob tramite `api.encryption.getPrekeyBundle`, esegue l'X3DH lato client tramite `x3dh.ts:initiateX3DH` e inizializza il ratchet di invio tramite `double-ratchet.ts:initializeRatchetSender`.

**4. double-ratchet.ts `encryptMessage`: avanzamento della chain key tramite HMAC.** In `double-ratchet.ts`, `deriveMessageKey(state.sending_chain_key)` calcola due valori:
 - `messageKey = HMAC-SHA-256(chainKey, "MessageKey")` - la chiave che cifrerà questo specifico messaggio
 - `nextChainKey = HMAC-SHA-256(chainKey, "ChainKey")` - la chain key per il messaggio successivo

La vecchia `sending_chain_key` viene immediatamente sovrascritta con `nextChainKey`. La `messageKey` viene usata una volta e mai memorizzata. Questo è il meccanismo della forward secrecy: dopo la cifratura, la message key non può essere recuperata dallo stato aggiornato della catena.

**5. primitives.ts `aesGcmEncrypt` con la message key.** In `double-ratchet.ts`, la `messageKey` da 32 byte viene importata come CryptoKey AES-GCM. I byte del `plaintext` (`TextEncoder.encode("Hello Bob")` = 9 byte) vengono cifrati. Un nonce casuale da 12 byte viene generato tramite `crypto.getRandomValues`. L'header del messaggio (chiave pubblica DH, numero del messaggio, lunghezza della catena precedente) viene serializzato in JSON e usato come Additional Authenticated Data (AAD). AES-GCM produce il ciphertext (9 byte di dati cifrati + 16 byte di authentication tag = 25 byte in totale).

**6. Restituzione di {ciphertext, nonce, header}.** L'oggetto `EncryptedMessage` in `double-ratchet.ts` contiene ciphertext e nonce come `Uint8Array` grezzi, più un oggetto tipizzato `MessageHeader`. Di ritorno in `frontend/src/crypto/crypto-service.ts`, ciphertext e nonce vengono codificati in base64 come stringhe, e l'header (incluso l'eventuale header X3DH in sospeso per i primi messaggi) viene convertito con JSON.stringify in un'unica stringa. Il risultato sono tre stringhe: `ciphertext`, `nonce` e `header`. Tutte e tre sono opache per chiunque non abbia la message key.

**7. Il WebSocket invia il payload JSON.** `websocket-manager.ts` chiama `sendEncryptedMessage`, che costruisce il messaggio in uscita:

```json
{
 "type": "encrypted_message",
 "recipient_id": "bob-uuid",
 "room_id": "room-uuid",
 "ciphertext": "base64...",
 "nonce": "base64...",
 "header": "{\"ratchet\":{\"dh_public_key\":\"...\",\"message_number\":0,...}}",
 "temp_id": "temp-abc123"
}
```

Questo JSON viene inviato tramite `ws.send(JSON.stringify(message))`. Se il WebSocket non è connesso, il messaggio viene accodato in `messageQueue` e inviato quando la connessione viene ripristinata.

**8. websocket.py riceve.** Il `websocket_endpoint` in `websocket.py` chiama `data = await websocket.receive_text`, interpreta il JSON con `json.loads(data)` e passa il dict risultante a `websocket_service.route_message`. A questo punto il server ha il blob cifrato. Non può decifrarlo. Non ha la message key, la chain key né alcuno stato del ratchet per la conversazione tra Alice e Bob (nel modello E2E).

**9. websocket_service.py instrada verso l'handler.** In `backend/app/services/websocket_service.py`, il tipo di messaggio `"encrypted_message"` viene smistato a `handle_encrypted_message` in `backend/app/services/websocket_service.py`. L'handler estrae `recipient_id`, `room_id`, `ciphertext`, `nonce` e `header` dal dict del messaggio.

**10. backend/app/services/message_service.py memorizza in SurrealDB.** L'handler chiama `message_service.store_encrypted_message`, che costruisce un documento SurrealDB:

```python
surreal_message = {
 "sender_id": str(sender_id),
 "recipient_id": str(recipient_id),
 "room_id": room_id,
 "ciphertext": ciphertext, # still base64, still encrypted
 "nonce": nonce, # still base64
 "header": header, # still JSON string
 "sender_username": sender_user.username,
 "created_at": now.isoformat,
 "updated_at": now.isoformat,
}
```

Il documento viene scritto in SurrealDB tramite `surreal_db.create_message(surreal_message)`. Il server memorizza il blob cifrato così com'è. Nessuna decifratura. Nessun accesso alle chiavi.

**11. La live query di SurrealDB scatta.** Nell'istante in cui il record del messaggio viene creato, SurrealDB spinge un evento `CREATE` a qualsiasi live query registrata che corrisponda a `recipient_id = bob_uuid`. È un push, non un pull. Non c'è alcun intervallo di polling. La latenza dalla scrittura al callback si misura in pochi millisecondi.

**12. websocket_manager.py raccoglie l'aggiornamento.** Il callback `_handle_live_message` in `websocket_manager.py` riceve il `LiveMessageUpdate`. Verifica che `update.action == "CREATE"` (ignorando aggiornamenti ed eliminazioni). Incapsula i dati del messaggio in uno schema Pydantic `EncryptedMessageWS` e chiama `self.send_message(user_id, ws_message.model_dump(mode="json"))`.

**13. ConnectionManager invia al/ai WebSocket di Bob.** In `websocket_manager.py`, il manager itera su tutte le connessioni attive di Bob (potrebbe avere l'app aperta sul telefono e sul laptop) e chiama `websocket.send_json(message)` su ciascuna. Se un invio fallisce, quella connessione viene aggiunta a una lista `dead_connections` e ripulita dopo il loop.

**14. Il websocket-manager.ts di Bob riceve.** Il metodo `handleMessage` in `websocket-manager.ts` interpreta il JSON da `event.data`. Il metodo `routeMessage` in `websocket-manager.ts` lo fa passare attraverso i type guard. `isEncryptedMessageWS(message)` corrisponde perché il messaggio ha `type: "encrypted_message"` e i campi richiesti. Il messaggio viene smistato a `handleWSMessage` in `message-handlers.ts`.

**15. crypto-service.ts `decrypt(alice_id, ciphertext, nonce, header)`.** L'handler dei messaggi chiama `CryptoService.decrypt` in `frontend/src/crypto/crypto-service.ts`. La stringa `header` viene interpretata per estrarre l'header del ratchet (chiave pubblica DH, numero del messaggio, lunghezza della catena precedente) ed eventualmente un header X3DH (se questo è il primo messaggio di una nuova conversazione).

**16. Caricamento dello stato del ratchet da IndexedDB.** Stesso meccanismo del passo 3, ma dal lato di Bob. Se non esiste alcuno stato del ratchet per Alice, e il messaggio include un header X3DH, `handleIncomingSession` in `frontend/src/crypto/crypto-service.ts` avvia:
 - La ricerca della identity key di Alice dall'header X3DH
 - La ricerca della chiave privata della signed prekey di Bob in IndexedDB
 - La ricerca della one-time prekey consumata, se ne è stata usata una
 - La chiamata a `receiveX3DH` da `x3dh.ts` per calcolare lo stesso segreto condiviso calcolato da Alice
 - La chiamata a `initializeRatchetReceiver` da `double-ratchet.ts` con il segreto condiviso e la coppia di chiavi della signed prekey di Bob

**17. Controllo delle chiavi saltate, avanzamento della catena di ricezione.** `decryptMessage` in `double-ratchet.ts` esegue la logica a tre casi:
 - Verifica se la chiave di questo messaggio era stata memorizzata in precedenza (hit nella cache delle chiavi saltate)
 - Verifica se la chiave pubblica DH di Alice nell'header è diversa dall'ultima chiave conosciuta (serve un passo del ratchet DH)
 - Verifica se il numero del messaggio è più avanti della posizione attuale (salta le chiavi intermedie)
 Poi deriva la message key: `messageKey = HMAC-SHA-256(receivingChainKey, "MessageKey")`.

**18. primitives.ts `aesGcmDecrypt`.** In `double-ratchet.ts` viene eseguita la decifratura AES-256-GCM. La message key da 32 byte viene importata come CryptoKey. Il nonce da 12 byte del messaggio viene usato come IV. Il JSON dell'header viene ricostruito e usato come AAD. `subtle.decrypt` verifica l'authentication tag da 16 byte e produce i 9 byte di testo in chiaro. Se il tag non corrisponde (messaggio manomesso, chiave sbagliata, nonce sbagliato, AAD sbagliato), la WebCrypto API lancia un `OperationError`.

**19. Restituzione di "Hello Bob".** I byte del testo in chiaro in `Uint8Array` vengono decodificati tramite `new TextDecoder.decode(plaintextBytes)` in `frontend/src/crypto/crypto-service.ts`, producendo la stringa `"Hello Bob"`. Lo stato del ratchet viene serializzato e salvato di nuovo in IndexedDB.

**20. Chat.tsx visualizza il testo in chiaro.** Il messaggio decifrato viene aggiunto allo store dei messaggi. Il componente reattivo SolidJS ridisegna la lista dei messaggi. Bob vede "Hello Bob" sullo schermo. L'intero viaggio dalla pressione del tasto di Alice allo schermo di Bob ha richiesto:
 - ~1ms per la cifratura lato client (HMAC + AES-GCM)
 - ~10-50ms per il round trip WebSocket (dipende dalla rete)
 - ~1ms per la scrittura in SurrealDB + l'attivazione della live query
 - ~10-50ms per la consegna WebSocket a Bob
 - ~1ms per la decifratura lato client
 Totale: all'incirca 25-100ms end to end.

Il testo in chiaro "Hello Bob" è esistito in memoria sul dispositivo di Alice e sul dispositivo di Bob. In nessun altro punto della pipeline, né sul server WebSocket, né in SurrealDB, né in PostgreSQL, né in Redis, è esistito il testo in chiaro. Questo è ciò che significa in pratica la crittografia end-to-end.

---

## Pattern di gestione degli errori

### Fallimenti della cifratura

Quando la cifratura fallisce (stato del ratchet corrotto, chiave non valida, ecc.), viene sollevato un `EncryptionError` in `backend/app/services/message_service.py`. La gerarchia di eccezioni in `exceptions.py` deriva da `AppException`. L'exception handler restituisce l'errore al client tramite il tipo di messaggio di errore WebSocket. Il messaggio non viene mai memorizzato, perché la cifratura è fallita prima della chiamata di memorizzazione.

### Fallimenti delle transazioni del database

`IntegrityError` viene catturato in ogni metodo di servizio che scrive nel database. Per esempio, in `backend/app/services/message_service.py`:

```python
except IntegrityError as e:
 await session.rollback
 raise DatabaseError("Failed to initialize conversation") from e
```

Il pattern è coerente: cattura l'errore di integrità di SQLAlchemy, esegui il rollback della sessione a uno stato pulito, poi solleva un `DatabaseError` personalizzato. Questo garantisce che il database non venga mai lasciato in uno stato parzialmente committato. La catena `from e` preserva l'eccezione originale per il logging.

### Disconnessione del WebSocket

Quando un WebSocket si disconnette (l'utente chiude la scheda, la rete cade), `WebSocketDisconnect` viene catturato in `websocket.py`. Il blocco `finally` in `websocket.py` esegue sempre `connection_manager.disconnect(websocket, user_uuid)`, che:
- Rimuove il WebSocket dal connection pool
- Se era l'ultima connessione di quell'utente: termina la live query SurrealDB, annulla il task di heartbeat e imposta la presenza su offline tramite Redis
- Se restano altre connessioni: rimuove solo questa e registra nei log il numero di connessioni rimanenti

---

## Strategia di test

### Unit test

**Testare l'accordo sulla chiave X3DH.** Genera le identity key per Alice e Bob. Genera una signed prekey per Bob. Genera una OPK per Bob. Assembla il prekey bundle di Bob. Esegui l'X3DH lato mittente come Alice tramite `initiateX3DH (frontend/src/crypto/x3dh.ts)`. Esegui l'X3DH lato ricevente come Bob tramite `receiveX3DH (frontend/src/crypto/x3dh.ts)`, passando la chiave pubblica effimera e la identity key di Alice dal risultato del mittente. Verifica con un'asserzione che `alice_result.shared_key == bob_result.shared_key`. Questo verifica la matematica commutativa del DH. Verifica anche che `alice_result.associated_data == bob_result.associated_data`, confermando che entrambi i lati calcolano lo stesso AAD.

Testa separatamente il caso senza OPK: imposta `bob_bundle.one_time_prekey = None`, passa `bob_one_time_prekey_private = None` sul lato ricevente. Verifica che le chiavi condivise corrispondano comunque (lo faranno, perché entrambi i lati ripiegano sulla variante a 3 DH).

Testa il fallimento della verifica della firma: corrompi un byte della firma della SPK, chiama `perform_x3dh_sender`, verifica che sollevi `ValueError("Invalid signed prekey signature")`.

**Testare il round-trip del Double Ratchet.** Inizializza mittente e ricevente con la chiave condivisa di X3DH. Cifra "message 1" sul lato mittente. Decifra sul lato ricevente. Verifica che il testo in chiaro corrisponda. Poi cifra "message 2" sul lato ricevente e decifra sul lato mittente. Questo testa un ciclo completo del ratchet in cui il ratchet DH avanza due volte (una per ogni cambio di direzione).

Continua con una sequenza più lunga: il mittente invia 5 messaggi di fila, il ricevente li decifra tutti e 5 (questo testa la catena simmetrica senza ratcheting DH), poi il ricevente invia 3 messaggi indietro, il mittente li decifra tutti e 3. Verifica ogni testo in chiaro.

**Testare la consegna fuori ordine.** Cifra i messaggi 0, 1, 2, 3, 4 sul mittente. Consegna per primo il messaggio 4. Il ricevente dovrebbe memorizzare le chiavi saltate per i messaggi 0, 1, 2, 3, decifrare con successo il messaggio 4, poi decifrare i messaggi 2, 0, 3, 1 in qualsiasi ordine usando le chiavi saltate memorizzate. Verifica che tutti e cinque i testi in chiaro siano corretti. Verifica che la cache delle chiavi saltate sia vuota dopo che tutti i messaggi sono stati decifrati (ogni chiave viene consumata all'uso).

**Testare l'applicazione del max skip.** Costruisci un `DoubleRatchet(max_skip=5)`. Cifra i messaggi da 0 a 20 sul mittente. Prova a consegnare per primo il messaggio 20 (che richiede 20 salti). Verifica che il ricevente sollevi `ValueError("Cannot skip 20 messages (MAX_SKIP=5)")`. Poi consegna il messaggio 4 (che richiede 4 salti). Verifica che abbia successo. Questo conferma che il limite viene applicato e che i piccoli buchi legittimi funzionano ancora.

**Testare la rimozione dalla cache.** Costruisci un `DoubleRatchet(max_skip=100, max_cache=10)`. Salta 10 messaggi per riempire la cache. Saltane altri 5. Verifica che la cache abbia ora 10 voci (5 di quelle vecchie sono state rimosse per fare spazio). Prova a decifrare uno dei messaggi rimossi. Verifica che la decifratura fallisca (la chiave non c'è più).

**Testare i flussi WebAuthn.** Simula con dei mock gli oggetti di risposta dell'authenticator come dizionari conformi alla specifica WebAuthn. Chiama `passkey_manager.generate_registration_options` con un ID utente e uno username di test. Verifica che le opzioni restituite includano l'RP ID, il nome dell'RP e la challenge corretti. Chiama `passkey_manager.verify_registration` con la risposta mock e la challenge. Verifica che la `VerifiedRegistration` restituita abbia l'ID della credenziale e il sign count corretti.

Per l'autenticazione, chiama `passkey_manager.generate_authentication_options`. Inserisci la challenge in una risposta di autenticazione mock. Chiama `passkey_manager.verify_authentication` con la chiave pubblica della credenziale memorizzata e un sign count attuale. Verifica che la `VerifiedAuthentication` restituita abbia un sign count incrementato.

Per il rilevamento dei cloni, chiama `verify_authentication` con `credential_current_sign_count=5` e una risposta mock in cui `new_sign_count=3`. Verifica che sollevi `ValueError("Signature counter anomaly detected")`.

### Test di integrazione

**Flusso completo dei messaggi (frontend).** Avvia due client puliti in `vitest`, esegui `generateIdentityKeyPair` / `generateSignedPreKey` / `generateOneTimePreKeys` per ciascuno, consegna ad Alice il bundle di Bob, esegui `initiateX3DH` e `initializeRatchetSender`, cifra un messaggio, poi dal lato di Bob esegui `receiveX3DH` + `initializeRatchetReceiver` e `decryptMessage`. Verifica che il testo in chiaro corrisponda. I file esistenti `frontend/src/crypto/x3dh.test.ts` e `frontend/src/crypto/double-ratchet.test.ts` coprono già questo scenario end-to-end.

**Ciclo di vita del WebSocket.** Avvia l'app di test FastAPI, registra due utenti tramite il flusso di autenticazione (che imposta un session cookie), apri due connessioni WebSocket che portano i cookie ed esercita il round-trip completo dei messaggi. Il WebSocket autenticato tramite cookie è l'unico percorso di autenticazione — non esiste più la query string `?user_id=`.

**Rollback del database in caso di fallimento.** Crea una sessione async. Avvia un'operazione che fallirà (per esempio, inserire una riga `IdentityKey` il cui `user_id` non esiste). Verifica che la sessione esegua un rollback pulito e sollevi `DatabaseError`. Verifica che nessun dato parziale sia stato committato interrogando la tabella in seguito.

**Rotazione delle chiavi.** Genera una nuova signed prekey sul client (`generateSignedPreKey`), chiama di nuovo `uploadKeys` con essa, poi recupera di nuovo il bundle come peer. Verifica che il bundle contenga la nuova SPK e che la sua firma sia valida rispetto alla identity key Ed25519 dell'utente. Conferma che la riga della SPK precedentemente attiva sia ora `is_active = False` in PostgreSQL.

**Esaurimento delle OPK.** Carica un utente con 2 OPK. Recupera il suo prekey bundle due volte (consumandole entrambe, marcandole `is_used = True`). Recuperalo una terza volta e verifica che il bundle venga restituito con `one_time_prekey = None`. Verifica che X3DH riesca comunque senza la OPK (il fallback a 3 DH).

---

## Insidie comuni di implementazione

### Insidia 1: riuso del nonce in AES-GCM

**Sintomo:** la decifratura riesce ma la verifica dell'authentication tag fallisce in modo intermittente (o peggio, la decifratura riesce e non ti accorgi della falla di sicurezza).

**Causa:** riutilizzare lo stesso nonce con la stessa chiave rompe completamente la sicurezza di GCM. AES-GCM usa il nonce per generare un keystream unico tramite la modalità CTR. Se due messaggi condividono lo stesso nonce e la stessa chiave, lo XOR dei due ciphertext annulla il keystream e produce lo XOR dei due testi in chiaro. Un attaccante può quindi usare l'analisi delle frequenze o attacchi known-plaintext per recuperare entrambi i messaggi. Inoltre, l'authentication tag diventa falsificabile.

**Correzione:** usa sempre `os.urandom(12)` per un nonce nuovo, come avviene in `frontend/src/crypto/double-ratchet.ts`. Poiché ogni message key è derivata da una posizione unica della chain key, il riuso del nonce tra message key diverse non è un problema. Il rischio si presenterebbe solo se la stessa message key venisse usata due volte, cosa che l'avanzamento della catena impedisce.

### Insidia 2: non verificare la signed prekey

**Sintomo:** X3DH riesce ma la comunicazione può essere intercettata da un man-in-the-middle.

**Causa:** saltare la verifica della firma permette a un attaccante di sostituire la SPK nel prekey bundle di Bob con la propria. Alice calcola un segreto condiviso con l'attaccante. L'attaccante calcola in modo indipendente un segreto condiviso con Bob usando la vera SPK di Bob. L'attaccante può ora leggere e modificare tutti i messaggi.

**Correzione:** verifica sempre prima dello scambio, come avviene in `frontend/src/crypto/x3dh.ts`:

```python
if not self.verify_signed_prekey(bob_bundle.signed_prekey,
 bob_bundle.signed_prekey_signature,
 bob_identity_public_ed25519):
 raise ValueError("Invalid signed prekey signature")
```

### Insidia 3: memorizzare le chiavi private sul server

**Sintomo:** la compromissione del server espone tutte le conversazioni.

**Causa:** se il materiale delle chiavi private è accessibile al server, chiunque comprometta il server (SQL injection, accesso SSH, leak di un backup) può decifrare ogni messaggio mai inviato.

**Correzione:** generazione delle chiavi lato client e archiviazione in IndexedDB. Il server memorizza solo chiavi pubbliche. Guarda `backend/app/services/prekey_service.py`: il metodo `store_client_keys` riceve dal client solo chiavi pubbliche. I campi `private_key` dei modelli `IdentityKey` e `SignedPrekey` vengono impostati a stringhe vuote `""` quando si memorizzano le chiavi caricate dal client. Le chiavi private risiedono esclusivamente nell'IndexedDB del browser, che è isolato in sandbox per origine.

### Insidia 4: confronto non a tempo costante

**Sintomo:** un side channel di timing fa trapelare il materiale di chiave byte per byte.

**Causa:** usare `===` o `==` per il confronto di array di byte in JavaScript va in short-circuit alla prima discrepanza. Un attaccante in grado di misurare i tempi di risposta con precisione sufficiente (una risoluzione al microsecondo è ottenibile attraverso una rete) può determinare quanti byte iniziali sono corretti e ricavare il resto con brute force.

**Correzione:** confronto basato su XOR come in `frontend/src/crypto/primitives.ts`:

```typescript
export function constantTimeEqual(a: Uint8Array, b: Uint8Array): boolean {
 if (a.length !== b.length) return false
 let result = 0
 for (let i = 0; i < a.length; i++) {
 result |= a[i] ^ b[i]
 }
 return result === 0
}
```

---

## Principi di organizzazione del codice

Il codebase segue una stratificazione rigorosa che separa le responsabilità e minimizza il raggio d'impatto dei bug.

- **Logica di cifratura in `frontend/src/crypto/`** (mai nel livello API). I file `frontend/src/crypto/x3dh.ts` e `frontend/src/crypto/double-ratchet.ts` non sanno nulla di HTTP, WebSocket o database. Prendono byte in ingresso e restituiscono byte in uscita. `X3DHManager` non importa SQLModel. `DoubleRatchet` non importa FastAPI. Questo li rende testabili in modo indipendente: puoi istanziare `X3DHManager` in un file di test, chiamare `generate_identity_keypair_x25519` e verificare l'output senza configurare un database, una connessione Redis o un web server. Li rende anche riutilizzabili: se volessi usare la stessa implementazione X3DH in un tool CLI o in un diverso server framework, potresti importare `frontend/src/crypto/` senza tirarti dietro alcuna dipendenza web.

- **Operazioni sul database in `services/`** (mai in core). I file `prekey_service.py`, `message_service.py` e `auth_service.py` gestiscono tutte le query SQLModel e le operazioni SurrealDB. Ricevono una `AsyncSession` come parametro (dependency injection) e richiamano `frontend/src/crypto/` per le operazioni crittografiche. Il livello crittografico non vede mai un oggetto sessione, e il livello dei servizi non chiama mai direttamente OpenSSL. Questo significa che una modifica allo schema del database riguarda solo i file dei servizi e i modelli, non la logica di cifratura.

- **Schemi Pydantic per TUTTI i confini delle API.** Ogni tipo di messaggio WebSocket ha uno schema corrispondente in `schemas/websocket.py` (es. `EncryptedMessageWS`, `TypingIndicatorWS`, `ErrorMessageWS`). Ogni request/response HTTP ha uno schema in `schemas/auth.py`, `schemas/rooms.py`, ecc. Questo fornisce validazione a runtime: se un client invia un payload malformato (campi mancanti, tipi sbagliati, stringhe sovradimensionate), Pydantic lo rifiuta prima che i dati raggiungano la logica dei servizi. Per esempio, `ENCRYPTED_CONTENT_MAX_LENGTH = 50000` in `config.py` limita la lunghezza del campo ciphertext, impedendo a un client di inviare un ciphertext da 10GB che consumerebbe la memoria del server.

- **Singleton a livello di modulo per i manager stateless.** In fondo a `backend/app/core/websocket_manager.py`: `connection_manager = ConnectionManager`. In fondo a `backend/app/core/redis_manager.py` e `backend/app/core/surreal_manager.py`: `redis_manager` e `surreal_db`. Le implementazioni di X3DH e del Double Ratchet vivono nel browser come semplici moduli TypeScript con funzioni esportate, quindi non c'è un singleton equivalente lato server per esse. I singleton Python o gestiscono pool di connessioni condivisi o sono wrapper stateless; è sicuro condividerli all'interno del processo perché (a) i manager stateless non hanno stato mutabile, quindi task async concorrenti non possono interferire tra loro, e (b) `connection_manager.active_connections` viene toccato solo all'interno dell'unico event loop.

- **Async dall'inizio alla fine (nessun I/O bloccante).** Ogni query al database usa `await session.execute(...)`. Ogni operazione WebSocket usa `await websocket.send_json(...)`. Ogni chiamata Redis usa metodi async. Ogni chiamata SurrealDB usa metodi async. Le uniche operazioni sincrone sono i calcoli crittografici CPU-bound (HMAC, HKDF, AES-GCM, scambio DH X25519), che si completano in microsecondi e non traggono beneficio dall'async. Se un'operazione crittografica diventasse lenta (es. un hashing delle password Argon2 che richiede 500ms), dovrebbe essere spostata su un thread pool tramite `asyncio.to_thread` per evitare di bloccare l'event loop. Ma le operazioni usate qui richiedono tutte meno di un millisecondo.

- **Gerarchia di eccezioni personalizzata in `exceptions.py`.** Ogni tipo di errore ha la propria classe di eccezione (`EncryptionError`, `DecryptionError`, `KeyExchangeError`, `DatabaseError`, `UserNotFoundError`, ecc.), tutte ereditate da `AppException`. Questo permette a `exception_handlers.py` di mappare ogni tipo di eccezione su un codice di stato HTTP e una risposta di errore appropriati. Un `UserNotFoundError` restituisce 404. Un `EncryptionError` restituisce 500. Un `ChallengeExpiredError` restituisce 401. Senza la gerarchia, ogni errore sarebbe un generico 500, e il client non avrebbe idea di cosa sia andato storto.

- **Il frontend rispecchia la struttura del backend.** La directory `crypto/` del frontend è parallela a `frontend/src/crypto/` del backend. `primitives.ts` fornisce le stesse operazioni di basso livello della libreria Python `cryptography`. `x3dh.ts` è parallelo a `frontend/src/crypto/x3dh.ts`. `double-ratchet.ts` è parallelo a `frontend/src/crypto/double-ratchet.ts`. `crypto-service.ts` è parallelo a `message_service.py` (il livello di orchestrazione). Questa simmetria rende più facile rintracciare un bug: se la decifratura fallisce, confronti passo per passo le implementazioni del frontend e del backend, verificando che su entrambi i lati siano usate le stesse costanti, lo stesso ordinamento dei byte e gli stessi parametri HKDF.

---

## Dipendenze e perché

### Backend

- **cryptography:** la libreria della Python Cryptographic Authority. Incapsula OpenSSL per le operazioni critiche per le prestazioni (AES-GCM, HKDF) e fornisce implementazioni native per le operazioni sulle curve (X25519, Ed25519). La libreria è sottoposta ad audit, ha un team dedicato alla risposta sulla sicurezza e segue un processo di responsible disclosure. È la scelta standard per la crittografia in Python quando servono primitive di basso livello. Esistono alternative come PyCryptodome, ma hanno una base di maintainer più piccola, una storia di audit meno rigorosa e uno stile di API diverso (PyCryptodome richiede una gestione più manuale dei buffer). La libreria `cryptography` ha anche il vantaggio di separare le primitive "hazmat" (materiali pericolosi) dalle ricette di alto livello, il che rende esplicito quando stai facendo qualcosa che richiede una gestione attenta.

- **py_webauthn:** implementa il protocollo WebAuthn/FIDO2 lato server. Gestisce il complesso parsing ASN.1 degli oggetti di attestation, la decodifica CBOR dei dati dell'authenticator, la verifica delle firme su più famiglie di algoritmi (ES256, RS256, EdDSA) e la gestione del ciclo di vita delle challenge. La specifica WebAuthn supera le 200 pagine. Scrivere da zero un'implementazione conforme richiederebbe mesi e inviterebbe bug sottili già solo nel parsing CBOR. py_webauthn fornisce anche comodi helper come `bytes_to_base64url` e `base64url_to_bytes` che usiamo in tutto il codice di cifratura.

- **sqlmodel:** combina il query builder di SQLAlchemy con la validazione di Pydantic. Definisci un modello una sola volta (come `User` o `IdentityKey`) e ottieni sia lo schema di una tabella del database sia uno schema di serializzazione Pydantic. Il supporto ad `AsyncSession` tramite `sqlmodel.ext.asyncio.session` si integra in modo pulito con gli handler async delle richieste di FastAPI. L'alternativa sarebbe usare modelli SQLAlchemy grezzi più schemi Pydantic separati, il che raddoppia le definizioni dei modelli e introduce il rischio di disallineamento (modificare una colonna nel modello SQLAlchemy ma dimenticare di aggiornare lo schema Pydantic).

- **surrealdb:** il client Python per il protocollo WebSocket di SurrealDB. La funzionalità chiave è il supporto alle live query: `LIVE SELECT * FROM messages WHERE recipient_id = $user_id` crea una sottoscrizione persistente che spinge i nuovi record corrispondenti a un callback. Questo elimina la necessità di un sistema pub/sub separato (come Redis Pub/Sub o RabbitMQ) per la consegna dei messaggi in tempo reale. SurrealDB funge sia da archivio dei messaggi sia da bus di eventi in tempo reale.

- **orjson:** un serializzatore JSON basato su Rust che è da 3 a 10 volte più veloce del modulo standard `json`. Quando il server serializza centinaia di messaggi WebSocket al secondo (ciascuno contenente ciphertext e header codificati in base64), la differenza è misurabile nell'utilizzo della CPU. orjson gestisce anche `datetime`, `UUID` e `bytes` nativamente senza serializzatori personalizzati, riducendo il codice boilerplate. Produce lo stesso output JSON della libreria standard, quindi è un sostituto diretto.

### Frontend

- **WebCrypto API (integrata):** primitive crittografiche native del browser. Nessuna dipendenza npm per la crittografia significa rischio di supply chain nullo per il codice più critico dal punto di vista della sicurezza dell'applicazione. Un aggiornamento malevolo di un pacchetto crittografico npm potrebbe rubare chiavi private o indebolire la cifratura. Con WebCrypto, l'implementazione crittografica fa parte del browser stesso, aggiornata tramite il processo di sicurezza del browser. `subtle.generateKey`, `subtle.deriveBits`, `subtle.encrypt`, `subtle.decrypt`, `subtle.sign` e `subtle.verify` coprono ogni operazione necessaria per X25519, Ed25519, AES-GCM, HKDF e HMAC. Le implementazioni girano in codice nativo C/C++/Rust, che è sia più veloce sia più resistente agli attacchi di timing rispetto alle implementazioni JavaScript. WebCrypto applica anche restrizioni sull'uso delle chiavi a livello di API (una chiave importata per "encrypt" non può essere usata per "decrypt"), fornendo difesa in profondità.

- **nanostores:** una libreria minimale di state management sotto 1KB compressa con gzip. Per un'applicazione di chat, le esigenze di gestione dello stato sono semplici: "chi è online", "quali messaggi ci sono in questa room", "l'utente sta digitando", "quale room è selezionata". Nanostores fornisce store reattivi `atom` (valore singolo) e `computed` (valore derivato) che si integrano con il sistema reattivo di SolidJS. Gli atom `$connectionStatus`, `$isConnected` e `$reconnectAttempts` in `websocket-manager.ts` dimostrano il pattern. A differenza di Redux (che richiede action, reducer e middleware per le operazioni async), nanostores è solo get/set/subscribe. La semplicità è il punto: meno astrazioni significano meno posti in cui i bug possono nascondersi.

- **@tanstack/solid-query:** gestisce lo stato del server (dati recuperati dalle API REST) separatamente dallo stato del client (interazioni con la UI, eventi WebSocket). Gestisce il caching (non recuperare di nuovo la lista delle room se è stata recuperata 30 secondi fa), il refetch in background (aggiornare la lista delle room quando la scheda torna in primo piano), lo stale-while-revalidate (mostrare subito i dati in cache, recuperare dati freschi in background) e le transizioni di stato errore/caricamento/successo. Senza di essa, ogni componente che ha bisogno di dati dal server dovrebbe gestire manualmente lo stato di `isLoading`, `error` e `data`, implementare la propria logica di invalidazione della cache e gestire le race condition tra fetch concorrenti. TanStack Query assorbe tutta quella complessità in una chiamata dichiarativa `createQuery`.
