# System Design Documentation Checklist
> What big tech teams produce before and after shipping — and why each doc exists.

---

## Legend

| Symbol | Meaning |
|--------|---------|
| 🔴 | Must exist before a single line of code |
| 🟡 | Should exist before implementation, can be refined in parallel |
| 🟢 | Produced after or during implementation |
| ⭐ | Never skipped at big tech (Google, Meta, Amazon, Apple, Microsoft) |

---

## Phase 1 — Before Implementation

### ⭐ 1. Product Requirements Document (PRD)
**Owner:** Product Manager  
**Purpose:** Define the *what* and *why*. No implementation should start without this.

**Must contain:**
- Problem statement & opportunity sizing
- User personas and jobs-to-be-done
- Functional requirements (features)
- Non-functional requirements (scale, latency, reliability targets)
- Out-of-scope (explicitly stated)
- Success metrics / KPIs
- Launch timeline

**Big tech signal:** At Amazon, this is the "PR/FAQ" (press release written before the product exists). At Google, it's the "PRD + Design Doc." No eng kickoff without it.

---

### ⭐ 2. Architecture Decision Records (ADRs)
**Owner:** Tech Lead / Architect  
**Purpose:** Document *why* decisions were made, not just what. The most regretted skip.

**Must contain:**
- Context (what problem, what constraints)
- Decision (what was chosen)
- Alternatives considered (and why rejected)
- Consequences (trade-offs accepted)
- Status: Proposed → Accepted → Deprecated

**Format:** One ADR per significant decision. Short (1–2 pages max). Stored in repo `/docs/adr/`.

**Big tech signal:** Amazon and Spotify use "RFCs." Google uses design docs with explicit alternatives sections. Uber calls them "Architecture Proposals."

---

### ⭐ 3. System Design Document (High-Level Design / HLD)
**Owner:** Senior Engineer / Architect  
**Purpose:** Define components, their responsibilities, and how they connect.

**Must contain:**
- System context diagram (external actors, system boundaries)
- Component diagram (services, queues, caches, CDN, etc.)
- Data flow overview
- Technology choices with rationale
- CAP theorem position (consistency vs. availability trade-offs)
- Scalability strategy (horizontal/vertical, sharding, partitioning)
- Failure modes and mitigation

---

### ⭐ 4. API Contract / Versioning Spec
**Owner:** Backend Engineer / API Platform Team  
**Purpose:** The single source of truth for all consumers (frontend, mobile, partners). Agreed before any build starts.

**Must contain:**
- OpenAPI / AsyncAPI spec (machine-readable)
- Versioning strategy (`/v1/`, header-based, etc.)
- Authentication & authorization model
- Request/response schemas with examples
- Error codes and standard error envelope
- Rate limiting and throttling policy
- Deprecation policy and sunset timeline
- Breaking vs. non-breaking change definitions

**Big tech signal:** Stripe's API contract discipline is the gold standard. Amazon enforces "API-first" — the contract ships before the service.

---

### ⭐ 5. Database Design
**Owner:** Backend Engineer / Data Engineer  
**Purpose:** Schema is the hardest thing to change in production. Design it right once.

**Must contain:**
- ER diagram (entities, relationships, cardinality)
- Table / collection definitions (fields, types, constraints, indexes)
- Primary key strategy (UUID, auto-increment, composite)
- Index strategy (query patterns drive this)
- Partitioning / sharding plan (if applicable)
- Data retention policy per entity
- Migration strategy (zero-downtime approach)
- Backup and restore approach

---

### ⭐ 6. Security Threat Model
**Owner:** Security Engineer / Tech Lead  
**Purpose:** Identify attack surfaces before they are built. Retrofit is 10× more expensive.

**Must contain:**
- Trust boundaries diagram (what crosses a boundary = a threat)
- STRIDE analysis (Spoofing, Tampering, Repudiation, Info Disclosure, DoS, Elevation of Privilege)
- Authentication design (OAuth2, OIDC, API keys, mTLS, etc.)
- Authorization model (RBAC, ABAC, ACL)
- Secrets management plan (no secrets in code or env vars)
- Encryption at rest and in transit (cipher suites, key rotation)
- Dependency vulnerability plan (SBOM, CVE scanning)
- Top OWASP risks addressed (injection, broken auth, SSRF, etc.)

**Big tech signal:** Google's "Design Review" has a mandatory security section. Amazon requires a Security Review for any service touching customer data.

