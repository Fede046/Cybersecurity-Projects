# Concetti fondamentali

Questo file spiega le idee di sicurezza su cui è costruito lo scanner. Alla fine dovresti sapere cos'è HTTP, cos'è un header, perché esistono i sei header che controlliamo e che tipo di attacco blocca ciascuno di essi.

Leggi questo file prima del codice. Il codice è breve; sono i concetti che ci stanno dietro a richiedere tempo.

## 1. Cos'è davvero HTTP

Quando digiti `github.com` nel tuo browser, qualcosa deve andare a recuperare la pagina. Quel "qualcosa" parla un protocollo chiamato **HTTP** (HyperText Transfer Protocol). HTTPS è lo stesso protocollo con la cifratura avvolta intorno.

La forma di base di una conversazione HTTP è:

```
                       HTTP request
   ┌─────────────┐ ───────────────────────▶ ┌─────────────┐
   │             │                          │             │
   │   Browser   │                          │   Server    │
   │   (you)     │                          │  (github)   │
   │             │ ◀─────────────────────── │             │
   └─────────────┘       HTTP response      └─────────────┘
```

Tu (il browser) invii una **request**. Il server restituisce una **response**. La request dice "Vorrei la pagina in /home/index.html, per favore." La response dice "Ecco quella pagina. Inoltre, ecco alcune informazioni sulla pagina."

Sia la request sia la response sono semplicemente **testo**, con una struttura specifica.

### Come appare una response reale

Se togliessi la cifratura e osservassi i byte che tornano da `https://example.com`, vedresti qualcosa del genere:

```
HTTP/2 200
content-type: text/html; charset=UTF-8
content-length: 1256
strict-transport-security: max-age=31536000
x-frame-options: DENY
cache-control: max-age=600
date: Tue, 13 May 2026 12:00:00 GMT

<!doctype html>
<html>
  <head><title>Example Domain</title></head>
  <body>...</body>
</html>
```

Tre parti da notare:

1. **La status line.** `HTTP/2 200` significa "HTTP versione 2, status code 200." 200 significa "OK, ecco la tua pagina." 404 significherebbe "Non ho quella pagina." 500 significherebbe "Mi sono rotto mentre cercavo di creare quella pagina."
2. **Gli header.** Tutto ciò che sta tra la status line e la riga vuota. Ogni header è una riga, nel formato `name: value`. Possono essercene decine.
3. **Il body.** Tutto ciò che viene dopo la riga vuota. L'HTML vero e proprio, l'immagine, il JSON o qualunque altra cosa il server ti stia inviando.

Gli **header** sono ciò che interessa a questo scanner. Alcuni header riguardano la cache, il tipo di contenuto, i cookie e così via. Quelli li ignoriamo. Esistono sei header specifici per la sicurezza, ed è su quelli che assegniamo il voto.

### Perché gli header non distinguono tra maiuscole e minuscole

A volte vedrai `Strict-Transport-Security` e a volte `strict-transport-security`. **Significano la stessa cosa.** L'RFC 7230, la specifica ufficiale di HTTP, dice che i nomi degli header sono case insensitive. Server e proxy diversi usano maiuscole e minuscole in modo diverso. Lo scanner deve gestirli tutti, ed è per questo che convertiamo in minuscolo entrambi i lati prima di confrontarli.

## 2. Cos'è davvero un security header

Un security header è semplicemente un normale header HTTP con un nome che il browser è stato programmato a riconoscere come istruzione di sicurezza. Non c'è nulla di magico. Il server invia `Strict-Transport-Security: max-age=31536000` e il browser pensa "ah, il sito mi sta dicendo di ricordarmi di usare solo HTTPS per parlargli per i prossimi 31.536.000 secondi (un anno)."

Se il server si dimentica di inviare l'header, il browser torna al suo comportamento predefinito, che di solito è "fai quello che vuoi, nessuna protezione speciale." È proprio su quel comportamento predefinito che contano gli attaccanti.

