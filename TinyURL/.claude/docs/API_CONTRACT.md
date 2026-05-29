# API Versioning Contract: TinyURL Service

> Formal API contract for the TinyURL Enterprise Link Management Infrastructure.
> Source of truth for all consumer integrations. Implementation detail lives in [LLD.md](LLD.md).

---

## 1. Versioning Strategy

### Scheme: URI Path Prefix

All API endpoints are versioned by a prefix segment in the URL path.

```
https://{host}/api/v{N}/{resource}
```

**Examples:**
```
https://tinyurl.internal/api/v1/urls
https://tinyurl.internal/api/v2/urls   ← future major version
```

**Why URI prefix, not headers:**
Header-based versioning (`Accept: application/vnd.tinyurl.v1+json`) is less visible, harder to test in browsers, and breaks cache keying. URI versioning is explicit, cache-friendly, and trivially routeable at the load balancer.

### Version Lifecycle

| Stage | Meaning | SLA |
|---|---|---|
| `current` | Actively developed, fully supported | Full |
| `deprecated` | No new features; bug fixes only | 6-month sunset window |
| `sunset` | Returns `410 Gone` on all endpoints | None |

The current major version is **v1**. No v2 exists yet.

### Version Header (informational, not routing)

Every response includes:
```
API-Version: 1
Deprecation: false           ← "true" when version is in sunset window
Sunset: <RFC 7231 date>      ← only present when Deprecation: true
```

---

## 2. Base URLs

| Environment | Base URL |
|---|---|
| Production | `https://api.tinyurl.com` |
| Staging | `https://api.staging.tinyurl.internal` |
| Local dev | `http://localhost:8080` |

The redirect hot path does **not** go through `/api/v1`. It runs at the root:
```
GET https://tinyurl.com/{short_code}   ← redirect engine
```

---

## 3. Authentication

### Mechanism: Bearer JWT

```
Authorization: Bearer <access_token>
```

- Tokens are signed with RS256 (asymmetric).
- Access token TTL: **15 minutes**.
- Refresh token TTL: **30 days** (rotated on each use).
- Token endpoint: `POST /api/v1/auth/token`

### Endpoint Access Matrix

| Endpoint | Anonymous | Authenticated |
|---|---|---|
| `GET /{short_code}` (redirect) | ✅ | ✅ |
| `POST /api/v1/urls` | ✅ (rate-limited) | ✅ |
| `GET /api/v1/urls/{code}` | ❌ | ✅ (owner only) |
| `DELETE /api/v1/urls/{code}` | ❌ | ✅ (owner only) |
| `GET /api/v1/urls/{code}/analytics` | ❌ | ✅ (owner only) |
| `GET /api/v1/urls` | ❌ | ✅ (own URLs only) |

---

## 4. Standard Response Envelope

### Success

Responses use the HTTP status code as the success signal. No wrapper object for single resources — return the resource directly.

```json
// 200 OK — single resource
{
  "short_code": "aB3xYz7",
  "short_url":  "https://tinyurl.com/aB3xYz7",
  "long_url":   "https://example.com/very/long/path",
  "created_at": "2026-05-29T10:00:00Z",
  "expires_at": "2026-05-30T00:00:00Z"
}
```

```json
// 200 OK — list resource
{
  "data": [...],
  "pagination": {
    "cursor":   "eyJpZCI6MTAwfQ==",
    "has_more": true,
    "limit":    50
  }
}
```

### Error Envelope

All errors return a consistent JSON body regardless of HTTP status:

