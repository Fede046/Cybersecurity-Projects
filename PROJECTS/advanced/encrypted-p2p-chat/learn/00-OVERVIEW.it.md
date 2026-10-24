# Encrypted P2P Chat

## Cos'è

Questa è un'applicazione di messaggistica peer-to-peer con crittografia end-to-end, di livello production, costruita sul Signal Protocol. Implementa X3DH (Extended Triple Diffie-Hellman) per lo scambio asincrono delle chiavi, l'algoritmo Double Ratchet per la cifratura dei messaggi con forward secrecy, e WebAuthn/Passkey per un'autenticazione resistente al phishing. Il server orchestra la consegna dei messaggi e memorizza blob cifrati, ma non vede mai messaggi in chiaro né chiavi private. Se qualcuno compromette il server, ottiene ciphertext e chiavi pubbliche. Nient'altro.

## Perché è importante

Il 22 gennaio 2020 le Nazioni Unite confermarono che il telefono di Jeff Bezos era stato compromesso tramite un messaggio WhatsApp. L'analisi forense di FTI Consulting fece risalire l'infezione a un file video inviato dall'account WhatsApp personale del principe ereditario saudita Mohammed bin Salman. Nel giro di poche ore dalla ricezione di quel messaggio, l'esfiltrazione di dati dal telefono aumentò di circa il 29.000 percento. Gli attaccanti usarono un exploit zero-day nello stack di rendering video di WhatsApp per installare uno spyware, probabilmente Pegasus di NSO Group. Ciò che rese questo attacco particolarmente istruttivo è ciò che non era: non era un fallimento della crittografia end-to-end. La cifratura E2E di WhatsApp (che usa il Signal Protocol) funzionò correttamente. Il contenuto del messaggio era cifrato in transito. Ma l'exploit avvenne sul dispositivo stesso, dopo la decifratura. Questa distinzione conta perché mostra esattamente dove la crittografia E2E traccia il suo confine. Protegge i dati in transito e a riposo sul server. Non protegge da endpoint compromessi. Comprendere questo confine è fondamentale per costruire sistemi di messaggistica cifrata e ragionarci sopra.

Facebook Messenger, al contrario, mostrò cosa succede quando la cifratura non è il default. Nel settembre 2019 fu trovato esposto online, senza protezione tramite password, un server contenente numeri di telefono collegati a centinaia di milioni di account Facebook. Il dataset includeva 419 milioni di record relativi a utenti di Stati Uniti, Regno Unito e Vietnam. Sebbene questo specifico leak riguardasse numeri di telefono e non il contenuto dei messaggi, il problema di fondo era strutturale: Facebook Messenger non abilitava la crittografia end-to-end di default. Le "Secret Conversations" esistevano come funzionalità opt-in, ma la stragrande maggioranza dei messaggi transitava sui server di Facebook in una forma che l'azienda poteva leggere. Questo significava che qualsiasi compromissione del server, mandato governativo o dipendente infedele poteva accedere direttamente al contenuto dei messaggi. Facebook non introdusse la cifratura E2E di default per Messenger fino alla fine del 2023. Per anni, miliardi di messaggi furono protetti solo dalla cifratura a livello di trasporto (TLS), che cifra i dati tra il client e il server ma lascia il server stesso come parte fidata con pieno accesso al testo in chiaro.

La controversia del 2020 sulla cifratura di Zoom dimostrò come il divario tra affermazioni di marketing e realtà crittografica possa erodere la fiducia degli utenti. I materiali di marketing e il white paper di sicurezza di Zoom affermavano che la piattaforma supportava la "end-to-end encryption". Il Citizen Lab dell'Università di Toronto indagò e scoprì che Zoom stava in realtà usando TLS per la cifratura del trasporto, con chiavi AES-128 generate dai server di Zoom stessa. Questo significava che Zoom aveva la capacità di decifrare qualsiasi riunione. Le chiavi venivano talvolta instradate attraverso server in Cina, anche per riunioni in cui tutti i partecipanti si trovavano in Nord America. Zoom alla fine chiuse un procedimento della FTC con un accordo e si impegnò a implementare una vera cifratura E2E, ma l'incidente mise in luce un punto critico: l'espressione "cifrato end-to-end" non significa nulla senza un'implementazione verificabile. In un vero sistema E2E, la generazione delle chiavi avviene sui dispositivi client, il server inoltra ciphertext, e in nessun momento il server ha accesso alle chiavi di decifratura. È esattamente ciò che implementa questo progetto.

Lo scandalo Cambridge Analytica del 2018, pur riguardando principalmente la raccolta di dati tramite la Graph API di Facebook anziché l'intercettazione di messaggi, rende evidente un principio più ampio: qualsiasi dato a cui un server può accedere, un server può farlo trapelare, venderlo o essere costretto a consegnarlo. Cambridge Analytica raccolse dati personali da circa 87 milioni di profili Facebook tramite un'app di quiz sulla personalità che sfruttava le politiche permissive di condivisione dei dati di Facebook. I dati furono usati per il targeting della pubblicità politica durante le elezioni presidenziali statunitensi del 2016 e il referendum sulla Brexit. La lezione per le applicazioni di messaggistica è semplice. Se il tuo server può leggere i messaggi, il tuo server è un rischio. Ogni dipendente con accesso al database, ogni governo con un'ordinanza del tribunale, ogni attaccante che viola la tua infrastruttura diventa una minaccia per la privacy degli utenti. Un design di server zero-knowledge, in cui il server gestisce l'instradamento e la memorizzazione del ciphertext ma non possiede mai le chiavi di decifratura, elimina questa intera categoria di rischio. È l'architettura usata da questo progetto.

