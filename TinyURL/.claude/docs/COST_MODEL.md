# Cost Model: TinyURL Service

> Monthly infrastructure cost estimate at target scale.
> All prices are AWS US-East-1 as of mid-2026. On-demand rates shown; reserved
> and Spot discounts addressed in §7.
> Figures are estimates — actual costs depend on traffic shape, reserved instance
> coverage, and enterprise discount agreements. Re-validate before provisioning.

---

## 1. Target Scale (from PRD + SYSTEM_DESIGN)

| Metric | Value |
|---|---|
| Daily redirects | 1B / day |
| Average redirect RPS | ~11,600 RPS |
| Peak redirect RPS | 50,000 RPS |
| URLs stored | 100M |
| Average URL record size | ~710 bytes |
| Redis hot set | ~14 GB (top 20% URLs → 80% traffic) |
| Cache hit target | ≥ 95% |
| Click events retained | 90 days |
| Write rate (shorten) | ~100–500 URLs/sec peak |

---

## 2. Infrastructure Bill of Materials

### 2.1 Compute — Read Service (Go, EKS)

The redirect path is the dominant cost driver. Each `c6g.xlarge` (4 vCPU, 8 GB) handles
~5,500 RPS at 95% cache-hit rate (near-pure Redis GET + 302 response). Peak 50k RPS
requires ~9.1 nodes; add 20% headroom.

| Config | Count | On-demand / mo | 1-yr reserved / mo |
|---|---|---|---|
| c6g.xlarge EKS worker nodes | **11** | $1,065 | $570 |

### 2.2 Compute — Write Service (Go, EKS)

Write volume is ~100× lower than reads. Two small nodes handle peak write load.

| Config | Count | On-demand / mo | 1-yr reserved / mo |
|---|---|---|---|
| c6g.large (2 vCPU, 4 GB) EKS worker nodes | **2** | $98 | $52 |

### 2.3 Kubernetes Control Plane (EKS)

| Item | Cost / mo |
|---|---|
| EKS cluster fee | $72 |

### 2.4 Database — PostgreSQL (RDS)

Primary (Multi-AZ for HA) + 2 read replicas. At 95% cache-hit rate, replicas only serve
the cache-miss tail (~580 RPS average). `db.r6g.xlarge` is sufficient.

| Component | Config | On-demand / mo | 1-yr reserved / mo |
|---|---|---|---|
| Primary (Multi-AZ) | db.r6g.xlarge (4 vCPU, 32 GB) | $700 | $390 |
| Read replica × 2 | db.r6g.large (2 vCPU, 16 GB) each | $290 | $160 |
| Storage | 200 GB gp3 (data + indexes) | $23 | $23 |
| **DB subtotal** | | **$1,013** | **$573** |

### 2.5 Cache — ElastiCache Redis (Cluster Mode)

3 primary shards + 3 replicas = 6 nodes. Each `cache.r6g.large` (2 vCPU, 13 GB) handles
~100k simple GET ops/sec — comfortably above the 11,600 average RPS spread across shards.
Sized for the 14 GB hot set with key metadata overhead.

| Config | Count | On-demand / mo | 1-yr reserved / mo |
|---|---|---|---|
| cache.r6g.large nodes | **6** | $716 | $437 |

### 2.6 Message Queue — MSK Kafka

3 brokers, replication factor 3. 1B clicks/day × ~120 bytes compressed ≈ 120 GB/day
ingested; 7-day replay window = ~840 GB across all replicas.

| Component | Config | Cost / mo |
|---|---|---|
| Brokers | kafka.m5.large × 3 | $208 |
| EBS storage | ~1 TB × 3 replicas (gp2) | $300 |
| **MSK subtotal** | | **$508** |

### 2.7 Analytics — ClickHouse (self-hosted on EC2)

90-day retention at 1B clicks/day. Raw data: ~90B rows × 200 bytes = 18 TB.
ClickHouse columnar compression (~10:1) reduces on-disk footprint to ~1.8 TB.

| Component | Config | On-demand / mo | 1-yr reserved / mo |
|---|---|---|---|
| Nodes | r6g.xlarge (4 vCPU, 32 GB) × 2 | $292 | $155 |
| EBS gp3 storage | 2 TB (compressed + 10% headroom) | $160 | $160 |
| **ClickHouse subtotal** | | **$452** | **$315** |

### 2.8 CDN

At 1B redirects/day, CDN choice dominates the bill if AWS CloudFront is used.

