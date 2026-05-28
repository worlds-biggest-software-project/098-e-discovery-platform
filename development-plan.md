# E-Discovery Platform — Development Plan

> Project: E-Discovery Platform (Candidate #98)
> Created: 2026-05-25

---

## Technology Decisions

### Decision 1: Data Model — Hybrid Relational + JSONB (Model 3) with Event-Sourced Audit Trail (from Model 2)

**Rationale:** The hybrid relational + JSONB model (data-model-suggestion-3) provides the fastest path to MVP by keeping the table count low (~20 tables) while accommodating the extreme variability in e-discovery metadata — different matter types (litigation vs. FOIA vs. investigation), jurisdictions (FRCP vs. GDPR vs. UK DPA), document types, and coding layouts — without schema migrations. Core entities (matters, custodians, documents, legal holds, productions) get typed relational columns for high-frequency queries; everything else goes into JSONB.

However, Model 2's event-sourced audit trail is architecturally critical for defensibility. FRCP Rule 37 sanctions require provable, tamper-evident records of every action. We adopt Model 2's append-only `events` table with SHA-256 hash chaining as the audit trail, running alongside the Model 3 relational schema. This is not full CQRS — the relational tables are the primary read/write store — but every state change also emits an immutable event for audit purposes.

Model 4's graph layer is deferred to Phase 8 (communication analytics). We design the relational schema to enable graph construction later without rework.

**Rejected alternatives:**
- Model 1 (full normalization): ~35+ tables, heavy DDL migration burden, too rigid for rapid MVP iteration across varied matter types
- Model 2 (full event sourcing): Implementation complexity of CQRS projection management is disproportionate for early development; adopted only the audit event stream
- Model 4 (graph-relational): Graph layer is valuable but premature before core EDRM workflow is proven; deferred to Phase 8

### Decision 2: Primary Database — PostgreSQL 16+

**Rationale:** Mature JSONB support with GIN indexing, Row Level Security (RLS) for multi-tenant isolation, partitioning for audit log tables, full-text search capabilities, and the largest talent pool. All four data model suggestions target PostgreSQL. The graph layer (Phase 8) can begin with recursive CTEs in PostgreSQL and graduate to Neo4j/Neptune when scale demands it.

### Decision 3: Search Engine — Elasticsearch 8.x (OpenSearch compatible)

**Rationale:** Full-text search with boolean, proximity, and metadata filtering is table-stakes for e-discovery review. Elasticsearch integrates natively with Apache Tika for document extraction, supports the query complexity lawyers expect (proximity operators, fuzzy matching, field-scoped searches), and scales horizontally for large matters (100M+ documents). OpenSearch compatibility ensures no vendor lock-in.

### Decision 4: Document Processing — Apache Tika 2.x

**Rationale:** Apache License 2.0 (permissive, patent grant included). Supports 1,000+ file formats including all critical e-discovery types: PST/OST/EML/MSG email, Office formats (DOCX/XLSX/PPTX), PDF, images (with OCR via Tesseract integration). Used internally by multiple commercial e-discovery platforms. Active Apache Foundation governance.

### Decision 5: Backend Framework — Node.js (TypeScript) with NestJS

**Rationale:** TypeScript provides type safety for the complex domain model. NestJS offers modular architecture with dependency injection, native OpenAPI/Swagger generation (OAS 3.1 compliance), built-in support for event emitters (for the audit event stream), and a large ecosystem. REST API with OpenAPI spec is a competitive differentiator (research shows Logikcull and Casepoint have no public APIs).

### Decision 6: Frontend — React 19 with TypeScript

**Rationale:** Document review workspaces require sophisticated UX: split-screen views, configurable coding panels, drag-and-drop batch management, real-time collaboration indicators. React's component model and ecosystem (TanStack Table for document grids, React Flow for email thread visualisation, D3.js for analytics dashboards) best serve these requirements.

### Decision 7: Object Storage — S3-compatible (AWS S3 / MinIO for self-hosted)

**Rationale:** Native files, TIFF images, extracted text, and production outputs require durable object storage. S3-compatible APIs support both cloud (AWS) and self-hosted (MinIO) deployments, addressing the data sovereignty gap that cloud-only incumbents (Everlaw, DISCO) cannot serve.

### Decision 8: Authentication — OAuth 2.0 + OpenID Connect

**Rationale:** Enterprise e-discovery deployments universally require SSO integration with Okta, Azure AD, or PingIdentity. OIDC is the documented standard across Relativity, Everlaw, and Nuix APIs. Local username/password as fallback for self-hosted deployments.

### Decision 9: AI/ML Framework — Python microservices (FastAPI) for TAR and privilege models

**Rationale:** The CAL algorithm (Cormack and Grossman, 2014) and privilege classification models require scikit-learn, PyTorch, and NLP libraries that are Python-native. FastAPI microservices called from the NestJS backend via internal REST API keep the ML stack independent and deployable separately.

### Decision 10: Licensing — AGPL-3.0

**Rationale:** The research identifies the OSS gap as the primary market opportunity. AGPL ensures the platform remains open-source even when hosted as SaaS, while permitting commercial dual-licensing. Apache Tika (Apache 2.0) is AGPL-compatible.

---

## Project Structure

```
e-discovery-platform/
├── apps/
│   ├── api/                      # NestJS backend (REST API)
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/         # OAuth 2.0 / OIDC / local auth
│   │   │   │   ├── tenants/      # Multi-tenant management
│   │   │   │   ├── matters/      # Matter lifecycle
│   │   │   │   ├── custodians/   # Custodian management
│   │   │   │   ├── legal-holds/  # Legal hold workflow
│   │   │   │   ├── collections/  # Data collection tracking
│   │   │   │   ├── documents/    # Document CRUD and metadata
│   │   │   │   ├── processing/   # Ingestion pipeline orchestration
│   │   │   │   ├── search/       # Elasticsearch query interface
│   │   │   │   ├── review/       # Review batches and coding
│   │   │   │   ├── privilege/    # Privilege workflow (FRE 502)
│   │   │   │   ├── productions/  # Production generation
│   │   │   │   ├── audit/        # Event-sourced audit trail
│   │   │   │   ├── tar/          # TAR model management
│   │   │   │   └── analytics/    # Dashboards and reporting
│   │   │   ├── common/           # Guards, interceptors, filters
│   │   │   ├── database/         # TypeORM/Drizzle entities, migrations
│   │   │   └── config/           # Environment configuration
│   │   └── test/
│   ├── web/                      # React frontend
│   │   ├── src/
│   │   │   ├── features/
│   │   │   │   ├── auth/
│   │   │   │   ├── matters/
│   │   │   │   ├── legal-holds/
│   │   │   │   ├── documents/
│   │   │   │   ├── review/       # Document review workspace
│   │   │   │   ├── privilege/
│   │   │   │   ├── productions/
│   │   │   │   ├── search/
│   │   │   │   └── analytics/
│   │   │   ├── components/       # Shared UI components
│   │   │   └── lib/              # API client, hooks, utilities
│   │   └── test/
│   └── ml/                       # Python AI/ML services
│       ├── tar/                  # TAR 2.0 / CAL implementation
│       ├── privilege/            # Privilege classification
│       ├── eca/                  # Early case assessment
│       └── common/               # Shared ML utilities
├── packages/
│   ├── shared-types/             # TypeScript types shared between api and web
│   ├── edrm-xml/                 # EDRM XML import/export library
│   ├── load-files/               # Concordance DAT / Opticon OPT generator
│   └── tika-client/              # Apache Tika REST client
├── infrastructure/
│   ├── docker/                   # Docker Compose for local dev
│   ├── k8s/                      # Kubernetes manifests
│   └── terraform/                # Cloud infrastructure
├── db/
│   ├── migrations/               # SQL migration files
│   ├── seeds/                    # Test data seeds
│   └── schemas/                  # JSON Schema definitions for JSONB columns
└── docs/
    ├── api/                      # OpenAPI spec
    ├── architecture/             # ADRs and architecture diagrams
    └── legal/                    # Compliance documentation
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Infrastructure
    │
    ├──► Phase 2: Document Processing Pipeline
    │       │
    │       ├──► Phase 3: Search & Document Review
    │       │       │
    │       │       ├──► Phase 5: TAR 2.0 (AI-Assisted Review)
    │       │       │       │
    │       │       │       └──► Phase 8: Communication Analytics (Graph)
    │       │       │
    │       │       ├──► Phase 6: Privilege Review & Logging
    │       │       │
    │       │       └──► Phase 7: Production Generation
    │       │
    │       └──► Phase 4: Email Threading & Near-Duplicate Detection
    │               │
    │               └──► Phase 3 (enhances review efficiency)
    │
    ├──► Phase 9: Legal Hold & Custodian Management
    │       │
    │       └──► Phase 10: Early Case Assessment
    │
    └──► Phase 11: DSAR Integration & GDPR Compliance
            │
            └──► Phase 12: Cross-Matter Learning & Advanced AI

Legend:
  ──► = "depends on" / "requires completion of"
```

**Critical path:** Phase 1 → Phase 2 → Phase 3 → Phase 7 (minimum viable production capability)

**Parallel tracks after Phase 1:**
- Track A: Document pipeline (Phases 2→3→5→6→7→8)
- Track B: Legal hold (Phase 9→10)
- Track C: Compliance (Phase 11→12)

---

## Phase 1: Foundation & Infrastructure

**Goal:** Establish the project skeleton, database schema, authentication, multi-tenancy, and CI/CD pipeline. No e-discovery domain logic yet — this phase builds the scaffolding everything else depends on.

**Duration estimate:** 4-5 weeks

### Task 1.1: Monorepo Initialisation and Tooling

**What:** Set up the monorepo with Turborepo (or Nx), configure TypeScript, ESLint, Prettier, and shared configurations across `apps/api`, `apps/web`, and `packages/*`. Add Docker Compose for local development (PostgreSQL 16, Elasticsearch 8, MinIO, Redis).

**Design:**

```typescript
// turbo.json
{
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "dev": { "cache": false, "persistent": true }
  }
}
```

```yaml
# infrastructure/docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ediscovery
      POSTGRES_USER: ediscovery
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: ["5432:5432"]
    volumes: ["pg_data:/var/lib/postgresql/data"]

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports: ["9200:9200"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  tika:
    image: apache/tika:2.9.2
    ports: ["9998:9998"]
```

**Testing:**
- `docker compose up` starts all services without errors
- `turbo build` compiles all packages and apps
- `turbo lint` passes with zero warnings
- `turbo test` runs empty test suites successfully
- Each service health check passes (pg_isready, ES cluster health, MinIO healthcheck)

### Task 1.2: Database Schema — Core Infrastructure Tables

**What:** Create PostgreSQL migration files for tenants, users, roles, permissions, and user_roles tables. Implement Row Level Security (RLS) policies for tenant isolation. Add seed data for system roles (admin, case_manager, supervising_attorney, reviewer).

**Design:**

```sql
-- db/migrations/001_core_infrastructure.sql

-- Tenants
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

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    oidc_subject    VARCHAR(255),
    oidc_issuer     VARCHAR(512),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- RLS policies
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_users ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Roles with JSONB permissions
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    matter_id       UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id,
                 COALESCE(matter_id, '00000000-0000-0000-0000-000000000000'))
);
```

```typescript
// apps/api/src/database/entities/tenant.entity.ts
@Entity('tenants')
export class Tenant {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 255 })
  name: string;

  @Column({ length: 100, unique: true })
  slug: string;

  @Column({ type: 'jsonb', default: {} })
  settings: Record<string, unknown>;

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date;
}
```

**Testing:**
- Migration runs forward and backward (rollback) without errors
- RLS policy blocks cross-tenant data access: insert user into tenant A, set `app.current_tenant_id` to tenant B, verify SELECT returns zero rows
- Seed data creates exactly 4 system roles with correct permissions arrays
- UNIQUE constraints reject duplicate tenant slugs and duplicate user emails within the same tenant
- Foreign key constraint rejects user creation with non-existent tenant_id

### Task 1.3: Audit Event Store

**What:** Implement the append-only events table from Model 2 for defensibility. Create the event emitter service that writes immutable, hash-chained events for every state change. This runs alongside the relational tables — not as a replacement.

**Design:**

```sql
-- db/migrations/002_audit_event_store.sql
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(100) NOT NULL,
    stream_id       UUID NOT NULL,
    event_type      VARCHAR(200) NOT NULL,
    event_version   INTEGER NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',
    occurred_at     TIMESTAMPTZ NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    event_hash      VARCHAR(64),
    previous_hash   VARCHAR(64),
    UNIQUE(stream_type, stream_id, event_version)
);

CREATE INDEX idx_events_stream ON events(stream_type, stream_id, event_version);
CREATE INDEX idx_events_tenant ON events(tenant_id, recorded_at);
CREATE INDEX idx_events_type ON events(event_type, recorded_at);
```

```typescript
// apps/api/src/modules/audit/audit-event.service.ts
import { createHash } from 'crypto';

@Injectable()
export class AuditEventService {
  async emit(params: {
    streamType: string;
    streamId: string;
    eventType: string;
    payload: Record<string, unknown>;
    tenantId: string;
    actorId: string;
    actorType: 'user' | 'system' | 'ai_agent';
  }): Promise<void> {
    const previousEvent = await this.getLatestEvent(
      params.streamType, params.streamId
    );
    const eventVersion = (previousEvent?.eventVersion ?? 0) + 1;
    const previousHash = previousEvent?.eventHash ?? null;

    const payloadStr = JSON.stringify(params.payload);
    const eventHash = createHash('sha256')
      .update(`${previousHash ?? ''}:${payloadStr}`)
      .digest('hex');

    await this.eventRepository.insert({
      streamType: params.streamType,
      streamId: params.streamId,
      eventType: params.eventType,
      eventVersion,
      payload: params.payload,
      tenantId: params.tenantId,
      actorId: params.actorId,
      actorType: params.actorType,
      occurredAt: new Date(),
      eventHash,
      previousHash,
    });
  }
}
```

**Testing:**
- Events table rejects UPDATE and DELETE via application-level guards (test that no repository method exposes update/delete)
- Hash chain verification: insert 5 events into the same stream, verify each event's `previous_hash` matches the prior event's `event_hash`
- Tamper detection: manually alter a middle event's payload in a test, run verification query, confirm broken chain is detected
- Concurrent event emission to the same stream: verify UNIQUE constraint on (stream_type, stream_id, event_version) prevents duplicate versions
- Bi-temporal query: insert event with `occurred_at` = 2026-03-15, verify it can be retrieved by both `occurred_at` and `recorded_at` ranges

### Task 1.4: Authentication — Local + OIDC

**What:** Implement local username/password authentication (bcrypt hashing) with JWT access/refresh tokens. Add OpenID Connect integration via Passport.js strategies for enterprise SSO (Okta, Azure AD). Implement RBAC middleware that checks user permissions against the roles JSONB.

**Design:**

```typescript
// apps/api/src/modules/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  @Post('login')
  async login(@Body() dto: LoginDto): Promise<TokenResponse> { ... }

  @Post('refresh')
  async refresh(@Body() dto: RefreshTokenDto): Promise<TokenResponse> { ... }

  @Get('oidc/callback')
  async oidcCallback(@Query() params: OidcCallbackDto): Promise<TokenResponse> { ... }
}

// apps/api/src/common/guards/permissions.guard.ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.get<string[]>(
      'permissions', context.getHandler()
    );
    const user = context.switchToHttp().getRequest().user;
    return requiredPermissions.every(p =>
      user.permissions.includes(p)
    );
  }
}
```

**Testing:**
- Login with valid credentials returns JWT with correct claims (tenant_id, user_id, permissions array)
- Login with invalid password returns 401
- Expired JWT returns 401; valid refresh token issues new access token
- RBAC guard: user with `document.review` permission can access review endpoint; user without permission gets 403
- Matter-scoped roles: user with reviewer role on matter A cannot access matter B documents
- OIDC flow: mock Okta callback returns valid user and creates/links account

### Task 1.5: REST API Skeleton and OpenAPI Spec

**What:** Create the NestJS module structure for all domain modules (empty controllers with route definitions, DTOs with validation decorators, Swagger decorators). Generate OpenAPI 3.1 spec. Set up API versioning (v1 prefix).

**Design:**

```typescript
// apps/api/src/modules/matters/matters.controller.ts
@ApiTags('Matters')
@Controller('v1/matters')
@UseGuards(AuthGuard, PermissionsGuard)
export class MattersController {
  @Get()
  @Permissions('matter.view')
  @ApiOperation({ summary: 'List matters for the current tenant' })
  @ApiResponse({ status: 200, type: [MatterResponseDto] })
  async list(@Query() query: ListMattersQueryDto): Promise<PaginatedResponse<MatterResponseDto>> { ... }

  @Post()
  @Permissions('matter.create')
  @ApiOperation({ summary: 'Create a new matter' })
  async create(@Body() dto: CreateMatterDto): Promise<MatterResponseDto> { ... }

  @Get(':id')
  @Permissions('matter.view')
  async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<MatterResponseDto> { ... }
}
```

**Testing:**
- `/api/docs` renders Swagger UI with all endpoints documented
- OpenAPI spec validates against OAS 3.1 schema (use `swagger-cli validate`)
- All DTO validation decorators reject invalid input (empty matter name, invalid UUID, non-ISO date)
- API versioning: `/v1/matters` returns 200; `/v2/matters` returns 404
- Pagination: `?page=1&limit=20` returns correct `total`, `page`, `limit`, `totalPages` metadata

### Task 1.6: CI/CD Pipeline

**What:** GitHub Actions workflows for: lint, test, build on every PR; Docker image builds on main; database migration check (ensure migrations are reversible).

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: ediscovery_test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx turbo lint
      - run: npx turbo test -- --coverage
      - run: npx turbo build
```

**Testing:**
- CI pipeline passes on a clean clone
- Deliberately introduce a lint error; verify CI fails
- Deliberately introduce a failing test; verify CI fails
- Docker image builds and starts successfully
- Migration up/down cycle completes without errors in CI

### Definition of Done — Phase 1
- [ ] Monorepo builds, lints, and tests cleanly
- [ ] Docker Compose starts PostgreSQL 16, Elasticsearch 8, MinIO, Redis, Tika
- [ ] Database migrations create all core tables with RLS policies
- [ ] Audit event store writes hash-chained events and detects tampering
- [ ] Local auth (login, JWT, refresh) and OIDC stub work end-to-end
- [ ] RBAC permissions guard enforces role-based access
- [ ] OpenAPI 3.1 spec generated and validates
- [ ] CI/CD pipeline runs lint, test, build on every PR
- [ ] All tests pass with >80% coverage on auth and audit modules

---

## Phase 2: Document Processing Pipeline

**Goal:** Build the document ingestion pipeline — from file upload through Tika extraction, metadata capture, deduplication, OCR, and Elasticsearch indexing. This is the EDRM "Processing" stage.

**Duration estimate:** 5-6 weeks

### Task 2.1: File Upload and Storage Service

**What:** Implement chunked file upload endpoint (multipart/form-data) that stores original files in S3-compatible object storage (MinIO). Support single files, ZIP archives, PST files, and load file imports (Concordance DAT with native files). Track upload jobs with status.

**Design:**

```typescript
// apps/api/src/modules/processing/upload.service.ts
@Injectable()
export class UploadService {
  async uploadFile(params: {
    matterId: string;
    collectionId: string;
    file: Express.Multer.File;
    custodianId?: string;
  }): Promise<UploadResult> {
    const hash = createHash('sha256')
      .update(file.buffer)
      .digest('hex');

    const s3Key = `${params.matterId}/native/${hash}/${file.originalname}`;
    await this.s3.putObject({
      Bucket: this.configService.get('S3_BUCKET'),
      Key: s3Key,
      Body: file.buffer,
      ContentType: file.mimetype,
    });

    return {
      s3Key,
      hashSha256: hash,
      fileName: file.originalname,
      fileSize: file.size,
      mimeType: file.mimetype,
    };
  }
}
```

**Testing:**
- Upload a 100MB file via chunked upload; verify it appears in MinIO with correct hash
- Upload a ZIP archive; verify it is stored as-is (extraction happens in Task 2.2)
- Upload a duplicate file (same SHA-256 hash); verify dedup flag is set without re-uploading content
- Reject upload exceeding tenant's max file size setting
- Verify audit event `FileUploaded` is emitted with file hash, size, and uploader

### Task 2.2: Apache Tika Integration and Text Extraction

**What:** Create a Tika client library (`packages/tika-client`) that calls Tika's REST API for text extraction, metadata extraction, and language detection. Process uploaded files through Tika and store results. Handle 1,000+ file format types including PDF, Office, email (EML/MSG), images.

**Design:**

```typescript
// packages/tika-client/src/tika-client.ts
export class TikaClient {
  constructor(private readonly baseUrl: string) {}

  async extractText(fileBuffer: Buffer, mimeType: string): Promise<string> {
    const response = await fetch(`${this.baseUrl}/tika`, {
      method: 'PUT',
      headers: {
        'Content-Type': mimeType,
        'Accept': 'text/plain',
      },
      body: fileBuffer,
    });
    return response.text();
  }

  async extractMetadata(fileBuffer: Buffer, mimeType: string): Promise<Record<string, string>> {
    const response = await fetch(`${this.baseUrl}/meta`, {
      method: 'PUT',
      headers: {
        'Content-Type': mimeType,
        'Accept': 'application/json',
      },
      body: fileBuffer,
    });
    return response.json();
  }

  async detectLanguage(text: string): Promise<string> {
    const response = await fetch(`${this.baseUrl}/language/string`, {
      method: 'PUT',
      headers: { 'Content-Type': 'text/plain' },
      body: text,
    });
    return response.text();  // ISO 639-1 code
  }
}
```

**Testing:**
- Extract text from PDF, DOCX, XLSX, PPTX, EML, MSG files; verify non-empty text output
- Extract metadata from DOCX; verify author, creation date, modification date are captured
- Detect language for English, Spanish, and German text samples
- Handle corrupt file gracefully: Tika returns error, processing status set to 'error' with message
- Process PST file: verify it is exploded into individual EML messages, each processed separately

### Task 2.3: Document Ingestion Pipeline (Queue-Based)

**What:** Implement a Bull/BullMQ job queue for asynchronous document processing. Each uploaded file goes through: (1) Tika extraction, (2) metadata mapping, (3) SHA-256 hash computation, (4) exact deduplication check, (5) OCR for images/scanned PDFs, (6) database record creation, (7) Elasticsearch indexing. Track pipeline status per document.

**Design:**

```typescript
// apps/api/src/modules/processing/ingestion.processor.ts
@Processor('document-ingestion')
export class IngestionProcessor {
  @Process()
  async processDocument(job: Job<IngestionJobData>): Promise<void> {
    const { matterId, s3Key, fileName, mimeType, hashSha256, custodianId, collectionId } = job.data;

    // 1. Download from S3
    const fileBuffer = await this.s3Service.getObject(s3Key);

    // 2. Extract text via Tika
    const extractedText = await this.tikaClient.extractText(fileBuffer, mimeType);

    // 3. Extract metadata via Tika
    const metadata = await this.tikaClient.extractMetadata(fileBuffer, mimeType);

    // 4. Check for exact duplicate within the matter
    const duplicate = await this.documentRepo.findOne({
      where: { matterId, hashSha256 }
    });

    // 5. OCR if scanned document (no extracted text but has pages)
    let ocrText: string | null = null;
    if (!extractedText && this.isImageOrScannedPdf(mimeType, metadata)) {
      ocrText = await this.ocrService.performOcr(fileBuffer, mimeType);
    }

    // 6. Map email-specific metadata (RFC 5322)
    const emailFields = this.mapEmailMetadata(metadata, mimeType);

    // 7. Create document record
    const document = await this.documentRepo.save({
      matterId,
      collectionId,
      docId: this.generateDocId(matterId),
      fileName,
      fileExtension: path.extname(fileName),
      fileSizeBytes: fileBuffer.length,
      mimeType,
      hashSha256,
      extractedText: extractedText || ocrText,
      textLanguage: await this.tikaClient.detectLanguage(extractedText || ocrText || ''),
      pageCount: parseInt(metadata['xmpTPg:NPages'] || '0') || null,
      metadata: metadata,
      processingStatus: 'completed',
      processedAt: new Date(),
      custodianId,
      isDuplicate: !!duplicate,
      ...emailFields,
    });

    // 8. Index in Elasticsearch
    await this.searchService.indexDocument(document);

    // 9. Emit audit event
    await this.auditService.emit({
      streamType: 'Document',
      streamId: document.id,
      eventType: 'DocumentIngested',
      payload: { fileName, hashSha256, mimeType, isDuplicate: !!duplicate },
      ...this.contextFromJob(job),
    });
  }
}
```

**Testing:**
- Upload 10 mixed-format files (PDF, DOCX, EML, JPG, XLSX); verify all 10 reach `completed` status
- Upload an image-only PDF; verify OCR produces searchable text
- Upload a duplicate file (same SHA-256); verify `is_duplicate` flag is set and original doc is referenced
- Process a file that Tika cannot handle; verify status is `error` with descriptive error message
- Verify email metadata (from, to, cc, subject, date, message_id) is correctly mapped for EML files
- Pipeline handles 100 concurrent jobs without deadlocks or data corruption
- Each processed document has an audit event `DocumentIngested` with correct payload

### Task 2.4: Elasticsearch Indexing

**What:** Configure Elasticsearch index mappings for documents. Index extracted text, metadata, email fields, and coding values. Support full-text search with boolean operators, proximity, date ranges, and metadata filtering.

**Design:**

```typescript
// apps/api/src/modules/search/elasticsearch.service.ts
const DOCUMENT_INDEX_MAPPING = {
  mappings: {
    properties: {
      matter_id: { type: 'keyword' },
      doc_id: { type: 'keyword' },
      extracted_text: {
        type: 'text',
        analyzer: 'standard',
        search_analyzer: 'standard',
      },
      file_name: { type: 'text', fields: { keyword: { type: 'keyword' } } },
      mime_type: { type: 'keyword' },
      custodian_id: { type: 'keyword' },
      email_from: { type: 'text', fields: { keyword: { type: 'keyword' } } },
      email_to: { type: 'text', fields: { keyword: { type: 'keyword' } } },
      email_subject: { type: 'text' },
      email_date: { type: 'date' },
      date_created: { type: 'date' },
      tags: { type: 'keyword' },
      coding: { type: 'object', dynamic: true },
      hash_sha256: { type: 'keyword' },
      is_duplicate: { type: 'boolean' },
      is_nist: { type: 'boolean' },
      ingested_at: { type: 'date' },
    },
  },
  settings: {
    number_of_shards: 2,
    number_of_replicas: 1,
    analysis: {
      analyzer: {
        ediscovery_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'stop', 'snowball'],
        },
      },
    },
  },
};
```

**Testing:**
- Index 1,000 sample documents; verify all are searchable
- Boolean search: `"patent" AND "infringement"` returns only documents containing both terms
- Proximity search: `"patent infringement"~5` returns documents with the terms within 5 words
- Date range filter: `email_date:[2024-01-01 TO 2024-06-30]` returns correct subset
- Metadata filter: `mime_type:application/pdf AND custodian_id:<uuid>` returns correct results
- Search with highlighting: returned hits include highlighted snippets showing search term context
- De-duplicated documents are excluded from search results by default (configurable)

### Task 2.5: de-NIST File Filtering

**What:** Implement NIST NSRL (National Software Reference Library) hash lookup to identify and flag known system files (OS files, application files) that are never relevant to e-discovery. Flag but do not delete.

**Design:**

```typescript
// apps/api/src/modules/processing/nist.service.ts
@Injectable()
export class NistService {
  private nistHashes: Set<string>;

