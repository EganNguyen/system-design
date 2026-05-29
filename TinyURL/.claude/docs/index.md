# TinyURL — Documentation Index

> URL shortening service: shorten long URLs into compact short links, redirect, and track analytics.
> Design target: 100M URLs stored, 1B redirects/day, p99 redirect < 10 ms.

---

## Legend

| Symbol | Meaning |
|---|---|
| ⭐ | Never skipped at big tech (Google, Meta, Amazon, Apple, Microsoft) |
| 🔴 | Must exist before a single line of code |
| 🟡 | Should exist before implementation; can be refined in parallel |
| 🟢 | Produced after or during implementation |
| ✅ | Written |
| ⚠️ | Partial — exists but incomplete |
| ❌ | Missing |

---

## Phase 1 — Before Implementation

| # | Document | File | Priority | Status | Skip Risk |
|---|---|---|---|---|---|
| 1 | Product Requirements (PRD) | [PRD.md](PRD.md) | ⭐ 🔴 | ✅ Complete | Critical |
| 2 | Architecture Decision Records (ADRs) | [ADRs.md](ADRs.md) | ⭐ 🔴 | ✅ Complete | Critical |
| 3 | System Design (HLD) | [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) | ⭐ 🔴 | ✅ Complete | Critical |
| 4 | API Contract / Versioning Spec | [API_CONTRACT.md](API_CONTRACT.md) | ⭐ 🔴 | ✅ Complete | Critical |
| 5 | Database Design | [DATABASE_DESIGN.md](DATABASE_DESIGN.md) | ⭐ 🔴 | ✅ Complete | Critical |
| 6 | Security Threat Model | [SECURITY.md](SECURITY.md) | ⭐ 🔴 | ✅ Complete | Critical |
| 7 | Low-Level Design (LLD) | [LLD.md](LLD.md) | 🟡 | ✅ Complete | High |
| 8 | Sequence Diagrams | [SEQUENCES.md](SEQUENCES.md) | 🟡 | ✅ Complete | High |
| 9 | UI/UX Design | [UIUX_DESIGN.md](UIUX_DESIGN.md) | 🟡 | ✅ Complete | High |
| 10 | Data Privacy & Compliance | [PRIVACY.md](PRIVACY.md) | 🔴 | ✅ Complete | Critical (EU/CA users) |
| 11 | Dependency & Third-Party Inventory | — | 🟡 | ❌ Missing | High |

---

## Phase 2 — During / Parallel to Implementation

| # | Document | File | Priority | Status | Skip Risk |
|---|---|---|---|---|---|
| 12 | Cost Model | [COST_MODEL.md](COST_MODEL.md) | 🟡 | ✅ Complete | High |
| 13 | Testing Strategy | [TESTING.md](TESTING.md) | 🟡 | ✅ Complete | High |
| 14 | Observability & Error Handling Design | — | 🟡 | ❌ Missing | High |

---

## Phase 3 — Before Launch

| # | Document | File | Priority | Status | Skip Risk |
|---|---|---|---|---|---|
| 15 | Operational Runbook | — | ⭐ 🔴 | ❌ Missing | Critical |
| 16 | Launch / Rollout Plan | — | ⭐ 🔴 | ❌ Missing | Critical |
| 17 | Data Migration Plan | — | 🟡 | ⚠️ Partial (DATABASE_DESIGN §9) | High if schema evolves |
| 18 | Disaster Recovery & BCP | — | 🟡 | ❌ Missing | High |

---

## Progress

```
Phase 1 — Before Implementation   10 / 11 complete   █████████░  91%
Phase 2 — During Implementation    2 / 3  complete   ██████████  67%
Phase 3 — Before Launch            0 / 4  complete   ░░░░░░░░░░   0%

Overall                           12 / 18 complete   ███████░░░  67%
```

---

## Document Summaries

### Phase 1

**[PRD.md](PRD.md)**
Problem, users, hypothesis, success metrics (50k RPS / p99 < 10ms / < 15% SaaS cost),
MVP scope (redirect engine + shorten API + async analytics), delivery milestones,
3 open questions, and 5 risk items.

**[ADRs.md](ADRs.md)**
10 decisions documented: counter + Base62 key generation (ADR-001), 302 vs 301 redirect
(ADR-002), URI path versioning (ADR-003), PostgreSQL with Cassandra migration path
(ADR-004), Redis Cluster (ADR-005), Kafka for async analytics (ADR-006), Cloudflare over
CloudFront — 62× cost saving (ADR-007), ClickHouse for analytics (ADR-008), Go as backend
language (ADR-009), soft delete over hard delete (ADR-010).

**[SYSTEM_DESIGN.md](SYSTEM_DESIGN.md)**
High-level architecture. Problem statement, capacity estimation (1B redirects/day,
100M URLs, 50k RPS peak), component topology (CDN → LB → Read/Write Service → Redis →
DB → Kafka → ClickHouse), Base62 encoding strategy, failure modes, monitoring, and
technology stack summary.

**[API_CONTRACT.md](API_CONTRACT.md)**
Formal API contract. URI versioning strategy (`/api/v1/`), JWT auth (RS256, 15min TTL),
endpoint access matrix, standard error envelope, rate-limit headers, all 6 endpoints
with full request/response schemas, breaking vs non-breaking change definitions,
6-month deprecation policy, and v1.0 changelog.

