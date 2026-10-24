<!-- © AngelaMos | 2026 | 01-CONCEPTS.md -->

# Concetti di sicurezza

Questo file tratta la teoria alla base del firewalling in Linux e i problemi
specifici che `fwrule` è progettato per rilevare. Ogni concetto qui illustrato
si ricollega al codice reale del progetto o a incidenti reali in cui qualcuno
ha commesso un errore e ne ha pagato le conseguenze.

---

## Architettura di Netfilter

Ogni firewall Linux di cui hai sentito parlare (iptables, nftables, firewalld, ufw)
è un frontend per lo stesso framework del kernel: **netfilter**. Si trova
all'interno dello stack di rete di Linux e fornisce cinque punti di hook in cui
il kernel può ispezionare, modificare o scartare i pacchetti mentre attraversano
il sistema.

### Flusso dei pacchetti

Quando un pacchetto arriva su una macchina Linux, prende uno di due percorsi
a seconda della tabella di routing. Se l'IP di destinazione appartiene a questa
macchina, il pacchetto va a INPUT. Se la destinazione è altrove e l'inoltro IP
è abilitato (`net.ipv4.ip_forward = 1`), va a FORWARD. Un pacchetto non passa
mai da entrambi.

```
                           NETWORK
                              |
                              v
                     +------------------+
                     |    PREROUTING    |  raw, mangle, nat (DNAT)
                     +------------------+
                              |
                       Routing Decision
                       /            \
                      v              v
              +----------+    +-----------+
              |  INPUT   |    |  FORWARD  |
              | filter,  |    | filter,   |
              | mangle,  |    | mangle,   |
              | security |    | security  |
              +----------+    +-----------+
                   |                |
                   v                v
             Local Process    +------------------+
                   |          |   POSTROUTING    |  mangle, nat (SNAT/MASQ)
                   v          +------------------+
              +----------+           |
              |  OUTPUT  |           v
              | raw,     |       NETWORK
              | mangle,  |
              | nat,     |
              | filter,  |
              | security |
              +----------+
                   |
                   v
              +------------------+
              |   POSTROUTING    |
              +------------------+
                   |
                   v
                NETWORK
```

### I cinque hook

| Hook | Quando scatta | Uso tipico |
|------|---------------|------------|
| PREROUTING | Pacchetto appena arrivato, prima della decisione di routing | DNAT (port forwarding), voce per il connection tracking |
| INPUT | Pacchetto destinato a questa macchina | Filtraggio del traffico in entrata verso i servizi locali |
| FORWARD | Pacchetto in transito (questa macchina funge da router) | Filtraggio tra segmenti di rete |
| OUTPUT | Pacchetto originato da un processo locale | Filtraggio del traffico in uscita |
| POSTROUTING | Pacchetto in procinto di uscire, dopo la decisione di routing | SNAT, masquerading per gateway NAT |

### Tabelle

Netfilter organizza le regole in cinque tabelle, ciascuna con un compito specifico:

| Tabella | Scopo | Chain disponibili |
|---------|-------|-------------------|
| **filter** | Decisioni di accept/drop/reject | INPUT, FORWARD, OUTPUT |
| **nat** | Network Address Translation | PREROUTING, OUTPUT, POSTROUTING |
| **mangle** | Modifica degli header dei pacchetti (TTL, TOS, marcatura) | Tutte e cinque |
| **raw** | Bypass del connection tracking | PREROUTING, OUTPUT |
| **security** | Etichettatura SELinux/AppArmor | INPUT, FORWARD, OUTPUT |

Lo strumento `fwrule` modella tutte e cinque nel suo enum `Table` (`src/models/models.v`),
ma la stragrande maggioranza dei ruleset nel mondo reale risiede in `filter`. È lì
che avvengono le decisioni di accept/drop, ed è lì che le misconfigurazioni causano
violazioni di sicurezza.