  async loadNistDatabase(filePath: string): Promise<void> {
    // Load NSRL hash set (SHA-256 hashes of known software files)
    // Available from https://www.nist.gov/itl/ssd/software-quality-group/national-software-reference-library-nsrl
    this.nistHashes = new Set<string>();
    // Stream-parse the NSRL RDS file
  }

  isNistFile(sha256Hash: string): boolean {
    return this.nistHashes.has(sha256Hash.toLowerCase());
  }
}
```

**Testing:**
- Load a subset of the NSRL database (1,000 hashes)
- Check a known Windows system DLL hash; verify it returns `true`
- Check a document hash; verify it returns `false`
- Verify de-NISTed documents have `is_nist = true` in the database
- Verify de-NISTed documents are excluded from review counts but not deleted

### Definition of Done — Phase 2
- [ ] File upload supports single files, ZIP archives, and PST files up to 10GB
- [ ] Apache Tika extracts text and metadata from PDF, Office, email, and image formats
- [ ] OCR produces searchable text from scanned documents and images
- [ ] SHA-256 deduplication correctly identifies and flags duplicate files
- [ ] Email metadata (RFC 5322 fields) correctly parsed and stored
- [ ] Elasticsearch indexes all documents with full-text, boolean, proximity, and date range search
- [ ] de-NIST filtering correctly identifies known system files
- [ ] Document processing pipeline handles 100+ concurrent jobs
- [ ] All processing actions emit audit events with hash chain integrity
- [ ] Processing error rate <1% on standard file format test corpus

---

## Phase 3: Search & Document Review Workspace

**Goal:** Build the document review UI — the core workspace where reviewers search, view, code, tag, and annotate documents. This is the EDRM "Review" stage and the most user-facing component.

**Duration estimate:** 6-7 weeks

### Task 3.1: Search Interface

**What:** Build the search page with a query builder supporting boolean operators (AND, OR, NOT), proximity operators, field-scoped searches (from:, to:, subject:, custodian:), date range pickers, file type filters, and saved searches. Display results in a paginated grid with sort options.

**Design:**

```typescript
// apps/web/src/features/search/SearchQueryBuilder.tsx
interface SearchQuery {
  textQuery: string;                    // free-text with boolean/proximity
  filters: {
    dateRange?: { start: Date; end: Date };
    custodianIds?: string[];
    mimeTypes?: string[];
    tags?: string[];
    codingValues?: Record<string, string[]>;
    isDuplicate?: boolean;
    isNist?: boolean;
  };
  sort: { field: string; direction: 'asc' | 'desc' };
  page: number;
  limit: number;
}

