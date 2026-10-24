# 01-CONCEPTS.md

# Concetti di DLP

## Cos'è la Data Loss Prevention?

La DLP è la pratica di rilevare e impedire che dati sensibili vengano memorizzati, trasmessi o acceduti in modi non autorizzati. Le tre modalità della DLP corrispondono alle tre superfici di scansione di questo progetto:

- **Dati at rest**: file su disco, record nei database, documenti nel cloud storage. Il nostro file scanner e il nostro database scanner coprono questa superficie.
- **Dati in movimento**: traffico di rete, chiamate API, trasmissioni email. Il nostro network scanner copre questa superficie.
- **Dati in uso**: contenuto della clipboard, catture dello schermo, memoria delle applicazioni. Non trattati qui (richiedono agent sugli endpoint).

La domanda fondamentale a cui la DLP risponde: "Dove sono i nostri dati sensibili, e sono protetti?"

## Tecniche di rilevamento

### Pattern matching con validazione

L'approccio più semplice: pattern regex che corrispondono a formati strutturali come gli SSN (XXX-XX-XXXX), i numeri di carta di credito (16 cifre con prefissi noti) e le API key (pattern di prefisso noti come `AKIA` per AWS).

Il problema della sola regex è il tasso di falsi positivi. La stringa `123-45-6789` corrisponde a un pattern SSN ma compare in dati di test, numeri di serie e interni telefonici. La stringa `4532015112830366` corrisponde a un pattern di carta Visa ma potrebbe essere un identificatore casuale di 16 cifre.

Ecco perché i sistemi DLP in produzione non si affidano mai alla sola regex. Aggiungono livelli di validazione:

**La validazione del checksum** elimina le corrispondenze strutturalmente non valide. I numeri di carta di credito usano l'algoritmo di Luhn: raddoppia una cifra sì e una no a partire da destra, sottrai 9 se il risultato supera 9, e verifica che il totale sia divisibile per 10. Un numero casuale di 16 cifre ha circa il 10% di probabilità di superare Luhn, il che resta comunque un segnale utile. Gli IBAN usano Mod-97 (ISO 7064): riordina il codice paese e le cifre di controllo, converti le lettere in numeri, e verifica che il risultato mod 97 sia uguale a 1. I numeri NHS usano Mod-11 con moltiplicazione pesata delle cifre.

**La validazione dell'area SSN** controlla che le prime tre cifre non siano 000, 666 o 900-999 (mai assegnate dalla SSA). Anche i numeri di gruppo e di serie devono essere diversi da zero. Questo elimina gli intervalli che la Social Security Administration non ha mai usato.

### Scoring delle keyword di contesto

Un numero di 9 cifre che corrisponde al formato SSN vicino alle parole "social security" ha più probabilità di essere un vero SSN rispetto allo stesso numero in una colonna etichettata "serial_number". Lo scoring di contesto scansiona una finestra bidirezionale attorno a ogni corrispondenza alla ricerca di keyword pertinenti:

```
For SSN patterns: "ssn", "social security", "social_security_number", "tax id"
For credit cards: "credit card", "card number", "payment", "billing"
For API keys: "api_key", "secret", "token", "authorization"
```

Le keyword trovate all'interno della finestra (default: 10 token in ciascuna direzione) aggiungono un boost da +0,05 a +0,35 a seconda della prossimità. Le keyword più vicine contribuiscono con più confidenza.

### Entropia di Shannon

Le stringhe dall'aspetto casuale spesso indicano segreti: API key, valori cifrati, credenziali codificate in base64. L'entropia di Shannon misura la casualità di una stringa:

```
H = -sum(p(x) * log2(p(x))) for each unique character x
```

Il testo inglese ha un'entropia di circa 3,5-4,5 bit per carattere. I dati codificati in base64 sono intorno a 5,5-6,0. I dati codificati in esadecimale sono intorno a 3,5-4,0. I dati veramente casuali si avvicinano a log2(alphabet_size). Una stringa di 40 caratteri con entropia superiore a 4,5 viene segnalata come potenziale segreto.

### Pipeline di confidence scoring

Ogni rilevamento produce un confidence score tra 0,0 e 1,0:

```
1. Regex match            -> base_score (0.10 to 0.85, configured per rule)
2. Checksum validation    -> +0.30 if the checksum passes
3. Context keyword search -> +0.05 to +0.35 based on keyword proximity
4. Entity co-occurrence   -> +0.10 to +0.20 if multiple PII types appear nearby
5. Final score capped at 1.0
```

Il punteggio si mappa sulla severità:
- 0,85+ = critical
- 0,65+ = high
- 0,40+ = medium
- 0,20+ = low
- sotto 0,20 = scartato

Una corrispondenza SSN (base 0,45) con area/gruppo/serie validi e la parola "ssn" nelle vicinanze ottiene 0,45 + 0,30 (la validazione dell'area funge da checksum implicito) + 0,15 (contesto) = 0,90, classificata come critical. Lo stesso pattern senza contesto ottiene 0,45, classificato come medium, il che è appropriato perché potrebbe essere un frammento di numero di telefono.

## Framework di compliance

I framework normativi definiscono quali tipi di dati richiedono protezione e cosa succede quando vengono esposti:

**HIPAA (Health Insurance Portability and Accountability Act)**: Definisce 18 tipi di Protected Health Information (PHI), tra cui SSN, numeri di cartella clinica, numeri di beneficiario del piano sanitario e identificatori biometrici. Un'entità soggetta che non protegge le PHI rischia multe da $100 a $50.000 per violazione (fino a 1,5 milioni di dollari all'anno per categoria). La violazione di Anthem del 2015 ha esposto 78,8 milioni di record e si è conclusa con un accordo da 16 milioni di dollari con l'HHS.

