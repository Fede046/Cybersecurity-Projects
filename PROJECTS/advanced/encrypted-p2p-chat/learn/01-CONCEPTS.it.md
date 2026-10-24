# Concetti di sicurezza

Questo documento tratta le fondamenta crittografiche e di autenticazione
dell'applicazione di chat P2P cifrata. Ogni concetto qui presente
corrisponde direttamente a codice nel repository. Se riesci a leggere e
capire questo documento, saprai esattamente cosa succede a un messaggio
dal momento in cui un utente lo digita al momento in cui appare sullo
schermo del destinatario, e perché ogni passaggio esiste.


## Crittografia end-to-end

### Cos'è

La crittografia end-to-end (E2EE) è un modello di comunicazione in cui
solo i due endpoint di una conversazione possono leggere i messaggi
scambiati tra loro. La proprietà che la definisce è che il server agisce
da relay cieco: memorizza e inoltra blob cifrati, ma non possiede alcun
materiale di chiave che gli permetterebbe di decifrarli. La cifratura
avviene sul dispositivo del mittente, la decifratura avviene sul
dispositivo del destinatario, e in nessun punto tra quei due dispositivi
esiste il testo in chiaro.

Questo è diverso dalla cifratura del trasporto (TLS/HTTPS), che cifra il
collegamento tra il tuo dispositivo e il server, e tra il server e
l'altro dispositivo, ma permette al server stesso di leggere tutto. Con
la cifratura del trasporto, il server è un intermediario fidato. Con
l'E2EE, il server è un corriere non fidato.

La distinzione conta perché la fiducia è una vulnerabilità. Se il server
può leggere i tuoi messaggi, allora chiunque comprometta il server può
leggere anch'esso i tuoi messaggi. Questo include gli attaccanti che
violano il server, i dipendenti infedeli e le agenzie governative con
l'autorità legale di obbligare l'operatore del server a fornire i dati.

### Perché è importante

La storia della sicurezza della messaggistica è una storia di server
compromessi, costretti o colti a mentire sul proprio accesso ai dati
degli utenti.

**2013: Edward Snowden e il programma PRISM della NSA.** Snowden fece
trapelare documenti classificati della NSA che mostravano come grandi
aziende tecnologiche tra cui Google, Microsoft, Yahoo, Facebook, Apple e
altre avessero fornito alla NSA accesso diretto lato server alle
comunicazioni degli utenti nell'ambito del programma di sorveglianza
PRISM. Poiché questi servizi usavano la cifratura del trasporto anziché
la cifratura E2E, le aziende detenevano le chiavi di decifratura sui
propri server e potevano soddisfare le richieste governative (o essere
costrette a farlo tramite ordinanze del tribunale FISA). Il server aveva
accesso, quindi il governo ottenne accesso. Questo fu il singolo
catalizzatore più importante per l'adozione della crittografia
end-to-end nei prodotti consumer.

**2019: L'hack WhatsApp di Jeff Bezos.** Nel gennaio 2020, l'analisi
forense di FTI Consulting concluse che l'iPhone del CEO di Amazon Jeff
Bezos era stato compromesso tramite un file video malevolo inviato su
WhatsApp dall'account del principe ereditario saudita Mohammed bin
Salman. Sebbene la cifratura E2E del Signal Protocol di WhatsApp avesse
protetto il contenuto del messaggio in transito, l'attacco prese di mira
il dispositivo endpoint stesso con uno spyware (attribuito a Pegasus di
NSO Group). Questo caso è istruttivo perché mostra sia la forza sia il
limite della crittografia E2E: protegge i messaggi tra i dispositivi, ma
se il dispositivo stesso è compromesso, l'attaccante legge il testo in
chiaro dopo la decifratura. La crittografia E2E difende dalla
compromissione della rete e del server, non dalla compromissione
dell'endpoint.

**2020: Lo scandalo della cifratura E2E di Zoom.** Zoom Video
Communications pubblicizzava il proprio prodotto come dotato di
"end-to-end encryption" per le videochiamate. Le indagini di The
Intercept (marzo 2020) e del Citizen Lab dell'Università di Toronto
rivelarono che Zoom deteneva le chiavi di cifratura sui propri server.
Le chiamate erano cifrate con AES-128 in modalità ECB (una modalità di
cifratura debole che fa trapelare pattern) tra il client e il server di
Zoom, ma l'infrastruttura di Zoom poteva decifrare tutto il contenuto
delle chiamate. L'accordo con la FTC del novembre 2020 impose a Zoom di
implementare misure di sicurezza reali e le vietò di travisare le
proprie pratiche di cifratura. Questo caso dimostra che la parola
"cifrato" senza il qualificatore "end-to-end" non significa nulla ai
fini della privacy.

**2021: La controversia sul logging di ProtonMail.** ProtonMail, un
provider svizzero di email cifrata che fa della privacy il proprio
argomento di marketing, fu obbligato da un'ordinanza di un tribunale
svizzero (richiesta tramite Europol per conto delle autorità francesi) a
registrare l'indirizzo IP e il fingerprint del browser di un attivista
climatico francese. Sebbene la cifratura E2E di ProtonMail significasse
che non poteva leggere il contenuto delle email, fu obbligato a
raccogliere metadati. Il server non poteva leggere i messaggi, ma poteva
identificare chi li inviava. Questo caso mostra che la crittografia E2E
risolve il problema del contenuto ma non risolve automaticamente il
problema dei metadati.

Lo schema è costante: qualsiasi server che PUÒ leggere i tuoi messaggi
prima o poi SARÀ costretto a farlo, che sia da un mandato governativo,
da attaccanti che violano l'infrastruttura o da insider che abusano del
proprio accesso. L'unica difesa affidabile è rendere architetturalmente
impossibile, in partenza, che il server legga i dati.

### Come funziona (in questo progetto)

```
Alice's Device Server Bob's Device
+----------------+ +----------------+ +----------------+
| Plaintext | | | | |
| "Hello Bob" | | Encrypted | | Plaintext |
| | | ---> | blob only | ---> | "Hello Bob" |
| v | | No keys | | ^ |
| AES-256-GCM | | No access | | AES-256-GCM |
| encrypt | | No decrypt | | decrypt |
+----------------+ +----------------+ +----------------+

Keys derived from Server stores Keys derived from
Double Ratchet on ciphertext, nonce, Double Ratchet on
Alice's device and header verbatim Bob's device
```

Lato server, `backend/app/services/message_service.py` implementa il
metodo `store_encrypted_message`. Guarda la docstring :
`"Stores client-encrypted message in SurrealDB (pass-through, no server encryption)"`.
La funzione riceve `ciphertext`, `nonce` e `header` come stringhe dal
client, e li memorizza direttamente in SurrealDB senza alcun passaggio
di decifratura o ricifratura. Il server è un passthrough. Scrive ciò che
riceve e legge ciò che ha scritto. In nessun momento importa una chiave
crittografica o chiama una funzione di decifratura su questi parametri
del messaggio.

Lato client, `frontend/src/crypto/crypto-service.ts` implementa il
metodo `encrypt`. Il client chiama `encryptMessage` da
`double-ratchet.ts` , riceve ciphertext e nonce, poi li invia al server
come stringhe codificate in base64. Il testo in chiaro non lascia mai il
processo del client. La vera e propria cifratura simmetrica avviene in
`frontend/src/crypto/primitives.ts` usando l'implementazione AES-GCM
della WebCrypto API, che gira nel modulo crittografico nativo del
browser anziché in JavaScript. Questo significa che il testo in chiaro
non è nemmeno accessibile agli strumenti di debug JavaScript durante
l'operazione di cifratura stessa.


## Il Signal Protocol

### Cos'è

Il Signal Protocol è un protocollo crittografico di ratcheting
sviluppato originariamente da Trevor Perrin e Moxie Marlinspike presso
Open Whisper Systems (ora Signal Foundation). È stato progettato per
fornire crittografia end-to-end alla messaggistica istantanea con forti
proprietà di forward secrecy e post-compromise security.

Il protocollo fu distribuito per la prima volta nell'applicazione
TextSecure (il predecessore di Signal) e fu descritto formalmente in una
serie di specifiche tecniche pubblicate su signal.org/docs. È stato
analizzato in modo indipendente e verificato formalmente in articoli
accademici, tra cui "A Formal Security Analysis of the Signal Messaging
Protocol" di Cohn-Gordon et al. a IEEE EuroS&P 2017, che ha dimostrato
che il protocollo soddisfa le proprietà di sicurezza dichiarate sotto le
assunzioni crittografiche standard.

Il Signal Protocol è oggi distribuito su scala enorme. WhatsApp ha
completato il rollout della cifratura Signal Protocol a tutti gli utenti
nell'aprile 2016, coprendo all'epoca oltre 1 miliardo di utenti (oggi
oltre 2 miliardi). Google Messages ha adottato il Signal Protocol per la
messaggistica RCS. Facebook Messenger lo offriva come modalità opzionale
"Secret Conversations". Skype lo ha implementato come "Private
Conversations". Il design del protocollo è stato abbastanza influente da
essere di fatto lo standard di settore per la messaggistica sicura.

### I due componenti principali

Il Signal Protocol combina due meccanismi crittografici distinti,
ciascuno dei quali risolve un problema diverso:

