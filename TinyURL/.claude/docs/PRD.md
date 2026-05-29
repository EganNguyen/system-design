# TinyURL — Enterprise Link Management Infrastructure

## Problem

Growth and Core Infrastructure Engineering teams at enterprise scale (SaaS platforms, large e-commerce, transactional notification systems) cannot use public SaaS link shorteners at 1B+ daily redirects: vendor costs become prohibitive, external network hops violate sub-10 ms latency requirements, and shared infrastructure breaches GDPR/CCPA data residency obligations. Standard open-source alternatives (Shlink, YOURLS) collapse under peak loads of 50,000+ RPS due to relational DB connection pool exhaustion and synchronous analytics write paths that block the redirect response.

## Evidence

- **Vendor cost taxation** — Assumption: at 100M distinct links and 1B daily redirects, third-party SaaS pricing exceeds the fully-loaded cost of an internal engineering team. Needs validation via: audit of projected API tier pricing against current headcount costs.
- **Latency-to-conversion penalty** — Industry benchmarks cite ~1% conversion drop per 100 ms of additional page latency on mobile. External SaaS redirects introduce an unpredictable network hop; vendor p99 spikes of 200–500 ms under morning peak traffic blasts are observable as user-visible blank-screen delay. Needs validation via: synthetic load test introducing artificial redirect delays (50 ms, 200 ms, 500 ms) and measuring downstream bounce rates.
- **OSS architectural degradation** — At 11,500 average RPS (1B/day) with peaks at 50,000+ RPS, naive B-Tree indexed relational stores exhaust connection pools, replicas fall behind producing intermittent 404s for freshly written URLs, and synchronous analytics writes eliminate any sub-10 ms redirect path. Needs validation via: load-test benchmark of standard Shlink + PostgreSQL at 20,000 RPS to document exact degradation point.

## Users

- **Primary**: Growth & Core Infrastructure Engineering teams at enterprise scale — the engineers who own transactional notification pipelines (SMS, email, push) where every message contains a personalized, tracked short link. They care about p99 latency, throughput headroom, deployment autonomy, and total infrastructure cost — not UI polish.
- **Not for**: Individual developers, general consumers, small marketing teams, or anyone whose link volume stays well below the SaaS vendor pain threshold.

## Hypothesis

We believe **a self-hosted, high-performance Link Management Infrastructure** will **eliminate prohibitive third-party vendor costs and prevent conversion loss from redirect latency** for **Growth & Infrastructure Engineering teams at enterprise scale**. We'll know we're right when **the system sustains a simulated peak load of 50,000 RPS with a p99 redirect latency of < 10 ms at a total operational infrastructure cost under 15% of equivalent enterprise SaaS pricing**.

## Success Metrics

| Metric | Target | How measured |
|---|---|---|
| p99 redirect latency at peak | < 10 ms | k6 load test at 50,000 RPS sustained for 10 min |
| Throughput ceiling | ≥ 50,000 RPS | k6 ramp test to failure threshold |
| Cache hit rate | ≥ 95% | Redis INFO stats during load test |
| Infrastructure cost vs. SaaS equivalent | < 15% | Cloud billing estimate vs. Bitly Enterprise tier pricing |
| Analytics data loss under peak | < 0.1% | Compare emitted events vs. stored events during load test |
| Redirect availability | ≥ 99.99% | Uptime monitor during 72-hour soak test |

## Scope

**MVP (v1)** — Minimum to validate the core latency, throughput, and cost hypothesis:

- **High-performance redirect engine** — serves HTTP 301/302 responses directly from an in-memory cache layer; the redirect path never touches the primary DB on cache hit.
- **Programmatic shorten API** — lean authenticated write endpoint designed for automated transactional service ingestion, not human users.
- **Asynchronous click analytics** — fully decoupled from the redirect path; click events are enqueued after the redirect response is issued, never blocking it.
- **Deterministic Base62 key generation** — conflict-free unique short code generation via a distributed range allocator or unique ID coordinator.

**Out of scope**

- Custom branded aliases (e.g., `/sale50`) — introduces key-collision complexity irrelevant to v1 hypothesis testing. Deferred to v2.
- URL expiry / TTL management — background garbage collection adds storage layer complexity; deferred until the core storage behavior stabilizes.
- Advanced user accounts, multi-tenant UI, RBAC — v1 is operated via environment configuration and infra metrics, not a management portal. Deferred to v2.
- Dynamic QR code generation — presentation-layer feature; irrelevant to the concurrency hypothesis. Deferred to v2.
- CDN / edge network infrastructure — this is an origin service. Edge routing is assumed to be handled by existing enterprise ingress (Cloudflare, AWS CloudFront, Nginx).
- End-user analytics dashboard — raw click stream data is emitted to a high-throughput datastore or enterprise log aggregator; no UI charting in scope.
- Link domain management & SSL issuance — domain routing and SSL termination are handled at the infrastructure ingress layer, not inside this service.

## Delivery Milestones

| # | Milestone | Outcome | Status | Plan |
|---|---|---|---|---|
| 1 | Redirect Engine + Cache Layer | p99 < 10 ms redirect from hot cache; cache hit rate ≥ 95% under 50k RPS | pending | — |
| 2 | Shorten API + Key Generation | Authenticated write endpoint functional; no key collisions under concurrent load | pending | — |
| 3 | Async Analytics Pipeline | Click events captured with < 0.1% loss; redirect p99 unaffected by analytics load | pending | — |
| 4 | Load Validation | Full 50,000 RPS soak test passes all success metrics; cost model confirmed | pending | — |

## Open Questions

- [ ] **Analytics write durability** — What level of click event loss is acceptable under catastrophic failure? If < 0.1% loss is required even during crashes, the message broker pipeline complexity increases significantly vs. an aggressive memory-buffered async approach.
- [ ] **Deployment topology** — Single-region or active-active multi-region? The < 10 ms requirement is achievable within a single region over enterprise LAN/VPC; globally it requires co-located edge nodes and multi-master cache synchronization. This heavily influences the architecture.
- [ ] **Cache long-tail distribution** — What fraction of traffic hits non-hot links? If cache hit rate drops materially below 95% due to long-tail access patterns, the persistent datastore must be sized and indexed to absorb the slack without cascading latency.

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Cache stampede on cold start or hot-key expiry | High | Critical — violates p99 target | Probabilistic early expiry + background refresh; mutex lock on first DB fetch per key |
| Async analytics buffer overflow at extreme peak | Medium | Medium — data loss > 0.1% threshold | Bounded channel with backpressure; overflow metric triggers alert, not silent drop |
| Key generation coordinator becomes bottleneck | Medium | High — blocks all shorten writes | Pre-allocated range caching per app node; coordinator only contacted when range is exhausted |
| Multi-region replication lag causing stale redirects | Medium | Medium — newly created links return 404 | Read-your-writes guarantee for the creating service; eventual consistency acceptable for cross-region |
| Compliance gap if analytics data includes raw IPs | High | High — GDPR/CCPA violation | Hash or truncate IPs at ingestion boundary; data residency deployment constraint documented in runbook |

---

*Status: DRAFT — requirements only. Implementation planning pending via `/plan`.*
