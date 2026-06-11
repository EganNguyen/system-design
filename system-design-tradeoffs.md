# System Design Trade-off Tables
> A reference guide for engineers making architectural decisions.
> Every gain costs something else — make trade-offs deliberately, explicitly, and reversibly.

---

## Table of Contents
1. [Core Performance Trade-offs](#1-core-performance-trade-offs)
2. [Data & Storage Trade-offs](#2-data--storage-trade-offs)
3. [Consistency & Availability Trade-offs](#3-consistency--availability-trade-offs)
4. [Scalability Trade-offs](#4-scalability-trade-offs)
5. [Reliability & Fault Tolerance Trade-offs](#5-reliability--fault-tolerance-trade-offs)
6. [Security & Compliance Trade-offs](#6-security--compliance-trade-offs)
7. [Architecture & Coupling Trade-offs](#7-architecture--coupling-trade-offs)
8. [Operational Trade-offs](#8-operational-trade-offs)
9. [Data Freshness & Sync Trade-offs](#9-data-freshness--sync-trade-offs)
10. [Schema Evolution Trade-offs](#10-schema-evolution-trade-offs)
11. [Disaster Recovery Trade-offs](#11-disaster-recovery-trade-offs)
12. [Fault Tolerance Techniques](#12-fault-tolerance-techniques)
13. [Master Trade-off Reference](#13-master-trade-off-reference)

---

## 1. Core Performance Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Low Latency | Throughput / Durability | Single writes, async ACK, edge caching | User-facing APIs, interactive UIs |
| High Throughput | Latency / Simplicity | Batching, pipelining, COPY protocol | Ingestion pipelines, bulk processing |
| Low Latency + High Throughput | Durability | In-memory processing, async replication | Metrics, analytics events |
| Durability | Performance | fsync, WAL, synchronous replication | Payments, ledgers, audit logs |
| Predictable Latency (p99) | Average throughput | Rate limiting, queue depth caps | SLA-bound APIs |

### Latency vs Throughput Operating Points

| Batch Config | Latency | Throughput | Use Case |
|---|---|---|---|
| No batching (1 write/req) | ~1ms | ~5k TPS | Interactive payments |
| Small batch (100 rows / 50ms) | ~50ms | ~20k TPS | Balanced ingestion |
| Large batch (2000 rows / 500ms) | ~500ms | ~80k TPS | Bulk data pipelines |

### Durability vs Throughput (PostgreSQL)

| Setting | Throughput | Data Loss Risk | Use For |
|---|---|---|---|
| `synchronous_commit = on` | ~5–10k TPS | Zero | Payment writes |
| `commit_delay` group batching | ~15–25k TPS | Zero (tiny window) | High-volume OLTP |
| `synchronous_commit = off` | ~50k TPS | Last ~200ms on crash | Outbox relay reads |
| Unlogged tables | ~200k TPS | All data on crash | Staging buffers only |

---

## 2. Data & Storage Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Fast Reads | Write speed / Storage | Indexes, caching, denormalization, CQRS | Read-heavy workloads |
| Fast Writes | Read speed / Consistency | Remove indexes, async, append-only log | Ingestion-heavy workloads |
| Storage Efficiency | CPU / Latency | Compression, normalization | Cold/archive data |
| Query Flexibility | Write performance | Normalization, relational model | Complex reporting |
| Cost Efficiency | Access latency | Tiered storage (hot/warm/cold) | Mixed-age datasets |
| Real-time Analytics | Write performance / Freshness | Columnar DB (ClickHouse), OLAP cubes | Dashboards, BI |
| Polyglot Persistence | Operational complexity | Neo4j (graphs), Elasticsearch (search), Redis (cache) | When one DB genuinely can't serve all patterns |

### Storage Tier Reference

| Tier | Medium | Cost/GB | Latency | Use Case |
|---|---|---|---|---|
| Hot | Local NVMe SSD | $$$ | μs | Active transactional data |
| Warm | Network SSD (EBS) | $$ | ms | Recent data, recent logs |
| Cool | HDD / Object storage | $ | ms–100ms | Historical queries |
| Cold | S3 / GCS | ¢ | 100ms+ | Backups, compliance |
| Archive | Glacier / Deep Archive | ¢ (tiny) | Hours | Long-term retention |

### Space vs Time Trade-offs

| Technique | Space Cost | Time Saved | Risk |
|---|---|---|---|
| Indexing | Medium (index size) | O(n) → O(log n) reads | Slower writes |
| Caching | Medium (memory) | 10–100x read speed | Stale data |
| Denormalization | High (duplicated data) | Eliminates JOINs | Write complexity |
| Precomputed aggregates | Low–Medium | O(n) → O(1) aggregations | Sync complexity |
| Materialized views | Medium | Complex queries pre-built | Refresh lag |

---

## 3. Consistency & Availability Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Strong Consistency | Availability / Latency | 2PC, Raft/Paxos, synchronous replication | Payments, inventory, ledgers |
| High Availability | Consistency | AP systems, async replication, eventual consistency | Global reads, non-critical data |
| Read-your-write Consistency | Scalability | Route reads to primary, sticky sessions | User profile updates |
| Causal Consistency | Throughput | Vector clocks, causal tracking | Social feeds, collaborative tools |
| Eventual Consistency | Immediacy | Async fan-out, background sync | Notifications, analytics |

### Consistency Level Reference

| Level | Definition | Latency Impact | Use Case |
|---|---|---|---|
| Linearizable | Every read sees latest write | Highest | Financial balances |
| Sequential | Writes ordered, reads may lag | High | Leader-based systems |
| Read-your-write | You always see your own writes | Medium | Profile updates |
| Causal | Causally related ops ordered | Medium | Comment threads |
| Eventual | All replicas converge eventually | Lowest | DNS, caching, CDN |

### Isolation Levels (PostgreSQL)

| Isolation Level | Throughput | Prevents | Use For |
|---|---|---|---|
| Read Committed | Highest | Dirty reads | Default, most reads |
| Repeatable Read | Medium | Dirty + non-repeatable reads | Reports, batch jobs |
| Serializable | ~30% slower | All anomalies | Payment debit/credit, double-spend prevention |

---

## 4. Scalability Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Horizontal Scale (writes) | Consistency / Complexity | Sharding, partitioning | > 100k TPS write load |
| Horizontal Scale (reads) | Consistency | Read replicas, CQRS | Read-heavy, analytics |
| Stateless Scale-out | State management complexity | Externalized session, JWT | API services, microservices |
| Vertical Scale | Cost ceiling | Bigger instances | Fastest path, single DB |
| Elastic Scale | Baseline cost | Autoscaling + queue buffering | Spiky / unpredictable traffic |

### Scale-up vs Scale-out

| Dimension | Scale Up (Vertical) | Scale Out (Horizontal) |
|---|---|---|
| Implementation speed | Fast (config change) | Slow (requires stateless design) |
| Consistency | Simple (single node) | Hard (distributed state) |
| Cost | Expensive at top end | Cheaper per unit at scale |
| Failure risk | SPOF | Resilient (N-1 tolerance) |
| Ceiling | Hard limit | Effectively unlimited |
| Debugging | Easy | Hard (distributed tracing needed) |

### Scalability Complexity Ladder

| Level | Architecture | Throughput | Complexity |
|---|---|---|---|
| 1 | Single DB + pgxpool | ~5k TPS | Low |
| 2 | + PgBouncer connection pool | ~15k TPS | Low |
| 3 | + Batch writes + COPY protocol | ~50k TPS | Medium |
| 4 | + Read replicas + CQRS | ~80k TPS writes | Medium |
| 5 | + Partitioning + parallel workers | ~150k TPS | High |
| 6 | + Kafka + CDC (Debezium) | ~500k+ TPS | Very High |

---

## 5. Reliability & Fault Tolerance Trade-offs

| Goal | Sacrifice | Technique | Complexity |
|---|---|---|---|
| Transient fault recovery | Latency (retry delay) | Retry with exponential backoff + jitter | Low |
| Cascading failure prevention | Latency (fail fast) | Circuit breaker (closed/open/half-open) | Medium |
| Resource exhaustion isolation | Resource utilization | Bulkhead pattern (per-service pools) | Medium |
| Silent failure detection | CPU / network | Health checks (liveness, readiness, startup) | Low |
| Hung dependency prevention | None | Timeout + context deadline propagation | Low |
| Idempotence / Exactly-Once | Latency / Storage | Idempotency keys, dedup tables, distributed locks | Medium |
| Distributed TX recovery | Complexity | Saga pattern + compensating transactions | Very High |
| Graceful degradation | UX completeness | Feature flags, circuit breakers, load shedding | Medium |

### Retry Strategy Reference

| Strategy | When to Use | Risk |
|---|---|---|
| No retry | Idempotency unknown | Data loss on transient fault |
| Immediate retry (1×) | Very brief transient faults | Thundering herd |
| Fixed delay retry | Simple systems | Synchronized retry storms |
| Exponential backoff | Standard distributed systems | Slow recovery |
| Backoff + jitter | High-concurrency systems | Best practice — spreads load |
| Dead letter queue | Unrecoverable messages | Manual intervention needed |

### Circuit Breaker States

| State | Behavior | Transition |
|---|---|---|
| Closed (normal) | All requests pass through | → Open when failure rate > threshold |
| Open (failing) | All requests rejected (fail fast) | → Half-Open after timeout |
| Half-Open (probing) | One probe request allowed | → Closed on success / Open on failure |

### Fault Tolerance Techniques

| Technique | Fault Handled | Latency Cost | Use When |
|---|---|---|---|
| Retry + backoff | Transient network faults | Added delay | All external calls |
| Circuit breaker | Cascading failures | Near-zero | Service-to-service |
| Timeout + deadline | Hung dependencies | None | Every external call |
| Bulkhead | Resource exhaustion | None | Mixed criticality workloads |
| Health checks | Silent failures | Minimal | Always |
| Idempotent writes | Duplicate requests | Small overhead | All mutating operations |
| Saga + compensation | Distributed TX failures | High | Multi-service writes |
| Chaos engineering | Unknown failure modes | None (test only) | Proactively, before incidents |

---

## 6. Security & Compliance Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Data Privacy | Latency / CPU | Encryption at rest + transit, tokenization | PII, financial data, healthcare |
| Zero-Trust Security | Latency / Complexity | mTLS, service mesh, identity-aware proxy | Microservices, regulated industries |
| Auth Security | Latency | JWT (stateless, fast) vs Session DB (revocable, slow) | Depends on revocation requirements |
| DDoS Protection | Cost | CDN + WAF + rate limiting at edge | Public-facing APIs |
| Compliance (SOC2/PCI) | Developer velocity | Audit logs, immutable trails, access controls | Regulated workloads |
| Secrets Security | Latency on fetch | Vault / KMS + in-memory caching with TTL rotation | All production secrets |

### Auth Strategy Comparison

| Strategy | Latency | Revocable | Scalability | Use Case |
|---|---|---|---|---|
| JWT (stateless) | ~0.1ms (verify signature) | No (until expiry) | Infinite | Stateless APIs, mobile |
| Session DB (Redis) | ~1ms (cache lookup) | Yes (instant) | High | Web apps needing logout |
| Session DB (Postgres) | ~5ms (DB lookup) | Yes (instant) | Medium | Small-scale, audit-required |
| mTLS (mutual TLS) | ~1ms (handshake amortized) | Yes (cert revocation) | High | Service-to-service |

---

## 7. Architecture & Coupling Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Service Isolation | Latency / Observability | Async messaging, event-driven architecture | Independent scaling, different failure domains |
| Strong Contracts | Flexibility | gRPC + Protobuf, schema registry | Stable APIs, cross-team boundaries |
| Fast Iteration | Consistency | REST + loose JSON contracts | Early stage, single team |
| Full Decoupling | Traceability (harder to trace) | Event streaming (Kafka), message queues | Large orgs, many consumers |
| Developer Velocity | System optimization | Microservices, managed services, serverless | Early stage, small teams |
| Monolith Simplicity | Independent deployability | Single deployable unit | Early stage — always start here |

### Coupling Spectrum

| Method | Latency | Coupling | Observability | Durability |
|---|---|---|---|---|
| Direct function call | < 0.01ms | Tight | Easy | N/A (in-process) |
| HTTP / REST | ~1ms | Loose | Standard | None (fire-and-forget risk) |
| gRPC | ~0.5ms | Loose + typed | Standard | None |
| Message queue (RabbitMQ) | ~5ms | Very loose | Harder | At-least-once |
| Event streaming (Kafka) | ~10ms | Fully decoupled | Hardest | Durable, replayable |

### Sync vs Async Decision

| Operation | Use Sync | Use Async |
|---|---|---|
| Payment debit / credit | ✅ | ❌ |
| Fraud check (before ACK) | ✅ | ❌ |
| Inventory reservation | ✅ | ❌ |
| Email / SMS notification | ❌ | ✅ |
| Search index update | ❌ | ✅ |
| Analytics event write | ❌ | ✅ |
| Cache invalidation | ❌ | ✅ |
| Audit log write | ❌ | ✅ |
| Warehouse notification | ❌ | ✅ |

---

## 8. Operational Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Full Observability | CPU / Storage / Latency | Structured logs + metrics + distributed tracing | Production systems |
| Low Monitoring Overhead | Visibility granularity | Sampling (1% traces), aggregated metrics | Very high-volume systems |
| Fast Incident Response | Storage cost | High-res short retention + low-res long retention | All production systems |
| Cost Reduction | Performance headroom | Right-sizing, spot instances, reserved capacity | Post-product-market-fit |
| Burst Handling | Baseline cost | Autoscaling + queue-based load leveling | Spiky traffic patterns |
| Simple Operations | Raw performance | Fewer components, managed services | Small teams |

### Observability Cost vs Value

| Signal | Collection Cost | Debug Value | Recommended Approach |
|---|---|---|---|
| Structured logs | Medium (disk I/O) | High | Always — with log levels |
| Metrics (Prometheus) | Low–Medium | High | Always — low cardinality only |
| Distributed traces | High (~5–10% latency) | Very High | Sample at 1–10% in prod |
| Continuous profiling | Medium (CPU) | High for perf | Sampling-based (pprof) |
| Synthetic monitoring | Low | Medium | Key user journeys only |

---

## 9. Data Freshness & Sync Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Real-time push (server→client) | Server connection count | Server-Sent Events (SSE) | Live dashboards, notifications |
| Bidirectional real-time | Complexity | WebSocket | Chat, collaborative editing, gaming |
| Simplicity over freshness | Freshness | Short polling | Low-frequency updates, simple clients |
| Reduced server load | Freshness | Long polling | Moderate frequency, simple infra |
| Offline-first | Sync complexity | Local-first + background sync | Mobile apps, poor connectivity |

### Real-time Technology Comparison

| Method | Latency | Server Load | Complexity | Direction | Use Case |
|---|---|---|---|---|---|
| Short polling (5s) | 0–5s | Very High ❌ | Low | Client→Server | Simple status checks |
| Long polling | 0–30s | High | Medium | Client→Server | Notifications, low freq |
| SSE | ~ms | Low | Low | Server→Client only | Dashboards, live feeds |
| WebSocket | ~ms | Low | High | Bidirectional | Chat, live collaboration |
| WebTransport | ~ms | Low | Very High | Bidirectional | Future — QUIC-based |

---

## 10. Schema Evolution Trade-offs

| Goal | Sacrifice | Technique | When to Use |
|---|---|---|---|
| Zero-downtime migrations | Deployment complexity | Expand/Contract pattern (multi-phase) | Large tables in production |
| Flexible schema | Validation / Auditability | NoSQL / schemaless (MongoDB, DynamoDB) | Rapidly changing domains |
| Strong schema contracts | Flexibility | Schema registry (Avro / Protobuf) | Event streams, cross-team APIs |
| Fast iteration | Schema discipline | JSON columns in PostgreSQL | Prototype / semi-structured data |

### Expand / Contract Migration Phases

| Phase | Action | Duration | Risk |
|---|---|---|---|
| 1 — Expand | Add new nullable column, dual-write old + new | Days–weeks | Zero (backward compatible) |
| 2 — Backfill | Migrate existing rows in small batches | Days–weeks | Low (no table lock) |
| 3 — Contract | Read from new column only, drop old column | Deploy cycle | Low (verify first) |

### Schema Compatibility Modes

| Mode | Rule | Allows | Safe For |
|---|---|---|---|
| Backward | New schema reads old data | Add optional fields | Consumer upgrades before producer |
| Forward | Old schema reads new data | Remove optional fields | Producer upgrades before consumer |
| Full | Both directions | Only additive, non-breaking | Maximum safety, strictest |
| None | No compatibility check | Anything | Development only |

---

## 11. Disaster Recovery Trade-offs

| Goal | Sacrifice | Technique | RPO | RTO | Cost |
|---|---|---|---|---|---|
| Basic recovery | Downtime tolerance | Backup to S3 + restore | Hours | Hours | $ |
| Reduced downtime | Complexity | Pilot Light (minimal standby) | Seconds | 10–30 min | $$ |
| Low downtime | Cost | Warm Standby (replica always running) | Near-zero | 1–5 min | $$$ |
| Near-zero downtime | Cost + Complexity | Active-Active multi-region | ~0 | ~0 | $$$$ |

### DR Strategy Detail

| Strategy | Description | Hard Problems | Use When |
|---|---|---|---|
| Backup / Restore | Periodic dump to object storage | Long restore time | Dev/staging, non-critical |
| Pilot Light | Minimal standby, stopped until needed | Startup time on failover | Moderate criticality |
| Warm Standby | Running replica, takes read traffic | DNS propagation delay | Most production payment systems |
| Active-Active | Both regions serve traffic simultaneously | Conflict resolution, write latency | Global, revenue-critical systems |

---

## 12. Fault Tolerance Techniques

### Prevention

| Technique | Protects Against | Cost |
|---|---|---|
| Active-passive replication | Node crash | Latency + infra cost |
| Active-active replication | Node + region failure | 2x infra + conflict resolution |
| N+1 redundancy | Single node failure | 1/N extra infrastructure |
| Bulkhead (per-service pools) | Resource exhaustion cascade | Idle resource overhead |
| Rate limiting | Overload, abuse | Redis round trip per request |

### Detection

| Technique | Detects | Overhead | Implementation |
|---|---|---|---|
| Liveness check (`/healthz`) | Process alive | Minimal | Always — restart on failure |
| Readiness check (`/readyz`) | Ready for traffic | Minimal | Always — gate LB inclusion |
| Startup check (`/startupz`) | Finished initializing | Minimal | Slow-starting services |
| Timeout + context deadline | Hung dependencies | None | Every external call |
| SLO alerting (error rate, latency) | Degraded service | Medium | Production systems |

### Isolation

| Technique | Isolates | Latency Impact | Complexity |
|---|---|---|---|
| Circuit breaker | Failing downstream services | Near-zero (fail fast) | Medium |
| Bulkhead | Slow consumer exhausting shared pool | None | Medium |
| Retry with backoff + jitter | Thundering herd on retry | Added delay | Low |
| Queue-based load leveling | Traffic spikes | Added latency | Medium |
| Timeout hierarchy | Cascading slow calls | None | Low |

### Recovery

| Technique | Recovers From | Automation Level | Complexity |
|---|---|---|---|
| Auto-restart (k8s) | Process crash | Full | Low |
| Automated DB failover (Patroni) | Primary DB failure | Full | Medium |
| Saga + compensating TX | Partial distributed TX | Full | Very High |
| Manual failover + DNS update | Region failure | Partial | Medium |
| Blue/green deployment | Bad deployment | Full | Medium |

---

## 13. Master Trade-off Reference

| Goal | Sacrifice | Technique | Complexity |
|---|---|---|---|
| **Low Latency** | Throughput | Async ACK, edge cache, connection pool | Low |
| **High Throughput** | Latency | Batching, COPY protocol, pipeline | Medium |
| **Durability** | Performance | fsync, WAL, sync replication | Low |
| **Strong Consistency** | Availability / Latency | 2PC, Raft/Paxos, serializable isolation | High |
| **High Availability** | Consistency | AP systems, async replication | Medium |
| **Idempotence / Exactly-Once** | Latency / Storage | Idempotency keys, dedup tables, locks | Medium |
| **Graceful Degradation** | UX completeness | Feature flags, circuit breakers, load shedding | Medium |
| **Data Freshness** | Server load / Bandwidth | WebSocket, SSE, long polling | Medium |
| **Schema Evolution** | Perf / Simplicity | Expand-contract, schema registry | High |
| **Disaster Recovery** | Infrastructure cost | Active-active, warm standby, pilot light | Very High |
| **Fault Tolerance** | Cost / Complexity | Circuit breakers, bulkheads, retries, chaos engineering | Medium–High |
| **Horizontal Scale** | Consistency | Stateless services, sharding, partitioning | High |
| **Fast Reads** | Write speed | Indexes, cache, CQRS, denormalize | Medium |
| **Fast Writes** | Read speed | Remove indexes, async, append-only log | Medium |
| **Security / Privacy** | Latency / CPU | Encryption, zero-trust, tokenization | Medium |
| **Observability** | CPU / Storage | Metrics, tracing, structured logs | Medium |
| **Cost Efficiency** | Performance | Tiered storage, spot instances, caching | Low |
| **Developer Velocity** | Optimization | Managed services, monolith-first | Low |
| **Loose Coupling** | Latency / Traceability | Event-driven, message queues | Medium |
| **Real-time Analytics** | Write perf / Freshness | Columnar DB, Lambda/Kappa architecture | High |
| **Polyglot Persistence** | Ops complexity | Specialized DBs per access pattern | Very High |
| **Compliance / Audit** | Velocity / Storage | Immutable logs, access controls | Medium |

---

## The Principal Engineer Decision Framework

```
When facing any architectural decision, ask in order:

1. MEASURE FIRST
   Where is actual time/cost spent? Don't optimize what isn't slow.

2. IS IT NECESSARY?
   Do you need 100k TPS today or does 10k serve current users?

3. MAKE THE TRADE-OFF EXPLICIT
   "We accept eventual consistency on reads to get 5x write throughput."
   Document it. Your future self will thank you.

4. PLAN REVERSIBILITY
   "We can undo this when we hit scale X by doing Y."

5. SET A MEASURABLE TRIGGER
   "We revisit this when metric M exceeds threshold T."
```

### Scale-Stage Guidance

| Stage | Priority | Architecture | Avoid |
|---|---|---|---|
| Day 1 | Correctness + velocity | Monolith, single DB, no cache | Premature distribution |
| Scale 1 | Reliability | Add replicas, circuit breakers, monitoring | Sharding too early |
| Scale 2 | Throughput | Caching, async, read replicas | Custom infrastructure |
| Scale 3 | Cost | Tiered storage, right-sizing, CDN | Over-engineering |
| Scale 4+ | Specific bottlenecks | Sharding, Kafka, custom solutions | Solving tomorrow's problems today |

> **Core Principle:** The best architects don't make perfect decisions.
> They make decisions that are **easy to change** when the context changes.

---

*Last updated: June 2026 | Based on production experience with Go + PostgreSQL + distributed systems*
