# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: E-Discovery Platform · Created: 2026-05-12

## Philosophy

This model combines the structural guarantees of relational tables for core entities with JSONB columns for variable, jurisdiction-specific, and matter-specific data. The core EDRM workflow (matters, custodians, legal holds, documents, productions) uses typed relational columns with foreign keys. But the areas where e-discovery data varies most — document metadata fields, coding layouts, custodian data source configurations, jurisdiction-specific hold requirements, and production format specifications — use JSONB columns that can accommodate any structure without DDL changes.

This design reflects the reality that e-discovery matters differ dramatically in their metadata requirements. A patent litigation matter needs different coding fields from a FOIA request. A UK matter subject to the Data Protection Act 2018 has different hold requirements from a US matter under FRCP. A production to the SEC has different format specifications than a production to opposing counsel. Rather than creating dozens of nullable columns or complex EAV (Entity-Attribute-Value) patterns, JSONB absorbs this variation naturally.

The approach is well-suited for rapid MVP development. The relational core provides integrity and query performance for the most common operations, while JSONB fields allow the platform to accommodate new matter types, jurisdictions, and workflows without database migrations. As usage patterns stabilise, frequently-queried JSONB fields can be promoted to typed columns.

**Best for:** Teams building a multi-jurisdiction, multi-matter-type platform that needs to ship an MVP quickly while retaining the ability to accommodate diverse client requirements without schema migrations.

**Trade-offs:**
- Pro: Fast to iterate — new fields for new matter types or jurisdictions require no DDL changes
- Pro: Core entities have relational integrity; JSONB handles the long tail of variation
- Pro: JSONB GIN indexes provide fast containment queries for flexible metadata
- Pro: Fewer tables than fully normalised model (~25 vs ~35+)
- Pro: Natural fit for PostgreSQL, which has mature JSONB support with indexing
- Con: JSONB fields lack referential integrity — the database cannot enforce that a JSONB reference is valid
- Con: Schema documentation must cover JSONB structures separately; less self-documenting than pure relational
- Con: Complex JSONB queries can be slower than equivalent relational joins
- Con: Risk of "JSONB creep" — over time, too much logic moves into untyped JSONB, eroding data quality
- Con: Reporting and BI tools may struggle with JSONB fields compared to flat relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EDRM Framework | Core EDRM stages are relational tables; stage-specific configuration is JSONB |
| EDRM XML v1.1 | Document metadata JSONB structure mirrors EDRM XML element attributes for native import/export |
| ISO/IEC 27050-3 | ESI lifecycle tracked in relational columns; jurisdiction-specific variations in JSONB |
| ISO/IEC 27037 | Chain-of-custody fields are relational (hash, timestamp, collector); forensic tool metadata in JSONB |
| FRCP Rules 26/34/37 | Legal hold and production workflows are relational; court-specific requirements in JSONB |
| FRE 502 | Privilege workflow is relational with attorney confirmation; privilege log format templates in JSONB |
| ISO 3166-1/2 | Jurisdiction codes as typed columns; jurisdiction-specific rules and requirements in JSONB |
| SHA-256 (FIPS 180-4) | Hash columns are typed VARCHAR(64); additional hash algorithms stored in metadata JSONB |
| RFC 5322 / MIME | Common email fields are typed columns; extended email headers in JSONB |
| Concordance DAT / Opticon OPT | Production format settings in JSONB; Bates numbering in typed columns |

---

## Core Infrastructure Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',  -- ISO 3166-1 alpha-2
    -- JSONB for tenant-specific settings that vary by deployment
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings: {
    --   "default_production_format": "concordance_dat",
    --   "default_image_format": "tiff",
    --   "max_matter_size_gb": 500,
    --   "retention_policy_days": 2190,
    --   "sso_config": {"provider": "okta", "issuer_url": "https://..."},
    --   "gdpr_enabled": true,
    --   "data_residency_rules": {"eu_data": "eu-west-1", "us_data": "us-east-1"}
    -- }
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
    -- JSONB for user preferences and profile data
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "timezone": "America/New_York",
    --   "bar_admissions": ["NY", "CA"],
    --   "notification_preferences": {"email": true, "in_app": true},
    --   "default_coding_layout": "uuid-..."
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

