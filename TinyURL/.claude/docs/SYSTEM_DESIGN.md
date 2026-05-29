# System Design: TinyURL

---

## 1. Problem Statement

Design a URL shortening service that converts arbitrary long URLs into compact, human-shareable short links and redirects users to the original destination when the short link is visited.

**Examples:**
```
Input:  https://www.example.com/blog/2026/05/a-very-long-article-title-that-nobody-wants-to-type
Output: https://tinyurl.com/aB3xYz7
```

---

## 2. Requirements

### 2.1 Functional Requirements

| Priority | Requirement                                                         |
|----------|---------------------------------------------------------------------|
| P0       | Shorten a long URL → return a unique short code                     |
| P0       | Redirect short URL to original long URL                             |
| P1       | Custom aliases (user-defined short codes)                           |
| P1       | URL expiry / TTL                                                    |
| P2       | Click analytics (count, geo, referrer, device)                      |
| P2       | User accounts (manage, delete, view owned URLs)                     |
| P3       | QR code generation per short URL                                    |

### 2.2 Non-Functional Requirements

| Requirement        | Target                                      |
|--------------------|---------------------------------------------|
| Availability       | 99.99% (< 52 min downtime/year)             |
| Redirect latency   | < 10ms p99                                  |
| Write latency      | < 100ms p99                                 |
| Durability         | No URL loss after successful write ACK      |
| Read/write ratio   | ~100:1 (read-heavy)                         |
| Consistency        | Eventual OK for analytics; strong for URLs  |

### 2.3 Out of Scope

- Browser extensions, SDKs, or third-party integrations
- A/B testing or link rotation
- Spam/phishing ML model training (rely on external APIs)

---

## 3. Capacity Estimation

### Assumptions

| Variable             | Value                    |
|----------------------|--------------------------|
| New URLs per day     | 100 million              |
| Read/write ratio     | 100:1                    |
| Average long URL     | 500 bytes                |
| Short code size      | ~10 bytes                |
| Metadata per record  | ~200 bytes               |
| Retention period     | 5 years                  |

### Calculations

```
Writes:
  100M URLs/day ÷ 86,400s = ~1,160 writes/sec (peak: ~5,000/sec)

Reads (redirects):
  100M × 100 = 10B reads/day ÷ 86,400s = ~115,000 reads/sec (peak: ~500,000/sec)

Storage (5 years):
  100M/day × 365 × 5 = 182.5B records
  182.5B × (500 + 10 + 200) bytes ≈ 128 TB raw
  With replication (3×): ~384 TB

Bandwidth:
  Read:  115,000/sec × 500 bytes ≈ 57 MB/s outbound
  Write: 1,160/sec  × 500 bytes ≈ 0.6 MB/s inbound

Cache (Redis):
  Assume top 20% URLs = 80% traffic (Pareto)
  20% of daily new = 20M URLs × 710 bytes ≈ 14 GB/day to cache hot set
```

---

## 4. High-Level Architecture

```
                         ┌─────────────────────────────────────────────┐
                         │                   Clients                   │
                         │   Browser  /  Mobile App  /  API Consumer   │
                         └────────────────────┬────────────────────────┘
                                              │
                                              ▼
                         ┌────────────────────────────────────────────┐
                         │           CDN (Cloudflare / CloudFront)    │
                         │   Caches 301 redirects at edge worldwide   │
                         └────────────────────┬───────────────────────┘
                                              │ (cache miss)
                                              ▼
                         ┌────────────────────────────────────────────┐
                         │         Load Balancer (AWS ALB / Nginx)    │
                         └──────────┬──────────────────┬─────────────┘
                                    │                  │
                       ┌────────────▼──────┐  ┌───────▼────────────┐
                       │   Read Service    │  │   Write Service     │
                       │   (Go — Fiber)    │  │   (Go — Fiber)      │
                       │  Redirect / Stats │  │  Shorten / Manage   │
                       └────────┬──────────┘  └──────────┬──────────┘
                                │                         │
                       ┌────────▼──────────────────────── ▼──────────┐
                       │              Redis Cluster (Cache)           │
                       │         url:{code} → long_url + meta         │
                       └────────┬──────────────────────────┬─────────┘
                                │ miss                      │ write-through
                       ┌────────▼──────────┐  ┌────────────▼─────────┐
                       │   DB Read Replica │  │   DB Primary          │
                       │   (SQL / NoSQL)   │  │   (SQL / NoSQL)       │
                       └───────────────────┘  └──────────────────────┘
                                │
                       ┌────────▼──────────────────────────────────────┐
                       │            Kafka (Analytics Events)            │
                       └────────────────────┬──────────────────────────┘
                                            │
                                   ┌────────▼────────┐
                                   │   ClickHouse    │
                                   │ (Analytics DB)  │
                                   └─────────────────┘
```

