# Standards & API Reference

> Project: E-Discovery Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27050-1:2019 — Information Technology: Electronic Discovery — Part 1: Overview and Concepts**
- URL: https://www.iso.org/standard/78647.html
- Defines the terminology, concepts, and processes of electronic discovery (e-discovery). Provides the conceptual foundation for all other parts of the ISO 27050 series, mapping directly to the EDRM lifecycle. Essential reference for any platform claiming international standards compliance.

**ISO/IEC 27050-2 — Electronic Discovery — Part 2: Guidance for Governance and Management**
- URL: https://www.iso27001security.com/html/27050.html
- Addresses organisational governance of e-discovery programs, covering policies, roles, responsibilities, and management frameworks for managing electronically stored information (ESI) through the discovery lifecycle.

**ISO/IEC 27050-3:2020 — Electronic Discovery — Part 3: Code of Practice**
- URL: https://www.iso.org/standard/78648.html
- Provides requirements and recommendations for the full ESI lifecycle: identification, preservation, collection, processing, review, analysis, and production. The most operationally relevant part for platform feature design and workflow validation.

**ISO/IEC 27050-4:2021 — Electronic Discovery — Part 4: Technical Readiness**
- URL: https://cdn.standards.iteh.ai/samples/74034/fedba3d25b1f4d69b32f02da46415907/ISO-IEC-27050-4-2021.pdf
- Guides organisations in planning, preparing for, and implementing e-discovery from both technology and process perspectives. Covers infrastructure, tooling, and readiness assessment criteria.

**ISO/IEC 27037:2012 — Guidelines for Identification, Collection, Acquisition, and Preservation of Digital Evidence**
- URL: https://www.iso.org/standard/44381.html
- Establishes guidelines for handling digital evidence in the initial forensic lifecycle (identification, collection, acquisition, preservation). Directly relevant to custodian data collection, forensic imaging, and chain-of-custody workflows within an e-discovery platform.

**ISO/IEC 27041 — Assurance for Digital Forensics**
- URL: https://www.iso27001security.com/html/27050.html
- Provides guidance on assurance aspects of digital forensic methods, ensuring that tools and procedures are fit for purpose. Relevant for technology-assisted review (TAR) validation and defensibility requirements.

**ISO/IEC 27042 — Analysis and Interpretation of Digital Evidence**
- URL: https://www.iso27001security.com/html/27050.html
- Covers the analysis and interpretation phases of digital forensics — the steps that follow collection and preservation. Relevant for AI-assisted document review, analytics, and near-duplicate detection features.

**ISO/IEC 27043 — Incident Investigation Principles and Processes**
- URL: https://www.iso27001security.com/html/27050.html
- Covers the broader incident investigation context within which forensic and e-discovery activities typically occur. Relevant for internal investigations and regulatory response workflows.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The baseline information security management standard. E-discovery platforms handling sensitive legal data are expected to be ISO 27001 certified, ensuring security controls, audit logging, access management, and breach response procedures are in place.

---

### W3C & IETF Standards

**RFC 5322 — Internet Message Format**
- URL: https://datatracker.ietf.org/doc/html/rfc5322
- Defines the syntax for email messages, including header fields (From, To, Date, Subject), message body structure, and CRLF line endings. Foundational for email ingestion, parsing, and threading in e-discovery platforms. Replaced RFC 2822.

**RFC 2045–2049 — MIME (Multipurpose Internet Mail Extensions)**
- URL: https://datatracker.ietf.org/doc/html/rfc2045
- Defines the structure and encoding of multi-part email messages and attachments. Essential for correctly ingesting email with attachments (PDFs, Office documents, images) during e-discovery collection.

**RFC 3156 — MIME Security with OpenPGP**
- URL: https://datatracker.ietf.org/doc/html/rfc3156
- Describes how OpenPGP encryption and signatures integrate with MIME. Relevant for handling encrypted email in e-discovery collection and processing workflows.

**RFC 4155 — The application/mbox Media Type**
- URL: https://datatracker.ietf.org/doc/html/rfc4155
- Defines the mbox file format for storing collections of email messages. Widely used as an export and ingestion format for email archives in e-discovery.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the Link header for establishing typed relationships between web resources. Relevant for building paginated REST APIs with navigable document collections.

**W3C PROV-DM — Provenance Data Model**
- URL: https://www.w3.org/TR/prov-dm/
- Defines a data model for recording data provenance (who created what, when, and how). Conceptually equivalent to legal chain of custody documentation; applicable to audit trail and evidence integrity logging in an e-discovery platform.

---

### Data Model & File Format Specifications