Quindi i security header sono fondamentalmente una **promessa** che il sito fa al tuo browser. "Fidati di me, non servo mai contenuti tramite HTTP in chiaro. Se mai mi vedi su HTTP in chiaro, qualcuno ti sta mentendo, ignoralo."

## 3. I sei header che valutiamo

Lo scanner controlla sei header. Ognuno blocca una specifica classe di attacchi. Li vedremo uno per uno.

### 3.1 Strict-Transport-Security (HSTS): gravità HIGH

**Cosa dice al browser**

"Per i prossimi N secondi, parlami sempre e solo tramite HTTPS. Mai HTTP in chiaro. Se qualcuno ti passa un link a `http://my-site.com`, trasformalo in `https://` prima di fare la richiesta."

**L'attacco che blocca: SSL stripping**

Immagina di essere in una caffetteria e di digitare `github.com` nel tuo browser (senza il prefisso `https://`). Il tuo browser, per impostazione predefinita, prova prima `http://github.com`. Il server di GitHub allora dice "in realtà, usa HTTPS per favore" e ti reindirizza. Il browser segue il redirect, passa a HTTPS e ora tutto è cifrato.

Nell'intervallo tra "hai inviato una richiesta HTTP in chiaro" e "è tornato il redirect", un attaccante sullo stesso wifi (con un laptop che esegue `bettercap` o `sslstrip`) può intercettare tutto. Si mette in mezzo:

```
   ┌────────┐    plain HTTP    ┌──────────┐    HTTPS    ┌────────┐
   │  You   │ ───────────────▶ │ Attacker │ ──────────▶ │ GitHub │
   └────────┘                  │  (MITM)  │             └────────┘
                               └──────────┘
```

L'attaccante continua a parlare con GitHub tramite vero HTTPS, ma parla con te tramite HTTP in chiaro e riscrive ogni link `https://` della pagina in `http://`, così non riesci mai a uscirne. Pensi che il sito sembri normale. L'attaccante legge la tua password.

**Come HSTS lo blocca.** La prima volta che visiti con successo GitHub tramite HTTPS, il tuo browser ricorda l'header `Strict-Transport-Security`. La volta successiva, anche se digiti `http://github.com`, il browser si rifiuta del tutto di inviare una richiesta HTTP in chiaro. Passa a `https://` localmente, prima che qualsiasi pacchetto lasci la tua macchina. Il trucco dell'attaccante di "intercettare il passaggio in HTTP in chiaro" smette di funzionare.

**Come appare il valore**

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

- `max-age=31536000`: ricorda questa regola per 31.536.000 secondi (un anno).
- `includeSubDomains`: applica la regola a `api.github.com`, `docs.github.com`, a tutto ciò che termina con `github.com`.

**Perché lo scanner richiede un `max-age` positivo.** Un server potrebbe inviare `Strict-Transport-Security:` senza valore, oppure `Strict-Transport-Security: max-age=0`. Entrambi sono inutili. `max-age=0` **disattiva** attivamente HSTS (dice al browser di dimenticare qualsiasi regola HSTS precedente per questo sito). Quindi la sola presenza non basta. Lo scanner riporta `weak` ogni volta che il valore non corrisponde allo schema `max-age = <positive integer>`, il che intercetta sia il caso vuoto sia il caso deliberatamente a zero.

**Esempio reale.** La demo di sslstrip di Moxie Marlinspike a Black Hat 2009 ha reso famoso questo attacco. Dopo quel talk, tutte le principali banche sono passate a imporre solo HTTPS. Nel 2026 quasi tutti i siti seri inviano HSTS. I siti che non lo fanno sono di solito vecchi sistemi interni.

### 3.2 Content-Security-Policy (CSP): gravità HIGH

**Cosa dice al browser**

"Ecco l'elenco esatto dei luoghi da cui sono disposto a caricare script, stili, immagini, font, frame e altre risorse. Se vedi qualcosa nella pagina che ti chiede di caricare da qualsiasi altro posto, rifiuta."

