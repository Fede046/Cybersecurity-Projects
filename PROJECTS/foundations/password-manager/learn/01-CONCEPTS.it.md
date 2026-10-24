# Concetti

Questo file spiega ogni idea crittografica usata dal password vault, partendo da zero assoluto. Se non hai mai riflettuto a fondo su cosa sia davvero la "cifratura", questo è il posto giusto da cui partire. Se sai già distinguere una KDF da un hash, puoi leggere in fretta, ma leggi le sezioni su LastPass e Adobe, perché sono il *motivo* per cui ogni scelta progettuale nel codice è fatta così com'è.

## Indice

1. [Il problema: conservare segreti che ti serviranno in seguito](#1-il-problema-conservare-segreti-che-ti-serviranno-in-seguito)
2. [Che cosa è davvero la "cifratura"](#2-che-cosa-è-davvero-la-cifratura)
3. [Simmetrica vs asimmetrica — usiamo la simmetrica](#3-simmetrica-vs-asimmetrica--usiamo-la-simmetrica)
4. [Il problema della chiave: da dove viene la chiave?](#4-il-problema-della-chiave-da-dove-viene-la-chiave)
5. [Key derivation function: rendere le password costose](#5-key-derivation-function-rendere-le-password-costose)
6. [Argon2id nello specifico, e perché](#6-argon2id-nello-specifico-e-perché)
7. [Salt: sventare la precomputazione](#7-salt-sventare-la-precomputazione)
8. [Block cipher e modalità: "semplicemente cifrato" non basta](#8-block-cipher-e-modalità-semplicemente-cifrato-non-basta)
9. [AES-256-GCM: riservatezza + autenticità in un unico pacchetto](#9-aes-256-gcm-riservatezza--autenticità-in-un-unico-pacchetto)
10. [Nonce: la cosa più pericolosa di questa codebase](#10-nonce-la-cosa-più-pericolosa-di-questa-codebase)
11. [random vs secrets: l'errore più comune lato Python](#11-random-vs-secrets-lerrore-più-comune-lato-python)
12. [Mettere tutto insieme: il threat model](#12-mettere-tutto-insieme-il-threat-model)
13. [Violazioni reali che hanno reso giuste queste scelte](#13-violazioni-reali-che-hanno-reso-giuste-queste-scelte)

---

## 1. Il problema: conservare segreti che ti serviranno in seguito

Un password manager ha un compito strano. Deve:

- Ricordare le tue password *esattamente* (niente fuzzy matching: `hunter2` e `Hunter2` sono password diverse).
- Rifiutarsi di consegnarle a chiunque tranne te.
- Sopravvivere al furto del tuo computer.
- Sopravvivere al sequestro del tuo computer e alla sua acquisizione forense.
- Farti comunque entrare con un'unica stringa breve che puoi tenere a mente.

Questi obiettivi sono in tensione. "Ricorda la password esattamente" spinge verso "conservala in chiaro da qualche parte". "Non darla a nessuno tranne te" spinge verso "non conservare nulla". Tutto il resto di questo documento riguarda come navigare questa tensione.

Il piano che la risolve: **conserva le password rimescolate con una chiave che solo la master password può ricreare.** Se qualcuno ruba il file, tutto ciò che ha è un blob rimescolato e un problema *molto costoso* da risolvere. Se digiti la master password, la chiave viene ricreata in pochi secondi e il blob viene decifrato.

Questa è l'intera forma del progetto. Il resto di questo documento spiega come funziona davvero ciascun pezzo di quel piano.

---

## 2. Che cosa è davvero la "cifratura"

La cifratura è una funzione. Prende in ingresso tre cose:

- Un dato (il **plaintext**), come `"my github password is hunter2"`.
- Una **chiave**, che è semplicemente un blocco di byte casuali, di solito 16 o 32 byte.
- Un **algoritmo** (una procedura specifica per rimescolare i dati usando quella chiave).

E produce una sola cosa in uscita:

- Il **ciphertext**, ovvero la versione rimescolata. Della stessa lunghezza del plaintext (più un piccolo overhead, ne parleremo più avanti), ma con ogni byte che sembra rumore.

```
┌──────────────┐
│ "hunter2..." │ ──┐
└──────────────┘   │
                   ▼
            ┌────────────┐    ┌─────────────────────────┐
            │ ENCRYPT()  │ ─► │ "Vh\x91\x03\x7f\xe2..." │
            └────────────┘    └─────────────────────────┘
                   ▲
┌──────────────┐   │
│  32-byte key │ ──┘
└──────────────┘
```

La **decifratura** è la stessa funzione al contrario. Stessa chiave, stesso algoritmo: entra il ciphertext, esce il plaintext.

```
┌─────────────────────────┐
│ "Vh\x91\x03\x7f\xe2..." │ ──┐
└─────────────────────────┘   │
                              ▼
                       ┌────────────┐    ┌──────────────┐
                       │ DECRYPT()  │ ─► │ "hunter2..." │
                       └────────────┘    └──────────────┘
                              ▲
┌──────────────┐              │
│  32-byte key │ ─────────────┘
└──────────────┘
```

Se hai la chiave, la decifratura è veloce e ti restituisce l'originale. Se hai la chiave sbagliata, anche di un solo bit, ottieni spazzatura, oppure (per gli algoritmi moderni) la funzione di decifratura si rifiuta del tutto di funzionare e solleva un errore. Noi usiamo il secondo tipo.

**Cosa questa operazione *non* è:** la cifratura non è un'operazione a senso unico come un hash. Una funzione di hash (SHA-256, MD5, ecc.) scarta deliberatamente informazione in modo che l'originale non possa mai essere recuperato. La cifratura conserva ogni bit: si limita a rimescolarli in modo che tu non possa leggerli senza la chiave. Il punto è proprio che l'originale sia recuperabile, *ma solo da te*.

---

## 3. Simmetrica vs asimmetrica — usiamo la simmetrica

Esistono due grandi famiglie di cifratura.

La **cifratura simmetrica** usa la *stessa* chiave per cifrare e decifrare. Come un armadietto con una sola chiave fisica: chiunque tenga la chiave può sia chiuderlo sia aprirlo. AES è il famoso algoritmo simmetrico. Noi lo usiamo.

La **cifratura asimmetrica** usa due chiavi *diverse* matematicamente correlate. Una ("pubblica") chiude, l'altra ("privata") apre. Puoi distribuire la chiave pubblica al mondo e chiunque può mandarti cose cifrate che solo tu puoi leggere. RSA e gli algoritmi a curve ellittiche sono i famosi algoritmi asimmetrici. È ciò che alimenta HTTPS all'inizio di ogni connessione, firma i tuoi aggiornamenti software e alimenta il login SSH. Noi **non** la usiamo.

Perché non l'asimmetrica? Perché un password manager è un'operazione per una sola persona. Non esiste un "tu cifri, poi qualcun altro decifra": *tu* cifri, *tu* decifri. La simmetrica è lo strumento giusto. Gli algoritmi asimmetrici sono inoltre enormemente più lenti per byte, il che conta nei rari casi in cui hanno senso e li esclude per tutto il resto.

---

## 4. Il problema della chiave: da dove viene la chiave?

Abbiamo appena detto "la cifratura ha bisogno di una chiave da 32 byte". Dove prende un essere umano 32 byte casuali?

Non dalla testa. Gli esseri umani non riescono a ricordare 32 byte casuali: sono 256 bit di entropia, come chiedere a qualcuno di memorizzare un numero di 78 cifre. Il massimo che un essere umano può fare senza scriverlo è qualcosa come `correct horse battery staple` (~40 bit) o magari una frase lunga (~60 bit). Il divario con i 256 bit è *enorme*.

Quindi non chiediamo una chiave all'essere umano. Gli chiediamo una **password** (che può ricordare) e la trasformiamo in una chiave usando una **key derivation function**.

```
┌──────────────────┐      ┌─────────────────┐     ┌─────────────────┐
│ "correct horse   │      │   KEY           │     │ <32 random      │
│  battery staple" │ ───► │   DERIVATION    │ ──► │  bytes>         │
└──────────────────┘      │   FUNCTION      │     └─────────────────┘
                          └─────────────────┘
                                  ▲
┌──────────────────┐              │
│ random salt      │ ─────────────┘
│ (stored in file) │
└──────────────────┘
```

La trasformazione è deterministica: stessa password + stesso salt producono sempre la stessa chiave. È così che possiamo ricalcolare la chiave domani quando l'utente torna. Ma la trasformazione è anche **deliberatamente lenta**, ed è questo il trucco che rende sicuro l'intero sistema.

---

## 5. Key derivation function: rendere le password costose

Immagina che un attaccante abbia rubato il tuo file vault. Dentro ci sono:

- Il salt (16 byte di dati casuali pubblici).
- Il ciphertext (le tue password cifrate).
- Il nome dell'algoritmo e i parametri.

L'attaccante ora vuole indovinare la tua master password. Il modo ingenuo: prendere ogni password candidata, calcolarne l'hash con il salt, provare il risultato come chiave contro il ciphertext e vedere se il risultato è valido.

Se la nostra derivazione della chiave fosse semplicemente `SHA-256(password + salt)`, l'attaccante potrebbe fare **miliardi di tentativi al secondo** su una GPU moderna. Anche una password dall'aspetto robusto come `Tr0ub4dor&3` verrebbe violata in meno di un minuto. Questo perché SHA-256 è stato progettato per la *velocità*: è ottimizzato per verificare l'integrità di file enormi in frazioni di secondo.

**La soluzione: usare una funzione deliberatamente lenta da calcolare.**

Se ogni tentativo richiede mezzo secondo, un milione di tentativi richiede 6 giorni, un miliardo di tentativi richiede 16 anni e mille miliardi di tentativi (l'universo delle "password comuni di 12 caratteri") richiedono 16.000 anni. L'utente legittimo paga il costo solo *una volta* per sessione: mezzo secondo va benissimo. L'attaccante lo paga a *ogni* tentativo, per sempre.

Questo è l'intero scopo di una key derivation function (una "KDF"). È un'operazione simile a un hash, ma regolata per essere costosa, sia in tempo di CPU sia in memoria.

Perché entrambe? Perché gli attaccanti non usano CPU. Usano GPU e hardware personalizzato chiamato ASIC. Una GPU ha migliaia di piccoli core di calcolo ma pochissima memoria veloce per core. Quindi se usiamo un algoritmo che richiede molta memoria (diciamo 64 megabyte) per tentativo, le migliaia di core della GPU non possono più funzionare in parallelo: avrebbero bisogno di un centinaio di gigabyte di memoria veloce solo per tentare in parallelo. Gli ASIC affrontano lo stesso problema. **Rendere la funzione vorace di memoria è ciò che neutralizza gli attacchi basati sull'hardware.**

Il termine tecnico è "memory-hard". Le KDF moderne sono tutte memory-hard. Quelle vecchie (PBKDF2 del 2000) sono CPU-hard ma non memory-hard, ed è per questo che sono cadute in disuso.

---

## 6. Argon2id nello specifico, e perché

Nel 2013 la comunità crittografica ha organizzato la **Password Hashing Competition**, un concorso aperto per scegliere una nuova KDF standard. I ricercatori hanno presentato i loro progetti, si sono attaccati a vicenda e, dopo due anni di analisi, il vincitore è stato [**Argon2**](https://www.password-hashing.net/argon2-specs.pdf), nello specifico la variante chiamata **Argon2id**.

Esistono tre varianti:

- **Argon2d** — massimizza la resistenza al cracking su GPU, ma è vulnerabile ad attacchi side-channel (in cui un attaccante osserva le temporizzazioni degli accessi alla memoria della tua CPU).
- **Argon2i** — massimizza la resistenza agli attacchi side-channel, ma è più debole contro il cracking su GPU.
- **Argon2id** — un ibrido che fa entrambe le cose, scelto come impostazione predefinita raccomandata.

Noi usiamo Argon2id. Lo fa anche la [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), lo fanno password manager come 1Password e Bitwarden, e lo fa lo strumento di cifratura dei file `age`.

Argon2id ha **tre manopole di regolazione** che controllano quanto è costoso:

| Manopola          | Cosa controlla                                                    | Il nostro default | Perché                                                                                                                       |
| ----------------- | ----------------------------------------------------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `time_cost`       | Numero di passate sul buffer di memoria                            | 3           | Robusto per l'uso interattivo; ogni passata in più è un ulteriore fattore di rallentamento.                                  |
| `memory_cost`     | KiB di memoria usati durante la derivazione (1 KiB = 1024 byte)    | 65536 (64 MiB) | Ben al di sopra di ogni profilo OWASP; sconfigge gli attaccanti GPU/ASIC che non hanno memoria veloce per core.           |
| `parallelism`     | Thread che Argon2 può usare                                        | 4           | Un singolo utente su un laptop moderno beneficia dell'accelerazione parallela. L'attaccante ottiene la stessa accelerazione, quindi per la sicurezza è neutro ma migliora la UX. |

Questi numeri si trovano in [`constants.py`](../src/password_manager/constants.py), così è facile aumentarli tra cinque anni quando i laptop saranno più veloci.

**Dettaglio importante:** i parametri sono memorizzati *dentro il file del vault*. Se cambi i default nel codice, i vecchi vault si aprono comunque perché ricordano con quali parametri sono stati creati. Ci torneremo: è anche ciò che permette al comando `change-password` di aggiornare i vecchi vault ai nuovi parametri.

---

## 7. Salt: sventare la precomputazione

Un **salt** è un dato casuale mescolato alla password prima della derivazione della chiave. Non è segreto: lo conserviamo in chiaro dentro il file del vault.

Perché è importante?

Immagina di non usare un salt. Allora `derive_key("hunter2") = <some fixed 32 bytes>`: lo stesso su ogni macchina, per ogni utente. Un attaccante potrebbe, *anni prima di qualsiasi violazione*, calcolare le chiavi per il milione di password comuni più diffuse e conservarle in un'enorme lookup table. Rubi un vault → cerchi la chiave → sblocchi il vault. Nessun lavoro per singolo vault.

Questa precomputazione si chiama **rainbow table**. Era un attacco reale: esistono rainbow table scaricabili per MD5 senza salt che coprono migliaia di miliardi di hash di password comuni.

Un salt distrugge questo attacco. Ora `derive_key("hunter2", <16 random bytes>) = <different 32 bytes>` per ogni vault, perché ogni vault ha un salt diverso. L'attaccante non può precomputare nulla: deve eseguire l'intero lavoro di Argon2 *dopo* aver rubato il file del vault, *per ogni vault che vuole attaccare*.

**Altri due usi dei salt:**

1. **Unicità per utente.** Due utenti che scelgono la stessa password (di nuovo `hunter2`) ottengono chiavi *diverse*, perché i loro salt differiscono. Utile per i database di password su larga scala, dove purtroppo molti utenti scelgono le stesse password.
2. **Unicità per vault dello stesso utente.** Se hai due vault con la stessa master password, i loro ciphertext sono completamente diversi perché i loro salt sono diversi. Non è una minaccia attiva per un password manager monoutente, ma è un beneficio gratuito dell'avere i salt.

La dimensione del salt conta meno di quanto si pensi. 16 byte (128 bit) è la raccomandazione standard. Più grande va bene; più piccolo comincia a rischiare collisioni sulla popolazione globale dei vault.

---

## 8. Block cipher e modalità: "semplicemente cifrato" non basta

Ora abbiamo una chiave. Dobbiamo effettivamente cifrare con essa il contenuto del vault. È qui che la maggior parte dei progetti di crittografia per principianti sbaglia, quindi presta attenzione.

**AES** è un "block cipher". Cifra i dati in blocchi di dimensione fissa (16 byte alla volta). Dati un blocco da 16 byte e una chiave, produce un altro blocco da 16 byte che sembra casuale. Da solo, AES è semplicemente una funzione da un blocco di 16 byte a un altro.

Ma il tuo vault è molto più grande di 16 byte. Quindi serve un modo per concatenare i blocchi. Quel modo si chiama **modalità di operazione** (mode of operation).

La modalità più semplice è **ECB** ("electronic codebook"): spezzare il plaintext in blocchi da 16 byte, cifrare ciascuno indipendentemente, concatenare i risultati. È sbagliato. È famosamente, emblematicamente sbagliato:

```
                Versione cifrata in ECB
                di un'immagine di Tux, il pinguino di Linux
                (il pinguino si vede ancora)

                ┌────────────────┐
                │ ░░░░░░░░░░░░░░ │
                │ ░░░██████░░░░░ │
                │ ░░██░░░░██░░░░ │      Blocchi di input identici
                │ ░░░░██████░░░░ │  →   producono blocchi di output
                │ ░░░░░██░░░░░░░ │      identici, quindi il contorno
                │ ░░░░██████░░░░ │      dell'immagine trapela.
                │ ░░░░██░░██░░░░ │
                │ ░░██░░░░░░██░░ │
                │ ░░░░░░░░░░░░░░ │
                └────────────────┘
```

Ecco perché "usiamo AES" senza specificare la modalità non ti dice quasi nulla sulla sicurezza di un sistema.

La modalità successiva è **CBC** (cipher block chaining), che fa lo XOR di ogni blocco con il blocco di ciphertext precedente prima di cifrare. Meglio di ECB: blocchi di plaintext identici ora producono blocchi di ciphertext diversi. Ma CBC ha un *altro* problema: non ti dice se il ciphertext è stato manomesso. Un attaccante può invertire bit specifici nel ciphertext per invertire bit *prevedibili* nel plaintext decifrato, anche senza la chiave. Si chiama **bit-flipping attack** ed è stato usato contro sistemi reali.

La risposta giusta è una **modalità autenticata**: una che non si limita a cifrare, ma appone all'output anche un sigillo a prova di manomissione.

---

## 9. AES-256-GCM: riservatezza + autenticità in un unico pacchetto

**GCM** ("Galois/Counter Mode") è una modalità autenticata per AES. Produce due cose:

1. I byte cifrati (della stessa lunghezza del plaintext).
2. Un **authentication tag** da 16 byte: una piccola impronta che dice "questo esatto ciphertext è stato prodotto da qualcuno che possiede questa esatta chiave".

```
┌──────────────────┐
│ plaintext bytes  │
│ (vault contents) │
└──────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│            AES-256-GCM                  │
│                                         │
│   key  ─┐                               │
│   nonce ├─► [scramble & authenticate]   │
│         │                               │
└─────────┴───────────────────────────────┘
         │                  │
         ▼                  ▼
┌─────────────────┐  ┌─────────────────┐
│ ciphertext      │  │ 16-byte auth    │
│ (same length as │  │ tag             │
│  plaintext)     │  │                 │
└─────────────────┘  └─────────────────┘
```

La libreria li raggruppa per te in un'unica stringa di byte: il "ciphertext" restituito dalla libreria cryptography è in realtà `ciphertext || tag`, con il tag concatenato in fondo.

**In fase di decifratura**, il cipher controlla il tag *prima* di darti il plaintext. Se il tag non corrisponde (perché la chiave è sbagliata, il ciphertext è stato modificato o il nonce è sbagliato), il cipher solleva un errore e si rifiuta di produrre qualsiasi plaintext. Questa è la protezione che rende davvero sicuro "cifrato con AES-GCM".

Usiamo nello specifico **AES-256**-GCM: il "256" è la dimensione della chiave in bit. Anche AES-128 va bene, ma le chiavi a 256 bit sono lo standard per i livelli di margine "non voglio mai più pensarci". La differenza di prestazioni sulle CPU moderne è trascurabile perché AES ha istruzioni dedicate della CPU (AES-NI su Intel/AMD, ARMv8 Crypto Extensions su ARM).

---

## 10. Nonce: la cosa più pericolosa di questa codebase

Un **nonce** ("number used once") è un valore passato ad AES-GCM insieme alla chiave. Deve essere unico per ogni cifratura eseguita con la stessa chiave. **Riutilizzare un nonce con la stessa chiave in GCM è catastrofico**: fa trapelare informazioni sul plaintext a chiunque stia osservando e, in alcuni casi, rivela la chiave di autenticazione stessa.

Questo è lo spigolo più tagliente di AES-GCM e il motivo per cui siamo estremamente attenti su di esso in [`crypto.py`](../src/password_manager/crypto.py).

Cosa significa concretamente "fa trapelare il plaintext"? Se cifri due messaggi diversi M1 e M2 con la stessa chiave K e lo stesso nonce N, allora `C1 XOR C2 = M1 XOR M2`. Un attaccante che vede C1 e C2 può calcolare `M1 XOR M2` senza conoscere la chiave. Da lì, se riesce a indovinare una qualunque parte di M1 o M2, può recuperare la parte corrispondente dell'altro.

**La soluzione è meccanica: genera un nuovo nonce casuale da 12 byte a ogni singola cifratura.** GCM consente in sicurezza fino a circa 2³² (4 miliardi) di cifrature sotto una chiave con nonce casuali da 12 byte. Un singolo essere umano non salverà mai il proprio vault 4 miliardi di volte.

Un nonce *non* è un salt. Le differenze:

| Proprietà       | Salt                           | Nonce                              |
| --------------- | ------------------------------ | ---------------------------------- |
| Usato in        | Derivazione della chiave (Argon2id) | Cifratura (AES-GCM)           |
| Quanto spesso   | Una volta per *vault* (impostato all'init) | Una volta per *cifratura* (a ogni salvataggio) |
| Deve essere unico? | Sì (tra tutti i vault)      | Sì (per chiave, per tutta la vita) |
| Segreto?        | No                             | No                                 |

Entrambi sono casuali, entrambi vanno nel file, entrambi sono pubblici. Ma hanno compiti diversi e durate di vita diverse.

---

## 11. random vs secrets: l'errore più comune lato Python

Python ha due moduli che producono numeri casuali, e la differenza conta più di quasi qualsiasi altra scelta di API in questo progetto:

- [`random`](https://docs.python.org/3/library/random.html) — usa l'algoritmo **Mersenne Twister**. Veloce, statisticamente uniforme, **prevedibile**. Se un attaccante vede 624 output consecutivi, può ricostruire lo stato interno e prevedere per sempre ogni output futuro.
- [`secrets`](https://docs.python.org/3/library/secrets.html) — preleva byte dalla sorgente casuale crittografica del sistema operativo (`/dev/urandom` su Linux/Mac, `BCryptGenRandom` su Windows). Imprevedibile per progettazione.

Salt, nonce e chiavi DEVONO provenire da `secrets`. Le password generate DEVONO provenire da `secrets`. Se usassi `random` per una qualsiasi di queste cose, un attaccante che vedesse un solo output potrebbe prevedere ogni password successiva generata dal tuo tool, per *ogni* utente, su *ogni* macchina, per sempre.

Questo progetto usa `secrets` ovunque conti:

- `crypto.generate_salt()` e `crypto.generate_nonce()` → `secrets.token_bytes()`.
- `generator.generate_password()` → `secrets.choice()` e `secrets.randbelow()`.
- Lo shuffle di Fisher-Yates in `generator._secure_shuffle()` → `secrets.randbelow()` invece di `random.shuffle()`.

La regola è semplice: **se l'output deve essere difficile da prevedere, usa `secrets`. Sempre.**

---

## 12. Mettere tutto insieme: il threat model

Un "threat model" è una risposta scritta a "chi può violare questo sistema, e come?". Ecco il nostro:

**Da cosa ci difendiamo:**

- **Furto del file del vault.** Qualcuno copia `vault.json` dal tuo laptop. Senza la master password, tutto ciò che ha è un problema di indovinello lento e costoso (Argon2id con i nostri default: ~0,5 secondi/tentativo, ~15 anni per un miliardo di tentativi).
- **Manomissione del file del vault.** Qualcuno modifica dei byte in `vault.json` per cercare di causare comportamenti di decifratura strani. L'authentication tag di AES-GCM si rifiuta del tutto di decifrare.
- **Perdita di alimentazione a metà salvataggio.** Il pattern atomic-rename + fsync significa che finisci sempre con il vault VECCHIO o con quello NUOVO, mai con metà dell'uno o dell'altro. Descritto in dettaglio in [02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md).
- **Due processi `pv` che salvano contemporaneamente.** Un advisory file lock li serializza, così il salvataggio di nessuno dei due viene sovrascritto silenziosamente.
- **Dimenticare la tua master password (e il file che resta al sicuro a riposo proprio per questo).** Questo è l'intero punto.

**Da cosa NON ci difendiamo esplicitamente:**

- **Un keylogger sulla tua macchina.** Se qualcosa legge ogni tasto premuto, vede la tua master password mentre la digiti, e la partita finisce lì. La difesa per questo si trova a un livello diverso (cifratura dell'intero disco, rilevamento di keylogger a livello di OS, chiavi di sicurezza hardware). Questo non è un progetto di secure input a livello di sistema.
- **Un attaccante che guarda il tuo schermo mentre il vault è sbloccato.** Il comando `pv get` stampa le password su stdout in chiaro. L'attaccante che legge il tuo schermo possiede comunque la sessione.
- **Un OS davvero compromesso che può leggere la memoria dei processi.** Mentre il vault è sbloccato, le entry decifrate e la chiave AES risiedono nella memoria del tuo processo. Un attaccante privilegiato che può leggere quella memoria vince. Il metodo `close()` su `UnlockedVault` è una cancellazione best-effort, non una garanzia.
- **Una master password debole.** Argon2id rende il brute force costoso ma non impossibile. Se la tua master password è `password123`, un attaccante disposto ad aspettare prima o poi la violerà. La difesa è a tuo carico: scegli qualcosa di lungo.
- **I backup sotto il tuo controllo.** Il tool scrive un solo file, in modo atomico e durevole. Fare il backup da qualche altra parte (una chiavetta USB, Syncthing, ecc.) è compito tuo. La cifratura significa che è sicuro fare il backup in posti a cui non affideresti il plaintext.

Essere onesti su ciò da cui un tool si difende e ciò da cui non si difende è di per sé una competenza di sicurezza. Gli incidenti del mondo reale avvengono quasi sempre ai confini di un threat model, non al suo interno.

---

## 13. Violazioni reali che hanno reso giuste queste scelte

**[Adobe 2013](https://www.troyhunt.com/adobe-credentials-and-serious/)** — 153 milioni di record. Adobe cifrava le password con una singola chiave in **modalità ECB** e **senza salt per record**. Risultato: password identiche producevano ciphertext identici. I ricercatori potevano raggruppare gli utenti con la stessa password senza conoscere la password stessa. Combinato con i suggerimenti di password conservati in chiaro, ampie frazioni delle password trapelate sono state recuperate nella stessa settimana. Lezione: la casualità per cifratura (salt, nonce) e le modalità autenticate (non ECB) non sono opzionali.

**[LinkedIn 2012](https://en.wikipedia.org/wiki/2012_LinkedIn_hack)** — 6,5 milioni di record. Password conservate come **SHA-1 senza salt**. SHA-1 è veloce su GPU; senza salt, l'attaccante poteva precomputare una rainbow table una volta sola e usarla per sempre. Il 90% degli hash è stato violato nel giro di giorni. Lezione: i salt più una KDF lenta (non un hash veloce) sono il minimo moderno.

**[LastPass 2022](https://blog.lastpass.com/posts/notice-of-recent-security-incident)** — backup cifrati dei vault rubati. I vault stessi usavano una vera KDF (PBKDF2 con 100.100 iterazioni al momento della violazione), ma PBKDF2 non è memory-hard, quindi gli attacchi con GPU contro master password deboli sono da allora su scala industriale. Diversi report pubblici descrivono attaccanti che violano sottoinsiemi di vault e usano le password recuperate per furti di criptovalute. Lezione: una KDF memory-hard (Argon2id, scrypt) è significativamente più forte di PBKDF2 contro l'hardware moderno. Noi usiamo Argon2id.

**[Heartbleed 2014](https://heartbleed.com)** — un bug di divulgazione della memoria in OpenSSL. Non riguarda direttamente la conservazione delle password, ma ha dimostrato un principio correlato: i byte di materiale segreto che vivono nella memoria di un processo sono reali e vulnerabili. La disciplina di `UnlockedVault.close()` che cancella la chiave e dell'istruzione `with` che minimizza quanto a lungo il vault resta sbloccato discende da questa lezione.

**[Yahoo 2013 / 2016](https://en.wikipedia.org/wiki/Yahoo!_data_breaches)** — 3 miliardi di record (la più grande violazione della storia). Password conservate come **MD5**. Nel 2016 MD5 era già da 20 anni considerato violato per la conservazione delle password. Lezione: la cryptographic agility (la capacità di aggiornare nel tempo le scelte di hash/KDF) conta. Il motivo per cui questo progetto memorizza i parametri della KDF *nel file del vault*, invece di codificarli nel codice, è che i vecchi vault possano essere aggiornati in seguito senza costringere gli utenti a perdere dati. Il comando `change-password` sfrutta proprio questa capacità.

---

## Dove andare dopo

Ora sai *perché* ogni scelta progettuale nel codice è quella che è. È ora di vedere *come* è organizzato.

**[02-ARCHITECTURE.it.md](./02-ARCHITECTURE.it.md)** spiega come il progetto è suddiviso in moduli, che aspetto ha il file del vault su disco e il flusso passo-passo di ogni comando CLI.

Dopodiché, **[03-IMPLEMENTATION.it.md](./03-IMPLEMENTATION.it.md)** percorre ogni file sorgente riga per riga, spiegando le funzionalità di Python man mano che compaiono.