---

## 5. Core User Flows

### 5.1 Shorten a URL (Write Path)

```
1.  Client          POST /api/v1/urls  { long_url, custom_code?, expires_in? }
2.  Write Service   Validate URL (format, scheme, blacklist, Safe Browsing)
3.  Write Service   Check custom alias availability (if provided)
4.  ID Generator    Fetch next counter → Base62 encode → 7-char short_code
5.  Write Service   INSERT INTO urls (short_code, long_url, ...) → Primary DB
6.  Write Service   SET url:{short_code} in Redis (write-through, TTL = expires_in)
7.  Write Service   Return 201 { short_url, short_code, expires_at }
```

### 5.2 Redirect (Read Path — Hot)

```
1.  Client          GET tinyurl.com/aB3xYz7
2.  CDN             Check edge cache → HIT → 301/302 Location header (no origin call)
3.  CDN             MISS → forward to Load Balancer
4.  Read Service    GET url:aB3xYz7 from Redis
                    ├── HIT  → async publish ClickEvent to Kafka → 302 redirect
                    └── MISS → query DB Read Replica
                               ├── FOUND   → cache SET → async Kafka → 302 redirect
                               └── NOT FOUND → 404
```

### 5.3 Analytics Event Processing (Async)

```
Read Service  →  Kafka topic: click_events
                      │
              ┌───────▼───────────┐
              │  Analytics Worker  │  (Go consumer group)
              │  Enriches:         │
              │  - GeoIP lookup    │
              │  - UA parsing      │
              │  - IP hashing      │
              └───────┬───────────┘
                      │
              ClickHouse (INSERT INTO click_events)
```

---

## 6. Key Design Decisions

### 6.1 301 vs 302 Redirect

| Type | Cached by Browser/CDN | Analytics Captured |
|------|-----------------------|--------------------|
| 301  | Yes (permanent)       | First visit only   |
| 302  | No                    | Every visit        |

**Decision:** Use **302** for all redirects so every click is captured. Cache at the CDN layer (not browser) with a short TTL (5 min) for hot codes to reduce origin load while still logging analytics.

---

### 6.2 Short Code Generation

**Chosen: Distributed Counter + Base62**

```
Global counter (Redis INCR or Snowflake ID)
  └─▶ Base62 encode (62^7 = ~3.5 trillion unique codes)
      └─▶ 7-character short code
```

Each app server pre-fetches a range of 10,000 IDs from the counter service, eliminating per-request network calls to the ID service on the hot write path.

**Why not hash-based?**
- MD5/SHA truncation has collision probability that grows with dataset size
- Requires a DB round-trip to check collision on every write
- Counter is deterministic and collision-free

---

### 6.3 Database Choice

The `urls` table has a simple access pattern: point lookups by `short_code`. This is well-suited to both SQL and NoSQL. The system uses a pluggable data layer (see LLD §18 for full comparison):

| Scale              | Recommended DB                              |
|--------------------|---------------------------------------------|
| 0 – 50M URLs       | PostgreSQL (single node or managed RDS)     |
| 50M – 500M URLs    | PostgreSQL + read replicas                  |
| 500M+ URLs         | Cassandra / ScyllaDB (wide-column, sharded) |

Analytics data (click events) always goes to **ClickHouse** (columnar, append-only, OLAP).

---

### 6.4 Caching Strategy

**Two-layer cache:**

```
L1: CDN (Cloudflare / CloudFront)
    - Caches redirect responses (302 + Location header)
    - TTL: 5 minutes for hot codes
    - Benefit: zero origin load for viral links

L2: Redis Cluster
    - Key: url:{short_code}  Value: { long_url, expires_at }
    - TTL: matches URL expiry, or 24h default
    - Target: >95% cache hit rate
    - Write-through on URL creation
    - Invalidate on delete / deactivation
```

