# Testing Strategy: TinyURL Service

> Practical, integration-first strategy for the Go backend + Next.js frontend stack.
> The highest ROI comes from integration tests that exercise real HTTP handlers
> against real dependencies (PostgreSQL, Redis) spun up via Testcontainers.
> Unit tests cover pure logic only. E2E and load tests cover the thin outer shell.

---

## 1. Philosophy: Integration Tests as the Core

The classic test pyramid is often misapplied to backend services. For TinyURL:

```
          /\
         /E2E\          ← few; smoke-test critical user journeys only
        /──────\
       / Load   \       ← validate PRD success metrics (50k RPS, p99 < 10ms)
      /──────────\
     / Integration\     ← THIS IS WHERE THE VALUE IS
    /──────────────\
   /   Unit Tests   \   ← pure logic only; don't mock what you can run for real
  /──────────────────\
```

**Why integration tests dominate here:**

- The redirect handler is ~15 lines of Go — the risk is not in the code, it's in the
  interaction: does the cache key format match what was written? Does a cache miss
  correctly fall through to the DB? Does a soft-deleted URL return `410` not `404`?
- Mocking Redis and PostgreSQL hides exactly these failure modes.
- With `testcontainers-go`, a real PostgreSQL + Redis instance starts in ~2s and gives
  complete confidence at reasonable speed.

**Rule of thumb:** If a test needs a mock of Redis, PostgreSQL, or Kafka — it is testing
the wrong thing. Use testcontainers and test the real interaction.

---

## 2. Test Layers

### Layer 1 — Unit Tests

**Cover:** Pure functions with no I/O dependencies. Nothing else.

**Do not write unit tests for:** HTTP handlers, DB queries, cache interactions — those
belong in integration tests.

| What to test | Package | Why unit |
|---|---|---|
| `base62Encode(n uint64) string` | `pkg/encoding` | Pure math, no I/O |
| `base62Decode(s string) uint64` | `pkg/encoding` | Pure math, no I/O |
| URL validation pipeline (scheme, IP range, length) | `pkg/validation` | Pure logic, injectable inputs |
| Reserved alias set lookup | `pkg/validation` | In-memory set, no I/O |
| Token bucket counter logic | `pkg/ratelimit` | Logic only; Redis interaction tested in integration |
| JWT sign / verify | `pkg/auth` | Crypto, no I/O |
| IP hashing (SHA-256 + salt) | `pkg/analytics` | Pure function |
| Short code collision retry loop | `pkg/shortener` | Logic with injected ID generator stub |

**Tools:** Go standard `testing` package. Table-driven tests.

```go
// Example: base62 round-trip property test
func TestBase62RoundTrip(t *testing.T) {
    cases := []uint64{0, 1, 61, 62, 3_500_000_000, math.MaxUint32}
    for _, n := range cases {
        got, err := base62Decode(base62Encode(n))
        require.NoError(t, err)
        assert.Equal(t, n, got, "round-trip failed for %d", n)
    }
}
```

**Coverage target:** ≥ 90% line coverage on `pkg/encoding`, `pkg/validation`, `pkg/auth`.
These are pure — anything less is a gap.

---

### Layer 2 — Integration Tests

**Cover:** Every HTTP endpoint against a real PostgreSQL + Redis. This is the primary
safety net for the service.