Il Signal Protocol è oggi lo standard di riferimento per la messaggistica cifrata. Signal (l'applicazione) ne è stata la pioniera. WhatsApp lo ha adottato nel 2016, portando la crittografia end-to-end a oltre 2 miliardi di utenti. Google Messages lo usa per le chat RCS. Anche la migrazione di Messenger di Meta alla cifratura E2E di default nel dicembre 2023 ha attinto ai concetti del Signal Protocol. Le innovazioni centrali del protocollo, X3DH per lo stabilimento asincrono delle chiavi e il Double Ratchet per la forward secrecy per singolo messaggio, risolvono problemi che i precedenti sistemi di messaggistica cifrata (come l'email cifrata con PGP) avevano lasciato aperti per decenni. Questo progetto implementa entrambi i protocolli da zero, dandoti esperienza diretta con lo stesso meccanismo crittografico che protegge miliardi di conversazioni reali. E abbinandolo all'autenticazione WebAuthn/Passkey, elimina il vettore d'attacco numero uno per la compromissione degli account: il phishing delle password. Secondo il Verizon Data Breach Investigations Report del 2023, il phishing è responsabile di circa il 36% di tutte le violazioni di dati. WebAuthn rende il phishing strutturalmente impossibile perché l'authenticator è legato al dominio di origine. Una falsa pagina di login su un dominio diverso semplicemente non può attivare la credenziale corretta.

## Cosa imparerai

### Protocolli crittografici

- **Scambio di chiavi X3DH per lo stabilimento asincrono della sessione.** Il problema fondamentale dello scambio di chiavi Diffie-Hellman è che entrambe le parti devono essere online nello stesso momento. Se Alice vuole inviare un messaggio a Bob, ma Bob è offline, il DH classico non può funzionare. X3DH (Extended Triple Diffie-Hellman) risolve il problema facendo pre-pubblicare a Bob sul server un bundle di chiavi pubbliche: una identity key a lungo termine, una signed prekey a medio termine e un insieme di one-time prekey monouso. Quando Alice vuole iniziare una conversazione, scarica il bundle di Bob ed esegue tre (o quattro, se è disponibile una one-time prekey) calcoli DH per derivare un segreto condiviso. Bob può completare la sua metà dello scambio quando torna online. **Tutta la matematica di X3DH gira nel browser** in `frontend/src/crypto/x3dh.ts` (`initiateX3DH` e `receiveX3DH`); il backend si limita a memorizzare e servire il bundle pubblico in `backend/app/services/prekey_service.py`. X25519 è usato per Diffie-Hellman, Ed25519 per le firme delle prekey.

- **Algoritmo Double Ratchet per forward secrecy e post-compromise security.** Una volta che X3DH ha stabilito un segreto condiviso iniziale, il Double Ratchet subentra per la cifratura continua dei messaggi. Combina due meccanismi di ratchet: un ratchet a chiave simmetrica (basato su HMAC) che avanza a ogni messaggio, e un ratchet Diffie-Hellman che avanza ogni volta che cambia il "turno" della conversazione. Il ratchet simmetrico fornisce la forward secrecy: se un attaccante ruba le tue chiavi attuali, non può decifrare i messaggi passati perché ogni message key viene derivata dalla chain key precedente e poi scartata. Il ratchet DH fornisce la post-compromise security (chiamata anche "future secrecy" o "self-healing"): anche se un attaccante compromette completamente il tuo stato attuale, non appena invii un nuovo messaggio con una nuova coppia di chiavi DH, l'attaccante perde l'accesso perché non conosce la nuova chiave privata. **Il Double Ratchet gira interamente nel browser** in `frontend/src/crypto/double-ratchet.ts` (`encryptMessage` / `decryptMessage`); il backend si limita a inoltrare il ciphertext risultante.

- **Cifratura autenticata AES-256-GCM.** Ogni messaggio è cifrato con AES-256-GCM (Galois/Counter Mode), che fornisce sia confidenzialità sia integrità. GCM è un cipher AEAD (Authenticated Encryption with Associated Data), il che significa che produce un ciphertext e un authentication tag. Se qualcuno manomette il ciphertext, la verifica del tag fallisce e la decifratura viene rifiutata. Gli "associated data" in questo progetto sono l'header del messaggio serializzato (chiave pubblica DH, numero del messaggio, lunghezza della catena precedente), che lega ogni ciphertext allo stato del ratchet che lo ha prodotto. L'implementazione usa nonce casuali da 12 byte e chiavi da 32 byte derivate dalla catena del Double Ratchet. Lo vedrai in `aesGcmEncrypt` / `aesGcmDecrypt` in `frontend/src/crypto/primitives.ts`, chiamate dall'interno di `encryptMessage` / `decryptMessage`.

