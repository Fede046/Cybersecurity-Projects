# Sfide di estensione

Queste sfide estendono la chat P2P cifrata con funzionalità reali usate nei sistemi di messaggistica in produzione. Sono ordinate per difficoltà e si basano l'una sull'altra dove indicato.

Ogni sfida fa riferimento a file reali di questo progetto. I numeri di riga ti indicano il codice esatto che devi capire prima di iniziare. Leggi prima quelle righe.

---

## Sfide facili

---

### Sfida 1: Conferme di lettura

**Cosa costruire:** Quando Bob legge un messaggio di Alice, invia ad Alice una conferma di lettura cifrata, così lei vede lo stato "Read" sul suo messaggio.

**Perché è importante:** Ogni app di messaggistica moderna ha le conferme di lettura. Signal, WhatsApp e iMessage le implementano tutte. La parte interessante è che le conferme di lettura stesse dovrebbero essere cifrate, perché sapere QUANDO qualcuno ha letto un messaggio è un metadato che vale la pena proteggere. Un avversario che può osservare la tempistica delle conferme può dedurre pattern di conversazione, reattività e persino orari di sonno.

**Cosa imparerai:**
- I pattern di messaggistica bidirezionale WebSocket
- Estendere il routing esistente dei tipi di messaggio
- La gestione dello stato dell'UI per lo stato di consegna dei messaggi

**Da dove iniziare a leggere:**

Il backend ha già l'impalcatura per questo. Guarda questi file:

- `backend/app/config.py` definisce già `WS_MESSAGE_TYPE_RECEIPT = "receipt"`
- `backend/app/services/backend/app/services/websocket_service.py` instrada già i messaggi receipt a `handle_read_receipt`
- `backend/app/services/backend/app/services/websocket_service.py` ha un'implementazione funzionante di `handle_read_receipt`
- `backend/app/schemas/websocket.py` definisce `ReadReceiptWS` con `message_id`, `user_id` e `read_at`

Quindi il backend è finito. Il lavoro è sul frontend.

**Approccio di implementazione:**

1. Quando il client di Bob decifra e mostra un messaggio di Alice, emetti una conferma tramite il WebSocket:
 ```json
 {
 "type": "receipt",
 "message_id": "messages:abc123",
 "sender_id": "alice-uuid-here"
 }
 ```

2. Il client di Alice riceve il `ReadReceiptWS` tramite l'handler dei messaggi WebSocket in `frontend/src/websocket/message-handlers.ts`. Aggiungi un handler per il tipo di messaggio `"receipt"`.

3. Aggiorna lo store dei messaggi (`frontend/src/stores/messages.store.ts`) per tracciare lo stato dei messaggi. Aggiungi un campo `status` con i valori: `"sending"`, `"sent"`, `"delivered"`, `"read"`.

4. Nel componente `MessageBubble.tsx` (`frontend/src/components/Chat/MessageBubble.tsx`), mostra lo stato sotto il testo del messaggio per i messaggi in uscita.

**Decisioni di design che devi prendere:**
- Le conferme dovrebbero essere cifrate? L'implementazione attuale le invia in chiaro sul WebSocket. Un sistema di produzione le cifrerebbe tramite il Double Ratchet, ma questo significa che ogni conferma fa avanzare lo stato del ratchet. È un compromesso significativo.
- Dovresti raggruppare le conferme in batch? Se Bob scorre 50 messaggi non letti, inviare 50 conferme individuali è uno spreco. Valuta di inviare una sola conferma per l'ID del messaggio più recente (implicando che anche tutti i messaggi precedenti sono letti).
- Cosa succede se Bob legge un messaggio mentre è offline (caricato da IndexedDB)? La conferma deve essere messa in coda e inviata quando il WebSocket si riconnette.

**Come testare:**
- Apri due tab del browser come utenti diversi
- Invia un messaggio dalla Tab A alla Tab B
- Verifica che la Tab A mostri inizialmente "Sent"
- Passa alla Tab B e apri la conversazione, verifica che la Tab A si aggiorni a "Read"
- Controlla la scheda Network di DevTools del browser (frame WS) per confermare che la conferma viaggi sul WebSocket, non con una richiesta HTTP separata
- Chiudi la Tab B, invia un altro messaggio dalla Tab A, riapri la Tab B, verifica che la conferma funzioni ancora

---

### Sfida 2: Indicatori di digitazione

**Cosa costruire:** Mostra "Alice is typing..." nella finestra di chat di Bob mentre Alice sta componendo un messaggio.

**Perché è importante:** Gli indicatori di presenza sono standard nella messaggistica. La sfida è implementarli in modo efficiente. Un'implementazione ingenua inonda il WebSocket di eventi di pressione dei tasti. Una buona implementazione fa throttling per ridurre al minimo la banda mantenendo reattiva l'UI.

**Cosa imparerai:**
- Throttling e debouncing degli eventi WebSocket
- La gestione di stato effimero (stato che non viene mai persistito in alcun database)
- Aggiornamenti reattivi del frontend con nanostores

**Da dove iniziare a leggere:**

Questa funzionalità è già in gran parte costruita. Studia ciò che esiste:

- `backend/app/config.py` definisce `WS_MESSAGE_TYPE_TYPING = "typing"`
- `backend/app/services/backend/app/services/websocket_service.py` instrada i messaggi typing a `handle_typing_indicator`
- `backend/app/services/backend/app/services/websocket_service.py` trasmette gli eventi typing alla stanza tramite `connection_manager.broadcast_to_room`
- `backend/app/schemas/websocket.py` definisce `TypingIndicatorWS` con `user_id`, `room_id`, `is_typing`
- `frontend/src/stores/typing.store.ts` è uno store typing completo con timeout di auto-clear (5 secondi,)
- `frontend/src/components/Chat/TypingIndicator.tsx` mostra già l'UI dell'indicatore di digitazione

Il backend e la maggior parte della gestione dello stato del frontend sono fatti. Ciò che manca:

**Approccio di implementazione:**

1. Nel componente `ChatInput.tsx` (`frontend/src/components/Chat/ChatInput.tsx`), aggiungi un handler `onInput` che invii un evento typing tramite il WebSocket.

2. Applica il throttling agli eventi typing. Non vuoi inviarne uno per ogni tasto premuto. Implementa una finestra di throttle di 3 secondi: invia un evento "is_typing: true", poi sopprimi tutti gli eventi successivi per 3 secondi. Quando l'utente smette di scrivere (nessuna pressione di tasti per 3 secondi), invia "is_typing: false".

3. Nell'handler dei messaggi WebSocket (`frontend/src/websocket/message-handlers.ts`), gestisci i messaggi `"typing"` in arrivo chiamando `setUserTyping` dallo store typing.

4. Collega il componente `TypingIndicator.tsx` alla vista della chat se non è già connesso.

**Pattern di throttling:**
```
Keystroke at t=0ms -> send "typing: true"
Keystroke at t=500ms -> suppress (within 3s window)
Keystroke at t=1.2s -> suppress
Keystroke at t=4.5s -> send "typing: true" (new window)
No input for 3s -> send "typing: false"
```

**Non cifrare gli eventi typing.** Sono metadati effimeri. Farli passare per il Double Ratchet farebbe avanzare lo stato del ratchet per dati usa e getta, sprecando chain key e aumentando il rischio di desincronizzazione.

**Come testare:**
- Apri due tab come utenti diversi nella stessa stanza
- Inizia a scrivere nella Tab A
- Verifica che "typing..." appaia nella Tab B entro 1 secondo
- Smetti di scrivere e verifica che l'indicatore scompaia dopo 5 secondi (la costante `TYPING_TIMEOUT_MS` in `typing.store.ts`)
- Scrivi rapidamente e verifica che ci sia un solo evento WebSocket per finestra di 3 secondi (controlla i frame WS in DevTools)
- Cambia stanza e verifica che lo stato di digitazione si resetti

---

### Sfida 3: Timestamp dei messaggi e ordinamento

**Cosa costruire:** Mostra timestamp leggibili sui messaggi ("2:34 PM", "Yesterday", "Jan 15") e assicurati che i messaggi vengano visualizzati in ordine cronologico corretto anche quando arrivano fuori ordine sul WebSocket.

**Perché è importante:** Il protocollo Double Ratchet gestisce la consegna fuori ordine a livello crittografico (vedi `frontend/src/crypto/double-ratchet.ts` per la logica delle skipped message key), ma anche l'UI deve gestire correttamente l'ordine di visualizzazione. È una fonte comune di bug nelle app di messaggistica. Se Alice invia il messaggio A e poi il messaggio B, ma B arriva a Bob per primo sul WebSocket, l'UI deve comunque mostrare A prima di B.

**Cosa imparerai:**
- L'ordinamento dei messaggi lato client con ordine stabile
- La formattazione delle date e la localizzazione con le API `Intl`
- Gestire lo sfasamento degli orologi (clock skew) tra dispositivi

**Da dove iniziare a leggere:**

- `backend/app/schemas/websocket.py` mostra che `EncryptedMessageWS` include un campo `timestamp`
- `frontend/src/crypto/message-store.ts` ordina già i messaggi per `created_at`
- `frontend/src/lib/date.ts` è il file di utility per le date esistente
- `frontend/src/components/Chat/MessageBubble.tsx` renderizza i singoli messaggi

**Approccio di implementazione:**

1. Lo store dei messaggi in `message-store.ts` ordina già per `created_at`. Verifica che funzioni quando i messaggi arrivano fuori ordine controllando che i nuovi messaggi vengano inseriti nella posizione corretta, non semplicemente aggiunti in fondo.

2. Crea un formatter dei timestamp. Usa `Intl.RelativeTimeFormat` per i messaggi recenti e `Intl.DateTimeFormat` per quelli più vecchi:
 - Meno di 1 minuto: "Just now"
 - Meno di 1 ora: "12 min ago"
 - Stesso giorno: "2:34 PM"
 - Ieri: "Yesterday at 2:34 PM"
 - Stesso anno: "Jan 15 at 2:34 PM"
 - Più vecchio: "Jan 15, 2025 at 2:34 PM"

3. Aggiungi intestazioni separatrici di data tra messaggi di giorni diversi. Quando renderizzi la lista dei messaggi, confronta la data di ciascun messaggio con quella del precedente. Se differiscono, inserisci un separatore "--- January 15, 2025 ---".