**Tools:**
- [`testcontainers-go`](https://golang.testcontainers.org/) — real Docker containers per test suite
- `net/http/httptest` — drives the Fiber/Chi app without binding a port
- [`testify`](https://github.com/stretchr/testify) — assertions and `require`

**Setup pattern:** One PostgreSQL + one Redis container shared across all tests in the
package via `TestMain`. Schema migrated once at startup. Each test runs inside a
transaction that is rolled back on completion — no test pollution, no truncation loops.

```go
func TestMain(m *testing.M) {
    ctx := context.Background()
    pg, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "postgres:16-alpine",
            ExposedPorts: []string{"5432/tcp"},
            Env:          map[string]string{"POSTGRES_PASSWORD": "test"},
            WaitingFor:   wait.ForListeningPort("5432/tcp"),
        },
        Started: true,
    })
    // run migrations, wire app, call m.Run()
    os.Exit(m.Run())
}
```

#### Scenario Table — Shorten API (`POST /api/v1/urls`)

| Scenario | Expected result |
|---|---|
| Valid URL, anonymous | 201; `short_code` in response; row in DB; key in Redis |
| Valid URL, authenticated user | 201; `user_id` set on the DB row |
| Invalid scheme (`ftp://`) | 400 VALIDATION_ERROR |
| Private IP in submitted URL (`http://10.0.0.1`) | 400 VALIDATION_ERROR |
| Loopback IP (`http://127.0.0.1`) | 400 VALIDATION_ERROR |
| IMDS URL (`http://169.254.169.254`) | 400 VALIDATION_ERROR |
| URL length > 2048 chars | 400 VALIDATION_ERROR |
| Blacklisted domain | 422 URL_BLOCKED |
| Custom alias, first use | 201; `short_code == custom_code` |
| Custom alias, already taken | 409 ALIAS_CONFLICT |
| Reserved alias (`admin`, `login`) | 409 ALIAS_CONFLICT |
| Anonymous exceeds rate limit | 429 RATE_LIMITED; `Retry-After` header present |
| `expires_in` provided | `expires_at` correct in DB and response |
| Error response never echoes submitted URL | `message` field contains no URL fragment |

#### Scenario Table — Redirect (`GET /{short_code}`)

| Scenario | Expected result |
|---|---|
| Cache hit, active URL | 302; `Location` = `long_url`; Kafka event enqueued |
| Cache miss, DB hit, active URL | 302; Redis key now present after response |
| Cache miss, DB miss | 404 |
| Cache miss, URL expired | 410 Gone; Redis key deleted |
| Cache hit, URL soft-deleted | 410 Gone |
| Short code that never existed | 404 — same response shape as deleted (no enumeration leak) |

#### Scenario Table — Auth & Ownership (IDOR)

| Scenario | Expected result |
|---|---|
| User A reads user B's URL (`GET /api/v1/urls/{code}`) | 403 FORBIDDEN |
| User A deletes user B's URL | 403 FORBIDDEN |
| User A reads own URL | 200 |
| User A requests analytics for user B's URL | 403 FORBIDDEN |
| Expired access token on any auth endpoint | 401 UNAUTHORIZED |
| Refresh token reused (second use of same token) | 401; all sessions for user revoked |

#### Scenario Table — Delete (`DELETE /api/v1/urls/{short_code}`)

| Scenario | Expected result |
|---|---|
| Owner deletes own URL | 204; `is_active = false` in DB; Redis key evicted |
| Subsequent redirect of deleted code | 410 Gone |
| Non-owner attempts delete | 403 FORBIDDEN |

#### Scenario Table — Analytics (`GET /api/v1/urls/{code}/analytics`)

| Scenario | Expected result |
|---|---|
| Valid date range (≤ 90 days) | 200; `total_clicks` matches seeded click events |
| Date range > 90 days | 400 VALIDATION_ERROR |
| Non-owner request | 403 FORBIDDEN |
| `ip_hash` absent from response body | Confirmed — field must never appear in API response |

#### Scenario Table — Expiry Cleanup Job

| Scenario | Expected result |
|---|---|
| 3 expired URLs + 2 active URLs in DB | Expired → `is_active = false`; Redis keys evicted; active rows untouched |
| Job runs again after cleanup | 0 rows processed |
| Expired URL accessed during/after cleanup | 410 Gone |

**Coverage target:** ≥ 80% line coverage on handler and service packages. The real target
is 100% of the scenario tables above — line coverage follows from that.

---

### Layer 3 — Contract Tests

**Cover:** Confirm every API response shape exactly matches API_CONTRACT.md — no extra
fields, no missing required fields, no format drift.

**Approach:** Decode each response into a strict struct. Separately assert that all error
responses include `code`, `message`, and `request_id`.

```go
func TestErrorEnvelopeShape(t *testing.T) {
    resp := postShorten(t, `{"long_url":"ftp://invalid"}`)
    require.Equal(t, 400, resp.StatusCode)

    var body struct {
        Error struct {
            Code      string `json:"code"`
            Message   string `json:"message"`
            RequestID string `json:"request_id"`
        } `json:"error"`
    }
    require.NoError(t, json.NewDecoder(resp.Body).Decode(&body))
    assert.Equal(t, "VALIDATION_ERROR", body.Error.Code)
    assert.NotEmpty(t, body.Error.RequestID)
    // URL must never be echoed back in the error message
    assert.NotContains(t, body.Error.Message, "ftp://")
}
```

---

### Layer 4 — E2E Tests (Playwright, Next.js UI)

**Cover:** Critical user journeys only. Not a substitute for integration tests.

**Tools:** Playwright (TypeScript) against a local `docker-compose` stack.

| Journey | Pass condition |
|---|---|
| Anonymous shortens a link | Short URL displayed; clicking it redirects to destination |
| Authenticated user shortens + views in dashboard | URL in dashboard; click count = 0 |
| User deletes a link | URL gone from list; old short link returns 410 |
| Analytics view | Chart renders; no raw IP visible anywhere in DOM |

**Run in CI:** On every PR to `main`. Failures block merge.

**Do not use E2E for:** Error handling edge cases, rate limiting, IDOR — too slow and
brittle at this layer. Those belong in integration tests.

---

### Layer 5 — Load Tests (k6)

**Cover:** Validate all six PRD success metrics under real traffic patterns.

**Tool:** [k6](https://k6.io/) with Prometheus remote write output.

#### Scenario 1 — Redirect Sustained Load (primary SLO gate)

```js
export const options = {
  stages: [
    { duration: '2m', target: 10000 },   // ramp up
    { duration: '10m', target: 50000 },  // sustain at 50k RPS
    { duration: '2m', target: 0 },       // ramp down
  ],
  thresholds: {
    'http_req_duration{type:redirect}': ['p(99)<10'],  // PRD: p99 < 10ms
    'http_req_failed': ['rate<0.0001'],                // PRD: 99.99% availability
  },
};

export default function () {
  const code = HOT_CODES[Math.floor(Math.random() * HOT_CODES.length)];
  const res = http.get(`http://api/${code}`, {
    redirects: 0,
    tags: { type: 'redirect' },
  });
  check(res, { 'is 302': (r) => r.status === 302 });
}
```

#### Scenario 2 — Cache Miss Storm

100k unique codes pre-seeded in DB but not in Redis. 50k RPS of random requests.
**Pass condition:** p99 < 50ms (DB fallback is slower than cache hit, but must not cascade).

#### Scenario 3 — Write Throughput

5k write RPS sustained for 5 minutes. **Pass condition:** p99 shorten latency < 200ms;
error rate < 0.1%.

#### Scenario 4 — 72-Hour Soak (pre-launch gate, run once)

Full 50k RPS redirect load for 72 hours. Checks for memory leaks, Redis key accumulation,
Kafka consumer lag growth, and DB connection pool exhaustion.
**Pass condition:** All PRD metrics hold for the full 72-hour window with no degradation trend.

#### Load Targets (from PRD)

| Metric | Target | Measured in |
|---|---|---|
| p99 redirect latency | < 10 ms | Scenario 1 |
| Throughput ceiling | ≥ 50,000 RPS | Scenario 1 |
| Cache hit rate | ≥ 95% | Scenario 1 (Redis `INFO stats`) |
| Analytics data loss | < 0.1% | Scenario 1 (emitted vs stored event count) |
| Redirect availability | ≥ 99.99% | Scenario 4 (soak) |
| Infra cost vs SaaS | < 15% | Scenario 4 (billing export post-soak) |

---

## 3. What Not to Test

| Temptation | Why to skip |
|---|---|
| Unit-test the Fiber/Chi router | Framework already tested; you'd be testing your config |
| Mock Redis in handler tests | Masks key-format bugs and TTL misconfiguration |
| Mock PostgreSQL in handler tests | Masks schema drift, index misses, and constraint behaviour |
| Unit-test SQL query strings | Integration tests catch these with real query execution |
| E2E-test every error state | Too slow and brittle — that's integration test territory |
| Chase 100% line coverage | Produces tests for trivial getters; focus on scenario coverage |

---

## 4. CI Pipeline Placement

```
Every commit (push / PR open)
  1. go vet + staticcheck                          < 30s
  2. Unit tests  (go test ./pkg/...)               < 60s
  3. Integration tests  (testcontainers)           < 3min
  4. Contract tests                                < 30s
                │ all pass → unblock PR merge