```json
{
  "error": {
    "code":       "URL_NOT_FOUND",
    "message":    "No active URL found for short code 'aB3xYz7'.",
    "request_id": "req_01hwx3k9vb4f6c"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `code` | string | Machine-readable error constant (see §7) |
| `message` | string | Human-readable description, safe to display |
| `request_id` | string | Unique per-request ID for log correlation |
| `details` | object? | Optional field-level validation errors |

**Validation error example:**
```json
{
  "error": {
    "code":    "VALIDATION_ERROR",
    "message": "Request body contains invalid fields.",
    "request_id": "req_01hwx3k9vb4f6c",
    "details": {
      "long_url":   "must be a valid http or https URL",
      "expires_in": "must be a positive integer"
    }
  }
}
```

---

## 5. Rate Limiting

Every response includes rate-limit state headers:

```
X-RateLimit-Limit:     100
X-RateLimit-Remaining: 87
X-RateLimit-Reset:     1748512800    ← Unix epoch when window resets
Retry-After:           42            ← only present on 429 responses
```

### Limits by Tier

| Tier | Shorten (writes) | Redirect reads |
|---|---|---|
| Anonymous (per IP) | 10 / hour | Unlimited |
| Free user | 100 / day | Unlimited |
| Pro user | 10,000 / day | Unlimited |
| Enterprise | Custom (negotiated) | Unlimited |

When a limit is exceeded the server returns `429 Too Many Requests` with `Retry-After`.

---

## 6. Endpoints

### 6.1 Redirect (Hot Path)

```
GET /{short_code}
Host: tinyurl.com
```

This endpoint is **not versioned** — it is a permanent, stable contract. Short codes are forever.

**Response:**

| Status | Condition |
|---|---|
| `302 Found` | Active URL found; `Location` header set to `long_url` |
| `301 Moved Permanently` | Used only when click analytics are disabled for the link |
| `404 Not Found` | Short code does not exist |
| `410 Gone` | Short code existed but has been deleted or expired |

```
HTTP/1.1 302 Found
Location: https://example.com/very/long/path
X-Request-ID: req_01hwx3k9vb4f6c
```

No body is returned on redirect responses.

---

### 6.2 Create Short URL

```
POST /api/v1/urls
Content-Type: application/json
Authorization: Bearer <token>   (optional for anonymous)
```

**Request body:**

| Field | Type | Required | Constraints |
|---|---|---|---|
| `long_url` | string | ✅ | Valid http/https URL, max 2048 chars |
| `custom_code` | string | ❌ | 3–20 chars, `[a-zA-Z0-9-]` only |
| `expires_in` | integer | ❌ | Seconds from now; min 60, max 31536000 (1 year) |

```json
{
  "long_url":    "https://example.com/very/long/url?with=params",
  "custom_code": "my-sale",
  "expires_in":  86400
}
```

**Response `201 Created`:**

```json
{
  "short_code": "my-sale",
  "short_url":  "https://tinyurl.com/my-sale",
  "long_url":   "https://example.com/very/long/url?with=params",
  "created_at": "2026-05-29T10:00:00Z",
  "expires_at": "2026-05-30T10:00:00Z"
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Malformed body or field constraint violated |
| `409` | `ALIAS_CONFLICT` | `custom_code` already taken |
| `422` | `URL_BLOCKED` | URL matched blacklist or Safe Browsing |
| `429` | `RATE_LIMITED` | Write limit exceeded for tier |

---

### 6.3 Get URL Info

```
GET /api/v1/urls/{short_code}
Authorization: Bearer <token>
```

**Path parameters:**

| Parameter | Type | Description |
|---|---|---|
| `short_code` | string | The 7-char (or custom) short code |

**Response `200 OK`:**

```json
{
  "short_code":  "aB3xYz7",
  "short_url":   "https://tinyurl.com/aB3xYz7",
  "long_url":    "https://example.com/...",
  "created_at":  "2026-05-29T10:00:00Z",
  "expires_at":  null,
  "click_count": 1042,
  "is_active":   true
}
```

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| `401` | `UNAUTHORIZED` | Missing or invalid token |
| `403` | `FORBIDDEN` | Token valid but caller does not own this URL |
| `404` | `URL_NOT_FOUND` | Short code does not exist |

---

### 6.4 List URLs

```
GET /api/v1/urls?limit=50&cursor=<opaque>
Authorization: Bearer <token>
```

**Query parameters:**

| Parameter | Type | Default | Constraints |
|---|---|---|---|
| `limit` | integer | 50 | 1–200 |
| `cursor` | string | — | Opaque pagination cursor from previous response |

**Response `200 OK`:**

```json
{
  "data": [
    {
      "short_code":  "aB3xYz7",
      "short_url":   "https://tinyurl.com/aB3xYz7",
      "long_url":    "https://example.com/...",
      "created_at":  "2026-05-29T10:00:00Z",
      "expires_at":  null,
      "click_count": 1042
    }
  ],
  "pagination": {
    "cursor":   "eyJpZCI6MTAwfQ==",
    "has_more": true,
    "limit":    50
  }
}
```

Pagination is cursor-based (not page/offset) to remain stable under concurrent inserts.

---

### 6.5 Delete URL

```
DELETE /api/v1/urls/{short_code}
Authorization: Bearer <token>
```

**Response `204 No Content`** — no body.

Side effects:
- Sets `is_active = false` (soft delete; record retained for analytics).
- Evicts `url:{short_code}` from Redis immediately.
- Subsequent `GET /{short_code}` redirects return `410 Gone`.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| `401` | `UNAUTHORIZED` | Missing or invalid token |
| `403` | `FORBIDDEN` | Caller does not own this URL |
| `404` | `URL_NOT_FOUND` | Short code does not exist |

---

### 6.6 Get Analytics

```
GET /api/v1/urls/{short_code}/analytics?from=2026-05-01&to=2026-05-29
Authorization: Bearer <token>
```

**Query parameters:**

| Parameter | Type | Required | Constraints |
|---|---|---|---|
| `from` | string (ISO 8601 date) | ✅ | `YYYY-MM-DD`; must be ≤ `to` |
| `to` | string (ISO 8601 date) | ✅ | `YYYY-MM-DD`; max range 90 days |

**Response `200 OK`:**

```json
{
  "short_code":   "aB3xYz7",
  "total_clicks": 5823,
  "by_day": [
    { "date": "2026-05-29", "clicks": 312 }
  ],
  "by_country": [
    { "country_code": "US", "clicks": 2100 },
    { "country_code": "IN", "clicks": 1200 }
  ],
  "top_referrers": [
    { "referrer": "twitter.com", "clicks": 900 }
  ]
}
```

Analytics data is **eventually consistent** — click events are ingested asynchronously via Kafka and may lag real-time by up to 60 seconds.

**Error responses:**

| Status | Code | Condition |
|---|---|---|
| `400` | `VALIDATION_ERROR` | Invalid date format or range exceeds 90 days |
| `401` | `UNAUTHORIZED` | Missing or invalid token |
| `403` | `FORBIDDEN` | Caller does not own this URL |
| `404` | `URL_NOT_FOUND` | Short code does not exist |

---

## 7. Error Code Reference

| Code | HTTP Status | Description |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Request body or query parameter failed validation |
| `UNAUTHORIZED` | 401 | Missing, malformed, or expired auth token |
| `FORBIDDEN` | 403 | Authenticated but not authorized for this resource |
| `URL_NOT_FOUND` | 404 | Short code does not exist |
| `ALIAS_CONFLICT` | 409 | Requested `custom_code` is already in use |
| `URL_BLOCKED` | 422 | URL matched domain blacklist or Safe Browsing |
| `RATE_LIMITED` | 429 | Request rate limit exceeded for this tier |
| `INTERNAL_ERROR` | 500 | Unexpected server error; check `request_id` in logs |
| `SERVICE_UNAVAILABLE` | 503 | Dependency unavailable (DB, cache); retry with backoff |

---

## 8. Breaking vs. Non-Breaking Changes

### Breaking (requires a new major version `vN+1`)

- Removing an endpoint
- Removing or renaming a required request field
- Removing or renaming a response field that consumers depend on
- Changing a field's type (e.g., `string` → `integer`)
- Changing the HTTP method of an existing endpoint
- Changing the HTTP status code of a success response
- Changing the authentication scheme
- Narrowing a field's accepted value range

### Non-Breaking (safe to ship in current version)

- Adding a new optional request field
- Adding a new response field (consumers must tolerate unknown fields)
- Adding a new endpoint
- Adding a new error code
- Expanding an accepted value range
- Improving error `message` text (only `code` is a stable contract)
- Performance improvements with no observable contract change

---

## 9. Deprecation Policy

1. **Announcement** — Deprecated versions are announced via email to registered API key holders and posted in the changelog with at least **6 months notice**.
2. **Header signal** — Responses from deprecated versions include `Deprecation: true` and `Sunset: <RFC 7231 date>` headers.
3. **Sunset** — On the sunset date, all endpoints in the deprecated version return `410 Gone`:
   ```json
   {
     "error": {
       "code":       "VERSION_SUNSET",
       "message":    "API v1 has been sunset. Please migrate to v2.",
       "request_id": "req_01hwx3k9vb4f6c"
     }
   }
   ```
4. **No extensions** — Sunset dates are hard. Emergency extensions are not granted.

---

## 10. Changelog

### v1.0 — 2026-05-29 (initial)

- `POST /api/v1/urls` — create short URL
- `GET /api/v1/urls/{code}` — get URL info
- `GET /api/v1/urls` — list URLs (cursor-paginated)
- `DELETE /api/v1/urls/{code}` — soft-delete URL
- `GET /api/v1/urls/{code}/analytics` — click analytics (90-day window)
- `GET /{short_code}` — redirect hot path (unversioned, permanent)
- Standard error envelope with machine-readable `code` field
- Cursor-based pagination on list endpoint
- `API-Version`, `Deprecation`, `Sunset` response headers

---

*Partial content expanded from [LLD.md §7](LLD.md). Implementation internals (caching, key generation, handler code) remain in LLD.*
