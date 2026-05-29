# How Large-Scale Apps Use Short-Link Redirect Systems

> Based on the TinyURL Enterprise Link Management Infrastructure PRD & System Design

---

## The Core Problem

At enterprise scale — 1B+ daily redirects, 100M+ distinct links — using a public SaaS shortener (Bitly, TinyURL.com) becomes untenable:

| Pain Point | What Breaks |
|---|---|
| **Vendor cost** | At 1B/day, SaaS pricing exceeds the cost of an internal engineering team |
| **Latency** | External network hops violate sub-10 ms redirect requirements; p99 vendor spikes of 200–500 ms cause measurable conversion loss |
| **Data residency** | Shared SaaS infrastructure breaches GDPR/CCPA obligations |
| **Throughput ceiling** | Open-source alternatives (Shlink, YOURLS) collapse at 50,000+ RPS due to DB connection exhaustion |

The solution: a **self-hosted, distributed short-link infrastructure** embedded inside the main app platform.

---

## Big Picture: Where Short-Link Lives in a Large-Scale App

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         LARGE-SCALE APP PLATFORM                             │
│                                                                              │
│  ┌────────────────────┐   ┌─────────────────────┐   ┌──────────────────┐   │
│  │  Transactional      │   │   Marketing /        │   │  Mobile App /    │   │
│  │  Notification       │   │   Growth Platform    │   │  Web Frontend    │   │
│  │  Pipeline           │   │   (Email campaigns,  │   │  (Deep links,    │   │
│  │  (SMS, Email, Push) │   │   landing pages)     │   │  share sheets)   │   │
│  └────────┬───────────┘   └──────────┬───────────┘   └──────┬───────────┘   │
│           │                          │                       │               │
│           └──────────────────────────▼───────────────────────┘               │
│                                      │                                       │
│                    ┌─────────────────▼───────────────────┐                  │
│                    │     Internal Short-Link Service       │                  │
│                    │     (Self-hosted TinyURL cluster)     │                  │
│                    │                                       │                  │
│                    │  • Shorten API  • Redirect Engine     │                  │
│                    │  • Analytics    • Governance          │                  │
│                    └─────────────────────────────────────┘                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

Every team that sends users to a URL routes through the central short-link service. This creates a **single plane of control** for analytics, routing, and governance.

---

## High-Level Architecture Diagram

```
                        ┌─────────────────────────────────────────────┐
                        │                 Internal Producers            │
                        │                                               │
                        │  [SMS Service]  [Email Service]  [Push Svc]  │
                        │       │               │               │       │
                        │       └───────────────┼───────────────┘       │
                        │                       │  POST /api/v1/urls    │
                        └───────────────────────┼───────────────────────┘
                                                │
                                                ▼
                        ┌───────────────────────────────────────────────┐
                        │           Short-Link Write Service             │
                        │                                               │
                        │  • Validates URL (Safe Browsing, blacklist)   │
                        │  • Generates Base62 short code (7 chars)      │
                        │  • Writes to Primary DB + Redis cache         │
                        │  • Returns short URL to producer              │
                        └───────────────────────────────────────────────┘
                                                │
                                    short link embedded in message
                                                │
                        ┌───────────────────────▼───────────────────────┐
                        │                  END USERS                     │
                        │          (click link in SMS / email)           │
                        └───────────────────────────────────────────────┘
                                                │
                                                ▼
                        ┌───────────────────────────────────────────────┐
                        │          CDN (Cloudflare / CloudFront)         │
                        │   Caches redirect responses at edge globally   │
                        │   Cache HIT → instant 302, zero origin call    │
                        └───────────────────┬───────────────────────────┘
                                            │ cache miss
                                            ▼
                        ┌───────────────────────────────────────────────┐
                        │         Load Balancer (AWS ALB / Nginx)        │
                        └──────────┬────────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │       Short-Link Read Service         │
                    │   (stateless, horizontally scaled)    │
                    │                                       │
                    │  1. Lookup short_code in Redis        │
                    │  2. HIT  → async fire ClickEvent      │
                    │           → 302 redirect to long URL  │
                    │  3. MISS → query DB Read Replica      │
                    │           → populate cache            │
                    │           → async fire ClickEvent     │
                    │           → 302 redirect              │
                    └────────┬──────────────────┬──────────┘
                             │                  │
              ┌──────────────▼──────┐   ┌───────▼──────────────────────┐
              │    Redis Cluster     │   │   Analytics Event Pipeline    │
              │  (L2 cache layer)    │   │                               │
              │                      │   │  Kafka → Analytics Worker     │
              │  url:{code} →        │   │  → GeoIP + UA enrichment     │
              │  { long_url, meta }  │   │  → IP hashing (GDPR)         │
              │  TTL: 24h default    │   │  → ClickHouse (analytics DB)  │
              └──────────┬──────────┘   └───────────────────────────────┘
                         │ miss
              ┌──────────▼──────────┐
              │    Primary DB        │
              │  + Read Replicas     │
              │  (PostgreSQL →       │
              │   Cassandra at       │
              │   500M+ URLs)        │
              └─────────────────────┘
```

