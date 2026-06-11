# System Design

> **Design before code.** Each case study here follows the same discipline that big-tech teams apply before writing a single line of production code: requirements, architecture decisions, API contracts, database schemas, security threat models, and more — all documented first.

---

## Philosophy

Most engineers learn system design through interviews. This project treats it as an engineering practice:

1. **Documentation is the deliverable** — docs are not an afterthought; they are Phase 1
2. **Justify every decision** — ADRs record what was chosen, what was rejected, and why
3. **Security by design** — threat models are completed before API design is finalized
4. **Measure everything** — success metrics are defined upfront with explicit measurement methods
5. **Understand the domain** — business context drives every technical trade-off

---

## Case Studies

| System | Domain | Status |
|---|---|---|
| [TinyURL](./TinyURL/) | URL shortening at enterprise scale | Phase 1 — 91% |
| [WhatsApp](./WhatsApp/) | Real-time messaging | In progress |
| [Instagram](./Instagram/) | Photo/video social platform | In progress |
| [Netflix](./Netflix/) | Video streaming at scale | In progress |
| [Uber](./Uber/) | Real-time ride matching | In progress |
| [Dropbox](./Dropbox/) | Distributed file storage | In progress |
| [Google Search](./Google%20Search/) | Web indexing & retrieval | In progress |
| [Pastebin](./Pastebin/) | Text snippet sharing | In progress |
| [Amazon](./Amazon/) | E-commerce platform | In progress |
| [Payment](./Payment/) | Payment processing | In progress |
| [Notification](./Notification/) | Push/SMS/email delivery | In progress |
| [Rate Limiter](./Rate%20Limiter/) | Distributed rate limiting | In progress |
| [Recommendation System](./Recommendation%20System/) | Personalization engine | In progress |
| [Search Engine](./Search%20Engine/) | Full-text search service | In progress |
| [Autocomplete](./Autocomplete/) | Type-ahead search | In progress |
| [Splunk](./Splunk/) | Log aggregation & analytics | In progress |

### Infrastructure Deep Dives

| Topic | What it covers |
|---|---|
| [API Gateway](./API%20Gateway/) | Routing, auth, rate limiting, observability |
| [Cache](./Cache/) | Eviction policies, invalidation, distributed caching |
| [Kafka](./Kafka/) | Event streaming, partitioning, consumer groups |
| [Key-Value Store](./Key-Value%20Store/) | Distributed KV design, consistency, replication |

---

## Documentation Framework

Every case study follows the same 18-document lifecycle defined in [system-design-docs-checklist.md](./system-design-docs-checklist.md).

### Phase 1 — Before Implementation

| Document | What it answers |
|---|---|
| PRD | Problem, users, success metrics, MVP scope, risks |
| ADRs | Architectural decisions with full rationale and rejected alternatives |
| System Design (HLD) | Component topology, data flow, capacity estimation, failure modes |
| API Contract | All endpoints, auth model, rate limits, error envelopes, versioning |
| Database Design | Schema, indexes, partitioning, replication, retention |
| Security Threat Model | STRIDE analysis, trust boundaries, auth design, OWASP mitigations |
| Low-Level Design (LLD) | Key algorithms, concurrency model, internal interfaces |
| Sequence Diagrams | Time-ordered flows for all critical paths |
| UI/UX Design | User flows, component library, accessibility targets |
| Data Privacy & Compliance | PII inventory, GDPR/CCPA mapping, erasure cascade |
| Dependency Inventory | Third-party services, licenses, SLAs, fallback plans |

### Phase 2 — During Implementation

| Document | What it answers |
|---|---|
| Cost Model | Unit economics, scaling cost curve, cost drivers |
| Testing Strategy | Test pyramid, coverage targets, load and chaos testing |
| Observability Design | Logging, metrics (RED), distributed tracing, SLOs/SLIs |

### Phase 3 — Before Launch

| Document | What it answers |
|---|---|
| Operational Runbook | Deployment, common failures, escalation, DR procedure |
| Launch / Rollout Plan | Feature flags, canary strategy, go/no-go criteria |
| Data Migration Plan | Zero-downtime migration, backfill, validation, rollback |
| DR / BCP | RTO/RPO targets, backup strategy, failover procedure |

---

## Spotlight: TinyURL

The most complete case study. Designs a self-hosted URL shortener capable of 50,000 RPS with sub-10 ms p99 redirect latency — without relying on any external SaaS.

```
Client → Cloudflare CDN → Load Balancer
                              ├── Read Service  → Redis Cluster → PostgreSQL
                              └── Write Service → PostgreSQL
                                                       ↓
                                               Kafka → ClickHouse (analytics)
```

**Key decisions:** Go backend · Redis read path · Base62 counter keys · 302 over 301 · Async analytics via Kafka · ClickHouse for time-series aggregation · Cloudflare over CloudFront (62× cost reduction)

Full documentation: [TinyURL/](./TinyURL/)

---

## What Big Tech Actually Does

| Company | Pre-implementation doc | Notable practice |
|---|---|---|
| **Amazon** | PR/FAQ + OP1 | "Work backwards" — write the press release first |
| **Google** | Design Doc (mandatory) | Peer review required; explicit alternatives section |
| **Meta** | ERD + Systems Review | SRE reviews capacity before any launch |
| **Stripe** | API contract first | API design reviewed by API council before coding |
| **Netflix** | Chaos Doc | DR/failure scenarios documented before build |
| **Uber** | Architecture Proposal (RFC) | ADR equivalent with mandatory security section |
| **Apple** | PRD + Legal review | Privacy review gates engineering start |

---

> **Rule of thumb:** Docs that are expensive to change — schema, API contract, security model — need maximum upfront rigor. The cost of a missing doc grows exponentially after deployment.
