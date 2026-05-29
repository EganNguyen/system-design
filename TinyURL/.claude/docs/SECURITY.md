# Security Threat Model: TinyURL Service

> STRIDE analysis per entry point. Supersedes the surface-level notes in SYSTEM_DESIGN §11 and LLD §16.
> This document defines threats, attack vectors, impact, and required mitigations.
> A finding here without a corresponding mitigation in LLD is an open security gap.

---

## 1. Trust Boundary Map

```
                    ┌──────────────────────────────────────────────────┐
                    │               UNTRUSTED ZONE                      │
                    │                                                    │
                    │  Anonymous Internet Users   Authenticated Users   │
                    │       (Link Clickers)         (API Consumers)     │
                    └─────────────────┬──────────────────┬─────────────┘
                                      │                  │
                           ─ ─ ─ ─ ─ ┼ ─ Trust Boundary ┼ ─ ─ ─ ─ ─
                                      │                  │
                    ┌─────────────────▼──────────────────▼────────────┐
                    │             PUBLIC ENTRY POINTS                   │
                    │                                                    │
                    │  [A] GET /{short_code}    [B] POST /api/v1/urls  │
                    │  [C] GET /api/v1/urls/*   [D] DELETE /api/v1/... │
                    │  [E] GET /api/v1/urls/*/analytics                │
                    └──────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────▼──────────────────────────────┐
                    │              INTERNAL ZONE                       │
                    │                                                  │
                    │  Redis Cache  Primary DB  Kafka  ClickHouse     │
                    │  ID Generator  Analytics Svc     Admin API      │
                    └──────────────────────────────────────────────────┘
```

### Entry Point Inventory

| ID | Entry Point | Auth Required | Caller |
|---|---|---|---|
| A | `GET /{short_code}` — redirect hot path | No | Any internet user |
| B | `POST /api/v1/urls` — create short URL | Optional (anonymous allowed) | API consumers, bots |
| C | `GET /api/v1/urls/{code}` / list | Yes (owner) | Authenticated users |
| D | `DELETE /api/v1/urls/{code}` | Yes (owner) | Authenticated users |
| E | `GET /api/v1/urls/{code}/analytics` | Yes (owner) | Authenticated users |
| F | Analytics ingest (Kafka consumer) | Internal only | Analytics service |
| G | Admin / ops API | Internal only | SRE / operations |

---

## 2. STRIDE Analysis

### Entry Point A — Redirect Hot Path (`GET /{short_code}`)

**Threat surface:** Highest volume, lowest latency, zero authentication.

---

#### A1 — Spoofing: Phishing via Malicious Redirect

| | |
|---|---|
| **Attack** | Attacker creates a short link pointing to a phishing page, credential harvester, or malware download. The short URL obscures the true destination. |
| **Impact** | Critical — reputational damage; service used as phishing infrastructure; domain blocklisted by browsers and email filters. |
| **Likelihood** | High — this is the most common abuse of URL shorteners. |

**Mitigations:**
- Validate all submitted URLs against **Google Safe Browsing API** at write time (async post-write; non-blocking for p99). Block on known-bad result before returning `201`.
- Maintain an **internal domain blacklist** table (see DATABASE_DESIGN §4.5). Redis hash lookup on every write; O(1).
- Integrate **PhishTank API** as a secondary signal on async re-scan job.
- Display a **preview interstitial** (`/preview/{short_code}`) as an opt-in feature so clickers can inspect the destination before being redirected.
- **Nightly re-scan job**: re-validate top-1000 click-volume URLs against Safe Browsing to catch links that were clean at creation but are now flagged.

---

#### A2 — Denial of Service: Redirect Flood / Cache Miss Storm

| | |
|---|---|
| **Attack** | Attacker sends millions of requests to `GET /{short_code}` targeting a single hot short code to exhaust app server or Redis connections. Alternatively, floods with random codes to generate cache misses, overloading the DB read replica. |
| **Impact** | High — violates the 99.99% availability SLO; cascading DB overload. |
| **Likelihood** | High at enterprise scale — coordinated load is a realistic incident. |