```
Signal Protocol = X3DH (initial handshake) + Double Ratchet (ongoing encryption)

X3DH: "How do Alice and Bob agree on a shared secret
 when Bob might be offline?"

 Solves the ASYNCHRONOUS key agreement problem.
 Bob uploads prekey bundles to the server ahead of time.
 Alice can start a conversation using Bob's prekeys
 without Bob being online.

Double Ratchet: "Once they share a secret, how do they
 encrypt each message with a UNIQUE key
 that can never be recovered?"

 Solves the FORWARD SECRECY and POST-COMPROMISE
 SECURITY problem. Every message gets its own
 ephemeral encryption key. Compromising one key
 does not reveal past or future messages.
```

X3DH viene eseguito una volta all'inizio di una conversazione. Il Double
Ratchet viene eseguito continuamente per ogni messaggio successivo.
L'output di X3DH (un segreto condiviso) è l'input del Double Ratchet (la
root key iniziale).


## X3DH (Extended Triple Diffie-Hellman)

### Cos'è

Lo scambio di chiavi Diffie-Hellman standard richiede che entrambe le
parti siano online nello stesso momento. Alice genera un valore, lo
invia a Bob, Bob genera un valore, lo rimanda ad Alice, ed entrambi
calcolano il segreto condiviso. Funziona bene per una telefonata o una
connessione live, ma non funziona per la messaggistica asincrona. Se
Alice vuole inviare un messaggio a Bob alle 3 di notte mentre il
telefono di Bob è spento, il DH standard non può procedere perché Bob
non è lì per generare e inviare la sua metà.

X3DH risolve il problema facendo pre-generare a Bob un insieme di
materiale di chiave e facendoglielo caricare sul server prima di andare
offline. Questo materiale pre-generato si chiama "prekey bundle". Quando
Alice vuole iniziare una conversazione, scarica dal server il prekey
bundle di Bob e lo usa per calcolare un segreto condiviso senza la
partecipazione di Bob. Quando Bob torna online, può calcolare lo stesso
segreto condiviso a partire dalle informazioni che Alice gli invia,
perché la matematica di Diffie-Hellman permette a entrambe le parti di
arrivare in modo indipendente allo stesso risultato.

L'"Extended Triple" in X3DH si riferisce al fatto che il protocollo
esegue tre o quattro operazioni Diffie-Hellman separate (non una sola)
per ottenere proprietà di sicurezza più forti di quelle che fornirebbe
un singolo scambio DH.

### Tipi di chiave

X3DH usa quattro tipi di chiavi. Ciascuno ha una durata e uno scopo
diversi. Capire perché esistono quattro tipi (anziché uno solo) è
essenziale per capire il modello di sicurezza.

**Identity Key (IK) -- A lungo termine, generata una volta per utente**

La identity key è una coppia di chiavi permanente che rappresenta
l'identità crittografica dell'utente. Viene generata una volta e
conservata per tutta la vita dell'account. Questo progetto genera due
coppie di identity key per utente:

- Una coppia di chiavi X25519 per le operazioni Diffie-Hellman
 (rif: `frontend/src/crypto/x3dh.ts`)
- Una coppia di chiavi Ed25519 per le firme digitali
 (rif: `frontend/src/crypto/x3dh.ts`)

La coppia di chiavi X25519 partecipa direttamente ai calcoli DH. La
coppia di chiavi Ed25519 firma la signed prekey per dimostrare che
appartiene alla stessa identità. Sono curve separate perché X25519 è una
funzione Diffie-Hellman (calcola segreti condivisi) mentre Ed25519 è uno
schema di firma (firma e verifica dati). Non sono intercambiabili. Le
chiavi private sono memorizzate nel database (lato server) o in
IndexedDB (lato client), e non cambiano mai a meno che l'utente non
reimposti esplicitamente la propria identità.

**Signed Prekey (SPK) -- A medio termine, ruotata ogni 48 ore**

La signed prekey è una coppia di chiavi X25519 che ruota periodicamente.
In questo progetto, la rotazione avviene ogni 48 ore come configurato in
`config.py` (`SIGNED_PREKEY_ROTATION_HOURS=48`).

Quando viene generata una nuova SPK (`frontend/src/crypto/x3dh.ts`), la
chiave pubblica viene firmata usando la identity key Ed25519 :
`signature = identity_private.sign(spk_public_bytes)`. Questa firma
dimostra che la SPK è stata creata dal detentore della identity key.
Quando Alice scarica il prekey bundle di Bob, verifica questa firma
prima di usare la SPK, il che impedisce a un man-in-the-middle di
sostituirla con la propria SPK.

Il periodo di rotazione è un compromesso. Una rotazione più breve
fornisce una forward secrecy migliore (perché le vecchie SPK vengono
eliminate, e qualsiasi segreto DH calcolato con esse diventa
irrecuperabile). Una rotazione più lunga significa meno operazioni di
gestione delle chiavi e meno complessità. La finestra di 48 ore usata
qui è coerente con la raccomandazione di Signal.

Le vecchie signed prekey vengono conservate (con `is_active = False`)
così che i messaggi in volo cifrati con la SPK precedente possano ancora
completare il loro X3DH iniziale sul ricevente. In questo codebase non
esiste un reaper automatico; l'eliminazione delle SPK inattive molto
vecchie è un'estensione suggerita in `04-CHALLENGES.md`.

**One-Time Prekey (OPK) -- Monouso, consumata ed eliminata**

Le one-time prekey sono coppie di chiavi X25519 che vengono usate
esattamente una volta e poi scartate. Vengono generate in batch
(rif: `frontend/src/crypto/x3dh.ts` per la generazione,
`backend/app/services/prekey_service.py` per il rifornimento in batch) e
caricate sul server.

Quando Alice avvia una conversazione con Bob, il server dà ad Alice una
delle OPK non usate di Bob e la marca come consumata
(`backend/app/services/prekey_service.py`). Questa OPK partecipa alla
quarta operazione DH (DH4) dell'handshake X3DH. Poiché la OPK viene
usata una sola volta e poi eliminata, fornisce un ulteriore livello di
forward secrecy specificamente per il messaggio iniziale di una
conversazione.

Se a Bob non restano OPK non usate (sono state tutte consumate da altri
utenti che hanno avviato conversazioni), X3DH ripiega su tre operazioni
DH invece di quattro. Il protocollo funziona comunque, ma il messaggio
iniziale ha una forward secrecy leggermente più debole perché la quarta
operazione DH viene saltata. Il sistema genera inizialmente 100 OPK
(`config.py`, `DEFAULT_ONE_TIME_PREKEY_COUNT=100`) e le rifornisce
quando la scorta scende sotto la metà.

**Ephemeral Key (EK) -- Generata per sessione, mai memorizzata**

La ephemeral key è una nuova coppia di chiavi X25519 generata da Alice
(la mittente) nel momento in cui avvia una conversazione. Viene usata in
DH2, DH3 e DH4 dell'handshake X3DH. Non viene mai memorizzata su disco;
esiste solo in memoria per la durata del calcolo dello scambio di
chiavi.

Nel codice, viene generata in `frontend/src/crypto/x3dh.ts`:

```python
alice_ek_private = X25519PrivateKey.generate
alice_ek_public = alice_ek_private.public_key
```

Dopo che il segreto condiviso è stato calcolato, Alice invia la chiave
pubblica effimera a Bob (così Bob può eseguire le stesse operazioni DH
dalla sua parte), e la chiave privata effimera viene scartata. Poiché la
componente privata non viene mai resa persistente, anche se il
dispositivo di Alice venisse compromesso in seguito, l'attaccante non
potrebbe recuperare la chiave privata effimera e quindi non potrebbe
ricalcolare il segreto condiviso iniziale.

### La matematica

X3DH esegue quattro operazioni Diffie-Hellman tra diverse combinazioni
di chiavi. Ogni operazione produce un segreto condiviso da 32 byte. I
quattro segreti vengono concatenati e passati a HKDF per produrre la
chiave condivisa finale.

```
Alice (sender) has: Bob (receiver) has:
 IK_A (identity private key) IK_B (identity public key)
 EK_A (ephemeral, just generated) SPK_B (signed prekey public)
 OPK_B (one-time prekey public)

DH Operations (each produces 32 bytes):

 DH1 = X25519(IK_A_private, SPK_B_public)
 Alice's identity x Bob's signed prekey

 DH2 = X25519(EK_A_private, IK_B_public)
 Alice's ephemeral x Bob's identity

 DH3 = X25519(EK_A_private, SPK_B_public)
 Alice's ephemeral x Bob's signed prekey

 DH4 = X25519(EK_A_private, OPK_B_public) [optional]
 Alice's ephemeral x Bob's one-time prekey

Key Material Derivation:

 input = 0xFF * 32 || DH1 || DH2 || DH3 || DH4
 salt = 0x00 * 32
 info = "X3DH"
 SK = HKDF-SHA256(salt, input, info, length=32)
```

Nel codebase, il lato mittente si trova in `frontend/src/crypto/x3dh.ts`:

- Riga 241: `dh1 = alice_ik_private.exchange(bob_spk_public)` -- DH1
- Riga 242: `dh2 = alice_ek_private.exchange(bob_ik_public)` -- DH2
- Riga 243: `dh3 = alice_ek_private.exchange(bob_spk_public)` -- DH3
- Riga 251: `dh4 = alice_ek_private.exchange(bob_opk_public)` -- DH4 (se la OPK è disponibile)
- Riga 252: `key_material = dh1 + dh2 + dh3 + dh4` -- concatenazione
- Righe 257-264: derivazione HKDF con prefisso `0xFF * 32` e stringa info `b'X3DH'`

Il lato ricevente in `frontend/src/crypto/x3dh.ts` esegue le stesse
operazioni ma con i ruoli invertiti. DH1 diventa
`bob_spk_private.exchange(alice_ik_public)` , perché Bob ha la chiave
privata SPK e la chiave pubblica IK di Alice. La proprietà di
Diffie-Hellman garantisce che `X25519(a_priv, B_pub)` produca lo stesso
risultato di `X25519(b_priv, A_pub)`, quindi entrambi i lati calcolano
segreti condivisi identici.

Il prefisso `0xFF * 32` anteposto (`f = b'\xff' * X25519_KEY_SIZE`) è un
padding fisso specificato dallo standard X3DH. Garantisce che l'input di
HKDF sia lungo almeno 32 byte anche nei casi limite e fornisce
separazione di dominio rispetto ad altri usi delle stesse chiavi.

### Perché quattro operazioni DH?

Ogni operazione DH fornisce una specifica proprietà di sicurezza. Se una
qualsiasi singola operazione venisse rimossa, diventerebbe possibile una
specifica classe di attacco.

**DH1: IK_A x SPK_B -- Autentica Alice presso Bob**

Questa operazione usa la identity key a lungo termine di Alice. Solo
Alice (la detentrice di IK_A_private) avrebbe potuto produrre questo
output DH con SPK_B_public. Quando Bob calcola lo stesso valore usando
SPK_B_private e IK_A_public, ha una prova crittografica che il messaggio
proviene da Alice. Senza DH1, chiunque conosca la SPK pubblica di Bob
potrebbe impersonare qualsiasi mittente.

**DH2: EK_A x IK_B -- Autentica Bob presso Alice**

Questa operazione usa la identity key a lungo termine di Bob. Solo Bob
(il detentore di IK_B_private) potrebbe riprodurre questo output DH.
Questo garantisce che Alice stia davvero parlando con Bob, e non con un
impostore che ha caricato sul server il proprio prekey bundle. Senza
DH2, un server malevolo potrebbe sostituire le chiavi di Bob con le
proprie ed eseguire un attacco man-in-the-middle.

**DH3: EK_A x SPK_B -- Fornisce forward secrecy**

Questa operazione usa la ephemeral key di Alice (generata da zero, mai
memorizzata) e la signed prekey di Bob (ruotata ogni 48 ore). Poiché
EK_A_private viene scartata immediatamente e SPK_B_private viene prima o
poi eliminata durante la rotazione, questo output DH diventa
irrecuperabile una volta che entrambe le chiavi non ci sono più. Anche se
le identity key sia di Alice sia di Bob venissero compromesse in
seguito, le chiavi di sessione passate derivate in parte da DH3 non
potrebbero essere ricalcolate. Questa è la garanzia centrale di forward
secrecy.

**DH4: EK_A x OPK_B -- Forward secrecy aggiuntiva per i messaggi iniziali**

Questa operazione usa la one-time prekey di Bob, che viene consumata ed
eliminata subito dopo l'uso. Fornisce forward secrecy specificamente per
il primo messaggio di una conversazione. Senza DH4, se un attaccante
compromettesse la SPK_private di Bob (che esiste per un massimo di 48
ore), potrebbe decifrare retroattivamente i messaggi iniziali inviati
durante quella finestra. DH4 garantisce che anche una SPK compromessa
non sia sufficiente, perché OPK_private è stata eliminata nel momento in
cui è stata usata.

DH4 impedisce anche i replay attack sull'handshake iniziale. Poiché la
OPK è monouso, un attaccante che registra il messaggio iniziale di Alice
non può riprodurlo in seguito: Bob ha già consumato la OPK, quindi il
server non fornirà di nuovo la stessa, e il lato di Bob non avrà la
chiave privata della OPK disponibile per un handshake riprodotto.

### Verifica del prekey bundle

Prima di eseguire qualsiasi operazione DH, Alice deve verificare che la
signed prekey di Bob appartenga davvero a Bob. Un server malevolo
potrebbe sostituirla con la propria SPK e intercettare le comunicazioni.

La verifica avviene in `frontend/src/crypto/x3dh.ts` all'interno di
`perform_x3dh_sender`:

```python
if not self.verify_signed_prekey(bob_bundle.signed_prekey,
 bob_bundle.signed_prekey_signature,
 bob_identity_public_ed25519):
 raise ValueError("Invalid signed prekey signature")
```

Il metodo `verify_signed_prekey` in `frontend/src/crypto/x3dh.ts` usa la
verifica della firma Ed25519. Decodifica i byte della chiave pubblica
SPK, i byte della firma e i byte della chiave pubblica di identità
Ed25519, poi chiama `identity_public.verify(signature_bytes, spk_public_bytes)` .
La verifica Ed25519 o riesce o solleva `InvalidSignature`. Se la
verifica fallisce, l'intero handshake X3DH viene interrotto.

Questa verifica è critica. Senza di essa, un attaccante lato server
potrebbe sostituire la SPK di Bob con una che controlla, eseguire le
operazioni DH usando la propria chiave privata e fare da proxy
trasparente ai messaggi tra Alice e Bob leggendo tutto. La firma Ed25519
lega la SPK alla identity key di Bob, rendendo rilevabile la
sostituzione.

Nota che questo funziona solo se Alice possiede la chiave pubblica di
identità autentica di Bob. In pratica, la verifica della identity key
viene fatta tramite i "safety number" o la "key verification" -- un
processo separato out-of-band in cui Alice e Bob confrontano di persona,
o tramite un canale secondario fidato, i fingerprint delle rispettive
identity key.


## Algoritmo Double Ratchet

### Cos'è

Il Double Ratchet è un algoritmo per gestire le chiavi di cifratura in
una conversazione in corso. È stato sviluppato da Trevor Perrin e Moxie
Marlinspike come parte del Signal Protocol, basandosi su lavori
precedenti del protocollo di messaggistica Off-the-Record (OTR).

Il Double Ratchet fornisce due proprietà di sicurezza critiche che vanno
oltre ciò che potrebbe fornire una chiave condivisa statica:

1. **Forward secrecy** -- Compromettere una chiave attuale non espone i
 messaggi passati. Anche se un attaccante ruba la chiave di cifratura
 attuale, non può derivare le chiavi precedenti e quindi non può
 decifrare i messaggi precedenti.