- **Derivazione delle chiavi HKDF-SHA256.** Gli output grezzi di Diffie-Hellman non sono adatti a essere usati direttamente come chiavi di cifratura. Hanno una distribuzione non uniforme e possono contenere distorsioni strutturali. HKDF (HMAC-based Key Derivation Function) estrae entropia dagli output DH e la espande in chiavi della lunghezza richiesta. Questo progetto usa HKDF-SHA256 ovunque: in X3DH per derivare il segreto condiviso iniziale dagli output DH concatenati (con il prefisso F `0xFF * 32` imposto dalla specifica e l'info `b'X3DH'`), e nella KDF_RK (derivazione della root key) del Double Ratchet, che usa la root key precedente come salt HKDF e il nuovo output DH come input keying material. Il passo della chain key (KDF_CK) usa HMAC-SHA256 con tag a singolo byte `0x01` (message key) e `0x02` (chain key successiva) secondo la specifica di Signal.

- **Firme digitali Ed25519 e Diffie-Hellman X25519.** Il progetto usa due operazioni su curva ellittica correlate ma distinte su Curve25519. X25519 è usato per lo scambio di chiavi Diffie-Hellman: due parti generano ciascuna una coppia di chiavi, si scambiano le chiavi pubbliche e calcolano un segreto condiviso. Ed25519 è usato per le firme digitali: il proprietario della identity key firma la propria signed prekey così che gli altri utenti possano verificare che sia stata davvero pubblicata da quell'identità, e non sostituita da un server malevolo. Le operazioni matematiche sono diverse (X25519 opera sulla forma di Montgomery della curva, Ed25519 sulla forma di Edwards), ma condividono la stessa curva sottostante. Ogni utente genera coppie di identity key X25519 ed Ed25519 separate in `generateIdentityKeyPair` (`frontend/src/crypto/x3dh.ts`); solo le metà pubbliche lasciano mai il dispositivo.

### Autenticazione e identità

- **Registrazione e autenticazione con passkey WebAuthn/FIDO2.** WebAuthn è uno standard W3C che permette agli utenti di autenticarsi usando la crittografia a chiave pubblica invece delle password. Durante la registrazione, il browser crea una nuova coppia di chiavi su un authenticator hardware o di piattaforma (una YubiKey, un lettore di impronte digitali, il secure enclave di un telefono). La chiave pubblica viene inviata al server. La chiave privata non lascia mai l'authenticator. Durante il login, il server invia una challenge casuale, l'authenticator la firma con la chiave privata e il server verifica la firma rispetto alla chiave pubblica memorizzata. Nessun segreto condiviso attraversa mai la rete. L'implementazione usa la libreria `py_webauthn` sul server (`backend/app/core/passkey/passkey_manager.py`) e l'API nativa del browser `navigator.credentials` sul client. Le opzioni di registrazione richiedono resident key (abilitando il login senza username) e **richiedono** la user verification (PIN/biometria), dato che questa è un'app di chat E2EE e l'accesso da un dispositivo incustodito non dovrebbe sbloccare la conversazione. Lo `user.id` WebAuthn è un `webauthn_user_handle` casuale da 64 byte secondo la specifica, memorizzato nella riga User e mai esposto ai client.

- **Crittografia a chiave pubblica per la verifica dell'identità.** Ogni utente del sistema ha due tipi di chiavi pubbliche: credenziali WebAuthn (per l'autenticazione con il server) e identity key del Signal Protocol (per la cifratura end-to-end con gli altri utenti). La credenziale WebAuthn dimostra al server che sei chi dichiari di essere. La identity key del Signal Protocol permette agli altri utenti di verificare le tue chiavi di cifratura. Sono coppie di chiavi separate che servono a scopi separati. La identity key è una coppia di chiavi Ed25519 che firma le signed prekey a medio termine dell'utente, permettendo ai destinatari di verificare che il bundle di prekey scaricato sia stato davvero pubblicato da quell'utente e non sostituito da un server compromesso.

- **Rilevamento dei cloni di authenticator tramite contatori di firma.** Ogni authenticator WebAuthn mantiene un contatore di firma che si incrementa a ogni cerimonia di autenticazione. Il server memorizza l'ultimo valore del contatore visto. Se il contatore torna indietro o resta uguale (quando sia il server sia l'authenticator tengono traccia di contatori diversi da zero), indica che l'authenticator potrebbe essere stato clonato. Si tratta di un noto meccanismo di sicurezza FIDO2. Il metodo `verify_authentication` in `passkey_manager.py` lo controlla esplicitamente: se `new_sign_count <= credential_current_sign_count`, solleva un `ValueError` con un messaggio sul possibile rilevamento di un authenticator clonato. Questo intercetta gli scenari in cui un attaccante ha duplicato fisicamente una chiave di sicurezza hardware.

### Architettura fullstack