PR merged to main
  5. E2E tests  (Playwright, docker-compose)       < 5min
                │ pass → allow deploy to staging

Staging deploy
  6. Load Scenario 1 (50k RPS, 10min sustained)
  7. Load Scenario 2 (cache miss storm)
                │ pass → allow deploy to production

Pre-launch gate  (run once before public traffic)
  8. 72-hour soak test (Scenario 4)
                │ pass → open to production traffic
```

**Fail policy:** Steps 1–4 block PR merge. Steps 5–7 block production deploy. Step 8
blocks opening to public traffic.

---

## 5. Coverage Targets

| Layer | Scope | Target |
|---|---|---|
| Unit | `pkg/encoding`, `pkg/validation`, `pkg/auth`, `pkg/analytics` | ≥ 90% line |
| Integration | Handler + service packages | ≥ 80% line; 100% scenario table rows |
| Contract | All API_CONTRACT.md response shapes | 100% of documented fields verified |
| E2E | 4 critical user journeys | 100% pass on every PR to `main` |
| Load | PRD success metrics | 100% of targets met before launch gate |

---

*Informed by: [PRD.md §Success Metrics](PRD.md), [API_CONTRACT.md](API_CONTRACT.md), [SECURITY.md §C1](SECURITY.md), [LLD.md §17](LLD.md).*
