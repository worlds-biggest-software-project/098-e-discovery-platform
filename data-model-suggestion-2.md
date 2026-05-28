# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: E-Discovery Platform · Created: 2026-05-12

## Philosophy

This model makes the audit trail the primary data structure, not an afterthought. Every state change in the e-discovery lifecycle — a legal hold issued, a document ingested, a reviewer coding a document as privileged, a production delivered — is recorded as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised materialised views (projections) serve the application's query needs.

This approach is architecturally aligned with the legal requirement for defensibility: in e-discovery, the ability to prove exactly what happened, when, and by whom is not a nice-to-have — it is a FRCP Rule 37 obligation. Event sourcing makes this proof structural rather than bolted-on. Courts can be shown the complete, tamper-evident event history for any document, review decision, or production. Bi-temporal queries ("what was the privilege designation for document X as of 3pm on March 15th?") become trivial — they are just event replay with a cutoff timestamp.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (commands that produce events) from read operations (queries against materialised views). This enables the review workspace to serve sub-second queries from denormalised read models while the write side maintains strict event ordering and consistency.

**Best for:** Organisations where full audit trail defensibility is the paramount requirement — government agencies, AmLaw 200 firms handling high-stakes litigation, and environments where FRCP Rule 37 sanctions risk drives architecture decisions.

**Trade-offs:**
- Pro: Complete, immutable, tamper-evident audit trail as a structural guarantee
- Pro: Bi-temporal queries are native — "what was true at time X?" is just event replay
- Pro: Event replay enables AI training on historical reviewer behaviour patterns
- Pro: Schema evolution is additive — new event types never break existing events
- Con: Higher implementation complexity; requires event store infrastructure and projection builders
- Con: Eventual consistency between event store and read models requires careful handling
- Con: Debugging requires understanding event sequences rather than inspecting table state
- Con: Storage overhead from storing every state change, not just current state
- Con: Materialised views must be rebuilt if projection logic changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EDRM Framework | Each EDRM stage transition is a domain event; the event stream IS the EDRM audit trail |
| ISO/IEC 27050-3 | ESI lifecycle events map to ISO 27050-3 code of practice stages |
| ISO/IEC 27037 | Chain-of-custody is an event sequence: collected → transferred → verified → processed |
| FRCP Rules 26/34/37 | Event immutability provides structural proof against spoliation allegations |
| FRE 502 | Privilege workflow events capture the full decision chain including AI suggestion and attorney confirmation |
| W3C PROV-DM | Event structure aligns with W3C Provenance Data Model (Entity, Activity, Agent triples) |
| SHA-256 (FIPS 180-4) | Event payloads include hash values; event store entries are themselves hashable for integrity chains |
| RFC 5322 / MIME | Email metadata captured as structured event payloads during ingestion events |
| Concordance DAT / Opticon OPT | Production events store Bates assignments; load files are generated from production event projections |

---

## Event Store Core

### Event Store Table

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Stream identification
    stream_type     VARCHAR(100) NOT NULL,  -- 'Matter', 'Document', 'LegalHold', 'CustodianHold', 'ReviewDecision', 'Production', etc.
    stream_id       UUID NOT NULL,          -- the aggregate root ID (e.g., document_id, matter_id)
    -- Event metadata
    event_type      VARCHAR(200) NOT NULL,  -- e.g., 'DocumentIngested', 'CodingDecisionMade', 'PrivilegeDesignated', 'ProductionDelivered'
    event_version   INTEGER NOT NULL,       -- sequence number within the stream (for ordering and optimistic concurrency)
    -- Payload
    payload         JSONB NOT NULL,         -- event-type-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- cross-cutting concerns (correlation_id, causation_id, ip_address, user_agent)
    -- Provenance (W3C PROV-DM aligned)
    tenant_id       UUID NOT NULL,
    actor_id        UUID,                   -- the user or system that caused the event (PROV Agent)
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',  -- 'user', 'system', 'ai_agent'
    -- Timestamps (bi-temporal)
    occurred_at     TIMESTAMPTZ NOT NULL,   -- when the real-world action happened (valid time / application time)
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the event was stored (transaction time / system time)
    -- Integrity
    event_hash      VARCHAR(64),            -- SHA-256 hash of (previous_event_hash + payload) for tamper detection
    previous_hash   VARCHAR(64),            -- hash of the preceding event in this stream
    -- Immutability constraint: no UPDATE or DELETE operations permitted in the application layer
    UNIQUE(stream_type, stream_id, event_version)
);

