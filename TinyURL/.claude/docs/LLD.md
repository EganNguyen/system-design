# Low-Level Design (LLD): TinyURL Service

---

## 1. Overview

TinyURL is a URL shortening service that converts long URLs into compact, shareable short links. This document covers the low-level design including data models, API contracts, component internals, and encoding strategies.

---

## 2. Functional Requirements (Recap)

- Shorten a long URL to a unique short code (e.g., `tinyurl.com/abc123`)
- Redirect short URL to original long URL
- Custom aliases (optional)
- URL expiry (optional, TTL-based)
- Analytics: click count, geo, referrer (optional)

---

## 3. Non-Functional Requirements (Recap)

- Read-heavy: ~100:1 read/write ratio
- Low latency redirects: < 10ms p99
- High availability: 99.99% uptime
- ~500M URLs stored, ~10B redirects/month

---

## 4. System Components

```
Client
  │
  ▼
Load Balancer (Nginx / AWS ALB)
  │
  ├──▶ Write Service (Go — URL Shortener API)
  │         │
  │         ▼
  │    ID Generator (Snowflake / Counter)
  │         │
  │         ▼
  │    Primary DB (Flexible — see §18)
  │         │
  │         ▼
  │    Cache Warmer → Redis Cache
  │
  └──▶ Read Service (Redirect API)
            │
            ▼
       Redis Cache (L1)
            │ (miss)
            ▼
       Read Replica DB (L2)
            │
            ▼
       Analytics Queue (Kafka)
                │
                ▼
         Analytics Service → ClickHouse / BigQuery
```

---

## 5. Data Models

### 5.1 URL Table (PostgreSQL or Cassandra)

```sql
CREATE TABLE urls (
    short_code      VARCHAR(8)      PRIMARY KEY,
    long_url        TEXT            NOT NULL,
    user_id         UUID,                        -- nullable (anonymous allowed)
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,                 -- nullable = no expiry
    is_custom       BOOLEAN         NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN         NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_urls_user_id    ON urls(user_id);
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;
```

**Column Notes:**
- `short_code`: Base62-encoded unique identifier (6–8 chars), e.g., `aB3xYz`
- `long_url`: Up to 2048 chars; stored as TEXT
- `expires_at`: Background job purges expired rows nightly

---

### 5.2 Analytics Table (ClickHouse / append-only)

```sql
CREATE TABLE click_events (
    event_id        UUID            DEFAULT generateUUIDv4(),
    short_code      VARCHAR(8)      NOT NULL,
    clicked_at      DateTime        NOT NULL,
    ip_hash         VARCHAR(64),    -- SHA-256 hashed for privacy
    country_code    VARCHAR(2),
    referrer        TEXT,
    user_agent      TEXT
) ENGINE = MergeTree()
ORDER BY (short_code, clicked_at);
```

---

### 5.3 User Table (optional, for registered users)

```sql
CREATE TABLE users (
    user_id         UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255)    UNIQUE NOT NULL,
    password_hash   VARCHAR(255)    NOT NULL,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    plan            VARCHAR(20)     NOT NULL DEFAULT 'free'  -- free | pro | enterprise
);
```

---

## 6. Short Code Generation

### Strategy: Base62 Encoding

**Alphabet:** `[0-9a-zA-Z]` → 62 characters

| Code Length | Possible URLs         |
|-------------|-----------------------|
| 6 chars     | 62⁶ = ~56.8 billion   |
| 7 chars     | 62⁷ = ~3.5 trillion   |
| 8 chars     | 62⁸ = ~218 trillion   |

**Recommended:** 7-char codes to support long-term scale.

### Algorithm Options

#### Option A — Counter + Base62 (Preferred)

```
1. Fetch next global counter from distributed counter service (e.g., Redis INCR or Snowflake ID)
2. Base62-encode the integer
3. Left-pad to 7 chars if needed
4. Check collision (rare but possible with counters = none, deterministic)
5. Write to DB
```

```go
const alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

func base62Encode(num uint64) string {
    if num == 0 {
        return string(alphabet[0])
    }
    result := []byte{}
    for num > 0 {
        result = append([]byte{alphabet[num%62]}, result...)
        num /= 62
    }
    return string(result)
}
```

#### Option B — MD5/SHA-1 Hash + Truncation

```
1. MD5(long_url + salt) → 128-bit hash
2. Take first 43 bits → Base62 encode to 7 chars
3. On collision, increment salt and retry (max 3 retries)
```