**EDRM XML Interchange Format**
- URL: https://edrm.net/resources/frameworks-and-standards/edrm-xml/
- The EDRM XML schema is the industry-standard, vendor-neutral format for moving ESI between e-discovery processing stages, software tools, and organisations. Provides a standard load file structure covering documents, metadata, and relationships. Released to the public domain in 2006; v1.1 released 2009.

**EDRM Production Standards (v2)**
- URL: https://edrm.net/resources/frameworks-and-standards/edrm-model/edrm-stages-standards/edrm-production-standards-version-2/
- Defines best practices for the production of ESI in litigation, covering file formats (TIFF, PDF, native), load files (DAT, OPT), and metadata requirements. The de facto standard for what law firms and courts expect in an e-discovery production.

**OpenAPI Specification 3.1 (OAS)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The standard for describing RESTful HTTP APIs. Any publicly exposed e-discovery platform API should publish an OAS document, enabling SDK generation, automated testing, and developer tooling.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used for validating request and response payloads in REST APIs. Underpins OpenAPI 3.1 data type definitions. Relevant for ESI metadata schemas and API contract validation.

**PST (Personal Storage Table) Format**
- URL: https://docs.microsoft.com/en-us/openspecs/office_file_formats/ms-pst
- Microsoft's proprietary but publicly documented format for storing Outlook email, calendar, and contact data. One of the most common source data formats in e-discovery; platforms must parse PST/OST files reliably.

**PDF/A (ISO 19005) — Archival PDF**
- URL: https://www.iso.org/standard/38920.html
- The ISO standard for long-term archival of PDF documents. E-discovery productions frequently require PDF/A format for image productions to ensure long-term readability and courtroom admissibility.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry-standard authorisation framework for delegated API access. E-discovery platforms and their integration APIs (e.g., Relativity One, Everlaw) use OAuth 2.0 client credentials and bearer token flows for programmatic access.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Identity layer built on top of OAuth 2.0. Used for SSO (Single Sign-On) integration with enterprise identity providers (Okta, Azure AD, PingIdentity) — a mandatory feature for enterprise e-discovery deployments.

**NIST SP 800-86 — Guide to Integrating Forensic Techniques into Incident Response**
- URL: https://csrc.nist.gov/pubs/sp/800/86/final
- Provides a four-step digital forensics process: identify, acquire and protect data; process collected data; analyse extracted data; report results. Widely referenced by government and enterprise e-discovery practitioners in the US; shapes platform workflow design.

**NIST SP 800-53 — Security and Privacy Controls for Information Systems**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The comprehensive US federal security control framework. Platforms serving government agencies (FOIA, government litigation) must demonstrate NIST 800-53 compliance. Covers access control, audit logging, incident response, and system integrity.

**SHA-256 Hash Standard (FIPS PUB 180-4)**
- URL: https://csrc.nist.gov/publications/detail/fips/180/4/final
- SHA-256 is the current industry and legal standard for cryptographic hash verification of digital evidence. Hash values are recorded at collection and recalculated at every transfer to prove data integrity. Courts, law enforcement agencies, and ISO/IEC 27037 all recognise SHA-256 as the required forensic integrity algorithm.

**OWASP Application Security Verification Standard (ASVS)**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Defines security requirements for web application development. Applicable to e-discovery platforms to ensure input validation, authentication, session management, and data protection controls meet enterprise security expectations.

**SOC 2 Type II**
- URL: https://www.aicpa.org/resources/landing/system-and-organization-controls-soc-suite-of-services
- The de facto enterprise cloud security attestation standard (not an ISO but a AICPA framework). Everlaw, Relativity One, and most SaaS e-discovery vendors publish SOC 2 Type II reports. Required by enterprise buyers and law firms with data handling obligations.

---

### Regulatory Frameworks

**Federal Rules of Civil Procedure (FRCP) — Rules 26, 34, 37**
- URL: https://www.law.cornell.edu/rules/frcp
- The US procedural rules governing ESI discovery obligations. Rule 26 defines scope, proportionality, and meet-and-confer requirements for ESI. Rule 34 governs production formats. Rule 37 defines sanctions for spoliation. These rules drive the functional requirements for legal hold, preservation, and production workflows in any US-market e-discovery platform.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- The EU's comprehensive privacy regulation. Article 17 (Right to Erasure) creates direct tension with e-discovery preservation obligations, but includes exceptions for legal proceedings (Art. 17(3)(e)). Platforms must provide EU data residency, cross-border transfer controls (SCCs), and DSAR workflow support.

**SEC Rule 17a-4 / FINRA Recordkeeping Requirements**
- URL: https://www.finra.org/rules-guidance/guidance/interpretations-financial-operational-rules/sea-rule-17a-4-and-related-interpretations
- US financial services sector record retention requirements mandating WORM (Write Once Read Many) storage, immediate accessibility for two years, and six-year total retention for regulated communications. E-discovery platforms in finserv must support compliant archive ingestion.