- **Backend FastAPI asincrono con ORM SQLModel.** Il backend è costruito con FastAPI, usando ovunque il suo supporto async nativo con `asyncio` e `async/await`. I modelli del database usano SQLModel, che unisce le capacità ORM di SQLAlchemy con la validazione dei dati di Pydantic. Il pattern app factory in `factory.py` crea l'istanza FastAPI con un context manager di lifespan che gestisce l'avvio (inizializzazione di PostgreSQL, connessione a Redis, connessione a SurrealDB) e lo spegnimento (disconnessione pulita). Le route API sono organizzate in quattro router: `auth` (registrazione / login WebAuthn / `/auth/me` / `/auth/logout`), `encryption` (upload e lookup del bundle pubblico di prekey), `rooms` (CRUD delle chat room con accesso subordinato all'appartenenza) e `websocket` (trasporto dei messaggi in tempo reale, autenticato tramite lo stesso session cookie). Tutti gli endpoint non di autenticazione dipendono da `current_user` in `app/core/dependencies.py`, che risolve il session cookie su Redis. I rate limit vengono applicati tramite `slowapi`. Il backend usa `ORJSONResponse` come classe di risposta predefinita per una serializzazione JSON più veloce.

- **Frontend reattivo SolidJS con TypeScript.** Il frontend usa SolidJS, un framework UI reattivo con reattività a grana fine (niente virtual DOM). La gestione dello stato usa Nanostores con il binding `@nanostores/solid` e `@nanostores/persistent` per la persistenza in localStorage. Il routing usa `@solidjs/router`. La gestione dello stato lato server usa `@tanstack/solid-query` per il recupero, il caching e la sincronizzazione dei dati. La struttura dei componenti è organizzata in cartelle di dominio: `Auth` (AuthCard, AuthForm, PasskeyButton), `Chat` (ChatInput, ChatHeader, ConversationList, MessageBubble, MessageList, EncryptionBadge, OnlineStatus, TypingIndicator, UserSearch), `Layout` (AppShell, Header, Sidebar, ProtectedRoute) e `UI` (Avatar, Badge, Button, Dropdown, Input, Modal, Spinner, Toast, Tooltip). Lo styling usa Tailwind CSS v4 tramite il plugin Vite.

- **Messaggistica in tempo reale via WebSocket.** I messaggi vengono consegnati in tempo reale tramite connessioni WebSocket. Il `websocket_manager.py` del backend mantiene un pool di connessioni e gestisce il broadcasting. Il `websocket/websocket-manager.ts` del frontend gestisce il ciclo di vita della connessione, la riconnessione e gli heartbeat. I tipi di messaggio includono messaggi cifrati, indicatori di digitazione, aggiornamenti di presenza, conferme di lettura e notifiche di errore. Il protocollo WebSocket trasporta ciphertext tra i client; il server instrada i messaggi senza decifrarli. L'intervallo di heartbeat è configurabile (default 30 secondi) e ogni utente può avere fino a 5 connessioni concorrenti (per più dispositivi o schede).

- **Architettura multi-database: PostgreSQL, SurrealDB, Redis.** Il progetto usa tre database, ciascuno scelto per uno specifico pattern di accesso. PostgreSQL (tramite `asyncpg` e SQLModel) memorizza i dati di autenticazione, i profili utente, le credenziali WebAuthn e il materiale **pubblico** delle prekey (identity key pubbliche, signed prekey pubbliche + firme, one-time prekey pubbliche non usate). Nessuna chiave privata risiede mai lato server. SurrealDB memorizza i dati di chat in tempo reale: messaggi (solo ciphertext), appartenenza alle room e presenza. La sua funzionalità di live query abilita notifiche push in tempo reale senza polling. Redis memorizza lo stato effimero: il contesto di registrazione WebAuthn (challenge + user handle + display name), le challenge di autenticazione (indicizzate per byte della challenge per la sicurezza contro i replay), i session token (`session:<token> -> user_id` con TTL di 24h) e i contatori di rate limit di `slowapi`. Ogni database ha la propria classe manager (`Base.py` per PostgreSQL, `surreal_manager.py` per SurrealDB, `redis_manager.py` per Redis).

- **Containerizzazione Docker con reverse proxy Nginx.** L'intera applicazione gira in container Docker orchestrati da Docker Compose. Il compose file di sviluppo (`dev.compose.yml`) definisce sei servizi: PostgreSQL 16 Alpine, SurrealDB (latest), Redis 8 Alpine, il backend FastAPI, il dev server Vite per il frontend e Nginx Alpine come reverse proxy. Ogni database ha health check per garantire che il backend parta solo dopo che le sue dipendenze sono pronte. La configurazione di Nginx instrada il traffico HTTP e fa da proxy alle connessioni WebSocket. Ci sono Dockerfile separati per lo sviluppo (con hot-reload tramite volume mount) e la produzione (con build ottimizzate) in `conf/docker/dev/` e `conf/docker/prod/`.

### Security engineering

- **Design del server zero-knowledge.** Il server di questa applicazione è progettato in modo da non poter leggere i messaggi nemmeno se lo volesse. Le chiavi private vengono generate sui dispositivi client (`generateIdentityKeyPair` in `frontend/src/crypto/x3dh.ts`) e memorizzate nell'IndexedDB del browser (`frontend/src/crypto/key-store.ts`). Il server memorizza solo chiavi pubbliche (`backend/app/services/prekey_service.py:store_client_keys` scrive solo le metà pubbliche; le colonne dello schema per le chiavi private sono state eliminate) e blob di ciphertext in SurrealDB. Quando Alice invia un messaggio a Bob, `crypto-service.ts:encrypt` esegue il Double Ratchet localmente, passa il ciphertext al WebSocket, e `message_service.store_encrypted_message` del server esegue una scrittura di puro pass-through — non esiste alcun percorso di codice `decrypt` nel backend. La serializzazione dell'header che diventa gli associated data di AES-GCM lega ogni ciphertext allo stato del ratchet che lo ha prodotto, così un server compromesso non può reindirizzare ciphertext tra sessioni senza rompere l'autenticazione.

- **Rotazione delle chiavi e gestione del ciclo di vita.** Le chiavi crittografiche in questo sistema hanno cicli di vita definiti, tutti guidati dal client. Le one-time prekey sono monouso: quando un mittente recupera il bundle di Bob, la OPK scelta viene marcata `is_used=True` lato server (così non può essere consegnata due volte), ma la OPK privata risiede solo nell'IndexedDB di Bob. Il client mantiene un pool di 100 OPK e lo rifornisce (ricaricando le nuove OPK pubbliche) quando il numero di quelle non usate scende sotto la metà (`crypto-service.ts:replenishOneTimePreKeys`). Le signed prekey ruotano ogni 48 ore; il client genera localmente una nuova SPK e riesegue `uploadKeys` per pubblicare la nuova chiave pubblica + firma. Le identity key sono a lungo termine e intenzionalmente non ruotano — sono il trust anchor che firma la SPK. Lo stato del ratchet è per conversazione, avanza a ogni messaggio e viene serializzato nello store dei ratchet in IndexedDB dopo ogni cifratura/decifratura. Le message key saltate (per la consegna fuori ordine) risiedono nello stato del ratchet in memoria, con un limite di `MAX_SKIP_MESSAGE_KEYS`.

- **Threat modeling per applicazioni di messaggistica.** Costruire questo progetto ti espone allo specifico threat model che il Signal Protocol è progettato per affrontare. Le minacce principali includono: un server compromesso che tenta di leggere o modificare i messaggi (mitigato dalla cifratura E2E e dal binding degli associated data), un attaccante che ruba le chiavi di sessione attuali di un utente (mitigato dalla forward secrecy del ratchet simmetrico), un attaccante che compromette completamente un dispositivo e poi ne perde l'accesso (mitigato dalla post-compromise security del ratchet DH), un man-in-the-middle che tenta di sostituire le chiavi pubbliche (mitigato dalla firma con identity key e dalla verifica opzionale del safety number), e un attaccante che intercetta le one-time prekey (mitigato dai tre calcoli DH di base che forniscono sicurezza anche senza la one-time prekey). Comprendere quali minacce affronta ciascun meccanismo è il cuore della progettazione di protocolli crittografici.

- **Crittografia lato client con WebCrypto API e archiviazione delle chiavi in IndexedDB.** Tutta la cifratura e decifratura avviene nel browser usando la WebCrypto API (`window.crypto.subtle`). Il file `primitives.ts` incapsula la WebCrypto API in funzioni per la generazione di chiavi X25519 e DH, firma e verifica Ed25519, derivazione delle chiavi HKDF, cifratura/decifratura AES-256-GCM, HMAC-SHA256 e funzioni di utilità come il confronto a tempo costante. Le chiavi private sono memorizzate in IndexedDB (`key-store.ts`) in un database chiamato `encrypted-chat-keys` con object store separati per identity key, signed prekey, one-time prekey e stati dei ratchet. IndexedDB è stato scelto rispetto a localStorage perché può memorizzare dati strutturati (inclusi oggetti `CryptoKey` nei browser che lo supportano) e non ha limiti di dimensione paragonabili al tipico tetto di 5-10MB di localStorage.

## Prerequisiti

### Conoscenze richieste

- **Programmazione asincrona in Python (asyncio, async/await).** L'intero backend è async. Gli handler delle route sono `async def`, le query al database usano `await` e il lifespan dell'applicazione è gestito da un `asynccontextmanager`. Devi capire coroutine, event loop e come `await` cede il controllo.
- **TypeScript e framework UI reattivi.** Il frontend è scritto in TypeScript con SolidJS. Dovresti trovarti a tuo agio con il sistema di tipi di TypeScript (generics, discriminated union, type guard) e con il concetto di stato reattivo (signal, effect, stato derivato). L'esperienza pregressa con React o Vue è trasferibile.
- **Comprensione di base della crittografia a chiave pubblica.** Devi sapere cos'è una coppia di chiavi (la chiave privata resta segreta, la chiave pubblica è condivisa), cosa significano firma e verifica, e il concetto generale di Diffie-Hellman (due parti derivano un segreto condiviso dalle rispettive coppie di chiavi pubblica/privata). Non hai bisogno di conoscere la matematica dietro le curve ellittiche.
- **Docker e Docker Compose.** L'ambiente di sviluppo esegue sei container. Devi sapere come leggere un compose file, eseguire `docker compose up`, controllare i log e fare debug dei problemi di avvio dei container.
- **Fondamenti di SQL e database.** I modelli PostgreSQL usano SQLModel (SQLAlchemy sotto il cofano). Dovresti essere in grado di leggere la definizione di un modello ORM e capire concetti come foreign key, indici e migrazioni.

### Strumenti richiesti

- **Docker e Docker Compose.** Per eseguire l'ambiente di sviluppo multi-servizio. Docker Desktop o Docker Engine con il plugin Compose.
- **Python 3.13+.** Il progetto richiede Python 3.13 o superiore, come specificato in `pyproject.toml` con `requires-python = ">=3.13"`. Le annotazioni di tipo e le funzionalità del linguaggio usate in tutto il codebase dipendono da questa versione.
- **Node.js 20+.** Per la toolchain di build del frontend SolidJS/Vite.
- **uv (package manager Python).** Questo progetto usa `uv` per la gestione delle dipendenze Python. Non usare mai pip direttamente. Installalo con `curl -LsSf https://astral.sh/uv/install.sh | sh` o `brew install uv`.
- **pnpm (package manager Node).** Questo progetto usa `pnpm` per la gestione delle dipendenze Node. Non usare mai npm direttamente. Installalo con `corepack enable && corepack prepare pnpm@latest --activate` o `npm install -g pnpm`.
- **just (command runner).** Il progetto usa un `justfile` per l'automazione della build, il linting, i test, le migrazioni del database e l'orchestrazione Docker. Installalo con `cargo install just`, `brew install just` o il package manager del tuo sistema.

### Utili ma opzionali

- **Esperienza pregressa con la programmazione WebSocket.** Il livello di messaggistica in tempo reale usa i WebSocket. La familiarità con il ciclo di vita di un WebSocket (connect, send, receive, close) e con concetti come heartbeat e riconnessione ti aiuterà, ma il codice è autoesplicativo.
- **Comprensione dello scambio di chiavi Diffie-Hellman.** Conoscere il protocollo DH di base (Alice e Bob generano ciascuno una coppia di chiavi, si scambiano le chiavi pubbliche, calcolano lo stesso segreto condiviso) renderà molto più facile seguire l'implementazione di X3DH. Il protocollo X3DH è essenzialmente la combinazione di quattro calcoli DH.
- **Familiarità con FastAPI o Flask.** Il backend usa i pattern di FastAPI: decorator delle route, dependency injection, schemi Pydantic per la validazione di request/response e middleware. Se hai usato un qualsiasi web framework Python, la struttura ti sarà familiare.
- **Conoscenza dell'API IndexedDB del browser.** L'archiviazione delle chiavi lato client usa IndexedDB direttamente (nessuna libreria wrapper). IndexedDB è un database chiave-valore asincrono e transazionale integrato in ogni browser moderno. Comprendere la sua API basata su callback (che questo progetto incapsula in Promise) ti aiuterà a seguire l'implementazione di `key-store.ts`.

## Avvio rapido

### Docker (consigliato)

```bash
# Clone and enter the project
cd PROJECTS/advanced/encrypted-p2p-chat

# Copy environment file and update passwords
cp .env.example .env

# Start all services (PostgreSQL, SurrealDB, Redis, backend, frontend, nginx)
docker compose -f dev.compose.yml up

# Backend API:      http://localhost:8000
# Frontend dev:     http://localhost:5173
# API docs:         http://localhost:8000/docs
# Nginx proxy:      http://localhost:80
```

### Setup manuale (senza Docker)

```bash
# Backend
cd backend
uv sync
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Frontend (separate terminal)
cd frontend
pnpm install
pnpm dev
```

Per il setup manuale, ti servono PostgreSQL, SurrealDB e Redis in esecuzione in locale. Aggiorna le stringhe di connessione in `.env` perché puntino a `localhost` invece che ai nomi dei servizi Docker.

### Usare il justfile

```bash
# See all available commands
just

# Full setup (backend deps, frontend deps, env file)
just setup

# Start dev environment
just dev-up

# Run linters
just lint

# Run tests
just test

# Database migrations
just migrate-local head
```

## Struttura del progetto

```
encrypted-p2p-chat/
|
|-- backend/
|   |-- app/
|   |   |-- api/                              # HTTP and WebSocket route handlers
|   |   |   |-- auth.py                       # WebAuthn registration and login endpoints
|   |   |   |-- encryption.py                 # X3DH prekey bundle upload and retrieval
|   |   |   |-- rooms.py                      # Chat room creation, listing, membership
|   |   |   +-- websocket.py                  # WebSocket connection and message routing
|   |   |
|   |   |-- core/
|   |   |   |-- passkey/
|   |   |   |   +-- passkey_manager.py        # WebAuthn credential registration and verification
|   |   |   |-- dependencies.py               # current_user, issue_session, revoke_session
|   |   |   |-- websocket_manager.py          # Connection pool, broadcasting, heartbeat management
|   |   |   |-- surreal_manager.py            # SurrealDB client: connect, query, live subscriptions
|   |   |   |-- redis_manager.py              # Redis client: sessions, challenges, ephemeral cache
|   |   |   |-- enums.py                      # Shared enumerations (message types, status codes)
|   |   |   |-- exceptions.py                 # Domain-specific exception classes
|   |   |   +-- exception_handlers.py         # FastAPI exception handler registration
|   |   |
|   |   |-- models/                           # SQLModel database models (PostgreSQL)
|   |   |   |-- Base.py                       # SQLAlchemy engine, session factory, init_db
|   |   |   |-- User.py                       # User account + 64-byte WebAuthn user_handle
|   |   |   |-- Credential.py                 # WebAuthn credential storage (public key, counter, BE/BS flags)
|   |   |   |-- IdentityKey.py                # Long-term X25519 + Ed25519 identity public keys (no privates)
|   |   |   |-- SignedPrekey.py               # Client-uploaded signed prekey publics + signatures
|   |   |   +-- OneTimePrekey.py              # Single-use prekey publics (consumed via is_used flag)
|   |   |
|   |   |-- services/                         # Business logic layer
|   |   |   |-- auth_service.py               # Registration / authentication / search (no key gen)
|   |   |   |-- prekey_service.py             # Public-only prekey bundle storage + lookup
|   |   |   |-- message_service.py            # Pass-through ciphertext writer (server never decrypts)
|   |   |   |-- presence_service.py           # Online/offline status tracking via SurrealDB
|   |   |   +-- websocket_service.py          # WebSocket message routing + membership check + per-user rate cap
|   |   |
|   |   |-- schemas/                          # Pydantic request/response models
|   |   |   |-- auth.py                       # Registration/login options and verification schemas
|   |   |   |-- rooms.py                      # Room creation and listing schemas
|   |   |   |-- websocket.py                  # WebSocket message payload schemas
|   |   |   |-- surreal.py                    # SurrealDB record schemas
|   |   |   +-- common.py                     # Shared schemas (health, root, pagination)
|   |   |
|   |   |-- config.py                         # Centralized settings, constants, environment loading
|   |   |-- factory.py                        # FastAPI app factory with lifespan context manager
|   |   +-- main.py                           # Uvicorn entry point
|   |
|   |-- tests/                                # pytest test suite
|   |   |-- conftest.py                       # Shared fixtures (in-memory async SQLite, test users)
|   |   |-- test_auth_service.py              # User CRUD and lookup
|   |   |-- test_prekey_service.py            # Bundle/upload regression tests (incl. broken-filter regression)
|   |   +-- test_session_auth.py              # Protected endpoints return 401 without session cookie
|   |
|   |-- alembic/                              # Database migration scripts
|   |   +-- env.py                            # Alembic environment configuration
|   +-- pyproject.toml                        # uv dependencies and tool configuration
|
|-- frontend/
|   |-- src/
|   |   |-- crypto/                           # Client-side Signal Protocol implementation
|   |   |   |-- primitives.ts                 # WebCrypto API wrappers (X25519, Ed25519, AES-GCM, HKDF)
|   |   |   |-- x3dh.ts                       # X3DH key exchange (client-side sender/receiver)
|   |   |   |-- double-ratchet.ts             # Double Ratchet state machine (client-side)
|   |   |   |-- crypto-service.ts             # Encryption orchestrator (ties X3DH + Ratchet together)
|   |   |   |-- key-store.ts                  # IndexedDB private key storage (identity, prekeys, ratchet)
|   |   |   +-- message-store.ts              # Encrypted message serialization and deserialization
|   |   |
|   |   |-- pages/                            # Route-level page components
|   |   |   |-- Home.tsx                      # Landing page
|   |   |   |-- Login.tsx                     # WebAuthn login flow
|   |   |   |-- Register.tsx                  # WebAuthn registration flow
|   |   |   |-- Chat.tsx                      # Main chat interface
|   |   |   +-- NotFound.tsx                  # 404 page
|   |   |
|   |   |-- components/
|   |   |   |-- Auth/                         # Authentication components
|   |   |   |   |-- AuthCard.tsx              # Card container for auth forms
|   |   |   |   |-- AuthForm.tsx              # Username/display name input form
|   |   |   |   +-- PasskeyButton.tsx         # WebAuthn credential creation trigger
|   |   |   |-- Chat/                         # Chat interface components
|   |   |   |   |-- ChatHeader.tsx            # Conversation header with user info
|   |   |   |   |-- ChatInput.tsx             # Message composition and send
|   |   |   |   |-- ConversationList.tsx      # Sidebar list of active conversations
|   |   |   |   |-- ConversationItem.tsx      # Single conversation preview
|   |   |   |   |-- MessageBubble.tsx         # Individual message display
|   |   |   |   |-- MessageList.tsx           # Scrollable message history
|   |   |   |   |-- EncryptionBadge.tsx       # Visual indicator of encryption status
|   |   |   |   |-- OnlineStatus.tsx          # User online/offline indicator
|   |   |   |   |-- TypingIndicator.tsx       # "User is typing..." display
|   |   |   |   |-- UserSearch.tsx            # Search for users to start conversations
|   |   |   |   +-- NewConversation.tsx       # New conversation creation flow
|   |   |   |-- Layout/                       # Page layout components
|   |   |   |   |-- AppShell.tsx              # Main app layout wrapper
|   |   |   |   |-- Header.tsx               # Top navigation bar
|   |   |   |   |-- Sidebar.tsx              # Side navigation panel
|   |   |   |   +-- ProtectedRoute.tsx       # Auth-gated route wrapper
|   |   |   +-- UI/                           # Reusable UI primitives
|   |   |       |-- Avatar.tsx               # User avatar display
|   |   |       |-- Badge.tsx                # Status badge component
|   |   |       |-- Button.tsx               # Button with variants
|   |   |       |-- Dropdown.tsx             # Dropdown menu
|   |   |       |-- IconButton.tsx           # Icon-only button
|   |   |       |-- Input.tsx               # Text input field
|   |   |       |-- Modal.tsx               # Dialog overlay
|   |   |       |-- Skeleton.tsx            # Loading placeholder
|   |   |       |-- Spinner.tsx             # Loading spinner
|   |   |       |-- TextArea.tsx            # Multiline text input
|   |   |       |-- Toast.tsx              # Notification toast
|   |   |       +-- Tooltip.tsx            # Hover tooltip
|   |   |
|   |   |-- services/                         # API client layer
|   |   |   |-- auth.service.ts               # Authentication API calls
|   |   |   +-- room.service.ts               # Room and message API calls
|   |   |
|   |   |-- stores/                           # Nanostores state management
|   |   |   |-- auth.store.ts                 # Current user + persistent userId hint
|   |   |   |-- rooms.store.ts                # Room list and active room state
|   |   |   |-- messages.store.ts             # Message history per conversation
|   |   |   |-- presence.store.ts             # User online status tracking
|   |   |   |-- typing.store.ts               # Typing indicator state
|   |   |   |-- settings.store.ts             # User preferences
|   |   |   +-- ui.store.ts                   # UI state (modals, sidebars, loading)
|   |   |
|   |   |-- websocket/                        # WebSocket client
|   |   |   |-- websocket-manager.ts          # Connection lifecycle and reconnection
|   |   |   +-- message-handlers.ts           # Incoming message type dispatch
|   |   |
|   |   |-- lib/                              # Shared utilities
|   |   |   |-- api-client.ts                 # HTTP client wrapper (fetch-based)
|   |   |   |-- base64.ts                     # Base64/Base64URL encoding helpers
|   |   |   |-- validators.ts                 # Input validation functions
|   |   |   +-- date.ts                       # Date formatting utilities
|   |   |
|   |   |-- types/                            # TypeScript type definitions
|   |   |   |-- encryption.ts                 # Crypto-related types and interfaces
|   |   |   |-- auth.ts                       # Authentication types
|   |   |   |-- chat.ts                       # Chat message and room types
|   |   |   |-- websocket.ts                  # WebSocket message types
|   |   |   |-- api.ts                        # API response types
|   |   |   |-- components.ts                 # Component prop types
|   |   |   +-- guards.ts                     # Type guard functions
|   |   |
|   |   |-- App.tsx                           # Root component with router
|   |   |-- config.ts                         # Environment configuration
|   |   |-- index.tsx                         # Application entry point
|   |   +-- index.css                         # Global styles and Tailwind imports
|   |
|   |-- vite.config.ts                        # Vite build configuration
|   |-- tsconfig.json                         # TypeScript compiler options
|   |-- eslint.config.js                      # ESLint configuration
|   +-- package.json                          # pnpm dependencies
|
|-- conf/
|   |-- docker/
|   |   |-- dev/
|   |   |   |-- fastapi.docker                # Dev Dockerfile for backend (hot reload)
|   |   |   +-- vite.docker                   # Dev Dockerfile for frontend (HMR)
|   |   +-- prod/
|   |       |-- fastapi.docker                # Production Dockerfile for backend
|   |       +-- vite.docker                   # Production Dockerfile for frontend (static build)
|   +-- nginx/
|       |-- dev.nginx                         # Development Nginx config
|       |-- prod.nginx                        # Production Nginx config
|       +-- http.conf                         # Shared HTTP settings
|
|-- learn/                                    # Educational documentation (you are here)
|-- .env.example                              # Template environment variables
|-- compose.yml                               # Production Docker Compose
|-- dev.compose.yml                           # Development Docker Compose
|-- justfile                                  # Command runner recipes
+-- LICENSE
```

## Panoramica dell'architettura

Il diagramma seguente mostra come i dati fluiscono attraverso il sistema durante uno scambio di messaggi cifrati:

```
+------------------+                                    +------------------+
|    Alice's       |                                    |     Bob's        |
|    Browser       |                                    |     Browser      |
|                  |                                    |                  |
|  +------------+  |                                    |  +------------+  |
|  | IndexedDB  |  |                                    |  | IndexedDB  |  |
|  | (private   |  |                                    |  | (private   |  |
|  |  keys)     |  |                                    |  |  keys)     |  |
|  +-----+------+  |                                    |  +-----+------+  |
|        |         |                                    |        |         |
|  +-----v------+  |                                    |  +-----v------+  |
|  | WebCrypto  |  |                                    |  | WebCrypto  |  |
|  | X3DH +     |  |                                    |  | X3DH +     |  |
|  | Ratchet    |  |                                    |  | Ratchet    |  |
|  +-----+------+  |                                    |  +-----+------+  |
|        |         |                                    |        ^         |
|  plaintext       |                                    |  plaintext       |
|  encrypted to    |                                    |  decrypted from  |
|  ciphertext      |                                    |  ciphertext      |
|        |         |                                    |        |         |
|  +-----v------+  |         +----------------+         |  +-----+------+  |
|  | WebSocket  +--+-------->|    Server      +-------->+--+ WebSocket  |  |
|  | Client     |  |         |                |         |  | Client     |  |
|  +------------+  |         | +------------+ |         |  +------------+  |
|                  |         | | FastAPI    | |         |                  |
+------------------+         | | WebSocket  | |         +------------------+
                             | | Router     | |
                             | +-----+------+ |
                             |       |        |
                             | +-----v------+ |
                             | | SurrealDB  | |
                             | | (encrypted | |
                             | |  blobs)    | |
                             | +------------+ |
                             |                |
                             | +------------+ |
                             | | PostgreSQL | |
                             | | (auth,     | |
                             | |  keys)     | |
                             | +------------+ |
                             |                |
                             | +------------+ |
                             | | Redis      | |
                             | | (challenges| |
                             | |  sessions) | |
                             | +------------+ |
                             +----------------+
```

Punti chiave:

1. **Il testo in chiaro non lascia mai il browser.** La cifratura avviene nella WebCrypto API prima che il messaggio raggiunga il WebSocket.
2. **Il server vede solo ciphertext.** Memorizza e inoltra blob cifrati. Non può decifrarli.
3. **Le chiavi private risiedono in IndexedDB.** Vengono generate dalla WebCrypto API e memorizzate lato client. Il server memorizza solo chiavi pubbliche.
4. **Tre database servono a scopi distinti.** PostgreSQL gestisce i dati relazionali di autenticazione, SurrealDB gestisce la memorizzazione dei messaggi in tempo reale e Redis gestisce lo stato effimero come i challenge token.

## Prossimi passi

I documenti rimanenti in questa cartella learn percorrono il progetto livello per livello:

- **01-CONCEPTS.md** -- Teoria del Signal Protocol, matematica di X3DH, meccanica del Double Ratchet, funzionamento interno di WebAuthn e le violazioni reali che hanno motivato questi protocolli. È qui che costruisci il modello mentale del perché di ogni decisione crittografica.

- **02-ARCHITECTURE.md** -- Design del sistema, schemi del database, diagrammi del flusso dei dati, architettura di sicurezza e il ragionamento dietro ogni scelta tecnologica. Tratta la suddivisione in tre database, il protocollo dei messaggi WebSocket e il design del server zero-knowledge.

- **03-IMPLEMENTATION.md** -- Spiegazione riga per riga delle implementazioni crittografiche, dei flussi di autenticazione, della messaggistica WebSocket e della gestione del ciclo di vita delle chiavi. Segue il codice attraverso uno scambio di messaggi completo, dalla registrazione all'upload delle chiavi, allo stabilimento della sessione, alla cifratura del messaggio, alla consegna e alla decifratura.

- **04-CHALLENGES.md** -- Progetti di estensione che vanno dall'aggiunta di conferme di lettura e indicatori di digitazione all'implementazione della cifratura di gruppo (Sender Keys), del supporto multi-dispositivo, dei messaggi a scomparsa e dello scambio di chiavi post-quantum usando la libreria `liboqs-python` già inclusa nelle dipendenze del progetto.

## Problemi comuni

**I servizi Docker non partono:**
Verifica che le porte 5432, 8001, 6379, 8000, 5173 e 80 non siano già in uso. Nota che SurrealDB è mappato sulla porta host 8001 (non la 8000, che è usata dal backend FastAPI). Esegui `docker compose -f dev.compose.yml down -v` per rimuovere container e volumi obsoleti, poi riavvia. Se un servizio specifico fallisce, controlla i suoi log con `just dev-logs postgres` (o qualunque sia il nome del servizio).

**La registrazione WebAuthn fallisce in sviluppo:**
WebAuthn richiede `localhost` o un'origine HTTPS valida. La configurazione di default usa `RP_ID=localhost` e `RP_ORIGIN=http://localhost` in `.env.example`. Assicurati che il tuo file `.env` abbia questi valori impostati correttamente. Se accedi al frontend su `http://localhost:5173`, allora `RP_ORIGIN` deve corrispondere esattamente a quell'origine (schema + host + porta). Se stai testando da un indirizzo IP della LAN o da un hostname diverso da localhost, ti servirà un certificato TLS, perché WebAuthn si rifiuterà di creare credenziali su origini non sicure diverse da localhost.

**Errore "No identity key" all'avvio di una chat:**
Dopo la registrazione WebAuthn, il client deve generare e caricare sul server le chiavi di cifratura **pubbliche** (le parti pubbliche della coppia di identity key, la signed prekey + firma, le one-time prekey pubbliche). Questo avviene in `crypto-service.ts:initialize`, che poi chiama `uploadPublicKeys` → `POST /encryption/upload-keys/{user_id}`. Se questo passo fallisce silenziosamente, l'utente avrà un account ma nessun bundle di prekey pubblicato, e gli altri utenti riceveranno un 400 "User has not uploaded keys yet" quando proveranno ad avviare una sessione X3DH. Controlla la console del browser per individuare errori durante il passo di generazione delle chiavi. Le cause comuni includono l'accesso a IndexedDB bloccato (la navigazione privata in alcuni browser limita IndexedDB), la WebCrypto API non disponibile (richiede un secure context) o il session cookie non inviato (verifica che la richiesta abbia `credentials: 'include'`, che il `api-client.ts` condiviso imposta già).

**Errori di migrazione del database:**
Se all'avvio vedi errori di mancata corrispondenza dello schema, esegui prima le migrazioni. Per Docker: `just migrate head`. Per lo sviluppo locale: `just migrate-local head`. Se devi ripartire da zero, `just dev-down` seguito dalla rimozione dei volumi Docker ti darà un database pulito.

**Connessione a SurrealDB rifiutata:**
SurrealDB gira sulla porta 8000 all'interno del container ma è mappato sulla porta 8001 dell'host. Il backend si connette a `ws://surrealdb:8000` (l'indirizzo della rete Docker interna). Se stai eseguendo il backend fuori da Docker, aggiorna `SURREAL_URL` in `.env` a `ws://localhost:8001`.

**Disconnessioni del WebSocket del frontend:**
Il client WebSocket ha la riconnessione automatica integrata, ma se il backend non è in esecuzione o l'URL di connessione è sbagliato, vedrai ripetuti tentativi di disconnessione/riconnessione nella console del browser. Verifica che `VITE_WS_URL` nel `.env` del frontend corrisponda all'effettivo endpoint WebSocket del backend. Il default è `ws://localhost:8000`.

## Progetti correlati

- **API Rate Limiter** (advanced) -- Implementa il rate limiting basato su Redis con sliding window e token bucket. Gli stessi pattern Redis (scadenza delle chiavi basata su TTL, operazioni di incremento atomiche) sono usati in questo progetto per la memorizzazione delle challenge WebAuthn e per il rate limiting dei messaggi a 60 messaggi al minuto per utente.

- **Bug Bounty Platform** (advanced) -- Un altro progetto fullstack che combina autenticazione, funzionalità in tempo reale e architettura multi-database. Condivide pattern architetturali come l'app factory FastAPI, l'astrazione del service layer e l'orchestrazione con Docker Compose.

- **Simple Port Scanner** (beginner) -- Introduce i fondamenti del networking TCP, inclusi la programmazione dei socket, gli stati delle porte e gli handshake di connessione. Questi concetti di livello rete stanno alla base del trasporto WebSocket usato per la consegna dei messaggi in tempo reale in questo progetto.
