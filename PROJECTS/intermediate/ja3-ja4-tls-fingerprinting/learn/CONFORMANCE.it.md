<!-- ©AngelaMos | 2026 -->
<!-- CONFORMANCE.md -->

# JA3/JA4 TLS Fingerprinting: conformità

Un fingerprint è utile solo se concorda con quello di tutti gli altri. Un JA4 calcolato da questo tool deve essere uguale al JA4 calcolato dall'implementazione di riferimento di FoxIO per lo stesso handshake, altrimenti un hash in un feed condiviso non significa nulla. Questo documento indica esattamente a cosa è conforme ciascun fingerprint, il vettore pubblicato a cui è vincolato nella test suite, e ogni punto in cui il tool restringe deliberatamente il proprio perimetro. Dove si dice "vincolato", esiste un known-answer test che fallisce se l'output si discosta.

## Specifiche di riferimento

| Fingerprint | Autorità | Documento |
|-------------|-----------|----------|
| JA3 / JA3S | Salesforce (2017) | l'originale [TLS Fingerprinting with JA3 and JA3S](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/) |
| JA4 / JA4S | FoxIO (2023) | [JA4](https://github.com/FoxIO-LLC/ja4/blob/main/technical_details/JA4.md), [JA4S](https://github.com/FoxIO-LLC/ja4/blob/main/technical_details/JA4S.md) |
| JA4H | FoxIO | [JA4H](https://github.com/FoxIO-LLC/ja4/blob/main/technical_details/JA4H.md) |
| JA4X | FoxIO | [JA4X](https://github.com/FoxIO-LLC/ja4/blob/main/technical_details/JA4X.md) |
| JA4T / JA4TS | FoxIO | [JA4T](https://github.com/FoxIO-LLC/ja4/blob/main/technical_details/JA4T.md) |
| Gestione di GREASE | IETF | [RFC 8701](https://www.rfc-editor.org/rfc/rfc8701.html) |
| Chiavi QUIC v1 | IETF | [RFC 9001](https://www.rfc-editor.org/rfc/rfc9001.html) Sezione 5.2, Appendice A |
| Chiavi QUIC v2 | IETF | [RFC 9369](https://www.rfc-editor.org/rfc/rfc9369.html) Sezione 3.3.1, Appendice A |

Le specifiche JA4+ di FoxIO hanno una propria licenza (FoxIO License 1.1), che disciplina la *specifica*, non questa sua implementazione indipendente. Questo tool è distribuito sotto AGPL 3.0.

## Conformità per fingerprint

### JA3 / JA3S

- **È conforme** all'ordine dei campi originale di Salesforce e all'MD5 dei campi decimali uniti da virgole.
- **È vincolato** ai due vettori client del post di Salesforce e a un vettore server, in `ja3.rs` (`salesforce_client_vector_one` produce `ada70206e40642a3e4461f35503241d5`; `salesforce_client_vector_two_empty_fields` copre il caso dei campi vuoti; `server_vector_round_trips_through_md5` copre JA3S).
- **Perimetro.** Calcolato dal ClientHello e dal ServerHello. GREASE viene eliminato secondo RFC 8701. JA3 viene mantenuto nonostante sia instabile sui browser moderni perché i feed pubblici di malware sono espressi con esso; vedi [01-CONCEPTS.md](./01-CONCEPTS.it.md).

### JA4 / JA4S

- **È conforme** alla costruzione JA4 di FoxIO: un prefisso leggibile di 10 caratteri, uno SHA-256 troncato della lista ordinata dei cipher e uno SHA-256 troncato della lista ordinata delle estensioni, con SNI e ALPN rimossi e i signature algorithm aggiunti nell'ordine originale.
- **È vincolato** ai vettori di sezione di FoxIO in `ja4.rs`: la sezione dei cipher `8daaf6152771` e la sezione delle estensioni `e5627efa2ab1`, che insieme al prefisso formano il canonico `t13d1516h2_8daaf6152771_e5627efa2ab1`. Un ClientHello completamente vuoto produce l'hash zero `000000000000` (`empty_input_is_the_zero_hash`), e viene verificato che il troncamento sia esattamente di 12 caratteri hex (`truncation_is_twelve_hex_chars`).
- **Perimetro.** Vengono prodotte sia la forma hashed sia quella raw; la CLI stampa la forma hashed ed espone la forma raw con `--json`. La version word è presa da `supported_versions`, non dal campo legacy congelato. JA4S calcola l'hash delle estensioni del server nell'ordine di trasmissione, non ordinate.

### JA4H

- **È conforme** alla costruzione JA4H di FoxIO su una singola richiesta HTTP in chiaro.
- **È vincolato** al parsing della request line e degli header in `ja4h.rs` (`parses_request_line_and_headers`, `version_codes`).
- **Perimetro.** Solo HTTP/1.x in chiaro. Le richieste HTTPS sono cifrate e gli header HTTP/2 sono compressi con HPACK, quindi nessuno dei due viene sottoposto a fingerprinting in modo passivo. Questa è una proprietà dell'osservazione passiva, non una limitazione dell'implementazione.

### JA4X

- **È conforme** alla costruzione JA4X di FoxIO: tre hash SHA-256 troncati sugli OID dell'emittente, sugli OID del soggetto e sugli OID delle estensioni, ciascuno nell'ordine DER.
- **È vincolato** ai test di estrazione degli OID in `ja4x.rs` (`extracts_issuer_oids_in_order`, `extracts_extension_oids`) ed è irrobustito dall'harness di fuzzing basato su property test in `tests/ja4x.rs`.
- **Perimetro.** JA4X passivo funziona su **TLS 1.2 e precedenti**, dove il messaggio Certificate è in chiaro; TLS 1.3 lo cifra. Il lettore DER (`der.rs`) è limitato: un certificato che termina a metà di un campo restituisce un errore o una sezione vuota, mai un panic (`certificate_ending_after_subject_is_handled_without_panic`). Una slice fuori dai limiti su un certificato costruito ad arte è un bug di sicurezza, non estetico, ed è per questo che questo percorso è sottoposto a fuzzing.

### JA4T / JA4TS

- **È conforme** al formato JA4T di FoxIO: window size, option kind, MSS e window scale, uniti da underscore.
- **È vincolato** ai vettori di FoxIO in `ja4t.rs` (`foxio_windows_default_vector` produce `64240_2-1-3-1-1-4_1460_8`; `foxio_windowed_vector`; `missing_mss_and_scale_report_zero`) e al decoder che percorre le opzioni del SYN in `pipeline/decode.rs` (`ja4t_walk_reproduces_the_windows_default_vector`, `ja4t_walk_counts_trailing_end_of_list_padding`, `ja4t_walk_survives_truncated_options`).
- **Perimetro.** Letto da un SYN (JA4T) o da un SYN-ACK (JA4TS). Un'opzione MSS o window scale mancante viene riportata come `0`, come da specifica. Il walker delle opzioni tollera liste di opzioni troncate e con padding.

## Conformità di QUIC

- **È conforme** a RFC 9001 (QUIC v1) e RFC 9369 (QUIC v2) per la derivazione delle chiavi dell'Initial del client e la rimozione della protezione del pacchetto.
- **È vincolato** ai vettori di chiavi dell'Appendice A delle RFC in `quic.rs`: `client_initial_keys_match_rfc9001_appendix_a1` e `client_initial_keys_match_rfc9369_appendix_a` derivano la chiave, l'IV e la chiave di header protection dal connection ID dell'appendice e li verificano byte per byte. La capture di integrazione `quic-with-several-tls-frames.pcapng` produce lo stabile `q13d0310h3_55b375c5d22e_cd85d2d88918`.
- **Perimetro e un confine deliberato.** Il tool decifra solo i pacchetti **Initial del client**, derivando le chiavi dal Destination Connection ID di ciascun pacchetto, senza alcun segreto del server e senza rilevamento della direzione. Non traccia la connection migration, non decifra i pacchetti 0-RTT o Handshake, e non riassembla attraverso un cambio di connection ID. Un campo di versione diverso da v1 o v2 incrementa `quic_version_unsupported` e viene saltato invece di essere indovinato. La decifratura completa del *pacchetto protetto* v2 non è vincolata a un vettore di pacchetto completo pubblicato (l'unica trascrizione disponibile aveva subito una corruzione dei byte durante il trasferimento); il **key schedule** v2 è vincolato all'Appendice A di RFC 9369, e il percorso di decifratura è lo stesso AEAD esercitato dal test sul pacchetto completo v1.

## Confini di parsing e robustezza

L'engine vieta `unsafe`, quindi quanto segue sono garanzie, non aspirazioni:

- **Una lunghezza malformata è un errore, mai una lettura fuori dai limiti.** Ogni parser legge attraverso il `Reader` con bounds-checking; un campo che dichiara più byte di quelli rimanenti restituisce `ParseError`. Le test suite `parse` e `reassembly` forniscono input troncati e sovrapposti e verificano che producano errori puliti.
- **Una capture ostile costa una quantità di memoria limitata.** La flow table limita il numero di flow tracciati, li elimina allo scadere di un idle timeout, e pone tetti sia ai byte contigui sia a quelli fuori ordine parcheggiati per ciascuna direzione. Uno stream che raggiunge un limite viene marcato come `capped` e smette di crescere. I contatori espongono `segments_dropped` e `unfinished_tls_streams`.
- **Un pacchetto QUIC che fallisce la verifica del tag AEAD viene scartato.** Il plaintext non verificato non viene mai passato al parser TLS.

## Non-obiettivi deliberati

Queste sono scelte, registrate in modo che non vengano scambiate per lacune:

- **Nessuna decifratura dei dati applicativi.** Il tool legge solo ciò che è in chiaro sulla rete. Non ha alcuna chiave privata e non vede mai i certificati TLS 1.3 o gli header HTTP/2.
- **Nessun probing attivo.** Il tool non invia nulla. Non può essere rilevato osservando eventuali scansioni.
- **Nessuna attribuzione.** Un fingerprint identifica una toolchain, non un attore. Il JA3 `72a589da586844d7f0818ce684948eea` condiviso da alcuni campioni di Emotet e Cobalt Strike ne è l'esempio permanente: stessa libreria TLS, malware diversi. Un verdetto `known_bad` significa "toolchain nota come sospetta", nient'altro.
- **Ancora nessun fingerprinting di DTLS.** Il marcatore di trasporto `d` esiste nel codice JA4, ma la cattura degli handshake DTLS non è collegata alla pipeline. Questa è l'estensione più naturale; vedi [04-CHALLENGES.md](./04-CHALLENGES.it.md).

## Come verificare di nuovo

```bash
cargo test --workspace        # every pinned vector above runs here
cargo test -p tlsfp-core quic  # just the QUIC key-schedule vectors
```

Rispetto a una seconda implementazione, cattura un handshake e confronta:

```bash
tlsfp pcap capture.pcapng --json | jq '.ja4.hash, .ja4.raw'
tshark -r capture.pcapng -T fields -e tls.handshake.ja4   # Wireshark's JA4
```

Se le due non concordano su una capture reale, si tratta di un bug che merita una segnalazione, e la forma raw nel JSON è il punto da cui iniziare a leggere.
