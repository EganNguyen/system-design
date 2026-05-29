# Sequence Diagrams: TinyURL Service

> Full interaction diagrams for all critical flows.
> Extends the partial ASCII diagrams in [LLD.md §9–10](LLD.md).
> All diagrams use Mermaid `sequenceDiagram` syntax.

---

## 1. Shorten Flow (with Collision Retry)

The write path: client submits a long URL, the service validates it, generates a short code,
persists it, and warms the cache. Collision retry covers the rare case where a generated
code already exists in the DB.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant WriteAPI as Write Service
    participant Validator as URL Validator
    participant SafeBrowsing as Safe Browsing API
    participant Blacklist as Redis Blacklist
    participant IDGen as ID Generator
    participant DB as Primary DB
    participant Cache as Redis Cache

    Client->>WriteAPI: POST /api/v1/urls { long_url, custom_code?, expires_in? }

    WriteAPI->>Validator: validate(long_url)
    Validator->>Validator: scheme in {http, https}?
    Validator->>Validator: resolve hostname — block RFC 1918 / loopback / IMDS IPs
    Validator->>Validator: length <= 2048 chars?

    alt Validation fails
        Validator-->>WriteAPI: invalid
        WriteAPI-->>Client: 400 VALIDATION_ERROR
    end

    WriteAPI->>Blacklist: GET blacklist:{domain}
    alt Domain is blacklisted
        Blacklist-->>WriteAPI: hit
        WriteAPI-->>Client: 422 URL_BLOCKED
    end

    WriteAPI-)SafeBrowsing: async check(long_url)
    Note right of SafeBrowsing: Non-blocking. Result<br/>processed post-write.<br/>If flagged, link is<br/>deactivated asynchronously.

    WriteAPI->>IDGen: next_id()
    IDGen-->>WriteAPI: counter_value
    WriteAPI->>WriteAPI: short_code = base62_encode(counter_value)

    WriteAPI->>DB: INSERT INTO urls (short_code, long_url, user_id, expires_at)

    alt Collision — short_code already exists (UNIQUE VIOLATION)
        DB-->>WriteAPI: constraint error
        Note over WriteAPI: Retry up to 3 times<br/>with incremented counter
        WriteAPI->>IDGen: next_id()
        IDGen-->>WriteAPI: counter_value + 1
        WriteAPI->>WriteAPI: short_code = base62_encode(new_value)
        WriteAPI->>DB: INSERT INTO urls (short_code, ...)
    end

    DB-->>WriteAPI: OK

    WriteAPI->>Cache: SET url:{short_code} { long_url, expires_at } EX ttl
    Cache-->>WriteAPI: OK

    WriteAPI-->>Client: 201 Created { short_code, short_url, expires_at }
```

---

## 2. Redirect Flow — Cache Hit

The hot path. No DB or external call. p99 target: < 10 ms.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant CDN
    participant ReadAPI as Read Service
    participant Cache as Redis Cache
    participant Kafka

    Client->>CDN: GET tinyurl.com/{short_code}

    alt CDN edge cache hit (301 cached indefinitely)
        CDN-->>Client: 301 Location: long_url
        Note right of CDN: Served at edge.<br/>No origin call made.
    end

    CDN->>ReadAPI: GET /{short_code}  (CDN cache miss — forward to origin)

    ReadAPI->>Cache: GET url:{short_code}
    Cache-->>ReadAPI: HIT { long_url, expires_at }

    ReadAPI->>ReadAPI: expires_at is null OR expires_at > NOW()?

    ReadAPI-)Kafka: publish ClickEvent { short_code, ip_hash, country, referrer, ts }
    Note right of Kafka: Fire-and-forget after<br/>response is sent.<br/>Does not block p99.

    ReadAPI-->>CDN: 302 Location: long_url
    CDN-->>Client: 302 Location: long_url
```

---

## 3. Redirect Flow — Cache Miss and Expired URL

Two alternate paths from the cache miss branch: record not found, expired, or active.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant CDN
    participant ReadAPI as Read Service
    participant Cache as Redis Cache
    participant DBReplica as DB Read Replica
    participant Kafka

    Client->>CDN: GET tinyurl.com/{short_code}
    CDN->>ReadAPI: GET /{short_code}  (CDN cache miss)

    ReadAPI->>Cache: GET url:{short_code}
    Cache-->>ReadAPI: MISS

    ReadAPI->>DBReplica: SELECT * FROM urls WHERE short_code = $1 AND is_active = true

    alt Record not found
        DBReplica-->>ReadAPI: 0 rows
        ReadAPI-->>CDN: 404 Not Found
        CDN-->>Client: 404 Not Found
    else Record found but expired (expires_at < NOW())
        DBReplica-->>ReadAPI: row { expires_at < NOW() }
        ReadAPI->>Cache: DEL url:{short_code}
        ReadAPI-->>CDN: 410 Gone
        CDN-->>Client: 410 Gone
    else Record found and active
        DBReplica-->>ReadAPI: row { long_url, expires_at }
        ReadAPI->>Cache: SET url:{short_code} { long_url, expires_at } EX ttl
        Cache-->>ReadAPI: OK
        ReadAPI-)Kafka: publish ClickEvent { short_code, ip_hash, country, referrer, ts }
        ReadAPI-->>CDN: 302 Location: long_url
        CDN-->>Client: 302 Location: long_url
    end