// apps/api/src/modules/search/search.service.ts
async search(matterId: string, query: SearchQuery): Promise<SearchResult> {
  const esQuery = {
    bool: {
      must: [
        { term: { matter_id: matterId } },
        query.textQuery ? {
          query_string: {
            query: query.textQuery,
            default_field: 'extracted_text',
            default_operator: 'AND',
          }
        } : { match_all: {} },
      ],
      filter: [
        query.filters.dateRange ? {
          range: { email_date: {
            gte: query.filters.dateRange.start,
            lte: query.filters.dateRange.end,
          }}
        } : null,
        query.filters.custodianIds?.length ? {
          terms: { custodian_id: query.filters.custodianIds }
        } : null,
        // ... additional filters
      ].filter(Boolean),
    },
  };
  return this.esClient.search({ index: `docs-${matterId}`, body: { query: esQuery } });
}
```

**Testing:**
- Boolean query `contract AND (breach OR violation)` returns correct result set
- Proximity query `"trade secret"~3` finds documents with terms within 3 words
- Field-scoped query `from:jsmith@acme.com` returns only emails from that sender
- Date range filter returns exactly the documents within the specified range
- Saved search persists and returns consistent results when re-executed
- Search result pagination returns correct total count, page numbers, and navigation
- Search with zero results displays appropriate empty state

### Task 3.2: Document Viewer

**What:** Build the document viewer panel that displays document content (extracted text, native file preview via iframe/PDF.js, image viewer), metadata panel, email header display, and family navigation (parent email → attachments). Support keyboard navigation between documents in a result set.

**Design:**

```typescript
// apps/web/src/features/review/DocumentViewer.tsx
interface DocumentViewerProps {
  documentId: string;
  viewMode: 'text' | 'native' | 'image';
  onNavigate: (direction: 'prev' | 'next') => void;
}

