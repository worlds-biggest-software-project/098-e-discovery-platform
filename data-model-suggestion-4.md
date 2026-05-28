# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: E-Discovery Platform · Created: 2026-05-12

## Philosophy

This model adds a property graph layer on top of a relational foundation, optimised for the relationship-heavy queries that dominate advanced e-discovery analytics: communication network analysis ("who talked to whom, how often, about what?"), custodian relationship mapping, privilege chain analysis ("is there an attorney in this communication chain?"), and conflict-of-interest detection. The operational CRUD (creating matters, ingesting documents, tracking legal holds) uses standard relational tables, but the analytics and AI layers operate on a graph representation.

E-discovery is fundamentally about relationships between people, documents, and communications. A patent litigation matter might involve 50 custodians with thousands of email threads between them — and the key question is often not "what does this document say?" but "who was communicating with whom about this topic during this period?" Graph databases excel at these traversal queries, which are expensive or impossible to express efficiently in SQL alone.

The graph layer can be implemented either as PostgreSQL tables with `graph_nodes` and `graph_edges` (using recursive CTEs for traversal) or as a dedicated graph database (Neo4j, Amazon Neptune) synced from the relational store. This model presents the PostgreSQL-native approach for simplicity, with notes on when a dedicated graph engine becomes necessary.

**Best for:** Organisations where communication pattern analysis, privilege chain detection, custodian relationship mapping, and conflict-of-interest analysis are primary use cases — typically complex multi-party litigation, government investigations, and corporate internal investigations.

**Trade-offs:**
- Pro: Communication network analysis queries are orders of magnitude faster than relational joins
- Pro: Privilege chain detection (traversing communication paths to find attorney involvement) is natural
- Pro: Visual analytics (communication graphs, custodian networks) are natively supported
- Pro: AI/ML feature engineering for NLP models benefits from graph-structured relationship data
- Pro: Conflict-of-interest detection across matters leverages cross-custodian graph queries
- Con: Increased complexity — two data representations (relational + graph) must be kept in sync
- Con: PostgreSQL graph traversal via recursive CTEs has depth and performance limits vs. dedicated graph DBs
- Con: Development team needs graph database skills in addition to SQL skills
- Con: More storage overhead from the graph layer duplicating relationship data
- Con: Graph queries are powerful but harder to optimise and debug than standard SQL

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EDRM Framework | Operational EDRM workflow in relational tables; graph layer models entity relationships across EDRM stages |
| ISO/IEC 27050-3 | ESI lifecycle in relational tables; graph captures entity relationships discovered during processing and review |
| ISO/IEC 27037 | Chain of custody in relational tables; graph models evidence provenance chains |
| FRCP Rules 26/34/37 | Legal hold and production in relational tables; graph enables proportionality analysis via custodian network scope |
| FRE 502 | Privilege chain analysis via graph traversal — "is there an attorney node within N hops of this communication?" |
| W3C PROV-DM | Graph edge types align with W3C provenance relationships (wasGeneratedBy, wasDerivedFrom, wasAttributedTo) |
| RFC 5322 | Email message-id and in-reply-to headers create natural graph edges between email nodes |
| ISO 3166-1/2 | Jurisdiction nodes enable geographic clustering of custodians and legal entities |
| SHA-256 (FIPS 180-4) | Hash integrity on document nodes |

---

## Relational Foundation (Operational CRUD)

### Core Tables

The operational tables are similar to Model 1 (normalised relational) but streamlined — the graph layer handles relationship analytics.

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE matters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_number   VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    matter_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    jurisdiction    VARCHAR(10),
    court_name      VARCHAR(500),
    case_number     VARCHAR(200),
    date_filed      DATE,
    lead_attorney_id UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
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
    is_attorney     BOOLEAN NOT NULL DEFAULT false,  -- critical for privilege graph traversal
    bar_admissions  TEXT[],  -- for attorney custodians
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_custodians_attorney ON custodians(tenant_id, is_attorney) WHERE is_attorney = true;

