# Architettura del sistema

Questo documento descrive l'architettura completa dell'applicazione di chat P2P cifrata. Implementa un sottoinsieme del Signal Protocol (X3DH + Double Ratchet) per la crittografia end-to-end, con WebAuthn/Passkey per l'autenticazione senza password. Il server è un relay che memorizza blob di ciphertext e non vede mai il contenuto in chiaro dei messaggi.

Tutto ciò che segue è basato sul codice sorgente effettivo. I riferimenti ai file usano il formato `filename.py:line-range` relativo alla directory `app/` del backend o `src/` del frontend.

---

## Architettura di alto livello

```
 HTTPS (TLS)
 ┌────────────────────────────────────────────────────────────┐
 │ Client Browser │
 │ ┌──────────────────────────────────────────────────────┐ │
 │ │ SolidJS 1.9 + TypeScript 5.9 │ │
 │ │ ┌──────────────┐ ┌──────────────┐ ┌────────────┐ │ │
 │ │ │ WebCrypto │ │ IndexedDB │ │ nanostores │ │ │
 │ │ │ X25519 │ │ Private │ │ Reactive │ │ │
 │ │ │ Ed25519 │ │ Keys │ │ State │ │ │
 │ │ │ AES-256-GCM │ │ Ratchet │ │ │ │ │
 │ │ │ HKDF-SHA256 │ │ States │ │ │ │ │
 │ │ └──────────────┘ └──────────────┘ └────────────┘ │ │
 │ └──────────────────────────┬───────────────────────────┘ │
 │ │ │
 └──────────────────────────────┼──────────────────────────────┘
 │
 HTTPS + WSS │
 │
 ┌──────────────────────────────┼──────────────────────────────┐
 │ Nginx Reverse Proxy │ │
 │ ┌───────────────────────────┴──────────────────────────┐ │
 │ │ /api/* ──────► HTTP ──────► FastAPI :8000 │ │
 │ │ /ws ──────► WS ──────► FastAPI :8000 │ │
 │ │ /* ──────► Static files (SolidJS build) │ │
 │ └──────────────────────────────────────────────────────┘ │
 └──────────────────────────────┬──────────────────────────────┘
 │
 ┌──────────────────────────────┼──────────────────────────────┐
 │ FastAPI Backend │ │
 │ (Python 3.13, ORJSONResponse, async/await throughout) │
 │ ┌──────────────────────────────────────────────────────┐ │
 │ │ Routers: auth, rooms, encryption, websocket │ │
 │ │ Services: auth, prekey, message, presence, websocket│ │
 │ │ Core: x3dh, double_ratchet, passkey, surreal, redis │ │
 │ └────────┬──────────────┬──────────────┬───────────────┘ │
 │ │ │ │ │
 └───────────┼──────────────┼──────────────┼───────────────────┘
 │ │ │
 asyncpg │ WebSocket │ redis. │
 (TCP 5432) │ (WS 8000) │ asyncio │
 │ │ (TCP 6379) │
 ┌───────────┴──┐ ┌───────┴──────┐ ┌───┴────────────┐
 │ PostgreSQL │ │ SurrealDB │ │ Redis 8 │
 │ 16-alpine │ │ (latest) │ │ (alpine) │
 │ │ │ │ │ │
 │ Auth data │ │ Messages │ │ Challenges │
 │ Credentials │ │ Presence │ │ Rate limits │
 │ X3DH keys │ │ Rooms │ │ │
 │ Ratchet │ │ Live │ │ TTL-based │
 │ states │ │ queries │ │ auto-expiry │
 └──────────────┘ └──────────────┘ └────────────────┘
```

Alcune cose che vale la pena notare su questo diagramma:

1. Tutti e tre i database servono a scopi distinti. PostgreSQL gestisce i dati relazionali che richiedono transazioni ACID (utenti, credenziali, chiavi crittografiche). SurrealDB gestisce i dati in tempo reale che beneficiano delle notifiche push delle live query (messaggi, presenza). Redis gestisce i dati effimeri che necessitano di scadenza automatica (challenge WebAuthn, contatori di rate limit).

2. La connessione WebSocket tra il client e FastAPI è il canale principale per la messaggistica in tempo reale. HTTP è usato solo per i flussi di autenticazione e la gestione delle chiavi.

3. Il client esegue tutta la cifratura e la decifratura. Il server riceve ciphertext e lo memorizza così com'è. Questo è il vincolo di design centrale da cui discende tutto il resto.

---

## Scomposizione dei componenti

### Backend FastAPI

**Scopo:** server API che gestisce autenticazione, gestione delle chiavi, inoltro dei messaggi e connessioni WebSocket.

**File chiave:** `factory.py`

L'applicazione usa il pattern factory. `create_app` costruisce l'istanza FastAPI:

```
create_app (factory.py)
 ├── FastAPI(
 │ title = settings.APP_NAME,
 │ default_response_class = ORJSONResponse,
 │ lifespan = lifespan,
 │ )
 ├── CORSMiddleware (factory.py)
 ├── GZipMiddleware, minimum_size=1000 (factory.py)
 ├── register_exception_handlers (factory.py)
 ├── Root endpoint "/" (factory.py)
 ├── Health endpoint "/health" (factory.py)
 └── Routers:
 ├── auth_router → /auth/*
 ├── rooms_router → /rooms/*
 ├── encryption_router → /encryption/*
 └── websocket_router → /ws
```

Il lifespan manager esegue le connessioni ai database all'avvio e le disconnessioni allo spegnimento:

```
lifespan(app) (factory.py)
 Startup:
 1. await init_db → PostgreSQL tables via SQLModel
 2. await redis_manager.connect → Redis connection pool
 3. await surreal_db.connect → SurrealDB WebSocket connection
 Shutdown:
 1. await redis_manager.disconnect
 2. await surreal_db.disconnect
```

L'ordine di avvio conta. PostgreSQL viene inizializzato per primo perché le tabelle di autenticazione devono esistere prima di qualsiasi elaborazione delle richieste. Seguono Redis e SurrealDB perché dipendono dall'avere un contesto applicativo in esecuzione. Se uno qualsiasi di questi fallisce, l'applicazione non parte.

**Interfacce:**