**Drawback:** Collision probability increases as DB fills; counter-based is preferred.

#### Option C — UUID v4 Truncation

```
1. Generate UUID v4
2. Take first 8 bytes, Base62-encode
3. Collision check against DB
```

**Drawback:** More DB round trips; worse than counter.

---

## 7. API Design

### 7.1 Create Short URL

```
POST /api/v1/urls
Content-Type: application/json
Authorization: Bearer <token>  (optional for anonymous)

Request:
{
  "long_url":    "https://example.com/very/long/url?with=params",
  "custom_code": "my-alias",          // optional
  "expires_in":  86400                // optional, seconds (86400 = 1 day)
}

Response 201 Created:
{
  "short_code": "aB3xYz7",
  "short_url":  "https://tinyurl.com/aB3xYz7",
  "long_url":   "https://example.com/very/long/url?with=params",
  "expires_at": "2026-05-30T00:00:00Z"
}

Error Responses:
  400 Bad Request     - Invalid URL format
  409 Conflict        - Custom alias already taken
  422 Unprocessable   - URL blacklisted / malicious
  429 Too Many Requests - Rate limit exceeded
```

### 7.2 Redirect (Core Hot Path)

```
GET /{short_code}

Response 301/302 Found:
  Location: https://example.com/very/long/url?with=params

Notes:
  - Use 301 (permanent) for caching at browser/CDN level (saves server load)
  - Use 302 (temporary) if click analytics must be captured server-side
  - Recommended: 302 redirect + async analytics write to Kafka
```

### 7.3 Get URL Info

```
GET /api/v1/urls/{short_code}
Authorization: Bearer <token>

Response 200 OK:
{
  "short_code":  "aB3xYz7",
  "long_url":    "https://example.com/...",
  "created_at":  "2026-05-29T10:00:00Z",
  "expires_at":  "2026-05-30T00:00:00Z",
  "click_count": 1042
}
```

### 7.4 Delete URL

```
DELETE /api/v1/urls/{short_code}
Authorization: Bearer <token>

Response 204 No Content

Side Effects:
  - Sets is_active = FALSE (soft delete)
  - Purges from Redis cache
  - Subsequent redirects return 410 Gone
```

### 7.5 Get Analytics

```
GET /api/v1/urls/{short_code}/analytics?from=2026-05-01&to=2026-05-29
Authorization: Bearer <token>

Response 200 OK:
{
  "short_code":   "aB3xYz7",
  "total_clicks": 5823,
  "by_day": [
    { "date": "2026-05-29", "clicks": 312 }
  ],
  "by_country": [
    { "country": "US", "clicks": 2100 },
    { "country": "IN", "clicks": 1200 }
  ],
  "top_referrers": [
    { "referrer": "twitter.com", "clicks": 900 }
  ]
}
```

---

## 8. Caching Strategy

### Redis Cache Schema

```
Key:   url:{short_code}
Value: JSON string  →  { "long_url": "...", "expires_at": "..." }
TTL:   Match URL expiry, or 24h for non-expiring URLs

Key:   blacklist:{domain}
Value: "1"
TTL:   No TTL (permanent until removed)
```

### Cache Flow (Read Path)

```
1. GET url:{short_code} from Redis
   ├── HIT  →  Return long_url immediately → async log click event
   └── MISS →  Query DB read replica
               ├── FOUND   → SET url:{short_code} in Redis → redirect
               └── NOT FOUND → 404
```

### Cache Invalidation

| Event              | Action                                      |
|--------------------|---------------------------------------------|
| URL deleted        | DEL url:{short_code} from Redis             |
| URL expired        | TTL handles naturally; cron job purges DB   |
| URL deactivated    | DEL url:{short_code} from Redis             |

---

## 9. Write Path — Sequence Diagram

```
Client  →  Write API  →  ID Generator  →  DB (Primary)  →  Redis
  │            │               │                │               │
  │──POST /urls│               │                │               │
  │            │──get next ID──│                │               │
  │            │◀──counter_val─│                │               │
  │            │──base62_encode(counter_val)     │               │
  │            │──INSERT INTO urls──────────────│               │
  │            │               │       ◀─ ACK ──│               │
  │            │──SET url:{code} ───────────────────────────────│
  │            │               │                │       ◀─ OK ──│
  │◀─ 201 ─────│               │                │               │
```

---

## 10. Read Path — Sequence Diagram

