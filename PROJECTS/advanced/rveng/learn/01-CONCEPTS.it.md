<!-- ©AngelaMos | 2026 -->
<!-- 01-CONCEPTS.md -->

# Concetti di Reverse Engineering

Fare reverse engineering di un binario è la pratica di ricostruire cosa fa un
programma a partire dai byte che vengono eseguiti, senza il codice sorgente. Lo
fai quando il sorgente non c'è: analizzare malware, fare l'audit di un driver
closed-source, capire una vulnerabilità a partire da una patch distribuita,
verificare se un'immagine firmware fa ciò che il vendor dichiara. Questo
documento copre i concetti che rveng insegna, ancorati al modo in cui lavorano i
veri analisti.

## Analisi statica contro analisi dinamica

Ci sono due modi per studiare un binario.

- L'**analisi dinamica** lo esegue e osserva: un debugger, una sandbox, un
  tracer. Vedi il comportamento reale, ma devi eseguire codice potenzialmente
  ostile, e il codice può rilevare la sandbox e mentire.
- L'**analisi statica** legge i byte senza eseguirli: fa il parsing del formato
  del file, disassembla le istruzioni, legge le stringhe e i simboli. Nulla viene
  eseguito, quindi nulla può attaccarti o nascondersi rifiutandosi di girare.

rveng è interamente statico, e non si tratta di un modo per aggirare un limite.
Una grande quantità di reverse engineering reale è statica. L'esempio famoso più
chiaro: nel 2017 il ransomware WannaCry fu rallentato in tutto il mondo quando un
analista trovò all'interno del campione un nome di dominio hardcoded che il
malware controllava prima di diffondersi, e registrò quel dominio. Quel dominio
era un "killswitch" che viveva nel binario come semplice stringa, il tipo di
artefatto che recuperi leggendo il file invece di tirare a indovinare. Leggere
stringhe e flusso di controllo da un file è una competenza di prim'ordine, ed è
la competenza che rveng valuta.

## Che cos'è davvero un binario

Un programma compilato su Linux è un file ELF (Executable and Linkable Format, il
formato della System V ABI). Non è un blob. Ha una struttura rigida e
documentata:

```
+---------------------+  offset 0
| ELF header          |  magic, class, entry point, where the tables are
+---------------------+
| program headers     |  how the loader maps the file into memory (segments)
+---------------------+
| .text   (code)      |  the machine instructions
| .rodata (constants) |  string literals, read-only data
| .data   (globals)   |  writable initialized data
| ...                 |
+---------------------+
| section headers     |  a table describing every section above
+---------------------+
| .symtab / .strtab   |  function and variable names (removed when stripped)
+---------------------+
```

L'header ELF all'offset 0 è la mappa per tutto il resto. I suoi primi quattro
byte sono sempre `7f 45 4c 46` (`\x7fELF`). Un campo, `e_entry`, contiene
l'indirizzo virtuale in cui inizia l'esecuzione. Un altro, `e_shoff`, punta alla
tabella degli header di sezione. Imparare a leggere a mano quegli offset fissi è
`02-elf-anatomy`, e il motore che lo fa è `elf.py`.

## Indirizzi, offset e perché differiscono

Due sistemi di coordinate attraversano ogni binario, e confonderli è l'errore più
comune dei principianti.

- Un **offset nel file** è una posizione nel file su disco. Il byte numero 0x1154
  nel file.
- Un **indirizzo virtuale** è dove un byte finisce in memoria una volta che il
  loader ha mappato il file. L'indirizzo 0x401154 a runtime.

Sono legati dalla sezione che contiene il byte: `vaddr = file_offset -
section.offset + section.addr`. Il campione `gate` è compilato `-no-pie`, il che
significa che non è position-independent e i suoi indirizzi sono fissati al
momento del link, quindi i numeri in questa documentazione sono stabili e puoi
ragionarci direttamente. Gli eseguibili position-independent (il default
moderno) vengono caricati a una base casuale, e lavoreresti invece con offset
rispetto a quella base. rveng usa `-no-pie` di proposito, così gli indirizzi
didattici non si spostano mai.

## I simboli, e il mondo senza di essi

Un **simbolo** è un nome associato a un indirizzo: la funzione `check` si trova a
`0x401146`, la entry `main` a `0x401164`. I simboli vivono in `.symtab`, e i nomi
in `.strtab`. Esistono per aiutare linker e debugger, non per eseguire il
programma, quindi una build di release o un malware di solito li **rimuove**
(strip): la sezione `.symtab` viene eliminata. Il codice gira in modo identico. È
solo anonimo.

Lo stripping è il motivo per cui il reverse engineering reale è difficile, e il
motivo per cui rveng ha una challenge stripped. Quando `check` non ha più un
nome, non puoi cercarlo. Devi trovare le funzioni in un altro modo (cercando il
prologo con cui inizia la maggior parte delle funzioni), leggere direttamente il
disassemblato e riconoscere il `cmp` contro una costante come il gate. È
`discover.py` che trova `sub_401146` dove una tabella dei simboli avrebbe detto
`check`.