2. **Post-compromise security (recupero dopo un'intrusione)** -- Dopo
 una compromissione delle chiavi, i messaggi futuri tornano sicuri non
 appena avviene un nuovo passo del ratchet DH. Se un attaccante ottiene
 temporaneamente accesso al materiale di chiave, perde l'accesso alla
 conversazione non appena le chiavi avanzano tramite un nuovo scambio
 Diffie-Hellman.

Il nome "Double Ratchet" si riferisce al fatto che combina due
meccanismi di ratcheting: un **ratchet DH** (ratchet Diffie-Hellman) che
avanza quando cambia il turno della conversazione, e un **ratchet
simmetrico** (hash ratchet) che avanza a ogni singolo messaggio.

### Le tre catene

Il Double Ratchet mantiene tre catene KDF collegate: la root chain, la
sending chain e la receiving chain.

```
Root Chain (KDF_RK)
 |
 |-- [DH ratchet step] --> new root key + new chain key
 |
 +-- Sending Chain (KDF_CK)
 | |-- advance --> Message Key 1 --> Encrypt msg 1
 | |-- advance --> Message Key 2 --> Encrypt msg 2
 | +-- advance --> Message Key 3 --> Encrypt msg 3
 |
 +-- Receiving Chain (KDF_CK)
 |-- advance --> Message Key 1 --> Decrypt msg 1
 |-- advance --> Message Key 2 --> Decrypt msg 2
 +-- advance --> Message Key 3 --> Decrypt msg 3
```

Ogni catena è una sequenza di operazioni di derivazione delle chiavi. La
root chain produce nuove chain key di invio e di ricezione tramite il
ratchet DH. Le catene di invio e di ricezione producono le singole
message key tramite il ratchet simmetrico. Ogni message key viene usata
esattamente una volta e poi scartata.

### Operazioni delle catene KDF

**KDF_RK: derivazione della root key**

La root chain avanza durante un passo del ratchet DH. Prende la root key
attuale e un nuovo output DH (da un nuovo scambio di chiavi DH) e
produce una nuova root key e una nuova chain key.

Riferimento: `frontend/src/crypto/double-ratchet.ts`

```python
def _kdf_rk(self, root_key: bytes, dh_output: bytes) -> tuple[bytes, bytes]:
 hkdf = HKDF(
 algorithm = hashes.SHA256,
 length = HKDF_OUTPUT_SIZE * 2, # 64 bytes total
 salt = root_key, # current root key as salt
 info = b'',
 )
 output = hkdf.derive(dh_output)
 new_root_key = output[: HKDF_OUTPUT_SIZE] # first 32 bytes
 new_chain_key = output[HKDF_OUTPUT_SIZE :] # last 32 bytes
 return new_root_key, new_chain_key
```

HKDF-SHA256 è usato con la root key attuale come salt e l'output DH come
input key material. L'output è di 64 byte, diviso a metà: i primi 32
byte diventano la nuova root key, gli ultimi 32 byte diventano la nuova
chain key. Questa divisione garantisce che conoscere la chain key non
riveli la root key, mantenendo la separazione tra la root chain e le
catene dei messaggi.

**KDF_CK: derivazione della chain key**

Le catene simmetriche avanzano a ogni messaggio. Ogni passo prende la
chain key attuale e produce la chain key successiva e una message key.

Riferimento: `frontend/src/crypto/double-ratchet.ts`

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

Vengono eseguiti due calcoli HMAC-SHA256 separati usando la stessa chain
key ma con costanti diverse:

```
chain_key --+-- HMAC(chain_key, 0x01) --> next_chain_key (kept for future)
 |
 +-- HMAC(chain_key, 0x02) --> message_key (used once, discarded)
```

L'uso di costanti diverse (0x01 e 0x02) è essenziale. Se venisse usata
la stessa costante, la chain key e la message key sarebbero identiche,
il che significherebbe che scoprire la message key (magari tramite un
attacco chosen plaintext) rivelerebbe la chain key e permetterebbe di
derivare tutte le chiavi future. Usando input HMAC diversi, la message
key e la chain key successiva sono crittograficamente indipendenti:
conoscerne una non rivela l'altra.

La message key viene usata esattamente una volta per cifrare o decifrare
un singolo messaggio, poi scartata. La chain key successiva sostituisce
quella attuale e viene usata per derivare la message key successiva.
Questa catena unidirezionale è ciò che fornisce la forward secrecy
all'interno di una singola epoca del ratchet DH.

### Il passo del ratchet DH

Il ratchet simmetrico gestisce il caso semplice: messaggi sequenziali
dallo stesso mittente. Ma da solo non può fornire la post-compromise
security. Se un attaccante compromette una chain key, può derivare tutte
le message key future di quella catena. Il ratchet DH risolve questo
problema.

Un passo del ratchet DH avviene ogni volta che cambia la direzione della
conversazione. Quando Alice riceve da Bob un messaggio che include una
nuova chiave pubblica DH (una che non ha mai visto prima), esegue un
passo del ratchet DH: genera una nuova coppia di chiavi DH, esegue uno
scambio DH con la nuova chiave pubblica di Bob e usa l'output per
derivare nuove root key e chain key tramite KDF_RK.

```
Message flow and DH ratchet steps:

Alice sends msgs 1,2,3 using DH keypair A1:
 A1 --> msg1(mk1), msg2(mk2), msg3(mk3)
 [symmetric ratchet advances 3 times, same DH key]

Bob receives, generates new DH keypair B1, sends reply:
 DH ratchet: root_key' = KDF_RK(root_key, DH(B1_priv, A1_pub))
 B1 --> msg4(mk1'), msg5(mk2')
 [new chain, new keys, A1 compromise no longer helps]

Alice receives, generates new DH keypair A2, sends reply:
 DH ratchet: root_key'' = KDF_RK(root_key', DH(A2_priv, B1_pub))
 A2 --> msg6(mk1''), msg7(mk2'')
 [new chain again, B1 compromise no longer helps]
```

Ogni passo del ratchet DH introduce nuova entropia casuale (dalla coppia
di chiavi DH appena generata) nella catena di derivazione delle chiavi.
Questo significa che anche se un attaccante avesse compromesso tutto il
materiale di chiave precedente, il nuovo scambio DH produce un segreto
condiviso che non può prevedere, e tutte le chiavi successive tornano
sicure.

L'implementazione si estende in `frontend/src/crypto/double-ratchet.ts`:

- `_dh_ratchet_send` : chiamata quando il mittente deve far avanzare il
 ratchet. Genera una nuova coppia di chiavi DH , esegue il DH con la
 chiave pubblica del peer e deriva nuove root key e chain key di
 invio .

- `_dh_ratchet_receive` : chiamata quando un messaggio ricevuto contiene
 una nuova chiave pubblica DH. Aggiorna la chiave pubblica del peer ,
 esegue il DH con la chiave privata esistente per derivare una nuova
 chain key di ricezione , poi genera una nuova chiave privata alla riga
 200 ed esegue un altro DH per derivare una nuova chain key di invio
 alle righe 208-211. Questo doppio passo DH sul lato ricevente
 garantisce che vengano aggiornate sia la catena di ricezione sia
 quella di invio.

### Gestione dei messaggi fuori ordine

I messaggi su internet possono arrivare fuori ordine. Se Alice invia i
messaggi 1, 2, 3 e Bob riceve 1, 3 (il messaggio 2 è in ritardo), Bob
deve:

1. Elaborare normalmente il messaggio 1 (derivare mk1, decifrare)
2. Durante l'elaborazione del messaggio 3, riconoscere che il messaggio 2 è stato saltato
3. Derivare e mettere in cache mk2 (così da poterla usare in seguito quando arriva il messaggio 2)
4. Derivare mk3 e decifrare il messaggio 3

Il meccanismo delle skipped message key gestisce questa situazione.
Riferimento: `frontend/src/crypto/double-ratchet.ts`.

`_store_skipped_message_keys` viene chiamata quando il numero del
messaggio ricevuto è maggiore del numero di messaggio atteso. Itera
lungo il buco, derivando e mettendo in cache ciascuna message key
saltata:

```python
chain_key = state.receiving_chain_key
for msg_num in range(state.receiving_message_number, until_message_number):
 chain_key, message_key = self._kdf_ck(chain_key)
 state.skipped_message_keys[(dh_public_key, msg_num)] = message_key
state.receiving_chain_key = chain_key
```

Le chiavi saltate sono memorizzate in un dizionario indicizzato da
`(dh_public_key,
message_number)`. Questa chiave a tupla è necessaria perché i numeri dei
messaggi si azzerano a ogni passo del ratchet DH: il messaggio 0 sotto la
chiave DH A1 è diverso dal messaggio 0 sotto la chiave DH A2.

`_try_skipped_message_key` verifica se un messaggio ricevuto corrisponde
a una chiave saltata messa in cache in precedenza. In caso affermativo,
la chiave in cache viene usata per la decifratura e poi rimossa dalla
cache (viene consumata da `dict.pop` ).

Dei limiti di sicurezza impediscono gli abusi. Un attaccante che invia
messaggi con numeri di messaggio enormi potrebbe costringere il ratchet
a derivare e memorizzare milioni di chiavi, esaurendo la memoria.
Vengono applicati due limiti:

- `MAX_SKIP_MESSAGE_KEYS = 1000` (`config.py`): non possono essere
 saltate più di 1000 message key in un singolo buco. Se arriva un
 messaggio che dichiara di essere il numero 5000 quando ci aspettiamo il
 messaggio 0, la decifratura viene rifiutata .

- `MAX_CACHED_MESSAGE_KEYS = 2000` (`config.py`): il numero totale di
 chiavi saltate in cache su tutte le epoche del ratchet. Se la cache è
 piena, le chiavi più vecchie vengono rimosse tramite
 `_evict_oldest_skipped_keys`.

### Dimostrazione della forward secrecy

Ecco una spiegazione passo passo del perché compromettere il materiale
di chiave al tempo T non espone i messaggi precedenti a T.

Supponi che al messaggio N un attaccante rubi la chain_key_N attuale.

```
What the attacker CAN compute (forward direction):
 chain_key_N -----> HMAC(chain_key_N, 0x01) = chain_key_N+1
 chain_key_N+1 ---> HMAC(chain_key_N+1, 0x01) = chain_key_N+2
 ... and so on for all future chain keys

What the attacker CANNOT compute (backward direction):
 chain_key_N <-/-- chain_key_N-1

 Why? Because HMAC is a one-way function.

 chain_key_N = HMAC(chain_key_N-1, 0x01)

 Given chain_key_N, you cannot solve for chain_key_N-1.
 This would require inverting HMAC-SHA256, which is
 computationally infeasible (preimage resistance).

Therefore:
 message_key_N-1 = HMAC(chain_key_N-1, 0x02) <-- UNREACHABLE
 message_key_N-2 = HMAC(chain_key_N-2, 0x02) <-- UNREACHABLE
 message_key_1 = HMAC(chain_key_1, 0x02) <-- UNREACHABLE
```

L'attaccante può decifrare i messaggi N+1, N+2, N+3 e così via (finché
il passo successivo del ratchet DH non introduce nuova entropia). Ma non
può decifrare alcun messaggio precedente a N. Tutte le message key
passate sono derivate da chain key che sono computazionalmente
inaccessibili avendo solo chain_key_N.

Ora considera cosa succede al passo successivo del ratchet DH. Bob invia
un messaggio con una nuova chiave pubblica DH B2. Alice genera una nuova
coppia di chiavi A3 ed esegue DH(A3_priv, B2_pub). Questo produce un
nuovo output DH che l'attaccante non può prevedere (perché non conosce
A3_priv, che è appena stata generata da dati casuali sicuri). La nuova
root key e la nuova chain key vengono derivate da questo nuovo output DH
tramite KDF_RK. La conoscenza della vecchia chain key da parte
dell'attaccante diventa inutile. Questa è la post-compromise security:
il sistema si autoripara.