La tabella `raw` merita una nota: le regole qui vengono eseguite prima del conntrack,
quindi puoi marcare il traffico ad alto volume (come il DNS su un resolver molto trafficato)
con `NOTRACK` per ignorare completamente il connection tracking. Questo è importante quando
la tabella di conntrack si riempie sui gateway NAT che gestiscono centinaia di migliaia
di connessioni concorrenti. Quando ciò accade, il kernel scarta le nuove connessioni
e si visualizza `nf_conntrack: table full, dropping packet` in dmesg.

---

## iptables vs nftables

### iptables: Lo strumento legacy

`iptables` è stata la CLI standard per firewall in Linux a partire dal kernel 2.4 (2001).
Un dump di `iptables-save` appare così:

```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p tcp --dport 22 -j ACCEPT
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
-A INPUT -j LOG --log-prefix "DROPPED: "
-A INPUT -j DROP
COMMIT
```

Il formato: `*filter` dichiara la tabella, `:INPUT DROP [0:0]` imposta la policy
della chain e i contatori di pacchetti/byte, `-A INPUT` aggiunge una regola in coda,
`-j ACCEPT` è il target di salto (jump), `COMMIT` applica la tabella in modo atomico.

Limitazioni da conoscere:

- Binari separati per IPv4 (`iptables`), IPv6 (`ip6tables`), ARP
  (`arptables`) e bridge filtering (`ebtables`)
- Le regole sono un elenco piatto di condizioni con estensioni di match
  (`-m conntrack`, `-m limit`, `-m multiport`)
- Gli aggiornamenti del ruleset sostituiscono una tabella alla volta, non l'intero
  ruleset in modo atomico

### nftables: Il sostituto

`nftables` ha sostituito iptables a partire da Linux 3.13 (2014). Debian 10+,
RHEL 8+, Fedora 18+ e Ubuntu 20.10+ lo adottano tutti come impostazione predefinita.
Lo stesso ruleset nella sintassi di nftables:

```
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        iifname "lo" accept
        ct state established,related accept
        tcp dport 22 accept
        tcp dport 80 accept
        tcp dport 443 accept
        log prefix "DROPPED: " drop
    }
}
```

### Differenze chiave

| Funzionalità | iptables | nftables |
|--------------|----------|----------|
| IPv4/IPv6 | Binari separati | Unificato (famiglia `inet`) |
| Sintassi | Basata su flag (`-p tcp --dport 22`) | Basata su espressioni (`tcp dport 22`) |
| Atomicità | Per tabella | Intero ruleset in un'unica transazione |
| Set | Nessun supporto nativo | Set e mappe nativi (`{ 22, 80, 443 }`) |
| Azioni multiple | Un target per regola | Concatenazione di più istruzioni |
| Prestazioni | Matching lineare delle regole | I set usano hash lookup (O(1) vs O(n)) |

La sintassi dei set rappresenta un miglioramento concreto. Questa riga di nftables:

```
tcp dport { 22, 80, 443 } accept
```

sostituisce tre regole iptables separate. Il kernel valuta il set con una
ricerca hash invece di scorrere tre regole in modo sequenziale.

### Perché entrambi esistono ancora

nftables include un livello di compatibilità (`iptables-nft`) che traduce
i comandi iptables in regole nftables dietro le quinte. Molte distribuzioni
lo installano di default, per cui l'esecuzione di `iptables` crea in realtà
regole nftables senza che l'utente lo sappia. Questo è il motivo per cui puoi
eseguire `iptables -L` su un sistema moderno e visualizzare le regole, per poi
eseguire `nft list ruleset` e vedere le medesime regole in formato nftables.

Il comando `fwrule export` gestisce la conversione tra i formati, il che è utile
durante la migrazione.

---

## Ordine di valutazione delle regole

### Vince il primo match (First-Match-Wins)

Questo è il concetto in assoluto più importante nella configurazione dei firewall:
**la prima regola che fa match vince**. Il kernel scorre ciascuna regola in ordine,
dall'alto verso il basso. Nel momento in cui un pacchetto corrisponde a una regola
con un target terminale (ACCEPT, DROP, REJECT), la valutazione si interrompe. Il
pacchetto non vede mai le regole rimanenti.

