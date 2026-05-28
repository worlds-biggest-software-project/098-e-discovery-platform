# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: E-Discovery Platform · Created: 2026-05-12

## Philosophy

This model follows classical normalized relational database design, mapping every EDRM concept to a dedicated table with strict foreign key relationships. Each entity in the e-discovery lifecycle — matters, custodians, legal holds, documents, review decisions, productions — gets its own table with well-defined columns, constraints, and indexes. Junction tables handle many-to-many relationships (custodians to matters, documents to tags, reviewers to batches).

The design mirrors how Relativity structures its object model: workspaces contain documents, documents have fields, reviewers make coding decisions, and productions package subsets for delivery. By normalizing fully, we get strong referential integrity, straightforward SQL queries, and the ability to enforce business rules (e.g., "a production cannot include un-reviewed documents") at the database level via constraints and triggers.

This approach is best suited for teams that value data integrity above all else, expect complex cross-entity reporting (e.g., "show me all privilege calls across all matters for custodian X"), and operate in regulatory environments where schema clarity matters for audit and compliance certification.

**Best for:** Organisations prioritising data integrity, complex cross-entity queries, and regulatory compliance certification (ISO 27001, SOC 2, NIST 800-53).

**Trade-offs:**
- Pro: Strong referential integrity; the database enforces business rules
- Pro: Straightforward SQL queries; no JSONB parsing or event replay needed
- Pro: Clear schema documentation for auditors and compliance officers
- Pro: Well-understood by most development teams; largest talent pool
- Con: High table count (~55-65 tables); schema migrations are heavyweight
- Con: Adding jurisdiction-specific or matter-specific fields requires DDL changes
- Con: Historical queries ("what was true on date X?") require explicit versioning tables
- Con: Schema rigidity can slow feature iteration in early development phases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EDRM Framework | Each EDRM stage (Identification → Production) maps to a table group; workflow state machines enforce stage transitions |
| EDRM XML v1.1 | Document and metadata tables mirror EDRM XML element structure for native import/export |
| ISO/IEC 27050-3 | ESI lifecycle tables (collection, processing, review, production) align with ISO 27050-3 code of practice |
| ISO/IEC 27037 | Chain-of-custody fields on collection records (hash values, timestamps, collector identity) |
| FRCP Rules 26/34/37 | Legal hold and production tables enforce preservation obligations and proportionality tracking |
| FRE 502 | Privilege review tables with attorney-in-the-loop confirmation workflow |
| ISO 3166-1/2 | Jurisdiction codes for multi-region matters using ISO country/subdivision codes |
| SHA-256 (FIPS 180-4) | Hash columns on documents and evidence items for forensic integrity verification |
| RFC 5322 / MIME | Email-specific metadata fields (from, to, cc, bcc, subject, message-id, in-reply-to) |
| Concordance DAT / Opticon OPT | Production tables store Bates ranges and image paths in load-file-compatible structure |
| OAuth 2.0 / OIDC | User authentication tables support SSO integration with enterprise identity providers |

---

## Core Infrastructure Tables

### Tenants and Organisations

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',  -- ISO 3166-1 alpha-2
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
```

### Users and Authentication

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),  -- NULL for SSO-only users
    oidc_subject    VARCHAR(255),  -- OpenID Connect subject identifier
    oidc_issuer     VARCHAR(512),  -- OIDC issuer URL
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
```

