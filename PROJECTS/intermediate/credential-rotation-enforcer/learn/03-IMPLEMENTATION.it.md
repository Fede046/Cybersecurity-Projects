<!--
©AngelaMos | 2026
03-IMPLEMENTATION.md
-->

# Walkthrough dell'implementazione

Questo documento ti indirizza verso i percorsi di codice più importanti. Leggilo con `tree src/` aperto in un'altra finestra.

## Il DSL delle policy (validato staticamente e al momento della registrazione)

`src/cre/policy/dsl.cr` dichiara il DSL all'interno di `module CRE::Policy::Dsl`. Chi lo usa deve includerlo esplicitamente:

```crystal
require "cre/policy/dsl"
include CRE::Policy::Dsl

policy "production-aws-secrets" do
  match { |c| c.kind.aws_secretsmgr? && c.tag(:env) == "prod" }
  max_age 30.days
  enforce :rotate_immediately
  notify_via :telegram, :structured_log
end
```

`with builder yield` rende ogni metodo del Builder invocabile senza receiver all'interno del blocco. Si applicano due tipi di rilevamento dei refusi:

- **A compile time** — `enforce :rotate_immediatly` (singolo argomento Symbol → enum `Action`) viene rifiutato dall'autocast di Crystal. `match { |c| c.kund }` (refuso su un getter di Credential) rompe la compilazione indicando il file della policy.
- **Al momento della registrazione** — `notify_via :telegrm, :slak` usa un overload splat-Symbol che esegue `Channel.parse?` su ciascun valore e solleva `BuilderError("unknown channel 'telegrm' in policy '<name>'")` quando il file viene caricato. L'autocast degli splat non arriva fino ai Symbol, quindi questo livello è la migliore alternativa possibile.

`Builder#build` solleva inoltre `BuilderError` quando mancano campi obbligatori (`matcher`, `max_age`, `enforce_action`). In ogni caso, una policy malformata non raggiunge mai un daemon in esecuzione.

## L'Event Bus (fanout tramite i channel di Crystal)

`src/cre/engine/event_bus.cr` espone `subscribe(buffer:, overflow:)` che restituisce un `Channel(Event)`. Il metodo `run` avvia una singola fiber dispatcher che legge da `@inbox` e inoltra sul channel di ciascun subscriber.

`dispatch` usa il `select` di Crystal per entrambe le modalità di overflow:
- `Drop` — `select … else` scarta l'evento quando il buffer è pieno e registra un warn.
- `Block` — `select … when timeout(@block_send_timeout)` attende fino a `@block_send_timeout` (default 5s) e solo allora scarta, registrando lo stallo. Questo isola il bus da un subscriber bloccato: il head-of-line blocking è limitato.

## La registrazione dei plugin Rotator

`src/cre/rotators/rotator.cr` dichiara una base astratta con un `REGISTRY = {} of Symbol => Rotator.class` a livello di classe. La macro `register_as` lo popola a compile time:

```crystal
abstract class Rotator
  REGISTRY = {} of Symbol => Rotator.class

  macro register_as(kind)
    ::CRE::Rotators::Rotator::REGISTRY[{{ kind }}] = self
  end
end
```

Quando un file come `src/cre/rotators/aws_secrets.cr` viene richiesto (require), la riga `register_as :aws_secretsmgr` viene eseguita a *compile time* e la classe compare in `Rotator::REGISTRY[:aws_secretsmgr]`. Nessuna lista centrale da mantenere.

## L'orchestrator in 4 passi

`src/cre/engine/rotation_orchestrator.cr` esegue il contratto:

```
generate -> rotator-specific (often produces the new value + cloud version_id)
apply    -> rotator-specific (often no-op for cloud rotators where generate already exposed)
verify   -> read back, byte-equal check
commit   -> promote new -> AWSCURRENT, demote old -> AWSPREVIOUS
```

Ogni passo pubblica sul bus `RotationStepStarted` e poi `RotationStepCompleted` oppure `RotationStepFailed`.

La gestione dei fallimenti ha due regimi:

- **apply o verify falliscono** — `rotator.rollback_apply(c, new_secret)` annulla la modifica lato cloud; la rotazione passa a `Failed`; il bus emette `RotationStepFailed` + `RotationFailed`.
- **commit fallisce** — sequenze di commit parziali su più chiamate (ad es. `UpdateSecretVersionStage` di AWS che restituisce 5xx a metà strada) non possono essere annullate in modo affidabile lato client. La rotazione passa a `Inconsistent` (uno stato terminale distinto da `Failed`) e l'orchestrator emette un `AlertRaised` critico così gli operatori sanno di dover intervenire.

Il percorso di successo è ora quello più pesante: quando i quattro passi si completano, l'orchestrator
1. sigilla `new_secret.ciphertext` con il `Crypto::Envelope` opzionale (AES-256-GCM, AAD = `cred=<id>|kind=<k>`),
2. inserisce una riga in `credential_versions` con la DEK cifrata (wrapped) + la versione della KEK,
3. aggiorna la riga della credenziale per incrementare `last_rotated_at`, impostare `current_version_id` sull'id della nuova versione e retrocedere quella vecchia a `previous_version_id`.