**L'attacco che blocca: cross-site scripting (XSS)**

L'XSS si verifica quando un attaccante riesce a iniettare il proprio JavaScript in una pagina che altri utenti vedranno. Esempio classico:

```
Comment box on the site allows: I love this product!
Attacker types:                 <script>steal_cookie()</script>
Site displays it verbatim.
Now every user who loads the comment runs the attacker's JS in their session.
```

Quel JS viene eseguito con la piena fiducia del sito, quindi può leggere i cookie, leggere il session token della pagina, inviare richieste all'API come se fosse l'utente e così via. Per anni è stata la classe di bug numero 7 nella OWASP Top 10.

**Come CSP lo blocca.** Il sito dice al browser "gli script possono provenire solo da `https://my-site.com` o da `https://cdn.my-site.com`." Quando il browser vede `<script>steal_cookie()</script>` incorporato inline nell'HTML, pensa "quello script non proviene da una delle origini consentite, mi rifiuto di eseguirlo." L'XSS iniettato diventa semplicemente testo inerte.

**Come appare il valore**

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

- `default-src 'self'`: per tutto ciò che non è elencato specificamente qui sotto, consenti solo contenuti da questo esatto sito.
- `script-src 'self' https://cdn.example.com`: il JavaScript può provenire da questo sito o da cdn.example.com.

Una CSP reale è molto più lunga perché i siti reali caricano font, analytics, contenuti incorporati, ecc. Una buona CSP **non** contiene **mai** `'unsafe-inline'` per gli script (che consentirebbe i tag `<script>` inline su cui si basa un attacco XSS).

**Esempio reale.** La CSP di GitHub è una delle più lunghe del settore. Dopo l'XSS su GitHub Pages del 2018, l'hanno resa significativamente più restrittiva. Puoi verificarlo tu stesso: carica github.com nel tuo browser, apri i dev tools, guarda i response header.

**Perché controlliamo solo la presenza.** Fare il parsing corretto di una CSP è difficile. Mozilla Observatory fa un'analisi molto più approfondita (cerca `unsafe-inline`, origini wildcard, `default-src` mancante, ecc.). Il nostro scanner controlla solo che l'header sia presente. È la scelta giusta per un progetto foundations: passa a Observatory quando vorrai un'analisi più approfondita.

### 3.3 X-Content-Type-Options: gravità MEDIUM

**Cosa dice al browser**

"Quando ti invio un file, ti sto dicendo il suo tipo con l'header `Content-Type`. Credimi. Non guardare i byte per decidere da solo."

**L'attacco che blocca: MIME sniffing**

Le versioni più vecchie di Internet Explorer avevano una funzionalità "utile": se un server diceva che un file era `text/plain` ma il contenuto sembrava HTML, IE lo trattava come HTML e lo renderizzava. L'intento era essere tolleranti verso i server configurati male. L'effetto fu un'enorme falla di sicurezza.

Attacco: un sito permette agli utenti di caricare un'immagine del profilo. L'attaccante carica un file chiamato `cute_cat.gif` i cui primi byte sembrano una GIF valida, ma che più avanti contiene `<script>steal_everything()</script>`. Il server lo memorizza. In seguito il server lo restituisce ad altri utenti con `Content-Type: image/gif`. IE guarda i byte, vede il tag `<script>`, dice "in realtà questo è HTML" ed esegue lo script. Ora l'attaccante ha un XSS tramite il caricamento di un'immagine.

**Come l'header lo blocca.** `X-Content-Type-Options: nosniff` dice al browser "non rimettere in discussione il mio Content-Type." Se dico `image/gif`, lo tratti come un'immagine, punto. Il tag script non viene mai eseguito.

**Come appare il valore**

```
X-Content-Type-Options: nosniff
```

È letteralmente l'unico valore consentito. Questo header è il più noioso dei sei perché c'è una sola impostazione corretta e basta.