| Option | Monthly cost | Notes |
|---|---|---|
| **Cloudflare Business/Enterprise** | **~$500** | Flat fee, unlimited requests and bandwidth. Strongly preferred at this scale. |
| AWS CloudFront | ~$28,000–32,000 | $0.0100 per 10k HTTPS requests × 30B/month = ~$30,000, plus egress. Prohibitively expensive. |

**Decision: Cloudflare.** CloudFront is cost-effective at low-to-medium volumes but scales
linearly with request count — at 1B/day it becomes the single largest line item by a wide
margin. This model uses **$500/month** for Cloudflare (conservative; enterprise plans
negotiate lower flat rates).

### 2.9 Load Balancer (ALB)

With HTTP/2 keep-alive, new connection rate is far below total RPS. Approximately 3–5 LCUs active.

| Item | Cost / mo |
|---|---|
| ALB fixed charge | $16 |
| LCU charges (~4 LCU × $0.008/hr) | $23 |
| **ALB subtotal** | **$39** |

### 2.10 Monitoring & Observability

Self-hosted Prometheus + Grafana on a single `t3.medium`; CloudWatch for ALB and RDS only.

| Item | Cost / mo |
|---|---|
| t3.medium (Prometheus + Grafana) | $30 |
| CloudWatch metrics + logs | $60 |
| **Monitoring subtotal** | **$90** |

### 2.11 Networking

| Item | Cost / mo |
|---|---|
| NAT Gateway × 2 AZs | $65 |
| Cross-AZ data transfer (~500 GB) | $50 |
| **Networking subtotal** | **$115** |

---

## 3. Total Monthly Cost

| Component | On-demand / mo | 1-yr reserved / mo |
|---|---|---|
| Read Service (EKS nodes) | $1,065 | $570 |
| Write Service (EKS nodes) | $98 | $52 |
| EKS control plane | $72 | $72 |
| PostgreSQL (primary + replicas + storage) | $1,013 | $573 |
| Redis ElastiCache (6 nodes) | $716 | $437 |
| MSK Kafka (brokers + storage) | $508 | $508 |
| ClickHouse (nodes + storage) | $452 | $315 |
| CDN (Cloudflare) | $500 | $500 |
| Load Balancer (ALB) | $39 | $39 |
| Monitoring | $90 | $90 |
| Networking | $115 | $115 |
| **Total** | **$4,668 / mo** | **$3,271 / mo** |

> Reserved instances applied to: EKS compute, RDS, ElastiCache, ClickHouse EC2.
> MSK, CDN, monitoring, and networking have no meaningful reserved pricing benefit.

---

## 4. Sensitivity Table

Cost at 10%, 50%, and 100% of target traffic.
Redis and Kafka have minimum viable configurations that don't shrink linearly at low tiers.

| Traffic tier | Redirects / day | Avg RPS | Read nodes | Redis nodes | Est. cost / mo (reserved) |
|---|---|---|---|---|---|
| **10%** — early / dev | 100M | 1,160 | 2 × c6g.large | 3 × cache.r6g.medium | **~$1,400** |
| **50%** — growth | 500M | 5,800 | 5 × c6g.xlarge | 6 × cache.r6g.large | **~$2,200** |
| **100%** — target | 1B | 11,600 | 11 × c6g.xlarge | 6 × cache.r6g.large | **~$3,271** |
| **200%** — headroom | 2B | 23,200 | 20 × c6g.xlarge | 9 × cache.r6g.xlarge | **~$5,800** |

**Components that do not scale linearly:**
- **Kafka storage** — driven by click event byte volume; 2× traffic ≈ 2× storage cost.
- **ClickHouse storage** — same: scales with event count, not redirects.
- **CDN** — Cloudflare flat fee stays constant across all tiers (up to contract limits).
- **PostgreSQL** — nearly flat; replicas only serve the cache-miss tail.

---

## 5. SaaS Comparison — PRD Validation

PRD success criterion: **infrastructure cost < 15% of equivalent enterprise SaaS pricing.**

### Bitly Enterprise Pricing at 1B Daily Redirects

Bitly's published tiers top out at ~$200/month for 1M clicks. At 30B clicks/month,
pricing becomes a custom enterprise contract. Reported rates for comparable volumes
range from **$50,000 to $150,000/month**, depending on API integration depth, SLA,
and branded domain count.

| Vendor | Est. cost / mo at 30B redirects | Basis |
|---|---|---|
| Bitly Enterprise | $50,000 – $150,000 | Custom contract (reported) |
| Rebrandly Enterprise | $40,000 – $100,000 | Custom contract (reported) |
| Short.io Enterprise | $30,000 – $80,000 | Custom contract (reported) |

