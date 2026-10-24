<!--
©AngelaMos | 2026
01-CONCEPTS.md
-->

# Concetti - La rotazione delle credenziali, nella pratica e nella storia

## Perché la rotazione delle credenziali è una disciplina a sé

Una credenziale è un fatto in uno stato condiviso: "questo token autorizza questo principal". Più a lungo quel fatto resta nello stato condiviso, più sono i punti da cui può trapelare. La compromissione ha un tempo di dimezzamento: ogni commit, ogni log di CI, ogni laptop, ogni nastro di backup ne estende il raggio. La rotazione è la disciplina che costringe quel fatto a scadere secondo una pianificazione **più breve del tempo di scoperta** dei percorsi di fuga che non riesci a vedere.

Il consenso dell'industria dopo il 2020 è cambiato in modo significativo:

- **NIST SP 800-63B-4** (versione finale luglio 2025): *vieta* la rotazione periodica delle password per le credenziali umane, MA continua a richiedere la rotazione per service account, API key e identità macchina.
- **NIST SP 800-57 Pt1 Rev5**: rotazione guidata dal cryptoperiod per ciascun tipo di chiave (DEK, KEK, chiavi di firma, MAC).
- **PCI DSS v4.0.1** (obbligatorio da marzo 2025): la regola dei 90 giorni resta il default; la via d'uscita del "Customized Approach" ti permette di sostituirla con monitoraggio continuo + rotazione basata sul rischio.
- **CA/Browser Forum SC-081v3**: la durata dei certificati TLS scende a 47 giorni entro il 2029.

Quindi la risposta pratica nel 2026 non è "ruota tutto ogni 90 giorni" - è **rotazione basata sul rischio** per le credenziali umane, **rotazione guidata dal cryptoperiod** per le chiavi e **JIT / effimere / workload identity** per i percorsi service-to-service a più alto rischio.

## Breach reali che hanno motivato questo design

| Anno | Incidente | Fallimento della credenziale alla radice |
|---|---|---|
| 2020 | SolarWinds Sunburst | Token di service account riutilizzato per ~9 mesi durante la permanenza dell'attaccante |
| 2021 | Codecov bash uploader | Una chiave di upload GCS trapelata, nessuna rotazione, 2 mesi di accesso alla supply chain |
| 2022 | Heroku/Travis OAuth | Token OAuth riutilizzati per npm - nessuna rotazione, nessuna riduzione dello scope |
| 2023 | Storm-0558 (Microsoft) | Chiave di firma MSA generata nel 2016, mai ruotata, trapelata tramite crash dump |
| 2023 | Okta support breach | File HAR contenenti token di sessione - nessun rilevamento dell'esfiltrazione, nessun token binding |
| 2024 | Snowflake / UNC5537 | Credenziali dei clienti trapelate anni prima, mai ruotate, nessuna MFA |
| 2024 | Microsoft Midnight Blizzard | Credenziali di test di un tenant legacy non di produzione, mai ruotate |
| 2025 | tj-actions CVE-2025-30066 | Furto di token GitHub Actions tramite un'action compromessa nella supply chain |

Lo schema si ripete: il **tempo** è l'alleato dell'attaccante. Ogni riga qui sopra aveva almeno una credenziale vissuta troppo a lungo in uno stato condiviso.

## Cosa previene questo enforcer (e cosa no)

