<!--
©AngelaMos | 2026
02-ARCHITECTURE.md
-->

# Architettura

## Panoramica del sistema

```
┌─────────────────────────────────────────────────────────────┐
│                       cre  (single Crystal binary)          │
│                                                             │
│   ┌──────────────────────────────────────────────────────┐  │
│   │                   Event Bus                          │  │
│   │             (typed Crystal channels)                 │  │
│   └────┬─────┬─────┬─────┬─────┬─────┬─────┬────────────┘  │
│        │     │     │     │     │     │     │               │
│   ┌────▼─┐ ┌─▼──┐ ┌▼───┐ ┌▼──┐ ┌▼────┐ ┌──▼──┐ ┌──────┐    │
│   │Sched │ │Rot.│ │Pol.│ │TUI│ │Tele.│ │Audit│ │Notify│    │
│   │ulers │ │Reg │ │Eval│ │   │ │Bot  │ │Log  │ │      │    │
│   └──┬───┘ └─┬──┘ └─┬──┘ └───┘ └─────┘ └──┬──┘ └──────┘    │
│      │       │      │                     │                 │
│      └───────┴──────┴─────────────────────┘                 │
│                       │                                     │
│              ┌────────▼─────────┐                           │
│              │  Persistence     │ ◄── SQLite (Tier 1)       │
│              │  (PG / SQLite)   │ ◄── PostgreSQL (T2/T3)    │
│              └──────────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

Tutti i componenti di lunga durata sono **fiber in un unico processo del sistema operativo**. Il bus è in-process - i channel di Crystal operano su scala di nanosecondi, quindi l'overhead architetturale è praticamente nullo.

## Componenti

### Event Bus (`src/cre/engine/event_bus.cr`)

Dispatch a ventaglio (fanout) tramite i channel di Crystal. Ogni subscriber ottiene il proprio channel limitato (bounded) e sceglie una policy di overflow:

| Subscriber | Overflow | Motivo |
|---|---|---|
| `AuditSubscriber` | `Block` | Non scartare mai gli eventi di audit; requisito di compliance |
| `Tui` | `Drop` | Una UI non aggiornata va bene; non può bloccare l'engine |
| `LogNotifier` | `Drop` | Log strutturati best-effort |
| `TelegramSubscriber` | `Drop` (buffer 128) | Comunque instabile a livello di rete |
| `RotationWorker` | `Block` (buffer 32) | Deve eseguire il dispatch delle rotazioni pianificate |

Il dispatcher è una singola fiber che legge dal channel inbox e scrive sui channel dei subscriber. Entrambe le modalità di overflow usano `select`, così un subscriber bloccato non può inchiodare il bus a tempo indeterminato:

- `Drop`: `select … else …` non bloccante — con il buffer pieno registra un warn e scarta l'evento.
- `Block`: `select … when timeout(@block_send_timeout) …` — se il buffer di un subscriber resta pieno oltre il timeout (default 5s), il bus scarta quel singolo evento, registra lo stallo e prosegue. Gli operatori alzano il timeout per i downstream lenti di cui si fidano (scritture del DB di audit) e lo abbassano per quelli inaffidabili.

### Rotator (`src/cre/rotators/`)

La classe astratta `Rotator` espone i quattro metodi del ciclo di vita più `rollback_apply`. I rotator concreti si auto-registrano a compile time:

```crystal
class AwsSecretsRotator < Rotator
  register_as :aws_secretsmgr
  ...
end
```

Aggiungere un quinto rotator significa inserire un singolo file in `src/cre/rotators/`. La macro lo aggancia al registry a compile time. Nessun cablaggio centrale da aggiornare.

I rotator ricevono il proprio client cloud tramite il costruttore (DI). Il comando CLI `run` collega il client giusto in base alle variabili d'ambiente / alla configurazione.

### Persistenza (`src/cre/persistence/`)

Due adapter dietro un'unica interfaccia:
- `Sqlite::SqlitePersistence` - singola connessione (`max_pool_size=1`) così la serializzazione delle scritture di SQLite è sicura; `synchronous=NORMAL`; i trigger `BEFORE UPDATE` e `BEFORE DELETE` su `audit_events` sollevano `audit_events is append-only`. Mutex a livello applicativo per simulare gli advisory lock (su SQLite l'astrazione è solo in-process). Usato per la demo Tier 1.
- `Postgres::PostgresPersistence` - tag JSONB, seq di audit `BIGSERIAL`, trigger `audit_events_no_update` che rifiuta UPDATE/DELETE/TRUNCATE, `pg_advisory_xact_lock` per il locking tra processi, e un indice unique parziale su `rotations(credential_id) WHERE state NOT IN ('completed','failed','aborted','inconsistent')` così due daemon non possono mai inserire rotazioni in corso sovrapposte per la stessa credenziale. Usato per i Tier 2/3.

Entrambi i backend condividono lo stesso runner delle migration (tabella `schema_migrations` + record `Step` tracciati per versione), quindi aggiungere una colonna è una riga `Step.new(N, ["ALTER TABLE ..."])` invece di modificare un groviglio di `IF NOT EXISTS`. I contratti dei repo (`CredentialsRepo`, `VersionsRepo`, `RotationsRepo`, `AuditRepo`) sono identici tra gli adapter; `Persistence` espone `transaction(&)` e `with_advisory_lock(key, &)` così il resto del sistema resta indipendente dal backend.

### Livelli crittografici (`src/cre/crypto/`, `src/cre/audit/`)

```
+--------------------+
|   Plaintext        |
+--------------------+
          |
          | AES-256-GCM(plain, DEK, AAD = tenant||cred||version, nonce 96b)
          v