**Perché lo scanner richiede `nosniff`.** Alcuni server configurati male inviano `X-Content-Type-Options: off` o altra spazzatura. L'header deve contenere letteralmente la stringa `nosniff` per essere utile. Se non lo fa, riportiamo `weak`.

**Esempio reale.** La maggior parte delle vulnerabilità di MIME sniffing riguardava IE dalla versione 6 alla 9. I browser moderni adottano comunque per impostazione predefinita il comportamento nosniff per le risorse `script` e `style`. L'header è ancora importante per i browser più vecchi e per i contesti che non riguardano gli script.

### 3.4 X-Frame-Options: gravità MEDIUM

**Cosa dice al browser**

"Non permettere ad altri siti web di inserirmi dentro un `<iframe>` nelle loro pagine."

**L'attacco che blocca: clickjacking**

Immagina che l'attaccante crei una pagina web che dice:

```
   ┌────────────────────────────────────────┐
   │  Win a free iPhone! Click here!        │
   │                                        │
   │   ┌──────────────────────────┐         │
   │   │ [INVISIBLE iframe with   │         │
   │   │  github.com loaded       │         │
   │   │  inside it, positioned   │         │
   │   │  so the "Delete repo"    │         │
   │   │  button sits exactly     │         │
   │   │  over the "Click here"   │         │
   │   │  button]                 │         │
   │   └──────────────────────────┘         │
   └────────────────────────────────────────┘
```

Sei loggato su GitHub in un'altra scheda. Vedi la pagina dell'"iPhone gratis" sul sito dell'attaccante. Clicchi. Il clic non finisce sul pulsante visibile "Click here". Attraversa l'iframe trasparente e finisce sul pulsante "Delete repo" di GitHub. Il tuo session cookie viaggia insieme al clic. GitHub pensa che tu volessi eliminare il repo, quindi lo elimina.

Questo è il **clickjacking**. Il CSS per impostare la sovrapposizione è banale. L'unica cosa che gli impedisce di funzionare su ogni sessione di login ovunque è il sito vittima che si rifiuta di essere inserito in un frame.

**Come l'header lo blocca.** `X-Frame-Options: DENY` dice al browser "se mi vedi caricato dentro un iframe in qualsiasi altra pagina, rifiutati di renderizzarmi." L'iframe di clickjacking resta vuoto. Il clic non va da nessuna parte dannosa.

**Come appare il valore**

```
X-Frame-Options: DENY            # never allow framing, ever
X-Frame-Options: SAMEORIGIN      # only allow framing by pages on the same domain
```

`X-Frame-Options` è il metodo vecchio. Il metodo moderno è la direttiva `frame-ancestors` all'interno di `Content-Security-Policy`. La maggior parte dei siti reali invia entrambi per motivi di compatibilità con i browser. Lo scanner controlla solo `X-Frame-Options` per mantenere le cose semplici.

**Esempio reale.** Nel 2008 la pagina delle impostazioni di Flash di Adobe era vulnerabile al clickjacking. Nel 2009 il pulsante "Follow this user" di Twitter fu sfruttato con il clickjacking per attacchi del tipo "Re-tweet anything". Nel 2010 Facebook ebbe un worm di clickjacking che si diffondeva tramite i "mi piace" su pagine truccate. Lo schema è sempre lo stesso.

### 3.5 Referrer-Policy: gravità LOW

**Cosa dice al browser**

"Quando lasci la mia pagina per andare su un altro sito, non dire a quell'altro sito esattamente da quale pagina provieni."

**La fuga di dati che blocca: referer leakage**

I browser, per impostazione predefinita, inviano un header `Referer` con ogni richiesta in uscita che dice "l'utente è arrivato qui da questo URL." Sì, si scrive `Referer` con una sola R. Era un errore di battitura nella specifica HTTP originale del 1996 e ce lo terremo per sempre. Esilarante.

Perché è importante. Supponi che il tuo sito abbia URL come:

```
https://my-site.com/password-reset?token=abc123xyz
```