4. Gestisci il clock skew. Se l'orologio di Alice è avanti di 5 minuti, i suoi messaggi avranno timestamp nel futuro. Limita gli orari mostrati in modo che non siano mai "nel futuro" rispetto all'orologio del destinatario.

**Come testare:**
- Invia rapidamente diversi messaggi tra due utenti e verifica l'ordine cronologico
- Usa DevTools del browser per limitare la rete a "Slow 3G", invia più messaggi e verifica che gli arrivi fuori ordine vengano comunque mostrati nell'ordine corretto
- Invia un messaggio, aspetta 5 minuti, ne invii un altro. Verifica che NON compaia un separatore di data (stesso giorno). Aspetta fino al giorno di calendario successivo e invia un altro messaggio. Verifica che il separatore compaia.

---

### Sfida 4: Stato online dell'utente

**Cosa costruire:** Mostra pallini verdi/grigi accanto ai nomi utente nella lista delle conversazioni e nell'intestazione della chat, indicando lo stato online/offline in tempo reale.

**Perché è importante:** Il progetto ha già un sistema di presenza completo nel backend e uno store di presenza nel frontend. Questa sfida li collega attraverso il livello WebSocket.

**Cosa imparerai:**
- I record di presenza di SurrealDB e il loro ciclo di vita
- Lo stato reattivo del frontend a partire dagli eventi WebSocket
- Indicatori UI con aggiornamenti in tempo reale

**Da dove iniziare a leggere:**

L'infrastruttura del backend è già costruita:

- `backend/app/core/websocket_manager.py` chiama `presence_service.set_user_online(user_id)` alla connessione WebSocket
- `backend/app/core/websocket_manager.py` chiama `presence_service.set_user_offline(user_id)` alla disconnessione (solo quando si chiude l'ultima connessione di quell'utente)
- `backend/app/services/presence_service.py` imposta lo stato online in SurrealDB
- `backend/app/services/presence_service.py` imposta lo stato offline
- `backend/app/services/backend/app/services/websocket_service.py` gestisce i messaggi di aggiornamento della presenza

Anche lo store del frontend è pronto:

- `frontend/src/stores/presence.store.ts` ha la mappa `$presenceByUser`, `setUserPresence`, `getUserStatus`, `isUserOnline`
- `frontend/src/components/Chat/OnlineStatus.tsx` mostra già l'indicatore di stato

**Approccio di implementazione:**

1. Quando un utente si connette tramite WebSocket, il backend aggiorna già la presenza in SurrealDB. Ciò che manca è trasmettere quella modifica ai contatti dell'utente. In `websocket_manager.py`, dopo che `presence_service.set_user_online(user_id)` ha successo, trasmetti un aggiornamento di presenza a tutte le stanze a cui l'utente appartiene.

2. Aggiungi un handler `PresenceUpdateWS` nell'handler dei messaggi WebSocket del frontend. Quando arriva un messaggio `"presence"`, chiama `setUserPresence` dallo store della presenza.

3. In `ConversationList.tsx` (`frontend/src/components/Chat/ConversationList.tsx`) e `ChatHeader.tsx` (`frontend/src/components/Chat/ChatHeader.tsx`), leggi da `$presenceByUser` per mostrare l'indicatore online/offline corretto.

4. Alla connessione WebSocket iniziale, richiedi la presenza attuale di tutti i contatti dell'utente. Potrebbe essere un nuovo endpoint HTTP (`GET /users/presence?user_ids=...`) o uno speciale messaggio WebSocket che restituisce uno snapshot di presenza in blocco.

**Come testare:**
- Apri la Tab A come User1, verifica che User1 risulti online nella lista delle conversazioni
- Apri la Tab B come User2, verifica che entrambi risultino online
- Chiudi la Tab B, verifica che User2 risulti offline nella Tab A entro pochi secondi
- Riapri la Tab B, verifica che User2 risulti di nuovo online

---

## Sfide intermedie

---

### Sfida 5: Ricerca nei messaggi (cifrati)

**Cosa costruire:** Permetti agli utenti di cercare nella propria cronologia di messaggi per parola chiave. Poiché i messaggi sono E2E cifrati, il server non può cercare. Tutta la ricerca deve avvenire lato client dopo la decifratura.

**Perché è importante:** È un vero problema ingegneristico che Signal, WhatsApp e iMessage risolvono ciascuno in modo diverso. La ricerca lato server è impossibile con la cifratura E2E, che è esattamente lo scopo della cifratura E2E. La ricerca lato client richiede un design accurato per bilanciare sicurezza, prestazioni e UX.

Signal consente di cercare i messaggi decifrati sul dispositivo. WhatsApp esegue il backup dei messaggi su Google Drive/iCloud (opzionalmente cifrato) per renderli ricercabili. iMessage usa un indice Spotlight locale. Ogni approccio ha proprietà di sicurezza diverse.

**Cosa imparerai:**
- La ricerca full-text lato client nelle applicazioni cifrate
- IndexedDB come database locale in grado di supportare la ricerca
- I compromessi di sicurezza tra ricercabilità e forward secrecy

**Da dove iniziare a leggere:**

- `frontend/src/crypto/message-store.ts` è lo store dei messaggi IndexedDB esistente con l'object store `decrypted_messages`
- `frontend/src/crypto/message-store.ts` crea indici su `room_id`, `created_at` e un indice composto `room_created`
- `frontend/src/crypto/message-store.ts` mostra come i messaggi vengono interrogati per stanza
- `frontend/src/crypto/key-store.ts` mostra `clearAllKeys` che cancella tutto lo stato crittografico al logout

**Approccio di implementazione:**

1. I messaggi decifrati sono già memorizzati in IndexedDB in `message-store.ts` (`saveDecryptedMessage`). Il plaintext è già persistito in locale. Non serve aggiungere uno store separato per il plaintext.

2. Aggiungi una funzione `searchMessages` a `message-store.ts`. IndexedDB non supporta nativamente la ricerca full-text. Hai due opzioni:

 Opzione A (semplice): Carica tutti i messaggi dell'utente, filtra con `String.prototype.includes` o una regex. Funziona per cronologie di messaggi piccole (sotto i 10.000 messaggi).

 Opzione B (performante): Costruisci un semplice indice invertito in un object store IndexedDB separato. A ogni salvataggio di un messaggio, tokenizza il plaintext, memorizza voci `{token: string, message_id: string}`. Le query di ricerca cercano i token nell'indice. Scala fino a centinaia di migliaia di messaggi.

3. Costruisci una UI di ricerca. Aggiungi un campo di ricerca alla sidebar. I risultati dovrebbero mostrare anteprime dei messaggi con il testo corrispondente evidenziato. Cliccando un risultato si dovrebbe navigare a quel messaggio nella conversazione e portarlo in vista con lo scroll.

4. Cancella l'indice di ricerca al logout. Aggiungilo a qualunque funzione di logout chiami già `clearAllKeys` e `clearAllMessages`.

**Considerazioni di sicurezza:**

Lo store locale del plaintext è un compromesso di sicurezza. Se il dispositivo è compromesso, la cronologia di ricerca espone il contenuto dei messaggi. Signal lo gestisce cercando solo nei messaggi della sessione corrente dell'app (senza un indice persistente). Valuta di offrire un'impostazione: "Enable message search (stores decrypted messages locally)" contro "Maximum security (no local message storage)".

Considera anche: i dati di IndexedDB sopravvivono alla pulizia della cache in alcuni browser. Usa `indexedDB.deleteDatabase` al logout, non solo la pulizia degli object store.

**Come testare:**
- Invia diversi messaggi contenenti parole chiave note tra due utenti
- Cerca una parola chiave e verifica che i messaggi corrispondenti compaiano nei risultati
- Cerca una parola chiave che compare in più stanze e verifica che i risultati siano raggruppati per stanza
- Esegui il logout e verifica che l'indice di ricerca sia cancellato (controlla IndexedDB nella scheda Application di DevTools)
- Cerca senza risultati e verifica che venga mostrato uno stato vuoto appropriato

---

### Sfida 6: Condivisione di file (cifrata)

**Cosa costruire:** Permetti agli utenti di inviare file cifrati (immagini, documenti, audio) tramite la chat. I file devono essere cifrati lato client con le stesse garanzie di cifratura E2E dei messaggi di testo.

**Perché è importante:** La condivisione di file è una funzionalità centrale di qualsiasi app di messaggistica. La sfida è cifrare in modo efficiente grandi quantità di dati binari mantenendo le stesse proprietà di sicurezza dei messaggi di testo. Un video da 50MB non può essere cifrato nello stesso modo di un messaggio di testo da 200 byte. Servono cifratura a chunk, indicatori di avanzamento e un percorso di upload separato.

**Cosa imparerai:**
- Cifrare grandi dati binari con AES-256-GCM
- La cifratura a chunk per l'efficienza di memoria
- Le API di upload dei file con gestione di dati binari
- La generazione di thumbnail per le anteprime delle immagini
- La gestione dei tipi MIME e la sicurezza del content-type

**Da dove iniziare a leggere:**

- `frontend/src/crypto/frontend/src/crypto/primitives.ts` è la funzione `aesGcmEncrypt`. Accetta `Uint8Array` per il plaintext, il che significa che i dati binari funzionano direttamente senza alcuna conversione.
- `frontend/src/crypto/frontend/src/crypto/primitives.ts` è `aesGcmDecrypt`
- `backend/app/services/backend/app/services/message_service.py` è `store_encrypted_message` che memorizza su SurrealDB
- `backend/app/config.py` definisce `ENCRYPTED_CONTENT_MAX_LENGTH = 50000` (limita il contenuto inline dei messaggi, non gli upload di file)

**Approccio di implementazione:**

1. **Client: leggi e cifra il file**
 - Usa `FileReader.readAsArrayBuffer` per ottenere i byte grezzi
 - Per file piccoli (sotto 1MB): cifra l'intero file come una singola operazione AES-256-GCM usando una message key del Double Ratchet
 - Per file grandi (oltre 1MB): suddividi in segmenti da 1MB. Cifra ogni chunk con una chiave derivata: `chunk_key = HKDF(message_key, salt=chunk_index)`
 - Ogni chunk ha il proprio nonce (non riutilizzare mai i nonce)