-- Append-only: no UPDATE or DELETE indexes needed; optimised for INSERT and sequential read
CREATE INDEX idx_events_stream ON events(stream_type, stream_id, event_version);
CREATE INDEX idx_events_tenant ON events(tenant_id, recorded_at);
CREATE INDEX idx_events_type ON events(event_type, recorded_at);
CREATE INDEX idx_events_actor ON events(actor_id, recorded_at);
CREATE INDEX idx_events_occurred ON events(occurred_at);

-- Partition by month for large deployments
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Event Type Taxonomy

```sql
-- Registry of all valid event types with their JSON schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,  -- JSON Schema (Draft 2020-12) for the event payload
    edrm_stage      VARCHAR(50),    -- which EDRM stage this event belongs to
    version         INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Event Types and Payloads

```sql
-- Example: DocumentIngested event
-- payload: {
--   "file_name": "contract_2024.pdf",
--   "file_extension": "pdf",
--   "file_size_bytes": 1048576,
--   "mime_type": "application/pdf",
--   "hash_sha256": "a1b2c3d4...",
--   "hash_md5": "e5f6g7h8...",
--   "custodian_id": "uuid-...",
--   "collection_id": "uuid-...",
--   "source_path": "/Users/jsmith/Documents/contract_2024.pdf",
--   "extracted_text_length": 15420,
--   "page_count": 12,
--   "email_metadata": null
-- }

-- Example: CodingDecisionMade event
-- payload: {
--   "coding_field_name": "Responsiveness",
--   "coding_field_id": "uuid-...",
--   "chosen_value": "Responsive",
--   "chosen_value_id": "uuid-...",
--   "batch_id": "uuid-...",
--   "time_spent_ms": 23400,
--   "ai_suggested": true,
--   "ai_confidence": 0.9234,
--   "ai_suggested_value": "Responsive"
-- }

-- Example: PrivilegeDesignated event
-- payload: {
--   "is_privileged": true,
--   "privilege_type": "attorney_client",
--   "privilege_basis": "Communication between in-house counsel J. Smith and VP Operations regarding pending litigation strategy",
--   "ai_flagged": true,
--   "ai_confidence": 0.8712,
--   "ai_model_version": "priv-v2.1.0",
--   "requires_redaction": false
-- }

-- Example: PrivilegeAttorneyConfirmed event (FRE 502 compliance)
-- payload: {
--   "privilege_designation_event_id": "uuid-of-the-PrivilegeDesignated-event",
--   "attorney_id": "uuid-...",
--   "confirmed": true,
--   "override_ai": false,
--   "notes": null
-- }

-- Example: LegalHoldIssued event
-- payload: {
--   "hold_name": "Smith v. Acme Corp - Preservation",
--   "hold_type": "litigation",
--   "preservation_scope": "All email, chat, and file storage from 2023-01-01 to present",
--   "custodian_ids": ["uuid-1", "uuid-2", "uuid-3"],
--   "notice_subject": "Litigation Hold Notice - Smith v. Acme Corp",
--   "notice_body_hash": "sha256-of-notice-content"
-- }

-- Example: ProductionGenerated event
-- payload: {
--   "production_name": "ACME First Production",
--   "production_number": "PROD-001",
--   "bates_prefix": "ACME",
--   "bates_start": 1,
--   "bates_end": 14523,
--   "document_count": 3241,
--   "image_format": "tiff",
--   "load_file_format": "concordance_dat",
--   "total_size_bytes": 52428800000
-- }
```

---

## Materialised Read Models (Projections)

These tables are derived from the event store. They can be rebuilt from scratch by replaying events.

### Matter Projection

```sql
CREATE TABLE mv_matters (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    matter_number   VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    matter_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    jurisdiction    VARCHAR(10),
    court_name      VARCHAR(500),
    case_number     VARCHAR(200),
    date_filed      DATE,
    date_closed     DATE,
    lead_attorney_id UUID,
    -- Aggregated statistics (updated by projection handlers)
    total_documents BIGINT DEFAULT 0,
    total_reviewed  BIGINT DEFAULT 0,
    total_privileged BIGINT DEFAULT 0,
    total_productions INTEGER DEFAULT 0,
    active_holds    INTEGER DEFAULT 0,
    -- Projection metadata
    last_event_id   UUID,           -- last event applied to this projection
    last_event_version INTEGER,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
);