**Cache eviction:** LRU. Top 20% of URLs handle ~80% of traffic; the working set fits comfortably in Redis memory at scale.

---

### 6.5 Read/Write Separation

Separate Go services (or separate Fiber route groups deployable independently) handle reads and writes:

- **Read Service** is stateless, scales horizontally to handle 500K req/sec
- **Write Service** scales independently; write volume is 100× lower
- Both talk to Redis; reads fall back to read replicas, writes go to primary only

---

## 7. Data Flow Diagram

```
[Client]
   │
   ├── POST /api/v1/urls ──▶ [Write Service]
   │                               │
   │                        Validate → ID Gen → DB Write → Cache Write
   │                               │
   │                        ◀── 201 short_url
   │
   └── GET /{code} ────────▶ [CDN]
                                │ miss
                             [Read Service]
                                │
                         Redis HIT ──▶ [Kafka] ──▶ [Analytics Worker] ──▶ [ClickHouse]
                                │
                         Redis MISS ──▶ [DB Replica] ──▶ Cache ──▶ [Kafka] ──▶ ...
```

---

## 8. API Surface (Summary)

| Method   | Endpoint                              | Description                     |
|----------|---------------------------------------|---------------------------------|
| `POST`   | `/api/v1/urls`                        | Create short URL                |
| `GET`    | `/{code}`                             | Redirect to long URL            |
| `GET`    | `/api/v1/urls/{code}`                 | Get URL metadata                |
| `DELETE` | `/api/v1/urls/{code}`                 | Deactivate URL (soft delete)    |
| `GET`    | `/api/v1/urls/{code}/analytics`       | Get click analytics             |
| `GET`    | `/api/v1/users/me/urls`               | List authenticated user's URLs  |

Full request/response contracts are in `LLD.md §7`.

---

## 9. Scalability

### Horizontal Scaling

```
Component          Scaling Strategy
─────────────────────────────────────────────────────────────
Read Service       Stateless; add pods behind ALB freely
Write Service      Stateless; scale independently
Redis              Redis Cluster (hash-slot sharding, 3+ nodes)
DB Primary         Vertical first; then shard by short_code prefix
DB Replicas        Add read replicas per region
Kafka              Partition click_events topic by short_code
ClickHouse         Distributed cluster with replica shards
CDN                Auto-scales; deploy to 200+ PoPs globally
```

### Bottleneck Analysis

| Bottleneck           | Trigger                         | Solution                                      |
|----------------------|---------------------------------|-----------------------------------------------|
| Redis memory         | Cache miss rate rises           | Increase shard count; evict with LRU          |
| DB read IOPS         | Cache hit rate < 90%            | Add read replicas; increase Redis capacity    |
| DB write throughput  | Write QPS > 5,000               | Batch writes; switch to Cassandra             |
| ID generator         | Counter contention at high QPS  | Pre-fetch ID ranges (10K per node)            |
| Kafka lag            | Analytics worker too slow       | Scale consumer group; increase partitions     |

---

## 10. Reliability & Fault Tolerance

### Failure Scenarios

| Component Fails      | User Impact                     | Mitigation                                         |
|----------------------|---------------------------------|----------------------------------------------------|
| Single Read pod      | None                            | ALB health checks; auto-restart (K8s)              |
| Redis node           | Cache miss spike → DB load      | Redis Cluster (automatic failover in < 30s)        |
| DB Primary           | Writes fail                     | Automatic failover to replica (RDS Multi-AZ / Patroni) |
| DB Replica           | Reads fall to primary           | Multiple replicas; circuit breaker                 |
| Kafka broker         | Analytics drops temporarily     | Kafka replication factor 3; consumer retry         |
| CDN PoP              | Slightly higher latency         | Anycast routing; traffic moves to next PoP         |
| ID Generator         | Cannot create new URLs          | Pre-fetched local ID ranges survive short outages  |

### Data Durability

- DB: synchronous replication to at least 1 replica before write ACK
- Redis: AOF persistence enabled; acceptable to lose < 1s of cache on crash (DB is source of truth)
- Kafka: retention 7 days; analytics consumer can replay on failure

---

## 11. Security