## Cifratura AES-256-GCM

### Cos'è

AES-256-GCM è il cipher simmetrico usato per cifrare ogni singolo
messaggio. È un cipher AEAD (Authenticated Encryption with Associated
Data), il che significa che fornisce sia confidenzialità (nessuno può
leggere il messaggio senza la chiave) sia integrità (nessuno può
modificare il messaggio senza essere rilevato) in un'unica operazione.

AES-256-GCM combina il block cipher AES in modalità Counter (CTR) per la
cifratura con GHASH per l'autenticazione. Il "256" si riferisce alla
dimensione della chiave (256 bit / 32 byte). "GCM" sta per
Galois/Counter Mode.

Ogni message key derivata dalla catena simmetrica del Double Ratchet
viene usata come chiave AES-256-GCM. Per ogni messaggio viene generato
un nuovo nonce casuale (chiamato anche IV - initialization vector). Il
ciphertext include un authentication tag da 128 bit che rileva qualsiasi
manomissione.

### Come viene usato

Il flusso di cifratura è:

1. Il Double Ratchet deriva una message key (32 byte) tramite KDF_CK
2. Viene generato un nonce casuale da 12 byte usando `os.urandom` (backend) o
 `crypto.getRandomValues` (frontend)
3. AES-256-GCM cifra il testo in chiaro usando la message key e il nonce
4. Gli associated data (identificatori di mittente e destinatario) vengono autenticati
 ma non cifrati
5. L'output è ciphertext + un authentication tag da 16 byte (GCM accoda
 il tag al ciphertext)

Implementazione backend in `frontend/src/crypto/double-ratchet.ts`:

```python
def _encrypt_with_message_key(self, message_key, plaintext, associated_data):
 aesgcm = AESGCM(message_key)
 nonce = os.urandom(AES_GCM_NONCE_SIZE) # 12 bytes from config.py
 ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data)
 return nonce, ciphertext
```

La decifratura backend in `frontend/src/crypto/double-ratchet.ts` cattura
le eccezioni `InvalidTag` , che indicano che il ciphertext è stato
manomesso, che è stata usata la chiave sbagliata o che gli associated
data non corrispondono. L'errore viene rilanciato come
`ValueError("Message tampered or corrupted")`
.

L'implementazione frontend in `frontend/src/crypto/primitives.ts` usa la
WebCrypto API:

```typescript
const nonce = generateRandomBytes(AES_GCM_NONCE_SIZE)
const ciphertext = await subtle.encrypt(
 {
 name: "AES-GCM",
 iv: nonce.buffer,
 additionalData: associatedData?.buffer,
 },
 cryptoKey,
 plaintext.buffer
)
```

La decifratura frontend in `frontend/src/crypto/primitives.ts` rispecchia
questo comportamento con `subtle.decrypt`. La WebCrypto API lancia una
`DOMException` se l'autenticazione fallisce, il che è funzionalmente
equivalente all'eccezione Python `InvalidTag`.

### Perché non semplicemente AES-CBC?

AES-CBC (Cipher Block Chaining) è l'altra modalità AES che si vede
comunemente. Fornisce confidenzialità ma non integrità. Un messaggio
cifrato in CBC può essere modificato da un attaccante (attacchi di
bit-flipping) senza che il destinatario rilevi la modifica. Per
aggiungere integrità serve un calcolo HMAC separato (Encrypt-then-MAC o
MAC-then-Encrypt), che aggiunge complessità e occasioni per errori di
implementazione.

GCM gestisce entrambe le cose in un'unica operazione. Ha anche vantaggi
pratici di prestazioni: la cifratura in modalità CTR di GCM è
parallelizzabile sui core della CPU e beneficia delle istruzioni
hardware AES-NI, mentre CBC è intrinsecamente sequenziale (ogni blocco
dipende dal blocco di ciphertext precedente).

AES-GCM è la modalità raccomandata dal NIST per le nuove applicazioni
(NIST SP 800-38D). È la cipher suite obbligatoria in TLS 1.3. Non c'è
alcuna ragione di sicurezza per preferire CBC a GCM nelle nuove
implementazioni.

### Parametri

Come definito in `config.py`:

```
AES_GCM_KEY_SIZE = 32 (256 bits)
AES_GCM_NONCE_SIZE = 12 (96 bits)
HKDF_OUTPUT_SIZE = 32 (256 bits)
```

Il nonce da 12 byte (96 bit) è la dimensione raccomandata per GCM. Nonce
più lunghi sono consentiti dalla specifica ma richiedono un'elaborazione
interna aggiuntiva. L'authentication tag è di 128 bit (16 byte), che è
il tag a lunghezza piena e il default sia per la libreria Python
`cryptography` sia per WebCrypto.

Con una chiave da 256 bit, AES-256-GCM fornisce 128 bit di sicurezza
contro gli attacchi di recupero della chiave (l'algoritmo di Grover
ridurrebbe AES-256 a 128 bit di sicurezza su un computer quantistico, ma
AES-128 scenderebbe a 64 bit, ed è per questo che le chiavi da 256 bit
sono la scelta lungimirante).

Il nonce deve essere unico per chiave. Poiché ogni message key del
Double Ratchet viene usata esattamente una volta, il requisito di
unicità del nonce è automaticamente soddisfatto anche se il generatore
di numeri casuali producesse una collisione: lo stesso nonce con una
chiave diversa non è un problema. È un punto sottile ma importante. Il
design a chiave-per-messaggio del Double Ratchet fa sì che il riuso del
nonce (che sarebbe catastrofico con una chiave statica) non sia qui una
preoccupazione realistica.


## WebAuthn / Passkey

### Cos'è

WebAuthn (Web Authentication) è uno standard W3C (pubblicato per la
prima volta nel marzo 2019, Level 2 nell'aprile 2021, Level 3 in
lavorazione) per l'autenticazione senza password basata sulla
crittografia a chiave pubblica. Invece delle password, gli utenti si
autenticano usando coppie di chiavi asimmetriche gestite da un
authenticator: una chiave di sicurezza hardware (YubiKey, SoloKey), un
authenticator di piattaforma (Touch ID, Face ID, Windows Hello,
biometria Android) o un authenticator cross-platform accessibile tramite
un telefono.

Il termine "Passkey" si riferisce a una credenziale discoverable
(chiamata anche "resident key") che viene sincronizzata tra i
dispositivi tramite un credential manager di piattaforma (iCloud
Keychain, Google Password Manager, 1Password, ecc.). Passkey è il nome
commerciale, pensato per il pubblico consumer, delle credenziali
discoverable WebAuthn.

L'intuizione chiave è che con WebAuthn la chiave privata non lascia mai
l'authenticator. Il server memorizza solo la chiave pubblica e un ID
della credenziale. L'autenticazione è un protocollo challenge-response:
il server invia una challenge casuale, l'authenticator la firma con la
chiave privata e il server verifica la firma con la chiave pubblica
memorizzata. Nessun segreto viene trasmesso, nessun segreto viene
memorizzato sul server, e non c'è nulla che un attaccante possa rubare
dal server che gli permetta di impersonare l'utente.

### Perché non le password?

Le password sono il principale vettore d'attacco per la compromissione
dell'autenticazione. Ecco perché, specificamente nel contesto di
un'applicazione di messaggistica cifrata:

**Phishing.** Un attaccante crea una replica convincente della pagina di
login e induce l'utente a inserire la propria password. Con WebAuthn,
l'authenticator lega crittograficamente la credenziale all'origine (nome
di dominio) del sito web. Se l'utente visita `evil-chat.com` invece di
`real-chat.com`, l'authenticator non userà la credenziale per
`real-chat.com` perché l'origine non corrisponde. L'utente non può
essere indotto ad autenticarsi sul sito sbagliato perché l'authenticator
semplicemente non risponderà alla challenge. Questo è automatico e non
richiede che l'utente sia consapevole dell'attacco.

**Credential stuffing.** Gli utenti riutilizzano le password tra i
servizi. Quando un servizio viene violato (e le violazioni di database
di password avvengono di continuo -- Collection #1 nel 2019 ha esposto
773 milioni di coppie email/password), gli attaccanti provano quelle
password su altri servizi. Le credenziali WebAuthn sono uniche per
relying party (sito web). Non c'è alcuna password da riutilizzare.

**Keylogger.** Il malware che cattura le battute dei tasti può
registrare le password mentre gli utenti le digitano. L'autenticazione
WebAuthn usa la verifica biometrica (impronta digitale, volto) o
l'inserimento di un PIN sul dispositivo authenticator, non sulla
tastiera. Anche se un keylogger catturasse un PIN, il PIN da solo è
inutile senza il possesso fisico del dispositivo authenticator.