2. **Client: carica il blob cifrato**
 - Crea un nuovo endpoint del backend: `POST /api/messages/file`
 - Invia il blob cifrato come multipart form data
 - Includi i metadati nella richiesta: nome file originale, tipo MIME, dimensione del file, numero di chunk, header di cifratura (stesso formato degli header dei messaggi di testo)

3. **Backend: memorizza il file cifrato**
 - Memorizza il blob cifrato su disco o in un object storage (non in SurrealDB, che non è progettato per grandi dati binari)
 - Memorizza i metadati del file in SurrealDB: `file_id`, `sender_id`, `recipient_id`, `room_id`, `filename`, `mime_type`, `size`, `chunk_count`, `storage_path`
 - Restituisci un `file_id` al client

4. **Client: invia il messaggio di file**
 - Invia un normale messaggio cifrato tramite il WebSocket, ma il contenuto in plaintext sono metadati JSON:
 ```json
 {
 "type": "file",
 "file_id": "abc123",
 "filename": "document.pdf",
 "mime_type": "application/pdf",
 "size": 1048576,
 "thumbnail": "<base64-encoded-encrypted-thumbnail>"
 }
 ```
 - Il destinatario decifra questi metadati, poi scarica il file cifrato tramite `file_id`, poi decifra il file lato client

5. **Per le immagini: genera thumbnail cifrate**
 - Prima dell'upload, usa `<canvas>` per ridimensionare l'immagine a una thumbnail (200x200)
 - Cifra la thumbnail separatamente
 - Includi la thumbnail cifrata inline nel messaggio (è abbastanza piccola)
 - Mostra subito la thumbnail, carica l'immagine completa al click

**Considerazioni di sicurezza:**
- Valida i limiti di dimensione dei file sul backend per prevenire il DoS (suggerimento: 100MB al massimo)
- Il server valida la dimensione del file e gli header del tipo MIME, ma non può verificare che il contenuto cifrato corrisponda al tipo MIME dichiarato. Il client deve validare dopo la decifratura.
- Valuta di usare chiavi di cifratura separate per i file e per i messaggi di testo. Se la chiave di un file viene compromessa, non deve compromettere il ratchet dei messaggi di testo.
- Rimuovi i dati EXIF dalle immagini prima della cifratura (EXIF contiene coordinate GPS, modello della fotocamera, timestamp)

**Dettaglio della cifratura a chunk:**

Per i file oltre 1MB serve una strategia di suddivisione in chunk. Ecco un approccio concreto:

1. Dividi il file in chunk da 1MB (1.048.576 byte)
2. Per ogni chunk, deriva una chiave per chunk: `chunk_key_i = HKDF(message_key, salt=uint32_to_bytes(i), info=b'file_chunk')`
3. Cifra ogni chunk: `encrypted_chunk_i = AES-GCM(chunk_key_i, chunk_i, associated_data=file_id || chunk_index || total_chunks)`
4. Il legame con i dati associati impedisce a un attaccante di riordinare, duplicare o troncare i chunk
5. Carica i chunk in sequenza o in parallelo (il server li memorizza come `{file_id}_chunk_{i}`)

Lato ricezione:
1. Scarica tutti i chunk
2. Deriva le stesse chiavi per chunk dalla message key
3. Decifra ogni chunk, verifica i dati associati
4. Concatena i chunk decifrati per ricostruire il file originale

**Tracciamento dell'avanzamento:**

Per i file grandi, mostra l'avanzamento di upload/download:
- Traccia i byte caricati/scaricati rispetto alla dimensione totale del file
- Mostra una barra di avanzamento nel componente `MessageBubble`
- Consenti l'annullamento a metà trasferimento (ripulisci gli upload parziali sul server)

**Come testare:**
- Invia un piccolo file di testo (.txt, sotto 1KB) tra due utenti, verifica che il destinatario possa scaricarlo e decifrarlo
- Invia un'immagine (.png, sotto 5MB), verifica che una thumbnail compaia subito e che l'immagine completa si carichi al click
- Invia un file grande (oltre 10MB), verifica che la cifratura a chunk funzioni e che compaia una barra di avanzamento
- Verifica che il filesystem del server contenga solo blob cifrati (apri un file memorizzato in un editor esadecimale, conferma che sembri dati casuali)
- Invia un file a un utente offline, verifica che possa scaricarlo e decifrarlo quando torna online
- Annulla l'upload di un file grande a metà trasferimento, verifica che i dati parziali vengano ripuliti
- Invia un file con un chunk manomesso (inverti un byte), verifica che la decifratura fallisca con un errore di autenticazione

---

### Sfida 7: Sincronizzazione multi-dispositivo

**Cosa costruire:** Permetti a un utente di essere connesso contemporaneamente su più dispositivi, con i messaggi sincronizzati su tutti i dispositivi mantenendo la cifratura E2E.

**Perché è importante:** È uno dei problemi più difficili della messaggistica cifrata E2E. Signal usa l'approccio dei "linked devices" in cui ogni dispositivo ha la propria identity key e la propria sessione ratchet con ogni contatto. WhatsApp usa un'architettura multi-dispositivo in cui il telefono è il dispositivo primario e gli altri dispositivi ottengono sessioni proxy. iMessage assegna a ogni dispositivo la propria coppia di chiavi e cifra ogni messaggio N volte (una per ogni dispositivo destinatario).

Ogni approccio ha proprietà diverse per sicurezza, UX e complessità. Questa sfida usa l'approccio di Signal perché è il più sicuro e la codebase supporta già più connessioni WebSocket per utente.

**Cosa imparerai:**
- La gestione delle identity key multi-dispositivo
- La moltiplicazione delle sessioni (N dispositivi significano N sessioni ratchet separate per contatto)
- Il collegamento e la verifica dei dispositivi
- La risoluzione dei conflitti nello stato crittografico distribuito

**Da dove iniziare a leggere:**

- `backend/app/core/websocket_manager.py` memorizza `active_connections: dict[UUID, list[WebSocket]]` che supporta già più connessioni per utente
- `backend/app/core/websocket_manager.py` applica `WS_MAX_CONNECTIONS_PER_USER` (predefinito 5, vedi `config.py`)
- `backend/app/core/websocket_manager.py` itera su TUTTE le connessioni di un utente quando invia un messaggio
- `backend/app/services/backend/app/services/prekey_service.py` inizializza le chiavi dell'utente con identity key, signed prekey e one-time prekey
- `backend/app/models/IdentityKey.py` memorizza una identity key per utente (questo deve cambiare)
- `backend/app/services/backend/app/services/message_service.py` `initialize_conversation` crea un ratchet per coppia di utenti (anche questo deve cambiare)

**Modifica dell'architettura:**

Attualmente, la relazione è:
```
Alice ----[one ratchet session]----> Bob
```

Con il multi-dispositivo diventa:
```
Alice-Phone ----[ratchet A1-B1]----> Bob-Phone
Alice-Phone ----[ratchet A1-B2]----> Bob-Laptop
Alice-Laptop ----[ratchet A2-B1]----> Bob-Phone
Alice-Laptop ----[ratchet A2-B2]----> Bob-Laptop
```

Se Alice ha 2 dispositivi e Bob ne ha 3, tra loro ci sono 2 x 3 = 6 sessioni ratchet. Quando Alice invia un messaggio a Bob, lo cifra 3 volte (una per ogni dispositivo di Bob). Ciascun dispositivo di Bob decifra in modo indipendente.

**Fasi di implementazione:**

**Fase 1: Modifiche allo schema del database**
- Aggiungi una colonna `device_id` ai modelli `IdentityKey`, `SignedPrekey` e `OneTimePrekey`. Cambia il vincolo di unicità su `IdentityKey` da `(user_id)` a `(user_id, device_id)`.
- Aggiungi una tabella `devices`: `device_id`, `user_id`, `device_name`, `created_at`, `last_active`.
- Lo stato del ratchet lato client in `frontend/src/crypto/key-store.ts` diventa indicizzato per `(peer_user_id, peer_device_id)` invece che solo per `peer_id`.

**Fase 2: Registrazione del dispositivo**
- Quando un utente registra un nuovo dispositivo (tramite WebAuthn), genera una nuova coppia di identity key per quel dispositivo.
- Ogni dispositivo carica il proprio prekey bundle (identity key, signed prekey, one-time prekey).
- L'endpoint del prekey bundle (`backend/app/services/prekey_service.py`) ha bisogno di un parametro `device_id`.

**Fase 3: Fan-out dei messaggi**
- Il client di Alice recupera tutti i prekey bundle per dispositivo di Bob, poi esegue X3DH e cifra una volta per dispositivo.
- Il payload WebSocket da Alice al server ora porta `device_id` insieme a `recipient_id`.
- La logica di inoltro in `backend/app/services/websocket_service.py:handle_encrypted_message` instrada ogni ciphertext verso la connessione del dispositivo destinatario.

**Fase 4: Collegamento dei dispositivi**
- Un nuovo dispositivo NON dovrebbe semplicemente registrarsi in modo indipendente. Dovrebbe essere "collegato" da un dispositivo esistente.
- Flusso di collegamento: il nuovo dispositivo mostra un codice QR contenente la sua chiave pubblica temporanea. Il dispositivo esistente scansiona il codice QR, esegue uno scambio di chiavi con il nuovo dispositivo e carica sul server un record di "device link".
- Senza il collegamento, il server potrebbe registrare un dispositivo rogue e intercettare i messaggi.

**Consegna dei messaggi a più dispositivi:**

Quando Alice invia un messaggio a Bob (che ha 3 dispositivi), il flusso è:

1. Il client di Alice recupera i prekey bundle di TUTTI i dispositivi di Bob: `GET /encryption/prekey-bundle/{bob_id}?all_devices=true`
2. Alice esegue X3DH separatamente con il prekey bundle di ciascun dispositivo (3 scambi di chiavi separati)
3. Alice cifra il messaggio 3 volte (una per sessione ratchet)
4. Alice invia 3 payload cifrati al server, ciascuno etichettato con `(recipient_id, device_id)`
5. Il server inoltra ciascun payload al dispositivo corretto

