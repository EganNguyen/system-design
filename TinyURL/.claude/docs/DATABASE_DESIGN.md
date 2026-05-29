# Database Design: TinyURL Service

---

## 1. Overview

This document covers the complete database design for TinyURL — schema definitions, indexing strategy, partitioning, access patterns, query examples, and migration approach. It is database-agnostic at the schema level, with driver-specific notes where behavior diverges.

**Source of truth for all persisted state.** Redis is a cache layer only; ClickHouse is append-only analytics. This document covers both the primary transactional store and the analytics store.

---

## 2. Database Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                     Transactional Store                         │
│                                                                 │
│   Primary DB  ──sync replication──▶  Replica 1 (same region)   │
│       │                             Replica 2 (read traffic)    │
│       │                             Replica 3 (cross-region)    │
│       │                                                         │
│   Failover: automatic (RDS Multi-AZ / Patroni / CockroachDB)    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Analytics Store                             │
│                                                                 │
│   ClickHouse Cluster  (shard 1, shard 2, shard 3)              │
│   Replica per shard — ReplicatedMergeTree engine               │
│   Write path: Kafka consumer → ClickHouse INSERT                │
│   Read path:  Dashboard queries → ClickHouse SELECT             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Entity Relationship Diagram

```
┌──────────────┐         ┌──────────────────────┐         ┌──────────────────┐
│    users     │         │        urls           │         │   click_events   │
│──────────────│         │──────────────────────│         │──────────────────│
│ user_id (PK) │◀──────┐ │ short_code (PK)       │◀──────┐ │ event_id (PK)    │
│ email        │       └─│ user_id (FK, nullable)│       └─│ short_code (FK)  │
│ password_hash│         │ long_url              │         │ clicked_at       │
│ created_at   │         │ created_at            │         │ country_code     │
│ plan         │         │ expires_at            │         │ referrer         │
│ is_active    │         │ is_custom             │         │ ip_hash          │
└──────────────┘         │ is_active             │         │ user_agent       │
                         │ click_count           │         └──────────────────┘
                         └──────────────────────┘
                                  │
                         ┌────────▼─────────┐
                         │   custom_domains  │
                         │─────────────────│
                         │ domain_id (PK)   │
                         │ user_id (FK)     │
                         │ domain           │
                         │ verified_at      │
                         │ is_active        │
                         └──────────────────┘
```

---

## 4. Schema Definitions

### 4.1 `users`

```sql
CREATE TABLE users (
    user_id         UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255)    NOT NULL,
    password_hash   VARCHAR(255)    NOT NULL,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    last_login_at   TIMESTAMPTZ,
    plan            VARCHAR(20)     NOT NULL DEFAULT 'free',
    is_active       BOOLEAN         NOT NULL DEFAULT TRUE,
    is_verified     BOOLEAN         NOT NULL DEFAULT FALSE,

    CONSTRAINT users_email_unique   UNIQUE (email),
    CONSTRAINT users_plan_check     CHECK (plan IN ('free', 'pro', 'enterprise'))
);

CREATE INDEX idx_users_email      ON users (email);
CREATE INDEX idx_users_created_at ON users (created_at);
```

**Column notes:**

| Column          | Notes                                                         |
|-----------------|---------------------------------------------------------------|
| `user_id`       | UUIDv4; never expose raw sequential IDs externally           |
| `password_hash` | bcrypt cost ≥ 12; never store plaintext                      |
| `plan`          | Controls rate limits and feature access in application layer  |
| `is_active`     | Soft-disable accounts without data loss                       |

---

### 4.2 `urls` (Core Table)

