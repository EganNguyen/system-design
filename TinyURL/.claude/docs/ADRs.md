# Architecture Decision Records: TinyURL Service

> One record per significant architectural decision.
> Format: Context → Decision → Alternatives → Consequences → Status.
> Stored together for discoverability; extract to `/docs/adr/ADR-NNN.md` per file if the repo grows.

---

## ADR-001: Counter + Base62 Encoding for Short Code Generation

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering

### Context

Every URL shortening request requires generating a globally unique, short, URL-safe string
(the short code). The code must be collision-free under concurrent writes at 500 writes/sec
peak, human-typeable, and as short as possible to minimise the final URL length. Three
strategies were evaluated.

### Decision

Use a **distributed counter incremented atomically (Redis INCR or Snowflake ID) with the
result Base62-encoded** to produce a 7-character short code. The counter starts at a large
random offset (≥ 10 billion) to prevent sequential enumeration.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **MD5/SHA-1 hash + truncation** | Collision probability grows as the dataset fills. At 100M URLs the birthday paradox makes collisions non-negligible with a 43-bit truncation. Retry-on-collision adds latency unpredictability. |
| **UUID v4 truncation** | UUID is 128 bits; truncating to 7 chars (42 bits) produces a collision rate similar to hashing. Requires a DB round-trip uniqueness check on every write. |
| **Random Base62 string** | Same birthday problem as UUID truncation. No ordering guarantee; no counter to pre-fetch ranges. |

### Consequences

**Positive:**
- Deterministic — no collision possible within a counter range; retry only needed if the same counter value is issued twice (prevented by atomic INCR).
- ID range pre-fetching: each app node claims a block of 1,000 counter values, eliminating per-request round trips to the counter service.
- Naturally ordered — useful for range scans and debugging.

**Negative:**
- Counter service is a new dependency. Mitigated by pre-fetched local ranges per node (coordinator only contacted when range exhausted).
- Starting offset reduces practical keyspace slightly (~3.5T − 10B), which is effectively infinite at target scale.
- Sequential codes reveal approximate creation volume if enumerated. Mitigated by the large random start offset.

---

## ADR-002: 302 (Temporary) Redirect over 301 (Permanent)

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering, Product

### Context

The redirect hot path must return an HTTP redirect to the destination URL. Two standard
status codes apply: `301 Moved Permanently` and `302 Found`. The choice has direct
implications for analytics capture and cache behaviour at CDN and browser level.

### Decision

Use **302 Found** for all redirects. Cache at the CDN layer with a short TTL (5 minutes
for hot codes) to reduce origin load while ensuring every click is routed through the
origin analytics pipeline.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **301 Permanent** | Browsers and CDNs cache 301 responses indefinitely. Once cached, subsequent clicks bypass the origin entirely — click events are never captured. Analytics become impossible without complex client-side workarounds. |
| **301 for analytics-disabled links, 302 for analytics-enabled** | Two-tier approach increases implementation complexity and creates inconsistent behaviour. Requires storing the analytics preference per link and checking it on every redirect. |

### Consequences

**Positive:**
- Every redirect reaches the origin, enabling accurate click analytics.
- CDN TTL is configurable per link; hot links can cache at edge for 5 minutes, reducing origin RPS while tolerating minor undercounting within the TTL window.
- Link destination can be updated (after a typo) without browsers serving stale 301 responses.

**Negative:**
- Higher origin traffic than 301 — mitigated by Redis L1 cache (95%+ hit rate) and CDN short-TTL caching.
- Analytics count is slightly lower than true click count during CDN TTL window — negligible at 5-minute TTL and acceptable per product requirements.

---

## ADR-003: URI Path Versioning over Header-Based Versioning

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** API Platform, Backend Engineering

### Context

The API must support versioning to allow breaking changes without forcing all consumers to
upgrade simultaneously. Two dominant strategies exist: embedding the version in the URI path
(`/api/v1/`) or expressing it via HTTP header (`Accept: application/vnd.tinyurl.v1+json`
or a custom `API-Version: 1` header).

### Decision