**Mitigations:**
- **CDN edge caching**: `301` responses cached at CDN edge indefinitely; `302` responses cached for a short TTL (e.g., 5 s). Peak traffic for hot links never reaches the origin.
- **IP-based rate limiting at the ALB/ingress layer** for redirect requests (e.g., 1,000 redirects/min/IP). Legitimate users do not click the same link 1,000 times per minute.
- **Cache stampede protection** on hot key expiry: probabilistic early expiry (PER algorithm) + per-key mutex so only one goroutine fetches from DB on miss.
- **Circuit breaker on Redis**: if Redis is unreachable, serve a configurable percentage of traffic from an in-process LRU (top-N hot codes) rather than failing all requests.

---

#### A3 — Information Disclosure: Short Code Enumeration

| | |
|---|---|
| **Attack** | Short codes are 7 chars, Base62. Attacker iterates sequentially or randomly to discover valid links, exposing URLs that were "private by obscurity" (e.g., internal staging links, pre-launch campaign URLs). |
| **Impact** | Medium — confidential destination URLs exposed. |
| **Likelihood** | Medium — trivial to automate with a simple loop. |

**Mitigations:**
- **Offset the counter**: start the global counter at a large random offset (e.g., 10 billion), not 0. This defeats sequential scanning.
- **Rate limit on `404` responses per IP**: if a caller receives more than 50 consecutive `404`s, treat as scanning and return `429` for a backoff window.
- **Do not distinguish "never existed" from "deleted"** — both return `410 Gone`. Limits information leakage about namespace occupancy.
- For genuinely sensitive links, support an optional **HMAC query token** (`?t=<token>`) — the redirect only works if the token is valid. Opt-in at creation time.

---

#### A4 — Tampering: CDN Cache Poisoning

| | |
|---|---|
| **Attack** | Attacker performs a man-in-the-middle between CDN and origin to poison the CDN cache with a malicious redirect destination for a legitimate short code. |
| **Impact** | High — all CDN cache hits for a popular link redirect to an attacker-controlled URL. |
| **Likelihood** | Low — requires network position, but worth defending. |

**Mitigations:**
- **Enforce HTTPS everywhere** — TLS from client to CDN, CDN to origin. No plain HTTP origin pull.
- **CDN origin shield**: CDN communicates with origin over a private backbone, not the public internet.

---

### Entry Point B — Create Short URL (`POST /api/v1/urls`)

**Threat surface:** Write path. Anonymous submissions allowed. Primary abuse vector.

---

#### B1 — Spoofing: Anonymous Write Abuse / Namespace Spam

| | |
|---|---|
| **Attack** | Attacker submits millions of creation requests without authentication to exhaust the short code namespace or generate spam links at scale. |
| **Impact** | High — namespace pollution; DB write overload; service used as bulk spam infrastructure. |
| **Likelihood** | High — anonymous writes are intentional in MVP scope, making this an obvious target. |

**Mitigations:**
- **Token bucket rate limiting per IP** (10 creates/hour anonymous, enforced via Redis Lua; see LLD §11).
- **Signed challenge token for anonymous creates**: the frontend issues a short-lived HMAC-signed challenge (`exp + ip + nonce`). `POST /api/v1/urls` must include this token. Distinguishes browser users from headless scripts without requiring a CAPTCHA.
- **Block datacenter/VPN IP ranges** from anonymous writes (configurable). Legitimate human users rarely post from AWS or Tor exit nodes.

---

#### B2 — Spoofing: SSRF via Submitted URL

| | |
|---|---|
| **Attack** | Attacker submits `http://169.254.169.254/latest/meta-data/` (AWS IMDSv1), `http://localhost:6379` (Redis), or `http://10.0.0.1/admin`. If the service ever fetches the submitted URL server-side (title scraping, preview generation, Safe Browsing pre-fetch via proxy), the attacker reaches internal infrastructure. |
| **Impact** | Critical — credential exfiltration via IMDS; internal service access. |
| **Likelihood** | Medium — only exploitable if the service fetches the URL server-side. |