Quest'ultimo passo è ciò che impedisce al policy evaluator di ri-pianificare la stessa rotazione a ogni tick — `Policy#overdue?` si basa su `c.rotation_anchor` (che è `last_rotated_at || created_at`), non su `updated_at`.

L'orchestrator non chiama mai direttamente l'audit. L'audit avviene automaticamente perché `AuditSubscriber` è sul bus in ascolto proprio di questi tipi di evento — l'orchestrator non può dimenticarsi di registrare. E poiché il percorso dell'orchestrator è l'unico che esegue `versions.insert` + `credentials.update`, lo stato lato persistenza resta coerente con la narrazione dell'audit log.

## Il signer SigV4 (il lavoro in stile AWS)

`src/cre/aws/signer.cr` implementa AWS SigV4 in stile RFC:

```
canonical_request = method + canonical_uri + canonical_query +
                    canonical_headers + signed_headers + payload_hash
string_to_sign    = "AWS4-HMAC-SHA256\n" + amz_date + "\n" +
                    credential_scope + "\n" + sha256(canonical_request)
signing_key       = HMAC chain (kSecret -> kDate -> kRegion -> kService -> kSigning)
signature         = HMAC(signing_key, string_to_sign)
```

L'header `Authorization` è costruito da `algorithm + Credential=... + SignedHeaders=... + Signature=...`. Include `X-Amz-Security-Token` quando viene fornito un session token STS.

Due file di test coprono il signer:
- `spec/unit/aws/signer_spec.cr` — idempotenza + forma dell'header Authorization tramite regex.
- `spec/unit/aws/signer_aws_vector_spec.cr` — usa gli input `get-vanilla` della suite di riferimento AWS (access key, secret, region, service, timestamp fisso) e fissa un vettore di regressione per la firma esatta prodotta dal nostro signer. Poiché emettiamo sempre `X-Amz-Content-SHA256` (richiesto da Secrets Manager e da altri servizi a protocollo JSON), la lista dei signed header è `host;x-amz-content-sha256;x-amz-date` — leggermente diversa dal vettore vanilla di AWS, quindi fissiamo i nostri byte invece di far corrispondere i loro. Qualsiasi modifica futura alla canonicalizzazione, alla derivazione della chiave o all'ordinamento degli header fa fallire il test.

## Integrità dell'audit log (a tre livelli)

`src/cre/audit/audit_log.cr` scrive i Livelli 1 + 2 a ogni `append`:
- `latest_hash` dal DB (genesis = 32 byte a zero per un log vuoto)
- `content_hash = HashChain.next_hash(prev_hash, canonical_payload)`
- `hmac = HmacRatchet#sign(content_hash)`; il ratchet avanza ogni 1024 righe
- Tutte e tre le colonne più `hmac_key_version` vengono persistite in un unico `INSERT` per riga

La verifica è suddivisa in tre metodi invocabili indipendentemente:
- `verify_hash_chain` — percorre ogni voce, ricalcola `SHA256(prev_hash || payload)` e lo confronta con `content_hash`.
- `verify_hmac_ratchet(seed_key)` — ripercorre il ratchet a partire dalla seed `CRE_HMAC_KEY_HEX`, ricalcola l'HMAC di ogni riga rispetto a `content_hash` e controlla che `hmac_key_version` corrisponda al punto in cui, secondo il ratchet, dovrebbe trovarsi la rotazione. Intercetta un attaccante che ha sistemato la hash chain ma non possiede la seed.
- `verify_batches(verifier)` — per ogni riga in `audit_batches`, recupera di nuovo le foglie `content_hash` corrispondenti, ricalcola la Merkle root e poi verifica la firma Ed25519 su `(start_seq, end_seq, root)`.

`src/cre/audit/batch_sealer.cr` costruisce le voci del Livello 3:
- Percorre i nuovi audit_events a partire da `last_sealed_seq`
- Costruisce un Merkle tree (`Merkle.root`) sul `content_hash` di ogni riga
- Firma `(start_seq, end_seq, root)` con Ed25519 tramite `Signing::Ed25519Signer`
- Inserisce in `audit_batches`

`src/cre/audit/batch_sealer_scheduler.cr` è la fiber che effettivamente guida il sealer in `cre run` / `cre watch`: chiama `seal_pending` una volta all'avvio, di nuovo ogni `CRE_SEAL_INTERVAL_SECONDS` (default 300s) e un'ultima volta allo spegnimento. Ogni sigillatura riuscita pubblica un evento tipizzato `AuditBatchSealed`, che l'audit subscriber riscrive nell'audit log come `audit.batch.sealed` — chiudendo il cerchio con la mappatura dei controlli SOC 2 / PCI-DSS / ISO / HIPAA che già si basa su quel tipo di evento.

