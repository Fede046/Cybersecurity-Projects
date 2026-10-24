# Approfondimento sul design del database

Questo documento spiega lo schema del database, le decisioni di design, le relazioni e le strategie di migrazione usate nella Bug Bounty Platform.

---

## Indice

1. [Panoramica dello schema](#panoramica-dello-schema)
2. [Diagramma entità-relazione](#diagramma-entità-relazione)
3. [Schemi delle tabelle](#schemi-delle-tabelle)
4. [Decisioni di design](#decisioni-di-design)
5. [Indici e performance](#indici-e-performance)
6. [Migrazioni](#migrazioni)
7. [Query comuni](#query-comuni)

---

## Panoramica dello schema

La piattaforma usa **PostgreSQL 18** con 9 tabelle principali:

| Tabella | Descrizione | Righe (tipiche) |
|-------|-------------|----------------|
| `users` | Account utente (ricercatori + aziende) | 1K-100K |
| `refresh_tokens` | Refresh token JWT | 5K-500K |
| `programs` | Programmi bug bounty | 100-10K |
| `assets` | Scope del programma (domini, API) | 500-50K |
| `reward_tiers` | Importi dei bounty per severità | 500-50K |
| `reports` | Segnalazioni di vulnerabilità | 10K-1M |
| `comments` | Comunicazione di triage | 50K-5M |
| `attachments` | File di proof-of-concept | 10K-1M |

---

## Diagramma entità-relazione

```
┌──────────────────────────────┐
│          users               │
│──────────────────────────────│
│ id (UUID v7) PK              │
│ email (unique, indexed)      │
│ hashed_password              │
│ full_name                    │
│ role (enum)                  │
│ is_active                    │
│ is_verified                  │
│ token_version                │
│ company_name                 │
│ bio                          │
│ website                      │
│ reputation_score             │
│ created_at, updated_at       │
└──────────────┬───────────────┘
               │
    ┌──────────┼──────────┬──────────────┐
    │          │          │              │
    │ 1        │ 1        │ 1            │
    │          │          │              │
    ▼ *        ▼ *        ▼ *            ▼ *
┌─────────────┐ ┌──────────┐ ┌────────────┐ ┌────────────────┐
│refresh_     │ │programs  │ │  reports   │ │                │
│tokens       │ │          │ │            │ │                │
│─────────────│ │──────────│ │────────────│ │                │
│id PK        │ │id PK     │ │id PK       │ │                │
│user_id FK   │ │company_id│ │program_id  │ │                │
│token_hash   │ │name      │ │researcher  │ │                │
│device_info  │ │slug (idx)│ │  _id FK    │ │                │
│family_id    │ │description│ │title      │ │                │
│ip_address   │ │rules     │ │description │ │                │
│expires_at   │ │response  │ │steps_to    │ │                │
│created_at   │ │  _sla_hrs│ │  _reproduce│ │                │
└─────────────┘ │status    │ │impact      │ │                │
                │  (indexed)│ │severity    │ │                │
                │visibility│ │  _submitted│ │                │
                │created_at│ │severity    │ │                │
                │updated_at│ │  _final    │ │                │
                └────┬─────┘ │status (idx)│ │                │
                     │       │cvss_score  │ │                │
           ┌─────────┼────┐  │cwe_id      │ │                │
           │         │    │  │bounty_amt  │ │                │
           │ 1       │ 1  │  │duplicate   │ │                │
           │         │    │  │  _of_id    │ │                │
           ▼ *       ▼ *  │  │triaged_at  │ │                │
        ┌─────┐  ┌─────┐ │  │resolved_at │ │                │
        │asset│  │reward│ │  │disclosed_at│ │                │
        │     │  │tier │ │  │created_at  │ │                │
        │─────│  │─────│ │  │updated_at  │ │                │
        │id PK│  │id PK│ │  └──────┬─────┘ │                │
        │prog │  │prog │ │         │       │                │
        │ _id │  │ _id │ │    ┌────┼────┐  │                │
        │type │  │seve │ │    │ 1       │ 1│                │
        │targ │  │ rity│ │    │         │  │                │
        │ et  │  │amnt │ │    ▼ *       ▼ *│                │
        │crea │  │crea │ │  ┌────────┐ ┌──┴────┐            │
        │ted  │  │ted  │ │  │comments│ │attach │            │
        │     │  │     │ │  │        │ │ments  │            │
        └─────┘  └─────┘ │  │────────│ │───────│            │
                         │  │id PK   │ │id PK  │            │
                         │  │report  │ │report │            │
                         │  │  _id FK│ │  _id  │            │
                         │  │author  │ │  FK   │            │
                         │  │  _id FK│ │file   │            │
                         │  │content │ │  _name│            │
                         │  │created │ │file   │            │
                         │  │  _at   │ │  _size│            │
                         │  │updated │ │file   │            │
                         │  │  _at   │ │  _type│            │
                         │  └────────┘ │s3_key │            │
                         │             │created│            │
                         │             │  _at  │            │
                         │             └───────┘            │
                         │                                  │
                         └──────────────────────────────────┘
```

---

## Schemi delle tabelle

### users

**Scopo:** Memorizzare gli account utente (ricercatori, aziende, admin)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE,
    role VARCHAR(50) DEFAULT 'user',  -- SafeEnum: stores value, not name
    token_version INTEGER DEFAULT 0,
    company_name VARCHAR(255),
    bio TEXT,
    website VARCHAR(255),
    reputation_score INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_users_email ON users(email);
```

**Campi chiave:**

- `id` - UUID v7 (ordinabile nel tempo, univoco a livello globale)
- `email` - Identificatore univoco (indicizzato per lookup veloci)
- `hashed_password` - Hash Argon2id (non memorizzare mai la password in chiaro)
- `role` - Pattern SafeEnum (memorizza "user", non "USER")
- `token_version` - Incrementalo per invalidare tutti i token
- `reputation_score` - Reputazione del ricercatore (bounty guadagnati, report accettati)

**Decisioni di design:**

1. **Tabella utenti unica** - Sia i ricercatori sia le aziende usano la stessa tabella
   - Alternativa: Tabelle separate `researchers` e `companies`
   - Motivo: Stessa autenticazione, riduce la complessità dei JOIN
   - Compromesso: Alcuni campi restano inutilizzati (i ricercatori non hanno `company_name`)

2. **Soft delete NON utilizzati** - Quando un utente viene eliminato, scompare
   - Alternativa: Aggiungere una colonna `deleted_at`
   - Motivo: La conformità al GDPR richiede una cancellazione vera
   - Eccezione: L'admin può impostare `is_active = false` per sospendere gli account

3. **Token versioning** - Incrementa per invalidare istantaneamente tutti i token
   - Alternativa: Mantenere una blacklist di token
   - Motivo: Più semplice, nessuna necessità di ripulire le vecchie voci della blacklist

### programs

**Scopo:** Programmi bug bounty ospitati dalle aziende

```sql
CREATE TABLE programs (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    company_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    rules TEXT,
    response_sla_hours INTEGER DEFAULT 72,
    status VARCHAR(50) DEFAULT 'draft',  -- draft, active, paused, closed
    visibility VARCHAR(50) DEFAULT 'public',  -- public, private, invite_only
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_programs_slug ON programs(slug);
CREATE INDEX idx_programs_status ON programs(status);
CREATE INDEX idx_programs_company_id ON programs(company_id);
```

**Campi chiave:**

- `slug` - Identificatore URL-friendly (`/programs/acme-corp`)
- `status` - Ciclo di vita del programma (indicizzato per filtrare i programmi attivi)
- `visibility` - Chi può vedere il programma
- `response_sla_hours` - Con quale rapidità l'azienda deve rispondere (default 72h)

**Decisioni di design:**

1. **Slug per gli URL** - Usa lo slug invece dell'ID negli URL
   - Alternativa: Usare l'UUID nell'URL (`/programs/018d3f54-8c3a-7000`)
   - Motivo: UX migliore, SEO-friendly
   - Vincolo: Deve essere univoco e immutabile

2. **Enum di stato** - Draft → Active → Paused/Closed
   - Draft: Non visibile, l'azienda lo sta configurando
   - Active: Accetta segnalazioni
   - Paused: Chiuso temporaneamente (nessuna nuova segnalazione)
   - Closed: Chiuso definitivamente

3. **Tracciamento dello SLA** - `response_sla_hours` stabilisce l'aspettativa
   - Usato per calcolare se l'azienda sta rispettando lo SLA
   - Può essere usato per notifiche automatiche

### reports

**Scopo:** Segnalazioni di vulnerabilità inviate dai ricercatori

```sql
CREATE TABLE reports (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    program_id UUID NOT NULL REFERENCES programs(id) ON DELETE CASCADE,
    researcher_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    steps_to_reproduce TEXT,
    impact TEXT,
    severity_submitted VARCHAR(50) DEFAULT 'medium',
    severity_final VARCHAR(50),
    status VARCHAR(50) DEFAULT 'new',  -- new, triaging, accepted, etc.
    cvss_score NUMERIC(3, 1),  -- 0.0 to 10.0
    cwe_id VARCHAR(20),  -- CWE-79, CWE-89, etc.
    bounty_amount INTEGER,
    duplicate_of_id UUID REFERENCES reports(id) ON DELETE SET NULL,
    triaged_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    disclosed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reports_program_id ON reports(program_id);
CREATE INDEX idx_reports_researcher_id ON reports(researcher_id);
CREATE INDEX idx_reports_status ON reports(status);
```

**Campi chiave:**

- `severity_submitted` - Valutazione del ricercatore
- `severity_final` - Valutazione finale dell'azienda (può differire)
- `status` - Ciclo di vita del report (indicizzato per il filtraggio)
- `cvss_score` - Common Vulnerability Scoring System (0.0-10.0)
- `cwe_id` - Common Weakness Enumeration (CWE-79 = XSS)
- `duplicate_of_id` - FK autoreferenziale per il tracciamento dei duplicati
- `triaged_at`, `resolved_at`, `disclosed_at` - Timestamp del ciclo di vita

**Decisioni di design:**

1. **Due campi di severità** - Inviata vs finale
   - Motivo: Il ricercatore può sovrastimare la severità
   - L'azienda può correggerla durante il triage
   - Trasparenza: Entrambi i valori sono visibili

2. **Workflow di stato** - Progressione lineare con ramificazioni
   ```
   NEW → TRIAGING → NEEDS_MORE_INFO → TRIAGING
         ↓
         ACCEPTED → RESOLVED → DISCLOSED
         ↓
         DUPLICATE
         ↓
         NOT_APPLICABLE
         ↓
         INFORMATIVE
   ```

3. **Tracciamento dei duplicati** - `duplicate_of_id` rimanda all'originale
   - Alternativa: Memorizzare gli ID dei duplicati in un array
   - Motivo: Semplicità, è possibile attraversare la catena dei duplicati
   - Avvertenza: Bisogna prevenire i riferimenti circolari

4. **CVSS + CWE** - Classificazione delle vulnerabilità standard del settore
   - Punteggio CVSS: Severità numerica (6.0-6.9 = medium)
   - CWE ID: Categoria della debolezza (CWE-79 = XSS)
   - Entrambi sono opzionali ma raccomandati

### refresh_tokens

**Scopo:** Memorizzare i refresh token JWT per l'autenticazione

```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,  -- SHA-256 hash
    device_info VARCHAR(255),
    ip_address VARCHAR(45),  -- IPv6-compatible
    family_id UUID NOT NULL,  -- For token rotation detection
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at);
```

**Campi chiave:**

- `token_hash` - Hash SHA-256 del refresh token (non in chiaro!)
- `family_id` - Traccia la catena di rotazione dei token
- `device_info` - Stringa dello user agent
- `ip_address` - Per il monitoraggio della sicurezza
- `expires_at` - I token scadono dopo 7 giorni

**Decisioni di design:**

1. **Hash dei token** - Memorizza l'hash SHA-256, non il testo in chiaro
   - Se il database viene compromesso, l'attaccante non può usare i token
   - Bisogna fare l'hash del token prima di interrogare il database

2. **Rotazione dei token** - Ogni refresh crea un nuovo token
   - Il vecchio token viene eliminato
   - Il nuovo token ha lo stesso `family_id`
   - Se il vecchio token viene riutilizzato → rilevato tramite mismatch del `family_id`

3. **Supporto multi-dispositivo** - Un utente può avere più refresh token
   - Ogni dispositivo/sessione ottiene il proprio token
   - Il logout elimina il token specifico
   - Il logout-all elimina tutti i token dell'utente

### assets

**Scopo:** Definire lo scope del programma (domini, API, app mobile)

```sql
CREATE TABLE assets (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    program_id UUID NOT NULL REFERENCES programs(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,  -- web, api, mobile, other
    target VARCHAR(255) NOT NULL,  -- *.example.com, api.example.com
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_assets_program_id ON assets(program_id);
```

**Campi chiave:**

- `type` - Categoria dell'asset (web app, API, app mobile, ecc.)
- `target` - L'asset vero e proprio (dominio, endpoint API, nome del pacchetto dell'app)

**Decisioni di design:**

1. **Tabella separata** - Non incorporata in `programs`
   - Motivo: I programmi possono avere molti asset
   - È più semplice aggiungere/rimuovere elementi dello scope
   - Si può interrogare "quali programmi includono example.com?"

### reward_tiers

**Scopo:** Definire gli importi dei bounty per severità

```sql
CREATE TABLE reward_tiers (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    program_id UUID NOT NULL REFERENCES programs(id) ON DELETE CASCADE,
    severity VARCHAR(50) NOT NULL,  -- critical, high, medium, low
    amount INTEGER NOT NULL,  -- in USD cents
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reward_tiers_program_id ON reward_tiers(program_id);
```

**Campi chiave:**

- `severity` - Critical, high, medium, low, informational
- `amount` - Bounty in centesimi di USD (ad es. 50000 = $500.00)

**Decisioni di design:**

1. **Memorizza i centesimi, non i dollari** - Evita problemi di virgola mobile
   - `amount = 50000` (intero)
   - Visualizzato come `$500.00` nella UI
   - Previene errori di arrotondamento

2. **Tier per programma** - Ogni programma imposta i propri bounty
   - Alternativa: Tier predefiniti a livello di piattaforma
   - Motivo: Flessibilità, le aziende hanno budget diversi

### comments

**Scopo:** Comunicazione tra ricercatore e azienda

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    report_id UUID NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    is_internal BOOLEAN DEFAULT FALSE,  -- Internal company notes
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_comments_report_id ON comments(report_id);
CREATE INDEX idx_comments_author_id ON comments(author_id);
```

**Campi chiave:**

- `is_internal` - Note solo per l'azienda (non visibili al ricercatore)
- `content` - Testo formattato in Markdown

**Decisioni di design:**

1. **Tabella separata** - Non incorporata in `reports`
   - Motivo: I report possono avere molti commenti
   - L'ordine cronologico è preservato
   - È possibile notificare i nuovi commenti

2. **Commenti interni** - `is_internal = true`
   - L'azienda può discutere il report in privato
   - Il ricercatore non li vede mai
   - Utili per le note di triage

### attachments

**Scopo:** File di proof-of-concept (screenshot, video, script)

```sql
CREATE TABLE attachments (
    id UUID PRIMARY KEY DEFAULT uuid_v7(),
    report_id UUID NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
    file_name VARCHAR(255) NOT NULL,
    file_size INTEGER NOT NULL,  -- in bytes
    file_type VARCHAR(100) NOT NULL,  -- image/png, video/mp4, etc.
    s3_key VARCHAR(255) NOT NULL,  -- S3 object key
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_attachments_report_id ON attachments(report_id);
```

**Campi chiave:**

- `file_name` - Nome originale del file
- `file_size` - In byte
- `file_type` - Tipo MIME
- `s3_key` - Chiave di storage S3 (il file vero e proprio non è nel database)

**Decisioni di design:**

1. **Solo metadati** - I file sono memorizzati in S3, non nel database
   - Motivo: Il database serve per dati strutturati, non per blob binari
   - Performance migliori
   - Storage più economico

2. **Nessun soft delete** - Quando un report viene eliminato, vengono eliminati anche gli allegati
   - Anche gli oggetti S3 vengono eliminati (tramite lifecycle policy o webhook)

---

## Decisioni di design

### UUID v7 vs Auto-Increment

**ID auto-increment:**
```sql
id SERIAL PRIMARY KEY  -- 1, 2, 3, 4, ...
```

Problemi:
- Prevedibili (si possono enumerare tutti i record indovinando gli ID)
- Non univoci a livello globale (non si possono unire i database)
- Richiedono un round-trip al database per essere generati

**UUID v4 (casuale):**
```sql
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

Problemi:
- Casuale = pessimo per gli indici del database
- Non ordinabile nel tempo (`ORDER BY id` non ha significato)

**UUID v7 (il meglio di entrambi):**
```sql
id UUID PRIMARY KEY DEFAULT uuid_v7()
```

Vantaggi:
- Ordinabile nel tempo (i primi 48 bit = timestamp Unix in ms)
- Univoco a livello globale (nessuna collisione)
- Buono per gli indici (ordine lessicografico = ordine cronologico)
- Sicuro (i bit rimanenti sono casuali)

```
018d3f54-8c3a-7000-a234-56789abcdef0
^^^^^^^^^^^^^^^^ ← timestamp (2024-01-15 12:34:56.789)
                ^^^^^^^^^^^^^^^^^^^^ ← random
```

### Pattern SafeEnum

**Problema:** L'enum predefinito di SQLAlchemy memorizza il nome Python:

```python
class Status(enum.Enum):
    ACTIVE = "active"
    PAUSED = "paused"

# Database stores: "ACTIVE" (the Python name)
```

Se rinomini l'enum:

```python
class Status(enum.Enum):
    RUNNING = "active"  # Renamed
    PAUSED = "paused"

# Database has "ACTIVE", Python doesn't recognize it → breaks!
```

**Soluzione: SafeEnum memorizza il valore:**

```python
class Status(SafeEnum):
    ACTIVE = "active"
    PAUSED = "paused"

# Database stores: "active" (the value)
```

Ora puoi rinominare in sicurezza:

```python
class Status(SafeEnum):
    RUNNING = "active"  # Value is still "active"
    PAUSED = "paused"

# Database has "active", maps to Status.RUNNING ✓
```

### Cascade Delete

Quando un record padre viene eliminato, cosa succede ai figli?

**Opzioni:**

1. `ON DELETE CASCADE` - Elimina i figli
2. `ON DELETE SET NULL` - Imposta la foreign key a NULL
3. `ON DELETE RESTRICT` - Impedisce l'eliminazione se esistono figli

**Le nostre scelte:**

```sql
-- User deleted → delete all their refresh tokens
user_id UUID REFERENCES users(id) ON DELETE CASCADE

-- User deleted → delete all their reports
researcher_id UUID REFERENCES users(id) ON DELETE CASCADE

-- Program deleted → delete all reports
program_id UUID REFERENCES programs(id) ON DELETE CASCADE

-- Report deleted → set duplicate_of_id to NULL
duplicate_of_id UUID REFERENCES reports(id) ON DELETE SET NULL
```

**Perché CASCADE per i report?**
- Quando un programma viene eliminato, i suoi report diventano orfani
- Non ha senso conservare i report di programmi eliminati
- Alternativa: Soft delete dei programmi (`deleted_at`)

**Perché SET NULL per i duplicati?**
- Se il report originale viene eliminato, i duplicati possono stare in piedi da soli
- Non applicare l'eliminazione a cascata (il duplicato potrebbe essere errato)

### Lazy Loading disabilitato

Il comportamento predefinito di SQLAlchemy:

```python
user = await session.get(User, user_id)
programs = user.programs  # ← Implicit query! (N+1 problem)
```

**Problema:** Ogni accesso scatena una query al database

**Soluzione:** Imposta `lazy="raise"`

```python
class User(Base):
    programs: Mapped[list[Program]] = relationship(
        back_populates="company",
        lazy="raise",  # ← Raise error on lazy load
    )

# Now you must explicitly load:
stmt = select(User).options(selectinload(User.programs))
user = await session.execute(stmt)
```

Obbliga gli sviluppatori a pensare all'efficienza delle query.

---

## Indici e performance

### Quando creare un indice

**Indicizza le colonne che sono:**
1. Usate nelle clausole WHERE (`WHERE email = ?`)
2. Usate nelle condizioni di JOIN (`ON users.id = reports.researcher_id`)
3. Usate per l'unicità (`UNIQUE INDEX`)
4. Usate per l'ordinamento (`ORDER BY created_at`)

**Non indicizzare:**
- Colonne a bassa cardinalità (`is_active` - solo 2 valori)
- Colonne interrogate raramente
- Tabelle piccole (< 1000 righe)

### I nostri indici

```sql
-- Unique indexes (also enforce uniqueness)
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_programs_slug ON programs(slug);

-- Foreign key indexes (for JOINs)
CREATE INDEX idx_reports_program_id ON reports(program_id);
CREATE INDEX idx_reports_researcher_id ON reports(researcher_id);
CREATE INDEX idx_comments_report_id ON comments(report_id);

-- Filter indexes (for WHERE clauses)
CREATE INDEX idx_programs_status ON programs(status);
CREATE INDEX idx_reports_status ON reports(status);
```

### Performance delle query

**Query lenta:**
```python
# Gets all reports, then filters in Python
reports = await session.execute(select(Report))
active_reports = [r for r in reports if r.status == "new"]
```

**Query veloce:**
```python
# Filters in database (uses index)
stmt = select(Report).where(Report.status == ReportStatus.NEW)
reports = await session.execute(stmt)
```

**EXPLAIN ANALYZE:**

```sql
EXPLAIN ANALYZE
SELECT * FROM reports WHERE status = 'new';

-- Good: Index Scan using idx_reports_status
-- Bad: Seq Scan on reports
```

---

## Migrazioni

### Perché Alembic?

- Version control per lo schema del database
- Generazione automatica delle migrazioni
- Supporto al rollback
- Collaborazione di team (tutti hanno lo stesso schema)

### Creare le migrazioni

```bash
# 1. Modify a model
class User(Base):
    bio: Mapped[str | None] = mapped_column(Text, default=None)  # Added

# 2. Generate migration
just migration "Add user bio field"

# 3. Review generated migration
# alembic/versions/20240115_add_user_bio.py
def upgrade():
    op.add_column('users', sa.Column('bio', sa.Text(), nullable=True))

def downgrade():
    op.drop_column('users', 'bio')

# 4. Apply migration
just migrate head
```

### Best practice per le migrazioni

1. **Rivedi sempre le migrazioni generate** - Alembic potrebbe non rilevare tutto
2. **Testa il rollback** - Assicurati che `downgrade()` funzioni
3. **Aggiungi gli indici in migrazioni separate** - Possono richiedere tempo su tabelle grandi
4. **Usa le migrazioni per le modifiche ai dati** - Non solo per lo schema

**Esempio di migrazione dei dati:**

```python
def upgrade():
    # Add column
    op.add_column('users', sa.Column('reputation_score', sa.Integer(), default=0))

    # Backfill existing users
    op.execute("UPDATE users SET reputation_score = 0 WHERE reputation_score IS NULL")

    # Make NOT NULL
    op.alter_column('users', 'reputation_score', nullable=False)
```

---

## Query comuni

### Ottenere i programmi attivi

```python
stmt = (
    select(Program)
    .where(Program.status == ProgramStatus.ACTIVE)
    .order_by(Program.created_at.desc())
)
programs = await session.execute(stmt)
```

### Ottenere i report di un utente

```python
stmt = (
    select(Report)
    .where(Report.researcher_id == user_id)
    .order_by(Report.created_at.desc())
)
reports = await session.execute(stmt)
```

### Ottenere un report con i commenti (Eager Loading)

```python
from sqlalchemy.orm import selectinload

stmt = (
    select(Report)
    .where(Report.id == report_id)
    .options(
        selectinload(Report.comments),
        selectinload(Report.attachments),
    )
)
report = await session.execute(stmt)
```

### Contare i report per stato

```python
from sqlalchemy import func

stmt = (
    select(Report.status, func.count(Report.id))
    .where(Report.program_id == program_id)
    .group_by(Report.status)
)
results = await session.execute(stmt)
```

### Trovare i report duplicati

```python
stmt = (
    select(Report)
    .where(Report.duplicate_of_id == original_report_id)
)
duplicates = await session.execute(stmt)
```

---

## Conclusione

Punti chiave:

1. **UUID v7** - ID ordinabili nel tempo e univoci a livello globale
2. **SafeEnum** - Memorizza i valori degli enum, non i nomi
3. **Indici** - Indicizza le foreign key, le colonne univoche e le colonne di filtro
4. **Migrazioni** - Version control per lo schema del database
5. **Relazioni** - Usa `lazy="raise"` per prevenire le query N+1
6. **Cascade Delete** - Pensa a cosa succede quando il padre viene eliminato

Per maggiori informazioni:
- Architettura del sistema: [ARCHITECTURE.it.md](./ARCHITECTURE.it.md)
- Design pattern: [PATTERNS.it.md](./PATTERNS.it.md)
- Funzionalità di sicurezza: [SECURITY.it.md](./SECURITY.it.md)
- Tutorial pratico: [GETTING-STARTED.it.md](./GETTING-STARTED.it.md)