Questo significa che la funzione `store_encrypted_message` in `backend/app/services/message_service.py` ha bisogno di un campo `device_id`. Lo schema dei messaggi in SurrealDB passa da `{sender_id, recipient_id, ciphertext}` a `{sender_id, sender_device_id, recipient_id, recipient_device_id, ciphertext}`.

**Verifica dei dispositivi:**

Come fa Alice a sapere che sta cifrando verso i dispositivi reali di Bob e non verso un dispositivo rogue iniettato dal server? È lo stesso problema di fiducia del caso a dispositivo singolo, ma amplificato. Ogni dispositivo ha il proprio "safety number" (un hash della coppia di identity key). Gli utenti dovrebbero verificare i safety number per ogni dispositivo, non solo per ogni contatto.

Valuta di implementare una lista dei dispositivi che mostri tutti i dispositivi registrati con i loro safety number:
```
Bob's Devices:
 Phone (registered Jan 15) - Safety: 4a7b 2c3d ...
 Laptop (registered Jan 20) - Safety: 9f1e 8d2c ...
 Tablet (registered Feb 1) - Safety: 3b5a 7c4e ...
```

**Come testare:**
- Registra due dispositivi per lo stesso utente (due profili browser diversi, oppure uno normale + uno in incognito)
- Invia un messaggio da un terzo utente all'utente multi-dispositivo
- Verifica che ENTRAMBI i dispositivi ricevano e possano decifrare il messaggio in modo indipendente
- Verifica che ogni dispositivo abbia uno stato ratchet indipendente (far avanzare il ratchet sul dispositivo A non influisce sul dispositivo B)
- Disconnetti un dispositivo, invia messaggi, riconnetti e verifica che il dispositivo riconnesso si allinei
- Registra un terzo dispositivo e verifica che stabilisca nuove sessioni ratchet con tutti i contatti esistenti
- Rimuovi un dispositivo e verifica che non possa più decifrare i nuovi messaggi

---

## Sfide avanzate

---

### Sfida 8: Cifratura di gruppo con Sender Keys

**Cosa costruire:** Implementa la cifratura delle chat di gruppo usando il protocollo Sender Keys (usato da Signal per la messaggistica di gruppo). In una chat di gruppo, ogni membro ha una "sender key" che distribuisce a tutti gli altri membri. Quando invii un messaggio di gruppo, cifri una sola volta con la tua sender key, e tutti i membri possono decifrare.

**Perché è importante:** Nella cifratura 1:1 cifri ogni messaggio una volta. In un gruppo di N membri, l'approccio ingenuo cifra ogni messaggio N-1 volte (una per ogni altro membro). Le Sender Keys riducono questo a una cifratura O(1) per messaggio, il che conta su larga scala. Un gruppo di 1000 membri richiederebbe 999 cifrature per messaggio senza le Sender Keys.

Signal usa le Sender Keys per le chat di gruppo. WhatsApp ha adottato lo stesso approccio. Il compromesso è una forward secrecy più debole rispetto al Double Ratchet, perché la chain della sender key avanza solo in una direzione (nessun DH ratchet).

**Cosa imparerai:**
- Il design e l'implementazione del protocollo Sender Keys
- La distribuzione e la rotazione delle chiavi di gruppo
- La gestione delle chiavi per l'aggiunta/rimozione di membri
- I compromessi di sicurezza tra cifratura di gruppo O(1) e cifratura a coppie O(N)

**Perché è difficile:**
- Aggiungere un nuovo membro richiede di distribuirgli TUTTE le sender key esistenti (tramite canali cifrati 1:1)
- Rimuovere un membro richiede di ruotare TUTTE le sender key (il membro rimosso conosce le vecchie chiavi e potrebbe derivarne tutte le future chain key)
- I membri che perdono una rotazione delle chiavi non possono decifrare i nuovi messaggi finché non ricevono la nuova sender key
- La forward secrecy è più debole rispetto all'1:1. Compromettere una sender key espone TUTTI i messaggi futuri di quel mittente fino alla rotazione successiva.

**Da dove iniziare a leggere:**

- `backend/app/api/rooms.py` crea le stanze con i partecipanti. Da qui parte il supporto ai gruppi.
- `frontend/src/crypto/double-ratchet.ts` ha la derivazione della chain key (`_kdf_ck`) che è lo stesso pattern usato nelle Sender Keys (chain a direzione singola)
- `backend/app/core/websocket_manager.py` `broadcast_to_room` invia a tutti i membri della stanza
- `frontend/src/crypto/frontend/src/crypto/primitives.ts` la cifratura AES-GCM funziona anche per i messaggi con sender key

**Fasi di implementazione:**

**Fase 1: Ricerca**

Leggi la specifica delle Sender Keys di Signal. Comprendi la differenza tra "sender key distribution message" (inviati tramite ratchet 1:1) e normali messaggi di gruppo (cifrati con la sender key).

Una sender key è una coppia: `(chain_key, signing_key)`. La chain key avanza come un ratchet a direzione singola (basato su HMAC). La signing key autentica il mittente.

**Fase 2: Schema del database**

Crea queste tabelle in SurrealDB:

```
sender_keys {
 group_id: string,
 user_id: string,
 chain_key: bytes,
 chain_iteration: int,
 signing_public_key: bytes,
 created_at: datetime
}

group_members {
 group_id: string,
 user_id: string,
 role: string, // "admin" | "member"
 joined_at: datetime
}
```

**Fase 3: Implementazione**

1. **Creazione del gruppo:** Quando Alice crea un gruppo con Bob e Carol:
 - Alice genera una coppia di sender key: `(chain_key_alice, signing_key_alice)`
 - Alice invia la sua sender key a Bob tramite il loro canale cifrato 1:1 esistente (Double Ratchet)
 - Alice invia la sua sender key a Carol allo stesso modo
 - Bob e Carol fanno lo stesso (ciascuno distribuisce la propria sender key a tutti gli altri membri)
 - Risultato: ogni membro ha N-1 sender key (una per ogni altro membro)

2. **Invio di un messaggio:** Alice cifra il messaggio di gruppo:
 - Deriva la message key: `message_key = HMAC-SHA256(chain_key, 0x01)`
 - Avanza la chain: `chain_key = HMAC-SHA256(chain_key, 0x02)`
 - Cifra: `ciphertext = AES-GCM(message_key, plaintext)`
 - Firma: `signature = Ed25519.sign(signing_key, ciphertext)`
 - Trasmetti al gruppo ciphertext + signature + chain_iteration

3. **Ricezione di un messaggio:** Bob decifra:
 - Cerca la sender key di Alice tramite `(group_id, sender_id)`
 - Se `chain_iteration > local_iteration`: avanza la chain key per allinearti
 - Deriva la message key, decifra, verifica la firma

4. **Rimozione di un membro:** Quando Carol viene rimossa:
 - Alice genera una NUOVA coppia di sender key
 - Alice distribuisce la nuova sender key a Bob (ma NON a Carol)
 - Bob fa lo stesso
 - Carol ha ancora le vecchie sender key ma non vengono più usate

5. **Aggiunta di un membro:** Quando Dave si unisce:
 - Tutti i membri esistenti distribuiscono a Dave le loro sender key correnti tramite canali 1:1
 - Dave genera e distribuisce la propria sender key a tutti i membri

**Strategia di rotazione delle chiavi:**

Le sender key dovrebbero ruotare periodicamente, non solo alla rimozione di un membro. Valuta di ruotare dopo ogni 100 messaggi o ogni 24 ore, a seconda di quale condizione si verifica per prima. Questo limita il blast radius di una sender key compromessa: un attaccante che ottiene la sender key di Alice può decifrare solo i messaggi della chain corrente di Alice, non quelli delle chain passate.

Quando una sender key ruota:
1. Il mittente genera una nuova coppia di sender key
2. Il mittente distribuisce la nuova chiave a tutti i membri del gruppo tramite canali cifrati 1:1
3. Il mittente include un flag di "key rotation" nel primo messaggio con la nuova chiave
4. I destinatari che ricevono un messaggio con una chain di sender key sconosciuta devono richiedere un key distribution message

**Gestione degli errori per le distribuzioni di chiavi perse:**

Se Bob perde una distribuzione di chiavi (era offline quando Alice ha ruotato la sua sender key), non può decifrare i nuovi messaggi di Alice. Gestiscilo con grazia:
1. Bob riceve un messaggio da Alice con un'iterazione della chain sconosciuta
2. Il client di Bob invia un messaggio "request sender key" ad Alice tramite il canale 1:1
3. Alice ridistribuisce a Bob la sua sender key corrente
4. Bob ora può decifrare i messaggi in buffer

Questo meccanismo di recupero è fondamentale. Senza di esso, una singola distribuzione di chiavi persa rompe in modo permanente il gruppo per quel membro.

**Come testare:**
- Crea un gruppo con 3 utenti
- Invia un messaggio da ciascun utente, verifica che tutti i membri possano decifrare
- Rimuovi un membro, invia un nuovo messaggio, verifica che il membro rimosso non possa decifrare
- Aggiungi un nuovo membro, verifica che possa decifrare i nuovi messaggi
- Verifica che non possa decifrare i messaggi inviati prima del suo ingresso (a meno che tu non implementi esplicitamente la condivisione della cronologia)
- Metti offline un membro, ruota una sender key, riportalo online, verifica che possa recuperare tramite il meccanismo di richiesta della chiave
- Invia 101 messaggi da un utente e verifica che scatti la rotazione automatica della chiave

---

### Sfida 9: Scambio di chiavi post-quantistico (X25519 + Kyber ibrido)

**Cosa costruire:** Aggiungi uno scambio di chiavi ibrido che combina il classico X25519 con CRYSTALS-Kyber (ML-KEM), rendendo l'accordo iniziale sulle chiavi resistente agli attacchi dei computer quantistici e mantenendo la sicurezza contro i computer classici.

**Perché è importante:** I computer quantistici che eseguono l'algoritmo di Shor possono rompere tutta la crittografia a chiave pubblica attuale basata sul problema del logaritmo discreto, inclusi X25519 ed Ed25519. Non è una speculazione ipotetica sulle tempistiche. Signal ha aggiunto la protezione post-quantistica (PQXDH) a settembre 2023. Google Chrome usa ML-KEM ibrido per TLS dal 2024. Il NIST ha finalizzato lo standard ML-KEM (FIPS 203) ad agosto 2024.