---

## How Each Major App Use Case Works

### 1. Transactional Notification Pipeline
**e.g., "Your order has shipped — track here: `app.co/x7kR2`"**

```
Order Service
    │
    ├── Creates personalized long URL
    │   (with user ID, order ID, UTM params)
    │
    ├── POST /api/v1/urls → Short-Link Write Service
    │   Returns: app.co/x7kR2
    │
    └── Embeds short link in SMS / Email / Push notification

User clicks → Short-Link Read Service → 302 → Destination page
                     │
                     └── ClickEvent → Kafka → ClickHouse
                         (delivery confirmation + engagement metric)
```

Every message gets a **unique, personalized short link**. The short-link service handles 100M+ new links/day from this flow alone.

---

### 2. Centralized Analytics & Attribution

All clicks flow through the redirect service, which emits structured events before issuing the redirect:

```
ClickEvent {
  short_code:  "x7kR2"
  long_url:    "https://app.com/orders/12345?utm_source=sms&utm_medium=..."
  timestamp:   2026-05-29T09:13:44Z
  ip_hash:     "sha256(ip + rotating_salt)"   ← GDPR-safe
  user_agent:  "Mozilla/5.0 ... iPhone ..."
  referrer:    "direct"
  geo_country: "US"
  geo_region:  "CA"
}
```

Because the redirect happens centrally, analytics are **channel-agnostic** — SMS, email, push, and web share the same click data model. Attribution doesn't require per-channel SDK integrations.

---

### 3. Dynamic Routing (Feature Flags / Experiments)

Short links decouple the **link distributed to users** from the **destination**:

```
Short Code:  app.co/summer-sale
             │
             ├── User A (mobile iOS)  → /sale?v=mobile-a
             ├── User B (mobile Android) → /sale?v=mobile-b
             ├── User C (desktop)    → /sale?v=desktop-control
             └── User D (EU)         → /eu/sale  (GDPR routing)
```

The redirect service reads routing rules from a **feature flag store** at lookup time. Millions of already-distributed links can be silently rerouted without re-sending messages.

---

### 4. Traffic Spike Handling

The two-layer cache absorbs viral or burst traffic:

```
Layer 1: CDN (Cloudflare / CloudFront)
         ├── Caches 302 + Location header at 200+ global PoPs
         ├── TTL: 5 minutes for hot codes
         └── A viral link with 1M clicks → ~0 origin requests during TTL window

Layer 2: Redis Cluster
         ├── Working set: top 20% of links = ~80% of traffic (Pareto)
         ├── 14 GB/day hot set fits comfortably in Redis memory
         ├── Target: ≥ 95% cache hit rate
         └── Cache miss → DB read replica (never primary)
```

Peak load target: **50,000 RPS sustained** with **p99 redirect latency < 10 ms**.

---

### 5. Governance, Security & Compliance

All link creation is gated through a **single authenticated write endpoint**:

```
Write Service enforces:
  ├── URL validation (format, scheme, private IP block — SSRF defense)
  ├── Malicious content scan (Google Safe Browsing API + PhishTank)
  ├── Rate limiting per producer service (token bucket in Redis)
  ├── Domain allowlist / blocklist
  └── Audit log of all link creation (who created, when, for which campaign)

Analytics pipeline enforces:
  ├── IP hashing at ingestion boundary (GDPR/CCPA)
  ├── Data residency routing (EU clicks → EU ClickHouse node)
  └── Retention policy enforced at DB level
```

No external SaaS shortener can enforce these controls. The internal service is the **only path** link creation goes through.

---

### 6. Cost Efficiency at Scale

| Approach | Cost at 1B redirects/day |
|---|---|
| Third-party SaaS (Bitly Enterprise) | ~$X,XXX/month (vendor pricing) |
| Self-hosted short-link cluster | ~15% of equivalent SaaS cost |

Infrastructure at target scale:

```
Component           Sizing at 50K RPS peak
─────────────────────────────────────────────────────
Read Service        N stateless pods behind ALB (auto-scale)
Write Service       M pods (write volume is 100× lower)
Redis Cluster       3+ nodes, LRU eviction, 14GB+ hot set
Primary DB          PostgreSQL (< 500M URLs) → Cassandra (500M+)
Read Replicas       Per region, absorb cache misses
Kafka               click_events topic, partitioned by short_code
ClickHouse          Distributed cluster, analytics-only writes
CDN                 Cloudflare / CloudFront (200+ global PoPs)
```

---

## Redirect Flow: Step-by-Step (Read Path)

```
User clicks:  app.co/x7kR2
      │
      ▼
[CDN Edge PoP]
      ├── Cache HIT  ─────────────────────────────────────► 302 → long URL
      │                                                     (< 1 ms, no origin)
      └── Cache MISS ──────────────────────────────────────────────────┐
                                                                       │
                                                              [Load Balancer]
                                                                       │
                                                           [Read Service pod]
                                                                       │
                                                            ┌──────────▼──────────┐
                                                            │  Redis GET url:x7kR2 │
                                                            └──────────┬──────────┘
                                                                       │
                                                   HIT ────────────────┤
                                                    │                  │ MISS
                                                    │          ┌───────▼──────────┐
                                                    │          │  DB Read Replica  │
                                                    │          │  SELECT long_url  │
                                                    │          └───────┬──────────┘
                                                    │                  │
                                                    │          Redis SET (populate)
                                                    │                  │
                                                    └──────────────────┘
                                                                       │
                                                        async: publish ClickEvent
                                                                to Kafka
                                                                       │
                                                         302 redirect → long URL
                                                            (p99 target: < 10 ms)
```

---

## Multi-Service Topology (Distributed Across Services)

Large-scale apps don't have a single TinyURL service — they have a **cluster of short-link services** that together form the platform:

```
┌──────────────────────────────────────────────────────────────┐
│                 Short-Link Platform Cluster                   │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  Write       │  │  Read        │  │  Analytics Worker  │ │
│  │  Service     │  │  Service     │  │  (Go consumer)     │ │
│  │  (N pods)    │  │  (M pods)    │  │  (K pods)          │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────┘ │
│         │                 │                    │             │
│  ┌──────▼─────────────────▼────────┐   ┌───────▼──────────┐ │
│  │       Redis Cluster              │   │  Kafka Cluster    │ │
│  │   (shared cache, all pods)       │   │  (click_events)   │ │
│  └──────────────────┬──────────────┘   └──────────────────┘ │
│                     │                                        │
│  ┌──────────────────▼──────────────────────────────────────┐ │
│  │        Database Layer                                    │ │
│  │  Primary DB ──► Read Replicas (3+)                       │ │
│  │  PostgreSQL (< 500M) → Cassandra/ScyllaDB (500M+)        │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │        Analytics Store (ClickHouse)                      │ │
│  │        Columnar, append-only, OLAP queries               │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

Each component scales **independently**:
- Read Service scales horizontally to absorb 500K req/sec
- Write Service scales separately (100× lower volume)
- Analytics Workers scale to drain Kafka lag
- Redis shards to absorb hot-key pressure

---

## Key Design Decisions Summary

| Decision | Choice | Why |
|---|---|---|
| **Redirect type** | 302 (temporary) | Every click is captured; CDN caches at edge with 5-min TTL |
| **Short code generation** | Distributed counter + Base62 | Collision-free, no DB round-trip per write; 62⁷ ≈ 3.5T unique codes |
| **Cache strategy** | CDN (L1) + Redis (L2) | CDN absorbs viral bursts; Redis covers non-CDN-cached long-tail |
| **Analytics path** | Fully async via Kafka | Redirect response is never blocked by analytics writes |
| **DB scaling path** | PostgreSQL → Cassandra | Start simple; switch at 500M+ URLs where point-lookup sharding matters |
| **IP privacy** | SHA-256 hash + rotating salt at ingestion | GDPR/CCPA compliant; no raw IPs stored |
| **Auth** | JWT (15 min) + refresh token rotation | Secure per-service authentication on write path |

---

## Success Metrics (from PRD)

| Metric | Target |
|---|---|
| p99 redirect latency at peak | **< 10 ms** at 50,000 RPS |
| Cache hit rate | **≥ 95%** (Redis + CDN) |
| Analytics data loss | **< 0.1%** under peak |
| Redirect availability | **≥ 99.99%** (< 52 min downtime/year) |
| Infrastructure cost vs. SaaS | **< 15%** of equivalent Bitly Enterprise pricing |

---

*Based on: TinyURL Enterprise Link Management Infrastructure — PRD v1.0 & System Design v1.0 (May 2026)*