**Mitigations:**
- **Never fetch submitted URLs server-side** in the hot create path. Safe Browsing API receives the URL string, not a proxied response.
- **URL validation pipeline** (synchronous, before any DB write):
  1. Parse — must be valid RFC 3986.
  2. Scheme must be `http` or `https` only. Reject `ftp://`, `javascript:`, `data:`, `file://`.
  3. Resolve hostname to IP. Reject if IP falls in:
     - `127.0.0.0/8` (loopback)
     - `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (RFC 1918 private)
     - `169.254.0.0/16` (link-local / IMDS)
     - `::1`, `fc00::/7` (IPv6 loopback / unique local)
  4. Block single-label hostnames (e.g., `http://internal`).
- **Re-resolve on redirect** (paranoia layer for new links < 1 hour old): re-validate the stored `long_url`'s resolved IP to defend against DNS rebinding attacks.

---

#### B3 — Tampering: Custom Alias Squatting

| | |
|---|---|
| **Attack** | Attacker pre-registers common aliases (`/login`, `/support`, `/admin`) before they are used by the platform or enterprise customers, causing confusion or enabling phishing. |
| **Impact** | Medium — brand confusion; potential phishing. |
| **Likelihood** | Medium — trivial to automate. |

**Mitigations:**
- Maintain a **reserved alias blocklist** of platform/brand terms (`login`, `admin`, `support`, `api`, `dashboard`, `help`, `billing`, `security`, etc.).
- Only verified enterprise accounts can claim aliases matching their registered domain prefix.

---

#### B4 — Information Disclosure: URL Leakage in Error Messages

| | |
|---|---|
| **Attack** | Error responses echo back the submitted `long_url` in the response body (e.g., `"URL 'http://internal.co/secret' is blocked"`), leaking internal URL structure. |
| **Impact** | Low-medium — internal URL structure exposed. |
| **Likelihood** | Medium — a common accidental implementation mistake. |

**Mitigations:**
- **Never echo the submitted URL in error responses**. Use generic messages: `"The submitted URL is not allowed."`.
- Internal structured logs may record the URL (acceptable); the HTTP response body must not.

---

### Entry Points C / D — Authenticated URL Management

**Threat surface:** Owner-scoped reads and deletes. JWT authentication required.

---

#### C1 — Elevation of Privilege: IDOR (Insecure Direct Object Reference)

| | |
|---|---|
| **Attack** | Attacker authenticates as user A, then calls `GET /api/v1/urls/{code}` or `DELETE /api/v1/urls/{code}` where `{code}` belongs to user B. If the server checks only that the token is valid (not that the caller owns the resource), user A reads or deletes user B's links. |
| **Impact** | High — unauthorized data access; mass deletion of target links. |
| **Likelihood** | High — IDOR is the #1 API vulnerability class. Trivially exploitable if not explicitly checked. |

**Mitigations:**
- Every DB query on authenticated endpoints must include `AND user_id = $caller_id` in the `WHERE` clause. Never fetch by `short_code` alone and check ownership afterward.
- **Integration test coverage is mandatory**: a test that authenticates as user A and requests user B's URL must return `403`, not `200` or `404`.

---

#### C2 — Spoofing: JWT Forgery / Token Theft

| | |
|---|---|
| **Attack** | Attacker steals a valid JWT (via XSS, network sniffing, or log exposure) and impersonates the victim. Alternatively, attacker forges a JWT if HS256 is used with a guessable secret. |
| **Impact** | High — full account takeover for the token lifetime. |
| **Likelihood** | Medium — token theft via XSS is common in web apps. |

**Mitigations:**
- Use **RS256** (asymmetric). Compromise of an API server does not expose the signing key.
- Access token TTL: **15 minutes**. Short window limits blast radius.
- **Refresh token rotation**: reuse of a previously rotated refresh token is treated as a compromise signal → revoke all sessions for the user.
- **Never log JWT values** — scrub `Authorization` headers from request logs.
- Store refresh tokens in **httpOnly, Secure, SameSite=Strict cookies** to prevent JavaScript access.
- Maintain a **token revocation list** in Redis for explicit logout and password resets (`SET revoked:{jti} 1 EX <remaining_ttl>`).

---

#### C3 — Repudiation: No Audit Trail for Destructive Actions

| | |
|---|---|
| **Attack** | A legitimate user (or attacker with stolen credentials) deletes a high-value short URL. Without an audit log, there is no forensic record of who deleted it, when, or from which IP. |
| **Impact** | Medium — loss of accountability; inability to reconstruct events for fraud or legal investigations. |
| **Likelihood** | High — a gap in the current design if not explicitly addressed. |