```sql
CREATE TABLE urls (
    short_code      VARCHAR(8)      PRIMARY KEY,
    long_url        TEXT            NOT NULL,
    user_id         UUID            REFERENCES users(user_id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,
    is_custom       BOOLEAN         NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN         NOT NULL DEFAULT TRUE,
    click_count     BIGINT          NOT NULL DEFAULT 0,

    CONSTRAINT urls_long_url_length CHECK (char_length(long_url) <= 2048),
    CONSTRAINT urls_short_code_fmt  CHECK (short_code ~ '^[0-9a-zA-Z_-]+$')
);

-- Primary access pattern: point lookup by short_code (covered by PK)

-- Secondary access patterns:
CREATE INDEX idx_urls_user_id
    ON urls (user_id, created_at DESC)
    WHERE user_id IS NOT NULL;

CREATE INDEX idx_urls_expires_at
    ON urls (expires_at)
    WHERE expires_at IS NOT NULL AND is_active = TRUE;

CREATE INDEX idx_urls_long_url
    ON urls USING hash (long_url);    -- dedup check: has this long_url been shortened before?
```

**Column notes:**

| Column        | Notes                                                                      |
|---------------|----------------------------------------------------------------------------|
| `short_code`  | 7-char Base62; PK enforces uniqueness, enables O(1) lookup                |
| `long_url`    | TEXT (up to 2048); not indexed for range queries — hash index for dedup   |
| `user_id`     | Nullable — anonymous users can shorten without an account                 |
| `expires_at`  | NULL = never expires; partial index keeps expiry scans fast               |
| `click_count` | Denormalized counter; incremented asynchronously via analytics worker     |
| `is_active`   | Soft delete; inactive codes return 410 Gone, never 404                    |

---

### 4.3 `custom_domains`

```sql
CREATE TABLE custom_domains (
    domain_id       UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID            NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    domain          VARCHAR(253)    NOT NULL,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    verified_at     TIMESTAMPTZ,
    is_active       BOOLEAN         NOT NULL DEFAULT FALSE,

    CONSTRAINT custom_domains_domain_unique UNIQUE (domain),
    CONSTRAINT custom_domains_domain_fmt
        CHECK (domain ~ '^([a-zA-Z0-9]([a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$')
);

CREATE INDEX idx_custom_domains_user_id ON custom_domains (user_id);
CREATE INDEX idx_custom_domains_domain  ON custom_domains (domain) WHERE is_active = TRUE;
```

---

### 4.4 `click_events` (ClickHouse — Analytics Store)

```sql
CREATE TABLE click_events
(
    event_id        UUID            DEFAULT generateUUIDv4(),
    short_code      LowCardinality(String),
    clicked_at      DateTime64(3, 'UTC'),
    country_code    LowCardinality(FixedString(2)),
    city            String,
    referrer        String,
    referrer_domain LowCardinality(String),
    device_type     LowCardinality(String),   -- desktop | mobile | tablet | bot
    browser         LowCardinality(String),
    os              LowCardinality(String),
    ip_hash         FixedString(64)           -- SHA-256 hex, PII-safe
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/click_events', '{replica}')
PARTITION BY toYYYYMM(clicked_at)
ORDER BY (short_code, clicked_at)
TTL clicked_at + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

**Design notes:**

- `LowCardinality` on repeated string columns (country, device, browser) enables dictionary encoding — significant compression and faster GROUP BY
- Partitioned by month → old data dropped cheaply, recent data queried efficiently
- Ordered by `(short_code, clicked_at)` → primary key for ClickHouse; range scans per code are fast
- TTL 90 days — data older than 90 days is automatically deleted (aligned with GDPR data minimisation; see PRIVACY.md §5)

**Materialized views for common aggregations:**

```sql
-- Daily rollup (pre-aggregated, queried by dashboard)
CREATE MATERIALIZED VIEW click_daily_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (short_code, day, country_code, device_type)
POPULATE AS
SELECT
    short_code,
    toDate(clicked_at)      AS day,
    country_code,
    device_type,
    count()                 AS clicks
FROM click_events
GROUP BY short_code, day, country_code, device_type;
```

---

### 4.5 `blacklist` (Transactional Store)

```sql
CREATE TABLE blacklist (
    entry_id        UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    domain          VARCHAR(253)    NOT NULL,
    reason          VARCHAR(50)     NOT NULL,   -- phishing | malware | spam | manual
    added_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    added_by        UUID            REFERENCES users(user_id),
    expires_at      TIMESTAMPTZ,

    CONSTRAINT blacklist_domain_unique UNIQUE (domain)
);