L'OpenSSL della stdlib di Crystal non espone wrapper di alto livello per Ed25519, quindi `src/cre/audit/signing.cr` accede a LibCrypto tramite FFI: `EVP_PKEY_new_raw_private_key`, `EVP_DigestSign`, ecc. La verifica con chiave pubblica è simmetrica: `Ed25519Verifier#verify(message, signature)`.

## Envelope encryption AEAD

`src/cre/crypto/aead.cr` esegue AES-256-GCM tramite FFI su LibCrypto (`OpenSSL::Cipher` della stdlib non espone `auth_data=` / `auth_tag` per GCM). L'envelope (`src/cre/crypto/envelope.cr`) genera una DEK da 32 byte per riga, cifra il plaintext con AES-256-GCM(plaintext, DEK, AAD), poi cifra (wrap) la DEK con la KEK usando un AEAD separato (con il proprio AAD `kek-wrap|v<version>`). Entrambi i ciphertext sono `nonce(12) || tag(16) || body`.

Per decifrare serve la KEK per fare l'unwrap della DEK, poi la DEK + l'AAD per decifrare il payload. Una discrepanza dell'AAD fa fallire la verifica del tag al livello interno; una discrepanza della versione della KEK fa fallire l'unwrap.

## TUI

`src/cre/tui/state.cr` mantiene una vista a scorrimento delle rotazioni attive + gli eventi recenti. `apply(ev)` è l'unico punto di ingresso che modifica lo stato; logica di aggiornamento pura, facile da testare.

`src/cre/tui/renderer.cr` disegna i quattro pannelli su qualsiasi IO. Gli escape ANSI passano da `src/cre/tui/ansi.cr` (solo stdlib). L'helper `pad` del renderer tiene conto della larghezza degli escape ANSI, così l'allineamento delle colonne è corretto anche con i colori.

`src/cre/tui/tui.cr` mette tutto insieme: si iscrive al bus (overflow Drop), avvia una fiber di tick + una fiber di eventi, che chiamano entrambe `maybe_render`, la quale limita la frequenza a `refresh_interval`.

## Bot Telegram

`src/cre/notifiers/telegram.cr` è un sottile wrapper HTTP::Client per la Telegram Bot API (nessuna dipendenza da tourmaline per il percorso di notifica). Agli errori viene oscurato il token del bot prima che finiscano nei log — Telegram richiede il token nel path dell'URL, quindi l'oscuramento è best-effort, ma blocca la fuga più ovvia.

`src/cre/notifiers/telegram_bot.cr` esegue il long-poll di `getUpdates` e fa il dispatch dei comandi. L'autenticazione avviene tramite allowlist di chat ID; il livello viewer (`/status`, `/queue`, `/history`, `/alerts`) è in sola lettura; il livello operator aggiunge `/rotate`. `/rotate <id>` pubblica `RotationScheduled` sul bus, che viene consumato dal `RotationWorker` (vedi `src/cre/engine/rotation_worker.cr`) — il worker risolve la credenziale, cerca il `Rotator` giusto nella tabella di dispatch guidata dall'ambiente, controlla `rotations.in_flight` per deduplicare e passa il controllo a `RotationOrchestrator`.

## Forma del livello di persistenza

`src/cre/persistence/repos.cr` dichiara i repo astratti (`CredentialsRepo`, `VersionsRepo`, `RotationsRepo`, `AuditRepo`) e i tipi di record (`RotationRecord`, `AuditEntry`, `AuditBatch`, più gli enum `RotatorKind` e `RotationState`). `RotationState::Inconsistent` è incluso in `TERMINAL_STATES` insieme a `Completed` / `Failed` / `Aborted`.

Entrambi gli adapter applicano le modifiche allo schema tramite record `Migrations::Step` indicizzati su una `version` monotona. La tabella `schema_migrations` tiene traccia di quali versioni sono state eseguite; le nuove alterazioni arrivano come nuove voci `Step.new(N, ["ALTER TABLE ..."])` invece di modificare il groviglio di istruzioni `IF NOT EXISTS`.

`audit_events` è la tabella custodita con più attenzione nello schema:
- Postgres ha il trigger originale `audit_events_no_update` (solleva `audit_events is append-only` su UPDATE/DELETE/TRUNCATE).
- SQLite ottiene la parità tramite due trigger `BEFORE UPDATE` / `BEFORE DELETE` che usano `RAISE(ABORT, '...')`.
- L'`INSERT` del repo non usa più `OR IGNORE` / `ON CONFLICT … DO NOTHING`, quindi un fallimento di un vincolo viene propagato all'applicazione invece di essere scartato silenziosamente.
- Il percorso di rescue dell'audit subscriber pubblica un `AlertRaised` critico e (per default) manda in panic il processo tramite `CRE_AUDIT_FAILURE_MODE=panic`.

Due livelli indipendenti; entrambi devono essere sovvertiti per falsificare la storia.