CREATE TABLE legal_holds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    hold_name       VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    date_issued     TIMESTAMPTZ,
    date_released   TIMESTAMPTZ,
    issued_by       UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE legal_hold_custodians (
    legal_hold_id   UUID NOT NULL REFERENCES legal_holds(id) ON DELETE CASCADE,
    custodian_id    UUID NOT NULL REFERENCES custodians(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    notified_at     TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    PRIMARY KEY (legal_hold_id, custodian_id)
);

CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    custodian_id    UUID NOT NULL REFERENCES custodians(id),
    collection_method VARCHAR(100) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'planned',
    source_hash_sha256 VARCHAR(64),
    collection_hash_sha256 VARCHAR(64),
    total_items     BIGINT,
    total_size_bytes BIGINT,
    collected_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    collection_id   UUID REFERENCES collections(id),
    doc_id          VARCHAR(100) NOT NULL,
    parent_id       UUID REFERENCES documents(id),
    family_id       UUID,
    file_name       VARCHAR(1000),
    file_extension  VARCHAR(50),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(255),
    hash_sha256     VARCHAR(64) NOT NULL,
    page_count      INTEGER,
    extracted_text  TEXT,
    -- Email fields (RFC 5322)
    email_from      VARCHAR(500),
    email_to        TEXT[],
    email_cc        TEXT[],
    email_subject   VARCHAR(2000),
    email_date      TIMESTAMPTZ,
    email_message_id VARCHAR(500),
    email_in_reply_to VARCHAR(500),
    -- Status
    processing_status VARCHAR(50) NOT NULL DEFAULT 'pending',
    custodian_id    UUID REFERENCES custodians(id),
    is_duplicate    BOOLEAN NOT NULL DEFAULT false,
    is_nist         BOOLEAN NOT NULL DEFAULT false,
    -- Review
    coding          JSONB NOT NULL DEFAULT '{}',
    privilege       JSONB,
    tags            TEXT[] DEFAULT '{}',
    tar_scores      JSONB DEFAULT '{}',
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, doc_id)
);

CREATE INDEX idx_docs_matter ON documents(matter_id);
CREATE INDEX idx_docs_hash ON documents(hash_sha256);
CREATE INDEX idx_docs_custodian ON documents(custodian_id);
CREATE INDEX idx_docs_family ON documents(family_id);
CREATE INDEX idx_docs_email_date ON documents(matter_id, email_date) WHERE email_date IS NOT NULL;
CREATE INDEX idx_docs_coding ON documents USING GIN(coding);
CREATE INDEX idx_docs_privilege ON documents USING GIN(privilege);

