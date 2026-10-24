# Concetti DNS e implicazioni di sicurezza

## Fondamenti del protocollo DNS

Il DNS (Domain Name System) è un database gerarchico distribuito che traduce nomi di dominio leggibili dall'uomo in indirizzi IP. Comprendere a fondo il DNS è fondamentale sia per difendere le reti sia per capire i vettori d'attacco.

### La gerarchia DNS
```
                          [Root Servers]
                         a-m.root-servers.net
                               |
                    +----------+----------+
                    |                     |
              [.com TLD]              [.org TLD]
           TLD nameservers         TLD nameservers
                    |                     |
              +-----+-----+               |
              |           |               |
         [example.com] [google.com]  [example.org]
      Authoritative NS  Authoritative NS
```

Il comando trace (`dnslookup/resolver.py:293-426`) implementa l'attraversamento di questa gerarchia. Parte dai root server (`resolver.py:307-312`):
```python
root_servers = [
    ("a.root-servers.net", "198.41.0.4"),
    ("b.root-servers.net", "170.247.170.2"),
    ("c.root-servers.net", "192.33.4.12"),
]
```

Questi 13 indirizzi logici di root server (in realtà centinaia di server fisici tramite anycast) sono hardcoded in ogni resolver DNS.

### Tipi di record DNS

Il progetto supporta otto tipi di record (`dnslookup/resolver.py:24-33`):

**Record A (indirizzo IPv4)**
- Associa il dominio a un indirizzo IPv4 a 32 bit
- Esempio: `example.com → 93.184.216.34`
- Nota di sicurezza: può essere dirottato per reindirizzare il traffico

**Record AAAA (indirizzo IPv6)**
- Associa il dominio a un indirizzo IPv6 a 128 bit
- Gli attaccanti prendono sempre più di mira l'IPv6 perché è meno monitorato

**Record MX (Mail Exchanger)**
- Specifica i mail server di un dominio
- Ha un campo di priorità (`resolver.py:148-150`)
- Sicurezza: rivela l'infrastruttura email, può essere falsificato per il phishing

**Record NS (Name Server)**
- Delega una zona a specifici server DNS
- Fondamentale per comprendere la gerarchia DNS
- Gli attaccanti li prendono di mira per il DNS hijacking

**Record TXT (dati di testo)**
- Testo arbitrario, spesso usato per:
  - SPF (verifica del mittente email)
  - DKIM (firma delle email)
  - Verifica del dominio
  - Talvolta abusato per il DNS tunneling

**Record CNAME (Canonical Name)**
- Alias da un dominio a un altro
- Può creare lunghe catene che impattano sulle prestazioni
- Sicurezza: può essere usato per nascondere l'infrastruttura reale

**Record SOA (Start of Authority)**
- Contiene i metadati della zona (`resolver.py:153`)
- Mostra il nameserver primario e il numero di serie
- Rivela la configurazione dello zone transfer

**Record PTR (Pointer)**
- Mappatura DNS inversa (IP → hostname)
- Usato nella validazione delle email e nel logging
- La sua assenza indica una scarsa igiene dell'infrastruttura

### Come funziona la risoluzione DNS

Quando interroghi `www.example.com`, ecco cosa succede (implementato in `resolver.py:293-426`):

1. **Query al root server** (`resolver.py:319-328`)
   - Chiede al root server informazioni su `.com`
   - Il root rimanda ai server TLD di `.com`

2. **Query al server TLD** (`resolver.py:359-380`)
   - Chiede al TLD informazioni su `example.com`
   - Il TLD rimanda ai nameserver autoritativi

3. **Query al server autoritativo** (`resolver.py:329-348`)
   - Ottiene la risposta effettiva
   - La risposta è marcata come autoritativa

4. **Cache del risultato**
   - Il campo TTL controlla la durata della cache (`output.py:52-61`)
   - Questo progetto non usa la cache (query nuove ogni volta)

La funzione trace lo mostra visivamente (`output.py:266-310`).

## Concetti di sicurezza

### DNS Cache Poisoning (CVE-2008-1447, attacco Kaminsky)

Le risposte DNS non hanno un'autenticazione forte. Un attaccante può:
1. Inviare una query al server DNS della vittima
2. Inondarlo di risposte contraffatte prima che arrivi la risposta reale
3. Se la risposta contraffatta arriva per prima e ha il transaction ID corretto, viene messa in cache

**Difese:**
- DNSSEC (firme crittografiche)
- Porte sorgente randomizzate
- Randomizzazione del transaction ID

Questo tool non implementa la validazione DNSSEC, ma ti mostra i dati DNS grezzi per capire cosa potrebbe essere falsificato.

### DNS Tunneling (MITRE T1071.004)

Esfiltrazione di dati attraverso query DNS. Un attaccante potrebbe:
1. Codificare i dati rubati in un sottodominio: `<base64-data>.attacker.com`
2. Il suo server autoritativo registra tutte le query nei log
3. I dati vengono estratti dai log delle query DNS

Il supporto ai record TXT in questo tool mostra quanti dati possono stare nel DNS (`resolver.py:151-152`):
```python
elif record_type == RecordType.TXT:
    value = rdata.to_text()
```

I record TXT possono contenere 255 caratteri per stringa, con più stringhe per record.

### Ricognizione DNS (MITRE T1590.002)

Gli attaccanti usano il DNS per mappare l'infrastruttura prima degli attacchi:
- I record A/AAAA rivelano indirizzi IP e hosting provider
- I record MX mostrano l'infrastruttura email
- I record NS espongono il provider DNS
- I record TXT fanno trapelare le configurazioni SPF/DKIM