Due ruleset con regole identiche ma in ordine diverso possono presentare proprietà
di sicurezza completamente differenti:

```
Ordering A (secure):                     Ordering B (broken):
1. -s 10.0.0.5 -p tcp --dport 22 -j DROP    1. -p tcp --dport 22 -j ACCEPT
2. -p tcp --dport 22 -j ACCEPT              2. -s 10.0.0.5 -p tcp --dport 22 -j DROP
```

L'ordinamento A blocca SSH da 10.0.0.5, poi permette l'accesso a tutti gli altri.
L'ordinamento B permette SSH da chiunque compreso 10.0.0.5. La regola 2 è codice morto.
Stesse regole, esito di sicurezza opposto.

### Policy delle chain

Ogni chain predefinita (built-in) ha una policy di default che scatta quando nessuna
regola fa match:

```
:INPUT DROP [0:0]     <-- default deny (fail-closed)
:INPUT ACCEPT [0:0]   <-- default accept (fail-open)
```

Default deny significa che qualsiasi cosa tu abbia dimenticato di consentire viene
bloccata. Default accept significa che qualsiasi cosa tu abbia dimenticato di bloccare
riesce a passare. Il comando `fwrule harden` genera sempre `DROP` su INPUT e FORWARD,
e `ACCEPT` su OUTPUT.

### Il problema dello shadowing

Lo shadowing è la misconfigurazione di firewall più comune. Si verifica quando una
regola generica e ampia posta all'inizio della chain impedisce silenziosamente a una
regola più specifica successiva di fare match.

Esamina questo ruleset numerato tratto da `testdata/iptables_conflicts.rules`:

```
Rule  7: -A INPUT -p tcp --dport 22 -j ACCEPT
Rule  8: -A INPUT -s 10.0.0.0/8 -p tcp --dport 22 -j ACCEPT
Rule  9: -A INPUT -p tcp --dport 80 -j ACCEPT
Rule 10: -A INPUT -p tcp --dport 80 -j ACCEPT
Rule 11: -A INPUT -s 192.168.1.0/24 -p tcp --dport 443 -j ACCEPT
Rule 12: -A INPUT -s 192.168.0.0/16 -p tcp --dport 443 -j DROP
```

Cosa succede:

- **La regola 8 è oscurata (shadowed) dalla regola 7.** La regola 7 accetta SSH da qualsiasi
  sorgente. La regola 8 accetta SSH solo da 10.0.0.0/8. Poiché la regola 7 ha già accettato
  tutto il traffico SSH, la regola 8 non potrà mai scattare. La funzione `find_shadowed_rules`
  in `src/analyzer/conflict.v` rileva questa anomalia controllando se i criteri di match della
  regola 7 siano un superset di quelli della regola 8.

- **Le regole 9 e 10 sono duplicati.** Entrambe accettano la porta TCP 80 senza alcuna restrizione
  sulla sorgente. La regola 10 è un peso morto.

- **Le regole 11 e 12 si contraddicono.** 192.168.1.0/24 si trova all'interno di 192.168.0.0/16.
  Gli host in 192.168.1.0/24 fanno match prima con la regola 11 (ACCEPT). Il resto di
  192.168.0.0/16 incontra la regola 12 (DROP). Questo potrebbe essere intenzionale, ma criteri
  sovrapposti con azioni opposte meritano sempre un'attenta revisione. La funzione
  `find_contradictions` lo segnala.

Lo strumento esegue questa analisi effettuando un confronto a coppie su ogni regola in ciascuna
chain. Per ogni coppia (i, j) in cui i < j, chiama
`match_is_superset(rules[i].criteria, rules[j].criteria)`. Quella funzione controlla
protocollo, indirizzo sorgente, indirizzo di destinazione, porte, interfacce e stati di
conntrack. Se ogni campo della regola precedente racchiude la regola successiva, quest'ultima
è shadowed.

---

## Connection Tracking (conntrack)

### Stateful vs Stateless