Un utente arriva su quella pagina e clicca un link esterno (per esempio, a un video di aiuto su YouTube). Il suo browser dice a YouTube "questo utente proviene da `https://my-site.com/password-reset?token=abc123xyz`." YouTube ora ha il token di reset della password dell'utente nei suoi access log. Ce l'ha anche chiunque abbia accesso ai log di YouTube.

È successo molte volte nella realtà. Lo schema: i token segreti negli URL trapelano tramite l'header Referer verso ogni risorsa di terze parti caricata dalla pagina.

**Come l'header lo blocca.** `Referrer-Policy: strict-origin-when-cross-origin` è un default sensato. Dice "quando l'utente va su un altro sito, comunica a quel sito solo la mia origine (`https://my-site.com`), mai l'URL completo con i parametri della query. Nascondi il path e la query string."

**Come appare il valore**

```
Referrer-Policy: strict-origin-when-cross-origin
Referrer-Policy: no-referrer                       # most paranoid
Referrer-Policy: same-origin                       # safe for outbound links
```

**Perché la gravità è LOW.** La referer leakage è reale e grave, ma dipende dal fatto che il sito metta segreti negli URL in primo luogo, che è un errore separato. L'header è un'utile difesa aggiuntiva, ma la sua assenza non significa una perdita garantita.

### 3.6 Permissions-Policy: gravità LOW

**Cosa dice al browser**

"Questa pagina non usa la fotocamera. Non permettere a nessun codice su questa pagina, inclusi gli script di terze parti incorporati, di chiedere all'utente l'accesso alla fotocamera."

Puoi farlo per fotocamera, microfono, geolocalizzazione, dispositivi USB, API di pagamento, accelerometro e un lungo elenco di altre funzionalità del browser.

**L'attacco che blocca: abuso delle funzionalità tramite terze parti compromesse**

Immagina che il tuo sito incorpori uno script di analytics di terze parti. Quella terza parte viene bucata. L'attaccante pubblica un aggiornamento malevolo dello script di analytics. Ora ogni pagina del tuo sito che include lo script esegue codice dell'attaccante con accesso a tutto ciò che il browser consente. Se l'attaccante scrive `navigator.mediaDevices.getUserMedia({ audio: true })`, l'utente riceve un prompt "questo sito vuole usare il tuo microfono". Molti utenti cliccheranno sì perché si fidano del tuo sito.

**Come l'header lo blocca.** `Permissions-Policy: camera=(), microphone=()` dice al browser "nessun codice su questa pagina può richiedere fotocamera o microfono, indipendentemente dalla provenienza. Non mostrare nemmeno il prompt." Lo script dell'attaccante riceve una risposta di rifiuto e va avanti.

