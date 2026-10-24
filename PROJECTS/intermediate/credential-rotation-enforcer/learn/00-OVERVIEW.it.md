<!--
©AngelaMos | 2026
00-OVERVIEW.md
-->

# Credential Rotation Enforcer - Panoramica

Un daemon in Crystal che **traccia** le credenziali, **applica** policy di rotazione definite come codice ed **esegue** il contratto di rotazione in quattro passi verso AWS Secrets Manager, HashiCorp Vault, i PAT fine-grained di GitHub e i file `.env` locali. Un singolo binario. TUI live. Bot Telegram bidirezionale. Audit log tamper-evident. Export firmato delle evidenze di compliance.

## Cosa dimostra questo progetto

| Concetto | Cosa vedrai nel codice |
|---|---|
| DSL di policy validato staticamente | `policies/*.cr` valutati dal compilatore Crystal; argomenti enum a simbolo singolo (`enforce :rotate_immediately`) e refusi nei blocchi `match {}` fanno fallire `crystal build`. Argomenti splat di simboli (`notify_via :telegram, :slack`) e campi obbligatori mancanti sollevano `BuilderError` al momento della registrazione della policy. In ogni caso, una policy malformata non raggiunge mai un daemon in esecuzione. |
| Architettura bus + plugin | Eventi tipizzati si propagano a ventaglio sui channel di Crystal; i subscriber (audit, TUI, Telegram, log) reagiscono in modo indipendente; i rotator si registrano a compile time tramite la macro `register_as :kind` |
| Contratto di rotazione in 4 passi | `generate -> apply -> verify -> commit`, sicuro a doppia versione (analogo di AWSCURRENT / AWSPENDING), con rollback in caso di fallimento tra apply e commit |
| Audit log tamper-evident | Hash chain SHA-256 + HMAC-SHA256 a ratchet + batch Merkle firmati Ed25519 (3 livelli indipendenti di integrità) |
| Envelope encryption AEAD | AES-256-GCM con DEK per riga cifrate (wrapped) da una KEK; AAD legato a `tenant_id || credential_id || version_id`; byte `algorithm_id` riservato per la crypto agility |
| TUI live scritta a mano | Solo escape ANSI (nessuna dipendenza da `crysterm`); i repaint guidati dagli eventi sono accorpati su un intervallo di tick; funziona su qualsiasi IO, quindi è testabile |

## Prerequisiti

- Crystal **1.20.0+** (vedi `https://crystal-lang.org/install/`)
- Per la demo Tier 1: nient'altro
- Per la demo Tier 2: Docker + Docker Compose
- Per il Tier 3 (cloud reale): account AWS, server Vault, token GitHub Apps

## Percorso demo in tre tier

Ogni tier insegna una lezione diversa. Eseguili in ordine per vedere il sistema evolvere.

### Tier 1 - Zero dipendenze

```
$ git clone <repo> && cd credential-rotation-enforcer
$ shards install && shards build cre
$ ./bin/cre demo
```

Cosa vedrai:
- Un file `.env` temporaneo con `API_KEY=oldvalue-aaa`
- Una credenziale simulata vecchia di 60 giorni che fa scattare una violazione di policy
- La narrazione live di tutti e 4 i passi di rotazione
- Lo stesso file `.env` con un nuovo valore casuale `API_KEY=...`
- La verifica della catena di audit che ne conferma l'integrità

Tempo di esecuzione: meno di 1 secondo.

### Tier 2 - Docker Compose

```
$ just demo-full
```

Avvia: PostgreSQL 16, LocalStack (AWS Secrets Manager), HashiCorp Vault in modalità dev, un servizio Flask che simula GitHub. CRE si connette a tutti e quattro e ruota una credenziale attraverso ciascun rotator. Dimostra chiamate di rete reali, autenticazione reale (SigV4 verso LocalStack, token verso Vault, bearer verso il finto GitHub), persistenza reale su Postgres, trigger di audit append-only reali.

Tempo di setup: ~2 minuti (soprattutto pull delle image).

### Tier 3 - Cloud reale

`cre run` e `cre watch` si rifiutano di partire senza due segreti da 32 byte:

```
export CRE_HMAC_KEY_HEX=$(openssl rand -hex 32)   # audit log seed key
export CRE_KEK_HEX=$(openssl rand -hex 32)        # envelope KEK
export CRE_SIGNING_KEY_HEX=$(openssl rand -hex 32)  # optional: enables Merkle-batch sealing
```

Poi copia `config/demo-full.cr.example`, imposta le variabili d'ambiente AWS / Vault / GitHub che documenta ed esegui `cre run` in modalità headless oppure `cre watch` per la TUI live. Senza `CRE_SIGNING_KEY_HEX`, il daemon funziona comunque ma salta il Livello 3 dell'audit log — anche `cre audit verify` salterà il livello Merkle.

## Cheat sheet dei sottocomandi

| Comando | Scopo |
|---|---|
| `cre run` | Daemon headless (produzione / systemd) — richiede `CRE_HMAC_KEY_HEX` + `CRE_KEK_HEX` |
| `cre watch` | Engine + TUI live in un unico processo — stessi requisiti di ambiente |
| `cre check` | Valutazione one-shot delle policy, exit code in base alle violazioni (adatto alla CI) |
| `cre rotate <id>` | Rotazione manuale di una singola credenziale — usa gli stessi rotator guidati dall'ambiente di `cre run` |
| `cre policy list` | Elenca le policy compilate nel binario |
| `cre policy show <name>` | Ispeziona una policy nel dettaglio |
| `cre export --framework=soc2` | Genera lo ZIP firmato delle evidenze di compliance |
| `cre audit verify` | Hash chain + ratchet HMAC (il livello Merkle si aggiunge con `--public-key=PATH` o `CRE_AUDIT_PUBLIC_KEY_HEX`) |
| `cre verify-bundle <zip>` | Ri-verifica offline di un bundle di evidenze (sha256 + firma del manifest + catena + Merkle) |
| `cre demo` | Demo Tier 1 a zero dipendenze |
| `cre tui-demo` | Anteprima della TUI di 8 secondi con eventi sintetici (niente daemon, niente DB) |
| `cre version` | Stampa la versione |

## Cosa leggere dopo

- **Architettura** -> `02-ARCHITECTURE.md` (event bus, persistenza, livelli crittografici)
- **Concetti** -> `01-CONCEPTS.md` (teoria della rotazione, controlli dei framework NIST/SOC2, breach reali)
- **Walkthrough del codice** -> `03-IMPLEMENTATION.md` (funzioni chiave, dove apportare modifiche)
- **Idee di estensione** -> `04-CHALLENGES.md` (aggiungere un 5° rotator, wrap ibrido ML-KEM, interfaccia web)