Senza il connection tracking, un firewall è stateless. Valuta ogni pacchetto
in modo isolato, senza alcuna memoria dei pacchetti precedenti. Se consenti il traffico
in entrata verso la porta 80, hai anche bisogno di una regola separata per consentire
ai pacchetti di risposta di uscire sulle porte temporanee/effimere (1024-65535). Questa
rappresenta un'enorme superficie di attacco.

Il connection tracking risolve questo problema. Il kernel mantiene una tabella di ogni
connessione attiva (memorizzata in `/proc/sys/net/netfilter/nf_conntrack_max`, tipicamente
65536 voci di default, ciascuna delle quali consuma circa 300-400 byte di memoria del kernel).
Ogni flusso tracciato viene classificato in uno stato.

### I quattro stati

| Stato | Significato | Esempio |
|-------|-------------|---------|
| **NEW** | Primo pacchetto di una connessione | TCP SYN, primo datagramma UDP |
| **ESTABLISHED** | Parte di un flusso bidirezionale | Qualsiasi cosa dopo lo scambio SYN/SYN-ACK |
| **RELATED** | Nuova connessione originata da una esistente | Canale dati FTP, risposte di errore ICMP |
| **INVALID** | Non associabile ad alcuna connessione nota | Pacchetto corrotto, sequenza TCP fuori finestra (out-of-window) |

### Perché ESTABLISHED,RELATED deve trovarsi vicino all'inizio

Osserva lo schema standard tratto da `testdata/iptables_basic.rules`:

```
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -p tcp --dport 22 -j ACCEPT
```

Su un server molto trafficato, oltre il 90% dei pacchetti appartiene a connessioni stabilite.
Se la regola di conntrack si trova in posizione 2, quei pacchetti fanno match immediatamente
e saltano tutto il resto. Se la collochi in fondo alla posizione 10, ogni pacchetto stabilito
deve passare attraverso 9 regole prima di fare match. Si tratta di migliaia di valutazioni
di regole non necessarie al secondo sotto carico.

La funzione `find_missing_conntrack` in `src/analyzer/optimizer.v` rileva due problemi:
chain prive di una regola ESTABLISHED/RELATED (warning), e chain in cui la regola esiste
ma è posizionata oltre il terzo slot (suggerimento informativo di spostarla più in alto).

### Connessioni RELATED

RELATED è meno ovvio di ESTABLISHED ma ugualmente importante. Due scenari:

**Canali dati FTP:** L'FTP usa la porta 21 per il controllo e una connessione separata
per il trasferimento dati. Il modulo helper `nf_conntrack_ftp` del kernel monitora il
canale di controllo, vede il comando PORT o PASV e marca la connessione dati risultante
come RELATED. Senza RELATED nella tua regola di conntrack, i trasferimenti dati FTP
si interrompono anche se la porta 21 è aperta.

**Errori ICMP:** Quando un pacchetto viene scartato da qualche parte nella rete, il router
che scarta invia indietro un messaggio ICMP "destination unreachable" o "time exceeded".
Questi pacchetti ICMP sono RELATED rispetto alla connessione originale. Senza RELATED,
la tua macchina non riceve mai questi messaggi di errore, il che compromette la path
MTU discovery e rende il debug di rete molto più difficile.

---

## Default Deny vs Default Accept

Questo è il principio del privilegio minimo applicato al traffico di rete.