**Come appare il valore**

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
```

Le parentesi vuote significano "nessuna origine è autorizzata a usare questa funzionalità." Puoi anche inserire esplicitamente delle origini in allowlist, ma per la maggior parte dei siti la risposta giusta per la maggior parte delle funzionalità è "nessuno."

**Perché la gravità è LOW.** Conta solo se (a) il tuo sito incorpora codice di terze parti, E (b) quella terza parte viene compromessa, E (c) la terza parte altrimenti proverebbe a usare queste funzionalità. È autentica difesa in profondità, ma è a diversi passi di distanza dalla superficie di attacco immediata.

## 4. La griglia di valutazione

Abbiamo quindi sei header. Ognuno ha una gravità (high, medium, low) che corrisponde a un valore in punti:

```
high   = 30 points each
medium = 15 points each
low    = 5 points each
```

L'attuale tabella delle regole ha 2 high, 2 medium, 2 low. Totale ottenibile: `2*30 + 2*15 + 2*5 = 90` punti. Un momento, non fa 100? Giusto. Il calcolo:

```
60 (two highs)   + 30 (two mediums) + 10 (two lows) = 100 points
```

Ho contato male. Rifacciamolo: high=30, due high fanno 60. Medium=15, due medium fanno 30. Low=5, due low fanno 10. Totale: 60+30+10 = **100**.

Per ogni header, lo scanner produce un finding:

- `ok` → guadagna il valore in punti pieno per quella regola
- `weak` → guadagna metà dei punti (l'header è presente ma il valore non è valido)
- `missing` → guadagna zero

Poi:

```
score = round( (earned points / total points) * 100 )
```

Il punteggio diventa un voto secondo una soglia standard per voti in lettere:

```
score >= 90 → A
score >= 80 → B
score >= 70 → C
score >= 60 → D
otherwise   → F
```

Questo rispecchia il funzionamento di Mozilla Observatory e securityheaders.com. Usano valori in punti esatti diversi e controllano più header, ma la forma è la stessa: finding ponderati, punteggio percentuale, voto in lettere.

## 5. Cosa NON fa questo scanner

Essere chiari sull'ambito è importante. Questo è un progetto foundations. **Non** è:

- **Un crawler.** Scansiona esattamente l'URL che gli fornisci. Una sola richiesta. Non segue i link all'interno della pagina e non valuta ogni sottopagina.
- **Un analizzatore di CSP.** Controlliamo solo che `Content-Security-Policy` esista. Non guardiamo dentro il valore alla ricerca di `unsafe-inline`, sorgenti wildcard, `default-src` mancante, ecc.
- **Un vulnerability scanner.** Non cerca di trovare SQL injection, XSS, open redirect o qualsiasi vera falla sfruttabile. Segnala solo la configurazione difensiva mancante.
- **Un'autorità.** I siti reali a volte omettono intenzionalmente certi header perché rompono una funzionalità di cui hanno bisogno. Un header mancante è un segnale che vale la pena indagare, non un verdetto automatico.

Quando vorrai un quadro più completo, passa a:
- **Mozilla Observatory** (`observatory.mozilla.org`): fa tutto ciò che facciamo noi più un'analisi approfondita della CSP, controlli sui cookie, valutazione della configurazione TLS.
- **securityheaders.com**: idea simile, interfaccia più semplice.
- **`nmap` con lo script http-security-headers**: per i nerd della riga di comando.

## 6. Riferimenti di settore

Se vuoi cercare queste cose da solo nella documentazione ufficiale:

- **MDN** (developer.mozilla.org) ha articoli autorevoli su ciascun header. Cerca il nome dell'header più "MDN."
- **OWASP Secure Headers Project** (owasp.org/www-project-secure-headers) ha l'elenco canonico e i valori raccomandati.
- **RFC 6797** è la specifica di HSTS. RFC 7034 è la specifica di X-Frame-Options. Leggere le specifiche è una competenza utile anche quando la specifica è noiosa.
- **CWE-693** "Protection Mechanism Failure" è l'ID common-weakness per gli header difensivi mancanti o configurati male.

## 7. Rapida autoverifica

Dovresti saper rispondere a queste domande prima di passare all'architettura:

1. Qual è la differenza tra una request HTTP e una response HTTP?
2. Dove si trovano gli header in una response (rispetto alla status line e al body)?
3. Quale header avrebbe impedito lo SSL stripping nella caffetteria?
4. Quale header avrebbe impedito l'attacco in stile clickjacking "delete repo"?
5. Perché lo scanner riporta `weak` invece di `ok` quando `X-Content-Type-Options` è presente ma il suo valore non è letteralmente `nosniff`?
6. Da che tipo di attacco difende la CSP, e perché lo scanner non analizza in profondità il valore della CSP?
7. Qual è il voto di un sito con punteggio 75? E con 60? E con 59?

Se qualcuna di queste ti sembra confusa, rileggi la sezione pertinente. L'implementazione nei prossimi due file avrà molto più senso una volta che questi concetti saranno solidi.

## Prossimo passo

Passa a **[02-ARCHITECTURE.md](./02-ARCHITECTURE.it.md)** per vedere come lo scanner è organizzato nel codice, oppure salta direttamente a **[03-IMPLEMENTATION.md](./03-IMPLEMENTATION.it.md)** per la spiegazione riga per riga.