-- Simplified RBAC: roles and permissions in a single join table
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    -- JSONB permissions instead of a separate permissions table
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example: ["matter.create", "matter.view", "document.review", "production.export"]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    matter_id       UUID,  -- NULL = global; non-NULL = matter-scoped
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, COALESCE(matter_id, '00000000-0000-0000-0000-000000000000'))
);
```

---

## Matter Management

```sql
CREATE TABLE matters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_number   VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    matter_type     VARCHAR(50) NOT NULL,  -- 'litigation', 'investigation', 'regulatory', 'foia', 'arbitration'
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    jurisdiction    VARCHAR(10),  -- ISO 3166-1/2
    -- Core court/case info as typed columns (queried frequently)
    court_name      VARCHAR(500),
    case_number     VARCHAR(200),
    date_filed      DATE,
    date_closed     DATE,
    lead_attorney_id UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    -- JSONB for matter-type-specific and jurisdiction-specific fields
    matter_details  JSONB NOT NULL DEFAULT '{}',
    -- Example for litigation: {
    --   "judge_name": "Hon. Jane Smith",
    --   "magistrate_name": "Hon. John Doe",
    --   "parties": [
    --     {"name": "Smith Corp", "type": "plaintiff", "counsel": "Baker & McKenzie"},
    --     {"name": "Acme Inc", "type": "defendant", "counsel": "Skadden"}
    --   ],
    --   "discovery_cutoff_date": "2026-09-15",
    --   "esi_protocol": {"agreed_format": "tiff_with_ocr", "search_term_protocol": "cooperative"},
    --   "proportionality_limit_gb": 50
    -- }
    -- Example for FOIA: {
    --   "foia_request_number": "2026-F-00123",
    --   "requesting_party": "Jane Reporter, NY Times",
    --   "exemptions_claimed": ["b5", "b6"],
    --   "response_deadline": "2026-06-15",
    --   "agency_component": "Office of General Counsel"
    -- }
    -- Example for UK matter: {
    --   "tribunal": "Employment Tribunal",
    --   "claimant": "Smith",
    --   "respondent": "Acme UK Ltd",
    --   "data_protection_basis": "legitimate_interest",
    --   "ico_notification_required": true
    -- }
    -- Coding layout configuration (which fields apply to this matter)
    coding_layout   JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"name": "Responsiveness", "type": "single_choice", "choices": ["Responsive", "Not Responsive", "Needs Further Review"], "required": true},
    --   {"name": "Issues", "type": "multi_choice", "choices": ["Patent Infringement", "Trade Secret", "Breach of Contract"]},
    --   {"name": "Confidentiality", "type": "single_choice", "choices": ["Confidential", "Highly Confidential - AEO", "Not Confidential"]},
    --   {"name": "Key Document", "type": "boolean"},
    --   {"name": "Notes", "type": "text"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
);