Il comando batch (`cli.py:266-350`) dimostra la ricognizione automatizzata su larga scala.

### DNS Amplification DDoS

L'attaccante invia piccole query DNS con IP sorgente falsificato (l'indirizzo della vittima). Il server DNS invia risposte di grandi dimensioni alla vittima. Il fattore di amplificazione può arrivare a 50x.

**Come individuarlo:**
- Pattern di query insoliti
- Volume elevato di query ANY (deprecate)
- Query per record TXT/DNSSEC di grandi dimensioni

### DNS Hijacking

Compromissione dell'infrastruttura DNS per reindirizzare il traffico:
- **Compromissione del registrar**: modifica dei record dei nameserver
- **Compromissione del nameserver**: modifica dei file di zona
- **Cache poisoning**: iniezione di record falsi nei resolver
- **BGP hijacking**: instradamento del traffico DNS verso l'attaccante

Incidenti reali:
- **Sea Turtle** (2019): ha preso di mira l'infrastruttura DNS governativa
- **MyEtherWallet** (2018): un BGP hijack ha reindirizzato verso un sito di phishing

## Problemi di privacy del DNS

Ogni query DNS è visibile a:
1. Il resolver DNS del tuo ISP
2. I nameserver autoritativi
3. Qualsiasi rete intermedia

Questo rivela la cronologia di navigazione. Soluzioni:
- **DNS over HTTPS (DoH)**: cifra le query in HTTPS
- **DNS over TLS (DoT)**: cifra le query in TLS
- **DNSCrypt**: cifra e autentica

Questo tool non implementa la cifratura, ma usa query standard sulla porta UDP 53.

## Sicurezza del Time-to-Live (TTL)

Il TTL controlla la durata della cache (`output.py:45-61`). Un TTL basso significa:
- Più query che arrivano ai server autoritativi
- Propagazione più rapida delle modifiche
- Meno opportunità per cache avvelenate e obsolete

Un TTL alto significa:
- Carico ridotto sull'infrastruttura DNS
- Incident response più lenta
- I record avvelenati persistono più a lungo

Gli attaccanti possono impostare TTL bassi sui domini malevoli per eludere le blacklist.

## Validazione DNSSEC

DNSSEC aggiunge firme crittografiche ai record DNS. Ogni zona firma i propri record con una chiave privata. I resolver verificano le firme usando le chiavi pubbliche.

**Catena di fiducia:**
1. La zona root firma la chiave pubblica di `.com`
2. `.com` firma la chiave pubblica di `example.com`  
3. `example.com` firma i propri record

Il comando WHOIS mostra lo stato DNSSEC (`whois_lookup.py:113-114`):
```python
if hasattr(w, "dnssec"):
    result.dnssec = str(w.dnssec) if w.dnssec else None
```

## Risposte di errore e il loro significato

Il resolver gestisce diverse condizioni di errore (`resolver.py:181-189`):

**NXDOMAIN**: il dominio non esiste
- Potrebbe indicare tentativi di typosquatting
- Utile per rilevare C2 di malware che usano DGA (domain generation algorithms)

**NOERROR con risposta vuota**: il dominio esiste ma non ha record di quel tipo
- Indica una configurazione errata o un setup incompleto

**SERVFAIL**: il server ha riscontrato un errore nell'elaborare la query
- Potrebbe indicare un fallimento della validazione DNSSEC
- Potrebbe suggerire che il server DNS è sotto attacco

**Timeout**: nessuna risposta ricevuta
- Problemi di rete
- Blocco da parte di un firewall
- Server DNS sovraccarico o down

## Operazioni async e prestazioni

Le query DNS sono I/O-bound. Il tool usa `asyncio` per la concorrenza (`resolver.py:233-242`):
```python
tasks = [
    query_record_type(domain, rt, resolver) for rt in record_types
]
query_results = await asyncio.gather(*tasks, return_exceptions=True)
```

In questo modo interroga tutti i tipi di record contemporaneamente invece che in sequenza. Per 7 tipi di record con 50ms di latenza ciascuno:
- In sequenza: 350ms
- In concorrenza: 50ms

Il comando batch applica questo approccio a più domini (`resolver.py:428-440`).

## Errori comuni e idee sbagliate

**Errore 1: fidarsi delle risposte DNS**
Il DNS non ha un'autenticazione integrata. Senza DNSSEC, le risposte potrebbero essere contraffatte.

**Errore 2: hardcodare gli indirizzi IP per evitare il DNS**
Gli IP cambiano. I servizi cloud usano IP dinamici. Il DNS offre flessibilità.

**Errore 3: ignorare il reverse DNS**
I record PTR aiutano a validare l'identità del server. La loro assenza è sospetta.

**Errore 4: non monitorare le query DNS**
I log delle query DNS rivelano ricognizione, esfiltrazione di dati e traffico C2.

**Errore 5: usare la cache in modo troppo aggressivo**
Dati DNS obsoleti possono persistere a lungo dopo modifiche all'infrastruttura.

## Standard di settore e riferimenti

**Riferimenti OWASP:**
- Testing for DNS Zone Transfer (OTG-INFO-002)
- Testing DNS Spoofing (OTG-INPVAL-007)

**Tecniche MITRE ATT&CK:**
- T1071.004: DNS tunneling per command and control
- T1590.002: ricognizione DNS
- T1584.002: compromissione dell'infrastruttura DNS

**RFC da studiare:**
- RFC 1035: specifica DNS
- RFC 4033-4035: DNSSEC
- RFC 7858: DNS over TLS
- RFC 8484: DNS over HTTPS

Ora vedi `02-ARCHITECTURE.md` per capire come questo tool implementa questi concetti nel codice.