CREATE INDEX idx_mv_matters_tenant ON mv_matters(tenant_id);
CREATE INDEX idx_mv_matters_status ON mv_matters(tenant_id, status);
```

### Document Projection

```sql
CREATE TABLE mv_documents (
    id              UUID PRIMARY KEY,
    matter_id       UUID NOT NULL,
    collection_id   UUID,
    doc_id          VARCHAR(100) NOT NULL,
    parent_id       UUID,
    family_id       UUID,
    -- Content snapshot
    file_name       VARCHAR(1000),
    file_extension  VARCHAR(50),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(255),
    hash_sha256     VARCHAR(64) NOT NULL,
    page_count      INTEGER,
    -- Email metadata (RFC 5322)
    email_from      VARCHAR(500),
    email_to        TEXT[],
    email_cc        TEXT[],
    email_subject   VARCHAR(2000),
    email_date      TIMESTAMPTZ,
    email_message_id VARCHAR(500),
    -- Current review state (latest coding decisions)
    current_coding  JSONB DEFAULT '{}',
    -- Example: {"Responsiveness": "Responsive", "Issues": ["Patent", "Trade Secret"]}
    -- Privilege state
    is_privileged   BOOLEAN,
    privilege_type  VARCHAR(100),
    privilege_attorney_confirmed BOOLEAN DEFAULT false,
    -- TAR scores
    tar_scores      JSONB DEFAULT '{}',
    -- Example: {"model-uuid-1": 0.9234, "model-uuid-2": 0.1456}
    -- Processing
    processing_status VARCHAR(50),
    custodian_id    UUID,
    custodian_name  VARCHAR(500),  -- denormalized for display
    source_path     VARCHAR(2000),
    -- Deduplication
    is_duplicate    BOOLEAN DEFAULT false,
    is_nist         BOOLEAN DEFAULT false,
    -- Tags (denormalized array for fast filtering)
    tag_names       TEXT[] DEFAULT '{}',
    -- Bates numbers from productions (denormalized)
    bates_numbers   JSONB DEFAULT '{}',
    -- Example: {"PROD-001": {"begin": "ACME0000001", "end": "ACME0000012"}}
    -- Projection metadata
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    ingested_at     TIMESTAMPTZ,
    UNIQUE(matter_id, doc_id)
);