CREATE INDEX idx_blacklist_domain ON blacklist USING hash (domain);
```

*This table is small; its contents are bulk-loaded into Redis on startup and refreshed every 5 minutes.*

---

## 5. Access Patterns & Query Catalog

### 5.1 Redirect (Hot Path — p99 < 10ms)

```sql
-- Called on every cache miss; must be sub-millisecond
SELECT long_url, expires_at, is_active
FROM   urls
WHERE  short_code = $1;
```

**Index used:** Primary key on `short_code` — O(1) B-tree lookup.

---

### 5.2 Create Short URL

```sql
INSERT INTO urls (short_code, long_url, user_id, expires_at, is_custom)
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT (short_code) DO NOTHING
RETURNING short_code, created_at;
```

`ON CONFLICT DO NOTHING` handles the rare case of a counter pre-fetch race. The application retries with the next counter value if 0 rows are returned.

---

### 5.3 Dedup Check (Has This Long URL Been Shortened?)

```sql
-- Optional: return existing short code if same long_url already exists for this user
SELECT short_code
FROM   urls
WHERE  long_url = $1
  AND  user_id  = $2
  AND  is_active = TRUE
LIMIT  1;
```

**Index used:** Hash index on `long_url` for equality lookup.

---

### 5.4 List User's URLs (Dashboard)

```sql
SELECT short_code, long_url, created_at, expires_at, click_count, is_active
FROM   urls
WHERE  user_id = $1
ORDER  BY created_at DESC
LIMIT  $2 OFFSET $3;
```

**Index used:** `idx_urls_user_id (user_id, created_at DESC)` — covers the query entirely (index-only scan).

---

### 5.5 Soft Delete URL

```sql
UPDATE urls
SET    is_active  = FALSE,
       updated_at = NOW()
WHERE  short_code = $1
  AND  user_id    = $2   -- ownership check
RETURNING short_code;
```

---

### 5.6 Expiry Cleanup (Nightly Cron)

```sql
-- Runs nightly at 02:00 UTC; batched to avoid table lock
DELETE FROM urls
WHERE  expires_at < NOW()
  AND  expires_at IS NOT NULL
  AND  is_active  = TRUE
LIMIT  10000;   -- repeat until 0 rows deleted
```

**Index used:** `idx_urls_expires_at` partial index — only scans rows with expiry set.

---

### 5.7 Increment Click Count (Async, Batched)

```sql
-- Called by analytics worker every 30s, not on every redirect
UPDATE urls
SET    click_count = click_count + $2,
       updated_at  = NOW()
WHERE  short_code  = $1;
```

Batched increments via a Go map: accumulate counts in memory for 30 seconds, then flush a single UPDATE per code. Avoids write amplification on hot URLs.

---

### 5.8 Analytics Queries (ClickHouse)

**Clicks per day for a short code:**
```sql
SELECT day, sum(clicks) AS total
FROM   click_daily_mv
WHERE  short_code = 'aB3xYz7'
  AND  day BETWEEN '2026-05-01' AND '2026-05-29'
GROUP BY day
ORDER BY day;
```

**Top countries for a short code:**
```sql
SELECT country_code, sum(clicks) AS total
FROM   click_daily_mv
WHERE  short_code = 'aB3xYz7'
GROUP BY country_code
ORDER BY total DESC
LIMIT  10;
```

**Unique visitors estimate (HyperLogLog):**
```sql
SELECT uniqHLL12(ip_hash) AS approx_unique_visitors
FROM   click_events
WHERE  short_code = 'aB3xYz7'
  AND  clicked_at >= now() - INTERVAL 30 DAY;