**Default deny** (l'unica policy sensata per la produzione):
```
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
```

Si costruisce una whitelist. Ogni servizio che deve essere raggiungibile ottiene una
regola ACCEPT esplicita. Qualsiasi cosa tu abbia dimenticato rimane bloccata. Se qualcuno
aggiunge un nuovo servizio sulla macchina senza aggiungere una regola firewall, il servizio
è irraggiungibile. Questo è fastidioso, ma sicuro. Lo noti e lo correggi.

**Default accept** (pericoloso):
```
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
```

Si costruisce una blacklist. Si prova a bloccare tutto ciò che è malevolo e si spera
di non aver dimenticato nulla. Quando qualcuno installa MySQL sulla macchina e questo
effettua il bind su 0.0.0.0:3306, risulta immediatamente raggiungibile dall'intera rete
Internet perché non hai mai inserito una regola per bloccarlo. Potresti non accorgertene
per mesi.

La differenza tra i due si riduce a cosa accade quando qualcosa va storto. Default deny
fallisce in chiusura / fail-closed (la direzione sicura). Default accept fallisce in
apertura / fail-open (la direzione pericolosa). Il Cloud Threat Report 2023 di Palo Alto
Unit 42 ha rilevato che il 76% delle organizzazioni aveva SSH esposto pubblicamente in almeno
un ambiente cloud, quasi sempre a causa di configurazioni equivalenti al default-accept
sui security group.

---

## Violazioni nel mondo reale

### Capital One (2019)

Nel marzo 2019, un ex dipendente di AWS ha sfruttato una vulnerabilità di Server-Side
Request Forgery (SSRF) in un WAF mal configurato che proteggeva l'infrastruttura AWS
di Capital One. Il WAF aveva un ruolo IAM con permessi eccessivi e le regole del
firewall consentivano all'istanza compromessa di raggiungere il servizio di metadati EC2
all'indirizzo 169.254.169.254. L'attaccante ha interrogato l'endpoint dei metadati per
ottenere credenziali IAM temporanee, le ha usate per elencare e scaricare bucket S3,
e ha esfiltrato i dati poiché il traffico in uscita non era limitato.

Una singola regola di firewall in uscita (egress) avrebbe bloccato l'esfiltrazione:

```
-A OUTPUT -d 169.254.169.254/32 -j DROP
```

Impatto: 100 milioni di richieste di credito esposte, 140.000 codici di Social Security,
80.000 numeri di conto corrente bancario. Capital One ha pagato una multa di 80 milioni
di dollari all'OCC e 190 milioni di dollari in accordi transattivi. (United States v. Paige A.
Thompson, Case No. CR19-159, W.D. Wash. 2019.)

### Imperva (2019)

Imperva ha reso noto un incidente di sicurezza in cui un'istanza interna di database
aveva un security group AWS mal configurato che consentiva l'accesso non autorizzato.
L'istanza esposta avrebbe dovuto essere isolata a livello di rete, ma le regole del suo
security group consentivano connessioni in ingresso che non avrebbero dovuto permettere.
Un attaccante ha ottenuto chiavi API dall'istanza e le ha usate per accedere ai dati dei clienti.
La causa principale è stata un security group eccessivamente permissivo su un'istanza che non
aveva mai bisogno di connettività esterna.

Questo è l'esatto schema che `fwrule` contrassegna come "overly permissive" (eccessivamente permissivo):
una regola che fa match sulla sorgente 0.0.0.0/0 su una porta che dovrebbe essere riservata a una
subnet interna.

### Bollettino di sicurezza NSA sui firewall IPsec VPN (U/OO/179891-20)

La National Security Agency ha pubblicato linee guida specificamente dedicate alle regole di firewall
mal configurate attorno all'infrastruttura VPN. Il bollettino ha documentato come gli avversari
sfruttino regole eccessivamente permissive sui gateway VPN per ottenere l'accesso iniziale a una rete,
per poi utilizzare la medesima segmentazione errata per muoversi lateralmente tra zone di rete che
dovrebbero essere isolate. La criticità specifica: regole del firewall che consentono al traffico VPN
di raggiungere subnet interne senza limitare quali servizi interni siano accessibili, trasformando
la VPN in un pass gratuito attraverso il perimetro.

### Equifax (2017, CVE-2017-5638)

La causa principale è stata una vulnerabilità non corretta da patch in Apache Struts, ma la violazione
è stata peggiorata in modo esponenziale da carenze del firewall e della rete. Un certificato SSL scaduto
su un dispositivo di monitoraggio di rete ha fatto sì che l'ispezione del traffico cifrato smettesse di
funzionare per 19 mesi senza che nessuno se ne accorgesse. Una segmentazione di rete configurata male ha
permesso all'attaccante di muoversi lateralmente attraverso i sistemi per 76 giorni dopo la compromissione
iniziale, accedendo a 48 database contenenti le informazioni di 147 milioni di persone. La combinazione
di mancata applicazione di patch, assenza di monitoraggio e assenza di segmentazione ha trasformato una
singola vulnerabilità di un'applicazione web in una delle più grandi violazioni di dati della storia.
Il costo finale ha superato gli 1,4 miliardi di dollari.