const DocumentViewer: React.FC<DocumentViewerProps> = ({ documentId, viewMode }) => {
  const { data: document } = useQuery(['document', documentId], () =>
    api.documents.get(documentId)
  );

  return (
    <div className="document-viewer">
      <DocumentToolbar document={document} viewMode={viewMode} />
      <div className="viewer-content">
        {viewMode === 'text' && <TextViewer text={document.extractedText} highlights={searchHighlights} />}
        {viewMode === 'native' && <NativeViewer s3Url={document.nativeFileUrl} mimeType={document.mimeType} />}
        {viewMode === 'image' && <ImageViewer pages={document.imagePages} />}
      </div>
      <MetadataPanel metadata={document.metadata} emailHeaders={document.emailFields} />
      {document.parentId && <FamilyNavigator familyId={document.familyId} currentId={documentId} />}
    </div>
  );
};
```

**Testing:**
- PDF renders correctly in the viewer via PDF.js
- Email displays RFC 5322 headers (From, To, CC, Subject, Date) in a formatted header block
- Extracted text view shows full text with search term highlighting
- Family navigation: clicking an email shows its attachments; clicking an attachment shows its parent email
- Keyboard shortcuts: Arrow keys navigate between documents in the result set
- Large document (500+ pages) loads progressively without freezing the UI

### Task 3.3: Coding Panel and Review Workflow

**What:** Build the coding panel that displays the matter's coding layout (defined in `matters.coding_layout` JSONB), captures reviewer decisions, and writes them to the `documents.coding` JSONB column. Support single-choice, multi-choice, boolean, text, and date field types. Track time spent per document. Show AI suggestions when available (Phase 5 dependency).

**Design:**

```typescript
// apps/web/src/features/review/CodingPanel.tsx
interface CodingPanelProps {
  document: Document;
  codingLayout: CodingField[];
  onSave: (decisions: CodingDecisions) => void;
}

// apps/api/src/modules/review/coding.service.ts
@Injectable()
export class CodingService {
  async saveDecisions(params: {
    documentId: string;
    matterId: string;
    reviewerId: string;
    batchId?: string;
    decisions: Record<string, { value: string | string[] | boolean; timeSpentMs: number }>;
  }): Promise<void> {
    const codingUpdate: Record<string, unknown> = {};
    for (const [fieldName, decision] of Object.entries(params.decisions)) {
      codingUpdate[fieldName] = {
        value: decision.value,
        reviewer_id: params.reviewerId,
        decided_at: new Date().toISOString(),
        time_spent_ms: decision.timeSpentMs,
        batch_id: params.batchId,
      };
    }

    await this.documentRepo.update(params.documentId, {
      coding: () => `coding || '${JSON.stringify(codingUpdate)}'::jsonb`,
    });

    await this.auditService.emit({
      streamType: 'Document',
      streamId: params.documentId,
      eventType: 'CodingDecisionMade',
      payload: { decisions: codingUpdate, batchId: params.batchId },
      tenantId: this.tenantId,
      actorId: params.reviewerId,
      actorType: 'user',
    });
  }
}
```

**Testing:**
- Single-choice field: selecting "Responsive" saves correctly; changing to "Not Responsive" overwrites
- Multi-choice field: selecting multiple issues saves as array; deselecting one removes it
- Boolean field: toggling "Key Document" saves true/false
- Text field: entering reviewer notes saves text value
- Time tracking: time spent per document is captured (start when document opens, stop when coding is saved)
- Concurrent reviewers: two reviewers coding different fields on the same document do not overwrite each other's coding
- Audit event `CodingDecisionMade` emitted for every coding save with field name, value, and reviewer

### Task 3.4: Review Batch Management

**What:** Build the batch assignment system: create batches from search results, assign batches to reviewers, track batch progress (assigned, in_progress, completed), and enable supervisors to reassign or return batches.

**Design:**

```typescript
// apps/api/src/modules/review/batch.service.ts
@Injectable()
export class BatchService {
  async createBatch(params: {
    matterId: string;
    batchName: string;
    documentIds: string[];
    assignedTo: string;
    assignedBy: string;
    dueDate?: Date;
    batchConfig?: Record<string, unknown>;
  }): Promise<ReviewBatch> {
    const batch = await this.batchRepo.save({
      matterId: params.matterId,
      batchName: params.batchName,
      assignedTo: params.assignedTo,
      assignedBy: params.assignedBy,
      documentCount: params.documentIds.length,
      dueDate: params.dueDate,
      batchConfig: params.batchConfig || {},
    });

    await this.batchDocRepo.save(
      params.documentIds.map((docId, idx) => ({
        batchId: batch.id,
        documentId: docId,
        position: idx,
      }))
    );

    return batch;
  }
}
```

**Testing:**
- Create a batch of 500 documents; verify all documents are assigned to the correct reviewer
- Reviewer opens batch; status changes to `in_progress`; batch progress counter increments as documents are coded
- Batch completion: when all documents are coded, status changes to `completed`
- Supervisor reassigns batch to different reviewer; new reviewer sees the batch in their queue
- Batch cannot include documents already in another active batch (prevents double-review)
- QC review: supervisor can flag random documents from a completed batch for quality check

### Task 3.5: Tagging System

**What:** Implement free-form tags (labels) that reviewers can apply to documents. Tags are matter-scoped, colour-coded, and searchable. Support bulk tagging from search results.

**Design:**

```typescript
// apps/api/src/modules/documents/tags.service.ts
@Injectable()
export class TagService {
  async tagDocuments(params: {
    matterId: string;
    documentIds: string[];
    tagName: string;
    userId: string;
  }): Promise<void> {
    // Ensure tag exists for this matter
    await this.ensureTag(params.matterId, params.tagName, params.userId);

    // Add tag to each document's tags array
    await this.documentRepo
      .createQueryBuilder()
      .update()
      .set({ tags: () => `array_append(tags, '${params.tagName}')` })
      .where('id IN (:...ids)', { ids: params.documentIds })
      .andWhere('matter_id = :matterId', { matterId: params.matterId })
      .andWhere('NOT (:tag = ANY(tags))', { tag: params.tagName })
      .execute();
  }
}
```

**Testing:**
- Create tag "Hot Document" with red colour; verify it appears in the tag list
- Apply tag to a document; verify it appears in the document's tags array
- Bulk tag 200 documents from search results; verify all 200 documents have the tag
- Remove tag from a document; verify it is removed from the array
- Search by tag: filtering by tag "Hot Document" returns exactly the tagged documents
- Tags are matter-scoped: tag "Hot Document" in matter A does not appear in matter B

### Definition of Done — Phase 3
- [ ] Search interface supports boolean, proximity, field-scoped, and date range queries
- [ ] Document viewer renders text, native files, and images with metadata panel
- [ ] Coding panel supports all field types with per-document time tracking
- [ ] Review batches can be created, assigned, tracked, and completed
- [ ] Tagging system supports create, apply, remove, bulk tag, and search by tag
- [ ] All review actions emit audit events
- [ ] Concurrent multi-reviewer access works without data corruption
- [ ] Keyboard shortcuts enable efficient document-by-document review
- [ ] Review workspace loads documents in <500ms (measured at P95)

---

## Phase 4: Email Threading & Near-Duplicate Detection

**Goal:** Reduce redundant review by grouping related documents. Email threading identifies the most inclusive message in a thread; near-duplicate clustering groups textually similar documents.

**Duration estimate:** 3-4 weeks

### Task 4.1: Email Threading Engine

**What:** Build an email threading engine that groups emails by RFC 5322 Message-ID / In-Reply-To / References headers. Identify the "most inclusive" email in each thread (the one containing all prior content). Allow reviewers to code at the thread level or document level.

**Design:**

```typescript
// apps/api/src/modules/processing/email-threading.service.ts
@Injectable()
export class EmailThreadingService {
  async buildThreads(matterId: string): Promise<void> {
    // 1. Get all emails in the matter with message_id and in_reply_to
    const emails = await this.documentRepo.find({
      where: { matterId, mimeType: In(['message/rfc822', 'application/vnd.ms-outlook']) },
      select: ['id', 'emailMessageId', 'emailInReplyTo', 'emailSubject', 'emailDate', 'extractedText'],
    });

    // 2. Build reply chains using Message-ID → In-Reply-To mapping
    const messageIdMap = new Map(emails.map(e => [e.emailMessageId, e]));
    const threads: Map<string, EmailThread> = new Map();

    for (const email of emails) {
      const rootId = this.findThreadRoot(email, messageIdMap);
      if (!threads.has(rootId)) {
        threads.set(rootId, { rootMessageId: rootId, members: [] });
      }
      threads.get(rootId)!.members.push(email);
    }

    // 3. Identify inclusive emails (longest text content in each thread)
    for (const thread of threads.values()) {
      thread.members.sort((a, b) => (b.extractedText?.length || 0) - (a.extractedText?.length || 0));
      thread.inclusiveDocumentId = thread.members[0].id;
    }

    // 4. Persist thread records
    for (const thread of threads.values()) {
      const emailThread = await this.threadRepo.save({
        matterId,
        threadRootMessageId: thread.rootMessageId,
        subjectNormalized: this.normalizeSubject(thread.members[0].emailSubject),
        documentCount: thread.members.length,
      });
      // Save members with position and inclusive flag
    }
  }
}
```

**Testing:**
- Thread 5 emails with proper In-Reply-To chain; verify they form one thread with correct ordering
- Identify inclusive email: the final reply (containing all quoted text) is marked `is_inclusive = true`
- Broken In-Reply-To header: fall back to subject-line matching for threading
- Cross-custodian thread: emails from different custodians in the same thread are grouped correctly
- Thread with attachments: attachments belong to the same family as their parent email, not the thread
- Thread-level coding: coding the inclusive email optionally propagates to all thread members

### Task 4.2: Near-Duplicate Detection

**What:** Implement near-duplicate detection using MinHash / Locality-Sensitive Hashing (LSH). Documents with text similarity above a configurable threshold (default: 90%) are grouped into near-duplicate clusters. Identify a "pivot" document in each cluster as the primary review copy.

**Design:**

```typescript
// apps/api/src/modules/processing/near-duplicate.service.ts
@Injectable()
export class NearDuplicateService {
  async detectNearDuplicates(matterId: string, threshold: number = 0.90): Promise<void> {
    // 1. Get all processed documents with extracted text
    const documents = await this.getDocumentsWithText(matterId);

    // 2. Compute MinHash signatures for each document
    const signatures = documents.map(doc => ({
      id: doc.id,
      signature: this.computeMinHash(doc.extractedText, NUM_PERMUTATIONS),
    }));

    // 3. LSH bucketing to find candidate pairs
    const candidatePairs = this.lshBucketing(signatures, NUM_BANDS, ROWS_PER_BAND);

    // 4. Verify candidates with Jaccard similarity
    const confirmedGroups = this.clusterBySimilarity(candidatePairs, threshold);

    // 5. Persist near-duplicate groups
    for (const group of confirmedGroups) {
      const pivot = group[0]; // highest page count as pivot
      await this.nearDupRepo.save({
        matterId,
        pivotDocumentId: pivot,
        similarityThreshold: threshold,
        memberDocumentIds: group,
        documentCount: group.length,
      });
    }
  }
}
```

**Testing:**
- Two documents differing only by a date in the header are detected as near-duplicates (>90% similarity)
- Two completely different documents are not grouped
- Cluster of 5 near-duplicates: pivot document is correctly identified as the most representative
- Threshold adjustment: setting threshold to 0.95 produces fewer, tighter clusters than 0.90
- Performance: 10,000 documents processed in under 5 minutes
- Near-duplicate groups are searchable and filterable in the review interface

### Definition of Done — Phase 4
- [ ] Email threading correctly groups related emails using RFC 5322 headers
- [ ] Inclusive email identification reduces redundant review
- [ ] Near-duplicate detection clusters similar documents with configurable threshold
- [ ] Review interface shows thread view and near-duplicate group view
- [ ] Thread-level and cluster-level coding propagation works correctly
- [ ] Processing handles 100K+ documents per matter within acceptable time

---

## Phase 5: TAR 2.0 — AI-Assisted Review

**Goal:** Implement Continuous Active Learning (CAL/TAR 2.0) based on the public-domain Cormack & Grossman (2014) algorithm. AI prioritises documents for review, reducing the number of documents attorneys must read.

**Duration estimate:** 5-6 weeks

### Task 5.1: CAL Algorithm Implementation

**What:** Implement the Continuous Active Learning algorithm as a Python FastAPI microservice. The model trains on reviewer coding decisions, scores all unreviewed documents, and surfaces the highest-scored documents next. The algorithm continuously re-trains as more decisions are made.

**Design:**

```python
# apps/ml/tar/cal_service.py
from fastapi import FastAPI
from sklearn.linear_model import LogisticRegression
from sklearn.feature_extraction.text import TfidfVectorizer