```

---

## 4. Custom Alias Creation

Same write path as §1 but the ID generator is replaced by the user-supplied alias,
with an availability check before the DB insert.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant WriteAPI as Write Service
    participant Validator as URL Validator
    participant ReservedList as Reserved Alias Set
    participant Cache as Redis Cache
    participant DB as Primary DB

    Client->>WriteAPI: POST /api/v1/urls { long_url, custom_code: "my-sale" }

    WriteAPI->>Validator: validate(long_url)
    Validator-->>WriteAPI: OK

    WriteAPI->>Validator: validate_alias("my-sale")
    Note over Validator: 3–20 chars, [a-zA-Z0-9-] only

    alt Alias format invalid
        Validator-->>WriteAPI: invalid format
        WriteAPI-->>Client: 400 VALIDATION_ERROR { details.custom_code }
    end

    WriteAPI->>ReservedList: is_reserved("my-sale")?
    Note over ReservedList: In-memory set: login, admin,<br/>support, api, dashboard…

    alt Alias is a reserved word
        ReservedList-->>WriteAPI: reserved
        WriteAPI-->>Client: 409 ALIAS_CONFLICT
    end

    WriteAPI->>Cache: GET url:my-sale
    alt Already in cache (taken)
        Cache-->>WriteAPI: HIT
        WriteAPI-->>Client: 409 ALIAS_CONFLICT
    end

    WriteAPI->>DB: SELECT 1 FROM urls WHERE short_code = 'my-sale'

    alt Already exists in DB
        DB-->>WriteAPI: row found
        WriteAPI-->>Client: 409 ALIAS_CONFLICT
    end

    DB-->>WriteAPI: 0 rows (available)

    WriteAPI->>DB: INSERT INTO urls (short_code='my-sale', long_url, is_custom=true, expires_at)
    DB-->>WriteAPI: OK

    WriteAPI->>Cache: SET url:my-sale { long_url, expires_at } EX ttl
    Cache-->>WriteAPI: OK

    WriteAPI-->>Client: 201 Created { short_code: "my-sale", short_url, expires_at }
```

---

## 5. URL Expiry Cleanup Job

Background cron that runs nightly. Lazy expiry is also handled inline during redirects
(§3) — this batch job cleans up what lazy expiry leaves behind.

```mermaid
sequenceDiagram
    autonumber
    participant Cron as Cron Scheduler
    participant CleanupJob as Cleanup Worker
    participant DB as Primary DB
    participant Cache as Redis Cache
    participant AuditLog as Audit Log

    Note over Cron: Fires daily at 02:00 UTC

    Cron->>CleanupJob: trigger cleanup_expired_urls()

    CleanupJob->>DB: SELECT short_code FROM urls<br/>WHERE expires_at < NOW()<br/>AND is_active = true<br/>LIMIT 1000

    DB-->>CleanupJob: [ short_code_1, short_code_2, ... ]

    loop For each batch of expired short codes
        CleanupJob->>DB: UPDATE urls SET is_active = false<br/>WHERE short_code IN (batch)
        DB-->>CleanupJob: rows_affected

        CleanupJob->>Cache: DEL url:{short_code} ... (pipelined batch)
        Cache-->>CleanupJob: OK

        CleanupJob->>AuditLog: append { action: EXPIRED, codes: batch, ts: NOW() }
    end

    alt More rows remain (next batch exists)
        CleanupJob->>CleanupJob: sleep 100ms (backpressure)
        CleanupJob->>DB: SELECT next batch...
        Note over CleanupJob: Continues until<br/>0 rows returned
    end

    CleanupJob-->>Cron: done { total_expired: N, duration_ms: M }

    Note over CleanupJob,Cache: Lazy expiry (inline, §3):<br/>On redirect, if expires_at < NOW()<br/>the Read Service returns 410 and<br/>DELs the cache key immediately —<br/>does not wait for the cron job.
```

---

## 6. Analytics Ingestion Pipeline

The async path from a click event publish to a queryable ClickHouse row.
This entire pipeline runs off the critical redirect path.