Il threat model che motiva tutto questo è "harvest now, decrypt later". Un avversario registra oggi il traffico cifrato, lo conserva e lo decifra tra 10 anni quando avrà un computer quantistico. La protezione PQ ibrida lo impedisce.

**Cosa imparerai:**
- I fondamenti della crittografia post-quantistica (incapsulamento di chiavi basato su reticoli)
- Il design dello scambio di chiavi ibrido (classico + PQ combinati)
- Il meccanismo di incapsulamento di chiavi CRYSTALS-Kyber / ML-KEM
- Aggiornare un protocollo crittografico in produzione senza rompere la retrocompatibilità

**Da dove iniziare a leggere:**

- `frontend/src/crypto/x3dh.ts` è `perform_x3dh_sender`. Questa è la funzione esatta che modificherai.
- `frontend/src/crypto/x3dh.ts` è il punto in cui vengono combinati i segreti condivisi DH:
 ```python
 dh1 = alice_ik_private.exchange(bob_spk_public)
 dh2 = alice_ek_private.exchange(bob_ik_public)
 dh3 = alice_ek_private.exchange(bob_spk_public)
 # dh4 = alice_ek_private.exchange(bob_opk_public) (optional)
 key_material = dh1 + dh2 + dh3 [+ dh4]
 ```
- `frontend/src/crypto/x3dh.ts` è il punto in cui HKDF deriva la chiave condivisa:
 ```python
 f = b'\xff' * X25519_KEY_SIZE
 hkdf = HKDF(algorithm=hashes.SHA256, length=X25519_KEY_SIZE,
 salt=b'\x00' * X25519_KEY_SIZE, info=b'X3DH')
 shared_key = hkdf.derive(f + key_material)
 ```
- `backend/pyproject.toml:30` include già `liboqs-python>=0.14.1` tra le dipendenze

**L'approccio ibrido:**

L'intuizione chiave è che combini ENTRAMBI i segreti condivisi, classico e post-quantistico. Se uno dei due è sicuro, il risultato combinato è sicuro. Questo significa:
- Se Kyber è rotto ma X25519 no, sei comunque al sicuro (sicurezza classica)
- Se X25519 è rotto (computer quantistico) ma Kyber no, sei comunque al sicuro (sicurezza PQ)
- Devono essere rotti entrambi contemporaneamente per compromettere la sessione

```
Classical: SK_classical = HKDF(DH1 || DH2 || DH3 [|| DH4])
Post-quantum: SK_pq = Kyber.Decapsulate(ciphertext, secret_key)
Combined: SK = HKDF(SK_classical || SK_pq, info=b'PQXDH')
```

**Fasi di implementazione:**

**Fase 1: Ricerca**

Leggi queste specifiche:
- La specifica PQXDH di Signal: https://signal.org/docs/specifications/pqxdh/
- NIST FIPS 203 (ML-KEM): https://csrc.nist.gov/pubs/fips/203/final

Comprendi la differenza tra un KEM (Key Encapsulation Mechanism) e uno scambio DH. In Kyber:
- Bob genera una coppia di chiavi Kyber e pubblica la chiave pubblica
- Alice chiama `Encapsulate(bob_public_key)` che restituisce `(shared_secret, ciphertext)`
- Alice invia `ciphertext` a Bob
- Bob chiama `Decapsulate(ciphertext, bob_private_key)` che restituisce `shared_secret`
- Ora entrambe le parti hanno lo stesso `shared_secret`

**Fase 2: Estensione del prekey bundle**

1. Aggiungi un campo `kyber_public_key` alla dataclass `PreKeyBundle` in `frontend/src/crypto/x3dh.ts`
2. Quando generi i prekey (`backend/app/services/prekey_service.py`), genera anche una coppia di chiavi Kyber usando `liboqs-python`:
 ```python
 import oqs
 kem = oqs.KeyEncapsulation("Kyber768")
 kyber_public = kem.generate_keypair
 kyber_private = kem.export_secret_key
 ```
3. Memorizza la chiave pubblica Kyber nel prekey bundle, la chiave privata nel database

**Fase 3: Estensione di X3DH**

Modifica `perform_x3dh_sender` in `frontend/src/crypto/x3dh.ts`:

1. Dopo aver calcolato il `key_material` classico, controlla se il bundle include una chiave Kyber
2. Se sì: incapsula rispetto alla chiave pubblica Kyber per ottenere `(pq_shared_secret, pq_ciphertext)`
3. Concatena: `combined_material = key_material + pq_shared_secret`
4. Cambia il parametro info di HKDF: `info=b'PQXDH'` (per distinguere dalle sessioni classiche)
5. Includi `pq_ciphertext` nell'`X3DHResult` in modo che possa essere inviato al destinatario

Modifica `perform_x3dh_receiver` in `frontend/src/crypto/x3dh.ts`:

1. Dopo aver calcolato il key material classico, controlla se è stato fornito un ciphertext PQ
2. Se sì: decapsula per recuperare `pq_shared_secret`
3. Combina e deriva come sopra

**Fase 4: Retrocompatibilità**

Se il peer non ha una chiave pubblica Kyber nel suo prekey bundle (non si è aggiornato), ripiega su X3DH solo classico. Il percorso di codice in `frontend/src/crypto/x3dh.ts` gestisce già l'one-time prekey opzionale con un if/else. Usa lo stesso pattern per Kyber.

**Dimensioni delle chiavi da tenere presenti:**
- Chiave pubblica X25519: 32 byte
- Chiave pubblica Kyber-768: 1184 byte
- Ciphertext Kyber-768: 1088 byte
- Segreto condiviso Kyber-768: 32 byte

La dimensione del prekey bundle aumenta in modo significativo. Tienine conto nello schema del database e nei limiti di dimensione dei messaggi WebSocket.

**Come testare:**
- Due client compatibili PQ stabiliscono una sessione. Verifica che la chiave condivisa differisca da quella di una sessione solo classica con lo stesso key material.
- Un client compatibile PQ parla con un client solo classico. Verifica che ripieghi con grazia.
- Genera 1000 sessioni PQ e verifica che tutte le chiavi condivise siano uniche.
- Benchmark: misura il tempo di X3DH con e senza Kyber. Kyber dovrebbe aggiungere meno di 10ms.

---

### Sfida 10: Messaggi che scompaiono con applicazione crittografica

**Cosa costruire:** Messaggi che si cancellano automaticamente dopo un periodo di tempo configurabile, con la cancellazione imposta a livello crittografico. La chiave di decifratura viene distrutta allo scadere del timer, rendendo il messaggio permanentemente illeggibile anche se il ciphertext persiste.

**Perché è importante:** La funzione dei messaggi che scompaiono di Signal è una delle più popolari. Ma le implementazioni ingenue cancellano solo l'elemento dell'UI mentre i dati restano nello storage. Se un esaminatore forense recupera il database, i messaggi "cancellati" sono ancora lì in plaintext o con le chiavi intatte.

Questa sfida implementa una vera cancellazione crittografica: il ciphertext resta in SurrealDB (non puoi garantire la cancellazione fisica da un database), ma la chiave necessaria a decifrarlo viene distrutta in modo sicuro. Senza la chiave, il ciphertext è indistinguibile dal rumore casuale.

**Cosa imparerai:**
- La cancellazione crittografica (distruggere le chiavi invece dei dati)
- La gestione del ciclo di vita delle chiavi basata su timer
- Il wiping sicuro della memoria (più difficile di quanto sembri in JavaScript e Python)
- La sincronizzazione distribuita dei timer tra mittente e destinatario

**Da dove iniziare a leggere:**

- `frontend/src/crypto/double-ratchet.ts` è `_kdf_ck` che deriva le message key. Ogni messaggio ottiene una chiave univoca.
- `frontend/src/crypto/double-ratchet.ts` è `_store_skipped_message_keys` con la logica di eviction -234. Questo è il pattern che estenderai per l'eviction basata su TTL.
- `frontend/src/crypto/double-ratchet.ts` è `_evict_oldest_skipped_keys` che cancella le chiavi.
- `frontend/src/crypto/key-store.ts` memorizza lo stato del ratchet in IndexedDB
- `frontend/src/crypto/frontend/src/crypto/primitives.ts` ha `generateRandomBytes` che usa `crypto.getRandomValues`

**Approccio di implementazione:**

1. **Impostazione della conversazione:** Aggiungi un'impostazione per conversazione: `disappear_after_seconds`. Valori: 0 (disabilitato), 30, 300, 3600, 86400, 604800. Memorizzala nei metadati della stanza in SurrealDB.

2. **Memorizzazione della message key con TTL:** Quando un messaggio viene cifrato, la message key normalmente viene scartata dopo la cifratura. Per i messaggi che scompaiono, il DESTINATARIO memorizza la message key in IndexedDB con un TTL:
 ```
 {
 message_id: "msg_abc123",
 message_key: <32 bytes>,
 created_at: "2025-01-15T14:30:00Z",
 expires_at: "2025-01-15T15:30:00Z" // created_at + disappear_after_seconds
 }
 ```

3. **Il timer parte alla lettura, non all'invio.** Il timer del destinatario parte quando decifra il messaggio per la prima volta. Se il destinatario resta offline per una settimana, ottiene comunque l'intera durata del timer dopo la lettura.

4. **Distruzione della chiave:** Quando il timer scade:
 - Frontend: sovrascrivi il key material con byte casuali prima di cancellarlo da IndexedDB. In JavaScript: `crypto.getRandomValues(keyBuffer)` poi cancella.
 - Backend: cancella semplicemente la riga di SurrealDB. Il server non detiene chiavi, quindi non c'è nulla da cancellare — ma il plaintext in cache locale del destinatario (e la message key, se è ancora nella cache delle skipped key) deve essere sovrascritto sul client.

5. **Modifiche all'UI:**
 - Mostra un timer di conto alla rovescia sui messaggi che scompaiono
 - Dopo la scadenza, sostituisci il contenuto del messaggio con "[Message expired]"
 - Il ciphertext resta in SurrealDB ma ora è permanentemente illeggibile

**Le parti difficili:**