app = FastAPI()

class CALModel:
    def __init__(self, matter_id: str, target_field: str, target_value: str):
        self.vectorizer = TfidfVectorizer(max_features=50000, stop_words='english')
        self.classifier = LogisticRegression(C=1.0, max_iter=1000)
        self.is_trained = False

    def train(self, texts: list[str], labels: list[int]) -> dict:
        """Train/retrain on current reviewer decisions."""
        X = self.vectorizer.fit_transform(texts)
        self.classifier.fit(X, labels)
        self.is_trained = True
        return {
            "recall_estimate": self._estimate_recall(X, labels),
            "documents_trained": len(labels),
        }

    def score(self, texts: list[str]) -> list[float]:
        """Score unreviewed documents for prioritisation."""
        X = self.vectorizer.transform(texts)
        probabilities = self.classifier.predict_proba(X)
        return probabilities[:, 1].tolist()  # probability of target class

    def _estimate_recall(self, X, labels) -> float:
        """Estimate recall using cross-validation."""
        from sklearn.model_selection import cross_val_score
        scores = cross_val_score(self.classifier, X, labels, cv=5, scoring='recall')
        return float(scores.mean())

@app.post("/tar/models/{matter_id}/train")
async def train_model(matter_id: str, request: TrainRequest) -> TrainResponse:
    model = get_or_create_model(matter_id, request.target_field, request.target_value)
    metrics = model.train(request.texts, request.labels)
    return TrainResponse(**metrics)

@app.post("/tar/models/{matter_id}/score")
async def score_documents(matter_id: str, request: ScoreRequest) -> ScoreResponse:
    model = get_model(matter_id, request.target_field, request.target_value)
    scores = model.score(request.texts)
    return ScoreResponse(scores=scores)
```

**Testing:**
- Train on 200 labeled documents (100 responsive, 100 not responsive); verify model scores responsive documents higher
- Re-train after 100 additional labels; verify recall estimate improves
- Score 10,000 documents; verify the top 100 by score have higher actual responsiveness rate than a random sample
- Model with insufficient training data (<20 labels) returns appropriate error
- CAL ranking: after each training round, the next batch served to the reviewer contains the highest-scored unreviewed documents
- Recall validation: on a held-out test set, the CAL model achieves >85% recall (measured against ground truth)

### Task 5.2: TAR Model Management API

**What:** Build the NestJS API endpoints for creating TAR models, triggering training rounds, retrieving scores, and monitoring model performance. Store model metadata and performance metrics in the `tar_models` table. Write TAR scores to the `documents.tar_scores` JSONB column.

**Design:**

```typescript
// apps/api/src/modules/tar/tar.controller.ts
@Controller('v1/matters/:matterId/tar-models')
export class TarController {
  @Post()
  @Permissions('tar.create')
  async createModel(@Param('matterId') matterId: string, @Body() dto: CreateTarModelDto) { ... }

  @Post(':modelId/train')
  @Permissions('tar.train')
  async trainModel(@Param('modelId') modelId: string) {
    // Fetch all coded documents for the target field
    // Send to ML service for training
    // Store updated metrics
  }

  @Post(':modelId/score')
  @Permissions('tar.score')
  async scoreDocuments(@Param('modelId') modelId: string) {
    // Fetch all unscored documents
    // Send to ML service for scoring
    // Update documents.tar_scores JSONB
  }

  @Get(':modelId/metrics')
  @Permissions('tar.view')
  async getMetrics(@Param('modelId') modelId: string) { ... }
}
```

**Testing:**
- Create TAR model; verify database record with correct matter_id, target field, and status 'training'
- Train model; verify recall, precision, and F1 estimates are stored
- Score documents; verify `tar_scores` JSONB on each document contains the model ID and score
- Review dashboard shows TAR performance chart with recall over training rounds
- Multiple TAR models per matter (e.g., one for "Responsive", one for "Privileged") work independently

### Task 5.3: AI-Assisted Review Integration in UI

**What:** Enhance the review workspace to show AI confidence scores alongside each document, display suggested coding values, and allow reviewers to accept or override AI suggestions. Track AI suggestion acceptance rate.

**Design:**

```typescript
// apps/web/src/features/review/AiSuggestion.tsx
interface AiSuggestionProps {
  fieldName: string;
  suggestedValue: string;
  confidence: number;
  onAccept: () => void;
  onOverride: (value: string) => void;
}

const AiSuggestion: React.FC<AiSuggestionProps> = ({
  fieldName, suggestedValue, confidence, onAccept, onOverride
}) => (
  <div className="ai-suggestion">
    <span className="confidence-badge" style={{ opacity: confidence }}>
      AI: {suggestedValue} ({(confidence * 100).toFixed(0)}%)
    </span>
    <button onClick={onAccept}>Accept</button>
  </div>
);
```

**Testing:**
- Documents with TAR score >0.8 show "Likely Responsive" indicator
- Reviewer accepts AI suggestion; `ai_suggested: true` is stored in the coding JSONB
- Reviewer overrides AI suggestion; `ai_suggested: true` and `ai_overridden: true` are stored
- AI acceptance rate dashboard shows percentage of AI suggestions accepted vs. overridden per reviewer
- AI suggestion is never shown as a final determination — reviewer must explicitly accept or override

### Definition of Done — Phase 5
- [ ] CAL algorithm implemented and tested with >85% recall on benchmark datasets
- [ ] TAR models can be created, trained, scored, and monitored per matter
- [ ] Document prioritisation: highest-scored documents are served first in review batches
- [ ] AI suggestions displayed in review UI with accept/override workflow
- [ ] AI acceptance rate tracked per reviewer and per model
- [ ] All AI actions emit audit events distinguishing human vs. AI actor
- [ ] TAR 2.0 training round completes in <10 minutes for 100K documents

---

## Phase 6: Privilege Review & Logging

**Goal:** Implement the FRE 502-compliant privilege review workflow: AI pre-flagging of potentially privileged documents, attorney-in-the-loop confirmation, and automated privilege log generation.

**Duration estimate:** 4-5 weeks

### Task 6.1: Privilege Classification Model

**What:** Build a privilege classification ML model (Python/FastAPI) that identifies potentially privileged communications — attorney-client privilege and work product doctrine. The model classifies based on: presence of attorney names/email domains, legal terminology patterns, and communication patterns (e.g., "privileged and confidential" disclaimers). This is pre-flagging only — all designations require attorney confirmation per FRE 502.

**Design:**

```python
# apps/ml/privilege/privilege_classifier.py
class PrivilegeClassifier:
    def __init__(self):
        self.attorney_patterns = [
            r'\b(attorney|counsel|lawyer|esq\.?)\b',
            r'\bprivileged\s+and\s+confidential\b',
            r'\battorney[\-\s]client\b',
            r'\bwork\s+product\b',
            r'\blegal\s+advice\b',
        ]
        self.model = None  # Trained classifier

    def pre_flag(self, document: dict) -> PrivilegePreFlagResult:
        """Pre-flag a document for potential privilege. Returns confidence score."""
        features = self._extract_features(document)
        if self.model:
            confidence = float(self.model.predict_proba([features])[0][1])
        else:
            confidence = self._rule_based_score(document)

        return PrivilegePreFlagResult(
            is_potentially_privileged=confidence > 0.5,
            confidence=confidence,
            privilege_type=self._infer_privilege_type(document),
            reasoning=self._generate_reasoning(document, features),
        )
```

**Testing:**
- Email from in-house counsel to CEO with "privileged and confidential" header flagged at >0.7 confidence
- Email between two non-attorney business people about sales targets NOT flagged
- Work product document (draft brief) flagged as work product doctrine
- Attorney domain detection: emails to/from @lawfirm.com addresses receive privilege score boost
- False positive rate <20% on test corpus (pre-flagging is intentionally over-inclusive)
- All pre-flagging results store `ai_flagged: true` in the `documents.privilege` JSONB

### Task 6.2: Attorney Confirmation Workflow

**What:** Build the privilege review queue where attorneys review AI-flagged documents, confirm or reject privilege designations, select privilege type and basis, and approve final designations. Enforce that only users with the `attorney` role can confirm privilege. Track FRE 502 compliance.

**Design:**

```typescript
// apps/api/src/modules/privilege/privilege.service.ts
@Injectable()
export class PrivilegeService {
  async confirmPrivilege(params: {
    documentId: string;
    attorneyId: string;
    isPrivileged: boolean;
    privilegeType?: string;
    privilegeBasis?: string;
    requiresRedaction?: boolean;
  }): Promise<void> {
    // Verify the user has attorney role
    const user = await this.userService.findById(params.attorneyId);
    if (!user.roles.some(r => r.name === 'supervising_attorney' || r.name === 'admin')) {
      throw new ForbiddenException('Only attorneys can confirm privilege designations');
    }

    const privilegeData = {
      is_privileged: params.isPrivileged,
      privilege_type: params.privilegeType,
      privilege_basis: params.privilegeBasis,
      attorney_confirmed: true,
      attorney_id: params.attorneyId,
      attorney_confirmed_at: new Date().toISOString(),
      requires_redaction: params.requiresRedaction || false,
    };

    await this.documentRepo.update(params.documentId, {
      privilege: privilegeData,
    });

    await this.auditService.emit({
      streamType: 'Document',
      streamId: params.documentId,
      eventType: 'PrivilegeAttorneyConfirmed',
      payload: privilegeData,
      actorId: params.attorneyId,
      actorType: 'user',
    });
  }
}
```

**Testing:**
- AI-flagged document appears in the privilege review queue
- Attorney confirms privilege with type "attorney_client" and basis text; `attorney_confirmed: true` is stored
- Attorney rejects AI flag; `is_privileged: false` and `attorney_confirmed: true` are stored
- Non-attorney user attempts to confirm privilege; receives 403 Forbidden
- Privilege confirmation emits `PrivilegeAttorneyConfirmed` audit event
- Privilege designation cannot be changed after production includes the document (requires explicit override)

### Task 6.3: Privilege Log Generator

**What:** Auto-generate a privilege log from all privilege-designated documents in a matter. The log includes standard fields: Bates range, date, document type, author, recipients, subject, privilege claimed, and description. Export as CSV (Concordance-compatible) and PDF.

**Design:**

```typescript
// apps/api/src/modules/privilege/privilege-log.service.ts
@Injectable()
export class PrivilegeLogService {
  async generateLog(matterId: string): Promise<PrivilegeLogEntry[]> {
    const privilegedDocs = await this.documentRepo.find({
      where: {
        matterId,
        privilege: Raw(alias => `${alias}->>'is_privileged' = 'true' AND ${alias}->>'attorney_confirmed' = 'true'`),
      },
    });

    return privilegedDocs.map(doc => ({
      batesBegin: this.getBatesForDoc(doc),
      batesEnd: this.getBatesEndForDoc(doc),
      docDate: doc.emailDate || doc.metadata?.date_created,
      docType: this.classifyDocType(doc),
      author: doc.emailFrom || doc.metadata?.author,
      recipients: [
        ...(doc.emailTo || []),
        ...(doc.emailCc || []),
      ].join('; '),
      subject: doc.emailSubject || doc.fileName,
      privilegeClaimed: this.formatPrivilegeType(doc.privilege.privilege_type),
      privilegeDescription: doc.privilege.privilege_basis,
    }));
  }