L'unica cosa che lo stripping non rimuove sono le informazioni di linking
dinamico. Un programma che chiama `printf` ha comunque bisogno del meccanismo di
`.dynsym` e `.plt` per trovare `printf` nella libc al momento del caricamento.
Quindi anche in un binario stripped le chiamate a funzioni di libreria possono
essere recuperate per nome. È ciò che fa `plt.py`.

## La PLT: come una chiamata esterna riottiene un nome

Quando `gate` chiama `atoi`, il compilatore non sa dove si troverà `atoi` in
memoria, quindi chiama un piccolo stub nella Procedure Linkage Table (`.plt`).
Quello stub salta attraverso uno slot della Global Offset Table (`.got.plt`) che
il loader riempie. Il legame tra "questo stub PLT" e "il nome `atoi`" è
memorizzato nella tabella di rilocazione `.rela.plt`, che punta in `.dynsym`, i
cui nomi vivono in `.dynstr`.

```
call 0x401050          the code calls a PLT stub
  0x401050: jmp [rip+X]   the stub jumps through a GOT slot at address G
.rela.plt: G -> dynsym[i] the relocation says slot G binds symbol index i
.dynsym[i].name -> .dynstr the symbol's name is "atoi"
```

Percorrere quella catena trasforma `call 0x401050` in `call atoi`. Il motore fa
esattamente questo in `plt.py`, ed è il motivo per cui il pannello del
disassemblato può etichettare le chiamate importate invece di mostrare indirizzi
nudi.

## Leggere un gate in assembly

Il cuore della maggior parte delle challenge stile crackme è un confronto che
alimenta un salto condizionale. In `gate`, la funzione `check` contiene:

```
cmp   DWORD PTR [rbp-0x4], 0x539     ; compare the input against 0x539 (1337)
jne   <fail path>                    ; if not equal, take the fail branch
<success path>
```

`0x539` è `1337` in decimale, e capstone in sintassi Intel lo stampa come
`0x539`. Riconoscere che il numero magico è `1337`, letto dal `cmp`, è
`05-disassembly`. Riconoscere che invertire il `jne` (byte di opcode `75`) in modo
che il salto non venga mai preso forza il percorso di successo è `03-patching`.
La competenza in entrambi i casi è leggere l'intento dalle istruzioni.

## Patching: il comportamento è fatto solo di byte

Un salto condizionale `jne` è costituito dai due byte `75 07` in questo binario.
Sovrascrivili con `90 90` (due no-op `nop`) e il salto sparisce, così il
controllo cade direttamente nel percorso di sblocco indipendentemente
dall'input. Questo è il binary patching: cambi il comportamento modificando
byte, senza ricompilare. È il meccanismo dietro i crack software storici, dietro
l'hot-patching legittimo e dietro il micro-patching di una funzione vulnerabile
senza distribuire una build completamente nuova.

rveng valuta una patch senza eseguirla. Ogni challenge di patch include il
binario originale e un target patchato di riferimento. I byte che invii vengono
applicati all'originale all'offset indicato, e il risultato viene confrontato con
il target di riferimento tramite un diff dei byte. Uguale significa corretto.
Questo valuta esattamente la competenza (produrre la modifica giusta) con zero
esecuzione. È il trucco in `patch.py`.

## Il ciclo solve-then-reveal

La pedagogia che rveng conserva dal suo predecessore è semplice e rigorosa:

1. Una challenge ti dà un binario e una missione concreta.
2. Usi gli strumenti per arrivare a una risposta partendo solo dal binario.
3. Invii. Il motore la valuta.
4. Solo una risposta corretta rivela il sorgente C originale.

Rivelare il sorgente solo dopo una risposta corretta è ciò che rende il ciclo
didattico. Devi arrivare alla risposta dalle prove a livello macchina, poi puoi
vedere che avevi ragione e perché. Questo è possibile solo perché ogni risposta è
verificabile da una macchina, il che costringe ogni challenge in una di tre
categorie di valutazione:

- **found-value**: individua un numero o una stringa nel binario e inviala. Il
  numero magico `1337`, o la stringa `the_flag_is_here`. I numeri sono
  normalizzati, quindi `0x539`, `539h` e `1337` corrispondono tutti.
- **identified-symbol**: dai il nome di una funzione, sezione o simbolo. Rispondi
  `check`, confrontato senza distinzione tra maiuscole e minuscole.
- **patched-bytes**: modifica dei byte per cambiare il comportamento, valutata
  tramite diff statico rispetto a un target di riferimento.

Ognuna delle sei challenge rientra in una di queste tre, e quella mappatura è il
ponte tra "il reverse engineering come mestiere" e "il reverse engineering che
una macchina può valutare". Il prossimo documento mostra come il sistema è
costruito attorno a questo.