| Threat                    | Mitigation                                                         |
|---------------------------|--------------------------------------------------------------------|
| Malicious URL submission  | Google Safe Browsing API + PhishTank + domain blacklist            |
| SSRF via submitted URLs   | Block private/loopback IPs; disallow non-HTTP(S) schemes           |
| Short code enumeration    | Counter offset (don't start at 0); optional code obfuscation       |
| DDoS on redirect endpoint | CDN absorbs at edge; rate limit by IP at ALB                       |
| Spam URL creation         | Rate limiting per IP + per user (token bucket in Redis)            |
| Analytics PII             | SHA-256 hash IPs with rotating salt before storage                 |
| Auth                      | JWT (15 min) + refresh token rotation; HTTPS enforced everywhere   |

---

## 12. Observability

### Metrics (Prometheus + Grafana)

- `redirect_latency_ms` (p50, p95, p99) — by short_code and region
- `cache_hit_rate` — Redis L1 and CDN hit %
- `url_create_rate` — writes/sec
- `db_query_duration_ms` — read and write separately
- `kafka_consumer_lag` — analytics pipeline health

### Alerts

| Alert                         | Threshold          | Action                          |
|-------------------------------|--------------------|---------------------------------|
| Redirect p99 > 50ms           | 5 min sustained    | PagerDuty — on-call             |
| Redis cache hit rate < 85%    | 10 min sustained   | Scale Redis / investigate       |
| DB primary replica lag > 10s  | Any                | PagerDuty — DB team             |
| Kafka consumer lag > 100K     | 15 min             | Scale analytics workers         |
| Error rate > 0.1%             | 5 min              | PagerDuty — on-call             |

### Distributed Tracing

- OpenTelemetry SDK in Go services
- Trace propagated across: Load Balancer → Read/Write Service → Redis → DB → Kafka
- Stored in Jaeger or AWS X-Ray

---

## 13. Deployment Architecture

```
                    ┌─────────────────────────────────────┐
                    │         GitHub Actions CI/CD         │
                    └──────────────────┬──────────────────┘
                                       │
                          ┌────────────▼───────────┐
                          │   ECR (Docker Images)  │
                          └────────────┬───────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │          Kubernetes (EKS)            │
                    │                                      │
                    │  ┌──────────────┐ ┌───────────────┐ │
                    │  │ Read Service │ │ Write Service  │ │
                    │  │  (N pods)    │ │   (M pods)     │ │
                    │  └──────────────┘ └───────────────┘ │
                    │                                      │
                    │  ┌──────────────┐ ┌───────────────┐ │
                    │  │  Next.js     │ │ Analytics      │ │
                    │  │  (Frontend)  │ │ Worker (Go)    │ │
                    │  └──────────────┘ └───────────────┘ │
                    └─────────────────────────────────────┘

Environments: dev → staging → production (blue/green deploy)
IaC: Terraform (AWS resources) + Helm charts (K8s manifests)
```

---

## 14. Technology Stack

| Layer              | Technology                                      |
|--------------------|-------------------------------------------------|
| Backend API        | **Go** (Fiber / Chi)                            |
| Frontend           | **Next.js 14** (App Router)                     |
| Primary DB         | Flexible (PostgreSQL → Cassandra at scale)      |
| Cache              | Redis 7 (Cluster)                               |
| Analytics DB       | ClickHouse                                      |
| Message Queue      | Apache Kafka                                    |
| CDN                | Cloudflare / CloudFront                         |
| Container Infra    | Kubernetes (EKS)                                |
| IaC                | Terraform + Helm                                |
| CI/CD              | GitHub Actions                                  |
| Monitoring         | Prometheus + Grafana + OpenTelemetry + Sentry   |

---

## 15. Open Questions / Future Considerations

| Topic                  | Notes                                                                  |
|------------------------|------------------------------------------------------------------------|
| Multi-region active-active | Route users to nearest region; sync DB via CockroachDB or DynamoDB Global Tables |
| Link preview (OG tags) | Serve a meta-redirect page instead of bare 302 for social previews    |
| Abuse ML model         | Replace rule-based blacklist with a trained classifier for phishing    |
| Paid tier features     | Branded domains (`yourbrand.co/abc`), password-protected links         |
| GDPR / data residency  | Store EU user data in EU region only; purge on account deletion        |

---

*Document version: 1.0 — May 2026*
*See also: `LLD.md` for data models, API contracts, and implementation details.*