### Networking predefinito di Docker/Kubernetes

Non si tratta di una singola violazione ma di un'estesa categoria di misconfigurazioni.
La rete bridge predefinita di Docker inserisce regole iptables direttamente nella chain
FORWARD e nella chain PREROUTING della tabella nat. Queste regole scavalcano i firewall
a livello host come UFW e firewalld, poiché le regole di Docker vengono valutate prima
delle regole del firewall host nella chain.

Cosa significa questo in pratica: configuri UFW su un host Docker e aggiungi regole per
bloccare la porta 3306. Docker pubblica un container MySQL sulla porta 3306. UFW segnala
che la porta è bloccata. La porta è in realtà aperta verso Internet perché le regole iptables
di Docker nella chain FORWARD accettano il traffico prima ancora che questo raggiunga le regole
di UFW.

```
Packet arrives
     |
     v
PREROUTING (Docker DNAT rule matches, rewrites destination)
     |
     v
FORWARD chain
     |
     +-> Docker's ACCEPT rule fires here  <-- UFW never sees this packet
     |
     +-> UFW's rules (never reached)
```

Kubernetes presenta lo stesso problema su larga scala. `kube-proxy` genera regole iptables
o nftables per ogni oggetto Service. Su un cluster con centinaia di servizi, possono esserci
migliaia di regole generate che nessun essere umano ha mai scritto o revisionato. Queste regole
interagiscono con il firewall dell'host in modi che non risultano evidenti osservando
esclusivamente la configurazione di Kubernetes oppure la configurazione del firewall host
da sola.

---

## Errori comuni di configurazione dei firewall

Questi sono i pattern specifici che `fwrule analyze` e `fwrule optimize` rilevano.
Ciascuno di essi corrisponde a una funzione in `src/analyzer/conflict.v` o
`src/analyzer/optimizer.v`:

- **Regole shadowed** (`find_shadowed_rules`): Un ACCEPT generico prima di un
  DENY specifico rende il DENY irraggiungibile. Severità: CRITICAL.

- **Conntrack mancante** (`find_missing_conntrack`): Nessuna regola ESTABLISHED/RELATED
  significa che ogni pacchetto attraversa l'intera chain. Su un server occupato,
  questo impatto è misurabile sul consumo di CPU. Severità: WARNING.

- **Nessun rate limiting su SSH** (`find_missing_rate_limits`): Porta 22 aperta con un
  semplice ACCEPT. Un attaccante esegue hydra con migliaia di tentativi di password
  al minuto. Un limite di 3/minuto con burst 5 rende il brute force impraticabile.
  Severità: WARNING.

- **Regole duplicate** (`find_duplicates`): Due regole con criteri di match identici
  e la stessa azione. La seconda è un peso morto che rende l'auditing più difficile.
  Severità: WARNING.

- **Regole contraddittorie** (`find_contradictions`): Criteri di match sovrapposti
  con azioni opposte (ACCEPT vs DROP). Potrebbe essere intenzionale, ma richiede
  una revisione umana. Severità: WARNING.

- **Policy predefinita di accept**: La policy della chain è ACCEPT, quindi qualsiasi cosa
  non esplicitamente bloccata riesce a passare. Questa è la singola misconfigurazione
  più comune sui server esposti a Internet.

- **Regole ridondanti** (`find_redundant_rules`): Una regola ristretta con la stessa
  azione di una regola più ampia che già la copre. Non è un rischio per la sicurezza,
  ma disordine che offusca la policy effettiva. Severità: INFO.

- **Logging mancante** (`find_missing_logging`): Una chain con una policy DROP ma
  senza alcuna regola LOG. Il traffico scartato non genera alcuna traccia di audit,
  il che rende l'incident response e la digital forensics notevolmente più difficili.
  Severità: INFO.