**Using the conservative Bitly floor ($50,000/month):**

| | Cost / mo | % of Bitly at $50k |
|---|---|---|
| Our infra — on-demand | $4,668 | **9.3%** ✅ |
| Our infra — 1-yr reserved | $3,271 | **6.5%** ✅ |
| Our infra — reserved + Spot (§7) | ~$2,700 | **5.4%** ✅ |
| PRD target | — | **< 15%** |

**The PRD target is met under all three pricing models**, even against the most conservative
SaaS baseline. At a $100k/month SaaS baseline the ratio drops to 2.7–4.7%.

---

## 6. Unit Economics

Cost per 1 million redirects — the most comparable metric against SaaS per-unit pricing.

| Pricing model | Total / mo | Redirects / mo | Cost per 1M redirects |
|---|---|---|---|
| On-demand | $4,668 | 30B | **$0.156** |
| 1-yr reserved | $3,271 | 30B | **$0.109** |
| Reserved + Spot read nodes | ~$2,700 | 30B | **$0.090** |
| Bitly Enterprise (est. $50k) | $50,000 | 30B | **$1.67** |
| Bitly Enterprise (est. $100k) | $100,000 | 30B | **$3.33** |

Self-hosted cost is **10–30× cheaper per million redirects** than enterprise SaaS at this volume.

---

## 7. Cost Optimisation Levers

Ordered by expected monthly savings impact:

| Lever | Est. saving | Notes |
|---|---|---|
| **1-yr reserved instances** (compute + RDS + Redis) | ~$1,400/mo | Commit once traffic reaches 50% of target and shape is predictable |
| **Spot instances for read service** | ~$400/mo additional | Read pods are stateless and restartable; K8s PodDisruptionBudget keeps ≥70% running; mix 30% on-demand + 70% Spot |
| **Compute Savings Plans** | Up to 66% off EC2 | More flexible than reserved; apply to EKS nodes across instance families |
| **Graviton (c6g / r6g already selected)** | Already baked in | 20% cheaper + up to 40% better perf than x86 equivalents |
| **Kafka self-hosted on EC2** (vs MSK) | ~$120/mo on brokers | Higher ops burden; viable once team has operational Kafka expertise |
| **ClickHouse Cloud** (vs self-hosted) | +$100/mo premium | Pays for itself in ops time at early stage; revisit when ClickHouse team matures |
| **Redis memory tuning** | 10–20% node reduction | `maxmemory-policy allkeys-lfu`; compress values; remove expired keys proactively |
| **CDN cache TTL increase** | Indirect | Higher CDN hit rate → lower origin RPS → smaller compute fleet required |

### Spot Strategy for Read Service

The read service is the largest single compute cost line. Spot is the highest-ROI lever:

```
Fleet composition at 100% traffic:
  3 × c6g.xlarge on-demand   (baseline, always-on)   → $156/mo
  8 × c6g.xlarge Spot        (burst, ~$0.014/hr ea)  → $81/mo

  vs. 11 × c6g.xlarge reserved                       → $570/mo

  Saving: ~$333/mo on compute alone
```

---

## 8. Open Cost Risks

| Risk | Potential impact | Mitigation |
|---|---|---|
| Kafka storage growth underestimated | MSK storage is $0.10/GB; larger-than-expected message sizes double cost | Monitor `kafka.log.size`; enforce message compression (snappy/lz4); tune schema |
| ClickHouse compression ratio < 10:1 | Could 2–3× storage cost to $400–500/mo | Benchmark compression on real click event schema before provisioning |
| CDN burst above Cloudflare contract | Overage fees or service throttling | Negotiate hard cap + overage rate in contract; monitor monthly request count |
| Multi-region active-active deployment | 2–3× infra cost ($6,500–10,000/mo) | Resolve deployment topology open question (PRD open question #2) before budgeting multi-region |
| PostgreSQL → Cassandra migration at scale | 1-month dual-running overlap + migration labour | Budget transition costs separately; plan at 500M URL threshold (SYSTEM_DESIGN §8) |

---

*Informed by: [SYSTEM_DESIGN.md §3 §9](SYSTEM_DESIGN.md), [LLD.md §17–18](LLD.md), [PRD.md §Hypothesis §Success Metrics](PRD.md), [DATABASE_DESIGN.md §8](DATABASE_DESIGN.md).*
*Prices: AWS US-East-1, mid-2026. Re-validate against current pricing before provisioning.*