Use **URI path prefix versioning**: `https://api.tinyurl.com/api/v{N}/resource`.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **Accept header versioning** | Not visible in browser address bar or curl output; harder to test and debug. Breaks HTTP caching — `Vary: Accept` is poorly supported by CDNs and proxies. Requires header inspection at the load balancer rather than simple URL prefix routing. |
| **Custom request header** (`API-Version: 1`) | Same routing and caching problems as Accept header. Not idiomatic REST. Many HTTP clients and API gateways do not propagate custom headers by default. |
| **Query parameter** (`?v=1`) | Violates REST resource semantics. Query params are frequently stripped or ignored by CDNs. Confusing in logs and error messages. |

### Consequences

**Positive:**
- Trivially routeable at the ALB/Nginx level using URL prefix rules — no header inspection required.
- Cache-friendly: CDN cache key includes the full URI path, so v1 and v2 responses are never confused.
- Human-readable: visible in logs, browser dev tools, and error messages without additional tooling.
- Easy to test with `curl` and Postman without setting custom headers.

**Negative:**
- Violates strict REST "resource identity" principles — `/api/v1/urls/aB3xYz7` and `/api/v2/urls/aB3xYz7` are technically the same resource expressed twice.
- Consumers must change their base URL on version upgrade. Acceptable given the 6-month deprecation window.

---

## ADR-004: PostgreSQL as Primary Transactional Store with Cassandra Migration Path

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering, Infrastructure

### Context

The `urls` table is the core data store. It requires strong consistency (a freshly written
short code must be immediately readable by any redirect request), fast point-lookups by
`short_code`, and support for owner-scoped queries (`WHERE user_id = ?`). At 100M URLs
(target), with a documented path to 500M+ (hyper-scale), the choice carries significant
long-term consequences.

### Decision

**PostgreSQL** as the primary transactional store for URL records, user accounts, and audit
logs. Explicit migration path to **Cassandra / ScyllaDB** for the `urls` table documented
at the 500M record threshold where write throughput exceeds a single-primary PostgreSQL
capacity.

### Alternatives Considered

| Option | Why rejected at this stage |
|---|---|
| **Cassandra / ScyllaDB from day one** | Eventual consistency makes newly created links potentially invisible to redirects for a short window — violating the core product contract. Owner-scoped queries (`WHERE user_id = ?`) require denormalized lookup tables. Operational complexity is high for a team bootstrapping a new service. |
| **DynamoDB** | Vendor lock-in to AWS. Limited secondary index flexibility. Pricing at 1B reads/day is punitive without careful capacity planning. |
| **CockroachDB** | Postgres-compatible and globally distributed but introduces higher write latency (~10–50ms per write vs <5ms on single-node PG) due to distributed transaction coordination. Overkill for MVP. |
| **PlanetScale (MySQL)** | No foreign key constraints. MySQL dialect diverges from PostgreSQL in ways that matter (`RETURNING`, JSONB, window functions). Team has stronger PostgreSQL expertise. |
| **MongoDB** | Document model adds no value for a simple key-value lookup. Schema-less is a liability when the data model is well-defined and migrations are managed with `golang-migrate`. |

### Consequences

**Positive:**
- Strong consistency: reads always see the latest write — important for newly created links.
- Rich query support: owner queries, expiry scans, analytics joins all work natively.
- Mature ecosystem: `golang-migrate`, `sqlx`, RDS Multi-AZ, Patroni — well-understood by the team.
- Migration path to Cassandra is explicit: PostgreSQL → read replicas (at 50M) → Cassandra for `urls` only (at 500M), with PostgreSQL retained for `users` and `audit_log`.

**Negative:**
- Single primary bottleneck above ~5,000 write QPS. Mitigated by low write volume (<<1% of read volume at target scale).
- Horizontal sharding of PostgreSQL is complex. Accepted — migration to Cassandra is explicitly planned before hitting this limit.

---

## ADR-005: Redis Cluster as Read Cache

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering, Infrastructure

### Context

The redirect hot path requires sub-millisecond `short_code → long_url` lookups. At 11,600
average RPS (50,000 peak), direct database queries on every redirect would require an
infeasible number of read replicas. An in-memory cache is mandatory.

### Decision