**HIPAA Security Rule (45 CFR Part 164)**
- URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- Governs safeguarding of electronic protected health information (ePHI). E-discovery matters in healthcare require platforms that can handle ePHI with appropriate access controls, audit logging, and Business Associate Agreement (BAA) coverage.

**CJIS Security Policy (FBI)**
- URL: https://www.fbi.gov/services/cjis
- Governs access to Criminal Justice Information Systems data. Law enforcement and government e-discovery platforms (criminal matters, FOIA requests) must satisfy CJIS authentication (MFA), encryption, and personnel vetting requirements.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — 2025-11-25 Specification**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- The open protocol (donated to the Linux Foundation in December 2025) that enables AI agents to access structured data from external systems. Highly relevant to an AI-native e-discovery platform: an MCP server could expose case documents, custodian data, legal hold status, and production metadata to LLM agents performing review, summarisation, and privilege analysis tasks.

---

## Similar Products — Developer Documentation & APIs

### Relativity / RelativityOne

- **Description:** The dominant enterprise e-discovery platform with 60+ integration APIs covering the full EDRM lifecycle — legal hold, collection, processing, review, production, and analytics.
- **API Documentation:** https://platform.relativity.com/RelativityOne/Content/REST_API/REST_API.htm
- **Developer Portal:** https://platform.relativity.com/
- **OpenAPI Spec Files:** Available for download from the developer portal for all REST API endpoints.
- **SDKs/Libraries:** .NET SDK via NuGet (official); community SDKs for Python and Java exist on GitHub (https://github.com/relativitydev).
- **Developer Guide:** https://platform.relativity.com/RelativityOne/Content/Getting_Started/Basic_REST_API_concepts.htm
- **Key APIs:** Object Manager (CRUD on documents and custom objects), Import Service (bulk ingestion), Export Service, Production Manager, Analytics APIs (conceptual indexing, structured analytics), Imaging, Legal Hold, Audit.
- **Standards:** REST/JSON, OpenAPI 3.x, .NET SOAP (legacy); NuGet packages.
- **Authentication:** OAuth 2.0 bearer tokens via Relativity Identity service; client credentials grant; also supports API key and Negotiated authentication for on-premises deployments.

---

### Everlaw

- **Description:** Cloud-native e-discovery platform with REST API covering document ingestion, legal hold management, search, user administration, and production workflows.
- **API Documentation:** https://api.everlaw.com/docs/
- **Support Article:** https://support.everlaw.com/hc/en-us/articles/360053498791-Organization-Admin-Everlaw-API
- **SDKs/Libraries:** No official SDK; standard REST client libraries (requests in Python, axios in Node.js, etc.).
- **Developer Guide:** https://www.everlaw.com/blog/2021/01/04/introducing-the-everlaw-api-2/
- **Key Endpoints:** Database/project listing, document upload via datasets, search, custodian management, legal hold operations, production workflows, user and permission management, billing, event logs.
- **Standards:** REST/JSON; standard HTTP response codes; resource-oriented URL design.
- **Authentication:** API key via Bearer token header (`Authorization: Bearer everlaw-api.XXXX.YYYYYYYY`). Keys are scoped with fine-grained permissions. Keys auto-expire after 90 days of inactivity. Rate limit: 25 requests/second per account.

---

### Nuix

- **Description:** Data investigation, forensic processing, and e-discovery platform with native Java/JRuby/Jython APIs and a REST SDK for automation at scale.
- **API Documentation:** https://developer.nuix.com/latest/
- **Developer Program:** https://www.nuix.com/nuix-developer-program
- **GitHub (Discover extension SDK):** https://github.com/Nuix/discover-extension-sdk
- **SDKs/Libraries:** Nuix Core Engine REST SDK; Java API; PowerShell and Python guides; UI Extension SDK for Nuix Discover.
- **Developer Guide:** https://developer.nuix.com/ (Nuix Workbench scripting, ECC REST API quick-start, Discover Connect Graph API).
- **Key APIs:** Core Engine REST SDK (case creation, evidence ingestion, processing, analytics), Discover Connect Graph API (querying structured data in Discover), Enterprise Collection Center (ECC) REST API (endpoint collection management).
- **Standards:** REST/JSON; Java API; native JRuby/Jython bindings.
- **Authentication:** Basic Authentication and Bearer token authentication for ECC REST API; Core Engine uses Nuix-specific credentials and licensing.

---

### Reveal (formerly Brainspace)

- **Description:** AI-powered e-discovery analytics and review platform with a REST API and connector framework for integrating with Relativity and other platforms.
- **API Documentation:** https://docs.revealdata.com/brainspace
- **Connector Guide:** https://docs.revealdata.com/brainspace/docs/reveal-brainspace-connector-guide
- **Developer Documentation Portal:** https://www.revealdata.com/documentation
- **SDKs/Libraries:** Reveal-Brainspace Connector (Java-based plug-in); direct REST API integration for Reveal Review and Brainspace workflows.
- **Developer Guide:** The connector guide at docs.revealdata.com covers setting up integrations with Relativity, Reveal Review, and other review platforms.
- **Key APIs:** Brainspace REST API (AI analytics, clustering, concept search); Reveal Review REST API (document metadata retrieval, workflow actions); connector framework for data import/export.
- **Standards:** REST/JSON; connector plug-in architecture.
- **Authentication:** Credential-based authentication via the connector configuration; specific token/key mechanism documented in the connector guide.

---

### Exterro (FTK / Legal Hold)

- **Description:** Legal governance, risk, and compliance platform with FTK (Forensic Toolkit) for digital forensics and FTK Connect for workflow automation and integration.
- **API Documentation:** https://www.exterro.com/digital-forensics-software/ftk-connect
- **SDKs/Libraries:** FTK Connect API (REST-based integration with ServiceNow, Jira, and enterprise systems); 190+ native connectors in FTK Central for data source ingestion.
- **Developer Guide:** https://www.exterro.com/digital-forensics-software
- **Key Capabilities:** FTK Connect APIs for triggering forensic workflows from ITSM platforms; enterprise data source connectors (email, cloud storage, mobile, collaboration tools); Exterro Legal Hold REST integrations.
- **Standards:** REST/JSON; enterprise integration via FTK Connect.
- **Authentication:** API key / enterprise SSO (specific authentication details require vendor engagement).

---

### Logikcull

- **Description:** Self-service e-discovery platform targeting mid-market law firms and corporate teams. Point-and-click integrations with cloud storage providers; no public REST API as of 2026.
- **Integration Documentation:** https://support.logikcull.com/en/collections/4078908-integrations
- **SDKs/Libraries:** None (no developer API available for programmatic access).
- **Integration Points:** Native connectors to Slack, Microsoft 365, Google Workspace, Box, Dropbox, and Google Vault; drag-and-drop upload.
- **Standards:** No public API standard; integration via native cloud connector configurations.
- **Authentication:** N/A (no API); cloud connector authentication via OAuth to respective platforms (Microsoft, Google, Box).

---

### Casepoint / OPEXUS

- **Description:** AI-powered e-discovery and FOIA platform (merged with OPEXUS in 2025, backed by Thoma Bravo). Serves corporate and government markets with litigation, investigation, and data compliance workflows.
- **Website:** https://www.casepoint.com/
- **API Documentation:** Not publicly accessible as of May 2026; available through enterprise contracts and the vendor portal.
- **SDKs/Libraries:** Not publicly documented.
- **Key Capabilities:** AI-assisted review, legal hold, FOIA response management, government data discovery, compliance workflows.
- **Standards:** REST/JSON (internal); SOC 2 Type II certified.
- **Authentication:** Enterprise SSO / SAML; API key access available through the vendor portal under enterprise agreements.

---

## Notes

**Emerging Standard — MCP for Legal AI Agents:** The Model Context Protocol is rapidly becoming relevant to e-discovery. As AI agents are increasingly used for privilege review, issue coding, and deposition preparation, MCP servers that expose case data to LLMs represent a new integration surface. Legal practitioners are actively debating the discoverability of MCP interaction logs (FRCP 26(b)(1) may require their production). Platforms building MCP server endpoints should design with audit-trail completeness from the outset.

**EDRM XML Adoption Gap:** Despite EDRM XML existing since 2008, proprietary load file formats (Relativity DAT/OPT, Concordance) remain dominant in actual productions. A true open-standard, interoperable e-discovery platform should treat EDRM XML as a first-class import/export format to reduce vendor lock-in.

**SHA-256 as the Forensic Hash Standard:** MD5 and SHA-1 are no longer considered collision-resistant and should not be used as the primary integrity hash in new platform implementations. SHA-256 (FIPS 180-4) is the current court-accepted, NIST-endorsed standard and is required by ISO/IEC 27037.

**API Maturity Landscape:** Relativity One and Everlaw offer the most mature, publicly documented APIs in the market. Nuix has a strong developer program for its Core Engine. Logikcull and Casepoint/OPEXUS have limited or no public APIs, representing a market gap that an open-source platform with a first-class REST API and MCP server could exploit.