**PCI-DSS (Payment Card Industry Data Security Standard)**: Richiede la protezione dei dati del titolare della carta: numeri di conto primari (PAN), nomi dei titolari, date di scadenza e codici di servizio. Il PAN deve essere reso illeggibile (cifrato, sottoposto a hash, troncato o tokenizzato). La violazione di Heartland Payment Systems (2008) ha compromesso 130 milioni di numeri di carte di credito ed è costata all'azienda 140 milioni di dollari in risarcimenti.

**GDPR (General Data Protection Regulation)**: Si applica ai dati personali dei residenti nell'UE, inclusi nomi, indirizzi email, numeri di telefono, indirizzi IP e dati di localizzazione. Le multe arrivano al 4% del fatturato globale annuo o a 20 milioni di euro, a seconda di quale sia più alto. Meta è stata multata di 1,2 miliardi di euro nel 2023 per aver trasferito dati di utenti UE negli USA senza garanzie adeguate.

**CCPA (California Consumer Privacy Act)**: Copre le informazioni personali dei residenti in California. Categorie simili al GDPR ma con meccanismi di applicazione diversi. I consumatori possono fare causa direttamente per violazioni di dati che coinvolgono informazioni personali non cifrate ($100-$750 per consumatore per incidente).

## Concetti di Network DLP

### Esfiltrazione DNS

Gli attaccanti codificano i dati rubati nelle query DNS per aggirare i firewall che non ispezionano il traffico DNS. I dati sono codificati nelle label dei sottodomini:

```
aGVsbG8gd29ybGQ.evil.com    (base64 "hello world" in subdomain)
```

Segnali di rilevamento:
- **Entropia della label**: i sottodomini legittimi (www, mail, api) hanno bassa entropia. I dati codificati in base64 hanno un'entropia superiore a 4,0
- **Lunghezza del QNAME**: le query normali sono sotto i 50 caratteri. Le query di esfiltrazione superano i 100+
- **Volume di query TXT**: i record TXT vengono usati per ricevere i dati esfiltrati. Un picco di query TXT verso un singolo dominio è sospetto
- **Lunghezza della label del sottodominio**: le label DNS sopra i 50 caratteri non sono quasi mai legittime

Il gruppo APT OilRig (attribuito all'Iran) ha usato ampiamente il DNS tunneling in campagne contro governi del Medio Oriente, codificando documenti rubati in query di sottodominio verso l'infrastruttura di command-and-control. DNSCat2 e Iodine sono tool open source che implementano questa tecnica.

### Identificazione del protocollo

La Deep Packet Inspection (DPI) identifica i protocolli applicativi dai prefissi in byte del payload senza fare affidamento sui numeri di porta:

- Le richieste HTTP iniziano con i metodi: `GET `, `POST `, `PUT `, `DELETE `
- Le risposte HTTP iniziano con `HTTP/`
- I record TLS iniziano con `\x16\x03` (handshake + versione TLS)
- Le connessioni SSH iniziano con `SSH-`
- SMTP inizia con `220 ` (saluto del server)

Questo conta perché i dati sensibili nel traffico HTTP (API key negli header, SSN nei body delle POST) richiedono una gestione diversa rispetto agli stessi dati in uno stream TLS cifrato (dove puoi solo segnalare che sono stati trasmessi dati sensibili, non leggerne il contenuto).

### Riassemblaggio degli stream TCP

I dati a livello applicativo si estendono su più pacchetti TCP. Il riassemblaggio ricostruisce lo stream di byte originale:

1. Traccia i flussi tramite la quadrupla: (src_ip, dst_ip, src_port, dst_port)
2. Usa chiavi di flusso bidirezionali così che entrambe le direzioni di una conversazione si mappino sullo stesso flusso
3. Memorizza i segmenti indicizzati per numero di sequenza TCP
4. Ordina per numero di sequenza e concatena i payload, deduplicando le ritrasmissioni

Senza riassemblaggio, un numero di carta di credito diviso su due pacchetti verrebbe mancato dal pattern matching sui singoli payload.

## Redaction

I report DLP non devono mai contenere i dati sensibili grezzi che rilevano. Strategie di redaction:

- **Parziale**: preserva la struttura ma maschera il contenuto: `***-**-6789`, `4532****0366`
- **Completa**: sostituisce interamente: `[REDACTED]`
- **Nessuna**: nessuna redaction (solo per il debug, mai nei report di produzione)

La redaction parziale è preferita per il triage perché gli analisti possono identificare il tipo di dato e il valore approssimativo senza esporre l'intero contenuto sensibile. Le ultime 4 cifre di un SSN o di una carta di credito sono comunemente usate come token di verifica e sono considerate non sensibili da PCI-DSS.