**Redis Cluster** (hash-slot sharding, 3 primary shards + 3 replicas) as the L1 cache.
Key: `url:{short_code}` → `{long_url, expires_at}`. TTL matches URL expiry or defaults to
24 hours. Write-through on URL creation; immediate eviction on delete or deactivation.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **Memcached** | No replication — a node failure causes a complete cache miss spike to the DB. No per-key TTL on atomic operations. No Lua scripting (required for the token-bucket rate limiter). Redis serves multiple use cases (cache + rate limiting + session/token store + blacklist) — consolidating infrastructure. |
| **In-process LRU per app node** | Cache is not shared across pods — a miss goes to DB even if another pod cached the key moments ago. Invalidation on URL deletion requires broadcasting to all pods. Memory pressure limits cache size to the pod's heap. |
| **In-process LRU + Redis (two-tier)** | Added complexity not justified at MVP scale. In-process LRU retained only as a circuit-breaker for top-N hot codes when Redis is unavailable. |

### Consequences

**Positive:**
- Shared cache across all app nodes — one write populates the cache for the entire fleet.
- Built-in replication and automatic failover (<30s) via Redis Cluster.
- Lua scripting enables atomic token-bucket rate limiting.
- Consolidates infrastructure: same Redis cluster serves cache, rate limiter, token revocation list, and blacklist.

**Negative:**
- Additional operational dependency. Mitigated by ElastiCache managed service.
- ~0.5–1ms network hop vs in-process. Well within the p99 < 10ms budget at 95%+ hit rate.

---

## ADR-006: Kafka for Asynchronous Analytics Event Streaming

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering

### Context

Every redirect generates a click event that must be recorded for analytics. Recording must
not block the redirect response (p99 < 10ms). Three approaches were evaluated: synchronous
DB write in the redirect handler, direct async ClickHouse insert, and Kafka as a durable
buffer.

### Decision

**Publish click events to a Kafka topic (`click_events`) fire-and-forget after the redirect
response is sent.** A separate analytics consumer reads from the topic in batches and inserts
into ClickHouse.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **Synchronous write to ClickHouse in redirect handler** | ClickHouse is optimised for batch inserts; single-row inserts at 11,600 RPS cause excessive merge operations. A slow ClickHouse node directly adds tail latency to the redirect path. |
| **Synchronous write to PostgreSQL in redirect handler** | Adds a second DB write to every redirect — catastrophic at 11,600 RPS average. PostgreSQL is not designed for append-only time-series at this volume. |
| **Direct async ClickHouse insert (no Kafka)** | Without a durable queue, click events are lost if the app pod crashes between sending the redirect and completing the insert. No replay capability for ClickHouse maintenance windows or consumer bugs. |
| **AWS Kinesis** | Vendor lock-in. Per-shard pricing becomes expensive at high throughput. Kafka's partition model is more flexible and better understood by the team. |

### Consequences

**Positive:**
- Zero impact on redirect p99 — publishing is fire-and-forget after the response is sent.
- 7-day Kafka retention provides a replay buffer for ClickHouse failures or consumer bugs.
- Consumer can batch-insert at optimal sizes (e.g., 500 rows / 100ms) for ClickHouse efficiency.
- IP hashing, geo-lookup, and schema validation happen in the consumer — outside the redirect critical path.
- Analytics pipeline can be upgraded or replaced independently of the redirect service.

**Negative:**
- Analytics data is eventually consistent — up to ~60 seconds lag between click and queryable result. Acceptable per product requirements.
- Additional operational dependency (MSK Kafka).
- Dead-letter queue handling adds implementation complexity (see SECURITY.md §F1).

---

## ADR-007: Cloudflare over AWS CloudFront as CDN

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Infrastructure, Finance

### Context

At 1B redirects/day (30B requests/month), the CDN is a major cost and latency lever. The
decision was driven primarily by unit economics at this request volume.

### Decision

**Cloudflare** (Business or negotiated Enterprise plan) at a flat monthly fee (~$500).

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **AWS CloudFront** | CloudFront charges $0.0100 per 10,000 HTTPS requests. At 30B requests/month: ~$30,000/month in request charges alone, plus ~$1,000/month in egress. This single line item would exceed the combined cost of all compute and databases — and fails the PRD cost target of < 15% SaaS equivalent. |
| **Fastly** | Per-request + per-GB pricing model comparable to CloudFront. Enterprise pricing is negotiable but typically starts above Cloudflare for equivalent features at this scale. |
| **Self-hosted Varnish / Nginx edge** | Requires operating PoPs in multiple regions, significant infra overhead, and no global anycast network. Latency and availability would be worse than a commercial CDN for a team of this size. |