**Mitigations:**
- Append an **audit log entry** on every `DELETE` and every `POST /api/v1/urls`, recording: `user_id`, `short_code`, `action`, `ip_hash`, `user_agent`, `timestamp`.
- Audit log is **append-only** — no update or delete operations permitted.
- Retain audit logs for **90 days minimum**.

---

### Entry Point E — Analytics (`GET /api/v1/urls/{code}/analytics`)

---

#### E1 — Information Disclosure: PII in Analytics Data

| | |
|---|---|
| **Attack** | The analytics pipeline stores raw IP addresses or precise geolocation in ClickHouse. A ClickHouse breach, misconfigured query, or IDOR on the analytics endpoint exposes users' location and browsing patterns. This is a GDPR Article 5 and CCPA violation. |
| **Impact** | Critical — regulatory fines; user trust destruction. |
| **Likelihood** | High — raw IPs are stored by default if not explicitly designed against. |

**Mitigations:**
- **Hash IPs at ingest boundary** before writing to Kafka: `SHA-256(ip + daily_rotating_salt)`. The raw IP never touches persistent storage.
- Store only **country-level geolocation** (ISO 3166-1 alpha-2), not city or coordinates.
- The analytics API response **never exposes `ip_hash`** — internal field only.
- **Data retention**: click event records deleted after 90 days via ClickHouse TTL partition drop.

---

#### E2 — Denial of Service: Expensive Analytics Query

| | |
|---|---|
| **Attack** | Authenticated attacker issues a query spanning years (`from=2020-01-01&to=2026-05-29`). ClickHouse scans a large partition range, consuming significant resources. |
| **Impact** | Medium — analytics service degradation. |
| **Likelihood** | Medium — easy to trigger accidentally or intentionally. |

**Mitigations:**
- **Maximum query window of 90 days** enforced at the API layer.
- **Query timeout**: ClickHouse queries exceeding 10 seconds are cancelled at the DB driver level.

---

### Entry Point F — Analytics Ingest (Kafka, Internal)

---

#### F1 — Tampering: Malformed or Injected Click Events

| | |
|---|---|
| **Attack** | A compromised internal service writes malformed or fabricated click events to the Kafka topic, inflating click counts or injecting XSS payloads into `referrer` / `user_agent` fields that render in the analytics dashboard. |
| **Impact** | Medium — data integrity corruption; stored XSS if fields are rendered unescaped. |
| **Likelihood** | Medium — insider threat or misconfigured service. |

**Mitigations:**
- **Schema validation** on Kafka consumer: every message validated against a strict schema before insert. Malformed messages routed to a dead-letter queue.
- **Sanitize string fields** (`referrer`, `user_agent`): strip HTML, limit to 2048 chars, enforce UTF-8.
- **Kafka topic ACLs**: only the redirect service's service account has write access to the click events topic.

---

### Entry Point G — Admin / Ops API (Internal)

---

#### G1 — Elevation of Privilege: Exposed Admin Endpoints

| | |
|---|---|
| **Attack** | Admin endpoints (blacklist management, user plan changes, cache flush) are accidentally exposed on the public-facing port, allowing an unauthenticated attacker to add arbitrary domains to the blacklist or flush the entire cache. |
| **Impact** | Critical — blacklist manipulation enables phishing links; cache flush triggers a complete miss storm. |
| **Likelihood** | Medium — misconfiguration is a common cause of admin exposure. |

**Mitigations:**
- Admin API runs on a **separate port** (e.g., `:9090`) that is **not exposed by the load balancer** — only reachable from within the internal VPC.
- Admin endpoints are **not in the `/api/v1/` namespace**. A request for an admin path received by the public router returns `404`.
- Admin actions require **mutual TLS (mTLS)** — the caller must present a valid client certificate issued by the internal CA.

---

## 3. Threat Summary Matrix

