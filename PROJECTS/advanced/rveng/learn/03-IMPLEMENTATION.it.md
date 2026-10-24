<!-- ©AngelaMos | 2026 -->
<!-- 03-IMPLEMENTATION.md -->

# Implementazione

Un percorso attraverso il motore, modulo per modulo, applicato al binario di
esempio `gate`. Ogni indirizzo e offset riportato sotto è reale e riproducibile:
`gate` è compilato con `gcc -no-pie -fno-stack-protector -O0`, quindi nulla si
sposta. Il codice è citato per nome di funzione, mai per numero di riga, perché i
numeri di riga invecchiano.

## elf.py: leggere il formato del file dai byte grezzi

`elf.py` implementa a mano il parser ELF64 con `struct`. Non usa una libreria per
leggere il formato, di proposito: lo scopo è imparare dove si trova ogni campo. I
test poi confrontano il risultato fatto a mano con pyelftools
(`test_matches_pyelftools`), quindi pyelftools è un oracolo di correttezza nella
test suite, non una dipendenza su cui il motore si appoggia a runtime.

`parse_header` è il punto di ingresso. Rifiuta tutto ciò che non è ELF64:

- I primi quattro byte devono essere `\x7fELF` (`ELF_MAGIC`).
- Il byte 4 (`EI_CLASS`) deve essere `ELFCLASS64`.
- Il byte 5 (`EI_DATA`) seleziona l'endianness, trasformata in un prefisso
  `struct` (`<` per little-endian, che è ciò che usa x86-64).

Poi legge offset fissi direttamente dall'header: `e_entry` a `0x18`, `e_shoff` a
`0x28`, `e_shnum` a `0x3C`, e così via. Per `gate` questo produce
`e_entry = 0x401060`, `e_shoff = 0x36a0` (13984), `e_shnum = 30`. `_validate_
section_table` poi controlla i limiti della tabella delle sezioni, così un file
malformato solleva `NotAnElf` invece di leggere fuori dai limiti.

`parse_sections` legge tutti gli `e_shnum` header di sezione, poi risolve il nome
di ciascuna sezione. I nomi non sono memorizzati inline; ogni header contiene un
offset `name` in una sezione di stringhe dedicata, e l'indice di quell'header è
`e_shstrndx`. Il parser legge i byte di quella sezione una volta sola ed estrae
ogni nome come stringa C. È così che un indice senza nome diventa `.text`,
`.rodata`, `.symtab`.

