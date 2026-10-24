<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# HSM Emulator: Panoramica

## Che cos'è

Un **Hardware Security Module** software scritto in Zig che viene compilato in un vero shared object PKCS#11 (Cryptoki). Qualsiasi programma che sa già parlare con una smartcard, una YubiKey o un HSM cloud (OpenSSL, `pkcs11-tool`, SunPKCS11 di Java, p11-kit) può caricare questo `.so` e usarlo per generare chiavi, firmare, cifrare e conservare segreti. Parla la stessa C ABI di un vero HSM, quindi per l'host è indistinguibile dall'hardware finché non guardi dove risiedono effettivamente i byte.

Lo scopo del progetto è capire, costruendolo, che cosa fa davvero un HSM e perché l'interfaccia verso di esso ha l'aspetto che ha. Ottieni un modulo funzionante su cui puoi sperimentare con strumenti standard, e una codebase abbastanza piccola da leggere in un pomeriggio.

## Perché è importante

Una chiave crittografica è tutto. Se un attaccante legge la tua chiave privata TLS, la tua chiave di code signing o la chiave root della tua CA da un file o dalla memoria di un processo, ogni firma e ogni certificato che quella chiave abbia mai prodotto diventa falsificabile. La difesa su cui l'industria si è assestata è non lasciare mai che l'applicazione tocchi la chiave. La chiave vive dentro un trust boundary separato (un chip, una card, un'appliance di rete) e l'applicazione le invia *richieste*: "firma questo", "decifra quello". La chiave entra una volta e non esce mai più.

Il costo di sbagliare non è ipotetico.

- **RSA SecurID, 2011.** Gli attaccanti hanno violato RSA ed esfiltrato i seed record legati ai token SecurID, poi li hanno usati per attaccare Lockheed Martin. La lezione che l'industria ne ha tratto: i segreti che autenticano tutto il resto devono stare dietro un confine hardware, non in un database che un intruso può leggere.
- **Stuxnet, 2010.** Il worm conteneva driver firmati con chiavi di code signing legittime rubate a Realtek e JMicron. Firme valide da chiavi rubate permettono al malware di superare indisturbato i controlli di fiducia. Le chiavi di code signing sono esattamente il tipo di cosa che appartiene a un HSM, dove la chiave firma ma non esce mai.
- **DigiNotar, 2011.** Un'autorità di certificazione è stata compromessa e ha emesso certificati fraudolenti per `*.google.com`, usati per intercettare il traffico in Iran. L'azienda non è sopravvissuta. Le chiavi root e di emissione di una CA sono il caso d'uso canonico di un HSM: la chiave deve essere utilizzabile per firmare ma impossibile da copiare.

**Scenari reali in cui si applica:**
- **Autorità di certificazione e PKI.** Le chiavi di firma di una CA vivono in un HSM. Il software della CA chiama `C_Sign`; la chiave non esiste mai in forma utilizzabile nella memoria del server della CA.
- **Code signing.** Apple, Microsoft e le distro Linux firmano gli artifact di release con chiavi custodite in HSM, così che una violazione di un server non possa produrre malware firmato.
- **Backend di pagamento e KMS.** AWS KMS e CloudHSM, gli HSM di pagamento (traduzione dei PIN, emissione di carte) e il TDE dei database delegano tutti la crittografia vera e propria a un modulo che espone un'interfaccia molto simile a questa.

## Cosa imparerai

Questo progetto insegna come funziona davvero la custodia delle chiavi sotto il cofano. Costruendolo tu stesso, capirai:

**Concetti di sicurezza:**
- **Custodia delle chiavi e trust boundary.** Perché "l'applicazione non vede mai la chiave" è l'intero obiettivo del design, e come un attributo come `CKA_SENSITIVE` si traduce in un rifiuto netto di restituire mai `CKA_VALUE` al chiamante.
- **Autenticazione e lockout.** Come un PIN viene stirato con Argon2id così che il valore su disco sia inutile a un ladro, e perché tre tentativi sbagliati devono bloccare il token invece di lasciare che un attaccante vada avanti a tentativi.
- **Cifratura at rest con key wrapping.** Il pattern envelope: una master key casuale cifra i tuoi segreti, e la master key è a sua volta avvolta (wrapped) sotto una chiave derivata dal tuo PIN. Cambi il PIN e ri-avvolgi una piccola chiave invece di ricifrare tutto.
- **Release of unverified plaintext (RUP).** Perché un decrypt AEAD in streaming che restituisce byte prima che il tag di autenticazione sia verificato è pericoloso, e come fare buffering finché il tag non viene verificato elimina il problema.
- **Side channel.** Perché confrontare un PIN o un MAC con un normale confronto byte per byte fa trapelare informazioni attraverso il timing, e cosa risolve il confronto a tempo costante.

**Competenze tecniche:**
- **Scrivere una C ABI a mano.** Disporre i tipi `extern struct` in modo che corrispondano a un header C byte per byte, e dimostrarlo con asserzioni a compile time `@sizeOf` / `@offsetOf` invece di sperare.
- **Chiamare una libreria C in modo sicuro da Zig.** Fare il binding della `libcrypto` di OpenSSL tramite dichiarazioni `extern` scritte a mano (RSA sta lì perché la standard library di Zig non ne ha), senza far trapelare i suoi simboli fuori dal tuo modulo.
- **Concorrenza su un confine C.** Tenere un'istanza globale dietro un lock, e usare un contatore di generazione così che un'operazione lenta iniziata prima che il token venisse reinizializzato non possa salvare risultati obsoleti.
- **Igiene dei segreti in un linguaggio di sistema.** Azzerare il materiale delle chiavi con `std.crypto.secureZero`, capire cosa ti compra il trucco del `volatile`, e liberare i segreti sull'heap senza lasciarne copie in giro.

**Strumenti e tecniche:**
- **`pkcs11-tool`** (da OpenSC), l'host da riga di comando standard per qualsiasi modulo PKCS#11. Lo userai per inizializzare il token, fare login, generare chiavi e firmare.
- **`pkcs11-spy`**, uno shim che si mette davanti a qualsiasi modulo e registra nel log ogni chiamata con i suoi argomenti e il suo codice di ritorno. È il miglior strumento di debug in assoluto per il lavoro sull'ABI.
- **OpenSSL come oracolo.** Verificare che una firma prodotta da questo modulo sia valida per OpenSSL, e che un segreto ECDH che deriva corrisponda a `openssl pkeyutl -derive` byte per byte.

## Prerequisiti

Non ti serve esperienza pregressa con gli HSM. Ti serve però una certa familiarità con quanto segue.

**Conoscenze richieste:**
- **Basi della C ABI.** Cos'è il layout in memoria di una struct, cos'è un puntatore a funzione, cosa significa "calling convention". L'intero modulo è un esercizio nel far corrispondere esattamente un'interfaccia C.
- **Crittografia simmetrica contro asimmetrica.** La differenza tra AES (una chiave condivisa) e RSA/EC (una coppia di chiavi). Non ti serve la matematica, ma dovresti sapere quale è quale e cosa significa "firmare" rispetto a "cifrare".
- **Zig di base, o la voglia di leggerlo.** Il codice usa tagged union, `comptime`, error union e slice. Se conosci Rust, Go o il C++ moderno, riesci a seguirlo.

**Strumenti di cui avrai bisogno:**
- **Zig 0.16.0** per compilare il modulo. La versione esatta conta; l'interfaccia `std.Io` e la build API sono cambiate nella 0.16.
- **OpenSC** per `pkcs11-tool` e `pkcs11-spy`. Su Debian o Ubuntu: `apt install opensc`.
- **Header di sviluppo di OpenSSL** (`libssl-dev`) così che il modulo possa linkare `libcrypto` per RSA.

**Utili ma non obbligatori:**
- Una lettura della [specifica base PKCS#11 v2.40](https://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html). Puoi anche limitarti a leggere [`01-CONCEPTS.md`](./01-CONCEPTS.it.md) e consultare la specifica quando qualcosa non è chiaro.
- Familiarità con il modo in cui è strutturato SoftHSM2 (l'HSM software open source di riferimento). Questo progetto ne prende in prestito la suddivisione in tre livelli.

## Avvio rapido

```bash
cd PROJECTS/advanced/hsm-emulator

# Build, cross-check the ABI, and confirm pkcs11-tool can load it
./install.sh

# Keep all state in a scratch directory instead of $HOME
export ANGELAMOS_HSM_TOKEN=/tmp/hsm-token
export ANGELAMOS_HSM_OBJECTS=/tmp/hsm-objects
MOD=zig-out/lib/libhsm.so

# Look at the empty token, then bring it to life
pkcs11-tool --module $MOD -L
pkcs11-tool --module $MOD --init-token --label demo --so-pin 12345678
pkcs11-tool --module $MOD --init-pin --so-pin 12345678 --pin 1234

# Generate a keypair and sign with it
pkcs11-tool --module $MOD -l --pin 1234 --keypairgen --key-type EC:prime256v1 --label k1
echo "hello hsm" > msg.bin
pkcs11-tool --module $MOD -l --pin 1234 --sign --mechanism ECDSA-SHA256 --label k1 \
    --input-file msg.bin --output-file sig.bin
```

Output atteso: `-L` mostra uno slot il cui token parte come `uninitialized`. Dopo `--init-token` il token riporta `flags: ... token initialized`. La generazione della coppia di chiavi stampa i nuovi handle degli oggetti pubblico e privato, e `--sign` scrive una firma da 64 byte in `sig.bin`. Se vuoi *osservare* ogni chiamata Cryptoki che lo strumento effettua, esegui qualsiasi comando tramite `just spy` (per esempio `just spy --keypairgen ...`) e leggi il log dello spy.

## Struttura del progetto

```
hsm-emulator/
├── src/
│   ├── ck.zig             # the hand-written Cryptoki ABI (the contract with the host)
│   ├── main.zig           # the one exported symbol and the 68-entry function table
│   ├── config.zig         # every tunable constant in one place
│   ├── core/              # global state, sessions, the object store, the token record
│   ├── api/               # one file per group of C_ functions (the entry points)
│   └── crypto/            # the actual cryptography (AES, EC, RSA, hashing, the keystore)
├── tests/abi_test.zig     # proves ck.zig matches the OASIS headers at build time
├── examples/smoke.zig     # loads the built .so and drives it like a real host
└── vendor/pkcs11/         # the official OASIS headers, used only for the cross-check
```

Il singolo file più importante da capire per primo è `src/ck.zig`. Tutto il resto esiste per riempire i puntatori a funzione che quel file dichiara.

## Prossimi passi

1. **Comprendi le idee.** Leggi [01-CONCEPTS.md](./01-CONCEPTS.it.md) per la custodia delle chiavi, il modello di login, la cifratura at rest e le difese contro i side channel, ciascuno ancorato a un incidente reale.
2. **Guarda il design.** Leggi [02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md) per la suddivisione in tre livelli, il modello a oggetti e come viene gestita la concorrenza.
3. **Percorri il codice.** Leggi [03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md) per seguire un'operazione completa dalla chiamata C giù fino alla crittografia e ritorno.
4. **Impara la crittografia.** Leggi [MECHANICS.md](./MECHANICS.it.md) per come funziona byte per byte ogni meccanismo (modalità AES, GCM, key wrap, ECDSA, ECDH, RSA, Argon2id).
5. **Verifica il contratto.** Leggi [CONFORMANCE.md](./CONFORMANCE.it.md) per il codice di ritorno esatto a ogni confine.
6. **Estendilo.** Leggi [04-CHALLENGES.md](./04-CHALLENGES.it.md) per progetti che vanno da "aggiungi un meccanismo" a "implementa l'interfaccia v3.0".

## Problemi comuni

**`pkcs11-tool` scrive file nella mia home directory**
```
~/.angelamos-hsm-token
~/.angelamos-hsm-objects
```
Soluzione: imposta `ANGELAMOS_HSM_TOKEN` e `ANGELAMOS_HSM_OBJECTS` prima di ogni comando. Il modulo ripiega su `$HOME/.angelamos-hsm-*` solo quando queste non sono impostate.

**`error: unable to find dynamic system library 'crypto'`**
Soluzione: installa il pacchetto di sviluppo di OpenSSL (`libssl-dev` su Debian o Ubuntu, `openssl-devel` su Fedora). RSA è l'unica famiglia di meccanismi che linka una libreria C.

**`C_Login` restituisce `CKR_PIN_LOCKED` e non funziona niente**
Soluzione: hai provato il PIN sbagliato tre volte e il token si è bloccato. Riesegui `--init-token` (con il PIN del Security Officer) per resettarlo, il che cancella anche tutti gli oggetti. Questo è il comportamento previsto; un attaccante non deve avere tentativi illimitati.

**Un semplice `zig build` si comporta diversamente dal modulo installato**
Soluzione: `zig build` senza flag è una build Debug. Il modulo distribuito è `zig build --release=safe`. Debug e ReleaseSafe differiscono nel modo in cui la memoria liberata e azzerata viene avvelenata (poisoning), il che conta quando stai ispezionando segreti azzerati. Usa `--release=safe` per ottenere lo stesso artifact.

## Progetti correlati

Se l'hai trovato interessante, dai un'occhiata a:
- **api-rate-limiter**: un altro progetto avanzato in cui la proprietà di sicurezza (correttezza in presenza di concorrenza) risiede in una gestione dello stato progettata con cura, qui con script Lua invece di un mutex Zig.
- **bug-bounty-platform**: mostra come materiale delle chiavi e credenziali vengono gestiti in un'applicazione completa, il livello che *chiamerebbe* un HSM come questo.
