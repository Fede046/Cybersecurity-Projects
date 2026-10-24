<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# zingela: Panoramica

zingela è uno scanner di porte di massa stateless, a line rate. Gli fornisci un intervallo di indirizzi e porte, lui invia una singola probe costruita ad hoc a ciascuno e classifica ogni risposta come open, closed o filtered senza mai mantenere una tabella delle connessioni. È scritto in Zig 0.16, viene distribuito come un singolo binario statico senza libpcap e senza libgmp, ed è il discendente diretto di due tool: masscan (Robert Graham, 2013) e zmap (Durumeric, Wustrow e Halderman, USENIX Security 2013).

Questa cartella insegna come funziona, dalla teoria della sicurezza fino ai byte sul cavo.

## Perché esiste lo scanning stateless

Un normale client TCP tiene traccia di ogni connessione: un socket, una macchina a stati, un timer di ritrasmissione, un buffer di ricezione. Va bene per qualche migliaio di connessioni. Crolla alla scala di internet. Per sondare tutti i 3,7 miliardi di indirizzi IPv4 instradabili su una porta, uno scanner stateful avrebbe bisogno di miliardi di strutture socket attive e della relativa contabilità del kernel. Non è fattibile su una sola macchina.

Il trucco stateless consiste nel portare lo stato dentro il pacchetto stesso. zingela codifica l'identità di ogni probe nel numero di sequenza iniziale TCP usando un hash con chiave. Quando arriva una risposta, il numero di acknowledgement riporta indietro quell'identità, e un singolo ricalcolo dell'hash dimostra che la risposta è davvero la reazione a una probe inviata da zingela. Nessuna tabella, nessun timer, nessuna memoria per target. Il lato di trasmissione e il lato di ricezione non condividono quasi nulla e girano a pieno regime.

È lo stesso design che nel 2013 permise a zmap di scansionare l'intero spazio IPv4 pubblico in circa 45 minuti da una singola macchina, un risultato che trasformò lo scanning a livello di internet da curiosità di ricerca a strumento di misura quotidiano.

## Il posizionamento onesto

Leggi questa sezione prima di credere a qualsiasi dichiarazione sulla velocità, nostra o di chiunque altro.

Su hardware Linux standard, masscan, zmap e zingela si scontrano tutti con lo stesso muro: il percorso di trasmissione `AF_PACKET` del kernel raggiunge il tetto intorno a 1,5-2,5 milioni di pacchetti al secondo su un singolo core. Ognuno di questi tool incontra quel limite. Non esiste un numero di throughput grezzo di cui vantarsi a parità di hardware, e zingela non ne rivendica nessuno.

L'unica strada oltre quel limite in masscan è proprietaria: PF_RING ZC, che richiede una licenza ntop a pagamento, un modulo kernel out-of-tree e specifiche NIC Intel. Le sue cifre di punta (10 Mpps nel README, di più nelle demo con doppia NIC) sono tutte PF_RING, mai un kernel standard. La strada di zingela è `AF_XDP`, presente nel kernel Linux mainline dalla versione 4.18, senza licenza e senza modulo proprietario. Quel backend è disponibile già oggi dietro il flag di build `-Dxdp`. Il confronto diretto completo con masscan su hardware 10GbE è lavoro futuro, e questi documenti non citeranno mai un line rate che non sia stato misurato su hardware reale.

Quindi la vittoria difendibile non è "più veloce". È: accelerazione senza paywall, memory safety dimostrata sotto i controlli ReleaseSafe di Zig, un singolo binario statico, una correttezza che masscan non ha mai risolto in modo pulito, e un'interfaccia moderna. La [sezione sulle prestazioni di MECHANICS](MECHANICS.it.md) mostra i numeri misurati dell'hot path e spiega perché la CPU non è mai il collo di bottiglia.

## Cosa funziona oggi

- SYN scan TCP su IPv4 e IPv6, usando un ring di trasmissione raw con kernel bypass.
- Scan UDP con payload specifici per protocollo e un reporting onesto open contro filtered.
- Un connect scan TCP che non richiede privilegi, per ambienti blindati o cloud.
- Rilevamento automatico degli hypervisor che scartano silenziosamente gli invii raw, con fallback alla modalità connect.
- Rilevamento di servizi e banner sulle porte aperte, opt-in e in due fasi.
- Una suite di stealth ed evasion, protetta da un riconoscimento esplicito di autorizzazione.
- Statelessness basata su cookie SipHash, permutazione degli indirizzi tramite gruppo ciclico, controllo del rate con token bucket e un'esclusione non scavalcabile degli intervalli di indirizzi riservati.
- Una dashboard live truecolor e output NDJSON adatto a grep.

## Avvio rapido

```
curl -fsSL https://angelamos.com/zingela/install.sh | bash
zingela scan --target 192.0.2.0/24 --ports 80,443 --rate 20000
```

L'installer mette `zingela` nel tuo PATH e assegna le capability per i raw socket, così gira senza sudo. Se sei su una macchina dove non puoi o non dovresti inviare pacchetti raw, aggiungi `--connect` per lo scanner non privilegiato.

Il target mostrato, `192.0.2.0/24`, è un intervallo riservato alla documentazione (RFC 5737) che zingela salta per design, quindi riporta zero target. Sostituiscilo con un intervallo che possiedi o che sei autorizzato a scansionare.

## Prerequisiti per la lettura

Trarrai il massimo da questi documenti se hai familiarità con il three-way handshake TCP, la forma degli header IPv4 e TCP e l'aritmetica modulare di base. Non hai bisogno di conoscere Zig; le spiegazioni passo passo citano funzioni e file, non dettagli del linguaggio. Dove un concetto ha un famoso esempio reale, lo ancoriamo lì.

## Il resto di questa cartella

| Documento | Cosa tratta |
|---|---|
| [01-CONCEPTS.md](01-CONCEPTS.it.md) | la teoria della sicurezza: statelessness, SYN cookie, cosa può osservare una scansione, scanning responsabile e gli incidenti che hanno plasmato tutto questo |
| [02-ARCHITECTURE.md](02-ARCHITECTURE.it.md) | il design del sistema: il modello a due motori, la mappa dei moduli e la scala dei backend, con diagrammi |
| [03-IMPLEMENTATION.md](03-IMPLEMENTATION.it.md) | un percorso guidato attraverso il codice, modulo per modulo |
| [MECHANICS.md](MECHANICS.it.md) | l'approfondimento a livello di byte e di matematica: il cookie, il checksum, il gruppo ciclico e le prestazioni misurate |
| [04-CHALLENGES.md](04-CHALLENGES.it.md) | modi per estendere zingela, da un vero benchmark AF_XDP a nuovi moduli di probe |