`parse_symbols` trova la sezione `.symtab`, segue il suo campo `link` fino alla
tabella delle stringhe corrispondente e legge ogni voce di simbolo da 24 byte:
offset del nome, valore (l'indirizzo del simbolo), dimensione e un byte `info`
compattato il cui nibble basso è il tipo e il nibble alto è il binding. Per
`gate`, questo recupera `check` a `0x401146` con dimensione 30, e `main` a
`0x401164`. `ElfImage` avvolge tutto questo e offre `section(name)`,
`symbol(name)` e `functions()` (simboli il cui tipo è `STT_FUNC`).

L'unica nota di design che vale la pena interiorizzare: una sezione può essere
`SHT_NOBITS` (`.bss`), il che significa che occupa memoria ma nessun byte nel
file. `Section.file_bytes` restituisce un valore vuoto per quelle, così nulla
cerca di leggere contenuto del file che non c'è.

## hex.py: il dump canonico

`hex.py` produce lo stesso layout di `xxd`: un offset di 8 cifre esadecimali,
sedici byte in esadecimale divisi in due gruppi da otto, poi la colonna ASCII.
`HexLine.ascii_
gutter` stampa un byte come carattere solo quando è nell'intervallo stampabile da
`0x20` a `0x7E`, e un `.` altrimenti. Quella colonna è tutta la competenza di
`01-hex-reading`: la stringa segreta `the_flag_is_here` è invisibile nelle
colonne esadecimali ma evidente nella colonna ASCII, e si trova in `.rodata`
all'offset nel file `0x2004`.

## strings.py: sequenze stampabili

`extract` percorre i byte, accumulando una sequenza ogni volta che vede un byte
stampabile e scaricando la sequenza come `FoundString` quando si interrompe, se
la sequenza ha raggiunto la lunghezza minima. `extract_in_section` limita questo
ai byte di una sola sezione mantenendo gli offset relativi al file, così puoi
chiedere specificamente le stringhe in `.rodata`. Questa è la vittoria a basso
costo del reverse engineering: prima di disassemblare qualsiasi cosa, leggi le
stringhe, e spesso ne salta fuori una password o una stringa di formato.

## patch.py: modificare il comportamento senza eseguire

Tre funzioni, tutte pure:

- `apply(original, offset, new_bytes)` restituisce un nuovo buffer con la
  modifica scritta dentro, con controllo dei limiti e mantenendo la lunghezza.
  Una patch non cambia mai la dimensione del file.
- `diff(a, b)` restituisce le differenze byte per byte tra due buffer di uguale
  lunghezza.
- `verify_patch(original, offset, submitted, known_good)` applica l'invio e
  restituisce se il risultato è uguale al target di riferimento.

Questo è il meccanismo che valuta una challenge di patch senza alcuna
esecuzione. Per il gate, il `jne` all'offset nel file `0x1154` è costituito dai
byte `75 07`. Sovrascriverli con `90 90` produce esattamente il target patchato
di riferimento, quindi `verify_patch` restituisce true. Nulla viene eseguito;
vengono confrontati due buffer.

## disasm.py: decodificare x86-64 con capstone

`_new_engine` configura capstone per x86 a 64 bit in sintassi Intel con la
modalità detail attiva. La sintassi Intel è quella in cui la costante viene
stampata come `0x539`; l'impostazione è `CS_OPT_SYNTAX_INTEL`. La modalità detail
è necessaria perché il motore ispeziona gli operandi di ogni istruzione, non solo
il suo testo.

Ogni istruzione decodificata diventa una dataclass `Instruction` che porta con sé
il suo indirizzo, il mnemonico, la stringa degli operandi, i byte grezzi e tre
annotazioni da cui dipendono i layer superiori:

- `immediate`: l'operando immediato di un'istruzione che non è di flusso, così
  `cmp [rbp-4],
  0x539` espone `0x539`. Viene estratto da `_immediate`, che legge l'operando
  `X86_OP_IMM`.
- `branch_target`: per un'istruzione di flusso di controllo (un salto
  condizionale, `call` o `jmp`), l'immediato è l'indirizzo di destinazione invece
  di un valore di dati. Il motore instrada l'immediato verso `branch_target` per
  le operazioni di flusso e verso `immediate` per tutto il resto, così i due non
  vengono mai confusi.
- `rip_target`: per un operando di memoria RIP-relative, l'indirizzo assoluto a
  cui punta, calcolato da `_rip_target` come `instruction address + instruction
  size + displacement`. È così che `lea rax, [rip+X]` che carica la stringa
  segreta si risolve in `0x402004`.

Ci sono tre modi per disassemblare:

- `disassemble_symbol` usa l'indice di sezione, il valore e la dimensione di un
  simbolo per ritagliare esattamente i byte di quella funzione. Questo è il
  percorso pulito quando i simboli esistono.
- `disassemble_at` disassembla a partire da un indirizzo virtuale grezzo fino al
  primo `ret`. Questo è il percorso per un binario stripped, dove hai un
  indirizzo (dalla scoperta) ma nessuna dimensione.
- `disassemble_text` decodifica l'intera sezione `.text` per l'analisi
  dell'intero binario, come le cross-reference.

`find_gate` restituisce la prima istruzione di confronto che porta un immediato,
che per `gate` è il `cmp ..., 0x539`. È la macchina che legge il gate nello
stesso modo in cui lo fa chi impara.

## plt.py: restituire i nomi alle chiamate importate

Un binario stripped o linkato dinamicamente chiama le funzioni della libc
attraverso stub PLT. `plt.py` ricostruisce la mappatura da stub a nome come
farebbe il loader:

- `_dynamic_names` legge `.dynsym` e risolve il nome di ogni simbolo dinamico da
  `.dynstr`.
- `_got_to_name` legge `.rela.plt`. Ogni rilocazione ha un indirizzo GOT di
  destinazione (`r_offset`) e un `r_info` compattato i cui 32 bit alti sono
  l'indice del simbolo dinamico. Mappa ogni slot GOT al nome importato.