- Cosa succede se il destinatario non legge mai il messaggio? Anche la copia del mittente dovrebbe comunque scomparire a un certo punto. Implementa una "maximum lifetime" che scatta indipendentemente dallo stato di lettura.
- E per gli screenshot? Non puoi impedirli. È un problema sociale, non tecnico. Signal mostra una notifica quando viene fatto uno screenshot (su mobile), ma è un tentativo best-effort.
- Il garbage collection di JavaScript non garantisce che la memoria venga cancellata. Quando sovrascrivi un `Uint8Array`, i vecchi valori possono ancora esistere in memoria finché il GC non recupera la pagina. Una vera cancellazione sicura in JavaScript non è possibile. Documenta questa limitazione.
- La cache delle skipped message key (`frontend/src/crypto/double-ratchet.ts`) memorizza le chiavi per i messaggi fuori ordine. Anche queste devono rispettare il TTL.

**Come testare:**
- Abilita i messaggi che scompaiono (timer di 30 secondi) in una conversazione
- Invia un messaggio, verifica che compaia normalmente
- Aspetta 30 secondi, verifica che il contenuto del messaggio scompaia e mostri "[Message expired]"
- Controlla IndexedDB per verificare che la message key non ci sia più
- Verifica che il ciphertext esista ancora in SurrealDB ma non possa essere decifrato (scrivi uno script di test che tenti la decifratura e confermi che fallisce)

---

## Sfide da esperti

---

### Sfida 11: Autenticazione deniabile (Triple DH)

**Tempo stimato:** 2-3 settimane

**Cosa costruire:** Implementa l'autenticazione deniabile in modo che né Alice né Bob possano dimostrare a una terza parte che l'altro ha inviato uno specifico messaggio. È una proprietà di privacy del Signal Protocol che la maggior parte delle implementazioni salta o sbaglia.

**Prerequisiti:** Solida comprensione dell'implementazione X3DH in `frontend/src/crypto/x3dh.ts`

**Perché è importante:** Nell'implementazione attuale, se Alice ha i messaggi firmati di Bob (ciphertext + tag di autenticazione), potrebbe potenzialmente mostrarli a una terza parte per dimostrare che Bob ha detto qualcosa. Questo perché i dati associati (`frontend/src/crypto/x3dh.ts`) legano entrambe le identity key alla sessione: `associated_data = alice_ik_public_bytes + bob_ik_public_bytes`.

L'autenticazione deniabile significa che Alice avrebbe potuto falsificare i messaggi da sola (perché ha anche il segreto condiviso), quindi non sono una prova crittografica valida della paternità di Bob. Questo conta per giornalisti, attivisti, whistleblower e chiunque possa essere costretto legalmente a dimostrare la paternità di un messaggio.

**Il problema centrale:**

X3DH fornisce un'autenticazione implicita tramite le operazioni DH che coinvolgono le identity key. Sia DH1 (`IK_A x SPK_B`) sia DH2 (`EK_A x IK_B`) coinvolgono almeno una identity key. Una terza parte che si fida di Alice e conosce entrambe le identity key potrebbe verificare che la sessione sia stata stabilita tra quelle specifiche identità.

In un protocollo deniabile, il MAC di autenticazione può essere calcolato da UNA QUALSIASI delle due parti, quindi nessuna delle due può dimostrare che l'ha calcolato l'altra.

**Da dove iniziare a leggere:**

- `frontend/src/crypto/x3dh.ts` è il calcolo dei dati associati: `associated_data = alice_ik_public_bytes + bob_ik_public_bytes`
- `frontend/src/crypto/double-ratchet.ts` è `encrypt_message` dove i dati associati vengono usati in AES-GCM
- `frontend/src/crypto/double-ratchet.ts` è `decrypt_message` che verifica il tag di autenticazione

**Approccio di implementazione:**

1. Sostituisci il legame dei dati associati con uno schema di commitment. Invece di `AD = IK_A || IK_B`, usa `AD = HMAC(shared_key, IK_A || IK_B)`. Entrambe le parti possono calcolare questo MAC (hanno entrambe la chiave condivisa), quindi nessuna può dimostrare che l'ha fatto l'altra.

2. In alternativa, implementa una variante "triple DH" in cui aggiungi un terzo DH: `DH5 = IK_A x IK_B`. Questo crea un segreto condiviso che richiede solo le identity key di Alice e Bob. Poiché entrambe le parti possono calcolarlo, una trascrizione di messaggi autenticati con questo segreto non è una prova contro nessuna delle due.

3. La modifica dei dati associati in `frontend/src/crypto/x3dh.ts` e la riga corrispondente in `frontend/src/crypto/x3dh.ts` devono cambiare insieme (mittente e destinatario devono produrre gli stessi dati associati).

**Come verificare la deniabilità:**

Scrivi un programma che:
1. Prenda una trascrizione di messaggi (ciphertext + tag di autenticazione)
2. Prenda SOLO il key material di Alice (non le chiavi private di Bob)
3. Tenti di falsificare una trascrizione valida che sembri identica
4. Se ci riesce, il protocollo è deniabile (Alice avrebbe potuto creare la trascrizione da sola)

---

### Sfida 12: Backup sicuro e recupero

**Tempo stimato:** 3-4 settimane

**Cosa costruire:** Permetti agli utenti di creare backup cifrati della cronologia dei messaggi e delle chiavi di cifratura, memorizzati lato server ma cifrati con una chiave di backup controllata dall'utente. Questo consente il recupero dell'account su un nuovo dispositivo senza perdere la cronologia dei messaggi.

**Prerequisiti:** Completa la Sfida 7 (sincronizzazione multi-dispositivo). Devi capire la gestione delle chiavi dei dispositivi prima di aggiungere il backup delle chiavi.

**Perché è importante:** Il più grande problema di UX della cifratura E2E è che perdere il dispositivo significa perdere per sempre la cronologia dei messaggi. È matematicamente inevitabile se il key material esiste solo su un dispositivo.

WhatsApp l'ha risolto con backup su Google Drive/iCloud cifrati con una password dell'utente (usando Argon2id). Signal l'ha risolto con il Secure Value Recovery (SVR) basato su PIN, che usa enclave SGX per limitare il rate dei tentativi di indovinare il PIN. Entrambi gli approcci hanno dei compromessi.

**Domande di pianificazione a cui devi rispondere prima di scrivere codice:**

1. Quale key material deve essere sottoposto a backup?
 - Identity key (chiavi private X25519 + Ed25519)
 - Stati ratchet attivi per ogni conversazione
 - Chiavi private delle signed prekey
 - Cronologia dei messaggi (opzionale, grande)

2. Come viene cifrato il backup?
 - Opzione A: PIN/password dell'utente -> Argon2id -> chiave AES-256-GCM
 - Opzione B: chiave di backup casuale mostrata come frase di recupero di 24 parole
 - Opzione C: HSM lato server con rate limiting (l'approccio SVR di Signal, richiede hardware)

3. Dove viene memorizzato il backup?
 - Lato server (blob cifrato in PostgreSQL o in un object storage)
 - File esportato dall'utente (come l'export in plaintext di Signal, ma cifrato)

