<!--
©AngelaMos | 2026
04-CHALLENGES.md
-->

# Sfide di estensione

Scegline una e consegnala come PR.

## Principiante

### 1. Aggiungi un quinto rotator: PostgreSQL `ALTER USER`
Un nuovo file `src/cre/rotators/postgres_user.cr` che ruota la password di un ruolo Postgres direttamente tramite `ALTER USER ... PASSWORD ...`. Usa il contratto in 4 passi esistente; verifica aprendo una nuova connessione con la nuova password. Inserisci il file - la macro lo registra automaticamente. Aggiungi gli spec unitari in `spec/unit/rotators/postgres_user_spec.cr`.

### 2. Subscriber notificatore per Slack
Replica `TelegramSubscriber` usando `chat.postMessage` di Slack. Aggiungi l'allowlist dei chat ID, la parametrizzazione del channel-id e la formattazione dei messaggi. Un unico nuovo file in `src/cre/notifiers/`.

### 3. Aggiungi `notify_via :slack` all'enum Channel
Una volta che hai un notificatore Slack, aggiungi `Slack` a `Channel` in `src/cre/policy/policy.cr`, gestiscine il dispatch nell'evaluator e scrivi una policy in `policies/` che lo usi.

## Intermedio

### 4. Dashboard web tramite SSE
Aggiungi `src/cre/web/` con un server HTTP Lucky/Kemal che si iscrive al bus e invia gli eventi come Server-Sent Events a una dashboard HTMX. Riusa `Tui::State` come modello dati - ha già la forma giusta. Il senso dell'architettura bus + plugin è che questo è un *nuovo subscriber*, non una riscrittura.

### 5. Wrap ibrido ML-KEM per la KEK
Aggiungi `algorithm_id = 0x03` all'envelope encryption: Curve25519 + ML-KEM-768 (Kyber) ibrido per il wrap della DEK. Fornisce forward secrecy contro futuri attacchi quantistici sui ciphertext catturati. Nota: ML-KEM è presente in OpenSSL 3.x; potresti dover aggiornare i binding FFI di LibCrypto.

### 6. Ancoraggio con OpenTimestamps
Ancora ogni Merkle root di `audit_batches` alla blockchain di Bitcoin tramite OpenTimestamps. Aggiunge un quarto livello di integrità: anche se l'intero DB e la chiave di firma sono compromessi, un auditor offline con un full node Bitcoin può verificare quando esisteva ciascun batch. Aggiorna `src/cre/audit/batch_sealer.cr` per pubblicare le prove OTS insieme alle firme Ed25519.

## Avanzato

### 7. Rotator di workload identity SPIFFE/SPIRE
Aggiungi un rotator che *sostituisce* le credenziali statiche con gli SVID SPIFFE (X.509 + JWT). Dimostra lo spostamento dell'industria post-2024, che abbandona del tutto la rotazione a favore di identità effimere basate sull'attestazione. Punti di intervento: nuovo client in `src/cre/spiffe/`, nuovo rotator in `src/cre/rotators/spiffe.cr`, nuovo tipo di credenziale in `src/cre/domain/credential.cr`.

### 8. Macchina a stati per il recupero dai crash
L'orchestrator marca già i fallimenti del passo di commit come `Inconsistent` ed emette un alert critico, quindi i fallimenti parziali di un singolo passo emergono in modo evidente. Ciò che ancora manca è la procedura di recupero *all'avvio*: se il daemon è stato terminato, ad esempio, tra un `apply` riuscito e l'inizio di `verify`, la riga in `rotations` resta nello stato `Verifying` senza nessuna fiber che la faccia avanzare. Implementa il protocollo di recupero: all'avvio, scansiona `rotations` cercando stati non terminali; per ciascuno, decidi in base a `(rotator_kind, last_step)` se invocare `rollback_apply`, ritentare dal passo fallito o passare a `Inconsistent`. Aggiungi `src/cre/engine/recovery.cr` con una semantica esplicita da macchina a stati, collegalo a `Engine#start` e aggiungi uno spec con fixture SQL in `spec/unit/engine/recovery_spec.cr` che costruisca ogni tipo di riga "stale" e verifichi l'esito corretto.

### 9. Supporto multi-tenant
Collega `tenant_id` end-to-end: una nuova migration aggiunge la colonna su `credentials` / `credential_versions` / `rotations` / `audit_events`; la costruzione dell'AAD in `RotationOrchestrator#persist_credential_version` si estende a `cred=<id>|kind=<k>|tenant=<t>`; i matcher delle policy guadagnano `c.tenant == "tenant-x"`; e le policy di row-level security di Postgres impongono l'isolamento a livello di DB. La decisione di design più importante: KEK per tenant (ogni tenant ruota in modo indipendente, ma il bootstrap deve destreggiarsi tra N chiavi) oppure KEK condivisa con DEK legate al tenant (configurazione più semplice, è l'AAD a fare il lavoro di isolamento).

### 10. Broker di credenziali JIT
Sostituisci completamente il contratto di rotazione per alcuni tipi di credenziale: invece di ruotare, *emetti* una nuova credenziale effimera a ogni accesso (TTL di 5-15 minuti). Richiede una nuova astrazione `Broker` accanto a `Rotator`, l'integrazione con AWS STS / impersonation dei service account GCP / Vault dynamic, e la logica di refresh del token lato consumer nelle app di esempio.

## Griglia di valutazione (per l'autovalutazione)

Una ottima PR di estensione:
- Ha test unitari in `spec/unit/<area>/` che rispecchiano la struttura dei sorgenti
- Ha un messaggio di commit mirato che spiega il *perché*, non solo il *cosa*
- Non rompe i test esistenti (`crystal spec spec/` deve essere verde)
- Introduce modifiche in 1-3 file; se tocca più di 5 file, probabilmente l'astrazione è sbagliata
- Fa emergere onestamente le modalità di fallimento (ad es. il broker JIT dovrebbe documentare chiaramente la complessità che sposta lato consumer)