### Roles and Permissions (RBAC)

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'reviewer', 'supervising_attorney', 'case_manager', 'admin'
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,  -- system roles cannot be deleted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- e.g., 'matter.create', 'document.review', 'production.export'
    description     TEXT,
    category        VARCHAR(50) NOT NULL  -- e.g., 'matter', 'document', 'production', 'admin'
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    matter_id       UUID REFERENCES matters(id) ON DELETE CASCADE,  -- NULL = global role; non-NULL = matter-scoped
    granted_by      UUID REFERENCES users(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(matter_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_matter ON user_roles(matter_id);
```

---

## Matter Management Tables

```sql
CREATE TABLE matters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_number   VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    matter_type     VARCHAR(50) NOT NULL,  -- 'litigation', 'investigation', 'regulatory', 'foia', 'arbitration'
    status          VARCHAR(50) NOT NULL DEFAULT 'active',  -- 'active', 'on_hold', 'closed', 'archived'
    jurisdiction    VARCHAR(10),  -- ISO 3166-1 alpha-2 or ISO 3166-2 subdivision
    court_name      VARCHAR(500),
    case_number     VARCHAR(200),
    judge_name      VARCHAR(255),
    date_filed      DATE,
    date_closed     DATE,
    lead_attorney_id UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
);

CREATE INDEX idx_matters_tenant ON matters(tenant_id);
CREATE INDEX idx_matters_status ON matters(tenant_id, status);
CREATE INDEX idx_matters_type ON matters(tenant_id, matter_type);

CREATE TABLE matter_parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id) ON DELETE CASCADE,
    party_name      VARCHAR(500) NOT NULL,
    party_type      VARCHAR(50) NOT NULL,  -- 'plaintiff', 'defendant', 'third_party', 'government_agency'
    party_role      VARCHAR(100),  -- 'opposing_party', 'co_defendant', 'intervenor'
    counsel_name    VARCHAR(500),
    counsel_firm    VARCHAR(500),
    counsel_email   VARCHAR(320),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_matter_parties_matter ON matter_parties(matter_id);