4. Come gestisci la perdita della chiave di backup?
 - Non recuperabile per design (il più sicuro)
 - Recupero sociale (N su M contatti fidati devono approvare, come lo Shamir's Secret Sharing)
 - Recupero lato server con verifica dell'identità (il più debole, richiede di fidarsi del server)

**Da dove iniziare a leggere:**

- `frontend/src/crypto/key-store.ts` salva le identity key in IndexedDB. Questi sono i dati di cui serve il backup.
- `frontend/src/crypto/key-store.ts` salva gli stati del ratchet
- `frontend/src/crypto/key-store.ts` `clearAllKeys` cancella tutto al logout
- `frontend/src/crypto/message-store.ts` salva i messaggi decifrati

**Fasi di implementazione:**

**Fase 1: Design del formato di backup**

Definisci uno schema JSON per il backup:
```json
{
 "version": 1,
 "created_at": "2025-01-15T14:30:00Z",
 "identity_keys": {
 "x25519_private": "<base64>",
 "x25519_public": "<base64>",
 "ed25519_private": "<base64>",
 "ed25519_public": "<base64>"
 },
 "ratchet_states": [
 {
 "peer_id": "bob-uuid",
 "root_key": "<base64>",
 "sending_chain_key": "<base64>",
 "receiving_chain_key": "<base64>",
 "dh_private_key": "<base64>",
 "dh_peer_public_key": "<base64>",
 "sending_message_number": 42,
 "receiving_message_number": 37,
 "previous_sending_chain_length": 12
 }
 ],
 "messages": []
}
```

**Fase 2: Cifratura del backup**

1. L'utente inserisce un PIN o una passphrase
2. Deriva la chiave: `backup_key = Argon2id(passphrase, salt, time_cost=3, memory_cost=65536, parallelism=4)`
3. Cifra: `encrypted_backup = AES-256-GCM(backup_key, JSON.stringify(backup_data))`
4. Memorizza `{salt, encrypted_backup, nonce}` sul server

**Fase 3: Endpoint di upload/download del backup**

- `POST /api/backup` - carica il blob di backup cifrato
- `GET /api/backup` - scarica il blob di backup cifrato
- `DELETE /api/backup` - cancella il backup

Il server NON ha MAI la chiave di backup. Memorizza solo byte cifrati.

**Fase 4: Flusso di ripristino**

1. L'utente effettua il login su un nuovo dispositivo con WebAuthn
2. Il server restituisce il blob di backup cifrato
3. L'utente inserisce il PIN/passphrase
4. Il client deriva la chiave di backup, decifra il backup
5. Il client scrive le identity key e gli stati del ratchet in IndexedDB
6. Il client riprende le conversazioni esistenti usando gli stati del ratchet ripristinati

**Fase 5: Gestione dei casi limite**

Cosa succede quando il dispositivo ripristinato e quello originale sono entrambi attivi? I loro stati del ratchet divergono immediatamente (il primo messaggio di uno dei due dispositivi fa avanzare il ratchet in modo diverso su ciascuno). È lo stesso problema della Sfida 7 (Multi-Device). La soluzione è: al ripristino, il nuovo dispositivo usa il backup come punto di partenza ma stabilisce NUOVE sessioni ratchet con tutti i contatti (in sostanza ricifrando con nuove chiavi ogni conversazione).

**Criteri di successo:**
- L'utente può creare un backup cifrato dall'UI
- Il backup può essere ripristinato su un nuovo dispositivo (profilo browser diverso)
- Il dispositivo ripristinato può inviare e ricevere messaggi nelle conversazioni esistenti
- Il blob di backup è inutile senza la passphrase (verifica tentando la decifratura con una passphrase sbagliata)
- L'amministratore del server non può leggere il contenuto del backup (verifica controllando il contenuto del database)

---

## Progetti Mix and Match

Questi combinano più sfide in un'unica applicazione coerente.

### Secure Team Messenger

**Combina:** Cifratura di gruppo (8) + Condivisione di file (6) + Conferme di lettura (1) + Multi-dispositivo (7)

Costruisci un'app di messaggistica per team simile alla chat di gruppo di Signal con allegati e stato di consegna su più dispositivi. Copre l'insieme di funzionalità di un concorrente base di Slack con cifratura E2E.

### Piattaforma per whistleblower

**Combina:** Messaggi che scompaiono (10) + Autenticazione deniabile (11) + Condivisione di file (6)

Costruisci una piattaforma in cui le fonti possono condividere documenti in modo sicuro con i giornalisti. I messaggi si autodistruggono dopo la lettura, non possono essere attribuiti a nessuna delle due parti, e i file sono cifrati in transito e a riposo. Pensa a SecureDrop ma con chat in tempo reale.

### Enterprise Secure Chat

**Combina:** Tutte le sfide facili + Cifratura di gruppo (8) + Backup/Recupero (12) + Multi-dispositivo (7)

Costruisci un sistema di messaggistica aziendale con funzionalità di compliance (backup e recupero per i legal hold) mantenendo la cifratura E2E per il contenuto dei messaggi. La tensione tra compliance (l'azienda deve poter fare audit dei messaggi) e privacy (la cifratura E2E lo impedisce) è una vera sfida di design del prodotto. Studia come la gestiscono Wickr Enterprise ed Element (Matrix).

---

## Sfide di performance

---

### Ottimizzare l'archiviazione dello stato del ratchet

L'implementazione attuale serializza l'intero stato del ratchet su PostgreSQL a ogni singolo messaggio. Guarda `backend/app/services/message_service.py` (`_save_ratchet_state_to_db`). Ogni chiamata a `send_encrypted_message` o `decrypt_received_message` innesca una scrittura completa dello stato del ratchet con `await session.commit` .

Questo è un collo di bottiglia. Per una conversazione con scambi rapidi di messaggi, stai scrivendo su PostgreSQL due volte per messaggio (una per la cifratura, una per la decifratura).

**Il tuo compito:** Profila l'overhead di serializzazione e implementa un livello di caching usando Redis.

**Approccio:**
1. Al primo caricamento dello stato del ratchet, mettilo in cache su Redis con una chiave come `ratchet:{user_id}:{peer_user_id}`
2. Dopo ogni avanzamento del ratchet, scrivi solo su Redis (sub-millisecondo)
3. Periodicamente (ogni 10 messaggi o ogni 30 secondi) esegui il flush su PostgreSQL (archiviazione durevole)
4. Alla disconnessione WebSocket (`websocket_manager.py`), esegui il flush di tutti gli stati in cache su PostgreSQL
5. All'avvio, carica da PostgreSQL (Redis non è durevole tra i riavvii)

**Misurazione:** Fai il benchmark dei messaggi al secondo prima e dopo con un semplice load test. Obiettivo: più di 100 messaggi al secondo per conversazione.

**Rischio:** Se il server va in crash tra la scrittura su Redis e il flush su PostgreSQL, perdi lo stato del ratchet. Questo significa che il messaggio successivo non verrà decifrato (desincronizzazione del ratchet). Mitiga con un task di flush in background e intervalli di flush brevi.

**Passi dettagliati di profiling:**

1. Aggiungi strumentazione di timing attorno a `_save_ratchet_state_to_db`:
 ```python
 import time
 start = time.perf_counter
 await session.commit
 elapsed_ms = (time.perf_counter - start) * 1000
 logger.info("Ratchet state save: %.2fms", elapsed_ms)
 ```

2. Esegui un load test: due utenti che si scambiano 1000 messaggi il più velocemente possibile. Misura:
 - Tempo medio per chiamata a `_save_ratchet_state_to_db`
 - Tempo medio per chiamata a `_load_ratchet_state_from_db`
 - Messaggi totali al secondo (end-to-end)
 - Utilizzo del connection pool di PostgreSQL (`DB_POOL_SIZE` in `config.py`)

3. Implementa il caching su Redis, ripeti lo stesso load test, confronta.

**Schema della cache Redis:**
```
Key: ratchet:{user_id}:{peer_user_id}
Value: JSON serialized ratchet state
TTL: 300 seconds (auto-expire as safety net)
```

A ogni avanzamento del ratchet, scrivi su Redis. Ogni 10 messaggi OPPURE ogni 30 secondi, esegui il flush su PostgreSQL. Alla disconnessione, esegui il flush immediatamente.

---

### Load testing dei WebSocket

Usa `locust` o `k6` per fare il load test del livello WebSocket. Determina quante connessioni WebSocket concorrenti può gestire una singola istanza del server.

**Cosa misurare:**
1. Massimo numero di connessioni WebSocket concorrenti prima dell'esaurimento della memoria
2. Throughput dei messaggi: messaggi al secondo con 100, 500, 1000, 5000 connessioni concorrenti
3. Latenza: latenza di consegna dei messaggi p50, p95, p99 a vari numeri di connessioni
4. Identifica il collo di bottiglia: è la CPU (dalle operazioni di cifratura)? La memoria (dal connection pool in `websocket_manager.py`)? L'I/O (dalle scritture su SurrealDB)? La contesa del GIL di Python?

**Obiettivo:** Documenta il punto di rottura e proponi una strategia di scaling orizzontale.

**Considerazioni sullo scaling:**

L'architettura attuale ha una singola istanza di `ConnectionManager` in `websocket_manager.py` che tiene in memoria tutte le connessioni WebSocket (`self.active_connections: dict[UUID, list[WebSocket]]`). Questo non scala orizzontalmente perché le connessioni sul Server A sono invisibili al Server B.

Per scalare su più istanze del server:
1. Usa Redis Pub/Sub per il routing dei messaggi tra istanze. Quando il Server A deve inviare un messaggio a un utente connesso al Server B, pubblica su un canale Redis. Il Server B si iscrive e inoltra al WebSocket locale.
2. Memorizza i metadati di connessione in Redis: `{user_id: [server_a_instance_id, server_b_instance_id]}`
3. Usa sticky session o consistent hashing per instradare, quando possibile, le richieste di upgrade WebSocket allo stesso server (riduce il traffico tra istanze)
4. Valuta di usare un gateway WebSocket dedicato (come Centrifugo o Soketi) davanti all'applicazione FastAPI

**Schema dello script di load test (k6):**
```javascript
import ws from 'k6/ws';
export default function {
 // The server expects a session cookie; load tests should sign in first
 // and pass the cookie via params.headers.Cookie.
 const url = 'ws://localhost:8000/ws';
 const params = { headers: { Cookie: 'chat_session=<session-token>' } };
 const res = ws.connect(url, params, function (socket) {
 socket.on('open', => {
 socket.send(JSON.stringify({
 type: 'encrypted_message',
 recipient_id: '...',
 room_id: '...',
 ciphertext: '...',
 nonce: '...',
 header: '...'
 }));
 });
 socket.on('message', (msg) => {});
 socket.setTimeout( => socket.close, 30000);
 });
}
```

Esegui con: `k6 run --vus 100 --duration 60s load_test.js`

---

## Sfide di sicurezza

---

### Implementare la Certificate Transparency per le identity key

Costruisci un sistema in cui le modifiche alle identity key vengono registrate in un log append-only verificabile, simile alla Certificate Transparency per i certificati TLS. Questo impedisce al server di sostituire silenziosamente le identity key per eseguire un attacco man-in-the-middle.

Attualmente, il server in `backend/app/services/prekey_service.py` serve i prekey bundle. Un server compromesso potrebbe servire una identity key falsa (la propria), eseguire X3DH con entrambe le parti e inoltrare i messaggi decifrati. Gli utenti non se ne accorgerebbero.

Un key transparency log rende la cosa rilevabile: ogni identity key viene registrata e i client possono fare audit del log per verificare che la propria chiave non sia stata sostituita.

**Bozza di implementazione:**

1. Crea un Merkle tree in cui ogni foglia è `(user_id, identity_key_hash, timestamp)`.
2. Quando un utente si registra o cambia la propria identity key, aggiungi una nuova foglia all'albero.
3. Pubblica periodicamente (ogni ora) la radice di Merkle su un log pubblico.
4. I client scaricano la radice di Merkle e verificano che la propria chiave sia inclusa (Merkle inclusion proof).
5. Se la chiave di un client è stata cambiata a sua insaputa (il server l'ha sostituita), la prova di Merkle fallirà OPPURE il client vedrà nell'albero una chiave inattesa.

**La parte difficile:** La consistenza. Il server deve dimostrare che l'albero è append-only (nessuna voce è stata rimossa o modificata). Questo richiede signed tree head e consistency proof tra stati consecutivi dell'albero.

**Ricerca:** Il progetto Key Transparency di Google, CONIKS (Key Verification for Messaging) e il deployment di key transparency di Signal (annunciato nel 2023).

---

### Fare l'audit dell'implementazione crittografica

Esegui un audit di sicurezza manuale delle implementazioni di X3DH e Double Ratchet. Non si tratta di scrivere codice. Si tratta di leggere il codice in modo critico.

**Checklist di audit:**

1. **Uso dei parametri HKDF** (`frontend/src/crypto/x3dh.ts`): Il salt è corretto? La stringa info è distinta tra usi diversi? Le lunghezze di output sono appropriate?

2. **Generazione dei nonce** (`frontend/src/crypto/double-ratchet.ts`): `os.urandom(AES_GCM_NONCE_SIZE)` viene chiamato a ogni cifratura? C'è qualche rischio di riutilizzo del nonce?

3. **Uso dell'API WebCrypto** (`frontend/src/crypto/primitives.ts`): Il parametro `iv` è sempre nuovo? I legami di `additionalData` sono corretti? Gli usi delle chiavi sono limitati correttamente?

4. **Timing side channel**: Guarda `frontend/src/crypto/primitives.ts` (`constantTimeEqual`). È davvero a tempo costante in JavaScript? (Spoiler: l'`if (a.length !== b.length) return false` è un ritorno anticipato che rivela informazioni sulla lunghezza. È un problema nel contesto?)

5. **Generazione di numeri casuali**: Verifica che tutta la casualità provenga da `os.urandom` (Python) o `crypto.getRandomValues` (JavaScript). Cerca qualsiasi uso di `random` o `Math.random`.

6. **Key material in memoria**: Le chiavi private vengono mai scritte nei log? Cerca le chiamate a `logger.debug` e `logger.info` che potrebbero stampare key material. Controlla che le chiavi codificate in base64url non siano incluse per sbaglio nei messaggi di errore.

Scrivi un report che documenti i tuoi finding.

---

### Implementare il Sealed Sender

Costruisci un sistema in cui il server non sa chi ha inviato un messaggio, ma solo a chi è destinato. Signal lo ha implementato come "Sealed Sender."

Attualmente, l'endpoint WebSocket in `websocket.py` riceve messaggi da utenti autenticati. Il server conosce `sender_id` (dalla connessione WebSocket) e `recipient_id` (dal payload del messaggio). Memorizza entrambi in SurrealDB (`backend/app/services/message_service.py`).

Con il Sealed Sender, l'identità del mittente è cifrata dentro il payload cifrato E2E. Il server vede solo il destinatario (necessario per il routing). Il mittente viene rivelato solo dopo che il destinatario decifra.

**Architettura:**
1. Livello esterno: cifra la busta di routing (che include il vero messaggio cifrato) con la chiave pubblica del SERVER. Il server la decifra per conoscere il destinatario, ma il campo del mittente è assente o cifrato.
2. Livello interno: il messaggio cifrato E2E così come funziona oggi, con l'identità del mittente dentro il payload cifrato.

Questo richiede che il server abbia una propria coppia di chiavi (separata da quella di qualsiasi utente) e che i client conoscano la chiave pubblica del server.

**Passi di implementazione:**

1. Genera una coppia di chiavi X25519 del server all'avvio. Pubblica la chiave pubblica del server su un endpoint ben noto: `GET /api/.well-known/server-key`.

2. Modifica il flusso di invio dei messaggi del client:
 - Costruisci il payload interno: il normale messaggio cifrato E2E (ciphertext, nonce, header) PIÙ `sender_id` in chiaro
 - Cifra il payload interno con il ratchet del destinatario previsto (come avviene attualmente)
 - Costruisci la busta esterna: `{recipient_id: "bob-uuid", sealed_payload: "<encrypted inner>"}`
 - Cifra la busta esterna con la chiave pubblica del server (semplice X25519 + AES-GCM)
 - Invia solo il ciphertext esterno sul WebSocket

3. Modifica il routing del server:
 - Decifra la busta esterna usando la chiave privata del server
 - Leggi `recipient_id` dalla busta decifrata
 - Inoltra `sealed_payload` al destinatario
 - Il server non vede mai `sender_id`

4. Modifica la decifratura del destinatario:
 - Decifra il payload sealed usando il ratchet E2E
 - Estrai `sender_id` dal contenuto decifrato
 - Mostra il messaggio con l'attribuzione corretta del mittente

**Limitazione:** Il server conosce comunque il destinatario (gli serve per il routing). E conosce l'indirizzo IP del mittente e la sua connessione WebSocket. Un vero anonimato del mittente richiede misure aggiuntive come Tor o le mix network. Il Sealed Sender nasconde l'identità del mittente ai log dei messaggi del server, non all'osservazione a livello di rete.

**Come testare:**
- Invia un messaggio tra due utenti con il Sealed Sender abilitato
- Controlla i record dei messaggi in SurrealDB sul server e verifica che `sender_id` sia assente o cifrato
- Verifica che il destinatario riesca ancora a identificare correttamente il mittente dopo la decifratura

---

## Sfide di contribuzione

---

### Scrivere test property-based

Usa Hypothesis (Python) per scrivere test property-based per le implementazioni crittografiche.

**Proprietà da verificare:**

1. **Simmetria di X3DH:** Per qualsiasi coppia di identity key valide e qualsiasi prekey bundle valido, `perform_x3dh_sender` e `perform_x3dh_receiver` producono lo stesso `shared_key`. Riferimento: `frontend/src/crypto/x3dh.ts` e `frontend/src/crypto/x3dh.ts`.

2. **Round-trip del Double Ratchet:** Per qualsiasi sequenza di messaggi in plaintext in qualsiasi ordine, cifrare con `encrypt_message` e decifrare con `decrypt_message` restituisce sempre il plaintext originale. Riferimento: `frontend/src/crypto/double-ratchet.ts` e `frontend/src/crypto/double-ratchet.ts`.

3. **Round-trip di AES-GCM:** Per qualsiasi plaintext e qualsiasi chiave valida, `aesGcmEncrypt` seguito da `aesGcmDecrypt` restituisce il plaintext originale. Riferimento: `frontend/src/crypto/primitives.ts`.

4. **Nessun riutilizzo del nonce:** Per N cifrature con la stessa chiave, tutti gli N nonce sono distinti. (Questo dovrebbe valere con probabilità schiacciante per nonce casuali da 12 byte, ma verificalo empiricamente per N=10000.)

5. **Forward secrecy:** Dopo aver fatto avanzare il ratchet, le vecchie message key non possono essere derivate dal nuovo stato. Genera uno stato del ratchet, cifra un messaggio, fai avanzare il ratchet di 100 passi, poi verifica che la message key del passo 0 non possa essere ricalcolata dallo stato al passo 100.

6. **Decifratura fuori ordine:** Per qualsiasi permutazione di N messaggi, decifrarli in quell'ordine permutato produce lo stesso insieme di plaintext della decifratura nell'ordine originale. Questo testa la logica delle skipped message key in `frontend/src/crypto/double-ratchet.ts`.

**Esempio di strategia Hypothesis:**

```python
from hypothesis import given, strategies as st

@given(
 message_count=st.integers(min_value=1, max_value=50),
 delivery_order=st.permutations(range(50)) # random permutation
)
def test_out_of_order_delivery(message_count, delivery_order):
 """All messages decrypt correctly regardless of delivery order."""
 # Generate N messages
 # Encrypt them in order (advancing the sending ratchet)
 # Decrypt them in the shuffled order
 # Assert all plaintexts match the originals
```

**Dove mettere i test:** I test di proprietà crittografici appartengono alla suite Vitest del frontend accanto a `x3dh.test.ts` e `double-ratchet.test.ts`. Aggiungi un nuovo file come `frontend/src/crypto/properties.test.ts` per i controlli randomizzati di round-trip e di manomissione. Riserva `backend/tests/` ai test di integrazione del backend (autenticazione, archiviazione dei prekey, applicazione delle sessioni).

---

### Aggiungere la verifica formale

Usa un symbolic model checker (ProVerif o Tamarin Prover) per verificare formalmente che l'implementazione X3DH + Double Ratchet soddisfi:

1. **Segretezza:** Un attaccante che controlla la rete non può apprendere il plaintext dei messaggi
2. **Autenticazione:** Un messaggio che si decifra con successo è stato inviato dal mittente dichiarato
3. **Forward secrecy:** Compromettere le chiavi a lungo termine non rivela le chiavi di sessione passate
4. **Post-compromise security:** Dopo che un attaccante perde l'accesso alle chiavi di sessione, i messaggi futuri sono di nuovo sicuri (questa è la proprietà del DH ratchet)

Questa è una sfida di livello ricerca. Inizia modellando solo X3DH prima di aggiungere il Double Ratchet.

**Per iniziare con ProVerif:**

ProVerif usa un process calculus per modellare i protocolli crittografici. Descrivi il protocollo come un insieme di processi (Alice, Bob, Attacker) e le primitive crittografiche che usano. ProVerif poi esplora tutti i possibili interleaving e le strategie dell'attaccante per trovare attacchi.

Un modello minimo di X3DH in ProVerif dovrebbe:
1. Definire i tipi per chiavi, nonce e messaggi
2. Modellare le operazioni DH come `fun dh(skey, pkey): key` con l'equazione `dh(skA, pk(skB)) = dh(skB, pk(skA))`
3. Modellare HKDF come un random oracle: `fun kdf(key, key, key): key`
4. Definire il processo di Alice: genera la chiave effimera, calcola DH1/DH2/DH3, deriva la chiave condivisa, cifra il messaggio
5. Definire il processo di Bob: riceve la chiave effimera, calcola DH1/DH2/DH3, deriva la chiave condivisa, decifra il messaggio
6. Query: `query attacker(message)` -- l'attaccante può apprendere il plaintext?

**Risorse:**
- Manuale di ProVerif: https://bblanche.gitlabpages.inria.fr/proverif/
- Tamarin Prover: https://tamarin-prover.com/
- "A Formal Security Analysis of the Signal Messaging Protocol" di Cohn-Gordon et al. (2017) -- questo paper ha verificato formalmente Signal usando Tamarin

---

## Tracker di completamento delle sfide

Usalo per tracciare i tuoi progressi. Spunta ogni sfida quando la completi.

- [ ] Facile 1: Conferme di lettura
- [ ] Facile 2: Indicatori di digitazione
- [ ] Facile 3: Timestamp dei messaggi e ordinamento
- [ ] Facile 4: Stato online dell'utente
- [ ] Intermedia 5: Ricerca nei messaggi (cifrati)
- [ ] Intermedia 6: Condivisione di file (cifrata)
- [ ] Intermedia 7: Sincronizzazione multi-dispositivo
- [ ] Avanzata 8: Cifratura di gruppo (Sender Keys)
- [ ] Avanzata 9: Scambio di chiavi post-quantistico
- [ ] Avanzata 10: Messaggi che scompaiono
- [ ] Esperta 11: Autenticazione deniabile
- [ ] Esperta 12: Backup sicuro e recupero