**Compromissione del server.** Se un server memorizza hash delle
password e il database viene rubato, gli attaccanti possono tentare il
cracking offline. La violazione di LastPass del 2023 ha esposto i vault
di password cifrati di oltre 25 milioni di utenti; gli utenti con master
password deboli si sono visti craccare i vault. Con WebAuthn, il server
memorizza solo chiavi pubbliche. Rubare chiavi pubbliche non dà nulla
all'attaccante: non puoi derivare una chiave privata da una chiave
pubblica, e non puoi falsificare una firma senza la chiave privata.

### Flusso di registrazione

```
Step 1: Client requests registration options
 Client ----> POST /auth/register/begin ----> Server

 Server:
 - Generates 32-byte random challenge (secrets.token_bytes)
 - Stores challenge in Redis with 10-minute TTL
 - Returns WebAuthn PublicKeyCredentialCreationOptions

Step 2: Browser creates credential
 Browser ----> navigator.credentials.create(options) ----> Authenticator

 Authenticator:
 - Prompts user for biometric/PIN verification
 - Generates new ECDSA or EdDSA keypair
 - Stores private key internally (NEVER exported)
 - Returns attestation object (signed credential public key)

Step 3: Client sends attestation for verification
 Client ----> POST /auth/register/complete ----> Server

 Server:
 - Verifies attestation signature
 - Verifies challenge matches stored value
 - Extracts credential public key and credential ID
 - Stores PUBLIC KEY + credential ID in PostgreSQL
 - Deletes challenge from Redis
 - Private key STAYS on authenticator -- server never sees it
```

Riferimento all'implementazione: `backend/app/core/passkey/passkey_manager.py`
(`generate_registration_options`). viene generata una challenge da 32
byte: `challenge = secrets.token_bytes(WEBAUTHN_CHALLENGE_BYTES)`. Alle
righe 74-87, le opzioni WebAuthn vengono costruite con la configurazione
dell'RP, le informazioni sull'utente e i requisiti dell'authenticator.
La selezione dell'authenticator specifica
`ResidentKeyRequirement.REQUIRED`, che forza la creazione di una
credenziale discoverable (passkey).

La verifica della registrazione in `backend/app/core/passkey/passkey_manager.py`
(`verify_registration`) chiama `verify_registration_response` alle righe
105-110, che convalida l'oggetto di attestation, controlla la challenge,
verifica l'RP ID e conferma l'origine.

### Flusso di autenticazione

```
Step 1: Client requests authentication options
 Client ----> POST /auth/authenticate/begin ----> Server

 Server:
 - Generates new 32-byte challenge
 - Stores challenge in Redis with 10-minute TTL
 - Returns WebAuthn PublicKeyCredentialRequestOptions

Step 2: Browser signs challenge
 Browser ----> navigator.credentials.get(options) ----> Authenticator

 Authenticator:
 - Prompts user for biometric/PIN
 - Signs challenge with stored private key
 - Increments signature counter
 - Returns assertion (signed challenge + counter)

Step 3: Client sends assertion for verification
 Client ----> POST /auth/authenticate/complete ----> Server

 Server:
 - Retrieves stored public key from PostgreSQL
 - Verifies signature using stored public key
 - Verifies challenge matches stored value
 - Checks signature counter INCREASED (clone detection!)
 - Updates stored counter value
 - Returns authenticated session
```

Riferimento all'implementazione: `backend/app/core/passkey/passkey_manager.py`
(`generate_authentication_options`). viene generata una nuova challenge.
Alle righe 148-153 vengono costruite le opzioni di autenticazione
WebAuthn.

La verifica dell'autenticazione in `backend/app/core/passkey/passkey_manager.py`
(`verify_authentication`). Alle righe 173-180, l'assertion viene
verificata rispetto alla challenge attesa, all'RP ID, all'origine e alla
chiave pubblica della credenziale memorizzata. Segue il controllo
critico di rilevamento dei cloni.

### Rilevamento dei cloni

Gli authenticator hardware (YubiKey, chiavi Titan, ecc.) mantengono un
contatore di firma interno che si incrementa ogni volta che
l'authenticator viene usato. Questo contatore è incluso nei dati
dell'assertion firmata. Il server memorizza l'ultimo valore del
contatore e verifica che ogni nuova autenticazione presenti un contatore
più alto.

Se il server riceve un'assertion con un valore del contatore che non è
aumentato (o è diminuito), questo indica una di due cose:

1. L'hardware dell'authenticator è stato clonato (il suo materiale di
 chiave è stato estratto e caricato su un secondo dispositivo)
2. È in corso un tentativo di replay attack

Entrambi gli scenari sono incidenti di sicurezza che giustificano il
blocco dell'autenticazione e l'avviso all'utente.

Riferimento: `backend/app/core/passkey/passkey_manager.py`:

```python
if (credential_current_sign_count != 0 and new_sign_count != 0
 and new_sign_count <= credential_current_sign_count):
 logger.error(
 "Signature counter did not increase: current=%s, new=%s. "
 "Possible cloned authenticator detected!",
 credential_current_sign_count,
 new_sign_count
 )
 raise ValueError(
 "Signature counter anomaly detected - potential cloned authenticator"
 )
```

Le condizioni `credential_current_sign_count != 0` e
`new_sign_count != 0` sono difensive: alcuni authenticator (in
particolare gli authenticator di piattaforma e le passkey) riportano
sempre un contatore pari a 0, indicando che non implementano il
rilevamento dei cloni basato sul contatore. Per quegli authenticator, il
controllo di rilevamento dei cloni viene saltato perché genererebbe
sempre un falso positivo. Questo è coerente con le indicazioni della
specifica WebAuthn sulla gestione degli authenticator che non supportano
i contatori di firma.


## Confronto a tempo costante

Un dettaglio che vale la pena evidenziare: `frontend/src/crypto/primitives.ts`
implementa un confronto di array di byte a tempo costante:

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

Questa funzione confronta due array di byte in tempo costante, il che
significa che il tempo di esecuzione non dipende da dove si trova la
prima differenza. Un confronto ingenuo (`a[i] !== b[i]` con un return
anticipato) fa trapelare informazioni attraverso il timing: se il primo
byte è diverso, la funzione ritorna immediatamente, e un attaccante che
misura il tempo di risposta può dedurre che il primo byte era sbagliato.
Iterando su tutti i byte e facendo l'OR dei risultati degli XOR, la
funzione impiega sempre lo stesso tempo indipendentemente dal fatto che
gli array differiscano al byte 0 o al byte 31.

Gli attacchi side-channel basati sul timing non sono teorici. Nel 2009,
Nate Lawson e Taylor Nelson dimostrarono attacchi di timing pratici
contro la verifica HMAC in un web framework, recuperando l'HMAC corretto
un byte alla volta. Il confronto a tempo costante elimina completamente
questo vettore d'attacco.


## Come si collegano questi concetti

Il diagramma seguente mostra come i cinque concetti principali si
collegano per formare l'architettura di sicurezza completa
dell'applicazione:

```
+-----------------------------------------------------------+
| AUTHENTICATION LAYER |
| |
| WebAuthn/Passkeys |
| | |
| +--> User identity established |
| +--> No password to steal, phish, or brute force |
| +--> Clone detection via signature counter |
| | |
| v |
| +----------------------------------------------------+ |
| | KEY AGREEMENT LAYER | |
| | | |
| | X3DH Key Exchange | |
| | +--> Asynchronous (works when peer is offline) | |
| | +--> 4 DH operations for mutual authentication | |
| | +--> Produces initial shared secret (32 bytes) | |
| | | | |
| | v | |
| | Double Ratchet Initialization | |
| | +--> Shared secret becomes root key | |
| | +--> Sending and receiving chains created | |
| +----------------------------------------------------+ |
| | |
| v |
| +----------------------------------------------------+ |
| | MESSAGE ENCRYPTION LAYER | |
| | | |
| | Double Ratchet (ongoing) | |
| | +--> KDF_CK derives per-message keys | |
| | +--> DH ratchet provides post-compromise security| |
| | +--> Skipped key cache handles out-of-order msgs | |
| | | | |
| | v | |
| | AES-256-GCM | |
| | +--> Encrypts plaintext with message key | |
| | +--> Random 12-byte nonce per message | |
| | +--> Authentication tag detects tampering | |
| +----------------------------------------------------+ |
| | |
| v |
| +----------------------------------------------------+ |
| | TRANSPORT LAYER | |
| | | |
| | WebSocket (real-time delivery) | |
| | +--> Carries encrypted blobs between clients | |
| | +--> Server sees only ciphertext + metadata | |
| | | |
| | SurrealDB (persistence) | |
| | +--> Stores encrypted messages at rest | |
| | +--> No decryption capability on server | |
| +----------------------------------------------------+ |
+-----------------------------------------------------------+
```

I livelli interagiscono in una sequenza rigorosamente dall'alto verso il basso per le nuove conversazioni:

1. L'utente si autentica con WebAuthn (dimostrando la propria identità
 senza una password)
2. X3DH stabilisce un segreto condiviso con il peer (anche se il peer è
 offline)
3. Il segreto condiviso inizializza il Double Ratchet
 (`frontend/src/crypto/double-ratchet.ts`)
4. Ogni messaggio viene cifrato con una chiave AES-256-GCM unica derivata
 dal ratchet (`frontend/src/crypto/double-ratchet.ts`)
5. Il messaggio cifrato viene trasmesso via WebSocket e memorizzato in
 SurrealDB (`backend/app/services/message_service.py`)