---

### 7. Low-Level Design (LLD)
**Owner:** Engineer implementing the component  
**Purpose:** Detailed design within a single service/module. Written per component.

**Must contain:**
- Class / module diagram
- Key algorithms and data structures
- Concurrency model (thread safety, locks, async patterns)
- Error handling strategy
- Memory and resource management
- Internal interfaces and contracts

---

### 8. Sequence Diagrams
**Owner:** Engineer  
**Purpose:** Show time-ordered interactions across components for critical flows.

**Cover at minimum:**
- Happy path (primary user journey)
- Authentication / authorization flow
- Error / retry flows
- Async / event-driven flows (publish → consume)
- Timeout and fallback paths

**Tool:** PlantUML, Mermaid, or Lucidchart stored in repo.

---

### 9. UI/UX Design
**Owner:** Product Designer  
**Purpose:** Visual and interaction specification engineers build against. Avoids "that's not what I meant" after coding.

**Must contain:**
- User flows (task completion paths)
- Wireframes → High-fidelity mockups (Figma)
- Component library / design system reference
- Responsive breakpoints
- Accessibility requirements (WCAG 2.1 AA minimum)
- Empty states, error states, loading states
- Copy / microcopy (not "TBD")

---

### 10. Data Privacy & Compliance Plan
**Owner:** Legal / Privacy Engineer  
**Purpose:** Regulatory requirements (GDPR, CCPA, HIPAA, SOC2) have teeth. Retrofitting is painful and expensive.

**Must contain:**
- PII inventory (what data, where stored, how long)
- Data classification (public, internal, confidential, restricted)
- Consent model and audit trail
- Data Subject Rights support (access, delete, export)
- Cross-border data transfer controls
- Applicable regulatory frameworks and controls mapped
- Privacy impact assessment (PIA) if required

---

### 11. Dependency & Third-Party Inventory
**Owner:** Tech Lead  
**Purpose:** Vendor lock-in, license risk, and supply-chain attacks surface here.

**Must contain:**
- All third-party services (SaaS APIs, SDKs, infrastructure)
- License type for each (MIT, GPL, proprietary — GPL has copy-left implications)
- SLA / uptime guarantee from each vendor
- Fallback plan if vendor is unavailable
- Cost at scale (unit economics per vendor)
- Data-sharing agreements in place (if vendor touches customer data)

---

## Phase 2 — During / Parallel to Implementation

### 12. Cost Model
**Owner:** Engineering + Finance  
**Purpose:** Infra costs that look small at prototype scale become budget-busting at prod scale.

**Must contain:**
- Unit cost breakdown (cost per request, per user, per GB)
- Scaling cost curve (what happens at 10×, 100× load)
- Top 3 cost drivers identified and optimized
- Reserved/spot/committed-use strategy
- Tagging and cost allocation to teams
- Budget alert thresholds

---

### 13. Testing Strategy
**Owner:** Engineering + QA  
**Purpose:** Define the test pyramid before writing tests, not after.

**Must contain:**
- Test pyramid targets (unit %, integration %, E2E %)
- Unit testing conventions and coverage threshold (e.g., 80%)
- Integration / contract testing approach (Pact, Postman, etc.)
- Performance / load testing plan (k6, Locust, Gatling)
- Chaos / resilience testing plan (failure injection)
- Security testing (SAST, DAST, pen test schedule)
- Test data management (synthetic data, anonymized prod data)
- Flaky test policy

---

### 14. Observability & Error Handling Design
**Owner:** Platform / SRE Engineer  
**Purpose:** You can't fix what you can't see. Define this before launch, not when production is on fire.

**Must contain:**
- Logging standard (structured JSON, required fields, log levels)
- Metrics (RED method: Rate, Errors, Duration — per service)
- Distributed tracing (trace propagation standard, sampling rate)
- Alerting runbook (alert → severity → owner → escalation path)
- SLOs defined (e.g., p99 latency < 500ms, error rate < 0.1%)
- SLIs and error budget policy
- Dashboards (what must exist before launch)

---

## Phase 3 — Before Launch (Post-Implementation)

### ⭐ 15. Operational Runbook
**Owner:** SRE / Engineering  
**Purpose:** The on-call engineer at 3am shouldn't have to figure this out under pressure.