```

---

## Legal Hold Tables

```sql
CREATE TABLE custodians (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    employee_id     VARCHAR(100),  -- HR system identifier
    first_name      VARCHAR(255) NOT NULL,
    last_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(320) NOT NULL,
    department      VARCHAR(255),
    title           VARCHAR(255),
    manager_name    VARCHAR(255),
    manager_email   VARCHAR(320),
    location        VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,  -- still employed/available
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_custodians_tenant ON custodians(tenant_id);
CREATE INDEX idx_custodians_name ON custodians(tenant_id, last_name, first_name);

CREATE TABLE custodian_data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    custodian_id    UUID NOT NULL REFERENCES custodians(id) ON DELETE CASCADE,
    source_type     VARCHAR(100) NOT NULL,  -- 'email_mailbox', 'onedrive', 'slack', 'local_drive', 'shared_drive', 'mobile_device'
    source_name     VARCHAR(500) NOT NULL,  -- e.g., 'john.smith@company.com', 'OneDrive - Marketing'
    source_location VARCHAR(1000),  -- path, URL, or identifier
    estimated_size_bytes BIGINT,
    notes           TEXT,
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_custodian_data_sources ON custodian_data_sources(custodian_id);

CREATE TABLE legal_holds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    hold_name       VARCHAR(500) NOT NULL,
    hold_type       VARCHAR(50) NOT NULL DEFAULT 'litigation',  -- 'litigation', 'regulatory', 'investigation'
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',  -- 'draft', 'active', 'released', 'expired'
    description     TEXT,
    preservation_scope TEXT,  -- what data types and date ranges to preserve
    date_issued     TIMESTAMPTZ,
    date_released   TIMESTAMPTZ,
    issued_by       UUID NOT NULL REFERENCES users(id),
    released_by     UUID REFERENCES users(id),
    reminder_interval_days INTEGER DEFAULT 90,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_legal_holds_matter ON legal_holds(matter_id);
CREATE INDEX idx_legal_holds_status ON legal_holds(status);

CREATE TABLE legal_hold_notices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_hold_id   UUID NOT NULL REFERENCES legal_holds(id) ON DELETE CASCADE,
    notice_type     VARCHAR(50) NOT NULL,  -- 'initial', 'reminder', 'release', 'escalation'
    subject         VARCHAR(500) NOT NULL,
    body_html       TEXT NOT NULL,
    body_text       TEXT NOT NULL,
    sent_at         TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE legal_hold_custodians (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_hold_id   UUID NOT NULL REFERENCES legal_holds(id) ON DELETE CASCADE,
    custodian_id    UUID NOT NULL REFERENCES custodians(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'notified', 'acknowledged', 'escalated', 'released'
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    escalated_at    TIMESTAMPTZ,
    released_at     TIMESTAMPTZ,
    escalation_count INTEGER NOT NULL DEFAULT 0,
    notes           TEXT,
    UNIQUE(legal_hold_id, custodian_id)
);

CREATE INDEX idx_lh_custodians_hold ON legal_hold_custodians(legal_hold_id);
CREATE INDEX idx_lh_custodians_custodian ON legal_hold_custodians(custodian_id);
CREATE INDEX idx_lh_custodians_status ON legal_hold_custodians(status);
```

---

## Collection Tables

```sql
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    custodian_id    UUID NOT NULL REFERENCES custodians(id),
    data_source_id  UUID REFERENCES custodian_data_sources(id),
    collection_method VARCHAR(100) NOT NULL,  -- 'forensic_image', 'targeted_collection', 'cloud_api', 'self_collection'
    status          VARCHAR(50) NOT NULL DEFAULT 'planned',  -- 'planned', 'in_progress', 'completed', 'failed', 'verified'
    date_range_start DATE,
    date_range_end  DATE,
    search_terms    TEXT[],  -- boolean search terms used for targeted collection
    collected_by    UUID REFERENCES users(id),
    collected_at    TIMESTAMPTZ,
    total_items     BIGINT,
    total_size_bytes BIGINT,
    -- Chain of custody / forensic integrity (ISO/IEC 27037)
    source_hash_sha256 VARCHAR(64),  -- SHA-256 hash of source at collection (FIPS 180-4)
    collection_hash_sha256 VARCHAR(64),  -- SHA-256 hash of collected data
    chain_of_custody_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collections_matter ON collections(matter_id);
CREATE INDEX idx_collections_custodian ON collections(custodian_id);
CREATE INDEX idx_collections_status ON collections(status);
```

---

## Document and Processing Tables

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    collection_id   UUID REFERENCES collections(id),
    -- Document identifiers
    doc_id          VARCHAR(100) NOT NULL,  -- internal unique document ID within the matter
    parent_id       UUID REFERENCES documents(id),  -- parent document (e.g., email for attachment)
    family_id       UUID,  -- groups email + all attachments as a family
    -- Content
    extracted_text  TEXT,
    ocr_text        TEXT,
    text_language   VARCHAR(10),  -- ISO 639-1 language code
    page_count      INTEGER,
    -- File metadata
    file_name       VARCHAR(1000),
    file_extension  VARCHAR(50),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(255),
    -- Hash values for integrity and deduplication (FIPS 180-4 / ISO 27037)
    hash_sha256     VARCHAR(64) NOT NULL,
    hash_md5        VARCHAR(32),  -- legacy; retained for backward compatibility with older productions
    -- Deduplication
    is_duplicate    BOOLEAN NOT NULL DEFAULT false,
    duplicate_of_id UUID REFERENCES documents(id),
    is_nist         BOOLEAN NOT NULL DEFAULT false,  -- de-NISTed (known system file)
    -- Email-specific metadata (RFC 5322)
    email_from      VARCHAR(500),
    email_to        TEXT[],  -- array of recipients
    email_cc        TEXT[],
    email_bcc       TEXT[],
    email_subject   VARCHAR(2000),
    email_date      TIMESTAMPTZ,
    email_message_id VARCHAR(500),  -- RFC 5322 Message-ID
    email_in_reply_to VARCHAR(500),  -- RFC 5322 In-Reply-To for threading
    -- General metadata
    author          VARCHAR(500),
    date_created    TIMESTAMPTZ,
    date_modified   TIMESTAMPTZ,
    date_accessed   TIMESTAMPTZ,
    custodian_id    UUID REFERENCES custodians(id),
    source_path     VARCHAR(2000),  -- original file path or mailbox location
    -- Processing status
    processing_status VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'processing', 'completed', 'error'
    processing_errors TEXT[],
    processed_at    TIMESTAMPTZ,
    -- Timestamps
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, doc_id)
);

CREATE INDEX idx_documents_matter ON documents(matter_id);
CREATE INDEX idx_documents_collection ON documents(collection_id);
CREATE INDEX idx_documents_custodian ON documents(custodian_id);
CREATE INDEX idx_documents_parent ON documents(parent_id);
CREATE INDEX idx_documents_family ON documents(family_id);
CREATE INDEX idx_documents_hash ON documents(hash_sha256);
CREATE INDEX idx_documents_processing ON documents(matter_id, processing_status);
CREATE INDEX idx_documents_email_date ON documents(matter_id, email_date) WHERE email_date IS NOT NULL;
CREATE INDEX idx_documents_mime ON documents(matter_id, mime_type);

-- Near-duplicate clusters
CREATE TABLE near_duplicate_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    similarity_threshold DECIMAL(5,4) NOT NULL DEFAULT 0.9000,
    pivot_document_id UUID NOT NULL REFERENCES documents(id),
    document_count  INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE near_duplicate_members (
    group_id        UUID NOT NULL REFERENCES near_duplicate_groups(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    similarity_score DECIMAL(5,4) NOT NULL,
    PRIMARY KEY (group_id, document_id)
);

CREATE INDEX idx_near_dup_members_doc ON near_duplicate_members(document_id);

-- Email threading
CREATE TABLE email_threads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    thread_root_message_id VARCHAR(500),  -- RFC 5322 Message-ID of the thread root
    subject_normalized VARCHAR(2000),
    document_count  INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE email_thread_members (
    thread_id       UUID NOT NULL REFERENCES email_threads(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    position_in_thread INTEGER NOT NULL,  -- 0 = root, 1 = first reply, etc.
    is_inclusive     BOOLEAN NOT NULL DEFAULT false,  -- true if this is the most inclusive version
    PRIMARY KEY (thread_id, document_id)
);

CREATE INDEX idx_email_thread_members_doc ON email_thread_members(document_id);
```

---

## Document Review Tables

```sql
-- Coding fields (configurable per matter, like Relativity's coding layouts)
CREATE TABLE coding_fields (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    field_name      VARCHAR(255) NOT NULL,
    field_type      VARCHAR(50) NOT NULL,  -- 'single_choice', 'multi_choice', 'text', 'date', 'boolean', 'numeric'
    is_required     BOOLEAN NOT NULL DEFAULT false,
    display_order   INTEGER NOT NULL DEFAULT 0,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, field_name)
);

CREATE TABLE coding_field_choices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coding_field_id UUID NOT NULL REFERENCES coding_fields(id) ON DELETE CASCADE,
    choice_value    VARCHAR(500) NOT NULL,
    display_order   INTEGER NOT NULL DEFAULT 0,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(coding_field_id, choice_value)
);

-- Tags (free-form labels applied to documents)
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    name            VARCHAR(255) NOT NULL,
    color           VARCHAR(7),  -- hex color code
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, name)
);

CREATE TABLE document_tags (
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    tagged_by       UUID NOT NULL REFERENCES users(id),
    tagged_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, tag_id)
);

-- Review batches (work queues for reviewers)
CREATE TABLE review_batches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    batch_name      VARCHAR(255) NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'assigned',  -- 'assigned', 'in_progress', 'completed', 'returned'
    document_count  INTEGER NOT NULL DEFAULT 0,
    reviewed_count  INTEGER NOT NULL DEFAULT 0,
    assigned_by     UUID NOT NULL REFERENCES users(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    due_date        TIMESTAMPTZ
);

CREATE INDEX idx_review_batches_matter ON review_batches(matter_id);
CREATE INDEX idx_review_batches_assignee ON review_batches(assigned_to, status);

CREATE TABLE review_batch_documents (
    batch_id        UUID NOT NULL REFERENCES review_batches(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    position        INTEGER NOT NULL,  -- order within batch
    PRIMARY KEY (batch_id, document_id)
);

-- Coding decisions (one row per document per coding field per reviewer)
CREATE TABLE coding_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id),
    coding_field_id UUID NOT NULL REFERENCES coding_fields(id),
    reviewer_id     UUID NOT NULL REFERENCES users(id),
    batch_id        UUID REFERENCES review_batches(id),
    -- Values (only one is populated based on field_type)
    choice_id       UUID REFERENCES coding_field_choices(id),
    choice_ids      UUID[],  -- for multi_choice fields
    text_value      TEXT,
    date_value      DATE,
    boolean_value   BOOLEAN,
    numeric_value   DECIMAL(18,4),
    -- AI assistance
    ai_suggested    BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   DECIMAL(5,4),  -- 0.0000 to 1.0000
    -- Timestamps
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    time_spent_ms   INTEGER  -- time reviewer spent on this document
);

CREATE INDEX idx_coding_decisions_doc ON coding_decisions(document_id);
CREATE INDEX idx_coding_decisions_field ON coding_decisions(coding_field_id);
CREATE INDEX idx_coding_decisions_reviewer ON coding_decisions(reviewer_id);
CREATE UNIQUE INDEX idx_coding_decisions_unique ON coding_decisions(document_id, coding_field_id, reviewer_id);
```

---

## Privilege Review Tables

```sql
CREATE TABLE privilege_designations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id),
    -- Privilege determination
    is_privileged   BOOLEAN NOT NULL,
    privilege_type  VARCHAR(100),  -- 'attorney_client', 'work_product', 'joint_defense', 'common_interest', 'deliberative_process'
    privilege_basis TEXT,  -- narrative explanation for the privilege log
    -- FRE 502 compliance
    attorney_reviewed BOOLEAN NOT NULL DEFAULT false,  -- an attorney has confirmed the designation
    attorney_id     UUID REFERENCES users(id),  -- the confirming attorney
    attorney_reviewed_at TIMESTAMPTZ,
    -- AI pre-flagging
    ai_flagged      BOOLEAN NOT NULL DEFAULT false,
    ai_confidence   DECIMAL(5,4),
    ai_model_version VARCHAR(100),
    -- Redaction
    requires_redaction BOOLEAN NOT NULL DEFAULT false,
    redaction_notes TEXT,
    -- Status
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',  -- 'pending', 'ai_flagged', 'attorney_confirmed', 'challenged', 'waived'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_privilege_doc ON privilege_designations(document_id);
CREATE INDEX idx_privilege_status ON privilege_designations(status);
CREATE INDEX idx_privilege_type ON privilege_designations(privilege_type);

-- Privilege log entries (formatted for court submission)
CREATE TABLE privilege_log_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    document_id     UUID NOT NULL REFERENCES documents(id),
    privilege_id    UUID NOT NULL REFERENCES privilege_designations(id),
    -- Standard privilege log fields
    bates_begin     VARCHAR(50) NOT NULL,
    bates_end       VARCHAR(50) NOT NULL,
    doc_date        DATE,
    doc_type        VARCHAR(100),  -- 'email', 'memorandum', 'letter', 'report'
    author          VARCHAR(500),
    recipients      TEXT,
    subject         VARCHAR(2000),
    privilege_claimed VARCHAR(200) NOT NULL,  -- 'Attorney-Client Privilege', 'Work Product Doctrine', etc.
    privilege_description TEXT NOT NULL,  -- brief description of why privilege applies
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_priv_log_matter ON privilege_log_entries(matter_id);
CREATE INDEX idx_priv_log_bates ON privilege_log_entries(bates_begin);
```

---

## Production Tables

```sql
CREATE TABLE productions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    production_name VARCHAR(500) NOT NULL,
    production_number VARCHAR(100),  -- e.g., 'PROD-001'
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',  -- 'draft', 'generating', 'qc_review', 'finalised', 'delivered'
    -- Format settings (aligned with EDRM Production Standards v2)
    image_format    VARCHAR(20) NOT NULL DEFAULT 'tiff',  -- 'tiff', 'pdf', 'native'
    load_file_format VARCHAR(50) NOT NULL DEFAULT 'concordance_dat',  -- 'concordance_dat', 'edrm_xml'
    include_native   BOOLEAN NOT NULL DEFAULT false,
    include_text_files BOOLEAN NOT NULL DEFAULT true,
    -- Bates numbering
    bates_prefix    VARCHAR(20) NOT NULL,  -- e.g., 'ACME'
    bates_start     BIGINT NOT NULL DEFAULT 1,
    bates_end       BIGINT,
    bates_padding   INTEGER NOT NULL DEFAULT 7,  -- number of digits, e.g., ACME0000001
    -- Confidentiality
    confidentiality_stamp VARCHAR(100),  -- 'CONFIDENTIAL', 'ATTORNEYS EYES ONLY', etc.
    -- Delivery
    delivered_to    VARCHAR(500),
    delivered_at    TIMESTAMPTZ,
    delivery_method VARCHAR(100),  -- 'sftp', 'secure_link', 'physical_media'
    -- Timestamps
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_productions_matter ON productions(matter_id);
CREATE INDEX idx_productions_status ON productions(status);

CREATE TABLE production_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    production_id   UUID NOT NULL REFERENCES productions(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    -- Bates stamping
    bates_begin     VARCHAR(50) NOT NULL,
    bates_end       VARCHAR(50) NOT NULL,
    page_count      INTEGER NOT NULL DEFAULT 1,
    -- Image paths (for Opticon OPT generation)
    image_paths     TEXT[],  -- array of relative paths to TIFF/PDF images per page
    native_path     VARCHAR(2000),  -- relative path to native file
    text_path       VARCHAR(2000),  -- relative path to extracted text file
    -- Redaction applied
    has_redactions  BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(production_id, document_id)
);

CREATE INDEX idx_prod_docs_production ON production_documents(production_id);
CREATE INDEX idx_prod_docs_document ON production_documents(document_id);
CREATE INDEX idx_prod_docs_bates ON production_documents(bates_begin);
```

---

## AI / TAR Tables

```sql
-- TAR 2.0 / Continuous Active Learning models
CREATE TABLE tar_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    model_name      VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL DEFAULT 'cal',  -- 'cal' (Continuous Active Learning), 'sal' (Simple Active Learning)
    target_field_id UUID NOT NULL REFERENCES coding_fields(id),  -- the coding field being predicted
    target_value    VARCHAR(500) NOT NULL,  -- the value being predicted (e.g., 'Responsive')
    status          VARCHAR(50) NOT NULL DEFAULT 'training',  -- 'training', 'active', 'paused', 'completed'
    -- Performance metrics
    recall_estimate DECIMAL(5,4),
    precision_estimate DECIMAL(5,4),
    f1_estimate     DECIMAL(5,4),
    richness_estimate DECIMAL(5,4),  -- estimated prevalence of responsive documents
    documents_scored BIGINT DEFAULT 0,
    documents_reviewed BIGINT DEFAULT 0,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tar_models_matter ON tar_models(matter_id);

CREATE TABLE tar_scores (
    document_id     UUID NOT NULL REFERENCES documents(id),
    tar_model_id    UUID NOT NULL REFERENCES tar_models(id) ON DELETE CASCADE,
    score           DECIMAL(5,4) NOT NULL,  -- 0.0000 to 1.0000 relevance score
    rank            BIGINT,
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, tar_model_id)
);

CREATE INDEX idx_tar_scores_model_rank ON tar_scores(tar_model_id, rank);
CREATE INDEX idx_tar_scores_model_score ON tar_scores(tar_model_id, score DESC);
```

---

## Audit Trail Tables

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,  -- 'document.reviewed', 'legal_hold.issued', 'production.exported', etc.
    entity_type     VARCHAR(100) NOT NULL,  -- 'document', 'legal_hold', 'production', etc.
    entity_id       UUID NOT NULL,
    -- Change details
    old_values      JSONB,  -- previous state (for updates)
    new_values      JSONB,  -- new state
    -- Context
    ip_address      INET,
    user_agent      VARCHAR(1000),
    session_id      VARCHAR(255),
    -- Timestamp (append-only, never updated)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partitioned by month for performance on large matter datasets
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_matter ON audit_log(matter_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_action ON audit_log(action, created_at);
```

---

## Early Case Assessment Tables

```sql
CREATE TABLE case_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    -- Data universe statistics
    total_custodians INTEGER,
    total_data_sources INTEGER,
    total_estimated_documents BIGINT,
    total_estimated_size_bytes BIGINT,
    -- Proportionality analysis (FRCP Rule 26)
    estimated_review_hours DECIMAL(10,2),
    estimated_review_cost DECIMAL(12,2),
    proportionality_score DECIMAL(5,4),  -- AI-generated score 0-1
    proportionality_memo TEXT,  -- AI-generated FRCP Rule 26 memo
    -- Status
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    generated_by    UUID NOT NULL REFERENCES users(id),
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_case_assessments_matter ON case_assessments(matter_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 6 | tenants, users, roles, permissions, role_permissions, user_roles |
| Matter Management | 2 | matters, matter_parties |
| Legal Hold | 4 | custodians, custodian_data_sources, legal_holds, legal_hold_notices, legal_hold_custodians |
| Collection | 1 | collections |
| Documents & Processing | 5 | documents, near_duplicate_groups, near_duplicate_members, email_threads, email_thread_members |
| Document Review | 6 | coding_fields, coding_field_choices, tags, document_tags, review_batches, review_batch_documents, coding_decisions |
| Privilege Review | 2 | privilege_designations, privilege_log_entries |
| Production | 2 | productions, production_documents |
| AI / TAR | 2 | tar_models, tar_scores |
| Audit Trail | 1 | audit_log |
| Early Case Assessment | 1 | case_assessments |
| **Total** | **~35** | Core schema; additional tables for DSAR, cross-matter learning, and custodian interviews would add ~10 more |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation without coordination, simplifies multi-region deployment, and prevents enumeration attacks on sequential IDs.

2. **SHA-256 as the primary hash** — aligned with FIPS 180-4 and ISO/IEC 27037; MD5 retained only for backward compatibility with older productions. SHA-256 is the court-accepted standard for forensic integrity.

3. **Document families via parent_id and family_id** — `parent_id` creates a strict hierarchy (email → attachment), while `family_id` groups all documents in the same family for family-level production and review decisions.

4. **Configurable coding fields per matter** — mirrors Relativity's coding layout model. The `coding_fields` / `coding_field_choices` / `coding_decisions` pattern allows each matter to define its own review taxonomy without schema changes.

5. **Privilege review as a separate workflow** — `privilege_designations` is intentionally separate from `coding_decisions` to enforce the FRE 502 requirement that privilege calls receive attorney confirmation. AI pre-flagging populates `ai_flagged` and `ai_confidence`; the `attorney_reviewed` flag must be set by a licensed attorney.

6. **Bates numbering at production time** — documents do not receive Bates numbers until they are added to a production. This allows the same document to appear in multiple productions with different Bates prefixes, matching industry practice.

7. **Audit log as append-only** — the `audit_log` table has no UPDATE or DELETE operations in the application layer. The JSONB `old_values` / `new_values` columns capture state changes for defensibility under FRCP Rules 26 and 37.

8. **Multi-tenant with row-level tenant_id** — every data table includes `tenant_id` for row-level isolation. PostgreSQL Row Level Security (RLS) policies should be applied to enforce tenant boundaries at the database level.

9. **Email metadata fields aligned with RFC 5322** — `email_message_id` and `email_in_reply_to` enable accurate email threading based on the standard Message-ID / In-Reply-To / References headers rather than subject-line matching.

10. **TAR scores as a separate table** — decouples AI scoring from document metadata. Multiple TAR models can score the same document independently, and scores can be recomputed without touching the documents table.