Per le conversazioni in corso, si ripetono solo i passi 4 e 5.
L'handshake X3DH avviene una volta per conversazione. Il Double Ratchet
poi gira in autonomia, derivando nuove chiavi per ogni messaggio senza
alcuna ulteriore interazione con il server per la gestione delle chiavi.


## Standard di settore

Questa sezione mappa le misure di sicurezza del progetto su specifici
framework di settore. Queste mappature sono utili per discussioni di
compliance, audit di sicurezza e threat modeling.

### OWASP Top 10 (2021)

**A02: Cryptographic Failures** -- Precedentemente "Sensitive Data
Exposure", questa categoria copre i fallimenti nell'implementazione
crittografica. Questo progetto la affronta tramite:
- AES-256-GCM con derivazione delle chiavi HKDF-SHA256 (nessun algoritmo debole)
- Chiavi da 256 bit che soddisfano i requisiti minimi NIST per l'uso dopo il 2030
- Chiavi uniche per messaggio (nessun riuso delle chiavi)
- Generazione corretta dei nonce tramite CSPRNG (`os.urandom`, `crypto.getRandomValues`)
- Nessuna memorizzazione di testo in chiaro sul server

**A04: Insecure Design** -- Copre le debolezze di sicurezza a livello di
architettura. L'architettura zero-knowledge previene intere classi di
attacchi lato server:
- La compromissione del server non rivela il contenuto dei messaggi
- Nessuna chiave di decifratura lato server da rubare
- I prekey bundle contengono solo materiale di chiave pubblica
- Il modello store-and-forward tratta tutti i messaggi come blob opachi

**A07: Identification and Authentication Failures** -- Copre
l'autenticazione difettosa. WebAuthn elimina gli attacchi di
autenticazione più comuni:
- Nessuna password significa nessun credential stuffing (CWE-521)
- Il binding all'origine previene il phishing (CWE-352)
- I contatori di firma rilevano gli authenticator clonati
- Il challenge-response previene i replay attack

### MITRE ATT&CK

**T1557: Adversary-in-the-Middle** -- Un attaccante intercetta le
comunicazioni tra due parti. La cifratura E2E con l'autenticazione
reciproca di X3DH impedisce attacchi MITM significativi. Anche se un
attaccante controlla il percorso di rete, vede solo ciphertext
AES-256-GCM. La verifica della signed prekey di X3DH (usando firme
Ed25519) impedisce all'attaccante di sostituire le chiavi con le
proprie.

**T1528: Steal Application Access Token** -- Un attaccante ruba un token
di autenticazione per impersonare un utente. Le credenziali WebAuthn
sono legate crittograficamente all'origine della relying party. Un token
rubato da un sito non può essere usato su un altro. La chiave privata
non lascia mai l'authenticator, quindi non c'è alcun token da rubare dal
server o dal traffico di rete.

**T1110: Brute Force** -- Un attaccante tenta di indovinare le
credenziali tramite tentativi esaustivi. Con WebAuthn, non ci sono
password da attaccare con brute force. L'autenticazione richiede il
possesso fisico dell'authenticator e la verifica biometrica (o PIN),
rendendo impossibile un brute force da remoto. La challenge cambia a
ogni tentativo di autenticazione, impedendo il replay delle assertion
catturate.

### Riferimenti CWE

**CWE-327: Use of a Broken or Risky Cryptographic Algorithm** -- Questo
progetto usa esclusivamente algoritmi approvati dal NIST e ampiamente
verificati: X25519 (Curve25519 ECDH), Ed25519 (EdDSA), AES-256-GCM,
HMAC-SHA256, HKDF-SHA256. Non viene implementata alcuna primitiva
crittografica personalizzata. Sia la libreria Python `cryptography` sia
la WebCrypto API del browser forniscono implementazioni ben testate.

**CWE-326: Inadequate Encryption Strength** -- Le chiavi simmetriche da
256 bit (AES-256) e le chiavi su curva ellittica da 256 bit (X25519,
Ed25519) forniscono un livello di sicurezza di 128 bit, che supera la
raccomandazione minima del NIST di 112 bit per l'uso fino al 2030 e
oltre (NIST SP 800-57 Part 1).

**CWE-330: Use of Insufficiently Random Values** -- La casualità
proviene da due fonti: `os.urandom` sul backend (che legge dal CSPRNG
del sistema operativo -- `/dev/urandom` su Linux) e
`crypto.getRandomValues` sul frontend (che usa il CSPRNG del browser).
Entrambe sono crittograficamente sicure. La generazione dei nonce in
`frontend/src/crypto/double-ratchet.ts` e `frontend/src/crypto/primitives.ts` usa
esclusivamente queste.

**CWE-311: Missing Encryption of Sensitive Data** -- Tutto il contenuto
dei messaggi è cifrato a riposo (SurrealDB memorizza solo ciphertext) e
in transito (WebSocket su TLS trasporta ciphertext AES-256-GCM). Lo
stato del Double Ratchet stesso viene serializzato e memorizzato, ma
questo stato non contiene alcun testo in chiaro; contiene materiale di
chiave per i messaggi futuri.


## Casi di studio reali

### Caso di studio 1: la controversia del 2020 sulla cifratura E2E di Zoom

**Cronologia.** Nel marzo 2020, The Intercept pubblicò un'inchiesta che
rivelava che le affermazioni di Zoom sulla "end-to-end encryption" erano
false. Il Citizen Lab dell'Università di Toronto pubblicò un rapporto di
approfondimento nell'aprile 2020, individuando che Zoom usava AES-128 in
modalità ECB e che le chiavi di cifratura venivano generate dai server
di Zoom e trasmesse ai partecipanti attraverso l'infrastruttura di Zoom.
Nel novembre 2020 la FTC presentò un reclamo, e Zoom accettò un accordo
che le imponeva di implementare un programma di sicurezza completo e di
smettere di travisare la propria cifratura.

**Cosa fallì a livello architetturale.** Il design di Zoom collocava le
chiavi di cifratura sul server. La connessione tra il client e il server
era cifrata (cifratura del trasporto tramite TLS), e i flussi multimediali
tra il server e i partecipanti erano cifrati con AES-128, ma il server
generava e deteneva tutto il materiale di chiave. Questo significava che
i server di Zoom potevano decifrare ogni chiamata. L'uso della modalità
ECB (Electronic Codebook) era un ulteriore fallimento: ECB cifra ogni
blocco in modo indipendente, il che significa che blocchi di testo in
chiaro identici producono blocchi di ciphertext identici, facendo
trapelare pattern strutturali nei dati. ECB è considerata insicura da
decenni ed è esplicitamente sconsigliata in ogni manuale moderno di
crittografia.

**Come questo progetto previene lo stesso fallimento.** In questo
progetto, il server non genera né detiene mai chiavi di cifratura per il
contenuto dei messaggi. La generazione delle chiavi avviene in due
punti:

1. Sul backend, `frontend/src/crypto/x3dh.ts` genera coppie di identity
 key X25519 usando `X25519PrivateKey.generate`, che richiama il
 generatore di numeri casuali di OpenSSL. Queste chiavi servono al
 protocollo X3DH, e le chiavi private sono memorizzate nel database per
 il percorso di scambio delle chiavi lato server.

2. Sul frontend, `frontend/src/crypto/primitives.ts` genera coppie di chiavi
 X25519 usando la WebCrypto API (`subtle.generateKey`), che usa il
 CSPRNG del browser supportato dall'hardware. Nel modello di cifratura
 lato client, queste chiavi non lasciano mai il browser.

La funzione `store_encrypted_message` (`backend/app/services/message_service.py`)
riceve dal client ciphertext già cifrato e lo memorizza direttamente in
SurrealDB senza alcuna decifratura lato server. Il ruolo del server è
esplicitamente quello di un relay cieco. Anche se l'intera
infrastruttura server venisse compromessa, l'attaccante otterrebbe solo
blob cifrati senza le corrispondenti chiavi di decifratura.

### Caso di studio 2: l'adozione del Signal Protocol da parte di WhatsApp (2016)

**Contesto.** Nel novembre 2014, Open Whisper Systems annunciò una
partnership con WhatsApp per integrare il Signal Protocol nel client di
messaggistica WhatsApp. Il rollout avvenne in modo incrementale: il
protocollo Axolotl di TextSecure (in seguito rinominato Signal Protocol)
fu distribuito prima per i messaggi da Android ad Android, poi esteso ai
messaggi di gruppo, ai contenuti multimediali e alle chiamate vocali su
tutte le piattaforme. La distribuzione completa fu annunciata
nell'aprile 2016, rendendo WhatsApp la più grande distribuzione di
cifratura E2E della storia, che all'epoca copriva oltre 1 miliardo di
utenti.

**Dettagli tecnici.** WhatsApp implementò la stessa combinazione X3DH +
Double Ratchet usata in questo progetto. Ogni messaggio riceve una
chiave AES unica tramite il meccanismo del ratchet, fornendo forward
secrecy su miliardi di messaggi giornalieri. L'implementazione di
WhatsApp memorizza i prekey bundle sui propri server (proprio come fa
questo progetto tramite `backend/app/services/prekey_service.py`),
permettendo lo stabilimento asincrono della sessione. L'handshake X3DH
viene eseguito quando un utente avvia una nuova conversazione, e il
Double Ratchet gira poi in modo continuo.