CREATE TABLE productions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    production_name VARCHAR(500) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    bates_prefix    VARCHAR(20) NOT NULL,
    bates_start     BIGINT NOT NULL DEFAULT 1,
    bates_end       BIGINT,
    format_config   JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE production_documents (
    production_id   UUID NOT NULL REFERENCES productions(id) ON DELETE CASCADE,
    document_id     UUID NOT NULL REFERENCES documents(id),
    bates_begin     VARCHAR(50) NOT NULL,
    bates_end       VARCHAR(50) NOT NULL,
    page_count      INTEGER NOT NULL DEFAULT 1,
    PRIMARY KEY (production_id, document_id)
);
```

---

## Graph Layer (Communication Network and Entity Relationships)

### Graph Node and Edge Tables

```sql
-- Graph nodes represent entities that participate in relationships
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    matter_id       UUID,  -- NULL for cross-matter nodes (e.g., custodians)
    -- Node type and identity
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: 'person', 'organization', 'document', 'email_address',
    --             'matter', 'topic', 'legal_entity', 'department', 'location'
    entity_id       UUID,  -- FK to the relational entity (custodian_id, document_id, etc.)
    -- Display properties
    label           VARCHAR(500) NOT NULL,  -- display name for visualisation
    -- Properties (flexible key-value attributes for the node)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for person node: {
    --   "email": "jsmith@acme.com",
    --   "department": "Engineering",
    --   "title": "VP Engineering",
    --   "is_attorney": false,
    --   "custodian_id": "uuid",
    --   "employee_id": "EMP-4521"
    -- }
    -- Example for email_address node: {
    --   "address": "jsmith@acme.com",
    --   "display_name": "John Smith",
    --   "is_internal": true,
    --   "domain": "acme.com"
    -- }
    -- Example for topic node: {
    --   "topic_name": "Patent Application #12345",
    --   "keywords": ["patent", "claims", "prior art", "filing"],
    --   "first_seen": "2024-01-15",
    --   "document_count": 234
    -- }
    -- Centrality and scoring (computed by graph algorithms)
    degree_centrality    DECIMAL(10,6) DEFAULT 0,
    betweenness_centrality DECIMAL(10,6) DEFAULT 0,
    pagerank            DECIMAL(10,6) DEFAULT 0,
    community_id        INTEGER,  -- community detection cluster assignment
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_graph_nodes_matter ON graph_nodes(matter_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_label ON graph_nodes(tenant_id, label);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN(properties);
CREATE INDEX idx_graph_nodes_community ON graph_nodes(community_id);
CREATE INDEX idx_graph_nodes_pagerank ON graph_nodes(matter_id, node_type, pagerank DESC);

-- Graph edges represent relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    matter_id       UUID,
    -- Edge endpoints
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    -- Edge type and direction
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types: 'sent_email_to', 'cc_on_email', 'bcc_on_email',
    --             'authored', 'received', 'attached_to', 'replies_to',
    --             'reports_to', 'member_of', 'related_to', 'references_topic',
    --             'part_of_thread', 'custodian_of', 'counsel_for',
    --             'privileged_communication_with'
    is_directed     BOOLEAN NOT NULL DEFAULT true,
    -- Edge weight (for frequency/strength of relationship)
    weight          DECIMAL(10,4) NOT NULL DEFAULT 1.0,
    -- Temporal scope (when was this relationship active?)
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    -- Properties (edge-specific attributes)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for sent_email_to edge: {
    --   "email_count": 47,
    --   "first_email_date": "2024-01-15",
    --   "last_email_date": "2024-09-20",
    --   "avg_emails_per_week": 2.3,
    --   "topics": ["patent", "budget", "hiring"],
    --   "sample_subjects": ["Re: Patent filing timeline", "Q3 Budget review"]
    -- }
    -- Example for privileged_communication_with edge: {
    --   "communication_count": 12,
    --   "privilege_type": "attorney_client",
    --   "attorney_node_id": "uuid",
    --   "date_range": {"start": "2024-03-01", "end": "2024-06-15"}
    -- }
    -- Source document reference (which document(s) created this edge)
    source_document_ids UUID[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_matter ON graph_edges(matter_id);
CREATE INDEX idx_graph_edges_weight ON graph_edges(matter_id, edge_type, weight DESC);
CREATE INDEX idx_graph_edges_temporal ON graph_edges(valid_from, valid_to);
CREATE INDEX idx_graph_edges_props ON graph_edges USING GIN(properties);
-- Composite index for bidirectional lookups (undirected queries)
CREATE INDEX idx_graph_edges_both ON graph_edges(source_node_id, target_node_id, edge_type);
CREATE INDEX idx_graph_edges_reverse ON graph_edges(target_node_id, source_node_id, edge_type);
```

### Graph Algorithm Results

```sql
-- Pre-computed graph algorithm outputs (updated periodically)
CREATE TABLE graph_analysis_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    algorithm       VARCHAR(100) NOT NULL,
    -- Algorithms: 'pagerank', 'betweenness_centrality', 'community_detection',
    --             'shortest_path', 'connected_components', 'influence_propagation'
    parameters      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"damping_factor": 0.85, "max_iterations": 100, "edge_types": ["sent_email_to"]}
    status          VARCHAR(50) NOT NULL DEFAULT 'running',
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    results_summary JSONB DEFAULT '{}',
    -- Example: {"communities_found": 7, "modularity": 0.72, "nodes_processed": 1234}
    created_by      UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_graph_analysis_matter ON graph_analysis_runs(matter_id);

-- Community detection results (groups of people who communicate more internally than externally)
CREATE TABLE graph_communities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_run_id UUID NOT NULL REFERENCES graph_analysis_runs(id) ON DELETE CASCADE,
    matter_id       UUID NOT NULL REFERENCES matters(id),
    community_label VARCHAR(255),
    member_count    INTEGER NOT NULL,
    -- Descriptive properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "dominant_department": "Engineering",
    --   "key_topics": ["patent", "product development"],
    --   "internal_edge_density": 0.45,
    --   "hub_nodes": ["uuid-1", "uuid-2"],
    --   "date_range_active": {"start": "2024-01", "end": "2024-09"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Graph Query Examples

### Find all people within 2 hops of a specific custodian (communication network)

```sql
-- Recursive CTE for graph traversal in PostgreSQL
WITH RECURSIVE communication_network AS (
    -- Base case: the starting custodian's node
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.properties->>'email' AS email,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = 'target-custodian-uuid'
      AND gn.node_type = 'person'
      AND gn.matter_id = 'target-matter-uuid'

    UNION ALL

    -- Recursive case: traverse email edges
    SELECT
        gn2.id AS node_id,
        gn2.label,
        gn2.properties->>'email' AS email,
        cn.depth + 1 AS depth,
        cn.path || gn2.id AS path
    FROM communication_network cn
    JOIN graph_edges ge ON (ge.source_node_id = cn.node_id OR ge.target_node_id = cn.node_id)
    JOIN graph_nodes gn2 ON (
        CASE WHEN ge.source_node_id = cn.node_id THEN ge.target_node_id ELSE ge.source_node_id END = gn2.id
    )
    WHERE cn.depth < 2
      AND gn2.node_type = 'person'
      AND gn2.id != ALL(cn.path)  -- prevent cycles
      AND ge.edge_type IN ('sent_email_to', 'cc_on_email')
      AND ge.matter_id = 'target-matter-uuid'
)
SELECT DISTINCT node_id, label, email, depth
FROM communication_network
ORDER BY depth, label;
```

### Privilege chain detection: find if any attorney is within the communication path

```sql
-- Detect potential privilege by finding attorney nodes connected to a document's sender/recipients
WITH document_participants AS (
    -- Get all person nodes involved in a specific document
    SELECT DISTINCT gn.id AS person_node_id, gn.label, gn.properties->>'is_attorney' AS is_attorney
    FROM documents d
    JOIN graph_edges ge ON d.id = ANY(ge.source_document_ids)
    JOIN graph_nodes gn ON (gn.id = ge.source_node_id OR gn.id = ge.target_node_id)
    WHERE d.id = 'target-document-uuid'
      AND gn.node_type = 'person'
),
attorney_connections AS (
    -- Check if any participant is an attorney OR is connected to an attorney within 1 hop
    SELECT
        dp.person_node_id,
        dp.label AS participant,
        dp.is_attorney AS participant_is_attorney,
        atty.id AS connected_attorney_id,
        atty.label AS connected_attorney_name,
        ge.edge_type AS connection_type
    FROM document_participants dp
    LEFT JOIN graph_edges ge ON (ge.source_node_id = dp.person_node_id OR ge.target_node_id = dp.person_node_id)
        AND ge.edge_type IN ('sent_email_to', 'cc_on_email', 'counsel_for')
    LEFT JOIN graph_nodes atty ON (
        CASE WHEN ge.source_node_id = dp.person_node_id THEN ge.target_node_id ELSE ge.source_node_id END = atty.id
    )
        AND atty.node_type = 'person'
        AND (atty.properties->>'is_attorney')::boolean = true
)
SELECT *
FROM attorney_connections
WHERE participant_is_attorney = 'true' OR connected_attorney_id IS NOT NULL;
```

### Top communicators by PageRank within a matter

```sql
SELECT
    gn.label AS person_name,
    gn.properties->>'email' AS email,
    gn.properties->>'department' AS department,
    gn.pagerank,
    gn.degree_centrality,
    gn.betweenness_centrality,
    gn.community_id
FROM graph_nodes gn
WHERE gn.matter_id = 'target-matter-uuid'
  AND gn.node_type = 'person'
ORDER BY gn.pagerank DESC
LIMIT 20;
```

### Communication volume between two custodians over time

```sql
SELECT
    date_trunc('week', ge.valid_from) AS week,
    ge.edge_type,
    SUM(ge.weight) AS email_count,
    array_agg(DISTINCT ge.properties->>'topics') AS topics
FROM graph_edges ge
WHERE ge.source_node_id = 'person-node-uuid-1'
  AND ge.target_node_id = 'person-node-uuid-2'
  AND ge.edge_type IN ('sent_email_to', 'cc_on_email')
  AND ge.matter_id = 'target-matter-uuid'
GROUP BY date_trunc('week', ge.valid_from), ge.edge_type
ORDER BY week;
```

### Cross-matter conflict detection (shared custodians across matters)

```sql
-- Find people who appear in multiple matters as custodians
SELECT
    gn.label AS person_name,
    gn.properties->>'email' AS email,
    array_agg(DISTINCT gn.matter_id) AS matter_ids,
    COUNT(DISTINCT gn.matter_id) AS matter_count
FROM graph_nodes gn
WHERE gn.tenant_id = 'target-tenant-uuid'
  AND gn.node_type = 'person'
  AND gn.matter_id IS NOT NULL
GROUP BY gn.label, gn.properties->>'email'
HAVING COUNT(DISTINCT gn.matter_id) > 1
ORDER BY matter_count DESC;
```

### Shortest path between two people (for relationship analysis)

```sql
WITH RECURSIVE shortest_path AS (
    SELECT
        ge.target_node_id AS current_node,
        ARRAY[ge.source_node_id, ge.target_node_id] AS path,
        1 AS hops
    FROM graph_edges ge
    WHERE ge.source_node_id = 'person-node-uuid-start'
      AND ge.edge_type IN ('sent_email_to', 'cc_on_email', 'reports_to')
      AND ge.matter_id = 'target-matter-uuid'

    UNION ALL

    SELECT
        ge.target_node_id AS current_node,
        sp.path || ge.target_node_id AS path,
        sp.hops + 1 AS hops
    FROM shortest_path sp
    JOIN graph_edges ge ON ge.source_node_id = sp.current_node
    WHERE ge.target_node_id != ALL(sp.path)  -- no cycles
      AND sp.hops < 5  -- max depth
      AND ge.edge_type IN ('sent_email_to', 'cc_on_email', 'reports_to')
      AND ge.matter_id = 'target-matter-uuid'
)
SELECT path, hops
FROM shortest_path
WHERE current_node = 'person-node-uuid-end'
ORDER BY hops
LIMIT 1;
```

---

## Graph Construction Pipeline

The graph is built as documents are ingested and processed:

```sql
-- Pseudocode for graph construction during email ingestion:

-- 1. When an email is ingested, create/update person nodes for all participants
-- INSERT INTO graph_nodes (tenant_id, matter_id, node_type, entity_id, label, properties)
-- VALUES (tenant, matter, 'person', custodian_id, 'John Smith', '{"email": "jsmith@acme.com", ...}')
-- ON CONFLICT (tenant_id, entity_id, matter_id) DO UPDATE SET properties = ...

-- 2. Create edges for email relationships
-- For each recipient in email_to: INSERT INTO graph_edges (source=sender, target=recipient, type='sent_email_to', weight=1)
-- For each recipient in email_cc: INSERT INTO graph_edges (source=sender, target=recipient, type='cc_on_email', weight=0.5)
-- If email has in_reply_to: INSERT INTO graph_edges (source=this_doc, target=parent_doc, type='replies_to')

-- 3. Aggregate edges: periodically merge individual email edges into weighted summary edges
-- UPDATE graph_edges SET weight = (SELECT COUNT(*) FROM documents WHERE ...) WHERE edge_type = 'sent_email_to' AND ...

-- 4. Run graph algorithms periodically (PageRank, community detection, betweenness centrality)
-- These update the centrality columns on graph_nodes and the community assignments
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
    changes         JSONB,
    context         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_matter ON audit_log(matter_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## When to Graduate to a Dedicated Graph Database

The PostgreSQL graph-in-tables approach works well up to:
- ~1M nodes and ~10M edges per matter
- Traversal depth of 3-5 hops
- Queries that don't require complex graph algorithms (Louvain community detection, etc.)

Consider migrating to Neo4j or Amazon Neptune when:
- Matters exceed 10M edges (complex multi-year investigations)
- Real-time graph algorithm execution is needed (not batch)
- Visual graph exploration with 100K+ visible nodes
- Cypher or Gremlin query expressiveness is needed for complex pattern matching

The sync pattern: relational tables remain the system of record for operational CRUD; the graph database is a read-replica synced via CDC (Change Data Capture) or event streaming.

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 2 | tenants, users |
| Matter Management | 1 | matters |
| Custodians & Legal Hold | 3 | custodians, legal_holds, legal_hold_custodians |
| Collection | 1 | collections |
| Documents | 1 | documents (with JSONB coding/privilege) |
| Production | 2 | productions, production_documents |
| Graph Layer | 4 | graph_nodes, graph_edges, graph_analysis_runs, graph_communities |
| Audit Trail | 1 | audit_log |
| **Total** | **~15** | Plus the relational tables from the graph layer; dedicated graph DB adds no SQL tables |

---

## Key Design Decisions

1. **Graph-in-PostgreSQL for initial deployment** — using `graph_nodes` and `graph_edges` tables with recursive CTEs avoids adding a separate database technology to the deployment. This keeps infrastructure simple for early adopters while enabling powerful communication analysis. The schema is designed for a clean migration path to Neo4j/Neptune when scale demands it.

2. **Separate nodes for persons vs. email addresses** — a person can have multiple email addresses (work, personal, alias). The graph models `person` nodes connected to `email_address` nodes via `owns_address` edges. This correctly handles the common e-discovery scenario where custodians use multiple email accounts.

3. **Edge weight for communication frequency** — `graph_edges.weight` stores the aggregated communication volume between two nodes. This enables "find the top 10 communication pairs" queries without counting individual emails, and feeds into PageRank and centrality calculations.

4. **Temporal edges with valid_from/valid_to** — communication relationships have a time dimension. Edges capture when the relationship was active, enabling queries like "who was John communicating with in Q1 2024?" This is essential for early case assessment where the relevant time period is defined early in litigation.

5. **is_attorney flag on custodian nodes** — the critical privilege chain detection use case requires knowing which nodes in the communication graph represent attorneys. This single boolean enables the "is there an attorney within N hops?" traversal that powers AI-assisted privilege pre-flagging.

6. **Pre-computed centrality scores on nodes** — PageRank, betweenness centrality, and degree centrality are computed by batch graph algorithm runs and stored directly on `graph_nodes`. This avoids recomputing expensive algorithms for every visualization or analytics query.

7. **Community detection for custodian grouping** — the Louvain community detection algorithm identifies natural communication clusters (e.g., "the patent team," "the executive group," "the external counsel team"). These communities inform batch assignment during review (assign all documents from a community to the same reviewer for context).

8. **source_document_ids on edges** — every graph edge tracks which document(s) created it. This provides traceability: clicking an edge in the visualization shows the actual emails or documents that establish the relationship.

9. **Cross-matter nodes via NULL matter_id** — person nodes with `matter_id = NULL` represent the global identity of a custodian across all matters. Matter-specific nodes link to these global nodes, enabling cross-matter conflict detection and review learning without leaking matter-specific data.

10. **Graph layer does not replace relational document review** — the graph is an analytics and AI layer, not the operational layer. Document coding, privilege calls, batch assignments, and production workflows all operate on the relational tables. The graph provides insights (who are the key communicators? where are the privilege chains?) that inform human reviewer decisions.