| Prefisso della route | Scopo | Endpoint principali |
|---|---|---|
| `/auth` | Registrazione + login + sessione WebAuthn | `register/begin`, `register/complete`, `authenticate/begin`, `authenticate/complete`, `me`, `logout`, `users/search` |
| `/encryption` | Memorizzazione e lookup del bundle pubblico di prekey | `prekey-bundle/{id}`, `upload-keys/{id}` |
| `/rooms` | CRUD delle chat room (subordinato all'appartenenza) | Elenco, creazione, lettura, eliminazione, lettura dei messaggi |
| `/ws` | Endpoint WebSocket (autenticato tramite session cookie) | Messaggistica in tempo reale, digitazione, presenza, conferme di lettura |

---

### Livello WebSocket

**Scopo:** comunicazione bidirezionale in tempo reale per messaggistica, indicatori di digitazione, aggiornamenti di presenza e conferme di lettura.

**File chiave:** `core/websocket_manager.py`, `api/websocket.py`, `services/websocket_service.py`

La classe `ConnectionManager` (`websocket_manager.py`) è il centro del sistema in tempo reale. Mantiene tre dizionari:

```python
self.active_connections: dict[UUID, list[WebSocket]] = {}
self.live_query_ids: dict[UUID, str] = {}
self.heartbeat_tasks: dict[UUID, asyncio.Task] = {}
```

Il primo mappa gli ID utente su liste di connessioni WebSocket. Ogni utente può avere fino a 5 connessioni simultanee (`WS_MAX_CONNECTIONS_PER_USER = 5`, `config.py`). Questo supporta l'uso multi-dispositivo senza esaurire le risorse del server. Quando arriva un sesto tentativo di connessione, il manager invia un errore e chiude il socket (`websocket_manager.py`).

**Ciclo di vita della connessione:**

```
connect(websocket, user_id) (websocket_manager.py)
 1. Accept WebSocket
 2. Check connection limit (max 5 per user)
 3. Add to active_connections pool
 4. Set user online via presence_service
 5. Start heartbeat loop (30s interval)
 6. Subscribe to SurrealDB live query for user's messages
 7. Return True

disconnect(websocket, user_id) (websocket_manager.py)
 1. Remove WebSocket from user's connection list
 2. If last connection:
 a. Set user offline via presence_service
 b. Kill SurrealDB live query
 c. Cancel heartbeat task
```

**Heartbeat:** il metodo `_heartbeat_loop` (`websocket_manager.py`) invia un ping ogni `WS_HEARTBEAT_INTERVAL` secondi (30s, da `config.py`). Se l'invio fallisce, la connessione viene marcata come morta e disconnessa. Questo intercetta le connessioni cadute silenziosamente che il solo TCP keepalive non rileverebbe.

**Sottoscrizione alle live query:** quando un utente si connette, `_subscribe_to_messages` (`websocket_manager.py`) registra una live query SurrealDB che osserva i nuovi messaggi di cui l'utente è il destinatario. SurrealDB spinge i nuovi messaggi al server in tempo reale tramite questa sottoscrizione. Il callback incapsula gli aggiornamenti e li smista tramite `_handle_live_message` (`websocket_manager.py`), che inoltra il payload cifrato a tutte le connessioni WebSocket attive dell'utente.

**Instradamento dei messaggi:** l'endpoint WebSocket (`websocket.py`) è sottile. Accetta le connessioni, delega a `connection_manager.connect` e poi resta in un loop a leggere messaggi JSON. Ogni messaggio viene instradato da `websocket_service.route_message`, che smista in base al campo `type`:

```
route_message(websocket, user_id, message) (backend/app/services/websocket_service.py)
 ├── "encrypted_message" → handle_encrypted_message
 ├── "typing" → handle_typing_indicator
 ├── "presence" → handle_presence_update
 ├── "receipt" → handle_read_receipt
 ├── "heartbeat" → handle_heartbeat
 └── unknown → send error response
```

**Pulizia delle connessioni morte:** quando `send_message` (`websocket_manager.py`) non riesce a inviare a una connessione, raccoglie la connessione morta e chiama `disconnect` su di essa. Questo impedisce l'accumulo di connessioni obsolete.

---

### Motore di cifratura

**Scopo:** implementa le primitive crittografiche del Signal Protocol per la crittografia end-to-end.

**File chiave:** `frontend/src/crypto/frontend/src/crypto/x3dh.ts`, `frontend/src/crypto/frontend/src/crypto/double-ratchet.ts`, `frontend/src/crypto/primitives.ts`

Il sistema di cifratura ha due livelli:

1. **X3DH (Extended Triple Diffie-Hellman):** stabilisce un segreto condiviso tra due utenti che non hanno mai comunicato prima, anche se uno dei due è offline.

2. **Double Ratchet:** usa il segreto condiviso di X3DH per derivare chiavi di cifratura per messaggio con forward secrecy. Ogni messaggio riceve una chiave unica, e compromettere una chiave non rivela i messaggi passati o futuri.

**X3DHManager** (`frontend/src/crypto/x3dh.ts`) gestisce:
- La generazione delle identity key (X25519 per il DH, Ed25519 per la firma)
- La generazione delle signed prekey con firme Ed25519 (`frontend/src/crypto/x3dh.ts`)
- La generazione delle one-time prekey (`frontend/src/crypto/x3dh.ts`)
- La verifica delle signed prekey (`frontend/src/crypto/x3dh.ts`)
- Lo scambio X3DH lato mittente (`frontend/src/crypto/x3dh.ts`)
- Lo scambio X3DH lato ricevente (`frontend/src/crypto/x3dh.ts`)

Lo scambio X3DH lato mittente (`perform_x3dh_sender`, righe 208-281) funziona così:

```
Alice wants to message Bob (who might be offline):

1. Alice fetches Bob's prekey bundle from server:
 - Bob's identity key (IK_B)
 - Bob's signed prekey (SPK_B) + signature
 - Bob's one-time prekey (OPK_B), if available

2. Alice verifies SPK_B signature using Bob's Ed25519 identity key

3. Alice generates ephemeral keypair (EK_A)

4. Four DH operations:
 DH1 = X25519(IK_A_private, SPK_B) # Alice identity x Bob signed prekey
 DH2 = X25519(EK_A_private, IK_B) # Alice ephemeral x Bob identity
 DH3 = X25519(EK_A_private, SPK_B) # Alice ephemeral x Bob signed prekey
 DH4 = X25519(EK_A_private, OPK_B) # Alice ephemeral x Bob one-time prekey
 # (only if OPK available)

5. Concatenate: key_material = DH1 || DH2 || DH3 [|| DH4]

6. Derive shared key via HKDF-SHA256:
 shared_key = HKDF(
 salt = 0x00 * 32,
 ikm = 0xFF * 32 || key_material,
 info = "X3DH",
 length = 32
 )

7. Return: shared_key, associated_data (IK_A_pub || IK_B_pub), EK_A_pub
```

Il prefisso `0xFF * 32` prima del materiale di chiave è un separatore di dominio. Corrisponde alla specifica di Signal e previene potenziali attacchi cross-protocol.

**DoubleRatchet** (`frontend/src/crypto/double-ratchet.ts`) gestisce:
- L'inizializzazione del mittente a partire dal segreto condiviso di X3DH (`frontend/src/crypto/double-ratchet.ts`)
- L'inizializzazione del ricevente (`frontend/src/crypto/double-ratchet.ts`)
- La cifratura dei messaggi con avanzamento della chain key (`frontend/src/crypto/double-ratchet.ts`)
- La decifratura dei messaggi con supporto per l'ordine non sequenziale (`frontend/src/crypto/double-ratchet.ts`)
- I passi del ratchet DH per la forward secrecy (`frontend/src/crypto/double-ratchet.ts`)
- La memorizzazione delle skipped message key per la consegna fuori ordine (`frontend/src/crypto/double-ratchet.ts`)

Il ratchet usa due catene KDF:

```
Root Key Chain:
 _kdf_rk(root_key, dh_output) → (new_root_key, new_chain_key)
 Uses HKDF-SHA256 with root_key as salt (frontend/src/crypto/double-ratchet.ts)

Symmetric Key Chain:
 _kdf_ck(chain_key) → (next_chain_key, message_key)
 Uses HMAC-SHA256 with 0x01 for chain key, 0x02 for message key
 (frontend/src/crypto/double-ratchet.ts)
```

Ogni messaggio è cifrato con AES-256-GCM usando un nonce casuale da 12 byte (`frontend/src/crypto/double-ratchet.ts`). La message key è derivata dalla catena di invio e usata esattamente una volta. Dopo la cifratura, la catena di invio avanza e la vecchia message key viene scartata.

**Limiti di sicurezza da `config.py`:**
- `MAX_SKIP_MESSAGE_KEYS = 1000` : numero massimo di messaggi che possono arrivare fuori ordine prima che il protocollo li rifiuti
- `MAX_CACHED_MESSAGE_KEYS = 2000` : numero massimo di chiavi saltate memorizzate prima della rimozione
- `AES_GCM_NONCE_SIZE = 12` : nonce da 96 bit per AES-GCM
- `HKDF_OUTPUT_SIZE = 32` : chiavi derivate da 256 bit

**Crittografia lato client:** il `primitives.ts` del frontend rispecchia la crittografia del backend usando la WebCrypto API. Fornisce:
- `generateX25519KeyPair`
- `x25519DeriveSharedSecret`
- `generateEd25519KeyPair`
- `ed25519Sign` / `ed25519Verify` (righe 99, 113)
- `hkdfDerive` / `hkdfDeriveKey` (righe 166, 194)
- `aesGcmEncrypt` / `aesGcmDecrypt` (righe 224, 261)
- `hmacSha256` / `hmacSha256Verify` (righe 310, 329)
- `constantTimeEqual` : confronto a tempo costante per prevenire side channel di timing

Il client ha anche `crypto-service.ts`, `double-ratchet.ts`, `x3dh.ts`, `key-store.ts` e `message-store.ts`, che orchestrano queste primitive nei flussi completi del protocollo.

**Un solo percorso di cifratura: lato client, il server è opaco.**

`MessageService` (`backend/app/services/message_service.py`) espone un unico metodo, `store_encrypted_message`, che riceve `ciphertext`, `nonce` e `header` come stringhe opache e le scrive direttamente in SurrealDB. Il server non ha alcun helper di decifratura né alcuno stato del ratchet — letteralmente non può leggere i messaggi perché non possiede mai le chiavi.

---

### Sistema di autenticazione

**Scopo:** autenticazione senza password tramite passkey WebAuthn/FIDO2.

**File chiave:** `core/passkey/passkey_manager.py`, `api/auth.py`, `services/auth_service.py`, `core/redis_manager.py`

Il `PasskeyManager` (`backend/app/core/passkey/passkey_manager.py`) incapsula la libreria `py_webauthn`. È configurato con le impostazioni della Relying Party (RP) da `config.py`:

```python
self.rp_id = settings.RP_ID # e.g., "localhost" or "chat.example.com"
self.rp_name = settings.RP_NAME # "Encrypted P2P Chat"
self.rp_origin = settings.RP_ORIGIN # "https://chat.example.com"
```

**Flusso di registrazione:**

```
Client Server Redis
 │ │ │
 │ POST /auth/register/begin│ │
 │ { username, display_name }│ │
 │ ─────────────────────────►│ │
 │ │ │
 │ PasskeyManager.generate_registration_options
 │ (backend/app/core/passkey/passkey_manager.py) │
 │ │ │
 │ │ SET webauthn:reg_challenge:{username}
 │ │ challenge_bytes, TTL=600s│
 │ │ ──────────────────────────►
 │ │ │
 │ ◄─── registration options (publicKey config) ────────│
 │ │ │
 │ Browser WebAuthn API │ │
 │ navigator.credentials │ │
 │ .create(options) │ │
 │ User touches │ │
 │ authenticator │ │
 │ │ │
 │ POST /auth/register/complete │
 │ { credential, username } │ │
 │ ─────────────────────────►│ │
 │ │ │
 │ │ GET+DEL webauthn:reg_challenge:{username}
 │ │ ──────────────────────────►
 │ │ ◄── challenge_bytes ──────│
 │ │ │
 │ PasskeyManager.verify_registration │
 │ (backend/app/core/passkey/passkey_manager.py) │
 │ │ │
 │ Create User in PostgreSQL │
 │ Store Credential in PostgreSQL │
 │ Initialize X3DH keys │
 │ (backend/app/services/prekey_service.py) │
 │ │ │
 │ ◄─── UserResponse (id, username, etc.) ──────────────│
```

**Flusso di autenticazione:**

Il flusso di autenticazione è simile ma usa `generate_authentication_options` e `verify_authentication`. Un dettaglio critico è il controllo del contatore di firma (`backend/app/core/passkey/passkey_manager.py`): se il nuovo contatore non è maggiore di quello memorizzato, solleva un `ValueError` che indica un authenticator potenzialmente clonato. Questo è il meccanismo di rilevamento dei cloni di WebAuthn.

**Memorizzazione delle challenge:** Redis memorizza le challenge con un TTL di 600 secondi (`WEBAUTHN_CHALLENGE_TTL_SECONDS = 600`, `config.py`). I metodi `get_registration_challenge` e `get_authentication_challenge` usano le pipeline di Redis per eseguire in modo atomico GET e DELETE della challenge (`redis_manager.py`). Questo garantisce che ogni challenge venga usata esattamente una volta.

La challenge stessa è composta da 32 byte casuali (`WEBAUTHN_CHALLENGE_BYTES = 32`, `config.py`), memorizzati in Redis come esadecimale.

**Endpoint di autenticazione** (`auth.py`):

| Endpoint | Metodo | Stato | Descrizione |
|---|---|---|---|
| `/auth/register/begin` | POST | 200 | Genera le opzioni di registrazione WebAuthn |
| `/auth/register/complete` | POST | 201 | Verifica la credenziale, crea utente + chiavi |
| `/auth/authenticate/begin` | POST | 200 | Genera le opzioni di autenticazione WebAuthn |
| `/auth/authenticate/complete` | POST | 200 | Verifica la credenziale, aggiorna il contatore |
| `/auth/users/search` | POST | 200 | Cerca utenti per username/display name |

---

### Livello database

**Scopo:** archiviazione persistente suddivisa su tre database costruiti ad hoc.

**PostgreSQL tramite SQLModel/SQLAlchemy async:** gestisce tutti i dati relazionali con garanzie ACID.

L'engine è configurato in `models/Base.py`:

```python
engine = create_async_engine(
 str(settings.DATABASE_URL), # postgresql+asyncpg://...
 echo = settings.DEBUG, # SQL logging in development
 pool_size = settings.DB_POOL_SIZE, # 20 (config.py)
 max_overflow = settings.DB_MAX_OVERFLOW, # 40 (config.py)
 pool_pre_ping = True, # Detect stale connections
)
```

La session factory (`Base.py`) crea istanze di `AsyncSession` con `expire_on_commit=False`, così che gli oggetti restino utilizzabili dopo il commit senza richiedere un refresh.

**SurrealDB tramite AsyncSurreal:** gestisce i dati di messaggistica in tempo reale. Il `SurrealDBManager` (`surreal_manager.py`) si connette a SurrealDB via WebSocket e fornisce metodi per il CRUD dei messaggi, la gestione delle room, il tracciamento della presenza e le sottoscrizioni alle live query. La funzionalità di live query è il motivo principale per cui è stato scelto SurrealDB: spinge i nuovi record ai sottoscrittori in tempo reale, eliminando la necessità di polling.

**Redis tramite redis.asyncio:** gestisce i dati effimeri. Il `RedisManager` (`redis_manager.py`) usa un pool di 50 connessioni (`redis_manager.py`) e memorizza le challenge come byte codificati in esadecimale con scadenza basata su TTL.

---

## Flusso dei dati

### Flusso principale: invio di un messaggio cifrato

Questo è il flusso più importante del sistema. Ecco cosa succede passo dopo passo quando Alice invia un messaggio a Bob, assumendo che abbiano già una sessione Double Ratchet stabilita:

```
Alice's Browser Server Bob's Browser
 │ │ │
 ┌────┴────────────────┐ │ │
 │ 1. User types msg │ │ │
 │ in ChatInput.tsx │ │ │
 └────┬────────────────┘ │ │
 │ │ │
 ┌────┴────────────────┐ │ │
 │ 2. crypto-service.ts│ │ │
 │ encrypts: │ │ │
 │ a. Advance send │ │ │
 │ chain via │ │ │
 │ HMAC-SHA256 │ │ │
 │ b. Derive unique │ │ │
 │ message key │ │ │
 │ c. AES-256-GCM │ │ │
 │ encrypt with │ │ │
 │ random nonce │ │ │
 │ d. Build header: │ │ │
 │ {dh_pub_key, │ │ │
 │ msg_number, │ │ │
 │ prev_chain} │ │ │
 └────┬────────────────┘ │ │
 │ │ │
 │ 3. WebSocket send: │ │
 │ { │ │
 │ type: "encrypted_message", │
 │ recipient_id: bob_uuid, │ │
 │ room_id: room_uuid, │ │
 │ ciphertext: "base64...",│ │
 │ nonce: "base64...", │ │
 │ header: "{...json...}", │ │
 │ temp_id: "client_123" │ │
 │ } │ │
 │ ───────────────────────────► │
 │ │ │
 │ ┌────────────┴───────────────┐ │
 │ │ 4. websocket.py │ │
 │ │ receives JSON, parses │ │
 │ │ routes via │ │
 │ │ websocket_service │ │
 │ └────────────┬───────────────┘ │
 │ │ │
 │ ┌────────────┴───────────────┐ │
 │ │ 5. websocket_service.py: │ │
 │ │ handle_encrypted_message│ │
 │ │ │ │
 │ │ Extracts fields, │ │
 │ │ opens DB session │ │
 │ └────────────┬───────────────┘ │
 │ │ │
 │ ┌────────────┴───────────────┐ │
 │ │ 6. message_service.py: │ │
 │ │ store_encrypted_message │ │
 │ │ │ │
 │ │ Looks up sender user │ │
 │ │ for username. │ │
 │ │ Stores ciphertext, │ │
 │ │ nonce, header AS-IS in │ │
 │ │ SurrealDB. │ │
 │ │ NO DECRYPTION. │ │
 │ └────────────┬───────────────┘ │
 │ │ │
 │ ┌────────────┴───────────────┐ │
 │ │ 7. SurrealDB live query │ │
 │ │ fires for Bob because │ │
 │ │ recipient_id matches │ │
 │ └────────────┬───────────────┘ │
 │ │ │
 │ ┌────────────┴───────────────┐ │
 │ │ 8. websocket_manager.py: │ │
 │ │ _handle_live_message │ │
 │ │ │ │
 │ │ Wraps as EncryptedMsgWS │ │
 │ └────────────┬───────────────┘ │
 │ │ │
 │ ┌────────────┴───────────────┐ │
 │ │ 9. send_message(bob_uuid) │ │
 │ │ │ │
 │ │ Sends to ALL of Bob's │ │
 │ │ active WebSockets │ │
 │ └────────────┬───────────────┘ │
 │ │ ─────────────────────────────►
 │ │ │
 │ │ ┌────────────────┴──┐
 │ │ │ 10. Bob's crypto- │
 │ │ │ service.ts │
 │ │ │ decrypts: │
 │ │ │ a. Check for │
 │ │ │ skipped │
 │ │ │ msg keys │
 │ │ │ b. If new DH │
 │ │ │ pub key, │
 │ │ │ DH ratchet │
 │ │ │ step │
 │ │ │ c. Advance │
 │ │ │ recv chain │
 │ │ │ d. Derive msg │
 │ │ │ key │
 │ │ │ e. AES-256-GCM│
 │ │ │ decrypt │
 │ │ └────────┬─────────┘
 │ │ │
 │ │ ┌────────┴─────────┐
 │ │ │ 11. Plaintext │
 │ │ │ rendered in │
 │ │ │ MessageList │
 │ │ └──────────────────┘
 │ │ │
 │ ◄── confirmation ─────────│ │
 │ { type: "message_sent", │ │
 │ temp_id: "client_123", │ │
 │ status: "sent" } │ │
```

Il server non vede mai il testo in chiaro. Agisce puramente da relay che memorizza e inoltra blob di ciphertext. Il metodo `store_encrypted_message` (`backend/app/services/message_service.py`) letteralmente si limita a incapsulare i campi ricevuti in un dict e a chiamare `surreal_db.create_message`.

Il messaggio di conferma rimandato ad Alice (`backend/app/services/websocket_service.py`) include il `temp_id`, così che il client possa abbinarlo al messaggio visualizzato in modo ottimistico nella UI.

---

### Flusso secondario: registrazione di un nuovo utente + setup delle chiavi

Questo flusso avviene una volta per utente.

```
Step 1: POST /auth/register/begin
 backend/app/services/auth_service.py
 → PasskeyManager generates challenge
 → Challenge stored in Redis with 600s TTL
 → Registration options returned to client

Step 2: Browser WebAuthn API (navigator.credentials.create)
 → User interacts with authenticator (Touch ID, YubiKey, etc.)
 → Browser creates credential bound to RP origin

Step 3: POST /auth/register/complete
 backend/app/services/auth_service.py
 → Challenge retrieved from Redis (GET+DEL atomic)
 → PasskeyManager verifies credential against challenge
 → User record created in PostgreSQL
 → Credential record created in PostgreSQL
 → prekey_service.initialize_user_keys called
 (backend/app/services/prekey_service.py)

Step 4: Server-side key initialization (backend/app/services/prekey_service.py)
 → Generate X25519 identity keypair (IK)
 → Generate Ed25519 signing keypair
 → Store both public+private in PostgreSQL identity_keys table
 → Generate signed prekey (SPK) signed with Ed25519 IK
 → Generate 100 one-time prekeys (OPKs)
 → Store all in PostgreSQL

Step 5: (Client-side, post-registration)
 Client generates its own X3DH keys using WebCrypto:
 → X25519 identity keypair
 → Ed25519 signing keypair
 → Signed prekey with signature
 → 100 one-time prekeys

Step 6: POST /encryption/upload-keys/{user_id}
 backend/app/api/encryption.py
 → prekey_service.store_client_keys
 (backend/app/services/prekey_service.py)
 → Only PUBLIC keys stored on server
 → Private keys remain in browser IndexedDB
```

Nota che qui c'è un doppio percorso. Il passo 4 è la generazione delle chiavi lato server (usata come fallback e per retrocompatibilità). I passi 5-6 sono il percorso di generazione delle chiavi lato client (il percorso di produzione preferito). Nel percorso lato client, il server non vede mai le chiavi private.

---

### Flusso secondario: stabilire una nuova sessione di chat (X3DH)

Quando Alice vuole scrivere a Bob per la prima volta:

```
Step 1: Alice's client requests Bob's prekey bundle
 GET /encryption/prekey-bundle/{bob_id}
 backend/app/api/encryption.py

Step 2: prekey_service.get_prekey_bundle(session, bob_id)
 backend/app/services/prekey_service.py
 → Fetch Bob's identity key (IK)
 → Fetch Bob's active signed prekey (SPK)
 → If no active SPK, auto-rotate (backend/app/services/prekey_service.py)
 → Fetch one unused one-time prekey (OPK)
 → Mark OPK as used (single-use guarantee,)
 → If unused OPK count < 20, auto-replenish 100 more
 (backend/app/api/encryption.py)
 → Return PreKeyBundle{IK, IK_ed25519, SPK, SPK_sig, OPK}

Step 3: Alice performs X3DH sender-side locally
 Using the prekey bundle and her own identity key:
 a. Verify SPK signature with Bob's Ed25519 IK
 b. Generate ephemeral keypair (EK)
 c. Compute DH1..DH4
 d. Derive shared_key via HKDF

Step 4: Initialize Double Ratchet with shared_key
 Alice's client calls double_ratchet.initialize_sender
 with the shared key and Bob's SPK as the initial peer key

Step 5: First message includes X3DH header
 The header contains Alice's ephemeral public key and
 identity key reference so Bob can derive the same
 shared secret when he comes online

Step 6: Bob receives the message
 Bob's client uses the X3DH header + his own private keys
 to perform the receiver-side X3DH exchange
 (frontend/src/crypto/x3dh.ts)
 Both parties now share the same root key for the
 Double Ratchet
```

La garanzia di uso singolo delle OPK è applicata a livello di database. Quando `get_prekey_bundle` recupera una OPK, la marca immediatamente con `is_used = True` ed esegue il commit (`backend/app/services/prekey_service.py`). Se qualcun altro richiede la stessa OPK in modo concorrente, ne otterrà una diversa o nessuna.

---

## Design pattern

### Pattern Application Factory

**Dove:** `factory.py`

**Perché:** la funzione `create_app` costruisce e restituisce un'istanza FastAPI completamente configurata. Questo separa la creazione dell'app dalla sua esecuzione (`main.py` si limita a chiamare `create_app`). Il vantaggio pratico è la testabilità: puoi chiamare `create_app` con configurazioni diverse nei test senza avviare un server. Il lifespan manager (`factory.py`) garantisce che tutti i database siano connessi prima della prima richiesta e correttamente disconnessi allo spegnimento.

### Pattern Service Layer

**Dove:** directory `services/` (auth_service.py, prekey_service.py, message_service.py, presence_service.py, websocket_service.py)

**Perché:** la business logic vive nelle classi di servizio, non nelle funzioni degli endpoint API. Il livello API (`api/`) è sottile: gestisce la validazione delle richieste e la formattazione delle risposte, poi delega ai servizi. I servizi sono singleton stateless istanziati a livello di modulo:

```python
message_service = MessageService # backend/app/services/message_service.py
prekey_service = PrekeyService # backend/app/services/prekey_service.py
auth_service = AuthService # backend/app/services/auth_service.py
websocket_service = WebSocketService # backend/app/services/websocket_service.py
```

Questo significa che qualsiasi endpoint può importare e chiamare qualsiasi servizio senza preoccuparsi dell'istanziazione o della dependency injection. Il compromesso è che i servizi non possono essere facilmente sostituiti a runtime, ma per questa applicazione non è necessario.

### Pattern Connection Pool

**Dove:** `websocket_manager.py`

**Perché:** il `ConnectionManager` mappa gli ID utente su liste di connessioni WebSocket. Questo supporta più dispositivi simultanei per utente (fino a 5, limite applicato in `websocket_manager.py`). Quando un messaggio deve essere consegnato, `send_message` itera su tutte le connessioni di quell'utente. Le connessioni morte vengono rilevate durante i tentativi di invio e ripulite immediatamente.

### Pattern Observer (live query)

**Dove:** `websocket_manager.py`, `surreal_manager.py`

**Perché:** le live query di SurrealDB implementano un sistema di notifica basato su push. Quando in SurrealDB viene creato un nuovo messaggio con `recipient_id = bob`, SurrealDB spinge quel record al server tramite il callback della live query. Il server lo inoltra poi alle connessioni WebSocket di Bob. Questo elimina completamente il polling. L'alternativa sarebbe che il server interrogasse periodicamente SurrealDB alla ricerca di nuovi messaggi, il che aggiungerebbe una latenza proporzionale all'intervallo di polling e sprecherebbe risorse quando non ci sono messaggi in attesa.

La sottoscrizione è per utente, non per room. `live_messages_for_user` (`surreal_manager.py`) osserva `WHERE recipient_id = '{user_id}'`, quindi il server riceve tutti i messaggi destinati a quell'utente indipendentemente dalla room.

### Pattern Singleton

**Dove:** istanze a livello di modulo in fondo a ciascun file manager:

```python
passkey_manager = PasskeyManager # backend/app/core/passkey/passkey_manager.py
connection_manager = ConnectionManager # backend/app/core/websocket_manager.py
surreal_db = SurrealDBManager # backend/app/core/surreal_manager.py
redis_manager = RedisManager # backend/app/core/redis_manager.py
```

**Perché:** questi manager o non mantengono alcuno stato per richiesta (`PasskeyManager` contiene solo la configurazione dell'RP) o gestiscono pool di connessioni condivisi (`surreal_db`, `redis_manager`, `connection_manager`). Una singola istanza per processo è il modello corretto. Le implementazioni di X3DH e del Double Ratchet vivono nel browser come semplici moduli TypeScript — non c'è alcuna controparte lato server che abbia bisogno di un singleton.

---

## Separazione dei livelli

```
┌─────────────────────────────────────────────────────────────────┐
│ API Layer (api/) │
│ ┌───────────┐ ┌──────────────┐ ┌─────────┐ ┌──────────────┐ │
│ │ auth.py │ │ encryption.py│ │ rooms.py│ │ websocket.py │ │
│ │ lines │ │ lines 1-127 │ │ │ │ lines 1-85 │ │
│ │ 1-104 │ │ │ │ │ │ │ │
│ └───────────┘ └──────────────┘ └─────────┘ └──────────────┘ │
│ HTTP/WS endpoints. Request validation. Response formatting. │
│ Thin wrappers that delegate to services. │
├─────────────────────────────────────────────────────────────────┤
│ Service Layer (services/) │
│ ┌──────────────┐ ┌───────────────┐ ┌─────────────────────┐ │
│ │ auth_service │ │ prekey_service│ │ message_service │ │
│ │ lines 1-601 │ │ lines 1-468 │ │ lines 1-469 │ │
│ └──────────────┘ └───────────────┘ └─────────────────────┘ │
│ ┌──────────────────┐ ┌──────────────────┐ │
│ │ presence_service │ │ websocket_service│ │
│ └──────────────────┘ └──────────────────┘ │
│ Business logic. Orchestration. Error handling. │
│ Stateless singletons. Import Core + Models. │
├─────────────────────────────────────────────────────────────────┤
│ Core Layer (core/) │
│ ┌─────────────────────┐ ┌─────────────────┐ │
│ │ passkey_manager │ │ dependencies │ │
│ │ (WebAuthn ceremony) │ │ (current_user) │ │
│ └─────────────────────┘ └─────────────────┘ │
│ ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│ │ websocket_manager│ │ surreal_mgr │ │ redis_manager │ │
│ │ (connection pool)│ │ (live query) │ │ (sessions+chal.) │ │
│ └──────────────────┘ └──────────────┘ └──────────────────┘ │
│ ┌──────────────────┐ ┌──────────────────────────────────┐ │
│ │ exceptions.py │ │ exception_handlers.py │ │
│ └──────────────────┘ └──────────────────────────────────┘ │
│ Database clients. WebSocket pool. Auth dependencies. │
│ No server-side cryptography lives here — that's the client. │
│ No imports from API or Services. │
├─────────────────────────────────────────────────────────────────┤
│ Model Layer (models/) │
│ ┌──────┐ ┌────────────┐ ┌─────────────┐ ┌──────────────┐ │
│ │ User │ │ Credential │ │ IdentityKey │ │ SignedPrekey │ │
│ └──────┘ └────────────┘ └─────────────┘ └──────────────┘ │
│ ┌──────────────┐ │
│ │ OneTimePrekey│ Public-only prekey material — no privates. │
│ └──────────────┘ │
│ SQLModel ORM classes. Data structures. Validation. │
│ Import only config constants and Base. │
├─────────────────────────────────────────────────────────────────┤
│ Schema Layer (schemas/) │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│ │ auth.py │ │ common.py│ │ rooms.py │ │ websocket.py │ │
│ └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
│ ┌──────────────┐ │
│ │ surreal.py │ │
│ └──────────────┘ │
│ Pydantic request/response models. API contracts. │
│ Serialization. No business logic. │
└─────────────────────────────────────────────────────────────────┘
```

**Regole di import:** API importa Services. Services importa Core + Models. Core non importa nulla da API o Services. Models non importa nulla tranne le costanti di configurazione e la classe Base. Schemas non importa nulla tranne le costanti di configurazione.

L'unica eccezione è `websocket_service.py`, che importa `connection_manager` da Core e `message_service` da Services. Questo è accettabile perché la gestione dei messaggi WebSocket si trova a cavallo tra entrambi i livelli.

---

## Modelli di dati

### Schema PostgreSQL (autenticazione + chiavi)

Tutti i modelli PostgreSQL ereditano da `BaseDBModel` (`models/Base.py`), che fornisce i campi timestamp `created_at` e `updated_at` con consapevolezza del fuso orario.

**users** (`models/User.py`)

```
┌────────────────────────────────────────────────────────────┐
│ users │
├──────────────────────┬──────────────────┬──────────────────┤
│ Column │ Type │ Constraints │
├──────────────────────┼──────────────────┼──────────────────┤
│ id │ UUID │ PK, default uuid4│
│ username │ VARCHAR(50) │ UNIQUE, INDEX │
│ display_name │ VARCHAR(100) │ NOT NULL │
│ is_active │ BOOLEAN │ default True │
│ is_verified │ BOOLEAN │ default False │
│ identity_key │ VARCHAR(500) │ nullable │
│ signed_prekey │ VARCHAR(500) │ nullable │
│ signed_prekey_sig │ VARCHAR(500) │ nullable │
│ one_time_prekeys │ TEXT │ nullable │
│ created_at │ TIMESTAMP(tz) │ NOT NULL │
│ updated_at │ TIMESTAMP(tz) │ NOT NULL │
├──────────────────────┴──────────────────┴──────────────────┤
│ Relationships: credentials (1:many → Credential) │
└────────────────────────────────────────────────────────────┘
```

**credentials** (`models/Credential.py`)

```
┌────────────────────────────────────────────────────────────┐
│ credentials │
├──────────────────────┬──────────────────┬──────────────────┤
│ Column │ Type │ Constraints │
├──────────────────────┼──────────────────┼──────────────────┤
│ id │ INTEGER │ PK, autoincrement│
│ credential_id │ VARCHAR(512) │ UNIQUE, INDEX │
│ public_key │ VARCHAR(1024) │ NOT NULL │
│ sign_count │ INTEGER │ default 0 │
│ aaguid │ VARCHAR(64) │ nullable │
│ backup_eligible │ BOOLEAN │ default False │
│ backup_state │ BOOLEAN │ default False │
│ attestation_type │ VARCHAR(50) │ nullable │
│ transports │ VARCHAR(200) │ nullable │
│ user_id │ UUID │ FK → users.id │
│ device_name │ VARCHAR(100) │ nullable │
│ last_used_at │ TIMESTAMP(tz) │ nullable │
│ created_at │ TIMESTAMP(tz) │ NOT NULL │
│ updated_at │ TIMESTAMP(tz) │ NOT NULL │
├──────────────────────┴──────────────────┴──────────────────┤
│ Relationships: user (many:1 → User) │
└────────────────────────────────────────────────────────────┘
```

**identity_keys** (`backend/app/models/IdentityKey.py`)

```
┌────────────────────────────────────────────────────────────┐
│ identity_keys │
├──────────────────────┬──────────────────┬──────────────────┤
│ Column │ Type │ Constraints │
├──────────────────────┼──────────────────┼──────────────────┤
│ id │ INTEGER │ PK, autoincrement│
│ user_id │ UUID │ FK → users.id │
│ │ │ UNIQUE, INDEX │
│ public_key │ VARCHAR(64) │ NOT NULL (X25519)│
│ public_key_ed25519 │ VARCHAR(64) │ NOT NULL │
│ created_at │ TIMESTAMP(tz) │ NOT NULL │
│ updated_at │ TIMESTAMP(tz) │ NOT NULL │
└────────────────────────────────────────────────────────────┘
```

Solo chiavi pubbliche. Il server non memorizza mai le chiavi private di identità — queste risiedono esclusivamente nell'IndexedDB del client.

**signed_prekeys** (`backend/app/models/SignedPrekey.py`)

```
┌────────────────────────────────────────────────────────────┐
│ signed_prekeys │
├──────────────────────┬──────────────────┬──────────────────┤
│ Column │ Type │ Constraints │
├──────────────────────┼──────────────────┼──────────────────┤
│ id │ INTEGER │ PK, autoincrement│
│ user_id │ UUID │ FK → users.id │
│ key_id │ INTEGER │ INDEX │
│ public_key │ VARCHAR(64) │ NOT NULL (X25519)│
│ signature │ VARCHAR(128) │ NOT NULL (Ed25519)│
│ is_active │ BOOLEAN │ default True │
│ expires_at │ TIMESTAMP(tz) │ nullable │
│ created_at │ TIMESTAMP(tz) │ NOT NULL │
│ updated_at │ TIMESTAMP(tz) │ NOT NULL │
└────────────────────────────────────────────────────────────┘
```

Rotazione: il client genera una nuova signed prekey all'incirca ogni 48 ore (`SIGNED_PREKEY_ROTATION_HOURS`) e riesegue `uploadKeys` per pubblicare la nuova chiave pubblica + firma. La SPK attiva precedente viene marcata `is_active=False` da `store_client_keys`, così che l'endpoint del bundle serva sempre la più recente.

**one_time_prekeys** (`backend/app/models/OneTimePrekey.py`)

```
┌────────────────────────────────────────────────────────────┐
│ one_time_prekeys │
├──────────────────────┬──────────────────┬──────────────────┤
│ Column │ Type │ Constraints │
├──────────────────────┼──────────────────┼──────────────────┤
│ id │ INTEGER │ PK, autoincrement│
│ user_id │ UUID │ FK → users.id │
│ key_id │ INTEGER │ INDEX │
│ public_key │ VARCHAR(64) │ NOT NULL (X25519)│
│ is_used │ BOOLEAN │ default False │
│ │ │ INDEX │
│ created_at │ TIMESTAMP(tz) │ NOT NULL │
│ updated_at │ TIMESTAMP(tz) │ NOT NULL │
└────────────────────────────────────────────────────────────┘
```

Il client carica le OPK pubbliche in batch da 100 (`DEFAULT_ONE_TIME_PREKEY_COUNT` in `frontend/src/types/encryption.ts`) e le ricarica quando il suo conteggio locale di OPK non usate scende sotto la metà. Quando `get_prekey_bundle` ne consegna una, imposta `is_used=True` così che la stessa OPK non venga mai offerta a due mittenti.

> Nota: non esistono tabelle `ratchet_states` o `skipped_message_keys`. Il Double Ratchet — inclusa la sua cache delle chiavi saltate — gira interamente nel browser e viene reso persistente in IndexedDB da `frontend/src/crypto/key-store.ts:saveRatchetState`. Mettere quello stato sul server significherebbe che il server detiene le chiavi simmetriche, vanificando l'E2EE.
│ created_at │ TIMESTAMP(tz) │ NOT NULL │
│ updated_at │ TIMESTAMP(tz) │ NOT NULL │
└────────────────────────────────────────────────────────────┘
```

Queste memorizzano le message key per la consegna fuori ordine. Quando il ratchet di ricezione avanza oltre un numero di messaggio che non è ancora stato ricevuto, la chiave per quel messaggio viene calcolata e memorizzata qui. Quando il messaggio saltato finalmente arriva, la sua chiave viene cercata e consumata.

### Schema SurrealDB (messaggi + presenza)

SurrealDB è schemaless, ma queste sono le strutture di documento che l'applicazione crea:

**messages** (creati tramite `surreal_manager.py`)

```
{
 id: "messages:ulid_here", // SurrealDB auto-generated
 sender_id: "uuid_string",
 recipient_id: "uuid_string",
 room_id: "rooms:ulid_here" | null,
 ciphertext: "base64url_encoded_bytes",
 nonce: "base64url_encoded_bytes",
 header: "{\"dh_public_key\":\"...\",\"message_number\":0,...}",
 sender_username: "alice",
 created_at: "2026-01-15T10:30:00Z",
 updated_at: "2026-01-15T10:30:00Z"
}
```

**presence** (creati tramite `surreal_manager.py`)

```
{
 id: "presence:`user_uuid`", // user-specific record ID
 user_id: "uuid_string",
 status: "online" | "away" | "offline",
 last_seen: "2026-01-15T10:30:00Z",
 updated_at: "time::now"
}
```

**rooms** (creati tramite `surreal_manager.py`)

```
{
 id: "rooms:ulid_here",
 name: "Room Name",
 type: "direct" | "group",
 members: ["uuid1", "uuid2"],
 created_at: "2026-01-15T10:30:00Z",
 updated_at: "2026-01-15T10:30:00Z"
}
```

**room_participants** (creati tramite `surreal_manager.py`)

```
{
 id: "room_participants:ulid_here",
 room_id: "rooms:ulid_here",
 user_id: "uuid_string",
 role: "member" | "admin",
 joined_at: "2026-01-15T10:30:00Z"
}
```

### Chiavi Redis

```
webauthn:reg_challenge:{username} → 32 bytes (hex-encoded), TTL 600s
webauthn:auth_challenge:{username} → 32 bytes (hex-encoded), TTL 600s
```

Questi sono gli unici due pattern di chiave attualmente in uso. Entrambi usano il pattern GET+DELETE con pipeline Redis (`redis_manager.py`) per un consumo atomico monouso. Il TTL garantisce che le challenge obsolete vengano ripulite automaticamente anche se il client non completa mai il flusso.

---

## Architettura di sicurezza

### Threat model

**Da cosa protegge il sistema:**

| Minaccia | Protezione | Come |
|---|---|---|
| Server compromesso | Cifratura E2E | Il server memorizza ciphertext, non ha mai le chiavi |
| Intercettazione sulla rete | TLS + E2E | Anche senza TLS, i messaggi sono cifrati con AES-256-GCM |
| Dump del database rubato | Separazione delle chiavi | PostgreSQL contiene solo chiavi pubbliche (percorso lato client). Le chiavi private risiedono nell'IndexedDB del browser. Le OPK sono monouso. |
| Phishing / furto di credenziali | WebAuthn | Le credenziali sono legate all'origine. Non possono essere riprodotte su un dominio diverso. |
| Replay attack | Nonce + contatori | I nonce AES-GCM sono casuali. I numeri dei messaggi sono sequenziali. Le challenge WebAuthn hanno un TTL. |
| Clonazione dell'authenticator | Contatore di firma | `backend/app/core/passkey/passkey_manager.py` verifica che il contatore aumenti strettamente. In caso contrario, l'autenticazione fallisce con un errore di rilevamento dei cloni. |
| Manomissione dei messaggi | AEAD | AES-256-GCM fornisce cifratura autenticata. Il ciphertext manomesso non supera il controllo del tag GCM (`frontend/src/crypto/double-ratchet.ts`). |
| Compromissione di una chiave (singola) | Forward secrecy | Il Double Ratchet genera regolarmente nuove chiavi DH. Compromettere una chain key rivela solo i messaggi futuri di quella catena, non i messaggi passati. |

**Cosa è fuori dal perimetro:**

| Minaccia | Perché |
|---|---|
| Dispositivo client compromesso | Se l'attaccante ha accesso al browser, ha accesso a IndexedDB (chiavi private) e può leggere il testo in chiaro. Non c'è difesa contro un endpoint completamente compromesso. |
| Attacchi side-channel sulla crittografia | La funzione `constantTimeEqual` in `frontend/src/crypto/primitives.ts` è tutta la mitigazione degli attacchi di timing presente. Una resistenza completa ai side channel richiederebbe implementazioni a tempo costante di tutte le primitive crittografiche, che la WebCrypto API in genere fornisce ma non garantisce. |
| Analisi dei metadati | Il server sa chi scrive a chi, quando, con quale frequenza e le dimensioni dei messaggi. Solo il contenuto è protetto, non i metadati. Proteggere i metadati richiederebbe qualcosa come mixnet o onion routing, che non è implementato. |
| Quantum computing | X25519 ed Ed25519 non sono post-quantum. Un computer quantistico sufficientemente potente potrebbe romperli. Lo scambio di chiavi post-quantum (es. ML-KEM/Kyber) non è implementato. |
| Supply chain dell'authenticator compromessa | Se l'hardware stesso dell'authenticator contiene una backdoor, WebAuthn non può rilevarlo. Il campo AAGUID può identificare il modello dell'authenticator ma non verificarne l'integrità. |

### Difesa in profondità

```
Layer 1: Transport Security (TLS/HTTPS via Nginx)
 │
 │ Protects: Data in transit between client and server
 │ Mechanism: TLS certificate, HTTPS enforcement
 │ Configuration: Nginx reverse proxy with SSL termination
 │
 ▼
Layer 2: Authentication (WebAuthn/Passkeys)
 │
 │ Protects: Identity verification, prevents impersonation
 │ Mechanism: Public-key cryptography, hardware-bound keys
 │ Key files: backend/app/core/passkey/passkey_manager.py, backend/app/services/auth_service.py
 │ Redis: Challenge storage with 600s TTL, one-time consumption
 │
 ▼
Layer 3: Key Exchange (X3DH Protocol)
 │
 │ Protects: Initial shared secret establishment
 │ Mechanism: 3-4 Diffie-Hellman operations + HKDF
 │ Key files: frontend/src/crypto/x3dh.ts, backend/app/services/prekey_service.py
 │ Properties: Asynchronous (works even if recipient offline)
 │ Deniable (either party could have forged the exchange)
 │
 ▼
Layer 4: Message Encryption (Double Ratchet + AES-256-GCM)
 │
 │ Protects: Message confidentiality and integrity
 │ Mechanism: Symmetric ratchet (HMAC chains) + DH ratchet
 │ Key files: frontend/src/crypto/double-ratchet.ts, frontend/src/crypto/primitives.ts
 │ Properties: Forward secrecy per-message
 │ Future secrecy (self-healing after compromise)
 │
 ▼
Layer 5: Key Lifecycle Management
 │
 │ Protects: Limits blast radius of any single key compromise
 │ Mechanism: SPK rotation every 48h, OPK single-use, old SPK cleanup
 │ Key files: backend/app/services/prekey_service.py (rotation),
 │ backend/app/services/prekey_service.py (cleanup),
 │ backend/app/services/prekey_service.py (replenishment)
 │ Constants: SIGNED_PREKEY_ROTATION_HOURS=48 (config.py)
 │ SIGNED_PREKEY_RETENTION_DAYS=7 (config.py)
 │ DEFAULT_ONE_TIME_PREKEY_COUNT=100 (config.py)
 │
 ▼
Layer 6: Rate Limiting and Abuse Prevention
 │
 Protects: Against brute force and DoS
 Mechanism: Per-user message rate limits, auth attempt limits
 Constants: RATE_LIMIT_MESSAGES_PER_MINUTE=60 (config.py)
 RATE_LIMIT_AUTH_ATTEMPTS=5 (config.py)
 WS_MAX_CONNECTIONS_PER_USER=5 (config.py)
```

---

## Configurazione

Tutta la configurazione vive in `config.py`. La classe `Settings` eredita da `pydantic_settings.BaseSettings` e carica i valori dalle variabili d'ambiente e dal file `.env`.

### Impostazioni dell'applicazione

| Variabile | Default | Descrizione |
|---|---|---|
| `ENV` | `"development"` | `development`, `production` o `testing` |
| `DEBUG` | `True` | Abilita il logging echo di SQL e gli endpoint della documentazione |
| `APP_NAME` | `"encrypted-p2p-chat"` | Nome dell'applicazione nei metadati |
| `SECRET_KEY` | (obbligatoria) | Segreto dell'applicazione, nessun default |

### Impostazioni PostgreSQL

| Variabile | Default | Descrizione |
|---|---|---|
| `POSTGRES_HOST` | `"localhost"` | Host del database |
| `POSTGRES_PORT` | `5432` | Porta del database |
| `POSTGRES_DB` | `"chat_auth"` | Nome del database |
| `POSTGRES_USER` | `"chat_user"` | Utente del database |
| `POSTGRES_PASSWORD` | `""` | Password del database |
| `DATABASE_URL` | (costruita automaticamente) | Stringa di connessione completa, assemblata da `field_validator` |
| `DB_POOL_SIZE` | `20` | Dimensione del connection pool di SQLAlchemy |
| `DB_MAX_OVERFLOW` | `40` | Connessioni di overflow del pool |

Il validator `assemble_db_connection` (`config.py`) costruisce l'URL a partire dai componenti se `DATABASE_URL` non è impostata esplicitamente. L'URL usa lo schema `postgresql+asyncpg://` per le connessioni asincrone.

### Impostazioni SurrealDB

| Variabile | Default | Descrizione |
|---|---|---|
| `SURREAL_HOST` | `"localhost"` | Host di SurrealDB |
| `SURREAL_PORT` | `8000` | Porta di SurrealDB |
| `SURREAL_USER` | `"root"` | Utente di SurrealDB |
| `SURREAL_PASSWORD` | (obbligatoria) | Password di SurrealDB |
| `SURREAL_NAMESPACE` | `"chat"` | Namespace di SurrealDB |
| `SURREAL_DATABASE` | `"production"` | Database di SurrealDB |
| `SURREAL_URL` | (costruita automaticamente) | URL WebSocket, assemblato |

### Impostazioni Redis

| Variabile | Default | Descrizione |
|---|---|---|
| `REDIS_HOST` | `"localhost"` | Host di Redis |
| `REDIS_PORT` | `6379` | Porta di Redis |
| `REDIS_PASSWORD` | `""` | Password di Redis (opzionale) |
| `REDIS_URL` | (costruita automaticamente) | URL di connessione, assemblato |

### Impostazioni WebAuthn

| Variabile | Default | Descrizione |
|---|---|---|
| `RP_ID` | `"localhost"` | ID della Relying Party (dominio) |
| `RP_NAME` | `"Encrypted P2P Chat"` | Nome visualizzato della Relying Party |
| `RP_ORIGIN` | `"http://localhost:3000"` | Origine attesa per la verifica delle credenziali |

`RP_ID` deve corrispondere al dominio che vede il browser. In produzione, sarebbe `"chat.example.com"`. Se non corrisponde, la verifica WebAuthn fallisce.

### Impostazioni WebSocket e di sicurezza

| Variabile | Default | Descrizione |
|---|---|---|
| `WS_HEARTBEAT_INTERVAL` | `30` | Secondi tra i ping di heartbeat |
| `WS_MAX_CONNECTIONS_PER_USER` | `5` | Numero massimo di connessioni WebSocket simultanee |
| `KEY_ROTATION_DAYS` | `90` | Periodo generale di rotazione delle chiavi |
| `MAX_SKIPPED_MESSAGE_KEYS` | `1000` | Numero massimo di messaggi fuori ordine per ratchet |
| `RATE_LIMIT_MESSAGES_PER_MINUTE` | `60` | Rate limit dei messaggi per utente |
| `RATE_LIMIT_AUTH_ATTEMPTS` | `5` | Numero massimo di tentativi di autenticazione prima del blocco |
| `CORS_ORIGINS` | `["http://localhost:3000", "http://localhost:5173"]` | Origini CORS consentite |

---

## Considerazioni sulle prestazioni

### Pooling delle connessioni al database

Pool PostgreSQL: 20 connessioni di base + 40 di overflow = 60 connessioni simultanee al massimo (`config.py`). L'engine usa `pool_pre_ping=True` (`models/Base.py`) per rilevare e sostituire le connessioni obsolete prima che le query falliscano.

Le sessioni async di SQLAlchemy vengono create per richiesta tramite la dipendenza `get_session` (`models/Base.py`). Le sessioni vengono riciclate automaticamente dopo ogni richiesta.

Pool Redis: 50 connessioni (`redis_manager.py`). Il pooling delle connessioni è gestito da `redis.asyncio.ConnectionPool`.

SurrealDB: una singola connessione WebSocket persistente per istanza dell'applicazione (`surreal_manager.py`). Il metodo `ensure_connected` (`surreal_manager.py`) si riconnette in modo lazy se la connessione cade.

### Throughput dei messaggi

Rate limit di 60 messaggi al minuto per utente (`config.py`). Questo si traduce in un messaggio al secondo sostenuto, il che è ragionevole per un'applicazione di chat.

Heartbeat WebSocket ogni 30 secondi (`config.py`). È abbastanza frequente da rilevare rapidamente le connessioni morte ma non così frequente da sprecare banda sulle connessioni inattive.

Le live query di SurrealDB sono basate su push (`surreal_manager.py`). Quando viene creato un nuovo messaggio, SurrealDB lo spinge immediatamente al callback sottoscritto. Non c'è un intervallo di polling, quindi la latenza è limitata al round-trip di rete più il tempo di elaborazione di SurrealDB.

### Overhead della cifratura

Costo approssimativo della cifratura per messaggio (basato sulle prestazioni tipiche di X25519/AES-256-GCM):

```
X25519 DH exchange: ~0.1ms
HMAC-SHA256 chain step: ~0.01ms
AES-256-GCM encrypt: ~0.001ms per KB (for typical chat messages)
HKDF derivation: ~0.01ms

Total per-message: < 1ms
```

È trascurabile rispetto alla latenza di rete (tipicamente 10-100ms). L'overhead della cifratura non è un collo di bottiglia per questa applicazione.

### Limiti di connessione

5 connessioni WebSocket per utente (`config.py`). Questo supporta un numero ragionevole di dispositivi (telefono, laptop, tablet, desktop, browser secondario) senza permettere a un singolo utente di esaurire la capacità WebSocket del server.

Il `ConnectionManager` consegna ogni messaggio a tutte le connessioni di un utente (`websocket_manager.py`), quindi il costo del multi-dispositivo scala linearmente con il numero di connessioni per utente.

---

## Decisioni di design

### Perché tre database?

È la domanda più ovvia sull'architettura, quindi merita una risposta diretta.

**PostgreSQL** memorizza i dati relazionali che richiedono transazioni ACID: utenti con username univoci, credenziali con foreign key verso gli utenti, identity key con vincoli di unicità per utente, stati del ratchet che devono essere aggiornati in modo atomico. Questi dati hanno relazioni complesse (un utente ha molte credenziali, uno stato del ratchet fa riferimento a due utenti, ecc.) e beneficiano dell'integrità referenziale di SQL.

**SurrealDB** memorizza i messaggi e i dati di presenza. La funzionalità chiave sono le live query: quando viene creato un messaggio, SurrealDB lo spinge ai sottoscrittori in tempo reale senza polling. Questo è il cuore dell'esperienza di chat. Potresti farlo con LISTEN/NOTIFY di PostgreSQL, ma le live query di SurrealDB sono più naturali per questo pattern e lo schema è orientato ai documenti, il che si adatta meglio alla struttura variabile dei messaggi cifrati.

**Redis** memorizza i dati effimeri che dovrebbero scadere automaticamente: challenge WebAuthn (TTL di 600s), contatori di rate limit. Non vorresti interrogare periodicamente PostgreSQL per ripulire le challenge scadute. Redis gestisce questo nativamente con il TTL delle chiavi.

Il compromesso è la complessità operativa. Gestire tre database significa tre cose che possono guastarsi, tre cose di cui fare il backup, tre cose da monitorare. La giustificazione è che ciascun database fa ciò che sa fare meglio, e cercare di far svolgere a un solo database tutti e tre i compiti creerebbe compromessi peggiori (es. polling per gli aggiornamenti in tempo reale, job manuali di scadenza per le challenge).

### Perché WebAuthn invece di JWT + password?

Le passkey WebAuthn sono resistenti al phishing per design. La credenziale è legata all'origine dell'RP, quindi non può essere usata su un dominio falso. Non ci sono password da rubare in caso di violazione del database. La chiave privata non lascia mai l'hardware dell'authenticator.

Il compromesso è il supporto da parte di browser e dispositivi. WebAuthn richiede un browser moderno e un authenticator (Touch ID, Windows Hello, YubiKey, ecc.). Il recupero dell'account è più difficile: se perdi il tuo unico authenticator, perdi l'accesso. Questo può essere mitigato registrando più authenticator.

### Perché SolidJS invece di React?

SolidJS usa una reattività a grana fine. Quando arriva un messaggio, vengono aggiornati solo gli specifici elementi DOM che dipendono da quel messaggio. React farebbe il diff dell'intero albero del virtual DOM per la lista dei messaggi. Per un'applicazione di chat in tempo reale in cui i messaggi arrivano frequentemente, questa differenza conta.

SolidJS ha anche una dimensione del bundle inferiore a React, il che aiuta il tempo di caricamento iniziale. La libreria nanostores fornisce uno store reattivo indipendente dal framework che si integra in modo naturale con il modello di reattività di SolidJS.

Il compromesso è la dimensione dell'ecosistema. React ha molte più librerie, tutorial e risposte su Stack Overflow. SolidJS è più piccolo e occasionalmente devi costruire qualcosa per cui React avrebbe una libreria.

### Perché solo un percorso di cifratura lato client?

Una bozza precedente di questo progetto prevedeva due percorsi di cifratura — uno lato server che caricava lo stato del ratchet da PostgreSQL e cifrava sul backend, e un passthrough lato client che memorizzava ciphertext opaco. L'audit (vedi `docs/plans/`) ha eliminato completamente il percorso lato server, incluse le sue tabelle `RatchetState` e `SkippedMessageKey` e l'intero modulo Python `app/core/encryption/`.

Perché? Perché finché esiste un'implementazione lato server di X3DH/Double Ratchet, esistono le colonne dello schema di cui ha bisogno (chiavi private, root del ratchet), e qualsiasi chiamante può invocarla — il server ha la capacità tecnica di leggere i messaggi degli utenti. "Marcato come deprecato" è un commento, non una difesa. L'unico modo per dichiarare onestamente la crittografia end-to-end è rendere fisicamente impossibile la partecipazione del server. Quindi ora `message_service.py` ha esattamente un metodo, `store_encrypted_message`, e non fa altro che passare `ciphertext` / `nonce` / `header` a SurrealDB.

---

## Architettura di deployment

### Sviluppo

```
docker compose -f dev.compose.yml up

┌───────────────────────────────────────────────────────────────┐
│ Docker Network (chat_network_dev) │
│ │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│ │ PostgreSQL │ │ SurrealDB │ │ Redis │ │
│ │ 16-alpine │ │ latest │ │ 8-alpine │ │
│ │ :5432→5432 │ │ :8000→8001 │ │ :6379→6379 │ │
│ │ │ │ file://data│ │ appendonly │ │
│ └─────────────┘ └─────────────┘ └─────────────┘ │
│ │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│ │ FastAPI │ │ Vite Dev │ │ Nginx │ │
│ │ :8000→8000 │ │ :5173→5173 │ │ :80→80 │ │
│ │ uvicorn │ │ HMR enabled│ │ reverse │ │
│ │ --reload │ │ hot module │ │ proxy │ │
│ │ vol: ./back │ │ vol: ./fro │ │ dev.nginx │ │
│ │ end │ │ ntend │ │ config │ │
│ └─────────────┘ └─────────────┘ └─────────────┘ │
│ │
│ Source: dev.compose.yml │
└───────────────────────────────────────────────────────────────┘
```

In sviluppo, tutti i servizi espongono le porte all'host per l'accesso diretto e il debug. Il backend monta come volume la directory `./backend` e gira con uvicorn `--reload` per il live reloading. Il frontend monta come volume `./frontend` ed esegue il dev server di Vite con HMR (Hot Module Replacement). Nginx sta davanti come reverse proxy, rispecchiando la topologia di produzione.

### Produzione

```
docker compose up

┌───────────────────────────────────────────────────────────────┐
│ Docker Network (chat_network) │
│ │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Nginx (:80, :443) │ │
│ │ │ │
│ │ /api/* ────────► upstream: backend:8000 (HTTP) │ │
│ │ /ws ────────► upstream: backend:8000 (WebSocket) │ │
│ │ /* ────────► Static SolidJS build (from image) │ │
│ │ │ │
│ └──────────────────────────┬───────────────────────────────┘ │
│ │ │
│ ┌──────────────────────────┼──────────────────────────────┐ │
│ │ │ │ │
│ │ ┌─────────────┐ ┌─────┴─────┐ ┌─────────────┐ │ │
│ │ │ PostgreSQL │ │ FastAPI │ │ SurrealDB │ │ │
│ │ │ 16-alpine │ │ gunicorn │ │ latest │ │ │
│ │ │ vol: data │ │ workers │ │ vol: data │ │ │
│ │ │ healthcheck │ │ no ports │ │ healthcheck│ │ │
│ │ │ restart: │ │ exposed │ │ restart: │ │ │
│ │ │ always │ │ to host │ │ always │ │ │
│ │ └─────────────┘ └───────────┘ └─────────────┘ │ │
│ │ │ │
│ │ ┌─────────────┐ │ │
│ │ │ Redis │ │ │
│ │ │ 8-alpine │ │ │
│ │ │ vol: data │ │ │
│ │ │ maxmem 2gb │ │ │
│ │ │ LRU evict │ │ │
│ │ └─────────────┘ │ │
│ │ │ │
│ └─────────────────────────────────────────────────────────┘ │
│ │
│ Source: compose.yml │
└───────────────────────────────────────────────────────────────┘
```

In produzione, il backend FastAPI non espone alcuna porta all'host. È accessibile solo tramite il container Nginx sulla rete Docker interna. I database usano volumi con nome per la persistenza e `restart: always` per il ripristino automatico. Il container del frontend serve i file statici di SolidJS precompilati tramite Nginx.

Il compose file (`compose.yml`) usa `depends_on` con `condition: service_healthy` per garantire che i database siano pronti prima che parta il backend. Ogni database ha un comando di healthcheck (PostgreSQL: `pg_isready`, SurrealDB: `/health`, Redis: `redis-cli ping`).

---

## Strategia di gestione degli errori

### Gerarchia di eccezioni personalizzate

Definita in `backend/app/core/exceptions.py`:

```
AppException (base)
 ├── UserExistsError → 409 Conflict
 ├── UserNotFoundError → 404 Not Found
 ├── UserInactiveError → 403 Forbidden
 ├── CredentialNotFoundError → 404 Not Found
 ├── CredentialVerificationError → 401 Unauthorized
 ├── ChallengeExpiredError → 400 Bad Request
 ├── DatabaseError → 500 Internal Server Error
 ├── AuthenticationError → 401 Unauthorized
 └── InvalidDataError → 400 Bad Request
```

Anche `slowapi.errors.RateLimitExceeded` viene gestita qui e mappata su 429 Too Many Requests.

### Exception handler

Registrati in `backend/app/core/exception_handlers.py` tramite `register_exception_handlers(app)`, chiamata da `factory.py`. Ogni tipo di eccezione è mappato su una funzione handler che restituisce una `JSONResponse` con il codice di stato HTTP appropriato.

Per gli errori sensibili dal punto di vista della sicurezza (`DatabaseError`), il body della risposta contiene un messaggio generico ("Internal server error") anziché il dettaglio effettivo dell'errore. Il dettaglio viene registrato nei log lato server ma non esposto al client.

### Formato degli errori WebSocket

Gli errori WebSocket vengono inviati al client come JSON:

```json
{
 "type": "error",
 "error_code": "invalid_json",
 "error_message": "Invalid JSON format"
}
```

I codici di errore includono: `max_connections`, `database_error`, `invalid_json`, `missing_type`, `unknown_type`, `processing_error`. Sono definiti inline in `websocket.py` e `websocket_manager.py` .

---

## Estendibilità

Dove aggiungere nuove funzionalità:

| Cosa | Dove | Passi |
|---|---|---|
| Nuovo endpoint API | Directory `api/` | 1. Crea il router in `api/new_feature.py` 2. Crea il servizio in `services/new_feature_service.py` 3. Registra il router in `factory.py` (dopo) |
| Nuovo modello di database | Directory `models/` | 1. Crea la classe del modello ereditando da `BaseDBModel` 2. Importala in un punto che viene caricato all'avvio (es. `models/__init__.py`) 3. `init_db` crea automaticamente la tabella tramite `SQLModel.metadata.create_all` |
| Nuovo algoritmo di cifratura | `frontend/src/crypto/` | 1. Aggiungi l'implementazione in `frontend/src/crypto/` 2. Aggiungi le costanti di configurazione in `config.py` 3. Collegalo in `message_service.py` o `crypto-service.ts` |
| Nuovo tipo di messaggio WebSocket | `websocket_service.py` | 1. Aggiungi la costante in `config.py` (WS_MESSAGE_TYPE_*) 2. Aggiungi il metodo handler in `WebSocketService` 3. Aggiungi il caso di instradamento in `route_message` |
| Nuova collezione SurrealDB | `surreal_manager.py` | 1. Aggiungi i metodi CRUD in `SurrealDBManager` 2. Aggiungi lo schema di risposta in `schemas/surreal.py` |

---

## Limitazioni

Queste sono limitazioni architetturali note, non bug:

1. **Nessuna cifratura per le chat di gruppo.** Il Double Ratchet è un protocollo a due parti. La cifratura delle chat di gruppo richiederebbe le Sender Keys (ciò che Signal usa per i gruppi, dove ogni membro mantiene un ratchet separato con ogni altro membro) oppure il protocollo MLS (Messaging Layer Security). Nessuno dei due è implementato.

2. **Nessuno scambio di chiavi post-quantum.** X25519 è vulnerabile all'algoritmo di Shor su un computer quantistico sufficientemente potente. Migrare a uno schema ibrido (X25519 + ML-KEM) renderebbe lo scambio di chiavi a prova di futuro, ma questo aggiunge complessità e le tempistiche della minaccia quantistica sono dibattute.

3. **Metadati non protetti.** Il server sa chi invia messaggi a chi, quando, con quale frequenza e la dimensione approssimativa di ogni messaggio. Solo il contenuto è cifrato. La protezione dei metadati richiederebbe tecniche come onion routing, padding o traffico fittizio, che aggiungono tutte una complessità e un costo prestazionale significativi.

4. **Nessuna eliminazione o scadenza dei messaggi.** Una volta che un messaggio è memorizzato in SurrealDB, vi resta a tempo indeterminato. Non c'è un TTL sui messaggi né una funzionalità "elimina per tutti". SurrealDB supporta il metodo `_schedule_room_deletion` per le room effimere (`surreal_manager.py`), ma questo opera a livello di room, non di messaggio.

5. **Deployment in una singola regione.** Il setup Docker Compose presuppone che tutti i servizi girino su una macchina o un cluster. Non c'è geo-distribuzione, nessuna CDN per il frontend e nessuna replica del database. Per un deployment di produzione che serve utenti in più regioni, dovresti aggiungere questi elementi.

6. **Nessuna coda dei messaggi offline.** Se un destinatario è offline quando viene inviato un messaggio, il messaggio viene memorizzato in SurrealDB ma consegnato solo quando il destinatario si connette e la live query scatta. Non c'è un meccanismo esplicito per recuperare i messaggi persi alla riconnessione, oltre al recupero da parte della live query. Il metodo `get_room_messages` (`surreal_manager.py`) esiste per recuperare la cronologia dei messaggi, ma il client deve chiamarlo esplicitamente.

---

## Riferimento ai file chiave

| Cosa | Dove |
|---|---|
| App factory | `backend/app/factory.py` |
| Lifespan (inizializzazione DB) | `backend/app/factory.py` (`lifespan`) |
| Impostazioni + costanti crittografiche | `backend/app/config.py` |
| Dipendenza di autenticazione | `backend/app/core/dependencies.py` (`current_user`) |
| Helper di sessione | `backend/app/core/dependencies.py` (`issue_session`, `revoke_session`) |
| Manager WebAuthn | `backend/app/core/passkey/passkey_manager.py` |
| Rilevamento dei cloni | `passkey_manager.PasskeyManager.verify_authentication` |
| Pool WebSocket | `backend/app/core/websocket_manager.py` |
| Client SurrealDB | `backend/app/core/surreal_manager.py` |
| Client Redis (sessioni, challenge) | `backend/app/core/redis_manager.py` |
| Tipi di eccezione | `backend/app/core/exceptions.py` |
| Exception handler | `backend/app/core/exception_handlers.py` |
| Memorizzazione pass-through dei messaggi | `backend/app/services/message_service.py` (`store_encrypted_message`) |
| Memorizzazione delle prekey pubbliche + lookup del bundle | `backend/app/services/prekey_service.py` |
| Servizio di autenticazione | `backend/app/services/auth_service.py` |
| Router dei messaggi WS (con verifica dell'appartenenza) | `backend/app/services/websocket_service.py` |
| Endpoint di autenticazione (incl. `/auth/me`, `/auth/logout`) | `backend/app/api/auth.py` |
| Endpoint di cifratura (bundle + upload) | `backend/app/api/encryption.py` |
| Room (CRUD subordinato all'appartenenza) | `backend/app/api/rooms.py` |
| Endpoint WebSocket (autenticato tramite cookie) | `backend/app/api/websocket.py` |
| Modello User (con `webauthn_user_handle`) | `backend/app/models/User.py` |
| Modello Credential | `backend/app/models/Credential.py` |
| Identity key pubbliche | `backend/app/models/IdentityKey.py` |
| Signed prekey pubbliche | `backend/app/models/SignedPrekey.py` |
| One-time prekey pubbliche | `backend/app/models/OneTimePrekey.py` |
| Engine DB + sessioni | `backend/app/models/Base.py` |
| Primitive crittografiche del client | `frontend/src/crypto/primitives.ts` |
| Orchestrazione crittografica del client | `frontend/src/crypto/crypto-service.ts` |
| Double Ratchet del client | `frontend/src/crypto/double-ratchet.ts` |
| X3DH del client | `frontend/src/crypto/x3dh.ts` |
| Key store IndexedDB del client | `frontend/src/crypto/key-store.ts` |
| Compose di produzione | `compose.yml` |
| Compose di sviluppo | `dev.compose.yml` |