**Impatto.** Quando nel 2016 il governo brasiliano ordinò a WhatsApp di
fornire il contenuto dei messaggi, WhatsApp dimostrò di non poter
ottemperare per ragioni architetturali: non possedeva le chiavi di
decifratura. Non era una decisione di policy né una promessa; era un
fatto matematico. Il protocollo rende dimostrabilmente impossibile per il
server decifrare i messaggi. La stessa situazione si verificò con le
richieste dell'FBI negli Stati Uniti e con le pretese dei governi di
India e Regno Unito.

**Collegamento con questo progetto.** L'implementazione di X3DH in
`frontend/src/crypto/x3dh.ts` e il Double Ratchet in
`frontend/src/crypto/double-ratchet.ts` implementano le stesse operazioni
crittografiche descritte nella specifica del Signal Protocol. Le stesse
quattro operazioni DH , la stessa derivazione HKDF , le stesse
operazioni delle catene KDF (righe 79-109 di frontend/src/crypto/double-ratchet.ts)
e lo stesso meccanismo delle skipped message key . Le specifiche del
protocollo sono pubbliche, le dimostrazioni formali di sicurezza sono
pubblicate, e l'implementazione le segue direttamente.

### Caso di studio 3: la violazione di LastPass del 2023

**Cronologia.** Nell'agosto 2022, un attaccante compromise la
workstation di uno sviluppatore di LastPass tramite un pacchetto
software multimediale di terze parti vulnerabile. Usando le credenziali
dello sviluppatore, l'attaccante accedette all'ambiente di sviluppo di
LastPass e rubò codice sorgente e informazioni tecniche. In un secondo
incidente, l'attaccante usò le informazioni della prima violazione per
prendere di mira il computer di casa di un ingegnere DevOps, installando
un keylogger che catturò la master password dell'ingegnere per un vault
aziendale di LastPass. Con questo accesso, l'attaccante esfiltrò i vault
di password cifrati dei clienti e i dati di backup dal cloud storage di
LastPass.

**Cosa fu esposto.** Furono rubati i vault cifrati di circa 25,6 milioni
di utenti. Sebbene i vault fossero cifrati con AES-256 usando la master
password di ciascun utente come input della derivazione della chiave, la
sicurezza dei vault dipendeva interamente dalla robustezza della master
password dell'utente. Gli utenti con master password corte, comuni o già
violate in precedenza si videro craccare i vault tramite attacchi brute
force offline. La violazione espose anche metadati non cifrati, inclusi
gli URL dei siti web, che rivelavano con quali servizi ciascun utente
avesse un account.

**Collegamento con questo progetto.** La violazione di LastPass dimostra
esattamente perché WebAuthn/Passkey sono superiori all'autenticazione
basata su password, anche quando le password vengono usate per derivare
chiavi di cifratura.

Nell'implementazione WebAuthn di questo progetto
(`backend/app/core/passkey/passkey_manager.py`), non c'è alcuna password.
L'utente si autentica con la biometria o un PIN sul proprio dispositivo
authenticator. L'authenticator detiene una chiave privata ECDSA o EdDSA
legata all'hardware e mai esportata. Se il database del server viene
completamente compromesso, l'attaccante ottiene solo chiavi pubbliche
delle credenziali e ID delle credenziali. Le chiavi pubbliche non
possono essere invertite per ottenere le chiavi private (servirebbe
risolvere il problema del logaritmo discreto su curve ellittiche, che è
computazionalmente infattibile). Non c'è alcuna master password da
attaccare con brute force, nessun hash di password da craccare e nessun
segreto equivalente a una password memorizzato sul server.

Il contrasto è netto: LastPass memorizzava segreti degli utenti protetti
da una password scelta dall'utente. Questo progetto memorizza segreti
degli utenti protetti da una chiave privata legata all'hardware che
l'utente non può scegliere, non può indebolire e non può riutilizzare
per sbaglio su un altro sito.


## Verifica la tua comprensione

Prima di passare al documento sull'architettura, dovresti essere in
grado di rispondere a queste domande. Se non riesci a rispondere a una
di esse con sicurezza, rileggi la sezione pertinente.

1. **Perché X3DH ha bisogno di quattro operazioni DH separate invece di
 una sola?** Quale specifica proprietà di sicurezza fornisce ciascuna
 operazione? Quale attacco diventa possibile se una qualsiasi singola
 operazione viene rimossa?

2. **Se un attaccante compromette una chain key del Double Ratchet al
 messaggio N, quali messaggi può decifrare?** Quali messaggi restano
 protetti? Quando l'attaccante perde l'accesso? Ripercorri la funzione
 KDF_CK per dimostrare la tua risposta.

3. **Perché WebAuthn è resistente agli attacchi di phishing, mentre le
 password tradizionali non lo sono?** Quale proprietà del protocollo
 WebAuthn impedisce a una falsa pagina di login di catturare credenziali
 utilizzabili? Suggerimento: pensa a cosa controlla l'authenticator
 prima di firmare la challenge.

4. **Cosa succede se Alice invia a Bob i messaggi 1, 2, 3, 4, 5 e Bob li
 riceve nell'ordine 1, 3, 5, 2, 4?** Ripercorri il meccanismo delle
 skipped message key per ciascun messaggio ricevuto. Quante chiavi sono
 in cache dopo l'elaborazione del messaggio 3? E dopo l'elaborazione del
 messaggio 5?

5. **Perché il Double Ratchet usa due costanti HMAC diverse (0x01 e
 0x02) in KDF_CK?** Cosa andrebbe storto se sia la chain key sia la
 message key fossero derivate con la stessa costante?


## Letture di approfondimento

### Specifiche essenziali

- Le specifiche del Signal Protocol: https://signal.org/docs/
- Specifica X3DH: https://signal.org/docs/specifications/x3dh/
- Specifica Double Ratchet: https://signal.org/docs/specifications/doubleratchet/
- Specifica WebAuthn Level 3: https://www.w3.org/TR/webauthn-3/

### Analisi accademiche

- "A Formal Security Analysis of the Signal Messaging Protocol" --
 Cohn-Gordon, Cremers, Dowling, Garratt, Stebila. IEEE EuroS&P 2017.
 Dimostrazione formale che il Signal Protocol soddisfa le proprietà di
 sicurezza dichiarate (scambio di chiavi autenticato, forward secrecy,
 post-compromise security) sotto l'assunzione Gap-DH.

- "The Signal Protocol: A Cryptographic Analysis" -- Cohn-Gordon et al.
 (2017). Versione estesa con dimostrazioni che coprono i componenti X3DH
 e Double Ratchet sia singolarmente sia composti.

- "On Ends-to-Ends Encryption: Asynchronous Group Messaging with Strong
 Security Guarantees" -- Cohn-Gordon et al. IEEE S&P 2018. Estende
 l'analisi agli scenari di messaggistica di gruppo.

### FIDO e WebAuthn

- Panoramica tecnica di FIDO2: https://fidoalliance.org/fido2/
- White paper della FIDO Alliance sulle passkey:
 https://fidoalliance.org/passkeys/
- NIST SP 800-63B Digital Identity Guidelines (tipi di authenticator e
 livelli di garanzia)

### Primitive crittografiche

- Daniel J. Bernstein, "Curve25519: new Diffie-Hellman speed records"
 (2006) -- L'articolo che ha introdotto la funzione X25519 usata per il
 DH in questo progetto.

- Daniel J. Bernstein, Niels Duif, Tanja Lange, Peter Schwabe, Bo-Yin
 Yang, "High-speed high-security signatures" (2012) -- L'articolo che ha
 introdotto Ed25519, usato per firmare le prekey.

- NIST SP 800-38D, "Recommendation for Block Cipher Modes of Operation:
 Galois/Counter Mode (GCM) and GMAC" -- La specifica della modalità
 AES-GCM usata per la cifratura dei messaggi.

- Hugo Krawczyk, "Cryptographic Extraction and Key Derivation: The HKDF
 Scheme" (2010) -- L'articolo alla base di HKDF, la funzione di
 derivazione delle chiavi usata in tutto il Double Ratchet.

### Contesto storico

- Nikita Borisov, Ian Goldberg, Eric Brewer, "Off-the-Record
 Communication, or, Why Not To Use PGP" (2004) -- Il protocollo OTR che
 ha introdotto il concetto di messaggistica ripudiabile e con forward
 secrecy, e che ha ispirato direttamente il design del Signal Protocol.

- Whitfield Diffie e Martin Hellman, "New Directions in Cryptography"
 (1976) -- L'articolo originale che ha introdotto la crittografia a
 chiave pubblica e lo scambio di chiavi Diffie-Hellman alla base di
 X3DH.

- Phil Zimmermann e PGP: le "Crypto Wars" degli anni '90, in cui il
 governo statunitense tentò di limitare l'esportazione della
 crittografia forte. Zimmermann pubblicò il codice sorgente di PGP in un
 libro stampato per aggirare i controlli sulle esportazioni sotto la
 protezione del Primo Emendamento. Questa storia è il motivo per cui oggi
 il software crittografico può essere distribuito liberamente.