- `_stub_got_slot` trova il `jmp [rip+disp]` all'interno di una voce PLT da 16
  byte (l'opcode è `ff 25`) e calcola attraverso quale slot GOT salta, come
  `entry
  address + jump offset + 6 + displacement`.
- `plt_map` mette tutto insieme: per ogni stub PLT, trova il suo slot GOT e cerca
  il nome legato a quello slot.

Per `gate` questo produce esattamente `0x401030 -> puts`, `0x401040 -> printf`,
`0x401050 -> atoi`. È per questo che il disassemblato di `main` può mostrare
`call atoi` invece di `call 0x401050`.

## xref.py e cfg.py: struttura sopra il flusso di istruzioni

`xref.py` trasforma le istruzioni decodificate in riferimenti. Ogni
`branch_target` diventa un riferimento di chiamata o di salto, ogni `rip_target`
diventa un riferimento a dati. `build_xrefs` li raggruppa per l'indirizzo a cui
puntano, così puoi chiedere "chi fa riferimento a `check`?" e `xrefs_to` filtra
su un singolo target. Eseguirlo sul campione trova `main` che chiama `check`.

`cfg.py` costruisce un grafo del flusso di controllo per una singola funzione
usando il classico algoritmo dei leader:

1. `_leaders` marca i confini dei basic block. Un blocco inizia alla prima
   istruzione, a ogni istruzione immediatamente successiva a un terminatore (un
   salto condizionale, `jmp` o `ret`) e a ogni target di salto che cade
   all'interno della funzione.
2. `build_cfg` taglia il flusso di istruzioni in corrispondenza di quei leader in
   `BasicBlock`.
3. Li collega: un salto condizionale emette un arco `taken` verso il suo target e
   un arco `fallthrough` verso il blocco successivo; un `jmp` emette un arco
   `jump`; un blocco semplice prosegue in fallthrough; un `ret` non emette nulla.

Per `check`, questo produce il diamante che ti aspetteresti: un blocco di
ingresso con `cmp`/`jne`, un arco taken verso il percorso di fallimento e un arco
fallthrough verso il percorso di successo.

## discover.py: trovare funzioni senza simboli

Quando un binario è stripped, `functions()` non restituisce nulla, quindi la
navigazione ha bisogno di un altro punto di ancoraggio. `discover_functions`
scansiona ogni sezione eseguibile alla ricerca del prologo di funzione standard
`55 48 89 e5`, che è `push rbp; mov rbp, rsp`, l'impostazione del frame con cui
inizia la maggior parte delle funzioni `-O0`. Ogni corrispondenza diventa una
`Discovered
Function` etichettata `sub_<address>`. Su `gate_stripped` questo riscopre
l'ingresso di `check` a `0x401146` anche se il suo nome non c'è più, e da lì
subentra il percorso di disassemblaggio per indirizzo. È esattamente così che
lavori su un vero binario stripped: niente nomi, quindi trovi il codice dalla sua
forma.

## challenge.py: il valutatore e la rivelazione

`challenge.py` contiene le tre specifiche di risposta (`FoundValue`,
`IdentifiedSymbol`, `PatchedBytes`) e la funzione `grade` che smista in base a
quale delle tre usa una challenge:

- Una risposta numerica `FoundValue` viene valutata tramite `normalize_int`, che
  accetta `0x539`, `539h` e `1337` e confronta come interi. Una `FoundValue`
  stringa viene confrontata dopo aver rimosso gli spazi e convertito in
  minuscolo.
- Un `IdentifiedSymbol` viene confrontato senza distinzione tra maiuscole e
  minuscole con il nome noto.
- Un invio `PatchedBytes` viene valutato da `verify_patch` rispetto al target di
  riferimento.

Il comportamento portante è alla fine di `grade`: restituisce il sorgente della
challenge solo quando la risposta è corretta. Una risposta sbagliata restituisce
`revealed_source =
None`. Quel singolo condizionale è la pedagogia solve-then-reveal, e funziona
perché ogni risposta è verificabile prima della rivelazione. Il layer API non
deve mai sapere come funziona la valutazione; chiama `grade` e restituisce
qualunque sorgente (se c'è) ne esca.