```
Client  →  Read API  →  Redis  →  DB Read Replica  →  Kafka
  │            │           │             │                │
  │──GET /code─│           │             │                │
  │            │──GET code─│             │                │
  │            │◀─ HIT ────│             │                │
  │            │────────────────────────────────────────▶ │ (async click event)
  │◀─ 302 ─────│           │             │                │

  (on MISS):
  │            │◀─ MISS ───│             │                │
  │            │────────────────────────▶│                │
  │            │◀─ long_url ────────────│                │
  │            │──SET code ────────────▶│                │
  │            │────────────────────────────────────────▶ │ (async click event)
  │◀─ 302 ─────│           │             │                │
```

---

## 11. Rate Limiting

### Strategy: Token Bucket per IP / User

| Tier        | Limit                        |
|-------------|------------------------------|
| Anonymous   | 10 creates / hour / IP       |
| Free user   | 100 creates / day            |
| Pro user    | 10,000 creates / day         |
| Enterprise  | Custom / negotiated          |

**Implementation:** Redis + Lua script (atomic decrement + TTL reset)

```lua
-- Redis Lua: token bucket check
local key    = KEYS[1]
local limit  = tonumber(ARGV[1])
local window = tonumber(ARGV[2])  -- seconds

local current = redis.call("INCR", key)
if current == 1 then
    redis.call("EXPIRE", key, window)
end
if current > limit then
    return 0  -- rate limited
end
return 1  -- allowed
```

---

## 12. URL Validation & Safety

### Validation Steps (on Write)

```
1. Parse URL → must be valid RFC 3986 format
2. Scheme must be http or https (block ftp://, javascript:, etc.)
3. Max length check → long_url ≤ 2048 chars
4. Domain blacklist check → Redis lookup
5. Google Safe Browsing API check (async, non-blocking for p99)
6. Custom alias check → alphanumeric + hyphen only, 3–20 chars
```

### Blacklist Sources

- Internal curated list (phishing, spam domains)
- Google Safe Browsing API
- PhishTank API
- Periodic crawler reports

---

## 13. Expiry & Cleanup

### Approach: Lazy Expiry + Background Job

```
Lazy check:
  - On redirect, if expires_at < NOW() → return 410 Gone
  - Remove from Redis (DEL key)

Background cron (daily at 02:00 UTC):
  DELETE FROM urls
  WHERE expires_at < NOW()
    AND expires_at IS NOT NULL;

  -- Also purge stale Redis keys (scan + TTL check)
```

---

## 14. Scalability Considerations

| Concern              | Solution                                                      |
|----------------------|---------------------------------------------------------------|
| Hot short codes      | CDN edge caching (301 responses), Redis cluster              |
| DB write throughput  | Counter service + async batch writes                         |
| DB read throughput   | Read replicas, Redis L1 cache (>95% cache hit target)        |
| ID generation        | Snowflake IDs or distributed counter with range pre-fetch    |
| Single region        | Multi-region active-passive; DNS failover                    |
| Counter contention   | Pre-fetch ID ranges per app server (e.g., 1000 IDs at once)  |

---

## 15. Failure Modes & Mitigations

| Failure                  | Impact                    | Mitigation                                          |
|--------------------------|---------------------------|-----------------------------------------------------|
| Redis down               | Cache miss → DB overload  | Redis Sentinel / Cluster; circuit breaker           |
| DB primary down          | No new writes             | Automatic failover (RDS Multi-AZ / Patroni)         |
| ID generator down        | Cannot create URLs        | Pre-fetched local ID ranges per node                |
| Analytics service lag    | Delayed stats             | Kafka buffers; stats are eventually consistent      |
| Hash collision            | Wrong redirect            | Pre-insert uniqueness check; retry with new salt    |

---

## 16. Security Considerations

- **Short code enumeration:** Codes are not sequential Base62 of small numbers; use large counter offsets or UUID-seeded generation to avoid guessing
- **SSRF via long URL:** Validate and block internal/private IP ranges in submitted URLs
- **Open redirect abuse:** Domain blacklist + Safe Browsing integration
- **Analytics PII:** Hash IPs before storage (SHA-256 + rotating salt)
- **Auth tokens:** JWT with 15-min expiry + refresh token rotation

---

## 17. Technology Stack Summary

