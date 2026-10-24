<!-- ©AngelaMos | 2026 -->
<!-- 00-OVERVIEW.md -->

# rveng: Panoramica

rveng è una piattaforma interattiva per imparare il reverse engineering. Ti
consegna un vero binario compilato, ti pone una domanda concreta su di esso
("con quale numero confronta il check?"), ti dà un hex viewer, un disassembler,
una mappa delle sezioni e uno scanner di stringhe nel browser per rispondere, e
valuta la tua risposta. Solo quando hai ragione rivela il sorgente C originale,
così colleghi il codice macchina che hai appena letto al codice che lo ha
prodotto.

È un unico motore di analisi Python, senza framework, con tre volti: una web app,
una API HTTP e (per costruzione) una libreria integrabile. Il motore fa il parsing
di ELF, disassembla x86-64 con capstone, risolve gli import tramite la PLT,
costruisce grafi del flusso di controllo, trova le funzioni nei binari stripped e
verifica le risposte alle challenge. Il layer FastAPI è un sottile adapter in
sola lettura sopra di esso. La app React è il volto con cui interagisce chi
impara.

Questa cartella insegna come funziona il tutto, dalla postura di sicurezza che lo
rende sicuro fino agli offset dei byte nel binario di esempio.

## L'unica regola che dà forma a tutto: nessuna esecuzione

rveng non esegue mai un binario. Ogni operazione consiste nel leggere e fare il
parsing di byte:

- Hex dump, parsing dell'header ELF, attraversamento delle sezioni, lettura dei
  simboli, scansione delle stringhe: tutte pure letture di byte.
- Il disassemblaggio è decodifica, non esecuzione. capstone legge i byte delle
  istruzioni e ne restituisce la forma testuale. Non trasferisce mai il controllo
  al codice decodificato.
- Le challenge di patch sono valutate con un diff statico dei byte rispetto a un
  target patchato di riferimento. Il binario patchato non viene mai eseguito.

Una "web app che analizza binari" ingenua invita gli utenti a caricare eseguibili
che il server poi esegue, il che significa dover mettere in sandbox per sempre
codice nativo non fidato. rveng evita del tutto quella superficie d'attacco. I
binari delle challenge sono selezionati e precompilati dall'autore e distribuiti
come asset statici. Se nulla viene mai eseguito, non c'è alcuna superficie di
esecuzione di codice arbitrario, nessuna sandbox da cui evadere e nessun percorso
di esaurimento delle risorse attraverso un binario ostile. Questo vincolo è
portante, non una preferenza. Aggiungere una funzionalità che esegue un binario
richiederebbe un design separato che mantenga questa postura.

## Funzionalità

- Un parser ELF64 scritto a mano: header, tabella delle sezioni, tabella dei
  simboli, tutti letti direttamente dai byte grezzi e verificati nei test
  confrontandoli con pyelftools.
- Disassemblaggio x86-64 tramite capstone, in sintassi Intel, con annotazione per
  ogni istruzione di confronti, salti condizionali, target delle call e
  riferimenti a dati RIP-relative.
- Risoluzione degli import tramite la PLT: una `call 0x401050` viene etichettata
  `atoi`, letta da `.plt`, `.rela.plt`, `.dynsym` e `.dynstr` come farebbe il
  loader.
- Cross-reference (chi chiama questa funzione, quali dati tocca) e un grafo del
  flusso di controllo a basic block per una singola funzione.
- Scoperta delle funzioni nei binari stripped tramite scansione delle sezioni
  eseguibili alla ricerca del prologo di funzione standard, così un binario a cui
  è stata rimossa la tabella dei simboli resta comunque navigabile.
- Sei challenge valutabili su un unico binario di esempio, che coprono le cinque
  competenze fondamentali di RE, con valutazione solve-then-reveal.
- Progressi salvati in SQLite dietro una piccola interfaccia, così può essere
  sostituito in modo pulito con lo storage proprio di un'applicazione host.
- Una web app React e una FastAPI in sola lettura, servite insieme per il
  self-hosting con un solo comando.

## Avvio rapido

rveng è self-hosted. Clonalo, avvialo con Docker, apri localhost. Non c'è nulla da
configurare, nessun account, nessun segreto, nessun servizio esterno.

```
cd PROJECTS/advanced/rveng
just up
# open http://localhost:8790
```

Questo fa la build della app React, la fa servire da nginx ed esegue il motore
FastAPI dietro di essa. Per sviluppare invece con hot-reload:

```
just dev-up
# open http://localhost:8791
```

Per eseguire i test del motore e il type check del frontend senza alcun server:

```
just test        # uv run pytest -q
just typecheck   # cd frontend && pnpm typecheck
```

## Il binario di esempio

Ogni esempio in questa documentazione usa `gate`, un piccolo programma compilato
con `gcc -no-pie -fno-stack-protector -O0`. Legge un numero da `argv`, lo
confronta con `1337` in una funzione chiamata `check` e stampa una stringa
segreta quando il numero corrisponde. Quell'unico binario basta per insegnare la
lettura esadecimale, lo scheletro ELF, il recupero di simboli e stringhe, la
lettura di un gate disassemblato, la patch del gate, e rifare tutto da capo una
volta rimossi i simboli.

## Dove andare dopo

- `01-CONCEPTS.md` per la teoria del reverse engineering e per capire perché la
  sola analisi statica ti porta molto lontano.
- `02-ARCHITECTURE.md` per il design un-motore-tre-volti e per come una richiesta
  lo attraversa.
- `03-IMPLEMENTATION.md` per un percorso attraverso ogni modulo del motore
  applicato al binario di esempio.
- `04-CHALLENGES.md` per le sei challenge, cosa insegna ciascuna e come
  aggiungerne di tue.