### Consequences

**Positive:**
- ~$500/month flat vs ~$31,000/month on CloudFront — a 62× cost reduction and the single largest optimisation in the entire system.
- Anycast network across 300+ PoPs; redirect latency from edge cache is typically <5ms for hot links.
- Built-in DDoS protection and WAF addresses the open WAF gap in SECURITY.md §5.
- No egress charges between Cloudflare edge and origin (unlike CloudFront).

**Negative:**
- Vendor dependency on a single CDN. Mitigated by: origin remains fully operational and traffic can be rerouted via DNS failover during a Cloudflare outage.
- Cloudflare DPA required for GDPR compliance — see PRIVACY.md §7.1.

---

## ADR-008: ClickHouse for Analytics over PostgreSQL / TimescaleDB

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering, Data Engineering

### Context

Click events are append-only, time-series in nature, and queried with aggregations
(`GROUP BY short_code, day, country_code`). At 1B events/day with 90-day retention
(~90B rows), the analytics store must handle high-throughput batch inserts and fast
analytical queries without impacting the transactional store.

### Decision

**ClickHouse** (self-hosted on EC2, replicated) as the dedicated analytics store.
PostgreSQL remains the transactional store only. ClickHouse is append-only — rows are
never updated; deleted only via TTL and GDPR erasure mutations.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **PostgreSQL (same instance or separate)** | Row-oriented storage — `GROUP BY` aggregations on 90B rows require enormous indexes and would contend with OLTP traffic. Not viable at this event volume. |
| **TimescaleDB** | Better than vanilla PostgreSQL for time-series but still row-oriented. Compression ratio ~2:1 vs ClickHouse ~10:1. At 90B rows, storage cost would be ~5× higher. |
| **BigQuery / Redshift** | Comparable query performance. Rejected due to per-query cost ($5/TB scanned on BigQuery), vendor lock-in, and data residency concerns (EU data routing to managed cloud OLAP is harder to enforce). |
| **Elasticsearch** | Optimised for full-text search, not aggregation analytics. High storage overhead. Not designed for 1B inserts/day. |
| **Storing pre-aggregates only in PostgreSQL** | Discards raw event data, making it impossible to answer new analytical questions retroactively or perform per-user analysis required for GDPR erasure. |

### Consequences

**Positive:**
- Columnar compression: 18TB raw → ~1.8TB on disk (10:1 ratio).
- Materialized views (`click_daily_mv`) pre-aggregate common queries — dashboard queries run in <100ms.
- TTL-based retention (`INTERVAL 90 DAY`) enforces GDPR data minimisation automatically.
- Complete separation of analytics from OLTP — no query contention on PostgreSQL.

**Negative:**
- ClickHouse mutations (GDPR row deletion) are asynchronous and can take minutes at scale — acceptable per PRIVACY.md §4.2.
- Self-hosted adds operational overhead. ClickHouse Cloud documented as alternative for teams without ClickHouse expertise (COST_MODEL §7).
- Append-only model means no `UPDATE` — analytics data is immutable, which is a feature (audit-safe) as much as a constraint.

---

## ADR-009: Go as Backend Language

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Engineering Lead

### Context

The redirect hot path must sustain 50,000 RPS at p99 < 10ms with minimal compute. The
write service requires safe concurrent request handling and low GC pause times. Language
choice directly affects node count and therefore infrastructure cost.

### Decision