```mermaid
sequenceDiagram
    autonumber
    participant ReadAPI as Read Service
    participant Kafka as Kafka<br/>(click_events topic)
    participant Consumer as Analytics Consumer
    participant SchemaVal as Schema Validator
    participant DLQ as Dead Letter Queue
    participant IPHasher as IP Hasher
    participant GeoLookup as MaxMind GeoIP
    participant ClickHouse

    ReadAPI-)Kafka: publish { short_code, raw_ip, user_agent, referrer, ts }
    Note right of ReadAPI: Fire-and-forget after<br/>redirect response is sent.<br/>Does not affect p99.

    Kafka->>Consumer: poll batch (max 500 msgs / 100ms wait)

    loop For each message in batch
        Consumer->>SchemaVal: validate(message)

        alt Schema invalid
            SchemaVal-->>Consumer: error
            Consumer->>DLQ: route to dead_letter_queue
            Note right of DLQ: Ops alert triggered<br/>if DLQ depth > threshold
        else Schema valid
            SchemaVal-->>Consumer: OK

            Consumer->>IPHasher: SHA-256(raw_ip + daily_rotating_salt)
            IPHasher-->>Consumer: ip_hash
            Note over IPHasher: raw_ip discarded here.<br/>Never written to storage.

            Consumer->>GeoLookup: country_code(raw_ip)
            GeoLookup-->>Consumer: "US"
            Note over GeoLookup: Country-level only.<br/>No city or coordinates.

            Consumer->>Consumer: sanitize(user_agent, referrer)<br/>strip HTML, limit 2048 chars, enforce UTF-8

            Consumer->>ClickHouse: INSERT INTO click_events<br/>{ short_code, ip_hash, country_code,<br/>  referrer, user_agent, clicked_at }
            ClickHouse-->>Consumer: OK
        end
    end

    Consumer->>Kafka: commit offset
    Note over Consumer,Kafka: Offset committed only after<br/>successful ClickHouse insert.<br/>Failed batches are retried<br/>before committing.
```

---

## 7. JWT Issue and Refresh Rotation

How a consumer obtains an access token and silently rotates it via the refresh token.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant AuthAPI as Auth Service
    participant DB as Users DB
    participant TokenStore as Redis Token Store

    Client->>AuthAPI: POST /api/v1/auth/token { email, password }

    AuthAPI->>DB: SELECT password_hash FROM users WHERE email = $1
    DB-->>AuthAPI: password_hash
    AuthAPI->>AuthAPI: bcrypt.compare(password, hash)

    alt Invalid credentials
        AuthAPI-->>Client: 401 UNAUTHORIZED
    end

    AuthAPI->>AuthAPI: sign access_token (RS256, exp +15min, jti: uuid_A)
    AuthAPI->>AuthAPI: sign refresh_token (RS256, exp +30d, jti: uuid_B)

    AuthAPI->>TokenStore: SET refresh:{user_id} uuid_B EX 2592000
    TokenStore-->>AuthAPI: OK

    AuthAPI-->>Client: 200 { access_token, refresh_token }
    Note right of Client: access_token stored in memory.<br/>refresh_token in httpOnly<br/>Secure SameSite=Strict cookie.

    Note over Client,AuthAPI: --- 15 minutes later: access_token expires ---

    Client->>AuthAPI: POST /api/v1/auth/refresh  (cookie: refresh_token)
    AuthAPI->>AuthAPI: verify refresh_token signature (RS256 public key)
    AuthAPI->>TokenStore: GET refresh:{user_id}

    alt Stored jti != token jti  (refresh token reuse — potential theft)
        TokenStore-->>AuthAPI: jti mismatch
        AuthAPI->>TokenStore: DEL refresh:{user_id}
        Note over AuthAPI: All sessions revoked.<br/>Treat as compromised credential.
        AuthAPI-->>Client: 401 UNAUTHORIZED
    end

    AuthAPI->>AuthAPI: issue new access_token (exp +15min)
    AuthAPI->>AuthAPI: issue new refresh_token (new jti: uuid_C)
    AuthAPI->>TokenStore: SET refresh:{user_id} uuid_C EX 2592000
    TokenStore-->>AuthAPI: OK

    AuthAPI-->>Client: 200 { access_token, refresh_token }
    Note right of Client: Old refresh_token is now invalid.<br/>Any reuse triggers session revocation.
```

---

## Flow Reference

| # | Flow | Extends |
|---|---|---|
| 1 | Shorten with collision retry | LLD §9, SYSTEM_DESIGN §6 |
| 2 | Redirect — cache hit | LLD §10, SYSTEM_DESIGN §6 |
| 3 | Redirect — cache miss / expired | LLD §10, SYSTEM_DESIGN §6 |
| 4 | Custom alias creation | LLD §7.1, SECURITY B3 |
| 5 | URL expiry cleanup job | LLD §13 |
| 6 | Analytics ingestion pipeline | LLD §4, SYSTEM_DESIGN §6, SECURITY F1 |
| 7 | JWT issue + refresh rotation | API_CONTRACT §3, SECURITY C2 |

---

*Source references: [LLD.md §9–10](LLD.md), [SYSTEM_DESIGN.md §6](SYSTEM_DESIGN.md), [SECURITY.md](SECURITY.md), [API_CONTRACT.md §3](API_CONTRACT.md).*