CREATE INDEX idx_mv_docs_matter ON mv_documents(matter_id);
CREATE INDEX idx_mv_docs_custodian ON mv_documents(custodian_id);
CREATE INDEX idx_mv_docs_hash ON mv_documents(hash_sha256);
CREATE INDEX idx_mv_docs_family ON mv_documents(family_id);
CREATE INDEX idx_mv_docs_processing ON mv_documents(matter_id, processing_status);
CREATE INDEX idx_mv_docs_privilege ON mv_documents(matter_id, is_privileged);
CREATE INDEX idx_mv_docs_tags ON mv_documents USING GIN(tag_names);
CREATE INDEX idx_mv_docs_coding ON mv_documents USING GIN(current_coding);
```

### Legal Hold Projection

```sql
CREATE TABLE mv_legal_holds (
    id              UUID PRIMARY KEY,
    matter_id       UUID NOT NULL,
    hold_name       VARCHAR(500) NOT NULL,
    hold_type       VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    date_issued     TIMESTAMPTZ,
    date_released   TIMESTAMPTZ,
    issued_by       UUID,
    total_custodians INTEGER DEFAULT 0,
    acknowledged_count INTEGER DEFAULT 0,
    pending_count   INTEGER DEFAULT 0,
    escalated_count INTEGER DEFAULT 0,
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_holds_matter ON mv_legal_holds(matter_id);
CREATE INDEX idx_mv_holds_status ON mv_legal_holds(status);

CREATE TABLE mv_custodian_holds (
    legal_hold_id   UUID NOT NULL,
    custodian_id    UUID NOT NULL,
    custodian_name  VARCHAR(500),
    custodian_email VARCHAR(320),
    status          VARCHAR(50) NOT NULL,
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    escalation_count INTEGER DEFAULT 0,
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (legal_hold_id, custodian_id)
);
```

### Review Progress Projection

```sql
CREATE TABLE mv_review_progress (
    matter_id       UUID NOT NULL,
    -- Overall progress
    total_documents BIGINT DEFAULT 0,
    reviewed_documents BIGINT DEFAULT 0,
    responsive_documents BIGINT DEFAULT 0,
    privileged_documents BIGINT DEFAULT 0,
    not_responsive_documents BIGINT DEFAULT 0,
    -- AI metrics
    ai_suggested_count BIGINT DEFAULT 0,
    ai_overridden_count BIGINT DEFAULT 0,
    avg_review_time_ms DECIMAL(10,2),
    -- Per-reviewer breakdown (JSONB for flexible display)
    reviewer_stats  JSONB DEFAULT '{}',
    -- Example: {"reviewer-uuid": {"reviewed": 1234, "avg_time_ms": 34500, "override_rate": 0.05}}
    -- TAR model performance
    tar_stats       JSONB DEFAULT '{}',
    -- Example: {"model-uuid": {"recall": 0.94, "precision": 0.87, "scored": 50000}}
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (matter_id)
);
```

### Production Projection

```sql
CREATE TABLE mv_productions (
    id              UUID PRIMARY KEY,
    matter_id       UUID NOT NULL,
    production_name VARCHAR(500) NOT NULL,
    production_number VARCHAR(100),
    status          VARCHAR(50) NOT NULL,
    bates_prefix    VARCHAR(20),
    bates_start     BIGINT,
    bates_end       BIGINT,
    document_count  INTEGER DEFAULT 0,
    image_format    VARCHAR(20),
    load_file_format VARCHAR(50),
    delivered_to    VARCHAR(500),
    delivered_at    TIMESTAMPTZ,
    created_by      UUID,
    last_event_id   UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_prods_matter ON mv_productions(matter_id);

CREATE TABLE mv_production_documents (
    production_id   UUID NOT NULL,
    document_id     UUID NOT NULL,
    bates_begin     VARCHAR(50) NOT NULL,
    bates_end       VARCHAR(50) NOT NULL,
    page_count      INTEGER DEFAULT 1,
    PRIMARY KEY (production_id, document_id)
);

CREATE INDEX idx_mv_prod_docs_bates ON mv_production_documents(bates_begin);
```

---

## Reference Data Tables

These are NOT event-sourced — they are stable reference data that events reference by ID.

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    oidc_subject    VARCHAR(255),
    oidc_issuer     VARCHAR(512),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    category        VARCHAR(50) NOT NULL
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    matter_id       UUID,  -- NULL = global; non-NULL = matter-scoped
    granted_by      UUID REFERENCES users(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(matter_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE TABLE custodians (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    employee_id     VARCHAR(100),
    first_name      VARCHAR(255) NOT NULL,
    last_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(320) NOT NULL,
    department      VARCHAR(255),
    title           VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- Coding field definitions (reference data for events)
CREATE TABLE coding_fields (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL,
    field_name      VARCHAR(255) NOT NULL,
    field_type      VARCHAR(50) NOT NULL,
    is_required     BOOLEAN NOT NULL DEFAULT false,
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, field_name)
);

CREATE TABLE coding_field_choices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coding_field_id UUID NOT NULL REFERENCES coding_fields(id) ON DELETE CASCADE,
    choice_value    VARCHAR(500) NOT NULL,
    display_order   INTEGER NOT NULL DEFAULT 0,
    UNIQUE(coding_field_id, choice_value)
);
```

---

## Temporal Query Examples

### "What was the privilege status of document X on March 15th?"

```sql
-- Replay privilege-related events for a specific document up to a cutoff time
SELECT
    e.event_type,
    e.payload->>'is_privileged' AS is_privileged,
    e.payload->>'privilege_type' AS privilege_type,
    e.payload->>'privilege_basis' AS privilege_basis,
    e.occurred_at,
    u.display_name AS actor
FROM events e
LEFT JOIN users u ON u.id = e.actor_id
WHERE e.stream_type = 'Document'
  AND e.stream_id = 'target-document-uuid'
  AND e.event_type IN ('PrivilegeDesignated', 'PrivilegeAttorneyConfirmed', 'PrivilegeOverridden', 'PrivilegeWaived')
  AND e.occurred_at <= '2026-03-15 23:59:59+00'
ORDER BY e.event_version ASC;
```

### "Show the complete chain of custody for this collection"

```sql
-- Full event history for a collection stream — this IS the chain of custody
SELECT
    e.event_type,
    e.occurred_at,
    e.payload,
    u.display_name AS actor,
    e.metadata->>'ip_address' AS ip_address,
    e.event_hash
FROM events e
LEFT JOIN users u ON u.id = e.actor_id
WHERE e.stream_type = 'Collection'
  AND e.stream_id = 'target-collection-uuid'
ORDER BY e.event_version ASC;
```

### "How many documents did reviewer X review per hour last week, with override rate?"

```sql
-- Aggregate reviewer activity from events
SELECT
    date_trunc('hour', e.occurred_at) AS hour,
    COUNT(*) AS decisions,
    AVG((e.payload->>'time_spent_ms')::int) AS avg_time_ms,
    SUM(CASE WHEN e.payload->>'ai_suggested' = 'true'
              AND e.payload->>'chosen_value' != e.payload->>'ai_suggested_value'
         THEN 1 ELSE 0 END)::decimal / NULLIF(COUNT(*), 0) AS override_rate
FROM events e
WHERE e.event_type = 'CodingDecisionMade'
  AND e.actor_id = 'reviewer-uuid'
  AND e.occurred_at >= now() - interval '7 days'
GROUP BY date_trunc('hour', e.occurred_at)
ORDER BY hour;
```

### Verify event chain integrity (tamper detection)

```sql
-- Check that hash chain is unbroken for a stream
WITH ordered_events AS (
    SELECT
        event_id,
        event_version,
        event_hash,
        previous_hash,
        LAG(event_hash) OVER (ORDER BY event_version) AS expected_previous_hash
    FROM events
    WHERE stream_type = 'Document'
      AND stream_id = 'target-document-uuid'
)
SELECT *
FROM ordered_events
WHERE event_version > 1
  AND previous_hash != expected_previous_hash;
-- Empty result = chain is intact; any rows = tampering detected
```

---

## Projection Rebuild Process

```sql
-- To rebuild a projection from scratch:
-- 1. Truncate the materialised view table
-- 2. Replay all events of the relevant type in order
-- 3. Apply each event to update the projection

-- Example pseudocode for document projection rebuild:
-- TRUNCATE mv_documents;
-- FOR EACH event IN (SELECT * FROM events WHERE stream_type = 'Document' ORDER BY stream_id, event_version):
--   CASE event.event_type:
--     'DocumentIngested':  INSERT INTO mv_documents (id, ...) VALUES (event.stream_id, event.payload->>'file_name', ...)
--     'CodingDecisionMade': UPDATE mv_documents SET current_coding = jsonb_set(current_coding, ...) WHERE id = event.stream_id
--     'PrivilegeDesignated': UPDATE mv_documents SET is_privileged = (event.payload->>'is_privileged')::boolean WHERE id = event.stream_id
--     'DocumentTagged':     UPDATE mv_documents SET tag_names = array_append(tag_names, event.payload->>'tag_name') WHERE id = event.stream_id
--     ...
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events (append-only source of truth), event_type_registry |
| Reference Data | 8 | tenants, users, roles, permissions, role_permissions, user_roles, custodians, coding_fields, coding_field_choices |
| Materialised Views | 8 | mv_matters, mv_documents, mv_legal_holds, mv_custodian_holds, mv_review_progress, mv_productions, mv_production_documents |
| **Total** | **~18** | Plus additional projections as query patterns emerge |

---

## Key Design Decisions

1. **Single event table as source of truth** — all domain state changes flow through one append-only table. This is the defensibility guarantee: the event log cannot be altered without breaking the hash chain, and it captures everything needed for FRCP Rule 37 compliance.

2. **Hash-chained events for tamper evidence** — each event stores the SHA-256 hash of (previous_event_hash + payload), creating a blockchain-like integrity chain per stream. If any event is modified, the chain breaks and the tampering is detectable. This exceeds the audit requirements of ISO 27001 and NIST 800-53.

3. **Bi-temporal timestamps** — `occurred_at` (when the action happened in the real world) and `recorded_at` (when the database stored it) enable both application-time and system-time queries. This is critical for privilege determinations: "the attorney confirmed the privilege call at 3pm" vs. "the system recorded the confirmation at 3:01pm."

4. **W3C PROV-DM alignment** — events capture Entity (stream_id), Activity (event_type), and Agent (actor_id/actor_type), aligning with the W3C Provenance Data Model for standardised provenance tracking.

5. **Materialised views are disposable** — all `mv_*` tables can be dropped and rebuilt by replaying events. This means read model schema changes do not require data migrations — just rebuild the projection with the new schema.

6. **Event payloads as JSONB** — each event type has its own payload schema (validated against JSON Schema in the event_type_registry). This avoids the wide-column problem of having every event field as a column, while still enabling JSONB indexing for common query patterns.

7. **Additive schema evolution** — new event types can be added without modifying existing events or tables. A new feature (e.g., "redaction applied") simply introduces a new event type (`RedactionApplied`) with its own payload schema.

8. **Coding decisions in denormalised JSONB on document projection** — `mv_documents.current_coding` stores the latest coding values as a JSONB object for fast filtering. The full coding history lives in the event stream.

9. **Actor type distinguishes human from AI** — `actor_type` ('user', 'system', 'ai_agent') enables querying "show me all AI-generated privilege flags" vs. "show me all attorney-confirmed privilege calls" — essential for FRE 502 compliance reporting.

10. **Separate projection for review progress** — `mv_review_progress` is a single-row-per-matter aggregate that powers the review dashboard. It avoids expensive COUNT queries over millions of documents by maintaining running totals updated by projection handlers.