**Go (Golang)** with Fiber or Chi HTTP framework for both the redirect service and the
write service.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **Node.js** | Single-threaded event loop; CPU-bound work (Base62 encoding, URL validation, JWT signing) blocks the loop. V8 GC pauses are less predictable than Go's concurrent GC at high throughput. ~3× higher memory per connection than Go. |
| **Python (FastAPI)** | GIL limits true parallelism for CPU-bound work. Benchmarks significantly below Go at >10k RPS. Startup time adds cold-start latency during Kubernetes pod restarts. |
| **Java (Spring Boot)** | JVM warm-up (5–15s) is problematic for Kubernetes liveness probes and horizontal autoscaling. JVM memory baseline is high (~256–512MB per pod vs ~20–50MB for Go). GC pauses add unpredictable p99 tail latency. |
| **Rust** | Comparable or superior raw performance to Go. Rejected due to steeper learning curve, longer CI compile times, and smaller hiring pool. Performance headroom over Go is not needed to hit the p99 < 10ms target. |

### Consequences

**Positive:**
- Goroutines enable tens of thousands of concurrent in-flight requests on a small node (c6g.xlarge handles ~5,500 RPS).
- Statically compiled single binary — fast Docker image builds, no runtime dependency, minimal container size.
- Built-in race detector (`go test -race`) catches concurrency bugs during development.
- `context.Context` propagation makes request cancellation and deadline enforcement idiomatic.

**Negative:**
- Verbose error handling (`if err != nil`) accepted as a team convention.
- Less expressive type system than Rust — some bug classes require runtime checks.
- Smaller analytics/ML library ecosystem vs Python — not relevant for this service.

---

## ADR-010: Soft Delete over Hard Delete for URL Records

**Date:** 2026-05-29
**Status:** Accepted
**Deciders:** Backend Engineering, Legal

### Context

When a user deletes a short URL, the system must decide whether to remove the DB row
immediately (hard delete) or mark it inactive while retaining the record (soft delete).
The short code must stop redirecting immediately in either case.

### Decision

**Soft delete**: set `urls.is_active = false` and immediately evict the Redis cache key.
The row is retained for up to 2 years post-deletion. A monthly cleanup job hard-deletes
rows older than 2 years.

### Alternatives Considered

| Option | Why rejected |
|---|---|
| **Immediate hard delete** | Destroys analytics history for the deleted URL. Forensic investigation of abuse (e.g., a phishing link deleted after being reported) becomes impossible — the `short_code` disappears as a join key for ClickHouse. |
| **Hard delete with analytics tombstone** | Requires writing a separate tombstone record to preserve `short_code` as a ClickHouse join key. Equivalent complexity to soft delete with more schema surface area. |

### Consequences

**Positive:**
- Analytics data for deleted URLs remains queryable by the owner within the retention window.
- Short code namespace is preserved — a deleted custom alias cannot be re-registered by another user, preventing brand confusion and phishing reuse.
- Forensic audit: security team can query `is_active = false` rows to investigate deleted abusive links.
- GDPR erasure is cleaner: analytics are anonymised but `short_code` is retained as a join key until hard-deleted.

**Negative:**
- DB table grows larger than active record count suggests. Mitigated by monthly hard-delete job on 2-year-old soft-deleted rows.
- All queries on active URLs must include `AND is_active = true` — enforced by linting convention and verified by integration tests.

---

## Decision Log

| ADR | Decision | Status | Date |
|---|---|---|---|
| 001 | Counter + Base62 for key generation | Accepted | 2026-05-29 |
| 002 | 302 redirect over 301 | Accepted | 2026-05-29 |
| 003 | URI path versioning over header versioning | Accepted | 2026-05-29 |
| 004 | PostgreSQL primary store with Cassandra migration path | Accepted | 2026-05-29 |
| 005 | Redis Cluster as read cache | Accepted | 2026-05-29 |
| 006 | Kafka for async analytics event streaming | Accepted | 2026-05-29 |
| 007 | Cloudflare over AWS CloudFront | Accepted | 2026-05-29 |
| 008 | ClickHouse for analytics store | Accepted | 2026-05-29 |
| 009 | Go as backend language | Accepted | 2026-05-29 |
| 010 | Soft delete over hard delete | Accepted | 2026-05-29 |

---

*Cross-references: [SYSTEM_DESIGN.md §6](SYSTEM_DESIGN.md), [LLD.md §6 §17–18](LLD.md), [API_CONTRACT.md §1](API_CONTRACT.md), [COST_MODEL.md §2.8](COST_MODEL.md), [PRIVACY.md §4](PRIVACY.md), [SECURITY.md §C3](SECURITY.md).*