**Must contain:**
- Service overview (what it does, who owns it, who to call)
- Architecture diagram (current, not aspirational)
- Deployment procedure (step-by-step, with rollback steps)
- Common failure modes and resolution steps
- Escalation path (L1 → L2 → L3 → vendor)
- Regular maintenance procedures
- Recovery time objective (RTO) and recovery point objective (RPO)
- DR / failover procedure

---

### ⭐ 16. Launch / Rollout Plan
**Owner:** Engineering + PM + SRE  
**Purpose:** Big bangs fail. Controlled rollouts catch issues before they become incidents.

**Must contain:**
- Feature flag / rollout strategy (0% → 1% → 10% → 100%)
- Canary / blue-green / shadow traffic plan
- Go/no-go criteria (specific, measurable thresholds)
- Rollback trigger and procedure (who decides, how fast)
- Communication plan (internal stakeholders, external users if applicable)
- Launch day runbook (who is on call, war room, escalation)
- Post-launch monitoring period (e.g., 72 hours heightened alert)

---

### 17. Data Migration Plan
**Owner:** Backend Engineer / Data Engineer  
**Purpose:** Applies any time existing data or schema evolves. Migrations are the #1 cause of outages at scale.

**Must contain:**
- Migration strategy (big bang vs. dual-write vs. online migration)
- Zero-downtime approach (expand/contract pattern)
- Backfill strategy and estimated duration
- Validation / reconciliation plan (row counts, checksums, spot checks)
- Rollback plan if migration corrupts data
- Cut-over procedure

---

### 18. Disaster Recovery & Business Continuity Plan (DR/BCP)
**Owner:** SRE / Infra  
**Purpose:** Not if, but when. Design for failure explicitly.

**Must contain:**
- RTO and RPO targets per service tier
- Backup strategy (frequency, retention, geo-redundancy)
- Failover procedure (automated vs. manual, tested)
- DR test schedule (tabletop + live failover drill)
- Data integrity checks post-recovery
- Communication templates for outages

---

## Summary Table

| # | Document | Phase | Owner | Skip Risk |
|---|----------|-------|-------|-----------|
| 1 | PRD | Before | PM | 🔴 Critical |
| 2 | ADRs | Before | Tech Lead | 🔴 Critical |
| 3 | System Design (HLD) | Before | Architect | 🔴 Critical |
| 4 | API Contract | Before | Backend Eng | 🔴 Critical |
| 5 | Database Design | Before | Backend Eng | 🔴 Critical |
| 6 | Security Threat Model | Before | Security Eng | 🔴 Critical |
| 7 | Low-Level Design (LLD) | Before | Engineer | 🟡 High |
| 8 | Sequence Diagrams | Before | Engineer | 🟡 High |
| 9 | UI/UX Design | Before | Designer | 🟡 High |
| 10 | Data Privacy & Compliance | Before | Privacy Eng | 🔴 Critical (regulated industries) |
| 11 | Dependency Inventory | Before | Tech Lead | 🟡 High |
| 12 | Cost Model | During | Eng + Finance | 🟡 High |
| 13 | Testing Strategy | During | Eng + QA | 🟡 High |
| 14 | Observability Design | During | SRE | 🟡 High |
| 15 | Operational Runbook | Before Launch | SRE | 🔴 Critical |
| 16 | Launch / Rollout Plan | Before Launch | PM + SRE | 🔴 Critical |
| 17 | Data Migration Plan | Before Launch | Backend Eng | 🟡 (if migration exists) |
| 18 | DR / BCP | Before Launch | SRE | 🟡 High |

---

## What Big Tech Actually Does

| Company | Pre-implementation doc | Notable practice |
|---------|----------------------|-----------------|
| **Amazon** | PR/FAQ + OP1 | "Work backwards" — write the press release first |
| **Google** | Design Doc (mandatory) | Peer review required; explicit alternatives section |
| **Meta** | ERD + Systems Review | SRE reviews capacity before any launch |
| **Stripe** | API contract first | API design reviewed by API council before coding |
| **Netflix** | Chaos Doc | DR/failure scenarios documented before build |
| **Uber** | Architecture Proposal (RFC) | ADR equivalent with mandatory security section |
| **Apple** | PRD + Legal review | Privacy review gates engineering start |

---

> **Rule of thumb:** Docs that are expensive to change (schema, API contract, security model) need maximum upfront rigor. Docs that evolve naturally (runbooks, test plans) can iterate. The cost of a missing doc grows exponentially after deployment.