**Previene:**
- Credenziali obsolete che vivono oltre la durata prevista (violazione di policy -> notifica o rotazione automatica)
- Drift silenzioso della rotazione (rilevamento attuale-vs-fingerprint nel DB)
- Dimenticarsi di registrare una rotazione (audit log automatico tramite subscriber del bus - l'orchestrator non può aggirarlo)
- Manomissione dell'audit log (integrità a 3 livelli: catena + ratchet HMAC + firma dei batch Ed25519)
- Trigger di rotazione non autorizzati (ACL del bot Telegram, la CLI richiede un ID di credenziale esplicito)

**NON previene:**
- La compromissione della KEK stessa (usa AWS KMS / HSM in produzione)
- Un attaccante che ha già esfiltrato una credenziale valida prima della rotazione (ruota il prima possibile, ma il fatto è già trapelato)
- Un insider che esegue `cre rotate <id>` legittimamente ma con intenti malevoli (l'audit log lo registra; non lo ferma)
- Un DNS hijack di api.telegram.org (usa webhook con mTLS per deployment hardened)

## Il contratto di rotazione in quattro passi, spiegato

Preso alla lettera dal template della Rotation Lambda di AWS Secrets Manager. La regola è: **tra il passo 2 e il passo 4, sia la credenziale vecchia SIA quella nuova sono valide.** Questa è la garanzia di sicurezza a doppia versione. I consumer concorrenti che usano l'una o l'altra credenziale hanno successo; nessuno va in crash a metà rotazione.

```
                     time -->
   step 1 (generate)    step 2 (apply)    step 3 (verify)    step 4 (commit)
        |                   |                   |                   |
   old: usable          old: usable         old: usable          old: REVOKED
   new: pending         new: usable         new: usable          new: current
        |                                                            |
        +-- rollback OK --+                +--+--+                   v
        upstream pending     verify failed?   |    irreversible
        artifact deletable   rollback_apply   |
                             revokes new      |
                                              v
                                              if verify passes,
                                              proceed to commit
```

Il passo 4 è l'unico passo **irreversibile**. Per design. Se il passo 4 fallisce parzialmente (il commit è riuscito per la nuova ma la vecchia non è stata effettivamente revocata), l'orchestrator marca la rotazione come `inconsistent` e solleva un alert - ti diciamo onestamente "qui serve un umano" invece di corrompere silenziosamente lo stato.

## Integrità dell'audit log, su tre livelli

| Livello | Meccanismo | Difende da |
|---|---|---|
| **Hash chain** | Il `content_hash` di ogni riga `= SHA256(prev_hash || canonical_payload)` | Modifiche silenziose - qualsiasi cambiamento rompe la catena in avanti |
| **Ratchet HMAC** | Ogni riga è sottoposta a HMAC con la chiave K_v; ogni 1024 righe si deriva K_{v+1} = HKDF(K_v, "ratchet") e si azzera K_v | Un attaccante che in seguito ottiene accesso in scrittura al DB + la chiave corrente non può comunque riscrivere le voci passate (la vecchia chiave non esiste più) |
| **Batch Merkle Ed25519** | Ogni ora: costruisce una Merkle root sulle foglie `content_hash`; firma la root con Ed25519 | Un auditor può verificare con **solo la chiave pubblica + i batch**, senza bisogno di accesso al DB |

La verifica è esposta come comando CLI:
```
$ cre audit verify --public-key=/etc/cre/audit_pubkey.hex
  ✓  hash chain: OK
  ✓  HMAC ratchet: OK
  ✓  Merkle batches: OK
✓ audit chain valid: 14892 entries
```

Il ratchet HMAC viene ripercorso a partire dalla seed key fornita tramite `CRE_HMAC_KEY_HEX`, quindi il verificatore ha bisogno della *stessa* seed usata da chi scrive. I batch Merkle sono firmati con `CRE_SIGNING_KEY_HEX` e verificati con la chiave pubblica corrispondente — inclusa nello ZIP di `cre export` così che un auditor offline possa eseguire `cre verify-bundle` senza accesso al DB.

## Copertura dei framework di compliance

Il bundle di export (`cre export --framework=soc2`) mappa gli eventi di audit su specifici controlli dei framework. Secondo `src/cre/compliance/control_mapping.cr`:

- **SOC 2** - CC6.1 (gestione degli accessi logici), CC6.6 (gestione delle vulnerabilità), CC6.7 (revisione degli accessi), CC4.1 (monitoraggio), CC7.1 / CC7.2 (rilevamento degli incidenti)
- **PCI DSS v4.0.1** - 8.3.9 / 8.6.3 (autenticazione e rotazione), 10.5.x (integrità dell'audit log), 3.7.4 (gestione delle chiavi)
- **ISO 27001:2022** - A.5.16 / A.5.17 / A.5.18 (identità e accessi), A.8.5 (autenticazione sicura), A.8.15 / A.8.16 (logging e monitoraggio), A.8.24 (crittografia)
- **HIPAA Security Rule** - 164.308(a)(5)(ii)(D) (gestione delle password), 164.312(b) (controlli di audit)