```

---

## 6. Indexing Strategy

### Transactional Store

| Table            | Index                                | Type    | Purpose                                    |
|------------------|--------------------------------------|---------|--------------------------------------------|
| `urls`           | `short_code` (PK)                    | B-tree  | Redirect lookup — hot path                 |
| `urls`           | `(user_id, created_at DESC)`         | B-tree  | Dashboard list queries                     |
| `urls`           | `expires_at` (partial)               | B-tree  | Expiry cleanup cron                        |
| `urls`           | `long_url`                           | Hash    | Dedup check on creation                    |
| `users`          | `email` (unique)                     | B-tree  | Login lookup                               |
| `blacklist`      | `domain`                             | Hash    | Domain check on URL submission             |
| `custom_domains` | `domain` (partial: is_active=true)   | B-tree  | Domain resolution on redirect              |

### Avoid

- Index on `long_url` with B-tree — TEXT columns with URLs are wide; hash index is sufficient for equality
- Index on `click_count` — updated frequently, never filtered on
- Composite index `(is_active, short_code)` — PK lookup already covers this; redundant

---

## 7. Partitioning & Sharding

### Transactional Store

At < 500M rows, PostgreSQL handles the `urls` table without partitioning. If needed:

```sql
-- Range partition by created_at (yearly buckets)
CREATE TABLE urls (
    short_code  VARCHAR(8)  NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE urls_2025 PARTITION OF urls
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE urls_2026 PARTITION OF urls
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

**Alternative at hyper-scale (Cassandra / ScyllaDB):**

```cql
CREATE TABLE urls (
    short_code  TEXT,
    long_url    TEXT,
    user_id     UUID,
    created_at  TIMESTAMP,
    expires_at  TIMESTAMP,
    is_active   BOOLEAN,
    click_count COUNTER,
    PRIMARY KEY (short_code)
) WITH compaction = { 'class': 'LeveledCompactionStrategy' }
  AND default_time_to_live = 0;
```

Cassandra distributes by `short_code` hash across nodes automatically — no manual shard management needed.

### ClickHouse (Analytics Store)

Sharded by `short_code` hash using Distributed table:

```sql
CREATE TABLE click_events_distributed ON CLUSTER tinyurl_cluster
AS click_events
ENGINE = Distributed(tinyurl_cluster, default, click_events, cityHash64(short_code));
```

---

## 8. Data Lifecycle & Retention

| Data                    | Retention Policy                                    | Mechanism                                          |
|-------------------------|-----------------------------------------------------|----------------------------------------------------|
| Active URLs             | Indefinite (unless expires_at set)                  | —                                                  |
| Expired URLs            | Deleted within 24h of expiry                        | Nightly cron DELETE                                |
| Deleted (soft) URLs     | 90 days (legal hold), then hard delete              | Monthly cleanup job                                |
| Anonymous URLs          | 1 year from last click (inactivity purge)           | Monthly cron on urls where user_id IS NULL         |
| Click events            | 90 days rolling                                     | ClickHouse TTL (`clicked_at + INTERVAL 90 DAY`)    |
| User accounts           | Retained until deletion request (GDPR)              | Account deletion API cascades to owned URLs        |
| Blacklist entries       | Until manually removed or expires_at                | No auto-expiry unless set                          |

---

## 9. Migrations

### Tooling: `golang-migrate`

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_create_urls.up.sql
├── 000002_create_urls.down.sql
├── 000003_create_custom_domains.up.sql
├── 000003_create_custom_domains.down.sql
├── 000004_create_blacklist.up.sql
└── 000004_create_blacklist.down.sql
```

**Apply:**
```bash
migrate -path ./migrations -database "$DATABASE_URL" up
```

**Rollback one step:**
```bash
migrate -path ./migrations -database "$DATABASE_URL" down 1
```

### Migration Rules

- **Always backward compatible** — new columns must be `DEFAULT` or `NULL`; never drop columns in the same release that removes their usage
- **No long-running locks** — use `CREATE INDEX CONCURRENTLY`; avoid `ALTER TABLE` on large tables in peak hours
- **Run in CI** — migration tests run on every PR against a fresh database
- **Checksums** — `golang-migrate` detects file changes after apply; never edit applied migration files

### Example: Adding a column safely

```sql
-- 000005_add_urls_title.up.sql
ALTER TABLE urls
    ADD COLUMN title VARCHAR(512) DEFAULT NULL;

-- 000005_add_urls_title.down.sql
ALTER TABLE urls
    DROP COLUMN IF EXISTS title;
```

---

## 10. Connection Management (Go)

```go
// sqlx pool configuration for Go services
db, err := sqlx.Connect("pgx", os.Getenv("DATABASE_URL"))
if err != nil {
    log.Fatal(err)
}

db.SetMaxOpenConns(50)          // max simultaneous DB connections per pod
db.SetMaxIdleConns(10)          // keep warm connections ready
db.SetConnMaxLifetime(5 * time.Minute)  // recycle connections (avoid stale)
db.SetConnMaxIdleTime(1 * time.Minute)  // close idle connections promptly
```

**Pool sizing:**
- Read Service pods: `MaxOpenConns = 25` (reads are fast, low hold time)
- Write Service pods: `MaxOpenConns = 50` (inserts hold slightly longer)
- Total connections to DB primary: `pods × MaxOpenConns` — size DB accordingly (PgBouncer recommended at scale)

### PgBouncer (Connection Pooler)

At > 50 pods, direct connections exhaust PostgreSQL's `max_connections`. Interpose PgBouncer in transaction pooling mode:

```
Go pods (N × 50 conns) ──▶ PgBouncer (pool of 100) ──▶ PostgreSQL (max_connections = 200)
```

---

## 11. Multi-DB Compatibility Notes

The schema above targets PostgreSQL syntax. Deviations per alternative DB:

| Feature                  | PostgreSQL          | MySQL / PlanetScale    | Cassandra / ScyllaDB       | DynamoDB              |
|--------------------------|---------------------|------------------------|----------------------------|-----------------------|
| UUID default             | `gen_random_uuid()` | `UUID()`               | `uuid()`                   | Application-generated |
| TIMESTAMPTZ              | Native              | `DATETIME(6)`          | `TIMESTAMP`                | Number (epoch ms)     |
| Partial indexes          | ✅ Supported        | ❌ Not supported        | N/A                        | N/A                   |
| Hash indexes             | ✅ Supported        | ❌ Use B-tree instead   | N/A (token-based routing)  | N/A                   |
| `ON CONFLICT DO NOTHING` | ✅ Supported        | `INSERT IGNORE`        | `IF NOT EXISTS`            | `ConditionExpression` |
| Full-text / regex CHECK  | ✅ Supported        | Limited                | ❌ Not supported            | ❌ Not supported       |
| `RETURNING` clause       | ✅ Supported        | ❌ Use `LAST_INSERT_ID` | ❌ Not supported            | Use response payload  |

---

## 12. Seed Data (Development)

```sql
-- dev/seed.sql

INSERT INTO users (user_id, email, password_hash, plan, is_verified) VALUES
    ('00000000-0000-0000-0000-000000000001', 'dev@tinyurl.com',
     '$2b$12$placeholder_hash', 'pro', TRUE);

INSERT INTO urls (short_code, long_url, user_id, is_custom) VALUES
    ('devtest1', 'https://example.com/page-one', '00000000-0000-0000-0000-000000000001', FALSE),
    ('devtest2', 'https://example.com/page-two', '00000000-0000-0000-0000-000000000001', FALSE),
    ('custom01', 'https://example.com/custom',  '00000000-0000-0000-0000-000000000001', TRUE);

INSERT INTO blacklist (domain, reason) VALUES
    ('malware-example.com', 'malware'),
    ('phishing-site.xyz',   'phishing');
```

---

*Document version: 1.0 — May 2026*
*See also: `SYSTEM_DESIGN.md` for architecture overview · `LLD.md` for API contracts and service internals.*
