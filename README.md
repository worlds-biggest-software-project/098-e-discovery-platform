# E-Discovery Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native e-discovery platform built to the EDRM framework — covering legal hold, collection, processing, review, privilege logging, and production.

E-Discovery Platform is a community-built alternative to the proprietary tools (Relativity, Everlaw, DISCO, Exterro) that dominate litigation technology. It targets government agencies, law schools, legal aid organisations, and small-to-mid-size firms that cannot justify $20K–$500K/year commercial platforms, while bringing modern AI capabilities — privilege review, proportionality analytics, and custodian interview automation — to the EDRM workflow.

---

## Why E-Discovery Platform?

- **No substantive open-source option exists.** Apache Tika provides document extraction, but no community project has ever built out the full EDRM workflow despite EDRM being an open, freely implementable framework.
- **Commercial pricing locks out most of the legal market.** Per-GB all-in costs run $48–$180/GB; enterprise platforms reach $50K–$500K+/year. Mid-size firms, legal aid, and government bodies are systematically priced out.
- **Privilege review is the dominant cost driver.** Privilege determinations consume 60–70% of total review cost. Incumbents assist with relevance review but do not generate FRE 502-compliant privilege logs end-to-end.
- **Proportionality analysis is unsupported end-to-end.** FRCP Rule 26 requires proportionality assessment before discovery commences, yet no current platform produces a defensible proportionality memo from the data universe.
- **Cloud lock-in raises sovereignty concerns.** Cloud-only vendors (Everlaw, DISCO) cannot serve customers with on-premises or sovereign-data requirements; on-prem incumbents (Relativity, Nuix) carry steep licensing and administration costs.

---

## Key Features

### EDRM Workflow Coverage

- Document ingestion pipeline using Apache Tika for extraction across 1,000+ formats
- Automated deduplication (exact and near-duplicate), OCR, and metadata preservation
- Full-text search with boolean, proximity, date, and metadata filtering
- Production in court-accepted formats (TIFF + OCR, native, PDF) with Concordance DAT and Opticon OPT load files
- Complete audit trail of review decisions for defensibility under FRCP Rules 26, 34, and 37

### Legal Hold & Custodian Management

- Notice creation, distribution, acknowledgment tracking, and automated escalation
- Custodian interview workflow that maps relevant data systems before collection
- Integration points for IT asset inventory and enterprise GRC systems
- DSAR (Data Subject Access Request) integration for organisations managing both litigation and privacy obligations

### Document Review Workspace

- Configurable coding fields, batch assignment, and concurrent multi-reviewer access
- Email thread visualisation and near-duplicate clustering to reduce redundant review
- Role-based access control across reviewer, supervising attorney, case manager, and admin roles
- Privilege log generation with standard fields (Bates range, author, recipient, date, privilege basis)

### AI-Assisted Review

- TAR 2.0 / Continuous Active Learning for document prioritisation and review scope reduction, based on the public-domain CAL algorithm (Cormack and Grossman, 2014)
- AI privilege pre-flagging using ML classification of attorney-client and work product communications
- Early case assessment dashboard with document universe statistics, custodian volumes, and projected review burden
- Cross-matter learning with confidentiality partitioning (backlog)

### Defensibility & Compliance

- Workflow aligned to the EDRM stages (Information Governance → Production)
- ISO/IEC 27050 alignment for international e-discovery processes
- FRE 502-aware privilege workflow with attorney-in-the-loop confirmation
- GDPR Article 17 / data minimisation handling for EU-sourced data

---

## AI-Native Advantage

Incumbents bolted AI onto legacy review platforms; this project designs around it. AI-native privilege review generates FRE 502-compliant privilege logs with Bates references, author/recipient mapping, privilege basis, and redaction recommendations — a workflow no current platform handles end-to-end. A proportionality analytics module ingests the data universe and produces a defensible FRCP Rule 26 memo. Custodian interview automation conducts structured interviews, maps data systems, and issues tailored hold notices, compressing a multi-week process to days. All AI outputs remain attorney-supervised in line with ABA Formal Opinion 512 and state bar guidance.

---

## Tech Stack & Deployment

- **Document extraction:** Apache Tika (Apache Licence 2.0) as the processing foundation
- **Search:** Elasticsearch / Apache Solr / Lucene compatible
- **Standards alignment:** EDRM framework, ISO/IEC 27050, FRCP Rules 26/34/37, FRE 502
- **Production formats:** TIFF + OCR, native, PDF with Concordance and Opticon load files
- **Deployment modes:** self-hosted (on-prem and private cloud) and managed cloud, addressing the data sovereignty gap left by cloud-only incumbents
- **Integrations:** Office 365, Google Workspace, Slack, Zoom for defensible cloud collection; SFTP and REST API for custom data sources

---

## Market Context

The global e-discovery market is **$18.73B in 2025**, projected to reach **$46.06B by 2034** at a 10.49% CAGR (Fortune Business Insights); cloud accounts for 76.65% of share in 2026. Commercial pricing ranges from $250/month (Logikcull) through $2K–$10K/month (Everlaw, DISCO) to $20K–$500K+/year (Relativity, Nuix, Exterro), with industry per-GB all-in costs of $48–$180. Primary buyers span AmLaw 200 litigation partners, Fortune 1000 in-house legal, mid-size law firms, government agencies (DOJ, SEC, state AGs), and solo/boutique litigators — with the lower-budget cohort largely unserved by current options.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