CREATE INDEX idx_matters_tenant ON matters(tenant_id);
CREATE INDEX idx_matters_status ON matters(tenant_id, status);
CREATE INDEX idx_matters_type ON matters(tenant_id, matter_type);
CREATE INDEX idx_matters_details ON matters USING GIN(matter_details);
```

---

## Custodian and Legal Hold Tables

```sql
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
    -- JSONB for variable custodian data (data sources, IT systems, interview results)
    data_sources    JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"type": "email_mailbox", "name": "jsmith@acme.com", "location": "Exchange Online", "estimated_size_gb": 12.5},
    --   {"type": "onedrive", "name": "OneDrive - J Smith", "location": "sharepoint.com/personal/jsmith", "estimated_size_gb": 45.0},
    --   {"type": "slack", "name": "Slack - jsmith", "channels": ["#legal", "#executive", "#general"]},
    --   {"type": "local_drive", "name": "Laptop C: drive", "device_id": "ACME-L-4521", "estimated_size_gb": 200}
    -- ]
    interview_data  JSONB,
    -- Example: {
    --   "interview_date": "2026-03-15",
    --   "interviewer": "Legal Hold Admin",
    --   "relevant_systems": ["email", "onedrive", "slack"],
    --   "date_range_relevant": {"start": "2023-01-01", "end": "2026-03-15"},
    --   "key_contacts_mentioned": ["Jane Doe", "Bob Manager"],
    --   "notes": "Custodian confirmed using personal phone for some business communications"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_custodians_tenant ON custodians(tenant_id);
CREATE INDEX idx_custodians_name ON custodians(tenant_id, last_name, first_name);
CREATE INDEX idx_custodians_data_sources ON custodians USING GIN(data_sources);

CREATE TABLE legal_holds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    hold_name       VARCHAR(500) NOT NULL,
    hold_type       VARCHAR(50) NOT NULL DEFAULT 'litigation',
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    description     TEXT,
    preservation_scope TEXT,
    date_issued     TIMESTAMPTZ,
    date_released   TIMESTAMPTZ,
    issued_by       UUID NOT NULL REFERENCES users(id),
    reminder_interval_days INTEGER DEFAULT 90,
    -- JSONB for hold notice templates and jurisdiction-specific requirements
    hold_config     JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "notice_template": {
    --     "subject": "Litigation Hold Notice - Smith v. Acme",
    --     "body_html": "<h1>Preservation Notice</h1>...",
    --     "reminder_subject": "Reminder: Litigation Hold - Smith v. Acme"
    --   },
    --   "jurisdiction_rules": {
    --     "require_written_acknowledgment": true,
    --     "max_escalation_before_counsel_notification": 3,
    --     "gdpr_data_minimization_applied": false
    --   },
    --   "escalation_chain": [
    --     {"after_days": 3, "action": "reminder"},
    --     {"after_days": 7, "action": "escalate_to_manager"},
    --     {"after_days": 14, "action": "escalate_to_counsel"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_legal_holds_matter ON legal_holds(matter_id);
CREATE INDEX idx_legal_holds_status ON legal_holds(status);

CREATE TABLE legal_hold_custodians (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_hold_id   UUID NOT NULL REFERENCES legal_holds(id) ON DELETE CASCADE,
    custodian_id    UUID NOT NULL REFERENCES custodians(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    escalated_at    TIMESTAMPTZ,
    escalation_count INTEGER NOT NULL DEFAULT 0,
    -- JSONB for hold-specific custodian data
    hold_details    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "specific_preservation_instructions": "Preserve all Slack DMs with Bob Manager",
    --   "acknowledgment_ip": "192.168.1.100",
    --   "acknowledgment_user_agent": "Mozilla/5.0...",
    --   "escalation_history": [
    --     {"date": "2026-03-18", "type": "reminder", "sent_to": "jsmith@acme.com"},
    --     {"date": "2026-03-25", "type": "escalate_to_manager", "sent_to": "manager@acme.com"}
    --   ]
    -- }
    UNIQUE(legal_hold_id, custodian_id)
);

CREATE INDEX idx_lh_custodians_hold ON legal_hold_custodians(legal_hold_id);
CREATE INDEX idx_lh_custodians_custodian ON legal_hold_custodians(custodian_id);
CREATE INDEX idx_lh_custodians_status ON legal_hold_custodians(status);
```

---

## Collection and Document Tables

```sql
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    custodian_id    UUID NOT NULL REFERENCES custodians(id),
    collection_method VARCHAR(100) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'planned',
    date_range_start DATE,
    date_range_end  DATE,
    collected_by    UUID REFERENCES users(id),
    collected_at    TIMESTAMPTZ,
    total_items     BIGINT,
    total_size_bytes BIGINT,
    -- Chain of custody (ISO/IEC 27037)
    source_hash_sha256 VARCHAR(64),
    collection_hash_sha256 VARCHAR(64),
    -- JSONB for collection-method-specific configuration and results
    collection_config JSONB NOT NULL DEFAULT '{}',
    -- Example for cloud collection: {
    --   "source": "microsoft_365",
    --   "search_terms": ["patent", "infringement", "claim"],
    --   "date_filter": {"start": "2023-01-01", "end": "2026-03-15"},
    --   "content_types": ["email", "onedrive", "teams_chat"],
    --   "oauth_scope": "Mail.ReadBasic",
    --   "collection_tool": "custom_graph_api_collector_v2",
    --   "forensic_notes": "Collected via Microsoft Graph API; full mailbox export"
    -- }
    -- Example for forensic image: {
    --   "imaging_tool": "FTK Imager 4.7",
    --   "image_format": "E01",
    --   "device_serial": "WDC-WD1003FZEX-00K3CA0",
    --   "write_blocker": "Tableau T35689",
    --   "examiner": "John Forensic, ACE certified",
    --   "verification_hash_match": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collections_matter ON collections(matter_id);
CREATE INDEX idx_collections_custodian ON collections(custodian_id);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    collection_id   UUID REFERENCES collections(id),
    doc_id          VARCHAR(100) NOT NULL,
    parent_id       UUID REFERENCES documents(id),
    family_id       UUID,
    -- Core file metadata (typed columns for frequent queries)
    file_name       VARCHAR(1000),
    file_extension  VARCHAR(50),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(255),
    hash_sha256     VARCHAR(64) NOT NULL,
    page_count      INTEGER,
    -- Text content
    extracted_text  TEXT,
    text_language   VARCHAR(10),
    -- Processing
    processing_status VARCHAR(50) NOT NULL DEFAULT 'pending',
    processed_at    TIMESTAMPTZ,
    custodian_id    UUID REFERENCES custodians(id),
    -- Deduplication
    is_duplicate    BOOLEAN NOT NULL DEFAULT false,
    is_nist         BOOLEAN NOT NULL DEFAULT false,
    -- Email core fields (RFC 5322 — typed for fast email-specific queries)
    email_from      VARCHAR(500),
    email_to        TEXT[],
    email_cc        TEXT[],
    email_subject   VARCHAR(2000),
    email_date      TIMESTAMPTZ,
    email_message_id VARCHAR(500),
    email_in_reply_to VARCHAR(500),
    -- JSONB for all other metadata (extensible without DDL changes)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example for a Word document: {
    --   "author": "Jane Smith",
    --   "last_modified_by": "Bob Jones",
    --   "company": "Acme Corp",
    --   "date_created": "2024-06-15T10:30:00Z",
    --   "date_modified": "2024-09-20T14:22:00Z",
    --   "revision_count": 12,
    --   "word_count": 4521,
    --   "template": "Legal Brief.dotx",
    --   "custom_properties": {"Matter Number": "2024-LIT-001", "Confidential": "Yes"}
    -- }
    -- Example for an email: {
    --   "email_bcc": ["secret@acme.com"],
    --   "email_importance": "high",
    --   "email_has_attachments": true,
    --   "email_attachment_count": 3,
    --   "email_categories": ["Legal", "Urgent"],
    --   "email_conversation_index": "01D8A2B3...",
    --   "email_transport_headers": "X-Mailer: Outlook 16.0..."
    -- }
    -- Coding decisions stored as JSONB (avoids separate coding_decisions table)
    coding          JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "Responsiveness": {"value": "Responsive", "reviewer_id": "uuid", "decided_at": "2026-04-01T10:00:00Z", "time_spent_ms": 23000, "ai_suggested": true, "ai_confidence": 0.92},
    --   "Issues": {"values": ["Patent", "Trade Secret"], "reviewer_id": "uuid", "decided_at": "2026-04-01T10:00:00Z"},
    --   "Confidentiality": {"value": "Confidential", "reviewer_id": "uuid", "decided_at": "2026-04-01T10:01:00Z"}
    -- }
    -- Privilege designation stored as JSONB
    privilege       JSONB,
    -- Example: {
    --   "is_privileged": true,
    --   "privilege_type": "attorney_client",
    --   "privilege_basis": "Communication between in-house counsel and VP Operations",
    --   "ai_flagged": true,
    --   "ai_confidence": 0.87,
    --   "attorney_confirmed": true,
    --   "attorney_id": "uuid",
    --   "attorney_confirmed_at": "2026-04-02T14:00:00Z",
    --   "requires_redaction": false
    -- }
    -- Tags as a simple array
    tags            TEXT[] DEFAULT '{}',
    -- TAR scores as JSONB
    tar_scores      JSONB DEFAULT '{}',
    -- Example: {"model-uuid-1": 0.9234, "model-uuid-2": 0.1456}
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, doc_id)
);

CREATE INDEX idx_docs_matter ON documents(matter_id);
CREATE INDEX idx_docs_collection ON documents(collection_id);
CREATE INDEX idx_docs_custodian ON documents(custodian_id);
CREATE INDEX idx_docs_parent ON documents(parent_id);
CREATE INDEX idx_docs_family ON documents(family_id);
CREATE INDEX idx_docs_hash ON documents(hash_sha256);
CREATE INDEX idx_docs_processing ON documents(matter_id, processing_status);
CREATE INDEX idx_docs_email_date ON documents(matter_id, email_date) WHERE email_date IS NOT NULL;
CREATE INDEX idx_docs_mime ON documents(matter_id, mime_type);
CREATE INDEX idx_docs_tags ON documents USING GIN(tags);
CREATE INDEX idx_docs_metadata ON documents USING GIN(metadata);
CREATE INDEX idx_docs_coding ON documents USING GIN(coding);
CREATE INDEX idx_docs_privilege ON documents USING GIN(privilege);
CREATE INDEX idx_docs_tar ON documents USING GIN(tar_scores);
```

---

## Review and Batch Management

```sql
CREATE TABLE review_batches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    batch_name      VARCHAR(255) NOT NULL,
    assigned_to     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'assigned',
    document_count  INTEGER NOT NULL DEFAULT 0,
    reviewed_count  INTEGER NOT NULL DEFAULT 0,
    assigned_by     UUID NOT NULL REFERENCES users(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    due_date        TIMESTAMPTZ,
    -- JSONB for batch-specific instructions and review guidance
    batch_config    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "review_instructions": "Focus on communications between J. Smith and external counsel. Flag any references to pending patent application.",
    --   "coding_fields_to_complete": ["Responsiveness", "Issues", "Confidentiality"],
    --   "priority": "high",
    --   "quality_check_rate": 0.10
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_batches_matter ON review_batches(matter_id);
CREATE INDEX idx_review_batches_assignee ON review_batches(assigned_to, status);

CREATE TABLE review_batch_documents (
    batch_id        UUID NOT NULL REFERENCES review_batches(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    position        INTEGER NOT NULL,
    PRIMARY KEY (batch_id, document_id)
);
```

---

## Email Threading and Near-Duplicate Clustering

```sql
CREATE TABLE email_threads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    thread_root_message_id VARCHAR(500),
    subject_normalized VARCHAR(2000),
    document_count  INTEGER NOT NULL DEFAULT 1,
    -- JSONB for thread analysis results
    thread_analysis JSONB DEFAULT '{}',
    -- Example: {
    --   "participants": ["jsmith@acme.com", "jdoe@lawfirm.com", "bob@acme.com"],
    --   "date_range": {"start": "2024-03-01", "end": "2024-03-15"},
    --   "inclusive_document_ids": ["uuid-1", "uuid-5"],
    --   "total_unique_content_pages": 23
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE email_thread_members (
    thread_id       UUID NOT NULL REFERENCES email_threads(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    position_in_thread INTEGER NOT NULL,
    is_inclusive     BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (thread_id, document_id)
);

CREATE TABLE near_duplicate_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    pivot_document_id UUID NOT NULL REFERENCES documents(id),
    similarity_threshold DECIMAL(5,4) NOT NULL DEFAULT 0.9000,
    member_document_ids UUID[] NOT NULL DEFAULT '{}',  -- array instead of junction table
    document_count  INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_near_dup_matter ON near_duplicate_groups(matter_id);
CREATE INDEX idx_near_dup_members ON near_duplicate_groups USING GIN(member_document_ids);
```

---

## Production Tables

```sql
CREATE TABLE productions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    production_name VARCHAR(500) NOT NULL,
    production_number VARCHAR(100),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- Bates numbering (typed — always needed)
    bates_prefix    VARCHAR(20) NOT NULL,
    bates_start     BIGINT NOT NULL DEFAULT 1,
    bates_end       BIGINT,
    bates_padding   INTEGER NOT NULL DEFAULT 7,
    -- JSONB for format configuration (varies by production)
    format_config   JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "image_format": "tiff",
    --   "dpi": 300,
    --   "color_mode": "black_and_white",
    --   "load_file_format": "concordance_dat",
    --   "include_native": true,
    --   "include_text_files": true,
    --   "include_ocr": true,
    --   "confidentiality_stamp": "CONFIDENTIAL - ATTORNEYS EYES ONLY",
    --   "stamp_position": "bottom_center",
    --   "redaction_color": "#000000",
    --   "slip_sheet_text": "Document withheld on grounds of privilege",
    --   "encoding": "UTF-8",
    --   "date_format": "MM/DD/YYYY",
    --   "volume_max_size_gb": 4.7,
    --   "sec_specific": {
    --     "production_format": "native_with_load_file",
    --     "include_parent_child_relationships": true
    --   }
    -- }
    -- Delivery information
    delivery_config JSONB DEFAULT '{}',
    -- Example: {
    --   "method": "sftp",
    --   "destination": "sftp://productions.lawfirm.com/acme-v-smith/",
    --   "delivered_at": "2026-04-15T09:00:00Z",
    --   "delivered_by": "uuid",
    --   "delivery_confirmation_hash": "sha256-of-entire-production"
    -- }
    document_count  INTEGER DEFAULT 0,
    total_pages     INTEGER DEFAULT 0,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_productions_matter ON productions(matter_id);

CREATE TABLE production_documents (
    production_id   UUID NOT NULL REFERENCES productions(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    bates_begin     VARCHAR(50) NOT NULL,
    bates_end       VARCHAR(50) NOT NULL,
    page_count      INTEGER NOT NULL DEFAULT 1,
    image_paths     TEXT[],
    native_path     VARCHAR(2000),
    text_path       VARCHAR(2000),
    has_redactions  BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (production_id, document_id)
);

CREATE INDEX idx_prod_docs_bates ON production_documents(bates_begin);
```

---

## Privilege Log Table

```sql
CREATE TABLE privilege_log_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    document_id     UUID NOT NULL REFERENCES documents(id),
    -- Standard privilege log fields (typed for export to Concordance DAT)
    bates_begin     VARCHAR(50) NOT NULL,
    bates_end       VARCHAR(50) NOT NULL,
    doc_date        DATE,
    doc_type        VARCHAR(100),
    author          VARCHAR(500),
    recipients      TEXT,
    subject         VARCHAR(2000),
    privilege_claimed VARCHAR(200) NOT NULL,
    privilege_description TEXT NOT NULL,
    -- JSONB for court-specific or jurisdiction-specific fields
    extended_fields JSONB DEFAULT '{}',
    -- Example for SEC production: {
    --   "redacted_portions": "Paragraphs 3-5",
    --   "in_camera_review_offered": true,
    --   "privilege_holder": "Acme Corp General Counsel"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_priv_log_matter ON privilege_log_entries(matter_id);
CREATE INDEX idx_priv_log_bates ON privilege_log_entries(bates_begin);
```

---

## TAR / AI Models

```sql
CREATE TABLE tar_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    model_name      VARCHAR(255) NOT NULL,
    target_field_name VARCHAR(255) NOT NULL,  -- the coding field name from the matter's coding_layout
    target_value    VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'training',
    -- JSONB for model performance and configuration
    model_config    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "model_type": "cal",
    --   "algorithm": "logistic_regression",
    --   "feature_set": "tfidf_50000",
    --   "recall_estimate": 0.94,
    --   "precision_estimate": 0.87,
    --   "f1_estimate": 0.90,
    --   "richness_estimate": 0.12,
    --   "documents_scored": 150000,
    --   "documents_reviewed": 5000,
    --   "training_rounds": 15,
    --   "last_trained_at": "2026-04-10T14:00:00Z"
    -- }
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tar_models_matter ON tar_models(matter_id);

-- TAR scores stored directly on the document via tar_scores JSONB column
-- No separate tar_scores table needed in this model
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    matter_id       UUID,
    user_id         UUID,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,  -- {"field": {"old": "value1", "new": "value2"}, ...}
    context         JSONB DEFAULT '{}',  -- ip_address, user_agent, session_id, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_matter ON audit_log(matter_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_log(action, created_at);
```

---

## JSONB Query Examples

### Find all documents where AI flagged privilege with confidence > 0.8

```sql
SELECT id, doc_id, file_name, privilege->>'privilege_type' AS priv_type,
       (privilege->>'ai_confidence')::decimal AS ai_conf
FROM documents
WHERE matter_id = 'target-matter-uuid'
  AND privilege->>'ai_flagged' = 'true'
  AND (privilege->>'ai_confidence')::decimal > 0.8
  AND (privilege->>'attorney_confirmed')::text IS DISTINCT FROM 'true'
ORDER BY (privilege->>'ai_confidence')::decimal DESC;
```

### Find all documents coded as Responsive but not yet privilege-reviewed

```sql
SELECT id, doc_id, file_name, coding->'Responsiveness'->>'value' AS responsiveness
FROM documents
WHERE matter_id = 'target-matter-uuid'
  AND coding->'Responsiveness'->>'value' = 'Responsive'
  AND privilege IS NULL
ORDER BY ingested_at;
```

### Find all custodians with Slack data sources

```sql
SELECT id, first_name, last_name, email,
       jsonb_array_elements(data_sources) AS slack_source
FROM custodians
WHERE tenant_id = 'target-tenant-uuid'
  AND data_sources @> '[{"type": "slack"}]';
```

### Get matter-type-specific fields for FOIA matters

```sql
SELECT matter_number, name,
       matter_details->>'foia_request_number' AS foia_number,
       matter_details->>'requesting_party' AS requester,
       matter_details->>'response_deadline' AS deadline
FROM matters
WHERE tenant_id = 'target-tenant-uuid'
  AND matter_type = 'foia'
  AND (matter_details->>'response_deadline')::date < now() + interval '30 days';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 4 | tenants, users, roles, user_roles |
| Matter Management | 1 | matters (parties in JSONB) |
| Custodians & Legal Hold | 3 | custodians, legal_holds, legal_hold_custodians |
| Collection | 1 | collections |
| Documents | 1 | documents (coding, privilege, tags, TAR scores all in JSONB) |
| Email & Near-Duplicate | 3 | email_threads, email_thread_members, near_duplicate_groups |
| Review | 2 | review_batches, review_batch_documents |
| Production | 2 | productions, production_documents |
| Privilege Log | 1 | privilege_log_entries |
| AI / TAR | 1 | tar_models (scores on document JSONB) |
| Audit Trail | 1 | audit_log |
| **Total** | **~20** | Significantly fewer than normalised model due to JSONB absorption |

---

## Key Design Decisions

1. **Coding decisions embedded in document JSONB** — instead of a separate `coding_decisions` table with rows per document per field per reviewer, coding decisions are stored directly in the `documents.coding` JSONB column. This eliminates the most expensive join in the review workflow (documents + coding_decisions) at the cost of losing relational constraints on coding field validity.

2. **Privilege designation embedded in document JSONB** — the `documents.privilege` JSONB column captures the full privilege workflow (AI flag, attorney confirmation, privilege type/basis) on the document itself. The separate `privilege_log_entries` table exists only for generating court-submission-formatted privilege logs.

3. **Matter parties in JSONB instead of a junction table** — `matter_details.parties` stores party information as a JSONB array. For most matters this is 2-10 parties, making a separate table unnecessary overhead. If complex party queries become important, the field can be promoted to a relational table.

4. **Custodian data sources in JSONB** — the variety of data source types (email, OneDrive, Slack, local drives, mobile devices, Teams, Zoom) makes a relational schema with typed columns for each impractical. JSONB handles the long tail naturally.

5. **Coding layout defined per matter as JSONB** — `matters.coding_layout` defines the coding fields, types, and choices for each matter as a JSON array. This replaces the `coding_fields` and `coding_field_choices` tables from the normalised model, enabling each matter to have a unique coding taxonomy without DDL changes.

6. **Production format configuration in JSONB** — production format requirements vary dramatically between recipients (SEC vs. opposing counsel vs. court). The `format_config` JSONB captures all format options (image format, DPI, load file type, stamps, volume sizing) without requiring columns for every possible option.

7. **Near-duplicate member IDs as UUID array** — instead of a junction table, `near_duplicate_groups.member_document_ids` stores member document IDs as a UUID array. This simplifies queries ("which group is this document in?") at the cost of not having foreign key enforcement on the array elements.

8. **GIN indexes on all major JSONB columns** — PostgreSQL GIN indexes support the `@>` (contains) operator on JSONB, enabling fast queries like "find all documents where coding contains Responsiveness = Responsive". These indexes are essential for production-grade query performance.

9. **TAR scores on document rather than separate table** — `documents.tar_scores` stores model scores as a JSONB map (`{model_id: score}`). This avoids a high-cardinality join table (every document x every model) while enabling fast score-based filtering via JSONB indexing.

10. **Typed columns for high-frequency query fields; JSONB for everything else** — email fields (from, to, subject, date, message_id), hash values, processing status, and custody are typed columns because they appear in virtually every query. Extended metadata, coding, privilege, and configuration data are JSONB because they vary by matter type, jurisdiction, and document type.