  async exportCsv(matterId: string): Promise<Buffer> { ... }
  async exportPdf(matterId: string): Promise<Buffer> { ... }
}
```

**Testing:**
- Generate privilege log for a matter with 50 privileged documents; verify all 50 appear in the log
- Log fields match standard privilege log format (Bates range, date, type, author, recipients, subject, privilege, description)
- CSV export opens correctly in Excel with proper field separation
- PDF export renders a formatted table suitable for court submission
- Documents without attorney confirmation are excluded from the log
- Log entry for a redacted document includes redaction notation

### Definition of Done — Phase 6
- [ ] Privilege classifier pre-flags potentially privileged documents with >80% recall (over-inclusive)
- [ ] Attorney confirmation workflow enforces FRE 502 (only attorney role can confirm)
- [ ] Privilege log generates with all standard fields
- [ ] Privilege log exports as CSV and PDF
- [ ] All privilege actions emit audit events with full provenance chain
- [ ] Privilege review queue shows AI confidence, flagged count, and completion progress

---

## Phase 7: Production Generation

**Goal:** Build the EDRM "Production" stage — packaging reviewed documents for delivery to opposing counsel, courts, or agencies. Support TIFF + OCR, native file, and PDF production formats with Concordance DAT and Opticon OPT load files.

**Duration estimate:** 4-5 weeks

### Task 7.1: Production Configuration and Document Selection

**What:** Build the production creation workflow: define production settings (Bates prefix, numbering, image format, load file format, confidentiality stamps), select documents from the reviewed set (by search, tag, or coding value), and validate that no un-reviewed or privileged documents are included.

**Design:**

```typescript
// apps/api/src/modules/productions/production.service.ts
@Injectable()
export class ProductionService {
  async createProduction(params: CreateProductionDto): Promise<Production> {
    const production = await this.productionRepo.save({
      matterId: params.matterId,
      productionName: params.productionName,
      productionNumber: params.productionNumber,
      batesPrefix: params.batesPrefix,
      batesStart: params.batesStart || 1,
      batesPadding: params.batesPadding || 7,
      formatConfig: {
        imageFormat: params.imageFormat || 'tiff',
        loadFileFormat: params.loadFileFormat || 'concordance_dat',
        includeNative: params.includeNative || false,
        includeTextFiles: params.includeTextFiles || true,
        confidentialityStamp: params.confidentialityStamp,
        dpi: params.dpi || 300,
      },
      status: 'draft',
      createdBy: params.userId,
    });
    return production;
  }

  async addDocuments(productionId: string, documentIds: string[]): Promise<ValidationResult> {
    // Validate: no privileged documents
    const privilegedDocs = await this.findPrivilegedDocs(documentIds);
    if (privilegedDocs.length > 0) {
      return { valid: false, errors: [`${privilegedDocs.length} privileged documents cannot be produced`] };
    }

    // Validate: no un-reviewed documents
    const unreviewedDocs = await this.findUnreviewedDocs(documentIds);
    if (unreviewedDocs.length > 0) {
      return { valid: false, warnings: [`${unreviewedDocs.length} documents have not been reviewed`] };
    }

    // Assign Bates numbers sequentially
    let batesCounter = await this.getNextBatesNumber(productionId);
    for (const docId of documentIds) {
      const doc = await this.documentRepo.findOneOrFail(docId);
      const pageCount = doc.pageCount || 1;
      await this.prodDocRepo.save({
        productionId,
        documentId: docId,
        batesBegin: this.formatBates(production.batesPrefix, batesCounter, production.batesPadding),
        batesEnd: this.formatBates(production.batesPrefix, batesCounter + pageCount - 1, production.batesPadding),
        pageCount,
      });
      batesCounter += pageCount;
    }

    return { valid: true };
  }
}
```

**Testing:**
- Create production with prefix "ACME", start 1, padding 7; first document gets Bates "ACME0000001"
- Adding a privileged document returns validation error
- Adding an un-reviewed document returns validation warning (can be overridden)
- Sequential Bates numbering: 3-page document gets ACME0000001-ACME0000003; next document starts at ACME0000004
- Production with 10,000 documents completes Bates assignment in <30 seconds

### Task 7.2: TIFF/PDF Image Generation

**What:** Convert documents to TIFF or PDF images for production. Apply Bates stamps, confidentiality stamps, and redactions to each page. Use LibreOffice for Office-to-PDF conversion and ImageMagick/Ghostscript for PDF-to-TIFF conversion.

**Design:**

```typescript
// apps/api/src/modules/productions/imaging.service.ts
@Injectable()
export class ImagingService {
  async generateImages(productionId: string): Promise<void> {
    const prodDocs = await this.prodDocRepo.find({ where: { productionId } });

    for (const prodDoc of prodDocs) {
      const document = await this.documentRepo.findOneOrFail(prodDoc.documentId);
      const nativeBuffer = await this.s3Service.getObject(document.s3Key);

      // Convert to PDF first (if not already PDF)
      let pdfBuffer: Buffer;
      if (document.mimeType === 'application/pdf') {
        pdfBuffer = nativeBuffer;
      } else {
        pdfBuffer = await this.convertToPdf(nativeBuffer, document.mimeType);
      }

      // Apply stamps (Bates number, confidentiality)
      pdfBuffer = await this.applyStamps(pdfBuffer, {
        batesPrefix: production.batesPrefix,
        batesStart: prodDoc.batesBegin,
        confidentialityStamp: production.formatConfig.confidentialityStamp,
      });

      // Convert to TIFF if required
      if (production.formatConfig.imageFormat === 'tiff') {
        const tiffPages = await this.pdfToTiff(pdfBuffer, production.formatConfig.dpi);
        // Upload each page to S3
        const imagePaths = [];
        for (let i = 0; i < tiffPages.length; i++) {
          const key = `${productionId}/images/${prodDoc.batesBegin}_${i + 1}.tif`;
          await this.s3Service.putObject(key, tiffPages[i]);
          imagePaths.push(key);
        }
        await this.prodDocRepo.update(prodDoc.id, { imagePaths });
      }
    }
  }
}
```

**Testing:**
- PDF document produces correctly stamped TIFF images (Bates number bottom-right, confidentiality stamp bottom-center)
- DOCX document converts to PDF then TIFF with correct page count
- Multi-page document produces one TIFF per page with sequential Bates numbers
- Bates stamp is legible at 300 DPI
- Confidentiality stamp "ATTORNEYS EYES ONLY" renders correctly
- Image resolution matches configured DPI (300 by default)

### Task 7.3: Concordance DAT & Opticon OPT Load File Generation

**What:** Generate load files in the two de facto standard formats: Concordance DAT (metadata) and Opticon OPT (image paths). These load files enable opposing counsel to import the production into their own review platform.

**Design:**

```typescript
// packages/load-files/src/concordance-dat.ts
export class ConcordanceDatGenerator {
  generate(documents: ProductionDocument[]): string {
    const DELIMITERS = {
      field: '\x14',      // ASCII 20 (Concordance field separator)
      text: '\xFE',       // ASCII 254 (text qualifier)
      newline: '\x0A',    // ASCII 10 (newline within fields)
    };

    const headers = [
      'BEGBATES', 'ENDBATES', 'BEGATTACH', 'ENDATTACH',
      'FROM', 'TO', 'CC', 'BCC', 'SUBJECT', 'DATECREATED',
      'DATEMODIFIED', 'CUSTODIAN', 'DOCTYPE', 'FILENAME',
      'FILEPATH', 'TEXTPATH', 'NATIVEPATH',
    ];

    let output = headers.join(DELIMITERS.field) + '\n';

    for (const doc of documents) {
      const fields = [
        doc.batesBegin, doc.batesEnd,
        doc.familyBatesBegin, doc.familyBatesEnd,
        doc.emailFrom, doc.emailTo, doc.emailCc, doc.emailBcc,
        doc.emailSubject, doc.dateCreated, doc.dateModified,
        doc.custodianName, doc.docType, doc.fileName,
        doc.imagePath, doc.textPath, doc.nativePath,
      ];
      output += fields.map(f => `${DELIMITERS.text}${f || ''}${DELIMITERS.text}`).join(DELIMITERS.field) + '\n';
    }
    return output;
  }
}

