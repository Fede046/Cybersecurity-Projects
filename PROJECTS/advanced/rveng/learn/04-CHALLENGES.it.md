<!-- ©AngelaMos | 2026 -->
<!-- 04-CHALLENGES.md -->

# Challenge

Sei challenge su un unico binario. Coprono le cinque competenze fondamentali del
reverse engineering, e la sesta toglie le rotelle di supporto rimuovendo i
simboli. Ognuna viene valutata, e ognuna rivela il proprio sorgente solo quando
hai ragione. Le risposte sono riportate qui perché questo è il documento
didattico; risolvile prima partendo dal binario, poi verifica.

Il campione `gate` legge un numero da `argv`, lo confronta con `1337` in una
funzione chiamata `check` e stampa il segreto `the_flag_is_here` quando il numero
corrisponde. Tutte e sei le challenge sono quell'unico programma, visto da
angolazioni diverse.

## Le cinque competenze, mappate sulla valutazione

Ogni challenge appartiene a una di tre categorie di risposta verificabili da una
macchina. Quella mappatura è ciò che permette a una macchina di valutare un
mestiere:

| # | Challenge | Modulo | Competenza | Categoria |
|---|-----------|--------|-------|----------|
| 01 | Leggi l'hex | hex-reading | leggere un dump, trovare un valore | found-value |
| 02 | Trova l'entry point | elf-anatomy | leggere l'header ELF | found-value |
| 03 | Inverti il gate | patching | cambiare i byte, cambiare il comportamento | patched-bytes |
| 04 | Dai un nome alla funzione | strings-symbols | leggere la tabella dei simboli | identified-symbol |
| 05 | Trova il gate | disassembly | leggere l'assembly | found-value |
| 06 | Gate al buio | disassembly (stripped) | lavorare senza simboli | found-value |

## 01: Leggi l'hex

Missione: una stringa segreta si trova nei dati in sola lettura del binario. Apri
l'hex viewer e leggi la colonna ASCII per recuperarla.

Risposta: `the_flag_is_here`, all'offset nel file `0x2004` in `.rodata`.

Cosa insegna: le due metà di un hex dump. Le colonne esadecimali sono rumore per
un essere umano; la colonna ASCII a destra è dove una stringa diventa leggibile.
Questa è la prima mossa su qualsiasi file sconosciuto, ed è gratuita.

## 02: Trova l'entry point

Missione: ogni file ELF registra dove inizia l'esecuzione. Leggi l'header e trova
l'entry point, il campo `e_entry`.

Risposta: `0x401060` (decimale `4198496`). Sono accettate entrambe le forme.

Cosa insegna: l'header ELF è la mappa dell'intero file, ed `e_entry` è un campo
fisso all'offset `0x18`. Imparare che l'header è una tabella rigida che puoi
leggere a mano, invece che magia, è la base per tutto il resto.

## 03: Inverti il gate

Missione: il check all'offset nel file `0x1154` è un salto condizionale (`jne`,
byte `75 07`) che salta il percorso di sblocco. Patcha quei due byte in modo che
il salto non venga mai preso e il percorso di sblocco venga sempre eseguito.
Invia i due byte sostitutivi in esadecimale.

Risposta: `9090` (due `nop`), applicati all'offset `0x1154`.

Cosa insegna: il comportamento è fatto di byte. Un singolo salto condizionale
decide l'esito, e neutralizzarlo con due `nop` forza il percorso di successo. Il
valutatore applica i tuoi byte all'originale e confronta il risultato con un
target patchato di riferimento tramite un diff statico, quindi vieni valutato
sulla produzione della modifica esatta, e nulla viene mai eseguito.

## 04: Dai un nome alla funzione

Missione: una funzione decide se il gate si apre. Leggi la tabella dei simboli e
dalle un nome.

Risposta: `check` (a `0x401146`, dimensione 30, in `.symtab`). Confrontato senza
distinzione tra maiuscole e minuscole.

Cosa insegna: i simboli sono nomi associati a indirizzi, e quando sono presenti
ti consegnano la mappa gratis. Questo è il percorso facile che la challenge
stripped in seguito ti toglie.

## 05: Trova il gate

Missione: questo binario confronta un numero con un valore magico. Disassembla la
funzione di check e trova la costante con cui confronta.

Risposta: `1337` (il `cmp ..., 0x539`). Accettato come `1337`, `0x539` o `539h`.

Cosa insegna: leggere l'intento dalle istruzioni. Il numero magico non è una
stringa e non è un simbolo; è un operando immediato dentro un `cmp`, e l'unico
modo per arrivarci è leggere il disassemblato di `check`.

## 06: Gate al buio

Missione: questo binario è stripped, quindi non ci sono nomi di funzione da
cercare. Scopri le funzioni dal loro prologo, disassembla quella che controlla un
numero e leggi la costante con cui confronta.

Risposta: `1337`, come nella challenge 05, ma raggiunta senza tabella dei
simboli.

Cosa insegna: il caso del mondo reale. Malware e build di release sono stripped,
quindi la ricerca dei simboli di `04` e il disassemblaggio per nome di `05`
falliscono entrambi qui. Trovi il codice dalla sua forma (il prologo
`push rbp; mov rbp, rsp`), disassembli per indirizzo grezzo fino al `ret` e
riconosci lo stesso gate. Questo è il reverse engineering quando il binario non
cerca di aiutarti.

## Estendere la piattaforma

Aggiungere una challenge significa aggiungere una directory di asset, nessuna
modifica al codice. Ogni challenge è una cartella sotto `challenges/` con tre
file:

```
challenges/07-your-challenge/
  challenge.json   id, module, title, mission, and the answer spec
  target           the compiled binary to analyze
  source.c         the source revealed on a correct answer
```

L'oggetto `answer` in `challenge.json` sceglie una delle tre categorie di
valutazione:

```
{ "category": "found_value",      "expected": 1337 }
{ "category": "found_value",      "expected": "the_flag_is_here" }
{ "category": "identified_symbol","name": "check" }
{ "category": "patched_bytes",    "offset": 4436, "patch": "9090" }
```

Il loader valida la directory all'avvio, e una singola challenge malformata viene
saltata con un warning invece di far crashare l'intero set. Da lì, idee che si
adattano al motore esistente senza alcuna nuova capacità:

- Una challenge che chiede quale sezione contiene una data stringa, valutata come
  identified-symbol sul nome della sezione.
- Una patch multi-byte che cambia una costante di confronto invece di rimuovere
  un salto.
- Un binario con una funzione esca, così la scoperta delle funzioni restituisce
  più di un candidato e chi impara deve scegliere il vero gate dal disassemblato.
- Una challenge found-value su un riferimento a dati RIP-relative, così chi
  impara deve calcolare un indirizzo assoluto da un'istruzione come fa
  `disasm.py`.

Idee che richiedono nuovo lavoro sul motore, e sarebbero veri e propri progetti:
una seconda architettura (ARM64 tramite le altre modalità di capstone), un volto
di analisi dinamica dietro un design separato in sandbox che preservi la postura
di non-esecuzione di questo backend, o una vista a grafo con layout automatico
sopra il grafo del flusso di controllo esistente.