| ID | Threat | Entry Point | Severity | Likelihood | Status |
|---|---|---|---|---|---|
| A1 | Phishing via malicious redirect | Redirect | Critical | High | Mitigated |
| A2 | Redirect flood / cache miss storm | Redirect | High | High | Mitigated |
| A3 | Short code enumeration | Redirect | Medium | Medium | Mitigated |
| A4 | CDN cache poisoning | Redirect | High | Low | Mitigated |
| B1 | Anonymous write abuse | Shorten API | High | High | Mitigated |
| B2 | SSRF via submitted URL | Shorten API | Critical | Medium | Mitigated |
| B3 | Custom alias squatting | Shorten API | Medium | Medium | Mitigated |
| B4 | URL leakage in error messages | Shorten API | Low | Medium | Mitigated |
| C1 | IDOR — unauthorized resource access | Auth API | High | High | Mitigated |
| C2 | JWT forgery / token theft | Auth API | High | Medium | Mitigated |
| C3 | No audit trail for deletions | Auth API | Medium | High | Mitigated |
| E1 | PII in analytics (raw IPs) | Analytics | Critical | High | Mitigated |
| E2 | Expensive analytics query DoS | Analytics | Medium | Medium | Mitigated |
| F1 | Malformed / injected click events | Kafka ingest | Medium | Medium | Mitigated |
| G1 | Exposed admin endpoints | Admin API | Critical | Medium | Mitigated |

---

## 4. Security Controls Summary

| Control | Where enforced | Covers |
|---|---|---|
| URL validation pipeline (scheme, IP, RFC 3986) | Write service, pre-DB | B2 |
| Google Safe Browsing + PhishTank + nightly re-scan | Write service, async | A1 |
| Domain blacklist (Redis hash + DB table) | Write service | A1, B3 |
| Token bucket rate limiting (Redis Lua) | All entry points | A2, B1 |
| IP range block (loopback, RFC 1918, link-local) | Write service | B2 |
| Counter offset (start at 10B+) | ID generation | A3 |
| 404-rate-per-IP circuit breaker | Redirect service | A3 |
| RS256 JWT + 15 min TTL + refresh rotation | Auth layer | C2 |
| Token revocation list (Redis) | Auth layer | C2 |
| httpOnly / Secure / SameSite=Strict cookie | Frontend / auth | C2 |
| Owner-scoped DB queries (`AND user_id = $caller_id`) | All auth endpoints | C1 |
| Append-only audit log | Write + delete paths | C3 |
| IP hashing at Kafka ingest boundary | Analytics ingest | E1 |
| Country-only geolocation storage | Analytics ingest | E1 |
| 90-day query window cap + ClickHouse timeout | Analytics API | E2 |
| Kafka topic ACLs + schema validation | Message broker | F1 |
| Admin API on separate port + VPC-only + mTLS | Infra / routing | G1 |
| HTTPS enforced everywhere (TLS at ALB) | Infra | A4, C2 |

---

## 5. Open Security Gaps

Identified but **not yet mitigated** in the current design. Required before public traffic.

| Gap | Risk | Required action |
|---|---|---|
| No WAF rule set | No layer-7 filtering for SQLi, path traversal in query params | Deploy WAF (AWS WAF or Cloudflare WAF) with OWASP Core Rule Set in front of the ALB |
| No abuse ML model | Rule-based blacklist misses novel phishing domains | Integrate a URL classifier as an async post-creation signal; flag for manual review |
| No slow-scan anomaly detection | Slow enumeration (1 req/sec over hours) bypasses per-minute rate limits | Emit per-IP 404-rate metrics to Prometheus; alert + auto-block IPs exceeding threshold over a 1-hour rolling window |
| Refresh token storage not specified | If stored in localStorage, XSS can steal them | Explicitly enforce `httpOnly Secure SameSite=Strict` cookie storage in the auth implementation spec |
| No data residency enforcement | Analytics click data may leave the designated region if Kafka or ClickHouse is misconfigured | Tag data with region at ingest; enforce regional topic/table routing; required for GDPR |

---

*Informed by: [SYSTEM_DESIGN.md §11](SYSTEM_DESIGN.md), [LLD.md §12 §16](LLD.md), [DATABASE_DESIGN.md §4.5](DATABASE_DESIGN.md), [API_CONTRACT.md §3 §5](API_CONTRACT.md).*
*Next: [PRIVACY.md](PRIVACY.md) — GDPR/CCPA data mapping and right-to-erasure flow.*