+--------------------+
|  ciphertext + tag  | -> credential_versions.ciphertext
+--------------------+

+----+
| DEK| (32 random bytes per row)
+----+
   |
   | KEK_v.wrap(DEK)  (envelope encryption)
   v
+--------------------+
|   wrapped DEK       | -> credential_versions.dek_wrapped + kek_version
+--------------------+

KEK source (per tier):
  Tier 1: env var CRE_KEK_HEX (64-hex chars = 32 bytes)
  Tier 2: AWS KMS via LocalStack
  Tier 3: real AWS KMS or HSM
```

Le DEK per riga eliminano il problema del birthday bound sul riuso del nonce di AES-GCM (la DEK di ogni riga cifra esattamente un messaggio). Il binding con l'AAD previene gli attacchi di scambio del ciphertext, in cui un attaccante con accesso in scrittura al DB prova a spostare il ciphertext di una riga a basso privilegio in una riga ad alto privilegio.

`algorithm_id` è riservato per la crypto agility:
- `0x01` = AES-256-GCM (oggi)
- `0x02` = XChaCha20-Poly1305 (nonce lungo, più semplice)
- `0x03` = wrap ibrido ML-KEM (forward secrecy post-quantum)

### Stack di integrità dell'audit log

Tre livelli, sempre più difficili da aggirare:

```
+-------------------------------------------------+
|  Layer 3: Ed25519-signed Merkle batches         |
|    audit_batches table, hourly seal, signed     |
|    over (start_seq, end_seq, root)              |
|    Auditor verifies with public key only.       |
+-------------------------------------------------+
                    |
                    | leaves: content_hash[]
                    v
+-------------------------------------------------+
|  Layer 2: HMAC ratchet                          |
|    K_v signs each row's content_hash; every     |
|    1024 rows -> K_{v+1} = HKDF(K_v, "ratchet"); |
|    K_v zeroized in memory                       |
+-------------------------------------------------+
                    |
                    v
+-------------------------------------------------+
|  Layer 1: Hash chain                            |
|    content_hash = SHA256(prev_hash ||           |
|                          canonical_payload)     |
|    rendering single-row tampering visible       |
+-------------------------------------------------+
                    |
                    v
+-------------------------------------------------+
|  PostgreSQL audit_events table                  |
|    - INSERT-only role grant                     |
|    - UPDATE/DELETE/TRUNCATE trigger refuses     |
+-------------------------------------------------+
```

I trigger di PG non sono strettamente necessari (la catena rileva comunque le manomissioni), ma falliscono in modo evidente al momento della scrittura, il che è molto più comodo per gli operatori. Il tier 1 su SQLite documenta la garanzia più debole.

## Concorrenza

| Ambito | Limite | Meccanismo |
|---|---|---|
| Per credenziale | 1 rotazione attiva | `RotationWorker` controlla `rotations.in_flight` prima del dispatch; PG impone anche un indice unique parziale, così i duplicati tra processi falliscono a livello di DB |
| Ciclo di vita per rotazione | 1 passo alla volta | L'orchestrator esegue `generate -> apply -> verify -> commit` in sequenza; `rollback_apply` scatta in caso di fallimento di apply/verify; un fallimento del commit marca la rotazione come `inconsistent` e solleva un alert critico |
| Event bus dell'engine | buffer per subscriber + timeout | `EventBus#dispatch` usa `select` sia per l'overflow Block sia per quello Drop (vedi la tabella dei subscriber del bus sopra) |

Fiber di Crystal + channel limitati = rate limiting pulito senza thread né lock.

## Ciclo di vita (cre run)

```
1. Bootstrap: validate CRE_HMAC_KEY_HEX + CRE_KEK_HEX (hard-fail if missing)
2. Open persistence (PG or SQLite); migrate! (versioned Step list)
3. Build Envelope from KEK; build optional Ed25519Signer from CRE_SIGNING_KEY_HEX
4. Load compiled-in policies (REGISTRY) + register rotators from env vars
5. Start EventBus.run (dispatcher fiber)
6. Start subscribers: AuditSubscriber, LogNotifier, RotationWorker, PolicyEvaluator
7. Start Scheduler (SchedulerTick every CRE_TICK_SECONDS)
8. Start BatchSealerScheduler if signer present (default every 5min)
9. Wire Telegram bot if TELEGRAM_TOKEN + chat IDs are set
10. Optionally start TUI + Snapshotter (cre watch)
11. Block on stop_signal channel; SIGINT triggers graceful drain
```

Spegnimento graceful: `engine.stop` pubblica `ShutdownRequested` e attende fino a 2s su `audit_subscriber.await_drain` — l'audit subscriber invia su un channel di completamento privato non appena elabora quell'evento, il che dimostra che ha gestito tutto ciò che era in coda prima di esso. Poi il bus chiude il proprio inbox e i channel dei subscriber; ogni fiber subscriber termina su `Channel::ClosedError`.