// packages/load-files/src/opticon-opt.ts
export class OpticonOptGenerator {
  generate(documents: ProductionDocument[]): string {
    let output = '';
    for (const doc of documents) {
      for (let i = 0; i < doc.imagePaths.length; i++) {
        const bates = this.formatBatesPage(doc.batesBegin, i);
        const isFirstPage = i === 0;
        output += `${bates},${doc.volumeName},${doc.imagePaths[i]},${isFirstPage ? 'Y' : ''},${isFirstPage ? 'D' : ''},${doc.imagePaths.length}\n`;
      }
    }
    return output;
  }
}
```

**Testing:**
- Concordance DAT file opens correctly in Relativity's import wizard (validated against format spec)
- DAT file uses correct delimiters (ASCII 20 field separator, ASCII 254 text qualifier)
- Opticon OPT file references correct image paths for each page
- First page of each document has 'Y' flag in the OPT file
- Load files for 10,000 documents generate in <60 seconds
- Round-trip test: export production, import DAT/OPT into a test instance, verify all metadata matches

### Task 7.4: Production Export and Delivery

**What:** Package the production (images, text files, native files, load files) into volumes, ZIP them, and make them available for download or SFTP delivery. Track delivery status and generate a production hash for integrity verification.

**Design:**

```typescript
// apps/api/src/modules/productions/export.service.ts
@Injectable()
export class ProductionExportService {
  async exportProduction(productionId: string): Promise<ExportResult> {
    const production = await this.productionRepo.findOneOrFail(productionId);

    // Generate load files
    const datContent = this.datGenerator.generate(prodDocs);
    const optContent = this.optGenerator.generate(prodDocs);

    // Package into volume structure
    // VOLUME001/
    //   IMAGES/001/  (TIFF files)
    //   NATIVES/001/  (native files if included)
    //   TEXT/001/     (extracted text files)
    //   ACME_PROD001.dat
    //   ACME_PROD001.opt

    // Create ZIP
    const zipBuffer = await this.createZip(productionId);

    // Upload to S3
    const zipKey = `${productionId}/export/PROD-${production.productionNumber}.zip`;
    await this.s3Service.putObject(zipKey, zipBuffer);

    // Compute production hash
    const productionHash = createHash('sha256').update(zipBuffer).digest('hex');

    await this.productionRepo.update(productionId, {
      status: 'finalised',
      batesEnd: lastBatesNumber,
      deliveryConfig: { productionHash },
    });

    return { downloadUrl: await this.s3Service.getSignedUrl(zipKey), productionHash };
  }
}
```

**Testing:**
- Export a production of 1,000 documents; verify ZIP contains correct folder structure
- DAT and OPT files are at the root of the volume
- Image files are in the IMAGES directory with correct naming
- Production hash (SHA-256) matches re-computation of the ZIP
- Download URL is time-limited (signed URL expires after configured period)
- Production status transitions: draft → generating → qc_review → finalised → delivered
- Audit event `ProductionExported` and `ProductionDelivered` emitted with hash and recipient

### Definition of Done — Phase 7
- [ ] Production workflow: create → select documents → generate images → generate load files → export
- [ ] TIFF images produced at 300 DPI with Bates stamps and confidentiality stamps
- [ ] Concordance DAT load file validates against industry format specification
- [ ] Opticon OPT load file correctly maps pages to images
- [ ] Privileged documents are blocked from production with validation error
- [ ] Production export packages into ZIP volumes with correct structure
- [ ] SHA-256 production hash enables integrity verification
- [ ] Production of 10,000 documents completes in <2 hours

---

## Phase 8: Communication Analytics (Graph Layer)

**Goal:** Add the graph layer from Model 4 — communication network analysis, custodian relationship mapping, and visual analytics. This enhances early case assessment and privilege chain detection.

**Duration estimate:** 4-5 weeks

### Task 8.1: Graph Construction Pipeline

**What:** Build the pipeline that constructs graph nodes (persons, email addresses, organisations) and edges (sent_email_to, cc_on_email, reports_to) from processed email documents. Run during document ingestion and as a batch job for existing matters.

**Design:**

```sql
-- db/migrations/010_graph_layer.sql
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    matter_id       UUID,
    node_type       VARCHAR(50) NOT NULL,
    entity_id       UUID,
    label           VARCHAR(500) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    degree_centrality    DECIMAL(10,6) DEFAULT 0,
    betweenness_centrality DECIMAL(10,6) DEFAULT 0,
    pagerank            DECIMAL(10,6) DEFAULT 0,
    community_id        INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    matter_id       UUID,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    is_directed     BOOLEAN NOT NULL DEFAULT true,
    weight          DECIMAL(10,4) NOT NULL DEFAULT 1.0,
    valid_from      TIMESTAMPTZ,
    valid_to        TIMESTAMPTZ,
    properties      JSONB NOT NULL DEFAULT '{}',
    source_document_ids UUID[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- Process 1,000 emails; verify person nodes created for all unique senders and recipients
- Email from A to B creates a `sent_email_to` edge from A's node to B's node
- CC recipients create `cc_on_email` edges with weight 0.5
- Duplicate edges are aggregated (weight increases, source_document_ids array grows)
- Cross-custodian edges correctly link internal and external persons
- Graph construction for 100,000 emails completes in <10 minutes

### Task 8.2: Graph Analytics and Visualization

**What:** Implement graph algorithms (PageRank, community detection, centrality) and a React-based graph visualization component showing communication networks. Enable attorneys to explore who communicated with whom, identify key custodians, and detect communication clusters.

**Design:**

```typescript
// apps/web/src/features/analytics/CommunicationGraph.tsx
import ReactFlow, { type Node, type Edge } from 'reactflow';

interface CommunicationGraphProps {
  matterId: string;
  filters: {
    dateRange?: { start: Date; end: Date };
    custodianIds?: string[];
    minWeight?: number;
  };
}

const CommunicationGraph: React.FC<CommunicationGraphProps> = ({ matterId, filters }) => {
  const { data } = useQuery(['graph', matterId, filters], () =>
    api.analytics.getCommunicationGraph(matterId, filters)
  );

  const nodes: Node[] = data?.nodes.map(n => ({
    id: n.id,
    data: { label: n.label, type: n.nodeType, pagerank: n.pagerank },
    position: n.layoutPosition,
    style: { width: Math.max(30, n.pagerank * 100) },  // Size by PageRank
  }));

  const edges: Edge[] = data?.edges.map(e => ({
    id: e.id,
    source: e.sourceNodeId,
    target: e.targetNodeId,
    label: `${e.weight} emails`,
    style: { strokeWidth: Math.max(1, e.weight / 10) },  // Thickness by weight
  }));

  return <ReactFlow nodes={nodes} edges={edges} />;
};
```

**Testing:**
- Graph renders with nodes sized by PageRank; most-connected people are largest
- Edges show communication volume; thicker lines = more emails
- Date range filter shows communication patterns within the selected period only
- Community detection: nodes are colour-coded by detected community
- Click a node to see all connected persons and email counts
- Click an edge to see the underlying documents
- Graph handles 1,000 nodes and 10,000 edges without UI performance degradation

### Definition of Done — Phase 8
- [ ] Graph construction pipeline creates nodes and edges from processed emails
- [ ] PageRank, centrality, and community detection algorithms run correctly
- [ ] Communication graph visualization renders in the review workspace
- [ ] Privilege chain detection: graph query identifies attorney nodes in communication paths
- [ ] Temporal filtering shows communication patterns within specific date ranges
- [ ] Graph analytics help identify key custodians for early case assessment

---

## Phase 9: Legal Hold & Custodian Management

**Goal:** Build the complete legal hold lifecycle — custodian identification, hold notice creation, distribution, acknowledgment tracking, escalation, and release. This is the EDRM "Preservation" stage.

**Duration estimate:** 4-5 weeks

### Task 9.1: Custodian Management

**What:** Build CRUD for custodians with data source mapping (email, OneDrive, Slack, local drives). Support custodian interview workflow where custodians identify their data systems. Store interview results and data source details in JSONB.

**Design:**

```typescript
// apps/api/src/modules/custodians/custodian.service.ts
@Injectable()
export class CustodianService {
  async create(dto: CreateCustodianDto): Promise<Custodian> {
    return this.custodianRepo.save({
      tenantId: dto.tenantId,
      firstName: dto.firstName,
      lastName: dto.lastName,
      email: dto.email,
      department: dto.department,
      title: dto.title,
      dataSources: dto.dataSources || [],
      interviewData: null,
    });
  }

  async recordInterview(custodianId: string, interviewData: InterviewDataDto): Promise<void> {
    await this.custodianRepo.update(custodianId, {
      interviewData: {
        interview_date: new Date().toISOString(),
        interviewer: interviewData.interviewerName,
        relevant_systems: interviewData.relevantSystems,
        date_range_relevant: interviewData.dateRange,
        key_contacts_mentioned: interviewData.keyContacts,
        notes: interviewData.notes,
      },
    });
  }
}
```

**Testing:**
- Create custodian with 3 data sources (email, OneDrive, Slack); verify all stored correctly
- Record interview data; verify JSONB contains interview date, relevant systems, and notes
- Duplicate custodian email within same tenant rejected
- Custodian search by name, department, and email works correctly
- Data source types validated against allowed values

### Task 9.2: Legal Hold Lifecycle

**What:** Implement the full legal hold workflow: create hold with preservation scope, attach custodians, send hold notices via email, track acknowledgments, auto-send reminders, escalate non-responders, and release holds when no longer needed.

**Design:**

```typescript
// apps/api/src/modules/legal-holds/legal-hold.service.ts
@Injectable()
export class LegalHoldService {
  async issueHold(holdId: string): Promise<void> {
    const hold = await this.holdRepo.findOneOrFail(holdId, { relations: ['custodians'] });

    // Send notices to all custodians
    for (const custodianHold of hold.custodians) {
      await this.emailService.send({
        to: custodianHold.custodian.email,
        subject: hold.holdConfig.notice_template.subject,
        html: this.renderNotice(hold, custodianHold.custodian),
      });

      await this.holdCustodianRepo.update(custodianHold.id, {
        status: 'notified',
        notifiedAt: new Date(),
      });
    }

    await this.holdRepo.update(holdId, { status: 'active', dateIssued: new Date() });

    await this.auditService.emit({
      streamType: 'LegalHold',
      streamId: holdId,
      eventType: 'LegalHoldIssued',
      payload: { holdName: hold.holdName, custodianCount: hold.custodians.length },
      actorType: 'user',
    });
  }

  async processAcknowledgment(holdId: string, custodianId: string): Promise<void> { ... }
  async sendReminders(holdId: string): Promise<void> { ... }
  async escalateNonResponders(holdId: string): Promise<void> { ... }
  async releaseHold(holdId: string, releasedBy: string): Promise<void> { ... }
}
```

**Testing:**
- Issue hold with 5 custodians; verify all 5 receive email notices and status changes to 'notified'
- Custodian acknowledges; status changes to 'acknowledged', timestamp recorded
- Reminder sent after configured interval (e.g., 3 days); reminder counter increments
- Escalation after 2 failed reminders: manager email sent, status changes to 'escalated'
- Release hold: all custodian statuses change to 'released', hold status changes to 'released'
- Dashboard shows hold progress: X acknowledged, Y pending, Z escalated
- Audit events emitted for: HoldIssued, CustodianNotified, CustodianAcknowledged, HoldReleased

### Definition of Done — Phase 9
- [ ] Custodian CRUD with data source mapping and interview recording
- [ ] Legal hold lifecycle: create → issue → track acknowledgments → remind → escalate → release
- [ ] Email notices sent with customisable templates
- [ ] Automatic reminders and escalation based on configurable intervals
- [ ] Legal hold dashboard shows real-time acknowledgment status
- [ ] All legal hold actions emit audit events for defensibility
- [ ] Multiple concurrent holds per matter supported

---

## Phase 10: Early Case Assessment

**Goal:** Build the early case assessment (ECA) dashboard that helps litigation teams understand the scope of a matter before committing to full review. Includes data universe statistics, custodian analysis, proportionality analytics, and estimated review burden.

**Duration estimate:** 3-4 weeks

### Task 10.1: Data Universe Dashboard

**What:** Build a dashboard showing matter-level statistics: total custodians, total data sources, total documents, total size, document type distribution, date range distribution, and custodian-level breakdowns.

**Design:**

```typescript
// apps/api/src/modules/analytics/eca.service.ts
@Injectable()
export class EcaService {
  async getDataUniverseStats(matterId: string): Promise<DataUniverseStats> {
    const stats = await this.documentRepo
      .createQueryBuilder('d')
      .select([
        'COUNT(DISTINCT d.custodian_id) AS total_custodians',
        'COUNT(*) AS total_documents',
        'SUM(d.file_size_bytes) AS total_size_bytes',
        'COUNT(*) FILTER (WHERE d.is_duplicate) AS duplicate_count',
        'COUNT(*) FILTER (WHERE d.is_nist) AS nist_count',
        'MIN(d.email_date) AS earliest_date',
        'MAX(d.email_date) AS latest_date',
      ])
      .where('d.matter_id = :matterId', { matterId })
      .getRawOne();

    const byType = await this.getDocCountByMimeType(matterId);
    const byCustodian = await this.getDocCountByCustodian(matterId);
    const byMonth = await this.getDocCountByMonth(matterId);

    return { ...stats, byType, byCustodian, byMonth };
  }
}
```

**Testing:**
- Dashboard correctly shows total documents, size, custodian count
- Document type chart shows distribution (PDF 40%, DOCX 20%, Email 30%, Other 10%)
- Date histogram shows monthly document volume distribution
- Custodian breakdown shows per-custodian document count and size
- De-duplicated and de-NISTed documents are shown separately from reviewable count
- Dashboard loads in <2 seconds for matters with 1M+ documents

### Task 10.2: Proportionality Estimator

**What:** Build an AI-assisted proportionality estimator that projects review hours, cost, and a proportionality score based on document volume, type distribution, and historical review rates. Generate a memo aligned with FRCP Rule 26.

**Design:**

```python
# apps/ml/eca/proportionality.py
class ProportionalityEstimator:
    REVIEW_RATES = {
        'email': 60,           # documents per hour (industry average)
        'office_document': 40,
        'pdf': 50,
        'image': 30,           # OCR review is slower
    }
    COST_PER_HOUR = 75.0       # blended reviewer rate (adjustable)

    def estimate(self, stats: DataUniverseStats) -> ProportionalityEstimate:
        total_review_hours = sum(
            count / self.REVIEW_RATES.get(doc_type, 45)
            for doc_type, count in stats.by_type.items()
        )
        total_review_cost = total_review_hours * self.COST_PER_HOUR

        return ProportionalityEstimate(
            estimated_review_hours=total_review_hours,
            estimated_review_cost=total_review_cost,
            proportionality_memo=self.generate_memo(stats, total_review_hours, total_review_cost),
        )
```

**Testing:**
- 100,000 documents estimates ~2,000 review hours at industry rates
- Cost estimate uses configurable reviewer hourly rate
- Proportionality memo includes: data universe scope, estimated burden, and FRCP Rule 26 language
- Memo is exportable as PDF for meet-and-confer discussions
- Estimate updates dynamically as documents are processed and de-duplicated

### Definition of Done — Phase 10
- [ ] ECA dashboard shows data universe statistics with charts
- [ ] Per-custodian breakdown shows document volumes and data source types
- [ ] Proportionality estimator projects review hours and cost
- [ ] FRCP Rule 26 proportionality memo generated as PDF
- [ ] Dashboard loads performantly for large matters (1M+ documents)

---

## Phase 11: DSAR Integration & GDPR Compliance

**Goal:** Add Data Subject Access Request (DSAR) management alongside legal hold for organisations managing both litigation and privacy obligations. Handle GDPR data minimisation and right-to-erasure tensions with preservation obligations.

**Duration estimate:** 3-4 weeks

### Task 11.1: DSAR Workflow

**What:** Build a DSAR management module that tracks data subject requests, identifies relevant documents across matters, manages redaction of personal data, and tracks response deadlines. Integrate with the legal hold module to flag conflicts (data subject requests vs. litigation preservation).

**Design:**

```typescript
// apps/api/src/modules/dsar/dsar.service.ts
@Injectable()
export class DsarService {
  async createRequest(dto: CreateDsarDto): Promise<DsarRequest> { ... }

  async identifyConflicts(dsarId: string): Promise<DsarConflict[]> {
    // Check if any data matching this DSAR is under legal hold
    const dsar = await this.dsarRepo.findOneOrFail(dsarId);
    const activeHolds = await this.holdRepo.find({
      where: {
        status: 'active',
        // Check custodians matching the data subject
      },
    });

    return activeHolds.map(hold => ({
      dsarId: dsar.id,
      holdId: hold.id,
      holdName: hold.holdName,
      conflict: 'Data subject request conflicts with active litigation hold',
      recommendation: 'GDPR Article 17(3)(e) exempts data required for legal proceedings',
    }));
  }
}
```

**Testing:**
- Create DSAR with data subject email; system identifies all documents involving that person
- Conflict detection: DSAR for a person under active legal hold returns conflict warning
- GDPR exemption: system flags Article 17(3)(e) exemption for litigation-preserved data
- DSAR response deadline tracking with alerts at 30, 15, and 5 days remaining
- DSAR export: package all data subject documents with personal data redactions applied

### Definition of Done — Phase 11
- [ ] DSAR creation and lifecycle management
- [ ] Conflict detection between DSARs and active legal holds
- [ ] GDPR Article 17(3)(e) exemption flagging
- [ ] Deadline tracking with automated alerts
- [ ] Data subject document identification across matters

---

## Phase 12: Cross-Matter Learning & Advanced AI

**Goal:** Enable AI models to learn from reviewer decisions across matters (with confidentiality controls) and deploy advanced AI features: generative AI document summarisation, natural-language search, and case strategy assistance.

**Duration estimate:** 5-6 weeks

### Task 12.1: Cross-Matter Learning with Confidentiality Partitioning

**What:** Build a system where TAR and privilege models can benefit from reviewer decisions made on similar document types in previous matters, without leaking matter-specific data. Use federated-learning-inspired techniques: share model weights, not documents.

**Design:**

```python
# apps/ml/cross-matter/federated_learning.py
class CrossMatterLearningService:
    def create_base_model(self, document_type: str, tenant_id: str) -> BaseModel:
        """
        Train a base model from anonymised reviewer patterns across
        completed matters within the same tenant.
        """
        # 1. Get completed matters for this tenant
        # 2. Extract feature vectors + labels (no document text, just TF-IDF features)
        # 3. Train aggregate model
        # 4. Store as tenant-level base model
        pass

    def apply_base_model(self, matter_id: str, model_type: str) -> list[float]:
        """
        Apply tenant base model to a new matter's documents for initial scoring.
        Matter-specific fine-tuning happens via TAR 2.0 as reviewers code.
        """
        pass
```

**Testing:**
- Base model trained on 3 completed matters scores new matter documents higher for responsive documents
- No document text from previous matters is stored in or retrievable from the base model
- Cross-matter learning only operates within the same tenant (no cross-tenant data leakage)
- Fine-tuning via TAR 2.0 overrides base model scores as matter-specific training progresses
- Audit log records when cross-matter model is applied, including source matter IDs (for compliance)

### Task 12.2: Generative AI Document Analysis

**What:** Add a generative AI assistant (via LLM API) that can summarise documents, answer questions about the document corpus, and suggest search terms. All AI outputs include disclaimers per ABA Formal Opinion 512. Implement an MCP server endpoint for structured AI agent access to case data.

**Design:**

```typescript
// apps/api/src/modules/ai/ai-assistant.service.ts
@Injectable()
export class AiAssistantService {
  async summariseDocument(documentId: string): Promise<AiSummary> {
    const doc = await this.documentRepo.findOneOrFail(documentId);

    const response = await this.llmClient.complete({
      system: 'You are a legal document analysis assistant. Provide factual summaries only. Do not provide legal advice.',
      prompt: `Summarise the following document:\n\n${doc.extractedText}`,
    });

    return {
      summary: response.text,
      disclaimer: 'AI-generated summary. Must be reviewed by a licensed attorney before reliance. ABA Formal Opinion 512 requires competent supervision of AI tools.',
      documentId,
      generatedAt: new Date(),
    };
  }
}
```

**Testing:**
- Document summary produces a concise 3-5 sentence summary of a legal document
- Summary includes ABA Formal Opinion 512 disclaimer
- Audit event `AiSummaryGenerated` emitted with model version, document ID, and actor_type='ai_agent'
- MCP server endpoint exposes case document metadata for LLM agent integration
- AI responses for privileged documents are blocked unless requesting user has attorney role

### Definition of Done — Phase 12
- [ ] Cross-matter base model improves initial document scoring for new matters
- [ ] Confidentiality partitioning verified: no document text leaks across matters
- [ ] Generative AI document summarisation with ABA-compliant disclaimers
- [ ] MCP server endpoint for structured AI agent access
- [ ] All AI interactions logged with actor_type='ai_agent' for defensibility
- [ ] AI features are opt-in and configurable per tenant

---

## Summary

| Phase | Name | Duration | Dependencies | Track |
|-------|------|----------|-------------|-------|
| 1 | Foundation & Infrastructure | 4-5 weeks | None | Core |
| 2 | Document Processing Pipeline | 5-6 weeks | Phase 1 | A |
| 3 | Search & Document Review | 6-7 weeks | Phase 2 | A |
| 4 | Email Threading & Near-Duplicate | 3-4 weeks | Phase 2 | A |
| 5 | TAR 2.0 (AI-Assisted Review) | 5-6 weeks | Phase 3 | A |
| 6 | Privilege Review & Logging | 4-5 weeks | Phase 3 | A |
| 7 | Production Generation | 4-5 weeks | Phase 3 | A |
| 8 | Communication Analytics (Graph) | 4-5 weeks | Phase 5 | A |
| 9 | Legal Hold & Custodian Management | 4-5 weeks | Phase 1 | B |
| 10 | Early Case Assessment | 3-4 weeks | Phase 9 | B |
| 11 | DSAR & GDPR Compliance | 3-4 weeks | Phase 1 | C |
| 12 | Cross-Matter Learning & Advanced AI | 5-6 weeks | Phase 5, Phase 11 | C |

**Total estimated duration:** 12-14 months (with Tracks A, B, C running in parallel after Phase 1)

**Minimum viable product (Phases 1-3, 7):** ~20-23 weeks — delivers document ingestion, search, review, and production, covering the core EDRM workflow that no open-source tool currently provides.

**Full platform (all phases):** ~50-60 weeks — delivers feature parity with mid-market commercial platforms (Logikcull, lower-tier Everlaw) plus OSS-unique capabilities (cross-matter learning, EDRM XML native support, self-hosted deployment).