| Layer              | Technology                                        |
|--------------------|---------------------------------------------------|
| Backend API        | **Go** (Fiber or Chi router)                      |
| Frontend           | **Next.js 14** (App Router, SSR + API Routes)     |
| Primary DB         | *Flexible — see §18*                              |
| Cache              | Redis 7 (Cluster mode)                            |
| Analytics DB       | ClickHouse                                        |
| Message Queue      | Apache Kafka                                      |
| ID Generation      | Snowflake ID / Redis INCR                         |
| CDN                | CloudFront / Cloudflare                           |
| Load Balancer      | AWS ALB / Nginx                                   |
| Container Infra    | Kubernetes (EKS)                                  |
| Monitoring         | Prometheus + Grafana + Sentry                     |

### Go — Backend Specifics

The Go service handles all redirect and write logic. Key packages:

```go
// go.mod dependencies
require (
    github.com/gofiber/fiber/v2      v2.52.0   // HTTP framework
    github.com/redis/go-redis/v9     v9.5.0    // Redis client
    github.com/jmoiron/sqlx          v1.3.5    // DB layer (driver-agnostic)
    github.com/golang-jwt/jwt/v5     v5.2.1    // JWT auth
    github.com/google/uuid           v1.6.0    // UUID generation
    go.uber.org/zap                  v1.27.0   // Structured logging
)
```

**Redirect handler (hot path):**

```go
func RedirectHandler(c *fiber.Ctx) error {
    code := c.Params("code")

    // L1: Redis cache
    longURL, err := cache.Get(ctx, "url:"+code).Result()
    if err == nil {
        go analyticsQueue.Publish(ClickEvent{Code: code, ...}) // async
        return c.Redirect(longURL, fiber.StatusFound)          // 302
    }

    // L2: DB fallback
    var url URLRecord
    if err := db.GetContext(ctx, &url, "SELECT * FROM urls WHERE short_code=$1 AND is_active=true", code); err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{"error": "not found"})
    }
    if url.ExpiresAt != nil && url.ExpiresAt.Before(time.Now()) {
        cache.Del(ctx, "url:"+code)
        return c.Status(fiber.StatusGone).JSON(fiber.Map{"error": "link expired"})
    }

    cache.Set(ctx, "url:"+code, url.LongURL, 24*time.Hour)
    go analyticsQueue.Publish(ClickEvent{Code: code, ...})
    return c.Redirect(url.LongURL, fiber.StatusFound)
}
```

### Next.js — Frontend Specifics

Next.js serves the dashboard UI and optionally proxies API calls.

```
app/
├── page.tsx                  # Landing + URL shortener form
├── dashboard/
│   ├── page.tsx              # User URL list
│   └── [code]/
│       └── page.tsx          # Analytics per short URL
├── api/
│   └── proxy/route.ts        # Optional: proxy to Go API (avoids CORS)
└── components/
    ├── ShortenForm.tsx
    ├── UrlTable.tsx
    └── ClickChart.tsx        # Recharts / Tremor
```

**Redirect note:** The actual `/{code}` redirect is handled by the **Go service**, not Next.js — routing short codes through a Node.js layer would add unnecessary latency. Next.js only serves `app.tinyurl.com/dashboard` etc.

---

## 18. Database Options (Flexible)

The data layer uses `sqlx` with a pluggable driver. Choose based on operational preference:

| Option                  | Best For                                   | Tradeoffs                                        |
|-------------------------|--------------------------------------------|--------------------------------------------------|
| **PostgreSQL**          | Default choice; strong consistency, JSONB  | Vertical scale limit; sharding is complex        |
| **CockroachDB**         | Global multi-region; Postgres-compatible   | Higher write latency vs single-node PG           |
| **PlanetScale (MySQL)** | Serverless / managed; branching workflow   | No foreign keys; MySQL dialect                   |
| **Cassandra / ScyllaDB**| Extreme write throughput, wide partition   | Eventual consistency; complex ops                |
| **DynamoDB**            | Serverless AWS; auto-scaling               | Vendor lock-in; limited query patterns           |
| **Turso (libSQL)**      | Edge-deployed SQLite; ultra-low latency    | Best for read-heavy, small datasets per region   |

### Recommendation by Scale

```
Early stage (0–50M URLs):       PostgreSQL (single node or RDS)
Growth (50M–500M URLs):         PostgreSQL + read replicas, or CockroachDB
Hyper-scale (500M+ URLs):       Cassandra / ScyllaDB for URL table;
                                 PostgreSQL retained for users/auth
```

### Migrations

Use **golang-migrate** for schema versioning, regardless of DB choice:

```bash
migrate -path ./migrations -database "$DATABASE_URL" up
```

---

*Document version: 1.1 — May 2026 (updated: Go + Next.js stack, flexible DB)*