**[DATABASE_DESIGN.md](DATABASE_DESIGN.md)**
Schema and access patterns. Table definitions (`users`, `urls`, `click_events`,
`click_daily_mv`, `blacklist`, `audit_log`), indexing strategy, ClickHouse columnar
schema, replication topology, query examples, retention schedule (90-day click events),
and migration tooling (`golang-migrate`).

**[SECURITY.md](SECURITY.md)**
STRIDE threat model across 7 entry points. 15 threats analysed (A1–G1), covering
phishing via redirect, SSRF, short-code enumeration, IDOR, JWT theft, analytics PII
exposure, cache stampede abuse, admin endpoint exposure. 17-control summary matrix.
5 open security gaps requiring action before public traffic.

**[LLD.md](LLD.md)**
Component internals. Data models, API shapes, Go redirect handler implementation,
Base62 encoding algorithms (counter + hash + UUID options), Redis cache schema,
rate limiting (Redis Lua token bucket), URL validation pipeline, expiry/cleanup approach,
failure modes, and flexible DB selection guide by scale tier.

**[SEQUENCES.md](SEQUENCES.md)**
7 Mermaid sequence diagrams: shorten flow (with collision retry), redirect cache hit,
redirect cache miss / expired URL, custom alias creation, URL expiry cleanup job,
analytics ingestion pipeline (Kafka → ClickHouse with IP hashing), and JWT
issue + refresh rotation.

**[UIUX_DESIGN.md](UIUX_DESIGN.md)**
UI/UX specification. Design philosophy (zero friction to first short link), design tokens
(colour, typography, spacing, motion), component library, screen flows (landing,
dashboard, analytics, settings), responsive breakpoints, and accessibility targets.

**[PRIVACY.md](PRIVACY.md)**
GDPR/CCPA compliance mapping. PII inventory, lawful basis per processing activity,
legitimate interest assessment for analytics, GDPR Chapter III rights + CCPA rights,
6-step right-to-erasure cascade, 90-day retention schedule (resolved conflict with
DATABASE_DESIGN), data residency requirements, third-party DPA checklist, privacy notice
requirements, and 72-hour breach response timeline.

**Dependency Inventory — ❌ to write**
All third-party services (Safe Browsing API, MaxMind GeoIP, AWS, Cloudflare, MSK,
monitoring vendor), license types, SLA/uptime guarantees, fallback plan if unavailable,
unit cost at scale, and DPA/data-sharing agreements per vendor.

---

### Phase 2

**[COST_MODEL.md](COST_MODEL.md)**
Bottom-up AWS cost estimate at target scale. Bill of materials (11 line items), total
$3,271/mo on 1-yr reserved pricing. Sensitivity table at 10%/50%/100%/200% traffic.
SaaS comparison: 6.5% of Bitly Enterprise at $50k baseline (PRD target: < 15%). Unit
economics: $0.109 per 1M redirects vs $1.67–3.33 on SaaS. Spot instance strategy and
5 open cost risks.

**[TESTING.md](TESTING.md)**
Integration-first test strategy for Go + Next.js stack. 5 layers: unit (pure functions,
≥90% coverage), integration (testcontainers-go + real PostgreSQL/Redis, scenario tables
per endpoint), contract (API_CONTRACT.md shape verification), E2E (4 Playwright journeys),
load (k6 scenarios validating all 6 PRD success metrics). CI pipeline placement and
fail-gate policy.

**Observability & Error Handling Design — ❌ to write**
Structured logging standard (JSON fields, levels, required keys), RED metrics per service
(Rate, Errors, Duration), distributed tracing (OpenTelemetry propagation, sampling rate),
alerting runbook (alert → severity → owner → escalation), SLO/SLI definitions and error
budget policy, and required dashboards before launch.

---

### Phase 3

**Operational Runbook — ❌ to write**
Service overview, deployment procedure with rollback steps, common failure playbooks
(high redirect latency, cache stampede, DB primary failover, Kafka consumer lag, analytics
disk full, abuse surge), escalation path (L1→L2→L3→vendor), RTO/RPO targets,
and DR/failover procedure.

**Launch / Rollout Plan — ❌ to write**
Feature flag / canary strategy (0%→1%→10%→100%), go/no-go criteria with measurable
thresholds, rollback trigger and decision owner, internal + external comms plan, launch
day runbook (on-call roster, war room, escalation), and 72-hour post-launch monitoring
period definition.

**Data Migration Plan — ⚠️ partial (DATABASE_DESIGN §9)**
Migration tooling (`golang-migrate`) and file structure covered. Missing: zero-downtime
expand/contract pattern, backfill strategy for schema changes, row-count reconciliation
plan, and rollback procedure for failed migrations.

**Disaster Recovery & BCP — ❌ to write**
RTO/RPO targets per service tier, backup strategy (frequency, retention, geo-redundancy),
automated vs manual failover procedures, DR test schedule (tabletop + live drill),
data integrity checks post-recovery, and communication templates for outage scenarios.

---

## Remaining Write Order

Priority order for the 7 outstanding documents:

| Priority | Document | Reason |
|---|---|---|
| 1 | **ADRs** | Most regretted skip; decisions already made, just need documenting |
| 2 | **Observability Design** | Required before any code ships — you can't fix what you can't see |
| 3 | **Operational Runbook** | Hard gate before launch; 3am on-call depends on this |
| 4 | **Launch / Rollout Plan** | Hard gate before public traffic |
| 5 | **Disaster Recovery & BCP** | Required for any enterprise SLA commitment |
| 6 | **Dependency Inventory** | Unblocks legal (DPA checklist) and vendor risk review |
| 7 | **Data Migration Plan** | Required before first schema change in production |
